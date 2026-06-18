# Production Readiness Review — Grafana + Alloy + Loki Stack

**Date:** June 16, 2026  
**Scope:** Read-only review — no files were modified.  
**Architecture:** Two-tier observability stack.  

| Component | Server | Purpose |
|---|---|---|
| Loki 3.4.2 | `grafana-main-server` (hub EC2) | Central log aggregation, S3-backed storage |
| Grafana 11.5.2 | `grafana-main-server` (hub EC2) | UI on port 80, Loki datasource auto-provisioned |
| Grafana Alloy 1.8.3 | `client-server` (app EC2) | Tails Laravel `*.log` files, ships to hub over VPC private IP |

**Log flow:** Laravel `*.log` → Alloy file scrape → `loki.process` pipeline → `http://HUB_IP:3100/loki/api/v1/push` → Loki (S3-backed) → Grafana

---

## Severity Legend

| Symbol | Severity | Meaning |
|---|---|---|
| 🔴 | **CRITICAL** | Data loss, security breach, or service failure is occurring or imminent |
| 🟡 | **WARNING** | Production risk that needs fixing before this is considered stable |
| 🔵 | **MINOR** | Quality / reliability improvement; not a blocker |

---

## File-by-File Findings

---

### `grafana-main-server/docker-compose.yml`

---

#### 🔴 CRITICAL — Port 3100 bound to all interfaces

```yaml
# Line 15
ports:
  - "3100:3100"
```

`0.0.0.0:3100` is the default expansion. With `auth_enabled: false` in Loki, anyone who can reach the host's public IP can push arbitrary log streams or execute unlimited queries against your data — no credentials required.

**Recommended fix:** Bind to the VPC private interface only, since Alloy connects over the private IP:

```yaml
ports:
  - "${HUB_PRIVATE_IP}:3100:3100"
```

Or remove the port mapping entirely and rely on AWS Security Groups to restrict port 3100 to the VPC CIDR.

---

#### 🟡 WARNING — No resource limits on any service

Neither `loki` nor `grafana` has a `deploy.resources.limits` block. On a shared EC2 instance, Loki's memory usage (query cache, ingester flush buffer) or a runaway Grafana query can starve the host.

**Recommended fix:**

```yaml
# Add under each service
deploy:
  resources:
    limits:
      cpus: "1.5"
      memory: 1500M
    reservations:
      memory: 512M
```

Tune to your instance size (t3.medium: ~4 GB RAM, 2 vCPU).

---

#### 🟡 WARNING — Grafana exposed on plain HTTP port 80

```yaml
# Line 39
ports:
  - "80:3000"
```

The Grafana admin password and session tokens travel over cleartext HTTP. Combined with a default username of `admin`, this is a credential-theft risk on any network that isn't fully private.

**Recommended fix:** Put an Nginx or ALB HTTPS terminator in front. Set `GF_SERVER_PROTOCOL=https` or enforce HTTPS at the proxy layer.

---

#### 🟡 WARNING — Default admin username hardcoded

```yaml
# Line 24
- GF_SECURITY_ADMIN_USER=admin
```

Using the default `admin` username makes the account a predictable brute-force target.

**Recommended fix:** Change to a non-guessable name, injected via `.env`:

```yaml
- GF_SECURITY_ADMIN_USER=${GRAFANA_ADMIN_USER:?set GRAFANA_ADMIN_USER in .env}
```

---

#### 🔵 MINOR — Missing `start_period` on Grafana healthcheck

On first boot, Grafana provisioning (datasource files) may delay readiness beyond the `retries × interval` window (5 × 15s = 75s). The container could be declared unhealthy before it finishes loading.

**Recommended fix:** Add `start_period: 30s` to the Grafana healthcheck block.

---

### `grafana-main-server/loki-config.yaml`

---

#### 🔴 CRITICAL — Placeholder S3 bucket name left in config

```yaml
# Line ~20
common:
  storage:
    s3:
      bucketnames: your-company-observability-data
```

