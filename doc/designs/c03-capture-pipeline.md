# C03 — Capture Pipeline

**Status:** Phase A Design — guides Phase B implementation
**Author:** Code Architect
**Date:** 2026-06-09
**Dependencies:** C01 (Platform Infrastructure), hardware (Ubertooth One, nRF52840 dongles)
**Blocks:** C04 (PCAP Rotation), C05 (Ingest Bridge), C06 (Gap Detector)

---

## 1. Overview

### 1.1 Purpose

C03 Capture Pipeline is the **real-time BLE packet acquisition layer** of the Tian'er Signal Intelligence Platform. It captures raw Bluetooth Low Energy advertising traffic from physical sniffer hardware — one Ubertooth One and up to three nRF52840 dongles — writes the raw bitstream to persistent PCAP storage (V02), and streams a structured, normalized representation to the ingest bridge (C05) through a pair of kernel FIFOs per sniffer.

### 1.2 Scope

C03 covers the **capture-time data path** from USB device to ingest FIFO:

| In Scope | Out of Scope |
|----------|-------------|
| Sniffer wrapper scripts (ubertooth-wrap, nrf-wrap, tshark-wrap) | PCAP rotation (C04) |
| Decoupled architecture: sniffer → PCAP → tail-feed → FIFO → tshark → ingest FIFO | Parsing tshark fields into DB rows (C05) |
| Per-DLT tshark field parameterisation | Gap detection and backfill (C06) |
| tail-feed process (backpressure boundary) | Database schema (C02) |
| Heartbeat companion (local file primary, DB secondary) | Container image builds (C14) |
| FIFO naming, creation, and lifecycle | systemd unit wiring (C12) |
| BLE channel selection per sniffer (default ch37) | Deep parser BLE dissection (C07) |
| Quadlet container configuration for sniffer + tshark instances | |

### 1.3 Boundaries

C03 receives USB device access from C01 (udev symlinks, group membership, dumpcap capabilities). It writes PCAP to V02 (host bind-mount) and communicates with C05 exclusively through named FIFOs on V03 (tmpfs). The sniffer-to-PCAP path is decoupled from the downstream pipeline — the sniffer writes only to disk, never to a FIFO. A `tail-feed` process bridges the two halves and is the **exclusive backpressure boundary**: if the ingest bridge (C05) cannot keep up with the packet rate, `tail-feed` blocks on the FIFO write while the sniffer continues capturing to disk without interruption.

### 1.4 Position in the System

```
┌────────────────────────────────────────────────────────────────────────┐
│                         C01 PLATFORM HOST                               │
│  (udev symlinks, users, groups, dumpcap, tmpfiles, Podman rootless)    │
└──────┬───────┬────────────────────────────┬────────────────────────────┘
       │       │                            │
       ▼       ▼                            ▼
┌──────────┐ ┌───────────────────────┐ ┌────────────────────────┐
│  C02 DB  │ │   C03 CAPTURE PIPELINE │ │  C05 INGEST BRIDGE     │
│PostgreSQL│ │                        │ │  (reads ingest FIFOs)  │
└──────────┘ │  sniffer → PCAP (V02)  │ └────────────────────────┘
             │  tail-feed → FIFO (V03)│
             │  tshark → ingest FIFO  │
             └──────┬────────────────┘
                    │
                    ▼
             ┌────────────────────────┐
             │  C04 PCAP ROTATION      │
             │  (rotates V02 files)    │
             └────────────────────────┘
```

C03 is a **Layer 1 component** in the build sequence (component-breakdown.md §4.1). It depends on C01 being provisioned and requires physical sniffer hardware attached. It blocks C04 (rotation acts on C03's PCAP output), C05 (ingest reads C03's ingest FIFO), and C06 (gap detector monitors C03's heartbeats and reads C03's PCAP).

### 1.5 Design Decisions Incorporated

This document implements the following resolved decisions from ADR-0001:

| Decision | Reference | Implementation |
|----------|-----------|---------------|
| Decoupled capture: sniffer writes PCAP to disk, tail-feed feeds FIFO | storage-strategy.md | §2.1: sniffer→PCAP→tail-feed→FIFO architecture |
| PCAP is source of truth, FIFO is transient transport | storage-strategy.md | §2.2: backpressure isolates at tail-feed |
| Configurable channel per sniffer, default ch37 | D-05, Q6 | §9.2: `sniffers.yaml` channel field |
| Per-DLT tshark field parameterisation | D-12 | §4.3: tshark-wrap.sh normalised output |
| Heartbeat: local file primary (`/var/lib/tianer/heartbeat/<name>.ts`), DB secondary | D-11 | §4.4: heartbeat companion |
| nRF USB PID 1915:522A (sniffer) + 1915:520f (DFU) | D-03 | §2.3: per-DLT handling |
| Persistent udev device symlinks `/dev/tianer/*` | Q4 | §11.2: USB passthrough |
| BLE advertising channels 37/38/39 (2402/2426/2480 MHz) [1] | inception | §2.4: channel selection |
| Container orchestration with `--cap-drop ALL` except C03 | Q1, storage-strategy | §11.1: Quadlet unit files |

---

## 2. High-Level Architecture (HLA)

### 2.1 Decoupled Capture Data Flow

The capture pipeline is split into two halves connected by a tail-feed bridge. The sniffer half writes exclusively to disk. The processing half reads from disk and feeds the ingest pipeline through FIFOs.

```
                         BACKPRESSURE BOUNDARY
                         ┌──────────────────────────────────────────────┐
                         │                                              │
  ┌──────────────┐       │  ┌────────────┐    ┌──────────┐    ┌────────┐
  │ Sniffer      │       │  │ tail-feed  │    │ tshark   │    │ Ingest │
  │ Hardware     │───────┼─►│ process    │───►│ (per DLT │───►│ Bridge │
  │ (Ubertooth/  │ V02   │  │ tail -c +0  │ V03│ params)  │ V03│ (C05)  │
  │  nRF52840)   │ disk  │  │  -F pcap   │FIFO│          │FIFO│        │
  └──────────────┘       │  └────────────┘    └──────────┘    └────────┘
                         │       ▲                  ▲               │
                         │       │                  │               │
                         │  Sniffer continues      │               │
                         │  writing to disk        │               │
                         │  (no effect)       tshark blocks      ingest
                         │                   on full FIFO     blocks on DB
                         └──────────────────────────────────────────────┘
```

**Data path per sniffer instance:**

```
1. Sniffer wrapper  → /var/lib/tianer/pcap/<name>/capture.pcap    (V02, regular file)
2. tail-feed        → reads V02 PCAP, writes /var/run/tianer/<name>.fifo  (V03, named pipe)
3. tshark           → reads <name>.fifo, writes /var/run/tianer/<name>-ingest.fifo  (V03)
4. Ingest bridge    → reads <name>-ingest.fifo, batches to PostgreSQL (V06)
```

### 2.2 Backpressure Isolation

The tail-feed process is the **single point of backpressure** for the entire capture pipeline. The design enforces the following invariants:

1. The sniffer wrapper **never** writes to a FIFO. Its output target is always a regular file on V02. This eliminates the risk of a blocked pipe causing the sniffer to drop packets or crash.

2. The tail-feed process reads from a growing PCAP file using `tail -c +0 -F`. When it writes to `<name>.fifo` and the downstream reader (tshark) is slow, the pipe buffer fills (default 64 KB on Linux), and `tail` blocks on the write end.

3. tshark reads from `<name>.fifo` and writes to `<name>-ingest.fifo`. If the ingest bridge (C05) is blocked (e.g., database outage), tshark blocks on the ingest FIFO write, which in turn causes tshark to stop reading from `<name>.fifo`, cascading backpressure to tail-feed.

4. The sniffer is **completely unaffected** by backpressure at any level. It continues writing to disk for the entire duration of the downstream blockage. The PCAP archive (source of truth) loses zero packets.

**Backpressure cascade sequence during a database outage:**

```
t₀: DB goes down
t₀+δ₁: Ingest bridge blocks on DB reconnect → stops reading <name>-ingest.fifo
t₀+δ₂: Ingest FIFO pipe fills (64 KB)        → tshark blocks on write to ingest FIFO
t₀+δ₃: tshark stops reading <name>.fifo      → PCAP FIFO pipe fills (64 KB)
t₀+δ₄: tail-feed blocks on write to <name>.fifo → no further data flows through FIFOs
t₀+∞:  Sniffer continues writing PCAP to V02  → ZERO data loss from source of truth
```

When the database recovers, the ingest bridge resumes reading, backpressure clears through the pipeline, and the gap detector (C06) identifies the outage window from heartbeat gaps and backfills from PCAP.

### 2.3 Per-DLT Handling

Capture pipeline instances are parameterised at launch by the sniffer's **Data Link Type (DLT)**. Different sniffer hardware produces PCAP with different link-layer headers, exposing different tshark protocol fields.

| Sniffer Type | DLT | DLT Name | tshark Protocol Namespace | Hardware |
|-------------|-----|----------|--------------------------|----------|
| Ubertooth One | 256 | `LINKTYPE_BLUETOOTH_LE_LL_WITH_PHDR` | `btle_rf.*`, `btle_le.*` | LPC175x + CC2400 |
| nRF52840 Sniffer | 272 | `LINKTYPE_NORDIC_BLE` | `nordic_ble.*` | nRF52840 ARM Cortex-M4 |

DLT values are assigned by tcpdump.org [2].

**DLT 256 (Ubertooth) characteristics:**
- Includes RF physical-layer metadata: channel, signal power, noise power, reference access address, CRC status, error count
- Link-layer fields: access address, PDU header, payload, CRC
- The `btle_rf` header provides per-packet signal quality that the nRF sniffer does not expose in its PCAP output

**DLT 272 (nRF) characteristics:**
- Includes board metadata: board ID, packet counter, timestamp
- RF metadata: RSSI (Received Signal Strength Indicator), channel
- Link-layer fields: access address, PDU header, payload, CRC
- The nRF sniffer firmware may or may not verify CRC at capture time; CRC-24 validation is deferred to the Deep Parser (C07, per Q7)

**Normalisation strategy:** The `tshark-wrap.sh` script (§4.3) maps per-DLT field names to a **single normalised pipe-delimited schema** consumed by the ingest bridge. The ingest bridge (C05) never deals with DLT-specific field names — it receives a consistent set of fields regardless of the source hardware.

### 2.4 BLE Channel Selection

BLE advertising uses three physical channels in the 2.4 GHz ISM band [1]:

| BLE Channel | Frequency | Purpose |
|-------------|-----------|---------|
| 37 | 2402 MHz | Primary advertising |
| 38 | 2426 MHz | Primary advertising |
| 39 | 2480 MHz | Primary advertising |

Ubertooth One is a **single-channel-at-a-time** device. It can monitor exactly one channel per capture session. The default channel is **37** (2402 MHz) per D-05 and Q6. The channel is configurable per sniffer in `sniffers.yaml`.

