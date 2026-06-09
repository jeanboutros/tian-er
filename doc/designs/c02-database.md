# C02 — Database

**Status:** Phase A Design — guides Phase B implementation  
**Author:** Code Architect  
**Date:** 2026-06-09  
**Dependencies:** C01 (Platform Infrastructure)  
**Blocks:** C05, C07, C09, C11 (and transitively C06, C08, C10, C12)

---

## 1. Overview

### 1.1 Purpose

C02 Database is the **central data store** for the entire Tian'er Signal Intelligence Platform. It runs **PostgreSQL 17 with TimescaleDB 2.23+** in a rootless Podman container backed by a persistent Podman volume (V06). Every sensor module — starting with the v1 Bluetooth module — writes its time-series observations here, queries aggregated views here, and stores enrichment data here. C02 defines the database roles, the schema namespaces, the table structure, the migration protocol, the continuous aggregates, the compression and retention policies, and the residency classification logic.

### 1.2 Scope

| In Scope | Out of Scope |
|----------|-------------|
| PostgreSQL 17 installation and configuration (containerized) | Host OS package installation (C01) |
| TimescaleDB 2.23+ extension setup | Podman/Quadlet container runtime setup (C01) |
| Database role creation and permission grants | Secrets generation (C01 `generate-secrets.sh`) |
| `bluetooth` schema and all v1 tables | Future module schemas (GPS, ADS-B, etc.) — architecture supports them but does not implement |
| Migration files (`db/migrations/`) and application protocol | Application SQL queries (owned by C05, C06, C09, C11) |
| `apply-migrations.sh` script | systemd unit wiring (C12) |
| Residency classifier PL/pgSQL function | Rule-based ML classifier (C08) |
| `_migrations` tracking table | |
| Continuous aggregates and refresh policies | |
| Compression and retention policies | |
| `pg_hba.conf` and `postgresql.conf` configuration | |

### 1.3 Boundaries

C02 owns the **database layer**: the running PostgreSQL instance, the schema definitions, the role permissions, and the migration lifecycle. It does **not** own the host packages (C01), the container Quadlet file (C14), the application queries (consumers), or the secrets files (C01). The `tianer` role with DDL privileges is **restricted to the C09 API server and the migration runner** — no other service may perform DDL.

```
┌───────────────────────────────────────────────────────────────┐
│                      C01 PLATFORM HOST                         │
│  (Podman runtime, secrets via EnvironmentFile, tianer-net)    │
└───────┬───────────────────────┬───────────────────┬───────────┘
        │                       │                   │
        ▼                       ▼                   ▼
┌───────────────┐    ┌──────────────────┐    ┌──────────────┐
│  C02 DATABASE  │    │  C05 Ingest      │    │ C09 REST API │
│  (PostgreSQL   │◄───│  Bridge          │    │ (FastAPI)    │
│   + TimescaleDB│    │  (tianer_writer) │    │ (tianer_ro + │
│   in container)│    └──────────────────┘    │  tianer)     │
│                │                            └──────┬───────┘
│                │◄──────────────────────────────────┘
│                │
│                │◄──── C06 Gap Detector (tianer_writer)
│                │◄──── C07 Deep Parser  (direct DB not required)
│                │◄──── C08 ML Enrichment (tianer_writer)
│                │◄──── C11 Grafana       (tianer_grafana)
└───────────────┘
```

### 1.4 Position in the System

C02 is a **Layer 0 shared platform component** in the build sequence (component-breakdown.md §4.1). It is the second component built, immediately after C01, because every Layer 1 and Layer 2 component depends on the database being available. The critical path `C01 → C02 → C05 → C09 → C10 → C12` requires C02 before the ingest bridge (C05) and REST API (C09) can be implemented.

---

## 2. High-Level Architecture (HLA)

### 2.1 Container Deployment Architecture

PostgreSQL runs as a **standalone container** managed by rootless Podman via a Quadlet `.container` file (defined in C14, deployed via C12). It is the only container that accesses the `tianer-postgres-data` Podman volume (V06).

```
┌───────────────────────────────────────────────────────────────┐
│                     PODMAN (rootless, user tianer)             │
│                                                               │
│  ┌───────────────────────────────────────────────────────┐   │
│  │              tianer-net (bridge network)                │   │
│  │                                                        │   │
│  │  ┌─────────────────────────┐    ┌───────────────────┐  │   │
│  │  │  tianer-postgres        │    │  tianer-platform   │  │   │
│  │  │  (standalone container) │    │  pod                │  │   │
│  │  │                         │    │  ┌───────────────┐ │  │   │
│  │  │  PostgreSQL 17          │◄───┼──┤ C05 Ingest    │ │  │   │
│  │  │  + TimescaleDB 2.23     │    │  │ Bridge        │ │  │   │
│  │  │                         │    │  ├───────────────┤ │  │   │
│  │  │  Port: 5432             │◄───┼──┤ C06 Gap       │ │  │   │
│  │  │  Volume: V06 (:rw)      │    │  │ Detector      │ │  │   │
│  │  │  Secrets: Environment-  │    │  ├───────────────┤ │  │   │
│  │  │  File injection         │    │  │ C09 REST API  │ │  │   │
│  │  │                         │    │  └───────────────┘ │  │   │
│  │  └─────────────────────────┘    └───────────────────┘  │   │
│  │                                                        │   │
│  │  ┌─────────────────────────┐                            │   │
│  │  │  tianer-grafana         │                            │   │
│  │  │  (standalone container) │                            │   │
│  │  │                         │                            │   │
│  │  │  Port: 3000             │                            │   │
│  │  │  Volume: V07 (:rw)      │                            │   │
│  │  └─────────┬───────────────┘                            │   │
│  └────────────┼────────────────────────────────────────────┘   │
│               │                                                │
└───────────────┼────────────────────────────────────────────────┘
                │
                ▼
    Reads PostgreSQL via tianer_grafana role
```

**Key deployment properties:**

| Property | Value |
|----------|-------|
| Container name | `tianer-postgres` |
| Image | `postgres:17-bookworm@sha256:...` (digest-pinned) |
| Network | `tianer-net` (bridge, created by C14 Quadlet `.network` file) |
| Published port | `127.0.0.1:5432:5432` (loopback only on host) |
| Volume | V06 `tianer-postgres-data` Podman volume, mounted at `/var/lib/postgresql/data` |
| Secrets | `EnvironmentFile=/etc/tianer/secrets/db_password.env` |
| Capabilities | `--cap-drop ALL` (no elevated privileges) |
| User | Runs as `postgres` user inside container (UID 999); container itself runs under rootless Podman |

### 2.2 Role Hierarchy

The database uses **four application roles**, none of which are superusers. The `postgres` superuser role exists only for administrative tasks (initial database creation, backup/restore, disaster recovery) and is never used by application code.

```
┌─────────────────────────────────────────────────────────┐
│                     postgres (superuser)                  │
│   Used only for: CREATE DATABASE, CREATE ROLE,           │
│   pg_basebackup, disaster recovery.                      │
│   NOT used by any application component.                 │
└────────────────────────┬────────────────────────────────┘
                         │
         ┌───────────────┼───────────────┐
         │               │               │
         ▼               ▼               ▼
┌─────────────┐  ┌─────────────┐  ┌──────────────┐
│   tianer    │  │tianer_writer│  │  tianer_ro   │
│  (DDL+DML)  │  │ (INSERT-    │  │ (SELECT-     │
│             │  │  only)      │  │  only)       │
│  Owns:      │  │             │  │              │
│  - bluetooth│  │ Used by:    │  │ Used by:     │
│    schema   │  │ - C05 Ingest│  │ - C09 API    │
│  - All      │  │   Bridge    │  │   read       │
│    objects  │  │ - C06 Gap   │  │   endpoints  │
│             │  │   Detector  │  │              │
│  Used by:   │  │             │  │              │
│  - C09 API  │  │ CAN:        │  │ CAN:         │
│    (writes, │  │ - INSERT    │  │ - SELECT     │
│     DDL)    │  │ - COPY      │  │ - USAGE on   │
│  - Migration │  │             │  │   schema     │
│    runner   │  │ CANNOT:     │  │              │
│             │  │ - SELECT    │  │ CANNOT:      │
│  CAN:       │  │ - UPDATE    │  │ - INSERT     │
│  - ALL on   │  │ - DELETE    │  │ - UPDATE     │
│    bluetooth│  │ - DDL       │  │ - DELETE     │
│    schema   │  │ - TRUNCATE  │  │ - DDL        │
└─────────────┘  └─────────────┘  └──────────────┘
                                           │
                         ┌─────────────────┘
                         ▼
                ┌──────────────────┐
                │ tianer_grafana   │
                │ (SELECT-only)    │
                │                  │
                │ Used by:         │
                │ - C11 Grafana    │
                │   dashboards     │
                │                  │
                │ CAN:             │
                │ - SELECT         │
                │ - USAGE on       │
                │   schema         │
                │                  │
                │ CANNOT:          │
                │ - INSERT         │
                │ - UPDATE         │
                │ - DELETE         │
                │ - DDL            │
                └──────────────────┘
```

**Role privilege summary:**

| Role | LOGIN | CONNECT on `tianer` DB | `bluetooth` Schema | Table Privileges | Consumer Components |
|------|-------|------------------------|---------------------|-----------------|---------------------|
| `postgres` | ✓ (peer only) | Owner (superuser) | Implicit all | ALL | Admin only (install, backup, restore) |
| `tianer` | ✓ (scram-sha-256) | ✓ (owner) | Owner | ALL (DDL + DML) | C09 API (write endpoints, continuous aggregate refresh), `apply-migrations.sh` |
| `tianer_writer` | ✓ (scram-sha-256) | ✓ | USAGE | INSERT only + default INSERT | C05 Ingest Bridge, C06 Gap Detector, C08 ML Enrichment |
| `tianer_ro` | ✓ (scram-sha-256) | ✓ | USAGE | SELECT only + default SELECT | C09 API (read-only endpoints) |
| `tianer_grafana` | ✓ (scram-sha-256) | ✓ | USAGE | SELECT only + default SELECT | C11 Grafana |

**Rationale for the `tianer_writer` INSERT-only restriction (Q9):** If the ingest bridge or gap detector is compromised, the attacker cannot SELECT (read sensitive data), UPDATE, DELETE, TRUNCATE, or DROP tables. The blast radius is limited to inserting rows — which can be cleaned up but cannot destroy existing data. This is defense in depth (PF-7).

**Rationale for separate `tianer_ro` and `tianer_grafana` roles:** While both have SELECT-only access, separating them allows independent password rotation and per-consumer connection limits. Grafana connections are pooled differently from API read connections.

### 2.3 Connection Paths

