# C06 — Gap Detector

**Status:** Phase A Design — guides Phase B implementation  
**Author:** Code Architect  
**Date:** 2026-06-09  
**Dependencies:** C02 (Database — `ingest_gaps` table, `tianer_writer` role, `raw_packets` hypertable, `sniffer_heartbeat`), C04 (PCAP Rotation — rotated `.pcap.zst` files and filename convention), C05 (Ingest Bridge — real-time ingest flow into `raw_packets`)  
**Blocks:** None downstream — C06 is a terminal recovery component

---

## 1. Overview

### 1.1 Purpose

C06 Gap Detector is the **ingest reliability and recovery component** of the Tian'er platform. It detects time windows where packets are missing from the `raw_packets` hypertable, backfills those gaps from the authoritative on-disk PCAP archive, and provides secondary verification that the PCAP-to-DB pipeline is complete. C06 implements Layers 3 and 4 of the five-layer data integrity architecture defined in `doc/designs/storage-strategy.md`.

### 1.2 Scope

| In Scope | Out of Scope |
|----------|-------------|
| Gap detection: compare time-bucketed `raw_packets` row counts against heartbeat state | Writing PCAP (C03 sniffer wrapper) |
| Gap backfill: reprocess rotated PCAP files (`.pcap.zst`) via `tshark` and insert into `raw_packets` | Rotating PCAP (C04) or capture pipeline (C03) |
| Secondary verification: PCAP packet count vs DB row count comparison | Deep parsing packet payloads (C07) |
| `ingest_gaps` table management (insert, update status) | DB schema definition (C02) |
| Heartbeat liveness evaluation | Sending heartbeats (C03 heartbeat.sh) |
| Idempotent backfill: `INSERT ON CONFLICT DO NOTHING` | Deduplication at query time (D-10) |
| Async Python execution with psycopg 3.1 | C++ ingest bridge logic (C05) |

### 1.3 Position in the System

C06 is a **Layer 2 component** in the build sequence (component-breakdown.md §4.1). It runs as a **Quadlet oneshot container** inside the `tianer-platform` pod, triggered by a **systemd timer**. It depends on:

- **C02 Database:** `raw_packets` hypertable (to query for gaps), `sniffer_heartbeat` (to determine sniffer liveness), `ingest_gaps` (to track and deduplicate gap records). Uses the `tianer_writer` role (INSERT-only per Q9 resolution).
- **C04 PCAP Rotation:** Timestamped compressed PCAP files stored at `/var/lib/tianer/pcap/<sniffer_name>/YYYYMMDD-HHMM.pcap.zst`. C06 mounts V02 **read-only**.
- **C05 Ingest Bridge:** The real-time ingest path that C06 validates — if C05 drops data, C06 detects it.

C06 does **not** block the critical path for the MVP pipeline. It is a production-reliability enhancement: without it, `raw_packets` gaps from C05 crashes or DB outages go undetected and un-recovered. With it, the platform guarantees eventual consistency between the PCAP source of truth and the database.

```
┌───────────────────────────────────────────────────────────────────────┐
│                     C02 DATABASE (PostgreSQL + TimescaleDB)            │
│                                                                       │
│  ┌─────────────────┐  ┌──────────────────┐  ┌──────────────────────┐ │
│  │ sniffer_heartbeat│  │  raw_packets     │  │  ingest_gaps         │ │
│  │ (liveness source)│  │  (hypertable)    │  │  (gap tracking)      │ │
│  └────────┬────────┘  └────────┬─────────┘  └──────────┬───────────┘ │
│           │                    │                        │             │
└───────────┼────────────────────┼────────────────────────┼─────────────┘
            │                    │                        │
            │  QUERY: heartbeats │  QUERY: COUNT(*)       │  INSERT/UPDATE
            │                    │  per time_bucket       │
            │                    │                        │
   ┌────────┴────────────────────┴────────────────────────┴────────────┐
   │                     C06 GAP DETECTOR                               │
   │                                                                    │
   │  ┌──────────────┐   ┌──────────────┐   ┌─────────────────────────┐ │
   │  │ detect_gaps  │──▶│ backfill_gap │──▶│ verify_secondary_count  │ │
   │  │ (asyncio     │   │ (tshark +    │   │ (capinfos vs DB COUNT)  │ │
   │  │  timer loop) │   │  INSERT)     │   │                         │ │
   │  └──────────────┘   └──────┬───────┘   └─────────────────────────┘ │
   │                            │                                       │
   └────────────────────────────┼───────────────────────────────────────┘
                                │
                                │ READ: .pcap.zst files
                                │
   ┌────────────────────────────┴───────────────────────────────────────┐
   │  V02: /var/lib/tianer/pcap/  (mounted :ro)                         │
   │  ┌────────────┐  ┌────────────┐  ┌────────────┐                   │
   │  │ ut1/       │  │ nrf1/      │  │ nrf2/      │                   │
   │  │ *.pcap.zst │  │ *.pcap.zst │  │ *.pcap.zst │                   │
   │  └────────────┘  └────────────┘  └────────────┘                   │
   └────────────────────────────────────────────────────────────────────┘
```

### 1.4 Five-Layer Data Integrity Architecture (Recap)

C06 implements two of the five defense-in-depth layers from `storage-strategy.md`:

| Layer | Name | Owner | What It Does |
|-------|------|-------|-------------|
| Layer 1 | PCAP Source of Truth | C03 + C04 | Raw packet bytes on disk |
| Layer 2 | Heartbeat Liveness | C03 | Sniffer-is-running signal every 30s |
| **Layer 3** | **Gap Detector** | **C06** | Detect missing time buckets; backfill from PCAP |
| **Layer 4** | **Secondary Verification** | **C06** | Cross-check PCAP packet count vs DB row count |
| Layer 5 | PCAP Integrity | C13 | Validate rotated files for corruption |

---

## 2. High-Level Architecture (HLA)

### 2.1 Component Diagram

```
                        systemd timer fires every BLESNIFF_GAP_SCAN_INTERVAL_MINUTES
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────┐
│  blesniff-gap-detector.container  (Quadlet oneshot, platform pod)     │
│                                                                      │
│  main() ──► asyncio.run(detect_and_backfill())                      │
│                │                                                     │
│                │  1. Determine which sniffers are running            │
│                │     (query sniffer_heartbeat: ts > NOW - 60s)       │
│                │                                                     │
│                │  2. For each active sniffer, for each time bucket   │
│                │     in [NOW - lookback, NOW - 1 min]:               │
│                │     ┌──────────────────────────────────────┐       │
│                │     │ SELECT sniffer_id,                     │       │
│                │     │   time_bucket('30s', ts) as bucket,   │       │
│                │     │   COUNT(*) as cnt                     │       │
│                │     │ FROM raw_packets                       │       │
│                │     │ WHERE sniffer_id = $1                  │       │
│                │     │   AND ts >= $2 AND ts < $3             │       │
│                │     │ GROUP BY bucket                        │       │
│                │     │ HAVING COUNT(*) = 0                    │       │
│                │     └──────────────────────────────────────┘       │
│                │                                                     │
│                │  3. Record each gap in ingest_gaps                  │
│                │     (INSERT ON CONFLICT DO NOTHING)                 │
│                │                                                     │
│                │  4. For each gap with status='open':                │
│                │     a. Locate PCAP file(s) via filename convention  │
│                │     b. Decompress with zstd                         │
│                │     c. Run tshark to extract fields                 │
│                │     d. INSERT into raw_packets (backfilled=true)    │
│                │        with ON CONFLICT DO NOTHING                  │
│                │     e. Update ingest_gaps status='closed'           │
│                │                                                     │
│                │  5. Secondary verification (periodic):              │
│                │     Compare PCAP packet counts vs DB row counts     │
│                │     per (sniffer_id, time window)                   │
│                │                                                     │
│                ▼                                                     │
│            Exit 0. Timer fires again at next interval.               │
└──────────────────────────────────────────────────────────────────────┘
```

### 2.2 Execution Model

C06 runs as a **oneshot** — it executes once, performs its full detection-and-backfill cycle, then exits. The systemd timer (or Quadlet timer unit) triggers it at a fixed interval:

```
Timer fires → container starts → main() runs → detect → backfill → verify → exit 0
                                                                               │
                  ┌────────────────────────────────────────────────────────────┘
                  │  (next timer interval)
                  ▼
            Timer fires → container starts → ...
```

This model provides:
- **Simplicity:** No long-running process to manage. No memory leaks over time. No async event loop running indefinitely.
- **Crash safety:** If the container crashes mid-backfill, the next timer event picks up where it left off (gaps remaining in 'open' or 'backfilling' status are re-evaluated).
- **Resource efficiency:** The container only consumes CPU/memory during the detection cycle. Between cycles, no resources are used.
- **Scheduling flexibility:** The scan interval is independently configurable; the secondary verification runs on a longer cadence.

### 2.3 Gap Definition (Contract GAP-1)

Per the inception spec §8.6, a gap exists for a `(sniffer_id, time_bucket)` pair within the lookback window when **all four** conditions are met:

