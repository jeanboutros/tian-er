# C13 — Observability

**Status:** Phase A Design — guides Phase B implementation
**Author:** Code Architect
**Date:** 2026-06-09
**Dependencies:** C01 (Platform Infrastructure), C02 (Database)
**Blocks:** C11 (Grafana Dashboards), C12 (Service Orchestration)
**Storage:** V04 (`/var/log/tianer/` :rw), `/var/lib/tianer/metrics/` (new)

---

## 1. Overview

### 1.1 Purpose

C13 Observability is the **central metrics collection, structured logging, and alerting specification** for the entire Tian'er Signal Intelligence Platform. It defines the Prometheus-compatible metrics catalogue [1], the TIANER structured log format, the Promtail-to-Loki log ingestion pipeline [8], the alert rules externalised to `/etc/tianer/alerts.yaml` [6], and the per-language metrics exposition mechanisms. Every component publishes its health and activity through C13; every operator diagnoses the system through C13. The observability architecture follows the monitoring design patterns described in Chapter 6 of the Google SRE book [5].

### 1.2 Scope

| In Scope | Out of Scope |
|----------|-------------|
| Full Prometheus metrics catalogue with labels, types, and collection method for C01/C02/C03/C05/C06/C07/C09 | C08 ML Enrichment metrics (post-MVP) |
| TIANER structured log format specification (keyword, fields, destinations) | Log rotation (C01 tmpfiles.d, C04 PCAP rotation) |
| Contract OBS-1: metrics exposition format (file-based and HTTP) | Application business logic (what each service does internally) |
| Contract OBS-2: structured log format contract | Grafana dashboard design (C11) |
| Promtail pipeline configuration | Loki deployment (post-MVP) |
| Alert thresholds externalised to `/etc/tianer/alerts.yaml` | Alert notification routing (email/Slack — post-MVP) |
| Metrics exposition mechanisms: file-based (C++) to `/var/lib/tianer/metrics/`, HTTP (Python) on component port | Metric scraping interval tuning (operator decision) |
| Prometheus and Promtail container deployment in `tianer-platform` pod | Physical host-side observability (host OS metrics are C01 responsibility, consumed by C13) |
| Per-component observability requirements aggregation | |

### 1.3 Boundaries

C13 sits at the **cross-cutting observability layer** — it does not own the data produced by components, but it defines the format, the catalogue, and the collection mechanism. Components own their metric values; C13 owns the specification they must conform to.

```
┌──────────────────────────────────────────────────────────────────────┐
│                         C13 OBSERVABILITY                             │
│  (Specification layer: metrics catalogue, log format, alerts,         │
│   collection architecture. Does not own runtime data.)                │
└────────────────────┬─────────────────────────────────────────────────┘
                     │ Defines format & contracts
     ┌───────────────┼───────────────┬───────────────┐
     ▼               ▼               ▼               ▼
┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐
│C01 Host │   │C02 DB   │   │C05/C07  │   │C06/C09  │
│Scripts  │   │Scripts  │   │C++ Svcs │   │Python   │
│         │   │         │   │(file)   │   │Svcs(HTTP)│
└────┬────┘   └────┬────┘   └────┬────┘   └────┬────┘
     │             │             │             │
     ▼             ▼             ▼             ▼
┌─────────────────────────────────────────────────────────────────────┐
│  /var/lib/tianer/metrics/*.prom    ←→    C09 :8080/metrics (HTTP)   │
│  (Textfile collector directory, bind-mounted into Prometheus)        │
└────────────────────────────────┬────────────────────────────────────┘
                                 │ Scraped by Prometheus
                                 ▼
                    ┌─────────────────────────┐
                    │    tianer-platform pod   │
                    │  ┌─────────────────────┐ │
                    │  │     Prometheus       │ │
                    │  │  (metrics scraper)   │ │
                    │  └─────────┬───────────┘ │
                    │            │              │
                    │  ┌─────────┴───────────┐ │
                    │  │     Promtail         │ │
                    │  │  (log collector)     │ │
                    │  └─────────────────────┘ │
                    └─────────┬───────────────┘
                              │
                              ▼
                    ┌─────────────────────────┐
                    │       Grafana (C11)      │
                    │  ┌─────────────────────┐ │
                    │  │ Prometheus datasource│ │
                    │  │ (dashboards + alerts)│ │
                    │  └─────────────────────┘ │
                    │  [Loki datasource        │
                    │   — post-MVP]            │
                    └─────────────────────────┘

Log path (parallel):
  Components → stderr/journald → Promtail → Loki (post-MVP)
  Components → /var/log/tianer/*.jsonl → Promtail → Loki (post-MVP)
```

### 1.4 Position in the System

C13 is a **shared platform component** (Layer 0), built after C01 and C02. The C11 Grafana dashboards depend on the metrics catalogue defined here. The C12 service orchestration depends on the Prometheus and Promtail container definitions. C13 is a specification component — its primary output is this design document and the contracts OBS-1 and OBS-2. The runtime components (Prometheus container, Promtail container, alert rules) are deployed by C14.

---

## 2. High-Level Architecture (HLA)

### 2.1 Observability Architecture

Tian'er observability follows a **dual-pipe** architecture:

```
                        ┌─────────────────────────────┐
                        │       tianer-platform pod    │
                        │                              │
  /var/lib/tianer/      │  ┌───────────────────────┐  │        ┌──────────┐
  metrics/              │  │     Prometheus          │  │        │ Grafana  │
  ──────────────────────┼──►  (scrapes textfiles     │──┼────────► (C11)    │
  *.prom                │  │   + HTTP endpoints)     │  │        │ Dash-    │
                        │  └───────────────────────┘  │        │ boards   │
  C09 :8080/metrics     │                              │        └──────────┘
  ──────────────────────┼──► Same Prometheus instance   │
                        │                              │
  journald +            │  ┌───────────────────────┐  │        ┌──────────┐
  /var/log/tianer/      │  │     Promtail            │  │        │ Grafana  │
  ──────────────────────┼──►  (tails journal + files  │──┼────────► Loki     │
  *.jsonl               │  │   → Loki push)          │  │        │ (post-   │
                        │  └───────────────────────┘  │        │  MVP)    │
                        └─────────────────────────────┘        └──────────┘
```

**Metrics pipe (Prometheus → Grafana):**
1. Components expose metrics in Prometheus text format
2. C++ services (C05, C07) write to textfiles in `/var/lib/tianer/metrics/`
3. Python services (C09, C06) expose HTTP `/metrics` endpoints
4. Host metrics (C01) and DB metrics (C02) are collected by scripts that write to textfiles
5. Prometheus (container in `tianer-platform` pod) scrapes the textfile directory and HTTP endpoints every 15 seconds [7]
6. Grafana (standalone container) reads Prometheus as a datasource

**Logs pipe (Promtail → Loki, post-MVP):**
1. All components emit structured JSON logs prefixed with `TIANER` keyword
2. Logs go to journald (via stderr) [4] and to `/var/log/tianer/<component>.jsonl` (for C++ services)
3. Promtail (container in `tianer-platform` pod) tails journald and the JSONL files
4. Promtail pushes to Loki for structured log querying in Grafana
5. Loki is deployed post-MVP; during MVP, logs are accessible via `journalctl` and `grep`

### 2.2 Metrics Collection Flow

```
┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ C01 Host     │  │ C02 DB       │  │ C05 Ingest   │  │ C07 Deep     │
│ metrics.sh   │  │ metrics.sh   │  │ Bridge (C++) │  │ Parser (C++) │
│ (cron 60s)   │  │ (cron 60s)   │  │              │  │              │
│              │  │              │  │ writes to    │  │ writes to    │
│ writes to    │  │ writes to    │  │ metrics file │  │ metrics file │
│ host.prom    │  │ postgres.prom│  │ every 15s    │  │ every 15s    │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
       │                 │                 │                 │
       ▼                 ▼                 ▼                 ▼
┌─────────────────────────────────────────────────────────────────┐
│           /var/lib/tianer/metrics/  (textfile collector)         │
│  host.prom  postgres.prom  blesniff-ingest-ut1.prom  ...        │
└──────────────────────────────┬──────────────────────────────────┘
                               │ Prometheus file_sd + scrape
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  ┌──────────────┐  ┌──────────────┐                              │
│  │ C09 REST API │  │ C06 Gap      │                              │
│  │ :8080/metrics│  │ Detector     │                              │
│  │ (HTTP)       │  │ :9091/metrics│                              │
│  └──────┬───────┘  └──────┬───────┘                              │
│         │                 │                                      │
│         └────────┬────────┘                                      │
│                  │ Prometheus HTTP scrape                        │
│                  ▼                                               │
│         ┌───────────────────┐                                    │
│         │    Prometheus     │                                    │
│         │  (scrape every    │                                    │
│         │   15s, evaluate   │                                    │
│         │   rules every 30s)│                                    │
│         └───────────────────┘                                    │
└─────────────────────────────────────────────────────────────────┘
```

### 2.3 Container Deployment

Both Prometheus and Promtail [8] run as containers in the `tianer-platform` pod alongside C05, C06, C09.