The nRF52840 Sniffer firmware supports **channel hopping** across all three advertising channels [4]. When an nRF dongle is configured with multiple channels in `sniffers.yaml`, the sniffer firmware hops between them, capturing packets from all three.

**Channel coverage strategy:** With one Ubertooth (single channel, default ch37) and one nRF dongle (hopping ch37/38/39), the platform achieves coverage of all three advertising channels simultaneously. Deduplication of packets seen on multiple channels is handled at query time in the database (D-05, D-10), not at capture time.

### 2.5 Container Pod Topology

All sniffer and tshark processes run within the **`tianer-capture` pod** with `Network=none`. This pod has no network interface — all communication between containers within the pod is through bind-mounted FIFOs (V03) and shared filesystem paths (V02).

```
┌─────────────────────────────────────────────────────────────┐
│              tianer-capture pod (Network=none)                │
│                                                               │
│  ┌─────────────────────┐  ┌─────────────────────┐            │
│  │ blesniff-sniffer@ut1│  │ blesniff-sniffer@nrf1│  ...      │
│  │                     │  │                     │            │
│  │ ubertooth-wrap.sh   │  │ nrf-wrap.sh         │            │
│  │   → capture.pcap    │  │   → capture.pcap    │            │
│  │ tail-feed           │  │ tail-feed           │            │
│  │   → ut1.fifo        │  │   → nrf1.fifo       │            │
│  │ heartbeat.sh        │  │ heartbeat.sh        │            │
│  └─────────────────────┘  └─────────────────────┘            │
│                                                               │
│  ┌─────────────────────┐  ┌─────────────────────┐            │
│  │ blesniff-tshark@ut1 │  │ blesniff-tshark@nrf1│  ...      │
│  │                     │  │                     │            │
│  │ tshark-wrap.sh      │  │ tshark-wrap.sh      │            │
│  │   r ut1.fifo        │  │   r nrf1.fifo       │            │
│  │   w ut1-ingest.fifo │  │   w nrf1-ingest.fifo│            │
│  └─────────────────────┘  └─────────────────────┘            │
│                                                               │
│  Volumes:                                                     │
│    V02 /var/lib/tianer/pcap/    :rw  (PCAP write)            │
│    V03 /var/run/tianer/         :rw  (FIFOs)                 │
│    V01 /etc/tianer/             :ro  (sniffers.yaml config)  │
│    V04 /var/log/tianer/         :rw  (structured logs)       │
│    V08 (tmpfs) /tmp             :rw  (per-container ephemeral)│
│                                                               │
│  USB passthrough:                                            │
│    --device /dev/tianer/ubertooth0                           │
│    --device /dev/tianer/nrf0                                 │
│    --device /dev/tianer/nrf1                                 │
│    --device /dev/tianer/nrf2                                 │
│    --group-add keep-groups                                   │
│                                                               │
│  Capabilities:                                                │
│    --cap-drop ALL                                            │
│    --cap-add CAP_NET_RAW,CAP_NET_ADMIN                       │
└─────────────────────────────────────────────────────────────┘
```

### 2.6 Interaction with Neighbouring Components

| Direction | Component | Mechanism | Protocol |
|-----------|-----------|-----------|----------|
| **Upstream** | C01 Platform | udev symlinks, group membership, filesystem layout, tmpfiles FIFOs | Host provisioning |
| **Upstream** | Hardware | USB bus, libusb / CDC ACM serial | USB 2.0 |
| **Internal** | tail-feed → tshark | Named FIFO (V03) | Binary PCAP stream |
| **Internal** | tshark → ingest FIFO | Named FIFO (V03) | Pipe-delimited text lines |
| **Downstream** | C05 Ingest Bridge | Named FIFO (V03) | Pipe-delimited text lines (CAPTURE-2) |
| **Downstream** | C04 PCAP Rotation | File rename + signal | SIGHUP to sniffer wrapper (ROTATION-1 coordination) |
| **Downstream** | C06 Gap Detector | Heartbeat files + PCAP files | Local file read (V02, :ro) |
| **Lateral** | C02 Database | DB heartbeat table | SQL INSERT (CAPTURE-3, secondary path) |

---

## 3. Data Model

### 3.1 Sniffer Configuration (`sniffers.yaml`)

The sniffer configuration file defines all sniffer instances. It is read at startup by each sniffer wrapper to determine its parameters.

**Schema:**

```yaml
# /etc/tianer/sniffers.yaml
# Read :ro by all sniffer containers (V01 mount)

sniffers:
  - id: 1
    name: ut1                  # Unique per-sniffer identifier
    type: ubertooth            # ubertooth | nrf
    device: /dev/tianer/ubertooth0   # USB device symlink (from C01 udev)
    mode: btle                 # btle (BLE advertising) | rx (BR/EDR survey, Ubertooth only)
    channels: [37]             # BLE channel(s). Single channel for Ubertooth; [37,38,39] for nRF.
    enabled: true              # Whether this sniffer starts at boot

  - id: 2
    name: nrf1
    type: nrf
    device: /dev/tianer/nrf0
    mode: ble
    channels: [37, 38, 39]
    enabled: true

  - id: 3
    name: nrf2
    type: nrf
    device: /dev/tianer/nrf1
    mode: ble
    channels: [37, 38, 39]
    enabled: false             # Disabled by default — enable when hardware present

  - id: 4
    name: nrf3
    type: nrf
    device: /dev/tianer/nrf2
    mode: ble
    channels: [37, 38, 39]
    enabled: false
```

**Field constraints:**

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `id` | integer | Yes | — | Unique numeric identifier for this sniffer. Maps to `sniffer_id` in DB. |
| `name` | string | Yes | — | Short identifier used in FIFO names, file paths, and metrics labels. Must match `[a-z][a-z0-9]*`. |
| `type` | enum | Yes | — | `ubertooth` or `nrf`. Determines wrapper script and DLT. |
| `device` | path | Yes | — | Absolute path to USB device symlink under `/dev/tianer/`. |
| `mode` | enum | No | `ble` | Capture mode: `ble` or `btle` (BLE advertising) for Ubertooth; `ble` only for nRF. |
| `channels` | list[int] | Yes | `[37]` | BLE advertising channel(s) to monitor. Ubertooth accepts exactly 1 channel. nRF accepts 1–3. |
| `enabled` | bool | No | `false` | If `false`, the sniffer instance is skipped at startup. |

### 3.2 Heartbeat File Schema

Each sniffer instance writes a local heartbeat timestamp file every 30 seconds. The heartbeat file is the **primary** liveness signal (D-11). The database heartbeat table (`bluetooth.sniffer_heartbeat`, see C02) is a **secondary** path, backfilled from the local file on recovery.

**File path:** `/var/lib/tianer/heartbeat/<sniffer_name>.ts`

**File content:** A single line containing the Unix epoch timestamp (integer seconds) of the last successful write.

**File format:**
```
1749472501
```

**Heartbeat lifecycle:**

1. **Creation:** Heartbeat script creates the file on first tick if it does not exist.
2. **Update:** Every 30 seconds (±2 seconds jitter), the script opens the file for writing, truncates it, and writes the current `time(NULL)` value.
3. **Atomicity:** Write is atomic at the filesystem level because the content is a single line smaller than the page size (4 KB). No partial writes are possible.
4. **Absence:** If the heartbeat file does not exist at startup, the gap detector treats the sniffer as offline from time zero. The heartbeat script creates the file on its first tick.
5. **Staleness:** The gap detector (C06) alerts when `(current_time - heartbeat_ts) > 60 seconds` (allowing for one missed update plus network/load jitter).

**Secondary path — DB heartbeat table:** The heartbeat script also attempts to write to `bluetooth.sniffer_heartbeat` every 30 seconds. On database outage, the DB write fails silently (log WARN, do not retry synchronously). The local file persists as the authoritative source. On database recovery, the gap detector backfills the heartbeat table from the local file timestamps.

### 3.3 FIFO Naming Convention

Each sniffer instance uses two named FIFOs on V03. The naming convention embeds the sniffer's `name` field:

| FIFO | Path Pattern | Example | Direction | Content |
|------|-------------|---------|-----------|---------|
| PCAP stream FIFO | `/var/run/tianer/<name>.fifo` | `/var/run/tianer/ut1.fifo` | tail-feed → tshark | Raw binary PCAP stream |
| Ingest FIFO | `/var/run/tianer/<name>-ingest.fifo` | `/var/run/tianer/ut1-ingest.fifo` | tshark → ingest bridge | Pipe-delimited normalised text lines |

**FIFO properties:**
- Type: Named pipe (S_IFIFO, mode 0660, owner `tianer:tianer`)
- Created by: `systemd-tmpfiles` at boot (see §11.4), recreated after host reboot
- Buffer size: Linux default 64 KB (controlled by `fs.pipe-max-size` sysctl; default is adequate)
- V03 is a tmpfs mount (`/var/run/tianer/`), so FIFOs exist in RAM and are destroyed on host reboot
- tshark opens the PCAP stream FIFO for reading; the ingest bridge opens the ingest FIFO for reading

### 3.4 PCAP File Layout

Each sniffer writes to a dedicated subdirectory under V02:

```
/var/lib/tianer/pcap/
├── ut1/
│   ├── capture.pcap              # Current active file (being written by sniffer)
│   ├── 20260609-1400.pcap        # Rotated archive (C04)
│   ├── 20260609-1430.pcap.zst    # Rotated + compressed (C04)
│   └── ...
├── nrf1/
│   ├── capture.pcap
│   └── ...
├── nrf2/
│   └── ...
└── nrf3/
    └── ...
```

**Current file naming:** The sniffer always writes to `capture.pcap` in its directory. C04 rotation renames this file to a timestamped name and the sniffer wrapper creates a new `capture.pcap` (see §3.5 for rotation coordination).

**DLT embedded in PCAP:** The Global Header of each PCAP file records the DLT value (256 for Ubertooth, 272 for nRF). This is set by the sniffer tool and verified at startup.

### 3.5 Rotation Coordination with C04

C03 and C04 share V02 and must coordinate file lifecycle:

1. **Normal operation:** C03 sniffer wrapper writes to `capture.pcap`. C04 rotation timer fires every `BLESNIFF_ROTATION_MINUTES`.
2. **Rotation trigger:** C04 sends `SIGHUP` to the sniffer wrapper process. (Per `ROTATION-1` contract in C04.)
3. **Sniffer wrapper response:** On SIGHUP, the wrapper (a) closes the current `capture.pcap` file descriptor, (b) C04 renames the old file to `YYYYMMDD-HHMM.pcap`, (c) the sniffer opens a new `capture.pcap` with a fresh PCAP global header.
4. **tail-feed response:** tail-feed uses `tail -F` (capital F), which detects the rename and automatically follows the new `capture.pcap` file.
5. **tshark continuity:** tshark reads from a FIFO, not a file — it is unaware of rotation. The tail-feed process handles the transition. There may be a brief pause (< 1 second) while the sniffer closes/reopens, but the PCAP stream is continuous across the boundary.

