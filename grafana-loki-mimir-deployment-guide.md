# Grafana, Loki, Mimir, and Alloy Deployment Guide

This guide provides an end-to-end deployment flow for a centralized observability stack using Grafana, Loki, Mimir, Grafana Alloy, and AWS S3.

## Architecture

One EC2 instance acts as the monitoring hub:

- Runs Grafana for dashboards, querying, and visualization.
- Runs Loki for log ingestion and querying.
- Runs Mimir for metrics ingestion and querying.
- Uses AWS S3 for Loki and Mimir object storage.

Each application EC2 instance runs a lightweight Docker-based Grafana Alloy agent:

- Sends logs to Loki.
- Sends metrics to Mimir.
- Does not run Grafana, Loki, or Mimir.
- Does not need direct S3 access.

Data flow:

```text
Application EC2
  Laravel/Nginx/PHP-FPM logs
  System/Nginx/PHP-FPM metrics
        |
        v
  Grafana Alloy
        |
        | logs:    http://HUB_PRIVATE_IP:3100/loki/api/v1/push
        | metrics: http://HUB_PRIVATE_IP:9009/api/v1/push
        v
Monitoring Hub EC2
  Loki  -> S3
  Mimir -> S3
  Grafana queries Loki and Mimir
```

Recommended paths:

| Server             | Path                                | Purpose                            |
| ------------------ | ----------------------------------- | ---------------------------------- |
| Monitoring hub EC2 | `/opt/grafana-main-hub`             | Main stack repository              |
| Monitoring hub EC2 | `/opt/grafana-main-hub/main-server` | Grafana, Loki, Mimir compose stack |
| Client EC2         | `/opt/grafana-nodes`                | Client repository copy             |
| Client EC2         | `/opt/grafana-nodes/alloy-docker`   | Alloy agent stack                  |

## Requirements

- AWS CLI configured locally or on an admin workstation.
- One private S3 bucket for observability storage.
- One EC2 instance for the monitoring hub.
- One or more client EC2 instances.
- Docker and Docker Compose plugin on every EC2 instance.
- Security groups that allow client instances to reach the monitoring hub on Loki and Mimir ports.

This guide assumes AWS region `ap-southeast-1` (Asia Pacific, Singapore).

## Deployment Variables

Set these values before running AWS commands:

```bash
export AWS_REGION="ap-southeast-1"
export OBS_BUCKET="S3_BUCKET_NAME"
export AWS_ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
export HUB_INSTANCE_ID="i-xxxxxxxxxxxxxxxxx"
export GRAFANA_ROOT_URL="http://MONITORING_HUB_PUBLIC_IP/"
export GRAFANA_ADMIN_PASSWORD="CHANGE_THIS_PASSWORD"
```

Replace:

- `AWS_REGION` only if you intentionally deploy outside `ap-southeast-1`.
- `OBS_BUCKET` with a globally unique S3 bucket name.
- `HUB_INSTANCE_ID` with the monitoring hub EC2 instance ID.
- `GRAFANA_ROOT_URL` with the public URL, private URL, or DNS name used to access Grafana.
- `GRAFANA_ADMIN_PASSWORD` with a strong password.

## 1. Create the S3 Bucket

Create one private S3 bucket for Loki and Mimir storage.

Create the bucket in `ap-southeast-1`:

```bash
aws s3api create-bucket \
  --bucket "$OBS_BUCKET" \
  --region "$AWS_REGION" \
  --create-bucket-configuration LocationConstraint="$AWS_REGION"
```

Enable versioning:

```bash
aws s3api put-bucket-versioning \
  --bucket "$OBS_BUCKET" \
  --versioning-configuration Status=Enabled
```

If you enable versioning, the lifecycle policy must also expire noncurrent object versions. Otherwise S3 expiration only creates delete markers and old object versions can continue consuming storage.

Block all public access:

```bash
aws s3api put-public-access-block \
  --bucket "$OBS_BUCKET" \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```

Enable default server-side encryption:

```bash
aws s3api put-bucket-encryption \
  --bucket "$OBS_BUCKET" \
  --server-side-encryption-configuration '{
    "Rules": [{"ApplyServerSideEncryptionByDefault": {"SSEAlgorithm": "AES256"}}]
  }'
```

Verify:

```bash
aws s3api get-bucket-location --bucket "$OBS_BUCKET"
aws s3api get-public-access-block --bucket "$OBS_BUCKET"
aws s3api get-bucket-encryption --bucket "$OBS_BUCKET"
```

