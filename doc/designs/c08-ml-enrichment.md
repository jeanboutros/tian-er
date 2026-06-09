# C08 — ML Enrichment

**Status:** Phase A Design — guides Phase B implementation
**Author:** Code Architect
**Date:** 2026-06-09
**Dependencies:** C07 (Deep Parser), C02 (Database)
**Blocks:** — (terminal component in enrichment pipeline)

---

## 1. Overview

### 1.1 Purpose

C08 ML Enrichment is the **rule-based device classification service** for the Tian'er Signal Intelligence Platform. It reads JSONL output from the C07 Deep Parser, matches every observation line against a catalog of known device signatures — manufacturer IDs and service UUIDs — classifies each device into one or more device classes, and writes the classification results to the `device_summary.enrichment_data` JSONB column and the `device_enrichment` table in the PostgreSQL database.

In the v1 release, the "ML" in the component name refers to **rule-based classification** only. Machine learning models (e.g., unsupervised clustering of BLE advertising patterns) are deferred to post-MVP. The v1 rule engine is grounded in the Bluetooth SIG Company Identifier registry [1] and the Bluetooth GATT Service UUID registry — every rule is traceable to a documented manufacturer or service identifier.

### 1.2 Scope

| In Scope | Out of Scope |
|----------|-------------|
| Read C07 JSONL lines from V05 (`/var/lib/tianer/data/`) | Produce JSONL (C07 responsibility) |
| Extract `manufacturer_id`, `service_uuids_16`, `service_uuids_128`, `local_name`, `flags` from each JSONL line | Parse BLE PDUs or PCAP files (C07 responsibility) |
| Map manufacturer ID to known vendor names via Bluetooth SIG Company ID lookup table | OUI lookup for Wi-Fi MACs (future Wi-Fi module) |
| Classify devices by rule-based matching: vendor + service UUID → device class | Machine learning model training or inference (post-MVP) |
| Write classification results to `device_summary.enrichment_data` JSONB | Update `device_summary.residency_class` (C02 `classify_residency()` function) |
| Insert per-observation records into `device_enrichment` table | Query or serve enrichment data (C09 REST API responsibility) |
| Compute a residency score per MAC from observation frequency within the JSONL batch | Real-time stream processing (C08 is batch; runs on C07 output) |
| Log unclassified observations with full context for operator review | Auto-label unknown manufacturers |

### 1.3 Boundaries

C08 is a **batch consumer** at the end of the deep-parsing pipeline. It does not run continuously — it is triggered by a Quadlet one-shot timer after C07 completes a batch run. It reads C07's JSONL output (mounted read-only from V05), classifies devices, writes to PostgreSQL via the `tianer_writer` role, and exits. If no new JSONL files exist since the last run, it exits cleanly with zero work done.

```
┌───────────────────────────────────────────────────────────────┐
│                      tianer-platform pod                        │
│                                                               │
│  ┌───────────────────────────────────────────────────────┐   │
│  │  C07 Deep Parser                                        │   │
│  │  Reads PCAP (V02 :ro)                                   │   │
│  │  Writes JSONL → V05 (:rw)                               │   │
│  └──────────────────────┬────────────────────────────────┘   │
│                         │ JSONL files                          │
│                         ▼                                      │
│  ┌───────────────────────────────────────────────────────┐   │
│  │  C08 ML Enrichment                                      │   │
│  │  Reads JSONL (V05 :ro)                                  │   │
│  │  Classifies devices                                     │   │
│  │  Writes → PostgreSQL (tianer_writer)                    │   │
│  └───────────────────────────────────────────────────────┘   │
│                                                               │
└───────────────────────────────────────────────────────────────┘
        │
        │ psycopg 3.1 async (parameterized INSERT/UPDATE)
        │
        ▼
┌───────────────────────┐
│  C02 PostgreSQL        │
│  bluetooth schema:     │
│  - device_enrichment   │
│  - device_summary      │
│    .enrichment_data    │
└───────────────────────┘
```

### 1.4 Position in the System

C08 is a **Layer 3 v1 Bluetooth module component** in the build sequence (component-breakdown.md §4.1). It depends on C07 (Deep Parser) for its input and C02 (Database) for its output. It blocks nothing — it is a terminal component in the enrichment pipeline. It is built alongside C10 (Frontend) and C11 (Grafana Dashboards) in Layer 3.

C08 runs as a **Quadlet one-shot container** triggered by a systemd timer — it is not a long-running service. Each invocation processes new JSONL files since the last run and exits. This design avoids persistent memory consumption and simplifies error recovery: a failed run is retried by the next timer tick.

---

## 2. High-Level Architecture (HLA)

### 2.1 Component Diagram

```
┌────────────────────────────────────────────────────────────────────┐
│                         C08 ML Enrichment                           │
│                                                                    │
│  ┌───────────────┐   ┌──────────────────┐   ┌──────────────────┐  │
│  │  JSONL Reader  │──▶│ Rule Classifier   │──▶│   DB Writer      │  │
│  │  (runner.py)   │   │  (classifier.py)  │   │   (runner.py)    │  │
│  │               │   │                  │   │                  │  │
│  │  - Scan V05   │   │  - OUI lookup    │   │  - Upsert        │  │
│  │    for new    │   │  - Service UUID  │   │    device_summary │  │
│  │    JSONL files│   │    matcher       │   │    .enrichment_  │  │
│  │  - Parse each │   │  - Device class  │   │    data           │  │
│  │    line into  │   │    assignment    │   │  - INSERT INTO   │  │
│  │    Observation│   │  - Compute       │   │    device_enrich- │  │
│  │    objects    │   │    residency     │   │    ment           │  │
│  │               │   │    score         │   │  - Parameterized │  │
│  └───────────────┘   └──────────────────┘   └──────────────────┘  │
│                                                                    │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │  State Tracker (state.json on V05)                            │ │
│  │  - Tracks last processed JSONL file cursor position            │ │
│  │  - Enables idempotent re-processing after crash                │ │
│  └──────────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────────┘
```

### 2.2 Data Flow

```
1. TIMER FIRES (systemd timer, e.g., every 5 minutes)
2. C08 container starts
3. runner.py scans V05 for JSONL files newer than last cursor (state.json)
4. For each new JSONL file (or new portion of active file):
   a. Read one JSON line at a time (streaming, not full-file load)
   b. Validate required fields (ts, mac, sniffer_id)
   c. Extract features: manufacturer_id, service_uuids, local_name, flags
   d. Run classifier rules to produce device classes
   e. Compute residency score from per-MAC observation count in this batch
   f. Batch INSERT into device_enrichment (100 rows at a time)
   g. Accumulate per-MAC classification results for device_summary update
5. After file processing complete:
   a. UPDATE device_summary.enrichment_data for each MAC observed
   b. Write state.json with new cursor position
6. Container exits 0
```

### 2.3 Neighbouring Components

| Neighbour | Direction | Contract | Description |
|-----------|-----------|----------|-------------|
| C07 Deep Parser | Upstream (input) | ML-1 (JSONL) | Writes JSONL files to V05. C08 reads them read-only. |
| C02 Database | Downstream (output) | DB-WRITE (DB role) | C08 writes to `device_enrichment` and updates `device_summary.enrichment_data` via the `tianer_writer` role. |
| C09 REST API | Downstream (consumer) | — | API reads `device_summary.enrichment_data` to serve device classes to the frontend. C08 is unaware of C09. |
| C11 Grafana | Downstream (consumer) | — | Grafana dashboards query `device_enrichment` for enrichment details. C08 is unaware of C11. |

### 2.4 Invocation Model

C08 is a **batch processor**, not a streaming service. It processes C07's output in the same order C07 writes it:

```
C07 Deep Parser (batch, runs on rotated PCAP)
    │
    │ writes JSONL files to V05:
    │   /var/lib/tianer/data/ut1/20260606-1200.jsonl
    │   /var/lib/tianer/data/nrf1/20260606-1200.jsonl
    │
    ▼
C08 ML Enrichment (batch, triggered by timer after C07)
    │
    │ reads JSONL from V05 (:ro)
    │ writes classification to PostgreSQL
    │
    ▼
Exit 0. Next timer tick processes any new files.
```

**Timer granularity:** 5 minutes by default. This means device enrichment data may be up to 5 minutes behind PCAP rotation. This is acceptable because enrichment is a non-critical enhancement — the `raw_packets` and `device_summary` tables are already populated by the real-time ingest pipeline before enrichment runs.

---

## 3. Data Model

### 3.1 Input: DEEP-1 JSONL Schema

C08 reads JSONL files conforming to the **DEEP-1 contract** (produced by C07 Deep Parser). Each line is a self-contained JSON object: `ts`, `mac`, and `sniffer_id` are mandatory; all other fields — including `advdata.manufacturer.company_id` (the Bluetooth SIG Company Identifier, a 16-bit value assigned by the Bluetooth SIG [1]) and `advdata.service_uuids_16` — are optional but provide the classification inputs.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `ts` | ISO 8601 string | **Yes** | Observation timestamp. Used as `observed_ts` in `device_enrichment`. |
| `sniffer_id` | integer | **Yes** | Sniffer that captured the packet. |
| `mac` | string (colon-hex) | **Yes** | 6-byte Bluetooth device address, e.g., `"aa:bb:cc:dd:ee:ff"`. Used as the primary key for classification. |
| `address_type` | string | No | `"public"` or `"random"`. Informational — stored but not used for classification in v1. |
| `rssi` | integer | No | Received signal strength in dBm. Informational. |
| `channel` | integer | No | BLE advertising channel (37, 38, 39). Informational. |
| `pdu_type` | string | No | Advertising PDU type name, e.g., `"ADV_IND"`. Informational. |
| `advdata` | object | **Yes** (may be empty) | Parsed advertising data structure. May be `{}` if no AD structures were parsed. |
| `advdata.flags` | integer | No | BLE Flags AD type value. |
| `advdata.local_name` | string | No | Device local name from AD type 0x08 or 0x09. Stored for human display; not used for classification in v1. |
| `advdata.tx_power` | integer | No | TX Power level in dBm. |
| `advdata.service_uuids_16` | array of strings | No | 16-bit service UUIDs, e.g., `["180D", "180F"]`. **Primary input for service-based classification.** |
| `advdata.service_uuids_128` | array of strings | No | 128-bit service UUIDs. Also checked for classification. |
| `advdata.manufacturer` | object | No | Manufacturer-specific AD type 0xFF data. |
| `advdata.manufacturer.company_id` | string (hex) | No | Bluetooth SIG Company Identifier, e.g., `"004C"`. **Primary input for manufacturer-based classification.** |
| `advdata.manufacturer.data_hex` | string (hex) | No | Associated manufacturer data payload. Stored in `device_enrichment.manufacturer_data`. |
| `advdata.service_data` | array | No | Service Data AD type. Informational in v1. |
| `raw_advdata_hex` | string | No | Full raw advertising data hex. Stored for debugging. |