Loki will attempt to connect to a bucket literally named `your-company-observability-data`. This causes all chunk writes and index shipping to fail with S3 `NoSuchBucket` errors. Loki will appear to run but silently loses all data beyond what fits in the in-memory ingester.

**Recommended fix:** Replace with the real bucket name, or inject it via environment variable:

```yaml
bucketnames: ${LOKI_S3_BUCKET}
```

and set `LOKI_S3_BUCKET` in `.env` / Compose environment.

---

#### 🔴 CRITICAL — IMDSv2 hop limit blocks container S3 access

Loki uses the EC2 IAM Instance Profile (no `access_key_id` / `secret_access_key` set). This relies on the Instance Metadata Service (IMDS). Docker containers are one network hop away from the host, and IMDSv2 enforces a `http-put-response-hop-limit` (default: **1**) that blocks container access to IMDS.

Without this fix, all S3 operations fail silently with `NoCredentialProviders`.

**Recommended fix:** On the hub EC2 instance, increase the hop limit:

```bash
aws ec2 modify-instance-metadata-options \
  --instance-id i-xxxxxxxxxxxx \
  --http-put-response-hop-limit 2 \
  --http-endpoint enabled
```

---

#### 🟡 WARNING — `auth_enabled: false` with a reachable push endpoint

```yaml
auth_enabled: false
```

With port 3100 accessible, any actor can push log streams with arbitrary labels, polluting the label index and consuming S3 storage and retention budget. There is no per-tenant isolation.

**Recommended fix:** Restrict port 3100 at the network level (Security Group: allow only VPC CIDR). If external log sources are needed, enable auth and issue a bearer token for each Alloy agent.

---

#### 🟡 WARNING — `reject_old_samples_max_age` dramatically exceeds retention

```yaml
limits_config:
  retention_period: 100d
  reject_old_samples_max_age: 4380h   # 182 days
```

Loki accepts logs timestamped up to 182 days in the past, but retention is only 100 days. Logs between 100–182 days old are ingested into S3 at cost and immediately eligible for deletion by the compactor. This also opens a replay-flood vector from any host that can reach port 3100.

**Recommended fix:** Align with a realistic backfill window (e.g., 168h = 7 days):

```yaml
reject_old_samples_max_age: 168h
```

---

#### 🟡 WARNING — No ingester WAL configured explicitly

No explicit `ingester.wal` block is defined. Loki 3.x defaults to enabling the WAL at `<path_prefix>/wal` (`/loki/wal`), which is inside the named Docker volume and does persist. However, without an explicit config, WAL behavior can silently change on version upgrades.

**Recommended fix:**

```yaml
ingester:
  wal:
    enabled: true
    dir: /loki/wal
```

---

#### 🟡 WARNING — IAM permissions for compactor retention require `s3:DeleteObject`

```yaml
compactor:
  retention_enabled: true
  delete_request_store: s3
```

For the compactor to delete expired chunks, the IAM role needs `s3:DeleteObject`. If the instance profile policy only grants `s3:GetObject` / `s3:PutObject`, retention will silently fail and data will accumulate in S3 indefinitely past 100 days.

**Recommended fix:** Confirm the instance profile policy includes:

```json
"Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject", "s3:ListBucket"]
```

---

#### 🟡 WARNING — `max_query_parallelism: 2` is extremely restrictive

```yaml
limits_config:
  max_query_parallelism: 2
```

Loki's default is 32. A value of 2 means at most 2 sub-queries run in parallel for any single query. Long time-range queries in Grafana will be very slow.

**Recommended fix:** Start at 8–16 on a t3.medium and tune based on observed CPU saturation.

---

#### 🟡 WARNING — `compaction_interval: 10m` is unusually aggressive

```yaml
compactor:
  compaction_interval: 10m
```

The default in Loki 3.x is 2h. Compacting every 10 minutes significantly increases S3 API call frequency (ListObjects, GetObject, PutObject) and the compactor's CPU usage, driving up AWS costs with no correctness benefit.