1. **Sniffer was running:** The sniffer's heartbeat (`sniffer_heartbeat.ts`) was updated within the last 60 seconds, indicating the sniffer was operational during the bucket. Heartbeats are evaluated from the DB table, which is backfilled from local heartbeat files per D-11. If the heartbeat is stale (>60s), the sniffer is considered **not running** and no gap is created — packets cannot be missing if no one was supposed to be capturing them.

2. **Bucket has zero packets:** `COUNT(*) FROM raw_packets WHERE sniffer_id = $1 AND ts >= bucket_start AND ts < bucket_end` is zero. The TimescaleDB `time_bucket()` function [1] is used to partition time into aligned intervals for efficient aggregate comparison.

3. **Bucket is closed:** The bucket's end time is at least 1 minute in the past (`bucket_end <= NOW() - INTERVAL '1 minute'`). This prevents race conditions where the live ingest path hasn't yet written all packets for the current bucket. A 1-minute safety margin covers the maximum ingest latency (P95 < 5 seconds) with generous headroom.

4. **Bucket is within lookback window:** The bucket start time is within the configured lookback hours (`BLESNIFF_GAP_LOOKBACK_HOURS`, default 1 hour). Gaps older than the lookback window are not detected — they represent permanent data loss from the DB perspective.

### 2.4 Backfill Strategy

When a gap is detected, C06 backfills the missing data from the PCAP archive:

```
┌──────────────────────────────────────────────────────────────────┐
│                      BACKFILL FLOW                               │
│                                                                  │
│  1. MAP gap → PCAP files                                         │
│     gap.bucket_start = 2026-06-09 14:32:00                       │
│     gap.bucket_end   = 2026-06-09 14:33:00                       │
│     rotation_interval = 30 minutes                               │
│     → Files: 20260609-1400.pcap.zst, 20260609-1430.pcap.zst      │
│                                                                  │
│  2. DECOMPRESS                                                   │
│     zstd -d -c 20260609-1400.pcap.zst | ...                      │
│     zstd -d -c 20260609-1430.pcap.zst | ...                      │
│     (piped, never written to disk)                               │
│                                                                  │
│  3. EXTRACT via tshark                                           │
│     tshark -r - -T fields -E separator=\| ...                    │
│     → Produces CONTRACT CAPTURE-2 normalized output              │
│                                                                  │
│  4. FILTER by timestamp                                          │
│     Keep only rows where ts >= gap.bucket_start                  │
│                         AND ts <  gap.bucket_end                  │
│                                                                  │
│  5. INSERT into raw_packets                                      │
│     INSERT INTO raw_packets (...) VALUES (...)                   │
│     ON CONFLICT DO NOTHING;   ← idempotent [2]                  │
│     (backfilled = true)                                         │
│                                                                  │
│  6. UPDATE ingest_gaps                                            │
│     SET status='closed', rows_backfilled=<count>                 │
└──────────────────────────────────────────────────────────────────┘
```

**Why `ON CONFLICT DO NOTHING`?** This is the PostgreSQL mechanism for idempotent inserts [2]. If the gap detector runs twice (due to a crash-and-restart cycle, or concurrent timer misfire), the second run attempts to re-insert rows that already exist. Without conflict handling, this would fail with a unique violation error. `ON CONFLICT DO NOTHING` silently skips already-existing rows, making the backfill operation idempotent [2]. This is critical for crash-only design (PF-8): the gap detector can be killed at any point and restarted safely.

The backfill `INSERT` uses the `tianer_writer` role (INSERT-only, no SELECT permission). Per Q9 resolution, this limits the blast radius of a compromised gap detector: it can insert rows but cannot read sensitive data, modify existing rows, or execute DDL.

### 2.5 Secondary Verification

Secondary verification (Layer 4 of the integrity architecture) provides **redundant detection** (PF-3): even if the gap detector itself is buggy or down, a mismatch between PCAP file packet counts and DB row counts is detected independently.

```
Secondary verification cadence: every 30 minutes (separate timer, or as final step of gap detector run)

For each sniffer, for each rotated PCAP file within the lookback window:
  1. Count packets in PCAP: capinfos -c <file>.pcap.zst   (or tshark -r - | wc -l after decompress)
  2. Count rows in DB:      SELECT COUNT(*) FROM raw_packets
                             WHERE sniffer_id = $1
                               AND ts >= <file_start> AND ts < <file_end>
  3. Compare: |PCAP - DB| / PCAP * 100
     If > 1% mismatch → alert TianerGapSecondaryMismatch
```

The secondary check runs less frequently than the primary gap detector to limit DB load (COUNT(*) queries on large time windows are non-trivial). The mismatch threshold of 1% allows for rare edge cases (e.g., tshark field extraction differences) while surfacing genuine data loss.

### 2.6 Container Architecture

C06 runs as a **Quadlet oneshot container** in the `tianer-platform` pod:

```ini
# deploy/containers/blesniff-gap-detector.container
[Container]
Image=localhost/tianer-gap-detector:latest
ContainerName=blesniff-gap-detector
Pod=tianer-platform.pod

EnvironmentFile=/etc/tianer/secrets/db_password.env
Environment=BLESNIFF_GAP_BUCKET_SECONDS=30
Environment=BLESNIFF_GAP_SCAN_INTERVAL_MINUTES=5
Environment=BLESNIFF_GAP_LOOKBACK_HOURS=1
Environment=BLESNIFF_PCAP_DIR=/var/lib/tianer/pcap

Volume=tianer-config:/etc/tianer:ro
Volume=tianer-pcap:/var/lib/tianer/pcap:ro
Volume=tianer-logs:/var/log/tianer:rw
Volume=tianer-fifo:/var/run/tianer:rw

Network=tianer-net

Exec=python3 -m tianer_gapdet --once

[Service]
Type=oneshot
RemainAfterExit=no

[Install]
WantedBy=blesniff.target
```

And a corresponding Quadlet `.timer` unit:

```ini
# deploy/containers/blesniff-gap-detector.timer
[Timer]
OnUnitActiveSec=300
AccuracySec=5s
Persistent=true

[Install]
WantedBy=timers.target
```

**Key container properties:**

| Property | Value | Rationale |
|----------|-------|-----------|
| `Type=oneshot` | Container runs once, then exits | No long-running daemon to manage |
| `RemainAfterExit=no` | Container is removed after exit | Fresh start each cycle; no stale state |
| V02 `:ro` | Read-only access to PCAP files | Cannot accidentally modify source of truth |
| `Network=tianer-net` | Access to PostgreSQL container | Required for DB queries and INSERT |
| `OnUnitActiveSec=300` | Triggers 300s after previous run completes | 5-minute default scan interval |
| `Persistent=true` | Records last trigger on disk | Catches up on missed scans after system reboot [3] |

---

## 3. Data Model (ERD)

### 3.1 The ingest_gaps Table

Defined in C02 §3.6. C06 writes and updates this table. Full schema:

```sql
CREATE TABLE bluetooth.ingest_gaps (
    id              BIGSERIAL PRIMARY KEY,
    sniffer_id      SMALLINT NOT NULL REFERENCES sniffers(sniffer_id),
    bucket_start    TIMESTAMPTZ NOT NULL,
    bucket_end      TIMESTAMPTZ NOT NULL,
    status          TEXT NOT NULL CHECK (status IN ('open','backfilling','closed','failed')),
    retry_count     INT NOT NULL DEFAULT 0,
    first_detected  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_attempted  TIMESTAMPTZ,
    rows_backfilled INT,
    last_error      TEXT,
    UNIQUE (sniffer_id, bucket_start)
);
```

**State machine for `status`:**

```
  ┌──────┐   gap detected    ┌──────────────┐   backfill succeeds   ┌────────┐
  │      │ ────────────────▶  │     open      │ ────────────────────▶ │ closed │
  │      │                    └──────┬────────┘                       └────────┘
  │      │                           │                                      ▲
  │ NULL │                           │ retry < 3 ────┐                      │
  │      │                           ▼                 │                     │
  │      │                    ┌──────────────┐         │        success      │
  │      │                    │ backfilling   │─────────┘────────────────────┘
  │      │                    └──────┬────────┘
  │      │                           │
  │      │                           │ retry >= 3 (exhausted)
  │      │                           ▼
  │      │                    ┌──────────────┐
  └──────┘                    │    failed     │  ← permanent: PCAP missing or corrupted
                              └──────────────┘
```

**Status semantics:**

| Status | Meaning | Next Action |
|--------|---------|-------------|
| `open` | Gap detected, not yet attempted | Pick up on next scan, transition to `backfilling` |
| `backfilling` | Backfill in progress (written by C06 just before starting backfill) | If C06 crashes and restarts, re-evaluate. If backfill succeeds, set to `closed`. If backfill fails, set back to `open` and increment `retry_count`. |
| `closed` | Successfully backfilled | Terminal state. No further action. |
| `failed` | Permanent failure after 3 retries | Terminal state. PCAP missing, corrupted, or unparseable. Operator investigation required. |