## 2. Configure S3 Retention

Use S3 lifecycle rules to control storage cost and object expiration.

Recommended 100-day retention:

- `0-30 days`: S3 Standard.
- `30-100 days`: S3 Intelligent-Tiering.
- `100+ days`: deleted.

Create the lifecycle policy:

```bash
mkdir -p /tmp/grafana-aws

cat > /tmp/grafana-aws/s3-lifecycle-policy.json << 'EOJSON'
{
  "Rules": [
    {
      "ID": "ObservabilityDataLifecycle",
      "Status": "Enabled",
      "Filter": { "Prefix": "" },
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "INTELLIGENT_TIERING"
        }
      ],
      "Expiration": {
        "Days": 100
      },
      "NoncurrentVersionExpiration": {
        "NoncurrentDays": 7
      }
    }
  ]
}
EOJSON
```

Apply it:

```bash
aws s3api put-bucket-lifecycle-configuration \
  --bucket "$OBS_BUCKET" \
  --lifecycle-configuration file:///tmp/grafana-aws/s3-lifecycle-policy.json
```

Verify:

```bash
aws s3api get-bucket-lifecycle-configuration --bucket "$OBS_BUCKET"
```

Important: Loki and Mimir retention should also match this S3 lifecycle policy. Do not rely only on S3 lifecycle deletion. The application configs should be set to retain data for the same period, for example `100d`.

## 3. Create IAM Policy and Role for the Monitoring Hub

Only the monitoring hub EC2 needs S3 access. Client EC2 instances do not need S3 permissions.

Create the S3 IAM policy document:

```bash
mkdir -p /tmp/grafana-aws

cat > /tmp/grafana-aws/iam-policy-grafana-s3.json << EOJSON
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "GrafanaS3BucketAccess",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetBucketLocation"
      ],
      "Resource": "arn:aws:s3:::$OBS_BUCKET"
    },
    {
      "Sid": "GrafanaS3ObjectAccess",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject"
      ],
      "Resource": [
        "arn:aws:s3:::$OBS_BUCKET/*"
      ]
    }
  ]
}
EOJSON
```

Create the IAM policy:

```bash
aws iam create-policy \
  --policy-name GrafanaS3Access \
  --policy-document file:///tmp/grafana-aws/iam-policy-grafana-s3.json
```

If a policy with this name already exists, reuse its ARN or create a new version instead of creating a duplicate policy with a different name.

Create the EC2 trust policy:

```bash
cat > /tmp/grafana-aws/ec2-trust-policy.json << 'EOJSON'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "ec2.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOJSON
```

Create the IAM role:

```bash
aws iam create-role \
  --role-name GrafanaServerRole \
  --assume-role-policy-document file:///tmp/grafana-aws/ec2-trust-policy.json
```

Attach the `GrafanaS3Access` policy:

```bash
aws iam attach-role-policy \
  --role-name GrafanaServerRole \
  --policy-arn "arn:aws:iam::$AWS_ACCOUNT_ID:policy/GrafanaS3Access"
```

Create and attach an instance profile:

```bash
aws iam create-instance-profile \
  --instance-profile-name GrafanaServerProfile

aws iam add-role-to-instance-profile \
  --instance-profile-name GrafanaServerProfile \
  --role-name GrafanaServerRole

aws ec2 associate-iam-instance-profile \
  --region "$AWS_REGION" \
  --instance-id "$HUB_INSTANCE_ID" \
  --iam-instance-profile Name=GrafanaServerProfile
```

If the EC2 already has an instance profile attached, use `replace-iam-instance-profile-association` instead of `associate-iam-instance-profile`:

```bash
ASSOCIATION_ID="$(aws ec2 describe-iam-instance-profile-associations \
  --region "$AWS_REGION" \
  --filters Name=instance-id,Values="$HUB_INSTANCE_ID" \
  --query 'IamInstanceProfileAssociations[0].AssociationId' \
  --output text)"

aws ec2 replace-iam-instance-profile-association \
  --region "$AWS_REGION" \
  --association-id "$ASSOCIATION_ID" \
  --iam-instance-profile Name=GrafanaServerProfile
```

Verify from the monitoring hub EC2:

```bash
aws sts get-caller-identity
aws s3 ls "s3://$OBS_BUCKET"
```

## 4. Configure Security Groups