**Contract enforcement:** If `ts`, `mac`, or `sniffer_id` is missing from a line, C08 logs the parse error, increments a counter, and skips the line. If `advdata` is `{}` (no AD structures parsed), C08 stores the observation but does not classify — no manufacturer or service data is available to match against.

### 3.2 Output: device_enrichment Table

C08 inserts one row per valid JSONL line into `bluetooth.device_enrichment`. Table schema (owned by C02, migration `0005_device_enrichment.sql`):

| Column | Type | Source Field | Description |
|--------|------|-------------|-------------|
| `id` | `BIGSERIAL` | Auto | Synthetic primary key. |
| `mac_address` | `BYTEA` | `mac` (decoded from hex) | 6-byte BD_ADDR. |
| `observed_ts` | `TIMESTAMPTZ` | `ts` (parsed to timestamp) | Observation timestamp. |
| `sniffer_id` | `SMALLINT` | `sniffer_id` | Source sniffer. |
| `local_name` | `TEXT` | `advdata.local_name` | Device name if present. |
| `service_uuids_16` | `TEXT[]` | `advdata.service_uuids_16` | PostgreSQL array of 16-bit UUIDs. |
| `service_uuids_128` | `TEXT[]` | `advdata.service_uuids_128` | PostgreSQL array of 128-bit UUIDs. |
| `manufacturer_id` | `INT` | `advdata.manufacturer.company_id` (hex→int) | Bluetooth SIG Company ID as integer. |
| `manufacturer_data` | `BYTEA` | `advdata.manufacturer.data_hex` (decoded) | Raw manufacturer data bytes. |
| `tx_power` | `SMALLINT` | `advdata.tx_power` | TX Power in dBm. |
| `flags` | `SMALLINT` | `advdata.flags` | BLE Flags bitfield. |
| `raw_advdata` | `BYTEA` | `raw_advdata_hex` (decoded) | Full raw advertising data. |

**Insert strategy:** Parameterized batch INSERT with `ON CONFLICT (mac_address, observed_ts, sniffer_id) DO NOTHING`. This makes the operation idempotent — re-running C08 on the same JSONL file will not create duplicate rows.

### 3.3 Output: device_summary.enrichment_data

C08 updates the `enrichment_data` JSONB column in `bluetooth.device_summary` for each MAC address that appears in the processed JSONL batch. PostgreSQL's JSONB type [4] stores JSON data in a decomposed binary format that supports indexing and the `||` merge operator — both used by C08 to accumulate classification results across runs.

```json
{
  "classes": ["apple_continuity", "battery_service_device"],
  "vendor": "Apple, Inc.",
  "device_name": "MyDevice",
  "last_enriched": "2026-06-09T14:30:00Z",
  "class_details": {
    "apple_continuity": {
      "match_type": "manufacturer",
      "manufacturer_id": 76,
      "confidence": "high"
    },
    "battery_service_device": {
      "match_type": "service_uuid",
      "service_uuid": "180F",
      "confidence": "high"
    }
  },
  "residency_score": 0.85,
  "observation_count": 423,
  "observation_window_hours": 24,
  "unclassified": false
}
```

| Field | Type | Description |
|-------|------|-------------|
| `classes` | `string[]` | List of device class labels assigned by the rule engine. Empty array if unclassified. |
| `vendor` | `string` or `null` | Human-readable vendor name from the Company ID lookup table. `null` if manufacturer unknown. |
| `device_name` | `string` or `null` | Most recently observed `local_name` for this MAC. `null` if never observed. |
| `last_enriched` | ISO 8601 string | When C08 last processed observations for this MAC. |
| `class_details` | `object` | Per-class metadata: which rule matched and with what confidence. |
| `residency_score` | `float` (0.0–1.0) | Heuristic score for how "resident" this device appears based on observation frequency within the processed batch. Higher = more consistently present. |
| `observation_count` | `integer` | Number of observations for this MAC in the current batch. |
| `observation_window_hours` | `float` | Time span of observations for this MAC in the current batch (max_ts − min_ts). |
| `unclassified` | `boolean` | `true` if no rules matched. The MAC was observed but no device class could be assigned. |

**Update strategy:** Upsert via `INSERT ... ON CONFLICT (mac_address) DO UPDATE` of the `enrichment_data` column. The update **merges** new classes with existing ones (set union), so that multiple C08 runs accumulate rather than overwrite classifications.

### 3.4 Entity-Relationship Summary

```
device_enrichment (per-observation)          device_summary (per-MAC)
┌──────────────────────────────────┐        ┌────────────────────────────┐
│ mac_address (FK)                 │───1:N──│ mac_address (PK)           │
│ observed_ts                      │        │ ...                        │
│ sniffer_id                       │        │ enrichment_data JSONB:     │
│ local_name                       │        │   classes[]                │
│ service_uuids_16[], _128[]       │        │   vendor                   │
│ manufacturer_id                  │        │   residency_score          │
│ manufacturer_data                │        │   unclassified             │
│ ...                              │        └────────────────────────────┘
└──────────────────────────────────┘                  ▲
        ▲                                             │
        │ POPULATED BY C08                             │ UPDATED BY C08
        │ (INSERT per observation)                     │ (UPSERT per MAC)
        │                                             │
   ┌────┴────────────┐                                │
   │ C07 Deep Parser  │                                │
   │ JSONL files (V05)│                                │
   └─────────────────┘                                │
                                                      │
                    ┌─────────────────────────────────┘
                    │
            ┌───────┴───────┐
            │ C09 REST API   │
            │ Reads classes  │
            │ from JSONB     │
            └───────────────┘
```

---

## 4. Low-Level Architecture (LLA)

### 4.1 Module Structure

```
modules/bluetooth/ml-enrichment/
├── pyproject.toml
├── src/tianer_ml/
│   ├── __init__.py
│   ├── runner.py          # CLI entry point, main orchestration
│   ├── classifier.py      # Rule matching engine
│   ├── features.py        # Feature extraction from JSONL lines
│   ├── oui_lookup.py      # Manufacturer ID → vendor name lookup table
│   ├── residency.py       # Residency score computation
│   ├── db.py              # PostgreSQL writer (psycopg 3.1 async)
│   └── state.py           # Cursor state tracking (state.json)
└── tests/
    ├── conftest.py
    ├── test_classifier.py
    ├── test_oui_lookup.py
    ├── test_residency.py
    ├── test_runner.py
    └── fixtures/
        ├── sample_apple.jsonl
        ├── sample_eddystone.jsonl
        ├── sample_unclassified.jsonl
        └── sample_malformed.jsonl
```

### 4.2 runner.py — Orchestration Logic

```python
"""
runner.py — CLI entry point for C08 ML Enrichment.

Reads JSONL from C07 output directory, classifies each observation,
writes to device_enrichment and device_summary tables.

Usage:
    python -m tianer_ml.runner --data-dir /var/lib/tianer/data

Env vars:
    TIANER_DB_HOST, TIANER_DB_PORT, TIANER_DB_NAME
    TIANER_DB_USER, TIANER_DB_PASSWORD
"""

import argparse
import asyncio
import json
import logging
import os
from pathlib import Path

from .classifier import classify_observation, DeviceClass
from .features import extract_features, Observation
from .db import DBWriter
from .state import StateTracker, Cursor

logger = logging.getLogger(__name__)


async def process_jsonl_file(
    filepath: Path,
    writer: DBWriter,
    state: StateTracker,
) -> int:
    """Process one JSONL file. Returns number of lines processed."""
    cursor = state.get_cursor(str(filepath))
    lines_processed = 0

    with open(filepath) as f:
        if cursor.line_offset > 0:
            f.seek(cursor.byte_offset)

        for line_no, line in enumerate(f, start=1):
            if line_no <= cursor.line_offset:
                continue

            try:
                obs = extract_features(line)
            except (json.JSONDecodeError, ValueError, KeyError) as e:
                logger.warning("JSONL parse error in %s line %d: %s",
                               filepath.name, line_no, e)
                writer.increment_malformed()
                continue

            classes = classify_observation(obs)
            obs.classes = classes
            await writer.write_observation(obs)

            lines_processed += 1

            if lines_processed % 1000 == 0:
                logger.info("Processed %d lines from %s",
                            lines_processed, filepath.name)

    # Flush remaining batched writes
    await writer.flush()

    # Update cursor
    state.update_cursor(str(filepath), Cursor(line_count=lines_processed))
    return lines_processed


async def main():
    parser = argparse.ArgumentParser(description="Tian'er ML Enrichment")
    parser.add_argument(
        "--data-dir",
        default="/var/lib/tianer/data",
        help="Directory containing C07 JSONL output files"
    )
    parser.add_argument(
        "--batch-size",
        type=int,
        default=100,
        help="Rows per DB insert batch"
    )
    parser.add_argument(
        "--state-file",
        default="/var/lib/tianer/data/state.json",
        help="Cursor state file path"
    )
    args = parser.parse_args()

    data_dir = Path(args.data_dir)
    if not data_dir.exists():
        logger.error("Data directory %s does not exist", data_dir)
        return 1

    state = StateTracker(Path(args.state_file))
    writer = DBWriter(batch_size=args.batch_size)

    try:
        await writer.connect()

        total_lines = 0
        # Process JSONL files sorted by name (which encodes timestamp)
        for jsonl_file in sorted(data_dir.glob("**/*.jsonl")):
            lines = await process_jsonl_file(jsonl_file, writer, state)
            total_lines += lines
            logger.info("Completed %s: %d lines", jsonl_file.name, lines)

        # After all observations processed, update device_summary
        await writer.update_device_summaries()

        logger.info("Run complete. Total lines: %d", total_lines)
        return 0

    except Exception as e:
        logger.exception("Fatal error: %s", e)
        return 1
    finally:
        await writer.close()


if __name__ == "__main__":
    logging.basicConfig(
        level=logging.INFO,
        format='TIANER | {"ts":"%(asctime)s","level":"%(levelname)s",'
               '"component":"ml-enrichment","msg":"%(message)s"}'
    )
    exit(asyncio.run(main()))
```

