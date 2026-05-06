# CIPS Ubuntu EC2 - Remove OpenTelemetry (Keep Grafana Alloy)

This is a simplified cleanup guide for the CIPS staging EC2 node where OpenTelemetry was used before, and now only Grafana Alloy is used.

CIPS paths and labels:

- Laravel app directory: `/data/cips`
- Alloy directory: `/opt/monitoring/alloy-docker`
- Instance name: `cips-staging`

## 1) Check what OpenTelemetry footprint still exists

```bash
sudo systemctl list-unit-files | grep -Ei 'otel|opentelemetry|collector'
docker ps --format '{{.Names}} {{.Image}}' | grep -Ei 'otel|opentelemetry|collector'
ps aux | grep -Ei 'otel|opentelemetry|collector' | grep -v grep
```

## 2) Remove OpenTelemetry from Laravel app

Run inside your Laravel app directory:

```bash
cd /data/cips
composer remove keepsuit/laravel-opentelemetry open-telemetry/sdk open-telemetry/exporter-otlp
rm -f config/opentelemetry.php
```

### 2a) Check for remaining OTel references in app code

Even after removing packages, middleware or other files may still import OTel classes and cause a 500 on every request. Scan for them:

```bash
grep -r "opentelemetry\|OpenTelemetry\|keepsuit" config/ bootstrap/ app/ 2>/dev/null
```

If any files appear (e.g. `app/Http/Middleware/TraceIdMiddleware.php`), remove the OTel dependency from them. For a middleware that only injected a trace ID into logs, replace the entire file with a safe no-op:

```bash
test -f app/Http/Middleware/TraceIdMiddleware.php

cat > app/Http/Middleware/TraceIdMiddleware.php << 'EOF'
<?php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;

class TraceIdMiddleware
{
    public function handle(Request $request, Closure $next)
    {
        return $next($request);
    }
}
EOF
```

### 2b) Remove OTEL\_\* variables from .env

```bash
grep -n '^OTEL_' /data/cips/.env || true
sed -i '/^OTEL_/d' /data/cips/.env
```

Also remove from:

- deployment secrets / CI variables
- process manager configs (if any)

### 2c) Fix storage permissions (if composer was run as root)

Running `composer` as root changes ownership of storage files, causing PHP-FPM to fail writing logs. Fix with:

```bash
chown -R www-data:www-data /data/cips/storage
chown -R www-data:www-data /data/cips/bootstrap/cache
```

### 2d) Clear Laravel caches

```bash
cd /data/cips
php artisan optimize:clear
```

> **Note:** If `artisan` itself throws an OTel class error, it means the config cache still references the deleted config. Manually delete the cache files first:
>
> ```bash
> rm -f bootstrap/cache/config.php
> rm -f bootstrap/cache/packages.php
> rm -f bootstrap/cache/services.php
> ```
>
> Then re-run `php artisan optimize:clear`.

## 3) Stop and disable OpenTelemetry services

```bash
sudo systemctl stop otelcol otel-collector aws-otel-collector 2>/dev/null || true
sudo systemctl disable otelcol otel-collector aws-otel-collector 2>/dev/null || true
sudo systemctl daemon-reload
```

## 4) Uninstall OpenTelemetry packages (Ubuntu)

```bash
sudo apt update
sudo apt remove -y otelcol otelcol-contrib aws-otel-collector || true
sudo apt autoremove -y
```

## 5) Delete leftover OpenTelemetry files (if present)

```bash
sudo rm -rf /etc/otelcol* /etc/otel-collector* /opt/aws/aws-otel-collector /var/lib/otelcol* /var/log/otel* 2>/dev/null || true
```

## 6) Remove old OpenTelemetry Docker containers/images (if used before)

```bash
docker ps -a --format '{{.ID}} {{.Names}} {{.Image}}' | grep -Ei 'otel|opentelemetry|collector'
```

If any appear, remove them explicitly.

## 7) Verify only Grafana Alloy path is active

```bash
cd /opt/monitoring/alloy-docker
docker compose ps
docker compose logs --tail=100 alloy
curl -s http://127.0.0.1:12345/ | head -1
```

Confirm Alloy is using the CIPS values:

```bash
grep -nE 'INSTANCE_NAME|ENVIRONMENT|LARAVEL_LOG_DIR|HUB_IP' /opt/monitoring/alloy-docker/.env
docker compose exec alloy env | grep -E 'INSTANCE_NAME|ENVIRONMENT|HUB_IP'
```

Expected CIPS values:

```text
INSTANCE_NAME=cips-staging
ENVIRONMENT=staging
LARAVEL_LOG_DIR=/data/cips/storage/logs
HUB_IP=172.31.27.45
```

If Alloy is healthy and no OpenTelemetry service/process/container remains, cleanup is complete.