Use private networking wherever possible. Do not expose Loki or Mimir publicly.
Loki and Mimir are unauthenticated in this single-node configuration, so security group restrictions are mandatory. Keep internal gRPC ports `9095` and `9096` closed to the public internet; the provided Compose file does not publish them.

Recommended security groups:

- `grafana-main-hub`: attached to the monitoring hub EC2.
- `grafana-client`: attached to application/client EC2 instances.

Monitoring hub inbound rules:

| Type       |   Port | Source                          | Purpose                       |
| ---------- | -----: | ------------------------------- | ----------------------------- |
| HTTP       |   `80` | Office/VPN CIDR                 | Grafana UI                    |
| SSH        |   `22` | Office/VPN CIDR                 | Admin SSH                     |
| Custom TCP | `3100` | `grafana-client` security group | Loki ingest from Alloy        |
| Custom TCP | `9009` | `grafana-client` security group | Mimir remote write from Alloy |

Client EC2 inbound rules:

| Type                 |         Port | Source                        | Purpose                      |
| -------------------- | -----------: | ----------------------------- | ---------------------------- |
| SSH                  |         `22` | Office/VPN CIDR               | Admin SSH                    |
| HTTP/HTTPS/app ports | app-specific | Load balancer or allowed CIDR | Existing application traffic |

Client EC2 outbound:

- Allow outbound to the hub private IP on `3100/tcp`.
- Allow outbound to the hub private IP on `9009/tcp`.

If default outbound is open, the clients can already reach the hub, assuming the hub inbound rules allow them.

Example AWS CLI rules:

```bash
export HUB_SG_ID="sg-hubxxxxxxxx"
export APP_SG_ID="sg-appxxxxxxxx"
export OFFICE_CIDR="x.x.x.x/32"

aws ec2 authorize-security-group-ingress \
  --region "$AWS_REGION" \
  --group-id "$HUB_SG_ID" \
  --protocol tcp \
  --port 3100 \
  --source-group "$APP_SG_ID"

aws ec2 authorize-security-group-ingress \
  --region "$AWS_REGION" \
  --group-id "$HUB_SG_ID" \
  --protocol tcp \
  --port 9009 \
  --source-group "$APP_SG_ID"

aws ec2 authorize-security-group-ingress \
  --region "$AWS_REGION" \
  --group-id "$HUB_SG_ID" \
  --protocol tcp \
  --port 80 \
  --cidr "$OFFICE_CIDR"

aws ec2 authorize-security-group-ingress \
  --region "$AWS_REGION" \
  --group-id "$HUB_SG_ID" \
  --protocol tcp \
  --port 22 \
  --cidr "$OFFICE_CIDR"
```

## 5. Set Up the Monitoring Hub EC2

Run these commands on the monitoring hub EC2.

Install Docker, Compose, Git, and AWS CLI.

Important: `apt upgrade -y` and Docker installation can restart services. On an existing production host, run this during a maintenance window and consider pinning Docker versions through the official Docker apt repository instead of using the convenience installer.

```bash
sudo apt update
sudo apt upgrade -y
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker "$USER"
sudo apt install -y docker-compose-plugin git awscli
```

Log out and log back in so the Docker group membership applies:

```bash
exit
```

Verify after logging in again:

```bash
docker --version
docker compose version
aws sts get-caller-identity
```

Clone the repository into the correct hub path:

```bash
cd /opt
sudo mkdir -p grafana-main-hub
sudo chown "$USER:$USER" grafana-main-hub
git clone https://github.com/irfanoneverse/grafana.git grafana-main-hub
cd /opt/grafana-main-hub/main-server
```

Before starting the stack, review these files:

```text
/opt/grafana-main-hub/main-server/docker-compose.yml
/opt/grafana-main-hub/main-server/loki-config.yaml
/opt/grafana-main-hub/main-server/mimir-config.yaml
/opt/grafana-main-hub/main-server/grafana/provisioning/datasources/datasources.yaml
```

Required values to update:

- Grafana admin password.
- Grafana root URL.
- Loki S3 bucket name.
- Loki S3 region and endpoint.
- Mimir S3 bucket name.
- Mimir S3 region and endpoint.
- Mimir `native_aws_auth_enabled: true` when using the EC2 IAM instance profile.
- Loki retention period.
- Mimir retention period.

Edit manually:

```bash
cp .env.example .env
nano .env
nano docker-compose.yml
nano loki-config.yaml
nano mimir-config.yaml
```

Expected Grafana environment values:

```yaml
GF_SECURITY_ADMIN_USER=admin
GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_ADMIN_PASSWORD:?set GRAFANA_ADMIN_PASSWORD in .env}
GF_USERS_ALLOW_SIGN_UP=false
GF_SERVER_ROOT_URL=${GRAFANA_ROOT_URL:?set GRAFANA_ROOT_URL in .env}
```

Example `main-server/.env`:

```env
GRAFANA_ADMIN_PASSWORD=CHANGE_THIS_PASSWORD
GRAFANA_ROOT_URL=http://MONITORING_HUB_PUBLIC_IP/
```

If Grafana is mapped as `80:3000`, users access Grafana on:

```text
http://MONITORING_HUB_PUBLIC_IP/
```

If Grafana is mapped as `3000:3000`, users access Grafana on:

```text
http://MONITORING_HUB_PUBLIC_IP:3000/
```

Start the stack:

```bash
cd /opt/grafana-main-hub/main-server
docker compose up -d
docker compose ps
```

Check logs:

```bash
docker compose logs --tail=80 grafana
docker compose logs --tail=80 loki
docker compose logs --tail=80 mimir
```

Expected containers:

- `grafana`
- `loki`
- `mimir`
- `alloy-hub`, if hub self-monitoring is enabled

## 6. Validate the Monitoring Hub

Run on the monitoring hub EC2:

```bash
curl -s http://127.0.0.1:80/api/health
curl -s http://127.0.0.1:3100/ready
curl -s http://127.0.0.1:9009/ready
```

Expected:

- Grafana returns health JSON.
- Loki returns a ready response.
- Mimir returns a ready response after startup.

Check S3 writes after the stack has been running:

```bash
aws s3 ls "s3://$OBS_BUCKET" --recursive --summarize --human-readable | tail
```

Verify write and delete permissions from the hub EC2:

```bash
echo test > /tmp/grafana-s3-test.txt
aws s3 cp /tmp/grafana-s3-test.txt "s3://$OBS_BUCKET/test/grafana-s3-test.txt"
aws s3 rm "s3://$OBS_BUCKET/test/grafana-s3-test.txt"
```

Open Grafana:

```text
http://MONITORING_HUB_PUBLIC_IP/
```

Default login:

- Username: `admin`
- Password: the value set in `GF_SECURITY_ADMIN_PASSWORD`.

In Grafana Explore:

- Select Loki and query `{job="laravel"}` after clients are connected.
- Select Mimir/Prometheus and query `up` after clients are connected.

## 7. Set Up Each Client EC2

Run these commands on each application/client EC2.

Install Docker, Compose, and Git.

Important: `apt upgrade -y` and Docker installation can restart services. On an existing production application host, run this during a maintenance window and consider pinning Docker versions through the official Docker apt repository instead of using the convenience installer.

```bash
sudo apt update
sudo apt upgrade -y
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker "$USER"
sudo apt install -y docker-compose-plugin git
```

Log out and log back in:

```bash
exit
```

Verify:

```bash
docker --version
docker compose version
docker run --rm hello-world
```

Set client-specific variables:

```bash
export HUB_IP="MONITORING_HUB_PRIVATE_IP"
export INSTANCE_NAME="app-prod-01"
export ENVIRONMENT="production"
export TZ_NAME="Asia/Kuala_Lumpur"
export LARAVEL_LOG_DIR="/path/to/laravel/storage/logs"
```

Use the monitoring hub private IP for `HUB_IP` when the client and hub are in the same VPC.

Check network access from the client EC2:

```bash
nc -vz "$HUB_IP" 3100
nc -vz "$HUB_IP" 9009
```

If `nc` is missing:

```bash
sudo apt update
sudo apt install -y netcat-openbsd
```

Clone the repository into the correct client path:

```bash
cd /opt
sudo mkdir -p grafana-nodes
sudo chown "$USER:$USER" grafana-nodes
git clone https://github.com/irfanoneverse/grafana.git grafana-nodes
cd /opt/grafana-nodes/alloy-docker
```

The expected final client path is:

```text
/opt/grafana-nodes/alloy-docker
```

## 8. Enable Nginx and PHP-FPM Status Endpoints

Alloy uses exporters to scrape local Nginx and PHP-FPM status endpoints.

Enable PHP-FPM status:

```bash
ls -lah /etc/php/*/fpm/pool.d/www.conf
grep -R "listen =" /etc/php/*/fpm/pool.d/www.conf

export PHP_VERSION="8.2"
export PHP_FPM_POOL="/etc/php/$PHP_VERSION/fpm/pool.d/www.conf"

sudo cp "$PHP_FPM_POOL" "$PHP_FPM_POOL.bak"
sudo sed -i 's#^;*pm.status_path = .*#pm.status_path = /fpm-status#' "$PHP_FPM_POOL"
grep -q '^pm.status_path' "$PHP_FPM_POOL" || echo 'pm.status_path = /fpm-status' | sudo tee -a "$PHP_FPM_POOL"
sudo systemctl reload "php$PHP_VERSION-fpm"
```

Replace `PHP_VERSION` with the active PHP-FPM version on the host. Avoid editing every PHP-FPM pool on mixed-version servers.

Confirm PHP-FPM is running:

```bash
systemctl --no-pager --type=service | grep php | grep fpm
ls -lah /run/php/
```

Find the active PHP-FPM socket:

```bash
ls /run/php/
```

Common examples:

```text
/run/php/php8.2-fpm.sock
/run/php/php8.3-fpm.sock
/run/php/php-fpm.sock
```

Set the socket path:

```bash
export PHP_FPM_SOCK="/run/php/php8.2-fpm.sock"
```

Create the local-only Nginx status server:

```bash
sudo tee /etc/nginx/conf.d/stub_status.conf > /dev/null << EONGINX
server {
    listen 127.0.0.1:8080;
    server_name localhost;

    location = /nginx_status {
        stub_status on;
        allow 127.0.0.1;
        deny all;
    }

    location = /fpm-status {
        fastcgi_pass unix:$PHP_FPM_SOCK;
        fastcgi_param SCRIPT_NAME     /fpm-status;
        fastcgi_param SCRIPT_FILENAME /fpm-status.php;
        fastcgi_param REQUEST_URI     /fpm-status;
        fastcgi_param QUERY_STRING    \$query_string;
        fastcgi_param REQUEST_METHOD  \$request_method;
        allow 127.0.0.1;
        deny all;
    }
}
EONGINX
```

Validate and reload Nginx:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

Verify endpoints:

```bash
curl -s http://127.0.0.1:8080/nginx_status
curl -s http://127.0.0.1:8080/fpm-status
```

Expected:

- `nginx_status` returns active connections and request counters.
- `fpm-status` returns PHP-FPM pool/process status.

If `fpm-status` returns `502`, update the `fastcgi_pass` socket path in `/etc/nginx/conf.d/stub_status.conf`, then reload Nginx.

## 9. Configure and Start Alloy on the Client

Create the runtime `.env` file:

```bash
cd /opt/grafana-nodes/alloy-docker
cp .env.example .env
nano .env
```

Example `.env`:

```env
HOSTNAME=app-prod-01
INSTANCE_NAME=app-prod-01
ENVIRONMENT=production
TZ_NAME=Asia/Kuala_Lumpur
LARAVEL_LOG_DIR=/path/to/laravel/storage/logs
HUB_IP=MONITORING_HUB_PRIVATE_IP
```

Confirm the Laravel log directory exists:

```bash
test -d "$LARAVEL_LOG_DIR" && ls -lah "$LARAVEL_LOG_DIR" | head
```

Start the Alloy stack:

```bash
cd /opt/grafana-nodes/alloy-docker
docker compose up -d
docker compose ps
docker compose logs --tail=80 alloy
```

Expected containers:

- `alloy`
- `nginx-exporter`
- `phpfpm-exporter`

## 10. Validate the Client

Run on the client EC2:

```bash
curl -s http://127.0.0.1:12345/ | head -1
curl -s http://127.0.0.1:9113/metrics | head -5
curl -s http://127.0.0.1:9253/metrics | head -5
```

Check network access to the hub:

```bash
nc -vz "$HUB_IP" 3100
nc -vz "$HUB_IP" 9009
```

Check Alloy logs:

```bash
cd /opt/grafana-nodes/alloy-docker
docker compose logs --tail=100 alloy
```

If `phpfpm-exporter` has no metrics:

```bash
docker compose logs --tail=80 phpfpm-exporter
curl -s http://127.0.0.1:8080/fpm-status
```

If `nginx-exporter` has no metrics:

```bash
docker compose logs --tail=80 nginx-exporter
curl -s http://127.0.0.1:8080/nginx_status
```

## 11. Validate Data in Grafana

From Grafana Explore, test Loki queries:

```logql
{instance="app-prod-01"}
{job="laravel", environment="production"}
{job="nginx"}
```

Replace `app-prod-01` and `production` with the real client values.

Test Mimir/Prometheus queries:

```promql
up
up{instance="app-prod-01"}
node_uname_info{instance="app-prod-01"}
nginx_connections_active{instance="app-prod-01"}
phpfpm_up{instance="app-prod-01"}
```

## 12. Operations

Monitoring hub commands:

```bash
cd /opt/grafana-main-hub/main-server
docker compose ps
docker compose logs -f grafana
docker compose logs -f loki
docker compose logs -f mimir
docker compose restart grafana
docker compose pull
docker compose up -d
```

Client commands:

```bash
cd /opt/grafana-nodes/alloy-docker
docker compose ps
docker compose logs -f alloy
docker compose logs -f nginx-exporter
docker compose logs -f phpfpm-exporter
docker compose restart alloy
docker compose pull
docker compose up -d
```

Stop the client monitoring agent only:

```bash
cd /opt/grafana-nodes/alloy-docker
docker compose down
```

This does not stop Nginx, PHP-FPM, Laravel, databases, or application services.

## 13. Troubleshooting

Loki or Mimir cannot access S3:

```bash
aws sts get-caller-identity
aws s3 ls "s3://$OBS_BUCKET"
docker compose logs --tail=120 loki
docker compose logs --tail=120 mimir
```

Check:

- The hub EC2 has the `GrafanaServerProfile` instance profile attached.
- The IAM policy is named `GrafanaS3Access`.
- The IAM policy bucket ARN matches the real bucket.
- Loki and Mimir configs use the correct bucket name and region.
- Mimir has `native_aws_auth_enabled: true` when using the EC2 IAM instance profile.
- The hub EC2 can reach the regional S3 endpoint.

Client cannot push logs or metrics:

```bash
nc -vz "$HUB_IP" 3100
nc -vz "$HUB_IP" 9009
docker compose logs --tail=100 alloy
```

Check:

- `HUB_IP` is the monitoring hub private IP.
- Hub security group allows inbound from the client security group on `3100` and `9009`.
- Client outbound rules allow access to the hub.
- Loki and Mimir containers are running on the hub.

No Laravel logs in Grafana:

```bash
ls -lah "$LARAVEL_LOG_DIR"
cd /opt/grafana-nodes/alloy-docker
docker compose logs --tail=100 alloy
```

Check:

- `LARAVEL_LOG_DIR` points to the real Laravel `storage/logs` directory.
- Docker Compose mounts that path into Alloy.
- The Laravel app is actively writing log files.

No Nginx or PHP-FPM metrics:

```bash
curl -s http://127.0.0.1:8080/nginx_status
curl -s http://127.0.0.1:8080/fpm-status
curl -s http://127.0.0.1:9113/metrics | head
curl -s http://127.0.0.1:9253/metrics | head
```

Check:

- Nginx status server exists at `/etc/nginx/conf.d/stub_status.conf`.
- PHP-FPM `pm.status_path` is set to `/fpm-status`.
- `fastcgi_pass` uses the real PHP-FPM socket path.
- Nginx was reloaded after config changes.

## 14. Security Notes

- Use IAM instance profiles instead of static AWS keys.
- Keep the S3 bucket private.
- Block S3 public access.
- Restrict Grafana UI access to office/VPN IPs.
- Restrict Loki and Mimir ports to client security groups or private CIDRs.
- Keep Loki and Mimir internal gRPC ports `9096` and `9095` unpublished unless there is a specific private-network need.
- Prefer private IPs for Alloy to hub traffic.
- Do not expose Loki or Mimir directly to the public internet.
- Use HTTPS for Grafana in production.
- Rotate the Grafana admin password after first login.
- Keep `main-server/.env` private and do not commit real passwords.

## 15. Cleanup

Stop the monitoring hub stack:

```bash
cd /opt/grafana-main-hub/main-server
docker compose down
```

Stop a client Alloy stack:

```bash
cd /opt/grafana-nodes/alloy-docker
docker compose down
```

Remove the IAM attachment only if the hub no longer needs S3 access:

```bash
aws iam detach-role-policy \
  --role-name GrafanaServerRole \
  --policy-arn "arn:aws:iam::$AWS_ACCOUNT_ID:policy/GrafanaS3Access"
```

Do not delete the S3 bucket unless you intentionally want to delete all stored logs and metrics.
