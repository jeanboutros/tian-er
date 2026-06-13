# Tian'er — Component Breakdown & Design Plan

**Status:** Phase A Synthesis  
**Generated from:** Expert review of `inception.md` v0.5  
**Date:** 2026-06-07  
**Reviewers:** Software Engineer, Hardware Engineer, Wireless Expert, Security Reviewer, Test Engineer

---

## 1. Expert Review Summary

All five specialists reviewed the inception document. Key verdicts:

| Reviewer | Overall Verdict | Blocking Findings | Critical Theme |
|----------|----------------|-------------------|----------------|
| Software Engineer | **REJECTED** | 7 | Silent data loss paths, missing failure modes, observability gaps |
| Hardware Engineer | **CONDITIONAL PASS** | 8 | USB topology, nRF PID mismatch, power/thermal budget |
| Wireless Expert | **CONDITIONAL PASS** | 6 | Single-channel limitation, DLT type differences, data whitening absent |
| Security Reviewer | **CONDITIONAL PASS** | 7 | No TLS, no secrets rotation, no disk encryption, input validation gaps |
| Test Engineer | **CONDITIONAL PASS** | 8 | CI infra undefined, contract test gaps, mock-sniffer fidelity |

### Cross-Cutting Critical Findings

These findings appeared in multiple reviews and must be resolved in design documents:

| Finding | Raised By | Impact |
|---------|-----------|--------|
| **nRF52840 USB PID `520f` vs `522A`** | Hardware + Wireless | nRF sniffer won't enumerate — complete block |
| **DLT type difference (Ubertooth vs nRF)** | Wireless + Software | tshark fields differ per sniffer — CONTRACT 8.4-A incomplete |
| **Database name `blesniff` vs `tianer`** | Software | Violates project naming convention, blocks future modules |
| **Version inconsistencies (PG 16/17, Node 20/24, Python 3.12/3.13)** | Software + Hardware | Implementation will install wrong versions |
| **Single-channel-at-a-time sniffer limitation undocumented** | Wireless | Architecture cannot cover all 3 adv channels with 1 sniffer |
| **PCAP dedup index too coarse** (same-timestamp packets silently dropped) | Software | Silent data loss violates "no acceptable data loss" goal |
| **No TLS on API, no secrets rotation, no disk encryption** | Security | Cleartext secrets on LAN, physically accessible device |
| **In-flight batch data loss on OOM kill** | Software | Gap detector cannot detect partial-bucket loss |
| **Heartbeat → gap detector cascade during DB outage** | Software | DB outage causes both false positive AND false negative gaps |

---

## 2. Component Breakdown

The platform is decomposed into 14 components organized by responsibility domain. Each component has its own design document, its own language/toolchain, and clear contracts with its neighbors.

### 2.1 Component Inventory

| # | Component | Language | Inception Ref | Purpose |
|---|-----------|----------|---------------|---------|
| C01 | Platform Infrastructure | Bash / systemd | §3, §5, §6 | OS bootstrap, users, groups, capabilities, udev, file system, config |
| C02 | Database | SQL (PostgreSQL/TimescaleDB) | §6.6, §8.7 | Roles, schema, migrations, continuous aggregates, compression, residency |
| C03 | Capture Pipeline | Bash / FIFO / tshark | §8.1, §8.3, §8.4 | Sniffer wrappers, FIFO IPC, tshark real-time parsing, heartbeat |
| C04 | PCAP Rotation | Bash / zstd / systemd timer | §8.2 | File rotation, compression, retention, integrity verification |
| C05 | Ingest Bridge | C++17 / libpqxx | §8.5 | Parse tshark output, batch, COPY to PostgreSQL |
| C06 | Gap Detector | Python 3.13 / psycopg | §8.6 | Detect ingest gaps, backfill from PCAP, idempotent recovery |
| C07 | Deep Parser | C++17 / libpcap / nlohmann-json | §8.8 | Parse rotated PCAP, dissect BLE PDUs, emit JSONL |
| C08 | ML Enrichment | Python 3.13 / uv | §8.9 | Rule-based device classification from JSONL |
| C09 | REST API | Python 3.13 / FastAPI | §8.10 | Serve device data, health, metrics; auth; serve frontend |
| C10 | Frontend | TypeScript / Vue 3 / Vite / Pinia | §8.11 | Device explorer, alerts, health dashboard, Grafana embedding |
| C11 | Grafana Dashboards | Grafana provisioning / SQL | §8.12 | Live metrics, device explorer, pipeline health dashboards |
| C12 | Service Orchestration | systemd / Makefile / Bash | §12.5, §13 | All systemd units, dependency chain, build/install paths, Makefile |
| C13 | Observability | Prometheus format / structured logs | §12.1, §12.2 | Metrics exposition, structured logging, health endpoints |
| C14 | Deployment Automation | Bash / Makefile | §13.1, §13.2 | `setup.sh`, build targets, install paths, rollback |