```
┌─────────────────────────────────────────────────────────────────┐
│                    tianer-platform pod                            │
│                                                                   │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────┐ │
│  │ C05      │ │ C06      │ │ C09      │ │Prometheus│ │Promtail│ │
│  │ Ingest   │ │ Gap      │ │ API      │ │          │ │        │ │
│  │ Bridge   │ │ Detector │ │ :8080    │ │ :9090    │ │        │ │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └────────┘ │
│                                                                   │
│  Shared volumes:                                                  │
│    /var/lib/tianer/metrics/  :ro (Prometheus reads textfiles)     │
│    /var/log/tianer/          :ro (Promtail tails logs)            │
│    /etc/tianer/              :ro (alerts.yaml, prometheus.yml)    │
│                                                                   │
│  Network: tianer-net (bridge)                                     │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. Data Model — Prometheus Metrics Catalogue

### 3.1 Naming Convention

All Tian'er metrics follow the Prometheus naming conventions [1] (which build upon the OpenMetrics specification [2]):

| Element | Convention | Example |
|---------|-----------|---------|
| Prefix | `tianer_` | `tianer_host_`, `tianer_db_`, `tianer_ingest_` |
| Suffix | `_total` (counter), `_seconds` (base unit), `_bytes`, `_ratio` | `tianer_ingest_packets_ingested_total` |
| Labels | lowercase, underscore-separated | `sniffer`, `sniffer_type`, `reason`, `mountpoint` |
| Module prefix | Module-specific metrics use the module prefix (e.g. `blesniff_`) for v1 Bluetooth module metrics to avoid future module name collisions | `blesniff_ingest_...` (not `tianer_ingest_...`) |

**Module prefix rule:** Shared platform metrics use `tianer_`. Module-specific metrics use the module prefix (e.g. `blesniff_`). This prevents future modules (GPS, ADS-B) from name-colliding.

### 3.2 C01 — Platform Infrastructure (Host-Level)

These metrics are collected by a host-side script `/usr/local/bin/tianer-metrics.sh`, triggered by a systemd timer every 60 seconds, writing to `/var/lib/tianer/metrics/host.prom`.

All C01 metrics are **shared platform metrics** with the `tianer_host_` prefix.

| Metric | Type | Labels | Description | Collection |
|--------|------|--------|-------------|------------|
| `tianer_host_disk_usage_bytes` | gauge | `mountpoint` | Used bytes on filesystem | `statvfs` on `/`, `/var/lib/tianer/pcap` |
| `tianer_host_disk_total_bytes` | gauge | `mountpoint` | Total bytes on filesystem | `statvfs` |
| `tianer_host_disk_usage_ratio` | gauge | `mountpoint` | Usage ratio (0.0–1.0) | Computed from above |
| `tianer_host_memory_available_bytes` | gauge | — | Available memory from `/proc/meminfo` `MemAvailable` | `/proc/meminfo` |
| `tianer_host_memory_total_bytes` | gauge | — | Total memory from `/proc/meminfo` `MemTotal` | `/proc/meminfo` |
| `tianer_host_cpu_temp_celsius` | gauge | — | SoC temperature in °C | `/sys/class/thermal/thermal_zone0/temp` ÷ 1000 |
| `tianer_host_cpu_throttled` | gauge | `reason` | 1 if throttled, 0 otherwise. `reason`: `undervoltage`, `arm_freq_cap`, `throttled`, `soft_temp_limit` | `vcgencmd get_throttled` bit decomposition |
| `tianer_usb_device_present` | gauge | `device` | 1 if device symlink exists, 0 otherwise | `test -e /dev/tianer/<device>` for `ubertooth0`, `nrf0`, `nrf1`, `nrf2` |
| `tianer_fifo_present` | gauge | `name` | 1 if FIFO exists, 0 otherwise | `test -p /var/run/tianer/<name>.fifo` |
| `tianer_host_uptime_seconds` | gauge | — | Host uptime in seconds | `/proc/uptime` |

**Example host.prom snippet:**
```
# HELP tianer_host_disk_usage_bytes Used bytes on filesystem
# TYPE tianer_host_disk_usage_bytes gauge
tianer_host_disk_usage_bytes{mountpoint="/"} 8520000000
tianer_host_disk_usage_bytes{mountpoint="/var/lib/tianer/pcap"} 3200000000
# HELP tianer_host_cpu_temp_celsius SoC temperature in degrees Celsius
# TYPE tianer_host_cpu_temp_celsius gauge
tianer_host_cpu_temp_celsius 48.8
# HELP tianer_usb_device_present 1 if USB device symlink exists
# TYPE tianer_usb_device_present gauge
tianer_usb_device_present{device="ubertooth0"} 1
tianer_usb_device_present{device="nrf0"} 1
tianer_usb_device_present{device="nrf1"} 0
tianer_usb_device_present{device="nrf2"} 0
```

### 3.3 C02 — Database

These metrics are collected by a host-side script `/usr/local/bin/tianer-db-metrics.sh`, triggered by a systemd timer every 60 seconds, querying PostgreSQL via `tianer_grafana` role and writing to `/var/lib/tianer/metrics/postgres.prom`.

| Metric | Type | Labels | Description | Collection |
|--------|------|--------|-------------|------------|
| `tianer_db_up` | gauge | — | 1 if `pg_isready` succeeds, 0 otherwise | `pg_isready -d tianer` every 15s |
| `tianer_db_size_bytes` | gauge | — | Total database size | `SELECT pg_database_size('tianer')` every 5 min |
| `tianer_db_connections_active` | gauge | `state` | Active connections by state | `SELECT state, count(*) FROM pg_stat_activity GROUP BY state` every 1 min |
| `tianer_db_connections_total` | gauge | — | `max_connections` setting | `SHOW max_connections` |
| `tianer_table_size_bytes` | gauge | `table` | Total relation size per table | `SELECT relname, pg_total_relation_size(relid) FROM pg_stat_user_tables` every 1 hour |
| `tianer_hypertable_chunks_total` | gauge | `hypertable` | Total chunks per hypertable | `SELECT hypertable_name, count(*) FROM timescaledb_information.chunks GROUP BY hypertable_name` every 1 hour |
| `tianer_hypertable_compressed_chunks` | gauge | `hypertable` | Compressed chunks per hypertable | Same query, filtered by `compression_status = 'Compressed'` |
| `tianer_cagg_refresh_last_duration_ms` | gauge | `cagg` | Last continuous aggregate refresh duration | `timescaledb_information.job_stats` every 5 min |
| `tianer_cagg_refresh_last_success` | gauge | `cagg` | 1 if last refresh succeeded, 0 otherwise | `timescaledb_information.job_stats` `last_run_status` |
| `tianer_compression_job_last_success` | gauge | — | 1 if last compression job succeeded | `timescaledb_information.job_stats` |
| `tianer_retention_job_last_success` | gauge | — | 1 if last retention job succeeded | `timescaledb_information.job_stats` |
| `tianer_dead_tuples_total` | gauge | `table` | Estimated dead tuples per table | `SELECT relname, n_dead_tup FROM pg_stat_user_tables` every 15 min |
| `tianer_wal_bytes_written_total` | counter | — | Total WAL bytes written | `pg_stat_wal` `wal_bytes` cumulative, every 1 hour |
| `tianer_db_migration_applied` | gauge | `migration` | 1 if migration is applied, 0 otherwise | `SELECT count(*) FROM bluetooth._migrations WHERE name = $1` at startup + every 1 hour |

### 3.4 C03 — Capture Pipeline

These metrics are written by the sniffer wrapper and tshark wrapper scripts (Bash) to `/var/lib/tianer/metrics/sniffer.prom`. Bash scripts write metrics by echoing lines to a temp file and atomically renaming (see §4.3).

All C03 metrics are **v1 Bluetooth module metrics** with the `blesniff_` prefix.

| Metric | Type | Labels | Description | Collection |
|--------|------|--------|-------------|------------|
| `blesniff_sniffer_running` | gauge | `sniffer`, `sniffer_type` | 1 if sniffer process is alive, 0 otherwise | `pgrep -f` on sniffer PID file every 30s |
| `blesniff_sniffer_packets_captured_total` | counter | `sniffer`, `sniffer_type` | Total packets captured by the sniffer hardware | Sniffer tool output parsing (ubertooth-rx / nRF Sniffer) |
| `blesniff_sniffer_channel` | gauge | `sniffer` | Current BLE advertising channel (37, 38, or 39) | `sniffers.yaml` current config |
| `blesniff_tshark_running` | gauge | `sniffer` | 1 if tshark process for this sniffer is alive | `pgrep -f` on tshark PID file |
| `blesniff_tshark_parse_errors_total` | counter | `sniffer` | Total tshark output lines that failed to parse | Parsed from tshark stderr or exit codes |
| `blesniff_heartbeat_age_seconds` | gauge | `sniffer` | Seconds since last heartbeat update | `stat` on heartbeat file mtime |

### 3.5 C05 — Ingest Bridge (C++)

These metrics are produced by the C++ ingest bridge process. The ingest bridge writes a Prometheus textfile to `/var/lib/tianer/metrics/blesniff-ingest-<sniffer>.prom` every 15 seconds. The file is written atomically (write to `.tmp`, then `rename()`).

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `blesniff_ingest_packets_ingested_total` | counter | `sniffer` | Total packets successfully parsed and batched |
| `blesniff_ingest_packets_copied_total` | counter | `sniffer` | Total packets successfully COPY-inserted to PostgreSQL |
| `blesniff_ingest_malformed_packets_total` | counter | `sniffer`, `reason` | Packets rejected during parsing. `reason`: `missing_field`, `invalid_ts`, `invalid_rssi`, `field_overflow`, `empty_line` |
| `blesniff_ingest_latency_seconds` | histogram | `sniffer` | End-to-end latency from tshark output timestamp to DB INSERT commit. Buckets: 0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1, 5, 10, 30, 60 |
| `blesniff_ingest_batch_size` | gauge | `sniffer` | Number of rows in the current/last batch |
| `blesniff_ingest_batch_flush_duration_ms` | histogram | `sniffer` | Time to flush a batch to PostgreSQL. Buckets: 1, 5, 10, 50, 100, 500, 1000, 5000 |
| `blesniff_ingest_batches_flushed_total` | counter | `sniffer`, `trigger` | Total batches flushed. `trigger`: `size` (reached 300K rows), `timeout` (5-minute timeout elapsed), `shutdown` |
| `blesniff_ingest_db_connected` | gauge | `sniffer` | 1 if connected to PostgreSQL, 0 otherwise |
| `blesniff_ingest_buffer_depth` | gauge | `sniffer` | Current number of rows in the in-memory buffer awaiting flush |
| `blesniff_ingest_reconnects_total` | counter | `sniffer` | Total PostgreSQL reconnection attempts |
| `blesniff_ingest_uptime_seconds` | gauge | `sniffer` | Process uptime in seconds |

**Histogram note:** C++ services use Prometheus text format histograms with explicit `_bucket`, `_sum`, and `_count` time series. The ingest bridge computes approximate histogram buckets internally and writes the full set.

### 3.6 C06 — Gap Detector (Python)

The gap detector is triggered by a systemd timer (oneshot). After each run, it writes results to `/var/lib/tianer/metrics/gapdetect.prom`. Alternatively, it provides a simple HTTP endpoint `:9091/metrics` (prometheus_client library) with the cumulative counters persisted between runs via the metrics file.

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `blesniff_gap_detected_total` | counter | `sniffer` | Total gaps detected (cumulative across all runs) |
| `blesniff_gap_backfill_rows_total` | counter | `sniffer` | Total rows backfilled from PCAP |
| `blesniff_gap_backfill_failures_total` | counter | `sniffer`, `reason` | Backfill attempts that failed. `reason`: `pcap_missing`, `pcap_corrupt`, `db_insert_failed`, `reparse_error` |
| `blesniff_gap_detection_duration_ms` | gauge | `sniffer` | Duration of the last gap detection run |
| `blesniff_gap_last_window_start_timestamp` | gauge | `sniffer` | Unix timestamp of last detected gap start (most recent run) |
| `blesniff_gap_last_window_end_timestamp` | gauge | `sniffer` | Unix timestamp of last detected gap end (most recent run) |
| `blesniff_gap_detector_runs_total` | counter | — | Total number of gap detector invocations |
| `blesniff_gap_detector_last_success` | gauge | — | 1 if last run completed successfully, 0 if errored |

### 3.7 C07 — Deep Parser (C++)

These metrics are produced by the C++ deep parser process and written to `/var/lib/tianer/metrics/deepparse.prom` every 15 seconds.

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `blesniff_deepparse_packets_processed_total` | counter | `sniffer` | Total PCAP packets processed |
| `blesniff_deepparse_packets_valid_ble_total` | counter | `sniffer` | Packets containing valid BLE advertising PDUs |
| `blesniff_deepparse_parse_errors_total` | counter | `reason` | Parse failures. `reason`: `not_ble`, `invalid_pdu_header`, `invalid_advdata_length`, `malformed_tlv`, `truncated_packet`, `decompression_error` |
| `blesniff_deepparse_crc_errors_total` | counter | `sniffer` | BLE packets with CRC-24 validation failure (when CRC validation is enabled) |
| `blesniff_deepparse_jsonl_bytes_written_total` | counter | `sniffer` | Total JSONL output bytes |
| `blesniff_deepparse_jsonl_records_written_total` | counter | `sniffer` | Total JSONL output records (one per valid BLE PDU) |
| `blesniff_deepparse_pcap_files_completed` | counter | `sniffer` | PCAP files fully processed |
| `blesniff_deepparse_processing_duration_ms` | gauge | `sniffer`, `stage` | Duration of last processing stage. `stage`: `read`, `dissect`, `write` |
| `blesniff_deepparse_memory_usage_bytes` | gauge | `sniffer` | Current resident memory (RSS) of the process |
| `blesniff_deepparse_uptime_seconds` | gauge | `sniffer` | Process uptime in seconds |

### 3.8 C09 — REST API (Python)

These metrics are exposed by the FastAPI application on `http://127.0.0.1:8080/metrics` using the `prometheus_client` Python library. The `/metrics` endpoint is bound to `127.0.0.1` only — no authentication is required on this endpoint because it is not reachable externally.

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `tianer_api_requests_total` | counter | `method`, `endpoint`, `status_code` | Total HTTP requests served |
| `tianer_api_request_duration_seconds` | histogram | `method`, `endpoint` | Request latency. Buckets: 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10 |
| `tianer_api_requests_in_flight` | gauge | — | Currently active requests |
| `tianer_api_auth_failures_total` | counter | `reason` | Failed authentication attempts. `reason`: `missing_key`, `invalid_key`, `expired_key` |
| `tianer_api_db_pool_size` | gauge | — | Current database connection pool size |
| `tianer_api_db_pool_available` | gauge | — | Available connections in the pool |
| `tianer_api_db_query_duration_seconds` | histogram | `query_name` | Database query duration |
| `tianer_api_health_status` | gauge | `component` | 1 if component healthy, 0 otherwise. `component`: `db`, `migrations` |

