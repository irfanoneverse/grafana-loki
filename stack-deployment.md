# Stack Deployment Guide

This is the end-to-end deployment guide for the Grafana, Loki, Mimir, and Alloy observability stack.

Detailed component guides:

- Main server: [grafana-main-server.md](grafana-main-server.md)
- Client server: [client-server.md](client-server.md)
- OpenTelemetry cleanup: [opentelemetry-cleanup.md](opentelemetry-cleanup.md)

## Architecture

One EC2 instance acts as the monitoring hub:

- Runs Grafana for dashboards and Explore.
- Runs Loki for log ingestion and querying.
- Runs Mimir for metrics ingestion and querying.
- Uses AWS S3 for Loki and Mimir object storage.
- Uses the EC2 IAM role instead of static AWS keys.

Each application EC2 instance runs the client stack:

- Grafana Alloy tails logs and sends them to Loki.
- Exporters expose Nginx and PHP-FPM metrics locally.
- Alloy sends metrics to Mimir.
- Client EC2 instances do not need S3 access.

Data flow:

```text
Application EC2
  Laravel, Nginx, PHP-FPM logs
  Nginx and PHP-FPM metrics
        |
        v
  client-server Alloy stack
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

| Server | Path | Purpose |
| --- | --- | --- |
| Monitoring hub EC2 | `/opt/grafana-main-hub` | Repository copy |
| Monitoring hub EC2 | `/opt/grafana-main-hub/grafana-main-server` | Main stack |
| Client EC2 | `/opt/grafana-nodes` | Repository copy |
| Client EC2 | `/opt/grafana-nodes/client-server` | Client stack |

## 1. AWS Console Preparation

Use the AWS Management Console for all AWS resource work.

Choose these values first:

| Value | Example |
| --- | --- |
| AWS Region | `ap-southeast-1` |
| S3 bucket | `your-company-observability-data` |
| Hub security group | `grafana-main-hub` |
| Client security group | `grafana-client` |
| IAM policy | `GrafanaS3Access` |
| IAM role | `GrafanaServerRole` |
| Retention | `100` days |

## 2. Create S3 Storage

In the AWS Console:

1. Go to **S3**.
2. Click **Create bucket**.
3. Enter a globally unique bucket name.
4. Select the same Region as the hub EC2.
5. Keep **ACLs disabled**.
6. Keep **Block all public access** fully enabled.
7. Enable **Bucket Versioning**.
8. Enable default encryption with **SSE-S3**.
9. Click **Create bucket**.

Create the lifecycle rule:

1. Open the bucket.
2. Go to **Management**.
3. Click **Create lifecycle rule**.
4. Name it `ObservabilityDataLifecycle`.
5. Apply it to all objects in the bucket.
6. Enable transition of current object versions.
7. Transition current versions to **Intelligent-Tiering** after `30` days.
8. Enable expiration of current object versions after `100` days.
9. Enable permanent deletion of noncurrent versions after `7` days.
10. Create the rule.

This replaces the removed S3 lifecycle JSON file.

## 3. Create IAM Access for the Hub

Create the policy:

1. Go to **IAM**.
2. Open **Policies**.
3. Click **Create policy**.
4. Choose **S3** in the visual editor.
5. Add bucket-level actions:
   - `ListBucket`
   - `GetBucketLocation`
6. Restrict the bucket resource to the observability bucket ARN.
7. Add object-level actions:
   - `PutObject`
   - `GetObject`
   - `DeleteObject`
8. Restrict the object resource to every object in the observability bucket.
9. Name the policy `GrafanaS3Access`.
10. Create the policy.

Create the role:

1. Go to **IAM**.
2. Open **Roles**.
3. Click **Create role**.
4. Choose **AWS service**.
5. Choose **EC2** as the use case.
6. Attach `GrafanaS3Access`.
7. Name the role `GrafanaServerRole`.
8. Create the role.

Attach the role:

1. Go to **EC2**.
2. Open **Instances**.
3. Select the monitoring hub instance.
4. Click **Actions**.
5. Choose **Security**.
6. Choose **Modify IAM role**.
7. Select `GrafanaServerRole`.
8. Save.

This replaces the removed IAM policy JSON file and keeps role setup in the AWS Console.

## 4. Configure Security Groups

In the AWS Console, configure the hub security group:

| Type | Port | Source | Purpose |
| --- | ---: | --- | --- |
| HTTP | `80` | office or VPN CIDR | Grafana UI |
| SSH | `22` | office or VPN CIDR | admin access |
| Custom TCP | `3100` | client security group | Loki ingest |
| Custom TCP | `9009` | client security group | Mimir remote write |

Configure client security group outbound:

| Type | Port | Destination | Purpose |
| --- | ---: | --- | --- |
| Custom TCP | `3100` | hub private IP or hub security group | send logs |
| Custom TCP | `9009` | hub private IP or hub security group | send metrics |

Keep Loki and Mimir private. Do not allow public internet sources for `3100` or `9009`.

## 5. Deploy the Monitoring Hub

On the hub EC2, install dependencies:

```text
sudo apt update
sudo apt upgrade -y
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker "$USER"
sudo apt install -y docker-compose-plugin git
```

Log out and back in:

```text
exit
```

Clone and configure:

```text
cd /opt
sudo mkdir -p grafana-main-hub
sudo chown "$USER:$USER" grafana-main-hub
git clone https://github.com/irfanoneverse/grafana.git grafana-main-hub
cd /opt/grafana-main-hub/grafana-main-server
cp .env.example .env
nano .env
nano loki-config.yaml
nano mimir-config.yaml
```

Set `.env`:

```env
GRAFANA_ADMIN_PASSWORD=CHANGE_THIS_PASSWORD
GRAFANA_ROOT_URL=http://MONITORING_HUB_PUBLIC_IP/
```

Set `loki-config.yaml`:

- S3 endpoint for the Region
- S3 bucket name
- S3 Region
- `retention_period: 100d`

Set `mimir-config.yaml`:

- S3 endpoint for the Region
- S3 bucket name
- S3 Region
- `native_aws_auth_enabled: true`
- `compactor_blocks_retention_period: 100d`

Start:

```text
docker compose up -d
docker compose ps
```

Validate:

```text
curl -s http://127.0.0.1:80/api/health
curl -s http://127.0.0.1:3100/ready
curl -s http://127.0.0.1:9009/ready
docker compose logs --tail=80 loki
docker compose logs --tail=80 mimir
```

## 6. Deploy Each Client

On each client EC2, install dependencies:

```text
sudo apt update
sudo apt upgrade -y
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker "$USER"
sudo apt install -y docker-compose-plugin git netcat-openbsd
```

Log out and back in:

```text
exit
```

Create local Nginx and PHP-FPM status endpoints as described in [client-server.md](client-server.md).

Clone and configure:

```text
cd /opt
sudo mkdir -p grafana-nodes
sudo chown "$USER:$USER" grafana-nodes
git clone https://github.com/irfanoneverse/grafana.git grafana-nodes
cd /opt/grafana-nodes/client-server
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

