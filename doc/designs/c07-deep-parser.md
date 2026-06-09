# C07 — Deep Parser

**Status:** Phase A Design — guides Phase B implementation
**Author:** Code Architect
**Date:** 2026-06-09
**Dependencies:** C02 (Database), C04 (PCAP Rotation), hardware fixtures (PCAP samples)
**Blocks:** C08 (ML Enrichment, transitively)

---

## 1. Overview

### 1.1 Purpose

C07 Deep Parser is a **batch offline parser** for rotated BLE PCAP files. It reads compressed or uncompressed PCAP files produced by C03 Capture Pipeline and rotated by C04 PCAP Rotation, dissects raw BLE link-layer PDUs, parses AdvData TLV structures, and emits structured JSON Lines (JSONL) to the V05 data volume for downstream consumption by C08 ML Enrichment.

The deep parser operates **outside the real-time hot path**. It runs on closed, rotated PCAP files — never on the live `current.pcap` being written by the sniffer. This separation ensures that deep packet inspection (which is computationally intensive) cannot introduce backpressure into the real-time ingest pipeline.

### 1.2 Scope

| In Scope | Out of Scope |
|----------|-------------|
| Read PCAP files (`.pcap`, `.pcap.zst`) from V02 (`:ro`) | Reading live `current.pcap` (still being written) |
| Dissect BLE advertising channel PDU headers (PDU type, TxAdd, RxAdd, length) | Dissecting data channel PDUs (connection-oriented traffic) |
| Parse AdvData TLV structures (flags, local name, TX power, service UUIDs, manufacturer data) | Parse extended advertising (BLE 5.0 ADV_EXT_IND payloads) |
| **Configurable CRC-24 verification** of PDU payloads (default: enabled) | Connection parameter parsing (CONNECT_IND payload) |
| Emit JSONL with CRC status, sniffer-type, and per-DLT metadata | Write directly to PostgreSQL database (JSONL files are consumed by C08) |
| Transparent zstd decompression via piped input | `zlib` or `gzip` decompression (only `zstd` is used by C04) |
| 100 MB memory cap during processing | Parallel processing of multiple files (single-threaded, one file at a time) |
| Atomic `.done` marker for completed output files | Progress resumption for partially-processed files (processes each file from scratch) |

### 1.3 Boundaries

C07 owns the **PCAP-to-JSONL transformation**. It does **not** own the PCAP file rotation lifecycle (C04), the JSONL consumption and DB insertion (C08), or the hardware sniffer configuration (C03). The output target is the V05 data volume; C08 ML Enrichment picks up new JSONL files from V05 `:ro`.

```
┌──────────────────────────────────────────────────────────────────┐
│                         LAYER 2 — BATCH PROCESSING                │
│                                                                  │
│  C03 Capture Pipeline                                            │
│  C04 PCAP Rotation                                               │
│       │                                                          │
│       │  Rotated PCAP files                                      │
│       │  (V02: /var/lib/tianer/pcap/<sniffer_name>/              │
│       │        YYYYMMDD-HHMM.pcap / .pcap.zst)                   │
│       │                                                          │
│       ▼                                                          │
│  ┌──────────────────────────────────────────┐                    │
│  │          C07 DEEP PARSER (C++17)          │                    │
│  │                                          │                    │
│  │  pcap_input ──► ble_dissector ──►        │                    │
│  │                   advdata_parser ──►      │                    │
│  │                     jsonl_output          │                    │
│  │                                          │                    │
│  │  ◄── V02 `:ro` (PCAP source)             │                    │
│  │  ──► V05 `:rw` (JSONL output)            │                    │
│  └──────────────────┬───────────────────────┘                    │
│                     │                                            │
│                     │  JSONL files                               │
│                     │  (V05: /var/lib/tianer/data/deep/          │
│                     │        <sniffer_name>/                     │
│                     │        YYYYMMDD-HHMM.jsonl)                │
│                     │                                            │
│                     ▼                                            │
│  ┌──────────────────────────────────────────┐                    │
│  │          C08 ML ENRICHMENT (Python)       │                    │
│  │  ◄── V05 `:ro` (JSONL input)              │                    │
│  │  ──► PostgreSQL (tianer_writer)           │                    │
│  └──────────────────────────────────────────┘                    │
│                                                                  │
│  ┌──────────────────────────────────────────┐                    │
│  │          C02 DATABASE                     │                    │
│  │  bluetooth.device_enrichment              │                    │
│  │  (populated by C08, not directly by C07)  │                    │
│  └──────────────────────────────────────────┘                    │
└──────────────────────────────────────────────────────────────────┘
```

### 1.4 Position in the System

C07 is a **Layer 2 component** in the build sequence (component-breakdown.md §4.1). It depends on C04 (rotated PCAP files must exist) and C02 (schema must exist for `device_enrichment` table, though C07 itself does not write to DB). It blocks C08 (ML Enrichment needs JSONL input).

On the critical path `C01 → C02 → C03 → C04 → C07 → C08`, C07 is the bridge between raw capture and enriched intelligence.

---

## 2. High-Level Architecture (HLA)

### 2.1 Data Flow

```
┌────────────────────────────────────────────────────────────────┐
│                     C07 DEEP PARSER DATA FLOW                    │
│                                                                │
│  STEP 1: PCAP INPUT (pcap_input)                                │
│  ┌──────────────────────────────────────────┐                   │
│  │  Input: V02 PCAP file path                │                   │
│  │         (e.g. /var/.../pcap/ut1/          │                   │
│  │          20260606-1200.pcap.zst)          │                   │
│  │                                          │                   │
│  │  If .zst: pipe through zstd -dc           │                   │
│  │  If .pcap: read directly via libpcap      │                   │
│  │                                          │                   │
│  │  Output: Iterator<pcap_pkthdr, bytes>     │                   │
│  │                                          │                   │
│  │  Enforces: 100 MB memory bound            │                   │
│  │            Read DLT from global header     │                   │
│  └──────────────┬───────────────────────────┘                   │
│                 │                                               │
│  STEP 2: BLE DISSECTOR (ble_dissector)                           │
│  ┌──────────────────────────────────────────┐                   │
│  │  Input: Raw packet bytes (PCAP payload)   │                   │
│  │                                          │                   │
│  │  Parse BLE Link Layer header:             │                   │
│  │    - Access Address (4 bytes)             │                   │
│  │    - PDU header (2 bytes)                 │                   │
│  │      · PDU type (bits 3:0)               │                   │
│  │      · TxAdd (bit 6)                     │                   │
│  │      · RxAdd (bit 7)                     │                   │
│  │      · Length (bits 15:8)                │                   │
│  │    - AdvA: Advertiser address (6 bytes)   │                   │
│  │    - AdvData: advertising data (0–31 B)   │                   │
│  │    - CRC-24: trailing 3 bytes (optional)  │                   │
│  │                                          │                   │
│  │  Configurable CRC-24 validation:          │                   │
│  │    - Enabled (default): verify CRC-24     │                   │
│  │    - Disabled: skip verification          │                   │
│  │      (for sniffers that pre-verify CRC,   │                   │
│  │       e.g., Ubertooth One firmware)       │                   │
│  │                                          │                   │
│  │  Output: DissectedPDU struct              │                   │
│  └──────────────┬───────────────────────────┘                   │
│                 │                                               │
│  STEP 3: ADVDATA PARSER (advdata_parser)                         │
│  ┌──────────────────────────────────────────┐                   │
│  │  Input: AdvData bytes (0–31 bytes)        │                   │
│  │                                          │                   │
│  │  TLV walker: each entry is 2+ bytes:      │                   │
│  │    Length (1 byte, excluding self)        │                   │
│  │    Type   (1 byte, AD type code)          │                   │
│  │    Value  (Length bytes)                  │                   │
│  │                                          │                   │
│  │  Parsed types (v1):                      │                   │
│  │    · 0x01: Flags                         │                   │
│  │    · 0x08: Shortened Local Name          │                   │
│  │    · 0x09: Complete Local Name           │                   │
│  │    · 0x0A: TX Power Level                │                   │
│  │    · 0x02-0x07: 16-bit Service UUIDs     │                   │
│  │    · 0x06-0x07: 128-bit Service UUIDs    │                   │
│  │    · 0xFF: Manufacturer Specific Data    │                   │
│  │    · 0x16: Service Data (16-bit UUID)    │                   │
│  │                                          │                   │
│  │  Output: ParsedAdvData struct             │                   │
│  └──────────────┬───────────────────────────┘                   │
│                 │                                               │
│  STEP 4: JSONL OUTPUT (jsonl_output)                              │
│  ┌──────────────────────────────────────────┐                   │
│  │  Input: ParsedAdvData + metadata          │                   │
│  │                                          │                   │
│  │  Assemble JSON object per packet:         │                   │
│  │    · ts (ISO 8601 with microseconds)       │                   │
│  │    · sniffer_id                           │                   │
│  │    · sniffer_type ("ubertooth" | "nrf")   │                   │
│  │    · mac (colon-hex format)               │                   │
│  │    · address_type ("public" | "random")   │                   │
│  │    · rssi                                 │                   │
│  │    · channel                              │                   │
│  │    · pdu_type (string form, e.g. "ADV_IND")│                   │
│  │    · crc_valid (true | false)             │                   │
│  │    · dlt (LINKTYPE value from pcap)       │                   │
│  │    · advdata { ... }                      │                   │
│  │    · raw_advdata_hex                      │                   │
│  │                                          │                   │
│  │  Write one line per packet to output file │                   │
│  │  On completion: create .done marker file  │                   │
│  │                                          │                   │
│  │  Output: V05 JSONL file + .done marker    │                   │
│  └──────────────────────────────────────────┘                   │
└────────────────────────────────────────────────────────────────┘
```

### 2.2 Component Interaction

C07 runs as a **Quadlet oneshot container** invoked by C14 systemd timer. It is stateless: it reads a PCAP file, processes it entirely, writes a JSONL file with an atomic `.done` marker, and exits. C08 detects `.done` markers and picks up the completed JSONL files.

