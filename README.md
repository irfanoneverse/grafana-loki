# Grafana Observability Stack

A centralized log observability setup using **Grafana + Loki** on one monitoring hub EC2, with **Grafana Alloy** running on each application server to ship Laravel logs.

---

## How It Works

```
Client EC2 (Laravel app)
  Grafana Alloy
    |
    | logs → http://HUB_PRIVATE_IP:3100/loki/api/v1/push
    v
Monitoring Hub EC2
  Loki  → S3 (long-term storage)
  Grafana (queries Loki, accessible on port 80)
```

- The hub runs Grafana and Loki only
- Client servers run a small Alloy agent — no monitoring stack on the app server
- Loki stores logs long-term in a private S3 bucket via the hub IAM role (no static AWS keys)
- Monitoring can be updated centrally without touching application servers

**Important Ports**

| Port | Where | Purpose | Public? |
| --- | --- | --- | --- |
| 80 | Hub | Grafana UI | Office/VPN only |
| 22 | Hub + Clients | SSH | Office/VPN only |
| 3100 | Hub | Loki — receives logs from Alloy | No |
| 12345 | Client | Alloy local UI | No |

---

# Part 1 — Main Server Setup

## 1. AWS — Create S3 Bucket

Loki uses S3 for long-term log storage. Only the hub EC2 needs S3 access.

1. Go to **S3** → **Create bucket**
2. Enter a globally unique bucket name, e.g. `your-company-observability-data`
3. Select the same Region as the hub EC2, e.g. `ap-southeast-1`
4. Under **Object Ownership** — keep **ACLs disabled**
5. Under **Block Public Access** — keep all four checkboxes enabled
6. Enable **Bucket Versioning**
7. Under **Default encryption** — choose **SSE-S3**
8. Click **Create bucket**

**Create a lifecycle rule to control storage cost:**

1. Open the bucket → **Management** → **Create lifecycle rule**
2. Name: `ObservabilityDataLifecycle`, scope: all objects in the bucket
3. Transition current versions to **Intelligent-Tiering** after `30` days
4. Expire current versions after `100` days
5. Permanently delete noncurrent versions after `7` days
6. Create the rule

> Keep Loki's `retention_period` in `loki-config.yaml` aligned with the S3 expiry (100d). If S3 deletes data sooner than Loki expects, queries may return incomplete results.

---

## 2. AWS — IAM Policy and Role

The hub needs an IAM role so Loki can read and write S3 without static keys.

**Create the IAM policy:**

1. **IAM** → **Policies** → **Create policy** → Visual editor → Service: **S3**
2. Bucket-level actions: `ListBucket`, `GetBucketLocation` — restrict to the observability bucket ARN
3. Object-level actions: `PutObject`, `GetObject`, `DeleteObject` — restrict to all objects in the bucket
4. Name the policy `GrafanaS3Access`

**Create the IAM role:**

1. **IAM** → **Roles** → **Create role**
2. Trusted entity: **AWS service** → **EC2**
3. Attach `GrafanaS3Access`
4. Name the role `GrafanaServerRole`

**Attach the role to the hub EC2:**

1. **EC2** → **Instances** → select the monitoring hub
2. **Actions** → **Security** → **Modify IAM role**
3. Select `GrafanaServerRole` → **Update IAM role**

> Client EC2 instances do not need S3 permissions.

---

## 3. AWS — Security Groups

**Hub security group** (`grafana-main-hub`) inbound rules:

| Type | Port | Source | Purpose |
| --- | --- | --- | --- |
| HTTP | 80 | Office or VPN CIDR | Grafana UI |
| SSH | 22 | Office or VPN CIDR | Admin access |
| Custom TCP | 3100 | `grafana-client` security group | Loki ingest from Alloy |

**Client security group** (`grafana-client`):

- Keep existing inbound rules for SSH, HTTP/HTTPS, load balancer traffic
- Outbound: ensure clients can reach the hub private IP on TCP `3100`

> Do not expose port `3100` to the public internet. Loki has no authentication in this setup — security groups are the access control layer.

---

## 4. Install Docker on the Hub EC2

SSH into the hub EC2 and run:

```bash
sudo apt update && sudo apt upgrade -y
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker "$USER"
sudo apt install -y docker-compose-plugin git
exit
```

Log back in, then verify:

```bash
docker --version
docker compose version
```

---

## 5. Deploy the Hub Stack

Clone the repository:

```bash
cd /opt
sudo mkdir -p grafana-main-hub
sudo chown "$USER:$USER" grafana-main-hub
git clone https://github.com/oneone666/grafana-main-server.git grafana-main-hub
cd /opt/grafana-main-hub
```

Create the environment file:

```bash
cp .env.example .env
nano .env
```

Set in `.env`:

```
GRAFANA_ADMIN_PASSWORD=a-strong-password
GRAFANA_ROOT_URL=http://YOUR_HUB_PUBLIC_IP/
```

Edit `loki-config.yaml` and update these three values to match your S3 setup:

```
endpoint: s3.YOUR_REGION.amazonaws.com
bucketnames: your-company-observability-data
region: YOUR_REGION
```

Start the stack:

```bash
docker compose up -d
docker compose ps
```

Expected running containers: `loki`, `grafana`

---

## 6. Validate the Hub

```bash
curl -s http://127.0.0.1:3100/ready
curl -s http://127.0.0.1:80/api/health

docker compose logs --tail=80 loki
docker compose logs --tail=80 grafana
```

Verify IAM and S3 access from the hub EC2:

```bash
aws sts get-caller-identity
aws s3 ls s3://your-company-observability-data
```

> If these fail, fix IAM before sending any client data. Loki needs S3 access to store logs correctly.

Open Grafana at `http://YOUR_HUB_PUBLIC_IP/` and log in with username `admin` and the password from `.env`.

---

## 7. Hub Operations

```bash
cd /opt/grafana-main-hub

docker compose ps
docker compose logs -f loki
docker compose logs -f grafana
docker compose restart grafana
docker compose pull && docker compose up -d
docker compose down
```

---

# Part 2 — Client Server Setup

Run this on each application EC2 that should ship logs to the hub.

---

## 1. Check Connectivity to the Hub

Before anything else, confirm the client can reach the hub:

```bash
nc -vz HUB_PRIVATE_IP 3100
```

If `nc` is not installed: `sudo apt install -y netcat-openbsd`

> If this fails, fix the hub security group inbound rules (Part 1 — Section 3) before continuing.

---

## 2. Install Docker on the Client EC2

```bash
sudo apt update && sudo apt upgrade -y
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker "$USER"
sudo apt install -y docker-compose-plugin git
exit
```

Log back in, then verify:

```bash
docker --version
docker compose version
```

---

## 3. Deploy the Client Agent

Clone the repository:

```bash
cd /opt
sudo mkdir -p grafana-nodes
sudo chown "$USER:$USER" grafana-nodes
git clone https://github.com/oneone666/grafana-client-docker.git grafana-nodes
cd /opt/grafana-nodes
```

Create the environment file:

```bash
cp .env.example .env
nano .env
```

Fill in all values:

```
HOSTNAME=laravel-app-01
INSTANCE_NAME=laravel-app-01
ENVIRONMENT=production
TZ_NAME=Asia/Kuala_Lumpur
LARAVEL_LOG_DIR=/path/to/laravel/storage/logs
PRIVATE_IPV4=10.0.0.10
PUBLIC_IPV4=203.0.113.10
HUB_IP=10.0.1.50
```

Confirm the Laravel log directory exists:

```bash
ls -lah /path/to/laravel/storage/logs
```

Start the agent:

```bash
docker compose up -d
docker compose ps
docker compose logs --tail=80 alloy
```

Expected running container: `alloy`

> This does not stop or touch Laravel, Nginx, PHP-FPM, or any other application service.

---

## 4. Validate the Client

```bash
curl -s http://127.0.0.1:12345/ | head -1

nc -vz HUB_PRIVATE_IP 3100

docker compose logs --tail=100 alloy
```

---

## 5. Verify Logs in Grafana

Open **Grafana → Explore → Loki** on the hub and try these queries:

```
{instance="laravel-app-01"}
{job="laravel", environment="production"}
{level="error"}
```

Replace `laravel-app-01` and `production` with the values from the client `.env`. Logs should appear within a minute of Alloy starting.

---

## 6. Client Operations

```bash
cd /opt/grafana-nodes

docker compose ps
docker compose logs -f alloy
docker compose restart alloy
docker compose pull && docker compose up -d
docker compose down
```

---

## Troubleshooting

**Loki cannot write to S3**

1. Confirm the hub EC2 has `GrafanaServerRole` — EC2 → Instances → Security tab
2. Confirm `GrafanaS3Access` is attached to that role
3. Confirm `loki-config.yaml` bucket name and region match the actual S3 bucket
4. Run `docker compose logs --tail=120 loki`

**Logs not appearing in Grafana**

1. Confirm hub security group allows the client security group on TCP `3100`
2. Confirm `HUB_IP` in client `.env` is the hub **private** IP
3. Confirm `LARAVEL_LOG_DIR` points to the real `storage/logs` directory
4. Run `docker compose logs -f alloy` on the client

**Security reminders**

- Use IAM roles, not static AWS access keys
- Keep port `3100` restricted to the client security group — never public
- Restrict Grafana port `80` to office or VPN — use HTTPS for production
- Keep `.env` files out of version control
- Rotate the Grafana admin password after first login
