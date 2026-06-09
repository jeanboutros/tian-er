# C11 — Grafana Dashboards

**Status:** Phase A Design — guides Phase B implementation
**Author:** Code Architect
**Date:** 2026-06-09
**Dependencies:** C02 (Database), C01 (Platform Infrastructure)
**Blocks:** C12 (Service Orchestration — final wiring)

---

## 1. Overview

### 1.1 Purpose

C11 Grafana Dashboards provides the **observability presentation layer** for the Tian'er Signal Intelligence Platform. It runs **Grafana 10.4+** in a standalone rootless Podman container on the `tianer-net` bridge network, reads PostgreSQL via the read-only `tianer_grafana` role, and serves four purpose-built dashboards for monitoring the v1 Bluetooth sensor module. Dashboards and datasources are deployed via Grafana's file-based provisioning system [1]. Authentication is **anonymous** (no login required) for the MVP phase because the dashboards are embedded in the Vue 3 frontend via iframe and only accessible on the LAN.

### 1.2 Scope

| In Scope | Out of Scope |
|----------|-------------|
| Grafana 10.4+ container deployment via Quadlet | Host OS package installation (C01) |
| Four provisioned dashboards with SQL panels | Grafana alerting (deferred to post-MVP per user decision) |
| `tianer_grafana` read-only PostgreSQL datasource | Alert rules, notification channels, contact points |
| Anonymous access with Viewer role [2] | User authentication, OAuth, LDAP (post-MVP) |
| BreachLab/cyberpunk dark theme (`xcyber360-theme`) | Custom plugin development |
| iframe embedding support (CSP + X-Frame-Options) [3] | Grafana Live streaming or real-time websocket panels |
| V07 `tianer-grafana-data` Podman volume for dashboards + config | Host file permissions (C01) |
| `grafana.ini` configuration and provisioning YAML | Grafana binary installation (container image) |

### 1.3 Boundaries

C11 owns the **Grafana presentation layer**: the running Grafana instance, the dashboard definitions, the datasource provisioning, and the anonymous-access configuration. It does **not** own the database schema (C02), the host package installation (C01), the Quadlet file deployment (C14), the frontend iframe integration (C10), or the secrets files (C01).

```
┌────────────────────────────────────────────────────────────┐
│                    C01 PLATFORM HOST                       │
│  (Podman runtime, secrets via EnvironmentFile,            │
│   tianer-net bridge, V07 volume)                          │
└────────────┬───────────────────┬───────────────────────────┘
             │                   │
             ▼                   ▼
┌────────────────────┐  ┌──────────────────────┐
│  C11 GRAFANA        │  │  C02 DATABASE        │
│  (standalone        │  │  (PostgreSQL 17 +    │
│   container)        │  │   TimescaleDB)       │
│                     │  │                      │
│  Port: 3000         │◄─│  tianer_grafana role │
│  Volume: V07 (:rw)  │  │  SELECT-only         │
│  Theme: cyberpunk   │  │                      │
│  Auth: anonymous    │  └──────────────────────┘
│  Embed: enabled     │
└────────┬────────────┘
         │
         │ iframe (anonymous, allow_embedding=true)
         ▼
┌────────────────────┐
│  C10 Frontend      │
│  (Vue 3 + Vite)    │
│  GrafanaPanel.vue  │
└────────────────────┘
```

### 1.4 Position in the System

C11 is a **Layer 3 shared platform component** in the build sequence (component-breakdown.md §4.1). It depends on C02 (Database) for the datasource and is built alongside C08 (ML Enrichment) and C10 (Frontend). It blocks C12 (Service Orchestration) only in the final wiring step — Grafana runs independently of the capture pipeline.

### 1.5 User Decisions Incorporated

| Decision | Source | Implementation |
|----------|--------|---------------|
| Anonymous Grafana access for MVP | User directive | `auth.anonymous enabled = true`, org role = Viewer [2] |
| Embedded iframe in Vue app | User directive + inception.md §8.11.2 | `allow_embedding = true`, `X-Frame-Options: sameorigin`, CSP `frame-ancestors 'self'` [3] |
| BreachLab/cyberpunk dark theme | User directive | `xcyber360-theme` plugin, set as default via `grafana.ini` `default_theme` |
| No alerts for MVP | User directive | No alert rules provisioned; no notification channels configured |
| Read-only DB via `tianer_grafana` | C02 §2.2, §5.1 | Datasource uses `tianer_grafana` role with SELECT-only privileges |
| Standalone container on `tianer-net` | component-breakdown.md §2.1a | Quadlet container, not in any pod |
| V07 `tianer-grafana-data` volume | storage-strategy.md | Podman volume for dashboards, plugins, and Grafana SQLite DB |

---

## 2. High-Level Architecture (HLA)

### 2.1 Container Deployment Architecture

Grafana runs as a **standalone container** — not part of any pod — on the `tianer-net` bridge network. It is the only container that accesses the `tianer-grafana-data` Podman volume (V07).

```
┌───────────────────────────────────────────────────────────────┐
│                   PODMAN (rootless, user tianer)              │
│                                                               │
│  ┌───────────────────────────────────────────────────────┐   │
│  │              tianer-net (bridge network, internal)     │   │
│  │                                                        │   │
│  │  ┌─────────────────────────────┐  ┌─────────────────┐ │   │
│  │  │  tianer-grafana             │  │ tianer-postgres │ │   │
│  │  │  (standalone container)     │  │                 │ │   │
│  │  │                             │  │                 │ │   │
│  │  │  Grafana 10.4+              │  │ Port: 5432      │ │   │
│  │  │  Port: 3000                 │  │                 │ │   │
│  │  │  Volume: V07 (:rw)          │  │                 │ │   │
│  │  │                             │  │                 │ │   │
│  │  │  Datasource: ───────────────┼──► tianer_grafana  │ │   │
│  │  │    tianer-postgres:5432     │  │   role           │ │   │
│  │  │                             │  │   SELECT-only    │ │   │
│  │  └──────────┬──────────────────┘  └─────────────────┘ │   │
│  └─────────────┼──────────────────────────────────────────┘   │
│                │                                               │
│  Host port:    │                                               │
│  127.0.0.1:3000│ (loopback only)                               │
│                │                                               │
│  ┌─────────────┴──────────────────────────────────────┐       │
│  │  C10 Frontend (Vue 3)                              │       │
│  │  GrafanaPanel.vue: <iframe src="/grafana/...">     │       │
│  │  (reverse-proxied through C09 API)                 │       │
│  └────────────────────────────────────────────────────┘       │
└───────────────────────────────────────────────────────────────┘
```

**Key deployment properties:**

| Property | Value |
|----------|-------|
| Container name | `tianer-grafana` |
| Image | `grafana/grafana-oss:10.4@sha256:...` (digest-pinned) |
| Network | `tianer-net` (bridge, created by C14 Quadlet `.network` file) |
| Published port | `127.0.0.1:3000:3000` (loopback only on host; frontend accesses via API reverse-proxy) |
| Volume | V07 `tianer-grafana-data` Podman volume, mounted at `/var/lib/grafana` |
| Config | `grafana.ini` and provisioning files bind-mounted from V01 (`/etc/tianer/grafana/`) |
| Secrets | `EnvironmentFile=/etc/tianer/secrets/grafana_db_password.env` |
| Capabilities | `--cap-drop ALL` (no elevated privileges) |
| Image policy | Digest-pinned (`@sha256:`), no `:latest` tags |
| Theme | BreachLab xcyber360, `default_theme = dark` |
| Auth | Anonymous, org role = Viewer |

### 2.2 Data Flow

```
                    ┌──────────────────────────────────┐
                    │        C02 PostgreSQL 17          │
                    │        + TimescaleDB 2.23         │
                    │                                   │
                    │  bluetooth.raw_packets            │
                    │  bluetooth.device_summary         │
                    │  bluetooth.device_enrichment      │
                    │  bluetooth.ingest_gaps            │
                    │  bluetooth.sniffer_heartbeat      │
                    │  bluetooth.sniffers               │
                    │  bluetooth.device_5min_buckets    │
                    │  (continuous aggregate)           │
                    └──────────────┬────────────────────┘
                                   │
                        SQL SELECT (tianer_grafana role)
                        scram-sha-256
                        tianer-net bridge (tianer-postgres:5432)
                                   │
                    ┌──────────────▼────────────────────┐
                    │        C11 GRAFANA                 │
                    │                                    │
                    │  ┌─────────────────────────────┐  │
                    │  │  TimescaleDB Datasource      │  │
                    │  │  (provisioned at startup)    │  │
                    │  └──────────┬──────────────────┘  │
                    │             │                      │
                    │  ┌──────────▼──────────────────┐  │
                    │  │  4 Dashboard JSONs           │  │
                    │  │  (provisioned at startup)    │  │
                    │  │                              │  │
                    │  │  1. live-metrics             │  │
                    │  │  2. device-explorer          │  │
                    │  │  3. per-device-drilldown     │  │
                    │  │     ($mac variable)          │  │
                    │  │  4. pipeline-health          │  │
                    │  └──────────┬──────────────────┘  │
                    │             │                      │
                    │  ┌──────────▼──────────────────┐  │
                    │  │  HTTP Server :3000           │  │
                    │  │  Anonymous access            │  │
                    │  │  allow_embedding = true      │  │
                    │  │  X-Frame-Options: sameorigin │  │
                    │  │  CSP: frame-ancestors 'self' │  │
                    │  └──────────────────────────────┘  │
                    └────────────────┬───────────────────┘
                                     │
                        HTTP (anonymous, Viewer role)
                        localhost:3000
                                     │
                    ┌────────────────▼───────────────────┐
                    │  C10 Frontend (Vue 3)              │
                    │  GrafanaPanel.vue                  │
                    │  <iframe src="/grafana/...">       │
                    └────────────────────────────────────┘
```