`runner.py` uses Python 3.13 `asyncio` [2] as its async runtime. The `asyncio.run()` function creates an event loop, runs the `main()` coroutine, and provides structured cancellation — all state (DB connections, `ResidencyTracker`) is cleaned up when the coroutine exits.

### 4.3 features.py — Feature Extraction

```python
"""
features.py — Extract structured observation from a JSONL line.

Parses a single DEEP-1 JSONL line and produces an Observation object
with normalized fields for the classifier.
"""

from dataclasses import dataclass, field
from datetime import datetime
from typing import Optional
import json


@dataclass
class Observation:
    """A single device observation extracted from a JSONL line."""
    ts: datetime
    mac: str                    # Colon-hex string, e.g., "aa:bb:cc:dd:ee:ff"
    sniffer_id: int
    address_type: Optional[str] = None    # "public" or "random"
    rssi: Optional[int] = None
    channel: Optional[int] = None
    pdu_type: Optional[str] = None
    # Parsed advertising data
    flags: Optional[int] = None
    local_name: Optional[str] = None
    tx_power: Optional[int] = None
    service_uuids_16: list[str] = field(default_factory=list)
    service_uuids_128: list[str] = field(default_factory=list)
    manufacturer_id: Optional[int] = None      # integer Company ID
    manufacturer_data: Optional[bytes] = None
    raw_advdata: Optional[bytes] = None
    # Classification results (populated after classification)
    classes: list = field(default_factory=list)


# Required fields per Contract ML-1
REQUIRED_FIELDS = {"ts", "mac", "sniffer_id"}


def extract_features(line: str) -> Observation:
    """
    Parse a DEEP-1 JSONL line into an Observation object.

    Raises:
        json.JSONDecodeError: Invalid JSON.
        KeyError: Missing required field (ts, mac, sniffer_id).
        ValueError: Invalid field format (e.g., unparseable timestamp).
    """
    data = json.loads(line)

    # Validate required fields
    missing = REQUIRED_FIELDS - set(data.keys())
    if missing:
        raise KeyError(f"Missing required fields: {missing}")

    advdata = data.get("advdata", {})

    # Extract manufacturer info
    mfr = advdata.get("manufacturer", {})
    manufacturer_id = None
    manufacturer_data = None
    if mfr:
        company_id_hex = mfr.get("company_id")
        if company_id_hex:
            manufacturer_id = int(company_id_hex, 16)
        data_hex = mfr.get("data_hex")
        if data_hex:
            manufacturer_data = bytes.fromhex(data_hex)

    # Extract raw advdata
    raw_hex = data.get("raw_advdata_hex")
    raw_advdata = bytes.fromhex(raw_hex) if raw_hex else None

    return Observation(
        ts=datetime.fromisoformat(data["ts"].replace("Z", "+00:00")),
        mac=data["mac"],
        sniffer_id=int(data["sniffer_id"]),
        address_type=data.get("address_type"),
        rssi=data.get("rssi"),
        channel=data.get("channel"),
        pdu_type=data.get("pdu_type"),
        flags=advdata.get("flags"),
        local_name=advdata.get("local_name"),
        tx_power=advdata.get("tx_power"),
        service_uuids_16=advdata.get("service_uuids_16", []),
        service_uuids_128=advdata.get("service_uuids_128", []),
        manufacturer_id=manufacturer_id,
        manufacturer_data=manufacturer_data,
        raw_advdata=raw_advdata,
    )
```

### 4.4 classifier.py — Rule Matching Engine

The classification pipeline runs in this order:

```
Observation
    │
    ├─► 1. Manufacturer ID lookup (OUI-like, via Company ID table)
    │      Vendor: 0x004C → "Apple, Inc."
    │      Vendor: 0x0006 → "Microsoft Corporation"
    │      (Unknown → vendor = None → goes to "unclassified" check at end)
    │
    ├─► 2. Service UUID matcher
    │      Check each service_uuid_16 against known service catalog:
    │        "180F" → battery_service_device
    │        "180D" → heart_rate_device
    │        "FEAA" → eddystone_beacon
    │        "FD6F" → exposure_notification
    │        "FE0F" → (checked in combination with manufacturer)
    │      Check each service_uuid_128 similarly.
    │
    ├─► 3. Combined rules (manufacturer + service)
    │      Apple (0x004C) + FE0F → apple_continuity
    │      Apple (0x004C) + any → apple_device
    │      Microsoft (0x0006) → microsoft_swift_pair
    │      Google (0x00E0) + FEAA → eddystone_beacon
    │
    ├─► 4. Fallback classification
    │      No manufacturer, no service UUIDs → unclassified
    │      Known manufacturer but no service-based class → vendor_only
    │
    └─► 5. Residency score computation (residency.py)
           Per-MAC within this batch:
             observation_count / batch_duration_hours → score (0–1)
```

```python
"""
classifier.py — Rule-based device classification engine.

v1 scope: Manufacturer ID → OUI lookup → Service UUID matching → Device class.

The classification catalog is traceable to Bluetooth SIG registries:
  - Company Identifiers: https://www.bluetooth.com/specifications/assigned-numbers/company-identifiers/
  - GATT Service UUIDs: https://www.bluetooth.com/specifications/assigned-numbers/
"""

from dataclasses import dataclass, field
from typing import Optional


@dataclass
class DeviceClass:
    """A single device class assignment."""
    label: str                          # e.g., "apple_continuity"
    match_type: str                     # "manufacturer", "service_uuid", "combined"
    confidence: str                     # "high", "medium", "low"
    detail: str                         # Human-readable explanation


# ── Bluetooth SIG Company ID → Vendor Name ────────────────────────

MANUFACTURER_TABLE: dict[int, str] = {
    0x004C: "Apple, Inc.",
    0x0006: "Microsoft Corporation",
    0x00E0: "Google LLC",
    0x0075: "Samsung Electronics Co., Ltd.",
    0x0002: "Intel Corporation",
    0x000D: "Texas Instruments, Inc.",
    0x0059: "Nordic Semiconductor ASA",
    0x0157: "Anhui Huami Information Technology Co., Ltd.",  # Xiaomi / Huami
    0x038F: "Tile, Inc.",
    0x0499: "Espressif Inc.",
    0xFFFF: "Internal Use (testing)",
    # Expanded as new devices are observed.
    # Full table: https://www.bluetooth.com/specifications/assigned-numbers/company-identifiers/
}


def lookup_vendor(manufacturer_id: Optional[int]) -> Optional[str]:
    """Look up a Bluetooth SIG Company ID to a human-readable vendor name."""
    if manufacturer_id is None:
        return None
    return MANUFACTURER_TABLE.get(manufacturer_id)


# ── Service UUID → Device Class ───────────────────────────────────

SERVICE_CLASS_RULES: dict[str, list[str]] = {
    "180F": ["battery_service_device"],         # Battery Service
    "180D": ["heart_rate_device"],              # Heart Rate
    "180A": ["device_information_device"],      # Device Information
    "1812": ["hid_device"],                     # Human Interface Device
    "FEAA": ["eddystone_beacon"],               # Google Eddystone
    "FD6F": ["exposure_notification"],          # Google/Apple Exposure Notification
    "FE9F": ["google_fast_pair"],               # Google Fast Pair
    "FEE0": ["microsoft_azure_device_provisioning"],  # Microsoft Azure DPS
    "FE61": ["tile"],                           # Tile finding network
    "FD3D": ["mi_fit"],                         # Xiaomi Mi Fit
    "0003": ["serial_port_device"],             # Serial Port Profile
}


# ── Combined Rules (manufacturer + service) ───────────────────────

COMBINED_RULES: list[dict] = [
    {
        "label": "apple_continuity",
        "manufacturer_ids": [0x004C],
        "service_uuids_16": ["FE0F"],
        "confidence": "high",
        "detail": "Apple Continuity protocol (Handoff, AirDrop, etc.)"
    },
    {
        "label": "apple_device",
        "manufacturer_ids": [0x004C],
        "service_uuids_16": [],  # Matches any Apple device (any/no service UUIDs)
        "confidence": "medium",
        "detail": "Apple device (iBeacon, AirPods, iPhone, etc.)"
    },
    {
        "label": "microsoft_swift_pair",
        "manufacturer_ids": [0x0006],
        "service_uuids_16": [],
        "confidence": "high",
        "detail": "Microsoft Swift Pair (Surface, Windows accessories)"
    },
    {
        "label": "samsung_device",
        "manufacturer_ids": [0x0075],
        "service_uuids_16": [],
        "confidence": "medium",
        "detail": "Samsung Bluetooth device"
    },
    {
        "label": "google_fast_pair_device",
        "manufacturer_ids": [],  # Any manufacturer
        "service_uuids_16": ["FE9F"],
        "confidence": "high",
        "detail": "Google Fast Pair compatible device"
    },
]


def classify_observation(obs) -> list[DeviceClass]:
    """
    Classify a single device observation.

    Args:
        obs: Observation object from features.py.

    Returns:
        List of DeviceClass assignments. May be empty (unclassified).
    """
    classes: list[DeviceClass] = []

    # Step 1: Manufacturer-only vendor identification
    vendor = lookup_vendor(obs.manufacturer_id)
    if vendor and obs.manufacturer_id:
        classes.append(DeviceClass(
            label=f"vendor_{vendor.lower().replace(' ', '_').replace(',', '')}",
            match_type="manufacturer",
            confidence="high",
            detail=f"Vendor identified: {vendor} (Company ID 0x{obs.manufacturer_id:04X})"
        ))

    # Step 2: Service UUID matching
    for uuid16 in obs.service_uuids_16:
        uuid16_upper = uuid16.upper()
        if uuid16_upper in SERVICE_CLASS_RULES:
            for label in SERVICE_CLASS_RULES[uuid16_upper]:
                classes.append(DeviceClass(
                    label=label,
                    match_type="service_uuid",
                    confidence="high",
                    detail=f"Service UUID 0x{uuid16_upper} recognized"
                ))

    for uuid128 in obs.service_uuids_128:
        uuid128_upper = uuid128.upper()
        if uuid128_upper in SERVICE_CLASS_RULES:
            for label in SERVICE_CLASS_RULES[uuid128_upper]:
                classes.append(DeviceClass(
                    label=label,
                    match_type="service_uuid",
                    confidence="high",
                    detail=f"128-bit Service UUID recognized"
                ))

    # Step 3: Combined rules (manufacturer + service intersection)
    for rule in COMBINED_RULES:
        if rule["manufacturer_ids"] and obs.manufacturer_id not in rule["manufacturer_ids"]:
            continue
        if rule["service_uuids_16"]:
            required_uuids = {u.upper() for u in rule["service_uuids_16"]}
            observed_uuids = {u.upper() for u in obs.service_uuids_16}
            if not required_uuids.issubset(observed_uuids):
                continue
        # Rule matched
        classes.append(DeviceClass(
            label=rule["label"],
            match_type="combined",
            confidence=rule["confidence"],
            detail=rule["detail"]
        ))

    # Step 4: Deduplicate class labels (keep highest confidence per label)
    seen: dict[str, DeviceClass] = {}
    for dc in classes:
        if dc.label not in seen or _confidence_order(dc.confidence) > _confidence_order(seen[dc.label].confidence):
            seen[dc.label] = dc
    return list(seen.values())


def _confidence_order(level: str) -> int:
    return {"low": 0, "medium": 1, "high": 2}.get(level, 0)
```