**Recommended fix:** `compaction_interval: 30m` (or the default `2h`).

---

#### 🔵 MINOR — gRPC port 9096 unnecessarily defined

```yaml
server:
  grpc_listen_port: 9096
```

In single-binary mode, there are no internal gRPC calls between components. This port listens but serves nothing useful. It is not exposed in Compose, so it is not a security risk, but it is unnecessary noise.

**Recommended fix:** Remove `grpc_listen_port: 9096` for a single-binary deployment.

---

### `grafana-main-server/grafana/provisioning/datasources/datasources.yaml`

---

#### 🟡 WARNING — Provisioned datasource is editable

There is no `editable: false` set. Any Grafana user with Editor or Admin rights can modify the Loki datasource URL/config through the UI. Changes survive until the next container restart (when provisioning re-applies), creating a confusing split between file config and the running state.

**Recommended fix:**

```yaml
datasources:
  - name: Loki
    # ... existing fields ...
    editable: false
```

---

#### 🟡 WARNING — No query timeout configured on the datasource

If a large time-range query against Loki runs unboundedly, it holds a Grafana connection and blocks the browser with no clear error until the browser times out.

**Recommended fix:**

```yaml
jsonData:
  maxLines: 1000
  timeout: 60
```

---

#### 🔵 MINOR — `maxLines: 1000` may be too low for incident debugging

A single exception trace with context can span hundreds of lines. Hitting this limit during an incident investigation is frustrating.

**Recommended fix:** Consider `maxLines: 5000`.

---

### `client-server/docker-compose.yml`

---

#### 🟡 WARNING — Alloy UI bound to `0.0.0.0:12345` on host network

```yaml
command:
  - --server.http.listen-addr=0.0.0.0:12345
network_mode: host
```

`network_mode: host` means port 12345 is open on the client EC2's **public** interface. Alloy's UI exposes the live pipeline config, component graph, WAL status, and internal metrics — with no authentication. An external actor can view your `HUB_IP`, `INSTANCE_NAME`, `ENVIRONMENT`, and pipeline topology.

**Recommended fix:**

```yaml
- --server.http.listen-addr=127.0.0.1:12345
```

---

#### 🟡 WARNING — No resource limits

Alloy on a production app server with `network_mode: host` and no memory limit can exhaust host RAM during a log burst (e.g., an exception flood), impacting the Laravel application running on the same host.

**Recommended fix:** Add `mem_limit: 256m` (Docker standalone) or `deploy.resources.limits.memory: 256M` (Swarm/Compose v3).

---

#### 🟡 WARNING — `HOSTNAME` env var conflicts with the Linux shell built-in

```yaml
environment:
  - HOSTNAME=${HOSTNAME:?set HOSTNAME in .env}
```

`HOSTNAME` is a Bash built-in always set on Linux. When Docker Compose resolves `${HOSTNAME}`, it reads from the calling shell's environment **first**, silently overriding the `.env` file value with the machine's actual kernel hostname. The `{:?}` validation guard will also never fire because `HOSTNAME` is never empty.

**Recommended fix:** Rename to a non-conflicting variable:

```yaml
- ALLOY_HOSTNAME=${ALLOY_HOSTNAME:?set ALLOY_HOSTNAME in .env}
```

Update `config.alloy` accordingly: `hostname = env("ALLOY_HOSTNAME")`.

---

#### 🔵 MINOR — Healthcheck uses hex port `3039` — brittle and undocumented

```yaml
healthcheck:
  test: ["CMD-SHELL", "grep -q ':3039' /proc/net/tcp /proc/net/tcp6 || exit 1"]
```

Port 12345 decimal = `0x3039` hex, which is how `/proc/net/tcp` represents it. This works on Linux but only checks that the socket is bound — not that Alloy is healthy and processing logs. It will silently break if the listen port changes.

**Recommended fix:** Use Alloy's built-in readiness endpoint:

```yaml
test: ["CMD-SHELL", "wget --no-verbose --tries=1 --spider http://127.0.0.1:12345/-/ready || exit 1"]
```