**The UNIQUE constraint** on `(sniffer_id, bucket_start)` serves two purposes:
1. **Prevents duplicate gap records:** Two scans of the same time window produce only one `ingest_gaps` row (the second `INSERT` is silently ignored via `ON CONFLICT DO NOTHING`).
2. **Enables idempotent gap tracking:** If C06 crashes and restarts, it can safely re-insert gap records without creating duplicates.

### 3.2 Gap ↔ PCAP File Mapping

The GAP-2 contract (defined in C04 §5.2) maps gap time windows to PCAP filenames. The mapping is deterministic:

```
Given:   gap.bucket_start, gap.bucket_end, rotation_interval_minutes (default: 30)
Algorithm:
  1. Compute the rotation-aligned window containing gap.bucket_start:
     rotation_minute = floor(gap.bucket_start.minute / rotation_interval_minutes) * rotation_interval_minutes
     file_start = gap.bucket_start.replace(minute=rotation_minute, second=0, microsecond=0)

  2. Compute the rotation-aligned window containing gap.bucket_end:
     rotation_minute = floor(gap.bucket_end.minute / rotation_interval_minutes) * rotation_interval_minutes
     file_end = gap.bucket_end.replace(minute=rotation_minute, second=0, microsecond=0)

  3. Generate all rotation-aligned timestamps between file_start and file_end:
     For t in [file_start, file_start + interval, ..., file_end]:
       filename = strftime(t, "YYYYMMDD-HHMM") + ".pcap.zst"
       search_paths.append(pcap_dir / sniffer_name / filename)

  4. Files may not exist for all generated timestamps (sniffer was not running).
     Only existing files are used for backfill.
```

**Example:**

```
gap.bucket_start = 2026-06-09 14:32:00 UTC
gap.bucket_end   = 2026-06-09 14:33:00 UTC
rotation_interval = 30 minutes

Mapped files:
  /var/lib/tianer/pcap/ut1/20260609-1400.pcap.zst   (covers 14:00–14:30)
  /var/lib/tianer/pcap/ut1/20260609-1430.pcap.zst   (covers 14:30–15:00)

Both files are decompressed and filtered to only keep rows within [14:32:00, 14:33:00).
```

---

## 4. Low-Level Architecture (LLA)

### 4.1 Module Decomposition

```
modules/bluetooth/gap-detector/src/tianer_gapdet/
├── __init__.py
├── cli.py            # argparse entry point, --once / --interval
├── detector.py       # Gap detection logic: SQL queries, heartbeat evaluation
├── backfill.py       # Backfill logic: PCAP file mapping, tshark invocation, INSERT
├── pcap_reader.py    # PCAP file utilities: find files by time window, decompress
├── verifier.py       # Secondary verification: PCAP count vs DB count
├── db.py             # Async PostgreSQL connection pool management
└── models.py         # Dataclasses: Gap, GapStatus, BackfillResult, VerifyResult
```

### 4.2 Core Types (`models.py`)

```python
from dataclasses import dataclass, field
from datetime import datetime, timedelta
from enum import Enum

class GapStatus(str, Enum):
    OPEN = "open"
    BACKFILLING = "backfilling"
    CLOSED = "closed"
    FAILED = "failed"

@dataclass(frozen=True)
class Gap:
    """A detected gap in raw_packets for a specific sniffer and time window."""
    sniffer_id: int
    bucket_start: datetime
    bucket_end: datetime

@dataclass
class BackfillResult:
    """Result of a single gap backfill attempt."""
    gap: Gap
    status: GapStatus
    rows_inserted: int = 0
    error: str | None = None

@dataclass
class VerifyResult:
    """Result of secondary verification for one PCAP file."""
    sniffer_id: int
    file_name: str
    file_start: datetime
    file_end: datetime
    pcap_packet_count: int
    db_packet_count: int
    mismatch_percent: float  # (|pcap - db| / pcap) * 100

@dataclass
class SnifferHeartbeat:
    """Liveness state for a sniffer from the heartbeat table."""
    sniffer_id: int
    last_heartbeat: datetime
    is_alive: bool  # True if last_heartbeat > NOW - 60s
```

### 4.3 Database Layer (`db.py`)

Uses **psycopg 3.1 async** for all PostgreSQL interactions [4]. The async model is essential because C06 performs I/O-bound operations (DB queries, PCAP file decompression, subprocess invocation) that must not block each other.

```python
import psycopg
from psycopg.rows import dict_row

class Database:
    """Async PostgreSQL connection pool for gap detector."""

    def __init__(self, dsn: str):
        self._dsn = dsn  # e.g. "host=tianer-postgres dbname=tianer user=tianer_writer password=..."

    async def connect(self) -> psycopg.AsyncConnection:
        """Create a new async connection using tianer_writer credentials."""
        return await psycopg.AsyncConnection.connect(
            self._dsn,
            options="-c statement_timeout=60000"  # 60s per statement timeout [5]
        )

    async def execute(self, query: str, *params) -> list[dict]:
        """Execute a query and return rows as dicts. Uses a fresh cursor."""
        async with await self.connect() as conn:
            async with conn.cursor(row_factory=dict_row) as cur:
                await cur.execute(query, params)
```

**Statement timeout:** Every connection to the database sets `statement_timeout=60000` (60 seconds) [5]. This prevents a runaway query (e.g., `COUNT(*)` on a very large hypertable without proper chunk exclusion) from blocking the gap detector indefinitely. If a query takes longer than 60 seconds, PostgreSQL cancels it and raises an error, which C06 catches, logs, and retries on the next scan.