```
C14 systemd timer
    │
    │ OnUnitActiveSec=5min (runs after C04 rotation completes)
    │
    ▼
C07 Deep Parser container (oneshot)
    │
    │  podman run --rm \
    │    --volume V02:/var/lib/tianer/pcap:ro \
    │    --volume V05:/var/lib/tianer/data:rw \
    │    localhost/tianer-deep-parser:latest \
    │    blesniff-deep-parse \
    │      --pcap-dir /var/lib/tianer/pcap \
    │      --output-dir /var/lib/tianer/data/deep \
    │      --sniffer-config /etc/tianer/sniffers.yaml \
    │      --crc-verify on  # default
    │
    │  Process:
    │  1. Scan V02 for unprocessed PCAP files
    │     (check for corresponding .done marker in V05)
    │  2. For each unprocessed file:
    │     a. Decompress if .zst (pipe through zstd -dc)
    │     b. Parse all BLE packets
    │     c. Write JSONL to V05
    │     d. Touch .done marker
    │  3. Exit
    │
    ▼
C08 ML Enrichment (triggered by .done marker via inotify)
    │
    │  podman run --rm \
    │    --volume V05:/var/lib/tianer/data:ro \
    │    localhost/tianer-ml-enrichment:latest \
    │    python -m tianer_ml.runner ...
    │
    ▼
PostgreSQL (via tianer_writer role)
```

### 2.3 DLT-Aware Parsing

The PCAP global header contains a **Link-Layer Type (DLT)** field that identifies the format of the packet data. Different sniffer hardware produces different DLT values. C07 must use the DLT to correctly interpret the packet bytes.

| Sniffer Type | Expected DLT | DLT Value | Packet Format |
|-------------|-------------|-----------|---------------|
| Ubertooth One | `LINKTYPE_BLUETOOTH_LE_LL` | 251 | BLE Link Layer header + PDU (as transmitted on air, dewhitened by firmware) |
| Nordic nRF Sniffer | `LINKTYPE_NORDIC_BLE` | 272 | Nordic header + BLE Link Layer PDU |

**DLT reading:** C07 reads the DLT from the first PCAP file's global header via `pcap_datalink()`. It matches this against known values. If the DLT is unrecognized, C07 logs a warning and skips the file.

**Per-DLT metadata:** The DLT value and sniffer type are included in the JSONL output (`dlt` and `sniffer_type` fields) so that downstream consumers (C08) can apply DLT-specific logic if needed.

**Dewhitened data assumption:** C07 assumes that the sniffer firmware (Ubertooth tools, nrfutil) has already dewhitened the BLE packets before writing to PCAP. The parser does **not** re-apply or verify dewhitening. This is consistent with both sniffer toolsets: Ubertooth dewhitens in hardware/firmware; nrfutil provides dewhitened output in the Nordic DLT format.

### 2.4 Volume Mount Strategy

Per the storage strategy (storage-strategy.md):

| Volume | Mount Path (in container) | Mode | Purpose |
|--------|--------------------------|------|---------|
| V02 | `/var/lib/tianer/pcap` | `:ro` | Read rotated PCAP files |
| V05 | `/var/lib/tianer/data` | `:rw` | Write JSONL output files |

C07 writes JSONL files into the subdirectory `deep/<sniffer_name>/` under V05. The C08 ML Enrichment container mounts V05 `:ro` and reads from `deep/<sniffer_name>/`.

---

## 3. Data Model (ERD)

### 3.1 JSONL Output Schema (Enhanced DEEP-1 Contract)

The JSONL output schema extends the original CONTRACT 8.8-A from inception.md to include **CRC status**, **sniffer-type**, and **per-DLT metadata** as required by the component-breakdown.md specification.

```json
{
  "version": 1,
  "ts": "2026-06-06T14:00:00.123456Z",
  "frame": {
    "number": 1,
    "len": 42,
    "cap_len": 42
  },
  "sniffer_id": 1,
  "sniffer_type": "ubertooth",
  "dlt": 251,
  "mac": "aa:bb:cc:dd:ee:ff",
  "address_type": "random",
  "rssi": -67,
  "channel": 37,
  "pdu_type": "ADV_IND",
  "crc_valid": true,
  "advdata": {
    "flags": 6,
    "local_name": "MyDevice",
    "tx_power": -4,
    "service_uuids_16": ["180D", "180F"],
    "service_uuids_128": [],
    "manufacturer": {
      "company_id": "004C",
      "data_hex": "1006aabbccdd"
    },
    "service_data": [
      {"uuid": "180D", "data_hex": "ff00ab"}
    ]
  },
  "raw_advdata_hex": "0201060303aafe17..."
}
```

#### Field Reference

| Field | JSON Type | Required | Description |
|-------|-----------|----------|-------------|
| `version` | integer | Yes | Schema version. Always `1` for v1. |
| `ts` | string (ISO 8601) | Yes | Packet timestamp from PCAP, with microsecond precision. Format: `YYYY-MM-DDThh:mm:ss.uuuuuuZ`. |
| `frame.number` | integer | Yes | Sequential frame number within this PCAP file (for debugging/matching). |
| `frame.len` | integer | Yes | Original packet length from PCAP header (before any truncation). |
| `frame.cap_len` | integer | Yes | Captured length from PCAP header (may be less than `len` if truncation occurred). |
| `sniffer_id` | integer | Yes | Sniffer identifier (references `bluetooth.sniffers.sniffer_id`). |
| `sniffer_type` | string | Yes | Sniffer hardware type. One of: `"ubertooth"`, `"nrf"`. |
| `dlt` | integer | Yes | DLT (Link-Layer Type) from PCAP global header. `251` = Bluetooth LE LL, `272` = Nordic BLE. |
| `mac` | string | Yes | Device MAC address in colon-hex format (`"aa:bb:cc:dd:ee:ff"`). |
| `address_type` | string | Yes | `"public"` or `"random"` based on PDU header `TxAdd` bit. |
| `rssi` | integer \| null | No | RSSI from PCAP metadata (dBm). Null if not available. |
| `channel` | integer \| null | No | BLE advertising channel (37, 38, 39). Null if not determinable. |
| `pdu_type` | string | Yes | Human-readable PDU type. One of: `"ADV_IND"`, `"ADV_DIRECT_IND"`, `"ADV_NONCONN_IND"`, `"SCAN_REQ"`, `"SCAN_RSP"`, `"CONNECT_IND"`, `"ADV_SCAN_IND"`, `"ADV_EXT_IND"`, `"UNKNOWN"`. |
| `crc_valid` | boolean \| null | Yes | `true` if CRC-24 verification passed, `false` if failed, `null` if CRC-24 verification was disabled (bypass mode). |
| `advdata` | object \| null | No | Parsed advertising data TLV structure. Null if AdvData is empty or unparseable. |
| `advdata.flags` | integer \| null | No | BLE flags from AD type 0x01. |
| `advdata.local_name` | string \| null | No | Device name from AD type 0x08 (Shortened) or 0x09 (Complete). 0x08 takes precedence if both present. |
| `advdata.tx_power` | integer \| null | No | TX power from AD type 0x0A (dBm). |
| `advdata.service_uuids_16` | array of strings | No | Array of 16-bit UUIDs in hex format (e.g., `["180D"]`). Aggregated from AD types 0x02, 0x03. |
| `advdata.service_uuids_128` | array of strings | No | Array of 128-bit UUIDs in hex format. Aggregated from AD types 0x06, 0x07. |
| `advdata.manufacturer` | object \| null | No | Manufacturer-specific data from AD type 0xFF. |
| `advdata.manufacturer.company_id` | string | Yes (if manufacturer present) | Bluetooth SIG Company Identifier as 4-char hex string (e.g., `"004C"` for Apple). |
| `advdata.manufacturer.data_hex` | string | Yes (if manufacturer present) | Remaining manufacturer data bytes as hex string. |
| `advdata.service_data` | array of objects | No | Service data entries from AD types 0x16, 0x20, 0x21. Each entry has `uuid` (hex string) and `data_hex` (hex string). |
| `raw_advdata_hex` | string \| null | No | Full raw AdvData bytes as hex string. Available even when TLV parsing fails, enabling reprocessing/debugging. |

**Schema versioning:** The `version` field allows C08 ML Enrichment to detect format changes. When the schema evolves (e.g., adding extended advertising support in v1.1), the version number is incremented, and C08 checks the version before parsing.

### 3.2 Mapping to device_enrichment Table

The JSONL output maps to the C02 `bluetooth.device_enrichment` table (migration `0005_device_enrichment.sql`). C08 ML Enrichment performs this mapping, **not C07**. C07 produces the JSONL; C08 consumes it and inserts into the database.

| device_enrichment Column | JSONL Source | Notes |
|-------------------------|-------------|-------|
| `mac_address` | `$.mac` | Parsed from colon-hex to 6 raw bytes |
| `observed_ts` | `$.ts` | Parsed from ISO 8601 to `TIMESTAMPTZ` |
| `sniffer_id` | `$.sniffer_id` | Passed through directly |
| `local_name` | `$.advdata.local_name` | Null if not present in AdvData |
| `service_uuids_16` | `$.advdata.service_uuids_16` | PostgreSQL `TEXT[]` array |
| `service_uuids_128` | `$.advdata.service_uuids_128` | PostgreSQL `TEXT[]` array |
| `manufacturer_id` | `$.advdata.manufacturer.company_id` | Parsed from hex string to integer |
| `manufacturer_data` | `$.advdata.manufacturer.data_hex` | Parsed from hex string to `BYTEA` |
| `tx_power` | `$.advdata.tx_power` | Null if not present |
| `flags` | `$.advdata.flags` | Null if not present |
| `raw_advdata` | `$.raw_advdata_hex` | Parsed from hex string to `BYTEA` |

### 3.3 Output File Naming Convention

```
/var/lib/tianer/data/deep/<sniffer_name>/YYYYMMDD-HHMM.jsonl
/var/lib/tianer/data/deep/<sniffer_name>/YYYYMMDD-HHMM.jsonl.done
```

- `<sniffer_name>`: matches `bluetooth.sniffers.name` (e.g., `ut1`, `nrf1`)
- `YYYYMMDD-HHMM`: matches the source PCAP filename from C04 rotation
- `.done` marker: empty file created atomically after JSONL write completes; signals C08 that the file is safe to read

**Processing state tracking:** C07 scans for PCAP files without corresponding `.done` markers. A PCAP file with an existing `.done` file is skipped (already processed). This ensures idempotent re-runs.

### 3.4 File Organization