---

## 4. Low-Level Architecture (LLA)

### 4.1 Sniffer Wrapper Scripts

Each sniffer type has a dedicated wrapper script that handles hardware initialisation, channel configuration, and PCAP output. Both wrappers follow the same contract: read configuration from environment and `sniffers.yaml`, write raw PCAP to a named file on V02, and exit cleanly on SIGTERM/SIGHUP.

#### 4.1.1 ubertooth-wrap.sh

```bash
#!/usr/bin/env bash
# ubertooth-wrap.sh — Capture BLE advertising from Ubertooth One (DLT 256)
# Invoked by Quadlet container. Reads config from environment.

set -euo pipefail

SNIFFER_NAME="${BLESNIFF_SNIFFER_NAME:?BLESNIFF_SNIFFER_NAME not set}"
SNIFFER_DEVICE="${BLESNIFF_SNIFFER_DEVICE:?BLESNIFF_SNIFFER_DEVICE not set}"
CHANNEL="${BLESNIFF_CHANNEL:-37}"
PCAP_DIR="${BLESNIFF_PCAP_DIR:-/var/lib/tianer/pcap}/${SNIFFER_NAME}"
PCAP_FILE="${PCAP_DIR}/capture.pcap"

# Ensure output directory exists
mkdir -p "${PCAP_DIR}"

# Capture BLE advertising on specified channel.
# ubertooth-btle -A 37 captures advertising on channel 37 [5].
# Output is PCAP to stdout; redirected to file.
# The -c flag specifies the capture file directly in some versions.
exec ubertooth-btle -U "${SNIFFER_DEVICE}" -c "${PCAP_FILE}" -A "${CHANNEL}"
```

**Key behaviours:**
- `ubertooth-btle` writes PCAP with DLT 256 directly to the specified file.
- Channel is passed as a command-line argument from `sniffers.yaml`.
- On SIGTERM/SIGHUP, ubertooth-btle exits cleanly. The wrapper script traps the signal, ensuring a clean PCAP file closure.
- If the device is not present, `ubertooth-btle` exits with an error; Podman restarts the container per `Restart=on-failure`.

#### 4.1.2 nrf-wrap.sh

```bash
#!/usr/bin/env bash
# nrf-wrap.sh — Capture BLE from nRF52840 Sniffer (DLT 272)
# Invoked by Quadlet container. Reads config from environment.

set -euo pipefail

SNIFFER_NAME="${BLESNIFF_SNIFFER_NAME:?BLESNIFF_SNIFFER_NAME not set}"
SNIFFER_DEVICE="${BLESNIFF_SNIFFER_DEVICE:?BLESNIFF_SNIFFER_DEVICE not set}"
CHANNELS="${BLESNIFF_CHANNELS:-37}"   # Comma-separated: 37,38,39
PCAP_DIR="${BLESNIFF_PCAP_DIR:-/var/lib/tianer/pcap}/${SNIFFER_NAME}"
PCAP_FILE="${PCAP_DIR}/capture.pcap"

mkdir -p "${PCAP_DIR}"

# nRF Sniffer uses nrfutil (Nordic command-line tool) [4].
# The sniffer can hop across multiple BLE advertising channels.
# Output is PCAP to specified file.
exec nrfutil ble-sniffer \
    --device "${SNIFFER_DEVICE}" \
    --channels "${CHANNELS}" \
    --output "${PCAP_FILE}"
```

**Key behaviours:**
- `nrfutil ble-sniffer` writes PCAP with DLT 272.
- Channels are comma-separated; the firmware hops between them.
- PID detection: `nrfutil` auto-detects whether the dongle is in sniffer mode (PID 522A) or DFU mode (PID 520f). If in DFU mode, the wrapper logs a warning and exits with a non-zero code — the operator must flash the sniffer firmware.

### 4.2 tail-feed Process

The tail-feed process bridges the gap between disk-based PCAP capture and FIFO-based streaming. It runs **in the same container** as the sniffer wrapper, started as a background process immediately after the sniffer begins writing.

**Process model:**

```bash
#!/usr/bin/env bash
# tail-feed.sh — Feed growing PCAP file into a named FIFO.
# This is the backpressure boundary. Blocked FIFO = sniffer unaffected.

PCAP_FILE="${1:?PCAP_FILE required}"
FIFO_PATH="${2:?FIFO_PATH required}"

# Wait for the PCAP file to exist (sniffer may take a moment to create it)
while [[ ! -f "${PCAP_FILE}" ]]; do
    sleep 0.1
done

# tail -c +0: start from beginning of file (byte 0), not last 10 lines
# tail -F: follow by filename, not inode — tracks rotation (reopens on rename)
# Output redirected to FIFO.
exec tail -c +0 -F "${PCAP_FILE}" > "${FIFO_PATH}"
```

**Key behaviours:**
- `tail -c +0`: Read from the first byte of the file (standard `tail -f` starts from the last 10 lines, which would skip the PCAP global header).
- `tail -F` (capital F): Follows the file by **name**, not inode. When C04 rotates the file (rename + create new), `tail -F` detects the rename, reopens the new file, and continues streaming. This is essential for rotation coordination (§3.5).
- Blocking behaviour: If the FIFO is full, `tail` blocks on the write syscall. No CPU is consumed during the block. The sniffer continues writing to disk, completely unaffected.
- Process lifecycle: tail-feed is started as a child of the sniffer wrapper. When the wrapper exits (SIGTERM, crash), the tail-feed is terminated by the container runtime.

**Startup ordering within the sniffer container:**
1. sniffer wrapper starts, opens PCAP file for writing
2. tail-feed starts (background), waits for PCAP file to exist, opens for reading
3. heartbeat companion starts (background), begins 30-second tick

### 4.3 tshark-wrap.sh — Per-DLT Parameterisation

The tshark wrapper reads raw PCAP from a FIFO and produces normalised pipe-delimited output to the ingest FIFO. It is **parameterised at launch** with the sniffer's DLT type so that it uses the correct tshark field names.

#### 4.3.1 Normalised Output Schema (CAPTURE-2 / INGEST-1)

Regardless of source DLT, tshark-wrap produces output lines in this format after post-processing (see §4.3.2 for per-DLT field extraction and post-processing pipeline):

```
<epoch_seconds>.<microseconds>|<mac_address>|<addr_type>|<rssi>|<channel>|<pdu_type>|<advdata_hex>
```

This 7-field normalized schema matches the INGEST-1 contract consumed by the ingest bridge (C05 §5.1). The sniffer name is not included in the line — it is injected by the ingest bridge's `PgWriter` constructor as the `sniffer_id` per C05 §3.2.

**Fields:**

| Position | Name | Type | Description |
|----------|------|------|-------------|
| 1 | `ts` | float | Frame arrival timestamp as epoch seconds with microsecond precision (from `frame.time_epoch`) |
| 2 | `mac_address` | colon-hex string (17 chars) | BLE advertiser MAC address (e.g., `aa:bb:cc:dd:ee:ff`). Empty for non-ADV packets. |
| 3 | `addr_type` | `0` or `1` | Address type extracted from the PDU header TxAdd bit: `0` = public, `1` = random. Empty if not determinable. |
| 4 | `rssi` | int | Received Signal Strength Indicator in dBm. Empty if not reported. |
| 5 | `channel` | int | BLE advertising channel (37, 38, or 39). Empty if not reported. |
| 6 | `pdu_type` | int | BLE PDU type extracted from the PDU header low nibble: 0=ADV_IND, 1=ADV_DIRECT_IND, 2=ADV_NONCONN_IND, 3=SCAN_REQ, 4=SCAN_RSP, 5=CONNECT_IND, 6=ADV_SCAN_IND. Empty if not determinable. |
| 7 | `advdata_hex` | hex string (even length) | Raw advertising data payload, hex-encoded. Up to 62 hex chars (31 bytes) for legacy BLE advertising. Empty string if none. |

**Example output line (Ubertooth, DLT 256):**
```
1749472501.234567|aa:bb:cc:dd:ee:ff|0|-48|37|0|0201060303e1ff0c0948656c6c6f576f726c64
```

**Example output line (nRF, DLT 272):**
```
1749472501.345678|aa:bb:cc:dd:ee:ff|1|-52|38|0|0201060303e1ff0c0948656c6c6f576f726c64
```

**Empty-field convention:** When a field is not available (e.g., `rssi` for a non-ADV packet or `mac_address` missing), it appears as an empty string between pipes: `||`. All fields except `ts` allow empty. This convention matches the INGEST-1 contract (C05 §5.1).

#### 4.3.2 Per-DLT tshark Invocation with Post-Processing

tshark is configured per DLT to extract raw fields, which are then post-processed into the 7-field normalised schema (§4.3.1). The post-processing extracts `addr_type` and `pdu_type` from the PDU header byte, converts the access address to colon-hex MAC format, and discards fields not needed by the ingest bridge (payload length, CRC, sniffer name).

