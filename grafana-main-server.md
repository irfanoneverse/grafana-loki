# Grafana Main Server Guide

This guide builds the AWS foundation and deploys the monitoring hub. The hub runs Grafana, Loki, Mimir, and hub self-monitoring Alloy from `grafana-main-server/`.

Use [client-server.md](client-server.md) for each Laravel or application EC2 client.

## Target Architecture

- One EC2 instance is the monitoring hub.
- The hub runs Docker Compose from `/opt/grafana-main-hub/grafana-main-server`.
- Grafana is exposed on port `80`.
- Loki receives logs on port `3100`.
- Mimir receives metrics on port `9009`.
- Loki and Mimir store long-term data in one private S3 bucket.
- Client EC2 instances send data to the hub over private networking.

Recommended names:

| Resource | Recommended value |
| --- | --- |
| S3 bucket | `your-company-observability-data` |
| IAM policy | `GrafanaS3Access` |
| IAM role | `GrafanaServerRole` |
| Instance profile | created automatically when attaching the role |
| Hub security group | `grafana-main-hub` |
| Client security group | `grafana-client` |

## 1. Prepare AWS Values

Decide these values before creating resources:

| Value | Example | Notes |
| --- | --- | --- |
| AWS Region | `ap-southeast-1` | Use the same Region for EC2 and S3. |
| S3 bucket name | `your-company-observability-data` | Must be globally unique. |
| Grafana URL | `http://MONITORING_HUB_PUBLIC_IP/` | Replace later with DNS/HTTPS if available. |
| Grafana admin password | a strong generated password | Store it in a password manager. |
| Office or VPN CIDR | `203.0.113.10/32` | Used for SSH and Grafana access. |

## 2. Create the S3 Bucket in the AWS Console

1. Go to the AWS Console.
2. Open **S3**.
3. Click **Create bucket**.
4. In **Bucket name**, enter your chosen bucket name, for example `your-company-observability-data`.
5. In **AWS Region**, choose the same Region as the monitoring hub EC2 instance, for example `ap-southeast-1`.
6. Under **Object Ownership**, keep **ACLs disabled**.
7. Under **Block Public Access settings for this bucket**, keep all four public access blocking checkboxes enabled.
8. Under **Bucket Versioning**, choose **Enable**. This helps recover from accidental overwrites or deletes.
9. Under **Default encryption**, choose **Server-side encryption with Amazon S3 managed keys (SSE-S3)**.
10. Leave **Bucket Key** at the default setting for SSE-S3.
11. Click **Create bucket**.

After the bucket is created:

1. Open the bucket.
2. Go to the **Permissions** tab.
3. Confirm **Block all public access** is **On**.
4. Go to the **Properties** tab.
5. Confirm **Bucket Versioning** is **Enabled**.
6. Confirm **Default encryption** shows **SSE-S3**.

## 3. Configure S3 Lifecycle Retention Manually

The deleted AWS JSON lifecycle file represented this rule:

- Rule name: `ObservabilityDataLifecycle`
- Scope: all objects in the bucket
- Transition current object versions to S3 Intelligent-Tiering after 30 days
- Expire current object versions after 100 days
- Expire noncurrent object versions after 7 days

Create it through the Console:

1. Open **S3**.
2. Open the observability bucket.
3. Go to **Management**.
4. In **Lifecycle rules**, click **Create lifecycle rule**.
5. For **Lifecycle rule name**, enter `ObservabilityDataLifecycle`.
6. Under **Choose a rule scope**, choose **Apply to all objects in the bucket**.
7. Confirm the warning that the rule applies to all objects.
8. Under **Lifecycle rule actions**, select:
   - **Transition current versions of objects between storage classes**
   - **Expire current versions of objects**
   - **Permanently delete noncurrent versions of objects**
9. In **Transition current versions of objects between storage classes**, click **Add transition**.
10. For **Choose storage class transitions**, select **Intelligent-Tiering**.
11. For **Days after object creation**, enter `30`.
12. In **Expire current versions of objects**, enter `100` days.
13. In **Permanently delete noncurrent versions of objects**, enter `7` days after objects become noncurrent.
14. Review the rule summary.
15. Click **Create rule**.

