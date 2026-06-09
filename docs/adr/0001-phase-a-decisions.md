# ADR-0001: Phase A Architecture Decisions

**Status:** Accepted
**Date:** 2026-06-07
**Deciders:** User (Jean)

## Context

During Phase A review of the Tian'er inception document and component breakdown, 24 architectural decisions were identified by 6 specialist reviewers and resolved.

## Decisions

### Naming & Paths

| ID | Decision | Resolution | Rationale |
|----|----------|-----------|-----------|
| D-01 | Database name | `tianer` | Project-level namespace; not module-specific |
| D-02 | Environment variable prefix | `TIANER_*` for shared platform, `BLESNIFF_*` for Bluetooth module | Future modules (GPS, ADS-B) need shared infrastructure with their own module prefixes |
| Q2 | Monorepo layout authority | `tian-er/` layout from component-breakdown.md is authoritative | Matches actual repo name; inception.md layout is older |

### Hardware

| ID | Decision | Resolution | Rationale |
|----|----------|-----------|-----------|
| D-03 | nRF52840 USB PID | Support both 1915:522A (sniffer mode) and 1915:520f (DFU mode) | Exact PID determined during hardware design phase with connected device |
| D-04 | USB topology | Powered USB hub | CM5 has 3 ports; needs 4+ dongle ports |
| Q3 | nRF PID determination timing | During detailed hardware design phase | Requires physical hardware for `lsusb` verification |
| Q4 | USB device symlink strategy | Persistent udev device names via port-path binding on host | Deterministic `/dev/tianer/nrf0` across reboots |

### Bluetooth Protocol

| ID | Decision | Resolution | Rationale |
|----|----------|-----------|-----------|
| D-05 | Channel coverage strategy | Start with 1 dongle, configurable per sniffer (default ch37), dedup at query time not insert time | Single-channel hardware limitation; expansion when more dongles available |
| D-10 | PCAP dedup index granularity | Query-time dedup via `DISTINCT ON (sniffer_id, ts, mac_address, pdu_type)`. No insert-time unique index. | `pdu_type` adds needed granularity; insert-time dedup would reject backfill |
| D-12 | tshark field configuration | Per-DLT parameterized config in tshark-wrap.sh with normalized output | Ubertooth and nRF use different DLT types and field names |
| Q5 | Bluetooth schema location | Isolated `bluetooth` schema under `tianer` database | Future modules (GPS, ADS-B) get their own schemas |
| Q6 | Single-dongle channel strategy | Configurable per sniffer in `sniffers.yaml`, default channel 37 | Operator changes via config reload |
| Q7 | CRC-24 verification | C07 Deep Parser adds configurable CRC-24 validation (default: enabled) | nRF Sniffer firmware may not verify CRC; silent data corruption risk |

### Security

| ID | Decision | Resolution | Rationale |
|----|----------|-----------|-----------|
| D-06 | TLS on API | Self-signed cert, mandatory | Encryption for LAN-accessible API |
| D-07 | Disk encryption | Accepted risk for MVP | LUKS if TPM available; otherwise document |
| Q8 | Secrets rotation | Deferred to post-MVP | Static keys acceptable for v1 |
| Q9 | DB role least-privilege | Write-only role (`tianer_writer`) for ingest/gap-detector streams. Read-only (`tianer_ro`) for UI. Full DDL (`tianer`) restricted to API and migrations. | Defense in depth — compromised ingest cannot DROP tables |
| Q10 | Supply chain integrity | Required for MVP | GPG verification, SHA256 checksums, digest pinning for containers, hash-pinned Python, `npm audit` |

### Infrastructure

| ID | Decision | Resolution | Rationale |
|----|----------|-----------|-----------|
| D-08 | Metrics exposition | File-based Prometheus text format for C++ services | C++ can't serve HTTP metrics endpoint as easily |
| D-09 | Ingest bridge buffer size | 300K rows (~60MB RAM), 5-minute timeout | Covers 5 minutes of DB downtime at 1000 pps |
| D-11 | Heartbeat fallback | Local file primary (`/var/lib/tianer/heartbeat/<name>.ts`), DB table secondary | DB outage must not trigger false-positive gap alerts |
| NEW | Container orchestration | Podman + Quadlet, all-container model | Consistency, reproducibility, sandboxing |
| Q1 | Container model | All components in rootless Podman containers. USB devices passed through from host. | Rootless preferred for security; USB passthrough via `--device` and `--group-add keep-groups` |
| Q11 | Persistent storage strategy | 9 volumes (V01-V09), 2 pods, per-container access matrix with `:ro` constraints | Detailed in `doc/designs/storage-strategy.md` |

## Consequences

- inception.md updated to reflect all decisions; no staleness banners (documents are current)
- component-breakdown.md updated with container orchestration, storage, and per-component artifact details
- AGENTS.md updated with Cross-Document Consistency Rule (no stale documents)
- storage-strategy.md created with complete volume inventory and access matrix
- All future design decisions follow this ADR process