**Connection pattern:** `tianer_writer` role — INSERT-only (can INSERT into `raw_packets` and `ingest_gaps`, can SELECT from `sniffer_heartbeat` via the gap detector's own internal queries, but cannot SELECT from `raw_packets` for non-aggregate purposes). The gap detector's `COUNT(*)` queries on `raw_packets` use the aggregate path, which is permitted for INSERT-privileged roles on hypertables in TimescaleDB.

### 4.4 Detection Logic (`detector.py`)

```python
from datetime import datetime, timedelta, timezone
from .db import Database
from .models import Gap, SnifferHeartbeat

class GapDetector:
    """Detects gaps in raw_packets by comparing time-bucketed counts
    against heartbeat liveness."""

    def __init__(self, db: Database, bucket_seconds: int = 30,
                 lookback_hours: int = 1, heartbeat_max_age_seconds: int = 60):
        self._db = db
        self._bucket_seconds = bucket_seconds
        self._lookback_hours = lookback_hours
        self._heartbeat_max_age = heartbeat_max_age_seconds

    async def get_alive_sniffers(self) -> list[SnifferHeartbeat]:
        """Return all sniffers whose last heartbeat is recent enough
        to indicate they are actively running."""
        query = """
            SELECT sniffer_id, ts as last_heartbeat,
                   (ts > NOW() - $1::interval) as is_alive
            FROM bluetooth.sniffer_heartbeat
            ORDER BY sniffer_id
        """
        rows = await self._db.execute(query,
            timedelta(seconds=self._heartbeat_max_age))
        return [SnifferHeartbeat(**row) for row in rows
                if row['is_alive']]

    async def find_gaps(self) -> list[Gap]:
        """Find all (sniffer_id, time_bucket) pairs where:
        1. Sniffer is alive (heartbeat is recent)
        2. Bucket is fully closed (end > 1 minute ago)
        3. Bucket has zero packets
        4. Bucket is within the lookback window
        """
        alive_sniffers = await self.get_alive_sniffers()
        gaps: list[Gap] = []

        bucket_interval = timedelta(seconds=self._bucket_seconds)
        lookback_start = datetime.now(timezone.utc) - timedelta(hours=self._lookback_hours)
        safety_margin = datetime.now(timezone.utc) - timedelta(minutes=1)

        # SQL: use TimescaleDB time_bucket() for efficient aggregate [1]
        query = """
            WITH expected_buckets AS (
                SELECT sniffer_id,
                       time_bucket($3::interval, gs.ts) AS bucket
                FROM generate_series($1::timestamptz, $2::timestamptz, $3::interval) gs(ts),
                     sniffer_heartbeat sh
                WHERE sh.sniffer_id = ANY($4::smallint[])
                  AND sh.ts > NOW() - $5::interval
            ),
            actual_counts AS (
                SELECT sniffer_id,
                       time_bucket($3::interval, ts) AS bucket,
                       COUNT(*) AS cnt
                FROM bluetooth.raw_packets
                WHERE sniffer_id = ANY($4::smallint[])
                  AND ts >= $1
                  AND ts < $2
                GROUP BY sniffer_id, bucket
            )
            SELECT eb.sniffer_id, eb.bucket, COALESCE(ac.cnt, 0) AS cnt
            FROM expected_buckets eb
            LEFT JOIN actual_counts ac
              ON eb.sniffer_id = ac.sniffer_id AND eb.bucket = ac.bucket
            WHERE COALESCE(ac.cnt, 0) = 0
            ORDER BY eb.sniffer_id, eb.bucket
        """
        alive_ids = [s.sniffer_id for s in alive_sniffers]

        rows = await self._db.execute(query,
            lookback_start,       # $1: WHERE ts >= 
            safety_margin,         # $2: WHERE ts <
            bucket_interval,       # $3: bucket width
            alive_ids,             # $4: sniffer IDs
            timedelta(seconds=self._heartbeat_max_age))  # $5: heartbeat max age

        for row in rows:
            gaps.append(Gap(
                sniffer_id=row['sniffer_id'],
                bucket_start=row['bucket'],
                bucket_end=row['bucket'] + bucket_interval,
            ))
        return gaps
```

**Key SQL design decisions:**

1. **`generate_series()` generates expected buckets:** This ensures that buckets where `COUNT(*) = 0` are explicitly enumerated — PostgreSQL won't return rows for buckets that don't exist in `raw_packets`. By generating the expected bucket series and LEFT JOIN-ing against actual counts, zero-count buckets surface naturally.

2. **`time_bucket()` with INTERVAL:** TimescaleDB's `time_bucket()` [1] provides efficient, index-aware time partitioning. Buckets align to UTC midnight (the default origin), which is acceptable since all Tian'er timestamps are UTC.

3. **Heartbeat JOIN restricts to alive sniffers:** The `WHERE sh.ts > NOW() - $5::interval` clause in the `expected_buckets` CTE ensures that only sniffers with a recent heartbeat generate expected buckets. Sniffers that are down do not produce false-positive gaps.

### 4.5 Backfill Logic (`backfill.py`)

```python
import asyncio
import subprocess
from datetime import datetime, timedelta
from pathlib import Path
from .db import Database
from .models import Gap, BackfillResult, GapStatus

class GapBackfiller:
    """Backfills missing raw_packets rows from rotated PCAP files."""

    def __init__(self, db: Database, pcap_dir: Path,
                 rotation_minutes: int = 30,
                 max_retries: int = 3):
        self._db = db
        self._pcap_dir = pcap_dir
        self._rotation_minutes = rotation_minutes
        self._max_retries = max_retries

    def _find_pcap_files(self, gap: Gap, sniffer_name: str) -> list[Path]:
        """Map a gap time window to PCAP filenames using the
        YYYYMMDD-HHMM.pcap.zst naming convention."""
        files = []
        t = gap.bucket_start.replace(minute=0, second=0, microsecond=0)
        while t.replace(minute=t.minute) < gap.bucket_end:
            filename = f"{t:%Y%m%d-%H%M}.pcap.zst"
            path = self._pcap_dir / sniffer_name / filename
            if path.exists():
                files.append(path)
            t += timedelta(minutes=self._rotation_minutes)
        return files

    async def _extract_rows(self, pcap_files: list[Path],
                            gap: Gap) -> list[dict]:
        """Decompress PCAP files and extract rows via tshark.
        Pipes: zstd -d -c → tshark -r - → parse CAPTURE-2 format."""
        rows = []
        for pcap_path in pcap_files:
            # Pipe zstd decompress into tshark
            zstd_proc = await asyncio.create_subprocess_exec(
                'zstd', '-d', '-c', str(pcap_path),
                stdout=asyncio.subprocess.PIPE,
                stderr=asyncio.subprocess.DEVNULL,
            )
            tshark_proc = await asyncio.create_subprocess_exec(
                'tshark', '-r', '-', '-l', '-n',
                '-T', 'fields', '-E', 'separator=|', '-E', 'header=n',
                '-E', 'quote=n', '-E', 'occurrence=f',
                '-e', 'frame.time_epoch',
                '-e', 'btle.advertising_address',
                '-e', 'btle.advertising_header.randomized_tx',
                '-e', 'nordic_ble.rssi',
                '-e', 'nordic_ble.channel',
                '-e', 'btle.advertising_header.pdu_type',
                '-e', 'btle.advertising_data',
                stdin=zstd_proc.stdout,
                stdout=asyncio.subprocess.PIPE,
                stderr=asyncio.subprocess.DEVNULL,
            )
            # Close zstd stdout in tshark's stdin context
            if zstd_proc.stdout:
                zstd_proc.stdout.close()

            # Read tshark output line by line, filter by timestamp
            async for line in tshark_proc.stdout:
                fields = line.decode().strip().split('|')
                if len(fields) < 7:
                    continue
                try:
                    ts = float(fields[0])
                except ValueError:
                    continue
                # Filter to gap window
                if ts < gap.bucket_start.timestamp() or ts >= gap.bucket_end.timestamp():
                    continue
                rows.append({
                    'ts': fields[0],
                    'mac_address': fields[1],
                    'address_type': fields[2],
                    'rssi': fields[3],
                    'channel': fields[4],
                    'pdu_type': fields[5],
                    'advdata': fields[6],
                })

            await asyncio.wait_for(zstd_proc.wait(), timeout=300)
            await asyncio.wait_for(tshark_proc.wait(), timeout=300)

        return rows

    async def backfill_gap(self, gap: Gap, sniffer_name: str) -> BackfillResult:
        """Backfill a single gap. Returns the result with status and row count."""
        # 1. Mark as 'backfilling'
        await self._db.execute("""
            UPDATE bluetooth.ingest_gaps
            SET status = 'backfilling', last_attempted = NOW()
            WHERE sniffer_id = $1 AND bucket_start = $2
        """, gap.sniffer_id, gap.bucket_start)

        # 2. Locate PCAP files
        pcap_files = self._find_pcap_files(gap, sniffer_name)
        if not pcap_files:
            await self._mark_failed(gap, "No PCAP files found for gap window")
            return BackfillResult(gap=gap, status=GapStatus.FAILED,
                                  error="No PCAP files found")

        # 3. Extract rows
        try:
            rows = await self._extract_rows(pcap_files, gap)
        except Exception as e:
            await self._increment_retry(gap, str(e))
            return BackfillResult(gap=gap, status=GapStatus.OPEN,
                                  error=str(e))

        # 4. Insert rows with ON CONFLICT DO NOTHING for idempotency [2]
        inserted = 0
        BATCH_SIZE = 500
        for i in range(0, len(rows), BATCH_SIZE):
            batch = rows[i:i + BATCH_SIZE]
            # Build multi-row INSERT with ON CONFLICT
            values_placeholders = []
            params = []
            for row in batch:
                values_placeholders.append(
                    "($%d::timestamptz AT TIME ZONE 'UTC', $%d::smallint,"
                    " $%d::bytea, $%d::smallint, $%d::smallint,"
                    " $%d::smallint, $%d::smallint, $%d::bytea, TRUE)"
                    % tuple(range(len(params) + 1, len(params) + 10))
                )
                # params ordered by raw_packets column order:
                # ts, sniffer_id, mac_address, address_type, rssi,
                # channel, pdu_type, advdata, backfilled
                params.extend([
                    datetime.fromtimestamp(float(row['ts']), tz=timezone.utc),
                    gap.sniffer_id,
                    bytes.fromhex(row['mac_address'].replace(':', '')) if row['mac_address'] else None,
                    int(row['address_type']) if row['address_type'] else None,
                    int(row['rssi']) if row['rssi'] else None,
                    int(row['channel']) if row['channel'] else None,
                    int(row['pdu_type']) if row['pdu_type'] else None,
                    bytes.fromhex(row['advdata']) if row['advdata'] else None,
                ])

            sql = f"""
                INSERT INTO bluetooth.raw_packets
                    (ts, sniffer_id, mac_address, address_type,
                     rssi, channel, pdu_type, advdata, backfilled)
                VALUES {','.join(values_placeholders)}
                ON CONFLICT DO NOTHING
            """
            await self._db.execute(sql, *params)
            inserted += len(batch)

        # 5. Mark as closed
        await self._db.execute("""
            UPDATE bluetooth.ingest_gaps
            SET status = 'closed', rows_backfilled = $3
            WHERE sniffer_id = $1 AND bucket_start = $2
        """, gap.sniffer_id, gap.bucket_start, inserted)

        return BackfillResult(gap=gap, status=GapStatus.CLOSED,
                              rows_inserted=inserted)

    async def _increment_retry(self, gap: Gap, error: str):
        """Increment retry count. If >= max_retries, mark as failed."""
        await self._db.execute("""
            UPDATE bluetooth.ingest_gaps
            SET retry_count = retry_count + 1,
                last_attempted = NOW(),
                last_error = $3,
                status = CASE WHEN retry_count + 1 >= $4 THEN 'failed' ELSE 'open' END
            WHERE sniffer_id = $1 AND bucket_start = $2
        """, gap.sniffer_id, gap.bucket_start, error, self._max_retries)

    async def _mark_failed(self, gap: Gap, error: str):
        """Mark a gap as permanently failed."""
        await self._db.execute("""
            INSERT INTO bluetooth.ingest_gaps
                (sniffer_id, bucket_start, bucket_end, status,
                 first_detected, last_attempted, last_error)
            VALUES ($1, $2, $3, 'failed', NOW(), NOW(), $4)
            ON CONFLICT (sniffer_id, bucket_start)
            DO UPDATE SET status = 'failed', last_error = $4
        """, gap.sniffer_id, gap.bucket_start, gap.bucket_end, error)
```

**Key implementation decisions:**

1. **Piped subprocesses:** `zstd -d -c | tshark -r -` avoids writing decompressed data to disk. The decompressed PCAP data exists only as a memory stream. This is both faster and disk-space-safe — a large 30-minute PCAP file decompresses to 50–200 MB, which would be wasteful to write to V03 tmpfs.

2. **Batch INSERT:** Rows are inserted in batches of 500 to balance throughput against memory usage. A single 30-second bucket at 1000 pps contains up to 30,000 rows. 500-row batches keep memory per batch under ~10 KB.

3. **`ON CONFLICT DO NOTHING` without a conflict target:** Since `raw_packets` has no unique index (per D-10 dedup-is-at-query-time), the `ON CONFLICT DO NOTHING` clause acts as a safety net. If a unique index is added later, the clause already handles it. Without a unique index, `ON CONFLICT DO NOTHING` is a no-op — rows are always inserted [2].

4. **Timeout on subprocess wait:** `asyncio.wait_for(proc.wait(), timeout=300)` prevents a hung `zstd` or `tshark` process from blocking the backfill indefinitely. After 5 minutes, the task is cancelled and the gap is retried on the next scan.

### 4.6 Secondary Verifier (`verifier.py`)

```python
from .db import Database
from .models import VerifyResult
from pathlib import Path

class SecondaryVerifier:
    """Cross-checks PCAP file packet counts against DB row counts."""

    def __init__(self, db: Database, pcap_dir: Path,
                 mismatch_threshold_pct: float = 1.0):
        self._db = db
        self._pcap_dir = pcap_dir
        self._threshold = mismatch_threshold_pct

    async def verify_file(self, sniffer_id: int, sniffer_name: str,
                          pcap_file: Path) -> VerifyResult | None:
        """Verify one PCAP file against the database. Returns a VerifyResult
        if the mismatch exceeds threshold, or None if counts match."""
        # Parse filename to get time window
        filename = pcap_file.name
        # YYYYMMDD-HHMM.pcap.zst → extract date and time
        date_part = filename[:8]   # YYYYMMDD
        time_part = filename[9:13] # HHMM
        file_start = datetime.strptime(f"{date_part}{time_part}", "%Y%m%d%H%M")
        file_end = file_start + timedelta(minutes=30)  # rotation interval

        # Count packets in PCAP
        proc = await asyncio.create_subprocess_exec(
            'capinfos', '-c', str(pcap_file),
            stdout=asyncio.subprocess.PIPE, stderr=asyncio.subprocess.PIPE)
        stdout, _ = await proc.communicate()
        pcap_count = int(stdout.decode().strip().split()[-1])

        # Count rows in DB for the same window
        rows = await self._db.execute("""
            SELECT COUNT(*) as cnt
            FROM bluetooth.raw_packets
            WHERE sniffer_id = $1
              AND ts >= $2
              AND ts < $3
        """, sniffer_id, file_start.astimezone(), file_end.astimezone())
        db_count = rows[0]['cnt']

        if pcap_count == 0:
            return None  # empty file, nothing to verify

        mismatch = abs(pcap_count - db_count) / pcap_count * 100
        if mismatch > self._threshold:
            return VerifyResult(
                sniffer_id=sniffer_id,
                file_name=filename,
                file_start=file_start,
                file_end=file_end,
                pcap_packet_count=pcap_count,
                db_packet_count=db_count,
                mismatch_percent=mismatch,
            )
        return None
```

### 4.7 Main Entry Point (`cli.py`)

```python
import argparse
import asyncio
import os
import signal
from pathlib import Path
from . import detector, backfill, verifier
from .db import Database

async def run_once():
    """Single-cycle execution: detect → record → backfill → verify."""
    db = Database(os.environ['BLESNIFF_DB_DSN'])
    pcap_dir = Path(os.environ.get('BLESNIFF_PCAP_DIR', '/var/lib/tianer/pcap'))
    bucket_seconds = int(os.environ.get('BLESNIFF_GAP_BUCKET_SECONDS', '30'))
    lookback_hours = int(os.environ.get('BLESNIFF_GAP_LOOKBACK_HOURS', '1'))

    # Phase 1: Detect gaps
    det = detector.GapDetector(db, bucket_seconds, lookback_hours)
    gaps = await det.find_gaps()
    print(f"Found {len(gaps)} gaps")

    # Phase 2: Record gaps in ingest_gaps table (idempotent)
    for gap in gaps:
        await db.execute("""
            INSERT INTO bluetooth.ingest_gaps
                (sniffer_id, bucket_start, bucket_end, status)
            VALUES ($1, $2, $3, 'open')
            ON CONFLICT (sniffer_id, bucket_start) DO NOTHING
        """, gap.sniffer_id, gap.bucket_start, gap.bucket_end)
    print(f"Recorded {len(gaps)} new gaps")

    # Phase 3: Backfill open gaps
    bf = backfill.GapBackfiller(db, pcap_dir)
    open_gaps_rows = await db.execute("""
        SELECT sniffer_id, bucket_start, bucket_end
        FROM bluetooth.ingest_gaps
        WHERE status IN ('open', 'backfilling')
        ORDER BY sniffer_id, bucket_start
    """)
    for row in open_gaps_rows:
        gap = Gap(row['sniffer_id'], row['bucket_start'], row['bucket_end'])
        # Map sniffer_id to name (query sniffers table)
        name_rows = await db.execute(
            "SELECT name FROM bluetooth.sniffers WHERE sniffer_id = $1",
            gap.sniffer_id)
        if not name_rows:
            continue
        sniffer_name = name_rows[0]['name']
        result = await bf.backfill_gap(gap, sniffer_name)
        print(f"  Backfill status: {result.status}, rows: {result.rows_inserted}")

    # Phase 4: Secondary verification (optional, run on separate cadence)
    if os.environ.get('BLESNIFF_GAP_VERIFY', '0') == '1':
        vf = verifier.SecondaryVerifier(db, pcap_dir)
        # ... iterate over recent PCAP files and verify each ...

def main():
    parser = argparse.ArgumentParser(description='Gap detector and backfill')
    parser.add_argument('--once', action='store_true',
                        help='Run one cycle and exit')
    parser.add_argument('--interval', type=int, default=300,
                        help='Run continuously with interval (seconds)')
    args = parser.parse_args()

    if args.once:
        asyncio.run(run_once())
    else:
        # Continuous mode with asyncio timer loop [6]
        async def loop():
            while True:
                await asyncio.sleep(args.interval)
                await run_once()
        asyncio.run(loop())

if __name__ == '__main__':
    main()
```

**Execution modes:**

- `--once`: Run one detection-backfill-verify cycle and exit. Used by the Quadlet oneshot container.
- `--interval N`: Run continuously, sleeping N seconds between cycles. Used for manual testing or debugging with a long-running process.

---

## 5. Inter-Component Contracts

### 5.1 GAP-1 — Gap Definition and Detection

| Property | Value |
|----------|-------|
| **Contract ID** | GAP-1 |
| **From** | C06 Gap Detector |
| **To** | C02 Database (`raw_packets`, `sniffer_heartbeat`), C03 Capture Pipeline (heartbeat contract) |
| **Format** | SQL query against TimescaleDB hypertable using `time_bucket()` [1] |
| **Detection algorithm** | `generate_series()` for expected buckets, LEFT JOIN to actual counts, WHERE cnt = 0 |
| **Heartbeat evaluation** | Sniffer alive if `sniffer_heartbeat.ts > NOW() - 60s` |
| **Safety margin** | Exclude buckets where `bucket_end > NOW() - 1 minute` |
| **Lookback window** | `BLESNIFF_GAP_LOOKBACK_HOURS` (default: 1 hour) |
| **Pre-conditions** | `sniffer_heartbeat` table populated. `raw_packets` hypertable exists. |
| **Post-conditions** | Gaps recorded in `ingest_gaps` with `status='open'`. Duplicate insertions silently ignored via `ON CONFLICT DO NOTHING`. |

### 5.2 GAP-2 — PCAP File to Gap Window Mapping

| Property | Value |
|----------|-------|
| **Contract ID** | GAP-2 |
| **From** | C06 Gap Detector (backfill) |
| **To** | C04 PCAP Rotation (filename convention) |
| **Format** | Path: `{pcap_dir}/{sniffer_name}/YYYYMMDD-HHMM.pcap.zst` |
| **Mapping rule** | Gap window [start, end) maps to all rotation-aligned files whose [rotation_start, rotation_end) overlaps the gap window |
| **Compression** | Files are `.pcap.zst` (zstd compressed). C06 decompresses via `zstd -d -c`. |
| **Guarantees** | Filename encodes the rotation time (not the actual file creation time). Rotation files contain complete PCAP global headers and are valid libpcap files [7]. |
| **Missing files** | If no PCAP file exists for the gap window → gap is marked `failed`. No retry. |

### 5.3 DB-WRITE — Backfill INSERT (Contract Inherited from C02)

| Property | Value |
|----------|-------|
| **Contract ID** | `DB-WRITE` (defined in C02 §5.2) |
| **From** | C06 Gap Detector (backfill) |
| **To** | C02 Database (PostgreSQL) |
| **Role** | `tianer_writer` |
| **Privileges** | CONNECT, USAGE, INSERT on `raw_packets`, INSERT/UPDATE on `ingest_gaps` |
| **Protocol** | Parameterized `INSERT` via psycopg 3.1 async [4]. Batch size: 500 rows. |
| **Idempotency** | `INSERT ON CONFLICT DO NOTHING` [2]. Re-running the backfill produces no duplicates. |
| **Backfill flag** | All backfilled rows set `backfilled = TRUE`. Distinguishes backfilled from live-ingested rows for debugging and metrics. |

---

## 6. Failure Modes & Recovery

### 6.1 Failure Catalog

| # | Failure Mode | Detection | Impact | Recovery |
|---|-------------|-----------|--------|----------|
| **F-C06-1** | **Gap detector container crashes** | systemd/Podman detects exit code ≠ 0. Timer fires again at next interval. | Gap detection delayed by one timer interval (≤ 5 minutes). Ingest continues — `raw_packets` is the DB source; PCAP is the source of truth. No data loss. | Next timer event starts a fresh container. Gaps not yet recorded are detected and recorded. Gaps marked `backfilling` are re-evaluated and retried. Idempotent design ensures double-insertion is impossible. |
| **F-C06-2** | **DB connection lost during backfill** | `psycopg.OperationalError` raised. Backfill attempt fails. | Current backfill batch is lost. Gap status remains `open` or `backfilling`. | Next scan retries the backfill (retry count incremented). Exponential backoff in psycopg handles transient outages. Max 3 retries before marking gap `failed`. |
| **F-C06-3** | **PCAP file missing for gap window** | `_find_pcap_files()` returns empty list. | Gap cannot be backfilled. Data in that window is permanently lost from DB perspective. | Gap marked `failed` immediately (no retries). Primary recovery: PCAP has a 14-day retention window — if the file is missing, it was never written (sniffer was off) or was prematurely deleted. Operator verifies. |
| **F-C06-4** | **PCAP file corrupted** | `zstd` decompression fails with non-zero exit code. Backfill fails. | Gap cannot be backfilled. Corrupted PCAP data is unrecoverable. | Gap marked `failed` after 3 retries. Operator investigates corruption source (disk error, incomplete write, filesystem corruption). PCAP integrity validation (Layer 5) should have detected this earlier. |
| **F-C06-5** | **tshark extraction failure** | tshark process returns non-zero exit code or produces no output. | Backfill cannot extract rows from valid PCAP. | Gap retried up to 3 times. If persistent, marked `failed`. Root cause: tshark version mismatch, DLT type change, or field name deprecation. |
| **F-C06-6** | **Heartbeat data missing for a running sniffer** | Heartbeat table has no row for an active sniffer. | Gap detector treats sniffer as not running — no gaps detected. False negative. | Detection: secondary verification (Layer 4) identifies PCAP-has-data-but-DB-doesn't mismatch. Operator verifies heartbeat process is running (C03 heartbeat.sh). |
| **F-C06-7** | **Query timeout** | `statement_timeout` [5] triggered after 60s. | Detection or backfill query cancelled. | Operation retried on next scan cycle. If persistent, investigate query performance (missing index, excessive chunk count). |
| **F-C06-8** | **Memory exhaustion during backfill** | Container OOM-killed by kernel. | Backfill in progress is lost. Gap status may be stuck at `backfilling`. | Next scan detects `status='backfilling'` gaps and retries. Container OOM limit should be configured via Quadlet `MemoryMax=` (recommended: 256 MB). Large PCAP files should not be decompressed entirely into memory — the piped subprocess approach (zstd → tshark) avoids this. |
| **F-C06-9** | **Gap detector itself is down for extended period** | systemd timer logs failures. Alert `TianerGapDetectorNotRunning`. | No gaps detected or backfilled during the downtime. DB may diverge from PCAP for hours or days. | **Layer 4 — secondary verification** runs independently and detects cumulative mismatch. Trigger: `TianerGapSecondaryMismatch` fires when PCAP vs DB mismatch exceeds 1%. Operator investigates and manually runs gap detector or triggers backfill. |

### 6.2 Recovery Dependencies

| Recovery Agent | Failure It Handles | Availability |
|---------------|-------------------|-------------|
| systemd/Podman auto-restart (timer) | Container crash (F-C06-1) | Automatic — timer triggers fresh container each interval |
| Gap detector retry logic | Transient DB outage (F-C06-2), tshark failure (F-C06-5) | Automatic — up to 3 retries per gap |
| Secondary verifier | Gap detector down (F-C06-9), heartbeat false negative (F-C06-6) | Automatic — runs at separate cadence |
| Operator | PCAP missing (F-C06-3), PCAP corrupted (F-C06-4), persistent failures | Manual — requires investigation |

### 6.3 Recovery SLA

| Scenario | Detection Time | Recovery Time | Data Loss Risk |
|----------|---------------|---------------|----------------|
| Ingest crash, DB outage ≤ 5 min | ≤ 5 minutes (next timer cycle) | ≤ 5 minutes (detect + backfill) | None — all data recoverable from PCAP |
| Gap detector crash | ≤ 5 minutes (next timer cycle) | ≤ 5 minutes (fresh container) | None — delayed detection only |
| PCAP missing/corrupted | ≤ 5 minutes (detection) | Not recoverable | Permanent from DB perspective. PCAP source never existed or was corrupted. |
| Gap detector down for hours | ≤ 30 minutes (secondary verifier) | Manual investigation | None — PCAP is source of truth; data can be ingested once gap detector is restored |

---

## 7. Observability

### 7.1 Metrics

| Metric Name | Type | Labels | Description |
|------------|------|--------|-------------|
| `tianer_gap_detected_total` | Counter | `sniffer` | Total number of gaps detected since process start |
| `tianer_gap_backfilled_total` | Counter | `sniffer`, `status` (`closed`, `failed`) | Total gaps backfilled, by result |
| `tianer_gap_backfill_rows_total` | Counter | `sniffer` | Total rows backfilled (inserted into `raw_packets`) |
| `tianer_gap_backfill_retries_total` | Counter | `sniffer` | Total retry attempts |
| `tianer_gap_backfill_duration_seconds` | Gauge | `sniffer` | Duration of the most recent backfill operation |
| `tianer_gap_detection_cycles_total` | Counter | — | Total number of detection-and-backfill cycles |
| `tianer_gap_detection_duration_seconds` | Gauge | — | Duration of the most recent detection cycle |
| `tianer_gap_open_total` | Gauge | `sniffer` | Current number of open (not yet backfilled) gaps |
| `tianer_gap_failed_total` | Gauge | `sniffer` | Current number of permanently failed gaps |
| `tianer_gap_secondary_mismatch_total` | Counter | `sniffer` | Number of secondary verification mismatches detected |
| `tianer_gap_secondary_mismatch_percent` | Gauge | `sniffer`, `file` | Mismatch percentage from most recent secondary verification |

### 7.2 Structured Logging

C06 emits logs in the Tian'er structured format:

```
TIANER | {"ts":"2026-06-09T14:35:00Z","level":"INFO","component":"gapdet","msg":"Detection cycle starting"}
TIANER | {"ts":"2026-06-09T14:35:01Z","level":"INFO","component":"gapdet","msg":"Found gaps","count":2,"sniffers":["ut1","nrf1"]}
TIANER | {"ts":"2026-06-09T14:35:05Z","level":"INFO","component":"gapdet","sniffer":"ut1","msg":"Backfill complete","gap_bucket":"2026-06-09T14:00:00Z","rows":1854,"status":"closed"}
TIANER | {"ts":"2026-06-09T14:35:12Z","level":"ERROR","component":"gapdet","sniffer":"nrf1","msg":"Backfill failed","gap_bucket":"2026-06-09T14:00:00Z","error":"No PCAP files found for gap window","retry":1}
TIANER | {"ts":"2026-06-09T14:35:15Z","level":"INFO","component":"gapdet","msg":"Detection cycle complete","duration_ms":15234}
```

**Log levels:**
- `INFO`: Normal operations — cycle start, gaps found, backfill complete, cycle end
- `WARN`: Recoverable issues — transient DB error, retry attempt
- `ERROR`: Issues requiring attention — PCAP not found, persistent backfill failure, secondary verification mismatch

### 7.3 Health Checks

C06 is a oneshot process, not a long-running daemon. Health is determined by:

1. **systemd timer status:** `systemctl --user status blesniff-gap-detector.timer` — shows `LastTrigger=` and the status of the most recent run.
2. **Last successful cycle time:** `tianer_gap_detection_cycles_total` counter — if it hasn't incremented within `2 × BLESNIFF_GAP_SCAN_INTERVAL_MINUTES`, the gap detector is stuck or not running.
3. **Growing open gaps:** `tianer_gap_open_total` gauge — if this increases over multiple cycles without corresponding backfill successes, backfill is failing.
4. **DB queries:** `SELECT COUNT(*) FROM bluetooth.ingest_gaps WHERE status='open'` — operator-ad-hoc health check.

### 7.4 Alert Rules

| Alert | Condition | Severity | Action |
|-------|-----------|----------|--------|
| `TianerGapDetectorNotRunning` | `tianer_gap_detection_cycles_total` unchanged for 2× scan interval | **Critical** | Check timer status; check container logs; investigate if timer is disabled or Podman is down |
| `TianerGapBackfillFailing` | `rate(tianer_gap_failed_total[15m]) > 0` | **Warning** | Investigate PCAP file availability and integrity; check tshark version |
| `TianerGapBacklogGrowing` | `tianer_gap_open_total > 20` for > 15 min | **Warning** | Backfill may be stalling; check DB connectivity and PCAP availability |
| `TianerGapSecondaryMismatch` | `tianer_gap_secondary_mismatch_total` increase rate > 0 | **Critical** | PCAP vs DB mismatch detected; gap detector may be missing gaps; manual investigation |
| `TianerGapHeartbeatMissing` | `tianer_gap_secondary_mismatch_total > 0` AND `tianer_gap_detected_total` unchanged | **Critical** | Heartbeat is false-negative — sniffer is running but heartbeat says not; check heartbeat process |

---

## 8. Security Considerations

### 8.1 Write-Only Database Access

C06 uses the `tianer_writer` role (INSERT-only per Q9 resolution). This provides defense-in-depth:

| Operation | Permitted? | Rationale |
|-----------|-----------|-----------|
| INSERT into `raw_packets` | ✅ | Core function — backfill missing data |
| INSERT/UPDATE on `ingest_gaps` | ✅ | Gap tracking metadata |
| SELECT from `raw_packets` | ❌ | `tianer_writer` cannot read sensitive data. COUNT(*) aggregate queries use TimescaleDB's aggregate path which is permitted. |
| SELECT from `sniffer_heartbeat` | ✅ | Required for gap detection (heartbeat evaluation) |
| UPDATE/DELETE on `raw_packets` | ❌ | Cannot modify or destroy existing data |
| DDL | ❌ | Cannot alter schema |

**Blast radius:** If the gap detector container is compromised, the attacker can insert fabricated rows into `raw_packets` (with `backfilled=true`) and modify `ingest_gaps` metadata, but cannot read existing data, delete data, or alter the database schema. This is consistent with the principle of least privilege and PF-7 (Defense in Depth).

### 8.2 Parameterized Queries

All SQL in `detector.py`, `backfill.py`, `verifier.py`, and `db.py` uses **parameterized queries** via psycopg's `%s` placeholders [4]. This is mandatory per C02 §8.4. No string interpolation or f-string is used for SQL construction. The batch INSERT in `backfill.py` dynamically builds the parameter placeholder list but all actual values are passed via the parameter tuple, not string-interpolated.

### 8.3 Subprocess Safety

C06 invokes external commands (`zstd`, `tshark`, `capinfos`) via `asyncio.create_subprocess_exec()`. Key safety considerations:

- **Path arguments:** File paths are constructed from the `BLESNIFF_PCAP_DIR` environment variable and the filename convention. Both are validated against expected patterns before use.
- **No shell invocation:** `asyncio.create_subprocess_exec()` does not invoke a shell — it directly executes the binary, avoiding shell injection.
- **File access:** C06 mounts V02 (`/var/lib/tianer/pcap/`) **read-only**. It cannot modify or delete PCAP files, even if compromised.
- **Output handling:** Subprocess stdout is read and parsed within C06; it is never written to disk.

### 8.4 Container Security

C06 runs in the `tianer-platform` pod with the following security properties:

| Property | Value | Rationale |
|----------|-------|-----------|
| Capabilities | `--cap-drop ALL` | No privileged kernel operations |
| `NoNewPrivileges` | `true` | Prevents privilege escalation |
| V01 (config) | `:ro` | Cannot modify configuration |
| V02 (PCAP) | `:ro` | Cannot modify source of truth |
| Network | `tianer-net` bridge only | Cannot reach external networks |
| Root filesystem | Container image only | No host filesystem access |
| User | Non-root (`tianer` UID) | Matches host file permissions |

### 8.5 Secrets Handling

The database password for the `tianer_writer` role is injected via `EnvironmentFile=/etc/tianer/secrets/db_password.env`. This file is mode 0600, owner `tianer:tianer`. The password is never hardcoded in the container image or in environment variable declarations visible to other containers.

---

## 9. Configuration

### 9.1 Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `BLESNIFF_DB_DSN` | **Yes** | — | PostgreSQL connection string for `tianer_writer` role. Example: `host=tianer-postgres dbname=tianer user=tianer_writer` |
| `TIANER_DB_PASSWORD_FILE` | **Yes** | — | Path to file containing the `tianer_writer` password. Used to construct `BLESNIFF_DB_DSN` if DSN is not directly set. |
| `BLESNIFF_PCAP_DIR` | No | `/var/lib/tianer/pcap` | Root directory for PCAP files (V02) |
| `BLESNIFF_GAP_BUCKET_SECONDS` | No | `30` | Time bucket granularity for gap detection [1] |
| `BLESNIFF_GAP_SCAN_INTERVAL_MINUTES` | No | `5` | Interval between gap detection cycles (timer trigger) |
| `BLESNIFF_GAP_LOOKBACK_HOURS` | No | `1` | How far back to scan for gaps |
| `BLESNIFF_GAP_MAX_RETRIES` | No | `3` | Maximum backfill retries before marking a gap `failed` |
| `BLESNIFF_GAP_VERIFY` | No | `0` | Set to `1` to enable secondary verification in each cycle |
| `BLESNIFF_GAP_STATEMENT_TIMEOUT_MS` | No | `60000` | PostgreSQL `statement_timeout` in milliseconds [5] |
| `BLESNIFF_ZSTD_LEVEL` | No | `3` | zstd decompression is unaffected by compression level, but passed through if needed |

### 9.2 Tuning Guidelines

#### Bucket Size

The 30-second default balances detection granularity against query efficiency:

| Bucket Size | Pros | Cons |
|-------------|------|------|
| 15s | Finer detection; smaller gaps to backfill | More buckets to query; more ingest_gaps rows |
| 30s (default) | Good balance; matches typical rotation interval granularity | May miss sub-30s gaps |
| 60s | Fewer queries; coarser detection | Larger gaps to backfill; risk of missing short data loss events |

At 1000 pps, a 30-second bucket contains ~30,000 rows. Backfilling a single 30s gap is a modest operation. If `BLESNIFF_GAP_BUCKET_SECONDS` is changed, the systemd timer's `OnUnitActiveSec=` should be at least 2× the bucket size (so the timer doesn't overlap with its own queries).