### 2.3 Provisioning Chain

Grafana loads configuration in this order at startup [1]:

```
1. grafana.ini        → Server settings (HTTP port, domain, anonymous auth, embedding, theme)
2. datasources/*.yaml → TimescaleDB datasource (connection string, credentials, TLS settings)
3. dashboards/*.yaml  → Dashboard provider (scans directory for JSON files)
4. dashboards/*.json  → Individual dashboard definitions (4 files)
```

All provisioning files are mounted read-only from V01 (`/etc/tianer/grafana/`). The Grafana SQLite database (user preferences, dashboard stars, last-viewed state) is stored on V07 at `/var/lib/grafana/grafana.db` and persists across container restarts.

---

## 3. Data Model

### 3.1 Consumed Database Objects

C11 does not own any database tables. It reads from objects owned by C02 (Database). All tables referenced below live in the `bluetooth` schema.

| Table/View | Schema | Purpose in Dashboards |
|------------|--------|----------------------|
| `raw_packets` | Hyperable | Packet-level counts, RSSI histograms, per-sniffer throughput (live-metrics, per-device-drilldown) |
| `device_summary` | Table | Device table rows, residency counts, new/total/lost device statistics (device-explorer, live-metrics) |
| `device_enrichment` | Table | Device names, manufacturer IDs, service UUIDs (per-device-drilldown enrichment panels) |
| `device_5min_buckets` | Continuous aggregate | Timeline charts, pre-aggregated per-MAC counts and RSSI trends (per-device-drilldown) |
| `ingest_gaps` | Table | Gap count, status distribution, backfill throughput (pipeline-health) |
| `sniffer_heartbeat` | Table | Sniffer liveness, last heartbeat time (pipeline-health) |
| `sniffers` | Table | Sniffer names, types, enabled state (pipeline-health) |

### 3.2 Datasource Configuration Data Model

The datasource YAML is the **single source of truth** for how Grafana connects to PostgreSQL:

```yaml
# grafana/provisioning/datasources/timescaledb.yaml
apiVersion: 1
datasources:
  - name: TimescaleDB
    type: postgres
    access: proxy
    url: tianer-postgres:5432
    database: tianer
    user: tianer_grafana
    secureJsonData:
      password: ${GRAFANA_DB_PASSWORD}
    jsonData:
      sslmode: disable
      postgresVersion: 1700
      timescaledb: true
      maxOpenConns: 5
      maxIdleConns: 2
      connMaxLifetime: 300
    isDefault: true
    editable: false
```

**Key fields and their rationale:**

| Field | Value | Rationale |
|-------|-------|-----------|
| `access` | `proxy` | Grafana server proxies queries to DB; no browser→DB direct connection needed |
| `url` | `tianer-postgres:5432` | Container name on `tianer-net` bridge; DNS resolution handled by Podman |
| `sslmode` | `disable` | Internal bridge network; TLS not needed between containers on same host |
| `postgresVersion` | `1700` | Grafana internal version code for PostgreSQL 17 (17 × 100) |
| `timescaledb` | `true` | Enables TimescaleDB-specific query builder macros in Grafana [4] |
| `maxOpenConns` | `5` | Conservative connection pool; ~30 total DB connections with 5 from Grafana |
| `editable` | `false` | Prevents UI changes to datasource; all config is in version-controlled provisioning files |

### 3.3 Dashboard Provisioning Data Model

```yaml
# grafana/provisioning/dashboards/default.yaml
apiVersion: 1
providers:
  - name: tianer
    orgId: 1
    folder: 'Tian'er'
    type: file
    disableDeletion: true
    updateIntervalSeconds: 10
    allowUiUpdates: false
    options:
      path: /etc/grafana/provisioning/dashboards
      foldersFromFilesStructure: false
```

| Field | Value | Rationale |
|-------|-------|-----------|
| `folder` | `'Tian'er'` | All dashboards grouped under a project-named folder |
| `disableDeletion` | `true` | Deleting a JSON file from disk does not delete the dashboard in Grafana (prevents accidental wipe) |
| `updateIntervalSeconds` | `10` | Rescans the dashboard directory every 10 seconds for changes |
| `allowUiUpdates` | `false` | Dashboard edits through the UI are not persisted (all dashboards are version-controlled as JSON files) |

---

## 4. Low-Level Architecture (LLA)

### 4.1 Dashboard Inventory

| # | Dashboard | File | Variables | Purpose |
|---|-----------|------|-----------|---------|
| 1 | **Live Metrics** | `live-metrics.json` | None | Real-time overview: packets/sec per sniffer, active devices, RSSI distribution, new devices today |
| 2 | **Device Explorer** | `device-explorer.json` | None | Sortable/filterable table of all observed devices with residency, counts, enrichment data |
| 3 | **Per-Device Drilldown** | `per-device-drilldown.json` | `$mac` (text, multi-value off) | Deep-dive on a single device: timeline, RSSI trend, channel distribution, enrichment details |
| 4 | **Pipeline Health** | `pipeline-health.json` | `$sniffer` (query, multi-value on) | Ingest rate, gap status, backfill progress, sniffer liveness, DB metrics |

### 4.2 Dashboard 1 — Live Metrics

**Purpose:** At-a-glance operational view. Displays real-time packet throughput, device activity, and signal distribution. Auto-refreshes every 30 seconds.

**Refresh interval:** 30s (configurable via dashboard time picker)

**Panels:**

#### Panel 1.1 — Packets Per Second (per sniffer)

```sql
SELECT
    $__timeGroupAlias(ts, '1m'),
    sniffer_id::text AS metric,
    COUNT(*) / 60.0 AS "pps"
FROM bluetooth.raw_packets
WHERE $__timeFilter(ts)
GROUP BY 1, 2
ORDER BY 1;
```

- **Visualization:** Time series (line chart)
- **Legend:** `{{sniffer_id}}`
- **Y-axis:** Packets per second (linear)

#### Panel 1.2 — Active Devices (last hour)

```sql
SELECT
    $__timeGroupAlias(last_seen, '5m'),
    COUNT(DISTINCT mac_address) AS "active_devices"
FROM bluetooth.device_summary
WHERE last_seen > NOW() - INTERVAL '1 hour'
  AND $__timeFilter(last_seen)
GROUP BY 1
ORDER BY 1;
```

- **Visualization:** Stat (single value showing current count) + sparkline
- **Value:** Current active device count

#### Panel 1.3 — Device Residency Distribution

```sql
SELECT
    residency_class,
    COUNT(*) AS "count"
FROM bluetooth.device_summary
WHERE residency_class IS NOT NULL
GROUP BY 1
ORDER BY 2 DESC;
```

- **Visualization:** Pie chart (or bar gauge)
- **Legend:** `resident`, `frequent`, `transient`, `new`, `lost`

#### Panel 1.4 — RSSI Distribution (histogram)

```sql
SELECT
    rssi,
    COUNT(*) AS "count"
FROM bluetooth.raw_packets
WHERE $__timeFilter(ts)
  AND rssi IS NOT NULL
  AND rssi BETWEEN -100 AND 0
GROUP BY 1
ORDER BY 1;
```

- **Visualization:** Bar chart (histogram)
- **X-axis:** RSSI (dBm), -100 to 0
- **Y-axis:** Packet count

#### Panel 1.5 — New Devices Today

```sql
SELECT COUNT(*) AS "new_today"
FROM bluetooth.device_summary
WHERE first_seen >= CURRENT_DATE;
```

- **Visualization:** Stat
- **Value:** Integer count

#### Panel 1.6 — Total Packets Ingested (24h)

```sql
SELECT
    sniffer_id::text AS metric,
    COUNT(*) AS "total"
FROM bluetooth.raw_packets
WHERE ts > NOW() - INTERVAL '24 hours'
GROUP BY 1
ORDER BY 1;
```

- **Visualization:** Bar chart
- **Legend:** `{{sniffer_id}}`

### 4.3 Dashboard 2 — Device Explorer

**Purpose:** Interactive device table for browsing and filtering all observed Bluetooth devices. Replaces (or complements) the Vue frontend device list for power users and operators.

**Refresh interval:** 1m

**Panels:**

#### Panel 2.1 — Device Table