```bash
#!/usr/bin/env bash
# tshark-wrap.sh — Parameterised tshark reader with post-processing.
# Invoked with BLESNIFF_DLT and BLESNIFF_SNIFFER_NAME from environment.
#
# Output: 7-field pipe-delimited normalised schema per CAPTURE-2 / INGEST-1:
#   ts|mac_address|addr_type|rssi|channel|pdu_type|advdata_hex

set -euo pipefail

DLT="${BLESNIFF_DLT:?BLESNIFF_DLT not set}"
SNIFFER_NAME="${BLESNIFF_SNIFFER_NAME:?BLESNIFF_SNIFFER_NAME not set}"
PCAP_FIFO="/var/run/tianer/${SNIFFER_NAME}.fifo"
INGEST_FIFO="/var/run/tianer/${SNIFFER_NAME}-ingest.fifo"

# tshark field expressions vary by DLT.
if [[ "${DLT}" == "256" ]]; then
    # Ubertooth One — DLT 256: LINKTYPE_BLUETOOTH_LE_LL_WITH_PHDR [2]
    TSHARK_FIELDS=(
        -e frame.time_epoch
        -e btle_le.advertising_address
        -e btle_le.advertising_header
        -e btle_rf.signal_power
        -e btle_rf.channel
        -e btle_le.data
    )
elif [[ "${DLT}" == "272" ]]; then
    # nRF52840 Sniffer — DLT 272: LINKTYPE_NORDIC_BLE [2]
    TSHARK_FIELDS=(
        -e frame.time_epoch
        -e nordic_ble.advertiser_address
        -e nordic_ble.packet_header
        -e nordic_ble.rssi
        -e nordic_ble.channel
        -e nordic_ble.payload
    )
else
    echo "TIANER | {\"ts\":\"$(date -Iseconds)\",\"level\":\"FATAL\",\"component\":\"tshark\",\"msg\":\"Unsupported DLT\",\"dlt\":\"${DLT}\"}" >&2
    exit 1
fi

# Run tshark reading from the PCAP FIFO, piping through post-processing.
# -i -: read from stdin (FIFO)
# -l: flush output after each packet (critical for real-time pipeline)
# -T fields: output only the requested fields [3]
# -E separator=\|: pipe-delimited output
# -E quote=n: no quoting of field values
# -E occurrence=f: for multi-occurrence fields, use the first occurrence
tshark \
    -i - \
    -l \
    -T fields \
    -E separator=\| \
    -E quote=n \
    -E occurrence=f \
    "${TSHARK_FIELDS[@]}" \
    < "${PCAP_FIFO}" \
    | awk -F'|' '
    # Post-processing pipeline: convert raw tshark fields to CAPTURE-2 7-field schema.
    # Input:   ts|address_raw|header_byte|rssi|channel|advdata_hex
    # Output:  ts|mac_address|addr_type|rssi|channel|pdu_type|advdata_hex
    {
        ts         = $1   # epoch seconds with microsecond precision
        addr_raw   = $2   # raw address (e.g., "aa:bb:cc:dd:ee:ff" or hex string without colons)
        header_hex = $3   # PDU header as hex byte (2 hex chars)
        rssi       = $4
        channel    = $5
        advdata    = $6   # hex-encoded advertising payload

        # Convert address to canonical colon-hex if needed.
        # If already colon-separated, keep as-is. Otherwise insert colons.
        mac_address = addr_raw
        if (addr_raw !~ /:/ && length(addr_raw) >= 12) {
            mac_address = substr(addr_raw,1,2) ":" substr(addr_raw,3,2) ":" \
                          substr(addr_raw,5,2) ":" substr(addr_raw,7,2) ":" \
                          substr(addr_raw,9,2) ":" substr(addr_raw,11,2)
        }

        # Extract addr_type and pdu_type from the PDU header byte.
        # The advertising channel PDU header byte 0 (BLE Core Spec Vol 6, Part B, §2.3):
        #   Bits [3:0]: PDU Type (low nibble)
        #   Bit  [4]:   RFU (reserved)
        #   Bit  [5]:   ChSel (channel selection)
        #   Bit  [6]:   TxAdd (0=public, 1=random address)
        #   Bit  [7]:   RxAdd (target address type)
        # tshark outputs the header as a hex byte in network order.
        # The hex byte's bit positions correspond directly: PDU type is the
        # low nibble (0x0F mask), TxAdd is bit 6 (0x40 mask).
        addr_type = ""
        pdu_type  = ""
        if (length(header_hex) >= 2) {
            cmd = "printf \"%d\" 0x" header_hex
            cmd | getline header_val
            close(cmd)
            addr_type = and(header_val, 0x40) ? "1" : "0"   # Bit 6 = TxAdd
            pdu_type  = and(header_val, 0x0F)                # Low nibble = PDU type
        }

        printf "%s|%s|%s|%s|%s|%s|%s\n", ts, mac_address, addr_type, rssi, channel, pdu_type, advdata
    }' \
    > "${INGEST_FIFO}"
```

**PDU Header Bit Extraction Details:**

The BLE advertising channel PDU header is 2 bytes (16 bits) per the Bluetooth Core Specification Vol 6, Part B, §2.3 [1]. tshark outputs this header as a hex byte string (2 or 4 hex chars depending on DLT). The PDU type and address type are extracted from the **first header byte** (byte 0):

```
Advertising Channel PDU Header, Byte 0:
  Bit 7:   RxAdd   (target address type: 0=public, 1=random)
  Bit 6:   TxAdd   (advertiser address type: 0=public, 1=random)
  Bit 5:   ChSel   (channel selection algorithm)
  Bit 4:   RFU     (reserved for future use)
  Bits 3-0: PDU Type (0=ADV_IND, 1=ADV_DIRECT_IND, 2=ADV_NONCONN_IND,
                       3=SCAN_REQ, 4=SCAN_RSP, 5=CONNECT_IND, 6=ADV_SCAN_IND)
```

The post-processing pipeline extracts:
- `addr_type` from bit 6 (TxAdd): `(header_val & 0x40) ? "1" : "0"`
- `pdu_type` from the low nibble bits [3:0]: `header_val & 0x0F`

This matches the ADR-0001 example: an `ADV_NONCONN_IND` (PDU type 2) with random address (TxAdd=1) produces header byte `0x42` (0x40 | 0x02), giving `addr_type=1, pdu_type=2`.

For data channel PDUs (not advertising), the header format differs; the pipeline attempts extraction regardless, with unexpected values producing empty fields that the ingest bridge handles as `std::nullopt`.

**Key behaviours:**
- `-l` (flush after each packet): Mandatory for real-time pipeline. Without it, tshark buffers output, causing ingest latency proportional to buffer size.
- `-E separator=\|`: Pipe-delimited output matches the CAPTURE-2 contract expected by the ingest bridge.
- `-E quote=n`: No quoting of field values — simplifies parser in C05.
- `-E occurrence=f`: For fields that may appear multiple times per packet (unlikely in BLE advertising), use the first occurrence.
- tshark reads from `stdin` redirected from the PCAP FIFO and pipes through `awk` post-processing to the ingest FIFO.
- Post-processing in `awk` provides zero-allocation, line-at-a-time transformation without buffering — suitable for the real-time hot path.
- If the input FIFO has no writer (e.g., sniffer not yet started), tshark blocks on the open syscall until tail-feed writes the first byte.

#### 4.3.3 DLT Determination

The DLT value is determined at container startup:

1. The Quadlet `.container` file specifies `BLESNIFF_DLT=256` or `BLESNIFF_DLT=272` via `Environment=` or `EnvironmentFile`.
2. The DLT value is derived from the sniffer's `type` field in `sniffers.yaml`:
   - `type: ubertooth` → DLT 256
   - `type: nrf` → DLT 272
3. A startup validation step reads the PCAP global header from the current file and verifies the DLT matches the configured value. If there is a mismatch, the wrapper logs a FATAL error and exits, preventing data corruption downstream.

### 4.4 Heartbeat Companion

The heartbeat companion is a lightweight shell script that runs alongside the sniffer wrapper in the same container. It writes a Unix epoch timestamp to the local heartbeat file every 30 seconds and best-effort writes to the database heartbeat table.

```bash
#!/usr/bin/env bash
# heartbeat.sh — Liveness heartbeat for a sniffer instance.
# Writes epoch timestamp to local file (primary) and DB (secondary).

set -euo pipefail

SNIFFER_NAME="${BLESNIFF_SNIFFER_NAME:?BLESNIFF_SNIFFER_NAME not set}"
HEARTBEAT_FILE="/var/lib/tianer/heartbeat/${SNIFFER_NAME}.ts"
INTERVAL="${BLESNIFF_HEARTBEAT_INTERVAL:-30}"

# Ensure heartbeat directory exists
mkdir -p "$(dirname "${HEARTBEAT_FILE}")"

# DB connection parameters (from environment, for secondary heartbeat path)
DB_HOST="${TIANER_DB_HOST:-127.0.0.1}"
DB_PORT="${TIANER_DB_PORT:-5432}"
DB_NAME="${TIANER_DB_NAME:-tianer}"
DB_USER="${BLESNIFF_DB_USER:-tianer_writer}"
DB_PASSWORD_FILE="${BLESNIFF_DB_PASSWORD_FILE:-/etc/tianer/secrets/db_password}"

# Resolve sniffer_id from the sniffers table (once at startup).
# The sniffer_heartbeat table uses sniffer_id (SMALLINT PK) per C02 §3.8,
# so we must map sniffer_name → sniffer_id before writing heartbeats.
SNIFFER_ID=""
if command -v psql >/dev/null 2>&1; then
    SNIFFER_ID=$(PGPASSWORD="$(cat "${DB_PASSWORD_FILE}" 2>/dev/null || true)" \
        psql -h "${DB_HOST}" -p "${DB_PORT}" -U "${DB_USER}" -d "${DB_NAME}" \
            -t -A -c "SELECT sniffer_id FROM bluetooth.sniffers WHERE name = '${SNIFFER_NAME}';" \
            2>/dev/null || true)
fi

if [[ -z "${SNIFFER_ID}" ]]; then
    echo "TIANER | {\"ts\":\"$(date -Iseconds)\",\"level\":\"WARN\",\"component\":\"heartbeat\",\"sniffer\":\"${SNIFFER_NAME}\",\"msg\":\"Could not resolve sniffer_id from sniffers table (DB may be down or sniffer not registered)\"}" >&2
fi

while true; do
    EPOCH_TS="$(date +%s)"

    # Primary: Local file (always attempted, always succeeds if FS is writable)
    if ! printf '%s\n' "${EPOCH_TS}" > "${HEARTBEAT_FILE}"; then
        echo "TIANER | {\"ts\":\"$(date -Iseconds)\",\"level\":\"ERROR\",\"component\":\"heartbeat\",\"sniffer\":\"${SNIFFER_NAME}\",\"msg\":\"Heartbeat file write failed\"}" >&2
    fi

    # Secondary: Database (best-effort; do NOT block the heartbeat loop).
    # Uses sniffer_id (PK), ts, status per C02 §3.8 schema.
    if [[ -n "${SNIFFER_ID}" ]] && command -v psql >/dev/null 2>&1; then
        PGPASSWORD="$(cat "${DB_PASSWORD_FILE}" 2>/dev/null || true)" \
        psql -h "${DB_HOST}" -p "${DB_PORT}" -U "${DB_USER}" -d "${DB_NAME}" \
            -c "INSERT INTO bluetooth.sniffer_heartbeat (sniffer_id, ts, status)
                VALUES (${SNIFFER_ID}, NOW(), 'running')
                ON CONFLICT (sniffer_id) DO UPDATE SET ts = NOW(), status = 'running'" \
            >/dev/null 2>&1 || \
            echo "TIANER | {\"ts\":\"$(date -Iseconds)\",\"level\":\"WARN\",\"component\":\"heartbeat\",\"sniffer\":\"${SNIFFER_NAME}\",\"msg\":\"DB heartbeat write failed (DB may be down)\"}" >&2
    fi

    sleep "${INTERVAL}"
done
```

**Key behaviours:**
- **sniffer_id resolution at startup:** On first tick, the script resolves the sniffer's `sniffer_id` from the `bluetooth.sniffers` table using the `BLESNIFF_SNIFFER_NAME` value. This maps the human-readable name to the numeric `sniffer_id` used as the PK in `sniffer_heartbeat` (per C02 §3.8). If the lookup fails (DB down, sniffer not yet registered), the script logs a WARN and skips DB heartbeat writes until the next restart — the local file heartbeat remains unaffected.
- **Primary path always first:** The local file write is attempted before the DB write. If the filesystem is writable, the heartbeat always updates.
- **DB heartbeat uses C02 schema columns:** The INSERT uses `sniffer_id` (SMALLINT PK), `ts` (TIMESTAMPTZ), and `status` (TEXT, value `'running'`). The `ON CONFLICT (sniffer_id)` upsert matches the C02 §3.8 PK definition.
- **DB path is best-effort:** DB write failures are logged at WARN level and do not block the loop. The heartbeat interval is not disrupted by DB outages.
- **On DB recovery:** The gap detector (C06) backfills the `bluetooth.sniffer_heartbeat` table from the local heartbeat file timestamps.
- **Interval jitter:** The `sleep` interval is a fixed 30 seconds (no jitter), producing a predictable heartbeat cadence for gap detection.