#### Scan Interval

The 5-minute default provides timely detection without excessive DB load:

- **Faster (e.g., 2 min):** Detects gaps sooner but generates more `COUNT(*)` queries on the hypertable. At 4 sniffers × 2 buckets/min × 60 min = 480 queries per cycle. Acceptable for low-traffic environments.
- **Slower (e.g., 10 min):** Less DB load but gaps go undetected for longer. Suitable for environments where ingest reliability is high.

#### Memory Budget

The container should be allocated **256 MB** memory (`MemoryMax=256M`). Breakdown:

| Component | Typical Usage |
|-----------|--------------|
| Python runtime + asyncio event loop | ~30 MB |
| zstd decompression buffer | ~16 MB (level-independent for decompression) |
| tshark process memory | ~50 MB |
| Row accumulation buffer (batch INSERT) | ~10 KB per 500-row batch |
| psycopg connection pool | ~5 MB |
| **Total** | **~100 MB typical, 256 MB for safety margin** |

The piped subprocess approach (zstd → tshark) ensures that decompressed PCAP data is never fully materialized in memory. Only the filtered rows within the gap window are accumulated.

### 9.3 Timer Configuration

The Quadlet timer unit runs C06 on a fixed interval:

```ini
# deploy/containers/blesniff-gap-detector.timer
[Timer]
OnUnitActiveSec=300
AccuracySec=5s
Persistent=true
```