### 2.1a Container Orchestration (Q1 Resolution)

All components run in Podman containers managed by Quadlet. USB devices are passed through from the host using persistent udev symlinks (Q4). The deployment model uses **rootless Podman** with `--group-add keep-groups` for the `tianer` user. See `docs/designs/storage-strategy.md` for the full volume inventory, per-container access matrix, and Quadlet unit file specifications.

**Container-Pod Architecture:**
- **tianer-capture pod** (Network=none): sniffer + tshark containers, one instance per sniffer
- **tianer-platform pod** (bridge): ingest, gap detector, deep parser, ML, API, heartbeat
- **Standalone:** PostgreSQL (tianer-postgres), Grafana (tianer-grafana)

**Security model** per `docs/project-management/reviews/TIANER-001-container-security-recommendation.md`:
- Zero `--privileged` containers
- `--cap-drop ALL` default; C03 only: `CAP_NET_RAW` + `CAP_NET_ADMIN`
- Digest-pinned images (`@sha256:`); no `:latest` tags
- Config volume (`/etc/tianer/`) mounted `:ro` everywhere
- PCAP volume mounted `:ro` for gap detector and deep parser

**Container image policy:** All container images must be built from minimal base images (Debian `slim` variant, Alpine, or distroless as appropriate per toolchain). Images must include only the libraries and tools required to run the component — no shells, package managers, or development headers in runtime images. Multi-stage builds are mandatory: the build stage may include compilers and dev headers; the runtime stage copies only the compiled artifact and its runtime dependencies. Target: each runtime image under 200 MB uncompressed, boot-to-ready under 3 seconds on a Raspberry Pi CM5.

### 2.2 Component Categories

**Shared Platform (built once, inherited by all modules):**
C01, C02, C09, C10, C11, C12, C13, C14

**v1 Bluetooth Module:**
C03, C04, C05, C06, C07, C08

---

## 3. Monorepo Directory Structure

The repository is organized as a multi-project monorepo. Each service is a self-contained project with its own build system, dependencies, and tests. The top-level `Makefile` orchestrates cross-project builds and deployment.