### 3.9 Post-MVP Metrics (C08 — ML Enrichment)

Reserved for post-MVP implementation. The metrics prefix will be `blesniff_ml_`.

### 3.10 Common Metric Labels

The following labels appear across multiple metrics and must be used consistently:

| Label | Values | Applies To |
|-------|--------|------------|
| `sniffer` | `ut1`, `nrf1`, `nrf2`, `nrf3` | C03, C05, C06, C07 |
| `sniffer_type` | `ubertooth`, `nrf52840` | C03 |
| `reason` | Component-specific (see per-component catalogue) | C05, C06, C07, C09 |
| `mountpoint` | `"/"`, `"/var/lib/tianer/pcap"` | C01 |
| `device` | `ubertooth0`, `nrf0`, `nrf1`, `nrf2` | C01 |
| `state` | `active`, `idle`, `idle in transaction` | C02 |
| `table` | `raw_packets`, `device_summary`, `device_enrichment`, `ingest_gaps`, `sniffer_heartbeat` | C02 |
| `hypertable` | `raw_packets`, `device_summary` | C02 |
| `cagg` | `device_5min_buckets` | C02 |
| `migration` | Migration name (e.g. `0001_initial_schema`) | C02 |
| `method` | `GET`, `POST`, `PUT`, `DELETE` | C09 |
| `endpoint` | `/api/devices`, `/api/health`, etc. | C09 |
| `status_code` | `200`, `401`, `404`, `500`, etc. | C09 |

---

## 4. Low-Level Architecture (LLA)

### 4.1 Metrics Exposition Mechanisms

Per D-08, metrics are exposed differently by language:

| Language | Mechanism | Directory/Port | Atomicity |
|----------|-----------|----------------|-----------|
| C++ (C05, C07) | File-based: write Prometheus text format to `.prom` file in `/var/lib/tianer/metrics/` | `/var/lib/tianer/metrics/<component>.prom` | Write to `.tmp`, `rename()` to `.prom` |
| Python (C09) | HTTP endpoint using `prometheus_client` library | `127.0.0.1:8080/metrics` | Built-in library atomicity |
| Python (C06) | HTTP endpoint using `prometheus_client` library OR file-based (if oneshot timer prevents persistent server) | `127.0.0.1:9091/metrics` OR `/var/lib/tianer/metrics/gapdetect.prom` | Built-in / atomic rename |
| Bash (C01, C02, C03) | File-based: script writes to temp file, atomically renames | `/var/lib/tianer/metrics/host.prom`, `postgres.prom`, `sniffer.prom` | Write to `.tmp`, `mv .tmp .prom` |

### 4.2 Textfile Collector Directory

**Path:** `/var/lib/tianer/metrics/`

The textfile collector pattern follows the Prometheus node_exporter `--collector.textfile.directory` convention [7]. This directory is created by C01 `create-dirs.sh` with:
```bash
mkdir -p /var/lib/tianer/metrics
chown tianer:tianer /var/lib/tianer/metrics
chmod 0755 /var/lib/tianer/metrics
```

Inside the `tianer-platform` pod, the directory is bind-mounted `:ro` into the Prometheus container. Prometheus uses the `file_sd_config` to discover `.prom` files:

```yaml
# prometheus.yml (in /etc/tianer/prometheus.yml, V01, mounted :ro)
scrape_configs:
  - job_name: 'tianer-textfile'
    scrape_interval: 15s
    file_sd_configs:
      - files:
          - '/var/lib/tianer/metrics/*.prom'
        refresh_interval: 30s

  - job_name: 'tianer-api'
    scrape_interval: 15s
    static_configs:
      - targets: ['localhost:8080']
    metrics_path: '/metrics'

  - job_name: 'tianer-gapdetect'
    scrape_interval: 15s
    static_configs:
      - targets: ['localhost:9091']
    metrics_path: '/metrics'
```

### 4.3 Atomic File Write for C++ and Bash Metrics

C++ services and Bash scripts must write metrics files atomically to prevent Prometheus from reading a partially-written file:

**Bash pattern:**
```bash
#!/usr/bin/env bash
# tianer-metrics.sh — write host metrics atomically
METRICS_DIR="/var/lib/tianer/metrics"
TMPFILE="${METRICS_DIR}/host.prom.tmp"
FINAL="${METRICS_DIR}/host.prom"

cat > "${TMPFILE}" <<'METRICS'
# HELP tianer_host_cpu_temp_celsius SoC temperature in degrees Celsius
# TYPE tianer_host_cpu_temp_celsius gauge
tianer_host_cpu_temp_celsius 48.8
METRICS

# Atomic rename
mv "${TMPFILE}" "${FINAL}"
```

**C++ pattern** (pseudocode at design level):
```cpp
// write_metrics_atomic() — used by C05 ingest bridge and C07 deep parser
void MetricsWriter::flush() {
    std::string tmp_path = metrics_dir_ + "/" + filename_ + ".tmp";
    std::string final_path = metrics_dir_ + "/" + filename_ + ".prom";

    // Write HELP/TYPE headers and metric lines
    std::ofstream out(tmp_path, std::ios::trunc);
    write_headers(out);
    write_metrics(out);
    out.close();

    // Atomic rename (on same filesystem — guaranteed atomic on Linux)
    if (rename(tmp_path.c_str(), final_path.c_str()) != 0) {
        // Log error via TIANER structured log
        LOG_ERROR("Failed to atomically rename metrics file",
                  "tmp", tmp_path, "final", final_path, "errno", errno);
    }
}
```

### 4.4 Prometheus Alert Rules

Alert rules are defined in `/etc/tianer/prometheus-alerts.yml` (V01, mounted `:ro` into Prometheus container). Threshold values are externalised to `/etc/tianer/alerts.yaml` (see §9) which is consulted during alert rule creation but not directly parsed by Prometheus — the Prometheus rules file contains the actual threshold values, derived from `alerts.yaml` by the deployment script. Alerting rules follow the Prometheus alerting rules specification [6].