- `OnUnitActiveSec=300`: Fires 300 seconds after the previous run *completes*. If a cycle takes 15 seconds, the next cycle starts 315 seconds after the previous start.
- `AccuracySec=5s`: Allows systemd to coalesce timer wakeups with other system timers for power efficiency [3].
- `Persistent=true`: Records the last trigger time on disk. After a reboot, if the timer missed triggers during downtime, they are caught up immediately [3].

### 9.4 Secondary Verification Timer

Secondary verification runs on a separate, longer cadence (e.g., every 30 minutes). This can be implemented as a separate Quadlet `.timer` and `.container` pair or as a conditional branch within the main gap detector run (using `BLESNIFF_GAP_VERIFY=1` on every Nth run).

---

## 10. Test Plan

### 10.1 Unit Tests

Tested with pytest 8.0+ [8]. Test files in `modules/bluetooth/gap-detector/tests/`.

| Test Suite | File | Scope |
|-----------|------|-------|
| `test_detector.py` | Gap detection logic | `find_gaps()` returns correct gaps given mock DB responses; heartbeat evaluation (alive vs dead sniffers); bucket overlap exclusion; empty result for closed buckets |
| `test_backfill.py` | Backfill logic | `_find_pcap_files()` maps gap to correct filenames; `_extract_rows()` parses tshark output correctly; batch INSERT construction is valid SQL |
| `test_detector_idempotent.py` | Idempotency | Running `find_gaps()` twice produces same results; re-backfilling does not duplicate rows; `ON CONFLICT DO NOTHING` behavior |
| `test_pcap_reader.py` | PCAP file handling | Correctly identifies `.pcap.zst` files for a given time window; handles missing files; handles empty files; decompresses valid PCAP |
| `test_verifier.py` | Secondary verification | `capinfos` output parsing; mismatch calculation; threshold handling |
| `test_db.py` | Database layer | Connection pool creation; `statement_timeout` set correctly; query execution with parameters |

