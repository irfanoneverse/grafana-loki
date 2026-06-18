# DEPLOYMENT.md — Production Deployment Guide
# Grafana + Loki + Alloy Observability Stack

**Version:** 1.0  
**Date:** June 2026  
**Stack:** Grafana Alloy 1.8.3 → Loki 3.4.2 → Grafana 11.5.2  
**Status:** Pre-Production — Required fixes must be applied before go-live  

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Current Architecture Review](#2-current-architecture-review)
3. [Production Readiness Assessment](#3-production-readiness-assessment)
4. [Required Fixes Before Production](#4-required-fixes-before-production)
5. [Recommended Architecture](#5-recommended-architecture)
6. [Main Server Deployment Guide](#6-main-server-deployment-guide)
7. [Client Server Deployment Guide](#7-client-server-deployment-guide)
8. [Security Hardening Checklist](#8-security-hardening-checklist)
9. [Monitoring & Alerting Setup](#9-monitoring--alerting-setup)
10. [Backup & Disaster Recovery](#10-backup--disaster-recovery)
11. [Scaling Strategy](#11-scaling-strategy)
12. [Cost Optimization Recommendations](#12-cost-optimization-recommendations)
13. [Operational Runbook](#13-operational-runbook)
14. [Validation Checklist](#14-validation-checklist)
15. [Go-Live Checklist](#15-go-live-checklist)
16. [Post-Deployment Verification](#16-post-deployment-verification)
17. [Future Improvements](#17-future-improvements)

---

## 1. Executive Summary

This document is the official production deployment guide for the centralized log observability stack used to aggregate Laravel application logs from approximately 20 EC2 instances. It incorporates findings from the repository code review (`REVIEW.md`, June 16 2026) and extends them with infrastructure sizing, security hardening, operational procedures, and a go-live checklist.

### Architecture in One Sentence

Twenty Laravel application EC2 instances each run a **Grafana Alloy** agent (Docker) that tails `storage/logs/*.log` files and pushes log lines over HTTP to a central **Loki** instance on a dedicated monitoring hub EC2. **Grafana** runs on the same hub and queries Loki for visualization. Log data is durably stored in **AWS S3**.

### Current Status: NOT Production-Ready

The implementation is architecturally sound and well-documented, but contains **4 critical blockers** and **18 warnings** that must be resolved before handling production workloads. The most severe risks are:

| Priority | Issue | Impact |
|---|---|---|
| P0 | IMDSv2 hop limit not increased | All S3 writes fail — Loki loses all data beyond in-memory buffer |
| P0 | S3 bucket name is a placeholder | Same as above — `NoSuchBucket` on every write |
| P0 | Port 3100 exposed to 0.0.0.0 with no auth | Anyone on the internet can push or read logs |
| P0 | No `.gitignore` | Admin password and IPs can be committed to git |

> **Operational Decision:** Do not allow production traffic to flow through this stack until all P0 items in Section 4 are resolved and verified.

### What Is Working Well

- Clean two-tier architecture with clear separation of concerns
- IAM Instance Profile (no static credentials) — correct approach
- S3 backend with TSDB v13 schema — appropriate for this scale
- Alloy runs in Docker with WAL persistence — survives agent restarts
- Log processing pipeline handles multi-channel Laravel logs correctly
- Security Group design intent is correct (port 3100 restricted to client SG)
- Lifecycle rules align S3 expiry with Loki retention period (100 days)

---

## 2. Current Architecture Review

### 2.1 Component Inventory

| Component | Version | Host | Role |
|---|---|---|---|
| Grafana Alloy | 1.8.3 | Each client EC2 | Log collection agent |
| Grafana Loki | 3.4.2 | Hub EC2 | Log aggregation and storage |
| Grafana | 11.5.2 | Hub EC2 | Visualization and querying |
| AWS S3 | — | AWS managed | Long-term chunk and index storage |
| Docker / Compose | Latest | All EC2s | Container runtime |

### 2.2 Data Flow

```
Laravel App (native / systemctl)
  └── writes to /var/www/app/storage/logs/*.log
        │
        │ bind mount (read-only)
        ▼
  [Docker] Grafana Alloy 1.8.3
        │
        │ local.file_match → loki.source.file
        │ → loki.process (regex, timestamp, labels)
        │ → loki.write (HTTP push)
        │
        │ http://HUB_PRIVATE_IP:3100/loki/api/v1/push
        │ (over VPC private network)
        ▼
  [Docker] Loki 3.4.2 (hub EC2)
        │
        ├── in-memory ingester → WAL (/loki/wal on named volume)
        └── chunk flush + TSDB index → AWS S3
              │
              ▼
        [Docker] Grafana 11.5.2
              └── Explore / Dashboards → queries Loki via HTTP
```

### 2.3 Label Schema

Every log line is tagged with the following Loki labels:

| Label | Source | Cardinality Risk |
|---|---|---|
| `job` | static (`laravel`) | Low |
| `app` | static (`laravel`) | Low |
| `hostname` | `HOSTNAME` env var | Low (~20 values) |
| `instance` | `INSTANCE_NAME` env var | Low (~20 values) |
| `environment` | `ENVIRONMENT` env var | Low (staging/production) |
| `level` | regex from log line | Low (debug/info/warning/error/critical) |
| `log_environment` | regex from log line | Low (matches app env) |
| `channel` | regex from filename | Low (laravel/audit/worker etc.) |
| `private_ipv4` | `PRIVATE_IPV4` env var | **Medium — static on EC2, low risk now** |
| `public_ipv4` | `PUBLIC_IPV4` env var | **HIGH — changes on EIP reassignment** |

> **Decision:** `public_ipv4` should be demoted from a label to Loki 3.x structured metadata. See Section 4.

### 2.4 Storage Design

- **Index type:** TSDB v13 (correct for Loki 3.x, optimal query performance)
- **Index period:** 24h (standard)
- **Object store:** S3 (`ap-southeast-1` by default — confirm matches your region)
- **Retention:** 100 days (aligned with S3 lifecycle expiry rule)
- **Compaction:** every 10 minutes (too frequent — see Section 4)
- **Local volume:** Named Docker volume `loki-data` for WAL and index cache

---

## 3. Production Readiness Assessment

### 3.1 Scalability — ⚠️ Conditional

**Current capacity:**

- 20 Alloy agents pushing logs over HTTP
- Single Loki instance (single-binary mode)
- Single Grafana instance

**Assessment:** Single-binary Loki with S3 backend scales well horizontally for read workloads but has a hard ceiling on ingest rate. For 20 modest Laravel applications, ingest rate is well within limits. The configured `ingestion_rate_mb: 10` (per tenant, but auth is disabled so effectively global) and `per_stream_rate_limit: 5MB` are adequate for this use case.

**Bottleneck:** If log volume spikes simultaneously across all 20 clients (e.g., a deployment that produces verbose logs), the single Loki ingester can queue and eventually begin rejecting samples. The `ingestion_burst_size_mb: 20` provides some headroom.

**Verdict:** Adequate for 20 clients. Will not scale to 50+ clients without moving to Loki distributed mode or adding a read-write split configuration.

### 3.2 Reliability — ❌ Not Production-Ready

**Single points of failure:**

| Component | SPOF Risk | Mitigation Available |
|---|---|---|
| Hub EC2 | Complete loss of ingest and query | Auto-recovery via S3 backend (data survives restart) |
| Loki container | Same as above | `restart: unless-stopped` handles crashes |
| Grafana container | Loss of visualization only | `restart: unless-stopped` handles crashes |
| S3 | Effectively zero (AWS SLA 99.99%) | Multi-AZ by design |
| Client Alloy WAL | Covers ~1–2h of logs on restart | Named Docker volume persists across container restarts |

**Key reliability gap:** Loki WAL is not explicitly configured. Loki 3.x defaults to `enabled: true` at `<path_prefix>/wal` but this must be explicit to survive version upgrades changing defaults.

**Verdict:** For a monitoring-only stack (not a SaaS product), a single Loki instance is acceptable, provided the EC2 has a robust recovery playbook and WAL is explicitly configured.

### 3.3 Security — ❌ Critical Gaps

| Risk | Severity | Status |
|---|---|---|
| Port 3100 on 0.0.0.0 | Critical | Not fixed |
| No Loki authentication | Critical | By design — mitigated only by SG |
| Grafana on HTTP only | Warning | Not fixed |
| Default admin username | Warning | Not fixed |
| No `.gitignore` | Critical | Not present |
| Alloy UI on 0.0.0.0:12345 | Warning | Not fixed |

**Verdict:** Security posture relies entirely on AWS Security Groups being correctly configured. This is acceptable in a VPC-only internal tool, but must be validated before go-live.

### 3.4 Performance — ⚠️ Needs Tuning

**`max_query_parallelism: 2`** is the most significant performance issue. On a t3.medium or larger instance with multiple vCPUs, this artificially limits Loki to 2 parallel query shards, making time-range queries over large datasets very slow.

**`compaction_interval: 10m`** generates 6 compaction cycles per hour, each of which issues S3 LIST and PUT API calls. At scale, this inflates costs and adds unnecessary I/O pressure.

**`maxLines: 1000`** in the Grafana datasource config limits log results to 1000 lines per query — useful as a default safeguard but too low for incident debugging where hundreds of thousands of lines may need to be scrolled.

**Verdict:** Query performance will degrade noticeably for time ranges beyond a few hours with the current `max_query_parallelism: 2`. Fix is a one-line config change.

### 3.5 Maintainability — ✅ Good

- Config is well-commented and readable
- `.env.example` clearly documents all required variables
- README documentation is thorough and step-by-step
- Docker Compose with named volumes and health checks follows best practices
- Separate repos/directories for hub and client are clean

### 3.6 Operational Complexity — ✅ Appropriate

For a 20-client deployment, single-binary Loki on one EC2 is the right complexity level. Distributed Loki (with separate ingesters, queriers, compactors) would be significant overengineering for this scale.

---

## 4. Required Fixes Before Production

This section lists every fix that must be applied, ordered by severity. **P0 items are hard blockers — the stack should not receive production traffic without them.**

---

### P0 — Critical Blockers

#### FIX-01: Increase IMDSv2 Hop Limit on Hub EC2

**Why:** Docker containers sit one network hop away from the EC2 host. IMDSv2 enforces `http-put-response-hop-limit=1` by default, which blocks the container from reaching IMDS to retrieve the IAM Instance Profile credentials. Without credentials, all Loki S3 operations fail with `NoCredentialProviders`. Loki will appear to run but silently discard all data beyond its in-memory ingester buffer.

**Fix:** Run on the hub EC2 (replace `i-xxxxxxxxxxxx` with the actual instance ID):

```bash
aws ec2 modify-instance-metadata-options \
  --instance-id i-xxxxxxxxxxxx \
  --http-put-response-hop-limit 2 \
  --http-endpoint enabled \
  --region ap-southeast-1
```

**Verify:**

```bash
# From inside the Loki container
docker exec -it loki wget -qO- http://169.254.169.254/latest/meta-data/iam/security-credentials/
# Should return the role name (GrafanaServerRole)
```

---

#### FIX-02: Replace Placeholder S3 Bucket Name

**Why:** The `loki-config.yaml` shipped with a placeholder value `your-company-observability-data`. If deployed without changing this, Loki attempts to write to a bucket that does not exist, producing `NoSuchBucket` errors on every chunk flush and index write.

**Fix in `grafana-main-server/loki-config.yaml`:**

```yaml
# Change this:
bucketnames: your-company-observability-data

# To your actual bucket name:
bucketnames: <your-actual-s3-bucket-name>
```

Alternatively, inject via environment variable for safer multi-environment deployments:

```yaml
# loki-config.yaml
bucketnames: ${LOKI_S3_BUCKET}
```

```yaml
# docker-compose.yml — add to loki environment:
environment:
  - LOKI_S3_BUCKET=${LOKI_S3_BUCKET:?set LOKI_S3_BUCKET in .env}
```

**Verify:**

```bash
docker compose logs --tail=50 loki | grep -i "s3\|bucket\|credential"
aws s3 ls s3://your-actual-bucket/loki/ --region ap-southeast-1
```

---

#### FIX-03: Restrict Loki Port 3100 to VPC Private IP

**Why:** The current `ports: ["3100:3100"]` binding exposes Loki on `0.0.0.0:3100`, which includes the public ENI of the EC2. Since `auth_enabled: false`, anyone who can reach the host IP on port 3100 (if the Security Group is misconfigured, even temporarily) can freely push log data or execute unlimited queries against your entire log history.

**Fix in `grafana-main-server/docker-compose.yml`:**

```yaml
# Option A — Bind to private IP only (most defensive)
# Add HUB_PRIVATE_IP to .env
ports:
  - "${HUB_PRIVATE_IP}:3100:3100"

# Option B — Remove the port mapping entirely
# Let Docker networking handle internal access; SG is the firewall
# (Only works if Alloy connects via host network or Docker DNS)
# NOT recommended with client-server separation — use Option A.
```

**Recommended:** Option A. Add `HUB_PRIVATE_IP` to `grafana-main-server/.env.example` and `.env`.

**Verify:**

```bash
# Should show binding on private IP only, not 0.0.0.0
ss -tlnp | grep 3100
# External probe (should time out or refuse)
curl --connect-timeout 3 http://PUBLIC_IP:3100/ready
```

---

#### FIX-04: Create `.gitignore` at Repository Root

**Why:** There is no `.gitignore` in the repository. The `.env` files created from `.env.example` contain `GRAFANA_ADMIN_PASSWORD`, `HUB_IP`, server IPs, and other sensitive values. A single `git add .` or IDE auto-stage will commit these credentials to version history, where they persist indefinitely even after deletion.

**Fix — create `/.gitignore`:**

```gitignore
# Environment secrets — never commit
.env
*.env
grafana-main-server/.env
client-server/.env

# Loki local data (if any)
grafana-main-server/loki-data/
grafana-main-server/grafana-data/

# OS artifacts
.DS_Store
Thumbs.db
```

---

### P1 — High Priority (Fix Before Production)

#### FIX-05: Add Resource Limits to All Services

**Why:** Without resource limits on a t3.medium (2 vCPU, 4 GB RAM), a single runaway Grafana query or a Loki compaction burst can exhaust host memory, triggering the OOM killer. This would kill either Loki or Grafana — or both — without warning, taking down the entire observability stack.

**Fix in `grafana-main-server/docker-compose.yml`:**

```yaml
services:
  loki:
    # ... existing config ...
    deploy:
      resources:
        limits:
          cpus: "1.5"
          memory: 2048M
        reservations:
          cpus: "0.5"
          memory: 512M

  grafana:
    # ... existing config ...
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 512M
        reservations:
          cpus: "0.1"
          memory: 128M
```

**Fix in `client-server/docker-compose.yml`:**

```yaml
services:
  alloy:
    # ... existing config ...
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 256M
        reservations:
          cpus: "0.05"
          memory: 64M
```

---

#### FIX-06: Configure Ingester WAL Explicitly

**Why:** Loki 3.x defaults to WAL-enabled at `<path_prefix>/wal`, but relying on undocumented defaults is fragile. A version upgrade could change the default without a breaking-change notice. Explicit config makes behavior auditable and upgrade-safe.

**Fix in `grafana-main-server/loki-config.yaml`:**

```yaml
ingester:
  wal:
    enabled: true
    dir: /loki/wal
  lifecycler:
    ring:
      replication_factor: 1
```

---

#### FIX-07: Correct `reject_old_samples_max_age` Alignment

**Why:** `reject_old_samples_max_age: 4380h` (182 days) allows logs timestamped up to 6 months in the past to be ingested, but `retention_period: 100d` means any log older than 100 days is eligible for immediate deletion by the compactor. Any data between 100–182 days old is written to S3 at full cost, then deleted immediately — wasted write operations and storage cost.

**Fix in `grafana-main-server/loki-config.yaml`:**

```yaml
limits_config:
  reject_old_samples_max_age: 168h  # 7 days — allows reasonable backfill
```

If a specific use case requires backfilling historical logs (e.g., migrating from another system), temporarily extend this value for the migration window only.

---

#### FIX-08: Increase `max_query_parallelism`

**Why:** `max_query_parallelism: 2` artificially caps query throughput. On a t3.medium with 2 vCPUs, even `max_query_parallelism: 8` is safe and will significantly speed up time-range queries spanning multiple days.

**Fix in `grafana-main-server/loki-config.yaml`:**

```yaml
limits_config:
  max_query_parallelism: 8  # Safe on t3.medium; increase to 16 on t3.large
```

---

#### FIX-09: Reduce Compaction Frequency

**Why:** `compaction_interval: 10m` results in 144 compaction cycles per day. Each cycle issues S3 LIST, GET, and PUT API calls. At S3 pricing of $0.005 per 1,000 requests, this generates unnecessary cost and adds I/O load on the ingester.

**Fix in `grafana-main-server/loki-config.yaml`:**

```yaml
compactor:
  compaction_interval: 2h  # Standard for single-node setups
```

---

#### FIX-10: Restrict Alloy UI to Localhost

**Why:** The Alloy container runs with `network_mode: host` and listens on `--server.http.listen-addr=0.0.0.0:12345`. This exposes the Alloy internal UI (pipeline graphs, WAL stats, active targets) to the public internet if the client EC2 security group has port 12345 open, or if the security group is ever modified. The UI has no authentication.

**Fix in `client-server/docker-compose.yml`:**

```yaml
services:
  alloy:
    command:
      - run
      - /etc/alloy/config.alloy
      - --server.http.listen-addr=127.0.0.1:12345  # localhost only
      - --storage.path=/var/lib/alloy/data
```

---

#### FIX-11: Demote `public_ipv4` from Label to Structured Metadata

**Why:** Each unique `{public_ipv4="x.x.x.x"}` value creates a new permanent stream entry in the TSDB index on S3. If Elastic IPs are reassigned, instances are cycled, or Auto Scaling Groups are used, the number of unique label combinations grows unboundedly, degrading index query performance over time (high label cardinality).

**Fix in `client-server/config.alloy`:**

```alloy
loki.process "laravel_pipeline" {
  stage.static_labels {
    values = {
      hostname     = env("HOSTNAME"),
      instance     = env("INSTANCE_NAME"),
      environment  = env("ENVIRONMENT"),
      private_ipv4 = env("PRIVATE_IPV4"),
      // REMOVED: public_ipv4 — moved to structured metadata below
    }
  }

  // ... existing stages ...

  // Attach IPs as structured metadata (Loki 3.x) — no stream cardinality impact
  stage.structured_metadata {
    values = {
      public_ipv4  = env("PUBLIC_IPV4"),
    }
  }
}
```

> **Note:** If you also want to remove `private_ipv4` from labels (since `instance` already uniquely identifies the server), do the same for it. Fewer labels = faster queries.

---

#### FIX-12: Add `loki.write` Retry/Batch/Timeout Configuration

**Why:** The current `loki.write` block has no explicit queue, retry, or timeout settings. If the hub is temporarily unreachable (restart, network blip), Alloy uses default retry behavior, which may not be tuned for a VPC-internal deployment. Explicit configuration makes behavior predictable and ensures log delivery resilience.

**Fix in `client-server/config.alloy`:**

```alloy
loki.write "loki_endpoint" {
  endpoint {
    url              = "http://" + env("HUB_IP") + ":3100/loki/api/v1/push"
    remote_timeout   = "10s"
    batch_wait       = "1s"
    batch_size       = "1MiB"
    min_backoff      = "500ms"
    max_backoff      = "5m"
    max_retries      = 10
  }
}
```

---

#### FIX-13: Fix `HOSTNAME` Variable Conflict

**Why:** `HOSTNAME` is a Linux shell built-in variable. When Docker Compose passes `HOSTNAME` as a container environment variable, the shell on the host may resolve it before Docker sees it, causing the container to receive the host's actual hostname rather than the value set in `.env`.

**Fix in `client-server/docker-compose.yml`:**

Rename the variable in the compose file, `.env.example`, and `config.alloy`:

```yaml
# docker-compose.yml — rename to APP_HOSTNAME
environment:
  - APP_HOSTNAME=${APP_HOSTNAME:?set APP_HOSTNAME in .env}
```

```alloy
# config.alloy — use APP_HOSTNAME
stage.static_labels {
  values = {
    hostname = env("APP_HOSTNAME"),
    ...
  }
}
```

Update `.env.example` to use `APP_HOSTNAME` instead of `HOSTNAME`.

---

#### FIX-14: Add Timestamp Format Robustness

**Why:** The current timestamp stage only handles `2006-01-02 15:04:05` (space separator, no microseconds). Laravel can emit timestamps in multiple formats depending on version and configuration: `2024-01-15T14:30:00` (T separator), `2024-01-15 14:30:00.123456` (with microseconds). A format mismatch causes silent parse failures — the log line is ingested without the parsed timestamp, defaulting to ingestion time instead.

**Fix in `client-server/config.alloy`:**

```alloy
stage.timestamp {
  source   = "timestamp"
  format   = "2006-01-02 15:04:05"
  location = env("TZ_NAME")
  fallback_formats = [
    "2006-01-02T15:04:05",
    "2006-01-02 15:04:05.000000",
    "2006-01-02T15:04:05.000000",
  ]
}
```

---

### P2 — Medium Priority (Fix Within First Two Weeks)

#### FIX-15: Add Multi-Line Parsing for Stack Traces

**Why:** Laravel exception logs emit multi-line stack traces. Each line is currently ingested as a separate log entry, making it difficult to correlate a single exception event in Grafana. This creates noise in error dashboards and makes alert rules on `level=error` fire multiple times per exception.

**Fix in `client-server/config.alloy`:**

```alloy
loki.source.file "log_scrape" {
  targets    = local.file_match.laravel_logs.targets
  forward_to = [loki.process.laravel_pipeline.receiver]
  tail_from_end = true

  // Merge continuation lines into the log entry that started with a timestamp
  decompression {
    enabled = false
  }
}

// Add before the regex stage:
stage.multiline {
  firstline     = "^\\[\\d{4}-\\d{2}-\\d{2}"
  max_wait_time = "3s"
  max_lines     = 128
}
```

---

#### FIX-16: Fix Alloy Healthcheck

**Why:** The healthcheck `grep -q ':3039' /proc/net/tcp /proc/net/tcp6` checks for a hardcoded hex port value (`0x3039 = 12345`). This is brittle — it depends on `/proc/net/tcp` format, which varies between kernel versions, and the hex value is opaque and easy to get wrong.

**Fix in `client-server/docker-compose.yml`:**

```yaml
healthcheck:
  test:
    [
      "CMD-SHELL",
      "wget --no-verbose --tries=1 --spider http://127.0.0.1:12345/-/ready || exit 1",
    ]
  interval: 30s
  timeout: 5s
  retries: 5
  start_period: 30s
```

---

#### FIX-17: Add `start_period` to Both Healthchecks

**Why:** On first boot, Grafana provisioning (loading datasource YAML, running migrations) can take longer than the `retries × interval` window (5 × 15s = 75s). Without `start_period`, Docker may declare the container unhealthy and restart it during normal initialization.

**Fix in `grafana-main-server/docker-compose.yml`:**

```yaml
# Both loki and grafana healthcheck blocks — add:
start_period: 60s
```

---

#### FIX-18: Make Grafana Admin Username Configurable

**Why:** Hardcoded `admin` username is a predictable brute-force target. Industry standard is to use a non-guessable admin account name.

**Fix in `grafana-main-server/docker-compose.yml`:**

```yaml
environment:
  - GF_SECURITY_ADMIN_USER=${GRAFANA_ADMIN_USER:?set GRAFANA_ADMIN_USER in .env}
  - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_ADMIN_PASSWORD:?set GRAFANA_ADMIN_PASSWORD in .env}
```

Add `GRAFANA_ADMIN_USER=CHANGE_THIS_USERNAME` to `.env.example`.

---

#### FIX-19: Add IAM Policy `s3:DeleteObject` Permission

**Why:** The README's IAM policy documentation includes `DeleteObject`, but this must be verified. Loki's compactor uses `DeleteObject` to physically remove expired log chunks from S3 when retention is enforced. Without this permission, retention enforcement silently fails — the compactor marks objects for deletion but S3 rejects the delete calls. Your bucket will grow unboundedly despite the configured 100-day retention.

**IAM policy — verify it includes:**

```json
{
  "Effect": "Allow",
  "Action": [
    "s3:PutObject",
    "s3:GetObject",
    "s3:DeleteObject"
  ],
  "Resource": "arn:aws:s3:::your-bucket-name/*"
},
{
  "Effect": "Allow",
  "Action": [
    "s3:ListBucket",
    "s3:GetBucketLocation"
  ],
  "Resource": "arn:aws:s3:::your-bucket-name"
}
```

---

#### FIX-20: Set Loki Datasource as Non-Editable and Add Query Timeout

**Why:** A provisioned datasource marked `editable: true` (the default) can be modified through the Grafana UI by any editor-role user, breaking the declarative provisioning. A query timeout prevents expensive runaway queries from consuming Loki resources.

**Fix in `grafana-main-server/grafana/provisioning/datasources/datasources.yaml`:**

```yaml
datasources:
  - name: Loki
    type: loki
    uid: loki
    access: proxy
    url: http://loki:3100
    isDefault: true
    editable: false
    jsonData:
      maxLines: 5000
      timeout: 120
```

---

## 5. Recommended Architecture

### 5.1 Instance Sizing — Main Server (Hub)

**Current planned:** t2.small (1 vCPU, 2 GB RAM)

**Assessment: t2.small is insufficient for production.**

**Why:**
- Loki's ingester maintains an in-memory block per active stream. With 20 clients × ~10 streams each (one per log channel + labels), plus WAL buffering and the embedded query cache (configured for 100 MB), peak memory usage easily reaches 1.5–2 GB.
- t2.small has only 2 GB RAM. After OS overhead (~400 MB) and Docker daemon (~200 MB), only ~1.4 GB remains for Loki and Grafana combined. This is too tight — any query spike or compaction burst will trigger OOM kills.
- t2 instances use CPU credits. A sustained log ingestion burst from 20 clients can exhaust credits, throttling CPU to baseline (5–20% for t2.small). Loki compaction and query processing will stall.
- t2 credits do not regenerate fast enough under production load.

**Recommended instance type:** `t3.medium` (2 vCPU, 4 GB RAM)

| Metric | t2.small | t3.medium | Recommendation |
|---|---|---|---|
| vCPU | 1 | 2 | t3.medium |
| RAM | 2 GB | 4 GB | t3.medium |
| CPU Credits | Burstable (limited) | Burstable (better baseline) | t3.medium |
| Network | Low | Up to 5 Gbps | t3.medium |
| On-Demand cost (ap-southeast-1) | ~$0.023/hr | ~$0.052/hr | ~$38/mo |
| Estimated memory headroom | < 200 MB | ~1.5 GB | ✅ Safe |

**Why not t3.large?** Not necessary at 20 clients. Save cost; upgrade to t3.large only if you onboard >40 clients or add Prometheus/Mimir to the hub.

**Memory budget on t3.medium (4 GB):**

| Component | Allocation |
|---|---|
| OS + Docker daemon | 400 MB |
| Loki (ingester + cache + WAL) | 2,048 MB (limit) |
| Grafana | 512 MB (limit) |
| Buffer / OS page cache | ~1,040 MB |
| **Total** | **4,000 MB** |

### 5.2 EBS Volume Sizing — Hub

| Purpose | Volume | Recommended Size | Notes |
|---|---|---|---|
| Root OS + Docker | gp3 root | 20 GB | OS, Docker images, Docker volumes (WAL, index cache) |
| Loki WAL | (part of root) | — | WAL data is inside `loki-data` Docker volume on root |

**Why not a separate EBS for Loki data?** The Loki WAL and index cache are transient — they are flushed to S3 regularly. The named Docker volume `loki-data` does not need to be large (1–2 GB typical). A 20 GB gp3 root is sufficient. Long-term storage is S3.

**gp3 root volume configuration:**

```
Size:        20 GB
Type:        gp3
IOPS:        3,000 (gp3 baseline, no additional cost)
Throughput:  125 MB/s (gp3 baseline, no additional cost)
Encryption:  Enabled (AWS managed key)
```

### 5.3 Instance Sizing — Client Servers

Alloy is a lightweight Go binary. On all client instance types (t2.micro through t3.large), Alloy's resource footprint is minimal:

| Metric | Typical Alloy Usage |
|---|---|
| CPU | < 1% continuous; brief spikes on file scan |
| Memory | 50–150 MB depending on WAL size |
| Network | < 1 Mbps per active log file |
| Disk (WAL) | 50–500 MB (bounded by WAL retention) |

No client instance type changes are required solely for Alloy. The resource limits in FIX-05 (`256M` memory, `0.5` CPU) are appropriate for all client sizes.

### 5.4 S3 Retention and Lifecycle Strategy

| Tier | S3 Storage Class | Age | Cost Impact |
|---|---|---|---|
| Hot (recent) | S3 Standard | 0–30 days | Standard pricing |
| Warm (older) | S3 Intelligent-Tiering | 30–100 days | Automated tiering to cheaper class |
| Expired | — | 100+ days | Deleted by S3 lifecycle rule |
| Noncurrent versions | — | 7+ days old version | Permanently deleted |

**Alignment verification:** Loki `retention_period: 100d` must match the S3 lifecycle expiry at 100 days. If they diverge, either Loki queries data S3 has deleted (gaps) or S3 stores data Loki will never read again (cost waste).

---

## 6. Main Server Deployment Guide

This section provides a complete, step-by-step production deployment procedure for the monitoring hub. This replaces the quick-start README for production use.

### 6.1 Prerequisites

- AWS account with access to EC2, S3, IAM, and Security Groups
- An SSH keypair for the hub EC2
- The hub EC2 is running Ubuntu 22.04 LTS or 24.04 LTS
- You have applied all fixes from Section 4

### 6.2 AWS Infrastructure Setup

#### Step 1: Create the S3 Bucket

```bash
# Replace values with your region and company name
BUCKET_NAME="your-company-observability-data"
REGION="ap-southeast-1"

aws s3api create-bucket \
  --bucket "${BUCKET_NAME}" \
  --region "${REGION}" \
  --create-bucket-configuration LocationConstraint="${REGION}"

# Block all public access
aws s3api put-public-access-block \
  --bucket "${BUCKET_NAME}" \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

# Enable versioning
aws s3api put-bucket-versioning \
  --bucket "${BUCKET_NAME}" \
  --versioning-configuration Status=Enabled

# Enable SSE-S3 default encryption
aws s3api put-bucket-encryption \
  --bucket "${BUCKET_NAME}" \
  --server-side-encryption-configuration '{
    "Rules": [{"ApplyServerSideEncryptionByDefault": {"SSEAlgorithm": "AES256"}}]
  }'
```

Apply the lifecycle policy:

```bash
aws s3api put-bucket-lifecycle-configuration \
  --bucket "${BUCKET_NAME}" \
  --lifecycle-configuration '{
    "Rules": [{
      "ID": "ObservabilityDataLifecycle",
      "Status": "Enabled",
      "Filter": {"Prefix": ""},
      "Transitions": [{"Days": 30, "StorageClass": "INTELLIGENT_TIERING"}],
      "Expiration": {"Days": 100},
      "NoncurrentVersionExpiration": {"NoncurrentDays": 7}
    }]
  }'
```

#### Step 2: Create IAM Policy and Role

```bash
# Create the policy document
cat > /tmp/grafana-s3-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:ListBucket", "s3:GetBucketLocation"],
      "Resource": "arn:aws:s3:::${BUCKET_NAME}"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:PutObject", "s3:GetObject", "s3:DeleteObject"],
      "Resource": "arn:aws:s3:::${BUCKET_NAME}/*"
    }
  ]
}
EOF

# Create the policy
aws iam create-policy \
  --policy-name GrafanaS3Access \
  --policy-document file:///tmp/grafana-s3-policy.json

# Create the role trust policy
cat > /tmp/ec2-trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "ec2.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

aws iam create-role \
  --role-name GrafanaServerRole \
  --assume-role-policy-document file:///tmp/ec2-trust-policy.json

aws iam attach-role-policy \
  --role-name GrafanaServerRole \
  --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/GrafanaS3Access

aws iam create-instance-profile \
  --instance-profile-name GrafanaServerProfile

aws iam add-role-to-instance-profile \
  --instance-profile-name GrafanaServerProfile \
  --role-name GrafanaServerRole
```

#### Step 3: Launch Hub EC2 Instance

```bash
# t3.medium, Ubuntu 22.04 LTS, gp3 20GB root, with IAM profile
aws ec2 run-instances \
  --image-id ami-XXXXXXXX \            # Ubuntu 22.04 LTS in your region
  --instance-type t3.medium \
  --key-name your-keypair-name \
  --security-group-ids sg-XXXXXXXX \  # grafana-main-hub SG
  --iam-instance-profile Name=GrafanaServerProfile \
  --block-device-mappings '[{
    "DeviceName": "/dev/sda1",
    "Ebs": {
      "VolumeType": "gp3",
      "VolumeSize": 20,
      "Encrypted": true,
      "DeleteOnTermination": true
    }
  }]' \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=grafana-monitoring-hub},{Key=Role,Value=observability-hub}]'
```

#### Step 4: Increase IMDSv2 Hop Limit (CRITICAL)

```bash
INSTANCE_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=grafana-monitoring-hub" \
  --query 'Reservations[0].Instances[0].InstanceId' \
  --output text)

aws ec2 modify-instance-metadata-options \
  --instance-id "${INSTANCE_ID}" \
  --http-put-response-hop-limit 2 \
  --http-endpoint enabled \
  --region "${REGION}"
```

#### Step 5: Configure Security Groups

**Hub Security Group (`grafana-main-hub`) — Inbound Rules:**

| Type | Protocol | Port | Source | Purpose |
|---|---|---|---|---|
| SSH | TCP | 22 | Office/VPN CIDR | Admin access |
| HTTP | TCP | 80 | Office/VPN CIDR | Grafana UI |
| Custom TCP | TCP | 3100 | `grafana-client` SG ID | Loki ingest (Alloy) |

**Hub Security Group — Outbound Rules:**

| Type | Protocol | Port | Destination | Purpose |
|---|---|---|---|---|
| HTTPS | TCP | 443 | 0.0.0.0/0 | AWS APIs (S3, IMDS) |
| All traffic | All | All | monitoring SG | Internal Docker (optional) |

> **Critical:** Port 3100 source must be the `grafana-client` Security Group ID, NOT a CIDR. This ensures only EC2 instances tagged as clients can push logs, regardless of IP changes.

### 6.3 Hub EC2 Software Setup

```bash
# SSH into the hub
ssh -i your-keypair.pem ubuntu@HUB_PUBLIC_IP

# Update system
sudo apt update && sudo apt upgrade -y

# Install Docker
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker "$USER"
sudo apt install -y docker-compose-plugin git awscli
sudo systemctl enable docker

# Log out and back in for Docker group
exit
ssh -i your-keypair.pem ubuntu@HUB_PUBLIC_IP

# Verify
docker --version
docker compose version
aws sts get-caller-identity  # Should return GrafanaServerRole
```

### 6.4 Deploy the Stack

```bash
cd /opt
sudo mkdir -p grafana-main-hub
sudo chown "$USER:$USER" grafana-main-hub
git clone https://github.com/yourorg/grafana-main-server.git grafana-main-hub
cd /opt/grafana-main-hub/grafana-main-server

# Create .env from template
cp .env.example .env
chmod 600 .env   # Restrict to owner only
nano .env
```

Set the following in `.env`:

```env
GRAFANA_ADMIN_USER=your-chosen-admin-name
GRAFANA_ADMIN_PASSWORD=a-very-strong-password-min-16-chars
GRAFANA_ROOT_URL=https://your-monitoring-domain.com/
HUB_PRIVATE_IP=10.x.x.x           # Hub EC2 private IP
LOKI_S3_BUCKET=your-actual-bucket-name
```

Update `loki-config.yaml` region and bucket (or use env var injection from FIX-02):

```bash
# Confirm Loki config has your actual bucket name (not the placeholder)
grep "bucketnames" loki-config.yaml
# Must NOT say: your-company-observability-data
```

Start the stack:

```bash
docker compose up -d
docker compose ps
# Wait ~60 seconds for Loki to reach healthy state, then:
docker compose logs --tail=100 loki
docker compose logs --tail=100 grafana
```

### 6.5 Validate Hub

```bash
# Loki ready
curl -s http://127.0.0.1:3100/ready
# Expected: "ready"

# Grafana health
curl -s http://127.0.0.1:80/api/health | python3 -m json.tool

# IAM + S3 access
aws sts get-caller-identity
aws s3 ls s3://your-actual-bucket-name/

# Check Loki can write to S3 (wait 5min, then check for loki/ prefix in bucket)
aws s3 ls s3://your-actual-bucket-name/loki/ --region ap-southeast-1

# Check no errors in Loki
docker compose logs loki | grep -i "error\|failed\|credential\|s3"
```

---

## 7. Client Server Deployment Guide

Run this procedure on each of the ~20 application EC2 instances that should ship logs to the hub.

### 7.1 Prerequisites Per Client

- Docker and Docker Compose installed
- Hub EC2 is running and healthy
- Client EC2 is in the `grafana-client` Security Group
- Laravel `storage/logs/` directory exists and is writable by the app

### 7.2 Verify Hub Connectivity

```bash
# Before installing anything — verify network path
nc -vz HUB_PRIVATE_IP 3100
# Expected: Connection to HUB_PRIVATE_IP 3100 port [tcp/*] succeeded!

# If nc is not available:
sudo apt install -y netcat-openbsd
# Or use:
curl --connect-timeout 5 http://HUB_PRIVATE_IP:3100/ready
```

If connectivity fails: check Security Groups — the client must be in `grafana-client` SG, and the hub's `grafana-main-hub` SG must allow TCP 3100 from `grafana-client` SG.

### 7.3 Install Docker (if not already present)

```bash
sudo apt update && sudo apt upgrade -y
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker "$USER"
sudo apt install -y docker-compose-plugin
sudo systemctl enable docker
exit  # Re-login for Docker group
```

### 7.4 Deploy the Alloy Agent

```bash
cd /opt
sudo mkdir -p grafana-nodes
sudo chown "$USER:$USER" grafana-nodes
git clone https://github.com/yourorg/grafana-client-docker.git grafana-nodes
cd /opt/grafana-nodes

cp .env.example .env
chmod 600 .env
nano .env
```

Set all values in `.env`:

```env
APP_HOSTNAME=laravel-app-01               # Unique per server — use descriptive name
INSTANCE_NAME=laravel-app-01             # Same as APP_HOSTNAME (or more specific)
ENVIRONMENT=production                    # staging or production
TZ_NAME=Asia/Kuala_Lumpur               # Must match server timezone exactly
LARAVEL_LOG_DIR=/var/www/myapp/storage/logs  # Full path to storage/logs/
PRIVATE_IPV4=10.0.x.x                   # This server's private IP
PUBLIC_IPV4=x.x.x.x                     # This server's public IP (structured metadata only)
HUB_IP=10.0.1.50                        # Hub EC2 private IP
```

Verify the log directory:

```bash
ls -lah "${LARAVEL_LOG_DIR}"
# Should show .log files — e.g., laravel-2026-06-18.log
```

Start the agent:

```bash
docker compose up -d
sleep 10
docker compose ps
docker compose logs --tail=80 alloy
```

### 7.5 Validate Client

```bash
# Alloy is running and listening on localhost:12345
curl -s http://127.0.0.1:12345/ | head -5

# Hub is still reachable
nc -vz "${HUB_IP}" 3100

# Alloy is processing logs — look for "level=info" entries about file targets
docker compose logs --tail=50 alloy | grep -i "target\|scrape\|push"
```

### 7.6 Verify Logs Arrive in Grafana

1. Open Grafana: `http://HUB_PRIVATE_IP/` (or your domain)
2. Navigate to **Explore** → select **Loki** datasource
3. Use the following queries:

```logql
# All logs from this instance
{instance="laravel-app-01"}

# Only error logs across all clients
{job="laravel", environment="production"} |= "ERROR"

# Error log count per instance in last 1h
sum by (instance) (count_over_time({job="laravel", level="error"}[1h]))
```

Logs should appear within 1–2 minutes of Alloy starting.

### 7.7 Rolling Deployment Across 20 Clients

Deploy clients in batches to avoid a simultaneous ingest spike:

```bash
# Batch 1: 5 clients
# Batch 2: 5 clients (10 minutes later)
# Batch 3: 5 clients (10 minutes later)
# Batch 4: 5 clients (10 minutes later)

# Monitor Loki ingest rate between batches:
# Grafana → Explore → Loki → {job="loki"} — check ingestion rate metrics
```

---

## 8. Security Hardening Checklist

### 8.1 Network Security

- [ ] **Hub Security Group:** Port 3100 source is `grafana-client` SG ID (not a CIDR)
- [ ] **Hub Security Group:** Port 80 restricted to office/VPN CIDR only
- [ ] **Hub Security Group:** Port 22 restricted to office/VPN CIDR only
- [ ] **Hub Security Group:** No rule allows port 3100 from `0.0.0.0/0`
- [ ] **Client Security Groups:** Port 12345 is NOT open in any inbound rule
- [ ] **Client Security Groups:** Outbound to hub private IP on TCP 3100 is allowed
- [ ] **VPC Flow Logs:** Enabled on the hub and client VPCs for audit trail

### 8.2 TLS / HTTPS

For production, Grafana must be served over HTTPS. Two recommended approaches:

**Option A — AWS Application Load Balancer (recommended):**

```
Client Browser → ALB (HTTPS :443, ACM certificate) → Hub EC2 (HTTP :80, internal only)
```

- Create an ALB in front of the hub EC2
- Attach an ACM (AWS Certificate Manager) certificate for your domain
- ALB listener: HTTPS:443 → forward to target group on port 80
- Hub Security Group: restrict port 80 to the ALB Security Group only
- Update `GRAFANA_ROOT_URL=https://your-domain.com/` in `.env`

**Option B — Nginx reverse proxy on hub EC2:**

```bash
sudo apt install -y nginx certbot python3-certbot-nginx
sudo certbot --nginx -d your-monitoring-domain.com
```

Nginx config (`/etc/nginx/sites-available/grafana`):

```nginx
server {
    listen 443 ssl;
    server_name your-monitoring-domain.com;

    ssl_certificate /etc/letsencrypt/live/your-monitoring-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your-monitoring-domain.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:3000;  # Grafana internal port
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}

server {
    listen 80;
    server_name your-monitoring-domain.com;
    return 301 https://$host$request_uri;
}
```

### 8.3 Grafana Hardening

- [ ] Change admin username from `admin` to a non-guessable name (FIX-18)
- [ ] Set a strong admin password (minimum 16 characters, mixed case, numbers, symbols)
- [ ] Disable `GF_USERS_ALLOW_SIGN_UP=false` (already done — verify in compose file)
- [ ] Rotate the admin password after first login
- [ ] Consider adding SSO via `GF_AUTH_GENERIC_OAUTH` or `GF_AUTH_SAML` for team access
- [ ] Set session cookie to secure: `GF_SESSION_COOKIE_SECURE=true` (requires HTTPS)
- [ ] Provision dashboards declaratively — never grant team members Admin role

### 8.4 Loki Hardening

- [ ] Bind port 3100 to private IP only (FIX-03)
- [ ] Do not enable `auth_enabled: true` unless you have multiple tenants — the SG is sufficient control for single-tenant internal use
- [ ] Loki container runs as non-root (verify: `docker compose exec loki id`)
- [ ] `loki-config.yaml` mounted as `:ro` (already done — verified in compose file)
- [ ] No `access_key_id` or `secret_access_key` in any config file — IAM role only (already done)

### 8.5 IAM Hardening

- [ ] `GrafanaServerRole` has only the `GrafanaS3Access` policy — no `AdministratorAccess`
- [ ] `GrafanaS3Access` policy is scoped to the exact bucket ARN (not `arn:aws:s3:::*`)
- [ ] Client EC2 instances do not have `GrafanaServerRole` or any S3 permissions
- [ ] S3 bucket has Block Public Access fully enabled
- [ ] S3 bucket policy does not grant public read or write

### 8.6 Secret Management

- [ ] `.gitignore` exists and excludes all `.env` files (FIX-04)
- [ ] `.env` files have permission `600` (owner read only): `chmod 600 .env`
- [ ] `.env` is not committed to git: `git status | grep .env` should return nothing
- [ ] Verify with: `git log --all --full-history -- "*.env"` — should return nothing
- [ ] For team environments: migrate secrets to AWS Secrets Manager or Parameter Store

---

## 9. Monitoring & Alerting Setup

### 9.1 Self-Monitoring Strategy

The current setup has no self-monitoring — Loki and Alloy failures are invisible until someone manually checks container logs or Grafana goes dark. This is the highest operational risk after the critical blockers.

**Minimal self-monitoring (implement before go-live):**

#### Step 1: Add Prometheus Self-Scrape for Loki

Add to `grafana-main-server/docker-compose.yml`:

```yaml
services:
  # ... existing services ...

  prometheus:
    image: prom/prometheus:v3.4.0
    container_name: prometheus
    restart: unless-stopped
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.retention.time=15d"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus
    ports:
      - "127.0.0.1:9090:9090"   # localhost only — no public exposure
    networks:
      - monitoring
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 512M

volumes:
  # ... existing volumes ...
  prometheus-data:
```

Create `grafana-main-server/prometheus/prometheus.yml`:

```yaml
global:
  scrape_interval: 30s
  evaluation_interval: 30s

scrape_configs:
  - job_name: loki
    static_configs:
      - targets: ["loki:3100"]
    metrics_path: /metrics

  - job_name: prometheus
    static_configs:
      - targets: ["localhost:9090"]
```

#### Step 2: Add Prometheus Datasource to Grafana

Add to `grafana-main-server/grafana/provisioning/datasources/datasources.yaml`:

```yaml
  - name: Prometheus
    type: prometheus
    uid: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: false
    editable: false
```

#### Step 3: Add Alloy Self-Metrics to `config.alloy`

```alloy
// Self-monitoring: scrape Alloy's own metrics
prometheus.scrape "alloy_self" {
  targets = [{"__address__" = "127.0.0.1:12345"}]
  forward_to = [prometheus.remote_write.hub.receiver]
  scrape_interval = "30s"
}

prometheus.remote_write "hub" {
  endpoint {
    url = "http://" + env("HUB_IP") + ":9090/api/v1/write"
  }
}
```

> **Note:** This requires enabling Prometheus remote write on the hub's Prometheus instance. Alternatively, expose Alloy metrics as a scrape target on the hub's Prometheus config.

### 9.2 Key Metrics to Monitor

#### Loki Health Metrics

| Metric | Alert Threshold | Meaning |
|---|---|---|
| `loki_ingester_blocks_per_chunk_summary` | — | Monitor chunk sizes |
| `loki_request_duration_seconds{route="/loki/api/v1/push"}` | p99 > 5s | Ingestion latency |
| `loki_distributor_bytes_received_total` | Rate drop to 0 | Alloy stopped pushing |
| `loki_compactor_runs_total` | No increment for 4h | Compactor stuck |
| `process_resident_memory_bytes{job="loki"}` | > 1.8 GB | Memory pressure |
| `loki_ruler_evaluations_failed_total` | Any increment | Alert evaluation failing |

#### Alloy Health Metrics (per client)

| Metric | Alert Threshold | Meaning |
|---|---|---|
| `loki_write_sent_entries_total` | Rate drop to 0 | Log shipping stopped |
| `loki_write_failed_entries_total` | Any increment | Failed deliveries |
| `component_controller_running_components` | < expected | Component crash |

### 9.3 Grafana Alert Rules

Create alert rules in Grafana (or provision via YAML) for:

#### Alert 1: Loki Ingest Rate Drop

```yaml
# Grafana alert rule (LogQL)
expr: sum(rate({job="loki"}[5m])) == 0
for: 5m
severity: critical
message: "No logs received by Loki for 5 minutes — check all Alloy agents"
```

#### Alert 2: High Loki Error Rate

```yaml
expr: rate({job="loki", level="error"}[5m]) > 10
for: 2m
severity: warning
message: "Loki is emitting errors at high rate"
```

#### Alert 3: Instance Not Reporting

```logql
# Alert if a specific instance hasn't logged in 15 minutes
absent_over_time({instance="laravel-app-01"}[15m])
```

### 9.4 Notification Channels

Configure at least one notification channel in Grafana before go-live:

```bash
# Grafana → Alerting → Contact Points → New contact point
# Recommended: Slack webhook or PagerDuty for critical alerts
# Email for warning-level alerts
```

Provision via YAML (`grafana/provisioning/alerting/contact-points.yaml`):

```yaml
apiVersion: 1
contactPoints:
  - name: slack-ops
    receivers:
      - uid: slack-ops-receiver
        type: slack
        settings:
          url: ${SLACK_WEBHOOK_URL}
          channel: "#ops-alerts"
```

---

## 10. Backup & Disaster Recovery

### 10.1 What Needs to Be Backed Up

| Data | Location | Backup Strategy | RPO |
|---|---|---|---|
| Log data (chunks) | S3 | S3 versioning + cross-region replication | S3 SLA |
| Loki TSDB index | S3 | Same as above | S3 SLA |
| Loki WAL (recent logs) | Docker volume `loki-data` | EC2 snapshot or EBS snapshot | ~1 hour |
| Grafana dashboards | Docker volume `grafana-data` | Dashboard export + provisioning YAML | Manual |
| Grafana config | Docker volume `grafana-data` | Export and commit to git | Immediate |
| `.env` (secrets) | EC2 filesystem | AWS Secrets Manager | Manual |
| Alloy WAL (recent) | Docker volume `alloy-data` | Survives container restart, not host failure | ~1 hour |

### 10.2 S3 Backup (Primary Log Storage)

Enable S3 Cross-Region Replication for disaster resilience:

```bash
# Create destination bucket in a second region
aws s3api create-bucket \
  --bucket "${BUCKET_NAME}-backup" \
  --region ap-southeast-2 \
  --create-bucket-configuration LocationConstraint=ap-southeast-2

# Enable replication on the primary bucket
aws s3api put-bucket-replication \
  --bucket "${BUCKET_NAME}" \
  --replication-configuration '{
    "Role": "arn:aws:iam::ACCOUNT_ID:role/S3ReplicationRole",
    "Rules": [{
      "Status": "Enabled",
      "Filter": {"Prefix": ""},
      "Destination": {
        "Bucket": "arn:aws:s3:::'"${BUCKET_NAME}"'-backup",
        "StorageClass": "STANDARD_IA"
      }
    }]
  }'
```

> **Cost consideration:** CRR approximately doubles S3 storage costs. For a pure observability store (logs are ephemeral), this may be optional — assess based on compliance requirements. S3 versioning alone provides protection against accidental deletion.

### 10.3 Grafana Dashboard Backup

Never rely solely on the Grafana database for dashboard storage. The `grafana-data` volume will be lost if the EC2 is terminated.

**Option A — Provision dashboards as code (recommended):**

Place dashboard JSON files in `grafana-main-server/grafana/provisioning/dashboards/` and mount them. Dashboard changes are committed to git automatically.

```yaml
# datasources.yaml — add dashboard provisioning
# grafana/provisioning/dashboards/dashboards.yaml
apiVersion: 1
providers:
  - name: default
    folder: Observability
    type: file
    options:
      path: /etc/grafana/provisioning/dashboards
```

**Option B — Periodic dashboard export:**

```bash
#!/bin/bash
# Run on hub EC2 — export all dashboards to git
GRAFANA_URL="http://localhost:80"
GRAFANA_AUTH="admin:your-password"
OUTPUT_DIR="/opt/grafana-backups/dashboards"

mkdir -p "${OUTPUT_DIR}"
DASHBOARDS=$(curl -s -u "${GRAFANA_AUTH}" "${GRAFANA_URL}/api/search?type=dash-db" | jq -r '.[].uid')

for uid in $DASHBOARDS; do
  curl -s -u "${GRAFANA_AUTH}" "${GRAFANA_URL}/api/dashboards/uid/${uid}" \
    | jq '.dashboard' > "${OUTPUT_DIR}/${uid}.json"
done

git -C /opt/grafana-backups add . && git -C /opt/grafana-backups commit -m "Dashboard backup $(date -I)"
```

### 10.4 EC2 Snapshot Schedule

```bash
# Create a snapshot policy for the hub EC2 root volume
aws dlm create-lifecycle-policy \
  --description "Grafana hub daily snapshots" \
  --state ENABLED \
  --execution-role-arn arn:aws:iam::ACCOUNT_ID:role/AWSDataLifecycleManagerDefaultRole \
  --policy-details '{
    "PolicyType": "EBS_SNAPSHOT_MANAGEMENT",
    "ResourceTypes": ["INSTANCE"],
    "TargetTags": [{"Key": "Role", "Value": "observability-hub"}],
    "Schedules": [{
      "Name": "daily",
      "CreateRule": {"Interval": 24, "IntervalUnit": "HOURS", "Times": ["03:00"]},
      "RetainRule": {"Count": 7},
      "CopyTags": true
    }]
  }'
```

This creates daily snapshots of the hub EC2 root volume and retains the last 7.

### 10.5 Disaster Recovery Procedure

**Scenario: Hub EC2 is terminated or corrupted**

Recovery time objective (RTO): ~30 minutes

1. Launch a new t3.medium EC2 with the same IAM role (`GrafanaServerProfile`)
2. Apply IMDSv2 hop limit increase (FIX-01)
3. Attach it to `grafana-main-hub` Security Group
4. SSH in, install Docker, clone the repo
5. Restore `.env` from AWS Secrets Manager (if stored there) or recreate from documentation
6. `docker compose up -d`
7. Loki will re-read the TSDB index from S3 and all historical logs are immediately available
8. The only data loss is logs in the WAL (last ~1–2 hours) that had not been flushed to S3

**Scenario: Alloy agent on a client crashes**

Recovery time objective: Automatic (Docker `restart: unless-stopped`)

1. Docker will automatically restart the Alloy container
2. The WAL volume (`alloy-data`) persists — undelivered logs are replayed on restart
3. No manual intervention required unless the WAL volume is also lost

**Scenario: S3 bucket accidentally deleted**

1. S3 versioning means objects are not permanently deleted immediately
2. Restore from noncurrent versions via `aws s3api list-object-versions` + `restore`
3. If permanently deleted, restore from cross-region replication bucket (if enabled)
4. Loki will need `--from scratch` re-indexing if TSDB index is lost — contact Grafana support

---

## 11. Scaling Strategy

### 11.1 Current Capacity (20 Clients)

At 20 Laravel application clients with typical log volumes (~10 log lines/second per client), the estimated aggregate ingest rate is:

| Metric | Estimate |
|---|---|
| Total log lines/sec | ~200 lps |
| Average log line size | ~250 bytes |
| Aggregate throughput | ~50 KB/s (~4.3 GB/day) |
| S3 storage at 100-day retention | ~430 GB |

At S3 Intelligent-Tiering pricing for `ap-southeast-1`, 430 GB costs approximately **$10–12/month** in storage fees.

The single-binary Loki on t3.medium is well within capacity for this load. The `ingestion_rate_mb: 10` limit (currently global due to no auth) equates to 10 MB/s — 200× the estimated ingest rate.

### 11.2 Scaling to 40–60 Clients

No architectural changes are required up to approximately 40–50 clients, provided:

1. The hub is upgraded from t3.medium to t3.large (2 vCPU → 2 vCPU, 4 GB → 8 GB RAM)
2. `max_query_parallelism` is increased to 16
3. Loki resource limits are updated (Loki: 4 GB memory limit)

**Upgrade procedure:**

```bash
# Stop the stack
cd /opt/grafana-main-hub/grafana-main-server
docker compose down

# Resize EC2 (requires stop/start)
# AWS Console → EC2 → Instance → Instance Type → Change to t3.large

# Start the stack on the new instance type
docker compose up -d
```

Since log data is on S3, no data migration is required. The upgrade is pure infra.

### 11.3 Scaling Beyond 60 Clients

At 60+ clients or high-volume applications (CI/CD logs, high-traffic APIs), consider:

**Option A — Loki Read/Write Split Mode:**

Separate Loki into write-path (ingesters) and read-path (queriers) on different instances. Requires migrating to a Loki ring coordinator.

**Option B — Separate Alloy Aggregation Tier:**

Deploy a Loki instance per cluster of 20 clients, then use Grafana's multi-datasource query capability to query across clusters.

**Option C — Grafana Cloud:**

For >100 clients, Grafana Cloud's Loki managed service eliminates infrastructure management at a predictable per-GB cost.

### 11.4 Auto Scaling Group Clients

If any client EC2s are in Auto Scaling Groups, ensure:

1. `INSTANCE_NAME` is set to a meaningful name (e.g., ASG logical name + instance ID) — not just the hostname
2. `PUBLIC_IPV4` is in structured metadata (not a label) — already recommended in FIX-11
3. The ASG launch template includes the Docker + Alloy setup in the user data script
4. The ASG security group is `grafana-client`

---

## 12. Cost Optimization Recommendations

### 12.1 Current Estimated Monthly Cost

| Component | Instance | Est. Monthly Cost (ap-southeast-1) |
|---|---|---|
| Hub EC2 (t3.medium) | On-Demand | ~$38 |
| Hub EBS (20 GB gp3) | — | ~$2 |
| S3 Standard (first 30 days, ~130 GB) | — | ~$3 |
| S3 Intelligent-Tiering (30-100 days, ~300 GB) | — | ~$7 |
| S3 API requests (PUT, GET, LIST) | — | ~$2 |
| Data transfer (EC2 → S3, intra-region) | Free | $0 |
| Data transfer (clients → hub, same VPC) | Free | $0 |
| **Total** | | **~$52/month** |

### 12.2 Cost Optimization Actions

#### Action 1: Use Reserved Instance for Hub EC2

If this stack is expected to run for 12+ months:

- 1-year Reserved Instance (t3.medium, no upfront, ap-southeast-1): ~$28/month (~26% saving)
- 1-year Reserved Instance (t3.medium, full upfront): ~$24/month (~37% saving)

**Recommendation:** Purchase a 1-year No Upfront Reserved Instance after the first month of stable production operation.

#### Action 2: Fix Compaction Interval (FIX-09)

Changing `compaction_interval` from 10m to 2h reduces S3 API calls from ~144/day to ~12/day. At $0.005 per 1,000 requests, the saving is minor in dollar terms but reduces unnecessary I/O.

#### Action 3: Tune S3 Intelligent-Tiering

Intelligent-Tiering has a $0.0025/1,000 object monitoring fee. For a high-object-count Loki store (many small chunk files), this can add up. Consider using standard S3 Standard-IA (Infrequent Access) instead for the warm tier if your access patterns are predictable.

**Modify lifecycle rule:** Replace `INTELLIGENT_TIERING` with `STANDARD_IA` for simpler cost modeling.

#### Action 4: Right-Size Retention

At 4.3 GB/day ingest, 100 days = 430 GB. If your debugging and audit needs are met by 30-day retention, reduce to 30 days:

- Storage cost drops from ~$10/month to ~$3/month
- S3 lifecycle: change expiry to 30 days
- Loki config: `retention_period: 30d`
- `reject_old_samples_max_age: 48h` (2 days is sufficient backfill)

#### Action 5: Use gp3 Instead of gp2 for EBS

If the hub EC2 was launched with a gp2 root volume, migrate to gp3:

```bash
# Modify EBS volume type (no downtime)
aws ec2 modify-volume \
  --volume-id vol-XXXXXXXX \
  --volume-type gp3 \
  --iops 3000 \
  --throughput 125
```

gp3 at 20 GB costs ~$1.60/month vs gp2 at ~$2.00/month (20% saving, free IOPS upgrade).

---

## 13. Operational Runbook

### 13.1 Daily Checks

```bash
# On hub EC2
cd /opt/grafana-main-hub/grafana-main-server

# Container status
docker compose ps

# Recent errors
docker compose logs --since 1h loki | grep -iE "error|warn|failed" | tail -20
docker compose logs --since 1h grafana | grep -iE "error|warn|failed" | tail -20

# S3 connectivity
aws sts get-caller-identity
aws s3 ls s3://your-bucket-name/loki/ | tail -5

# Disk usage (WAL should not grow unboundedly)
df -h /
docker system df
```

### 13.2 Weekly Checks

```bash
# Check S3 storage growth
aws s3 ls s3://your-bucket-name --recursive --summarize | tail -2
# Compare with previous week — should grow by ~4.3 GB/day × 7 = ~30 GB

# Check Loki chunk count
aws s3 ls s3://your-bucket-name/loki/chunks/ --recursive --summarize | tail -2

# Review Grafana logs for auth failures (possible unauthorized access attempts)
docker compose logs --since 7d grafana | grep -i "login\|auth\|failed" | wc -l
```

### 13.3 Monthly Checks

- Review S3 cost in AWS Cost Explorer — compare against estimate
- Review EC2 CPU credit balance (t3 instances) — if credits are consistently at zero, consider upgrading to t3.large
- Pull updated Docker images and test in staging before production:
  ```bash
  docker compose pull
  docker compose up -d  # Rolls over to new images with zero downtime (Compose v2)
  ```
- Rotate Grafana admin password
- Verify `.gitignore` is in place: `git status | grep .env`
- Review Security Group rules for unauthorized changes

### 13.4 Upgrade Procedure

```bash
# 1. Notify the team (dashboards will be unavailable for ~60s)
# 2. On hub EC2:
cd /opt/grafana-main-hub/grafana-main-server

# Pull new images
docker compose pull

# Review changes (compare image digests)
docker compose images

# Apply update (Compose restarts each service with new image)
docker compose up -d

# Verify health
sleep 30
docker compose ps
curl -s http://127.0.0.1:3100/ready
curl -s http://127.0.0.1:80/api/health
```

Client Alloy upgrade (run per client or via automation):

```bash
cd /opt/grafana-nodes
docker compose pull
docker compose up -d
docker compose logs --tail=20 alloy
```

### 13.5 Log Rotation and Cleanup

Alloy only reads and ships logs — it does not manage log file rotation. Laravel log rotation is managed by:

- Laravel's built-in `daily` channel (creates new file per day)
- `logrotate` on the OS (should be configured separately)

Verify `logrotate` is configured for Laravel logs on client servers:

```bash
cat /etc/logrotate.d/laravel  # Should exist per application
# If not, create it:
sudo tee /etc/logrotate.d/laravel << EOF
/var/www/myapp/storage/logs/*.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    create 0664 www-data www-data
    sharedscripts
}
EOF
```

> **Important:** Alloy's WAL ensures that when a log file is rotated (renamed/deleted), any unshipped lines are delivered from the WAL before the handle is dropped. No logs should be lost during rotation.

### 13.6 Troubleshooting Guide

**Problem: Loki returns "context deadline exceeded" on queries**

Cause: Query spans too large a time range, `max_query_parallelism` too low, or large number of streams.

```bash
# Check active query count
curl -s http://127.0.0.1:3100/metrics | grep loki_query
# Increase max_query_parallelism in loki-config.yaml (FIX-08)
# Narrow query time range in Grafana
```

**Problem: Alloy not shipping logs**

```bash
# On client
docker compose logs -f alloy

# Common causes:
# 1. HUB_IP wrong — nc -vz HUB_IP 3100
# 2. LARAVEL_LOG_DIR wrong — ls -lah /host/logs/laravel/
# 3. Log files have no write activity — ls -lah LARAVEL_LOG_DIR --sort=time
# 4. WAL full — docker volume inspect alloy_alloy-data
```

**Problem: Grafana dashboard shows "No data"**

```bash
# Check Loki query directly
curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
  --data-urlencode 'query={job="laravel"}' \
  --data-urlencode "start=$(date -d '1 hour ago' +%s)000000000" \
  --data-urlencode "end=$(date +%s)000000000" \
  | python3 -m json.tool | head -30
# If empty, check Alloy is actually shipping
```

**Problem: S3 access denied errors in Loki**

```bash
# Verify IAM role
aws sts get-caller-identity  # Must return GrafanaServerRole

# Check hop limit
aws ec2 describe-instance-metadata-options --instance-id i-XXXXXXXX
# http-put-response-hop-limit must be >= 2

# Test S3 from inside Loki container
docker exec -it loki wget -qO- \
  "http://169.254.169.254/latest/meta-data/iam/security-credentials/"
```

**Problem: High memory usage on hub**

```bash
# Check Docker container memory
docker stats --no-stream

# If Loki is above its limit, check active streams
curl -s http://127.0.0.1:3100/metrics | grep loki_ingester_streams_created_total

# If Grafana is high, a dashboard query is running hot — check active queries
docker compose logs grafana | grep -i "slow\|timeout\|query"
```

---

## 14. Validation Checklist

Run through this checklist after completing deployment and before opening to the team.

### 14.1 Infrastructure Validation

- [ ] Hub EC2 instance type is t3.medium or larger
- [ ] Hub EC2 root volume is 20 GB gp3, encrypted
- [ ] Hub EC2 has `GrafanaServerProfile` (IAM instance profile) attached
- [ ] `aws sts get-caller-identity` from hub returns `GrafanaServerRole`
- [ ] IMDSv2 hop limit is `2`: `aws ec2 describe-instance-metadata-options --instance-id i-XXX`
- [ ] S3 bucket exists and has Block Public Access enabled
- [ ] S3 lifecycle rule expires at 100 days (or your chosen retention)
- [ ] S3 versioning is enabled
- [ ] Hub Security Group: port 3100 source is `grafana-client` SG only
- [ ] Hub Security Group: port 80 source is office/VPN CIDR only

### 14.2 Hub Stack Validation

- [ ] `docker compose ps` shows `loki` and `grafana` both `healthy`
- [ ] `curl -s http://127.0.0.1:3100/ready` returns `ready`
- [ ] `curl -s http://127.0.0.1:80/api/health` returns HTTP 200
- [ ] Loki logs show no S3 errors: `docker compose logs loki | grep -i "s3\|error" | wc -l`
- [ ] S3 bucket has `loki/` prefix with recent objects: `aws s3 ls s3://bucket/loki/`
- [ ] `loki-config.yaml` has the real bucket name (not the placeholder)
- [ ] `.env` has a strong, non-default `GRAFANA_ADMIN_PASSWORD`
- [ ] `.env` has a non-default `GRAFANA_ADMIN_USER` (not `admin`)
- [ ] `.env` is not tracked in git: `git status`

### 14.3 Security Validation

- [ ] Port 3100 is NOT reachable from the public internet: `curl --connect-timeout 3 http://HUB_PUBLIC_IP:3100/ready` should time out
- [ ] Grafana login page appears only over expected channel (HTTP restricted to VPN / HTTPS via ALB)
- [ ] Alloy UI is NOT reachable from outside the client EC2: `curl --connect-timeout 3 http://CLIENT_PUBLIC_IP:12345/` should time out
- [ ] `.gitignore` exists: `cat .gitignore | grep ".env"`
- [ ] No `.env` files in git history: `git log --all -- "**/.env"`

### 14.4 Client Agent Validation

- [ ] `docker compose ps` on each client shows `alloy` as `healthy`
- [ ] `docker compose logs alloy | grep -i "error\|failed"` returns no active errors
- [ ] Logs appear in Grafana within 2 minutes of Alloy starting
- [ ] `{instance="client-name"}` query returns results in Grafana Explore
- [ ] Log levels (`level=error`, `level=info`) are correctly parsed
- [ ] Log channel (`channel=laravel`, `channel=worker`) is correctly extracted

---

## 15. Go-Live Checklist

This is the final gate before directing production traffic to the stack. All P0 and P1 items must be ✅ before go-live.

### P0 — Hard Blockers (Must Be ✅)

- [ ] **FIX-01 applied:** IMDSv2 hop limit = 2 on hub EC2
- [ ] **FIX-02 applied:** Real S3 bucket name in `loki-config.yaml` (no placeholder)
- [ ] **FIX-03 applied:** Loki port 3100 bound to private IP or SG-restricted
- [ ] **FIX-04 applied:** `.gitignore` created, `.env` files excluded
- [ ] **S3 write verified:** `aws s3 ls s3://bucket/loki/` shows recent objects
- [ ] **Loki healthy:** `curl http://127.0.0.1:3100/ready` returns `ready`
- [ ] **Grafana reachable:** Login works with configured admin credentials

### P1 — Required Before Production Traffic

- [ ] **FIX-05 applied:** Resource limits on all containers
- [ ] **FIX-06 applied:** Ingester WAL explicitly configured
- [ ] **FIX-07 applied:** `reject_old_samples_max_age: 168h`
- [ ] **FIX-08 applied:** `max_query_parallelism: 8`
- [ ] **FIX-09 applied:** `compaction_interval: 2h`
- [ ] **FIX-10 applied:** Alloy UI on `127.0.0.1:12345`
- [ ] **FIX-11 applied:** `public_ipv4` in structured metadata, not label
- [ ] **FIX-12 applied:** `loki.write` retry/batch/timeout configured
- [ ] **FIX-13 applied:** `HOSTNAME` renamed to `APP_HOSTNAME`
- [ ] **FIX-14 applied:** Timestamp fallback formats configured
- [ ] **FIX-19 verified:** IAM policy includes `s3:DeleteObject`
- [ ] **FIX-20 applied:** Datasource `editable: false`, timeout set, `maxLines: 5000`
- [ ] At least one Grafana alert rule is active (ingest rate drop)
- [ ] At least one notification channel is configured and tested
- [ ] Dashboard provisioning is committed to git (not only in Grafana DB)
- [ ] All 20 client agents are deployed, healthy, and shipping logs
- [ ] Grafana admin password has been rotated from initial deployment value

### Communication

- [ ] Team has been notified of Grafana URL and login credentials
- [ ] On-call engineer knows the hub EC2 SSH details
- [ ] Operational runbook (Section 13) has been reviewed by at least one team member
- [ ] Escalation path for S3/IAM issues is documented

---

## 16. Post-Deployment Verification

Run these checks in the first 24–48 hours after go-live.

### Hour 1

```bash
# Verify all clients are shipping (should see ~20 unique instances)
# In Grafana Explore:
# count by (instance) (count_over_time({job="laravel"}[5m]))
# Should return one row per client

# Monitor Loki memory usage
watch -n 30 'docker stats --no-stream loki grafana'
# Loki should stay well below 2048M memory limit

# Check S3 object count is growing
aws s3 ls s3://your-bucket/loki/ --recursive --summarize | tail -2
```

### Hour 6

```bash
# Verify retention is working (check compactor logs)
docker compose logs loki | grep -i "compact\|retention\|delete" | tail -20

# Run a multi-hour query and verify response time is acceptable
# In Grafana: {job="laravel", level="error"} over last 6h
# Should return in < 10 seconds

# Check S3 request volume in AWS Cost Explorer
# S3 → Management → Storage Lens → Activity → Request count
```

### Day 2

```bash
# Verify no memory growth (Loki should be stable)
docker stats --no-stream loki | awk '{print $4}'  # Memory usage

# Check that log volume is as expected
# In Grafana: sum(rate({job="laravel"}[5m])) by (instance)
# Each client should show a non-zero rate during business hours

# Confirm TSDB index is in S3
aws s3 ls s3://your-bucket/loki/index/ --recursive | wc -l
# Should show index files accumulating
```

### Week 1

- Review Grafana query response times — if p99 > 10s, increase `max_query_parallelism`
- Review S3 cost in Cost Explorer — verify it tracks the estimate (~$0.30/day)
- Review EC2 CPU credit balance — if below 50% of max, plan instance upgrade
- Run a simulated Loki restart and verify recovery:
  ```bash
  docker compose restart loki
  # After 30s: curl http://127.0.0.1:3100/ready
  # Verify logs still appear in Grafana (WAL replay should backfill any gap)
  ```

---

## 17. Future Improvements

These items are not required for production launch but represent the next iteration of the stack.

### 17.1 HTTPS for Grafana (High Priority)

**Why:** Grafana credentials and session tokens travel over HTTP in the current setup. Even restricted to VPN, this is a risk if the VPN connection itself is not encrypted or if team members access from untrusted networks.

**Implementation:** Deploy an ALB with an ACM certificate, or add Nginx + Let's Encrypt to the hub.

**Effort:** 2–4 hours.

### 17.2 Grafana Authentication via SSO (Medium Priority)

**Why:** Individual team members should have named accounts rather than sharing the admin credential.

**Implementation options:**
- `GF_AUTH_GENERIC_OAUTH` with your identity provider (Google, GitHub, Azure AD)
- `GF_AUTH_SAML` for enterprise IdP integration
- Grafana's built-in team management (create editor accounts, remove shared admin use)

**Effort:** 4–8 hours.

### 17.3 Provisioned Dashboards for Log Observability (Medium Priority)

The `grafana/provisioning/` directory currently only contains datasources. Add standard dashboards for:

- **Log volume by instance** — line chart showing logs/sec per client over time
- **Error rate by instance** — alert-worthy view of `level=error` across all clients
- **Loki stack health** — ingestion rate, memory, compaction status (using Prometheus metrics)
- **Log channel breakdown** — pie chart of log volume by `channel` label

Store all dashboards as JSON in `grafana/provisioning/dashboards/` and mount them in the Compose file. This ensures any new hub deployment is immediately operational without manual dashboard recreation.

**Effort:** 4–8 hours per dashboard.

### 17.4 Alloy Self-Monitoring (Medium Priority)

Add a `prometheus.scrape` component to each Alloy `config.alloy` that scrapes `http://127.0.0.1:12345/metrics` and forwards to the hub's Prometheus instance. This enables:

- Alerting when an Alloy agent stops shipping (`loki_write_sent_entries_total` rate drops to 0)
- WAL size monitoring (detect when WAL is growing, indicating delivery issues)
- Log parse error detection (`loki_process_dropped_lines_total`)

**Effort:** 2 hours.

### 17.5 Infrastructure as Code (Low Priority — Long Term)

The current setup is deployed manually. For repeatability, especially when onboarding new clients, consider:

- **Terraform:** EC2, S3 bucket, IAM role, Security Groups, lifecycle policies
- **Ansible:** Docker installation, `docker compose up`, `.env` templating across multiple clients

This would reduce a 30-minute manual client onboarding to a 2-minute automated run.

**Effort:** 1–2 weeks for full IaC coverage.

### 17.6 Log Anomaly Detection (Low Priority — Long Term)

Grafana's ML-powered log pattern detection (available in Grafana Cloud, or via the open-source `loki-canary` + `logcli` approach) can surface unusual log patterns without requiring manually written alert rules. Useful for detecting silent failures in Laravel background jobs.

### 17.7 Loki Query Frontend and Caching (Low Priority — Scale)

At 40+ clients or complex dashboard queries, add a Loki query frontend with a Redis cache layer. This separates query scheduling from ingest, improves parallelism, and caches repeated range queries:

```yaml
# loki-config.yaml — future addition
query_scheduler:
  max_outstanding_requests_per_tenant: 100

frontend:
  log_queries_longer_than: 5s
  compress_responses: true
```

---

## Appendix A — Configuration Reference

### Hub `.env` Template

```env
# Grafana credentials
GRAFANA_ADMIN_USER=CHANGE_THIS_USERNAME
GRAFANA_ADMIN_PASSWORD=CHANGE_THIS_PASSWORD_MIN_16_CHARS
GRAFANA_ROOT_URL=https://your-monitoring-domain.com/

# Hub network
HUB_PRIVATE_IP=10.x.x.x

# Loki S3 backend
LOKI_S3_BUCKET=your-actual-bucket-name
```

### Client `.env` Template

```env
# Identity
APP_HOSTNAME=laravel-app-01
INSTANCE_NAME=laravel-app-01
ENVIRONMENT=production

# Timezone (must match server timezone)
TZ_NAME=Asia/Kuala_Lumpur

# Log directory (full path on host)
LARAVEL_LOG_DIR=/var/www/myapp/storage/logs

# Network IPs
PRIVATE_IPV4=10.0.x.x
PUBLIC_IPV4=x.x.x.x

# Hub connection
HUB_IP=10.0.1.50
```

### Key Ports Reference

| Port | Host | Internal | Purpose | Who Accesses |
|---|---|---|---|---|
| 80 | Hub | Grafana :3000 | Grafana UI | Office/VPN or ALB |
| 3100 | Hub | Loki :3100 | Log ingestion + query API | Alloy agents (private IP) |
| 9090 | Hub | Prometheus :9090 | Metrics (future) | Internal only |
| 12345 | Client | Alloy :12345 | Alloy debug UI | Localhost only (after FIX-10) |

### Loki Label Cardinality Budget

| Label | Max Unique Values | Notes |
|---|---|---|
| `job` | 1 | Static |
| `app` | 1 | Static |
| `environment` | 2 | staging, production |
| `hostname` | 20 | One per client EC2 |
| `instance` | 20 | Same as hostname |
| `level` | 6 | debug/info/notice/warning/error/critical |
| `log_environment` | 2 | Same as Laravel APP_ENV |
| `channel` | ~5 | laravel, worker, audit, etc. |
| `private_ipv4` | 20 | One per client (static) |
| **Total stream count** | **~2,400** | Well within Loki's default limit |

Removing `public_ipv4` from labels eliminates a cardinality risk. Using structured metadata for `public_ipv4` is the correct approach per FIX-11.

---

*Document prepared by DevOps Architecture Review — June 2026.*  
*Based on repository review of: `README.md`, `REVIEW.md`, `grafana-main-server/docker-compose.yml`, `grafana-main-server/loki-config.yaml`, `grafana-main-server/grafana/provisioning/datasources/datasources.yaml`, `grafana-main-server/.env.example`, `client-server/docker-compose.yml`, `client-server/config.alloy`, `client-server/.env.example`, `client-server/README.md`.*
