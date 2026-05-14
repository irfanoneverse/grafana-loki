# Client Server Guide

This guide deploys the client-side monitoring stack from `client-server/` on each Laravel or application EC2 instance.

The application stays native on the host. Only these monitoring containers run in Docker:

- `alloy`
- `nginx-exporter`
- `phpfpm-exporter`

The client sends logs to Loki and metrics to Mimir on the monitoring hub.

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

## 4. Enable PHP-FPM Status

Find the active PHP-FPM pool:

```text
ls -lah /etc/php/*/fpm/pool.d/www.conf
grep -R "listen =" /etc/php/*/fpm/pool.d/www.conf
```

Edit only the active pool file:

```text
sudo nano /etc/php/8.2/fpm/pool.d/www.conf
```

Replace `8.2` with the active PHP version. Add or update this line:

```ini
pm.status_path = /fpm-status
```

Restart or reload PHP-FPM:

```text
sudo systemctl reload php8.2-fpm
```

Confirm PHP-FPM is running and note the socket path:

```text
systemctl --no-pager --type=service | grep php | grep fpm
ls -lah /run/php/
```

Common socket examples:

- `/run/php/php8.2-fpm.sock`
- `/run/php/php8.3-fpm.sock`
- `/run/php/php-fpm.sock`

## 5. Enable Local Nginx and PHP-FPM Metrics Endpoints

Create a local-only Nginx status server:

```text
sudo nano /etc/nginx/conf.d/stub_status.conf
```

Use this content, replacing the socket path with the real one from `/run/php/`:

```nginx
server {
    listen 127.0.0.1:8080;
    server_name localhost;

    location = /nginx_status {
        stub_status on;
        allow 127.0.0.1;
        deny all;
    }

    location = /fpm-status {
        fastcgi_pass unix:/run/php/php8.2-fpm.sock;
        fastcgi_param SCRIPT_NAME     /fpm-status;
        fastcgi_param SCRIPT_FILENAME /fpm-status.php;
        fastcgi_param REQUEST_URI     /fpm-status;
        fastcgi_param QUERY_STRING    $query_string;
        fastcgi_param REQUEST_METHOD  $request_method;
        allow 127.0.0.1;
        deny all;
    }
}
```

Validate and reload Nginx:

```text
sudo nginx -t
sudo systemctl reload nginx
```

Verify the endpoints:

```text
curl -s http://127.0.0.1:8080/nginx_status
curl -s http://127.0.0.1:8080/fpm-status
```

Expected:

- `nginx_status` returns active connections and request counters.
- `fpm-status` returns PHP-FPM pool status.

If `fpm-status` returns `502`, the `fastcgi_pass` socket path is wrong. Update it, test Nginx again, and reload Nginx.

## 6. Deploy the Client Stack

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
- `nginx-exporter`
- `phpfpm-exporter`

## 7. Validate the Client

Check local services:

```text
curl -s http://127.0.0.1:12345/ | head -1
curl -s http://127.0.0.1:9113/metrics | head -5
curl -s http://127.0.0.1:9253/metrics | head -5
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
docker compose logs --tail=80 nginx-exporter
docker compose logs --tail=80 phpfpm-exporter
```

## 8. Validate Data in Grafana

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
nginx_connections_active{instance="laravel-app-01"}
phpfpm_up{instance="laravel-app-01"}
```

Replace `laravel-app-01` and `staging` with the real client values.

## 9. Optional Laravel OpenTelemetry Cleanup

Run this only if the Laravel app previously used OpenTelemetry packages and is now moving to Alloy-based logs and metrics.

Use [opentelemetry-cleanup.md](opentelemetry-cleanup.md) for the cleanup workflow.

## 10. Operations

Common client commands:

```text
cd /opt/grafana-nodes/client-server
docker compose ps
docker compose logs -f alloy
docker compose logs -f nginx-exporter
docker compose logs -f phpfpm-exporter
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
