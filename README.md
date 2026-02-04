# VictoriaMetrics Migration Lab (Prometheus → VictoriaMetrics)

This repo is a hands-on lab for practicing a production-style migration from Prometheus to VictoriaMetrics.

It sets up a small observability stack:

- **OpenTelemetry Collector** receives OTLP metrics + traces
- **Prometheus** scrapes metrics from the collector (Prometheus exporter)
- **Grafana** visualizes metrics
- **Jaeger** stores traces
- **VictoriaMetrics** is the destination metrics backend

> This is a learning lab / demo. It is not production-hardened (no HA, no auth, no TLS, no retention planning beyond defaults).

NOTE: remote_write is already enabled in this repo, disable it before you start the lab
Remove the remote_write section from prometheus.yml
---

## Goals

### 1) Run a local observability lab
- OTLP → OTel Collector
- OTel Collector → Prometheus scrape endpoint
- Prometheus → Grafana dashboards
- OTel Collector → Jaeger traces

### 2) Generate telemetry
Use `telemetrygen` to generate:
- metrics (OTLP)
- traces/spans (OTLP)

### 3) Migrate Prometheus metrics to VictoriaMetrics
Production-style approach:
1. Enable **Prometheus remote_write** to send *new* samples to VictoriaMetrics (dual-write). 
2. Create a **Prometheus TSDB snapshot**.
3. Import the snapshot into VictoriaMetrics using **vmctl** (backfill history).
4. Validate in Grafana by comparing Prometheus vs VictoriaMetrics.

Notes:
- **remote_write does not backfill** historical data. It only sends new samples going forward.
- **vmctl snapshot import** is used to backfill the historical Prometheus TSDB blocks.
- **Traces are not migrated to VictoriaMetrics** in this lab. Traces stay in Jaeger. VictoriaMetrics is for metrics.

---

## Requirements

### System
- Linux or macOS (Linux recommended)
- Docker Engine + Docker Compose v2
- Network ports open locally:
  - 9090 Prometheus
  - 3000 Grafana
  - 16686 Jaeger UI
  - 8428 VictoriaMetrics
  - 4317 OTLP gRPC
  - 4318 OTLP HTTP (optional)

### Tools (optional but helpful)
- curl

---

## Quickstart

### Start the stack
docker compose up -d
docker compose ps

Open:
- Prometheus: http://localhost:9090
- Grafana: http://localhost:3000  (default login: admin / admin)
- Jaeger: http://localhost:16686
- VictoriaMetrics UI: http://localhost:8428/vmui

---

## Generate telemetry

### Generate traces
docker compose run --rm telemetrygen traces \
  --otlp-endpoint otel-collector:4317 \
  --otlp-insecure \
  --duration 2m \
  --rate 50 \
  --workers 2 \
  --service "lab-svc"

Validate:
- Jaeger UI → Service = lab-svc → Find Traces

---

### Generate metrics
docker compose run --rm telemetrygen metrics \
  --otlp-endpoint otel-collector:4317 \
  --otlp-insecure \
  --duration 5m \
  --rate 200 \
  --workers 4 \
  --service "lab-svc"

Prometheus validation queries:
- up{job="otel-collector"}
- last_over_time(gen[3h])
- count_over_time(gen[3h])

---

## Migration: Prometheus → VictoriaMetrics

### Step 1: Enable remote_write
Add to prometheus.yml:

remote_write:
  - url: "http://victoriametrics:8428/api/v1/write"

Reload Prometheus:
curl -X POST http://localhost:9090/-/reload

Validate in VictoriaMetrics:
- last_over_time(gen[15m])

---

### Step 2: Create Prometheus snapshot
curl -X POST http://localhost:9090/api/v1/admin/tsdb/snapshot
docker compose exec prometheus ls -lah /prometheus/data/snapshots

---

### Step 3: Import snapshot with vmctl
docker volume ls | grep prom_data

Set:
PROM_VOL="victoriametrics-migration_prom_data"
SNAPSHOT_DIR="PASTE_SNAPSHOT_DIR_NAME"

Run:
docker run --rm \
  --network victoriametrics-migration_default \
  -v "$(docker volume inspect -f '{{.Mountpoint}}' $PROM_VOL):/prometheus" \
  victoriametrics/vmctl:latest \
  prometheus \
  -s \
  --disable-progress-bar \
  --prom-snapshot="/prometheus/data/snapshots/${SNAPSHOT_DIR}" \
  --vm-addr="http://victoriametrics:8428"

---

## Validation

### New data via remote_write
increase(gen[5m])

### Historical data via snapshot
count_over_time(gen[3h])

---

## Non-goals
- Production hardening
- HA setups
- Migrating traces to VictoriaMetrics (doesn't make sense to do this anyways)
