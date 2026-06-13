# Phase A Review: Software Engineer

| Field | Value |
|-------|-------|
| Reviewer | Software Engineer |
| Phase | A (Design Review) |
| Date | 2026-06-07 |
| Artifact | `docs/designs/inception.md` v0.5 |
| Review scope | Component boundaries/contracts, data flow architecture, technology choices, scalability, error propagation, deployment, observability, design-for-failure |

---

## Area 1: Component Boundaries and Contracts

**Finding:** The document defines 5 formal contracts (8.4-A, 8.5-A, 8.6-A, 8.8-A, 8.10-A) with explicit change policy in section 9. This is strong. However, several critical interfaces are undocumented:

1. **Sniffer wrapper → FIFO (section 8.1):** No contract defines the PCAP stream format written to the FIFO. The wrapper uses `tee` to split stdout to both a file and a FIFO, but there is no contract for what happens when the FIFO reader (tshark) is not yet up. Section 8.3 says "sniffer writes to a readerless pipe drop bytes" — this is a silent data loss path with no contract governing recovery.

2. **PCAP file → deep parser (section 8.8):** No contract defines the DLT type, link-layer header, or PCAP format variant expected. Ubertooth uses `DLT_BLUETOOTH_LE_LL_WITH_PHDR` while nRF uses a different DLT. This is a contract gap.

3. **Deep parser → ML enrichment (section 8.9):** CONTRACT 8.8-A defines JSONL, but section 8.9's pipeline invocation pipes stdout of `ble-deep-parse` into `blesniff_ml.runner`. There is no contract for error signaling between these components — a malformed JSONL line is silently skipped per 8.9 acceptance criteria, but there is no contract for how many skips are tolerable before the pipeline halts.

4. **Heartbeat companion → DB (section 8.1):** The heartbeat SQL is shown but not formalized as a contract. Multiple components (gap detector, health endpoint, Grafana dashboard) depend on it.

**Confidence:** 90  
**Verdict:** FAIL

---

## Area 2: Data Flow Architecture and Silent Data Loss

**Finding:** Three silent data loss paths exist that the document does not adequately address:

1. **Dedup index collision (section 8.7):** The unique index `uq_raw_packets_dedup ON raw_packets (sniffer_id, ts, mac_address)` means two packets from the same sniffer with the same timestamp and MAC are silently deduplicated at the DB level. On a busy channel, BLE advertisements repeat every 20ms-2s. If the ingest bridge receives two packets with identical `(sniffer_id, ts, mac)` — which can happen when tshark rounds timestamps to microsecond precision — the second INSERT is silently dropped. The backfill path uses `ON CONFLICT DO NOTHING`, which silently swallows these too. There is no counter, no log, no metric for deduplicated packets. This directly contradicts the system goal: "Persist all captured packets... with no acceptable data loss" (section 1, goal 2).

2. **In-flight batch loss (section 8.5):** When the ingest bridge process is killed (OOM, signal, crash), packets buffered in the Batcher but not yet flushed to PostgreSQL are silently lost. The failure mode table says "in-flight packets are recoverable from PCAP via gap detector" — but the gap detector only detects zero-packet buckets (CONTRACT 8.6-A), not specific missing packets within a bucket. If a bucket has 999 packets instead of 1000, the gap detector does not detect it. Claimed recovery is unverifiable.

3. **FIFO readerless pipe (section 8.3):** If tshark dies, `tee` blocks but "no packet loss to file" is claimed. However, the OS pipe buffer is finite. If tshark stays down long enough, the `tee` process blocks, and if the sniffer binary fills its stdout pipe buffer, the sniffer itself stalls or the kernel drops data. The exact pipe buffer size and timeout behavior are not specified. Section 8.3 states "writes to a readerless pipe drop bytes" — which is it? Blocking or dropping? This ambiguity must be resolved.

**Confidence:** 95  
**Verdict:** BLOCKING

---

## Area 3: Missing Failure Modes

**Finding:** The AGENTS.md principle "Design for Failure" requires every component to document failure modes and recovery. Four components have incomplete or missing failure mode documentation:

1. **Deep Parser (section 8.8):** No failure modes table. What happens when libpcap encounters a corrupt packet? When zstd decompression fails mid-stream? When output disk is full? When the process is killed midway through a 30-minute PCAP?

2. **ML Enrichment (section 8.9):** No failure modes table. What happens when the DB connection drops during enrichment writes? When JSONL input is truncated? When `device_summary.enrichment_data` UPDATE conflicts with the ingest bridge's concurrent writes?