---

#### 🔵 MINOR — Missing `start_period` on healthcheck

Alloy needs a few seconds to load config and open file positions. Without `start_period`, Docker may mark it unhealthy before initialization completes.

**Recommended fix:** Add `start_period: 15s`.

---

### `client-server/config.alloy`

---

#### 🟡 WARNING — No multi-line log handling for Laravel exception stack traces

There is no `stage.multiline` block. Laravel exception logs span multiple lines:

```
[2024-01-01 12:00:00] production.ERROR: Unhandled exception {"exception":"..."}
#0 /var/www/html/vendor/.../Pipeline.php(141): ...
#1 /var/www/html/vendor/.../Pipeline.php(119): ...
```

Without multi-line handling, each stack trace line becomes a separate Loki log entry. Lines starting with `#0`, `#1`, etc. don't match the `[YYYY-MM-DD]` regex, so they land in Loki without `level` or `environment` labels — making exception correlation nearly impossible.

**Recommended fix:** Add a `stage.multiline` block inside `loki.process "laravel_pipeline"`, before the first `stage.regex`:

```alloy
stage.multiline {
  firstline     = "^\\["
  max_wait_time = "3s"
}
```

---

#### 🟡 WARNING — Timestamp format does not handle `T` separator or fractional seconds

```alloy
// Regex captures: 2024-01-01T12:00:00.123456  (allows T and microseconds)
expression = "^\\[(?P<timestamp>\\d{4}-\\d{2}-\\d{2}[T ]\\d{2}:\\d{2}:\\d{2}\\.?\\d*)\\]..."

// Format only handles: 2024-01-01 12:00:00  (space only, no fractional seconds)
format = "2006-01-02 15:04:05"
```

Any log line with a `T` separator or microseconds will silently fail timestamp parsing. Alloy falls back to ingestion time, creating phantom time skew in Grafana (logs appear timestamped when they arrived at Alloy, not when they were generated).

**Recommended fix:** Either tighten the regex to only accept what Laravel actually produces, or use fallback formats:

```alloy
stage.timestamp {
  source          = "timestamp"
  format          = "2006-01-02 15:04:05"
  format_fallback = ["2006-01-02T15:04:05", "2006-01-02 15:04:05.000000"]
  location        = env("TZ_NAME")
}
```

---

#### 🟡 WARNING — `loki.write` has no retry, batch, or timeout configuration

```alloy
loki.write "loki_endpoint" {
  endpoint {
    url = "http://" + env("HUB_IP") + ":3100/loki/api/v1/push"
  }
}
```

With all-default settings, during a Loki outage Alloy retries with exponential backoff but the WAL is bounded. Under a log flood (e.g., an exception loop), the WAL can fill before Loki recovers, causing silent log loss with no visible error.

**Recommended fix:**

```alloy
loki.write "loki_endpoint" {
  endpoint {
    url            = "http://" + env("HUB_IP") + ":3100/loki/api/v1/push"
    batch_wait     = "1s"
    batch_size     = "1MiB"
    remote_timeout = "10s"
    min_backoff    = "500ms"
    max_backoff    = "5m"
    max_retries    = 10
  }
}
```

---

#### 🟡 WARNING — `public_ipv4` as a Loki label risks stream cardinality growth

```alloy
stage.static_labels {
  values = {
    private_ipv4 = env("PRIVATE_IPV4"),
    public_ipv4  = env("PUBLIC_IPV4"),
  }
}
```

Each unique `{public_ipv4="x.x.x.x"}` value creates a new Loki stream permanently in the TSDB index on S3. For elastic IPs or auto-scaling groups that cycle IPs, the stream count grows unboundedly over time.