### 4.5 oui_lookup.py — Vendor Lookup Table

The "OUI lookup" for BLE uses the Bluetooth SIG Company Identifier registry rather than IEEE MAC OUI — because the manufacturer ID comes from AD type 0xFF (Manufacturer Specific Data), not from the MAC address OUI.

```python
"""
oui_lookup.py — Bluetooth SIG Company ID → Vendor Name lookup table.

Source: Bluetooth SIG Assigned Numbers
URL: https://www.bluetooth.com/specifications/assigned-numbers/company-identifiers/

This table maps the 16-bit Company Identifier (from BLE AD type 0xFF) to
a human-readable vendor name. It is the BLE equivalent of IEEE MAC OUI lookup.
"""

# See classifier.py MANUFACTURER_TABLE for the canonical table.
# This module provides the reverse lookup and validation utilities.

from .classifier import MANUFACTURER_TABLE, lookup_vendor

__all__ = ["MANUFACTURER_TABLE", "lookup_vendor"]
```

### 4.6 residency.py — Residency Score Computation

The residency score is a **per-MAC heuristic** computed within each C08 batch. It is not the authoritative residency classification (that belongs to C02 `classify_residency()`), but rather a lightweight signal included in the enrichment data:

```python
"""
residency.py — Compute per-MAC residency score from observation frequency.

A residency score quantifies how "resident" a device appears within a
single C08 batch. It ranges from 0.0 (single brief observation) to
1.0 (continuously present throughout the batch window).

This score is supplemental to the authoritative residency classification
performed by C02's classify_residency() function.
"""

from collections import defaultdict
from datetime import datetime, timedelta
from typing import Optional


@dataclass
class MACStats:
    """Aggregated statistics for a MAC within a processing batch."""
    mac: str
    observation_count: int
    first_seen: datetime
    last_seen: datetime
    distinct_channels: set[int]
    rssi_readings: list[int]

    @property
    def window_hours(self) -> float:
        delta = (self.last_seen - self.first_seen).total_seconds()
        return max(delta / 3600.0, 0.0)

    @property
    def residency_score(self) -> float:
        """
        Compute residency score (0.0–1.0).

        Factors:
        - Observations per hour (normalized to 100/hr = 1.0)
        - Window span (longer = higher)
        - Distinct channels (multi-channel = more likely resident)
        """
        if self.observation_count == 0:
            return 0.0

        # Density score: observations per hour, capped at 100/hr
        density = min(self.observation_count / max(self.window_hours, 0.01), 100.0) / 100.0

        # Window score: presence over at least 1 hour
        window = min(self.window_hours / 1.0, 1.0) if self.window_hours > 0 else 0.0

        # Channel diversity: bonus for appearing on multiple channels
        channel_bonus = min(len(self.distinct_channels) / 3.0, 0.2)  # max 0.2 bonus

        score = (density * 0.5) + (window * 0.3) + channel_bonus
        return round(min(score, 1.0), 2)


class ResidencyTracker:
    """Accumulate MAC statistics during batch processing."""

    def __init__(self):
        self._stats: dict[str, MACStats] = {}

    def observe(self, mac: str, ts: datetime, channel: Optional[int],
                rssi: Optional[int]) -> None:
        if mac not in self._stats:
            self._stats[mac] = MACStats(
                mac=mac, observation_count=0,
                first_seen=ts, last_seen=ts,
                distinct_channels=set(), rssi_readings=[]
            )
        stats = self._stats[mac]
        stats.observation_count += 1
        stats.first_seen = min(stats.first_seen, ts)
        stats.last_seen = max(stats.last_seen, ts)
        if channel is not None:
            stats.distinct_channels.add(channel)
        if rssi is not None:
            stats.rssi_readings.append(rssi)

    def get_score(self, mac: str) -> float:
        stats = self._stats.get(mac)
        return stats.residency_score if stats else 0.0

    def get_stats(self, mac: str) -> Optional[MACStats]:
        return self._stats.get(mac)

    @property
    def all_macs(self) -> list[str]:
        return list(self._stats.keys())
```

### 4.7 db.py — Database Writer

```python
"""
db.py — PostgreSQL writer for C08 ML Enrichment.

Uses psycopg 3.1 async [3] with parameterized queries via the tianer_writer role.
Implements batch INSERT into device_enrichment and UPSERT into device_summary.
"""

import os
import logging
from typing import Optional
import psycopg
from psycopg import sql

from .features import Observation
from .classifier import lookup_vendor
from .residency import ResidencyTracker

logger = logging.getLogger(__name__)


class DBWriter:
    """Batch write observations and enrichment data to PostgreSQL.

    Uses psycopg 3.1 async [3] to perform non-blocking database I/O.
    All queries use parameterized placeholders (%s), eliminating SQL injection.
    """

    def __init__(self, batch_size: int = 100):
        self.batch_size = batch_size
        self.conn: Optional[psycopg.AsyncConnection] = None
        self._batch: list[Observation] = []
        self._malformed_count: int = 0
        self._tracker = ResidencyTracker()

    async def connect(self) -> None:
        """Connect to PostgreSQL using tianer_writer role."""
        self.conn = await psycopg.AsyncConnection.connect(
            host=os.getenv("TIANER_DB_HOST", "tianer-postgres"),
            port=int(os.getenv("TIANER_DB_PORT", "5432")),
            dbname=os.getenv("TIANER_DB_NAME", "tianer"),
            user=os.getenv("TIANER_DB_USER", "tianer_writer"),
            password=os.getenv("TIANER_DB_PASSWORD", ""),
        )
        logger.info("Connected to PostgreSQL as tianer_writer")

    async def write_observation(self, obs: Observation) -> None:
        """Add an observation to the batch. Flush when batch is full."""
        self._batch.append(obs)
        # Track residency stats
        self._tracker.observe(obs.mac, obs.ts, obs.channel, obs.rssi)
        if len(self._batch) >= self.batch_size:
            await self._flush_observations()

    async def _flush_observations(self) -> None:
        """Insert accumulated observations into device_enrichment."""
        if not self._batch:
            return

        # Decode MAC from colon-hex to bytes
        rows = []
        for obs in self._batch:
            mac_hex = obs.mac.replace(":", "")
            mac_bytes = bytes.fromhex(mac_hex) if len(mac_hex) == 12 else None
            if mac_bytes is None:
                logger.warning("Invalid MAC address: %s", obs.mac)
                continue

            rows.append((
                mac_bytes,
                obs.ts,
                obs.sniffer_id,
                obs.local_name,
                obs.service_uuids_16,
                obs.service_uuids_128,
                obs.manufacturer_id,
                obs.manufacturer_data,
                obs.tx_power,
                obs.flags,
                obs.raw_advdata,
            ))

        async with self.conn.cursor() as cur:
            await cur.executemany("""
                INSERT INTO bluetooth.device_enrichment
                    (mac_address, observed_ts, sniffer_id, local_name,
                     service_uuids_16, service_uuids_128,
                     manufacturer_id, manufacturer_data,
                     tx_power, flags, raw_advdata)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                ON CONFLICT (mac_address, observed_ts, sniffer_id)
                DO NOTHING
            """, rows)

        logger.debug("Flushed %d observations to device_enrichment", len(rows))
        self._batch.clear()

    async def update_device_summaries(self) -> None:
        """Update device_summary.enrichment_data for all observed MACs."""
        await self._flush_observations()  # Flush any remaining

        async with self.conn.cursor() as cur:
            for mac in self._tracker.all_macs:
                stats = self._tracker.get_stats(mac)
                if not stats:
                    continue

                # Collect unique classes across all observations for this MAC
                # (already accumulated during processing)

                # Build enrichment_data JSONB
                enrichment = {
                    "vendor": lookup_vendor(
                        # Use last observed manufacturer_id for this MAC
                        next((o.manufacturer_id for o in self._batch
                              if o.mac == mac and o.manufacturer_id), None)
                    ),
                    "device_name": next((o.local_name for o in self._batch
                                         if o.mac == mac and o.local_name), None),
                    "last_enriched": stats.last_seen.isoformat(),
                    "residency_score": stats.residency_score,
                    "observation_count": stats.observation_count,
                    "observation_window_hours": round(stats.window_hours, 2),
                    "unclassified": False,  # Updated below if needed
                }

                # Merge with existing enrichment_data (union of classes)
                mac_hex = mac.replace(":", "")
                mac_bytes = bytes.fromhex(mac_hex)

                await cur.execute("""
                    INSERT INTO bluetooth.device_summary
                        (mac_address, first_seen, last_seen, enrichment_data)
                    VALUES (%s, %s, %s, %s)
                    ON CONFLICT (mac_address) DO UPDATE
                    SET enrichment_data =
                        bluetooth.device_summary.enrichment_data
                        || EXCLUDED.enrichment_data,
                        last_seen = GREATEST(
                            bluetooth.device_summary.last_seen,
                            EXCLUDED.last_seen
                        )
                """, (mac_bytes, stats.first_seen, stats.last_seen,
                      psycopg.types.json.Json(enrichment)))

            logger.info("Updated device_summary for %d MACs",
                        len(self._tracker.all_macs))

    def increment_malformed(self) -> None:
        """Increment the malformed line counter (for metrics)."""
        self._malformed_count += 1

    async def flush(self) -> None:
        """Flush all pending writes."""
        await self._flush_observations()

    async def close(self) -> None:
        """Close the database connection."""
        await self._flush_observations()
        if self.conn:
            await self.conn.close()
            logger.info("Disconnected from PostgreSQL")
```