3. **Frontend (section 8.11):** No failure modes table. What happens when the API is unreachable? When the API key expires or changes? When WebSocket/SSE connections (if any) drop?

4. **Grafana (section 8.12):** No failure modes table. What happens when the TimescaleDB data source is down? When dashboard JSON is malformed? When anonymous auth is misconfigured?

**Confidence:** 92  
**Verdict:** BLOCKING

---

## Area 4: Observability Mechanism

**Finding:** Section 12.2 defines a metrics catalogue and states: "a sidecar table sniffer_metrics_snapshot updated by each service" (RECOMMENDED DEFAULT). This is marked TBD — the actual mechanism is not designed. Critical gaps:

1. **No structured logging schema.** Section 12.1 says "structured key=value where the language supports it" but defines no schema, no required fields, no correlation IDs. Without a structured log schema, cross-component debugging is ad-hoc.

2. **No Prometheus push mechanism.** C++ services (ingest bridge, deep parser) have no HTTP server and cannot expose `/metrics`. The "sidecar table" mechanism requires a DB connection, which may itself be the failure being observed. This is a circular dependency.

3. **SIGUSR1 metric dump (section 8.5)** is a human-facing workaround, not an observability mechanism. It writes to stderr in an ad-hoc format. No parser, no aggregation, no alerting.

4. **No distributed tracing.** A packet traverses sniffer → FIFO → tshark → ingest bridge → PostgreSQL. There is no trace ID or correlation mechanism. When a user reports "packet X didn't appear in the dashboard," there is no way to trace where it was lost.

**Confidence:** 88  
**Verdict:** BLOCKING

---

## Area 5: Error Propagation

**Finding:** Error propagation between components relies entirely on process exit codes and systemd restart. There is no structured error signaling:

1. **tshark parse failure (section 8.4):** DECISION 8.4.2 says "tshark emits a blank line on parse failure." The ingest bridge must distinguish blank lines from actual empty fields. Blank lines in a pipe-delimited format with 7 fields are ambiguous — is `|||||||` a malformed line or 7 empty fields?

2. **Ingest bridge → gap detector:** When the ingest bridge drops packets due to buffer overflow (10000-row limit, section 8.5), the gap detector has no signal. It only detects zero-packet buckets, not partial drops.

3. **Component crash → recovery:** All components rely on `systemd Restart=on-failure`, but there is no mechanism for dependent components to detect that an upstream has restarted. For example, if tshark restarts, the ingest bridge reading its stdout gets EOF and exits too (failure mode: "stdin closes → drain final batch, exit 0"). This is correct but means the entire per-sniffer pipeline collapses and restarts on any single component failure, with no graduated recovery.

**Confidence:** 85  
**Verdict:** BLOCKING

---

## Area 6: Scalability and Resource Constraints

**Finding:** The target hardware is a Raspberry Pi CM5 with 8 GB RAM. Several design choices push resource limits:

1. **4 concurrent sniffers × per-sniffer processes:** Each sniffer requires: sniffer wrapper (bash + tee), tshark, ingest bridge (C++ with libpqxx connection). That's 12 processes minimum, each with its own PG connection. The `max_connections` default for PostgreSQL is 100, so 4 connections is fine, but the document does not account for the API server's connection pool, Grafana's connections, the gap detector's connections, and the ML enrichment's connections. Total could reach 15-20 concurrent connections on 8 GB RAM.

2. **FIFO buffer sizing:** Named pipes on Linux have a default 64 KB buffer. At 1500 packets/sec with ~100 bytes per tshark output line, the FIFO fills in ~400ms. If tshark is CPU-starved on the Pi (which also runs PostgreSQL, Grafana, and FastAPI), it cannot drain fast enough.

3. **PCAP file growth:** 30-minute rotation at 1500 pkts/sec = 2.7M packets per 30-min file. At ~50 bytes per raw BLE packet, that's ~135 MB per file per sniffer, or 540 MB per rotation cycle for 4 sniffers. On 32 GB eMMC with a 14-day retention, that's 540 MB × 48 × 14 ≈ 363 GB before compression — far exceeding storage. Even with zstd at 5:1 compression, that's ~72 GB. The document does not include a storage budget calculation.

**Confidence:** 82  
**Verdict:** FAIL

---

## Area 7: Technology Choices

**Finding:** Generally sound. Specific issues:

1. **libpqxx 7.8 with PostgreSQL 17:** The inception doc cites PG 17 but libpqxx 7.8 may not have explicit PG 17 support. The libpqxx README should be checked for compatible PG versions. Confidence is moderate since the wire protocol is stable across PG versions.