```
┌──────────────────┐                    ┌──────────────────┐
│  C05 Ingest      │                    │  C06 Gap         │
│  Bridge (C++)    │                    │  Detector (Python)│
│  libpqxx 7.8     │                    │  psycopg 3.1     │
└────────┬─────────┘                    └────────┬─────────┘
         │                                       │
         │ tianer_writer                         │ tianer_writer
         │ scram-sha-256                         │ scram-sha-256
         │ COPY protocol (streaming)             │ INSERT (idempotent)
         │                                       │
         ▼                                       ▼
┌─────────────────────────────────────────────────────────────┐
│                   tianer-postgres:5432                       │
│                                                             │
│  pg_hba.conf: host tianer tianer_writer 127.0.0.1/32 scram  │
│                                                             │
│              ┌──────────────────────────┐                   │
│              │     Database: tianer      │                   │
│              │                          │                   │
│              │  Schema: bluetooth        │                   │
│              │  ┌─────────────────────┐ │                   │
│              │  │ raw_packets         │ │                   │
│              │  │ (hypertable)        │ │                   │
│              │  │ sniffer_heartbeat   │ │                   │
│              │  │ ingest_gaps         │ │                   │
│              │  │ device_summary      │ │                   │
│              │  │ device_enrichment   │ │                   │
│              │  │ sniffers            │ │                   │
│              │  │ _migrations         │ │                   │
│              │  └─────────────────────┘ │                   │
│              │                          │                   │
│              │  Continuous Aggregates:   │                   │
│              │  - device_5min_buckets    │                   │
│              │                          │                   │
│              │  Functions:              │                   │
│              │  - classify_residency()  │                   │
│              └──────────────────────────┘                   │
│                                                             │
└────────────────┬──────────────────────────┬─────────────────┘
                 │                          │
     tianer / tianer_ro            tianer_grafana
     scram-sha-256                 scram-sha-256
     SELECT / INSERT / DDL         SELECT only
                 │                          │
                 ▼                          ▼
         ┌──────────────┐          ┌──────────────┐
         │  C09 REST API │          │ C11 Grafana  │
         │  (FastAPI)    │          │ Dashboards   │
         │  asyncpg pool │          │ Provisioned  │
         │               │          │ datasource   │
         │  Write path:  │          │              │
         │   tianer role │          │ SQL queries  │
         │  Read path:   │          │ for panels   │
         │   tianer_ro   │          │              │
         └──────────────┘          └──────────────┘
```

**Connection protocol rules:**

| Consumer | Role | Protocol | Connection Pooling | Typical Connections |
|----------|------|----------|-------------------|---------------------|
| C05 Ingest Bridge | `tianer_writer` | `pqxx::stream_to` (COPY) | Per-sniffer process (1 conn each) | 1–4 |
| C06 Gap Detector | `tianer_writer` | psycopg async pool | 2 connections per process | 2 |
| C08 ML Enrichment | `tianer_writer` | psycopg async pool | 1 connection | 1 |
| C09 API (write) | `tianer` | asyncpg pool | 5 connections | 5 |
| C09 API (read) | `tianer_ro` | asyncpg pool | 10 connections | 10 |
| C11 Grafana | `tianer_grafana` | Grafana built-in PG pool | Configurable (default: 5) | 5 |
| Migration runner | `tianer` | psql (ad-hoc) | 1 at a time | 1 |
| **Total** | | | | **~25–30** |

PostgreSQL `max_connections` is configured at **50** (generous headroom for 25–30 typical connections). This covers normal operation plus administrative connections (pg_basebackup, operator `psql` sessions). Per the PostgreSQL wiki on connection pooling [4], connection contention increases past the optimal pool size; the `max_connections` value provides headroom without exceeding the performance "knee" for this workload.

---

## 3. Data Model (ERD)

### 3.1 Schema Namespace Strategy

The `tianer` database contains per-module schemas. The v1 Bluetooth module uses the `bluetooth` schema. Future modules (GPS, ADS-B, Wi-Fi, etc.) will each get their own schema:

```
tianer (database)
├── bluetooth          ← v1 module (this document)
│   ├── _migrations
│   ├── sniffers
│   ├── sniffer_heartbeat
│   ├── raw_packets (hypertable)
│   ├── device_summary
│   ├── device_enrichment
│   ├── ingest_gaps
│   ├── device_5min_buckets (continuous aggregate)
│   └── classify_residency() (function)
├── gps                ← future module
├── adsb               ← future module
└── public             ← reserved, may contain cross-module shared tables
```

**Search path:** All migration SQL files set `SET search_path TO bluetooth;` explicitly. Application code references tables using fully-qualified names (`bluetooth.raw_packets`). Cross-schema queries (for future cross-module correlation) will use schema-qualified references.

### 3.2 Entity-Relationship Diagram

```
┌──────────────────┐
│    _migrations    │  ← Tracks applied migration filenames
│──────────────────│
│ name       TEXT PK│
│ applied_at TIMESTZ│
└──────────────────┘

┌──────────────────┐          ┌──────────────────────────┐
│    sniffers       │          │   sniffer_heartbeat       │
│──────────────────│          │──────────────────────────│
│ sniffer_id  PK   │◄─────────│ sniffer_id    PK, FK      │
│ name        UNIQ │          │ ts            TIMESTAMPTZ │
│ type        TEXT │          │ status        TEXT        │
│ device_path TEXT │          └──────────────────────────┘
│ enabled     BOOL │
│ created_at  TIMESTZ
└──────┬───────────┘
       │
       │ 1:N
       │
       ▼
┌──────────────────────────────────────────────┐
│              raw_packets (HYPERTABLE)          │
│──────────────────────────────────────────────│
│ ts            TIMESTAMPTZ NOT NULL            │  ← Partition key
│ sniffer_id    SMALLINT NOT NULL               │  ← FK → sniffers
│ mac_address   BYTEA NOT NULL                  │
│ address_type  SMALLINT                        │  ← 0=public, 1=random
│ rssi          SMALLINT                        │
│ channel       SMALLINT                        │
│ pdu_type      SMALLINT                        │
│ advdata       BYTEA                           │
│ backfilled    BOOLEAN NOT NULL DEFAULT FALSE  │
│                                               │
│ INDEXES:                                      │
│  - (mac_address, ts DESC)                     │
│  - (sniffer_id, ts DESC)                      │
│                                               │
│ DEDUP: Query-time only                        │
│  DISTINCT ON (sniffer_id, ts,                 │
│               mac_address, pdu_type)          │
│  No insert-time unique constraint (D-10)      │
│                                               │
│ TIMESCALEDB:                                  │
│  - hypertable, chunk_time_interval=1h        │
│  - compressed after 7 days                   │
│  - compressed segmentby=mac_address           │
│  - compressed orderby=ts DESC                 │
│  - retention: drop chunks after 90 days      │
└──────┬───────────────────────────────────────┘
       │
       │ Aggregated by mac_address
       │ (via continuous aggregate + manual refresh)
       │
       ▼
┌──────────────────────────────────────────────┐
│           device_summary                      │
│──────────────────────────────────────────────│
│ mac_address      BYTEA PRIMARY KEY           │
│ address_type     SMALLINT                     │
│ first_seen       TIMESTAMPTZ NOT NULL        │
│ last_seen        TIMESTAMPTZ NOT NULL        │
│ total_count      BIGINT NOT NULL DEFAULT 0   │
│ distinct_days    INT NOT NULL DEFAULT 0       │
│ residency_class  TEXT                         │  ← 'resident','frequent','transient'
│ last_classified  TIMESTAMPTZ                  │     'new','lost','unknown'
│ enrichment_data  JSONB                        │  ← {"classes":[...], ...}
│                                               │
│ INDEXES:                                      │
│  - (last_seen DESC)                           │
│                                               │
│ UPDATED BY:                                   │
│  - Continuous aggregate refresh (counts)     │
│  - classify_residency() function             │
│  - C08 ML Enrichment (enrichment_data)       │
└──────────────────────────────────────────────┘
       │
       │ 1:N (multiple enrichment records per device)
       │
       ▼
┌──────────────────────────────────────────────┐
│          device_enrichment                    │
│──────────────────────────────────────────────│
│ id               BIGSERIAL PRIMARY KEY       │
│ mac_address      BYTEA NOT NULL              │
│ observed_ts      TIMESTAMPTZ NOT NULL        │
│ sniffer_id       SMALLINT NOT NULL           │
│ local_name       TEXT                         │
│ service_uuids_16 TEXT[]                       │
│ service_uuids_128 TEXT[]                      │
│ manufacturer_id  INT                          │
│ manufacturer_data BYTEA                       │
│ tx_power         SMALLINT                     │
│ flags            SMALLINT                     │
│ raw_advdata      BYTEA                        │
│                                               │
│ UNIQUE: (mac_address, observed_ts, sniffer_id)│
│ INDEX:  (mac_address, observed_ts DESC)       │
│                                               │
│ POPULATED BY: C07 Deep Parser → C08 ML        │
└──────────────────────────────────────────────┘

┌──────────────────────────────────────────────┐
│              ingest_gaps                      │
│──────────────────────────────────────────────│
│ id               BIGSERIAL PRIMARY KEY       │
│ sniffer_id       SMALLINT NOT NULL           │
│ bucket_start     TIMESTAMPTZ NOT NULL        │
│ bucket_end       TIMESTAMPTZ NOT NULL        │
│ status           TEXT NOT NULL               │  ← 'open','backfilling',
│                                               │     'closed','failed'
│ retry_count      INT NOT NULL DEFAULT 0      │
│ first_detected   TIMESTAMPTZ DEFAULT NOW()   │
│ last_attempted   TIMESTAMPTZ                  │
│ rows_backfilled  INT                          │
│ last_error       TEXT                         │
│                                               │
│ UNIQUE: (sniffer_id, bucket_start)            │
│ CHECK: status IN (...list above...)           │
│                                               │
│ POPULATED BY: C06 Gap Detector                │
└──────────────────────────────────────────────┘
```

### 3.3 raw_packets — Hypertable Design

`raw_packets` is the **primary time-series table**. It stores every packet observation from every sniffer. It is a TimescaleDB hypertable partitioned by `ts` with 1-hour chunks.

#### 3.3.1 Column Details

| Column | Type | Constraints | Description |
|--------|------|------------|-------------|
| `ts` | `TIMESTAMPTZ` | `NOT NULL` | Packet timestamp (sniffer-reported, not ingest time). Partition key for hypertable. |
| `sniffer_id` | `SMALLINT` | `NOT NULL` | References `sniffers.sniffer_id`. Identifies which sniffer captured this packet. |
| `mac_address` | `BYTEA` | `NOT NULL` | 6-byte Bluetooth device address (BD_ADDR). Stored as raw bytes for index efficiency. |
| `address_type` | `SMALLINT` | nullable | `0` = public address, `1` = random address. Null when not determinable (e.g., non-ADV packets). |
| `rssi` | `SMALLINT` | nullable | Received Signal Strength Indicator in dBm (typically -100 to -20). |
| `channel` | `SMALLINT` | nullable | BLE advertising channel index: 37 (2402 MHz), 38 (2426 MHz), 39 (2480 MHz). |
| `pdu_type` | `SMALLINT` | nullable | BLE advertising PDU type per Core Spec (e.g., `0` = ADV_IND, `1` = ADV_DIRECT_IND, `2` = ADV_NONCONN_IND, `3` = SCAN_REQ, `4` = SCAN_RSP, `5` = CONNECT_IND, `6` = ADV_SCAN_IND). |
| `advdata` | `BYTEA` | nullable | Raw advertising data payload bytes. Up to 31 bytes per BLE spec for legacy advertising. |
| `backfilled` | `BOOLEAN` | `NOT NULL DEFAULT FALSE` | `TRUE` when the row was inserted by the gap detector during backfill. Distinguishes live-ingested from backfilled rows. |

#### 3.3.2 Deduplication Strategy (D-10)

**No insert-time unique index.** When multiple sniffers observe the same packet on the same channel, or when the gap detector backfills a time window that overlaps with live ingest, duplicate rows may exist. Deduplication happens at **query time**:

```sql
-- Canonical dedup query for raw_packets
SELECT DISTINCT ON (sniffer_id, ts, mac_address, pdu_type)
    ts, sniffer_id, mac_address, address_type, rssi, channel, pdu_type, advdata, backfilled
FROM raw_packets
WHERE ...
ORDER BY sniffer_id, ts, mac_address, pdu_type, backfilled DESC;
-- backfilled DESC: prefer backfilled rows over live rows
-- (backfilled rows may have richer metadata from C07 Deep Parser)
```