```
tian-er/
├── README.md
├── Makefile                           # Top-level orchestration (C12)
├── .editorconfig
├── .gitignore
├── .pre-commit-config.yaml
│
├── doc/
│   └── designs/                        # All design documents
│       ├── inception.md                # Original v0.5 inception spec
│       ├── component-breakdown.md     # This document
│       ├── c01-platform-infrastructure.md
│       ├── c02-database.md
│       ├── c03-capture-pipeline.md
│       ├── c04-pcap-rotation.md
│       ├── c05-ingest-bridge.md
│       ├── c06-gap-detector.md
│       ├── c07-deep-parser.md
│       ├── c08-ml-enrichment.md
│       ├── c09-rest-api.md
│       ├── c10-frontend.md
│       ├── c11-grafana-dashboards.md
│       ├── c12-service-orchestration.md
│       ├── c13-observability.md
│       └── c14-deployment-automation.md
│
├── deploy/                             # C01, C14: host bootstrap & automation
│   ├── setup.sh                        # Idempotent host bootstrap
│   ├── apt-packages.txt
│   ├── udev/
│   │   └── 99-tianer.rules            # udev rules (replaces 99-blesniff.rules)
│   ├── systemd/                        # C12: all systemd units
│   │   ├── tianer.target              # Shared platform target
│   │   ├── blesniff.target            # Bluetooth module target
│   │   ├── blesniff-sniffer@.service
│   │   ├── blesniff-tshark@.service
│   │   ├── blesniff-ingest@.service
│   │   ├── blesniff-rotate.service
│   │   ├── blesniff-rotate.timer
│   │   ├── blesniff-gap-detector.service
│   │   ├── blesniff-gap-detector.timer
│   │   ├── blesniff-api.service
│   │   └── blesniff-classify.service
│   ├── scripts/
│   │   ├── create-user.sh
│   │   ├── create-dirs.sh
│   │   ├── setup-wireshark.sh
│   │   ├── install-postgres.sh
│   │   ├── install-grafana.sh
│   │   ├── install-ubertooth.sh
│   │   ├── rotate-pcap.sh
│   │   ├── generate-secrets.sh
│   │   ├── backup.sh
│   │   └── verify-integrity.sh
│   └── config/
│       └── tianer.env.example          # Environment template (TIANER_* prefix)
│
├── modules/                            # Sensor modules (one per signal type)
│   └── bluetooth/                      # v1 module: blesniff
│       ├── sniffers/                   # C03: sniffer wrapper scripts
│       │   ├── ubertooth-wrap.sh
│       │   ├── nrf-wrap.sh
│       │   ├── tshark-wrap.sh          # Parameterized per sniffer type (DLT)
│       │   ├── heartbeat.sh
│       │   └── tests/
│       │       ├── test_fifo_create.bats
│       │       ├── test_dual_write.bats
│       │       └── test_args.bats
│       ├── ingest-bridge/              # C05: C++17
│       │   ├── CMakeLists.txt
│       │   ├── src/
│       │   │   ├── main.cpp
│       │   │   ├── parser.cpp / .hpp  # CONTRACT 8.4-A parser
│       │   │   ├── batcher.cpp / .hpp
│       │   │   ├── pg_writer.cpp / .hpp
│       │   │   └── config.cpp / .hpp
│       │   └── tests/                  # GoogleTest
│       ├── gap-detector/               # C06: Python 3.13
│       │   ├── pyproject.toml
│       │   ├── src/tianer_gapdet/      # Package under project namespace
│       │   │   ├── __init__.py
│       │   │   ├── detector.py
│       │   │   ├── backfill.py
│       │   │   ├── pcap_reader.py
│       │   │   └── db.py
│       │   └── tests/
│       ├── deep-parser/                # C07: C++17
│       │   ├── CMakeLists.txt
│       │   ├── src/
│       │   │   ├── main.cpp
│       │   │   ├── pcap_input.cpp / .hpp
│       │   │   ├── ble_dissector.cpp / .hpp
│       │   │   ├── advdata_parser.cpp / .hpp
│       │   │   └── jsonl_output.cpp / .hpp
│       │   └── tests/
│       └── ml-enrichment/              # C08: Python 3.13
│           ├── pyproject.toml
│           ├── src/tianer_ml/
│           │   ├── classifier.py
│           │   ├── features.py
│           │   └── runner.py
│           └── tests/
│
├── platform/                           # Shared platform services
│   ├── api/                            # C09: Python FastAPI
│   │   ├── pyproject.toml
│   │   ├── src/tianer_api/
│   │   │   ├── main.py
│   │   │   ├── auth.py
│   │   │   ├── models.py
│   │   │   ├── db.py
│   │   │   └── routers/
│   │   │       ├── devices.py
│   │   │       ├── alerts.py
│   │   │       └── health.py
│   │   └── tests/
│   └── frontend/                       # C10: Vue 3 + Vite + TypeScript
│       ├── package.json
│       ├── tsconfig.json
│       ├── vite.config.ts
│       ├── src/
│       │   ├── main.ts
│       │   ├── App.vue
│       │   ├── router/
│       │   ├── stores/                  # Pinia
│       │   ├── views/
│       │   ├── components/
│       │   ├── api/                     # Generated TS API client
│       │   └── types/
│       └── tests/                       # Vitest + Vue Test Utils
│
├── db/                                 # C02: Database schema
│   ├── migrations/
│   │   ├── 0001_init.sql
│   │   ├── 0002_continuous_aggregates.sql
│   │   ├── 0003_compression_policies.sql
│   │   ├── 0004_residency_classifier.sql
│   │   └── 0005_device_enrichment.sql
│   ├── roles/
│   │   └── init-roles.sql              # DB roles: tianer, tianer_ro, tianer_grafana
│   ├── seed/
│   └── tests/                           # pgTAP
│
├── grafana/                            # C11: Grafana provisioning
│   ├── provisioning/
│   │   ├── datasources/
│   │   │   └── timescaledb.yaml
│   │   └── dashboards/
│   │       └── default.yaml
│   └── dashboards/
│       ├── live-metrics.json
│       ├── device-explorer.json
│       ├── per-device-drilldown.json
│       └── pipeline-health.json
│
├── observability/                       # C13: Metrics & logging config
│   ├── metrics/
│   │   ├── catalogue.yaml              # Full Prometheus metrics catalogue
│   │   └── alert-rules.yaml
│   └── logging/
│       └── structured-format-spec.yaml # JSON structured log format
│
├── tools/                               # Development & test utilities
│   ├── pcap-replay.py
│   ├── mock-sniffer.py
│   ├── generate-pcap.py               # Synthetic PCAP (pre-hardware)
│   ├── seed-test-data.py
│   └── verify-mock-fidelity.py         # Mock-sniffer ↔ real PCAP conformance
│
├── tests/                               # Cross-cutting test suites
│   ├── fixtures/
│   │   ├── pcap/
│   │   └── expected/
│   ├── integration/
│   │   ├── test_capture_pipeline.sh    # FIFO → tshark → ingest end-to-end
│   │   ├── test_gap_detection.sh
│   │   ├── test_deep_parser_ml_contract.sh
│   │   ├── test_api_to_db.sh
│   │   └── test_prereqs.sh
│   ├── e2e/
│   │   └── smoke.sh
│   └── perf/                            # Performance benchmarks
│       ├── ingest_throughput.sh
│       ├── api_latency.sh
│       └── deep_parser_throughput.sh
│
├── ci/                                  # CI pipeline
│   ├── docker-compose.yml              # PostgreSQL + TimescaleDB for CI
│   ├── lint.sh
│   ├── test-all.sh
│   └── build-all.sh
│
└── docs/                                # User/operator documentation
    ├── runbooks/
    │   ├── deploy.md
    │   ├── recover-from-disk-full.md
    │   ├── add-new-sniffer.md
    │   ├── rotate-secrets.md
    │   └── hardware-recovery.md
    ├── adr/                             # Architecture Decision Records
    ├── decisions-log.md
    ├── tshark-fields.md                 # Verified tshark field names per DLT
    └── storage-budget.md               # Formal storage calculation
```