```sql
SELECT
    encode(mac_address, 'hex') AS "MAC",
    CASE address_type
        WHEN 0 THEN 'public'
        WHEN 1 THEN 'random'
        ELSE 'unknown'
    END AS "Address Type",
    first_seen AS "First Seen",
    last_seen AS "Last Seen",
    total_count AS "Observations",
    distinct_days AS "Distinct Days",
    residency_class AS "Residency",
    last_seen > NOW() - INTERVAL '1 hour' AS "Active"
FROM bluetooth.device_summary
WHERE
    ($address_type IS NULL OR address_type = CAST($address_type AS SMALLINT))
    AND ($residency IS NULL OR residency_class = $residency)
    AND ($min_observations IS NULL OR total_count >= CAST($min_observations AS BIGINT))
ORDER BY last_seen DESC
LIMIT 1000;
```

- **Visualization:** Table
- **Column styles:**
  - `Active`: Boolean shown as green/red indicator
  - `Residency`: Color-coded cell (resident=green, frequent=blue, transient=yellow, new=cyan, lost=red)
- **Variables (filter dropdowns):**
  - `$address_type`: `public`, `random` (with "All" option)
  - `$residency`: `resident`, `frequent`, `transient`, `new`, `lost` (with "All" option)
  - `$min_observations`: Integer input, default 0

#### Panel 2.2 — Device Count by Residency (summary stat row)

```sql
SELECT
    residency_class,
    COUNT(*) AS "count"
FROM bluetooth.device_summary
WHERE residency_class IS NOT NULL
GROUP BY 1;
```

- **Visualization:** Stat row (one stat per residency class)

### 4.4 Dashboard 3 — Per-Device Drilldown

**Purpose:** Deep-dive view for a single device. Accessed by clicking a MAC address row in the Device Explorer table (via dashboard link) or by setting the `$mac` variable manually.

**Variable:** `$mac` — text input, no multi-value, no "All" option. Default: most recently seen device MAC.

**Panels:**

#### Panel 3.1 — Device Identity Card

```sql
SELECT
    encode(mac_address, 'hex') AS "MAC",
    CASE address_type
        WHEN 0 THEN 'Public'
        WHEN 1 THEN 'Random'
        ELSE 'Unknown'
    END AS "Address Type",
    first_seen AS "First Seen",
    last_seen AS "Last Seen",
    total_count AS "Total Observations",
    distinct_days AS "Distinct Days Observed",
    residency_class AS "Residency Class"
FROM bluetooth.device_summary
WHERE mac_address = decode(replace($mac, ':', ''), 'hex');
```

- **Visualization:** Table (single-row identity card)
- **Note:** `$mac` is in colon-hex format (e.g., `aa:bb:cc:dd:ee:ff`). The query strips colons and decodes from hex to `BYTEA` for the comparison.

#### Panel 3.2 — Observation Timeline (5-min buckets)

```sql
SELECT
    bucket AS "time",
    packet_count,
    avg_rssi,
    min_rssi,
    max_rssi
FROM bluetooth.device_5min_buckets
WHERE mac_address = decode(replace($mac, ':', ''), 'hex')
  AND $__timeFilter(bucket)
ORDER BY bucket;
```

- **Visualization:** Time series (dual Y-axis)
  - Left Y-axis: packet_count (bar)
  - Right Y-axis: avg_rssi (line)

#### Panel 3.3 — RSSI Trend (raw packets)

```sql
SELECT
    ts AS "time",
    rssi
FROM bluetooth.raw_packets
WHERE mac_address = decode(replace($mac, ':', ''), 'hex')
  AND $__timeFilter(ts)
  AND rssi IS NOT NULL
ORDER BY ts;
```

- **Visualization:** Scatter plot (time vs RSSI)
- **Point size:** 2 (small dots for scatter density)

#### Panel 3.4 — Channel Distribution

```sql
SELECT
    channel::text,
    COUNT(*) AS "count"
FROM bluetooth.raw_packets
WHERE mac_address = decode(replace($mac, ':', ''), 'hex')
  AND $__timeFilter(ts)
  AND channel IS NOT NULL
GROUP BY 1
ORDER BY 2 DESC;
```

- **Visualization:** Pie chart
- **Legend:** Channel 37, 38, 39

#### Panel 3.5 — Observations by Sniffer

```sql
SELECT
    s.name AS "Sniffer",
    COUNT(*) AS "Observations"
FROM bluetooth.raw_packets rp
JOIN bluetooth.sniffers s ON rp.sniffer_id = s.sniffer_id
WHERE rp.mac_address = decode(replace($mac, ':', ''), 'hex')
  AND $__timeFilter(rp.ts)
GROUP BY s.name
ORDER BY 2 DESC;
```

- **Visualization:** Bar chart

#### Panel 3.6 — Enrichment Data

```sql
SELECT
    observed_ts AS "Timestamp",
    local_name AS "Device Name",
    array_to_string(service_uuids_16, ', ') AS "16-bit Service UUIDs",
    array_to_string(service_uuids_128, ', ') AS "128-bit Service UUIDs",
    manufacturer_id AS "Mfr ID",
    encode(manufacturer_data, 'hex') AS "Mfr Data",
    tx_power AS "TX Power (dBm)",
    flags AS "Flags"
FROM bluetooth.device_enrichment
WHERE mac_address = decode(replace($mac, ':', ''), 'hex')
ORDER BY observed_ts DESC
LIMIT 100;
```

- **Visualization:** Table

#### Dashboard Link from Device Explorer

Each MAC address row in the Device Explorer table contains a **dashboard link** that navigates to the Per-Device Drilldown dashboard with `$mac` pre-populated. The link is configured in the Device Explorer dashboard JSON:

```json
{
  "links": [
    {
      "asDropdown": false,
      "icon": "external link",
      "targetBlank": false,
      "title": "Device Drilldown",
      "url": "/d/per-device-drilldown?var-mac=${__value.raw}",
      "includeVars": false,
      "keepTime": true
    }
  ]
}
```

### 4.5 Dashboard 4 — Pipeline Health

**Purpose:** Operational health monitoring for the ingest pipeline. Shows ingest throughput, gap detection status, sniffer liveness, and DB metrics.

**Refresh interval:** 30s

**Variable:** `$sniffer` — query-based multi-select from `bluetooth.sniffers` table (default: all).

```sql
SELECT name, sniffer_id::text
FROM bluetooth.sniffers
WHERE enabled = true
ORDER BY sniffer_id;
```

**Panels:**

#### Panel 4.1 — Ingest Rate (packets/sec, per sniffer)

```sql
SELECT
    $__timeGroupAlias(ts, '1m'),
    sniffer_id::text AS metric,
    COUNT(*) / 60.0 AS "pps"
FROM bluetooth.raw_packets
WHERE $__timeFilter(ts)
  AND ($sniffer = 'All' OR sniffer_id::text = ANY(string_to_array($sniffer, ',')))
GROUP BY 1, 2
ORDER BY 1;
```

- **Visualization:** Time series
- **Note:** Uses Grafana multi-value variable expansion. `$sniffer = 'All'` bypasses the filter.

#### Panel 4.2 — Gap Status

```sql
SELECT
    status,
    COUNT(*) AS "count"
FROM bluetooth.ingest_gaps
WHERE ($sniffer = 'All' OR sniffer_id::text = ANY(string_to_array($sniffer, ',')))
GROUP BY 1
ORDER BY 2 DESC;
```

- **Visualization:** Bar gauge (or stat row)
- **Status colors:** `open`=yellow, `backfilling`=blue, `closed`=green, `failed`=red

#### Panel 4.3 — Gap Timeline

```sql
SELECT
    bucket_start AS "time",
    status,
    rows_backfilled
FROM bluetooth.ingest_gaps
WHERE $__timeFilter(bucket_start)
  AND ($sniffer = 'All' OR sniffer_id::text = ANY(string_to_array($sniffer, ',')))
ORDER BY bucket_start DESC
LIMIT 100;
```

- **Visualization:** State timeline (or table)
- **Color by:** status

#### Panel 4.4 — Sniffer Heartbeat Liveness

```sql
SELECT
    s.name AS "Sniffer",
    s.type AS "Type",
    h.ts AS "Last Heartbeat",
    EXTRACT(EPOCH FROM (NOW() - h.ts))::int AS "Seconds Since Heartbeat",
    CASE
        WHEN (NOW() - h.ts) > INTERVAL '60 seconds' THEN 'STALE'
        ELSE 'OK'
    END AS "Status"
FROM bluetooth.sniffers s
LEFT JOIN bluetooth.sniffer_heartbeat h ON s.sniffer_id = h.sniffer_id
WHERE s.enabled = true
  AND ($sniffer = 'All' OR s.sniffer_id::text = ANY(string_to_array($sniffer, ',')))
ORDER BY s.sniffer_id;
```

- **Visualization:** Table
- **Column styles:**
  - `Seconds Since Heartbeat`: Color gradient (green < 30s, yellow 30-60s, red > 60s)
  - `Status`: Red background if STALE

#### Panel 4.5 — Backfill Throughput (rows backfilled per gap)