**Rationale:**
- Insert-time unique indexes would reject gap detector backfill rows (which replay packets that may already exist).
- `DISTINCT ON (sniffer_id, ts, mac_address, pdu_type)` provides the granularity needed to identify duplicates while preserving all observations from different sniffers.
- The `pdu_type` field in the dedup key prevents different PDU types at the same microsecond from being collapsed — this was the key refinement in D-10.

#### 3.3.3 Index Strategy

| Index | Columns | Purpose |
|-------|---------|---------|
| `idx_raw_packets_mac_ts` | `(mac_address, ts DESC)` | Device lookup: "all packets from this MAC, most recent first" |
| `idx_raw_packets_sniffer_ts` | `(sniffer_id, ts DESC)` | Per-sniffer time-range queries, gap detection |
| (TimescaleDB auto) | `(ts DESC)` | Hypertable native time-partitioning; chunk exclusion on time-range filters |

No index on `pdu_type` alone — it is used only in the `DISTINCT ON` key, not for filtering.

#### 3.3.4 Compression Strategy

`raw_packets` chunks older than 7 days are automatically compressed by TimescaleDB. Per the TimescaleDB documentation [2], compression policies schedule automatic background compression of chunks after they reach a given age:

```sql
ALTER TABLE raw_packets SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'mac_address',
    timescaledb.compress_orderby = 'ts DESC'
);

SELECT add_compression_policy('raw_packets',
    compress_after => INTERVAL '7 days',
    if_not_exists => TRUE);
```

**Compression parameters:**
- `segmentby = 'mac_address'`: groups rows by MAC address before compression — enables efficient per-device queries on compressed chunks without full decompression.
- `orderby = 'ts DESC'`: stores newest rows first within each compressed segment — enables fast LIMIT queries ("most recent N packets for this MAC").

**Expected compression ratio:** 8:1 to 15:1 for raw packet data (BLE advertising payloads are highly repetitive). A 25 GB PCAP retention window translates to approximately 2–3 GB of compressed hypertable storage.

#### 3.3.5 Retention Policy

Per the TimescaleDB documentation [2], retention policies automatically drop chunks older than a specified interval:

```sql
SELECT add_retention_policy('raw_packets',
    drop_after => INTERVAL '90 days',
    if_not_exists => TRUE);
```

Chunks older than 90 days are automatically dropped. This is **3× the PCAP retention window** (14 days), providing 76 days of queryable history beyond what's recoverable from PCAP. If longer-term analysis is needed, the PCAP archive (source of truth) can be re-parsed by C07 Deep Parser.

#### 3.3.6 Chunk Sizing

Per the TimescaleDB documentation [2], hypertables partition data into chunks for scalability. The `chunk_time_interval` controls the time range each chunk covers:

```sql
SELECT create_hypertable('raw_packets', 'ts',
    chunk_time_interval => INTERVAL '1 hour',
    if_not_exists => TRUE);
```

**1-hour chunks** are sized for the expected data rate: at 1000 packets/sec per sniffer with 4 sniffers, a 1-hour chunk holds approximately 14.4 million rows. At an estimated 80 bytes per row (uncompressed), this is ~1.1 GB per hour. This keeps chunks small enough for efficient compression jobs while large enough to avoid excessive chunk count.

### 3.4 device_summary — Materialized Device State

`device_summary` is a per-MAC-address aggregate table maintained by:
1. **Continuous aggregate refresh** — updates `total_count`, `last_seen`
2. **Scheduled `classify_residency()` calls** — updates `residency_class`, `last_classified`
3. **C08 ML Enrichment** — updates `enrichment_data` JSONB

| Column | Type | Description |
|--------|------|-------------|
| `mac_address` | `BYTEA` | Primary key. 6-byte BD_ADDR. |
| `address_type` | `SMALLINT` | Most recently observed address type for this MAC. |
| `first_seen` | `TIMESTAMPTZ` | Earliest observation timestamp for this MAC. |
| `last_seen` | `TIMESTAMPTZ` | Most recent observation timestamp for this MAC. |
| `total_count` | `BIGINT` | Cumulative observation count across all sniffers. |
| `distinct_days` | `INT` | Number of distinct calendar days this MAC has been observed. |
| `residency_class` | `TEXT` | One of: `'unknown'`, `'new'`, `'lost'`, `'resident'`, `'frequent'`, `'transient'`. |
| `last_classified` | `TIMESTAMPTZ` | When `classify_residency()` was last run for this MAC. |
| `enrichment_data` | `JSONB` | Structured enrichment data from C08 ML, e.g., `{"classes": ["apple_continuity", "battery_service_device"]}`. |

**Note on MAC randomization:** The `address_type` column distinguishes public (0) from random (1) addresses. Devices using BLE privacy (random resolvable/non-resolvable addresses) will have multiple `device_summary` rows — one per random address. Cross-address device correlation is deferred to a future release. The C10 Frontend displays a warning badge for `address_type = 1` entries in the residency view.

### 3.5 device_enrichment — Deep Packet Enrichment Data

Populated by C07 Deep Parser → C08 ML Enrichment pipeline. One row per unique `(mac_address, observed_ts, sniffer_id)` tuple.

| Column | Type | Description |
|--------|------|-------------|
| `id` | `BIGSERIAL` | Synthetic primary key. |
| `mac_address` | `BYTEA` | Observed device MAC. |
| `observed_ts` | `TIMESTAMPTZ` | Timestamp from the original packet. |
| `sniffer_id` | `SMALLINT` | Sniffer that captured the packet. |
| `local_name` | `TEXT` | BLE device name from AD type 0x08 or 0x09 (Complete/Shortened Local Name). |
| `service_uuids_16` | `TEXT[]` | Array of 16-bit service UUIDs observed in advertising data (e.g., `{"180D","180F"}`). |
| `service_uuids_128` | `TEXT[]` | Array of 128-bit service UUIDs observed. |
| `manufacturer_id` | `INT` | Bluetooth SIG Company Identifier from manufacturer-specific AD type 0xFF. |
| `manufacturer_data` | `BYTEA` | Associated manufacturer data payload. |
| `tx_power` | `SMALLINT` | TX Power Level from AD type 0x0A (dBm). |
| `flags` | `SMALLINT` | BLE Flags from AD type 0x01. |
| `raw_advdata` | `BYTEA` | Full raw advertising data bytes (for debugging/reprocessing). |

### 3.6 ingest_gaps — Gap Tracking

Populated by C06 Gap Detector. Tracks time buckets where `raw_packets` data is missing.

| Column | Type | Description |
|--------|------|-------------|
| `id` | `BIGSERIAL` | Synthetic primary key. |
| `sniffer_id` | `SMALLINT` | Affected sniffer. |
| `bucket_start` | `TIMESTAMPTZ` | Start of the gap window. |
| `bucket_end` | `TIMESTAMPTZ` | End of the gap window. |
| `status` | `TEXT` | `'open'` (detected, not yet attempted), `'backfilling'` (in progress), `'closed'` (successfully backfilled), `'failed'` (permanent failure). |
| `retry_count` | `INT` | Number of backfill attempts. Maximum 3. |
| `first_detected` | `TIMESTAMPTZ` | When the gap was first identified. |
| `last_attempted` | `TIMESTAMPTZ` | When the last backfill attempt was made. |
| `rows_backfilled` | `INT` | Number of rows successfully backfilled (NULL if still open). |
| `last_error` | `TEXT` | Error message from the last failed attempt. |

**UNIQUE constraint** on `(sniffer_id, bucket_start)` prevents duplicate gap records for the same sniffer and time window.

### 3.7 sniffers — Sniffer Registry

Static configuration table. Rows are inserted once during deployment and rarely change.

| Column | Type | Description |
|--------|------|-------------|
| `sniffer_id` | `SMALLINT` | Primary key. 1–4 for v1. |
| `name` | `TEXT` | Human-readable name (`ut1`, `nrf1`, `nrf2`, `nrf3`). Unique. |
| `type` | `TEXT` | Sniffer type (`ubertooth`, `nrf`). |
| `device_path` | `TEXT` | udev symlink path (`/dev/tianer/ubertooth0`, `/dev/tianer/nrf0`, etc.). |
| `enabled` | `BOOLEAN` | Whether this sniffer is actively used. Disabled sniffers are not started by C12 orchestration. |
| `created_at` | `TIMESTAMPTZ` | Row creation timestamp. |

### 3.8 sniffer_heartbeat — Sniffer Liveness

Updated every 30 seconds by the sniffer heartbeat process. Used by C06 Gap Detector to determine whether a sniffer was running during a potentially-gapped time window.

| Column | Type | Description |
|--------|------|-------------|
| `sniffer_id` | `SMALLINT` | Primary key. References `sniffers.sniffer_id`. |
| `ts` | `TIMESTAMPTZ` | Last heartbeat timestamp. |
| `status` | `TEXT` | `'running'` or `'stopped'`. |

**Note on heartbeat fallback (D-11):** The local file at `/var/lib/tianer/heartbeat/<name>.ts` is the **primary** heartbeat source. The `sniffer_heartbeat` table is the **secondary**. If the database is down during a heartbeat window, the local file persists; on recovery, the DB table is backfilled from the local file. This prevents DB outages from causing false-positive gap detection.

### 3.9 _migrations — Schema Version Tracking

Tracks which migration files have been applied. This is the **mechanism for idempotent schema deployment**.

| Column | Type | Description |
|--------|------|-------------|
| `name` | `TEXT` | Primary key. Migration filename (e.g., `'0001_init'`). |
| `applied_at` | `TIMESTAMPTZ` | When the migration was applied. Default: `NOW()`. |

### 3.10 Continuous Aggregate: device_5min_buckets

TimescaleDB continuous aggregate that pre-computes per-MAC, per-5-minute-bucket statistics from `raw_packets`. Per the TimescaleDB documentation [2], continuous aggregates automatically refresh on a configurable schedule, keeping pre-computed aggregates current:

```sql
CREATE MATERIALIZED VIEW device_5min_buckets
WITH (timescaledb.continuous) AS
SELECT
    mac_address,
    time_bucket('5 minutes', ts) AS bucket,
    COUNT(*)                AS packet_count,
    AVG(rssi)::SMALLINT     AS avg_rssi,
    MIN(rssi)               AS min_rssi,
    MAX(rssi)               AS max_rssi,
    array_agg(DISTINCT sniffer_id) AS sniffers
FROM raw_packets
GROUP BY mac_address, bucket
WITH NO DATA;
```

**Refresh policy:**
```sql
SELECT add_continuous_aggregate_policy('device_5min_buckets',
    start_offset => INTERVAL '1 hour',
    end_offset   => INTERVAL '5 minutes',
    schedule_interval => INTERVAL '1 minute',
    if_not_exists => TRUE);
```

- `start_offset = 1 hour`: Re-refresh up to 1 hour back (covers late-arriving backfill data).
- `end_offset = 5 minutes`: Exclude the most recent 5 minutes (avoids refreshing incomplete buckets).
- `schedule_interval = 1 minute`: Refresh every minute — keeps aggregates current with sub-2-minute staleness.

### 3.11 Continuous Aggregate Dependency Graph

```
raw_packets (hypertable)
    │
    │ time_bucket('5 minutes', ts)
    │ GROUP BY mac_address, bucket
    │
    ▼
device_5min_buckets (continuous aggregate)
    │
    │ Used by:
    ├── C09 API: /api/devices/{mac}/timeline endpoint
    ├── C11 Grafana: per-device-drilldown dashboard
    ├── C11 Grafana: device-explorer dashboard
    │
    │ Feeds (indirectly):
    └── classify_residency() function
            │
            │ Updates:
            ▼
        device_summary.residency_class
```

### 3.12 Residency Classifier Function

