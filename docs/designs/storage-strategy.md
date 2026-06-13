# Tian'er Container Persistent Storage Strategy

**Author:** Software Engineer  
**Date:** 2026-06-08  
**Status:** Phase A Design — incorporated into component design docs

## Volume Inventory

| ID | Name | Type | Host Path | Containers | Mode | Lifecycle |
|----|------|------|-----------|------------|------|-----------|
| V01 | tianer-config | Bind-mount | /etc/tianer/ | ALL | :ro | Persistent |
| V02 | tianer-pcap | Bind-mount | /var/lib/tianer/pcap/ | sniffer(rs), rotate(rw), gapdetect(ro), deepparse(ro) | mixed | Persistent |
| V03 | tianer-fifo | Bind-mount (tmpfs) | /var/run/tianer/ | sniffer(rs), tshark(rs), ingest(rs) | :rw | Ephemeral (tmpfs) |
| V04 | tianer-logs | Bind-mount | /var/log/tianer/ | ALL | :rw | Persistent |
| V05 | tianer-data | Bind-mount | /var/lib/tianer/data/ | deepparse(rw), ml-classify(ro) | mixed | Persistent |
| V06 | tianer-postgres-data | Podman volume | tianer-postgres-data.volume | postgres | :rw | Persistent |
| V07 | tianer-grafana-data | Podman volume | tianer-grafana-data.volume | grafana | :rw | Persistent |
| V08 | tianer-capture-tmp | Per-container /tmp in capture pod | tmpfs | — (per-pod ephemeral) | Container-private | Tmpfs=/tmp | Ephemeral (destroyed with pod) |
| V09 | tianer-platform-tmp | Per-container /tmp in platform pod | tmpfs | — (per-container ephemeral) | Container-private | Tmpfs=/tmp | Ephemeral (destroyed with container) |

## Data Flow Ownership

### Decoupled Capture Architecture

The sniffer container writes PCAP exclusively to disk (V02). A separate `tail -f` process running alongside the sniffer in the capture pod reads the growing PCAP file and feeds the FIFO (V03). This decouples the sniffer from downstream backpressure: if the ingest bridge blocks because the database is down, `tail -f` blocks on the FIFO write, but the sniffer continues capturing to disk without interruption.

- sniffer@ writes raw PCAP → V02 (exclusively, no FIFO dependency)
- tail-feed@ reads growing PCAP from V02, writes PCAP stream → V03 (.fifo)
- tshark@ reads V03 (.fifo), writes structured fields → V03 (-ingest.fifo)
- ingest@ reads V03 (-ingest.fifo), batch COPY → PostgreSQL (V06)
- rotate renames/compresses → V02
- gapdetect reads V02 (ro), backfill INSERT → PostgreSQL
- deepparse reads V02 (ro), writes JSONL → V05
- ml-classify reads V05 (ro), writes enrichment → PostgreSQL
- api reads PostgreSQL only, serves frontend from image (no volume)
- grafana reads PostgreSQL (tianer_grafana role)

### Backpressure Isolation

The FIFO (V03) is a kernel pipe buffer (default 64KB on Linux). When the ingest bridge cannot keep up with the packet rate, the pipe fills and `tail -f` blocks on the write end. The sniffer container is unaffected — it continues writing to PCAP (V02), which is a regular file on a bind-mounted host directory. No data is lost from the PCAP archive (source of truth).

During a database outage:
1. `tail -f` blocks on FIFO → tshark stops reading → ingest stops
2. Sniffer continues writing PCAP → rotate continues compressing old files
3. When DB recovers → ingest resumes → gap detector identifies the outage window → backfills from PCAP
4. Zero packet loss from PCAP archive. Gap window is bounded by PCAP retention (14 days).

## Cross-Container Access Constraints

- Only sniffer@ and rotate write to V02
- gapdetect and deepparse mount V02 :ro
- Only deepparse writes to V05
- ml-classify mounts V05 :ro
- Only postgres mounts V06
- V01 is :ro on ALL containers
- No --privileged, no podman socket mounts

## Secrets Handling

- PostgreSQL creds: EnvironmentFile=/etc/tianer/secrets/db_password.env (not via V01)
- API key: Environment= injection
- TLS cert/key: V01 :ro mount for uvicorn

## Container Inventory

11 container types across 2 pods (tianer-capture, tianer-platform) + 2 standalone (postgres, grafana)
Quadlet files in deploy/containers/ (14 .container, 2 .volume, 1 .network, 2 .pod files)