### 3.1 Key Structural Decisions

| Decision | Rationale |
|----------|-----------|
| `modules/bluetooth/` instead of `services/sniffer-wrapper/` | Future modules (GPS, ADS-B) each get their own directory under `modules/`. The Bluetooth module is the first. |
| `platform/` for shared services | API and frontend are shared across all modules. Separated from module-specific code. |
| `tianer_*` Python package namespace | Project-level namespace, not module-level. Gap detector is `tianer_gapdet`, ML is `tianer_ml`, API is `tianer_api`. |
| `deploy/udev/99-tianer.rules` | Project-level naming. Replaces `99-blesniff.rules`. Rules map devices under `/dev/tianer/`. |
| `deploy/config/tianer.env.example` | Shared env vars use `TIANER_*` prefix. Module-specific vars use `BLESNIFF_*` prefix. |
| `db/roles/init-roles.sql` creates `tianer` database | Matches project naming convention. Module tables go in `bluetooth` schema. |
| `ci/docker-compose.yml` added | Required for integration tests in CI (flagged by test engineer). |

---

## 4. Component Sequencing & Dependency Map

### 4.1 Build Sequence (Layers)

Components are built in layers. Each layer depends on the layer below. Within a layer, components can be built in parallel.

```
Layer 0: ─────────────────────────────────────────────
  C01  Platform Infrastructure
  C02  Database
  C14  Deployment Automation (scaffold — scripts grow with each layer)

Layer 1: ─────────────────────────────────────────────
  C03  Capture Pipeline
  C04  PCAP Rotation
  C05  Ingest Bridge
  C13  Observability (metrics exposition, log format)

Layer 2: ─────────────────────────────────────────────
  C06  Gap Detector
  C07  Deep Parser
  C09  REST API

Layer 3: ─────────────────────────────────────────────
  C08  ML Enrichment
  C10  Frontend
  C11  Grafana Dashboards

Layer 4: ─────────────────────────────────────────────
  C12  Service Orchestration (final wiring, depends on all)
```

### 4.2 Dependency Graph

```
C01 Platform Infrastructure ─────────┬─────┬─────┬──────┬──────┬──────┐
                                       │     │     │      │      │
C02 Database ──────────────────────────┼─────┼─────┼──────┼──────┤
  │                                    │     │     │      │      │
  ├─► C05 Ingest Bridge ───────────────┼─────┼─────┼──────┤      │
  │     │                              │     │     │      │      │
  │     ├─► C06 Gap Detector ──────────┼─────┼─────┤      │      │
  │     │                              │     │     │      │      │
  ├─► C03 Capture Pipeline ────────────┼─────┤     │      │      │
  │     │                              │     │     │      │      │
  │     └─► C04 PCAP Rotation ─────────┼─────┤     │      │      │
  │                                    │     │     │      │      │
  ├─► C07 Deep Parser ─────────────────┼─────┤     │      │      │
  │     │                              │     │     │      │      │
  │     └─► C08 ML Enrichment ─────────┼─────┤     │      │      │
  │                                    │     │     │      │      │
  ├─► C09 REST API ─────────────────────┼─────┘     │      │      │
  │     │                              │           │      │      │
  │     └─► C10 Frontend ──────────────┼───────────┘      │      │
  │                                    │                  │      │
  ├─► C11 Grafana Dashboards ──────────┘                  │      │
  │                                                       │      │
  └─► C13 Observability ──────────────────────────────────┘      │
                                                                 │
  C12 Service Orchestration ◄──────────────────────────────────┘
      (wires everything together, requires all components)

  C14 Deployment Automation (runs throughout, grows with each layer)
```

### 4.3 Dependency Table

| Component | Depends On | Blocks |
|-----------|-----------|--------|
| C01 Platform Infrastructure | — | C02, C03, C05, C13 |
| C02 Database | C01 | C05, C07, C09 |
| C03 Capture Pipeline | C01, hardware | C04, C05, C06 |
| C04 PCAP Rotation | C03 | C06, C07 |
| C05 Ingest Bridge | C02, C03 | C06, C09 |
| C06 Gap Detector | C02, C04, C05 | — |
| C07 Deep Parser | C02, C04, hardware fixtures | C08 |
| C08 ML Enrichment | C07 | — |
| C09 REST API | C02 | C10 |
| C10 Frontend | C09 | — |
| C11 Grafana Dashboards | C02 | — |
| C12 Service Orchestration | All above | — |
| C13 Observability | C01 | C09, C12 |
| C14 Deployment Automation | C01 (starts at Layer 0, grows incrementally) | — |