`classify_residency()` is a PL/pgSQL function that classifies a device's residency pattern based on its observation history. Called by the C09 API on a schedule (e.g., hourly) or on-demand for individual devices.

**Classification rules:**

| Rule | Condition | Classification |
|------|-----------|---------------|
| No data | `first_seen IS NULL` | `'unknown'` |
| First seen recently | `first_seen > NOW() - INTERVAL '24 hours'` | `'new'` |
| Gone for a while | `last_seen < NOW() - INTERVAL '14 days'` | `'lost'` |
| High-frequency regular | ≥ 5 distinct days AND ≥ 700 observations in last 7 days | `'resident'` |
| Medium-frequency regular | ≥ 3 distinct days in last 7 days | `'frequent'` |
| Low-frequency | Everything else | `'transient'` |

**Threshold rationale:**
- **700 observations in 7 days** ≈ 100/day ≈ one observation every ~15 minutes. This captures always-present devices like smart home hubs, TV streamers, and environmental sensors.
- **5 distinct days in 7 days** ensures the device is actually present regularly, not just bursty on a couple of days.
- **3 distinct days** catches devices that are present most days but with lower frequency (e.g., a fitness tracker that only advertises when moving).

These thresholds are **configurable** in the function body and should be tuned after observing real-world data.

**Full function (from inception.md §8.7, refined):**

```sql
CREATE OR REPLACE FUNCTION classify_residency(
    p_mac BYTEA,
    p_now TIMESTAMPTZ DEFAULT NOW()
) RETURNS TEXT LANGUAGE plpgsql AS $$
DECLARE
    v_first      TIMESTAMPTZ;
    v_last       TIMESTAMPTZ;
    v_days_7     INT;
    v_packets_7  BIGINT;
BEGIN
    SELECT first_seen, last_seen INTO v_first, v_last
    FROM device_summary WHERE mac_address = p_mac;

    IF v_first IS NULL THEN
        RETURN 'unknown';
    END IF;

    IF v_first > p_now - INTERVAL '24 hours' THEN
        RETURN 'new';
    END IF;

    IF v_last < p_now - INTERVAL '14 days' THEN
        RETURN 'lost';
    END IF;

    SELECT COUNT(DISTINCT date_trunc('day', ts)), COUNT(*)
        INTO v_days_7, v_packets_7
    FROM raw_packets
    WHERE mac_address = p_mac
      AND ts > p_now - INTERVAL '7 days';

    IF v_days_7 >= 5 AND v_packets_7 >= 700 THEN
        RETURN 'resident';
    END IF;

    IF v_days_7 >= 3 THEN
        RETURN 'frequent';
    END IF;

    RETURN 'transient';
END $$;
```

**Performance note:** The subquery against `raw_packets` for the 7-day window benefits from TimescaleDB chunk exclusion (only scans chunks within the 7-day range) and the `idx_raw_packets_mac_ts` index.

---

## 4. Low-Level Architecture (LLA)

### 4.1 Migration Numbering and Application Protocol

All schema changes go through numbered SQL migration files in `db/migrations/`. Migrations are applied in **lexicographic order** (which matches numeric order for zero-padded filenames). Once merged to `main`, migration files are **immutable** — new changes require new migration files.

#### 4.1.1 Migration File Inventory

| File | Purpose | Tables Created | Other Changes |
|------|---------|---------------|---------------|
| `0001_init.sql` | Core schema | `_migrations`, `sniffers`, `sniffer_heartbeat`, `raw_packets`, `device_summary`, `ingest_gaps` | Creates `bluetooth` schema, enables `timescaledb` extension, creates hypertable, creates indexes |
| `0002_continuous_aggregates.sql` | Continuous aggregates | `device_5min_buckets` (materialized view) | Adds refresh policy |
| `0003_compression_policies.sql` | Storage management | — | Enables compression on `raw_packets`, adds compression and retention policies |
| `0004_residency_classifier.sql` | Residency logic | — | Creates `classify_residency()` function |
| `0005_device_enrichment.sql` | Deep parser output | `device_enrichment` | Creates indexes |

#### 4.1.2 Migration Application Protocol

`db/apply-migrations.sh` (invoked by `make db-up` and during deployment):

```
┌──────────────────────────────────────────────────────┐
│              apply-migrations.sh                      │
│                                                      │
│  1. Connect as tianer role to tianer database        │
│     using password from EnvironmentFile              │
│                                                      │
│  2. Read _migrations table for applied filenames     │
│                                                      │
│  3. Scan db/migrations/*.sql in lexicographic order  │
│                                                      │
│  4. For each migration NOT in _migrations:            │
│     ┌─────────────────────────────────────────────┐  │
│     │ BEGIN TRANSACTION                           │  │
│     │   Execute migration SQL                      │  │
│     │   INSERT INTO _migrations (name) VALUES (...) │  │
│     │ COMMIT                                      │  │
│     └─────────────────────────────────────────────┘  │
│     On failure: ROLLBACK transaction. Log error.     │
│     Exit with code 1.                                │
│                                                      │
│  5. Report: "Applied N migrations. Database is at     │
│     migration M."                                    │
└──────────────────────────────────────────────────────┘
```

**Key properties:**
- **One migration, one transaction.** If a migration fails, the entire migration is rolled back, and `_migrations` is not updated. The next run will re-attempt the same migration.
- **Idempotent SQL.** Every migration uses `IF NOT EXISTS` / `IF EXISTS` guards so that re-running a partially-applied migration is safe.
- **No downgrades.** Migrations are forward-only. To undo a schema change, create a new migration that performs the reverse operation. (This is the standard approach for production databases where downgrading is rarely needed and forward-fixing is safer.)
- **`SET search_path TO bluetooth;`** is the first statement in every migration file. This prevents accidental object creation in the `public` schema.

#### 4.1.3 Migration File Template

```sql
-- Migration: 0001_init
-- Description: Core schema for v1 Bluetooth module
-- Depends on: None (first migration)
-- Applied by: C02 apply-migrations.sh

SET search_path TO bluetooth;

-- ... DDL statements with IF NOT EXISTS guards ...

INSERT INTO _migrations (name) VALUES ('0001_init')
    ON CONFLICT DO NOTHING;
```

#### 4.1.4 Migration Runner Prerequisites

The migration runner requires:
1. PostgreSQL container running and healthy (reachable at `127.0.0.1:5432`)
2. `TIANER_DB_PASSWORD` available from `/etc/tianer/secrets/db_password`
3. `tianer` role with LOGIN and DDL privileges on the `tianer` database
4. `db/migrations/` directory readable

### 4.2 Role Creation and Permission Grants

Database roles are initialized once by `db/roles/init-roles.sql`, executed by `deploy/scripts/install-postgres.sh` during C01 host bootstrap. Per the PostgreSQL 17 CREATE ROLE documentation [1], roles are defined at the database cluster level and can be configured with LOGIN, password, and privilege attributes:

```sql
-- Main service role: owns the schema, can DDL and DML.
CREATE ROLE tianer WITH LOGIN PASSWORD :'tianer_pw';
CREATE DATABASE tianer OWNER tianer;

-- Read-only role used by Grafana.
CREATE ROLE tianer_grafana WITH LOGIN PASSWORD :'grafana_pw';
GRANT CONNECT ON DATABASE tianer TO tianer_grafana;
-- Inside tianer database:
GRANT USAGE ON SCHEMA bluetooth TO tianer_grafana;
ALTER DEFAULT PRIVILEGES IN SCHEMA bluetooth
    GRANT SELECT ON TABLES TO tianer_grafana;

-- Read-only role used by the API for queries that don't need write access.
CREATE ROLE tianer_ro WITH LOGIN PASSWORD :'ro_pw';
GRANT CONNECT ON DATABASE tianer TO tianer_ro;
GRANT USAGE ON SCHEMA bluetooth TO tianer_ro;
ALTER DEFAULT PRIVILEGES IN SCHEMA bluetooth
    GRANT SELECT ON TABLES TO tianer_ro;

-- Write-only role for ingest bridge and gap detector streams.
-- Can INSERT and COPY but cannot SELECT or DDL.
CREATE ROLE tianer_writer WITH LOGIN PASSWORD :'writer_pw';
GRANT CONNECT ON DATABASE tianer TO tianer_writer;
GRANT USAGE ON SCHEMA bluetooth TO tianer_writer;
GRANT INSERT ON ALL TABLES IN SCHEMA bluetooth TO tianer_writer;
ALTER DEFAULT PRIVILEGES IN SCHEMA bluetooth
    GRANT INSERT ON TABLES TO tianer_writer;
```

**Post-migration grants:** After new tables are added via migration (e.g., `device_enrichment` in `0005`), the migration file must explicitly grant appropriate privileges to `tianer_grafana`, `tianer_ro`, and `tianer_writer` for the new table:

```sql
-- In 0005_device_enrichment.sql:
GRANT SELECT ON device_enrichment TO tianer_grafana, tianer_ro;
GRANT INSERT ON device_enrichment TO tianer_writer;
```

---

## 5. Inter-Component Contracts

### 5.1 Contract DB-READ: Grafana → PostgreSQL

| Property | Value |
|----------|-------|
| **Contract ID** | `DB-READ` |
| **From** | C11 Grafana Dashboards |
| **To** | C02 Database (PostgreSQL) |
| **Role** | `tianer_grafana` |
| **Privileges** | CONNECT on `tianer` database, USAGE on `bluetooth` schema, SELECT on all tables |
| **Auth** | scram-sha-256 password from `/etc/tianer/secrets/grafana_db_password` |
| **Connection** | TCP to `tianer-postgres:5432` on `tianer-net` bridge |
| **Pool** | Grafana built-in PostgreSQL connection pool (configurable, default 5 max open) |
| **Query pattern** | Read-only SELECT queries for dashboard panels. Mostly aggregate queries against `raw_packets` (via continuous aggregates) and `device_summary`. |
| **SLA** | Panel queries must complete within 10 seconds (Grafana default query timeout). |

**Example query (device-explorer dashboard):**
```sql
SELECT
    encode(mac_address, 'hex') AS mac,
    residency_class,
    total_count,
    last_seen
FROM bluetooth.device_summary
WHERE last_seen > NOW() - INTERVAL '1 hour'
ORDER BY last_seen DESC
LIMIT 100;
```

### 5.2 Contract DB-WRITE: Ingest Bridge → PostgreSQL

| Property | Value |
|----------|-------|
| **Contract ID** | `DB-WRITE` |
| **From** | C05 Ingest Bridge, C06 Gap Detector, C08 ML Enrichment |
| **To** | C02 Database (PostgreSQL) |
| **Role** | `tianer_writer` |
| **Privileges** | CONNECT on `tianer` database, USAGE on `bluetooth` schema, INSERT on all tables |
| **Auth** | scram-sha-256 password from `TIANER_DB_PASSWORD_FILE` |
| **Connection** | TCP to `tianer-postgres:5432` on `tianer-net` bridge |
| **Protocol (C05)** | `pqxx::stream_to` (PostgreSQL COPY protocol) — fast bulk insert |
| **Protocol (C06/C08)** | Parameterized INSERT statements via psycopg 3.1 async |
| **SLA** | C05: 1000 rows/sec P95 ingest latency < 5 seconds. C06: gap backfill within 5 minutes of DB recovery. |

**COPY protocol safety:** The PostgreSQL `COPY` protocol is **parameterized by design** — it streams binary or text data rows, not SQL statements. It is immune to SQL injection because no SQL parsing occurs for the row data. This is important because `tianer_writer` is a restricted role that cannot execute arbitrary SQL, only INSERT and COPY.

### 5.3 Contract DB-API: REST API → PostgreSQL