### 4.5 Startup Sequence

The full startup sequence for a single sniffer instance:

```
Container: blesniff-sniffer@<name>
  │
  ├── 1. Read /etc/tianer/sniffers.yaml → extract config for this sniffer
  ├── 2. Validate: device symlink exists? channels valid? type supported?
  ├── 3. Start sniffer wrapper (ubertooth-wrap or nrf-wrap)
  │        └── Opens PCAP file, begins writing
  ├── 4. Start tail-feed (tail -c +0 -F <pcap_file> > <name>.fifo)
  │        └── Blocks until PCAP file has data
  ├── 5. Start heartbeat companion
  │        └── Begins 30-second tick loop
  └── 6. Enter wait loop: monitor sniffer PID.
           On SIGTERM → kill tail-feed, heartbeat; wait for sniffer to exit.

Container: blesniff-tshark@<name>  (starts after PCAP FIFO has a writer)
  │
  ├── 1. Read BLESNIFF_DLT, BLESNIFF_SNIFFER_NAME from environment
  ├── 2. Wait for <name>.fifo to exist and have a writer (poll or block on open)
  ├── 3. Start tshark-wrap with per-DLT field parameters
  │        └── Reads from <name>.fifo, writes to <name>-ingest.fifo
  └── 4. tshark exits when read end of FIFO gets EOF (sniffer stopped)
```

**Dependencies between containers:**
- `blesniff-tshark@.container` depends on `blesniff-sniffer@.container` (systemd `Requires=` and `After=` directives in the generated unit). The tshark container must wait until the sniffer container has created the PCAP FIFO and tail-feed has written the first byte.

### 4.6 Error Handling in Wrapper Scripts

All wrapper scripts follow a consistent error handling pattern:

1. **Config validation:** All required environment variables are checked with `${VAR:?message}` — missing variables cause immediate exit with a clear error message, preventing silent misconfiguration.
2. **Device check:** The sniffer wrapper verifies the device symlink exists before launching the sniffer binary. If absent, it logs an ERROR and exits.
3. **FIFO readiness:** tshark-wrap blocks on opening the PCAP FIFO for reading. If the tail-feed is not yet writing, this is a normal state (not an error). The open syscall returns when a writer connects.
4. **Exit trap:** A `trap` handler in the container entrypoint ensures child processes (sniffer, tail-feed, heartbeat) are terminated on SIGTERM.
5. **Structured logging:** All error and warning messages use the `TIANER | {...}` structured log format for consistent routing to Loki.

---

## 5. Inter-Component Contracts

### 5.1 CAPTURE-1 — Sniffer to PCAP

| Property | Value |
|----------|-------|
| **Contract ID** | CAPTURE-1 |
| **From** | Sniffer wrapper (ubertooth-wrap.sh or nrf-wrap.sh) |
| **To** | PCAP file on V02 (`/var/lib/tianer/pcap/<name>/capture.pcap`) |
| **Format** | Binary PCAP (libpcap 1.10 format) |
| **DLT** | 256 (Ubertooth One: `LINKTYPE_BLUETOOTH_LE_LL_WITH_PHDR`) or 272 (nRF52840 Sniffer: `LINKTYPE_NORDIC_BLE`) |
| **Transport** | Regular file write to V02 bind-mount |
| **Guarantees** | Zero data loss from hardware to disk. Sniffer writes only to disk, never to a FIFO. Write failures are fatal — the sniffer exits and triggers a container restart. |
| **Rotation contract** | The sniffer wrapper responds to SIGHUP by closing and reopening the PCAP file (coordinates with C04 ROTATION-1). |
| **Metadata** | PCAP global header records: DLT, snapshot length (256 bytes for BLE advertising), endianness (little-endian on ARM64). Per-packet headers record: timestamp (seconds + microseconds), captured length, original length. |

**DLT 256 packet structure (Ubertooth One):**
```
[RF metadata header: 6 fields]
[BTLE Link Layer: access_address (4) + PDU header (2) + payload (0-37) + CRC (3)]
```

**DLT 272 packet structure (nRF52840 Sniffer):**
```
[Nordic BLE metadata header: board_id + counter + flags + channel + RSSI + event_counter + timestamp + delta + ...]
[BTLE Link Layer: access_address + PDU header + payload + CRC]
```

### 5.2 CAPTURE-2 — tshark to Ingest Bridge

This contract is the C03 producer side of **INGEST-1** (C05 §5.1). The two contracts describe the same schema from the producer and consumer perspectives, respectively.

| Property | Value |
|----------|-------|
| **Contract ID** | CAPTURE-2 |
| **From** | tshark-wrap.sh (per-DLT parameterised tshark with awk post-processing) |
| **To** | Ingest bridge (C05), via ingest FIFO on V03 |
| **Format** | Pipe-delimited (`|`) text lines, one per BLE packet |
| **Transport** | Named FIFO (`/var/run/tianer/<name>-ingest.fifo`), mode 0660, owner `tianer:tianer` |
| **Line encoding** | UTF-8, no quoting, no escaping. Newline (`\n`) terminates each record. |
| **Schema** | 7 pipe-separated fields: `ts|mac_address|addr_type|rssi|channel|pdu_type|advdata_hex` |
| **Consumer contract** | Mirrors INGEST-1 (C05 §5.1). The ingest bridge is the authoritative consumer of this schema. |
| **Normalisation guarantee** | Fields are in a consistent order and format regardless of source DLT. The per-DLT mapping and awk post-processing pipeline in tshark-wrap.sh ensures the ingest bridge never receives DLT-specific field names or raw PDU header bytes. |
| **Missing fields** | Fields not present in a given packet are output as empty strings (e.g., `||-52|`) — the ingest bridge treats empty strings as `std::nullopt` per INGEST-1. |
| **sniffer_name exclusion** | The sniffer name is **not** included in the line. It is injected by the ingest bridge's `PgWriter` constructor as `sniffer_id` (C05 §3.2). This eliminates redundant per-line data. |
| **Replaces** | CONTRACT 8.4-A from inception document |

### 5.3 CAPTURE-3 — Heartbeat Protocol

| Property | Value |
|----------|-------|
| **Contract ID** | CAPTURE-3 |
| **From** | Heartbeat companion script (`heartbeat.sh`) |
| **To** | Local filesystem (primary): `/var/lib/tianer/heartbeat/<sniffer_name>.ts` and Database (secondary): `bluetooth.sniffer_heartbeat` table |
| **Primary path** | Local file on V02. Contains a single Unix epoch timestamp (integer). Updated every 30 seconds. Read by gap detector (C06) via V02 `:ro` mount. |
| **Secondary path** | PostgreSQL table `bluetooth.sniffer_heartbeat` (see C02 §3.8 schema). Upsert on `sniffer_id` (PK per C02 schema). Heartbeat script first resolves `sniffer_name → sniffer_id` from the `sniffers` table at startup, then writes `(sniffer_id, ts, status)` with `status='running'` via `ON CONFLICT (sniffer_id)`. Written best-effort; failures are logged but do not block the heartbeat loop. |
| **DB outage behaviour** | Primary local file continues updating. Secondary DB writes fail (logged at WARN). On DB recovery, gap detector (C06) backfills the DB heartbeat table from the local file timestamps. No false-positive gap alerts during DB outage (PF-3: redundant detection). |
| **Interval** | 30 seconds (±2 seconds jitter from OS scheduling). |
| **Staleness detection** | Gap detector (C06) reads the local heartbeat file. Alert when `(now - heartbeat_ts) > 60 seconds`. This allows one full missed update plus timing tolerance. |
| **Replaces** | Heartbeat protocol from inception document, updated per D-11 (local file primary, DB secondary). |

---

## 6. Failure Modes & Recovery

### 6.1 Failure Catalog

| # | Failure Mode | Detection | Impact | Recovery |
|---|-------------|-----------|--------|----------|
| **F-C03-1** | **Sniffer process crash** | Container exit code ≠ 0; heartbeat file stops updating. `tianer_sniffer_crash_total` counter increments. | One sniffer channel lost. Other sniffers in pod unaffected (PF-4: graceful degradation). PCAP file closed cleanly or truncated at last write. | Podman `Restart=on-failure` with `RestartSec=5s`. Gap detector identifies outage window from heartbeat gap and backfills DB from PCAP. Zero loss from PCAP source. |
| **F-C03-2** | **USB device disconnected** | Sniffer tool exits with device error. `tianer_usb_device_present{device="nrf0"} == 0` metric. Logged at ERROR level. | One sniffer channel lost; others continue. `lsusb` no longer shows the device. `/dev/tianer/<name>` symlink disappears. | Operator reconnects dongle; udev recreates symlink. Sniffer container auto-restarts. Sniffer reconnects and resumes capture. Gap detector backfills outage window. |
| **F-C03-3** | **tshark process crash (FIFO reader death)** | tshark container exits. `tianer_tshark_crash_total` increments. Ingest FIFO has no writer — ingest bridge (C05) gets EOF on read. | Ingest stops for that sniffer. tail-feed blocks on PCAP FIFO write (backpressure propagates). Sniffer unaffected — continues writing PCAP. | Podman restarts tshark container. tshark reopens PCAP FIFO and resumes reading. Since `tail -c +0 -F` starts from the file beginning, tshark re-reads from the start. The ingest bridge must handle duplicate records via DB dedup (D-10: query-time dedup). |
| **F-C03-4** | **DLT mismatch at startup** | Startup validation reads PCAP global header DLT and compares to `BLESNIFF_DLT`. Mismatch triggers FATAL log and exit. | Sniffer instance does not start. Alert fires. Other sniffers unaffected. | Operator corrects `sniffers.yaml` type field or `BLESNIFF_DLT` environment variable. Rebuilt container restarts. |
| **F-C03-5** | **Backpressure cascade (DB outage)** | Ingest FIFO fills → tshark blocks on write → PCAP FIFO fills → tail-feed blocks on write. `tianer_ingest_fifo_full_total` metric increments. | Pipeline stalled downstream of tail-feed. Sniffer continues writing PCAP to V02. No PCAP data loss. DB ingest paused. | When DB recovers, backpressure clears in reverse order: ingest bridge resumes reading → tshark unblocks → tail-feed unblocks. Gap detector backfills missed DB rows from PCAP. |
| **F-C03-6** | **FIFO buffer full (high packet rate)** | tail-feed blocks on FIFO write. `tianer_pcap_fifo_full_total` metric increments. | Same as F-C03-5 but triggered by high packet rate rather than downstream outage. Sniffer unaffected. | Backpressure self-clears when packet rate drops. If sustained, consider larger pipe buffer (`fs.pipe-max-size`) or additional ingest bridge instances. |
| **F-C03-7** | **PCAP disk full (V02)** | Sniffer `write()` syscall returns ENOSPC. Wrapper logs FATAL error and exits. `tianer_disk_usage_percent > 90` alert fires. | Sniffer stops. Current PCAP file truncated at last successful write. PCAP data loss for packets arriving after disk full. | C04 emergency purge of oldest PCAP files. Operator expands storage or replaces SD card. Sniffer container restarts on freed space. Gap detector backfills any recoverable data. |
| **F-C03-8** | **Heartbeat script stuck/crashed** | Heartbeat file age exceeds 60 seconds. Heartbeat companion is a child process of sniffer container — crash detection is via container health check. `tianer_heartbeat_missed_total` metric increments. | Gap detector reports a gap for the heartbeat outage window. If the sniffer is still running, DB is still receiving data — this is a **false positive gap alert** (mitigated by secondary gap verification, PF-3). | Heartbeat companion is restarted as part of sniffer container restart. During the outage, PCAP is still being written — gap detector's secondary verification (PCAP row count vs DB row count) will confirm no actual data loss. |
| **F-C03-9** | **tail-feed crash** | Container health check detects tail-feed PID missing. PCAP FIFO has no writer — tshark gets EOF and exits (triggers F-C03-3). | Pipeline stalls at PCAP FIFO level. Sniffer unaffected — still writes PCAP to disk. | Container restart restarts all processes including tail-feed. tail-feed reopens PCAP file and resumes feeding FIFO. tshark restarts and re-reads. |
| **F-C03-10** | **nRF dongle in DFU mode (PID 520f)** | `nrfutil ble-sniffer` fails to connect. Wrapper logs ERROR: "Device in DFU mode, PID 520f". `tianer_nrf_dfu_mode_total` counter increments. | Sniffer instance does not start. Operator must flash sniffer firmware. | Operator runs `nrfutil dfu program --firmware <sniffer_firmware.zip>` to flash the sniffer firmware. After flash, device re-enumerates with PID 522A. Container auto-restarts. |

