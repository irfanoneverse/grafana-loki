# Client Server — Laravel Log Shipping (Loki)

Runs Grafana Alloy in Docker to ship Laravel app logs to a central Loki instance. Does not touch your app or its services.

## Prerequisites

- Docker + Docker Compose on the app server
- Loki running on your monitoring hub, reachable via private IP

## Setup

```bash
cp .env.example .env
nano .env
```

Set these in `.env`:

| Variable | Description |
|---|---|
| `HOSTNAME` / `INSTANCE_NAME` | Unique name for this server |
| `ENVIRONMENT` | `staging` or `production` |
| `TZ_NAME` | Server timezone, e.g. `Asia/Kuala_Lumpur` |
| `LARAVEL_LOG_DIR` | Full path to `storage/logs` on this server |
| `PRIVATE_IPV4` / `PUBLIC_IPV4` | IPs of this server |
| `HUB_IP` | Private IP of the server running Loki |

Then start:

```bash
docker compose up -d
docker compose logs -f alloy   # verify it's running
```

## Logs collected

- `$LARAVEL_LOG_DIR/*.log` — all Laravel channel logs

Each log line is labelled with `hostname`, `instance`, `environment`, `level`, and `channel`.

## Notes

- Loki endpoint: `http://<HUB_IP>:3100` — ensure port `3100` is open from this server to the hub
- To stop: `docker compose down`
- To restart after config changes: `docker compose restart alloy`