The `||` operator in `ON CONFLICT DO UPDATE` performs a shallow merge of two JSONB objects [4], ensuring that multiple C08 runs accumulate class labels (a set union of top-level keys) rather than overwriting prior classifications.

### 4.8 state.py — Cursor State Tracking

```python
"""
state.py — Cursor state tracking for idempotent re-processing.

Tracks which JSONL files have been processed and up to what line.
Enables C08 to resume after a crash without re-processing entire files.
"""

import json
from dataclasses import dataclass, field
from pathlib import Path
from typing import Optional


@dataclass
class Cursor:
    """Position within a JSONL file."""
    line_offset: int = 0          # Last processed line number
    byte_offset: int = 0          # File byte position
    total_lines_in_file: int = 0  # Total lines in file when cursor was saved


class StateTracker:
    """Persist processing state to a JSON file on V05."""

    def __init__(self, state_path: Path):
        self._path = state_path
        self._data: dict = {}
        self._load()

    def _load(self) -> None:
        if self._path.exists():
            with open(self._path) as f:
                self._data = json.load(f)

    def _save(self) -> None:
        self._path.parent.mkdir(parents=True, exist_ok=True)
        with open(self._path, "w") as f:
            json.dump(self._data, f, indent=2, default=str)

    def get_cursor(self, filepath: str) -> Cursor:
        """Get the cursor for a file, or a fresh cursor if never processed."""
        entry = self._data.get(filepath, {})
        return Cursor(
            line_offset=entry.get("line_offset", 0),
            byte_offset=entry.get("byte_offset", 0),
            total_lines_in_file=entry.get("total_lines", 0),
        )

    def update_cursor(self, filepath: str, cursor: Cursor) -> None:
        """Save cursor position after processing."""
        self._data[filepath] = {
            "line_offset": cursor.line_offset,
            "byte_offset": cursor.byte_offset,
            "total_lines": cursor.total_lines_in_file,
            "last_processed": str(Path(filepath).stat().st_mtime),
        }
        self._save()
```

---

## 5. Inter-Component Contracts

### 5.1 Contract ML-1: C07 JSONL → C08 ML Enrichment Input

| Property | Value |
|----------|-------|
| **Contract ID** | `ML-1` |
| **From** | C07 Deep Parser |
| **To** | C08 ML Enrichment |
| **Format** | JSON Lines — one JSON object per line, UTF-8 encoded |
| **Transport** | Filesystem: V05 `/var/lib/tianer/data/` |
| **Access** | C07 writes (V05 `:rw`), C08 reads (V05 `:ro`) |

#### Required Fields (must be present on every line, or C08 skips the line)

```
ts          (string, ISO 8601)    — Observation timestamp
mac         (string, colon-hex)  — Bluetooth device address
sniffer_id  (integer)           — Source sniffer ID
advdata     (object)            — Parsed advertising data (may be empty: {})
```

#### Optional Fields (enrich classification when present)

```
advdata.service_uuids_16              — string[]: 16-bit service UUIDs
advdata.service_uuids_128             — string[]: 128-bit service UUIDs
advdata.manufacturer.company_id       — string (hex): Bluetooth SIG Company ID
advdata.manufacturer.data_hex         — string (hex): Manufacturer data payload
advdata.local_name                    — string: Device local name
advdata.flags                         — integer: BLE Flags
advdata.tx_power                      — integer: TX Power in dBm
rssi                                  — integer: RSSI in dBm
channel                               — integer: Advertising channel
pdu_type                              — string: PDU type name
address_type                          — string: "public" or "random"
raw_advdata_hex                       — string (hex): Raw advertising data
```

#### Example

```jsonl
{"ts":"2026-06-06T14:00:00.123456Z","sniffer_id":1,"mac":"aa:bb:cc:dd:ee:ff","rssi":-67,"channel":37,"pdu_type":"ADV_IND","advdata":{"flags":6,"local_name":"MyDevice","tx_power":-4,"service_uuids_16":["180D","180F"],"manufacturer":{"company_id":"004C","data_hex":"100601234567"},"service_uuids_128":[],"service_data":[]},"raw_advdata_hex":"02011a0303aafe17..."}
```

Formatting note: JSONL lines are typically compact (no extra whitespace) when C07 writes them. C08's parser accepts both compact and pretty-printed lines.

### 5.2 Contract DB-WRITE: C08 → PostgreSQL

C08 uses the shared `DB-WRITE` contract (C02 §5.2) to connect to PostgreSQL as the `tianer_writer` role, using parameterized INSERT queries via psycopg 3.1. C08 does not execute DDL, SELECT, UPDATE (except for the JSONB merge in `device_summary`), or DELETE statements.

---

## 6. Failure Modes & Recovery

### 6.1 Failure Catalog

| # | Failure Mode | Detection | Impact | Recovery |
|---|-------------|-----------|--------|----------|
| **F-C08-1** | **JSONL parse error (malformed line)** | `json.JSONDecodeError` in `extract_features()`. Logged at WARNING level. | That observation is skipped. No data inserted for that line. Other lines in the file continue processing. | No recovery needed — the observation is lost for enrichment but the underlying PCAP is the source of truth. C07 can re-parse the PCAP and emit a corrected JSONL file. C08 processes the corrected file on the next run. |
| **F-C08-2** | **JSONL missing required field (ts, mac, sniffer_id)** | `KeyError` in `extract_features()`. Logged at WARNING level. | Observation skipped. Counter `tianer_ml_malformed_lines_total` incremented. | Same as F-C08-1. C07 re-parse from PCAP. |
| **F-C08-3** | **DB connection failure** | `psycopg.OperationalError` during `connect()` or `write_observation()`. | C08 cannot write to PostgreSQL. Entire run fails. | Exponential backoff reconnect (1s, 2s, 4s, 8s, 16s, cap 30s). If persistent: container exits non-zero. Systemd timer retries on next tick. State file (`state.json`) records the last successful cursor position — on next run, C08 resumes from where it left off. No duplicate INSERTs (ON CONFLICT DO NOTHING handles re-processing). |
| **F-C08-4** | **DB write failure mid-batch** | `psycopg.Error` during `executemany()`. | The current in-memory batch (≤100 rows) is lost. The state file has not been updated, so the cursor points to the last position before this batch. | Container restarts. On next run, C08 resumes from the cursor position, re-processes the affected lines, and re-inserts. ON CONFLICT DO NOTHING prevents duplicates. |
| **F-C08-5** | **Container OOM kill** | Podman/sytemd detects non-zero exit or signal. Container exits with code 137 (SIGKILL). | In-memory batch + residency tracker lost. State file may not reflect latest progress. | Same as F-C08-4. Resume from last saved cursor. Re-processing is safe. |
| **F-C08-6** | **Disk full on V05 (cannot write state.json)** | `OSError` during `StateTracker._save()`. | State is not persisted. Next run reprocesses files from the beginning. | Alert on `tianer_disk_usage_percent > 80%` (C01/C13). C04 emergency purge of old uncompressed PCAP files (different volume). C08 re-processing is safe (idempotent inserts). |
| **F-C08-7** | **Unknown manufacturer ID** | `lookup_vendor()` returns `None`. | Device is classified as `"unclassified"`. The observation is still stored in `device_enrichment` for future re-classification. `enrichment_data.unclassified = true`. | No immediate recovery action. C08 logs the unknown manufacturer ID at INFO level. Operator can add the ID to `MANUFACTURER_TABLE` and re-run C08. Future C08 runs on new JSONL data will classify additional observations from the same device. |
| **F-C08-8** | **C07 JSONL file not yet available** | `data_dir.glob("**/*.jsonl")` returns empty list. | C08 has no work to do. | Exit 0 cleanly. This is normal operation — the timer fires speculatively. |
| **F-C08-9** | **C07 JSONL file truncated (C07 still writing)** | `json.JSONDecodeError` on last line (incomplete JSON). | Last line skipped. Logged at WARNING. | C08's state cursor records the last successfully processed line. Next run re-reads from that position and processes the now-complete file. |

### 6.2 Idempotency Guarantees

C08 is designed to be **safe to re-run** on the same input:

| Operation | Idempotency Mechanism |
|-----------|----------------------|
| INSERT into `device_enrichment` | `ON CONFLICT (mac_address, observed_ts, sniffer_id) DO NOTHING` — duplicate rows are silently ignored. |
| UPDATE `device_summary.enrichment_data` | `enrichment_data \|\| EXCLUDED.enrichment_data` — new data is merged (set union with existing), not overwritten. |
| State tracking | Cursor-based state file enables resume from last successful position. Re-processing from an earlier position is safe due to idempotent DB operations. |

### 6.3 Recovery Service Dependency

| Recovery Agent | Failure It Handles | Is It Available After Reboot? |
|---------------|-------------------|-------------------------------|
| Systemd timer (`tianer-ml-classify.timer`) | Container crash, F-C08-3, F-C08-4, F-C08-5 | Yes — timer fires automatically after reboot |
| Podman auto-restart (`Restart=on-failure`) | Container crash mid-run | Yes — but C08 is a oneshot; it relies on the timer for retries, not container restart |
| State file (V05) | Crash recovery, resume from interrupt | Yes — persisted on V05 (bind-mount to host) |
| C07 Deep Parser | JSONL parse errors in source data | Manual — re-run C07 on the PCAP source to produce corrected JSONL |
| C02 PostgreSQL | Connection failure | Automatic — exponential backoff reconnect |

### 6.4 Data Loss Quantification (PF-10)

- **JSONL parse errors:** One observation line lost per malformed line. The underlying PCAP is the source of truth — C07 can re-parse and re-emit. Loss is bounded by the number of malformed lines in a single JSONL file (expected: 0 under normal operation).
- **DB write failure mid-batch:** ≤100 rows (batch_size) lost from the in-memory batch. Recoverable on next run by re-processing. Zero permanent loss.
- **Container OOM kill:** Same as DB write failure. Recoverable on next run.
- **Unclassified observations:** No classifications assigned, but the raw observation is preserved in `device_enrichment`. Future classifier updates can retroactively classify.

---

## 7. Observability

### 7.1 Metrics

C08 exposes metrics at the end of each run by writing to a structured log line that C13 can scrape:

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `tianer_ml_lines_processed_total` | counter | `sniffer_id` | Total JSONL lines successfully processed |
| `tianer_ml_malformed_lines_total` | counter | — | Lines skipped due to parse errors or missing fields |
| `tianer_ml_observations_inserted_total` | counter | `sniffer_id` | Rows inserted into `device_enrichment` |
| `tianer_ml_devices_classified_total` | counter | — | Unique MACs classified in this run |
| `tianer_ml_devices_unclassified_total` | counter | — | MACs with no matching rules (unclassified) |
| `tianer_ml_class_distribution` | gauge | `class_label` | Count of devices assigned each class label |
| `tianer_ml_run_duration_seconds` | gauge | — | Wall-clock duration of the last run |
| `tianer_ml_db_write_errors_total` | counter | — | Number of database write failures |
| `tianer_ml_run_status` | gauge (0/1) | — | 1 if last run completed successfully, 0 if failed |

**Metric exposition:** At the end of each run, C08 writes a structured log line:

```
TIANER | {"ts":"2026-06-09T14:30:00Z","level":"INFO","component":"ml-enrichment","msg":"run_metrics","lines_processed":15234,"malformed":2,"observations_inserted":15232,"devices_classified":87,"devices_unclassified":12,"class_distribution":{"apple_continuity":23,"eddystone_beacon":15,"battery_service_device":31,"unclassified":12},"run_duration_seconds":4.2,"db_write_errors":0,"run_status":"success"}
```

### 7.2 Structured Logging

All C08 log output goes to `stderr` (captured by Podman → journald). Format per C13 structured logging specification:

```
TIANER | {"ts":"...","level":"INFO|WARN|ERROR","component":"ml-enrichment","msg":"...","detail":{...}}
```

| Log Level | When Used |
|-----------|----------|
| `INFO` | Startup, file processing progress, completion summary, normal exit |
| `WARNING` | JSONL parse error (line skipped), unknown manufacturer ID, DB connection retry |
| `ERROR` | Fatal DB connection failure, state file write failure, unhandled exception |

### 7.3 Health Check

C08 is a batch processor, not a service — it has no health endpoint. Its health is inferred from:

1. **Timer execution:** systemd timer fires on schedule; failures are visible in `systemctl --user status tianer-ml-classify.service`.
2. **Exit code:** 0 = success, non-zero = failure. systemd records `ExecMainStatus`.
3. **Metric `tianer_ml_run_status`:** C13 monitors this metric. If run_status = 0 for 3 consecutive timer ticks, alert fires.
4. **Staleness:** If `device_summary.enrichment_data->>'last_enriched'` is older than 6× timer interval (30 minutes), alert fires — C08 is not running.

### 7.4 Alert Summary

| Alert Name | Trigger | Severity | Action |
|------------|---------|----------|--------|
| `TianerMLRunFailed` | `tianer_ml_run_status == 0` for 3+ consecutive runs | **Warning** | Check Podman logs: `podman logs tianer-ml-classify`. Check DB connectivity. Check V05 disk space. |
| `TianerMLStale` | No `run_status == 1` in > 30 minutes | **Warning** | Verify timer is enabled: `systemctl --user is-enabled tianer-ml-classify.timer`. |
| `TianerMLHighMalformedRate` | `tianer_ml_malformed_lines_total / tianer_ml_lines_processed_total > 0.01` (1%) | **Warning** | C07 JSONL output may have format errors. Investigate C07 output. |

---

## 8. Security Considerations

### 8.1 Database Access — Least Privilege

C08 connects to PostgreSQL as `tianer_writer` — a role that can **INSERT** but cannot SELECT, UPDATE, DELETE, or DDL. This enforces Q9 least-privilege:

| Operation | Allowed? | Justification |
|-----------|----------|---------------|
| INSERT into `device_enrichment` | ✅ | Core function — writing enrichment observations. |
| Parameterized UPSERT on `device_summary` | ⚠️ (requires INSERT privilege) | The `ON CONFLICT DO UPDATE` syntax requires INSERT privilege on the table. C02 migration `0005_device_enrichment.sql` must grant INSERT on `device_summary` to `tianer_writer` for the enrichment_data column specifically. This is the narrowest possible privilege expansion. |
| SELECT from any table | ❌ | C08 does not need to read existing data — it writes only. |
| DDL | ❌ | No table creation or alteration. |
| DELETE | ❌ | No data removal. |

**Privilege note for C02 implementer:** Migration `0005_device_enrichment.sql` must include:
```sql
GRANT INSERT ON device_summary TO tianer_writer;
```
This is required because C08 uses `INSERT ... ON CONFLICT DO UPDATE` to upsert `enrichment_data`, which requires INSERT permission on the target table.

### 8.2 Input Validation

C08 validates every JSONL line before processing:

| Check | Failure Response |
|-------|-----------------|
| Valid JSON | `json.JSONDecodeError` → skip line, log WARNING |
| `ts` field is parseable ISO 8601 | `ValueError` → skip line, log WARNING |
| `mac` field is 17-character colon-hex string | `ValueError` → skip line, log WARNING |
| `sniffer_id` is a positive integer | `ValueError` → skip line, log WARNING |
| `manufacturer_id` (if present) is a valid 16-bit hex | `ValueError` → set to None, continue |
| `service_uuids_16` (if present) is an array of 4-character hex strings | Non-conforming entries are silently dropped from the array |

**No raw SQL** — all DB queries use parameterized placeholders (`%s` in psycopg). User-controlled data (MAC, local_name, service UUIDs) is never concatenated into SQL strings.

### 8.3 Filesystem Access

C08 mounts exactly two host paths:

| Volume | Path | Mode | Purpose |
|--------|------|------|---------|
| V05 | `/var/lib/tianer/data/` | `:ro` | Read C07 JSONL output |
| V01 | `/etc/tianer/` | `:ro` | Read DB connection config |

C08 does **not** have write access to V05 (JSONL input is read-only). The state file is written inside the container's ephemeral `/tmp` and is not persisted across container runs — state tracking relies on DB-based cursor position (the `last_enriched` timestamp in `device_summary.enrichment_data` and the `ON CONFLICT DO NOTHING` dedup).

**Correction to state file design (§4.8):** In containerized deployment, state is tracked by observing which JSONL files are newer than the last `last_enriched` timestamp across all MACs in the batch. The state.json is an optimization for bare-metal debugging; in the container, the cursor is derived from file modification timestamps. This avoids needing `:rw` on V05.

### 8.4 Supply Chain Integrity (Q10)

The C08 container image is built from `python:3.13-slim` (digest-pinned). Python dependencies are installed with `uv --require-hashes` to verify package integrity:

```toml
# pyproject.toml (dependency section)
[tool.uv]
require-hashes = true

[project]
dependencies = [
    "psycopg[binary]>=3.1,<4.0",
]
```

The `uv.lock` file contains SHA256 hashes for every transitive dependency. CI verifies the lock file against installed packages before building the container image.

### 8.5 Data Sensitivity

C08 processes BLE advertising data including:
- **MAC addresses** — persistent identifiers for public-address devices. Stored as `BYTEA` in the database.
- **Local names** — may contain user-assigned device names (e.g., "John's iPhone"). Stored as TEXT.
- **Manufacturer data** — opaque binary payloads. Content depends on manufacturer; may contain sensor readings or device identifiers.

All data stays within the container network (`tianer-net`, no external route) and within the Podman volumes on the host. No data is transmitted off-device.

---

## 9. Configuration

### 9.1 Environment Variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `TIANER_DB_HOST` | `tianer-postgres` | PostgreSQL hostname (container name on `tianer-net`) |
| `TIANER_DB_PORT` | `5432` | PostgreSQL port |
| `TIANER_DB_NAME` | `tianer` | Database name |
| `TIANER_DB_USER` | `tianer_writer` | Database role |
| `TIANER_DB_PASSWORD` | (from secrets) | Role password, injected via `EnvironmentFile` |
| `TIANER_ML_DATA_DIR` | `/var/lib/tianer/data` | C07 JSONL output directory |
| `TIANER_ML_BATCH_SIZE` | `100` | Rows per DB insert batch |
| `TIANER_ML_LOG_LEVEL` | `INFO` | Logging level (DEBUG, INFO, WARNING, ERROR) |