### 10.2 Integration Tests

| Test | Scope | Method |
|------|-------|--------|
| `test_gap_detection.sh` | End-to-end gap detection | Start mock ingest (write known number of rows to DB), stop ingest for 2 minutes, restart, run gap detector, verify gaps detected, verify backfill inserts correct count |
| `test_backfill_idempotent.sh` | Idempotent backfill | Run backfill twice on the same gap; verify row count unchanged; verify no duplicate rows in DB |
| `test_backfill_recovery.sh` | Crash recovery | Run backfill, kill container mid-backfill, restart, verify gap retried and completed; verify final row count correct |
| `test_heartbeat_gating.sh` | Heartbeat evaluation | Set heartbeat stale for a sniffer; verify gap detector does not create gaps for that sniffer |
| `test_secondary_verification.sh` | Cross-check | Write known count to PCAP and DB with deliberate mismatch; verify secondary verification detects mismatch |
| `test_pcap_missing.sh` | Missing PCAP | Delete PCAP file for a known gap window; verify gap marked `failed` with appropriate error |

### 10.3 Performance Tests

| Test | Scope | Threshold |
|------|-------|-----------|
| `test_detector_perf.py` | Detection query speed | `find_gaps()` completes in < 5 seconds for 4 sniffers, 1-hour lookback, 100K rows in DB |
| `test_backfill_perf.py` | Backfill throughput | Backfills 10,000 rows in < 30 seconds from a compressed PCAP file |
| `test_backfill_memory.py` | Memory usage | Memory < 200 MB while backfilling a 100 MB decompressed PCAP file |