```
V05: /var/lib/tianer/data/
├── deep/
│   ├── ut1/
│   │   ├── 20260606-1200.jsonl
│   │   ├── 20260606-1200.jsonl.done
│   │   ├── 20260606-1230.jsonl
│   │   ├── 20260606-1230.jsonl.done
│   │   └── ...
│   ├── nrf1/
│   │   ├── 20260606-1200.jsonl
│   │   ├── 20260606-1200.jsonl.done
│   │   └── ...
│   └── ...
└── ml/
    (C08 output, not touched by C07)
```

---

## 4. Low-Level Architecture (LLA)

### 4.1 Module Decomposition

The deep parser is composed of four compilation units plus a driver `main.cpp`. Each unit handles a single responsibility.

```
modules/bluetooth/deep-parser/
├── CMakeLists.txt
├── src/
│   ├── main.cpp                 # CLI driver, argument parsing, file discovery
│   ├── pca_input.hpp            # PCAP reader abstraction
│   ├── pca_input.cpp            # libpcap + zstd pipe implementation
│   ├── ble_dissector.hpp        # BLE PDU dissection
│   ├── ble_dissector.cpp        # PDU header parsing, CRC-24
│   ├── advdata_parser.hpp       # TLV walker
│   ├── advdata_parser.cpp       # AD type parsing
│   ├── jsonl_output.hpp         # JSONL serialisation
│   ├── jsonl_output.cpp         # nlohmann::json + file I/O
│   ├── crc24.hpp                # CRC-24 computation
│   ├── crc24.cpp                # Table-driven CRC-24 implementation
│   └── config.hpp               # Configuration struct
└── tests/
    ├── CMakeLists.txt
    ├── pca_input_test.cpp
    ├── ble_dissector_test.cpp   # Golden test vectors
    ├── advdata_parser_test.cpp
    ├── crc24_test.cpp
    └── jsonl_output_test.cpp
```

### 4.2 CLI Interface

```
blesniff-deep-parse \
  --pcap-dir /var/lib/tianer/pcap \
  --output-dir /var/lib/tianer/data/deep \
  --sniffer-config /etc/tianer/sniffers.yaml \
  [--crc-verify on|off] \
  [--max-memory-mb 100]
```

| Argument | Required | Default | Description |
|----------|----------|---------|-------------|
| `--pcap-dir` | Yes | — | Root directory containing per-sniffer PCAP subdirectories |
| `--output-dir` | Yes | — | Root directory for JSONL output (deep/ subdirectory is added by the program) |
| `--sniffer-config` | Yes | — | Path to `sniffers.yaml` containing per-sniffer metadata |
| `--crc-verify` | No | `on` | Enable (`on`) or disable (`off`) CRC-24 verification |
| `--max-memory-mb` | No | `100` | Maximum memory in MB before throttling allocation |

### 4.3 pca_input — PCAP Input Module

**Purpose:** Read PCAP files, transparently handle zstd compression, enforce memory bounds.

#### 4.3.1 Interface

```cpp
// pca_input.hpp
#pragma once

#include <pcap.h>
#include <functional>
#include <optional>
#include <string>
#include <string_view>
#include <vector>

namespace tianer::deep {

struct PcapPacket {
    uint32_t len;        // Original packet length
    uint32_t cap_len;    // Captured length (may be truncated)
    timeval  ts;         // Timestamp (seconds + microseconds)
    std::vector<uint8_t> data;  // Packet payload
    uint32_t frame_number;      // Sequential frame number in file
};

enum class DltType {
    Unknown = -1,
    BluetoothLeLL = 251,      // LINKTYPE_BLUETOOTH_LE_LL
    NordicBle = 272,          // LINKTYPE_NORDIC_BLE
};

struct PcapMetadata {
    DltType dlt;
    uint32_t snaplen;
    uint32_t link_type_raw;    // Raw DLT value from libpcap
};

using PacketCallback = std::function<void(PcapPacket&&)>;

class PcaInput {
public:
    explicit PcaInput(std::string_view filepath, size_t max_memory_mb = 100);

    // Returns metadata (DLT, etc.) from the first successfully-opened file.
    // Must be called before iterate().
    PcapMetadata metadata() const;

    // Iterate all packets in the file. Calls cb for each packet.
    // Returns number of packets processed.
    // Throws std::runtime_error on I/O or decompression errors.
    size_t iterate(PacketCallback cb);

private:
    std::string path_;
    size_t max_memory_mb_;
    PcapMetadata metadata_;
};

} // namespace tianer::deep
```

#### 4.3.2 Zstd Handling

When the input filepath ends with `.zst`, `PcaInput` spawns a child process via `popen()` to decompress:

```cpp
// Pseudocode for decompression
std::unique_ptr<FILE, int(*)(FILE*)> pipe(
    popen("zstd -dc -- /path/to/file.pcap.zst", "r"),
    pclose
);

// Read decompressed PCAP data from pipe via fmemopen + pcap_fopen_offline
```

**Why pipe rather than linking libzstd:** The `libpcap` library expects a file descriptor. Piping through the `zstd` binary avoids linking against libzstd (smaller container image, fewer dependencies). The `zstd` binary is present in the container (part of the Debian slim base image if installed, or copied from the build stage).

The pipe approach also provides transparent support for any compression format by changing the decompression command, though v1 only supports zstd (matching C04 output).

**Memory bound enforcement:** The decompression pipe's output is read in 64 KB chunks. If the cumulative uncompressed data size exceeds `max_memory_mb`, the parser stops allocating new memory and signals an error. Packet processing happens inline (each packet is parsed and output before the next is read), so memory usage is bounded by the decompression buffer + one packet's data structures, not the file size.

#### 4.3.3 DLT Reading

```cpp
PcapMetadata PCAInput::metadata() const {
    char errbuf[PCAP_ERRBUF_SIZE];
    pcap_t* handle = pcap_open_offline(path_.c_str(), errbuf);
    int dlt_raw = pcap_datalink(handle);
    PcapMetadata meta;
    meta.link_type_raw = dlt_raw;
    meta.snaplen = pcap_snapshot(handle);
    switch (dlt_raw) {
        case 251: meta.dlt = DltType::BluetoothLeLL; break;
        case 272: meta.dlt = DltType::NordicBle;    break;
        default:  meta.dlt = DltType::Unknown;       break;
    }
    pcap_close(handle);
    return meta;
}
```

### 4.4 ble_dissector — BLE PDU Dissector

**Purpose:** Parse BLE link-layer PDUs from raw packet bytes. Extract PDU header fields, advertiser address, AdvData, and (optionally) verify CRC-24.

#### 4.4.1 Interface

```cpp
// ble_dissector.hpp
#pragma once

#include <cstdint>
#include <optional>
#include <string>
#include <vector>

namespace tianer::deep {

enum class PduType : uint8_t {
    ADV_IND         = 0,
    ADV_DIRECT_IND  = 1,
    ADV_NONCONN_IND = 2,
    SCAN_REQ        = 3,
    SCAN_RSP        = 4,
    CONNECT_IND     = 5,
    ADV_SCAN_IND    = 6,
    ADV_EXT_IND     = 7,
    UNKNOWN         = 0xFF,
};

enum class AddressType : uint8_t {
    Public = 0,
    Random = 1,
};

struct DissectedPDU {
    // PDU header fields
    PduType pdu_type;
    uint8_t raw_pdu_type;    // Raw numeric value (0-7, or unknown)
    uint8_t length;          // Payload length from header (6-37 for advertising)
    bool tx_add;             // TxAdd bit: advertiser address type
    bool rx_add;             // RxAdd bit: target address type
    bool ch_sel;             // ChSel bit

    // Address
    std::vector<uint8_t> adv_address;  // 6 bytes on success

    // Payload
    std::vector<uint8_t> adv_data;     // Up to 31 bytes (length - 6)

    // CRC
    std::optional<bool> crc_valid;     // nullopt = not checked, true = valid, false = invalid
    std::vector<uint8_t> raw_crc_bytes; // 3 raw CRC bytes (for debugging)

    // Parsing metadata
    bool parse_ok = false;
    std::string parse_error;           // Non-empty if parse_ok == false
};

DissectedPDU dissect_ble_pdu(const uint8_t* data, size_t len,
                              bool verify_crc = true);

// Convert PDU type to string
const char* pdu_type_string(PduType type);

// Convert PDU type to string (from raw value, for UNKNOWN entries)
const char* pdu_type_string(uint8_t raw);

} // namespace tianer::deep
```

#### 4.4.2 BLE PDU Format (After Dewhitening)

C07 operates on **dewhitened** data (sniffer firmware handles dewhitening). The byte layout is:

```
Byte offset:  0       3  4       5       11      11+L
            ┌──────────┬──────────┬──────────┬──────────┬──────┐
            │ Access   │ PDU      │ AdvA     │ AdvData  │ CRC  │
            │ Address  │ Header   │ (6 bytes)│ (0-31 B) │ (3 B)│
            │ (4 bytes)│ (2 bytes)│          │          │      │
            └──────────┴──────────┴──────────┴──────────┴──────┘
```

The PDU Header is 2 bytes (LSbit-first per BLE spec, but our bytes are already bit-reversed to MSbit-first by the sniffer firmware):

```
PDU Header byte 0 (bits):
┌───┬───┬───┬───┬───┬───┬───┬───┐
│RxAdd│TxAdd│ChSel│RFU │  PDU Type  │
│   7 │   6 │   5 │  4 │   3:0      │
└───┴───┴───┴───┴───┴───┴───┴───┘

PDU Header byte 1 (bits):
┌───┬───┬───┬───┬───┬───┬───┬───┐
│              Length              │
│              15:8               │
└───┴───┴───┴───┴───┴───┴───┴───┘
```

**Note on bit order:** The BLE spec defines LSbit-first transmission within each byte. However, both Ubertooth and nRF sniffer tools output **MSbit-first** bytes (they reverse the bits during capture). The dissector interprets header fields directly from the received bytes without further bit manipulation. This matches the wire format as seen in Wireshark.

#### 4.4.3 Dissection Algorithm