```sql
SELECT
    bucket_start AS "time",
    sniffer_id::text AS "Sniffer",
    rows_backfilled
FROM bluetooth.ingest_gaps
WHERE status = 'closed'
  AND rows_backfilled IS NOT NULL
  AND $__timeFilter(bucket_start)
  AND ($sniffer = 'All' OR sniffer_id::text = ANY(string_to_array($sniffer, ',')))
ORDER BY bucket_start DESC
LIMIT 50;
```

- **Visualization:** Bar chart
- **Y-axis:** Rows backfilled

#### Panel 4.6 — DB Connection Pool

```sql
SELECT
    state,
    COUNT(*) AS "connections"
FROM pg_stat_activity
WHERE datname = 'tianer'
  AND usename = 'tianer_grafana'
GROUP BY state;
```

- **Visualization:** Stat row
- **Values:** `active`, `idle`, `idle in transaction`

#### Panel 4.7 — DB Size

```sql
SELECT
    pg_size_pretty(pg_database_size('tianer')) AS "Database Size",
    pg_size_pretty(pg_total_relation_size('bluetooth.raw_packets')) AS "raw_packets Size",
    pg_size_pretty(pg_total_relation_size('bluetooth.device_summary')) AS "device_summary Size"
```

- **Visualization:** Table (single row)

### 4.6 Dashboard JSON Structure

Each dashboard JSON file follows the standard Grafana dashboard model [5] (version `"schemaVersion": 39` for Grafana 10.4). Key top-level fields:

```json
{
  "annotations": { "list": [] },
  "editable": false,
  "fiscalYearStartMonth": 0,
  "graphTooltip": 0,
  "id": null,
  "links": [],
  "panels": [],
  "refresh": "30s",
  "schemaVersion": 39,
  "tags": ["tianer", "bluetooth"],
  "templating": { "list": [] },
  "time": { "from": "now-6h", "to": "now" },
  "timepicker": {},
  "timezone": "browser",
  "title": "Live Metrics",
  "uid": "tianer-live-metrics",
  "version": 0
}
```

**UID naming convention:** `tianer-<dashboard-slug>` (e.g., `tianer-live-metrics`, `tianer-device-explorer`, `tianer-per-device-drilldown`, `tianer-pipeline-health`).

**Dashboard UID stability:** UIDs are stable across deploys because they are hardcoded in the JSON files (not auto-generated). This ensures that frontend iframe URLs and dashboard links are permanent.

### 4.7 Dashboard Refresh Strategy

| Dashboard | Default Refresh | Rationale |
|-----------|----------------|-----------|
| Live Metrics | 30s | Operational view; needs near-real-time metrics |
| Device Explorer | 1m | Device summary table; 1-minute refresh is sufficient for device discovery |
| Per-Device Drilldown | 1m | Device detail; user selects time range manually |
| Pipeline Health | 30s | Operational monitoring; gap detection and ingest throughput need freshness |

All refresh intervals are configurable via the dashboard time picker. Users can set per-session overrides without modifying the JSON files.

---

## 5. Inter-Component Contracts

### 5.1 Contract DB-READ: Grafana → PostgreSQL

C11 consumes the `DB-READ` contract defined in C02 §5.1. The full contract is reproduced here for completeness.

| Property | Value |
|----------|-------|
| **Contract ID** | `DB-READ` |
| **From** | C11 Grafana Dashboards |
| **To** | C02 Database (PostgreSQL 17 + TimescaleDB 2.23) |
| **Role** | `tianer_grafana` |
| **Privileges** | CONNECT on `tianer` database, USAGE on `bluetooth` schema, SELECT on all tables |
| **Auth** | scram-sha-256 password from `/etc/tianer/secrets/grafana_db_password` |
| **Connection** | TCP to `tianer-postgres:5432` on `tianer-net` bridge |
| **Pool** | Grafana built-in PostgreSQL connection pool: max 5 open, max 2 idle, lifetime 300s |
| **Query pattern** | Read-only SELECT queries for dashboard panels. Mostly aggregate queries against continuous aggregates (`device_5min_buckets`) and `device_summary`. |
| **SLA** | Panel queries must complete within 10 seconds (Grafana default query timeout). |
| **Parameterization** | All queries use Grafana's native template variable substitution (`$mac`, `$__timeFilter`, `$sniffer`) which is parameterized by the Grafana server — no string concatenation. |

### 5.2 Contract EMBED: Grafana → Frontend

C11 serves dashboards that are embedded in C10's Vue 3 frontend via iframe. This contract is defined here and consumed by C10.

| Property | Value |
|----------|-------|
| **Contract ID** | `EMBED` |
| **From** | C11 Grafana Dashboards |
| **To** | C10 Frontend (Vue 3) |
| **Embed method** | `<iframe>` element pointing to `http://tianer-grafana:3000/d/<dashboard-uid>` |
| **Auth** | Anonymous (no credentials required in iframe) |
| **Headers** | `X-Frame-Options: sameorigin` (allows iframe from same origin only); CSP `frame-ancestors 'self'` |
| **Reverse proxy** | C09 REST API proxies `/grafana/*` → `http://tianer-grafana:3000` to ensure same-origin embedding |
| **Supported dashboards** | All four (live-metrics, device-explorer, per-device-drilldown, pipeline-health) |
| **Variable passthrough** | `$mac` and `$sniffer` can be set via URL query parameters in the iframe `src` |
| **Theme consistency** | Cyberpunk dark theme matches the Vue frontend's dark mode |
| **SLA** | Dashboard iframe loads within 3 seconds on LAN |

**C09 API reverse-proxy configuration (reference for C09 implementation):**

The C09 FastAPI server proxies `/grafana/*` requests to the Grafana container. This ensures the iframe appears same-origin with the Vue app, satisfying the `X-Frame-Options: sameorigin` requirement.

```
Browser                      C09 API (:8080)                C11 Grafana (:3000)
  │                              │                               │
  │  GET /grafana/d/tianer-live  │                               │
  ├─────────────────────────────►│                               │
  │                              │  GET /d/tianer-live-metrics   │
  │                              ├──────────────────────────────►│
  │                              │                               │
  │                              │  HTML + JS + CSS              │
  │                              │◄──────────────────────────────┤
  │  HTML + JS + CSS             │                               │
  │◄─────────────────────────────┤                               │
  │                              │                               │
  │  (Grafana JS loads in iframe, makes API calls via C09 proxy)│
  │  GET /grafana/api/datasources/proxy/...                     │
  ├─────────────────────────────►│                               │
  │                              │  GET /api/datasources/proxy/..│
  │                              ├──────────────────────────────►│
```

**Note to C09 implementer:** The reverse proxy must strip the `/grafana` prefix and forward to `http://tianer-grafana:3000`. It must handle both the dashboard HTML pages and the Grafana API calls (datasource proxy, search, annotations).

---

## 6. Failure Modes & Recovery

### 6.1 Failure Catalog