| Property | Value |
|----------|-------|
| **Contract ID** | `DB-API` |
| **From** | C09 REST API (FastAPI) |
| **To** | C02 Database (PostgreSQL) |
| **Role (read)** | `tianer_ro` — for all `/api/devices`, `/api/alerts`, `/api/stats` read endpoints |
| **Role (write)** | `tianer` — for continuous aggregate refresh, residency classification refresh, and any future write endpoints |
| **Privileges (read)** | CONNECT, USAGE, SELECT |
| **Privileges (write)** | CONNECT, USAGE, ALL on `bluetooth` schema |
| **Auth** | scram-sha-256. Read pool uses `tianer_ro` password; write pool uses `tianer` password. |
| **Connection** | TCP to `tianer-postgres:5432` on `tianer-net` bridge |
| **Pooling** | `asyncpg` connection pool: 5 connections for writes (`tianer`), 10 connections for reads (`tianer_ro`) |
| **Query pattern** | Parameterized queries only. No string concatenation for SQL. |
| **SLA** | P95 endpoint latency < 200 ms for cached aggregates. P95 < 500 ms for raw packet queries. |

---

## 6. Failure Modes & Recovery

### 6.1 Failure Catalog

| # | Failure Mode | Detection | Impact | Recovery |
|---|-------------|-----------|--------|----------|
| **F-C02-1** | **PostgreSQL container crashes** | Podman detects exit code ≠ 0. `systemctl --user is-failed tianer-postgres`. All consumers get connection refused. | All DB-dependent services fail. C05 Ingest Bridge buffers packets (300K rows, ~60MB RAM). C03 continues writing PCAP. No PCAP data loss. | Podman auto-restarts (systemd `Restart=on-failure`, `RestartSec=5s`). PostgreSQL performs WAL recovery on startup (automatic, typically < 5 seconds). Gap detector backfills any missed buckets from PCAP. |
| **F-C02-2** | **DB connection loss (network)** | Consumers receive `pqxx::broken_connection`, `psycopg.OperationalError`, or connection timeout. | Same as F-C02-1 but container may still be running (e.g., `tianer-net` bridge issue). | Consumers implement exponential backoff reconnect: 1s, 2s, 4s, 8s, max 30s. C05 buffers packets in memory (300K row limit). Gap detector backfills on recovery. |
| **F-C02-3** | **Corrupt hypertable chunk** | TimescaleDB built-in chunk verification (`timescaledb_information.chunks` view). Query errors on affected chunks. | Queries spanning the corrupted chunk fail. Other chunks are unaffected. | Identify corrupted chunk via `SELECT * FROM timescaledb_information.chunks WHERE compression_status = 'Compressed'` and verify. Drop corrupted chunk: `SELECT drop_chunks('raw_packets', older_than => ...)`. Restore from pg_basebackup, or accept data loss for that window (recoverable from PCAP). |
| **F-C02-4** | **Migration failure** | `apply-migrations.sh` exit code ≠ 0. Error logged to stderr. | Database remains at previous migration state. New tables/columns not created. Dependent components (C07, C08 for `device_enrichment`) fail. | Migration runs in a transaction — failed migration is fully rolled back. Operator fixes the migration SQL. Re-run `apply-migrations.sh`. The `_migrations` table was not updated, so the fixed migration will be applied. |
| **F-C02-5** | **Disk full (V06 volume)** | `tianer_host_disk_usage_percent > 80%` (C01 host metric). PostgreSQL writes start failing. | PostgreSQL may refuse new writes (INSERT/COPY). WAL cannot be written. **Database may enter read-only mode or panic.** Ingest buffer fills; gap detector records gaps. PCAP capture continues (different volume). | Alert at 80% (C01). At 90%, C04 emergency purge of old uncompressed PCAP files (different volume). Increase V06 size if possible: `podman volume create` supports resizing on some filesystems. Manual `VACUUM FULL` to reclaim bloat. Last resort: temporarily disable TimescaleDB compression to reduce write amplification. |
| **F-C02-6** | **Continuous aggregate refresh fails** | TimescaleDB logs job failure. `device_5min_buckets` data becomes stale. | Dashboards show outdated aggregate data. `device_summary.total_count` and `last_seen` don't update. `classify_residency()` uses stale data. Ingest still works — `raw_packets` is the source of truth. | Manual refresh: `CALL refresh_continuous_aggregate('device_5min_buckets', NULL, NULL);`. Check `timescaledb_information.jobs` for job status. Restart the background worker if needed. |
| **F-C02-7** | **Compression job fails** | TimescaleDB logs compression failure. Chunks older than 7 days remain uncompressed. | Storage consumption increases over time. Query performance unaffected (uncompressed chunks are faster to query, just larger). | Manual compression: `SELECT compress_chunk(i) FROM show_chunks('raw_packets', older_than => INTERVAL '7 days') i;`. Check for lock conflicts. |
| **F-C02-8** | **Retention policy fails** | TimescaleDB logs retention failure. Old chunks are not dropped. | Storage grows indefinitely. Potential disk full (→ F-C02-5). | Manual drop: `SELECT drop_chunks('raw_packets', older_than => INTERVAL '90 days');`. Verify policy is still active: `SELECT * FROM timescaledb_information.jobs WHERE proc_name = 'policy_retention';`. |
| **F-C02-9** | **Full database loss** | All tables missing. `pg_isready` returns "no database". V06 volume corrupted or deleted. | Complete platform outage. No DB services function. PCAP files on V02 (separate volume) are unaffected — they are the source of truth. | **Recovery path 1 (preferred):** Restore from `pg_basebackup` (nightly cron job). Reapply any migrations newer than the backup. Gap detector backfills missed data from PCAP (up to 14 days of PCAP retention). **Recovery path 2 (full rebuild):** Drop and recreate database. Apply all migrations. Replay all PCAP files through C07 Deep Parser → C05-style ingest. This recovers all data from the PCAP archive (source of truth). Time to recover: proportional to PCAP volume (hours for TB-scale). |
| **F-C02-10** | **WAL corruption** | PostgreSQL fails to start. Log shows "invalid magic number" or "WAL segment removed". | Database won't start. | `pg_resetwal` as last resort (data loss risk). If container image is intact but data is corrupt, restore from backup (F-C02-9 recovery path 1). |

### 6.2 Connection Recovery — Exponential Backoff Reconnect

All consumers (C05, C06, C09, C11) implement the same reconnect strategy when the database connection is lost:

```
Attempt 1: wait 1s  →  retry
Attempt 2: wait 2s  →  retry
Attempt 3: wait 4s  →  retry
Attempt 4: wait 8s  →  retry
Attempt 5: wait 16s →  retry
Attempt 6+: wait 30s (cap) → retry indefinitely
```

**Maximum backoff cap:** 30 seconds. This ensures that after a brief DB restart (typically < 10 seconds on CM5), consumers reconnect within 1–2 seconds on average.

**Infinite retry:** Consumers never give up on the database. The PCAP archive (V02) absorbs all data during extended outages. Gap detector (C06) backfills on recovery.

### 6.3 Recovery Service Dependency

| Recovery Agent | Failure It Handles | Is It Available After Reboot? |
|---------------|-------------------|-------------------------------|
| Podman auto-restart (systemd) | Container crash (F-C02-1) | Yes — systemd `Restart=on-failure` |
| PostgreSQL WAL recovery | Crash recovery (F-C02-1) | Yes — built-in, automatic |
| `apply-migrations.sh` (re-run) | Migration failure (F-C02-4) | Manual — operator intervention |
| C06 Gap Detector | Missed ingest buckets (F-C02-1, F-C02-2) | Yes — auto-starts, backfills from PCAP |
| C04 PCAP Rotation | Disk full (F-C02-5) | Yes — auto-triggered by timer |
| `pg_basebackup` (nightly cron) | Full DB loss (F-C02-9) | Manual — operator triggers restore |
| TimescaleDB background worker | Compression/retention/refresh failures (F-C02-6,7,8) | Yes — auto-restarts with PostgreSQL |

---

## 7. Observability

### 7.1 Database Metrics

Metrics are collected by C13 (Observability) and exposed as Prometheus metrics for C11 Grafana dashboards.

| Metric | Source | Collection Method | Labels | Alert Threshold |
|--------|--------|------------------|--------|-----------------|
| `tianer_db_up` | `pg_isready` | C13 host script every 15s | — | == 0 → **critical** |
| `tianer_db_size_bytes` | `pg_database_size('tianer')` | SQL query every 5 min | — | Warning > 4.5 GB (90% of 5 GB budget) |
| `tianer_db_connections_active` | `pg_stat_activity` count | SQL query every 1 min | `state` (active, idle, idle_in_transaction) | > 40 (80% of max_connections=50) → **warning** |
| `tianer_table_size_bytes` | `pg_total_relation_size()` | SQL query every 1 hour | `table` (raw_packets, device_summary, etc.) | — |
| `tianer_hypertable_chunks_total` | `timescaledb_information.chunks` count | SQL query every 1 hour | `hypertable` | — |
| `tianer_hypertable_compressed_chunks` | `timescaledb_information.chunks` count (filtered) | SQL query every 1 hour | `hypertable` | — |
| `tianer_cagg_refresh_last_duration_ms` | `timescaledb_information.job_stats` | SQL query every 5 min | `cagg` (device_5min_buckets) | > 60,000 (1 min) → **warning** |
| `tianer_cagg_refresh_last_status` | `timescaledb_information.job_stats` | SQL query every 5 min | `cagg` | `!= 'Success'` → **critical** |
| `tianer_compression_job_last_status` | `timescaledb_information.job_stats` | SQL query every 1 hour | — | `!= 'Success'` → **warning** |
| `tianer_retention_job_last_status` | `timescaledb_information.job_stats` | SQL query every 1 hour | — | `!= 'Success'` → **warning** |
| `tianer_query_slow_queries_1h` | `pg_stat_statements` | SQL query every 15 min | — | — |
| `tianer_dead_tuples_total` | `pg_stat_user_tables` | SQL query every 15 min | `table` | > 100,000 per table → schedule VACUUM |
| `tianer_wal_bytes_written_1h` | `pg_stat_wal` | SQL query every 1 hour | — | — |
| `tianer_db_migration_state` | `_migrations` table | SQL query at startup + every 1 hour | `last_applied` | Expected migration not present → **critical** |

### 7.2 Query Performance Monitoring

`pg_stat_statements` is enabled in `postgresql.conf`. Per the PostgreSQL 17 documentation [1], this module provides per-query statistics including calls, total time, mean time, and cache hit ratio:

```ini
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.track = all
```

This provides per-query performance data: total calls, total time, mean time, rows returned, cache hit ratio. The C11 Grafana "pipeline-health" dashboard includes a panel showing the top 10 slowest queries by mean execution time from `pg_stat_statements`.

### 7.3 TimescaleDB Background Job Monitoring

TimescaleDB background jobs (compression, retention, continuous aggregate refresh) are monitored via `timescaledb_information.job_stats`. Stale or failed jobs produce alerts:

```sql
-- Check if any background job has failed
SELECT job_id, proc_name, last_run_status, last_run_started_at, last_run_duration
FROM timescaledb_information.job_stats
WHERE last_run_status != 'Success';
```

### 7.4 Structured Logging

PostgreSQL container logs are captured by Podman and written to journald. All logs are prefixed with the `TIANER` keyword in the standard structured format:

```
TIANER | {"ts":"2026-06-09T14:30:00Z","level":"INFO","component":"postgres","msg":"database system is ready to accept connections"}
TIANER | {"ts":"2026-06-09T14:30:05Z","level":"WARN","component":"postgres","msg":"checkpoint starting: time"}
TIANER | {"ts":"2026-06-09T14:45:00Z","level":"ERROR","component":"postgres","msg":"could not write to file \"pg_wal/...\": No space left on device"}
```