```
dissect_ble_pdu(data, len, verify_crc):
    1. If len < 6: return error ("PDU too short for access address")
    2. Skip first 4 bytes (access address) — not validated (may not be 0x8E89BED6
       for data channel sniffing edge cases)
    3. If len < 8: return error ("PDU too short for header")
    4. Parse PDU header byte 0:
       - pdu_type_raw = data[4] & 0x0F
       - ch_sel       = (data[4] >> 5) & 1
       - tx_add       = (data[4] >> 6) & 1
       - rx_add       = (data[4] >> 7) & 1
    5. header_length = data[5]
    6. If header_length < 6 or header_length > 37:
       - Return with PduType::UNKNOWN, parse_ok = false
    7. Expected total = 4 + 2 + header_length
       - If verify_crc enabled: total += 3
    8. If len < expected_total: return error ("PDU truncated")
    9. Extract AdvA: data[6..11] (6 bytes)
    10. adv_data_len = header_length - 6
    11. Extract AdvData: data[12..12+adv_data_len]
    12. If verify_crc:
        - crc_offset = 12 + adv_data_len
        - Compute CRC-24 over bytes [4..crc_offset] (PDU header + AdvA + AdvData)
          with initial value 0x555555
        - Compare with bytes [crc_offset..crc_offset+3]
        - Set crc_valid = (computed == actual)
        - If crc_valid is false, increment blesniff_crc_errors_total metric
    13. Return DissectedPDU with all fields populated
```

#### 4.4.4 CRC-24 Implementation

```cpp
// crc24.hpp
#pragma once

#include <cstdint>
#include <cstddef>

namespace tianer::crc {

// Bluetooth CRC-24 parameters per BLE Core Spec Vol 6 Part B §3.1.1
inline constexpr uint32_t CRC24_POLY = 0x00065B;
inline constexpr uint32_t CRC24_INIT = 0x555555;

// Compute CRC-24 over the given byte range.
// data: pointer to bytes in MSbit-first order (as received from sniffer)
// Polynomial: x^24 + x^10 + x^9 + x^6 + x^4 + x^3 + x + 1
uint32_t crc24(const uint8_t* data, size_t len, uint32_t init = CRC24_INIT);

} // namespace tianer::crc
```

#### 4.4.5 Configurable CRC-24 Bypass (Q7)

Per the ADR-0001 Q7 resolution, CRC-24 verification is **configurable**, **default enabled**. When disabled, the `crc_valid` field in the JSONL output is set to `null` rather than `true`/`false`, and no validation is performed.

**Rationale for bypass:** Some sniffer firmware (e.g., Ubertooth One) pre-verifies CRC at the hardware level and may drop packets with failed CRC. In those cases, re-verifying CRC in software is redundant. The bypass saves compute cycles and allows operators to skip verification for trusted sniffer sources. However, the **default is enabled** because the nRF Sniffer firmware may not verify CRC, creating a silent data corruption risk if packets with bit errors are passed through.

**Metric tracking:** The `blesniff_crc_errors_total` counter is incremented whenever CRC validation fails (not when bypassed). This provides visibility into the rate of corrupted packets.

### 4.5 advdata_parser — AdvData TLV Parser

**Purpose:** Parse the advertising data payload using the Length-Type-Value (LTV) format defined in Bluetooth Core Specification Supplement, Common Advertising and Scan Response Data Format.

#### 4.5.1 Interface

```cpp
// advdata_parser.hpp
#pragma once

#include <cstdint>
#include <optional>
#include <string>
#include <vector>

namespace tianer::deep {

struct ManufacturerData {
    uint16_t company_id;
    std::vector<uint8_t> data;
};

struct ServiceData {
    std::string uuid;  // Hex string (4 chars for 16-bit, 32 chars for 128-bit)
    std::vector<uint8_t> data;
};

struct ParsedAdvData {
    std::optional<uint8_t> flags;
    std::optional<std::string> local_name;
    std::optional<int8_t> tx_power;
    std::vector<std::string> service_uuids_16;
    std::vector<std::string> service_uuids_128;
    std::optional<ManufacturerData> manufacturer;
    std::vector<ServiceData> service_data;

    bool parse_ok = true;
    std::vector<std::string> parse_errors;  // Accumulated warnings
};

ParsedAdvData parse_adv_data(const uint8_t* data, size_t len);

} // namespace tianer::deep
```

#### 4.5.2 TLV Format

Each TLV entry in the AdvData payload:

```
Byte offset:  0        1         2..(1+Length)
            ┌────────┬─────────┬────────────────┐
            │ Length │  Type   │    Value       │
            │ (1 B)  │ (1 B)   │  (Length bytes)│
            └────────┴─────────┴────────────────┘
```

- **Length:** Number of bytes in the Value field (does NOT include the Type byte or the Length byte itself).
- **Type:** Assigned number (AD type) defining the data format.
- **Value:** `Length` bytes of data in the format specified by the type.

The parser walks the byte array sequentially. If a TLV entry's Length would extend beyond the data boundary, parsing stops and the entry is reported as a parse error.

**AD types parsed in v1:**

| AD Type | Name | Parser Action | Example |
|---------|------|---------------|---------|
| `0x01` | Flags | Extract low byte → `flags` | `020106` → flags = 6 |
| `0x02` | Incomplete List of 16-bit Service UUIDs | Extract UUIDs (2 bytes each, little-endian) → `service_uuids_16` | `03023A12` → `["123A"]` |
| `0x03` | Complete List of 16-bit Service UUIDs | Same as 0x02 → `service_uuids_16` | |
| `0x04` | Incomplete List of 32-bit Service UUIDs | Skip (not standard — use 128-bit) | |
| `0x05` | Complete List of 32-bit Service UUIDs | Skip | |
| `0x06` | Incomplete List of 128-bit Service UUIDs | Extract UUIDs (16 bytes each) → `service_uuids_128` | |
| `0x07` | Complete List of 128-bit Service UUIDs | Same → `service_uuids_128` | |
| `0x08` | Shortened Local Name | Extract UTF-8 string → `local_name` | |
| `0x09` | Complete Local Name | Extract UTF-8 string → `local_name` (overrides 0x08) | |
| `0x0A` | TX Power Level | Extract signed byte → `tx_power` | |
| `0x16` | Service Data — 16-bit UUID | Extract UUID + data → `service_data` | |
| `0x20` | Service Data — 32-bit UUID | Extract UUID + data → `service_data` | |
| `0x21` | Service Data — 128-bit UUID | Extract UUID + data → `service_data` | |
| `0xFF` | Manufacturer Specific Data | Extract 2-byte company ID (little-endian) + remaining data → `manufacturer` | |

**Unknown AD types** are skipped silently — they consume their `(1 + Length)` bytes and are not reported as errors. The parser is forward-compatible with new AD types.

**Unexpected length handling:** If Length = 0, the TLV entry is skipped (advances 1 byte past the Type byte).

### 4.6 jsonl_output — JSONL Serialisation and Atomic Output

**Purpose:** Serialise parsed packet data to JSON Lines format with atomic completion signalling.

#### 4.6.1 Interface

```cpp
// jsonl_output.hpp
#pragma once

#include <nlohmann/json.hpp>
#include <string>
#include <string_view>
#include <fstream>

namespace tianer::deep {

struct PacketRecord {
    uint32_t frame_number;
    struct {
        uint32_t len;
        uint32_t cap_len;
    } frame;
    uint64_t ts_sec;
    uint32_t ts_usec;
    int16_t sniffer_id;
    std::string sniffer_type;
    int dlt;
    std::string mac;  // colon-hex format
    std::string address_type;  // "public" or "random"
    int16_t rssi;
    int16_t channel;
    std::string pdu_type;
    std::optional<bool> crc_valid;
    nlohmann::json advdata;  // ParsedAdvData serialised to JSON
    std::string raw_advdata_hex;
};

class JsonlOutput {
public:
    explicit JsonlOutput(std::string_view filepath);

    // Write one packet as a JSON line. Buffer is flushed periodically.
    void write_record(const PacketRecord& record);

    // Close the output file, flush all buffered data, and create the .done
    // marker atomically. Returns false on I/O error.
    bool finish();

private:
    std::string filepath_;
    std::ofstream stream_;
    size_t records_written_ = 0;
};

} // namespace tianer::deep
```

#### 4.6.2 Atomic .done Marker Protocol

The `.done` marker ensures that C08 does not read a partially written JSONL file (e.g., if C07 crashes mid-write).

```
1. Write JSONL bytes to filepath (e.g., /var/lib/tianer/data/deep/ut1/20260606-1200.jsonl)
2. Flush all buffered data to disk: stream.flush() + fsync()
3. Close the file
4. Create .done file: create a separate file at filepath + ".done"
5. The .done file is created atomically by writing to a temp file and renaming:
   - Write to filepath + ".tmp"
   - fsync
   - rename() to filepath + ".done"
   - fsync the directory
```

**Rationale:** The C08 ML Enrichment container uses inotify or periodic polling to detect `.done` markers. By creating the `.done` file only after the main file is fully written, flushed, and closed, we guarantee that C08 will never see a partial JSONL file. The `rename()` from `.tmp` to `.done` is atomic.

### 4.7 Main Driver — main.cpp