### 4.5 Structured Logging Format (TIANER)

All components MUST emit logs in the TIANER structured format. The complete specification is defined in Contract OBS-2 (§5.2). The format is:

```
TIANER | {"ts":"ISO8601","level":"LEVEL","component":"NAME",...}
```

**Log destinations by component type:**

| Source | Primary Destination | Secondary Destination | Collection Path |
|--------|-------------------|----------------------|-----------------|
| Host scripts (C01, C02 metrics) | journald via stderr | — | journald → Promtail → Loki |
| C++ services (C05 Ingest, C07 Deep Parser) | `/var/log/tianer/<component>.jsonl` | stderr (journald) | File tail → Promtail → Loki |
| Python services (C06, C09) | journald via stderr | — | journald → Promtail → Loki |
| Container stdout/stderr (all) | journald via Podman conmon | — | journald → Promtail → Loki |
| Bash scripts (C03 wrappers) | journald via stderr | — | journald → Promtail → Loki |

**C++ dual-write rationale:** C++ services write both to structured log files and journald. The `.jsonl` files provide local inspectability without `journalctl`; journald provides unified collection for Promtail. The log file path is `/var/log/tianer/<component>.jsonl` where `<component>` is `ingest-<sniffer>` (e.g. `ingest-ut1.jsonl`) or `deepparse.jsonl`.

### 4.6 Promtail Pipeline

Promtail runs as a container in the `tianer-platform` pod. It is configured to:

1. **Tail journald** for all containers and host services
2. **Tail `/var/log/tianer/*.jsonl`** for C++ service log files
3. **Extract structured fields** from the JSON payload after `TIANER |`
4. **Push to Loki** (Loki endpoint configured via environment variable; post-MVP) [8]

```yaml
# promtail-config.yml (in /etc/tianer/, V01, mounted :ro)
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: ${LOKI_URL:-http://localhost:3100/loki/api/v1/push}  # Post-MVP

scrape_configs:
  # Scrape journald (all Tian'er services via containers + host)
  - job_name: journal
    journal:
      json: false
      max_age: 12h
      labels:
        job: tianer-journal
    relabel_configs:
      - source_labels: ['__journal__systemd_unit']
        target_label: 'unit'

  # Scrape C++ JSONL log files
  - job_name: tianer-jsonl
    static_configs:
      - targets:
          - localhost
        labels:
          job: tianer-jsonl
          __path__: /var/log/tianer/*.jsonl

    pipeline_stages:
      # Extract JSON payload after "TIANER | "
      - regex:
          expression: '^TIANER \| (?P<json>\{.*\})$'
      - json:
          expressions:
            level:    level
            component: component
            sniffer:  sniffer
            msg:      msg
            ts:       ts
      - labels:
          level:
          component:
          sniffer:
      - timestamp:
          source: ts
          format: RFC3339
```

### 4.7 Metrics Collection Timing

| Collection | Frequency | Rationale |
|-----------|-----------|-----------|
| Prometheus scrape | 15 seconds | Balances freshness with overhead; 15s is Prometheus default and sufficient for all Tian'er metrics |
| Textfile refresh (`file_sd`) | 30 seconds | Twice the scrape interval to ensure newly written files are discovered |
| C++ metric file write | 15 seconds | Matches scrape interval; each scrape always reads a fresh file |
| Host metrics script (C01) | 60 seconds | Host-level metrics change slowly; 60s is sufficient |
| DB metrics script (C02) | 60 seconds | DB metrics change slowly; some queries run less frequently (see §3.3) |
| Alert rule evaluation | 30 seconds | Matches `evaluation_interval` in Prometheus config |
| C06 gap detector | On-demand via systemd timer (every 5 min) | Matches gap detection frequency |
| Loki push (post-MVP) | Near-real-time (Promtail batches every 1s) | Logs should be queryable quickly for debugging |

---

## 5. Inter-Component Contracts

### 5.1 Contract OBS-1 — Metrics Exposition Format

**Contract ID:** OBS-1
**From:** All components (C01, C02, C03, C05, C06, C07, C09)
**To:** C13 Observability (Prometheus scraper)
**Format:** Prometheus text exposition format (version 0.0.4) [2]

#### 5.1.1 File-Based Exposition (C++, Bash)