Why this matters:

- Loki and Mimir retain data for `100d` in their config.
- S3 lifecycle deletion keeps storage cost bounded.
- The 7-day noncurrent version cleanup prevents bucket versioning from retaining old deleted data forever.

## 4. Create the IAM Policy Manually

The deleted IAM JSON file granted only the S3 permissions Loki and Mimir need. Recreate it in the Console with the visual editor.

1. Open **IAM** in the AWS Console.
2. Go to **Policies**.
3. Click **Create policy**.
4. Select the **Visual** editor.
5. For **Service**, choose **S3**.
6. In **Actions allowed**, add these bucket-level actions:
   - `ListBucket`
   - `GetBucketLocation`
7. In **Resources**, open **bucket**.
8. Choose **Specific**.
9. Click **Add ARN**.
10. Enter the bucket name, for example `your-company-observability-data`.
11. Save that bucket ARN.
12. Add these object-level actions:
   - `PutObject`
   - `GetObject`
   - `DeleteObject`
13. In **Resources**, open **object**.
14. Choose **Specific**.
15. Click **Add ARN**.
16. Enter the same bucket name.
17. For **Object name**, choose **Any** so the ARN covers every object under the bucket.
18. Click **Next**.
19. For **Policy name**, enter `GrafanaS3Access`.
20. Add this description: `Allows the Grafana monitoring hub to store Loki and Mimir data in the observability S3 bucket.`
21. Review that the policy includes only:
   - Bucket ARN: `arn:aws:s3:::your-company-observability-data`
   - Object ARN: `arn:aws:s3:::your-company-observability-data/*`
22. Click **Create policy**.

Do not grant this policy to client EC2 instances. Only the monitoring hub needs S3 access.

## 5. Create and Attach the EC2 IAM Role

1. Open **IAM**.
2. Go to **Roles**.
3. Click **Create role**.
4. For **Trusted entity type**, choose **AWS service**.
5. For **Use case**, choose **EC2**.
6. Click **Next**.
7. Search for `GrafanaS3Access`.
8. Select the policy.
9. Click **Next**.
10. For **Role name**, enter `GrafanaServerRole`.
11. Add this description: `EC2 role used by the Grafana monitoring hub for Loki and Mimir S3 storage.`
12. Click **Create role**.

Attach the role to the monitoring hub EC2:

1. Open **EC2**.
2. Go to **Instances**.
3. Select the monitoring hub instance.
4. Click **Actions**.
5. Choose **Security**.
6. Choose **Modify IAM role**.
7. Select `GrafanaServerRole`.
8. Click **Update IAM role**.

If another role is already attached, confirm it is not needed before replacing it. If it is needed, add the `GrafanaS3Access` policy to the existing role instead.

## 6. Configure Security Groups in the AWS Console

Use private networking for Loki and Mimir. Do not expose ports `3100` or `9009` to the public internet.

### Hub Security Group

1. Open **EC2**.
2. Go to **Security Groups**.
3. Create or open the hub security group, recommended name `grafana-main-hub`.
4. On **Inbound rules**, click **Edit inbound rules**.
5. Add these rules:

| Type | Port | Source | Purpose |
| --- | ---: | --- | --- |
| HTTP | `80` | your office or VPN CIDR | Grafana UI |
| SSH | `22` | your office or VPN CIDR | administration |
| Custom TCP | `3100` | `grafana-client` security group | Loki ingest from Alloy |
| Custom TCP | `9009` | `grafana-client` security group | Mimir remote write from Alloy |

6. Save the rules.

### Client Security Group

1. Open or create the client security group, recommended name `grafana-client`.
2. Keep existing application inbound rules for SSH, HTTP, HTTPS, or load balancer traffic.
3. On **Outbound rules**, allow the client to reach the hub on:
   - TCP `3100`
   - TCP `9009`