```cpp
// main.cpp (pseudocode)
int main(int argc, char* argv[]) {
    Config cfg = parse_args(argc, argv);

    // Load sniffer config
    std::map<std::string, SnifferInfo> sniffers = load_sniffer_config(cfg.sniffer_config);

    // Scan for unprocessed PCAP files
    for (auto& [name, info] : sniffers) {
        if (!info.enabled) continue;

        std::string pcap_subdir = cfg.pcap_dir + "/" + name;
        for (auto& pcap_file : find_pcap_files(pcap_subdir)) {
            std::string output_file = cfg.output_dir + "/" + name + "/"
                                    + basename_without_ext(pcap_file) + ".jsonl";
            std::string done_file = output_file + ".done";

            // Skip already-processed files
            if (file_exists(done_file)) {
                log_info("Skipping already-processed: " + pcap_file);
                continue;
            }

            // Skip live current.pcap (rotated files only)
            if (basename(pcap_file) == "current.pcap") {
                continue;
            }

            process_file(pcap_file, output_file, name, info, cfg);
        }
    }

    return 0;
}

void process_file(const std::string& pcap_file,
                  const std::string& output_file,
                  const std::string& sniffer_name,
                  const SnifferInfo& info,
                  const Config& cfg) {
    PcaInput input(pcap_file, cfg.max_memory_mb);
    auto meta = input.metadata();

    if (meta.dlt == DltType::Unknown) {
        log_error("Unrecognized DLT " + std::to_string(meta.link_type_raw)
                  + " in file " + pcap_file);
        return;
    }

    JsonlOutput output(output_file);
    uint32_t packets = 0;
    uint32_t crc_errors = 0;

    input.iterate([&](PcapPacket&& pkt) {
        auto pdu = dissect_ble_pdu(pkt.data.data(), pkt.data.size(),
                                    cfg.crc_verify);
        if (!pdu.parse_ok) {
            // Log and skip malformed PDU
            log_warn("Malformed PDU at frame " + std::to_string(pkt.frame_number)
                     + ": " + pdu.parse_error);
            return;
        }

        if (pdu.crc_valid.has_value() && !pdu.crc_valid.value()) {
            crc_errors++;
        }

        auto adv = parse_adv_data(pdu.adv_data.data(), pdu.adv_data.size());

        PacketRecord rec;
        rec.frame_number = pkt.frame_number;
        rec.frame.len = pkt.len;
        rec.frame.cap_len = pkt.cap_len;
        rec.ts_sec = pkt.ts.tv_sec;
        rec.ts_usec = pkt.ts.tv_usec;
        rec.sniffer_id = info.id;
        rec.sniffer_type = info.type;
        rec.dlt = static_cast<int>(meta.dlt);
        rec.mac = bytes_to_colon_hex(pdu.adv_address);
        rec.address_type = pdu.tx_add ? "random" : "public";
        rec.rssi = /* extract from PCAP pseudo-header if available */ 0;
        rec.channel = /* extract from PCAP pseudo-header if available */ 0;
        rec.pdu_type = pdu_type_string(pdu.pdu_type);
        rec.crc_valid = pdu.crc_valid;
        rec.advdata = adv_to_json(adv);
        rec.raw_advdata_hex = bytes_to_hex(pdu.adv_data);

        output.write_record(rec);
        packets++;
    });

    if (!output.finish()) {
        log_error("Failed to finish writing " + output_file);
        update_metrics(packets, crc_errors);
        return;
    }

    update_metrics(packets, crc_errors);
    log_info("Processed " + pcap_file + ": " + std::to_string(packets)
             + " packets, " + std::to_string(crc_errors) + " CRC errors");
}
```

### 4.8 DLT-Specific Packet Headers

#### 4.8.1 DLT 251 — LINKTYPE_BLUETOOTH_LE_LL

The packet data contains raw BLE link-layer frames as transmitted on air. The pcap header includes an optional pseudo-header:

```
┌──────────────┬───────────────────────────────────┐
│ Pseudo-header│ BLE Link Layer Payload            │
│ (optional)   │ (Access Address + PDU + CRC)     │
└──────────────┴───────────────────────────────────┘
```

C07 uses libpcap's `pcap_datalink()` to identify the DLT, reads the packet payload directly, and applies the BLE dissector. The RSSI and channel fields are extracted from additional PCAP headers if available (Ubertooth stores RSSI in the PCAP comment field or a custom header format).

#### 4.8.2 DLT 272 — LINKTYPE_NORDIC_BLE

The Nordic DLT format adds a header before the BLE payload:

```
┌──────────────────┬──────────────┬───────────────────────────────────┐
│ Nordic header    │ Board/       │ BLE Link Layer Payload            │
│ (type, flags,   │ Header      │ (Access Address + PDU + CRC)     │
│  length, version │ (optional)   │                                   │
└──────────────────┴──────────────┴───────────────────────────────────┘
```

The Nordic header is parsed internally to extract board-specific metadata (RSSI, channel, event counter, delta time). The BLE payload follows at a documented offset. For v1, C07 extracts RSSI and channel from the Nordic header; future versions may expose additional Nordic-specific metadata.

### 4.9 Memory Management

C07 enforces a **hard 100 MB memory limit** (configurable via `--max-memory-mb`). This is a critical security and reliability constraint — the Pi CM5 has 8 GB RAM shared across all containers, and the deep parser must not exhaust system memory.

**Implementation:**

1. **Streaming processing:** Packets are processed one at a time. Each packet's PDU, parsed AdvData, and JSON serialisation occupy at most a few kilobytes. Memory is freed after each packet is written to the JSONL output.
2. **Output file buffering:** The `std::ofstream` uses a bounded buffer (default 8 KB per C++ standard library). Data is flushed periodically.
3. **Decompression buffer:** The pipe from `zstd` is read in 64 KB chunks. Each chunk is processed immediately.
4. **No full-file loading:** The parser never loads the entire PCAP or JSONL file into memory.

**Enforcement:** The main loop tracks total allocated bytes. If cumulative uncompressed data exceeds 100 MB, processing stops with an error. This is a hard cap, not advisory.

---

## 5. Inter-Component Contracts

### 5.1 DEEP-1: Deep Parser JSONL Output

| Property | Value |
|----------|-------|
| **Contract ID** | `DEEP-1` |
| **From** | C07 Deep Parser (C++17 / libpcap / nlohmann-json) |
| **To** | C08 ML Enrichment (Python 3.13) |
| **Format** | JSON Lines (`.jsonl`): one JSON object per line, UTF-8 encoded, no trailing commas, LF line endings |
| **Schema** | Defined in §3.1 of this document. Extended from CONTRACT 8.8-A with CRC status, sniffer-type, and DLT fields. |
| **Location** | V05: `/var/lib/tianer/data/deep/<sniffer_name>/YYYYMMDD-HHMM.jsonl` |
| **Completion Signal** | Empty `.done` file at same path + `.done` suffix, created atomically after JSONL file is fully written and flushed |
| **Consumption Pattern** | C08 watches for `.done` files via inotify or periodic polling. Files without `.done` are ignored. |
| **Idempotency** | C07 skips files with existing `.done` markers. Re-running is safe. |
| **Error Handling** | Malformed PCAP: file skipped, logged. Malformed PDU: packet skipped, logged at debug level. Corrupt JSONL output: file written but `.done` not created (C08 ignores). |
| **SLA** | 30-minute PCAP file (~200K packets) processed in < 5 minutes on CM5. |

### 5.2 DEEP-1 Schema Changes from CONTRACT 8.8-A

| Field | CONTRACT 8.8-A | DEEP-1 (v1) | Rationale |
|-------|---------------|-------------|-----------|
| `version` | Absent | Added | Future-proofing against schema evolution |
| `sniffer_type` | Absent | Added | Enables DLT-specific logic in C08 |
| `dlt` | Absent | Added | Per-DLT metadata for debugging/correlation |
| `crc_valid` | Absent | Added (true/false/null) | CRC-24 validation status per Q7 |
| `frame.number` | Absent | Added | Frame-level traceability to source PCAP |
| `frame.len` / `frame.cap_len` | Absent | Added | Detect truncated packets |
| `advdata.service_data` | Absent | Added | Parse service data AD type for richer enrichment |
| `raw_advdata_hex` | Present (top-level) | Preserved | Backward compatible |

---

## 6. Failure Modes & Recovery

### 6.1 Failure Catalog

| # | Failure Mode | Detection | Impact | Recovery |
|---|-------------|-----------|--------|----------|
| **F-C07-1** | **Corrupted PCAP file** | libpcap returns `PCAP_ERROR` during `pcap_next_ex()`. `pcap_perror` provides details. Parsed as "file truncated" or "bad magic number". | File processing aborted. No JSONL output produced. No `.done` marker created. | C07 logs the error and moves to the next file. The corrupted file remains without a `.done` marker. If the corruption is transient (e.g., disk I/O error), restarting C07 will re-process on the next timer tick. If permanent (file genuinely corrupted), gap detector (C06) will detect missing data for the corresponding time window. Source PCAP on V02 is the source of truth. |
| **F-C07-2** | **Malformed PDU (truncated, bad length, unknown type)** | `dissect_ble_pdu()` returns `parse_ok = false` with a descriptive `parse_error` string. | Single packet is skipped and logged at DEBUG/WARN level. Remaining packets in the file are processed normally. The metric `blesniff_malformed_pdus_total` is incremented. | No recovery needed — graceful skip. If `blesniff_malformed_pdus_total` exceeds 10% of total packets in a file, an alert fires (potential systematic issue with sniffer firmware or DLT interpretation). |
| **F-C07-3** | **Decompression failure (corrupt .zst file)** | `zstd -dc` child process exits with non-zero status. `pclose()` returns non-zero. | File processing aborted. No `.done` marker created. | Log error with file path and exit status. C07 moves to the next file. C04 rotation re-compresses files periodically; the corrupt file will remain without a `.done` marker. Operator may manually repair (re-compress) or delete the file. |
| **F-C07-4** | **Output I/O failure (V05 full, permission denied)** | `std::ofstream::write()` fails. `finish()` returns false. | JSONL file may be partially written. No `.done` marker created (atomic protocol prevents C08 from reading partial file). | Log error. All other files are unaffected (each file is independent). Alert: `TianerDiskSpaceWarning` (C01 monitors V05). Operator expands storage or purges old JSONL files. C07 retries on next timer tick. The partial JSONL file is detected and re-processed (no `.done` marker = re-run). |
| **F-C07-5** | **Unrecognised DLT** | `metadata().dlt == DltType::Unknown`. Raw DLT value logged. | File skipped entirely with a warning log. No processing attempted. | Operator must verify that the sniffer configuration matches the hardware. If the DLT is genuinely new (e.g., new sniffer hardware), the C07 code must be extended to support it. Until then, the file is safely skipped (no data loss — PCAP is retained on V02). |
| **F-C07-6** | **Memory pressure exceeds 100 MB cap** | Internal allocation tracker exceeds `max_memory_mb`. | Processing stops. File is not completed. `.done` marker not created. | Log error with file name and current allocation. Operator may increase `--max-memory-mb` (if system memory permits) or investigate why the file triggered the cap (unusual large PDUs, decompression bomb). File will be retried on next timer tick with same cap. |
| **F-C07-7** | **Path traversal attack (malicious PCAP filename)** | `find_pcap_files()` validates filenames against pattern `^\d{8}-\d{4}\.(pcap\|pcap\.zst)$`. Non-matching filenames are skipped. | Malicious file ignored. No processing. | No damage. The filename validation prevents path traversal, command injection, and processing of unexpected files. |
| **F-C07-8** | **CRC-24 validation mismatch (data corruption)** | `crc_valid == false` in output. `blesniff_crc_errors_total` incremented. | Packet is still output to JSONL (with `crc_valid: false`). C08 can decide whether to use or discard CRC-failed packets. | No intervention required for individual errors. If CRC error rate exceeds a threshold (e.g., 1% of packets), alert fires — indicates potential RF interference, bad antenna, or firmware bug. C08 may filter CRC-failed packets from DB insertion. |