### 7.5 Health Check

The C09 API `/api/health` endpoint returns database status:

```json
{
    "status": "ok",
    "db": "ok",
    "db_migration": "0005_device_enrichment",
    "db_connections": 23,
    "db_size_mb": 1250
}
```

The `db` field is determined by a lightweight query:
```sql
SELECT 1;
```

If the query fails, `db` is `"error"` and `status` is `"degraded"`.

### 7.6 Alert Summary

| Alert Name | Trigger | Severity | Action |
|------------|---------|----------|--------|
| `TianerDBDown` | `tianer_db_up == 0` for > 30s | **Critical** | Investigate container status; check V06 disk space; restart if needed |
| `TianerDBSizeWarning` | `tianer_db_size_bytes > 4.5 GB` | **Warning** | Check retention policy; verify compression is working; consider increasing V06 size |
| `TianerDBConnectionHigh` | `tianer_db_connections_active > 40` | **Warning** | Check for connection leaks; restart leaking service |
| `TianerCaggRefreshFailed` | `tianer_cagg_refresh_last_status != 'Success'` | **Critical** | Manual refresh; investigate TimescaleDB background worker |
| `TianerCompressionFailed` | `tianer_compression_job_last_status != 'Success'` | **Warning** | Check for lock conflicts; manual compression |
| `TianerRetentionFailed` | `tianer_retention_job_last_status != 'Success'` | **Warning** | Manual chunk drop; investigate policy |
| `TianerMigrationStale` | Expected migration not in `_migrations` | **Critical** | Run `apply-migrations.sh` |

---

## 8. Security Considerations

### 8.1 Network Isolation

PostgreSQL listens on **all interfaces inside the container** but the container's port 5432 is published only to `127.0.0.1:5432` on the host via the Quadlet `PublishPort` directive. Containers on the `tianer-net` bridge network can reach it directly at `tianer-postgres:5432`. No external network access is possible.

```
┌─────────────────────────────────────────────────────────┐
│                      HOST (CM5)                          │
│                                                         │
│  127.0.0.1:5432 ← only the host loopback                │
│       ↑                                                 │
│       │ Podman port forwarding                          │
│       │                                                 │
│  ┌────┴──────────────────────────────────┐              │
│  │  tianer-net (internal bridge, no      │              │
│  │  external route)                      │              │
│  │                                       │              │
│  │  tianer-postgres:5432                 │              │
│  │  tianer-platform pod containers       │              │
│  │  tianer-grafana:3000                  │              │
│  └───────────────────────────────────────┘              │
│                                                         │
│  EXTERNAL NETWORK: No route to 5432                     │
└─────────────────────────────────────────────────────────┘
```

### 8.2 Authentication

All application roles use **scram-sha-256** authentication [1]. Passwords are 32-byte random strings generated by `openssl rand -base64 32` (C01 `generate-secrets.sh`). Per the PostgreSQL 17 documentation [1], `scram-sha-256` performs SCRAM-SHA-256 authentication as described in RFC 7677, providing challenge-response password verification without sending the password in plaintext.

**`pg_hba.conf` configuration** (per the PostgreSQL 17 `pg_hba.conf` specification [1]):

```
# TYPE  DATABASE  USER               ADDRESS          METHOD
# Local administration via Unix socket (inside container)
local  all       postgres                             peer
local  tianer    tianer,tianer_grafana,tianer_ro,
                 tianer_writer                        scram-sha-256

# TCP connections (all via tianer-net bridge)
host   tianer    tianer_grafana      tianer-net        scram-sha-256
host   tianer    tianer_ro           tianer-net        scram-sha-256
host   tianer    tianer              tianer-net        scram-sha-256
host   tianer    tianer_writer       tianer-net        scram-sha-256

# Reject everything else
host   all       all                 0.0.0.0/0         reject
host   all       all                 ::/0               reject
```

**Key points:**
- `peer` auth for `postgres` superuser — only the local Unix socket, only as the OS `postgres` user inside the container.
- All application roles use `scram-sha-256` over TCP.
- All non-`tianer-net` connections are rejected.
- The `tianer-net` bridge is internal (NAT-only, no external route). Containers on this network can reach each other but have no path to the internet unless explicitly configured.

### 8.3 Role Least-Privilege Enforcement

| Role | Can SELECT? | Can INSERT/COPY? | Can UPDATE? | Can DELETE? | Can DDL? | Can TRUNCATE? |
|------|-------------|-------------------|-------------|-------------|----------|---------------|
| `tianer_writer` | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ |
| `tianer_ro` | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `tianer_grafana` | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `tianer` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

**Enforcement verification:**
```sql
-- Verify tianer_writer cannot SELECT
SET ROLE tianer_writer;
SELECT * FROM raw_packets LIMIT 1;
-- Expected: ERROR: permission denied for table raw_packets

-- Verify tianer_ro cannot INSERT
SET ROLE tianer_ro;
INSERT INTO sniffers (sniffer_id, name, type) VALUES (99, 'test', 'mock');
-- Expected: ERROR: permission denied for table sniffers

-- Verify tianer_grafana cannot DELETE
SET ROLE tianer_grafana;
DELETE FROM raw_packets WHERE sniffer_id = 1;
-- Expected: ERROR: permission denied for table raw_packets
```

### 8.4 Parameterized Queries Mandate

**All application code that queries PostgreSQL MUST use parameterized queries.** This applies to:

| Consumer | Language | Parameterization Mechanism |
|----------|----------|---------------------------|
| C05 Ingest Bridge | C++ / libpqxx | `pqxx::stream_to` (COPY protocol — inherently parameterized) |
| C06 Gap Detector | Python / psycopg | `cursor.execute("INSERT ... VALUES (%s, %s, ...)", params)` |
| C08 ML Enrichment | Python / psycopg | `cursor.execute("INSERT ... VALUES (%s, %s, ...)", params)` |
| C09 REST API | Python / asyncpg | `await conn.execute("SELECT ... WHERE mac = $1", mac)` |
| C11 Grafana | SQL in dashboard JSON | Grafana's `$mac` template variables are parameterized |

**Prohibited pattern:**
```python
# NEVER DO THIS
mac = request.query_params['mac']
cursor.execute(f"SELECT * FROM raw_packets WHERE mac_address = '{mac}'")
```

**Required pattern:**
```python
mac = request.query_params['mac']
cursor.execute("SELECT * FROM raw_packets WHERE mac_address = %s", (mac,))
```

### 8.5 Secrets Handling

Database passwords are **never** stored in:
- Environment variables visible to other containers
- Configuration files (YAML, JSON, TOML)
- Container images
- Git repository

Passwords are stored in `/etc/tianer/secrets/` (mode 0700, owner `tianer:tianer`), each with mode 0600:

| File | Consumer |
|------|----------|
| `/etc/tianer/secrets/db_password` | `tianer` role (C05, C06, C08, C09 write path, migration runner) |
| `/etc/tianer/secrets/ro_db_password` | `tianer_ro` role (C09 read path) |
| `/etc/tianer/secrets/writer_db_password` | `tianer_writer` role (C05, C06, C08) |
| `/etc/tianer/secrets/grafana_db_password` | `tianer_grafana` role (C11 Grafana) |

Containers receive passwords via `EnvironmentFile` injection in their Quadlet `.container` file:

```
[Container]
EnvironmentFile=/etc/tianer/secrets/db_password.env
```

The `.env` file is a key-value file with one variable:
```
TIANER_DB_PASSWORD=<32-byte-base64-string>
```

This is read by Podman and injected into the container's environment at startup. It is never visible to other containers and never persisted in the container image.

### 8.6 Audit Logging (Post-MVP)

For MVP, PostgreSQL's built-in logging (`log_statement = 'ddl'`, `log_connections = on`, `log_disconnections = on`) provides basic auditability. Post-MVP, consider:
- **`pgaudit` extension** for fine-grained audit logging of DML operations.
- **`pg_stat_statements`** already provides query-level statistics for anomaly detection.

### 8.7 Supply Chain Integrity (Q10)

The PostgreSQL 17 + TimescaleDB 2.23 container image is built from the official `postgres:17-bookworm` image pinned by digest (`@sha256:...`), not by tag (`:latest` or `:17`). The TimescaleDB extension is installed during the container build from the official TimescaleDB apt repository with GPG key verification. This is documented in C14 (Deployment Automation).

---

## 9. Configuration

### 9.1 Environment Variables

| Variable | Default | Purpose | Set By |
|----------|---------|---------|--------|
| `TIANER_DB_HOST` | `tianer-postgres` | Database hostname (container name on tianer-net) | C14 Quadlet `Environment=` |
| `TIANER_DB_PORT` | `5432` | Database port | C14 Quadlet `Environment=` |
| `TIANER_DB_NAME` | `tianer` | Database name | C14 Quadlet `Environment=` |
| `TIANER_DB_USER` | varies by consumer | Role name (`tianer`, `tianer_writer`, etc.) | Consumer-specific Quadlet |
| `TIANER_DB_PASSWORD` | (from secrets file) | Role password | `EnvironmentFile` injection |

### 9.2 postgresql.conf Tuning for Raspberry Pi CM5 (8 GB RAM)

Tuning values are set in the container image's `postgresql.conf` (or via `ALTER SYSTEM`). These are **conservative defaults** appropriate for the CM5 hardware profile. Operators can tune further based on observed performance.

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| `shared_buffers` | `512 MB` | 6.25% of 8 GB RAM. Conservative for mixed workload (ingest + queries). The PostgreSQL documentation recommends 15–25% for dedicated DB servers [1], but Tian'er shares the CM5 with other containers. |
| `effective_cache_size` | `4 GB` | 50% of 8 GB RAM. Informs the query planner about OS page cache. The Linux kernel will use available RAM for file caching beyond `shared_buffers`. |
| `work_mem` | `16 MB` | Per-operation sort/hash memory. With ~10 concurrent query operations, worst-case is ~160 MB. Conservative for the mixed workload. |
| `maintenance_work_mem` | `128 MB` | For VACUUM, CREATE INDEX, ALTER TABLE. 128 MB is sufficient for the data volumes expected. |
| `wal_buffers` | `16 MB` | Default is 1/32 of `shared_buffers`, capped at 16 MB. Sufficient for the write throughput (4K rows/sec at ~80 bytes = ~320 KB/sec). |
| `max_wal_size` | `2 GB` | Maximum WAL size before automatic checkpoint. 2 GB covers heavy ingest periods. |
| `min_wal_size` | `512 MB` | Minimum WAL size to retain. |
| `checkpoint_completion_target` | `0.9` | Spread checkpoint I/O over 90% of the checkpoint interval. Reduces I/O spikes. |
| `random_page_cost` | `1.1` | SSD storage (eMMC). Default 4.0 assumes spinning disk. Lower value encourages index scans. |
| `effective_io_concurrency` | `200` | SSD I/O concurrency. |
| `max_connections` | `50` | Headroom above typical 25–30 connections. |
| `log_statement` | `'ddl'` | Log all DDL for audit trail. Does not log SELECT/INSERT/UPDATE (performance impact). |
| `log_connections` | `on` | Log every connection attempt (audit trail). |
| `log_disconnections` | `on` | Log disconnections (detect connection leaks). |
| `log_checkpoints` | `on` | Log checkpoint activity (WAL monitoring). |
| `shared_preload_libraries` | `'timescaledb,pg_stat_statements'` | Required extensions. TimescaleDB must be preloaded. `pg_stat_statements` for query monitoring. |
| `timescaledb.telemetry_level` | `'off'` | Disable telemetry (privacy). |

### 9.3 pg_hba.conf

As specified in §8.2 Authentication. The file is generated during container image build and is not modified at runtime.

### 9.4 TimescaleDB Configuration