**Recommended fix:** Drop `public_ipv4` as a label (it's redundant with `instance`). If IP visibility is needed, use Loki 3.x structured metadata instead:

```alloy
stage.structured_metadata {
  values = {
    public_ipv4  = "public_ipv4",
    private_ipv4 = "private_ipv4",
  }
}
```

This attaches the values to log entries without creating new streams.

---

#### 🔵 MINOR — `tail_from_end = true` skips pre-existing logs on first deploy

```alloy
loki.source.file "log_scrape" {
  tail_from_end = true
}
```

On initial deployment or if the WAL volume is wiped, Alloy will not backfill existing log file content. Existing error history will not be visible in Grafana after a fresh deploy.

**Recommended fix:** Document this behavior in the README. If backfill is required, temporarily set `tail_from_end = false` for the first run.

---

#### 🔵 MINOR — No self-monitoring: Alloy's own metrics are not collected

Neither `config.alloy` nor any Compose file scrapes Alloy's Prometheus metrics endpoint at `http://localhost:12345/metrics`. If Alloy silently drops logs (WAL overflow, failed Loki push, regex parse errors), there is no alert or dashboard to detect it.

**Recommended fix:** Add a `prometheus.scrape` component for Alloy self-metrics and forward to a Prometheus instance, or document the metrics endpoint for manual inspection.

---

### `.env.example` Files — Both Servers

---

#### 🔴 CRITICAL — No `.gitignore` file in the repository

There is no `.gitignore` at the repo root or in either subdirectory. The `.env` files (created from `.env.example`) containing `GRAFANA_ADMIN_PASSWORD`, `HUB_IP`, and IP addresses are not excluded from git tracking. A single `git add .` would commit credentials.

**Recommended fix:** Create `/.gitignore` at the repo root:

```gitignore
# Environment secrets — never commit these
.env
*.env
grafana-main-server/.env
client-server/.env
```

---

#### 🟡 WARNING — `GRAFANA_ROOT_URL` defaults to plain HTTP

```env
# grafana-main-server/.env.example
GRAFANA_ROOT_URL=http://MONITORING_HUB_PUBLIC_IP/
```

If Grafana is placed behind an HTTPS load balancer, `GF_SERVER_ROOT_URL` must use `https://` or Grafana generates incorrect redirect URLs and OAuth callbacks.

**Recommended fix:** Default the example to HTTPS:

```env
GRAFANA_ROOT_URL=https://YOUR_MONITORING_DOMAIN/
```

---

## Cross-Cutting Issues

---

#### 🟡 WARNING — No self-monitoring for Loki

Loki exposes metrics at `http://loki:3100/metrics`. Neither Alloy nor any other component scrapes it. Loki compaction failures, ingestion throttling (per-stream rate limit hits), S3 errors, and query timeouts are all invisible unless container logs are checked manually.

---

#### 🟡 WARNING — No alerting configured anywhere

There are no Grafana alert rules, Loki ruler rules, or notification channels provisioned. A complete S3 outage, Alloy crash, or Loki OOM-kill would go undetected until someone checks the Grafana UI.

---

#### 🔵 MINOR — No dashboard provisioning

The `grafana/provisioning/` directory contains only `datasources/`. There are no pre-built dashboards for log volume, error rate, ingestion latency, or Loki internals. An operator deploying this for the first time has no visibility into stack health out of the box.

---

## Consolidated Summary

| # | Severity | File | Issue |
|---|---|---|---|
| 1 | 🔴 CRITICAL | `grafana-main-server/docker-compose.yml` | Port 3100 bound to all interfaces, no auth |
| 2 | 🔴 CRITICAL | `grafana-main-server/loki-config.yaml` | Placeholder S3 bucket name — data is being lost |
| 3 | 🔴 CRITICAL | `grafana-main-server/loki-config.yaml` | IMDSv2 hop limit blocks S3 access from container |
| 4 | 🔴 CRITICAL | `(repo root)` | No `.gitignore` — secrets can be committed |
| 5 | 🟡 WARNING | `grafana-main-server/docker-compose.yml` | No resource limits on Loki or Grafana |
| 6 | 🟡 WARNING | `grafana-main-server/docker-compose.yml` | Grafana on plain HTTP, default `admin` username |
| 7 | 🟡 WARNING | `grafana-main-server/loki-config.yaml` | `auth_enabled: false` + exposed port = unauthenticated access |
| 8 | 🟡 WARNING | `grafana-main-server/loki-config.yaml` | `reject_old_samples_max_age` (182d) far exceeds retention (100d) |
| 9 | 🟡 WARNING | `grafana-main-server/loki-config.yaml` | No explicit WAL config for ingester |
| 10 | 🟡 WARNING | `grafana-main-server/loki-config.yaml` | IAM policy must include `s3:DeleteObject` for retention to work |
| 11 | 🟡 WARNING | `grafana-main-server/loki-config.yaml` | `max_query_parallelism: 2` causes very slow queries |
| 12 | 🟡 WARNING | `grafana-main-server/loki-config.yaml` | `compaction_interval: 10m` inflates S3 API costs |
| 13 | 🟡 WARNING | `grafana-main-server/datasources.yaml` | Provisioned datasource is editable via UI |
| 14 | 🟡 WARNING | `grafana-main-server/datasources.yaml` | No query timeout on Loki datasource |
| 15 | 🟡 WARNING | `client-server/docker-compose.yml` | Alloy UI on `0.0.0.0:12345` (host network) — publicly exposed |
| 16 | 🟡 WARNING | `client-server/docker-compose.yml` | No resource limits on Alloy |
| 17 | 🟡 WARNING | `client-server/docker-compose.yml` | `HOSTNAME` conflicts with Linux shell built-in |
| 18 | 🟡 WARNING | `client-server/config.alloy` | No multi-line parsing — exception stack traces split into individual log entries |
| 19 | 🟡 WARNING | `client-server/config.alloy` | Timestamp format mismatch — `T` separator or microseconds cause silent parse failure |
| 20 | 🟡 WARNING | `client-server/config.alloy` | `loki.write` has no retry/batch/timeout config |
| 21 | 🟡 WARNING | `client-server/config.alloy` | `public_ipv4` / `private_ipv4` as labels risk stream cardinality growth |
| 22 | 🟡 WARNING | `grafana-main-server/.env.example` | `GRAFANA_ROOT_URL` defaults to HTTP |
| 23 | 🟡 WARNING | `(cross-cutting)` | No self-monitoring for Loki or Alloy metrics |
| 24 | 🟡 WARNING | `(cross-cutting)` | No alerting rules or notification channels |
| 25 | 🔵 MINOR | `grafana-main-server/docker-compose.yml` | Missing `start_period` on Grafana healthcheck |
| 26 | 🔵 MINOR | `grafana-main-server/loki-config.yaml` | gRPC port 9096 unnecessarily defined |
| 27 | 🔵 MINOR | `grafana-main-server/datasources.yaml` | `maxLines: 1000` too low for incident debugging |
| 28 | 🔵 MINOR | `client-server/docker-compose.yml` | Healthcheck uses brittle `/proc/net/tcp` hex port `3039` |
| 29 | 🔵 MINOR | `client-server/docker-compose.yml` | Missing `start_period` on Alloy healthcheck |
| 30 | 🔵 MINOR | `client-server/config.alloy` | `tail_from_end = true` skips pre-existing logs on first deploy |
| 31 | 🔵 MINOR | `client-server/config.alloy` | No Alloy self-monitoring (metrics not scraped) |
| 32 | 🔵 MINOR | `(cross-cutting)` | No Grafana dashboard provisioning |

---

## Recommended Fix Order

Fix these first — they represent active failures or high-severity security risks:

1. **Replace placeholder S3 bucket name** — Loki is silently losing all data if not already changed.
2. **IMDSv2 hop limit** — Run the `aws ec2 modify-instance-metadata-options` command on the hub EC2. Without it, all S3 operations fail regardless of IAM policy.
3. **Restrict port 3100** — Bind to private IP or remove the port mapping; rely on Security Groups.
4. **Alloy UI to `127.0.0.1:12345`** — One flag change; stops pipeline internals being publicly browsable.
5. **Add `.gitignore`** — One file; prevents secrets from ever entering git history.