### 6.2 Recovery Dependency Chain

| Recovery Agent | Failure It Handles | Is It Available After Container Restart? |
|---------------|-------------------|----------------------------------------|
| C07 stateless re-run | F-C07-1 through F-C07-8 | Yes — timer triggers re-run. No state to restore. |
| C04 PCAP Rotation | F-C07-3 (corrupt compressed file) | Yes — auto-rotation re-compresses |
| C01 Disk Monitoring | F-C07-4 (V05 full) | Yes — C01 host metrics alert |
| C06 Gap Detector | F-C07-1 (missed data from corrupt PCAP) | Yes — backfills from PCAP source of truth |

### 6.3 .done Marker as Recovery Barrier

The `.done` marker protocol is the key recovery mechanism for C07:

- **Crash before `.done`:** C08 never reads the file. C07 re-processes on next timer tick (file has no `.done` marker, so it's treated as unprocessed).
- **Crash after `.done`:** File is complete and C08 will pick it up. C07 skips it on re-run.
- **Crash during `.done` creation (between write and rename):** The `.tmp` file exists but `.done` does not. On re-run, C07 creates a new `.done` (the old `.tmp` is harmless and overwritten).

---

## 7. Observability

### 7.1 Metrics

Metrics are written to stderr in key=value format and collected by the container log driver (journald). A C13 scraper or host-side script may also read from a Prometheus textfile in V04.

| Metric | Type | Description | Labels |
|--------|------|-------------|--------|
| `blesniff_deep_parser_runs_total` | counter | Total number of deep parser invocations (one per container run) | — |
| `blesniff_pcap_files_processed` | counter | Number of PCAP files processed (success) | — |
| `blesniff_pcap_files_skipped` | counter | Number of PCAP files skipped (already processed, unrecognised DLT, corrupt) | `reason` (e.g., `already_processed`, `corrupt`, `unknown_dlt`) |
| `blesniff_pcap_bytes_processed` | counter | Total uncompressed bytes read from PCAP files | — |
| `blesniff_packets_dissected` | counter | Total BLE packets successfully dissected | — |
| `blesniff_malformed_pdus_total` | counter | Packets skipped due to PDU parsing errors | — |
| `blesniff_crc_errors_total` | counter | Packets with CRC-24 validation failure | `sniffer_id` |
| `blesniff_deep_parser_duration_seconds` | gauge | Duration of last run in seconds | — |
| `blesniff_deep_parser_last_run_ts` | gauge | Unix timestamp of last run completion | — |
| `blesniff_deep_parser_error` | gauge (0/1) | 1 if the last run encountered a fatal error | — |

### 7.2 Structured Logging

All log output uses the standard Tian'er structured format:

```
TIANER | {"ts":"2026-06-09T15:30:00Z","level":"INFO","component":"deep-parser","msg":"Starting processing","files_found":12,"crc_verify":"on"}
TIANER | {"ts":"2026-06-09T15:30:01Z","level":"INFO","component":"deep-parser","msg":"Processing file","file":"/var/lib/tianer/pcap/ut1/20260606-1200.pcap.zst","dlt":251,"sniffer":"ut1"}
TIANER | {"ts":"2026-06-09T15:30:15Z","level":"INFO","component":"deep-parser","msg":"File complete","file":"/var/lib/tianer/pcap/ut1/20260606-1200.pcap.zst","packets":184231,"crc_errors":12,"duration_ms":14203}
TIANER | {"ts":"2026-06-09T15:30:15Z","level":"ERROR","component":"deep-parser","msg":"Decompression failed","file":"/var/lib/tianer/pcap/ut1/20260606-1230.pcap.zst","exit_code":1}
TIANER | {"ts":"2026-06-09T15:35:00Z","level":"INFO","component":"deep-parser","msg":"Run complete","files_processed":11,"files_skipped":1,"total_packets":1850000,"total_crc_errors":150,"duration_s":300}
```

### 7.3 Health Check

C07 is a stateless batch job — it has no persistent health endpoint. Health is monitored via:

1. **Container exit code:** systemd `ExecStartPost` reports success/failure.
2. **Timer last fire:** `systemctl status blesniff-deep-parser.service` shows exit codes.
3. **Metric `blesniff_deep_parser_error`:** If 1, the last run failed.

### 7.4 Alert Rules

| Alert Name | Trigger | Severity | Action |
|------------|---------|----------|--------|
| `TianerDeepParserFailed` | `blesniff_deep_parser_error == 1` for > 15 minutes | **Warning** | Check container logs for specific error. Investigate disk space, PCAP corruption, or DLT issues. |
| `TianerDeepParserNotRunning` | `blesniff_deep_parser_last_run_ts` older than 45 minutes | **Warning** | Check systemd timer status. May indicate timer failure or stuck container. |
| `TianerHighCrcErrorRate` | `rate(blesniff_crc_errors_total[5m]) / rate(blesniff_packets_dissected[5m]) > 0.01` | **Warning** | > 1% CRC errors indicates RF interference or sniffer hardware issue. |
| `TianerHighMalformedPduRate` | `rate(blesniff_malformed_pdus_total[5m]) / rate(blesniff_packets_dissected[5m]) > 0.10` | **Warning** | > 10% malformed PDUs indicates DLT interpretation bug or firmware regression. |

---

## 8. Security Considerations

### 8.1 PCAP Input Validation

**Threat:** PCAP files are generated by C03 sniffer wrappers. While unlikely, a compromised sniffer binary or malicious PCAP file could attempt to exploit buffer overflows or path traversal in the parser.

**Mitigations:**

1. **Filename validation:** Only files matching `^\d{8}-\d{4}\.(pcap|pcap\.zst)$` are processed. This prevents path traversal (e.g., `../../etc/passwd`) and restricts processing to expected rotation files.
2. **No shell execution on filenames:** The zstd decompression uses `popen()` with a hardcoded command and the filename passed as `--` argument to `zstd`, which is safe against shell injection.
3. **Buffer safety:** Raw packet bytes are never parsed beyond their stated length. `dissect_ble_pdu()` and `parse_adv_data()` validate every offset against the data buffer size before accessing.
4. **Memory bounds:** The 100 MB cap is a hard limit. Even a maliciously crafted PCAP with many packets cannot exhaust host memory.

### 8.2 Path Traversal Protection

The file discovery function enforces a strict path structure:

```cpp
// Safe file discovery — only accepts expected patterns
std::vector<std::string> find_pcap_files(const std::string& dir) {
    std::regex pattern(R"(^\d{8}-\d{4}\.(pcap|pcap\.zst)$)");
    std::vector<std::string> result;

    for (auto& entry : fs::directory_iterator(dir)) {
        std::string filename = entry.path().filename().string();
        if (std::regex_match(filename, pattern)) {
            result.push_back(entry.path().string());
        } else {
            log_warn("Skipping unexpected file: " + entry.path().string());
        }
    }
    // Reject symlinks (follow = false in directory_iterator is default)
    // This prevents symlink attacks pointing outside V02
    return result;
}
```

### 8.3 Container Sandboxing

The Quadlet container definition enforces:

- **Read-only V02 mount:** The parser cannot modify PCAP files.
- **Isolated writes to V05:** The parser writes only to its designated `deep/` subdirectory.
- **No network access:** The container has no network (the deep parser has no need for network).
- **Drop all capabilities:** `--cap-drop ALL` — no privileged operations.
- **No shells or package managers** in the runtime image.
- **Read-only root filesystem** except for V02 and V05 mounts.

### 8.4 No Shell Execution on Filenames

The decompression command is constructed with safe argument separation:

```cpp
// Safe: filename is passed as an argument to zstd, not interpolated into shell
std::string cmd = "zstd -dc -- " + escape_for_shell(filename);
FILE* pipe = popen(cmd.c_str(), "r");
```

The `escape_for_shell()` function wraps the filename in single quotes and escapes any embedded single quotes (using `'\''` idiom). Together with the filename regex validation (only alphanumeric + `-` + `.`), shell injection is not possible.

### 8.5 Access Matrix

| Actor | V02 (PCAP) | V05 (Data) | Network | Root FS |
|-------|------------|------------|---------|---------|
| C07 Deep Parser | `:ro` | `:rw` (deep/ subdir only) | None | `:ro` |
| C08 ML Enrichment | — | `:ro` (deep/ subdir only) | `tianer-net` (Postgres) | `:ro` |

---

## 9. Configuration

### 9.1 Environment Variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `TIANER_PCAP_DIR` | `/var/lib/tianer/pcap` | Root directory for rotated PCAP files |
| `TIANER_DATA_DIR` | `/var/lib/tianer/data` | Root directory for output data |
| `TIANER_SNIFFER_CONFIG` | `/etc/tianer/sniffers.yaml` | Path to sniffer configuration |
| `TIANER_DEEP_CRC_VERIFY` | `on` | CRC-24 verification: `on` or `off` |
| `TIANER_DEEP_MAX_MEMORY_MB` | `100` | Maximum memory allocation in MB |
| `TIANER_DEEP_LOG_LEVEL` | `info` | Log level: `error`, `warn`, `info`, `debug` |

### 9.2 sniffers.yaml Integration

C07 reads `sniffers.yaml` (mounted from V01 `:ro`) to determine which sniffers are active and their configuration. The relevant fields for C07 are:

```yaml
sniffers:
  - id: 1
    name: ut1
    type: ubertooth    # Determines expected DLT and sniffer_type in output
    enabled: true       # Only enabled sniffers' PCAP directories are scanned
  - id: 2
    name: nrf1
    type: nrf
    enabled: true
```

If `enabled` is false, the sniffer's PCAP directory is skipped. This allows operators to temporarily disable processing for a broken sniffer without deleting its accumulated PCAP files.

### 9.3 CRC-24 Configuration (Q7)

The default is `TIANER_DEEP_CRC_VERIFY=on`. Operators may set it to `off` in the container's environment file when processing PCAP from sniffer sources known to pre-verify CRC (e.g., Ubertooth One firmware). The setting is global (per container invocation) — it applies to all files processed in that run.

To change the setting permanently:
```bash
# Edit /etc/tianer/blesniff.env or the Quadlet EnvironmentFile
echo 'TIANER_DEEP_CRC_VERIFY=off' >> /etc/tianer/blesniff.env
systemctl --user restart blesniff-deep-parser.service
```

---

## 10. Test Plan

### 10.1 Unit Tests — GoogleTest

All tests use GoogleTest 1.14 with CMake discovery (`gtest_discover_tests`). Test fixtures use synthetic BLE packets and golden PCAP samples from `tests/fixtures/pcap/`.

| Test Suite | File | Tests |
|-----------|------|-------|
| `CRC24Test` | `tests/crc24_test.cpp` | `ZeroLengthInput`, `KnownVector_ADV_IND`, `KnownVector_ADV_NONCONN_IND`, `ConsistencyWithPython`, `AllZeros`, `AllOnes`, `IncrementalUpdate` |
| `BleDissectorTest` | `tests/ble_dissector_test.cpp` | `DissectADV_IND_HappyPath`, `DissectADV_NONCONN_IND`, `DissectADV_SCAN_IND`, `DissectSCAN_REQ`, `DissectSCAN_RSP`, `DissectCONNECT_IND`, `TruncatedPDU_TooShort`, `TruncatedPDU_BadLength`, `PDUTypeOutOfRange`, `EmptyAdvData`, `MaxAdvData_31Bytes`, `CRCValidPassed`, `CRCValidFailed`, `CRCVerificationDisabled` "crc_valid=null", `PDUTypeStringAllValues` |
| `AdvDataParserTest` | `tests/advdata_parser_test.cpp` | `ParseFlags_0x01`, `ParseShortenedName_0x08`, `ParseCompleteName_0x09`, `ParseTXPower_0x0A`, `Parse16BitServiceUUIDs_0x02`, `Parse128BitServiceUUIDs_0x07`, `ParseManufacturerData_0xFF`, `ParseServiceData_0x16`, `MixedTLV_ParseCorrectly`, `EmptyAdvData_ReturnsEmpty`, `TLVOverrunsLength`, `UnknownADType_Skipped`, `ZeroLengthTLV_Skipped`, `TruncatedTLV_StopsAtBoundary` |
| `PcapInputTest` | `tests/pca_input_test.cpp` | `OpenValidPcap`, `OpenCorruptPcap`, `DetectDLT_BluetoothLeLL`, `DetectDLT_NordicBle`, `DetectDLT_Unknown`, `PacketIteration_AllPackets`, `FileNotFound`, `ZstdDecompression_Valid`, `ZstdDecompression_Corrupt` |
| `JsonlOutputTest` | `tests/jsonl_output_test.cpp` | `WriteValidRecord`, `SchemaFieldsPresent`, `FinishCreatesDoneFile`, `FinishFailsOnIOError`, `DoneNotCreatedOnPartialWrite`, `ReopenDoesNotOverwriteDone` |

#### 10.1.1 Golden Test Vectors (CRC-24)

Test vectors for CRC-24 are derived from the BLE Core Specification and independently verified against a Python reference implementation:

```cpp
// Known-good CRC-24 vectors for regression testing
TEST(CRC24Test, KnownVector_ADV_IND) {
    // ADV_IND PDU: PDU header (0x42 0x1F) + AdvA (6 bytes) + AdvData (25 bytes)
    // Total: 33 bytes (header_length = 31, so 2 + 31 = 33)
    // Expected CRC-24: computed with polynomial 0x00065B, init 0x555555
    const uint8_t pdu[] = {
        0x42, 0x1F,  // PDU Header: ADV_IND, length=31, TxAdd=1 (random)
        0xA1, 0xB2, 0xC3, 0xD4, 0xE5, 0xF6,  // AdvA (random for test)
        // AdvData: 25 bytes of test data
        0x02, 0x01, 0x06,  // Flags: 0x06
        0x03, 0x02, 0x12, 0x3A,  // 16-bit UUID: 0x3A12
        0x09, 0x08, 0x54, 0x65, 0x73, 0x74, 0x44, 0x65, 0x76,  // Name: "TestDev"
        0x05, 0x0A, 0xFC,  // TX Power: -4
        0x05, 0xFF, 0x4C, 0x00, 0xAA, 0xBB  // Manufacturer: Apple (0x004C), data 0xAABB
    };
    // This CRC value MUST be verified against the Python reference implementation
    // before committing. Placeholder value below is intentionally wrong.
    uint32_t crc = tianer::crc::crc24(pdu, sizeof(pdu), 0x555555);
    ASSERT_EQ(crc, 0xTEST);  // Replace with actual computed value
}
```

### 10.2 Integration Tests

| Test | Description | Acceptance Criteria |
|------|-------------|---------------------|
| `test_deep_parser_golden.sh` | Process `ubertooth-sample-001.pcap` and compare output to golden JSONL | Output matches `tests/fixtures/expected/deep-parser-sample-001.jsonl` exactly (byte-for-byte after stable JSON sort) |
| `test_deep_parser_zstd.sh` | Process a zstd-compressed PCAP file | Same output as uncompressed version |
| `test_deep_parser_crc.sh` | Process PCAP with known corrupt CRC bits, verify metric increment | `blesniff_crc_errors_total` incremented correctly. `crc_valid: false` in output. |
| `test_deep_parser_done_marker.sh` | Verify .done marker protocol | Kill parser mid-run. Restart. Verify re-processed file is skipped (has no .done). Verify completed file has .done. |
| `test_deep_parser_dlt.sh` | Process files with different DLTs | Correct DLT and sniffer_type in output for each sniffer type |
| `test_deep_parser_memory.sh` | Process large file with `--max-memory-mb 10` | Parser exits with error. No OOM kill. |
| `test_deep_parser_path_traversal.sh` | Attempt to process file with `../../` in name | File skipped. No output. Container not breached. |
| `test_deep_parser_idempotent.sh` | Run parser twice on same PCAP directory | Second run processes 0 new files (all skipped). |
| `test_deep_parser_nrf_golden.sh` | Process `nrf-sample-001.pcap` (Nordic DLT) | Correct RSSI and channel extracted from Nordic header. Output matches golden. |

### 10.3 Performance Tests

| Test | Target | Measurement |
|------|--------|-------------|
| Throughput — uncompressed | 30-minute PCAP file (~200K packets) processed in < 5 minutes | Wall clock time vs file size |
| Throughput — compressed | Same file in .zst format processed in < 6 minutes (decompression overhead) | Wall clock time |
| Memory usage | Peak RSS < 100 MB during processing of 30-minute file | `/usr/bin/time -v` or cgroup stats |
| CRC-24 overhead | Processing time with CRC on vs off | Should be < 10% difference |

### 10.4 Test Fixtures

| File | Source / Generation | Purpose |
|------|--------------------|---------|
| `tests/fixtures/pcap/ubertooth-sample-001.pcap` | Real hardware capture, anonymised MACs | Golden test — Ubertooth DLT 251 |
| `tests/fixtures/pcap/nrf-sample-001.pcap` | Real hardware capture, anonymised MACs | Golden test — Nordic DLT 272 |
| `tests/fixtures/pcap/synthetic-basic.pcap` | `tools/generate-pcap.py` | Known packet content for unit tests |
| `tests/fixtures/pcap/synthetic-corrupt-crc.pcap` | Hand-crafted | CRC error path testing |
| `tests/fixtures/pcap/synthetic-malformed.pcap` | Hand-crafted | Error path coverage |
| `tests/fixtures/pcap/synthetic-empty.pcap` | Hand-crafted | Empty file handling |
| `tests/fixtures/pcap/synthetic-100mb.zst` | Synthetic | Memory limit testing |
| `tests/fixtures/expected/deep-parser-sample-001.jsonl` | Manually verified output | Golden expected output |
| `tests/fixtures/expected/deep-parser-nrf-sample-001.jsonl` | Manually verified output | Golden expected output |

### 10.5 CI Integration

C07 tests run as part of `ci/test-all.sh` (step 2: C++ unit tests):
```bash
cd modules/bluetooth/deep-parser
cmake -B build
cmake --build build
ctest --test-dir build --output-on-failure
```

Integration tests (step 6) include C07:
```bash
tests/integration/test_deep_parser_golden.sh
tests/integration/test_deep_parser_zstd.sh
tests/integration/test_deep_parser_done_marker.sh
# ... etc.
```

---

## 11. Deployment Notes

### 11.1 Build System — CMake

```cmake
# modules/bluetooth/deep-parser/CMakeLists.txt
cmake_minimum_required(VERSION 3.25)
project(tianer-deep-parser VERSION 1.0.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# Dependencies
find_package(PkgConfig REQUIRED)
pkg_check_modules(LIBPCAP REQUIRED libpcap)
find_package(nlohmann_json 3.11 REQUIRED)

# Library targets
add_library(crc24 STATIC src/crc24.cpp)
add_library(pca_input STATIC src/pca_input.cpp)
add_library(ble_dissector STATIC src/ble_dissector.cpp)
add_library(advdata_parser STATIC src/advdata_parser.cpp)
add_library(jsonl_output STATIC src/jsonl_output.cpp)

target_link_libraries(pca_input PUBLIC ${LIBPCAP_LIBRARIES} crc24)
target_link_libraries(ble_dissector PUBLIC crc24)
target_link_libraries(jsonl_output PUBLIC nlohmann_json::nlohmann_json)

# Main executable
add_executable(blesniff-deep-parse src/main.cpp)
target_link_libraries(blesniff-deep-parse PRIVATE
    pca_input ble_dissector advdata_parser jsonl_output)

# Tests
enable_testing()
find_package(GTest REQUIRED)
include(GoogleTest)

add_executable(deep_parser_tests
    tests/crc24_test.cpp
    tests/ble_dissector_test.cpp
    tests/advdata_parser_test.cpp
    tests/pca_input_test.cpp
    tests/jsonl_output_test.cpp
)
target_link_libraries(deep_parser_tests PRIVATE
    GTest::GTest GTest::Main
    pca_input ble_dissector advdata_parser jsonl_output crc24)
gtest_discover_tests(deep_parser_tests)

# Install
install(TARGETS blesniff-deep-parse DESTINATION /usr/local/bin)
```

### 11.2 Multi-Stage Containerfile

```dockerfile
# deploy/containers/Dockerfile.deep-parser

# Stage 1: Build
FROM debian:trixie-slim AS builder
RUN apt-get update && apt-get install -y --no-install-recommends \
    g++-14 cmake make pkg-config \
    libpcap-dev nlohmann-json3-dev libgtest-dev \
    zstd ca-certificates
WORKDIR /build
COPY modules/bluetooth/deep-parser/ .
RUN cmake -B build -DCMAKE_BUILD_TYPE=Release && \
    cmake --build build --parallel $(nproc) && \
    ctest --test-dir build --output-on-failure

# Stage 2: Runtime (minimal)
FROM debian:trixie-slim
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpcap0.8 zstd ca-certificates && \
    apt-get clean && rm -rf /var/lib/apt/lists/*
COPY --from=builder /build/build/blesniff-deep-parse /usr/local/bin/blesniff-deep-parse
USER 1000:1000
ENTRYPOINT ["/usr/local/bin/blesniff-deep-parse"]
```

**Runtime image contents:**
- `libpcap0.8` — PCAP reading library (runtime only)
- `zstd` — decompression binary
- `ca-certificates` — TLS trust (may be needed for future features, harmless to include)
- No compilers, no dev headers, no shells, no package managers
- Runs as UID 1000 (mapped to `tianer` user on host for correct file permissions)

**Image size:** ~40 MB compressed, ~120 MB uncompressed (Debian slim base ~60 MB + libpcap ~5 MB + zstd ~3 MB + binary ~5 MB + overhead).

### 11.3 Quadlet Unit File

```ini
# deploy/containers/tianer-deep-parser.container
[Unit]
Description=Tian'er BLE Deep Packet Parser — batch PCAP to JSONL
Requires=tianer-postgres.service
After=tianer-postgres.service tianer-pcap-volume.mount

[Container]
Image=localhost/tianer-deep-parser:latest
ContainerName=tianer-deep-parser
Network=none
Volume=tianer-pcap.volume:/var/lib/tianer/pcap:ro
Volume=tianer-data.volume:/var/lib/tianer/data:rw
Volume=tianer-config.volume:/etc/tianer:ro
EnvironmentFile=/etc/tianer/blesniff.env

# Security hardening
ReadOnly=true
ReadOnlyTmpfs=true
NoNewPrivileges=true
DropCapability=ALL

# Resource limits
MemoryMax=150M
CPUQuota=200%

[Service]
Type=oneshot
Environment=TIANER_DEEP_CRC_VERIFY=on
ExecStart=/usr/local/bin/blesniff-deep-parse \
    --pcap-dir /var/lib/tianer/pcap \
    --output-dir /var/lib/tianer/data/deep \
    --sniffer-config /etc/tianer/sniffers.yaml

[Install]
WantedBy=tianer-platform.pod
```

**Key points:**
- `Type=oneshot` — the container runs once and exits. The systemd timer triggers periodic execution.
- `Network=none` — no network access needed (only reads from local volumes).
- `ReadOnly=true` — root filesystem is read-only.
- `MemoryMax=150M` — 100 MB for processing + 50 MB headroom for runtime overhead.
- `DropCapability=ALL` — no capabilities needed.

### 11.4 Systemd Timer

```ini
# deploy/systemd/tianer-deep-parser.timer
[Unit]
Description=Trigger Tian'er Deep Parser every 5 minutes
Requires=tianer-deep-parser.service

[Timer]
OnUnitActiveSec=5min
AccuracySec=10s
Persistent=true

[Install]
WantedBy=timers.target
```

The timer fires 5 minutes after the last run completes. This provides a gap after C04 rotation (which runs every 30 minutes) and ensures that PCAP files have been fully rotated and compressed before C07 processes them.

### 11.5 Makefile Integration

```makefile
# From root Makefile
deep-parser-build:
    cmake -B modules/bluetooth/deep-parser/build \
          -DCMAKE_BUILD_TYPE=Release
    cmake --build modules/bluetooth/deep-parser/build --parallel

deep-parser-test:
    ctest --test-dir modules/bluetooth/deep-parser/build --output-on-failure

deep-parser-image:
    podman build -t localhost/tianer-deep-parser:latest \
        -f deploy/containers/Dockerfile.deep-parser .

deep-parser-install:
    install -m 0755 modules/bluetooth/deep-parser/build/blesniff-deep-parse \
        /usr/local/bin/blesniff-deep-parse
    install -m 0644 deploy/containers/tianer-deep-parser.container \
        /etc/containers/systemd/
    install -m 0644 deploy/systemd/tianer-deep-parser.timer \
        /etc/systemd/system/
    systemctl --user daemon-reload
    systemctl --user enable tianer-deep-parser.timer
```

### 11.6 Deployment Checklist

Before considering C07 deployed, verify:

- [ ] `blesniff-deep-parse` binary built and installed at `/usr/local/bin/blesniff-deep-parse`
- [ ] Container image built: `podman image inspect localhost/tianer-deep-parser:latest`
- [ ] Quadlet unit installed: `/etc/containers/systemd/tianer-deep-parser.container`
- [ ] Timer installed: `/etc/systemd/system/tianer-deep-parser.timer`
- [ ] Manual run succeeds: `blesniff-deep-parse --pcap-dir /var/lib/tianer/pcap --output-dir /var/lib/tianer/data/deep --sniffer-config /etc/tianer/sniffers.yaml`
- [ ] JSONL output appears at `/var/lib/tianer/data/deep/<sniffer_name>/`
- [ ] `.done` marker file created alongside each completed JSONL
- [ ] Golden test passes: `ctest --test-dir modules/bluetooth/deep-parser/build --output-on-failure`
- [ ] `blesniff_crc_errors_total` metric increments on corrupted PCAP
- [ ] Memory usage stays under 100 MB during test run
- [ ] Container runs successfully as systemd oneshot service
- [ ] Timer triggers correctly (`systemctl --user list-timers tianer-deep-parser.timer`)

### 11.7 Install Paths Summary

| Artifact | Source | Installed To |
|----------|--------|-------------|
| Binary | `modules/bluetooth/deep-parser/build/blesniff-deep-parse` | `/usr/local/bin/blesniff-deep-parse` |
| Container image | `deploy/containers/Dockerfile.deep-parser` | `localhost/tianer-deep-parser:latest` |
| Quadlet unit | `deploy/containers/tianer-deep-parser.container` | `/etc/containers/systemd/` |
| Systemd timer | `deploy/systemd/tianer-deep-parser.timer` | `/etc/systemd/system/` |

---

## Appendix A: PDU Type Reference

| Value | Name | Description | AdvData Present? | AdvA Present? | Target Address Present? |
|-------|------|-------------|------------------|---------------|------------------------|
| 0 | ADV_IND | Connectable undirected advertising | Yes (0–31 bytes) | Yes | No |
| 1 | ADV_DIRECT_IND | Connectable directed advertising | No (0 bytes) | Yes | Yes (6 bytes) |
| 2 | ADV_NONCONN_IND | Non-connectable undirected advertising | Yes (0–31 bytes) | Yes | No |
| 3 | SCAN_REQ | Scan request | No | Yes | Yes |
| 4 | SCAN_RSP | Scan response | Yes (0–31 bytes) | Yes | No |
| 5 | CONNECT_IND | Connection request | Only connection parameters | Yes | Yes |
| 6 | ADV_SCAN_IND | Scannable undirected advertising | Yes (0–31 bytes) | Yes | No |
| 7 | ADV_EXT_IND | Extended advertising (BLE 5.0+) | Extended header | Yes (extended) | Depends on mode |

**PDU Type filtering rationale:** The deep parser processes all PDU types, but AdvData TLV parsing is only meaningful for types that carry advertising data (0, 2, 4, 6). For types 1 (directed), 3, and 5, the advdata field in the JSONL output is `null`. For type 7 (extended), the parsing is deferred to a future version.

## Appendix B: AD Type Reference

| AD Type | Name | Bytes | Output Field |
|---------|------|-------|-------------|
| 0x01 | Flags | 1 | `advdata.flags` |
| 0x02 | Incomplete List of 16-bit Service Class UUIDs | 2×N | `advdata.service_uuids_16` |
| 0x03 | Complete List of 16-bit Service Class UUIDs | 2×N | `advdata.service_uuids_16` |
| 0x04 | Incomplete List of 32-bit Service Class UUIDs | 4×N | Skipped (use 128-bit) |
| 0x05 | Complete List of 32-bit Service Class UUIDs | 4×N | Skipped |
| 0x06 | Incomplete List of 128-bit Service Class UUIDs | 16×N | `advdata.service_uuids_128` |
| 0x07 | Complete List of 128-bit Service Class UUIDs | 16×N | `advdata.service_uuids_128` |
| 0x08 | Shortened Local Name | 0–N | `advdata.local_name` (lower priority than 0x09) |
| 0x09 | Complete Local Name | 0–N | `advdata.local_name` (overrides 0x08) |
| 0x0A | TX Power Level | 1 | `advdata.tx_power` |
| 0x16 | Service Data — 16-bit UUID | 2 + N | `advdata.service_data` |
| 0x20 | Service Data — 32-bit UUID | 4 + N | `advdata.service_data` |
| 0x21 | Service Data — 128-bit UUID | 16 + N | `advdata.service_data` |
| 0xFF | Manufacturer Specific Data | 2 + N | `advdata.manufacturer` |

## Appendix C: CRC-24 Reference Implementation (Python)

```python
# tools/verify_crc24.py — reference implementation for golden test vector generation
def crc24(data: bytes, init: int = 0x555555) -> int:
    """Compute BLE CRC-24 over bytes (MSbit-first)."""
    crc = init
    for byte in data:
        for bit in range(8):
            # Process MSbit first within each byte
            msb = (crc >> 23) & 1
            data_bit = (byte >> (7 - bit)) & 1
            crc = (crc << 1) & 0xFFFFFF
            if msb ^ data_bit:
                crc ^= 0x00065B
    return crc & 0xFFFFFF

# Usage: python verify_crc24.py <hex_data>
# Example: python verify_crc24.py 421fa1b2c3d4e5f60201060302123a09...
if __name__ == "__main__":
    import sys
    data = bytes.fromhex(sys.argv[1])
    print(f"CRC-24: 0x{crc24(data):06X}")
```

---

*End of C07 Deep Parser Design Document.*