### 9.2 Container EnvironmentFile

The Quadlet `.container` file for C08 uses `EnvironmentFile` to inject the DB password securely:

```ini
[Container]
EnvironmentFile=/etc/tianer/secrets/db_password.env
```

The `db_password.env` file contains:
```
TIANER_DB_PASSWORD=<32-byte-base64-random-string>
```

### 9.3 Classification Rule Tuning

The manufacturer table, service UUID rules, and combined rules in `classifier.py` are the **configuration surface** for classification behaviour. To add a new device class, an operator edits `classifier.py` and rebuilds the container image. In v1, there is no external rules file — this keeps the operational surface minimal while the rule set is small. Post-MVP, rules may be externalized to a YAML or JSON file on V01 (`:ro` mount).

### 9.4 Residency Score Tuning

The residency score weights in `residency.py` are tunable:

```python
score = (density * DENSITY_WEIGHT) + (window * WINDOW_WEIGHT) + channel_bonus
```

Defaults: `DENSITY_WEIGHT = 0.5`, `WINDOW_WEIGHT = 0.3`, max channel bonus 0.2. These can be adjusted if real-world data shows skew toward one factor.

---

## 10. Test Plan

### 10.1 Unit Tests — pytest

| Test File | What It Tests | Key Assertions |
|-----------|--------------|----------------|
| `test_extract_features.py` | `features.extract_features()` on valid and malformed JSONL lines | Valid line → correct `Observation` object. Missing `ts` → `KeyError`. Invalid JSON → `json.JSONDecodeError`. Empty `advdata` → `manufacturer_id=None`, empty service arrays. Manufacturer hex `"004C"` → `manufacturer_id=76`. |
| `test_classifier.py` | `classifier.classify_observation()` — rule matching | Apple Continuity (mfr=0x004C + FE0F) → `["vendor_apple_inc", "apple_continuity", "apple_device"]`. Eddystone (FEAA) → `["eddystone_beacon"]`. Battery Service (180F) → `["battery_service_device"]`. Unknown manufacturer → no manufacturer-based class; service classes still matched. No rules match → empty list `[]`. |
| `test_oui_lookup.py` | `oui_lookup.lookup_vendor()` — Company ID → vendor name | `0x004C` → `"Apple, Inc."`. `0x0006` → `"Microsoft Corporation"`. `0x0000` → `None` (not in table). `None` → `None`. |
| `test_residency.py` | `residency.ResidencyTracker` — score computation | Single observation, 1 second window → score ~0.01. 100 observations over 1 hour → score ~0.5. Multi-channel bonus adds up to 0.2. Empty tracker → 0.0. |
| `test_db.py` | `db.DBWriter` — batch insert logic | Batch of 100 → flush triggers at 100. Batch of 50 + flush() → all 50 written. Duplicate (mac, ts, sniffer_id) → ON CONFLICT DO NOTHING skips. |
| `test_state.py` | `state.StateTracker` — cursor persistence | Fresh file → cursor at (0,0). Update cursor → saved. Re-load → restored. Non-existent state file → empty data. |
| `test_runner.py` | `runner.process_jsonl_file()` — end-to-end file processing | Valid fixture file → all lines processed, state updated. Empty file → 0 lines. Malformed lines → skipped, count incremented, rest processed. |

### 10.2 Fixture Files

| Fixture | Content | Purpose |
|---------|---------|---------|
| `tests/fixtures/sample_apple.jsonl` | 100 lines of Apple Continuity advertising (mfr=0x004C, service=FE0F) | Classifier golden test |
| `tests/fixtures/sample_eddystone.jsonl` | 50 lines of Eddystone beacon advertising (service=FEAA) | Service UUID classification |
| `tests/fixtures/sample_unclassified.jsonl` | 50 lines with no manufacturer and no known service UUIDs | Unclassified path |
| `tests/fixtures/sample_malformed.jsonl` | 10 valid lines + 3 malformed lines (missing ts, invalid JSON, missing mac) | Error handling |
| `tests/fixtures/sample_mixed.jsonl` | 200 lines with mixed device types (Apple, Eddystone, Battery Service, unclassified) | Integration test |

### 10.3 Integration Tests

| Test | Description | Acceptance Criteria |
|------|-------------|---------------------|
| `test_ml_e2e.py` | Full pipeline: read fixture JSONL → classify → insert into test DB → verify | All 200 lines from `sample_mixed.jsonl` processed. `device_enrichment` has 200 rows. `device_summary.enrichment_data` for Apple MAC contains `["vendor_apple_inc", "apple_continuity", "apple_device"]`. |
| `test_ml_idempotent.py` | Run C08 twice on the same fixture file | Second run produces no new `device_enrichment` rows (ON CONFLICT DO NOTHING). `device_summary.enrichment_data` classes unchanged. |
| `test_ml_db_reconnect.py` | Kill PostgreSQL mid-run, verify recovery | C08 detects connection loss, retries with backoff, reconnects, resumes from state. No duplicate rows. |
| `test_ml_empty_dir.py` | Run with no JSONL files in data directory | Exit 0, no errors, 0 lines processed. |
| `test_ml_resume.py` | Kill C08 mid-file, re-run | State file records partial progress. Second run resumes from cursor. Total rows inserted = file line count (minus malformed). No duplicates. |

### 10.4 Performance Tests

| Test | Target | Measurement |
|------|--------|-------------|
| JSONL parsing throughput | ≥ 10,000 lines/second on CM5 | Wall-clock time for 100K-line fixture |
| DB insert throughput | ≥ 500 rows/second (batch size 100) | psycopg batch insert time |
| Memory usage | < 200 MB for 500K-line file | RSS measurement during processing |
| Startup time | < 3 seconds (container start to first line processed) | Wall-clock from container start |

### 10.5 Acceptance Criteria

1. **Classification correctness:** Known fixture `sample_mixed.jsonl` produces expected class labels for Apple Continuity, Eddystone, and Battery Service devices.
2. **Unclassified handling:** Devices with no known manufacturer and no known service UUIDs are stored with `enrichment_data.unclassified = true`, and no fake labels are generated.
3. **Error resilience:** Malformed JSONL lines are skipped and counted. Processing continues on subsequent lines.
4. **Idempotency:** Re-running C08 on the same input produces no duplicate rows in `device_enrichment` and no duplicate class labels in `device_summary.enrichment_data`.
5. **DB connection resilience:** C08 recovers from transient DB connection failures via exponential backoff.
6. **Resume from interrupt:** Killing C08 mid-file and re-running processes only the remaining lines (no re-processing of already-inserted rows needed, but idempotent inserts make re-processing safe).
7. **Performance:** 10,000 lines/sec parse throughput, 500 rows/sec DB insert throughput, < 200 MB memory for 500K lines.

### 10.6 CI Integration

C08 tests are run as part of `ci/test-all.sh` step 3 (Python unit tests):

```bash
uv run pytest modules/bluetooth/ml-enrichment/tests/ -v --tb=short
```

Integration tests require a test PostgreSQL database (provided by `ci/docker-compose.yml`). They are run in step 6 (integration tests).

---

## 11. Deployment Notes

### 11.1 Container Image

The C08 container image is built from `python:3.13-slim` using a multi-stage Dockerfile:

```dockerfile
# Stage 1: Build (install dependencies)
FROM python:3.13-slim AS builder
WORKDIR /build
COPY modules/bluetooth/ml-enrichment/pyproject.toml modules/bluetooth/ml-enrichment/uv.lock ./
RUN pip install uv && uv sync --frozen --no-dev

# Stage 2: Runtime (minimal)
FROM python:3.13-slim
COPY --from=builder /build/.venv /app/.venv
COPY modules/bluetooth/ml-enrichment/src/tianer_ml/ /app/tianer_ml/
ENV PATH="/app/.venv/bin:$PATH"
ENTRYPOINT ["python", "-m", "tianer_ml.runner"]
```

**Image properties:**
- Base: `python:3.13-slim@sha256:...` (digest-pinned per Q10)
- Dependencies: only `psycopg[binary]` (for PostgreSQL connection)
- Target size: < 200 MB uncompressed
- Target boot: < 3 seconds to ready on Raspberry Pi CM5
- No shell, no package manager, no dev headers in runtime stage

### 11.2 Quadlet Unit Files

#### Timer: `deploy/containers/tianer-ml-classify.timer`

```ini
[Unit]
Description=Tian'er ML Enrichment Timer
After=tianer-platform.pod

[Timer]
OnCalendar=*-*-* *:00/5:00
AccuracySec=30s
Persistent=true

[Install]
WantedBy=timers.target
```

Triggers every 5 minutes on the minute boundary. `Persistent=true` ensures missed runs (e.g., due to host reboot) fire immediately on restart.

#### Container: `deploy/containers/tianer-ml-classify.container`

```ini
[Unit]
Description=Tian'er ML Enrichment — Rule-Based Device Classification
After=tianer-platform.pod
Requires=tianer-platform.pod

[Container]
Image=localhost/tianer-ml-classify:latest
ContainerName=tianer-ml-classify
Pod=tianer-platform.pod
Volume=/var/lib/tianer/data:/var/lib/tianer/data:ro
Volume=/etc/tianer:/etc/tianer:ro
EnvironmentFile=/etc/tianer/secrets/db_password.env
Environment=TIANER_DB_HOST=tianer-postgres
Environment=TIANER_DB_PORT=5432
Environment=TIANER_DB_NAME=tianer
Environment=TIANER_DB_USER=tianer_writer
Environment=TIANER_ML_DATA_DIR=/var/lib/tianer/data
Environment=TIANER_ML_BATCH_SIZE=100
Environment=TIANER_ML_LOG_LEVEL=INFO

# Security hardening
NoNewPrivileges=true
DropCapability=ALL
ReadOnly=true

# Resource limits
MemoryMax=256M
CPUQuota=100%

[Service]
Type=oneshot
RemainAfterExit=no
```

