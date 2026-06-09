# C04 — PCAP Rotation

**Status:** Phase A Design — guides Phase B implementation
**Author:** Code Architect
**Date:** 2026-06-09
**Dependencies:** C03 (Capture Pipeline — PCAP file location, SIGHUP coordination, heartbeat directory)
**Blocks:** C06 (Gap Detector — reads rotated PCAP files for backfill), C07 (Deep Parser — reads rotated PCAP files for dissection)

---

## 1. Overview

### 1.1 Purpose

C04 PCAP Rotation manages the lifecycle of raw packet capture files on persistent storage (V02). It rotates the active `capture.pcap` file to a timestamped archive name, compresses the archived file with zstd to reduce disk consumption, and purges files older than the configured retention period. Rotation is coordinated with the sniffer process via SIGHUP to ensure zero data loss during the rename boundary.

### 1.2 Scope

C04 covers the **post-capture file lifecycle**: rename, compress, and delete. In scope:

| In Scope | Out of Scope |
|----------|-------------|
| Rotation script (`rotate-pcap.sh`) | Writing PCAP (C03 sniffer wrapper) |
| SIGHUP coordination with sniffer wrapper | Reading PCAP for backfill (C06) |
| zstd compression of rotated files | Reading PCAP for deep parsing (C07) |
| Retention-based deletion | PCAP integrity verification (storage-strategy.md Layer 5, runs independently) |
| systemd timer or Quadlet oneshot scheduling | Database schema or ingest (C02, C05) |
| Per-sniffer rotation directories | BLE protocol dissection (C07) |

### 1.3 Boundaries

C04 shares V02 (`/var/lib/tianer/pcap/`) with C03, C06, and C07. It writes (rename, compress, delete) while C06 and C07 read (`:ro`). The rotation script communicates with C03 sniffer wrappers through the ROTATION-1 contract: send SIGHUP, await acknowledgment, then rename.

### 1.4 Position in the System

```
┌──────────────────────────────────────────────────────────────┐
│                    C03 CAPTURE PIPELINE                       │
│  sniffer wrapper → capture.pcap (V02)                        │
│  tail-feed → FIFO (V03)                                      │
│  tshark → ingest FIFO                                        │
└──────────────┬──────────────────────────┬────────────────────┘
               │ SIGHUP (ROTATION-1)      │ PCAP directory
               ▼                          ▼
┌──────────────────────────┐  ┌────────────────────────────────┐
│  C04 PCAP ROTATION        │  │  C06 GAP DETECTOR              │
│  rename → compress →      │  │  reads rotated PCAP (:ro)       │
│  delete (retention)       │  │  backfills DB                  │
│                           │  └────────────────────────────────┘
│  V02 (:rw)                │
└──────────────────────────┘  ┌────────────────────────────────┐
                              │  C07 DEEP PARSER                │
                              │  reads rotated PCAP (:ro)       │
                              │  writes JSONL → V05             │
                              └────────────────────────────────┘
```

C04 is **Layer 2** in the build sequence (component-breakdown.md §4.1). It depends on C03 providing the PCAP directory layout and SIGHUP handler. It blocks C06 and C07, which read the rotated files. C04 does not block the critical path (C01 → C02 → C03 → C05 → C09 → C10) for MVP, but is required for production operation — without rotation, the disk fills and capture stops (F-C03-7).

### 1.5 Design Decisions Incorporated

| Decision | Source | Implementation |
|----------|--------|---------------|
| PCAP is source of truth; rotation must never lose data | storage-strategy.md | SIGHUP coordination: sniffer closes, rotation renames, sniffer opens new file |
| Decoupled capture: sniffer writes only to disk | C03 §2.2 | Rotation operates on files, never on the sniffer process directly |
| V02 bind-mount `/var/lib/tianer/pcap/` shared with sniffer pod | storage-strategy.md V02 | Rotation run on host or in platform pod with V02 `:rw` |
| Rotation naming: `YYYYMMDD-HHMM.pcap` then `.pcap.zst` | component-breakdown.md C04 artifact requirements | §4.2 naming convention |
| Rotation trigger: systemd timer or Quadlet oneshot | component-breakdown.md §5.2 | §4.1 scheduling |
| 14-day retention (source of truth window) | storage-strategy.md §5 recovery SLA | §9.2 `BLESNIFF_RETENTION_DAYS` |

---

## 2. High-Level Architecture (HLA)

### 2.1 Rotation Lifecycle

The rotation lifecycle has four phases per sniffer per rotation interval:

```
                        SYSTEMD TIMER FIRES
                              │
                              ▼
                    ┌─────────────────────┐
                    │ 1. SIGNAL SNIFFER    │
                    │    Send SIGHUP to    │
                    │    sniffer wrapper   │
                    └─────────┬───────────┘
                              │
                              ▼
                    ┌─────────────────────┐
                    │ 2. WAIT / RENAME    │
                    │    Wait for sniffer  │
                    │    to close FD.      │
                    │    rename(2) capture │
                    │    .pcap → timestamp.│
                    │    pcap              │
                    └─────────┬───────────┘
                              │
                              ▼
                    ┌─────────────────────┐
                    │ 3. COMPRESS         │
                    │    zstd --rm        │
                    │    timestamp.pcap   │
                    │    → timestamp.pcap │
                    │    .zst             │
                    └─────────┬───────────┘
                              │
                              ▼
                    ┌─────────────────────┐
                    │ 4. PURGE            │
                    │    find + delete    │
                    │    files older than │
                    │    retention days   │
                    └─────────────────────┘
```

### 2.2 SIGHUP-Safe Rotation Protocol

The rotation script must coordinate with the sniffer wrapper to avoid data loss during the rename. The protocol is:

1. **Send SIGHUP** to the sniffer wrapper PID (read from PID file, not `pgrep`).
2. **Wait for acknowledgment:** The sniffer wrapper, on receiving SIGHUP, closes the current `capture.pcap` file descriptor and writes a marker file (`/var/run/tianer/<name>.rotated`) to signal completion. (Note: This marker file is on V03 tmpfs, which is shared within the capture pod.)
3. **Rename the old file:** `rename(capture.pcap, YYYYMMDD-HHMM.pcap)`. The `rename(2)` syscall is atomic per POSIX: "If newpath already exists, it will be atomically replaced" [1]. This means the sniffer's new `capture.pcap` (opened after the SIGHUP handler returns) is never at risk of collision.
4. **Clean up marker:** Remove the `.rotated` marker file.
5. **Compress:** `zstd --rm YYYYMMDD-HHMM.pcap` produces `YYYYMMDD-HHMM.pcap.zst`.

**Why SIGHUP and not SIGUSR1?** SIGHUP is the standard Unix signal for "configuration reload" or "log rotation." The sniffer wrapper interprets SIGHUP as "close and reopen the output file." This follows the well-established log rotation convention used by daemons like Apache, nginx, and syslog-ng [2].

**Race condition analysis:** The critical window is between the sniffer closing `capture.pcap` and C04 renaming it. During this window:
- tail-feed (`tail -F`): Uses `tail -F` (capital F), which tracks by name, not inode. When `capture.pcap` is renamed, `tail` detects the file disappearance, waits, and when a new `capture.pcap` appears, reopens it automatically. The PCAP stream may pause briefly (< 1 second) but no data is lost.
- Other readers (C06, C07): These components read **rotated** files only, never the active `capture.pcap`. They are unaffected by rotation timing.

### 2.3 Sniffer Wrapper SIGHUP Handler