All TimescaleDB policies are configured via SQL in the migration files and can be adjusted without re-creating the database:

| Policy | Migration | Default Value | How to Change |
|--------|-----------|---------------|---------------|
| Hypertable chunk interval | `0001_init.sql` | 1 hour | `SELECT set_chunk_time_interval('raw_packets', INTERVAL '2 hours');` |
| Continuous aggregate refresh interval | `0002_continuous_aggregates.sql` | 1 minute | `SELECT alter_job(job_id, schedule_interval => INTERVAL '2 minutes') FROM timescaledb_information.jobs WHERE proc_name = 'policy_refresh_continuous_aggregate';` |
| Continuous aggregate start offset | `0002_continuous_aggregates.sql` | 1 hour | Via `alter_job` config |
| Continuous aggregate end offset | `0002_continuous_aggregates.sql` | 5 minutes | Via `alter_job` config |
| Compression after | `0003_compression_policies.sql` | 7 days | `SELECT alter_job(job_id, config => '{"compress_after":"14 days"}') FROM ...;` |
| Compression segmentby | `0003_compression_policies.sql` | `mac_address` | `ALTER TABLE raw_packets SET (timescaledb.compress_segmentby = 'sniffer_id');` |
| Retention drop after | `0003_compression_policies.sql` | 90 days | `SELECT alter_job(job_id, config => '{"drop_after":"180 days"}') FROM ...;` |

---

## 10. Test Plan

### 10.1 Unit Tests — pgTAP Schema Verification

`db/tests/` contains pgTAP-style SQL test files that verify the schema, constraints, and indexes. Each test file runs inside a transaction that rolls back at the end.

| Test File | What It Verifies |
|-----------|-----------------|
| `test_schema_exists.sql` | `bluetooth` schema exists and is owned by `tianer`. `timescaledb` extension is installed. |
| `test_tables_exist.sql` | All 8 tables/views exist (`_migrations`, `sniffers`, `sniffer_heartbeat`, `raw_packets`, `device_summary`, `device_enrichment`, `ingest_gaps`, `device_5min_buckets`). |
| `test_raw_packets_hypertable.sql` | `raw_packets` is a hypertable. Chunk interval is 1 hour. Compression is enabled. |
| `test_indexes_exist.sql` | All specified indexes exist on `raw_packets`, `device_summary`, `device_enrichment`, `ingest_gaps`. |
| `test_constraints.sql` | `sniffer_heartbeat.sniffer_id` references `sniffers`. `ingest_gaps.status` has CHECK constraint. `device_enrichment` UNIQUE constraint on `(mac_address, observed_ts, sniffer_id)`. |
| `test_cagg_exists.sql` | `device_5min_buckets` continuous aggregate exists with correct schema. |
| `test_classify_residency.sql` | `classify_residency()` function exists and returns valid classification strings. |
| `test_compression_policy.sql` | Compression policy is active on `raw_packets` with correct `compress_after` interval. |
| `test_retention_policy.sql` | Retention policy is active on `raw_packets` with correct `drop_after` interval. |

### 10.2 Integration Tests

| Test | Description | Acceptance Criteria |
|------|-------------|---------------------|
| `test_migration_apply.sh` | Run `db/apply-migrations.sh` against a clean database. | All 5 migrations applied. Exit code 0. `_migrations` has 5 rows. |
| `test_migration_idempotent.sh` | Re-run `apply-migrations.sh`. | No errors. Exit code 0. `_migrations` still has 5 rows. |
| `test_migration_rollback.sh` | Inject a deliberately-broken migration. | Migration fails. Transaction rolled back. `_migrations` does not include the broken migration. |
| `test_role_permissions.sh` | Verify each role's privileges. | `tianer_writer` can INSERT but not SELECT/UDPATE/DELETE/DROP. `tianer_ro` can SELECT but not INSERT. `tianer_grafana` can SELECT but not INSERT. |
| `test_hypertable_insert.sh` | Insert 10,000 synthetic rows into `raw_packets`. | Rows are distributed across chunks. Query returns correct count. |
| `test_continuous_aggregate.sh` | Insert synthetic data spanning 30 minutes. Wait for cagg refresh. | `device_5min_buckets` contains expected buckets with correct counts. |
| `test_compression_manual.sh` | Manually compress a chunk. Verify compressed status. | Chunk shows as compressed in `timescaledb_information.chunks`. Query still returns same data. |
| `test_residency_classifier.sh` | Insert synthetic data for resident/frequent/transient/new/lost MACs. Call `classify_residency()`. | Each MAC classified correctly per the rules in §3.12. |
| `test_query_time_dedup.sh` | Insert duplicate rows (same `sniffer_id, ts, mac_address, pdu_type`). Run DISTINCT ON query. | Duplicates are collapsed. Distinct count < total count. |
| `test_ingest_gaps_workflow.sh` | Insert gap record. Simulate backfill. Update status to 'closed'. | Full lifecycle: open → backfilling → closed. UNIQUE constraint prevents duplicate gaps. |

### 10.3 Performance Tests

| Test | Target | Measurement |
|------|--------|-------------|
| COPY throughput | 10,000 rows/sec sustained | `pg_stat_statements` + wall clock |
| Continuous aggregate refresh latency | < 2 minutes for 1 hour of data at 4K rows/sec | `timescaledb_information.job_stats.last_run_duration` |
| Query: "last 100 packets for MAC" | < 50 ms P95 with warm cache | `EXPLAIN ANALYZE` |
| Query: "device_5min_buckets for 24 hours" | < 200 ms P95 | `EXPLAIN ANALYZE` |
| Compression ratio | ≥ 8:1 on 7-day-old chunks | `pg_total_relation_size` before/after compression |
| Chunk creation overhead | < 10 ms per chunk creation | Measure during sustained ingest |

### 10.4 CI Integration

`db/tests/` is run as part of `ci/test-all.sh` (step 5: "SQL migrations smoke: apply all migrations to a fresh DB"). The CI pipeline spins up a PostgreSQL 17 + TimescaleDB container via `ci/docker-compose.yml`, applies migrations, and runs the pgTAP test suite.

### 10.5 Acceptance Criteria

1. **Schema integrity:** All 8 tables exist in `bluetooth` schema. All indexes, constraints, and policies are present.
2. **Migrations:** `db/apply-migrations.sh` applies all migrations idempotently. Re-running after a failure succeeds.
3. **Role permissions:** `tianer_writer` cannot SELECT. `tianer_ro` and `tianer_grafana` cannot INSERT, UPDATE, or DELETE. `tianer` can DDL.
4. **Hypertable:** `raw_packets` is a hypertable partitioned by `ts` with 1-hour chunks.
5. **Continuous aggregates:** `device_5min_buckets` refreshes and contains correct aggregates.
6. **Compression:** Chunks older than 7 days are compressed with segmentby=`mac_address`.
7. **Retention:** Chunks older than 90 days are dropped automatically.
8. **Residency classifier:** `classify_residency()` returns correct classifications for all 6 classes.
9. **Query-time dedup:** `DISTINCT ON (sniffer_id, ts, mac_address, pdu_type)` correctly collapses duplicates.
10. **Performance:** Sustained COPY throughput ≥ 10,000 rows/sec. Query latency within targets.

---

## 11. Deployment Notes

### 11.1 Container Image

The PostgreSQL container image is built as part of C14 (Deployment Automation). It uses a multi-stage Dockerfile:

```dockerfile
# Stage 1: Build (install TimescaleDB from apt)
FROM postgres:17-bookworm AS builder
RUN apt-get update && apt-get install -y --no-install-recommends \
    gnupg curl ca-certificates
# Add TimescaleDB GPG key and apt repo
RUN curl -fsSL https://packagecloud.io/timescale/timescaledb/gpgkey | gpg --dearmor -o /usr/share/keyrings/timescaledb.gpg
RUN echo "deb [signed-by=/usr/share/keyrings/timescaledb.gpg] https://packagecloud.io/timescale/timescaledb/ubuntu/ noble main" \
    > /etc/apt/sources.list.d/timescaledb.list
RUN apt-get update && apt-get install -y --no-install-recommends \
    timescaledb-2-postgresql-17

# Stage 2: Runtime (minimal)
FROM postgres:17-bookworm
COPY --from=builder /usr/lib/postgresql/17/lib/timescaledb*.so /usr/lib/postgresql/17/lib/
COPY --from=builder /usr/share/postgresql/17/extension/timescaledb* /usr/share/postgresql/17/extension/
COPY deploy/config/postgresql.conf /etc/postgresql/postgresql.conf
COPY deploy/config/pg_hba.conf /etc/postgresql/pg_hba.conf
```

**Image size target:** Under 200 MB (the `postgres:17-bookworm` base image is ~150 MB; TimescaleDB adds ~15 MB; config files are negligible).

### 11.2 Quadlet Unit

`deploy/containers/tianer-postgres.container` (Quadlet file, deployed by C14):

```ini
[Unit]
Description=Tian'er PostgreSQL 17 + TimescaleDB 2.23
After=tianer-net.network
Requires=tianer-net.network
Before=tianer-platform.pod

[Container]
Image=localhost/tianer-postgres:latest
ContainerName=tianer-postgres
Network=tianer-net
PublishPort=127.0.0.1:5432:5432
Volume=tianer-postgres-data.volume:/var/lib/postgresql/data
EnvironmentFile=/etc/tianer/secrets/db_password.env
Environment=POSTGRES_DB=tianer
Environment=POSTGRES_USER=tianer
Environment=POSTGRES_INITDB_ARGS=--encoding=UTF8 --locale=C.UTF-8

# Security hardening
NoNewPrivileges=true
DropCapability=ALL
AddCapability=CHOWN DAC_OVERRIDE FOWNER SETGID SETUID

# Resource limits
MemoryMax=1G
CPUQuota=200%

[Service]
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=default.target
```

**Key points:**
- `PublishPort=127.0.0.1:5432:5432` — only loopback binding on host.
- `EnvironmentFile` injects the DB password securely.
- `MemoryMax=1G` — caps PostgreSQL at 1 GB RAM. Conservative for CM5 8 GB total.
- `DropCapability=ALL` with minimal adds — only filesystem ownership capabilities needed by PostgreSQL.
- `tianer-postgres-data.volume` — the persistent Podman volume (V06).

### 11.3 Podman Volume Creation

`deploy/containers/tianer-postgres-data.volume` (Quadlet volume definition):

```ini
[Volume]
```

Minimal definition — Quadlet creates a Podman volume with default settings (local driver, overlay filesystem). The volume persists across container restarts and rebuilds.

### 11.4 Install Script Integration

`deploy/scripts/install-postgres.sh` (invoked by `deploy/setup.sh` during C01 bootstrap):

```bash
#!/usr/bin/env bash
set -euo pipefail

# This script is idempotent. It:
# 1. Builds the tianer-postgres container image (if not already built)
# 2. Creates the Podman volume tianer-postgres-data (if not exists)
# 3. Applies DB roles via init-roles.sql (RUNS ONCE — guarded by role existence check)
# 4. Applies migrations via apply-migrations.sh (idempotent)

echo "TIANER | {\"ts\":\"$(date -Iseconds)\",\"level\":\"INFO\",\"component\":\"postgres\",\"msg\":\"Building tianer-postgres image\"}"
podman build -t localhost/tianer-postgres:latest -f deploy/containers/Dockerfile.postgres .

echo "TIANER | {\"ts\":\"$(date -Iseconds)\",\"level\":\"INFO\",\"component\":\"postgres\",\"msg\":\"Creating Podman volume if not exists\"}"
podman volume exists tianer-postgres-data || podman volume create tianer-postgres-data

echo "TIANER | {\"ts\":\"$(date -Iseconds)\",\"level\":\"INFO\",\"component\":\"postgres\",\"msg\":\"Starting tianer-postgres for role init\"}"
# ... (start container, wait for health, run init-roles.sql, stop container)

echo "TIANER | {\"ts\":\"$(date -Iseconds)\",\"level\":\"INFO\",\"component\":\"postgres\",\"msg\":\"Installing Quadlet unit files\"}"
install -m 0644 deploy/containers/tianer-postgres.container /etc/containers/systemd/
install -m 0644 deploy/containers/tianer-postgres-data.volume /etc/containers/systemd/
systemctl daemon-reload

echo "TIANER | {\"ts\":\"$(date -Iseconds)\",\"level\":\"INFO\",\"component\":\"postgres\",\"msg\":\"PostgreSQL container image built and Quadlet unit installed\"}"
```

