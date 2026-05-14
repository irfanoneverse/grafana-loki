# Client Server Guide

This guide deploys the client-side monitoring stack from `client-server/` on each Laravel or application EC2 instance.

The application stays native on the host. Only these monitoring containers run in Docker:

- `alloy`

The client sends application and service logs to Loki, and system metrics such as CPU, memory, disk, filesystem, and network usage to Mimir on the monitoring hub.

## 1. Client Values

Decide these values for each EC2 instance:

| Value | Example | Notes |
| --- | --- | --- |
| Hub private IP | `10.0.1.50` | Use the private IP when the hub is in the same VPC. |
| Instance name | `laravel-app-01` | Must be unique per client. |
| Environment | `staging` or `production` | Used as a Grafana label. |
| Timezone | `Asia/Kuala_Lumpur` | IANA timezone name. |
| Laravel log directory | `/home/forge/site/storage/logs` | Confirm on each server. |
| Laravel app directory | `/home/forge/site` | Needed only for cleanup tasks. |

Forge deployments often use paths under `/home/forge/SITE_NAME`. Do not assume the path. Confirm the real Laravel directory before editing `.env`.

## 2. Confirm AWS Networking in the Console

Before touching the client EC2:

1. Open the AWS Console.
2. Go to **EC2**.
3. Open **Instances**.
4. Select the monitoring hub instance and note its **Private IPv4 address**.
5. Select the client instance and confirm it is in the expected VPC or has private network routing to the hub.
6. Go to **Security Groups**.
7. Open the hub security group, usually `grafana-main-hub`.
8. Confirm inbound rules allow the client security group on:
   - TCP `3100` for Loki
   - TCP `9009` for Mimir
9. Confirm Grafana port `80` and SSH port `22` are limited to your office or VPN CIDR.
10. Open the client security group, usually `grafana-client`.
11. Confirm outbound rules allow the client to reach the hub private IP on TCP `3100` and `9009`.

The client does not need an S3 IAM role for this monitoring design.

## 3. Install Docker on the Client EC2

Connect to the client EC2 over SSH and install Docker:

```text
sudo apt update
sudo apt upgrade -y
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker "$USER"
sudo apt install -y docker-compose-plugin git netcat-openbsd
```

Log out and log back in:

```text
exit
```

Verify:

```text
docker --version
docker compose version
docker run --rm hello-world
```

## 4. Deploy the Client Stack

Clone the repository:

```text
cd /opt
sudo mkdir -p grafana-nodes
sudo chown "$USER:$USER" grafana-nodes
git clone https://github.com/irfanoneverse/grafana.git grafana-nodes
cd /opt/grafana-nodes/client-server
```

Create the runtime environment file:

```text
cp .env.example .env
nano .env
```

Set the client values:

```env
HOSTNAME=laravel-app-01
INSTANCE_NAME=laravel-app-01
ENVIRONMENT=staging
TZ_NAME=Asia/Kuala_Lumpur
LARAVEL_LOG_DIR=/path/to/laravel/storage/logs
HUB_IP=10.0.1.50
```

Confirm the Laravel log directory exists:

```text
ls -lah /path/to/laravel/storage/logs
```

Start the stack:

```text
docker compose up -d
docker compose ps
docker compose logs --tail=80 alloy
```

Expected containers:

- `alloy`

## 5. Validate the Client

Check local services:

```text
curl -s http://127.0.0.1:12345/ | head -1
```

Check hub connectivity:

```text
nc -vz 10.0.1.50 3100
nc -vz 10.0.1.50 9009
```

Replace `10.0.1.50` with the hub private IP.

Check logs:

```text
docker compose logs --tail=100 alloy
```

## 6. Validate Data in Grafana

Open Grafana on the hub and go to **Explore**.

Loki queries:

```logql
{instance="laravel-app-01"}
{job="laravel", environment="staging"}
{job="nginx"}
```

Mimir or Prometheus queries:

```promql
up
up{instance="laravel-app-01"}
node_uname_info{instance="laravel-app-01"}
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle", instance="laravel-app-01"}[5m])) * 100)
100 - ((node_filesystem_avail_bytes{instance="laravel-app-01", fstype=~"ext[234]|btrfs|xfs|zfs", mountpoint!~".*boot.*"} * 100) / node_filesystem_size_bytes{instance="laravel-app-01", fstype=~"ext[234]|btrfs|xfs|zfs", mountpoint!~".*boot.*"})
```

Replace `laravel-app-01` and `staging` with the real client values.

This stack intentionally does not collect Nginx `stub_status` metrics or PHP-FPM pool metrics. Nginx and PHP-FPM log files are still collected if the mounted log paths exist.

## 7. Optional Laravel OpenTelemetry Cleanup

Run this only if the Laravel app previously used OpenTelemetry packages and is now moving to Alloy-based logs and metrics.

Use [opentelemetry-cleanup.md](opentelemetry-cleanup.md) for the cleanup workflow.

## 8. Operations

Common client commands:

```text
cd /opt/grafana-nodes/client-server
docker compose ps
docker compose logs -f alloy
docker compose restart alloy
docker compose pull
docker compose up -d
```

Stop the monitoring agent only:

```text
cd /opt/grafana-nodes/client-server
docker compose down
```

This does not stop Nginx, PHP-FPM, Laravel, databases, or application services.