**Key properties:**
- `Type=oneshot` — C08 is a batch processor, not a long-running service.
- `ReadOnly=true` — Container filesystem is read-only. V05 is already `:ro`.
- `Volume` mounts V05 (JSONL input, read-only) and V01 (config, read-only).
- `EnvironmentFile` injects DB password securely.
- `MemoryMax=256M` — conservative memory cap for CM5.
- `Pod=tianer-platform.pod` — shares the platform pod network namespace (access to `tianer-net` bridge for DB connectivity).

### 11.3 Build and Install

```bash
# Build container image
podman build -t localhost/tianer-ml-classify:latest \
    -f deploy/containers/Containerfile.ml-classify .

# Install Quadlet units
install -m 0644 deploy/containers/tianer-ml-classify.container \
    /etc/containers/systemd/
install -m 0644 deploy/containers/tianer-ml-classify.timer \
    /etc/containers/systemd/
systemctl --user daemon-reload

# Enable the timer (starts on next tick)
systemctl --user enable tianer-ml-classify.timer

# Manual invocation for testing
systemctl --user start tianer-ml-classify.service
```

### 11.4 Makefile Integration

```makefile
.PHONY: build-ml-classify
build-ml-classify:
    podman build -t localhost/tianer-ml-classify:latest \
        -f deploy/containers/Containerfile.ml-classify .

.PHONY: test-ml-classify
test-ml-classify:
    uv run pytest modules/bluetooth/ml-enrichment/tests/ -v --tb=short
```

### 11.5 Python Dependency Management

```bash
# During development: add dependencies
uv add psycopg psycopg[binary]
uv lock  # Generates uv.lock with SHA256 hashes

# In CI and container build: install from lock file
uv sync --frozen --no-dev
```

The `uv.lock` file is committed to the repository and ensures reproducible builds with verified package hashes (Q10 supply chain integrity).

### 11.6 Docker Compose (CI)

The CI pipeline (`ci/docker-compose.yml`) includes:

```yaml
services:
  postgres:
    image: postgres:17-bookworm
    environment:
      POSTGRES_DB: tianer
      POSTGRES_USER: tianer
      POSTGRES_PASSWORD: test_password
    ports:
      - "5432:5432"

  ml-test:
    build:
      context: .
      dockerfile: deploy/containers/Containerfile.ml-classify
    environment:
      TIANER_DB_HOST: postgres
      TIANER_DB_PORT: 5432
      TIANER_DB_NAME: tianer
      TIANER_DB_USER: tianer
      TIANER_DB_PASSWORD: test_password
    volumes:
      - ./tests/fixtures/ml:/var/lib/tianer/data:ro
    depends_on:
      postgres:
        condition: service_healthy
```

### 11.7 Deployment Checklist

Before considering C08 deployed, verify:

- [ ] Container image builds successfully (`podman build ...`)
- [ ] `uv.lock` is present and contains hashes for all dependencies
- [ ] Quadlet `.container` and `.timer` files installed at `/etc/containers/systemd/`
- [ ] Timer enabled: `systemctl --user is-enabled tianer-ml-classify.timer`
- [ ] `tianer_writer` role has INSERT privilege on both `device_enrichment` and `device_summary` tables
- [ ] V05 mounted read-only in container (`:ro`)
- [ ] Manual invocation succeeds: `systemctl --user start tianer-ml-classify.service` exits 0
- [ ] C07 has written at least one JSONL file to V05 before first C08 run
- [ ] `device_summary.enrichment_data` contains classification data after a successful run
- [ ] Alerts configured in C13 for `TianerMLRunFailed` and `TianerMLStale`

---

## Appendix A: Classification Rule Catalog

### A.1 Manufacturer-Based Rules

| Company ID (hex) | Vendor | Device Class Label | Confidence |
|-----------------|--------|-------------------|------------|
| `0x004C` | Apple, Inc. | `vendor_apple_inc` | high |
| `0x004C` + `FE0F` | Apple, Inc. | `apple_continuity` | high |
| `0x004C` (any service) | Apple, Inc. | `apple_device` | medium |
| `0x0006` | Microsoft Corporation | `vendor_microsoft_corporation` | high |
| `0x0006` (any) | Microsoft Corporation | `microsoft_swift_pair` | high |
| `0x00E0` | Google LLC | `vendor_google_llc` | high |
| `0x0075` | Samsung Electronics Co., Ltd. | `vendor_samsung_electronics_co_ltd` | high |
| `0x0075` (any) | Samsung Electronics Co., Ltd. | `samsung_device` | medium |
| `0x0059` | Nordic Semiconductor ASA | `vendor_nordic_semiconductor_asa` | high |
| `0x038F` | Tile, Inc. | `vendor_tile_inc` | high |
| `0x0499` | Espressif Inc. | `vendor_espressif_inc` | high |

### A.2 Service UUID-Based Rules

| Service UUID | Standard Name | Device Class Label | Confidence |
|-------------|---------------|-------------------|------------|
| `180F` | Battery Service | `battery_service_device` | high |
| `180D` | Heart Rate | `heart_rate_device` | high |
| `180A` | Device Information | `device_information_device` | high |
| `1812` | Human Interface Device | `hid_device` | high |
| `FEAA` | Eddystone | `eddystone_beacon` | high |
| `FD6F` | Exposure Notification | `exposure_notification` | high |
| `FE9F` | Google Fast Pair | `google_fast_pair` | high |
| `FEE0` | Microsoft Azure DPS | `microsoft_azure_device_provisioning` | high |
| `FE61` | Tile | `tile` | high |
| `FD3D` | Xiaomi Mi Fit | `mi_fit` | high |

### A.3 Adding New Classification Rules

To add a new classification rule:

1. **For a new manufacturer:** Add the Company ID and vendor name to `MANUFACTURER_TABLE` in `classifier.py`.
2. **For a new service UUID class:** Add the UUID and class label to `SERVICE_CLASS_RULES` in `classifier.py`.
3. **For a new combined rule:** Add an entry to `COMBINED_RULES` in `classifier.py` specifying the manufacturer IDs and required service UUIDs.
4. Rebuild the container image (`make build-ml-classify`).
5. No database migration or table change is required — the rule logic is in application code.

Post-MVP, rules should be externalized to a YAML file on V01 so that new rules can be added without rebuilding the container image.

---

## Appendix B: Classifier Output Example

For an Apple device advertising Continuity:

**Input JSONL line (compact):**
```json
{"ts":"2026-06-06T14:00:00.123456Z","sniffer_id":1,"mac":"aa:bb:cc:dd:ee:ff","rssi":-67,"channel":37,"pdu_type":"ADV_IND","advdata":{"flags":6,"local_name":"My iPhone","service_uuids_16":["FE0F","180F"],"manufacturer":{"company_id":"004C","data_hex":"100601234567"}},"raw_advdata_hex":"0201060303aafe17..."}
```

**device_enrichment row:**
```
mac_address:     \xaabbccddeeff
observed_ts:     2026-06-06 14:00:00.123456+00
sniffer_id:      1
local_name:      "My iPhone"
service_uuids_16: {"FE0F","180F"}
manufacturer_id: 76  (0x004C)
manufacturer_data: \x100601234567
flags:           6
```

**device_summary.enrichment_data for this MAC:**
```json
{
  "classes": [
    "vendor_apple_inc",
    "apple_continuity",
    "apple_device",
    "battery_service_device"
  ],
  "vendor": "Apple, Inc.",
  "device_name": "My iPhone",
  "last_enriched": "2026-06-06T14:00:00.123456+00:00",
  "class_details": {
    "vendor_apple_inc": {
      "match_type": "manufacturer",
      "manufacturer_id": 76,
      "confidence": "high"
    },
    "apple_continuity": {
      "match_type": "combined",
      "manufacturer_id": 76,
      "service_uuid": "FE0F",
      "confidence": "high"
    },
    "apple_device": {
      "match_type": "combined",
      "manufacturer_id": 76,
      "confidence": "medium"
    },
    "battery_service_device": {
      "match_type": "service_uuid",
      "service_uuid": "180F",
      "confidence": "high"
    }
  },
  "residency_score": 0.85,
  "observation_count": 423,
  "observation_window_hours": 4.5,
  "unclassified": false
}
```

---

*End of C08 ML Enrichment Design Document.*

---

## References

[1] Bluetooth SIG, Inc. "Assigned Numbers." https://www.bluetooth.com/specifications/assigned-numbers/, 2024. The Bluetooth SIG Assigned Numbers document contains the Company Identifier registry (16-bit IDs mapping to vendor names) and GATT Service UUID assignments — the authoritative sources for `MANUFACTURER_TABLE` and `SERVICE_CLASS_RULES` in `classifier.py` (§4.4, §A.1, §A.2).

[2] Python Software Foundation. "asyncio — Asynchronous I/O — Python 3.13 Documentation." https://docs.python.org/3.13/library/asyncio.html, 2024. Python 3.13 asyncio library provides the async/await syntax, `asyncio.run()` entry point, and `asyncio.Task` infrastructure used by C08's `runner.py` (§4.2) for concurrent batch processing of JSONL files.

[3] D. Varrazzo and The Psycopg Team. "Psycopg 3 Documentation — Concurrent Operations." https://www.psycopg.org/psycopg3/docs/advanced/async.html, 2024. Psycopg 3.1's `AsyncConnection` and `AsyncCursor` provide the async PostgreSQL interface used in `db.py` (§4.7). The `executemany()` method enables batch INSERT with parameterized queries. §"Asynchronous operations" documents the `async with await psycopg.AsyncConnection.connect()` pattern.

[4] PostgreSQL Global Development Group. "PostgreSQL 17 Documentation: 8.14. JSON Types." https://www.postgresql.org/docs/17/datatype-json.html, 2024. §8.14.3 documents the `@>` containment operator and `||` concatenation operator for JSONB — the latter used by C08 to merge new `enrichment_data` with existing data in `device_summary` via `ON CONFLICT DO UPDATE` (§3.3, §4.7). Also documents the GIN indexing support for JSONB columns used by C09's read path.