## Container Image Policy

All runtime images must be minimal:
- **Base image:** Debian slim variant (or Alpine for Python-only services)
- **Multi-stage builds mandatory:** Build stage has compilers/dev-headers; runtime stage has only the compiled binary + runtime libs
- **No shells, no package managers, no dev headers** in runtime images
- **Target size:** Under 200 MB uncompressed per image
- **Target boot:** Under 3 seconds to ready on Raspberry Pi CM5
- **Rationale:** Fast cold-start after host reboot or container crash. Minimal attack surface — less code in the image means fewer CVEs.

## Disaster Recovery

- **Host reboot:** V03 recreated by systemd-tmpfiles. V02/V06 persistent. Containers auto-restart via systemd linger. Gap detector backfills any missed buckets from PCAP.
- **DB outage:** Sniffer continues writing to PCAP. `tail -f` blocks on FIFO. No PCAP data loss. Gap detector backfills on DB recovery.
- **Container crash (sniffer):** PCAP file has last write at crash point. Podman auto-restarts. Gap detector identifies outage window from heartbeat gap and backfills.
- **Container crash (ingest):** Ingest buffer lost. Gap detector detects missing time bucket and backfills from PCAP.
- **Disk full:** Sniffer write fails → sniffer exits → podman restarts → still fails → alert fires. PCAP rotation protects against gradual fill.
- **V06 corruption:** Restore from pg_basebackup, or full rebuild from PCAP archive (source of truth).

## Layered Failure Handling

The capture pipeline implements defense-in-depth for data integrity through multiple independent failure-detection and recovery layers.

### Layer 1: PCAP File — Source of Truth
- **What:** Every raw bit from the sniffer is written to rotating PCAP files (V02) before any processing.
- **Failure mode:** Disk full → `tee`/write fails → sniffer process exits → systemd/Podman restarts → still fails → alert fires.
- **Recovery:** Free disk space or expand storage. PCAP rotation (C04) protects against gradual fill.
- **Detection:** `tianer_disk_usage_percent` metric; alert at 80%.

### Layer 2: Heartbeat — Liveness
- **What:** Local file heartbeat (`/var/lib/tianer/heartbeat/<name>.ts`) updated every 30s per sniffer. DB heartbeat table is secondary.
- **Failure mode:** Sniffer process crashes → heartbeat file stops updating.
- **Recovery:** Podman auto-restarts the sniffer container. Gap detector identifies the outage window from heartbeat gaps.
- **Detection:** Heartbeat file age > 60s → alert. DB heartbeat table backfilled from local file on recovery.

### Layer 3: Gap Detector — Ingest Integrity
- **What:** Compares time-bucketed packet counts in DB against expected traffic windows from heartbeat.
- **Failure mode:** DB outage, ingest crash, or FIFO backpressure causes missing time buckets.
- **Recovery:** Backfills from PCAP files (V02). Idempotent INSERT (ON CONFLICT DO NOTHING).
- **Detection:** `tianer_gap_detected_total` counter. Gap detector itself is monitored by secondary verification.

### Layer 4: Secondary Verification — Cross-Validation
- **What:** Periodic comparison: PCAP file packet count (via `capinfos`) vs DB row count per time window.
- **Failure mode:** Gap detector itself is down or buggy, missing gaps that Layer 3 should have caught.
- **Recovery:** Manual investigation triggered by alert. Raw PCAP is always available for ad-hoc analysis.
- **Detection:** Count mismatch > 1% → critical alert. Runs on a different schedule than the gap detector.

### Layer 5: PCAP Integrity — Source Verification
- **What:** Periodic `capinfos` or `tshark` validation of rotated PCAP files for corruption.
- **Failure mode:** Filesystem corruption, incomplete writes, compression errors.
- **Recovery:** Flag corrupted files; skip in gap detector backfill. Accept data loss for that file window.
- **Detection:** `tianer_pcap_corrupt_total` counter. Alert if > 0 files corrupted in 24h.

### Data Loss Quantification (PF-10)
- **PCAP (source of truth):** Zero loss under normal operation. During disk-full events, loss window = time from disk-full to alert (~60s).
- **DB ingest:** Up to 30s bucket granularity during normal operation. Up to 5-minute gap detection window during DB outages.
- **Recovery SLA:** Gap detector backfills within 5 minutes of DB recovery. All data recoverable from PCAP for 14 days.