### 4.4 Critical Path

```
C01 → C02 → C05 → C06
                         ↘
C01 → C03 → C04 → C07 → C08
                         ↘
C01 → C02 → C09 → C10   → C12
```

The critical path to a working end-to-end pipeline is:
**C01 → C02 → C03 → C05 → C09 → C10 → C12** (platform + DB + capture + ingest + API + frontend + orchestration)

### 4.5 MVP vs Full Build

**MVP (minimum viable pipeline):** C01 + C02 + C03 + C05 + C09 + C12
- Capture packets, ingest to DB, expose via API, orchestrated by systemd.
- No: gap detector, deep parser, ML, frontend, Grafana, rotation.

**Full v1 build:** All 14 components.

The MVP allows real-world testing with live hardware immediately, while the remaining components enhance reliability and usability.

---

## 5. Design Document Artifacts per Component

Every component design document MUST contain the following sections. This is the mandatory artifact set.

### 5.1 Required Sections (All Components)

| Section | Purpose | Required For |
|---------|---------|-------------|
| **1. Overview** | Purpose, scope, boundaries, position in the system | All |
| **2. High-Level Architecture (HLA)** | Component diagram, data flow, neighbouring components, contracts | All |
| **3. Data Model (ERD)** | Entities, relationships, fields, constraints, indexes | C02, C05, C06, C07, C08, C09 |
| **4. Low-Level Architecture (LLA)** | Module decomposition, class diagrams, key algorithms, error handling | C05, C07, C06, C09, C10 |
| **5. Inter-Component Contracts** | Input/output schemas, message formats, API endpoints | All (where contracts exist) |
| **6. Failure Modes & Recovery** | Failure catalog, detection, propagation, recovery procedures, SLA | All |
| **7. Observability** | Metrics exposed, logging format, health checks, alert rules | All |
| **8. Security Considerations** | Attack surface, authentication, input validation, data sensitivity | All |
| **9. Configuration** | Environment variables, config files, defaults, tuning parameters | All |
| **10. Test Plan** | Unit tests, integration tests, performance tests, acceptance criteria | All |
| **11. Deployment Notes** | Build steps, install paths, systemd units, dependencies | All |
| **12. Container Integration** | Quadlet unit verification, volume mount correctness, pod networking | C01, C12, C14 |

### 5.2 Per-Component Artifact Details

**Container Storage (Q11):** Each component design document must reference the volume inventory from `docs/designs/storage-strategy.md` and specify which volumes it mounts, with what access mode (`:ro` or `:rw`). The per-component artifact lists below have been updated with the relevant storage volumes.

#### C01 — Platform Infrastructure

| Artifact | Description |
|----------|-------------|
| HLA | Raspberry Pi CM5 hardware topology diagram (USB ports, dongles, storage) |
| HLA | USB power budget table |
| HLA | Thermal envelope analysis |
| Data Model | User/group/capability matrix |
| Failure Modes | USB disconnect, disk full, thermal throttle, power loss |
| Security | Disk encryption decision, physical security, sudo policy |
| Storage | Host directory bootstrap (V01-V05), tmpfiles.d config, tianer user with correct groups |

#### C02 — Database

| Artifact | Description |
|----------|-------------|
| HLA | PostgreSQL/TimescaleDB architecture, role hierarchy |
| **ERD** | Full entity-relationship diagram: `sniffers`, `sniffer_heartbeat`, `raw_packets` (hypertable), `device_summary`, `device_enrichment`, `ingest_gaps`, `_migrations` |
| ERD | `bluetooth` schema isolation — v1 tables under `bluetooth` schema, future modules under their own schemas |
| ERD | `raw_packets` unique constraint: query-time dedup via `DISTINCT ON (sniffer_id, ts, mac_address, pdu_type)` — no unique index at insert time. Dedup at query time not insert time per D-05/D-10. |
| ERD | Continuous aggregate dependency graph |
| LLA | Migration numbering and application protocol |
| Failure Modes | DB connection loss, corrupt hypertable, migration failure rollback |
| Observability | Table sizes, query performance, replication lag |
| Storage | V06 (tianer-postgres-data Podman volume), secrets via EnvironmentFile |

#### C03 — Capture Pipeline