The sniffer wrapper must implement a SIGHUP handler. The handler logic (implemented in C03 §3.5, referenced here for contract clarity):

```
ON SIGHUP:
  1. Close the current capture.pcap file descriptor.
     (ubertooth-btle and nrfutil both support SIGHUP-initiated clean shutdown.)
  2. Write marker: echo "done" > /var/run/tianer/<name>.rotated
  3. Open a new capture.pcap with a fresh PCAP global header.
  4. Resume capture to the new file.
```

If the sniffer binary (`ubertooth-btle` or `nrfutil ble-sniffer`) does not natively handle SIGHUP with file re-open, the wrapper must trap SIGHUP, kill and restart the sniffer process, directing its output to the new file. In either case, the **contract** is that the `.rotated` marker is written only after the old `capture.pcap` FD is closed.

### 2.4 Interaction with tail-feed

The tail-feed process in the sniffer container uses `tail -c +0 -F capture.pcap`. The `-F` flag (capital F) is critical:

- `-f` (lowercase) follows by **inode**. When the file is renamed via `rename(2)`, the inode remains the same but the name changes — `tail -f` continues reading the now-renamed file until EOF, then exits.
- `-F` (uppercase) follows by **name**. When `capture.pcap` disappears (renamed), `tail -F` detects this, sleeps briefly, and checks for a new `capture.pcap`. When one appears (sniffer's new output file), it reopens it [3].

This means tail-feed automatically bridges the rotation boundary without any explicit coordination from C04. The brief pause during rotation is absorbed by the FIFO buffer (64 KB default, ~15-30 BLE packets at 30 bytes each).

### 2.5 Compression Strategy

Rotated PCAP files are compressed with **zstd** using default compression level 3. Rationale:

| Factor | Analysis |
|--------|----------|
| **Compression speed** | Level 3 is the zstd default. On Raspberry Pi CM5 (ARM Cortex-A76, 4 cores at 2.4 GHz), zstd achieves > 200 MB/s per core for compression on fast modes [4]. A typical 30-minute PCAP file (~50-200 MB of raw BLE advertising traffic at ~1000 pps) compresses in under 5 seconds. |
| **Decompression speed** | > 500 MB/s per core [4]. C06 and C07 decompress on read; this is negligible overhead compared to PCAP parsing and BLE dissection. |
| **Compression ratio** | BLE advertising PCAP is highly compressible — repetitive access addresses, similar payloads, sparse RF metadata. Expect 3:1 to 5:1 ratio with default settings, reducing 200 MB PCAP to ~50-70 MB `.pcap.zst`. |
| **Incremental/streaming** | zstd frames are independent. C06 and C07 can seek and decompress individual frames without decompressing the entire file. This enables efficient random access for gap backfill and deep parsing. |
| **Integrity** | zstd includes a 32-bit checksum in each frame (enabled by default with `-C`). Corrupted compressed data is detected on decompression [4]. |

**Trade-off — higher compression levels:** Levels 10-19 increase compression ratio but decrease speed dramatically (speed halves every 2 levels [4]). Since PCAP files are intermediate artifacts (source of truth, but eventually deleted after 14 days), compression speed is prioritized over ratio. The operator may override the level via `BLESNIFF_ZSTD_LEVEL`.

**Multi-threading:** `zstd -T0` uses all available CPU cores. On the CM5 (4 cores), this parallelizes compression across cores, reducing wall-clock compression time by ~3× for large files. This is the default in the rotation script.

### 2.6 Scheduling

Rotation is triggered by a **systemd timer** running on the host (not inside a container). The timer fires at a fixed interval, default 30 minutes (`BLESNIFF_ROTATION_MINUTES`).

**Host-level scheduling rationale:** Rotation is a host-level operation — it accesses the host filesystem (V02 bind-mount) and signals processes (sniffer wrapper). Running it on the host (via systemd timer + service) or in the `tianer-platform` pod (via Quadlet oneshot container) avoids the complexity of cross-container signal delivery and filesystem access. The rotation script only needs access to V02 (`:rw`) and the ability to signal processes via their host PID.

**Alternative: Quadlet oneshot container.** A `blesniff-rotate.container` Quadlet with `Type=oneshot` can be placed in the `tianer-platform` pod (which has V02 `:rw`). This keeps all Tian'er logic containerized. However, signaling processes across pod boundaries requires the host PID namespace (`--pid=host`), which is a security trade-off. The deployment notes (§11) provide both options.

### 2.7 Neighbouring Components

| Direction | Component | Mechanism | Protocol |
|-----------|-----------|-----------|----------|
| **Upstream** | C03 Capture Pipeline | Signal (SIGHUP) + file marker (`.rotated`) | ROTATION-1 contract |
| **Downstream** | C06 Gap Detector | File read (V02 `:ro`) | Decompressed PCAP (libpcap) |
| **Downstream** | C07 Deep Parser | File read (V02 `:ro`) | Decompressed PCAP (libpcap) |
| **Lateral** | C12 Service Orchestration | systemd timer unit | systemd.unit activation |
| **Lateral** | C13 Observability | Metrics endpoint + structured logs | Prometheus text format + Loki |

---

## 3. Data Model

### 3.1 PCAP File Naming Convention

Rotated PCAP files follow a deterministic naming convention that encodes the rotation timestamp:

```
Format:  YYYYMMDD-HHMM.pcap
Example: 20260609-1430.pcap
         │       │
         │       └── Hour and minute of rotation (UTC, 24-hour)
         └────────── Year, month, day of rotation (UTC)
```

After compression:
```
Format:  YYYYMMDD-HHMM.pcap.zst
Example: 20260609-1430.pcap.zst
```

**Design rationale:**
- **UTC timestamps** avoid timezone ambiguity. All timestamps in the Tian'er platform are UTC.
- **Sortable lexicographically:** The `YYYYMMDD-HHMM` format sorts correctly as a string. `ls` output matches chronological order without extra flags.
- **No sub-minute precision:** Rotation granularity is at the minute level (the finest resolution of the timer). A rotation at 14:30:27 still produces `20260609-1430.pcap`.
- **No sniffer name in filename:** The sniffer name is in the directory path (`/var/lib/tianer/pcap/ut1/`). Embedding it in the filename would break the GAP-2 contract which maps directory to sniffer identity.

### 3.2 Directory Layout

```
/var/lib/tianer/pcap/
├── ut1/
│   ├── capture.pcap                  # Active: being written by sniffer wrapper
│   ├── 20260609-1400.pcap.zst        # Rotated + compressed: 14:00-14:30 window
│   ├── 20260609-1430.pcap.zst        # Rotated + compressed: 14:30-15:00 window
│   ├── 20260609-1500.pcap            # Rotated, awaiting compression (in-progress)
│   └── ...
├── nrf1/
│   ├── capture.pcap
│   └── ...
├── nrf2/
│   ├── capture.pcap
│   └── ...
└── nrf3/
    ├── capture.pcap
    └── ...
```

**Key invariants:**
1. Exactly one `capture.pcap` per active sniffer directory — the current write target.
2. All other `.pcap` files are rotated archives (compressed or awaiting compression).
3. No `.pcap.zst` file may be written to directly — `.zst` files are the result of `zstd --rm` on a `.pcap` file.
4. A `.pcap` file without a corresponding `.pcap.zst` indicates compression was interrupted — the rotation script retries on next invocation.

### 3.3 PCAP File Structure Reference

Each file is a valid PCAP savefile per the libpcap file format [5]:

**Global header (24 octets):**

| Offset | Size | Field | Value (for Tian'er) |
|--------|------|-------|---------------------|
| 0x00 | 4 | Magic number | `0xa1b2c3d4` (microsecond timestamps) |
| 0x04 | 2 | Major version | 2 |
| 0x06 | 2 | Minor version | 4 |
| 0x08 | 4 | Reserved1 | 0 |
| 0x0C | 4 | Reserved2 | 0 |
| 0x10 | 4 | Snapshot length | 256 (BLE advertising packets fit in 47 bytes max) |
| 0x14 | 4 | Link-layer type | 251 (Ubertooth) or 195 (nRF52840 Sniffer) |

**Per-packet header (16 octets):**

| Offset | Size | Field |
|--------|------|-------|
| 0x00 | 4 | Timestamp seconds (Unix epoch) |
| 0x04 | 4 | Timestamp microseconds |
| 0x08 | 4 | Captured packet length |
| 0x0C | 4 | Original (untruncated) packet length |

Each per-packet header is immediately followed by the raw packet data.

**C04's interaction with PCAP structure:** The rotation script does not parse PCAP files — it only renames, compresses, and deletes them. However, downstream components (C06, C07) depend on the global header DLT value for correct parsing. If a rotation inadvertently corrupts a file (e.g., truncation during compression), the DLT value in the global header is the first indication of corruption.

### 3.4 Compressed File Properties

| Property | Value |
|----------|-------|
| Compression algorithm | zstd (Zstandard) per RFC 8878 [6] |
| Default compression level | 3 (fast, good ratio) |
| Integrity check | 32-bit checksum per frame (enabled by default) |
| Multi-threading | `-T0` (all CPU cores) |
| Source file handling | `--rm` (remove after successful compression) |
| File extension | `.pcap.zst` |

### 3.5 Rotation State

The rotation script maintains minimal state on disk:

- **PID file:** `/var/run/tianer/rotate.lock` — prevents concurrent rotation runs. If the lock exists and the PID is still alive, a second rotation invocation exits immediately with a log warning.
- **Last rotation timestamp:** Stored in the timer unit's persistent state (systemd `Persistent=yes` for the `.timer` unit) to handle missed rotations after system suspend or reboot. No separate state file is needed.

---

## 4. Low-Level Architecture (LLA)

### 4.1 Rotation Script (`rotate-pcap.sh`)

The rotation script is a Bash script invoked by systemd timer or Quadlet oneshot container. It iterates over all active sniffer directories and performs the rotation lifecycle.

```bash
#!/usr/bin/env bash
# rotate-pcap.sh — Rotate, compress, and purge PCAP files.
# Invoked by systemd timer (host) or Quadlet oneshot (platform pod).

set -euo pipefail

PCAP_DIR="${BLESNIFF_PCAP_DIR:-/var/lib/tianer/pcap}"
RUN_DIR="${BLESNIFF_RUN_DIR:-/var/run/tianer}"
LOCK_FILE="${RUN_DIR}/rotate.lock"
ROTATION_MINUTES="${BLESNIFF_ROTATION_MINUTES:-30}"
RETENTION_DAYS="${BLESNIFF_RETENTION_DAYS:-14}"
ZSTD_LEVEL="${BLESNIFF_ZSTD_LEVEL:-3}"

# Structured logging helper
log() {
    local level="$1" msg="$2"
    printf 'TIANER | {"ts":"%s","level":"%s","component":"rotate","msg":"%s"}\n' \
        "$(date -Iseconds)" "$level" "$msg"
}

# Acquire lock to prevent concurrent rotation
# Use flock on a file descriptor for atomic lock acquisition
exec 200>"${LOCK_FILE}"
if ! flock -n 200; then
    log "WARN" "Another rotation is in progress (lock held). Exiting."
    exit 0
fi

# Iterate over sniffer directories
for sniffer_dir in "${PCAP_DIR}"/*/; do
    [[ -d "${sniffer_dir}" ]] || continue
    sniffer_name="$(basename "${sniffer_dir}")"
    current_file="${sniffer_dir}/capture.pcap"
    rotated_marker="${RUN_DIR}/${sniffer_name}.rotated"

    # Skip if no active capture file
    if [[ ! -f "${current_file}" ]]; then
        log "INFO" "No active capture.pcap for ${sniffer_name}. Skipping."
        continue
    fi

    # Skip if file is empty (no packets written)
    if [[ ! -s "${current_file}" ]]; then
        log "INFO" "Empty capture.pcap for ${sniffer_name}. Skipping."
        continue
    fi

    # Phase 1: Signal the sniffer wrapper
    pid_file="${RUN_DIR}/${sniffer_name}.pid"
    if [[ -f "${pid_file}" ]]; then
        sniffer_pid="$(<"${pid_file}")"
        if kill -0 "${sniffer_pid}" 2>/dev/null; then
            log "INFO" "Sending SIGHUP to ${sniffer_name} (PID ${sniffer_pid})."
            kill -HUP "${sniffer_pid}"

            # Phase 2: Wait for acknowledgment (sniffer closed old FD)
            local wait_start
            wait_start="$(date +%s)"
            local timeout=10  # seconds
            while [[ ! -f "${rotated_marker}" ]]; do
                if (( $(date +%s) - wait_start > timeout )); then
                    log "ERROR" "Timeout waiting for ${sniffer_name} rotation marker."
                    break
                fi
                sleep 0.1
            done
            rm -f "${rotated_marker}"
        else
            log "WARN" "${sniffer_name} PID ${sniffer_pid} not alive. Rotating without signal."
        fi
    else
        log "WARN" "No PID file for ${sniffer_name}. Rotating without signal."
    fi

    # Phase 2b: Rename the old file
    rotation_ts="$(date -u +%Y%m%d-%H%M)"
    rotated_file="${sniffer_dir}/${rotation_ts}.pcap"

    if [[ -f "${current_file}" ]]; then
        # rename(2) is atomic on the same filesystem [1]
        mv "${current_file}" "${rotated_file}"
        log "INFO" "Rotated ${sniffer_name}/capture.pcap → ${rotation_ts}.pcap"
    fi

    # Phase 3: Compress (backgrounded to avoid blocking other sniffers)
    (
        if [[ -f "${rotated_file}" ]]; then
            zstd -"${ZSTD_LEVEL}" -T0 --rm -q "${rotated_file}"
            log "INFO" "Compressed ${sniffer_name}/${rotation_ts}.pcap → .pcap.zst"
        fi
    ) &
done

# Wait for all background compression jobs
wait

# Phase 4: Purge expired files
find "${PCAP_DIR}" -type f \( -name '*.pcap.zst' -o -name '*.pcap' \) \
    ! -name 'capture.pcap' \
    -mtime "+${RETENTION_DAYS}" \
    -print -delete 2>/dev/null |
while IFS= read -r deleted_file; do
    log "INFO" "Purged: ${deleted_file}"
done

log "INFO" "Rotation cycle complete. Duration: ${SECONDS}s."

# Lock released automatically on script exit (flock on FD 200)
exit 0
```

### 4.2 Key Algorithmic Decisions

#### 4.2.1 Atomic Rename

The script uses `mv` (which invokes `rename(2)`) to rename `capture.pcap` to the timestamped name. Per the Linux man-pages: "If newpath already exists, it will be atomically replaced, so that there is no point at which another process attempting to access newpath will find it missing" [1]. This means:

- The sniffer's new `capture.pcap` (opened after the SIGHUP handler completes) never collides with the rename — `rename(2)` atomically swaps the directory entry.
- tail-feed (`tail -F`) sees `capture.pcap` disappear and reappear as an atomic event at the filesystem level. No partial reads occur.
- The rotation window (when `capture.pcap` is absent) is bounded by: (SIGHUP handler close time) + (rename syscall time) + (sniffer reopen time). This is typically < 100 ms.

#### 4.2.2 Locking

The script uses `flock` on a lock file descriptor (FD 200) for mutual exclusion. `flock -n` is non-blocking — if the lock is held, the script exits immediately with a log warning. This prevents:

1. **Concurrent rotation runs:** A systemd timer misfire or manual invocation while rotation is in progress.
2. **Lock file staleness:** `flock` is kernel-managed — if the rotation process crashes, the kernel releases the lock automatically. No stale lock file to clean up.

#### 4.2.3 Parallel Compression

Compression (`zstd`) is launched as a background process (`&`) per sniffer. This allows compression of multiple sniffer output files to proceed in parallel. On a 4-core CM5 with 2-4 sniffer directories, all compression jobs complete within the same window.

If a sniffer directory produces a very large file (e.g., high-traffic environment, 500+ MB in 30 minutes), the `zstd -T0` flag already parallelizes internal compression across cores. Backgrounding across sniffers adds a second level of parallelism.

#### 4.2.4 Retention Purging

Purge uses `find ... -mtime +${RETENTION_DAYS} -delete`. The `-mtime` test uses the file's last modification time (mtime), which for a rotated file is the time it was written by the sniffer (i.e., the rotation timestamp). The `+N` operator matches files modified more than N days ago.

**Why mtime, not filename date?** The filename encodes the rotation trigger time, but the file's mtime reflects the actual capture window. If rotation is delayed (e.g., the timer fires late), mtime preserves the correct retention boundary. Both `.pcap` and `.pcap.zst` files are purged — if a `.pcap` file hasn't been compressed yet but is past retention, it is deleted anyway (the data is beyond the source-of-truth window).

### 4.3 Signal Coordination Detail

The SIGHUP protocol between C04 and C03 requires a **PID file** (`/var/run/tianer/<sniffer_name>.pid`) written by the sniffer wrapper on startup. This PID file is on V03 (tmpfs), accessible to both the sniffer container and the host (if rotation runs on host) or to other containers in the pod (if rotation runs in platform pod).

**PID file format:**
```
12345
```

A single line containing the decimal PID of the sniffer wrapper process (or the container's init process, if the wrapper runs as PID 1).

**Marker file format:**
```
done
```

The marker file `/var/run/tianer/<sniffer_name>.rotated` signals that the sniffer has closed the old PCAP file descriptor. Its content is not meaningful — only its existence matters.

**Timeout handling:** If the sniffer does not write the marker within 10 seconds, the rotation script proceeds with the rename anyway (logging an ERROR). If the sniffer is truly dead, its PID is absent and the script proceeds without signaling (WARN). If the sniffer is hung, the 10-second timeout prevents indefinite blocking of the rotation cycle. The rotated file may have the last write at the signal time rather than a clean close — the PCAP global header is still valid, and the per-packet headers are intact up to the last successful write.

### 4.4 Compression Error Handling

`zstd --rm` removes the source file only after successful compression. If compression fails (disk full, memory allocation failure, file corruption), the source `.pcap` file is preserved. The rotation script logs an ERROR and moves on to the next sniffer — the uncompressed file remains and is retried on the next rotation cycle.

**Partial compression output:** If `zstd` is killed mid-compression (OOM, signal), a partial `.pcap.zst` file may exist. On the next rotation cycle, the script compresses the source `.pcap` again. The `--rm` flag will remove the old source file on successful compression. The pre-existing partial `.zst` is overwritten.

### 4.5 Interaction with Heartbeat Directory

The rotation script shares the V02 directory tree with the heartbeat directory (`/var/lib/tianer/heartbeat/`). These are adjacent subdirectories under `/var/lib/tianer/`:

```
/var/lib/tianer/
├── pcap/
│   ├── ut1/
│   ├── nrf1/
│   └── ...
├── heartbeat/
│   ├── ut1.ts
│   ├── nrf1.ts
│   └── ...
└── data/
    └── ...
```

The `find` command in the purge phase is scoped to `"${PCAP_DIR}"` only — it does not traverse into `heartbeat/` or `data/`. Heartbeat files have their own lifecycle (always overwritten, never deleted).

---

## 5. Inter-Component Contracts

### 5.1 ROTATION-1 — PCAP Rotation Protocol

| Property | Value |
|----------|-------|
| **Contract ID** | ROTATION-1 |
| **From** | C04 (rotation script) |
| **To** | C03 (sniffer wrapper) |
| **Format** | POSIX signal (SIGHUP) + file marker (`.rotated`) + atomic rename (`rename(2)`) |
| **Transport** | Host process signal (kill) + V03 tmpfs marker file + V02 filesystem rename |
| **Guarantees** | Zero data loss during rotation. No partial writes in rotated files. Atomic rename prevents "file missing" windows for open readers. |
| **Failure mode** | If sniffer does not respond within 10s → proceed with rename anyway (PCAP truncated at last write, still valid). If sniffer PID not found → proceed without signaling. |
| **Pre-conditions** | Sniffer wrapper has SIGHUP handler registered. PID file at `/var/run/tianer/<name>.pid` exists and contains valid PID. |
| **Post-conditions** | `capture.pcap` renamed to `YYYYMMDD-HHMM.pcap`. Sniffer wrapper has opened a new `capture.pcap` with fresh global header. Marker file deleted. |

**Rotation signal flow:**

```
C04 rotate-pcap.sh                    C03 sniffer wrapper
     │                                      │
     │──── SIGHUP ──────────────────────►   │
     │                                      │ (trap handler fires)
     │                                      │ (close capture.pcap FD)
     │                                      │ (write .rotated marker)
     │◄─── .rotated marker detected ────    │
     │                                      │ (open new capture.pcap)
     │──── rename(capture.pcap,             │
     │      YYYYMMDD-HHMM.pcap) ────────►   │
     │                                      │ (resume capture to new file)
     │──── rm .rotated marker ──────────►   │
     │                                      │
```

**PCAP global header guarantee:** After rotation, the new `capture.pcap` must have a valid PCAP global header. The sniffer wrapper is responsible for writing this header before any packet data. This is normally handled by `ubertooth-btle` and `nrfutil ble-sniffer`, which write the global header at startup. If the wrapper restarts the sniffer process, the new process writes a fresh global header.

### 5.2 GAP-2 — PCAP File to Gap Window Mapping

| Property | Value |
|----------|-------|
| **Contract ID** | GAP-2 |
| **From** | C04 (filename convention) |
| **To** | C06 (Gap Detector) |
| **Format** | Directory path + filename convention |
| **Mapping** | `/var/lib/tianer/pcap/<sniffer_name>/YYYYMMDD-HHMM.pcap.zst` → capture window `[HH:MM, HH:MM+BLESNIFF_ROTATION_MINUTES)` |
| **Guarantees** | Filename encodes the lower bound of the capture window (the rotation start time). The upper bound is the next rotation timestamp. A rotated file contains all packets captured between the previous rotation and the current rotation. |
| **Decompression** | Gap detector reads compressed files via `zstd -d -c` and pipes to PCAP parser. |

### 5.3 DEEP-2 — PCAP File Access for Deep Parsing

| Property | Value |
|----------|-------|
| **Contract ID** | DEEP-2 (defined fully in C07 design document; referenced here for naming convention) |
| **From** | C04 (rotated PCAP files) |
| **To** | C07 (Deep Parser) |
| **Format** | Timestamped `.pcap.zst` files in per-sniffer directories |
| **Guarantees** | Files are complete and valid PCAP (libpcap format). Files older than retention period may be absent. |

---

## 6. Failure Modes & Recovery

### 6.1 Failure Catalog

| # | Failure Mode | Detection | Impact | Recovery |
|---|-------------|-----------|--------|----------|
| **F-C04-1** | **Sniffer unresponsive to SIGHUP** | Rotated marker not written within 10s timeout. `tianer_rotation_timeout_total` counter increments. Logged at ERROR level. | Rotation proceeds without clean file closure. Rotated PCAP file is truncated at the last write before the SIGHUP. No data loss from previous writes — all pre-SIGHUP packets are in the PCAP. The sniffer continues writing to the old FD (which is now linked to the renamed file) or the new FD. In the worst case, a few packets may be written to the old file after the rename. | If sniffer is alive but hung, operator restarts the sniffer container. Next rotation cycle succeeds. Gap detector (C06) detects any missing buckets and backfills. |
| **F-C04-2** | **Compression failure (disk full)** | `zstd` exits with non-zero status. Source `.pcap` file preserved (--rm is conditional on success). `tianer_rotation_compress_fail_total` counter increments. Logged at ERROR level. | Rotated `.pcap` file exists uncompressed. Disk consumption is ~3-5× higher than compressed. No data loss — the uncompressed PCAP is valid and readable by C06/C07. | C04 emergency purge removes oldest files. If disk is critically full (< 10% remaining), operator expands storage. Compression retried on next rotation cycle. |
| **F-C04-3** | **Disk full during purge** | Purge phase cannot delete files. `tianer_rotation_purge_fail_total` counter increments. | Old files accumulate beyond retention period. Disk consumption grows without bound. | Alert `TianerDiskFull` fires at 90%. Operator manually purges or expands storage. C04 continues attempting purge on each cycle. |
| **F-C04-4** | **Concurrent rotation detected** | `flock -n` fails (lock held). Script exits with log WARN. `tianer_rotation_concurrent_total` counter increments. | The second rotation invocation exits immediately. The first rotation continues normally. | No recovery needed — the lock prevents concurrent execution. The next timer event (one rotation interval later) runs normally. |
| **F-C04-5** | **Renamed file collision** | `mv` fails with EEXIST (file already exists). Logged at ERROR level. | The existing timestamped file (from a previous rotation at the same minute, e.g., manual invocation) is not overwritten. `mv` will not overwrite a non-directory with a non-directory without `-f`. | Operator investigates why two rotation cycles produced the same timestamp. Manual resolution: rename or remove the existing file. With `-f` flag considered for production but rejected — silent overwrites are worse than an ERROR. |
| **F-C04-6** | **zstd OOM kill** | `zstd` process killed by kernel OOM killer. `tianer_rotation_compress_fail_total` increments. | Partial `.pcap.zst` file may exist. Source `.pcap` preserved (OOM kill occurs before `--rm` executes). | Next rotation cycle: script compresses the source `.pcap` again. Pre-existing partial `.zst` is overwritten. If OOM is recurrent, reduce `BLESNIFF_ZSTD_LEVEL` or increase `ZSTD_NBTHREADS` (fewer threads → less memory). |
| **F-C04-7** | **Rotation script crash (segfault, unexpected exit)** | systemd timer detects exit code ≠ 0. `tianer_rotation_crash_total` increments. | Partial rotation: some sniffer directories may have been renamed but not compressed. `flock` released on process death (kernel cleanup). | Next timer event picks up where left off: uncompressed `.pcap` files will be compressed; uncompressed files past retention will be purged. No data loss — all writes happened before the crash. |
| **F-C04-8** | **PID file contains stale PID** | `kill -0 <pid>` returns 0 (PID exists but is a different process). Script sends SIGHUP to wrong process. | SIGHUP sent to an unrelated process. The effect depends on that process's SIGHUP handler — most daemons reload configuration; most user processes ignore SIGHUP or exit. | This scenario requires the PID to have been recycled between the PID file write and the rotation. PID recycling on Linux is slow (PID_MAX is 4,194,304 on 64-bit), and the sniffer wrapper updates the PID file on every startup. Mitigation: PID file is on tmpfs (V03), which is cleared on host reboot — stale PIDs can only occur within a single uptime cycle. |

### 6.2 Recovery Procedures

#### Rotation Timeout (F-C04-1)

```
1. Rotation script logs ERROR: "Timeout waiting for <sniffer_name> rotation marker."
2. Script proceeds with rename after 10s timeout.
3. Rotated PCAP file is valid up to the last pre-SIGHUP write.
4. Sniffer wrapper continues (hung or slow) — operator investigates.
5. Next rotation cycle retries normally — if sniffer recovered, marker is written promptly.
6. Gap detector (C06) identifies any data gap from the timeout window and backfills from the new capture.pcap or subsequent files.
```

#### Disk Full Recovery (F-C04-2, F-C04-3)

```
1. Alert "TianerDiskFull" fires (≥ 90% disk usage).
2. Rotation script logs ERROR for compression or purge failures.
3. Emergency purge: operator manually runs:
     find /var/lib/tianer/pcap -name '*.pcap.zst' -mtime +7 -delete
   (This reduces retention from 14 to 7 days to free space immediately.)
4. Operator expands storage (larger SD card, or mounts additional volume).
5. System recovers — next rotation cycle completes successfully.
```

### 6.3 Recovery SLA

| Scenario | Detection Time | Recovery Time | Data Loss |
|----------|---------------|---------------|-----------|
| SIGHUP timeout (F-C04-1) | < 10s (during rotation) | Manual investigation | Zero from PCAP; last 10s of writes may be in wrong file |
| Compression failure (F-C04-2) | < 1s (zstd exit code) | Next rotation cycle (≤ 30 min) | Zero — source `.pcap` preserved |
| Disk full (F-C04-3) | < 1 min (disk metrics) | Manual (operator expands storage) | Packets after disk full are lost at capture (F-C03-7) |
| Rotation crash (F-C04-7) | < 1s (exit code) | Next timer event (≤ 30 min) | Zero — source `.pcap` preserved |
| Concurrent rotation (F-C04-4) | < 1s (lock failure) | Self-clearing (next timer) | Zero — first rotation completes normally |

---

## 7. Observability

### 7.1 Metrics

| Metric Name | Type | Labels | Source | Description |
|------------|------|--------|--------|-------------|
| `tianer_rotation_cycles_total` | Counter | — | rotation script | Total number of rotation cycles completed |
| `tianer_rotation_duration_seconds` | Gauge | — | rotation script | Duration of the most recent rotation cycle |
| `tianer_rotation_files_rotated_total` | Counter | `sniffer` | rotation script | Number of PCAP files renamed per sniffer |
| `tianer_rotation_bytes_compressed_total` | Counter | `sniffer` | rotation script | Total uncompressed bytes compressed |
| `tianer_rotation_bytes_after_compression_total` | Counter | `sniffer` | rotation script | Total bytes after compression |
| `tianer_rotation_compress_duration_seconds` | Gauge | `sniffer` | rotation script | Duration of most recent compression per sniffer |
| `tianer_rotation_timeout_total` | Counter | `sniffer` | rotation script | Number of times rotation timed out waiting for sniffer marker |
| `tianer_rotation_compress_fail_total` | Counter | `sniffer` | rotation script | Number of compression failures |
| `tianer_rotation_purge_fail_total` | Counter | — | rotation script | Number of purge phase failures |
| `tianer_rotation_concurrent_total` | Counter | — | rotation script | Number of concurrent rotation attempts detected |
| `tianer_rotation_crash_total` | Counter | — | rotation script | Number of rotation script crashes |
| `tianer_pcap_file_count` | Gauge | `sniffer` | rotation script | Current number of PCAP files (active + archived) per sniffer |
| `tianer_pcap_disk_bytes` | Gauge | `sniffer` | rotation script | Total disk usage in bytes per sniffer directory |

### 7.2 Structured Logging

The rotation script emits logs in the Tian'er structured format:

```
TIANER | {"ts":"ISO8601","level":"LEVEL","component":"rotate","sniffer":"NAME","msg":"MESSAGE",...}
```

**Component value:** `rotate`

**Log levels used:**
- `INFO`: Normal operational events (rotation started, file renamed, compressed, purged, cycle complete)
- `WARN`: Recoverable issues (concurrent rotation detected, sniffer PID absent, empty capture file skipped)
- `ERROR`: Problems requiring attention (SIGHUP timeout, compression failure, purge failure, rename collision)

### 7.3 Health Checks

The rotation script is a oneshot process, not a daemon. Health is determined by:

1. **systemd timer last invocation status:** `systemctl status blesniff-rotate.timer` — shows `LastTrigger=` and result of the associated service unit.
2. **Prometheus metrics:** `tianer_rotation_crash_total` — if this increments, rotation has crashed.
3. **PCAP file count:** `tianer_pcap_file_count` growing unboundedly indicates purge failure.
4. **Uncompressed file count:** If `.pcap` files (without `.zst`) accumulate across multiple cycles, compression is failing.

### 7.4 Alert Rules

| Alert | Condition | Severity | Action |
|-------|-----------|----------|--------|
| `TianerRotationCrashed` | `tianer_rotation_crash_total` increases | **Critical** | Check rotation script logs; systemd timer may have disabled itself after StartLimitBurst |
| `TianerRotationTimeout` | `tianer_rotation_timeout_total` increase rate > 0 | **Warning** | Sniffer wrapper may be unresponsive; check container health |
| `TianerCompressFail` | `tianer_rotation_compress_fail_total` increase rate > 0 | **Critical** | Disk may be full; check V02 usage |
| `TianerPurgeFail` | `tianer_rotation_purge_fail_total` increase rate > 0 | **Critical** | Disk may be full or permissions broken |
| `TianerPcapFileCountHigh` | `tianer_pcap_file_count` > 750 (14 days × 48 rotations/day × max 4 sniffers) | **Warning** | Purge may be failing or retention period too long |
| `TianerDiskFull` | `tianer_host_disk_usage_percent{pcap} > 90` | **Critical** | Trigger emergency purge; expand storage |

---

## 8. Security Considerations

### 8.1 Signal Injection

The rotation script sends SIGHUP based on a PID read from a file on V03. An attacker with write access to V03 could overwrite the PID file, causing the rotation script to signal an arbitrary process.

**Attack vector:** Write a different PID (e.g., PID 1, `systemd`) into `/var/run/tianer/ut1.pid`. The rotation script sends SIGHUP to PID 1, which causes systemd to reload its configuration (its documented SIGHUP behavior [7]).

**Mitigation:**
1. V03 (`/var/run/tianer/`) is mode 0750, owner `tianer:tianer`. Only the `tianer` user and processes in the `tianer` group can write to it.
2. The PID file is written by the sniffer wrapper, which runs as the `tianer` user inside the container. No untrusted process has write access to V03.
3. Before signaling, the script verifies the PID corresponds to an expected process: `kill -0 $pid` confirms the PID exists. An additional check could read `/proc/$pid/cmdline` to verify it matches `ubertooth-btle` or `nrfutil`, but this is a defense-in-depth measure, not a primary mitigation.

### 8.2 PCAP File Permissions

Rotated PCAP files inherit the permissions of the sniffer wrapper's output file. The directory `/var/lib/tianer/pcap/` is mode 0750, owner `tianer:tianer`. Rotated files are readable only by `tianer` group members (sniffer containers, gap detector, deep parser) and root.

**Data sensitivity:** PCAP files contain BLE MAC addresses, service UUIDs, and manufacturer data (per C03 §8.4). After rotation and compression, the `.pcap.zst` files contain the same data in compressed form. Access is restricted to the `tianer` group.

### 8.3 Compression Side-Channel

zstd compression ratios could theoretically leak information about packet content (e.g., repetitive vs. random payloads compress differently). This is a low-risk concern for Tian'er — the platform operates on a trusted LAN, and PCAP files are not transmitted over the network. If PCAP files are ever transmitted (e.g., remote backup), the file-level permissions and transport encryption (TLS) provide confidentiality.

### 8.4 Host Process Signaling

If rotation runs on the host (systemd service), the script must be able to signal container processes. This requires:

1. **PID namespace sharing:** The sniffer container uses `--pid=host` (in Quadlet: `PodmanArgs=--pid=host`), making its PID file contain a host-visible PID.
2. **Signal permission:** The rotation script runs as `tianer` user (same UID as the container process). `kill()` to a process owned by the same UID succeeds.
3. **No root requirement:** The rotation script does not require root privileges. It runs as `tianer:tianer`.

**Trade-off of `--pid=host`:** Exposing the host PID namespace to a container increases the attack surface — a compromised container can see and potentially signal host processes. Mitigation:
- The capture pod has `Network=none` and `--cap-drop ALL` with only `CAP_NET_RAW` and `CAP_NET_ADMIN` added. No network access, no privilege escalation path.
- If the security trade-off is unacceptable, the alternative is to run rotation inside the platform pod as a Quadlet oneshot, which communicates with sniffer containers via shared V03 marker files (no PID signaling needed).

### 8.5 Shell Injection

The rotation script uses shell variables for directory paths. If `BLESNIFF_PCAP_DIR` or `BLESNIFF_RUN_DIR` is set to a malicious value, path traversal or command injection is possible.

**Mitigation:**
1. These variables are set via `EnvironmentFile=/etc/tianer/blesniff.env` (V01), which is mount `:ro` and owned by root.
2. Paths are validated: the script ensures `PCAP_DIR` starts with `/var/lib/tianer/pcap` before proceeding.
3. All variable references in file operations are quoted (`"${PCAP_DIR}"`) to prevent word splitting.

### 8.6 Container Capabilities (if Quadlet oneshot)

If rotation runs as a Quadlet container in the platform pod, it requires:
- `V02:/var/lib/tianer/pcap:rw` — write access for rename, compression, deletion
- `V03:/var/run/tianer:rw` — read PID files, write lock file
- `V04:/var/log/tianer:rw` — write structured logs
- No network, no USB, no special capabilities
- `--cap-drop ALL`, `NoNewPrivileges=true`

---

## 9. Configuration

### 9.1 Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `BLESNIFF_PCAP_DIR` | No | `/var/lib/tianer/pcap` | Root directory for PCAP files (V02) |
| `BLESNIFF_RUN_DIR` | No | `/var/run/tianer` | Runtime directory for PID files and lock (V03) |
| `BLESNIFF_ROTATION_MINUTES` | No | `30` | Interval in minutes between rotation cycles |
| `BLESNIFF_RETENTION_DAYS` | No | `14` | Number of days to retain compressed PCAP files |
| `BLESNIFF_ZSTD_LEVEL` | No | `3` | zstd compression level (1-19; higher = better ratio, slower) |
| `BLESNIFF_LOG_LEVEL` | No | `info` | Log verbosity: debug, info, warn, error |

### 9.2 Tuning Guidelines

#### Rotation Interval

The 30-minute default balances two concerns:
- **File granularity:** Files too large (> 500 MB) are slow to compress and unwieldy for backfill. Files too small (< 10 MB) create excessive file count overhead.
- **Backfill granularity:** The gap detector (C06) works at the file level — a 30-minute file means a gap of up to 30 minutes is the finest backfill granularity.

**Adjustment:** For high-traffic environments (BLE-dense urban areas with thousands of devices), reduce to 15 minutes. For low-traffic (rural, few devices), increase to 60 minutes to reduce file overhead.

#### Retention Period

14 days is the default source-of-truth window (storage-strategy.md). This aligns with the gap detector's backfill capability: any gap within 14 days can be recovered from PCAP.

**Disk sizing table:**

| Rotation Interval | Traffic (pps) | Avg Packet Size | Daily Volume (uncompressed) | Daily Volume (compressed, ~4:1) | 14-day Storage |
|-------------------|--------------|-----------------|----------------------------|--------------------------------|----------------|
| 30 min | 100 | 30 bytes | ~259 MB | ~65 MB | ~910 MB |
| 30 min | 1000 | 30 bytes | ~2.6 GB | ~650 MB | ~9.1 GB |
| 30 min | 5000 | 30 bytes | ~13 GB | ~3.25 GB | ~45.5 GB |
| 15 min | 1000 | 30 bytes | ~2.6 GB | ~650 MB | ~9.1 GB |
| 60 min | 1000 | 30 bytes | ~2.6 GB | ~650 MB | ~9.1 GB |

For the typical Tian'er deployment (2-4 sniffers, each seeing ~1000 pps in an urban environment), 14-day storage requires approximately 36 GB (4 sniffers × 9.1 GB). This fits comfortably on a 128 GB SD card alongside the OS and database.

#### Compression Level

| Level | Speed (relative) | Ratio (BLE PCAP, typical) | Memory (compression) |
|-------|-----------------|--------------------------|---------------------|
| 1 | 1× (fastest) | ~2.5:1 | ~2 MB |
| 3 (default) | 0.8× | ~4:1 | ~4 MB |
| 5 | 0.5× | ~4.5:1 | ~8 MB |
| 10 | 0.1× | ~5.5:1 | ~16 MB |
| 19 | 0.01× | ~6:1 | ~128 MB |

The default level 3 is recommended for the CM5. At 1000 pps and 30-minute rotation, a ~50 MB PCAP file compresses in ~0.5 seconds at level 3. Higher levels provide diminishing returns on BLE advertising data (which is mostly repetitive access addresses and similar payloads).

---

## 10. Test Plan

### 10.1 Unit Tests

Tested with bats-core. Test files in `modules/bluetooth/rotation/tests/`.

| Test Suite | File | Scope |
|-----------|------|-------|
| `test_rotate_sighup.bats` | SIGHUP protocol | Mock sniffer with SIGHUP handler; verify marker written; verify rotate renames file; verify new capture.pcap created |
| `test_rotate_compress.bats` | Compression | Verify `.pcap` → `.pcap.zst` with `--rm`; verify source removed only on success; verify decompress produces identical content |
| `test_rotate_purge.bats` | Retention purge | Create files with varying mtimes; verify only files older than retention deleted; verify active `capture.pcap` never purged |
| `test_rotate_locking.bats` | Locking | Launch two concurrent rotations; verify second exits without modifying files |
| `test_rotate_empty.bats` | Edge cases | Empty capture file (zero bytes) skipped; missing directory skipped; sniffer PID not found handled |
| `test_rotate_naming.bats` | Filename convention | Verify timestamp format matches `YYYYMMDD-HHMM.pcap`; verify lexicographic sortability |

### 10.2 Integration Tests

| Test | Scope | Method |
|------|-------|--------|
| `test_rotation_e2e.sh` | Full rotation cycle | Start mock sniffer (writes synthetic PCAP with timestamps), run rotation script, verify renamed file complete, verify compressed file, verify sniffer continues to new capture.pcap, verify tail-feed follows new file |
| `test_rotation_sniffer_crash.sh` | Sniffer unresponsive | Kill sniffer before rotation; verify rotation proceeds without marker; verify PCAP file valid |
| `test_rotation_disk_full.sh` | Disk full recovery | Fill test directory; verify compression fails gracefully; verify source preserved |
| `test_rotation_gap_detector.sh` | GAP-2 contract | Complete rotation cycle, verify gap detector can map filename to time window, verify gap detector can decompress and read packets |

### 10.3 Performance Tests

| Test | Scope | Threshold |
|------|-------|-----------|
| `test_compression_perf.sh` | Compression throughput | 200 MB PCAP compressed in < 10s at level 3 on CM5 |
| `test_rotation_duration.sh` | Full rotation cycle | < 30s for 4 sniffer directories with 50 MB PCAP each |
| `test_filesystem_overhead.sh` | File count overhead | 48 rotations/day × 14 days = 672 files per sniffer — `ls` completes in < 1s |

### 10.4 Acceptance Criteria

| Criterion | Measurement | Threshold |
|-----------|-------------|-----------|
| **Zero data loss on rotation** | PCAP byte count before rotation == after rotation (decompressed) | No bytes lost |
| **Atomic rename** | No partial reads by tail-feed during rotation | tail-feed produces continuous stream across rotation boundary |
| **Compression success rate** | `tianer_rotation_compress_fail_total` over 100 cycles | 100% success (0 failures) |
| **Purge correctness** | Files older than retention days deleted; files within retention preserved | 100% accuracy |
| **Concurrent safety** | Two simultaneous rotation invocations | One proceeds, one exits (no double-rename) |
| **Rotation cycle duration** | Wall-clock time for complete cycle | ≤ 30 seconds for typical deployment (4 sniffers, 50 MB each) |
| **Compression ratio** | Compressed size / uncompressed size | ≤ 0.35 (≥ 2.85:1 ratio) for real BLE traffic |

---

## 11. Deployment Notes

### 11.1 Option A: Host-Level systemd Timer

The simplest deployment — rotation runs directly on the host as a systemd service triggered by a timer.

**Service unit (`/etc/systemd/system/blesniff-rotate.service`):**

```ini
[Unit]
Description=Tian'er PCAP Rotation
After=blesniff.target
Requires=blesniff.target

[Service]
Type=oneshot
User=tianer
Group=tianer
EnvironmentFile=/etc/tianer/blesniff.env
ExecStart=/usr/local/lib/tianer/rotate-pcap.sh

# Security hardening
PrivateTmp=yes
ProtectSystem=strict
ProtectHome=yes
ReadWritePaths=/var/lib/tianer/pcap /var/run/tianer /var/log/tianer
NoNewPrivileges=yes

[Install]
WantedBy=blesniff.target
```

**Timer unit (`/etc/systemd/system/blesniff-rotate.timer`):**

```ini
[Unit]
Description=Tian'er PCAP Rotation Timer
Requires=blesniff-rotate.service

[Timer]
# Fire every ROTATION_MINUTES after the last activation completed
OnUnitActiveSec=1800
# Also fire 2 minutes after boot (to catch up if system was down)
OnBootSec=120
# Catch up on missed runs (e.g., system was suspended)
Persistent=yes
# Allow up to 30s of randomized delay to avoid thundering-herd
RandomizedDelaySec=30
# Accuracy: 1 second is fine for minute-level rotation
AccuracySec=1s

[Install]
WantedBy=timers.target
```

**Key timer options:**
- `OnUnitActiveSec=1800`: Fire 30 minutes after the last rotation **completed** (not started). If rotation takes 10 seconds, the next fires at 30:10, not 30:00. This prevents timer drift.
- `OnBootSec=120`: Fire 2 minutes after boot to run the first rotation. This gives the sniffer containers time to start and begin writing.
- `Persistent=yes`: If the system was down during the scheduled rotation, run immediately on boot. This catches up on missed purges [8].
- `RandomizedDelaySec=30`: Add up to 30 seconds of random jitter. If multiple Tian'er deployments run identical timers, this prevents them all hitting a remote filesystem simultaneously.
- `AccuracySec=1s`: Set to 1 second (from default 1 minute) to ensure rotation happens close to the scheduled time. PCAP rotation is not a power-saving operation — accuracy matters more than coalescing.

**Installation:**
```bash
sudo cp deploy/systemd/blesniff-rotate.{service,timer} /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now blesniff-rotate.timer
```

### 11.2 Option B: Quadlet Oneshot Container

An alternative deployment runs rotation as a Quadlet oneshot container in the platform pod. This keeps all Tian'er logic containerized.

**Quadlet file (`/etc/containers/systemd/blesniff-rotate.container`):**

```ini
[Container]
ContainerName=blesniff-rotate
Image=localhost/tianer-rotate:latest

# Environment
EnvironmentFile=/etc/tianer/blesniff.env

# Pod: runs in platform pod (has V02 :rw access)
Pod=tianer-platform.pod

# Volumes
Volume=/etc/tianer:/etc/tianer:ro           # V01: config
Volume=/var/lib/tianer/pcap:/var/lib/tianer/pcap:rw   # V02: PCAP
Volume=/var/run/tianer:/var/run/tianer:rw   # V03: runtime
Volume=/var/log/tianer:/var/log/tianer:rw   # V04: logs

# Security
DropCapability=ALL
NoNewPrivileges=true

# Exec
Exec=/usr/local/lib/tianer/rotate-pcap.sh

# Oneshot: runs to completion, exits
Type=oneshot

[Unit]
Description=Tian'er PCAP Rotation
After=tianer-capture.pod

[Install]
WantedBy=blesniff.target
```

**Timer: host-level systemd timer triggers the Quadlet service:**
```ini
[Unit]
Description=Tian'er PCAP Rotation Timer (Quadlet)

[Timer]
OnUnitActiveSec=1800
OnBootSec=120
Persistent=yes
RandomizedDelaySec=30
AccuracySec=1s

[Install]
WantedBy=timers.target
```

The Quadlet approach has a key limitation: the container cannot signal processes in the capture pod (separate PID namespace) unless `--pid=host` is configured. With `--pid=host`, the container sees host PIDs and can signal the sniffer processes directly. Without it, the SIGHUP coordination must use an alternative mechanism:

**File-based coordination (no `--pid=host`):** Instead of sending SIGHUP, the rotation script writes a trigger file (`/var/run/tianer/<name>.rotate`) on V03. The sniffer wrapper monitors this file (via `inotify` or periodic polling) and initiates the rotation sequence on detection. This avoids PID namespace issues entirely. The trade-off is increased latency (polling interval) and additional sniffer wrapper complexity.

### 11.3 Container Image

**Rotation container image (`tianer-rotate`):**
- Base: `debian:trixie-slim` (ARM64)
- Packages: `zstd`, `coreutils`, `findutils`, `util-linux` (for `flock`)
- Entrypoint: `/usr/local/lib/tianer/rotate-pcap.sh`
- Target size: ≤ 30 MB compressed (lightweight, single-purpose)
- No network, no USB, no special capabilities

### 11.4 Install Paths

| Artifact | Source Path | Install Path |
|----------|-------------|-------------|
| rotate-pcap.sh | `modules/bluetooth/rotation/rotate-pcap.sh` | `/usr/local/lib/tianer/rotate-pcap.sh` |
| blesniff-rotate.service | `deploy/systemd/blesniff-rotate.service` | `/etc/systemd/system/blesniff-rotate.service` |
| blesniff-rotate.timer | `deploy/systemd/blesniff-rotate.timer` | `/etc/systemd/system/blesniff-rotate.timer` |
| Quadlet .container | `deploy/containers/blesniff-rotate.container` | `/etc/containers/systemd/blesniff-rotate.container` |

### 11.5 Manual Rotation

Operators can trigger an immediate rotation:

```bash
# Host-level systemd service
sudo systemctl start blesniff-rotate.service

# Quadlet (systemd user instance)
systemctl --user start blesniff-rotate.service
```

To force a purge without rotation:
```bash
sudo -u tianer BLESNIFF_ROTATION_MINUTES=9999 /usr/local/lib/tianer/rotate-pcap.sh
# (With an impossibly long rotation interval, only the purge phase runs meaningfully)
```

---

## References

[1] Linux man-pages Project. "rename(2) — Linux manual page." https://man7.org/linux/man-pages/man2/rename.2.html, 2026-02-08. Section: "If newpath already exists, it will be atomically replaced, so that there is no point at which another process attempting to access newpath will find it missing."

[2] W. Richard Stevens. _Advanced Programming in the UNIX Environment_, 3rd Edition. Addison-Wesley, 2013. Chapter 10: Signals — SIGHUP convention for daemon configuration reload and log file rotation.

[3] GNU Coreutils. "tail(1) — output the last part of files." https://www.gnu.org/software/coreutils/manual/html_node/tail-invocation.html. `-F` option: "same as `--follow=name --retry`."

[4] Yann Collet. "zstd(1) — Zstandard compression tool." zstd 1.5.7, February 2025. https://man.archlinux.org/man/zstd.1. Compression speed: "from fast modes at > 200 MB/s per core, to strong modes with excellent compression ratios." Decompression speed: "a very fast decoder, with speeds > 500 MB/s per core, which remains roughly stable at all compression settings."

[5] The Tcpdump Group. "pcap-savefile(5) — libpcap savefile format." https://www.tcpdump.org/manpages/pcap-savefile.5.txt, 2025-01-21. Per-file header: 24 octets with magic number 0xa1b2c3d4, version 2.4, snapshot length, link-layer type. Per-packet header: 16 octets with timestamps and lengths.

[6] Y. Collet, M. Kucherawy, Ed. "RFC 8878 — Zstandard Compression and the 'application/zstd' Media Type." IETF, February 2021. https://www.ietf.org/rfc/rfc8878.txt.

[7] Lennart Poettering et al. "systemd(1) — systemd system and service manager." systemd 260.2. https://man.archlinux.org/man/systemd.1. SIGHUP behavior: "systemd will reload its configuration."

[8] Lennart Poettering et al. "systemd.timer(5) — Timer unit configuration." systemd 260.2. https://man.archlinux.org/man/systemd.timer.5. `Persistent=` option: "If true, the time when the service unit was last triggered is stored on disk. When the timer is activated, the service unit is triggered immediately if it would have been triggered at least once during the time when the timer was inactive."

[9] Linux Foundation. "Filesystem Hierarchy Standard 3.0 — §5.8 /var/lib : Variable state information." https://refspecs.linuxfoundation.org/FHS_3.0/fhs/ch05s08.html. "This hierarchy holds state information pertaining to an application or the system. State information is data that programs modify while they run, and that pertains to one specific host."

[10] Linux Foundation. "Filesystem Hierarchy Standard 3.0 — §5.1 /var : Variable data." https://refspecs.linuxfoundation.org/FHS_3.0/fhs/ch05.html. "/var contains variable data files. This includes spool directories and files, administrative and logging data, and transient and temporary files."