### 11.5 apply-migrations.sh

`db/apply-migrations.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Idempotent migration runner.
# Reads credentials from environment variables (injected via EnvironmentFile).
# Applies all unapplied migrations in lexicographic order.

DB_HOST="${TIANER_DB_HOST:-tianer-postgres}"
DB_PORT="${TIANER_DB_PORT:-5432}"
DB_NAME="${TIANER_DB_NAME:-tianer}"
DB_USER="tianer"
DB_PASSWORD="${TIANER_DB_PASSWORD}"

MIGRATIONS_DIR="$(dirname "$0")/migrations"

export PGPASSWORD="${DB_PASSWORD}"

echo "TIANER | {\"ts\":\"$(date -Iseconds)\",\"level\":\"INFO\",\"component\":\"postgres\",\"msg\":\"Checking database connectivity\"}"
until pg_isready -h "${DB_HOST}" -p "${DB_PORT}" -U "${DB_USER}" -d "${DB_NAME}" -t 5 > /dev/null 2>&1; do
    echo "TIANER | {\"ts\":\"$(date -Iseconds)\",\"level\":\"WARN\",\"component\":\"postgres\",\"msg\":\"Waiting for database\"}"
    sleep 2
done

applied=0
for migration in "${MIGRATIONS_DIR}"/*.sql; do
    name=$(basename "${migration}" .sql)
    
    # Check if already applied
    already=$(psql -h "${DB_HOST}" -p "${DB_PORT}" -U "${DB_USER}" -d "${DB_NAME}" -tA \
        -c "SELECT 1 FROM bluetooth._migrations WHERE name = '${name}' LIMIT 1;" 2>/dev/null || true)
    
    if [[ "${already}" == "1" ]]; then
        echo "TIANER | {\"ts\":\"$(date -Iseconds)\",\"level\":\"INFO\",\"component\":\"postgres\",\"msg\":\"Migration already applied\",\"migration\":\"${name}\"}"
        continue
    fi
    
    echo "TIANER | {\"ts\":\"$(date -Iseconds)\",\"level\":\"INFO\",\"component\":\"postgres\",\"msg\":\"Applying migration\",\"migration\":\"${name}\"}"
    
    if psql -h "${DB_HOST}" -p "${DB_PORT}" -U "${DB_USER}" -d "${DB_NAME}" \
        -v ON_ERROR_STOP=1 --single-transaction -f "${migration}" > /dev/null 2>&1; then
        applied=$((applied + 1))
        echo "TIANER | {\"ts\":\"$(date -Iseconds)\",\"level\":\"INFO\",\"component\":\"postgres\",\"msg\":\"Migration applied\",\"migration\":\"${name}\"}"
    else
        echo "TIANER | {\"ts\":\"$(date -Iseconds)\",\"level\":\"ERROR\",\"component\":\"postgres\",\"msg\":\"Migration failed\",\"migration\":\"${name}\"}"
        exit 1
    fi
done

echo "TIANER | {\"ts\":\"$(date -Iseconds)\",\"level\":\"INFO\",\"component\":\"postgres\",\"msg\":\"Migrations complete\",\"applied\":${applied}}"
```

### 11.6 Backup Strategy

Nightly `pg_basebackup` via the `tianer` user's crontab (or systemd timer, to be defined in C12). Per the PostgreSQL 17 documentation [1], `pg_basebackup` takes a base backup of a running PostgreSQL database cluster without affecting other clients:

```bash
#!/usr/bin/env bash
# deploy/scripts/backup.sh — nightly PostgreSQL backup
set -euo pipefail

BACKUP_DIR="/var/lib/tianer/backups/postgres"
RETENTION_DAYS=14

mkdir -p "${BACKUP_DIR}"
BACKUP_FILE="${BACKUP_DIR}/tianer_$(date +%Y%m%d).tar.gz"

echo "TIANER | {\"ts\":\"$(date -Iseconds)\",\"level\":\"INFO\",\"component\":\"postgres\",\"msg\":\"Starting pg_basebackup\"}"

pg_basebackup -h tianer-postgres -p 5432 -U postgres \
    -D - -Ft -z -X fetch > "${BACKUP_FILE}"

# Remove backups older than retention
find "${BACKUP_DIR}" -name 'tianer_*.tar.gz' -mtime "+${RETENTION_DAYS}" -delete

echo "TIANER | {\"ts\":\"$(date -Iseconds)\",\"level\":\"INFO\",\"component\":\"postgres\",\"msg\":\"Backup complete\",\"file\":\"${BACKUP_FILE}\"}"
```

**Backup details:**
- Format: tar + gzip (`-Ft -z`)
- WAL method: `fetch` (simple, sufficient for nightly backup)
- Retention: 14 days (matches PCAP retention)
- Location: `/var/lib/tianer/backups/postgres/` on the SD card or external storage

### 11.7 Deployment Checklist

Before considering C02 deployed, verify:

- [ ] `tianer-postgres` container is running and healthy (`systemctl --user is-active tianer-postgres`)
- [ ] `pg_isready` returns "accepting connections"
- [ ] TimescaleDB extension is loaded: `SELECT extversion FROM pg_extension WHERE extname = 'timescaledb';` returns `2.23.x` or later
- [ ] All 5 migrations are applied: `SELECT COUNT(*) FROM bluetooth._migrations;` returns `5`
- [ ] All 4 roles exist: `SELECT rolname FROM pg_roles WHERE rolname IN ('tianer','tianer_writer','tianer_ro','tianer_grafana');` returns 4 rows
- [ ] Role permissions verified per §10.2 `test_role_permissions.sh`
- [ ] `raw_packets` is a hypertable with 1-hour chunk interval
- [ ] Continuous aggregate `device_5min_buckets` exists
- [ ] Compression policy is active on `raw_packets`
- [ ] Retention policy is active on `raw_packets`
- [ ] `pg_hba.conf` rejects non-loopback connections
- [ ] Backup script is scheduled (cron or systemd timer)
- [ ] Database secrets files exist with mode 0600

---

## Appendix A: Storage Budget

| Item | Allocated | Notes |
|------|-----------|-------|
| V06 Podman volume | 5 GB | PostgreSQL data directory on eMMC |
| `raw_packets` (uncompressed, 7 days) | ~3.5 GB | At 1000 pps × 4 sniffers × 80 bytes/row × 7 days × 86400 sec/day = ~19 GB → compressed 8:1 → ~2.4 GB. Conservatively budget 3.5 GB for overhead. |
| `raw_packets` (compressed, 7–90 days) | ~1 GB | 83 days of compressed data at ~12 MB/day. |
| `device_summary` | < 100 MB | One row per unique MAC. 10K devices × ~1 KB = ~10 MB. Generous headroom. |
| `device_enrichment` | < 200 MB | Depends on deep parser throughput. Conservatively budget 200 MB. |
| Indexes | ~1 GB | All indexes on `raw_packets`, `device_summary`, `device_enrichment`. |
| WAL | ~500 MB | `max_wal_size = 2 GB` but steady-state is lower. |
| **Total (conservative)** | **~5 GB** | Fits within the 5 GB V06 allocation. |

**Growth monitoring:** If `pg_database_size('tianer')` exceeds 4.5 GB (90% of 5 GB), the `TianerDBSizeWarning` alert fires. At that point, the operator should verify retention and compression policies are working correctly, or increase V06 volume size.

## Appendix B: Migration File Summary

| File | Creates | Policies/Extensions | Grants |
|------|---------|---------------------|--------|
| `0001_init.sql` | `bluetooth` schema, `_migrations`, `sniffers`, `sniffer_heartbeat`, `raw_packets` (hypertable), `device_summary`, `ingest_gaps` | `timescaledb` extension, hypertable, 3 indexes | — |
| `0002_continuous_aggregates.sql` | `device_5min_buckets` (cagg) | Refresh policy (1 min interval, 1h start offset, 5min end offset) | — |
| `0003_compression_policies.sql` | — | Compression enabled on `raw_packets`, compression policy (7 days), retention policy (90 days) | — |
| `0004_residency_classifier.sql` | `classify_residency()` function | — | — |
| `0005_device_enrichment.sql` | `device_enrichment` | UNIQUE constraint, 1 index | SELECT to `tianer_ro`, `tianer_grafana`; INSERT to `tianer_writer` |

## Appendix C: Residency Class Reference

| Class | Criteria | Meaning |
|-------|----------|---------|
| `unknown` | No data in `device_summary` for this MAC | Never observed (should rarely appear in practice). |
| `new` | `first_seen` within last 24 hours | Recently appeared. Could be a new permanent device or a transient visitor. |
| `lost` | `last_seen` > 14 days ago | Was previously observed but has not been seen recently. May have moved, been turned off, or changed its MAC (BLE privacy). |
| `resident` | ≥ 5 distinct observation days AND ≥ 700 total observations in last 7 days | High-confidence permanent presence. Examples: smart home hubs, always-on Bluetooth audio devices, environmental sensors. |
| `frequent` | ≥ 3 distinct observation days in last 7 days (but doesn't meet `resident` thresholds) | Regularly present but not continuous. Examples: fitness trackers that only advertise when worn, phones that are home evenings/weekends. |
| `transient` | Everything else — fewer than 3 distinct days in last 7 days | Passing device. Visitor's phone, car Bluetooth system driving by, delivery person's scanner. |

---

## References

[1] PostgreSQL Global Development Group. "PostgreSQL 17 Documentation." https://www.postgresql.org/docs/17/, 2024 — Documents CREATE ROLE roles and LOGIN attributes (§4.2), `pg_hba.conf` format and authentication methods including `scram-sha-256` (§8.2), `pg_basebackup` for taking base backups (§11.6), `pg_stat_statements` for query performance monitoring (§7.2), `shared_buffers` server configuration tuning (§9.2), and SCRAM-SHA-256 password authentication (§8.2) as described in RFC 7677.

[2] TimescaleDB. "TimescaleDB Documentation." https://docs.timescale.com — Documents `create_hypertable()` for creating hypertables with configurable chunk intervals (§3.3.6), `add_compression_policy()` for automatic background compression of chunks (§3.3.4), `add_retention_policy()` for automatic chunk dropping (§3.3.5), and continuous aggregates with refresh policies (§3.10, §3.11).

[3] PostgreSQL Global Development Group. "pg_basebackup." https://www.postgresql.org/docs/17/app-pgbasebackup.html — Documents base backup format options (`-Ft -z` for tar+gzip), WAL method choices (`-X fetch` for simple fetch), and replication connection requirements (§11.6).

[4] PostgreSQL Wiki. "Number Of Database Connections." https://wiki.postgresql.org/wiki/Number_Of_Database_Connections — Documents best practices for connection pooling: performance degrades past the "knee" of optimal connections due to lock contention, context switching, and cache line contention; recommends limiting active connections to `(core_count * 2) + effective_spindle_count` (§2.3).

---

*End of C02 Database Design Document.*