| Artifact | Description |
|----------|-------------|
| HLA | Sniffer → tee → FIFO → tshark data flow diagram |
| HLA | Per-sniffer-type DLT and tshark field configuration |
| **Contract** `CAPTURE-1` | Sniffer wrapper → FIFO (PCAP binary format, DLT per sniffer type) |
| **Contract** `CAPTURE-2` | tshark output → ingest bridge (replaces CONTRACT 8.4-A, parameterized per DLT) |
| Contract Note | CAPTURE-2 produces a **normalized** pipe-delimited output regardless of source DLT. The per-DLT parameterization in `tshark-wrap.sh` ensures the ingest bridge receives a consistent schema. |
| **Contract** `CAPTURE-3` | Heartbeat protocol: local file primary (`/var/lib/tianer/heartbeat/<sniffer_name>.ts`), DB table secondary (`sniffer_heartbeat`). On DB outage, local file persists; on recovery, DB is backfilled from local file. |
| Configuration | Channel strategy: configurable per sniffer in `sniffers.yaml`; default channel 37; operator changes via config reload |
| Failure Modes | FIFO reader death, sniffer crash, DLT mismatch, backpressure cascade |
| Security | Shell injection via config values, USB device access |
| **BLE Protocol Notes** | Single-channel limitation, channel selection strategy, dewhitening status, CRC verification |
| Storage | V02 (:rw for PCAP), V03 (:rw for FIFOs), USB device passthrough, V08 (capture pod tmpfs) |

#### C04 — PCAP Rotation

| Artifact | Description |
|----------|-------------|
| HLA | Rotation trigger → rename → compress → delete lifecycle |
| **Contract** `ROTATION-1` | PCAP filename convention (YYYYMMDD-HHMM.pcap / .pcap.zst) |
| Failure Modes | Rotation during write (SIGHUP race), disk full, compression failure |
| Security | PCAP data sensitivity (contains BLE MACs) |
| Storage | V02 (:rw for rotation/compression) |

#### C05 — Ingest Bridge

| Artifact | Description |
|----------|-------------|
| HLA | Parser → Batcher → PgWriter pipeline |
| **ERD** | `Packet` struct ↔ `raw_packets` row mapping |
| LLA | Batcher flush policy: max 300K rows (~60MB RAM per sniffer instance) or 5-minute timeout, whichever first. This covers 5 minutes of DB downtime at 1000 pps. |
| LLA | `parse_line()` state machine, PgWriter reconnection logic |
| LLA | Write-ahead log / checkpoint for crash recovery |
| **Contract** `INGEST-1` | tshark field input schema (replaces CONTRACT 8.5-A) |
| **Contract** `INGEST-2` | Ingest bridge → PostgreSQL COPY protocol |
| Failure Modes | PG outage (buffer sizing + recovery), malformed input, OOM kill, partial batch loss |
| Observability | `blesniff_packets_ingested_total`, `blesniff_ingest_latency_seconds`, `blesniff_malformed_packets_total`, buffer depth gauge |
| Security | Input validation ranges, field length limits |
| Storage | V03 (:rw for ingest FIFO read) |

#### C06 — Gap Detector

| Artifact | Description |
|----------|-------------|
| HLA | Gap detection → backfill → verification cycle |
| LLA | Gap definition algorithm (CONTRACT 8.6-A) |
| LLA | Backfill strategy (PCAP location, reparse, idempotent insert) |
| **Contract** `GAP-1` | Heartbeat-based gap definition |
| **Contract** `GAP-2` | PCAP filename → gap window mapping |
| Failure Modes | Missing PCAP, corrupted PCAP, heartbeat false negative, backfill INSERT failure |
| Observability | `blesniff_gap_detected_total`, `blesniff_backfill_rows_total` |
| **Secondary verification** | Periodic PCAP row count vs DB row count comparison |
| Storage | V02 (:ro for backfill PCAP reads) |

#### C07 — Deep Parser

| Artifact | Description |
|----------|-------------|
| HLA | PCAP input → BLE dissector → AdvData TLV → JSONL output |
| **ERD** | JSONL output schema ↔ `device_enrichment` table mapping |
| LLA | BLE PDU dissector state machine, AdvData TLV parser |
| **Contract** `DEEP-1` | JSONL output schema (replaces CONTRACT 8.8-A) |
| Failure Modes | Corrupted PCAP, malformed PDU, decompression failure, output I/O failure |
| Security | PCAP input validation, memory bounds (100MB limit) |
| **BLE Protocol** | DLT type expected, dewhitened data assumption |
| **BLE Protocol** | CRC-24 validation: configurable validation step (default: enabled). When disabled, bypass validation for sniffer sources known to pre-verify CRC (e.g., Ubertooth One firmware). Validation failure increments `blesniff_crc_errors_total` metric. |
| Storage | V02 (:ro for PCAP source), V05 (:rw for JSONL output) |

#### C08 — ML Enrichment

| Artifact | Description |
|----------|-------------|
| HLA | JSONL reader → rule classifier → DB writer |
| LLA | Rule matching engine (manufacturer ID → service UUID → class) |
| Data Model | `device_summary.enrichment_data` JSONB schema |
| Failure Modes | JSONL parse error, DB write failure, unknown manufacturer |
| **Contract** `ML-1` | Deep parser JSONL → ML enrichment input (required vs optional fields) |
| Storage | V05 (:ro for JSONL input) |