4. If default outbound is open, this is already allowed, but a narrower rule to the hub private IP or hub security group is better.

## 7. Prepare the Monitoring Hub EC2

Connect to the hub EC2 over SSH and install the host packages:

```text
sudo apt update
sudo apt upgrade -y
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker "$USER"
sudo apt install -y docker-compose-plugin git
```

Log out and back in so Docker group membership applies:

```text
exit
```

Verify after logging in again:

```text
docker --version
docker compose version
```

## 8. Deploy the Main Server Stack

Clone the repository on the hub EC2:

```text
cd /opt
sudo mkdir -p grafana-main-hub
sudo chown "$USER:$USER" grafana-main-hub
git clone https://github.com/irfanoneverse/grafana.git grafana-main-hub
cd /opt/grafana-main-hub/grafana-main-server
```

Create the runtime environment file:

```text
cp .env.example .env
nano .env
```

Set:

```env
GRAFANA_ADMIN_PASSWORD=CHANGE_THIS_PASSWORD
GRAFANA_ROOT_URL=http://MONITORING_HUB_PUBLIC_IP/
```

Edit the storage configs:

```text
nano loki-config.yaml
nano mimir-config.yaml
```

In `loki-config.yaml`, set:

- `endpoint` to `s3.YOUR_REGION.amazonaws.com`, for example `s3.ap-southeast-1.amazonaws.com`
- `bucketnames` to the observability bucket name
- `region` to the bucket Region
- `retention_period` to `100d`

In `mimir-config.yaml`, set:

- `endpoint` to `s3.YOUR_REGION.amazonaws.com`
- `bucket_name` to the observability bucket name
- `region` to the bucket Region
- `native_aws_auth_enabled` to `true`
- `compactor_blocks_retention_period` to `100d`

Start the stack:

```text
docker compose up -d
docker compose ps
docker compose logs --tail=80 grafana
docker compose logs --tail=80 loki
docker compose logs --tail=80 mimir
```

Expected containers:

- `grafana`
- `loki`
- `mimir`
- `alloy-hub`

## 9. Validate the Hub

Run these checks on the hub EC2:

```text
curl -s http://127.0.0.1:80/api/health
curl -s http://127.0.0.1:3100/ready
curl -s http://127.0.0.1:9009/ready
```

Expected results:

- Grafana returns health JSON.
- Loki returns a ready response.
- Mimir returns a ready response after startup.

Validate S3 writes through the AWS Console:

1. Open **S3**.
2. Open the observability bucket.
3. Check for Loki and Mimir object prefixes after the stack has been running and clients have sent data.
4. If no objects appear, review the hub EC2 IAM role, bucket name, Region, and Loki/Mimir container logs.

Open Grafana at:

```text
http://MONITORING_HUB_PUBLIC_IP/
```

Login:

- Username: `admin`
- Password: the value in `GRAFANA_ADMIN_PASSWORD`

## 10. Operations

Common hub commands:

```text
cd /opt/grafana-main-hub/grafana-main-server
docker compose ps
docker compose logs -f grafana
docker compose logs -f loki
docker compose logs -f mimir
docker compose restart grafana
docker compose pull
docker compose up -d
```

If Loki or Mimir cannot access S3:

1. In **EC2**, confirm the hub instance has `GrafanaServerRole`.
2. In **IAM**, confirm `GrafanaServerRole` has `GrafanaS3Access`.
3. In **IAM Policies**, confirm the bucket ARN and object ARN use the real bucket name.
4. In **S3**, confirm the bucket is in the Region configured in `loki-config.yaml` and `mimir-config.yaml`.
5. On the hub EC2, check:

```text
docker compose logs --tail=120 loki
docker compose logs --tail=120 mimir
```

## 11. Handoff to Client Servers

For each Laravel or application EC2:

- Use [client-server.md](client-server.md).
- Use the hub private IP for `HUB_IP` when the servers are in the same VPC.
- Confirm the hub security group allows the client security group on `3100` and `9009`.
- Confirm the client has no S3 IAM policy unless it needs S3 for unrelated application work.