### 6.2 Recovery Procedures

#### Sniffer Container Restart (F-C03-1)

```
1. sniffer process exits (crash, SIGKILL, or fatal error)
2. Podman detects exit code ≠ 0
3. Podman (via systemd) waits RestartSec=5s
4. Container restarts → entrypoint runs
5. Wrapper validates config, device, and PCAP directory
6. Sniffer opens capture.pcap (appends if file exists, creates new if not)
7. tail-feed starts: tail -c +0 -F capture.pcap → reads from start or resumes
8. tshark container restarts (triggered by systemd dependency on sniffer container)
9. tshark reopens PCAP FIFO and resumes processing
10. Gap detector identifies outage window from heartbeat gap → backfills from PCAP
```

#### USB Reconnect (F-C03-2)

```
1. Operator physically reconnects dongle
2. udev detects device → creates /dev/tianer/<name> symlink
3. Sniffer container auto-restarts (Restart=on-failure)
4. Wrapper validates device symlink exists → launches sniffer
5. Capture resumes on configured channel
6. Gap detector identifies outage from heartbeat gap → backfills from PCAP
```

#### Backpressure Recovery (F-C03-5)

```
1. Downstream blockage clears (e.g., DB recovers)
2. Ingest bridge reconnects to DB → resumes reading ingest FIFO
3. tshark unblocks on ingest FIFO write → resumes reading PCAP FIFO
4. tail-feed unblocks on PCAP FIFO write → resumes reading PCAP file
5. Pipeline returns to normal throughput within ~1 second
6. Gap detector identifies blocked time window → backfills from PCAP
```

### 6.3 Recovery SLA

| Scenario | Detection Time | Recovery Time | Data Loss |
|----------|---------------|---------------|-----------|
| Sniffer crash (F-C03-1) | < 30s (heartbeat staleness) | < 10s (Podman restart) | Zero from PCAP. DB gap ≤ 60s |
| USB disconnect (F-C03-2) | < 30s (heartbeat staleness or USB metric) | Manual (operator reconnects) | Zero from PCAP. DB gap = disconnect duration |
| tshark crash (F-C03-3) | < 5s (container exit) | < 10s (Podman restart) | Zero from PCAP. Ingest pauses for restart duration |
| Backpressure cascade (F-C03-5) | < 5s (ingest bridge health check) | Dependent on downstream recovery | Zero from PCAP |
| Disk full (F-C03-7) | < 60s (disk usage metric) | Manual (operator expands storage) | Packets lost after disk full |

---

## 7. Observability

### 7.1 Metrics

All metrics use the Prometheus text format and are emitted by each sniffer and tshark container.

| Metric Name | Type | Labels | Source | Description |
|------------|------|--------|--------|-------------|
| `tianer_sniffer_packets_captured_total` | Counter | `sniffer`, `channel` | sniffer wrapper | Total BLE packets captured since startup |
| `tianer_sniffer_uptime_seconds` | Gauge | `sniffer` | sniffer wrapper | Seconds since sniffer process started |
| `tianer_sniffer_crash_total` | Counter | `sniffer` | Podman / systemd | Number of sniffer process crashes |
| `tianer_tshark_crash_total` | Counter | `sniffer` | Podman / systemd | Number of tshark process crashes |
| `tianer_tshark_packets_processed_total` | Counter | `sniffer` | tshark-wrap | Total packets processed by tshark |
| `tianer_pcap_bytes_written_total` | Counter | `sniffer` | sniffer wrapper | Total bytes written to PCAP file |
| `tianer_pcap_fifo_full_total` | Counter | `sniffer` | tail-feed | Number of times tail-feed blocked on full FIFO |
| `tianer_ingest_fifo_full_total` | Counter | `sniffer` | tshark-wrap | Number of times tshark blocked on full ingest FIFO |
| `tianer_heartbeat_missed_total` | Counter | `sniffer` | heartbeat companion | Number of missed heartbeat intervals |
| `tianer_dlt_wrong_total` | Counter | `sniffer` | startup validation | Number of DLT mismatch errors at startup |
| `tianer_nrf_dfu_mode_total` | Counter | `sniffer` | nrf-wrap | Number of times nRF was in DFU mode |

### 7.2 Structured Logging

All C03 processes emit logs in the standard Tian'er structured format:

```
TIANER | {"ts":"ISO8601","level":"LEVEL","component":"COMPONENT","sniffer":"NAME","msg":"MESSAGE",...}
```

| Component values | Processes |
|-----------------|-----------|
| `sniffer` | ubertooth-wrap.sh, nrf-wrap.sh |
| `tshark` | tshark-wrap.sh |
| `heartbeat` | heartbeat.sh |
| `tail-feed` | tail-feed.sh |

**Log levels used:**
- `INFO`: Normal operational events (capture started, channel configured, rotation handled)
- `WARN`: Recoverable issues (DB heartbeat write failed, packet dropped by sniffer firmware, USB reconnection pending)
- `ERROR`: Problems requiring attention (device not found, disk write failure, DLT mismatch)
- `FATAL`: Unrecoverable (config missing, unsupported DLT) — process exits after logging

### 7.3 Health Checks

Each container exposes a health check that verifies the critical processes are running:

**Sniffer container health check:**

```bash
# Verify sniffer process is running
pgrep -f "ubertooth-btle\|nrfutil ble-sniffer" >/dev/null || exit 1

# Verify tail-feed is running
pgrep -f "tail -c +0 -F" >/dev/null || exit 1

# Verify heartbeat is running
pgrep -f "heartbeat.sh" >/dev/null || exit 1

exit 0
```

**tshark container health check:**

```bash
# Verify tshark process is running
pgrep -f "tshark -i -" >/dev/null || exit 1

# Verify output FIFO exists and has tshark as writer
test -p "/var/run/tianer/${BLESNIFF_SNIFFER_NAME}-ingest.fifo" || exit 1

exit 0
```

### 7.4 Alert Rules

Per C13 (Observability), the following alerts are relevant to C03:

| Alert | Condition | Severity | Action |
|-------|-----------|----------|--------|
| `TianerSnifferDown` | `tianer_sniffer_uptime_seconds == 0` for any sniffer | **Critical** | Check sniffer container status; inspect logs; gap detector will backfill |
| `TianerHeartbeatMissed` | `tianer_heartbeat_missed_total` increase rate > 0 for > 60s | **Warning** | Check heartbeat companion; if sniffer is running, verify filesystem writable |
| `TianerFifoFull` | `tianer_pcap_fifo_full_total` increase rate > 0 | **Warning** | Downstream ingest may be slow; check ingest bridge health |
| `TianerDLTMismatch` | `tianer_dlt_wrong_total > 0` | **Critical** | Sniffer config is wrong for this hardware; correct `sniffers.yaml` |
| `TianerDiskFull` | `tianer_host_disk_usage_percent{pcap} > 90` | **Critical** | Trigger emergency PCAP purge; expand storage |

---

## 8. Security Considerations

### 8.1 Shell Injection via Configuration Values

The sniffer configuration is read from `sniffers.yaml` and injected into shell environment variables. Malicious configuration values could exploit shell interpretation:

**Attack vector:** An attacker with write access to `/etc/tianer/sniffers.yaml` could inject shell metacharacters into fields like `name`, `device`, or `channels`.

**Example:**
```yaml
sniffers:
  - name: 'ut1; rm -rf / #'
    type: ubertooth
    ...
```

**Mitigation:**
1. `/etc/tianer/sniffers.yaml` is mounted `:ro` into containers (V01) and is owned by `tianer:tianer` with mode 0640. Only the root operator can modify it on the host.
2. All wrapper scripts validate the `name` field against a regex (`^[a-z][a-z0-9]*$`) before constructing paths. Invalid names cause immediate exit.
3. Device paths are validated to start with `/dev/tianer/` — arbitrary paths are rejected.
4. Channel values are validated as integers in the range [37, 38, 39].
5. All variable substitutions use `${VAR:?message}` (fail on undefined) and are quoted in command invocations to prevent word splitting.
6. The `yq` tool (or a minimal YAML-to-env parser) reads `sniffers.yaml` in a controlled way rather than using `source` or `eval`.

### 8.2 USB Device Access

Sniffer containers must access USB devices via `/dev/tianer/*` symlinks. This is achieved through:

1. **Group membership:** The `tianer` user belongs to `plugdev` (USB access) and `dialout` (CDC ACM serial). Container processes inherit these groups via `--group-add keep-groups` in Quadlet.
2. **Device passthrough:** Each container receives only the specific device symlink it needs (`--device /dev/tianer/ubertooth0`), not the entire `/dev` tree.
3. **No `--privileged`:** Zero containers run privileged. The only added capabilities are `CAP_NET_RAW` and `CAP_NET_ADMIN` (required by dumpcap for packet capture at the kernel level).
4. **Device not found:** If a device symlink does not exist, the sniffer wrapper logs an ERROR and exits. The container does not crash-loop; Podman's `Restart=on-failure` with `StartLimitBurst=5` prevents endless restart cycles.

### 8.3 Container Capabilities

```
Tian'er capture pod (Network=none):
  --cap-drop ALL
  --cap-add CAP_NET_RAW
  --cap-add CAP_NET_ADMIN
  --security-opt no-new-privileges
  ReadOnlyPaths=/etc/tianer          # V01: config is read-only
  ReadWritePaths=/var/lib/tianer/pcap # V02: PCAP writes
  ReadWritePaths=/var/run/tianer      # V03: FIFO read/write
  ReadWritePaths=/var/log/tianer      # V04: log writes
  TmpFS=/tmp                          # V08: ephemeral temp
```

### 8.4 PCAP Data Sensitivity

PCAP files contain raw BLE advertising data, including:
- Device MAC addresses (public or random resolvable)
- Service UUIDs (may identify device type/manufacturer)
- Manufacturer-specific data
- TX power levels and RSSI (may reveal device proximity)

**Protection:**
- V02 is on a bind-mounted host directory (`/var/lib/tianer/pcap/`) with mode 0750, owner `tianer:tianer`.
- Only the `tianer` user and root can read PCAP files.
- PCAP is never exposed over the network (tianer-capture pod has Network=none).
- Access to PCAP is through C09 REST API (authenticated) or Grafana (read-only DB role), not direct file access.

---

## 9. Configuration

### 9.1 Configuration Sources

C03 reads configuration from three sources, in priority order:

| Priority | Source | Scope | Example |
|----------|--------|-------|---------|
| 1 (highest) | Container environment (`Environment=` or `EnvironmentFile`) | Per-sniffer instance | `BLESNIFF_SNIFFER_NAME=nrf1` |
| 2 | `/etc/tianer/sniffers.yaml` (V01, `:ro`) | All sniffers | Channel, type, device path |
| 3 (lowest) | `/etc/tianer/blesniff.env` (V01, `:ro`) | Bluetooth module defaults | `BLESNIFF_ROTATION_MINUTES` |

### 9.2 Per-Sniffer Environment Variables

The following environment variables are set per sniffer instance in the Quadlet `.container` file:

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `BLESNIFF_SNIFFER_NAME` | Yes | — | Sniffer instance name (ut1, nrf1, nrf2, nrf3) |
| `BLESNIFF_SNIFFER_TYPE` | Yes | — | `ubertooth` or `nrf` |
| `BLESNIFF_SNIFFER_DEVICE` | Yes | — | `/dev/tianer/<symlink>` |
| `BLESNIFF_CHANNEL` / `BLESNIFF_CHANNELS` | No | `37` | Single channel (int) or comma-separated list (e.g., `37,38,39`) |
| `BLESNIFF_DLT` | Yes (tshark container) | — | 256 (Ubertooth) or 272 (nRF) |
| `BLESNIFF_PCAP_DIR` | No | `/var/lib/tianer/pcap` | Parent directory for PCAP files |
| `BLESNIFF_FIFO_DIR` | No | `/var/run/tianer` | Parent directory for FIFOs |
| `BLESNIFF_HEARTBEAT_INTERVAL` | No | `30` | Heartbeat interval in seconds |
| `BLESNIFF_LOG_LEVEL` | No | `info` | Log verbosity: debug, info, warn, error |

### 9.3 Channel Configuration per Sniffer

**Ubertooth One (single channel):**
```yaml
# sniffers.yaml
sniffers:
  - name: ut1
    type: ubertooth
    channels: [37]  # Exactly one channel (hardware limitation)
```

The Ubertooth One can monitor exactly one BLE advertising channel at a time per D-05. The default channel is 37 (2402 MHz). The operator may change this to 38 or 39 via `sniffers.yaml` and a container restart.

**nRF52840 Sniffer (multi-channel hopping):**
```yaml
# sniffers.yaml
sniffers:
  - name: nrf1
    type: nrf
    channels: [37, 38, 39]  # Hop across all three advertising channels
```

The nRF Sniffer firmware supports channel hopping across all three advertising channels simultaneously. The list can contain 1, 2, or 3 channels. The default is `[37]` for a single-channel deployment per Q6.

### 9.4 BLE Advertising Channels Reference

| Channel | Frequency | Wi-Fi Overlap | Typical Usage |
|---------|-----------|---------------|---------------|
| 37 | 2402 MHz | None (below Wi-Fi ch1) | Least congested; recommended default |
| 38 | 2426 MHz | Wi-Fi ch1 overlap | May have Wi-Fi interference in urban areas |
| 39 | 2480 MHz | Wi-Fi ch13/14 overlap | May have Wi-Fi interference |

Channel 37 (2402 MHz) is the recommended default for single-channel deployment because it sits below the 2.4 GHz Wi-Fi band (which starts at 2412 MHz) and typically has the least interference.

---

## 10. Test Plan

### 10.1 Unit Tests

All wrapper scripts are tested with bats-core (Bash Automated Testing System). Test files live in `modules/bluetooth/sniffers/tests/`.

| Test Suite | File | Scope |
|-----------|------|-------|
| `test_fifo_create.bats` | FIFO creation and permission tests | Verify FIFOs are created with correct mode, owner, and group |
| `test_dual_write.bats` | Dual-write (PCAP + FIFO) integrity | Verify PCAP file grows; verify FIFO delivers complete stream |
| `test_args.bats` | Argument validation | Verify required env var checks; verify DLT validation; verify channel range |
| `test_dlt_param.bats` | DLT parameterisation | Verify tshark-wrap selects correct field expressions for each DLT |
| `test_heartbeat.bats` | Heartbeat file format | Verify timestamp format; verify atomic overwrite; verify directory creation |

### 10.2 Integration Tests

| Test | Scope | Method |
|------|-------|--------|
| `test_capture_pipeline.sh` | End-to-end: sniffer → PCAP → tail-feed → FIFO → tshark → ingest FIFO | Start mock sniffer (generates synthetic PCAP), verify tail-feed, verify tshark output matches CAPTURE-2 schema |
| `test_backpressure.sh` | Backpressure propagation | Fill ingest FIFO (stop reader), verify tail-feed blocks, verify sniffer continues writing PCAP, verify no packet loss in PCAP |
| `test_rotation_coordination.sh` | SIGHUP rotation | Signal sniffer wrapper, verify new PCAP file created, verify tail-feed follows new file, verify no gaps in tshark output |
| `test_dlt_mismatch.sh` | DLT mismatch detection | Configure wrong DLT, verify startup validation catches mismatch, verify container exits with FATAL log |

### 10.3 Hardware Tests

| Test | Hardware Required | Scope |
|------|-------------------|-------|
| `test_ubertooth_capture.sh` | Ubertooth One | Verify Ubertooth captures on channel 37; verify DLT 256 PCAP output; verify tshark field extraction |
| `test_nrf_capture.sh` | nRF52840 Dongle | Verify nRF Sniffer captures; verify DLT 272 PCAP output; verify channel hopping; verify multi-channel output |
| `test_multi_sniffer.sh` | 2+ dongles | Verify two sniffers run concurrently without interference |
| `test_usb_disconnect.sh` | Any dongle | Physically disconnect during capture; verify heartbeat stops; verify container restart on reconnect |

### 10.4 Acceptance Criteria

| Criterion | Measurement | Threshold |
|-----------|-------------|-----------|
| **PCAP data integrity** | PCAP packet count vs bytes written | Zero discrepancy (write-and-verify) |
| **FIFO throughput** | Packets/sec through PCAP FIFO | ≥ 1000 pps sustained (typical BLE advertising rate) |
| **Backpressure isolation** | PCAP packets written during 60s FIFO block | Zero loss (all packets in PCAP) |
| **tshark normalisation** | Fields in ingest FIFO output | All 7 fields present (per CAPTURE-2); consistent order; DLT-independent |
| **Heartbeat liveness** | Heartbeat file update interval | Every 30s ± 5s |
| **DLT validation** | Mismatch detection | 100% of mismatches detected at startup |
| **Startup time** | Sniffer ready → first packet to PCAP | ≤ 5 seconds from container start |
| **Container restart** | Process crash → capture resumed | ≤ 30 seconds (Podman restart + sniffer init) |

---

## 11. Deployment Notes

### 11.1 Quadlet Container Files

C03 is deployed as two Quadlet `.container` files per sniffer instance, using systemd template instantiation (`@` notation). The `.container` files are placed in `/etc/containers/systemd/` and managed by `systemctl --user` under the `tianer` user.

#### 11.1.1 Sniffer Container (`blesniff-sniffer@.container`)

```ini
[Container]
# Per-sniffer instance: blesniff-sniffer@ut1, blesniff-sniffer@nrf1, etc.
# The %i specifier expands to the sniffer name (ut1, nrf1, nrf2, nrf3).

ContainerName=blesniff-sniffer-%i
Image=localhost/tianer-sniffer:latest

# Environment: loaded from host env files (V01) and per-instance template
EnvironmentFile=/etc/tianer/tianer.env
EnvironmentFile=/etc/tianer/blesniff.env
Environment=BLESNIFF_SNIFFER_NAME=%i

# Network
Pod=tianer-capture.pod
Network=none                                # Inherited from pod

# Volumes
Volume=/etc/tianer:/etc/tianer:ro           # V01
Volume=/var/lib/tianer/pcap:/var/lib/tianer/pcap:rw   # V02
Volume=/var/run/tianer:/var/run/tianer:rw   # V03
Volume=/var/log/tianer:/var/log/tianer:rw   # V04
Volume=/tmp:/tmp:rw                          # V08 (tmpfs)

# USB passthrough — device varies by instance
# Configured per instance via drop-in or separate file
# Example: --device /dev/tianer/ubertooth0 for %i=ut1

# Capabilities
AddCapability=CAP_NET_RAW
AddCapability=CAP_NET_ADMIN
DropCapability=ALL

# Security
NoNewPrivileges=true
GroupAdd=keep-groups

# Exec
Exec=/usr/local/lib/tianer/wrap/sniffer-entrypoint.sh

# Restart
# Restart=, RestartSec=, StartLimitBurst= per systemd.service(5) [6].
Restart=on-failure
RestartSec=5
StartLimitBurst=5

[Unit]
Description=Tian'er BLE Sniffer %i
After=network.target tianer-postgres.service
Requires=tianer-postgres.service
Wants=tianer-capture.pod

[Install]
WantedBy=blesniff.target
```

#### 11.1.2 tshark Container (`blesniff-tshark@.container`)