#### C09 — REST API

| Artifact | Description |
|----------|-------------|
| HLA | FastAPI app → routers → DB pool → static frontend |
| **ERD** | Pydantic models ↔ DB tables mapping |
| LLA | Auth middleware, query builder, pagination |
| **Contract** `API-1` | REST endpoint definitions (replaces CONTRACT 8.10-A, with MAC encoding) |
| Failure Modes | DB down, slow queries, auth failure |
| Security | Parameterized queries mandate, TLS requirement, rate limiting, API key rotation |
| Observability | Request latency histogram, error rate, active connections |
| Storage | V01 (:ro for TLS certs), frontend static files embedded in image |

#### C10 — Frontend

| Artifact | Description |
|----------|-------------|
| HLA | Vue app → Pinia stores → API client → REST API |
| LLA | Component tree, routing, state management |
| Failure Modes | API unreachable, stale data, auth failure |
| Security | API key stored where? (never in client-side code) |
| Observability | Error boundary reporting, loading states |
| Storage | None — served from API container image |

#### C11 — Grafana Dashboards

| Artifact | Description |
|----------|-------------|
| HLA | Grafana → TimescaleDB datasource, provisioning chain |
| LLA | Per-dashboard SQL queries, panel layout |
| Failure Modes | Datasource down, query timeout, provisioning failure |
| Security | Bind address, anonymous access scope |
| Storage | V07 (grafana-data Podman volume) |

#### C12 — Service Orchestration

| Artifact | Description |
|----------|-------------|
| HLA | systemd dependency graph, target hierarchy |
| LLA | Per-service ExecStart/Stop, ordering, sandboxing |
| Failure Modes | Circular dependency, service crash loop, ordering violation |
| Security | Per-service ReadWritePaths, NoNewPrivileges, SystemCallFilter |
| Storage | Quadlet unit file deployment to `/etc/containers/systemd/` |

#### C13 — Observability

| Artifact | Description |
|----------|-------------|
| HLA | Metrics collection → exposition → scraping architecture |
| Data Model | Full Prometheus metrics catalogue with labels |
| LLA | Per-service metrics exposition mechanism (file-based for C++, HTTP for Python) |
| **Contract** `OBS-1` | Metrics file format and directory convention |
| Failure Modes | Metrics collector down, stale metrics, metric cardinality explosion |
| Storage | V04 (:rw for structured logs) |

#### C14 — Deployment Automation

| Artifact | Description |
|----------|-------------|
| HLA | `setup.sh` phase diagram (16 steps) |
| LLA | Per-phase idempotency guarantees, rollback procedures |
| Failure Modes | Partial setup failure, version mismatch, rollback |
| Security | Supply chain integrity: GPG verification for apt, SHA256 checksums for downloaded binaries (nrfutil, ubertooth), digest pinning for container images (`@sha256:`), hash-pinning for Python (`uv --require-hashes`), `npm audit` in CI |
| **Storage budget** | Formal disk space calculation per component |
| Storage | Host directory creation, Podman volume creation, tmpfiles install |

### 5.3 Contract Registry (Consolidated)

All inter-component contracts are numbered in this project's namespace:

| Contract ID | From | To | Format | Replaces |
|-------------|------|-----|--------|----------|
| CAPTURE-1 | Sniffer wrapper | FIFO | Binary PCAP per DLT | — |
| CAPTURE-2 | tshark | Ingest bridge | Pipe-delimited fields per DLT config | CONTRACT 8.4-A |
| CAPTURE-3 | Heartbeat | DB | SQL INSERT | — |
| INGEST-1 | tshark line | Parser | C++ `Packet` struct | CONTRACT 8.5-A |
| INGEST-2 | PgWriter | PostgreSQL | COPY protocol | — |
| ROTATION-1 | Rotation | Filesystem | Filename convention | — |
| GAP-1 | Detector | DB | Gap definition rule | CONTRACT 8.6-A |
| GAP-2 | Backfill | PCAP files | Filename → bucket window | — |
| DEEP-1 | Deep parser | ML / filesystem | JSONL with CRC status field, sniffer-type field, per-DLT metadata | CONTRACT 8.8-A |
| ML-1 | ML enrichment | DB | enrichment_data JSONB | — |
| API-1 | API | Frontend | REST/JSON | CONTRACT 8.10-A |
| OBS-1 | All services | Scraper | Metrics file/HTTP format | — |

---

## 6. Engineering-for-Failure Principles

These principles must be embedded in every design document. They are non-negotiable for a system that is "reliable enough to depend on."

### 6.1 Principles