**Convention:**
- Files are written to `/var/lib/tianer/metrics/<component>.prom`
- File names use lowercase, hyphen-separated: `host.prom`, `postgres.prom`, `blesniff-ingest-ut1.prom`, `blesniff-ingest-nrf1.prom`, `sniffer.prom`, `deepparse.prom`, `gapdetect.prom`
- One metric file per process (C05 ingest bridge writes one file per sniffer instance)
- Files follow the [Prometheus text format](https://prometheus.io/docs/instrumenting/exposition_formats/#text-based-format) [2]:
  - `# HELP <metric_name> <description>`
  - `# TYPE <metric_name> <type>` where type is `counter`, `gauge`, or `histogram`
  - Metric lines: `<metric_name>{label="value",...} <value> <optional_timestamp>`
- Timestamps MUST NOT be included in textfile metrics (Prometheus adds its own scrape timestamp)
- Files are written atomically (write to `.tmp`, then `rename()` to `.prom`)
- Files MUST end with a newline character

**Validation rules:**
- Metric names match `[a-zA-Z_:][a-zA-Z0-9_:]*`
- Label names match `[a-zA-Z_][a-zA-Z0-9_]*`
- Values are valid float64
- `HELP` string is present for each metric (required by Prometheus)
- `TYPE` string is present for each metric (required by Prometheus)

#### 5.1.2 HTTP Exposition (Python)

**Convention:**
- Endpoint: `/metrics` on the component's HTTP port
- Content type: `text/plain; version=0.0.4`
- Python services use the `prometheus_client` library which handles format compliance
- C09 API: `/metrics` endpoint on port 8080, bound to `127.0.0.1`
- C06 Gap Detector: `/metrics` endpoint on port 9091, bound to `127.0.0.1` (if persistent process), or file-based if oneshot

### 5.2 Contract OBS-2 — Structured Log Format

**Contract ID:** OBS-2
**From:** All components (C01–C14)
**To:** C13 Observability (Promtail → Loki), operators (grep, journalctl)
**Format:** `TIANER | <JSON>`

#### 5.2.1 Mandatory Keyword

Every log line emitted by any Tian'er component MUST begin with the literal string `TIANER` followed by a space, a pipe character, a space, and a valid JSON object.

**Regex for extraction:** `^TIANER \| (.+)$`

#### 5.2.2 JSON Structure

The JSON object embedded in the log line MUST contain these fields:

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `ts` | Yes | string | ISO 8601 timestamp with timezone offset (e.g. `2026-06-09T14:32:01Z`, `2026-06-09T14:32:01+00:00`). RFC 3339 compliant. |
| `level` | Yes | string | One of: `DEBUG`, `INFO`, `WARN`, `ERROR`, `FATAL` |
| `component` | Yes | string | Component identifier: `host`, `sniffer`, `tshark`, `ingest`, `gapdetect`, `deepparse`, `ml`, `api`, `grafana`, `postgres`, `prometheus`, `promtail` |
| `msg` | Yes | string | Human-readable message describing the event. Should be sentence-case, no trailing punctuation. Maximum 1024 characters. |
| `sniffer` | Conditional | string | Sniffer instance name: `ut1`, `nrf1`, `nrf2`, `nrf3`. Required when the log event is associated with a specific sniffer instance. |

Additional fields are permitted as component-specific key=value pairs:
- Keys must be `snake_case` strings
- Values may be strings, numbers, or booleans
- No nested objects (flat JSON only — required for Loki label extraction)
- No arrays
- Maximum 20 additional fields per log line

#### 5.2.3 Example Log Lines

```
TIANER | {"ts":"2026-06-09T14:32:01Z","level":"INFO","component":"host","msg":"Created tianer system user","uid":999}
TIANER | {"ts":"2026-06-09T14:35:00Z","level":"INFO","component":"sniffer","sniffer":"ut1","msg":"Capture started","channel":37,"packets_per_sec":127}
TIANER | {"ts":"2026-06-09T14:40:00Z","level":"WARN","component":"ingest","sniffer":"ut1","msg":"Database connection lost","retry_in_s":5}
TIANER | {"ts":"2026-06-09T14:40:15Z","level":"ERROR","component":"ingest","sniffer":"ut1","msg":"Failed to flush batch","batch_size":285000,"error":"connection refused","rows_lost":285000}
TIANER | {"ts":"2026-06-09T14:45:00Z","level":"INFO","component":"host","msg":"Host metrics collected","cpu_temp_c":48.8,"disk_usage_pct":62,"mem_avail_mb":3200}
TIANER | {"ts":"2026-06-09T15:00:00Z","level":"FATAL","component":"ingest","sniffer":"nrf1","msg":"Ingest bridge crashing","reason":"unhandled exception","signal":"SIGSEGV"}
```

#### 5.2.4 Validation Rules (OBS-2)

1. Every log line MUST begin with `TIANER | `
2. The JSON after `TIANER | ` MUST be valid JSON (parseable by `jq`)
3. The JSON MUST contain `ts`, `level`, `component`, and `msg`
4. `level` MUST be one of: `DEBUG`, `INFO`, `WARN`, `ERROR`, `FATAL`
5. `component` MUST be one of the recognised component identifiers
6. If a `sniffer` field is present, it MUST be one of: `ut1`, `nrf1`, `nrf2`, `nrf3`
7. `ts` MUST be valid ISO 8601 with timezone
8. JSON MUST be flat (no nested objects or arrays)

#### 5.2.5 Grepability

The `TIANER` keyword prefix enables single-pattern grep across all services:

```bash
# All Tian'er logs across all components
grep '^TIANER ' /var/log/tianer/*.jsonl

# Only errors
grep '^TIANER |.*"level":"ERROR"' /var/log/tianer/*.jsonl

# From journald (all containers + host)
journalctl | grep '^TIANER '

# Specific component and level from journald
journalctl CONTAINER_NAME=tianer-ingest | grep '^TIANER |.*"level":"ERROR"'

# Parse with jq for structured queries
grep '^TIANER ' /var/log/tianer/ingest-ut1.jsonl | sed 's/^TIANER | //' | jq 'select(.level == "ERROR")'
```

---

## 6. Failure Modes & Recovery

### 6.1 Failure Mode Catalogue

| ID | Failure | Component(s) | Detection | Propagation | Recovery | SLA |
|----|---------|-------------|-----------|-------------|----------|-----|
| FM-OBS-01 | **Prometheus container crash** | C13 Prometheus | Grafana shows "no data"; `tianer-prometheus` systemd unit shows `inactive` | All dashboards go blank; alert rules stop evaluating | Podman auto-restarts Prometheus; Prometheus replays WAL on restart; gap in scrape data is ≤ restart time (~3 seconds) | < 5s data gap per crash |
| FM-OBS-02 | **Metrics file not updated** (C++ service hung) | C05, C07 | Prometheus scrape sees stale timestamp; `tianer_ingest_uptime_seconds` stops incrementing | Metrics for that component show last value until TTL | Operator investigates hung process; systemd/Podman watchdog restarts stuck process; metric resumes updating | < 30s detection (scrape interval × 2) |
| FM-OBS-03 | **Metrics directory full** | C01 (disk) | `tianer_host_disk_usage_ratio{mountpoint="/var/lib/tianer/metrics"}` > 0.90 | All file-based metrics stop updating; Prometheus scrapes stale data | Operator clears stale `.tmp` files; rotates old metrics files; increases disk quota | Alert at 80%, critical at 90%. Metrics files are < 1 MB total — this is a disk-level failure, not a metrics-specific one |
| FM-OBS-04 | **Promtail container crash** | C13 Promtail | Loki shows no new log entries; `tianer-promtail` systemd unit shows `inactive` | Logs are not forwarded to Loki; journald and JSONL files continue accumulating locally | Podman auto-restarts Promtail; Promtail replays from its position file on restart; gap in Loki is ≤ restart time | < 5s log gap per crash. Local files preserve all logs — no data loss |
| FM-OBS-05 | **Prometheus rule evaluation failure** | C13 Prometheus | Prometheus logs: `rule evaluation failed`; alert `TianerPrometheusRuleEvalFailed` fires (meta-alert) | Alert rules stop firing; failures go undetected | Operator checks Prometheus logs for the specific rule error; corrects syntax and reloads via `POST /-/reload` | Alert within 60s (evaluation interval × 2). No data loss — rules re-evaluate on reload |
| FM-OBS-06 | **Metric cardinality explosion** | C05, C07, C09 | `prometheus_tsdb_head_series` grows unbounded; Prometheus memory usage spikes | Prometheus slows down; scrapes time out; alerts may fire on stale data | Operator identifies the high-cardinality label (e.g. unbounded `device_id` or `mac_address`); restricts label values or adds relabel_config to drop the offending series | Detection by Grafana dashboard panel monitoring `prometheus_tsdb_head_series` |
| FM-OBS-07 | **Atomic rename race condition** | C05, C07, Bash scripts | Prometheus reads partially-written `.tmp` file (if `file_sd` picks up `.tmp` before rename) | Corrupted metric values in Prometheus; may cause alert false positives | C05/C07 MUST write to a `.tmp` file with a different filename pattern than the `.prom` glob. Prometheus `file_sd` MUST only match `*.prom`, not `*.tmp`. The `rename()` is atomic on the same filesystem. | Prevented by design: `*.tmp` files are never scraped |
| FM-OBS-08 | **Tainted metrics (component writes incorrect values)** | Any | No automated detection for semantic errors (T3-level); operator observes improbable values in Grafana | Misleading dashboards; alert false positives or false negatives | Cross-verification: compare metrics from independent sources (e.g. PCAP packet count vs DB row count for gap detection). Operator investigation | Manual detection; post-MVP: anomaly detection on metric value ranges |
| FM-OBS-09 | **Stale metrics (component stopped but last value persists)** | C05, C07, C06 | Prometheus `up` metric for HTTP targets is 0; textfile metrics have no inherent staleness detection | Dashboards show last-known value indefinitely; no alert because gauge looks healthy | For file-based metrics: include a `_timestamp_seconds` metric that Prometheus can use for staleness. For HTTP: `up` metric is 0 when scrape fails. Prometheus staleness handling (default 5 min) marks series as stale. | Staleness detected within 5 min (Prometheus default). Alert rules SHOULD include `unless` clauses to check for recent data. |
| FM-OBS-10 | **Log disk fills up (V04 full)** | C01, C13 | `tianer_host_disk_usage_ratio{mountpoint="/var/log/tianer"}` > 0.90 | C++ services cannot write JSONL files; `write()` returns `ENOSPC`; service logs the error and continues (graceful degradation — no crash) | Log rotation (managed by `logrotate` or systemd `journald` settings). Operator extends V04 or reduces retention. | Alert at 80%. Services continue operating — only log file writes are affected. journald logs unaffected (separate disk). |

### 6.2 Recovery Procedures

**Prometheus crash recovery:**
```bash
# Check status
systemctl --user status tianer-prometheus

# Check recent crashes
journalctl --user -u tianer-prometheus --since "10 minutes ago"

# Force restart
systemctl --user restart tianer-prometheus

# Verify scraping resumes
curl -s http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | {job: .labels.job, health: .health}'
```

**Promtail crash recovery:**
```bash
# Check status
systemctl --user status tianer-promtail

# Force restart
systemctl --user restart tianer-promtail

# Verify Loki ingestion (post-MVP)
curl -s http://localhost:3100/ready
```

**Stale metrics investigation:**
```bash
# Check when a textfile metric was last updated
stat /var/lib/tianer/metrics/blesniff-ingest-ut1.prom

# Check if the process is alive
pgrep -f blesniff-ingest

# Check Prometheus staleness
curl -s 'http://localhost:9090/api/v1/query?query=blesniff_ingest_packets_ingested_total' | jq '.data.result'
```

---

## 7. Observability (Meta)

### 7.1 Observing the Observability Stack

The observability stack itself must be observable. Prometheus exports its own metrics at `/metrics`, which are scraped as part of the normal pipeline:

| Self-Metric | Description | Alert |
|-------------|-------------|-------|
| `prometheus_tsdb_head_series` | Total active series in TSDB head | > 10,000 → cardinality warning |
| `prometheus_engine_query_duration_seconds` | Query evaluation latency | p99 > 5s → warning |
| `prometheus_rule_evaluation_failures_total` | Rule evaluation failures | Any increase → critical |
| `prometheus_notifications_dropped_total` | Alert notifications dropped | Any increase → critical |
| `prometheus_target_scrape_pool_targets` | Configured scrape targets | == 0 → no targets configured |
| `up` (per scrape target) | Target reachability | == 0 for any target → critical |
| `promtail_read_bytes_total` | Bytes read by Promtail | Flatline → Promtail stuck |
| `promtail_dropped_entries_total` | Log entries dropped by Promtail | Any increase → critical |
| `process_resident_memory_bytes` (Prometheus) | Prometheus memory usage | > 512 MB → warning |

### 7.2 Health Check

The C09 API `/api/health` endpoint includes the observability stack status when available:

```json
{
    "status": "ok",
    "components": {
        "db": "ok",
        "prometheus": "ok",
        "promtail": "ok"
    }
}
```

The `prometheus` health is determined by querying `http://localhost:9090/-/healthy`. The `promtail` health is determined by querying `http://localhost:9080/ready`.

### 7.3 Structured Logging for the Observability Stack

Prometheus and Promtail containers emit logs to journald via Podman conmon. These logs are not prefixed by default. A wrapper script or the container entrypoint MUST add the `TIANER` keyword prefix for consistency:

```
# Prometheus container wraps the binary with a log transformer
TIANER | {"ts":"2026-06-09T14:30:00Z","level":"INFO","component":"prometheus","msg":"Server is ready to receive web requests"}
TIANER | {"ts":"2026-06-09T14:30:01Z","level":"INFO","component":"promtail","msg":"Promtail started","version":"2.9.0"}
```

---

## 8. Security Considerations

### 8.1 Metrics Endpoint Access Control

| Endpoint | Binding | Authentication | Rationale |
|----------|---------|---------------|-----------|
| C09 `/metrics` (port 8080) | `127.0.0.1` only | None | Loopback-only binding prevents external access. No secrets in metric names or labels. |
| C06 `/metrics` (port 9091) | `127.0.0.1` only | None | Same as above. Only scraped by local Prometheus. |
| Prometheus `:9090` | `127.0.0.1` only | None | No external access needed. Grafana connects via `tianer-net` bridge inside Podman. |
| Grafana `:3000` | `127.0.0.1` only | Grafana login | Accessed via C09 reverse proxy or local browser. |

### 8.2 No Secrets in Metric Labels

**Hard rule:** Metric labels MUST NOT contain secrets, API keys, passwords, or personally identifiable information.

Labels are exposed in plaintext in the `/metrics` endpoint output and stored in Prometheus TSDB. This includes:
- No API keys in labels
- No database connection strings in labels
- No BLE MAC addresses in labels (MAC addresses may be classified as PII in some jurisdictions; they are stored in the database but MUST NOT appear in Prometheus metric labels)
- No raw PCAP payload data in labels

Acceptable labels: `sniffer`, `sniffer_type`, `reason`, `mountpoint`, `device`, `state`, `table`, `method`, `endpoint`, `status_code`, `component`, `trigger`, `stage`, `query_name`, `migration`, `hypertable`, `cagg`, `channel`.

**Catch-all rule for C++ services:** When a label value could be unbounded or user-controlled, restrict it. If a `reason` label needs a new value, add it to the approved list in this document first.

### 8.3 Prometheus No-Authentication Risk

Prometheus (`:9090`) and the metrics endpoints (`:8080/metrics`, `:9091/metrics`) are bound to `127.0.0.1` only with no authentication. An attacker who gains local shell access to the Raspberry Pi could:

1. Read all metrics (revealing system state, packet counts, error rates — not secrets)
2. Query Prometheus API (read-only by default; write is disabled in Tian'er config)

**Mitigation:** The `tianer` user has no login shell, no password, and no sudo. Physical and SSH access to the device is controlled at the OS level (C01). The attack surface is local only — no network exposure.

**Post-MVP:** Consider adding HTTP basic auth to Prometheus and `/metrics` endpoints if remote access is ever required.

### 8.4 Log Message Security

Log messages (the `msg` field and additional JSON fields in the TIANER format) MUST NOT contain:
- Database passwords or connection strings
- API keys
- TLS private key material
- Full BLE advertising payloads (may contain PII)
- Session tokens or JWTs

Error messages may include sanitized context (e.g. `"error": "Connection refused"`) but not raw error strings that might leak internal paths or credentials.

---

## 9. Configuration

### 9.1 Alert Thresholds — `/etc/tianer/alerts.yaml`

Alert thresholds are externalised to `/etc/tianer/alerts.yaml` (V01, mounted `:ro` into the Prometheus container). This file is the single source of truth for all Tian'er alert thresholds. Operators edit this file to tune alert sensitivity; Prometheus reloads rules on `POST /-/reload` without restart.

```yaml
# /etc/tianer/alerts.yaml — Tian'er alert thresholds
# Mounted :ro into Prometheus container (V01 /etc/tianer/)
# Changes take effect on Prometheus reload (POST /-/reload)
# No restart needed.

alerts:
  # C01 — Host infrastructure
  host:
    disk:
      warning_ratio: 0.80
      critical_ratio: 0.90
      evaluation_interval: 60s
    memory:
      warning_ratio: 0.85    # (1 - mem_available / mem_total)
      critical_ratio: 0.95
    thermal:
      throttle_duration_threshold: 60s  # CPU must be throttled this long before alert
    usb:
      missing_duration_threshold: 30s   # Device absent this long before alert
    fifo:
      missing_duration_threshold: 30s

  # C02 — Database
  database:
    up:
      down_duration_threshold: 30s
    size:
      warning_bytes: 4831838208    # 4.5 GB (90% of 5 GB V06 budget)
    connections:
      warning_ratio: 0.80          # 80% of max_connections

  # C03 — Capture Pipeline
  capture:
    sniffer:
      down_duration_threshold: 30s
    heartbeat:
      stale_seconds: 120           # Heartbeat not updated in 120 seconds → sniffer likely dead

  # C05 — Ingest Bridge
  ingest:
    malformed_rate_warning: 10     # > 10 malformed packets per minute per sniffer
    malformed_rate_critical: 100   # > 100 malformed packets per minute
    db_disconnected_duration: 30s  # DB disconnected > 30s → alert
    buffer_depth_warning: 100000   # Buffer > 100K rows → backpressure alert

  # C06 — Gap Detector
  gap:
    detection_failure: 1           # Any failed gap detection run

  # C09 — REST API
  api:
    error_rate_5xx_threshold: 0.05 # > 5% of requests are 5xx
    latency_p99_threshold_ms: 5000 # p99 latency > 5 seconds
    auth_failure_rate_threshold: 10 # > 10 auth failures per minute

  # C13 — Observability (meta)
  observability:
    prometheus:
      series_count_warning: 10000
      memory_mb_warning: 512
      scrape_failures: 3           # > 3 consecutive scrape failures
    promtail:
      dropped_entries: 1           # Any dropped log entry
      read_bytes_flatline: 300     # No bytes read in 5 minutes → Promtail stuck
```

### 9.2 Prometheus Configuration — `/etc/tianer/prometheus.yml`

```yaml
# /etc/tianer/prometheus.yml
# Mounted :ro into Prometheus container (V01)
global:
  scrape_interval: 15s
  evaluation_interval: 30s
  scrape_timeout: 10s

alerting:
  alertmanagers:
    - static_configs:
        - targets: []  # No Alertmanager in MVP; alerts visible in Prometheus UI only

rule_files:
  - '/etc/tianer/prometheus-alerts.yml'

scrape_configs:
  # C01/C02/C05/C06/C07 — textfile collector
  - job_name: 'tianer-textfile'
    scrape_interval: 15s
    static_configs:
      - targets: ['localhost']
        labels:
          instance: 'tianer-cm5'
    file_sd_configs:
      - files:
          - '/var/lib/tianer/metrics/*.prom'
        refresh_interval: 30s

  # C09 — REST API
  - job_name: 'tianer-api'
    scrape_interval: 15s
    static_configs:
      - targets: ['localhost:8080']
    metrics_path: '/metrics'

  # C06 — Gap Detector (if using HTTP exposition)
  - job_name: 'tianer-gapdetect'
    scrape_interval: 15s
    static_configs:
      - targets: ['localhost:9091']
    metrics_path: '/metrics'

  # C13 — Prometheus self-scrape
  - job_name: 'prometheus'
    scrape_interval: 30s
    static_configs:
      - targets: ['localhost:9090']
```

### 9.3 Prometheus Alert Rules — `/etc/tianer/prometheus-alerts.yml`

```yaml
# /etc/tianer/prometheus-alerts.yml
# Prometheus alert rules — auto-generated from /etc/tianer/alerts.yaml by deploy script.
# DO NOT EDIT DIRECTLY — edit alerts.yaml and run tianer-generate-alerts.sh.
groups:
  - name: tianer-host
    rules:
      - alert: TianerHostDiskWarning
        expr: tianer_host_disk_usage_ratio > 0.80
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Host disk usage > 80% on {{ $labels.mountpoint }}"
          description: "Disk {{ $labels.mountpoint }} is at {{ $value | humanizePercentage }} usage."

      - alert: TianerHostDiskCritical
        expr: tianer_host_disk_usage_ratio > 0.90
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Host disk usage > 90% on {{ $labels.mountpoint }}"
          description: "Disk {{ $labels.mountpoint }} is critically full at {{ $value | humanizePercentage }}. Stop non-essential containers and purge old PCAP files."

      - alert: TianerHostThermalThrottle
        expr: tianer_host_cpu_throttled{reason="throttled"} == 1
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "CPU is thermal throttling"
          description: "Throttling has been active for > 60 seconds. Check cooling."

      - alert: TianerUSBDeviceMissing
        expr: tianer_usb_device_present == 0
        for: 30s
        labels:
          severity: critical
        annotations:
          summary: "USB device {{ $labels.device }} is missing"
          description: "Device /dev/tianer/{{ $labels.device }} has been absent for > 30 seconds."

      - alert: TianerHostMemoryWarning
        expr: (1 - tianer_host_memory_available_bytes / tianer_host_memory_total_bytes) > 0.85
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Host memory usage > 85%"
          description: "Memory usage is at {{ $value | humanizePercentage }}. Check for memory leaks."

  - name: tianer-database
    rules:
      - alert: TianerDBDown
        expr: tianer_db_up == 0
        for: 30s
        labels:
          severity: critical
        annotations:
          summary: "PostgreSQL is down"
          description: "pg_isready has failed for > 30 seconds. Check tianer-postgres container."

      - alert: TianerDBConnectionHigh
        expr: tianer_db_connections_active / tianer_db_connections_total > 0.80
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Database connections > 80% of max"
          description: "Active connections are at a high percentage of max_connections."

      - alert: TianerCaggRefreshFailed
        expr: tianer_cagg_refresh_last_success == 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Continuous aggregate refresh failed"
          description: "Cagg {{ $labels.cagg }} last refresh failed. Check timescaledb_information.job_stats."

  - name: tianer-ingest
    rules:
      - alert: TianerIngestDBDisconnected
        expr: blesniff_ingest_db_connected == 0
        for: 30s
        labels:
          severity: critical
        annotations:
          summary: "Ingest bridge {{ $labels.sniffer }} disconnected from database"
          description: "Ingest bridge cannot reach PostgreSQL. Packets are buffering in memory."

      - alert: TianerIngestHighMalformed
        expr: rate(blesniff_ingest_malformed_packets_total[1m]) > 100
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "High malformed packet rate on {{ $labels.sniffer }}"
          description: "> 100 malformed packets/min. Check tshark output format compatibility."

  - name: tianer-gap
    rules:
      - alert: TianerGapDetected
        expr: increase(blesniff_gap_detected_total[5m]) > 0
        labels:
          severity: warning
        annotations:
          summary: "Ingest gap detected on {{ $labels.sniffer }}"
          description: "Gap detector found a time window with missing packets. Backfill should be in progress."

      - alert: TianerBackfillFailed
        expr: rate(blesniff_gap_backfill_failures_total[5m]) > 0
        labels:
          severity: critical
        annotations:
          summary: "Backfill failures on {{ $labels.sniffer }}"
          description: "Gap backfill is failing. Check PCAP file availability and DB connectivity."

  - name: tianer-observability
    rules:
      - alert: TianerPrometheusScrapeFailure
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Prometheus scrape target {{ $labels.job }} is down"
          description: "Cannot scrape {{ $labels.job }}. Metrics from this target are stale."

      - alert: TianerPromtailDroppingLogs
        expr: promtail_dropped_entries_total > 0
        labels:
          severity: critical
        annotations:
          summary: "Promtail is dropping log entries"
          description: "Log entries are being dropped. Check Loki endpoint or Promtail memory."
```

### 9.4 Scrape Interval Rationale

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| `scrape_interval` | 15 seconds | Prometheus default. Balances freshness with CPU/memory overhead. At 100 metrics with 10 labels each (~1000 series), 15s scraping on a Pi CM5 is negligible. |
| `evaluation_interval` | 30 seconds | Twice the scrape interval avoids false positives from a single missed scrape. |
| `scrape_timeout` | 10 seconds | Shorter than scrape interval to prevent overlapping scrapes |
| Textfile `refresh_interval` | 30 seconds | Twice the scrape interval ensures new `.prom` files are discovered promptly |
| Alert `for` duration | 30s–5m (per alert) | Short `for` for obvious failures (DB down); longer `for` for threshold-based alerts to absorb transient spikes |

---

## 10. Test Plan

### 10.1 Unit Tests

| Test | Description | Acceptance |
|------|-------------|------------|
| `test_obs2_format_validation` | Validate that a TIANER log line passes all OBS-2 rules | Given valid JSON with required fields → PASS. Given missing `ts` → FAIL. Given invalid `level` value → FAIL. |
| `test_obs1_metric_name_validation` | Validate Prometheus metric name format | `tianer_host_disk_usage_bytes` → PASS. `tianer host disk` (spaces) → FAIL. `1tianer_metric` (starts with digit) → FAIL. |
| `test_obs1_label_name_validation` | Validate Prometheus label name format | `mountpoint` → PASS. `Mount Point` (spaces) → FAIL. |
| `test_atomic_rename_writes_complete_file` | Verify that atomic rename never produces a partial file | Write 1000 lines, `rename()`, read back → all 1000 lines present. Repeat 100×. |
| `test_textfile_parseable_by_prometheus` | Parse a generated `.prom` file with Prometheus text format parser | `promtool check metrics <file>` exits 0 |
| `test_alert_rule_syntax` | All alert rules in `prometheus-alerts.yml` are valid PromQL | `promtool check rules prometheus-alerts.yml` exits 0 |
| `test_prometheus_config_valid` | `prometheus.yml` is valid | `promtool check config prometheus.yml` exits 0 |

### 10.2 Integration Tests

| Test | Description | Acceptance |
|------|-------------|------------|
| `test_file_sd_discovers_new_file` | Prometheus discovers a newly created `.prom` file within 35s | File appears in `targets` list within two `refresh_interval` cycles |
| `test_atomic_rename_not_read_partial` | During a scrape, Prometheus reads a `.prom` file being renamed | Prometheus reads either the old file (complete) or the new file (complete). Never a partial read. Verified by running 1000 scrapes during 1000 renames. |
| `test_c09_metrics_endpoint_returns_prometheus_format` | C09 `/metrics` returns valid Prometheus text | `curl -s http://127.0.0.1:8080/metrics \| promtool check metrics` exits 0 |
| `test_alert_fires_on_condition` | When `tianer_db_up` drops to 0 for 35s, the alert `TianerDBDown` fires | Alert is `FIRING` in Prometheus alerts API |
| `test_alert_resolves` | When `tianer_db_up` returns to 1, `TianerDBDown` resolves | Alert state transitions to `INACTIVE` within 1 evaluation interval |
| `test_promtail_parses_tianer_log` | Promtail correctly extracts `level`, `component`, `sniffer` from a TIANER log line | Labels appear in Loki query results (post-MVP) |
| `test_log_file_rotation_no_data_loss` | When `/var/log/tianer/ingest-ut1.jsonl` is rotated, no log lines are lost | Promtail position file correctly tracks rotated file; all lines appear in Loki |

### 10.3 Format Validation Tests

The following validation scripts are part of the C13 test suite:

```bash
#!/usr/bin/env bash
# tests/test_obs2_format.sh — validate TIANER log format compliance
set -euo pipefail

# Test 1: Valid log line
echo 'TIANER | {"ts":"2026-06-09T14:32:01Z","level":"INFO","component":"host","msg":"Test message"}' | \
  validate_tianer_log || { echo "FAIL: Valid log line rejected"; exit 1; }

# Test 2: Missing TIAMER prefix
echo 'OTHER | {"ts":"2026-06-09T14:32:01Z","level":"INFO","component":"host","msg":"Test"}' | \
  validate_tianer_log && { echo "FAIL: Line without TIANER prefix accepted"; exit 1; }

# Test 3: Invalid JSON
echo 'TIANER | {invalid json}' | \
  validate_tianer_log && { echo "FAIL: Invalid JSON accepted"; exit 1; }

# Test 4: Missing required field
echo 'TIANER | {"level":"INFO","component":"host","msg":"Test"}' | \
  validate_tianer_log && { echo "FAIL: Missing 'ts' field accepted"; exit 1; }

# Test 5: Invalid level value
echo 'TIANER | {"ts":"2026-06-09T14:32:01Z","level":"CRITICAL","component":"host","msg":"Test"}' | \
  validate_tianer_log && { echo "FAIL: Invalid level value accepted"; exit 1; }

# Test 6: Nested JSON (should reject)
echo 'TIANER | {"ts":"2026-06-09T14:32:01Z","level":"INFO","component":"host","msg":"Test","nested":{"key":"value"}}' | \
  validate_tianer_log && { echo "FAIL: Nested JSON accepted"; exit 1; }

echo "ALL OBS-2 format validation tests PASSED"
```

```bash
#!/usr/bin/env bash
# tests/test_prometheus_metrics_format.sh — validate Prometheus metrics format
set -euo pipefail

METRICS_DIR="/var/lib/tianer/metrics"

for file in "${METRICS_DIR}"/*.prom; do
    echo "Validating ${file}..."
    promtool check metrics "${file}" || { echo "FAIL: ${file} is not valid Prometheus text format"; exit 1; }
done

echo "ALL Prometheus metrics format tests PASSED"
```

### 10.4 Acceptance Criteria for C13

1. All metrics in the catalogue (§3) are documented with type, labels, and collection method
2. Contract OBS-1 specifies file and HTTP exposition formats with validation rules
3. Contract OBS-2 specifies the TIANER log format with validation rules
4. All alert rules in `prometheus-alerts.yml` reference thresholds from `alerts.yaml`
5. Prometheus `prometheus.yml` is valid and parseable by `promtool`
6. Alert rules file is valid and parseable by `promtool`
7. No secrets or PII appear in metric labels
8. All metric files use atomic rename
9. Promtail pipeline extracts structured fields from TIANER log lines
10. The C09 `/metrics` endpoint returns valid Prometheus text format

---

## 11. Deployment Notes

### 11.1 Container Images

| Container | Base Image | Purpose | Ports | MVP/Post-MVP |
|-----------|-----------|---------|-------|-------------|
| Prometheus | `prom/prometheus:v2.54.0@sha256:...` (digest-pinned) | Scrape metrics, evaluate alert rules | `127.0.0.1:9090:9090` | MVP |
| Promtail | `grafana/promtail:3.0.0@sha256:...` (digest-pinned) | Collect logs, push to Loki | `127.0.0.1:9080:9080` | MVP (collects logs; Loki target is configured but may be absent) |
| Loki | `grafana/loki:3.0.0@sha256:...` (digest-pinned) | Log aggregation and querying | `127.0.0.1:3100:3100` | Post-MVP |

### 11.2 Quadlet Unit Files

Prometheus and Promtail are deployed as part of the `tianer-platform` pod via Quadlet `.container` files managed by C14.

**Prometheus Quadlet** (conceptual — final syntax in C14):
```
[Container]
Image=prom/prometheus:v2.54.0@sha256:...
Pod=tianer-platform.pod
PublishPort=127.0.0.1:9090:9090
Volume=/etc/tianer/prometheus.yml:/etc/prometheus/prometheus.yml:ro
Volume=/etc/tianer/prometheus-alerts.yml:/etc/prometheus/prometheus-alerts.yml:ro
Volume=/var/lib/tianer/metrics:/var/lib/tianer/metrics:ro
Volume=/var/lib/tianer/prometheus-data:/prometheus:rw
Environment=PROMETHEUS_STORAGE_RETENTION=30d
```

**Promtail Quadlet** (conceptual — final syntax in C14):
```
[Container]
Image=grafana/promtail:3.0.0@sha256:...
Pod=tianer-platform.pod
PublishPort=127.0.0.1:9080:9080
Volume=/etc/tianer/promtail-config.yml:/etc/promtail/config.yml:ro
Volume=/var/log/tianer:/var/log/tianer:ro
Volume=/run/log/journal:/run/log/journal:ro
Volume=/var/lib/tianer/promtail-positions:/tmp:rw
Environment=LOKI_URL=http://localhost:3100
```

### 11.3 Build Steps

C13 is a specification component. Its "build" consists of deploying configuration files:

```bash
# Part of C14 deployment automation:
# 1. Copy config files to /etc/tianer/
cp deploy/config/prometheus.yml /etc/tianer/prometheus.yml
cp deploy/config/prometheus-alerts.yml /etc/tianer/prometheus-alerts.yml
cp deploy/config/promtail-config.yml /etc/tianer/promtail-config.yml
cp deploy/config/alerts.yaml /etc/tianer/alerts.yaml
chmod 0644 /etc/tianer/prometheus.yml /etc/tianer/prometheus-alerts.yml /etc/tianer/promtail-config.yml /etc/tianer/alerts.yaml

# 2. Create metrics directory
mkdir -p /var/lib/tianer/metrics
chown tianer:tianer /var/lib/tianer/metrics
chmod 0755 /var/lib/tianer/metrics

# 3. Create Prometheus data directory (Podman volume)
mkdir -p /var/lib/tianer/prometheus-data
chown tianer:tianer /var/lib/tianer/prometheus-data

# 4. Create Promtail positions directory
mkdir -p /var/lib/tianer/promtail-positions
chown tianer:tianer /var/lib/tianer/promtail-positions

# 5. Validate Prometheus configuration
promtool check config /etc/tianer/prometheus.yml
promtool check rules /etc/tianer/prometheus-alerts.yml

# 6. Install host metrics collection timer (part of C01/C12)
systemctl --user enable tianer-metrics.timer
systemctl --user enable tianer-db-metrics.timer
systemctl --user start tianer-metrics.timer
systemctl --user start tianer-db-metrics.timer
```

### 11.4 Post-MVP: Loki Deployment

Loki is deferred to post-MVP. During the MVP phase:
- Promtail collects logs to its internal buffer
- If no Loki endpoint is configured, Promtail starts in dry-run mode (collects but does not push)
- Logs are accessible via `journalctl` and `grep` on `/var/log/tianer/*.jsonl`
- The Promtail position file tracks progress — when Loki is deployed, it catches up from the last position

Post-MVP deployment adds:
1. Loki container in `tianer-platform` pod
2. Loki data volume (Podman volume for TSDB and index)
3. Grafana Loki datasource configuration
4. Log query panels in C11 dashboards

### 11.5 Metrics Directory Mount

The `/var/lib/tianer/metrics/` directory is:
- Created on the host by C01 `create-dirs.sh`
- Owned by `tianer:tianer` (mode 0755)
- Bind-mounted `:ro` into the Prometheus container
- Writable by host scripts and containers that need to write metrics (C05, C07, C01 metrics script, C02 metrics script)

For containers that need write access to this directory (C05 Ingest Bridge, C07 Deep Parser), an additional bind mount with `:rw` is used. Since Prometheus mounts `:ro`, there is no conflict — only Prometheus needs read-only access.

### 11.6 Dependencies

| Dependency | Required For | Version |
|-----------|-------------|---------|
| `promtool` | Config validation during deployment | 2.54.0+ (packaged with Prometheus container) |
| `jq` | Log parsing and validation in tests | 1.7+ (available in Debian Trixie) |
| `systemd-linger` | User-level timers for metrics scripts | Enabled by C01 |
| `stat` (coreutils) | File metadata queries | Any |

---

## Appendix A: Full Metrics Reference (Quick-Lookup)

### A.1 Shared Platform Metrics (tianer_ prefix)

| Metric | Type | Source | Labels |
|--------|------|--------|--------|
| `tianer_host_disk_usage_bytes` | gauge | C01 | `mountpoint` |
| `tianer_host_disk_total_bytes` | gauge | C01 | `mountpoint` |
| `tianer_host_disk_usage_ratio` | gauge | C01 | `mountpoint` |
| `tianer_host_memory_available_bytes` | gauge | C01 | — |
| `tianer_host_memory_total_bytes` | gauge | C01 | — |
| `tianer_host_cpu_temp_celsius` | gauge | C01 | — |
| `tianer_host_cpu_throttled` | gauge | C01 | `reason` |
| `tianer_usb_device_present` | gauge | C01 | `device` |
| `tianer_fifo_present` | gauge | C01 | `name` |
| `tianer_host_uptime_seconds` | gauge | C01 | — |
| `tianer_db_up` | gauge | C02 | — |
| `tianer_db_size_bytes` | gauge | C02 | — |
| `tianer_db_connections_active` | gauge | C02 | `state` |
| `tianer_db_connections_total` | gauge | C02 | — |
| `tianer_table_size_bytes` | gauge | C02 | `table` |
| `tianer_hypertable_chunks_total` | gauge | C02 | `hypertable` |
| `tianer_hypertable_compressed_chunks` | gauge | C02 | `hypertable` |
| `tianer_cagg_refresh_last_duration_ms` | gauge | C02 | `cagg` |
| `tianer_cagg_refresh_last_success` | gauge | C02 | `cagg` |
| `tianer_compression_job_last_success` | gauge | C02 | — |
| `tianer_retention_job_last_success` | gauge | C02 | — |
| `tianer_dead_tuples_total` | gauge | C02 | `table` |
| `tianer_wal_bytes_written_total` | counter | C02 | — |
| `tianer_db_migration_applied` | gauge | C02 | `migration` |
| `tianer_api_requests_total` | counter | C09 | `method`, `endpoint`, `status_code` |
| `tianer_api_request_duration_seconds` | histogram | C09 | `method`, `endpoint` |
| `tianer_api_requests_in_flight` | gauge | C09 | — |
| `tianer_api_auth_failures_total` | counter | C09 | `reason` |
| `tianer_api_db_pool_size` | gauge | C09 | — |
| `tianer_api_db_pool_available` | gauge | C09 | — |
| `tianer_api_db_query_duration_seconds` | histogram | C09 | `query_name` |
| `tianer_api_health_status` | gauge | C09 | `component` |

### A.2 Bluetooth Module Metrics (blesniff_ prefix)

| Metric | Type | Source | Labels |
|--------|------|--------|--------|
| `blesniff_sniffer_running` | gauge | C03 | `sniffer`, `sniffer_type` |
| `blesniff_sniffer_packets_captured_total` | counter | C03 | `sniffer`, `sniffer_type` |
| `blesniff_sniffer_channel` | gauge | C03 | `sniffer` |
| `blesniff_tshark_running` | gauge | C03 | `sniffer` |
| `blesniff_tshark_parse_errors_total` | counter | C03 | `sniffer` |
| `blesniff_heartbeat_age_seconds` | gauge | C03 | `sniffer` |
| `blesniff_ingest_packets_ingested_total` | counter | C05 | `sniffer` |
| `blesniff_ingest_packets_copied_total` | counter | C05 | `sniffer` |
| `blesniff_ingest_malformed_packets_total` | counter | C05 | `sniffer`, `reason` |
| `blesniff_ingest_latency_seconds` | histogram | C05 | `sniffer` |
| `blesniff_ingest_batch_size` | gauge | C05 | `sniffer` |
| `blesniff_ingest_batch_flush_duration_ms` | histogram | C05 | `sniffer` |
| `blesniff_ingest_batches_flushed_total` | counter | C05 | `sniffer`, `trigger` |
| `blesniff_ingest_db_connected` | gauge | C05 | `sniffer` |
| `blesniff_ingest_buffer_depth` | gauge | C05 | `sniffer` |
| `blesniff_ingest_reconnects_total` | counter | C05 | `sniffer` |
| `blesniff_ingest_uptime_seconds` | gauge | C05 | `sniffer` |
| `blesniff_gap_detected_total` | counter | C06 | `sniffer` |
| `blesniff_gap_backfill_rows_total` | counter | C06 | `sniffer` |
| `blesniff_gap_backfill_failures_total` | counter | C06 | `sniffer`, `reason` |
| `blesniff_gap_detection_duration_ms` | gauge | C06 | `sniffer` |
| `blesniff_gap_last_window_start_timestamp` | gauge | C06 | `sniffer` |
| `blesniff_gap_last_window_end_timestamp` | gauge | C06 | `sniffer` |
| `blesniff_gap_detector_runs_total` | counter | C06 | — |
| `blesniff_gap_detector_last_success` | gauge | C06 | — |
| `blesniff_deepparse_packets_processed_total` | counter | C07 | `sniffer` |
| `blesniff_deepparse_packets_valid_ble_total` | counter | C07 | `sniffer` |
| `blesniff_deepparse_parse_errors_total` | counter | C07 | `reason` |
| `blesniff_deepparse_crc_errors_total` | counter | C07 | `sniffer` |
| `blesniff_deepparse_jsonl_bytes_written_total` | counter | C07 | `sniffer` |
| `blesniff_deepparse_jsonl_records_written_total` | counter | C07 | `sniffer` |
| `blesniff_deepparse_pcap_files_completed` | counter | C07 | `sniffer` |
| `blesniff_deepparse_processing_duration_ms` | gauge | C07 | `sniffer`, `stage` |
| `blesniff_deepparse_memory_usage_bytes` | gauge | C07 | `sniffer` |
| `blesniff_deepparse_uptime_seconds` | gauge | C07 | `sniffer` |

---

## Appendix B: Alert Summary

| Alert Name | Severity | Component | Trigger |
|------------|----------|-----------|---------|
| `TianerHostDiskWarning` | Warning | C01 | Disk usage > 80% |
| `TianerHostDiskCritical` | Critical | C01 | Disk usage > 90% |
| `TianerHostThermalThrottle` | Warning | C01 | CPU throttling > 60s |
| `TianerUSBDeviceMissing` | Critical | C01 | USB device absent > 30s |
| `TianerHostMemoryWarning` | Warning | C01 | Memory usage > 85% |
| `TianerDBDown` | Critical | C02 | DB unreachable > 30s |
| `TianerDBConnectionHigh` | Warning | C02 | Connections > 80% max |
| `TianerCaggRefreshFailed` | Critical | C02 | Cagg refresh failed |
| `TianerIngestDBDisconnected` | Critical | C05 | Ingest disconnected from DB > 30s |
| `TianerIngestHighMalformed` | Critical | C05 | > 100 malformed packets/min |
| `TianerGapDetected` | Warning | C06 | New ingest gap found |
| `TianerBackfillFailed` | Critical | C06 | Backfill failures occurring |
| `TianerPrometheusScrapeFailure` | Critical | C13 | Prometheus cannot reach a scrape target |
| `TianerPromtailDroppingLogs` | Critical | C13 | Promtail dropping log entries |

## References

[1] Prometheus Authors. "Metric and Label Naming." https://prometheus.io/docs/practices/naming/, 2024. Prometheus best practices for metric naming conventions, base units, label usage, and suffix rules.

[2] Prometheus Authors. "Prometheus Exposition Formats." https://prometheus.io/docs/instrumenting/exposition_formats/, 2024. Text-based exposition format specification (version 0.0.4) underlying the OpenMetrics standard.

[3] OpenMetrics Project. "OpenMetrics Specification." https://github.com/prometheus/OpenMetrics, 2024. The format underlying Prometheus exposition; defines `# HELP`, `# TYPE`, metric naming, and label rules.

[4] systemd. "systemd-journald — Journal Service." https://www.freedesktop.org/software/systemd/man/latest/systemd-journald.service.html, 2024. Structured logging service used for container STDOUT/STDERR capture.

[5] Beyer, B., Jones, C., Petoff, J., Murphy, N.R. "Site Reliability Engineering, Chapter 6: Monitoring Distributed Systems." O'Reilly Media, 2016. https://sre.google/sre-book/monitoring-distributed-systems/. Observability design patterns for distributed systems: the four golden signals, alerting on symptoms vs. causes, and avoiding alert fatigue.

[6] Prometheus Authors. "Alerting Rules." https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/, 2024. Defining alerting rules, the `for` clause, `labels` and `annotations` fields, and templating.

[7] Prometheus Community. "node_exporter — Textfile Collector." https://github.com/prometheus/node_exporter#textfile-collector, 2024. Textfile collector pattern: directory of `*.prom` files, atomic write via `.tmp` rename, no timestamps.

[8] Grafana Labs. "Promtail — Pipeline Stages." https://grafana.com/docs/loki/latest/send-data/promtail/stages/, 2024. Promtail log collection agent: pipeline stages for regex, JSON extraction, label assignment, and timestamp parsing.

---