```ini
[Container]
ContainerName=blesniff-tshark-%i
Image=localhost/tianer-tshark:latest

EnvironmentFile=/etc/tianer/tianer.env
EnvironmentFile=/etc/tianer/blesniff.env
Environment=BLESNIFF_SNIFFER_NAME=%i
# BLESNIFF_DLT set per-instance: 256 for ubertooth, 272 for nrf

Pod=tianer-capture.pod
Network=none

Volume=/etc/tianer:/etc/tianer:ro           # V01
Volume=/var/run/tianer:/var/run/tianer:rw   # V03
Volume=/var/log/tianer:/var/log/tianer:rw   # V04
Volume=/tmp:/tmp:rw                          # V08 (tmpfs)

AddCapability=CAP_NET_RAW
AddCapability=CAP_NET_ADMIN
DropCapability=ALL

NoNewPrivileges=true
GroupAdd=keep-groups

Exec=/usr/local/lib/tianer/wrap/tshark-entrypoint.sh

# Restart [6]
Restart=on-failure
RestartSec=5
StartLimitBurst=5

[Unit]
Description=Tian'er tshark for Sniffer %i
After=blesniff-sniffer-%i.service
Requires=blesniff-sniffer-%i.service
Wants=tianer-capture.pod

[Install]
WantedBy=blesniff.target
```

### 11.2 USB Device Mapping

Each sniffer instance maps to a specific `/dev/tianer/` symlink. The Quadlet files use per-instance override (drop-in) files to specify the correct `--device` mapping. The mapping table is:

| Instance `%i` | Device symlink | USB VID:PID | Hardware |
|---------------|---------------|-------------|----------|
| `ut1` | `/dev/tianer/ubertooth0` | `1d50:6002` | Ubertooth One |
| `nrf1` | `/dev/tianer/nrf0` | `1915:522a` / `1915:520f` | nRF52840 Dongle #1 |
| `nrf2` | `/dev/tianer/nrf1` | `1915:522a` / `1915:520f` | nRF52840 Dongle #2 |
| `nrf3` | `/dev/tianer/nrf2` | `1915:522a` / `1915:520f` | nRF52840 Dongle #3 |

The `--group-add keep-groups` directive ensures that the `plugdev` and `dialout` supplementary group memberships of the `tianer` user are preserved inside the container, granting access to the USB device nodes.

### 11.3 Volume Mounts Summary

| Volume ID | Host Path | Container Path | Mode | Purpose |
|-----------|-----------|---------------|------|---------|
| V01 | `/etc/tianer/` | `/etc/tianer/` | `:ro` | Config, secrets, sniffers.yaml |
| V02 | `/var/lib/tianer/pcap/` | `/var/lib/tianer/pcap/` | `:rw` (sniffer), `:ro` (tshark) | PCAP file storage |
| V03 | `/var/run/tianer/` | `/var/run/tianer/` | `:rw` | FIFOs (tmpfs) |
| V04 | `/var/log/tianer/` | `/var/log/tianer/` | `:rw` | Structured logs |
| V08 | (tmpfs) | `/tmp/` | `:rw` | Per-container ephemeral temp |

**Note on V02 access mode:** The sniffer container mounts V02 `:rw` for writing PCAP. The tshark container does not need V02 (it reads PCAP via the FIFO, not directly from disk). The tshark container's volume list above omits V02.

### 11.4 tmpfiles.d Update for Ingest FIFOs

The `CAPTURE-2` contract introduces ingest FIFOs (`<name>-ingest.fifo`) that are not currently in C01's `tmpfiles.d` configuration. C01 must be updated to include these additional named pipes:

```
# /etc/tmpfiles.d/tianer.conf — additional ingest FIFOs
p      /var/run/tianer/ut1-ingest.fifo     0660 tianer    tianer    -   -
p      /var/run/tianer/nrf1-ingest.fifo    0660 tianer    tianer    -   -
p      /var/run/tianer/nrf2-ingest.fifo    0660 tianer    tianer    -   -
p      /var/run/tianer/nrf3-ingest.fifo    0660 tianer    tianer    -   -
```

These FIFOs are created at boot by `systemd-tmpfiles --create` and recreated after every host reboot, ensuring the tshark containers always have a writable output pipe.

### 11.5 Container Image Requirements

**Sniffer container image:**
- Base: Debian Trixie slim (ARM64)
- Packages: `ubertooth-tools`, `nrfutil` (Nordic command-line tools), `tshark`, `wireshark-common`, `coreutils` (for `tail`), `yq` (YAML parser)
- Entrypoint: `/usr/local/lib/tianer/wrap/sniffer-entrypoint.sh`
- Target size: ≤ 180 MB compressed

**tshark container image:**
- Base: Debian Trixie slim (ARM64)
- Packages: `tshark`, `wireshark-common`, `coreutils`
- Entrypoint: `/usr/local/lib/tianer/wrap/tshark-entrypoint.sh`
- Target size: ≤ 120 MB compressed

### 11.6 Build and Install Paths

| Artifact | Source Path | Install Path |
|----------|-------------|-------------|
| ubertooth-wrap.sh | `modules/bluetooth/sniffers/ubertooth-wrap.sh` | `/usr/local/lib/tianer/wrap/ubertooth-wrap.sh` |
| nrf-wrap.sh | `modules/bluetooth/sniffers/nrf-wrap.sh` | `/usr/local/lib/tianer/wrap/nrf-wrap.sh` |
| tshark-wrap.sh | `modules/bluetooth/sniffers/tshark-wrap.sh` | `/usr/local/lib/tianer/wrap/tshark-wrap.sh` |
| heartbeat.sh | `modules/bluetooth/sniffers/heartbeat.sh` | `/usr/local/lib/tianer/wrap/heartbeat.sh` |
| tail-feed.sh | `modules/bluetooth/sniffers/tail-feed.sh` | `/usr/local/lib/tianer/wrap/tail-feed.sh` |
| sniffer-entrypoint.sh | `modules/bluetooth/sniffers/sniffer-entrypoint.sh` | `/usr/local/lib/tianer/wrap/sniffer-entrypoint.sh` |
| tshark-entrypoint.sh | `modules/bluetooth/sniffers/tshark-entrypoint.sh` | `/usr/local/lib/tianer/wrap/tshark-entrypoint.sh` |
| sniffers.yaml | `deploy/config/sniffers.yaml` | `/etc/tianer/sniffers.yaml` |
| Quadlet `.container` files | `deploy/containers/` | `/etc/containers/systemd/` |

### 11.7 Instance Scaling

The template design (`@` notation) allows adding or removing sniffer instances without modifying code:

1. **Add a new nRF sniffer:** Edit `sniffers.yaml` to add a new entry with `enabled: true`. Deploy a new Quadlet drop-in for the USB device mapping. Run `systemctl --user daemon-reload && systemctl --user start blesniff-sniffer@nrf4`.
2. **Disable a sniffer:** Set `enabled: false` in `sniffers.yaml`. Run `systemctl --user stop blesniff-sniffer@nrf2 && systemctl --user disable blesniff-sniffer@nrf2`.
3. **Change channel:** Edit `sniffers.yaml` channels field. Run `systemctl --user restart blesniff-sniffer@ut1`.

No container image rebuild is required for configuration changes.

---

## References

### External Sources

[1] Bluetooth SIG. "Bluetooth Core Specification v5.4, Vol 6, Part A — Physical Layer." https://www.bluetooth.com/specifications/specs/core-specification-5-4/, 2023.
  - §A.1: BLE advertising channel mapping — channel 37→2402 MHz, 38→2426 MHz, 39→2480 MHz.
  - Vol 6, Part B, §2.3: Advertising channel PDU header format (PDU Type, TxAdd, RxAdd, ChSel bit fields).

[2] The Tcpdump Group. "Link-Layer Header Types." https://www.tcpdump.org/linktypes.html, 2024.
  - `LINKTYPE_BLUETOOTH_LE_LL_WITH_PHDR` (DLT 256): Bluetooth Low Energy link-layer packets with pseudo-header (Ubertooth One).
  - `LINKTYPE_NORDIC_BLE` (DLT 272): Nordic Semiconductor nRF Sniffer for Bluetooth LE packets.

[3] Wireshark Foundation. "tshark(1) — Dump and Analyze Network Traffic." https://www.wireshark.org/docs/man-pages/tshark.html, 2024.
  - `-T fields`: Output only requested fields.
  - `-e <field>`: Add a field to the list of fields to display.
  - `-E separator=, quote=, occurrence=`: Control field printing format.
  - `-l`: Flush output after each packet (line-buffered).

[4] Nordic Semiconductor. "nRF Sniffer for Bluetooth LE — User Guide." https://docs.nordicsemi.com/bundle/ug_sniffer_ble/page/UG/sniffer_ble/intro.html, 2023.
  - `nrfutil ble-sniffer`: Command-line tool for nRF52840 BLE packet capture.
  - Channel hopping across BLE advertising channels 37, 38, 39.
  - USB PID: 1915:522A (sniffer mode), 1915:520f (DFU/bootloader mode).

[5] Great Scott Gadgets. "Ubertooth — Software, Firmware, and Hardware Designs." https://github.com/greatscottgadgets/ubertooth, 2020.
  - `ubertooth-btle`: BLE advertising capture tool for Ubertooth One.
  - Single-channel capture at a time (hardware limitation).
  - Output in PCAP format with DLT 256.

[6] Freedesktop.org. "systemd.service(5) — Service Unit Configuration." https://www.man7.org/linux/man-pages/man5/systemd.service.5.html, 2024.
  - `Restart=`: Configures whether the service shall be restarted (no, on-success, on-failure, on-abnormal, on-watchdog, on-abort, always).
  - `RestartSec=`: Time to sleep before restarting a service.
  - `StartLimitBurst=`, `StartLimitIntervalSec=`: Unit start rate limiting.

### Internal Project References

The following project-internal documents provide further design context and decisions:

- `component-breakdown.md` §5.2 C03 — Component artifact requirements for C03.
- `storage-strategy.md` — Decoupled capture architecture, backpressure isolation, V02/V03/V08.
- `c01-platform-infrastructure.md` — USB udev symlinks, filesystem layout, tmpfiles, Quadlet prerequisites.
- `c02-database.md` — `bluetooth.sniffer_heartbeat` table, DB schema.
- `c04-pcap-rotation.md` — Rotation coordination via SIGHUP (ROTATION-1).
- `c05-ingest-bridge.md` — Ingest FIFO reader, CAPTURE-2 contract consumer.
- `ADR-0001` — Archived design decisions:
  - D-03: nRF USB PID: 1915:522A (sniffer) + 1915:520f (DFU).
  - D-05: Channel strategy — configurable per sniffer, default ch37.
  - D-11: Heartbeat — local file primary, DB secondary.
  - D-12: tshark — per-DLT parameterised configuration.
  - Q4: USB device symlink strategy — persistent `/dev/tianer/` names.
  - Q6: Single-dongle channel strategy — configurable per sniffer.
  - Q7: CRC-24 verification — deferred to C07 Deep Parser.