| # | Principle | Description | Example in Tian'er |
|---|-----------|-------------|---------------------|
| PF-1 | **Design for Failure** | Every component must document what fails, how it's detected, and how it recovers. No component may have an undocumented failure mode. | Ingest bridge documents OOM kill, PG outage, malformed input — and recovery for each. |
| PF-2 | **Recovery by Design** | Every data path must have a documented recovery mechanism. If data can be lost, the loss must be detectable and quantifiable. | PCAP files are the recovery source for DB ingest gaps. Gap detector is the recovery agent. Secondary verification detects if gap detector itself misses gaps. |
| PF-3 | **Redundant Detection** | Critical failure conditions must be detectable by at least two independent mechanisms. | Ingest gaps detected by: (1) gap detector vs heartbeat, (2) periodic PCAP packet count vs DB row count comparison. |
| PF-4 | **Graceful Degradation** | When a component fails, the system degrades gracefully, not catastrophically. Partial function is better than total function. | If one sniffer crashes, others continue. If API is down, Grafana still reads DB. If gap detector is down, alerts fire. |
| PF-5 | **Observability by Design** | Every failure mode must produce an observable signal. Silent failures are the highest severity class. | Every component exposes health metrics. Structured logs. Dead-letter files for malformed data. Metrics for every failure counter. |
| PF-6 | **Blast Radius Minimization** | A compromise or failure in one component must not automatically cascade to others. | Per-service systemd sandboxing. API key auth (not shared DB credentials for frontend). ReadWritePaths per service. |
| PF-7 | **Defense in Depth** | Security controls at every layer, not just the perimeter. | TLS on API, API key auth, parameterized queries, input validation, file permissions, pg_hba restrictions, systemd sandboxing. |
| PF-8 | **Crash-Only Design** | Components should be safe to kill at any time. Recovery on restart must be automatic and correct. | Ingest bridge: Batcher data lost on OOM, but recoverable from PCAP via gap detector. No "clean shutdown required" dependencies. |
| PF-9 | **No Single Point of Recovery** | If a single component performing recovery is itself down, an alert must fire immediately. | Gap detector down triggers a critical alert. Secondary gap verification runs on a different schedule. |
| PF-10 | **Quantified Data Loss SLA** | "No data loss" must be precisely defined. Where loss is possible, the acceptable window and detection mechanism must be stated. | PCAP: no data loss (disk is truth). DB ingest: up to 30-second bucket granularity delay during normal operation; up to 5-minute recovery window via gap detector. |

### 6.2 Implementation Phases

These principles are implemented in two phases:

1. **MVP Phase:** Design documents include Failure Modes & Recovery sections for every component. The MVP pipeline (C01+C02+C03+C05+C09+C12) implements basic failure detection and restart.
2. **Post-MVP Phase:** Add redundant detection (secondary gap verification), alerts for recovery-critical services, structured logging, TLS, secrets rotation, disk encryption, and the full observability stack.

---

## 7. Resolved Decisions

All decisions from the specialist reviews have been resolved. See `docs/project-management/passports/TIANER-001-passport.md` for the complete decision log (24 decisions: D-01 through D-12, NEW container orchestration, Q1 through Q11).

| ID | Decision | Resolution |
|----|----------|------------|
| D-01 | Database name | `tianer` |
| D-02 | Env var prefix | `TIANER_*` (shared), `BLESNIFF_*` (module) |
| D-03 | nRF USB PID | Both `522A` (sniffer) and `520f` (DFU); determined in detail design |
| D-04 | USB topology | Powered USB hub |
| D-05 | Channel strategy | Configurable per sniffer; dedup at query time |
| D-06 | API TLS | Self-signed cert, mandatory |
| D-07 | Disk encryption | Accepted risk for MVP |
| D-08 | Metrics exposition | File-based Prometheus text format |
| D-09 | Ingest buffer | 300K rows, 5 min, ~60MB RAM |
| D-10 | Dedup index | Add `pdu_type`; query-time dedup |
| D-11 | Heartbeat fallback | Local file primary, DB secondary |
| D-12 | tshark config | Per-DLT parameterized |
| NEW | Container orchestration | Podman + Quadlet, all-container |
| Q1 | Container model | All containers, rootless Podman, USB passthrough |
| Q2 | Monorepo layout | `tian-er/` authoritative (component-breakdown) |
| Q3 | nRF PID determination | During detailed design with hardware |
| Q4 | USB symlinks | Persistent udev device names |
| Q5 | Bluetooth schema | Isolated `bluetooth` schema |
| Q6 | Channel strategy | Configurable per sniffer |
| Q7 | CRC-24 | Verify in C07 Deep Parser |
| Q8 | Secrets rotation | Deferred to post-MVP |
| Q9 | DB roles | Write-only for streams, read-only for UI |
| Q10 | Supply chain integrity | Required for MVP |
| Q11 | Storage strategy | 7 volumes, 2 pods, per-container matrix (see storage-strategy.md) |