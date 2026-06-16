# Grafana Main Server — Deployment Guide

Deploys **Loki + Grafana** on a single EC2 monitoring hub. Client servers ship Laravel logs to Loki via Grafana Alloy. Loki stores logs long-term in S3.

## Architecture

```
Client EC2 (Laravel app)
  Grafana Alloy
    |
    | logs → http://HUB_PRIVATE_IP:3100/loki/api/v1/push
    v
Monitoring Hub EC2
  Loki  → S3 (long-term storage)
  Grafana (queries Loki, port 80)
```

Recommended deploy path on hub: `/opt/grafana-main-hub/grafana-main-server`

---

## 1. AWS — Create S3 Bucket

1. Go to **S3** → **Create bucket**
2. Set a globally unique bucket name, e.g. `your-company-observability-data`
3. Select the same Region as the hub EC2
4. Keep **ACLs disabled** and **Block all public access** fully enabled
5. Enable **Bucket Versioning**
6. Enable default encryption with **SSE-S3**
7. Click **Create bucket**

Create a lifecycle rule to control storage costs:

1. Open the bucket → **Management** → **Create lifecycle rule**
2. Name: `ObservabilityDataLifecycle`, scope: all objects
3. Transition current versions to **Intelligent-Tiering** after `30` days
4. Expire current versions after `100` days
5. Permanently delete noncurrent versions after `7` days
6. Create the rule

---

## 2. AWS — IAM Policy and Role

**Create the policy:**

1. **IAM** → **Policies** → **Create policy** → Visual editor → Service: **S3**
2. Bucket-level actions: `ListBucket`, `GetBucketLocation` — restrict to the observability bucket ARN
3. Object-level actions: `PutObject`, `GetObject`, `DeleteObject` — restrict to all objects in the bucket
4. Name the policy `GrafanaS3Access`

**Create the role:**

1. **IAM** → **Roles** → **Create role**
2. Trusted entity: **AWS service** → **EC2**
3. Attach `GrafanaS3Access`
4. Name the role `GrafanaServerRole`

**Attach to the hub EC2:**

1. **EC2** → **Instances** → select the hub instance
2. **Actions** → **Security** → **Modify IAM role**
3. Select `GrafanaServerRole` → Save

---

## 3. AWS — Security Groups

**Hub security group** (`grafana-main-hub`) inbound rules:

| Type | Port | Source | Purpose |
|---|---:|---|---|
| HTTP | `80` | office or VPN CIDR | Grafana UI |
| SSH | `22` | office or VPN CIDR | admin access |
| Custom TCP | `3100` | `grafana-client` security group | Loki ingest from Alloy |

**Client security group** (`grafana-client`) outbound — ensure clients can reach the hub on TCP `3100`.

Do not expose port `3100` to the public internet.

---

## 4. Install Docker on the Hub EC2

```bash
sudo apt update && sudo apt upgrade -y
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker "$USER"
sudo apt install -y docker-compose-plugin git
exit  # log out and back in for Docker group to apply
```

---

## 5. Deploy the Stack

```bash
cd /opt
sudo mkdir -p grafana-main-hub
sudo chown "$USER:$USER" grafana-main-hub
git clone https://github.com/irfanoneverse/grafana-production.git grafana-main-hub
cd /opt/grafana-main-hub/grafana-main-server
```

```bash
cp .env.example .env
nano .env
```

Set in `.env`:

```env
GRAFANA_ADMIN_PASSWORD=a-strong-password
GRAFANA_ROOT_URL=http://YOUR_HUB_PUBLIC_IP/
```

Edit `loki-config.yaml` — update these three values to match your setup:

```yaml
endpoint: s3.YOUR_REGION.amazonaws.com
bucketnames: your-company-observability-data
region: YOUR_REGION
```

Start:

```bash
docker compose up -d
docker compose ps
```

---

## 6. Validate

```bash
curl -s http://127.0.0.1:3100/ready   # should return: ready
curl -s http://127.0.0.1:80/api/health  # should return Grafana health JSON
docker compose logs --tail=80 loki
```

Open Grafana at `http://YOUR_HUB_PUBLIC_IP/` and log in with `admin` and the password from `.env`.

Go to **Explore** → select **Loki** → run a query like `{job="laravel"}` once a client is connected.

After clients have been running for a few minutes, confirm Loki objects appear in the S3 bucket under **S3** → open the bucket → look for a `loki/` prefix.

---

## 7. Operations

```bash
cd /opt/grafana-main-hub/grafana-main-server

docker compose ps
docker compose logs -f loki
docker compose logs -f grafana
docker compose restart grafana
docker compose pull && docker compose up -d  # upgrade images
docker compose down  # stop everything
```

---

## 8. Troubleshooting

**Loki cannot write to S3:**
1. Confirm the hub EC2 has `GrafanaServerRole` attached (EC2 → Instances → Security tab)
2. Confirm `GrafanaS3Access` is attached to that role
3. Confirm `loki-config.yaml` bucket name and region match the actual S3 bucket
4. Check: `docker compose logs --tail=120 loki`

**Client logs not appearing in Grafana:**
1. Confirm hub security group allows the client security group on TCP `3100`
2. Confirm the client `.env` has the correct `HUB_IP` (hub private IP)
3. Check Alloy logs on the client: `docker compose logs -f alloy`

---

## 9. Security Notes

- Use IAM roles, not static AWS access keys
- Keep the S3 bucket private with public access blocking enabled
- Restrict Grafana (port `80`) to office or VPN CIDR — do not expose to the public internet without HTTPS and proper auth
- Restrict Loki (port `3100`) to the client security group only
- Keep `.env` out of version control — it is in `.gitignore`
- Rotate the Grafana admin password after first login