Start:

```text
docker compose up -d
docker compose ps
docker compose logs --tail=80 alloy
```

Validate:

```text
curl -s http://127.0.0.1:12345/ | head -1
curl -s http://127.0.0.1:9113/metrics | head -5
curl -s http://127.0.0.1:9253/metrics | head -5
nc -vz MONITORING_HUB_PRIVATE_IP 3100
nc -vz MONITORING_HUB_PRIVATE_IP 9009
```

## 7. Validate in Grafana

Open Grafana:

```text
http://MONITORING_HUB_PUBLIC_IP/
```

Go to **Explore**.

Loki examples:

```logql
{instance="app-prod-01"}
{job="laravel", environment="production"}
{job="nginx"}
```

Mimir examples:

```promql
up
up{instance="app-prod-01"}
node_uname_info{instance="app-prod-01"}
nginx_connections_active{instance="app-prod-01"}
phpfpm_up{instance="app-prod-01"}
```

## 8. Troubleshooting

Loki or Mimir cannot use S3:

1. In **EC2**, confirm the hub instance has `GrafanaServerRole`.
2. In **IAM**, confirm the role has `GrafanaS3Access`.
3. In **IAM Policies**, confirm the policy references the real bucket name.
4. In **S3**, confirm the bucket Region matches the Loki and Mimir config.
5. On the hub EC2, check Loki and Mimir logs.

Client cannot push data:

1. In **EC2 Security Groups**, confirm hub inbound allows the client security group on `3100` and `9009`.
2. Confirm the client `.env` uses the hub private IP.
3. Confirm Loki and Mimir are running on the hub.
4. Check Alloy logs on the client.

No logs:

1. Confirm `LARAVEL_LOG_DIR` points to the real Laravel `storage/logs` directory.
2. Confirm Docker Compose mounted that path.
3. Confirm Laravel is writing log files.

No Nginx or PHP-FPM metrics:

1. Confirm `http://127.0.0.1:8080/nginx_status` works on the client.
2. Confirm `http://127.0.0.1:8080/fpm-status` works on the client.
3. Confirm the PHP-FPM socket in `stub_status.conf` is correct.

## 9. Security Notes

- Use IAM roles, not static AWS access keys.
- Keep the S3 bucket private.
- Keep S3 public access blocking enabled.
- Restrict Grafana to office or VPN networks unless HTTPS and authentication are fully configured.
- Restrict Loki and Mimir to client security groups.
- Prefer private IPs for client-to-hub traffic.
- Keep `grafana-main-server/.env` private.
- Rotate the Grafana admin password after first login.

## 10. Cleanup

Stop the hub stack:

```text
cd /opt/grafana-main-hub/grafana-main-server
docker compose down
```

Stop a client stack:

```text
cd /opt/grafana-nodes/client-server
docker compose down
```

To remove AWS access manually:

1. Open **EC2**.
2. Select the hub instance.
3. Use **Actions** -> **Security** -> **Modify IAM role**.
4. Remove or replace `GrafanaServerRole` only if the hub no longer needs S3.
5. Open **IAM** and detach `GrafanaS3Access` only after confirming no hub uses it.

Do not delete the S3 bucket unless you intentionally want to delete all stored logs and metrics.