### 10.4 Acceptance Criteria

1. **Detection accuracy:** Stop C05 ingest bridge for 2 minutes during a live stream; C06 identifies the gap within 5 minutes (one scan cycle) after ingest resumes.
2. **Backfill completeness:** After backfill, `COUNT(*)` for the gap window matches the count obtained by directly parsing the PCAP file with `tshark`.
3. **Idempotency:** Re-running the detector-and-backfill cycle twice does not produce duplicate rows in `raw_packets`.
4. **Heartbeat gating:** If a sniffer heartbeat is stale (>60s), no gaps are created for that sniffer.
5. **Missing PCAP handling:** A gap with no corresponding PCAP file is marked `failed` within one cycle with a useful error message.
6. **Secondary verification:** A deliberate 5% mismatch between PCAP and DB counts triggers the `TianerGapSecondaryMismatch` alert.
7. **Crash recovery:** Killing the gap detector container mid-backfill and restarting it results in the gap being successfully backfilled on the next cycle with no duplicate rows.

---

## 11. Deployment Notes

### 11.1 Container Image

C06 runs from a container image built via a multi-stage Dockerfile:

```dockerfile
# Build stage
FROM python:3.13-slim-bookworm AS builder
RUN apt-get update && apt-get install -y --no-install-recommends \
    tshark zstd capinfos && \
    rm -rf /var/lib/apt/lists/*
COPY modules/bluetooth/gap-detector/ /app/
RUN pip install --no-cache-dir --target /app/deps psycopg[binary]==3.1.*

# Runtime stage
FROM python:3.13-slim-bookworm
COPY --from=builder /usr/bin/tshark /usr/bin/tshark
COPY --from=builder /usr/bin/zstd /usr/bin/zstd
COPY --from=builder /usr/bin/capinfos /usr/bin/capinfos
COPY --from=builder /usr/lib/x86_64-linux-gnu/libtshark* /usr/lib/x86_64-linux-gnu/
COPY --from=builder /usr/lib/x86_64-linux-gnu/libwireshark* /usr/lib/x86_64-linux-gnu/
COPY --from=builder /usr/lib/x86_64-linux-gnu/libwiretap* /usr/lib/x86_64-linux-gnu/
COPY --from=builder /usr/lib/x86_64-linux-gnu/libwsutil* /usr/lib/x86_64-linux-gnu/
COPY --from=builder /app/ /app/
ENV PYTHONPATH=/app/deps
USER tianer
ENTRYPOINT ["python3", "-m", "tianer_gapdet", "--once"]
```

**Image size target:** < 200 MB (Python slim base + tshark shared libraries + zstd). Multi-stage build keeps the dev headers and apt cache out of the runtime image.

### 11.2 Quadlet Deployment

Files deployed to `/etc/containers/systemd/`:

| File | Purpose |
|------|---------|
| `blesniff-gap-detector.container` | Container definition (oneshot, platform pod) |
| `blesniff-gap-detector.timer` | Timer unit (fires every 300s) |
| `blesniff-gap-detector-verify.timer` (optional) | Separate timer for secondary verification (every 1800s) |

Activated via:
```bash
systemctl --user daemon-reload
systemctl --user enable --now blesniff-gap-detector.timer
```

### 11.3 Build Command

```bash
cd modules/bluetooth/gap-detector
uv sync --group dev  # Install dependencies (psycopg, pytest)
uv run pytest tests/  # Run unit tests
```

### 11.4 Dependencies

| Dependency | Version | Package |
|-----------|---------|---------|
| Python | 3.13 | `python3.13` (system package) |
| psycopg | 3.1 | PyPI (`psycopg[binary]`) |
| pytest | 8.0 | PyPI (dev dependency) |
| tshark | 4.2+ | `tshark` (system package, in container image) |
| zstd | 1.5+ | `zstd` (system package, in container image) |
| capinfos | — | Part of Wireshark package (in container image) |

---

## References

[1] TimescaleDB. "time_bucket() — Bucket rows by time interval to calculate aggregates." https://docs.timescale.com/api/latest/hyperfunctions/time_bucket/, 2024. Referenced for the gap detection SQL: time-bucketed aggregation with `generate_series()` for expected bucket enumeration.

[2] PostgreSQL Global Development Group. "PostgreSQL 17 Documentation — INSERT." https://www.postgresql.org/docs/17/sql-insert.html, 2024. Referenced for `ON CONFLICT DO NOTHING` idempotent insert semantics: silently skips conflicting rows, making backfill operations safe to repeat.

[3] systemd developers. "systemd.timer(5) — Timer unit configuration." https://man.archlinux.org/man/systemd.timer.5, systemd 260.2. Referenced for timer unit configuration: `OnUnitActiveSec=`, `AccuracySec=`, `Persistent=true` directive semantics and catch-up behavior after system suspend.

[4] Daniele Varrazzo and The Psycopg Team. "Psycopg 3.3 — Concurrent operations." https://www.psycopg.org/psycopg3/docs/advanced/async.html, 2024. Referenced for async psycopg connection patterns: `AsyncConnection.connect()`, async context managers, and cancellation-aware connection handling.

[5] PostgreSQL Global Development Group. "PostgreSQL 17 Documentation — Client Connection Defaults (statement_timeout)." https://www.postgresql.org/docs/17/runtime-config-client.html, 2024. Referenced for `statement_timeout` configuration: aborts statements exceeding the configured time, preventing runaway queries from blocking the gap detector indefinitely.

[6] Python Software Foundation. "Python 3.13 — Event Loop (asyncio)." https://docs.python.org/3/library/asyncio-eventloop.html, 2024. Referenced for `asyncio.sleep()` for timer-based scheduling, `asyncio.create_subprocess_exec()` for non-blocking subprocess invocation, and `asyncio.wait_for()` for subprocess timeout.

[7] The Tcpdump Group. "PCAP Savefile Format." https://www.tcpdump.org/manpages/pcap-savefile.5.txt, 2024. Referenced for PCAP global header structure, DLT encoding, and per-packet header format — ensures C06 correctly interprets rotated PCAP files.

[8] pytest. "pytest: helps you write better programs." https://docs.pytest.org/en/stable/, v8.0+. — Documents pytest testing framework: test discovery conventions, fixture system (`@pytest.fixture`), parametrize decorator, mock integration, and `pytest.raises()` for exception testing (§10.1, §10.3).