2. **pyshark for gap detector (section 8.6):** pyshark wraps tshark by spawning a subprocess per capture session. For batch reprocessing of PCAP files, this is extremely slow compared to using tshark directly or using a native Python PCAP library like `scapy` or `pycfile`. The gap detector backfill path (section 8.6, backfill.py) processes PCAP files — using pyshark means one tshark subprocess per backfill operation, with significant per-packet overhead from subprocess IPC.

3. **Python 3.13 vs 3.12 in section headers:** The document says Python 3.13 (table in section 5) but task descriptions reference "Python 3.12" (sections 8.6, 8.9). This inconsistency must be resolved.

**Confidence:** 78  
**Verdict:** ADVISORY

---

## Area 8: Deployment and Operational Concerns

**Finding:**

1. **`deploy/setup.sh` version in section 13.2** references "PostgreSQL 16" and "Node.js 20", contradicting section 5 which pins PostgreSQL 17 and Node.js 24 LTS. This is a copy-paste error from an earlier version.

2. **No rollback mechanism.** Section 13.1 defines `make install` but no `make uninstall` or rollback procedure. If a deploy breaks, the operator has no documented recovery path.

3. **No health check for deep parser or ML enrichment.** These are batch-mode components, but there is no mechanism to verify they ran successfully on their last invocation. The gap detector only covers the ingest pipeline.

**Confidence:** 85  
**Verdict:** FAIL

---

## Summary of Findings

| # | Area | Severity | Confidence | Verdict | Description |
|---|------|----------|-----------|---------|-------------|
| 1 | Component boundaries/contracts | Major | 90 | FAIL | 4 undocumented interfaces |
| 2 | Silent data loss | Blocking | 95 | BLOCKING | Dedup index, in-flight batch, FIFO ambiguity |
| 3 | Missing failure modes | Blocking | 92 | BLOCKING | 4 components lack failure mode documentation |
| 4 | Observability | Blocking | 88 | BLOCKING | TBD mechanism, no structured logging, no tracing |
| 5 | Error propagation | Blocking | 85 | BLOCKING | No structured error signaling, cascade restart |
| 6 | Scalability | Major | 82 | FAIL | No storage budget, FIFO sizing, connection count |
| 7 | Technology choices | Minor | 78 | ADVISORY | pyshark performance, Python version inconsistency |
| 8 | Deployment | Major | 85 | FAIL | Version contradictions, no rollback, no batch health |

---

## Overall Verdict

**REJECTED** — 4 blocking findings (areas 2-5), 3 major findings (areas 1, 6, 8). The document cannot proceed to implementation until:

1. All silent data loss paths are documented with detection and recovery mechanisms.
2. Failure modes are documented for all 4 missing components.
3. The observability mechanism is designed (not TBD).
4. Structured error propagation between components is specified.

---

## Self-Audit Checklist

| # | Check | Status |
|---|-------|--------|
| 1 | Read the complete artifact? | YES — all 2910 lines |
| 2 | Every finding includes a confidence score? | YES |
| 3 | Every finding is actionable? | YES — specific sections and line items |
| 4 | No speculative claims without evidence? | YES — all findings reference specific document sections |
| 5 | Blocking findings justified by severity? | YES — silent data loss violates stated system goals |
| 6 | Advisory findings clearly separated? | YES — area 7 is advisory |
| 7 | No duplicate findings across areas? | YES |
| 8 | Verdict is consistent with finding severities? | YES — 4 blocking → REJECTED |

---

## Flags for PM

| Flag ID | Type | Description | Urgency |
|---------|------|-------------|---------|
| FLAG-SE-001 | Decision | Dedup index `uq_raw_packets_dedup` silently drops packets. Must decide: accept loss with counter, or change schema to allow duplicates? | Blocking |
| FLAG-SE-002 | Decision | In-flight batch loss during ingest bridge crash: gap detector cannot detect partial bucket loss. Must add per-packet sequence numbers or accept the loss window. | Blocking |
| FLAG-SE-003 | Decision | Observability mechanism must be chosen: sidecar DB table, Prometheus push gateway, file-based, orStatsD. Document commitment before implementation. | Blocking |
| FLAG-SE-004 | Clarification | Python 3.12 vs 3.13: section 5 pins 3.13, sections 8.6/8.9 reference 3.12. Resolve. | High |
| FLAG-SE-005 | Decision | Storage budget calculation needed before hardware procurement. Current retention + 4 sniffers may exceed 32 GB eMMC even with compression. | High |
| FLAG-SE-006 | Ticket | `deploy/setup.sh` references PG 16 and Node 20 — update to PG 17 and Node 24. | Medium |