| # | Failure Mode | Detection | Impact | Recovery |
|---|-------------|-----------|--------|----------|
| **F-C11-1** | **Grafana container crashes** | Podman detects exit code ≠ 0. `systemctl --user is-failed tianer-grafana`. Frontend iframe shows connection error. | Dashboards unavailable (in iframe and at `:3000`). No data loss — DB and PCAP unaffected. Backend pipeline continues normally. | Podman auto-restarts via systemd `Restart=on-failure`, `RestartSec=5s`. Grafana SQLite DB (V07) preserves user preferences. Cold start: ~3 seconds on CM5. |
| **F-C11-2** | **Database unreachable (PostgreSQL down or network issue)** | Grafana datasource health check fails. Dashboards show "Database connection error" in panels. | All dashboards show errors. No data loss — raw data in DB and PCAP unaffected. | Grafana retries queries on next dashboard refresh (automatic). When DB recovers, panels re-render on next refresh cycle. No manual intervention needed. |
| **F-C11-3** | **Slow queries (query timeout > 10s)** | Grafana query timeout. Panels show "Query timed out" error. `pg_stat_statements` shows slow queries. | Affected panels show no data. Other panels on the same dashboard may still render. | Tune the query (add indexes, use continuous aggregates). Reduce time range. The continuous aggregate `device_5min_buckets` is designed to prevent slow queries on `raw_packets`. |
| **F-C11-4** | **Provisioning file corrupted** | Grafana fails to start or dashboards don't appear. Log shows "Failed to provision datasource" or "Failed to read dashboard JSON". | Dashboard(s) missing or datasource offline. | Restore provisioning files from Git (`grafana/provisioning/`). Files are bind-mounted read-only from V01 — they cannot be corrupted by Grafana itself. Rebuild container image if necessary. |
| **F-C11-5** | **V07 volume full** | `tianer_host_disk_usage_percent > 80%`. Grafana SQLite DB cannot write. Log shows "database is full" or "disk I/O error". | Dashboard preferences and stars not saved. Dashboards still render (reads are unaffected). | Grafana SQLite DB is small (< 50 MB typical). If V07 fills, it likely means another volume has consumed the host disk. Resolve the disk-full issue at C01 host level. Grafana gracefully degrades (dashboards load from provisioning files, only user prefs lost). |
| **F-C11-6** | **C09 reverse-proxy down** | Frontend iframe shows connection refused. Direct access at `http://<host>:3000` still works. | Dashboards unavailable through Vue frontend. Direct Grafana access still functional for operators. | Restart C09 API. Grafana itself is unaffected. No data loss. |
| **F-C11-7** | **Anonymous access misconfigured** | Grafana shows login screen instead of dashboards. Iframe shows Grafana login page. | Dashboards require login — unusable for MVP anonymous mode. | Verify `grafana.ini` has `[auth.anonymous] enabled = true` and `org_role = Viewer`. Restart Grafana container after fix. Provisioning files are on V01 (immutable from Grafana's perspective), so misconfiguration is a deployment issue, not a runtime corruption. |
| **F-C11-8** | **Plugin (xcyber360 theme) missing or incompatible** | Grafana starts but theme is not applied. Panels render with default Grafana light/dark theme. | Cosmetic issue. Dashboards function normally. | Verify the `xcyber360-theme` plugin is installed in the container image. If the plugin version is incompatible with Grafana 10.4, fall back to Grafana's built-in dark theme (`default_theme = dark`). Update the Dockerfile to pin a compatible plugin version. |

### 6.2 Recovery Service Dependency

| Recovery Agent | Failure It Handles | Is It Available After Reboot? |
|---------------|-------------------|-------------------------------|
| Podman auto-restart (systemd) | Container crash (F-C11-1) | Yes — systemd `Restart=on-failure` |
| Grafana built-in retry | DB query failures (F-C11-2) | Yes — automatic on next refresh cycle |
| C09 API restart (systemd) | Reverse-proxy down (F-C11-6) | Yes — systemd `Restart=on-failure` |
| Operator (manual) | Slow queries (F-C11-3), provisioning corruption (F-C11-4), misconfiguration (F-C11-7) | N/A — manual intervention |
| Git restore | Provisioning corruption (F-C11-4) | Manual — `git checkout grafana/` |

### 6.3 Graceful Degradation (PF-4)

Grafana is a **read-only consumer** of the database. When it fails, the capture pipeline and PCAP archive continue unaffected. This is a deliberate architectural choice:

| What fails | What still works |
|-----------|-----------------|
| Grafana container crashes | Sniffer capture (C03), PCAP rotation (C04), ingest bridge (C05), REST API (C09), frontend device list (C10, reads from API not Grafana) |
| DB unreachable from Grafana | Same as above. Grafana shows errors but platform is healthy. |
| C09 reverse-proxy down | Direct Grafana access at `http://<host>:3000`. Frontend device list via API (if API is up). |

**Note on blast radius (PF-6):** Grafana's anonymous read-only access and its position as a downstream consumer mean that even if Grafana is compromised (e.g., someone accesses it on the LAN), the attacker can only read pre-aggregated dashboard data via `tianer_grafana` (SELECT-only). They cannot modify data, drop tables, or escalate to other containers.

---

## 7. Observability

### 7.1 Grafana Self-Monitoring

Grafana exposes its own metrics in Prometheus format at `/metrics` (Grafana 10.4 default). These are scraped by C13 (Observability).

| Metric | Description | Alert Threshold (Post-MVP) |
|--------|-------------|---------------------------|
| `grafana_http_request_duration_seconds` | HTTP request latency histogram | P95 > 5s |
| `grafana_http_request_duration_seconds_count` | Total HTTP requests | — |
| `grafana_http_requests_in_flight` | Active concurrent requests | > 20 |
| `grafana_datasource_errors_total` | Datasource query errors (counter) | Rate > 0 for 5 min → **warning** |
| `grafana_datasource_request_duration_seconds` | Datasource query latency | P95 > 10s (timeout threshold) |
| `grafana_dashboard_views_total` | Dashboard view count | — |
| `grafana_alerting_active_alerts` | Active alerts (0 for MVP) | — (no alert rules defined for MVP) |

### 7.2 Dashboard-Level Health

The Pipeline Health dashboard (§4.5) serves as Grafana's **self-service health check**. An operator can open this dashboard and immediately see:

- Ingest rate (is data flowing?)
- Gap status (are there unaddressed gaps?)
- Sniffer heartbeat liveness (are sniffers alive?)
- DB connection pool (is DB healthy from Grafana's perspective?)

### 7.3 Structured Logging

Grafana container logs are captured by Podman and written to journald. Logs include:

- Startup messages (provisioning status, datasource health check, theme loading)
- Datasource connection errors
- Query timeouts
- Dashboard provisioning failures

Logs follow the standard Tian'er structured log format:

```
TIANER | {"ts":"2026-06-09T14:30:00Z","level":"INFO","component":"grafana","msg":"HTTP Server Listen","address":"[::]:3000","protocol":"http","subUrl":"","socket":""}
TIANER | {"ts":"2026-06-09T14:30:01Z","level":"INFO","component":"grafana","msg":"Datasource provisioning started","path":"/etc/grafana/provisioning/datasources"}
TIANER | {"ts":"2026-06-09T14:30:02Z","level":"ERROR","component":"grafana","msg":"Datasource health check failed","datasource":"TimescaleDB","error":"connection refused"}
```

### 7.4 Health Check

The C09 API `/api/health` endpoint reports Grafana status:

```json
{
    "status": "ok",
    "grafana": "ok",
    "grafana_url": "http://tianer-grafana:3000"
}
```

If Grafana is unreachable (container down), `grafana` is `"error"` and `status` is `"degraded"`. The platform remains operational — only dashboards are unavailable.

---

## 8. Security Considerations

### 8.1 Network Isolation

Grafana listens on port 3000 on **all interfaces inside the container**. The container's port 3000 is published to `127.0.0.1:3000` on the host via the Quadlet `PublishPort` directive. The C10 Frontend accesses Grafana through the C09 API reverse-proxy at `/grafana/*` — it never connects directly to port 3000 from the browser.

```
┌─────────────────────────────────────────────────────────────┐
│                      HOST (CM5)                              │
│                                                             │
│  127.0.0.1:3000 ← host loopback only (operator direct       │
│       ↑            access for debugging)                    │
│       │                                                     │
│       │ Podman port forwarding                              │
│       │                                                     │
│  ┌────┴──────────────────────────────────────┐              │
│  │  tianer-net (internal bridge, no          │              │
│  │  external route)                          │              │
│  │                                           │              │
│  │  tianer-grafana:3000                      │              │
│  │  tianer-postgres:5432                     │              │
│  │  tianer-platform pod (C09 API)            │              │
│  └───────────────────────────────────────────┘              │
│                                                             │
│  Browser → C09 API :8080 → /grafana/* → tianer-grafana:3000 │
│  Browser never connects to :3000 directly                   │
└─────────────────────────────────────────────────────────────┘
```

**Key security properties:**
- **No external access:** Port 3000 is bound to `127.0.0.1` only. Even if an attacker is on the LAN, they cannot reach Grafana directly.
- **Same-origin embedding:** The browser sees Grafana content as coming from the C09 API origin (`http://<host>:8080/grafana/`), satisfying `X-Frame-Options: sameorigin`.
- **tianer-net isolation:** The `tianer-net` bridge is internal — containers on it can reach each other but have no path to the internet.

### 8.2 Anonymous Access Configuration

Anonymous access is enabled **only for the MVP phase**. This is a deliberate trade-off: the dashboards are LAN-only and embedded in the frontend, making authentication friction counterproductive for the initial deployment [2].

**`grafana.ini` anonymous auth section:**

```ini
[auth.anonymous]
# Enable anonymous access
enabled = true

# Anonymous viewer — cannot edit dashboards, create alerts, or modify datasources
org_role = Viewer

# Organization for anonymous users
org_name = Tian'er

# Hide the login page (anonymous users are not prompted to login)
hide_login_page = true
```

**Risks and mitigations:**

| Risk | Mitigation |
|------|-----------|
| Anyone on LAN can view dashboards | Dashboards only show aggregated data, not raw packets. `tianer_grafana` has SELECT-only — no write, delete, or DDL capability. |
| Anyone on LAN could embed Grafana on external site | `X-Frame-Options: sameorigin` prevents iframe loading on other domains. |
| Anonymous user could attempt to edit dashboards | `allowUiUpdates: false` in provisioning; `editable: false` in dashboard JSON. All dashboards are version-controlled — UI edits are discarded on next provisioning rescan. |
| Anonymous user could create new datasources | Datasource is provisioned with `editable: false`. UI-based datasource creation is disabled for the Viewer role. |

**Post-MVP migration path:** Add Grafana OAuth (Google, GitHub) or LDAP authentication. The migration is straightforward: change `[auth.anonymous] enabled = false`, configure `[auth.generic_oauth]` block, and set `org_role = Editor` for authenticated users.

### 8.3 Embedding Security

Per user decision, dashboards are embedded in the Vue 3 frontend via iframe. Security headers enforce same-origin embedding:

**`grafana.ini` security section:**

```ini
[security]
# Allow embedding in iframes from same origin only
allow_embedding = true

# Strict cookie security
cookie_samesite = strict
cookie_secure = false  # HTTP for internal-only traffic

# Content Security Policy — allow frames from same origin
content_security_policy = "frame-ancestors 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval';"
```

**C09 API reverse-proxy must set additional headers for the proxied response:**

```
X-Frame-Options: SAMEORIGIN
Content-Security-Policy: frame-ancestors 'self'
```

**Why `cookie_secure = false`:** All traffic is internal (container-to-container on `tianer-net` bridge, or loopback on host). TLS is not configured between containers on the same host. If the C09 API is later exposed externally with TLS, the API's reverse-proxy to Grafana still uses plain HTTP (internal) — the browser-to-API connection is the TLS boundary.

### 8.4 Authentication (Post-MVP)

For MVP, no authentication is configured. The post-MVP plan is documented here for future implementers:

```ini
[auth.generic_oauth]
enabled = true
name = Google
client_id = $GOOGLE_CLIENT_ID
client_secret = $GOOGLE_CLIENT_SECRET
scopes = openid email profile
auth_url = https://accounts.google.com/o/oauth2/auth
token_url = https://accounts.google.com/o/oauth2/token
api_url = https://openidconnect.googleapis.com/v1/userinfo
role_attribute_path = contains(groups[*], 'tianer-admin') && 'Admin' || 'Viewer'
allow_sign_up = true
```

### 8.5 Role Least-Privilege

The `tianer_grafana` PostgreSQL role has the narrowest possible privilege set:

| Operation | Allowed? | Rationale |
|-----------|---------|-----------|
| SELECT | ✅ | Required for dashboard queries |
| INSERT | ❌ | No — Grafana never writes to the database |
| UPDATE | ❌ | No |
| DELETE | ❌ | No |
| TRUNCATE | ❌ | No |
| DDL (CREATE/ALTER/DROP) | ❌ | No |
| COPY | ❌ | No |
| USAGE on schema | ✅ | Required to access tables |
| CONNECT on database | ✅ | Required to establish connection |

This is enforced at the PostgreSQL level (C02 §8.3). Even if the Grafana container is compromised and an attacker gains SQL execution, they can only read data.

### 8.6 Secrets Handling

The Grafana database password is stored in `/etc/tianer/secrets/grafana_db_password` (mode 0600, owner `tianer:tianer`). It is injected into the Grafana container via `EnvironmentFile`:

**`/etc/tianer/secrets/grafana_db_password.env`:**
```
GRAFANA_DB_PASSWORD=<32-byte-base64-random-string>
```

The provisioning datasource YAML references this via the environment variable:

```yaml
secureJsonData:
  password: ${GRAFANA_DB_PASSWORD}
```

Grafana's built-in variable expansion (`${VAR_NAME}`) resolves this at startup. The password is never written to the Grafana configuration files on disk (V07 contains the SQLite DB, not config files).

### 8.7 Container Image Supply Chain

The Grafana container image is pulled from `docker.io/grafana/grafana-oss` and pinned by digest:

```
Image=grafana/grafana-oss:10.4.10@sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

The `:10.4.10` tag is included for human readability; the `@sha256:` digest ensures the exact image bits are verified by Podman at pull time. This is enforced by C14's supply chain integrity (Q10).

---

## 9. Configuration

### 9.1 Environment Variables

| Variable | Default | Purpose | Set By |
|----------|---------|---------|--------|
| `GRAFANA_DB_PASSWORD` | (from secrets file) | `tianer_grafana` PostgreSQL role password | `EnvironmentFile=/etc/tianer/secrets/grafana_db_password.env` |
| `GF_SERVER_HTTP_PORT` | `3000` | Grafana HTTP listen port | Quadlet `Environment=` |
| `GF_SERVER_DOMAIN` | `tianer-grafana` | Server domain (used for redirect URLs) | Quadlet `Environment=` |
| `GF_SERVER_ROOT_URL` | `%(protocol)s://%(domain)s:%(http_port)s/grafana/` | Root URL for reverse-proxy (C09 strips `/grafana` prefix) | Quadlet `Environment=` |

### 9.2 grafana.ini — Complete Configuration

The canonical `grafana.ini` is defined in `deploy/config/grafana.ini` and bind-mounted into the container at `/etc/grafana/grafana.ini` via V01. It is mounted read-only.

```ini
# ===================================
# Tian'er Grafana Configuration
# C11 Grafana Dashboards
# ===================================

# ---- Server ----
[server]
protocol = http
http_addr =
http_port = 3000
domain = tianer-grafana
root_url = %(protocol)s://%(domain)s:%(http_port)s/grafana/
serve_from_sub_path = true
enable_gzip = true
router_logging = false  # Reduce log noise in production

# ---- Database (Grafana's internal SQLite) ----
[database]
type = sqlite3
path = /var/lib/grafana/grafana.db
# SQLite WAL mode for better concurrent read performance
# (Grafana's internal queries against its own config DB)

# ---- Security ----
[security]
# Anonymous access is the only auth mechanism for MVP
allow_embedding = true
cookie_samesite = strict
cookie_secure = false
strict_transport_security = false
content_security_policy = "frame-ancestors 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval';"
disable_gravatar = true
# Allow C09 API reverse-proxy to set X-Forwarded headers
# (Grafana is behind C09's proxy)
data_source_proxy_whitelist = tianer-postgres:5432

# ---- Anonymous Auth ----
[auth.anonymous]
enabled = true
org_name = Tian'er
org_role = Viewer
hide_login_page = true

# ---- Disable All Other Auth Methods ----
[auth.basic]
enabled = false

[auth.ldap]
enabled = false

[auth.generic_oauth]
enabled = false

# ---- Disable Signup (not needed with anonymous Viewer) ----
[users]
allow_sign_up = false
allow_org_create = false
auto_assign_org = true
auto_assign_org_role = Viewer

# ---- Logging ----
[log]
mode = console
level = info

# ---- Metrics (for C13 scraping) ----
[metrics]
enabled = true
basic_auth_username =
basic_auth_password =

# ---- Dashboards ----
[dashboards]
versions_to_keep = 5
default_home_dashboard_path = /etc/grafana/provisioning/dashboards/live-metrics.json

# ---- Plugins ----
[plugins]
enable_alpha = false
allow_loading_unsigned_plugins =

# ---- Rendering (disable headless browser rendering for MVP — not needed) ----
[rendering]
server_url =
callback_url =

# ---- Date formats ----
[date_formats]
full_date = YYYY-MM-DD HH:mm:ss
interval_second = HH:mm:ss
interval_minute = HH:mm
interval_hour = MM/DD HH:mm
interval_day = MM/DD
interval_month = YYYY-MM
interval_year = YYYY

# ---- Unified Alerting (disabled for MVP) ----
[unified_alerting]
enabled = false

[alerting]
enabled = false
execute_alerts = false

# ---- External image storage (disabled — no image rendering) ----
[external_image_storage]
provider =
```

### 9.3 Config File Deployment

All configuration files are deployed from the repository to the host filesystem, then bind-mounted into the container:

```
Repository                          Host (V01)                         Container
──────────                          ──────────                         ─────────
grafana/provisioning/
  datasources/timescaledb.yaml  →   /etc/tianer/grafana/provisioning/ → /etc/grafana/provisioning/
                                      datasources/timescaledb.yaml       (bind-mounted :ro)

grafana/provisioning/
  dashboards/default.yaml       →   /etc/tianer/grafana/provisioning/ → /etc/grafana/provisioning/
                                      dashboards/default.yaml            (bind-mounted :ro)

grafana/dashboards/
  live-metrics.json             →   /etc/tianer/grafana/provisioning/ → /etc/grafana/provisioning/
  device-explorer.json              dashboards/                          dashboards/
  per-device-drilldown.json                                            (bind-mounted :ro)
  pipeline-health.json

deploy/config/grafana.ini       →   /etc/tianer/grafana/grafana.ini  → /etc/grafana/grafana.ini
                                                                        (bind-mounted :ro)
```

### 9.4 Theme Configuration

The BreachLab cyberpunk dark theme is installed as a Grafana plugin during the container image build (see §11.1 Dockerfile). It is set as the default theme in `grafana.ini`:

```ini
# Set in the [server] section or as an environment variable
# The cyberpunk theme overrides the built-in dark theme
```

The theme is applied via environment variable in the Quadlet file:

```
Environment=GF_DEFAULT_THEME=dark
Environment=GF_PLUGIN_XCYBER360_THEME_ENABLED=true
```

If the `xcyber360-theme` plugin is unavailable (e.g., network issues during image build), the container falls back to Grafana's built-in dark theme (`default_theme = dark`). The dashboards are designed with dark backgrounds and cyberpunk-appropriate color palettes (neon greens, cyans, and magenta on dark backgrounds).

---

## 10. Test Plan

### 10.1 Unit Tests — Dashboard Query Validation

Each dashboard JSON file is validated for structural correctness and query validity before deployment.

| Test | Description | Acceptance Criteria |
|------|-------------|---------------------|
| `test_dashboard_json_valid.sh` | Validate each `.json` file is parseable JSON and has required top-level fields | All 4 files are valid JSON with `"uid"`, `"title"`, `"panels"`, `"schemaVersion"` fields |
| `test_datasource_yaml_valid.sh` | Validate `timescaledb.yaml` is parseable YAML with required fields | Valid YAML with `datasources[0].name`, `.type`, `.url`, `.database`, `.user` |
| `test_dashboard_provisioning_yaml_valid.sh` | Validate `default.yaml` is parseable YAML | Valid YAML with `providers[0].name`, `.folder`, `.options.path` |
| `test_sql_syntax.sh` | Extract all SQL queries from dashboard JSONs and validate syntax against a test PostgreSQL instance | All `SELECT` statements parse without syntax errors; no `INSERT`, `UPDATE`, `DELETE`, `DROP`, or `TRUNCATE` statements |
| `test_sql_readonly.sh` | Verify all queries use `SELECT` only (no write operations) | Grep dashboard JSONs for write keywords — must return zero matches |

### 10.2 Integration Tests

| Test | Description | Acceptance Criteria |
|------|-------------|---------------------|
| `test_grafana_starts.sh` | Start Grafana container with provisioning files. Check health endpoint. | `curl -s http://127.0.0.1:3000/api/health` returns HTTP 200 with `"database": "ok"` within 30 seconds of container start |
| `test_datasource_healthy.sh` | Verify the TimescaleDB datasource is healthy in Grafana | `curl -s http://127.0.0.1:3000/api/datasources/1/health` returns `{"status":"OK","message":"Datasource updated"}` |
| `test_dashboards_loaded.sh` | Verify all 4 dashboards appear in Grafana search | `curl -s http://127.0.0.1:3000/api/search?type=dash-db` returns 4 items with expected UIDs |
| `test_anonymous_access.sh` | Access dashboard without authentication | `curl -s http://127.0.0.1:3000/d/tianer-live-metrics` returns HTTP 200 with HTML content |
| `test_embed_header.sh` | Verify `X-Frame-Options` header is set | `curl -sI http://127.0.0.1:3000/d/tianer-live-metrics` includes `X-Frame-Options: sameorigin` |
| `test_dashboard_query_execution.sh` | Execute a representative query from each dashboard against a test DB with seed data | All 4 dashboards' queries return results (non-empty for seed data, empty for empty DB) |
| `test_per_device_variable.sh` | Verify `$mac` variable substitution works | Query for a known test MAC returns correct rows |
| `test_multi_sniffer_variable.sh` | Verify `$sniffer` multi-value variable works | Query with `$sniffer = 'All'` returns rows for all sniffers |
| `test_grafana_restart_persistence.sh` | Restart container and verify dashboards and datasource still exist | After restart, `api/search` still returns 4 dashboards; datasource still healthy |

### 10.3 Performance Tests

| Test | Target | Measurement |
|------|--------|-------------|
| Container cold-start time | < 3 seconds to HTTP 200 on `/api/health` | `time podman start tianer-grafana` + health check loop |
| Dashboard load time | < 5 seconds for full dashboard render | Browser DevTools network tab — time to `DOMContentLoaded` |
| Query execution: device_5min_buckets (24h, single MAC) | < 200ms | `EXPLAIN ANALYZE` or Grafana query inspector |
| Query execution: raw_packets count (1h, all sniffers) | < 500ms | `EXPLAIN ANALYZE` |
| Memory usage (idle, all 4 dashboards loaded) | < 256 MB | `podman stats tianer-grafana` |
| Memory usage (under concurrent query load, 10 users) | < 512 MB | `podman stats tianer-grafana` |

### 10.4 Security Tests

| Test | Description | Acceptance Criteria |
|------|-------------|---------------------|
| `test_no_login_page.sh` | Attempt to access Grafana login page | `curl -s http://127.0.0.1:3000/login` returns HTTP 302 redirect (anonymous access bypasses login) |
| `test_no_dashboard_edit.sh` | Attempt to save a dashboard edit via API | `curl -s -X POST http://127.0.0.1:3000/api/dashboards/db` returns HTTP 401 or 403 |
| `test_no_datasource_create.sh` | Attempt to create a new datasource via API | `curl -s -X POST http://127.0.0.1:3000/api/datasources` returns HTTP 401 or 403 |
| `test_port_not_externally_bound.sh` | Verify port 3000 is not listening on `0.0.0.0` | `ss -tlnp | grep 3000` shows only `127.0.0.1:3000`, not `0.0.0.0:3000` |
| `test_no_write_sql.sh` | Verify no dashboard contains INSERT/UPDATE/DELETE/TRUNCATE/DROP | Grep returns zero matches |

### 10.5 CI Integration

The Grafana tests run as part of `ci/test-all.sh` (step 9: "Grafana smoke: start container, verify dashboards, run queries"). The CI pipeline:

1. Starts a PostgreSQL 17 + TimescaleDB container with seed data (`ci/docker-compose.yml`)
2. Builds the `tianer-grafana` container image
3. Starts the Grafana container with provisioning files
4. Waits for Grafana to be healthy (`/api/health`)
5. Runs the integration and security test suites
6. Verifies all SQL queries execute without errors

### 10.6 Acceptance Criteria

1. **Container starts:** `systemctl --user start tianer-grafana` results in a running container with HTTP 200 on `/api/health` within 30 seconds.
2. **Datasource healthy:** TimescaleDB datasource shows as healthy in Grafana UI and API.
3. **All 4 dashboards present:** Live Metrics, Device Explorer, Per-Device Drilldown, and Pipeline Health are listed in Grafana search.
4. **Queries execute:** Representative queries from each dashboard return results against a database with seed data.
5. **Anonymous access:** Visiting `http://<host>:3000/d/tianer-live-metrics` renders the dashboard without a login prompt.
6. **Embedding supported:** `X-Frame-Options: sameorigin` header is present on dashboard responses.
7. **CSP correct:** `Content-Security-Policy` header includes `frame-ancestors 'self'`.
8. **Cyberpunk theme:** Dark theme with neon accents is applied (or Grafana built-in dark theme as fallback).
9. **Read-only enforcement:** No dashboard contains INSERT, UPDATE, DELETE, TRUNCATE, or DROP statements.
10. **Secure:** Port 3000 is bound to `127.0.0.1` only. No write access to DB possible via `tianer_grafana`.
11. **Restart-safe:** After container restart, all dashboards and datasource are preserved.

---

## 11. Deployment Notes

### 11.1 Container Image

The Grafana container image is built as part of C14 (Deployment Automation). It uses a multi-stage Dockerfile to install the cyberpunk theme plugin:

```dockerfile
# deploy/containers/Dockerfile.grafana
# Stage 1: Build — install plugins
FROM grafana/grafana-oss:10.4.10 AS builder
USER root
RUN grafana cli --pluginUrl https://github.com/BreachLab/xcyber360-theme/archive/refs/heads/main.zip \
    plugins install breachlab-xcyber360-theme
# Fallback: if plugin install fails, image still builds with built-in dark theme
USER grafana

# Stage 2: Runtime — copy plugins from builder, no build tools
FROM grafana/grafana-oss:10.4.10
COPY --from=builder /var/lib/grafana/plugins /var/lib/grafana/plugins
USER grafana
```

**Image details:**

| Property | Value |
|----------|-------|
| Base image | `grafana/grafana-oss:10.4.10@sha256:...` (digest-pinned) |
| Installed plugins | `breachlab-xcyber360-theme` (dark cyberpunk theme) |
| Image size target | Under 250 MB (base Grafana OSS image is ~230 MB; plugin adds ~5 MB) |
| User | `grafana` (UID 472 inside container) |
| Entrypoint | Default Grafana entrypoint (`/run.sh`) |

### 11.2 Quadlet Unit

`deploy/containers/tianer-grafana.container` (Quadlet file, deployed by C14 to `/etc/containers/systemd/`):

```ini
[Unit]
Description=Tian'er Grafana 10.4 Dashboards
After=tianer-net.network tianer-postgres.service
Requires=tianer-net.network
Wants=tianer-postgres.service

[Container]
Image=localhost/tianer-grafana:latest
ContainerName=tianer-grafana
Network=tianer-net
PublishPort=127.0.0.1:3000:3000

# Volumes
Volume=tianer-grafana-data.volume:/var/lib/grafana
Volume=/etc/tianer/grafana/grafana.ini:/etc/grafana/grafana.ini:ro
Volume=/etc/tianer/grafana/provisioning:/etc/grafana/provisioning:ro

# Secrets
EnvironmentFile=/etc/tianer/secrets/grafana_db_password.env

# Environment overrides
Environment=GF_DEFAULT_THEME=dark
Environment=GF_SERVER_DOMAIN=tianer-grafana
Environment=GF_SERVER_ROOT_URL=%(protocol)s://%(domain)s:%(http_port)s/grafana/
Environment=GF_SERVER_SERVE_FROM_SUB_PATH=true

# Security hardening
NoNewPrivileges=true
DropCapability=ALL

# Resource limits
MemoryMax=512M
CPUQuota=100%

# Health check
HealthCmd=curl --fail http://localhost:3000/api/health || exit 1
HealthInterval=30s
HealthRetries=3
HealthStartPeriod=60s

[Service]
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=default.target
```

**Key points:**
- `PublishPort=127.0.0.1:3000:3000` — only loopback binding on host. No external access.
- `grafana.ini` and provisioning directory are bind-mounted read-only (`:ro`) from V01.
- `grafana-data.volume` is the persistent Podman volume (V07) for the SQLite DB and any plugin data.
- `EnvironmentFile` injects the DB password securely.
- `MemoryMax=512M` — caps Grafana at 512 MB RAM (conservative for CM5 8 GB total).
- `HealthCmd` uses Grafana's own health endpoint; Podman health checks integrated with systemd.
- `HealthStartPeriod=60s` — gives Grafana time to start up and provision before health checks begin.

### 11.3 Podman Volume Creation

`deploy/containers/tianer-grafana-data.volume` (Quadlet volume definition):

```ini
[Volume]
```

Minimal definition — Quadlet creates a Podman volume with default settings (local driver, overlay filesystem). The volume persists across container restarts and rebuilds.

**Volume contents:**
```
/var/lib/grafana/
├── grafana.db              # Grafana internal SQLite database
├── plugins/                 # Installed plugins (xcyber360 theme)
│   └── breachlab-xcyber360-theme/
└── png/                     # Rendered panel images (if rendering is later enabled)
```

### 11.4 Install Script Integration

`deploy/scripts/install-grafana.sh` (invoked by `deploy/setup.sh` during C01 bootstrap):

```bash
#!/usr/bin/env bash
set -euo pipefail

# This script is idempotent. It:
# 1. Builds the tianer-grafana container image (if not already built)
# 2. Creates the Podman volume tianer-grafana-data (if not exists)
# 3. Copies provisioning files from repo to /etc/tianer/grafana/
# 4. Copies grafana.ini to /etc/tianer/grafana/
# 5. Installs the Quadlet unit file
# 6. Reloads systemd and enables the service

echo "TIANER | {\"ts\":\"$(date -Iseconds)\",\"level\":\"INFO\",\"component\":\"grafana\",\"msg\":\"Building tianer-grafana image\"}"
podman build -t localhost/tianer-grafana:latest -f deploy/containers/Dockerfile.grafana .

echo "TIANER | {\"ts\":\"$(date -Iseconds)\",\"level\":\"INFO\",\"component\":\"grafana\",\"msg\":\"Creating Grafana volume if not exists\"}"
podman volume exists tianer-grafana-data || podman volume create tianer-grafana-data

echo "TIANER | {\"ts\":\"$(date -Iseconds)\",\"level\":\"INFO\",\"component\":\"grafana\",\"msg\":\"Deploying Grafana config and provisioning files\"}"
install -d -m 0755 /etc/tianer/grafana/provisioning/datasources
install -d -m 0755 /etc/tianer/grafana/provisioning/dashboards
install -m 0644 grafana/provisioning/datasources/timescaledb.yaml /etc/tianer/grafana/provisioning/datasources/
install -m 0644 grafana/provisioning/dashboards/default.yaml /etc/tianer/grafana/provisioning/dashboards/
install -m 0644 grafana/dashboards/*.json /etc/tianer/grafana/provisioning/dashboards/
install -m 0644 deploy/config/grafana.ini /etc/tianer/grafana/

echo "TIANER | {\"ts\":\"$(date -Iseconds)\",\"level\":\"INFO\",\"component\":\"grafana\",\"msg\":\"Installing Quadlet unit files\"}"
install -m 0644 deploy/containers/tianer-grafana.container /etc/containers/systemd/
install -m 0644 deploy/containers/tianer-grafana-data.volume /etc/containers/systemd/
systemctl --user daemon-reload

echo "TIANER | {\"ts\":\"$(date -Iseconds)\",\"level\":\"INFO\",\"component\":\"grafana\",\"msg\":\"Grafana container image built and Quadlet unit installed\"}"
```

### 11.5 Repository File Inventory

```
grafana/
├── provisioning/
│   ├── datasources/
│   │   └── timescaledb.yaml          # PostgreSQL datasource configuration
│   └── dashboards/
│       └── default.yaml              # Dashboard provider configuration
├── dashboards/
│   ├── live-metrics.json             # Dashboard 1: Live Metrics
│   ├── device-explorer.json          # Dashboard 2: Device Explorer
│   ├── per-device-drilldown.json     # Dashboard 3: Per-Device Drilldown ($mac variable)
│   └── pipeline-health.json          # Dashboard 4: Pipeline Health

deploy/
├── containers/
│   ├── Dockerfile.grafana            # Grafana container image build
│   ├── tianer-grafana.container      # Quadlet unit file
│   └── tianer-grafana-data.volume    # Quadlet volume definition
├── config/
│   └── grafana.ini                   # Grafana server configuration
└── scripts/
    └── install-grafana.sh            # Idempotent deploy script
```

### 11.6 Deployment Checklist

Before considering C11 deployed, verify:

- [ ] `tianer-grafana` container image built successfully (`podman image exists localhost/tianer-grafana`)
- [ ] `tianer-grafana-data` volume created (`podman volume exists tianer-grafana-data`)
- [ ] Provisioning files copied to `/etc/tianer/grafana/provisioning/` with correct permissions (0644)
- [ ] `grafana.ini` copied to `/etc/tianer/grafana/grafana.ini` with permissions 0644
- [ ] Quadlet unit and volume files installed to `/etc/containers/systemd/`
- [ ] `systemctl --user daemon-reload` executed
- [ ] `systemctl --user start tianer-grafana` succeeds (container running)
- [ ] `curl -s http://127.0.0.1:3000/api/health` returns `{"database": "ok", ...}` within 60 seconds
- [ ] TimescaleDB datasource shows healthy in Grafana UI (`/datasources`)
- [ ] All 4 dashboards appear in Grafana search (`/dashboards`)
- [ ] Anonymous access works: `curl -s http://127.0.0.1:3000/d/tianer-live-metrics` returns HTML (no redirect to /login)
- [ ] `X-Frame-Options: sameorigin` header verified on dashboard responses
- [ ] `Content-Security-Policy` header includes `frame-ancestors 'self'`
- [ ] Cyberpunk dark theme applied (or built-in dark theme as fallback)
- [ ] Port 3000 bound to `127.0.0.1` only (verified with `ss -tlnp | grep 3000`)
- [ ] `podman stats tianer-grafana` shows memory usage under 256 MB at idle
- [ ] Container auto-restarts after `podman kill tianer-grafana` (systemd `Restart=on-failure`)
- [ ] `systemctl --user enable tianer-grafana` ensures auto-start on boot (via `linger`)
- [ ] Dashboard queries execute successfully against seed data (all panels show data, not errors)

---

## Glossary

| Term | Definition |
|------|-----------|
| **Provisioning** | Grafana's mechanism for declaring datasources and dashboards as code (YAML/JSON files), loaded at startup and periodically rescanned |
| **V01** | `tianer-config` bind-mount volume — `/etc/tianer/` on host, mounted `:ro` in all containers |
| **V07** | `tianer-grafana-data` Podman volume — persistent storage for Grafana's SQLite DB and plugins |
| **tianer-net** | Internal Podman bridge network connecting `tianer-postgres`, `tianer-grafana`, and `tianer-platform` pod |
| **Embedding** | Displaying Grafana dashboards inside the Vue 3 frontend via `<iframe>` elements, with same-origin security headers |
| **Anonymous access** | No login required; all visitors are authenticated as the anonymous `Viewer` user (MVP-only) |
| **Dashboard UID** | Stable unique identifier for each dashboard (e.g., `tianer-live-metrics`). Used in URLs and iframe `src` attributes |

---

*End of C11 Grafana Dashboards Design Document.*

## References

[1] Grafana Labs. "Provision Grafana — Grafana Documentation." https://grafana.com/docs/grafana/latest/administration/provisioning/, 2024.

[2] Grafana Labs. "Grafana Authentication — Anonymous Authentication." https://grafana.com/docs/grafana/latest/setup-grafana/configure-security/configure-authentication/grafana/#anonymous-authentication, 2024.

[3] W3C. "Content Security Policy Level 2 — W3C Recommendation, 15 December 2016." https://www.w3.org/TR/CSP2/, December 2016. See §7.7 `frame-ancestors` directive.

[4] Timescale, Inc. "time_bucket() — TimescaleDB API Reference." https://docs.timescale.com/api/latest/hyperfunctions/time_bucket/, 2024.

[5] Grafana Labs. "Dashboard JSON Model — Grafana Documentation." https://grafana.com/docs/grafana/latest/dashboards/build-dashboards/view-dashboard-json-model/, 2024. Describes dashboard schema, templating variables, and `schemaVersion`.
