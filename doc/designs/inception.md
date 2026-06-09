# 天耳 (Tian'er) - Signal Intelligence Platform

## v1 Technical Design: Passive Bluetooth Sensor Module

**Status:** Draft v0.5
**Author:** Jean (with Claude)
**Last updated:** 2026-06-07
**Target audience:** Autonomous coding agents implementing the platform end to end.

---

## About 天耳 (Tian'er)

The project is named **天耳**, pronounced *Tian'er*, "Heavenly Ear." The name is a deliberate companion to **天眼** (*Tianyan*, "Heavenly Eye"), the spirit of which inspires this work. Tianyan looks outward at the universe through one giant aperture. Tian'er listens inward, to the sky directly overhead and the room directly around it, to anything that propagates through the air. The ambition is the same: build the instrument, point it at reality, see what is actually there.

Tian'er is a platform, not a single sensor. Wireless emissions are the first frontier because they are the easiest and the densest layer of ambient information around a modern household. But the architectural choice is deliberate: the same data pipeline that handles Bluetooth advertisements is designed to eventually carry GPS observations, ADS-B flight telemetry, broadcast radio metadata, Wi-Fi probe requests, cellular passive signaling, ISM-band telemetry from sensors and appliances, microwave oven leakage signatures, and whatever else turns out to be worth listening for. Section "Scope and Roadmap" below catalogs the planned modules. Under the sky, anything that propagates can find a home in this project.

The deeper inspiration is philosophical, not technical. There is a particular kind of innovation that emerges only under constraint, when a builder is told that a given system is closed to them. Denied the established path, the builder does not wait for permission; they construct their own, and they often construct it faster, more deliberately, and ultimately more capable than the original. This pattern has repeated across many domains in recent decades, at the scale of entire industries: indigenous chip design and fabrication, indigenous launch vehicles and orbital programs, indigenous high-speed rail networks, indigenous highway and aviation systems, indigenous defense capabilities. Each of those began as a deficit. Each ended as a capability that exists today because someone, somewhere, decided the answer to "you cannot have this" was "watch."

The principle generalizes downward as well as upward. The lesson, stripped of any larger framing, is simple: when you are denied the right to live inside someone else's reality, you build your own; and if you build with care and patience, the one you build is the better one.

Tian'er is the personal-scale expression of that idea. The author has no factory, no national budget, no thousand-engineer institute. The available raw material is one small computer, a handful of hobbyist sensors, a deep stack of free software that the world's open-source community has already built, the time to learn what is needed, and an unreasonable enthusiasm for the work. The bet of this project is that those ingredients, combined with deliberate architectural design and patient iteration, are sufficient to produce a real instrument: not a demonstration, not a toy, but a tool that yields useful intelligence about the propagating environment it inhabits and that does so reliably enough to depend on.

In keeping with that intent, the rest of this document is precise where it can be and honest where it must be. The architecture prioritizes reliability over novelty. The implementation prefers off-the-shelf, well-documented, openly verifiable components. Every version pin and procedural recommendation is cited so that an autonomous agent or a future maintainer can verify the source. The work is done in the open.

**A note on naming:** the project as a whole is 天耳 / Tian'er. The technical namespace used throughout the codebase, in systemd units, in file paths, in environment variables, and in PostgreSQL role names is the shorter ASCII identifier `tianer`. The current v1 sensor module additionally uses the prefix `blesniff` for components that are specifically Bluetooth-related (sniffer wrapper, ingest bridge, deep parser). Future sensor modules will introduce their own prefixes (`gpsrx`, `adsbrx`, `cellrx`, and so on) alongside the shared `tianer` infrastructure. Where this document uses `blesniff`, the agent should read it as "the v1 Bluetooth sensor module of Tian'er."

---

## Scope and Roadmap

Tian'er is built in successive sensor modules. Each module is independently shippable, runs alongside the others, and uses the same shared infrastructure (PostgreSQL with TimescaleDB, FastAPI, Vue.js frontend, Grafana dashboards, systemd orchestration, the ingest and gap-detection patterns described in section 8). The shared infrastructure is built once. Sensor modules plug into it.

### v1 Scope (this document)

| Module | Status | Sensors |
|--------|--------|---------|
| Bluetooth (BLE + BR/EDR survey) | v1, this document | Ubertooth One, Nordic nRF52840 dongle |

The entire remainder of this document (sections 1 through 19) specifies the v1 Bluetooth sensor module and the shared infrastructure it sits on. Sections 1 through 7 describe choices that the shared infrastructure inherits; sections 8 onward describe the Bluetooth-specific components.

### Planned Future Modules

These are not specified in this document. They are listed here so that the v1 architecture remains compatible with them and so that future contributors understand the long-term shape of the project. Each entry will receive its own design document when it is the active work front.

| Module | Sensor candidates | Notes |
|--------|-------------------|-------|
| GNSS / GPS | Software-defined radio (RTL-SDR, HackRF) or dedicated GPS receiver with NMEA output | Passive observation of satellite ephemeris and PVT data; useful as ground truth for time and location. |
| ADS-B flight telemetry | RTL-SDR or dedicated 1090 MHz receiver running `dump1090` | Aircraft position, identification, altitude, velocity broadcast at 1090 MHz. Well-defined open ecosystem. |
| Wi-Fi probe and management frames | Atheros / Ralink monitor-mode adapters or ESP32 in promiscuous mode | Passive observation of probe requests, beacons, deauths. Complements BLE for device fingerprinting. |
| Cellular passive signaling | RTL-SDR with appropriate front-end; tools like `srsRAN` or `gr-gsm` for 2G, `srsLTE` for LTE | Limited to broadcast channels; no decryption. |
| ISM-band telemetry (433 MHz, 868 MHz, 915 MHz) | RTL-SDR with appropriate front-end; `rtl_433` | Smart meters, weather stations, garage doors, doorbells, environmental sensors. |
| Microwave oven and consumer-appliance RF leakage | Broadband 2.4 GHz monitor (RTL-SDR or a tuned receiver) | Identifies appliance activity by its incidental RF signature. |
| Broadcast FM / DAB radio | RTL-SDR | RDS metadata, station identification, signal strength surveys. |
| LoRaWAN | RTL-SDR or dedicated LoRa concentrator | Long-range telemetry from environmental and infrastructure sensors. |
| AIS (marine vessel tracking) | RTL-SDR at 162 MHz | Vessel position broadcasts; relevant on coastal deployments. |
| Acoustic and ultrasonic | Calibrated microphone | Different propagation medium but the same architectural pattern: continuous capture, rolling on-disk recording, streaming feature extraction, time-series database. |

### Cross-Module Architectural Commitments

The v1 design holds these properties so that future modules can reuse the infrastructure without restructuring:

1. **Per-module sensor wrapper.** Each sensor type has its own wrapper that dual-writes to a rotating raw file and a FIFO. The pattern is identical to section 8.1; only the binary changes (`ubertooth-btle`, `nrfutil ble-sniffer`, `dump1090`, `rtl_433`, `gpsd`, etc.).
2. **Per-module ingest path.** Each sensor type has its own ingest bridge that knows how to parse its sensor's output and write to its own observation table. The bridge follows the contract in section 8.5 (batched `COPY` into PostgreSQL).
3. **Per-module schema.** Each sensor type owns its own tables under the same database. The v1 schema introduces `sniffers`, `sniffer_heartbeat`, `raw_packets`, `device_summary`, and `device_enrichment`. Future modules will introduce parallel tables: `gps_observations`, `adsb_observations`, `aircraft_summary`, and so on. Schema migrations remain numbered globally.
4. **Shared database, shared API, shared frontend, shared dashboards.** TimescaleDB, FastAPI, Vue.js, and Grafana are not re-deployed per module. They are extended.
5. **Shared gap detection and reliability pattern.** The gap detector in section 8.6 is generic: it monitors `(sensor_id, time_bucket)` zeros against a heartbeat table and backfills from rotated raw files. Future modules implement the same heartbeat and rotation conventions and reuse the detector logic with minor parameterization.

### What This Means for v1 Implementation

Agents implementing v1 must:
* Build the Bluetooth module to the spec in sections 8 through 14 as the active deliverable.
* When naming shared infrastructure components (database, API, frontend, configuration directories), use the project-level identifier `tianer` rather than the module-level identifier `blesniff`. Examples: `/etc/tianer/blesniff.env` for the Bluetooth-specific env file under the project's shared config root; PostgreSQL database name `tianer` with a `bluetooth` schema; FastAPI routes under `/api/bluetooth/`.
* When naming module-specific components (sniffer wrappers, ingest bridges, parsers), use the module identifier `blesniff`.
* Resist the temptation to over-generalize v1. The Bluetooth schema does not need to be sensor-agnostic; it needs to be correct and fast for Bluetooth. Future modules add new tables, not new abstractions over existing ones.

**DECISION REQUIRED 0.1:** Confirm the prefix scheme. `RESOLVED (D-01, D-02): project = tianer, v1 module = blesniff, future modules use their own short prefixes. Database name = tianer. Shared env vars = TIANER_*, module env vars = BLESNIFF_*.`

---

## 0. How to Use This Document

This document is structured for spec-driven development by autonomous agents. Conventions:

| Marker | Meaning |
|--------|---------|
| **DECISION REQUIRED N.N** | A choice that affects implementation. A `RECOMMENDED DEFAULT` is provided. Agents implement the recommended default and mark the choice in code with a `// DECISION 5.5.1` comment so it can be revisited. |
| **ACCEPTANCE CRITERIA** | The observable conditions that must be true for a task to be considered complete. |
| **VERIFICATION** | The exact commands or tests that prove the acceptance criteria are met. |
| **CONTRACT** | A formal interface between components. Changes must be coordinated. |
| `[^source-id]` | Citation reference. The full source is listed in the References section at the end of the document. Every version pin, protocol claim, and procedural recommendation that depends on upstream documentation is cited so agents can verify the underlying source independently. |

Agents must run the verification commands for a task before reporting it complete. If a verification fails, the agent must diagnose and fix before moving to the next task.

All paths are absolute unless otherwise noted. All commands assume the user `tianer` (created in section 6.1) unless noted as `root`.

---

## 1. System Goals

This document specifies the v1 Bluetooth sensor module of the Tian'er platform. The goals listed here apply to that module. The shared infrastructure (database, API, frontend, dashboards, orchestration) is built once and inherited by future modules per the roadmap above.

The v1 Bluetooth module must:

1. Continuously capture Bluetooth (BR/EDR and BLE) packets from one to four sniffers in parallel.
2. Persist all captured packets to disk in rotating PCAP files, with no acceptable data loss.
3. Ingest packet metadata into a time-series database with end-to-end latency under 5 seconds (P95).
4. Detect and backfill any gap in the real-time ingest pipeline from on-disk PCAP files.
5. Provide aggregated reporting:
   * Total observation count per device.
   * Time since last observation per device (resolved from milliseconds to days).
   * Immediate flagging of previously unseen devices.
   * Residency classification of devices observed regularly over time.
6. Run a separate batch pipeline for deep packet inspection of payload contents, feeding higher-level summaries and machine-learning analysis.
7. Present results through a mixed dashboard (Grafana panels plus a custom Vue.js interface).

Beyond the module itself, v1 also delivers the shared platform infrastructure that future modules will reuse: PostgreSQL with TimescaleDB, the FastAPI backend skeleton, the Vue.js frontend skeleton, the Grafana provisioning, the systemd orchestration patterns, and the user/group/permissions setup in section 6.

---

## 2. Non-Goals (v1 Module)

The following are out of scope for the v1 Bluetooth module specifically. Items marked with [future module] are explicitly planned for later releases of Tian'er per the Scope and Roadmap above; agents must not implement them in v1 but should not preclude them either.

* Active probing or device interaction. Tian'er is passive only across all modules.
* Real-time decryption of paired/encrypted BLE traffic.
* Cloud or off-device storage.
* Location triangulation across multiple Bluetooth sniffers (the data is captured to enable it later within the Bluetooth module).
* IRK-based BLE address resolution.
* Capture of non-Bluetooth signals [future module]: GPS, ADS-B, Wi-Fi, cellular, ISM band, FM/DAB, LoRaWAN, AIS, acoustic. These are entirely out of scope for v1 but reserved for future modules.
* Cross-sensor-type correlation [future module]: linking a Bluetooth device observation to a Wi-Fi probe from the same physical device, for example, is a v2-or-later concern.

---

## 3. Hardware Environment

| Item | Specification |
|------|--------------|
| Host | Raspberry Pi Compute Module 5, 8 GB RAM |
| Primary storage | eMMC 32 GB |
| Secondary storage | SanDisk SD card (data partition) |
| Optional storage | NVMe (not provisioned in v1) |
| Sniffers in scope | Ubertooth One, Nordic nRF sniffer (nrfutil-driven) |
| Sniffers excluded from v1 | nRF with AT-command serial output, ESP32 hex dumps |
| Concurrent sniffers | 1 to 4, configurable |

**OS:** Raspberry Pi OS 64-bit (Trixie, Debian 13). See ADR-0001.

**USB mapping:** Persistent udev device names on host per D-03/Q4. See ADR-0001.

---

## 4. Repository Structure

The entire platform lives in a single monorepo. Agents must respect this layout. New files outside this structure require a design update.

```
tian-er/
├── README.md                          # Project overview, quickstart
├── Makefile                           # Top-level commands (see section 13.1)
├── .editorconfig
├── .gitignore
├── .pre-commit-config.yaml            # Pre-commit hooks (formatters, linters)
├── doc/
│   └── designs/                        # Design documents
│       ├── inception.md                # This document
│       ├── component-breakdown.md
│       ├── storage-strategy.md
│       └── ...                         # Per-component design docs (c01 through c14)
│
├── deploy/
│   ├── setup.sh                       # Idempotent host bootstrap
│   ├── udev/
│   │   └── 99-tianer.rules
│   ├── containers/                     # Quadlet unit files
│   ├── systemd/
│   │   ├── blesniff-sniffer@.service
│   │   ├── blesniff-tshark@.service
│   │   ├── blesniff-ingest@.service
│   │   ├── blesniff-rotate.service
│   │   ├── blesniff-rotate.timer
│   │   ├── blesniff-gap-detector.service
│   │   ├── blesniff-gap-detector.timer
│   │   ├── blesniff-api.service
│   │   └── blesniff.target            # Aggregates all services
│   ├── scripts/                       # Helper shell scripts
│   │   ├── create-dirs.sh
│   │   ├── install-deps.sh
│   │   ├── install-postgres.sh
│   │   ├── install-grafana.sh
│   │   └── rotate-pcap.sh
│   └── config/
│       └── tianer.env.example         # Documented environment template
├── modules/
│   └── bluetooth/
│       ├── sniffers/                   # Shell scripts that wrap each sniffer binary
│       │   ├── ubertooth-wrap.sh
│       │   ├── nrf-wrap.sh
│       │   └── tests/
│       ├── ingest-bridge/              # C++17, libpqxx
│       │   ├── CMakeLists.txt
│       │   ├── src/
│       │   │   ├── main.cpp
│       │   │   ├── parser.cpp / .hpp   # Parse tshark fields output
│       │   │   ├── batcher.cpp / .hpp  # Time + size batching
│       │   │   ├── pg_writer.cpp / .hpp # PostgreSQL COPY
│       │   │   └── config.cpp / .hpp
│       │   └── tests/                  # GoogleTest
│       ├── gap-detector/               # Python 3.13
│       │   ├── pyproject.toml
│       │   ├── src/tianer_gapdet/
│       │   │   ├── __init__.py
│       │   │   ├── detector.py
│       │   │   ├── backfill.py
│       │   │   ├── pcap_reader.py
│       │   │   └── db.py
│       │   └── tests/                  # pytest
│       ├── deep-parser/                # C++17, libpcap
│       │   ├── CMakeLists.txt
│       │   ├── src/
│       │   │   ├── main.cpp
│       │   │   ├── pcap_input.cpp / .hpp
│       │   │   ├── ble_dissector.cpp / .hpp
│       │   │   ├── advdata_parser.cpp / .hpp
│       │   │   └── jsonl_output.cpp / .hpp
│       │   └── tests/
│       └── ml-enrichment/              # Python 3.13
│           ├── pyproject.toml
│           ├── src/tianer_ml/
│           │   ├── classifier.py
│           │   ├── clusterer.py        # RPA clustering (future)
│           │   └── features.py
│           └── tests/
├── platform/
│   ├── api/                            # Python FastAPI
│   │   ├── pyproject.toml
│   │   ├── src/tianer_api/
│   │   │   ├── main.py
│   │   │   ├── routers/
│   │   │   ├── models.py              # Pydantic models
│   │   │   └── db.py
│   │   └── tests/
│   └── frontend/                       # Vue 3 + Vite + TypeScript
│       ├── package.json
│       ├── tsconfig.json
│       ├── vite.config.ts
│       ├── src/
│       │   ├── main.ts
│       │   ├── App.vue
│       │   ├── router/
│       │   ├── stores/                 # Pinia
│       │   ├── views/                  # DeviceList, DeviceDetail, Alerts, Health
│       │   ├── components/
│       │   ├── api/                    # Generated TypeScript API client
│       │   └── types/
│       └── tests/                      # Vitest + Vue Test Utils
├── db/
│   ├── migrations/                    # Numbered SQL migrations
│   │   ├── 0001_init.sql
│   │   ├── 0002_continuous_aggregates.sql
│   │   ├── 0003_compression_policies.sql
│   │   └── 0004_residency_classifier.sql
│   ├── seed/                          # Seed data for tests
│   └── tests/                          # pgTAP tests
├── grafana/
│   ├── provisioning/
│   │   ├── datasources/
│   │   │   └── timescaledb.yaml
│   │   └── dashboards/
│   │       └── default.yaml
│   └── dashboards/                    # Dashboard JSON
│       ├── live-metrics.json
│       ├── device-explorer.json
│       ├── per-device-drilldown.json
│       └── pipeline-health.json
├── tools/
│   ├── pcap-replay.py                 # Replay PCAP into FIFO for testing
│   ├── mock-sniffer.py                # Stub sniffer for integration tests
│   └── seed-test-data.py              # Generate synthetic raw_packets rows
├── tests/
│   ├── fixtures/
│   │   ├── pcap/                      # Real captured PCAP samples
│   │   │   ├── ubertooth-sample-001.pcap
│   │   │   └── nrf-sample-001.pcap
│   │   └── expected/                  # Expected outputs for golden tests
│   ├── integration/                   # Cross-service tests
│   │   ├── test_ingest_pipeline.sh
│   │   ├── test_gap_detection.sh
│   │   └── test_api_to_db.sh
│   └── e2e/                           # Full-stack smoke tests
│       └── smoke.sh
└── ci/
    ├── lint.sh
    ├── test-all.sh
    └── build-all.sh
```

---

## 5. Tech Stack (Pinned Versions)

Version pins below are based on the verified state of upstream releases as of 2026-06-06. All listed versions are minimum acceptable; agents prefer the latest patch release within the listed major version line. CI verifies installed versions match the pin.

| Layer | Tool | Version | Source / Justification |
|-------|------|---------|------------------------|
| OS | Raspberry Pi OS 64-bit | Trixie (Debian 13) | Released by Raspberry Pi October 2025 [^rpi-trixie]; based on Debian 13 (released August 2025) [^debian-trixie-release] |
| Database | PostgreSQL | 17 | Default version in Debian Trixie apt repository [^debian-pg17]; PG 17 LTS supported until 2029 [^pg-support] |
| TS extension | TimescaleDB | 2.23 or later | First TimescaleDB version with PostgreSQL 18 support; supports PG 16, 17, 18 [^ts-pg18]. Install via PGDG/Timescale apt repo [^ts-install] |
| C++ compiler | g++ | 14.2 | Default in Debian Trixie [^debian-trixie-release]; C++17 baseline, C++20 features allowed |
| C++ build | CMake | 3.25 minimum | Required for modern target features. Trixie ships >= 3.25 |
| C++ test | GoogleTest | 1.14 | Stable, widely supported |
| C++ Postgres | libpqxx | 7.8 | Stable C++ binding for PG 17 wire protocol [^libpqxx] |
| C++ PCAP | libpcap | 1.10 | Default in Debian Trixie |
| Python | CPython | 3.13 | Default Python 3 on Debian Trixie [^debian-py313]; latest minor is 3.14.3 (Feb 2026) [^py3143] but 3.13 is what apt installs without extra repos. 3.13.12 is the latest 3.13 maintenance release [^py31312] |
| Python pkg mgr | uv | 0.4 or later | Faster than pip, reproducible locks |
| Python test | pytest | 8.0 | |
| Python async test | pytest-asyncio | 0.23 | |
| FastAPI | fastapi | 0.110 or later | |
| FastAPI server | uvicorn | 0.27 or later | |
| Async PG | psycopg | 3.1 | psycopg[binary] |
| PCAP (Python) | pyshark | 0.6 | wraps tshark |
| Wireshark | tshark | 4.2 or later | apt: `tshark` package. CONTRACT 8.4-A requires the `-T fields` and `-l` options that have been stable since 2.x |
| Ubertooth | ubertooth-tools | matching firmware release | Built from source per upstream guide [^ubertooth-getting-started] |
| Nordic | nrfutil | 7.x with ble-sniffer subcommand | Official Nordic command-line utility [^nrfutil] |
| Node.js | node | 24 LTS | Active LTS until October 2026 [^node-lts]; subsequent LTS line is Node 26 (LTS October 2026). Install via NodeSource apt repo for ARM64 |
| Frontend build | Vite | 5.0 or later | |
| Frontend | Vue | 3.4 or later | |
| Frontend state | Pinia | 2.1 or later | |
| TypeScript | tsc | 5.3 or later | strict mode |
| Frontend test | Vitest | 1.2 or later | |
| Vue test | @vue/test-utils | 2.4 or later | |
| Dashboards | Grafana | 10.4 or later | apt: `grafana` package via official Grafana apt repo |
| Process mgr | systemd | 257 (host default on Trixie) [^debian-trixie-release] | |
| Compression | zstd | 1.5 or later | Default on Trixie |
| Pre-commit | pre-commit | 3.6 or later | |
| Linter (C++) | clang-tidy | 19 (default on Trixie) [^debian-trixie-release] | |
| Formatter (C++) | clang-format | 19 (default on Trixie) | |
| Linter (Python) | ruff | 0.3 or later | |
| Formatter (Python) | ruff format | 0.3 or later | replaces black |
| Type check (Python) | mypy | 1.8 or later | strict |

> Note on PostgreSQL major version: the choice of PG 17 over PG 18 is deliberate. PG 17 is the version shipped by the Debian Trixie apt repository [^debian-pg17], which means agents can install it via `apt install postgresql-17 postgresql-server-dev-17` without adding the PGDG repository. PG 18.4 is the current stable upstream [^pg184] and PG 19 Beta 1 is available as of June 4, 2026 [^pg19-beta], but agents would need to add the PGDG apt repository to obtain them. TimescaleDB 2.23 and later support PG 16, 17, and 18 [^ts-pg18], so the choice can be revisited. See ADR-0001 for the resolved decision.

**PostgreSQL:** 17 (Trixie default). See ADR-0001.

**Python:** 3.13 (Trixie default). See ADR-0001.


---

## 6. Prerequisites and Configuration

This section defines all OS-level prerequisites that must be satisfied before any service in section 8 can run. It covers user and group setup (so that no service requires `root` or `sudo` at runtime), file capabilities for tools that need privileged access (notably `dumpcap`), udev rules that grant hardware access via group membership, file system layout, and runtime configuration files.

The guarantee: after section 6.1 through 6.4 are complete, every command listed in section 8 onwards can be executed by the `tianer` system user with no `sudo` and no `root`.

### 6.1 Service User and Group Setup

A dedicated unprivileged system user `tianer` owns all platform processes. It is added to the following groups:

| Group | Reason | Reference |
|-------|--------|-----------|
| `tianer` | Primary group, owns service files | n/a |
| `plugdev` | USB device access for Ubertooth via udev rule | Ubertooth Getting Started guide [^ubertooth-getting-started] |
| `dialout` | Serial port access (`/dev/ttyACM*` for nRF dongles) | Debian convention; default group ownership of TTY devices |
| `wireshark` | Permission to invoke `dumpcap` for live capture (which tshark uses when reading from a FIFO) | Wireshark capture privileges documentation [^wireshark-privileges] |

**Implementation** (runs once at host bootstrap; idempotent):

`deploy/scripts/create-user.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

# 1. Create tianer system user with home in /var/lib/tianer, no login shell.
if ! id -u tianer >/dev/null 2>&1; then
    useradd --system \
            --home-dir /var/lib/tianer \
            --create-home \
            --shell /usr/sbin/nologin \
            --user-group \
            --comment "Tian'er Signal Intelligence Platform service user" \
            tianer
fi

# 2. Ensure required groups exist. plugdev and dialout exist by default on
#    Debian; wireshark is created when wireshark-common is configured to allow
#    non-root capture (see section 6.3). Create wireshark group if not present.
for grp in plugdev dialout wireshark; do
    if ! getent group "$grp" >/dev/null; then
        groupadd --system "$grp"
    fi
done

# 3. Add tianer to all required groups. usermod -aG is the idempotent form.
usermod -aG plugdev,dialout,wireshark tianer

# 4. Verify membership.
groups tianer
```

The script uses `usermod -aG` to append group memberships [^usermod-man]. The `-a` flag preserves existing memberships; without it, `usermod -G` would replace them and lock the user out of any unlisted groups.

**ACCEPTANCE CRITERIA for 6.1:**
1. `id tianer` lists `tianer` plus `plugdev`, `dialout`, `wireshark` among its groups.
2. The user has no login shell (verified by `getent passwd tianer | awk -F: '{print $7}'` returning `/usr/sbin/nologin`).
3. Re-running `create-user.sh` produces no errors.

**VERIFICATION:**
```bash
sudo deploy/scripts/create-user.sh
id tianer | grep -qE 'plugdev.*dialout.*wireshark|wireshark.*plugdev|dialout.*plugdev.*wireshark'
[[ "$(getent passwd tianer | cut -d: -f7)" == "/usr/sbin/nologin" ]]
```

### 6.2 udev Rules for Sniffer Hardware

USB sniffer devices are owned by `root:plugdev` with mode `0660` via udev rules. This pattern is the recommended approach by the Ubertooth project [^ubertooth-getting-started] and the standard Linux pattern for non-root hardware access [^udev-rules]. Members of the `plugdev` group can access the device without `sudo`.

**Ubertooth One:** USB vendor ID `1d50:6002` [^ubertooth-getting-started].

**Nordic nRF52840 Dongle (sniffer firmware):** USB vendor ID `1915:522A` (primary, sniffer mode) or `1915:520f` (secondary, DFU/bootloader mode). The dongle also presents as a USB CDC ACM serial device at `/dev/ttyACM*`; `dialout` group grants access to that node, while `plugdev` covers the raw USB device.

`deploy/udev/99-tianer.rules`:

```
# Ubertooth One - USB-only access via libusb. Group plugdev.
SUBSYSTEM=="usb", ATTRS{idVendor}=="1d50", ATTRS{idProduct}=="6002", \
    GROUP="plugdev", MODE="0660", \
    SYMLINK+="tianer/ubertooth%n"

# Nordic nRF52840 Dongle running nRF Sniffer firmware. Raw USB access via plugdev,
# serial node already group dialout by default.
# Primary PID 522A (sniffer mode); secondary PID 520f (DFU/bootloader mode).
# Both supported per D-03/Q3 resolution. Actual PID determined during detailed
# design phase with connected hardware.
SUBSYSTEM=="usb", ATTRS{idVendor}=="1915", ATTRS{idProduct}=="522a", \
    GROUP="plugdev", MODE="0660"
SUBSYSTEM=="tty", ATTRS{idVendor}=="1915", ATTRS{idProduct}=="522a", \
    SYMLINK+="tianer/nrf%n", \
    GROUP="dialout", MODE="0660"
SUBSYSTEM=="usb", ATTRS{idVendor}=="1915", ATTRS{idProduct}=="520f", \
    GROUP="plugdev", MODE="0660"
SUBSYSTEM=="tty", ATTRS{idVendor}=="1915", ATTRS{idProduct}=="520f", \
    SYMLINK+="tianer/nrf%n", \
    GROUP="dialout", MODE="0660"

# Nordic nRF52840 DK (development kit) - alternative ID set
SUBSYSTEM=="usb", ATTRS{idVendor}=="1366", \
    GROUP="plugdev", MODE="0660"
SUBSYSTEM=="tty", ATTRS{idVendor}=="1366", \
    SYMLINK+="tianer/nrf%n", \
    GROUP="dialout", MODE="0660"
```

**NOTE:** The nRF52840 USB PID (`1915:520f` in the rule above) is a placeholder. The actual sniffer firmware PID must be verified against connected hardware during T01. The project's `nrf52840-sniffer` skill documents PID `1915:522A` as the confirmed sniffer firmware PID. Per D-03/Q3 resolution: support both `522A` (primary, sniffer mode) and `520f` (secondary, DFU/bootloader mode). The exact value will be determined during the detailed hardware design phase.

**USB PID:** Dual-support 1915:522A (sniffer) + 1915:520f (DFU). Verification during T01. See ADR-0001.

Apply rules:
```bash
sudo install -m 0644 deploy/udev/99-tianer.rules /etc/udev/rules.d/99-tianer.rules
sudo udevadm control --reload-rules
sudo udevadm trigger
```

**ACCEPTANCE CRITERIA for 6.2:**
1. With an Ubertooth plugged in, `ls -l /dev/tianer/ubertooth0` shows the symlink and the target file is `root:plugdev` mode `0660`.
2. The `tianer` user can read the device: `sudo -u tianer -g plugdev ubertooth-util -v` succeeds.
3. Equivalent for nRF dongle.

### 6.3 Wireshark / dumpcap Capability Setup

When `tshark` reads from a named pipe (FIFO), it delegates capture to the `dumpcap` helper binary. `dumpcap` requires capabilities `CAP_NET_RAW` and `CAP_NET_ADMIN` to capture from interfaces, and Debian/Ubuntu Wireshark packages provide a `wireshark` group that limits invocation [^wireshark-blog].

The Debian-recommended setup uses `dpkg-reconfigure wireshark-common` to enable non-root capture; this both creates the `wireshark` group and applies the file capabilities and group ownership to `/usr/bin/dumpcap` [^wireshark-capture-privileges].

`deploy/scripts/setup-wireshark.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

# 1. Install Wireshark CLI tools (tshark + dumpcap). Avoid the GUI on a Pi.
apt-get install -y --no-install-recommends tshark

# 2. Enable non-root capture. This is the official Debian mechanism and is
#    idempotent: it answers Yes to the "should non-superusers be able to
#    capture packets?" debconf question.
echo "wireshark-common wireshark-common/install-setuid boolean true" \
    | debconf-set-selections
DEBIAN_FRONTEND=noninteractive dpkg-reconfigure wireshark-common

# 3. The above sets group ownership of /usr/bin/dumpcap to 'wireshark' and
#    applies cap_net_raw,cap_net_admin+eip via setcap. Verify.
getcap /usr/bin/dumpcap | grep -q 'cap_net_admin.*cap_net_raw\|cap_net_raw.*cap_net_admin'

# 4. tianer user is already in the wireshark group via create-user.sh.
```

`setcap` applies file capabilities to the binary [^setcap-man]. The capability set `cap_net_raw,cap_net_admin+eip` grants effective, inherited, and permitted bits, allowing dumpcap to capture without setuid root.

**Note on FIFO capture and capabilities:** Reading from a FIFO does not strictly require these capabilities at the kernel level since no network interface is touched. However, `dumpcap` checks for them at startup when invoked with `-i` regardless of source [^wireshark-capture-privileges]. Granting the capabilities is the lowest-friction path.

**ACCEPTANCE CRITERIA for 6.3:**
1. `getcap /usr/bin/dumpcap` shows `cap_net_admin,cap_net_raw+eip` (order may vary).
2. The `tianer` user can run a capture from a FIFO: `sudo -u tianer tshark -i /var/run/tianer/test.fifo -c 1` does not return EPERM.
3. The same command run as a user not in `wireshark` fails with a permission error.

### 6.4 File System Layout and Ownership

All paths used by the platform are pre-created with `tianer:tianer` ownership before any service starts. systemd-tmpfiles handles paths under `/var/run/`; a setup script handles persistent paths.

`/etc/tmpfiles.d/tianer.conf` (managed by systemd-tmpfiles at boot) [^tmpfiles-d]:

```
# Type Path                            Mode UID       GID       Age Argument
d      /var/run/tianer                 0750 tianer    tianer    -   -
d      /var/log/tianer                 0750 tianer    tianer    -   -
p      /var/run/tianer/ut1.fifo        0660 tianer    tianer    -   -
p      /var/run/tianer/nrf1.fifo       0660 tianer    tianer    -   -
p      /var/run/tianer/nrf2.fifo       0660 tianer    tianer    -   -
p      /var/run/tianer/nrf3.fifo       0660 tianer    tianer    -   -
```

`deploy/scripts/create-dirs.sh` (persistent paths, run once):

```bash
#!/usr/bin/env bash
set -euo pipefail

install -d -o tianer -g tianer -m 0750 /var/lib/tianer
install -d -o tianer -g tianer -m 0750 /var/lib/tianer/pcap
install -d -o tianer -g tianer -m 0700 /etc/tianer
install -d -o tianer -g tianer -m 0700 /etc/tianer/secrets
install -d -o tianer -g tianer -m 0750 /opt/tianer
install -d -o root     -g root     -m 0755 /usr/share/tianer
install -d -o root     -g root     -m 0755 /usr/share/tianer/frontend
install -d -o root     -g root     -m 0755 /usr/share/tianer/migrations
install -d -o root     -g root     -m 0755 /usr/local/lib/tianer/wrap

# Apply tmpfiles.d immediately for non-persistent paths so we don't need to reboot.
systemd-tmpfiles --create /etc/tmpfiles.d/tianer.conf
```

**ACCEPTANCE CRITERIA for 6.4:**
1. `stat -c '%U %G %a' /var/lib/tianer` returns `tianer tianer 750`.
2. `stat -c '%U %G %a' /etc/tianer/secrets` returns `tianer tianer 700`.
3. After reboot, FIFOs at `/var/run/tianer/*.fifo` exist with mode `0660` and owner `tianer:tianer`.

### 6.5 systemd Unit Privilege Drop

Every tianer systemd unit declares `User=tianer` and `Group=tianer`, so processes always run unprivileged [^systemd-exec-user]. Capabilities are not granted to units (only to specific binaries via setcap in section 6.3).

Example skeleton applied to every unit in `deploy/systemd/`:

```ini
[Service]
User=tianer
Group=tianer
SupplementaryGroups=plugdev dialout wireshark
EnvironmentFile=/etc/tianer/tianer.env
# Module-specific override also available:
# EnvironmentFile=/etc/tianer/blesniff.env

# Hardening (per systemd.exec documentation)
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true
ReadWritePaths=/var/lib/tianer /var/log/tianer /var/run/tianer
```

`SupplementaryGroups=` ensures the process has the necessary auxiliary groups even if the user database is read in a restricted execution environment [^systemd-exec-user]. `NoNewPrivileges=true` blocks setuid escalation [^systemd-exec-hardening]. `ReadWritePaths=` grants write access to the platform's data directories while `ProtectSystem=strict` makes the rest of the filesystem read-only to the service.

**ACCEPTANCE CRITERIA for 6.5:**
1. `systemctl show blesniff-api.service -p User -p Group` returns `User=tianer` `Group=tianer`.
2. `ps -o user,group,cmd -p $(pgrep blesniff-api)` shows `tianer tianer` (not `root`).
3. No `sudo` lines exist in any systemd unit's `ExecStart=`.

### 6.6 PostgreSQL Role Setup

PostgreSQL is installed by `apt install postgresql-17` [^debian-pg17]. The default `postgres` superuser remains for administrative tasks; the platform uses three application roles, none of which are superusers.

`db/sql/init-roles.sql` (run once by `deploy/scripts/install-postgres.sh` via `sudo -u postgres psql -f`):

```sql
-- Main service role: owns the schema, can DDL and DML.
CREATE ROLE tianer WITH LOGIN PASSWORD :'tianer_pw';
CREATE DATABASE tianer OWNER tianer;

-- Read-only role used by Grafana.
CREATE ROLE tianer_grafana WITH LOGIN PASSWORD :'grafana_pw';
GRANT CONNECT ON DATABASE tianer TO tianer_grafana;
\c tianer
GRANT USAGE ON SCHEMA bluetooth TO tianer_grafana;
ALTER DEFAULT PRIVILEGES IN SCHEMA bluetooth GRANT SELECT ON TABLES TO tianer_grafana;

-- Read-only role used by the API for queries that don't need write access.
-- The API uses tianer for writes (continuous aggregate refresh, residency
-- classification job) and tianer_ro for read-only endpoints.
CREATE ROLE tianer_ro WITH LOGIN PASSWORD :'ro_pw';
GRANT CONNECT ON DATABASE tianer TO tianer_ro;
GRANT USAGE ON SCHEMA bluetooth TO tianer_ro;
ALTER DEFAULT PRIVILEGES IN SCHEMA bluetooth GRANT SELECT ON TABLES TO tianer_ro;

-- Write-only role for ingest bridge and gap detector streams.
-- Can INSERT and COPY but cannot SELECT or DDL.
-- Per Q9 resolution: write-only for streams, read-only for UI.
CREATE ROLE tianer_writer WITH LOGIN PASSWORD :'writer_pw';
GRANT CONNECT ON DATABASE tianer TO tianer_writer;
GRANT USAGE ON SCHEMA bluetooth TO tianer_writer;
GRANT INSERT ON ALL TABLES IN SCHEMA bluetooth TO tianer_writer;
ALTER DEFAULT PRIVILEGES IN SCHEMA bluetooth GRANT INSERT ON TABLES TO tianer_writer;
```

`pg_hba.conf` is configured for local Unix-socket connections only (peer auth for `postgres`, md5/scram for the three app roles), with the host loopback also allowed for the API and Grafana service connections:

```
# Local administration via Unix socket
local   all             postgres                                peer
local   tianer          tianer                                  scram-sha-256
local   tianer          tianer_grafana                          scram-sha-256
local   tianer          tianer_ro                               scram-sha-256

# Loopback TCP (Grafana, API, writer)
host    tianer          tianer_grafana    127.0.0.1/32          scram-sha-256
host    tianer          tianer_ro         127.0.0.1/32          scram-sha-256
host    tianer          tianer            127.0.0.1/32          scram-sha-256
host    tianer          tianer_writer     127.0.0.1/32          scram-sha-256

# Reject everything else
host    all             all                 0.0.0.0/0           reject
host    all             all                 ::/0                reject
```

**ACCEPTANCE CRITERIA for 6.6:**
1. `sudo -u tianer psql -h 127.0.0.1 -U tianer -d tianer -c 'SELECT 1'` succeeds without prompting for system password.
2. `sudo -u tianer psql -h 127.0.0.1 -U tianer_ro -d tianer -c 'INSERT INTO sniffers VALUES (1)'` fails with a permission error.
3. From a different host on the LAN, `psql -h <pi> ...` is rejected.

### 6.7 sudo Policy

The runtime contract: no service requires `sudo`.

The only legitimate `sudo` invocations are at install time (running `deploy/setup.sh` as root) and during manual operator interventions (`systemctl restart`, log inspection). The platform does not install any sudoers entries. Specifically:

* No `NOPASSWD` rules.
* No service may call `sudo` in `ExecStart=`.
* Operator-level commands like `make services-restart` use `systemctl` directly; on a Pi these typically require a polkit rule for password-less `systemctl` for the operator user [^polkit-systemd], but that is operator preference, not a platform requirement.

**ACCEPTANCE CRITERIA for 6.7:**
1. `grep -r sudo deploy/systemd/` returns nothing.
2. `ls /etc/sudoers.d/tianer*` returns no files.

### 6.8 Configuration Files

All runtime configuration lives in `/etc/tianer/tianer.env` (shared, sourced by every systemd unit) and `/etc/tianer/sniffers.yaml` (per-sniffer settings). Both files are mode `0640`, owned by `root:tianer` so that the service can read them but only an administrator can edit them.

#### 6.8.1 `blesniff.env` (key-value, shell-sourced)

The Bluetooth module uses `/etc/tianer/blesniff.env` for module-specific settings:

```bash
# Database
BLESNIFF_DB_HOST=127.0.0.1
BLESNIFF_DB_PORT=5432
BLESNIFF_DB_NAME=tianer
BLESNIFF_DB_USER=tianer
BLESNIFF_DB_PASSWORD_FILE=/etc/tianer/secrets/db_password

# Paths
BLESNIFF_PCAP_DIR=/var/lib/tianer/pcap
BLESNIFF_FIFO_DIR=/var/run/tianer
BLESNIFF_LOG_DIR=/var/log/tianer

# Rotation
BLESNIFF_ROTATION_MINUTES=30
BLESNIFF_PCAP_RETENTION_DAYS=14
BLESNIFF_COMPRESSION_DELAY_MINUTES=5

# Ingest
BLESNIFF_BATCH_MAX_ROWS=500
BLESNIFF_BATCH_MAX_LATENCY_MS=1000
BLESNIFF_INGEST_LOG_LEVEL=info

# Gap detection
BLESNIFF_GAP_BUCKET_SECONDS=30
BLESNIFF_GAP_SCAN_INTERVAL_MINUTES=5
BLESNIFF_GAP_LOOKBACK_HOURS=1

# API
BLESNIFF_API_HOST=127.0.0.1
BLESNIFF_API_PORT=8080
BLESNIFF_API_KEY_FILE=/etc/tianer/secrets/api_key

# Frontend
BLESNIFF_FRONTEND_DIR=/usr/share/tianer/frontend
BLESNIFF_GRAFANA_URL=http://127.0.0.1:3000
```

#### 6.8.2 `sniffers.yaml`

```yaml
sniffers:
  - id: 1
    name: ut1
    type: ubertooth
    device: /dev/tianer/ubertooth0
    mode: btle           # btle | rx (Classic survey)
    channels: [37, 38, 39]
    enabled: true
  - id: 2
    name: nrf1
    type: nrf
    device: /dev/tianer/nrf0
    mode: ble
    enabled: true
  - id: 3
    name: nrf2
    type: nrf
    device: /dev/tianer/nrf1
    enabled: false
  - id: 4
    name: nrf3
    type: nrf
    device: /dev/tianer/nrf2
    enabled: false
```

#### 6.8.3 Secrets

* `/etc/tianer/secrets/db_password`, `/etc/tianer/secrets/api_key`, `/etc/tianer/secrets/grafana_db_password`.
* Mode `0600`, owner `tianer:tianer`.
* Generated by `deploy/scripts/generate-secrets.sh` on first run; idempotent (does not overwrite).
* Never committed to the repository.

#### 6.8.4 Migrations and Schema Versioning

* SQL migration files are numbered and applied in lexicographic order by `db/apply-migrations.sh`.
* The script tracks applied migrations in a `_migrations` table.
* All schema changes go through a new migration file; existing files are immutable once merged.

### 6.9 End-of-Section Verification

After 6.1 through 6.8 are complete, the following must hold. This block is run by `tests/integration/test_prereqs.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

# User and groups
id tianer >/dev/null || { echo FAIL: user; exit 1; }
for grp in plugdev dialout wireshark; do
    id -nG tianer | grep -qw "$grp" || { echo "FAIL: not in $grp"; exit 1; }
done

# Capabilities
getcap /usr/bin/dumpcap | grep -q 'cap_net_raw.*cap_net_admin\|cap_net_admin.*cap_net_raw' || \
    { echo FAIL: dumpcap caps; exit 1; }

# Paths
[[ "$(stat -c '%U:%G:%a' /var/lib/tianer)" == "tianer:tianer:750" ]] || { echo FAIL: /var/lib/tianer; exit 1; }
[[ "$(stat -c '%U:%G:%a' /etc/tianer/secrets)" == "tianer:tianer:700" ]] || { echo FAIL: /etc/tianer/secrets; exit 1; }

# udev rule installed
[[ -f /etc/udev/rules.d/99-tianer.rules ]] || { echo FAIL: udev rule; exit 1; }

# Database
sudo -u tianer psql -h 127.0.0.1 -U tianer -d tianer -c 'SELECT 1' >/dev/null || { echo FAIL: db connect; exit 1; }

# sudo policy
[[ ! -f /etc/sudoers.d/tianer ]] || { echo FAIL: sudoers entry exists; exit 1; }

# Smoke: tianer can read sniffer device (assuming one is plugged in; skip otherwise)
if [[ -e /dev/tianer/ubertooth0 ]]; then
    sudo -u tianer ubertooth-util -v >/dev/null || { echo FAIL: ubertooth access; exit 1; }
fi

echo "PREREQUISITES OK"
```

**ACCEPTANCE CRITERIA for 6.9:**
1. The script exits 0.



---

## 7. High-Level Architecture

```
+----------------+      +----------------+      +----------------+
|  Sniffer #1    |      |  Sniffer #2    |      |  Sniffer #N    |
|  (Ubertooth)   |      |  (nRF)         |      |  (...)         |
+-------+--------+      +-------+--------+      +-------+--------+
        | PCAP stdout           | PCAP stdout           | PCAP stdout
        v                       v                       v
   +--------+              +--------+              +--------+
   |splitter|              |splitter|              |splitter|
   |(tee)   |              |(tee)   |              |(tee)   |
   +-+----+-+              +-+----+-+              +-+----+-+
     |    |                  |    |                  |    |
     v    v                  v    v                  v    v
  +----+ +----+           +----+ +----+           +----+ +----+
  |PCAP| |FIFO|           |PCAP| |FIFO|           |PCAP| |FIFO|
  |file| |    |           |file| |    |           |file| |    |
  +----+ +-+--+           +----+ +-+--+           +----+ +-+--+
   |       |               |       |               |       |
   |       v               |       v               |       v
   |   +-------+           |   +-------+           |   +-------+
   |   |tshark |           |   |tshark |           |   |tshark |
   |   | -l    |           |   | -l    |           |   | -l    |
   |   +---+---+           |   +---+---+           |   +---+---+
   |       | stdout            |       |                   |
   |       v               |       v               |       v
   |   +-------+           |   +-------+           |   +-------+
   |   |ingest |           |   |ingest |           |   |ingest |
   |   |bridge |           |   |bridge |           |   |bridge |
   |   +---+---+           |   +---+---+           |   +---+---+
   |       |               |       |               |       |
   |       +---------------+-------+---------------+-------+
   |                       |
   |                       v
   |          +-----------------------------+
   |          |       TimescaleDB           |
   |          |  raw_packets hypertable     |
   |          |  device_summary table       |
   |          +--+--------------+-----------+
   |             ^              |
   |             |              v
   |        +----+------+   +--------+   +----------+
   |        |    gap    |   |Grafana |   | FastAPI  |
   +------> | detector  |   |panels  |   | + Vue.js |
            +-----------+   +--------+   +----------+
                  ^
                  |
            +-----+------+
            | deep parser|  (batch, runs on rotated files)
            | (C++)      |
            +-----+------+
                  |
                  v
            +-------------+
            | ML enrich   |
            | (Python)    |
            +------+------+
                   |
                   v
            device_enrichment table
```

---

## 8. Component Specifications

**Deployment Model (Q1/Q11 resolution):** All components run in Podman containers managed by Quadlet. USB devices are passed through from the host. Persistent storage uses the volume strategy documented in `doc/designs/storage-strategy.md`. The systemd-based descriptions below are retained for per-component logic; the containerized deployment model supersedes the bare-metal service descriptions for operational concerns.

Each component section follows the same structure:

* **Purpose**
* **Inputs / Outputs**
* **Configuration**
* **Health check**
* **Failure modes**
* **Test plan**

### 8.1 Sniffer Wrapper

**Purpose:** Run a sniffer binary, dual-write its PCAP output to a rotating file and a FIFO.

**Inputs:**
* `sniffers.yaml` (read by systemd unit template)
* `/etc/tianer/blesniff.env`

**Outputs:**
* PCAP stream into `${BLESNIFF_PCAP_DIR}/<sniffer_name>/current.pcap`
* PCAP stream into `${BLESNIFF_FIFO_DIR}/<sniffer_name>.fifo`

**Implementation:**

`modules/bluetooth/sniffers/ubertooth-wrap.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
source /etc/tianer/tianer.env
source /etc/tianer/blesniff.env

SNIFFER_NAME="${1:?sniffer name required}"
DEVICE="${2:?device path required}"
MODE="${3:-btle}"

mkdir -p "${BLESNIFF_PCAP_DIR}/${SNIFFER_NAME}"
FIFO="${BLESNIFF_FIFO_DIR}/${SNIFFER_NAME}.fifo"
PCAP="${BLESNIFF_PCAP_DIR}/${SNIFFER_NAME}/current.pcap"

[[ -p "${FIFO}" ]] || mkfifo -m 0660 "${FIFO}"

case "${MODE}" in
  btle) exec ubertooth-btle -A -c - | tee "${PCAP}" > "${FIFO}" ;;
  rx)   exec ubertooth-rx -q - | tee "${PCAP}" > "${FIFO}" ;;
  *) echo "unknown mode: ${MODE}" >&2; exit 2 ;;
esac
```

Equivalent `nrf-wrap.sh` uses `nrfutil ble-sniffer sniff --port "${DEVICE}" --output-pcap-file /dev/stdout`.

**DECISION REQUIRED 8.1.1:** Verify that `ubertooth-btle -c -` and `nrfutil ble-sniffer ... --output-pcap-file /dev/stdout` actually write valid PCAP to stdout. `RECOMMENDED DEFAULT: Validate during T03/T04. If stdout output is not supported, fall back to writing to a named pipe and reading from it.`

**Health check:**

The wrapper writes a heartbeat row to the database every 30 seconds:

```sql
INSERT INTO sniffer_heartbeat (sniffer_id, ts, status)
VALUES ($1, NOW(), 'running')
ON CONFLICT (sniffer_id) DO UPDATE SET ts = NOW(), status = 'running';
```

A separate `heartbeat.sh` companion process runs alongside the sniffer wrapper for this.

**Failure modes:**

| Failure | Behavior |
|---------|----------|
| Sniffer binary exits | systemd restarts after 5s (Restart=on-failure, RestartSec=5s) |
| FIFO reader (tshark) dies | tee writes block; pipeline stalls but no packet loss to file. systemd dependency restarts tshark, pipeline resumes. |
| Disk full | tee fails; sniffer process exits; alert via Grafana panel watching disk usage. |

**Test plan:**

| Test | Type | File |
|------|------|------|
| Wrapper creates FIFO if missing | Unit | `modules/bluetooth/sniffers/tests/test_fifo_create.bats` |
| Wrapper writes to both file and FIFO | Integration | `tests/integration/test_dual_write.sh` |
| Wrapper handles invalid mode argument | Unit | `modules/bluetooth/sniffers/tests/test_args.bats` |
| Heartbeat row updated every 30s | Integration | `tests/integration/test_heartbeat.sh` |

**ACCEPTANCE CRITERIA:**
1. Starting the wrapper produces a valid PCAP file readable by `tshark -r`.
2. Concurrently, the same packets appear in the FIFO when read.
3. Killing the FIFO reader does not drop bytes from the file.
4. `sniffer_heartbeat` row is updated within 60 seconds of start.

**VERIFICATION:**
```bash
# Start in test mode
./modules/bluetooth/sniffers/ubertooth-wrap.sh ut1-test /dev/tianer/ubertooth0 btle &
sleep 10
# Verify file is valid PCAP
tshark -r "${BLESNIFF_PCAP_DIR}/ut1-test/current.pcap" -c 1
# Verify FIFO has data
timeout 5 tshark -r "${BLESNIFF_FIFO_DIR}/ut1-test.fifo" -c 1
# Verify heartbeat
psql -t -c "SELECT status FROM sniffer_heartbeat WHERE sniffer_id = 1 AND ts > NOW() - INTERVAL '1 minute';" | grep running
```

**Skill-set:** Linux shell scripting, POSIX named pipes, systemd, Ubertooth and nRF tooling.

---

### 8.2 PCAP File Rotation

**Purpose:** Rotate the per-sniffer `current.pcap` every 30 minutes to a timestamped file, compress old files, enforce retention.

**Implementation:**

* `deploy/scripts/rotate-pcap.sh` is triggered by `blesniff-rotate.timer` every 30 minutes on minute boundaries (0 and 30 past the hour).
* For each enabled sniffer:
  1. Send `SIGHUP` to the sniffer wrapper (which closes the current PCAP and reopens via a small wrapper that re-execs the tee target).
  2. Rename `current.pcap` to `YYYYMMDD-HHMM.pcap` where HHMM is the rotation boundary just passed.
  3. Schedule compression after `BLESNIFF_COMPRESSION_DELAY_MINUTES`.
  4. Delete files older than `BLESNIFF_PCAP_RETENTION_DAYS`.

**DECISION REQUIRED 8.2.1:** Rotation mechanism. `RECOMMENDED DEFAULT: Rotation is "clean" via wrapper restart (DECISION 5.2.3 option 1 in v0.1). Expected gap per rotation: under 500 ms. The gap detector ignores buckets that overlap a rotation boundary.`

**DECISION REQUIRED 8.2.2:** Rotation trigger. `RECOMMENDED DEFAULT: Pure time-based via systemd timer. Filenames track the boundary, not the actual rotation moment.`

**systemd timer:**

`deploy/systemd/blesniff-rotate.timer`:
```ini
[Unit]
Description=Rotate BLE sniffer PCAP files every 30 minutes

[Timer]
OnCalendar=*-*-* *:00/30:00
AccuracySec=1s
Persistent=true

[Install]
WantedBy=timers.target
```

**Failure modes:**

| Failure | Behavior |
|---------|----------|
| Rotation fails for one sniffer | Other sniffers continue. Error logged to journald. |
| Disk full at rotation | Old uncompressed files are compressed early; if still full, oldest files are deleted ahead of retention schedule. Configurable via `BLESNIFF_EMERGENCY_PURGE=true`. |
| Compression fails | Original .pcap retained; retry on next rotation. |

**Test plan:**

| Test | Type | File |
|------|------|------|
| Rotation produces correctly named files | Integration | `tests/integration/test_rotation_naming.sh` |
| Compression runs after delay | Integration | `tests/integration/test_compression.sh` |
| Retention deletes old files | Integration | `tests/integration/test_retention.sh` |
| Rotation gap is under 1 second | Integration | `tests/integration/test_rotation_gap.sh` |

**ACCEPTANCE CRITERIA:**
1. At each 30-minute boundary, a new PCAP file appears with name `YYYYMMDD-HHMM.pcap`.
2. The previous file is closed and remains valid.
3. Files older than the compression delay are compressed to `.pcap.zst` and the original is removed.
4. Files older than retention are deleted.

**VERIFICATION:**
```bash
# Simulate accelerated rotation by setting BLESNIFF_ROTATION_MINUTES=1 in a test env
systemctl --user start blesniff-rotate.timer
sleep 130
ls -la "${BLESNIFF_PCAP_DIR}/ut1-test/" | grep -E '[0-9]{8}-[0-9]{4}\.pcap'
```

**Skill-set:** Bash scripting, systemd timers, zstd, file system management.

---

### 8.3 Named Pipes (FIFOs)

**Purpose:** Lossless in-memory IPC between sniffer wrapper and tshark.

**Implementation:**

`/etc/tmpfiles.d/tianer.conf`:
```
d /var/run/tianer 0750 tianer tianer -
p /var/run/tianer/ut1.fifo 0660 tianer tianer -
p /var/run/tianer/nrf1.fifo 0660 tianer tianer -
p /var/run/tianer/nrf2.fifo 0660 tianer tianer -
p /var/run/tianer/nrf3.fifo 0660 tianer tianer -
```

`systemd-tmpfiles --create /etc/tmpfiles.d/tianer.conf` runs at boot.

**Dependency ordering:** tshark unit specifies `Before=blesniff-sniffer@.service` to ensure the reader is up before writes begin. Without this, sniffer writes to a readerless pipe drop bytes.

**ACCEPTANCE CRITERIA:**
1. FIFOs exist at boot with correct permissions.
2. tshark is started before its corresponding sniffer wrapper.

**VERIFICATION:**
```bash
systemctl reboot
# After reboot:
ls -la /var/run/tianer/*.fifo  # All present
systemctl status blesniff-tshark@ut1.service blesniff-sniffer@ut1.service
# tshark "Active since" timestamp must precede sniffer "Active since"
```

**Skill-set:** Linux IPC, systemd-tmpfiles, systemd ordering.

---

### 8.4 tshark Real-Time Parser

**Purpose:** Read PCAP stream from FIFO, emit structured fields per packet to stdout.

**Implementation:**

`modules/bluetooth/sniffers/tshark-wrap.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
source /etc/tianer/blesniff.env

SNIFFER_NAME="${1:?}"
FIFO="${BLESNIFF_FIFO_DIR}/${SNIFFER_NAME}.fifo"

# CONTRACT 8.4-A: tshark output format
# Each line: epoch_ts|adv_address|adv_address_type|rssi|channel|pdu_type|payload_hex
# Empty fields are represented as empty strings between separators.
exec tshark -i "${FIFO}" -l -n \
  -T fields -E separator=\| -E header=n -E quote=n -E occurrence=f \
  -e frame.time_epoch \
  -e btle.advertising_address \
  -e btle.advertising_header.randomized_tx \
  -e nordic_ble.rssi \
  -e nordic_ble.channel \
  -e btle.advertising_header.pdu_type \
  -e btle.advertising_data
```

**tshark fields:** Per-DLT parameterized config. Field names verified during T07. See ADR-0001.

**DECISION REQUIRED 8.4.2:** Parse-failure behavior. `RECOMMENDED DEFAULT: tshark emits a blank line on parse failure. Ingest bridge increments a "malformed_packets" counter and skips. After 100 consecutive malformed packets, the bridge logs ERROR and triggers a Grafana alert.`

**CONTRACT 8.4-A (tshark to ingest bridge output format):**

```
1717689600.123456|aa:bb:cc:dd:ee:ff|1|-67|37|0|02011a0303aafe17...
```

Fields:
| Position | Field | Type | Empty allowed |
|----------|-------|------|---------------|
| 1 | epoch timestamp (seconds, fractional) | float string | no |
| 2 | advertising address | colon-hex string | yes (non-adv packets) |
| 3 | address type (0=public, 1=random) | "0" or "1" | yes |
| 4 | RSSI (dBm) | int string | yes |
| 5 | channel | int string | yes |
| 6 | PDU type | int string | yes |
| 7 | advertising data (raw) | hex string | yes |

**Test plan:**

| Test | Type | File |
|------|------|------|
| Output format matches CONTRACT 8.4-A on sample PCAP | Integration | `tests/integration/test_tshark_format.sh` |
| Line buffering produces output without delay | Integration | `tests/integration/test_tshark_buffering.sh` |
| Malformed input handled | Unit | `modules/bluetooth/sniffers/tests/test_tshark_malformed.bats` |

**ACCEPTANCE CRITERIA:**
1. Feeding `tests/fixtures/pcap/ubertooth-sample-001.pcap` via FIFO yields stdout output matching CONTRACT 8.4-A within 1 second.
2. Empty fields are correctly represented as empty between pipes.
3. Process exits cleanly on SIGTERM.

**VERIFICATION:**
```bash
# Replay fixture into FIFO
mkfifo /tmp/test.fifo
./tools/pcap-replay.py tests/fixtures/pcap/ubertooth-sample-001.pcap /tmp/test.fifo &
# Run tshark wrapper, compare to golden
./modules/bluetooth/sniffers/tshark-wrap.sh test 2>/dev/null | head -10 > /tmp/actual.txt
diff /tmp/actual.txt tests/fixtures/expected/tshark-ubertooth-sample-001.txt
```

**Skill-set:** tshark, BLE link-layer fields, Linux IPC.

---

### 8.5 Ingest Bridge (C++)

**Purpose:** Read tshark stdout, parse CONTRACT 8.4-A, batch, write to PostgreSQL via `COPY`.

**DECISION REQUIRED 8.5.1:** Language. `RECOMMENDED DEFAULT: C++17 with libpqxx 7.8. Justification: aligns with deep-parser language, avoids Python in the hot path, no runtime dependencies beyond libstdc++.`

**Module layout:**

```
modules/bluetooth/ingest-bridge/src/
├── main.cpp           # Entry point, argument parsing, signal handling
├── config.{cpp,hpp}   # Reads BLESNIFF_* env vars
├── parser.{cpp,hpp}   # Tokenize CONTRACT 8.4-A lines into Packet structs
├── batcher.{cpp,hpp}  # Time + size batching
└── pg_writer.{cpp,hpp}# libpqxx connection + COPY
```

**Core types (`parser.hpp`):**

```cpp
struct Packet {
    std::chrono::system_clock::time_point ts;
    std::optional<std::array<uint8_t, 6>> mac;
    std::optional<bool> address_random;   // true = random, false = public
    std::optional<int16_t> rssi;
    std::optional<int16_t> channel;
    std::optional<int16_t> pdu_type;
    std::vector<uint8_t> advdata;
};

// CONTRACT 8.5-A: Parse a single tshark-format line.
// Returns std::nullopt on malformed input.
std::optional<Packet> parse_line(std::string_view line);
```

**Core types (`batcher.hpp`):**

```cpp
class Batcher {
public:
    Batcher(size_t max_rows, std::chrono::milliseconds max_latency);
    // Returns true if the batch should be flushed now.
    bool add(Packet p);
    std::vector<Packet> drain();
};
```

**Core types (`pg_writer.hpp`):**

```cpp
class PgWriter {
public:
    explicit PgWriter(const Config& cfg, int16_t sniffer_id);
    // Throws on connection failure. Caller decides whether to retry.
    void write(std::span<const Packet> batch);
private:
    pqxx::connection conn_;
    int16_t sniffer_id_;
};
```

`PgWriter::write` builds an `pqxx::stream_to` against `raw_packets` and streams rows.

**DECISION REQUIRED 8.5.2:** One bridge per sniffer or shared. `RECOMMENDED DEFAULT: One process per sniffer. Simpler systemd model, parallel ingest, ~4 PG connections total which is trivial.`

**DECISION REQUIRED 8.5.3:** Cross-sniffer duplicate handling. `RECOMMENDED DEFAULT: Store every observation per sniffer. raw_packets.sniffer_id distinguishes them. No deduplication at ingest. Future triangulation needs all observers.`

**Failure modes:**

| Failure | Behavior |
|---------|----------|
| stdin closes | Drain final batch, exit 0. |
| Postgres connection drops | Exponential backoff reconnect (1s, 2s, 4s, max 30s). Packets accumulate in batch buffer up to 10000 rows; older drops with a counter increment. |
| Malformed line | Log at debug level, increment `malformed_packets_total`. |
| Out of memory | Process killed; systemd restarts; in-flight packets are recoverable from PCAP via gap detector. |

**Metrics exposed via SIGUSR1:**

On `SIGUSR1`, the process writes a one-line metric snapshot to stderr:
```
ingest_metrics sniffer_id=1 packets_in=12345 batches_flushed=67 malformed=2 last_flush_ms=312 batch_size_avg=185
```

**Test plan:**

| Test | Type | File |
|------|------|------|
| `parse_line` happy path | Unit (GoogleTest) | `tests/parser_test.cpp` |
| `parse_line` malformed inputs (15 cases) | Unit | `tests/parser_test.cpp` |
| `Batcher` flushes on row count | Unit | `tests/batcher_test.cpp` |
| `Batcher` flushes on latency | Unit | `tests/batcher_test.cpp` |
| `PgWriter` writes rows correctly | Integration (against test DB) | `tests/pg_writer_test.cpp` |
| End-to-end: fixture line → DB row | Integration | `tests/integration/test_ingest_e2e.sh` |
| Reconnect on PG restart | Integration | `tests/integration/test_pg_reconnect.sh` |

**ACCEPTANCE CRITERIA:**
1. Given a stream of N valid CONTRACT 8.4-A lines, exactly N rows appear in `raw_packets` with matching values.
2. Sustained throughput of at least 1000 rows/sec on the target Pi without OOM.
3. P95 ingest latency (line received to row visible) under 1500 ms.
4. After `kill -STOP $(pgrep postgres) && kill -CONT $(pgrep postgres)` simulating a brief PG outage, no packets are lost from a 60-second test stream.

**VERIFICATION:**
```bash
# Build
cd modules/bluetooth/ingest-bridge && cmake -B build && cmake --build build
# Run unit tests
ctest --test-dir build --output-on-failure
# Integration
./tests/integration/test_ingest_e2e.sh
# Throughput
./tools/pcap-replay.py tests/fixtures/pcap/large-sample.pcap.zst /tmp/test.fifo --rate=1500 &
./modules/bluetooth/ingest-bridge/build/blesniff-ingest --sniffer-id 99 --input /tmp/test.fifo &
sleep 60
psql -t -c "SELECT COUNT(*) FROM raw_packets WHERE sniffer_id = 99 AND ts > NOW() - INTERVAL '1 minute';"
# Expected: ~90000 rows (1500/s * 60s)
```

**Skill-set:** C++17, libpqxx, PostgreSQL `COPY`, signal handling, CMake.

---

### 8.6 Gap Detector (Python)

**Purpose:** Detect gaps in `raw_packets`, backfill from on-disk PCAP files.

**Module layout:**

```
modules/bluetooth/gap-detector/src/blesniff_gapdet/
├── __init__.py
├── detector.py        # Find gaps
├── backfill.py        # Reprocess PCAP files
├── pcap_reader.py     # pyshark wrapper, yields normalized rows
├── db.py              # asyncpg / psycopg
└── cli.py             # argparse, --once / --daemon
```

**CONTRACT 8.6-A (gap definition):**

A gap exists when, for a `(sniffer_id, time_bucket)` pair within the lookback window:
* The sniffer was marked `running` for the entire bucket (per `sniffer_heartbeat` with HOLE of >60s = not running).
* `COUNT(*) FROM raw_packets WHERE sniffer_id = $1 AND ts >= bucket_start AND ts < bucket_end` is zero.
* The bucket is fully closed (its `bucket_end <= NOW() - 1 minute`, to avoid racing live data).

**Detection logic (`detector.py`):**

```python
async def find_gaps(
    conn: AsyncConnection,
    bucket_seconds: int,
    lookback_hours: int,
) -> list[Gap]:
    """
    Return list of (sniffer_id, bucket_start, bucket_end) where data is missing
    but the sniffer was supposed to be running. See CONTRACT 8.6-A.
    """
```

**Backfill logic (`backfill.py`):**

```python
async def backfill_gap(
    conn: AsyncConnection,
    gap: Gap,
    pcap_dir: Path,
) -> BackfillResult:
    """
    Locate the PCAP file(s) covering the gap window using filename convention
    YYYYMMDD-HHMM.pcap or .pcap.zst, reprocess with tshark, insert with
    backfilled=true and ON CONFLICT DO NOTHING.
    """
```

**Heartbeat:** Local file primary, DB table secondary per D-11. See ADR-0001.

**`ingest_gaps` table (to track backfill state):**

```sql
CREATE TABLE ingest_gaps (
    id              BIGSERIAL PRIMARY KEY,
    sniffer_id      SMALLINT NOT NULL,
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

**Failure modes:**

| Failure | Behavior |
|---------|----------|
| PCAP file missing for the gap | Mark gap `failed`, log; do not retry. |
| PCAP file corrupted | Mark gap `failed`; preserve `last_error`. |
| Backfill INSERT fails | Mark gap `failed`; retry next cycle up to 3 times. |
| Heartbeat data missing | Treat sniffer as not running; do not create gap. |

**Test plan:**

| Test | Type | File |
|------|------|------|
| Detect zero-packet bucket with active heartbeat | Unit | `tests/test_detector.py` |
| Ignore bucket without heartbeat | Unit | `tests/test_detector.py` |
| Skip live buckets within last minute | Unit | `tests/test_detector.py` |
| Backfill inserts rows with backfilled=true | Integration | `tests/integration/test_backfill.sh` |
| Idempotent: re-run does not duplicate | Integration | `tests/integration/test_backfill_idempotent.sh` |
| Picks up gzip and zstd compressed files | Unit | `tests/test_pcap_reader.py` |

**ACCEPTANCE CRITERIA:**
1. Stop ingest bridge for 2 minutes during a stream; gap detector identifies the gap within 5 minutes after.
2. After backfill completes, `COUNT(*)` for the gap window matches the count obtained by directly parsing the PCAP file.
3. Re-running the detector twice does not double-insert.
4. Missing PCAP file leaves gap status `failed` with a useful error message.

**VERIFICATION:**
```bash
# Use the smoke test
./tests/integration/test_gap_detection.sh
```

**Skill-set:** Python 3.13, asyncpg, pyshark, libpcap concepts, SQL, async patterns.

---

### 8.7 Database Schema (TimescaleDB)

**Migrations live in `db/migrations/`.** Each migration is idempotent (`IF NOT EXISTS` everywhere) and tracked in `_migrations`.

**`0001_init.sql`:**

```sql
CREATE EXTENSION IF NOT EXISTS timescaledb;

CREATE SCHEMA IF NOT EXISTS bluetooth;
SET search_path TO bluetooth;

CREATE TABLE IF NOT EXISTS _migrations (
    name TEXT PRIMARY KEY,
    applied_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS sniffers (
    sniffer_id   SMALLINT PRIMARY KEY,
    name         TEXT NOT NULL UNIQUE,
    type         TEXT NOT NULL,
    device_path  TEXT,
    enabled      BOOLEAN NOT NULL DEFAULT true,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS sniffer_heartbeat (
    sniffer_id SMALLINT PRIMARY KEY REFERENCES sniffers(sniffer_id),
    ts         TIMESTAMPTZ NOT NULL,
    status     TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS raw_packets (
    ts            TIMESTAMPTZ NOT NULL,
    sniffer_id    SMALLINT    NOT NULL,
    mac_address   BYTEA       NOT NULL,           -- 6 bytes BD_ADDR
    address_type  SMALLINT,                       -- 0=public, 1=random
    rssi          SMALLINT,
    channel       SMALLINT,
    pdu_type      SMALLINT,
    advdata       BYTEA,
    backfilled    BOOLEAN     NOT NULL DEFAULT FALSE
);

SELECT create_hypertable('raw_packets', 'ts',
    chunk_time_interval => INTERVAL '1 hour',
    if_not_exists => TRUE);

CREATE INDEX IF NOT EXISTS idx_raw_packets_mac_ts
    ON raw_packets (mac_address, ts DESC);
CREATE INDEX IF NOT EXISTS idx_raw_packets_sniffer_ts
    ON raw_packets (sniffer_id, ts DESC);
-- Dedup happens at query time via DISTINCT ON per D-10 resolution. No insert-time unique index.
-- Performance index for time-range queries per sniffer (TimescaleDB hypertable auto-partitions by time).
CREATE INDEX IF NOT EXISTS idx_raw_packets_lookup
    ON raw_packets (sniffer_id, ts DESC);

CREATE TABLE IF NOT EXISTS device_summary (
    mac_address      BYTEA       PRIMARY KEY,
    address_type     SMALLINT,
    first_seen       TIMESTAMPTZ NOT NULL,
    last_seen        TIMESTAMPTZ NOT NULL,
    total_count      BIGINT      NOT NULL DEFAULT 0,
    distinct_days    INT         NOT NULL DEFAULT 0,
    residency_class  TEXT,
    last_classified  TIMESTAMPTZ,
    enrichment_data  JSONB
);

CREATE INDEX IF NOT EXISTS idx_device_summary_last_seen
    ON device_summary (last_seen DESC);

CREATE TABLE IF NOT EXISTS ingest_gaps (
    id              BIGSERIAL PRIMARY KEY,
    sniffer_id      SMALLINT NOT NULL,
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

INSERT INTO _migrations(name) VALUES ('0001_init') ON CONFLICT DO NOTHING;
```

**Schema Isolation (Q5):** Tables for the v1 Bluetooth module live in a `bluetooth` schema under the `tianer` database. Future sensor modules (GPS, ADS-B, etc.) will use their own schemas. The `CREATE SCHEMA IF NOT EXISTS bluetooth` statement precedes all v1 table creation.

**`0002_continuous_aggregates.sql`:**

```sql
CREATE MATERIALIZED VIEW IF NOT EXISTS device_5min_buckets
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

SELECT add_continuous_aggregate_policy('device_5min_buckets',
    start_offset => INTERVAL '1 hour',
    end_offset   => INTERVAL '5 minutes',
    schedule_interval => INTERVAL '1 minute',
    if_not_exists => true);

INSERT INTO _migrations(name) VALUES ('0002_continuous_aggregates') ON CONFLICT DO NOTHING;
```

**`0003_compression_policies.sql`:**

```sql
ALTER TABLE raw_packets SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'mac_address',
    timescaledb.compress_orderby = 'ts DESC'
);

SELECT add_compression_policy('raw_packets',
    compress_after => INTERVAL '7 days',
    if_not_exists => true);

SELECT add_retention_policy('raw_packets',
    drop_after => INTERVAL '90 days',
    if_not_exists => true);

INSERT INTO _migrations(name) VALUES ('0003_compression_policies') ON CONFLICT DO NOTHING;
```

**`0004_residency_classifier.sql`:**

```sql
CREATE OR REPLACE FUNCTION classify_residency(p_mac BYTEA, p_now TIMESTAMPTZ DEFAULT NOW())
RETURNS TEXT LANGUAGE plpgsql AS $$
DECLARE
    v_first      TIMESTAMPTZ;
    v_last       TIMESTAMPTZ;
    v_days_7     INT;
    v_packets_7  BIGINT;
BEGIN
    SELECT first_seen, last_seen INTO v_first, v_last FROM device_summary WHERE mac_address = p_mac;
    IF v_first IS NULL THEN RETURN 'unknown'; END IF;

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

    IF v_days_7 >= 5 AND v_packets_7 >= 700 THEN RETURN 'resident'; END IF;
    IF v_days_7 >= 3 THEN RETURN 'frequent'; END IF;

    RETURN 'transient';
END $$;

INSERT INTO _migrations(name) VALUES ('0004_residency_classifier') ON CONFLICT DO NOTHING;
```

**DECISION REQUIRED 8.7.1:** Residency thresholds. `RECOMMENDED DEFAULT: as encoded in classify_residency above. Tune after observing real data.`

**DECISION REQUIRED 8.7.2:** BLE MAC randomization strategy. `RECOMMENDED DEFAULT: v1 stores all MACs, marks address_type. Residency analysis is computed but flagged as unreliable for address_type=1 (random) devices. The frontend displays a warning badge for random-MAC devices in the residency view. Clustering of RPAs is deferred to v2 (deep parser must extract IRK-friendly features first).`

**ACCEPTANCE CRITERIA:**
1. `make db-up` applies all migrations and reports 0 errors.
2. `psql -c "SELECT name FROM _migrations ORDER BY name;"` lists all four migrations.
3. Inserting 1000 fixture rows into `raw_packets` and waiting 2 minutes results in continuous aggregate rows.
4. `SELECT classify_residency(mac) FROM device_summary` returns valid classes.

**Skill-set:** PostgreSQL administration, TimescaleDB, SQL schema design, PL/pgSQL.

---

### 8.8 Deep Packet Parser (C++)

**Purpose:** Parse rotated PCAP files, extract rich BLE protocol fields, emit JSON Lines.

**Module layout:**

```
modules/bluetooth/deep-parser/src/
├── main.cpp              # CLI: --in <pcap-or-zst> --out <jsonl-or-stdout>
├── pcap_input.{cpp,hpp}  # libpcap reader; transparent zstd via piped decompression
├── ble_dissector.{cpp,hpp}  # Link-layer + advertising PDU parsing
├── advdata_parser.{cpp,hpp} # TLV parsing of AdvData payload
└── jsonl_output.{cpp,hpp}
```

**CONTRACT 8.8-A (JSONL output schema):**

One JSON object per line, fields:

```json
{
  "ts": "2026-06-06T14:00:00.123456Z",
  "sniffer_id": 1,
  "mac": "aa:bb:cc:dd:ee:ff",
  "address_type": "random",
  "rssi": -67,
  "channel": 37,
  "pdu_type": "ADV_IND",
  "advdata": {
    "flags": 6,
    "local_name": "MyDevice",
    "tx_power": -4,
    "service_uuids_16": ["180D", "180F"],
    "service_uuids_128": [],
    "manufacturer": {
      "company_id": "004C",
      "data_hex": "1006..."
    },
    "service_data": []
  },
  "raw_advdata_hex": "02011a0303aafe..."
}
```

**DECISION REQUIRED 8.8.1:** BLE dissection library. `RECOMMENDED DEFAULT: Roll our own dissector for the v1 field set. Avoids the cost of forking processes per packet. Reference: Bluetooth Core Specification v5.4, Volume 6 Part B for Link Layer, Volume 3 Part C for GAP` [^ble-core-spec].

**DECISION REQUIRED 8.8.2:** Field scope. `RECOMMENDED DEFAULT: As listed in CONTRACT 8.8-A. Connection events (CONNECT_REQ details, LL Control PDUs) are noted but not parsed in v1. Add in v1.1.`

**`device_enrichment` schema (added by migration 0005):**

```sql
CREATE TABLE IF NOT EXISTS device_enrichment (
    id                BIGSERIAL PRIMARY KEY,
    mac_address       BYTEA NOT NULL,
    observed_ts       TIMESTAMPTZ NOT NULL,
    sniffer_id        SMALLINT NOT NULL,
    local_name        TEXT,
    service_uuids_16  TEXT[],
    service_uuids_128 TEXT[],
    manufacturer_id   INT,
    manufacturer_data BYTEA,
    tx_power          SMALLINT,
    flags             SMALLINT,
    raw_advdata       BYTEA,
    UNIQUE (mac_address, observed_ts, sniffer_id)
);
CREATE INDEX idx_device_enrichment_mac ON device_enrichment(mac_address, observed_ts DESC);
```

**Test plan:**

| Test | Type | File |
|------|------|------|
| Dissect known ADV_IND frame | Unit | `tests/dissector_test.cpp` |
| Parse all standard AdvData TLV types | Unit | `tests/advdata_test.cpp` |
| End-to-end on fixture PCAP yields golden JSONL | Integration | `tests/integration/test_deep_parser_golden.sh` |
| Handles zstd-compressed input | Integration | `tests/integration/test_deep_parser_zstd.sh` |
| Performance: process 30-min fixture in under 5 minutes | Performance | `tests/integration/test_deep_parser_perf.sh` |

**ACCEPTANCE CRITERIA:**
1. Running on `tests/fixtures/pcap/ubertooth-sample-001.pcap` produces JSONL matching `tests/fixtures/expected/deep-parser-sample-001.jsonl`.
2. zstd input works transparently.
3. Memory usage stays under 100 MB for a 30-minute file.

**VERIFICATION:**
```bash
cd modules/bluetooth/deep-parser && cmake -B build && cmake --build build && ctest --test-dir build --output-on-failure
./build/blesniff-deep-parse --in ../../tests/fixtures/pcap/ubertooth-sample-001.pcap > /tmp/out.jsonl
diff <(jq -S . /tmp/out.jsonl) <(jq -S . ../../tests/fixtures/expected/deep-parser-sample-001.jsonl)
```

**Skill-set:** C++17, libpcap, BLE Core Specification (V6 Part B, V3 Part C), bit-level parsing, JSON serialization (nlohmann/json), CMake.

---

### 8.9 ML Enrichment (Python)

**Purpose:** Read deep-parser JSONL, derive higher-level features, write to `device_enrichment` and update `device_summary.enrichment_data`.

**Module layout:**

```
modules/bluetooth/ml-enrichment/src/blesniff_ml/
├── classifier.py    # Rule-based + sklearn pipeline
├── clusterer.py     # RPA clustering (placeholder in v1)
├── features.py      # Feature extraction from JSONL
└── runner.py        # CLI: reads JSONL stdin, writes to DB
```

**v1 scope:**

* **Rule-based device class** from manufacturer ID and service UUIDs:
  * `0x004C` (Apple) + service `FE0F` → `apple_continuity`
  * `0x0006` (Microsoft) → `microsoft_swift_pair`
  * Service `0x180F` → `battery_service_device`
  * Service `0xFEAA` → `eddystone_beacon`
  * Service `0xFD6F` → `exposure_notification`
* **Output:** appends to `device_summary.enrichment_data` JSONB with a `classes` array.

**DECISION REQUIRED 8.9.1:** Initial ML scope. `RECOMMENDED DEFAULT: Rule-based classifier only in v1. Full ML (sklearn-based) deferred until baseline rule-based labels exist for training data.`

**Pipeline invocation:**

```bash
ble-deep-parse --in /var/lib/tianer/pcap/ut1/20260606-1200.pcap.zst \
  | python -m blesniff_ml.runner --sniffer-id 1
```

**Test plan:**

| Test | Type | File |
|------|------|------|
| Apple manufacturer ID classifies correctly | Unit | `tests/test_classifier.py` |
| Eddystone service classifies | Unit | `tests/test_classifier.py` |
| Unknown manufacturer not labeled | Unit | `tests/test_classifier.py` |
| Integration: JSONL stream → DB row | Integration | `tests/integration/test_ml_e2e.sh` |

**ACCEPTANCE CRITERIA:**
1. Known-fixture JSONL produces expected `device_summary.enrichment_data.classes` values.
2. Pipeline tolerates JSONL parse errors (logs, skips).

**Skill-set:** Python 3.13, JSON streaming, BLE protocol knowledge, future: pandas + scikit-learn.

---

### 8.10 Backend API (FastAPI)

**Purpose:** Serve REST API to the frontend; serve compiled frontend bundle.

**Module layout:**

```
platform/api/src/blesniff_api/
├── main.py            # FastAPI app, middleware, mount static
├── db.py              # asyncpg pool
├── auth.py            # API key middleware
├── models.py          # Pydantic models
└── routers/
    ├── devices.py
    ├── alerts.py
    └── health.py
```

**CONTRACT 8.10-A (API endpoints):**

| Method | Path | Query Params | Response |
|--------|------|--------------|----------|
| GET | `/api/health` | (none) | `{"status":"ok","db":"ok","sniffers":[{"id":1,"name":"ut1","last_heartbeat":"...","running":true}]}` |
| GET | `/api/devices` | `q`, `class`, `seen_after`, `sort`, `limit`, `offset` | `{"total":N,"items":[DeviceSummary,...]}` |
| GET | `/api/devices/{mac}` | (none) | `DeviceDetail` |
| GET | `/api/devices/{mac}/timeline` | `from`, `to`, `bucket=5m` | `{"buckets":[{"ts":"...","count":N,"avg_rssi":-67},...]}` |
| GET | `/api/devices/{mac}/enrichment` | `limit=100` | `[EnrichmentRecord,...]` |
| GET | `/api/alerts/new-devices` | `since` (default last 24h) | `[DeviceSummary,...]` |
| GET | `/api/stats/overview` | (none) | `{"total_devices":N,"active_last_hour":N,"new_today":N,...}` |
| GET | `/metrics` | (none) | Prometheus text format |

`DeviceSummary` Pydantic model:
```python
class DeviceSummary(BaseModel):
    mac: str                    # "aa:bb:cc:dd:ee:ff"
    address_type: Literal["public","random","unknown"]
    first_seen: datetime
    last_seen: datetime
    total_count: int
    distinct_days: int
    residency_class: str | None
    classes: list[str] = []
```

**Authentication:** All `/api/*` endpoints require `X-API-Key` header matching the contents of `BLESNIFF_API_KEY_FILE`. `/metrics` is bound to localhost only and does not require auth.

**DECISION REQUIRED 8.10.1:** API endpoint completeness. `RECOMMENDED DEFAULT: Above set covers v1. /api/devices/{mac}/timeline limited to 90 days of history.`

**Test plan:**

| Test | Type | File |
|------|------|------|
| Each endpoint returns expected schema | Unit | `tests/test_routers_*.py` |
| Auth rejects missing API key | Unit | `tests/test_auth.py` |
| Query params validate | Unit | `tests/test_validation.py` |
| Pagination works | Integration | `tests/integration/test_pagination.sh` |
| /api/health reports DB down correctly | Integration | `tests/integration/test_health.sh` |

**ACCEPTANCE CRITERIA:**
1. OpenAPI schema available at `/docs` and matches CONTRACT 8.10-A.
2. P95 endpoint latency under 200 ms for cached aggregates.
3. Missing API key returns 401.

**Skill-set:** Python 3.13, FastAPI, asyncpg, Pydantic v2, async testing.

---

### 8.11 Frontend (Vue 3 + Vite + TypeScript)

**Module layout:**

```
frontend/src/
├── main.ts
├── App.vue
├── router/index.ts          # Vue Router routes
├── api/
│   ├── client.ts            # fetch wrapper with API key header
│   └── types.ts             # Mirror of CONTRACT 8.10-A
├── stores/
│   ├── devices.ts           # Pinia store for device list
│   └── alerts.ts
├── views/
│   ├── DeviceList.vue
│   ├── DeviceDetail.vue
│   ├── Alerts.vue
│   └── Health.vue
└── components/
    ├── GrafanaPanel.vue     # iframe wrapper
    ├── DeviceCard.vue
    └── ResidencyBadge.vue
```

**Routing:**

| Path | View |
|------|------|
| `/` | DeviceList |
| `/devices/:mac` | DeviceDetail |
| `/alerts` | Alerts |
| `/health` | Health |

**DECISION REQUIRED 8.11.1:** UI library. `RECOMMENDED DEFAULT: TailwindCSS for utility styling + headlessui/vue for primitives. Avoids component-library lock-in.`

**DECISION REQUIRED 8.11.2:** Grafana embedding. `RECOMMENDED DEFAULT: iframe with anonymous read-only access enabled in Grafana, restricted to LAN.`

**Test plan:**

| Test | Type | File |
|------|------|------|
| DeviceList renders mock data | Component (Vitest) | `tests/DeviceList.test.ts` |
| API client adds X-API-Key header | Unit | `tests/client.test.ts` |
| ResidencyBadge shows warning for random-MAC devices | Component | `tests/ResidencyBadge.test.ts` |
| Routing navigates correctly | Component | `tests/router.test.ts` |

**ACCEPTANCE CRITERIA:**
1. `npm run build` produces a `dist/` bundle.
2. `npm run test` passes.
3. With the API running, `npm run dev` and visiting `/` lists devices.
4. Lighthouse accessibility score >= 90.

**Skill-set:** TypeScript (strict), Vue 3 Composition API, Vite, Pinia, TailwindCSS, Vitest.

---

### 8.12 Grafana

**Provisioning files** are read at Grafana startup from `/etc/grafana/provisioning/`. Symlinked from `grafana/provisioning/` in the repo by `deploy/setup.sh`.

**Data source (`grafana/provisioning/datasources/timescaledb.yaml`):**

```yaml
apiVersion: 1
datasources:
  - name: TimescaleDB
    type: postgres
    url: 127.0.0.1:5432
    database: tianer
    user: tianer_grafana
    secureJsonData:
      password: $__file{/etc/tianer/secrets/grafana_db_password}
    jsonData:
      sslmode: disable
      postgresVersion: 1700
      timescaledb: true
    isDefault: true
```

**Dashboards (saved JSON):**

| File | Purpose |
|------|---------|
| `live-metrics.json` | Packets per second per sniffer, total devices last hour, RSSI histogram |
| `device-explorer.json` | Sortable device table |
| `per-device-drilldown.json` | Templated dashboard with $mac variable |
| `pipeline-health.json` | Ingest lag, gap count, backfill status |

**ACCEPTANCE CRITERIA:**
1. Grafana starts and shows the TimescaleDB data source as healthy.
2. All four dashboards load without query errors.
3. Anonymous access works on the LAN address; disabled on the public interface.

**Skill-set:** Grafana provisioning, SQL, JSON dashboard authoring.

---

## 9. Inter-Component Contracts (Summary)

Quick reference of every contract referenced by ID elsewhere in the document.

| ID | From | To | Format | Section |
|----|------|-----|--------|---------|
| CONTRACT 8.4-A | tshark | ingest bridge | Pipe-delimited fields | 8.4 |
| CONTRACT 8.5-A | tshark line | C++ `Packet` struct | Parser function signature | 8.5 |
| CONTRACT 8.6-A | gap detector | DB | Gap definition rule | 8.6 |
| CONTRACT 8.8-A | deep parser | ML enrichment | JSONL | 8.8 |
| CONTRACT 8.10-A | API | frontend | REST/JSON | 8.10 |

**Contract change policy:** Any change to a contract requires:
1. A new ADR in `docs/adr/`.
2. A migration plan if the change is not backward compatible.
3. Updated tests on both sides of the contract.
4. Bumping the contract ID suffix letter (8.4-A → 8.4-B).

---

## 10. Testing Strategy

### 10.1 Test Pyramid

| Level | Scope | Tools | Where it lives | When it runs |
|-------|-------|-------|----------------|--------------|
| Unit | Single function or class, no I/O | GoogleTest (C++), pytest (Python), Vitest (TS), bats (bash) | `modules/*/tests/`, `platform/*/tests/` | Every commit (pre-commit + CI) |
| Integration | Two or more components with real DB/files | shell scripts driving compiled binaries | `tests/integration/` | CI on push to main and PR |
| End-to-end | Full pipeline on the Pi (or Pi-like container) | `tests/e2e/smoke.sh` | Pi smoke target | Manual + nightly on Pi |
| Performance | Throughput and latency benchmarks | custom scripts | `tests/perf/` | Manual + weekly on Pi |

### 10.2 Per-Language Test Conventions

**C++ (GoogleTest):**
* Naming: `<module>_test.cpp` per source module.
* Discovery: CMake `add_test` via `gtest_discover_tests`.
* Coverage target: 80% line coverage on parser and dissector modules. Measured by `gcov`.
* Mocks: avoid heavy mocking; prefer fakes (e.g., `FakeWriter` implementing `IWriter`).

**Python (pytest):**
* Naming: `tests/test_<module>.py`.
* Async: use `@pytest.mark.asyncio` (pytest-asyncio).
* Fixtures: `tests/conftest.py` provides `db_conn` (auto-rollback transaction), `tmp_pcap` (temporary PCAP file), `mock_tshark_output` (stub stdout).
* Coverage target: 85% on `gap-detector` and `api`. Measured by `coverage.py`.

**TypeScript (Vitest):**
* Naming: `*.test.ts` adjacent to source or under `tests/`.
* DOM: `@vue/test-utils` + `jsdom`.
* API mocking: `msw` (Mock Service Worker).
* Coverage target: 75% on stores and API client.

**Shell (bats-core):**
* Naming: `test_<feature>.bats`.
* Fixtures: scripts that create temp dirs in `mktemp -d`, clean up in teardown.

### 10.3 Database Testing

* `db/tests/` contains pgTAP-style SQL files (or simpler `psql` scripts).
* Each test runs in a transaction that rolls back at the end.
* The integration test harness spins up a separate test database `tianer_test` per run.
* Seed data from `tests/fixtures/sql/seed_*.sql`.

### 10.4 Mocking Real Sniffers

For tests that don't have hardware, use one of:

1. **PCAP replay (preferred):** `tools/pcap-replay.py` reads a fixture PCAP and writes it to a FIFO at a configurable rate, preserving inter-packet timing or accelerating.
2. **Mock sniffer process:** `tools/mock-sniffer.py` emits synthetic packets with controllable parameters (rate, MAC pool, RSSI variance).

Both are invoked by integration tests instead of `ubertooth-btle` or `nrfutil`.

### 10.5 Performance Tests

| Test | Target |
|------|--------|
| Ingest throughput | 1500 packets/sec sustained for 5 min, no drops |
| Ingest P95 latency | < 1500 ms (line received → DB row visible) |
| API P95 latency | < 200 ms on aggregated endpoints |
| Deep parser throughput | 30-min PCAP file processed in < 5 min |
| End-to-end pipeline latency | < 5 sec |

### 10.6 CI Pipeline (`ci/test-all.sh`)

Sequential phases. Any failure stops the run.

1. `pre-commit run --all-files` (formatting, lint, basic checks).
2. C++ unit tests: `cmake -B build && ctest --test-dir build`.
3. Python unit tests: `uv run pytest modules/*/tests/ platform/*/tests/`.
4. TypeScript unit tests: `cd platform/frontend && npm test`.
5. SQL migrations smoke: apply all migrations to a fresh DB.
6. Integration tests: `tests/integration/*.sh` against a docker-compose stack.
7. Frontend build: `cd platform/frontend && npm run build`.
8. Container build (optional, for distribution).

---

## 11. Test Fixtures and Sample Data

### 11.1 PCAP Fixtures

Located in `tests/fixtures/pcap/`. Committed to the repo (real captured data, anonymized MACs).

| File | Source | Duration | Packets | Purpose |
|------|--------|----------|---------|---------|
| `ubertooth-sample-001.pcap` | Ubertooth One | 30s | ~3000 | Smoke test |
| `nrf-sample-001.pcap` | nRF52840 | 30s | ~3000 | Smoke test |
| `ubertooth-large.pcap.zst` | Ubertooth | 30m | ~200k | Throughput tests |
| `ubertooth-malformed.pcap` | Hand-crafted | n/a | 10 | Error path tests |
| `mixed-rpa.pcap` | Synthetic | 5m | ~5k | RPA-handling tests |

**DECISION REQUIRED 11.1.1:** Source of fixture PCAPs. `RECOMMENDED DEFAULT: Capture during T03/T04 with real hardware in a controlled environment (Faraday cage or remote site) to avoid capturing third-party devices. MACs anonymized via deterministic hash before commit.`

### 11.2 Expected Output Fixtures

`tests/fixtures/expected/`:

| File | Purpose |
|------|---------|
| `tshark-ubertooth-sample-001.txt` | Golden tshark output for the Ubertooth fixture |
| `deep-parser-sample-001.jsonl` | Golden JSONL from deep parser |
| `device-summary-after-sample-001.json` | Expected `device_summary` rows after ingesting the fixture |

### 11.3 Synthetic Data Generator

`tools/seed-test-data.py` generates `raw_packets` rows for testing without real sniffers:

```
./tools/seed-test-data.py --devices 100 --packets-per-device 500 --start "2026-06-01 00:00" --duration-hours 168
```

Used to populate a test DB for frontend development and Grafana dashboard authoring.

---

## 12. Cross-Cutting Concerns

### 12.1 Logging

* All services log to `journald` via stderr.
* Format: structured key=value where the language supports it; otherwise plain prose.
* Log levels: `error`, `warn`, `info`, `debug`. Default `info`. Per-service override via `BLESNIFF_<SERVICE>_LOG_LEVEL`.

### 12.2 Metrics

* Prometheus format on `/metrics` endpoint of the API.
* Each service reports internal counters via a local TCP push (the API aggregates), or by writing to a known file path read by the API. `RECOMMENDED DEFAULT: a sidecar table sniffer_metrics_snapshot updated by each service.`

Metrics catalogue:
| Metric | Type | Labels |
|--------|------|--------|
| `blesniff_packets_ingested_total` | counter | `sniffer_id` |
| `blesniff_ingest_latency_seconds` | histogram | `sniffer_id` |
| `blesniff_malformed_packets_total` | counter | `sniffer_id` |
| `blesniff_gap_detected_total` | counter | `sniffer_id` |
| `blesniff_backfill_rows_total` | counter | `sniffer_id` |
| `blesniff_sniffer_running` | gauge (0/1) | `sniffer_id` |
| `blesniff_disk_used_bytes` | gauge | `mount_point` |

### 12.3 Backup

* Nightly `pg_basebackup` to a configured destination.
* Backup retention: 14 days.

**DECISION REQUIRED 12.3.1:** Backup destination. `RECOMMENDED DEFAULT: External USB drive mounted at /mnt/backup if present; otherwise skip with a warning. v1 acceptable to skip.`

### 12.4 Security

* All web interfaces bound to `127.0.0.1` by default.
* External access via the Pi's existing WireGuard tunnel (out of scope for this project to configure).
* API key required for `/api/*`. Generated on first deploy.
* PostgreSQL accepts only local connections (Unix socket and 127.0.0.1).
* No sudoers entries created for the `tianer` user.

### 12.5 Service Orchestration

All services run under systemd. Key units:

| Unit | Purpose | Type |
|------|---------|------|
| `blesniff-sniffer@<name>.service` | Per-sniffer wrapper | template |
| `blesniff-tshark@<name>.service` | Per-sniffer tshark | template |
| `blesniff-ingest@<name>.service` | Per-sniffer ingest bridge | template |
| `blesniff-rotate.service` + `.timer` | PCAP rotation | oneshot + timer |
| `blesniff-gap-detector.service` + `.timer` | Gap detection | oneshot + timer (5 min) |
| `blesniff-api.service` | FastAPI | simple |
| `blesniff.target` | Pulls in all enabled sniffers | target |

Dependency chain per sniffer: `postgresql -> blesniff-tshark@<n> -> blesniff-ingest@<n> -> blesniff-sniffer@<n>` (ingest must be ready to read tshark's stdout, tshark must be ready to read the FIFO before sniffer writes).

RESOLVED by Q1: all components now run in rootless Podman containers managed by Quadlet. USB devices are passed through from the host. See component-breakdown.md §2.1a for the container-pod architecture.

---

## 13. Deployment and Setup Automation

### 13.1 Makefile Targets

The root `Makefile` is the agent's entry point.

| Target | Purpose |
|--------|---------|
| `make help` | List targets with descriptions |
| `make setup` | Idempotent host bootstrap: install packages, create users, configure systemd, apply DB migrations |
| `make build` | Build all components (C++, frontend, Python wheels if any) |
| `make install` | Install built artifacts into system locations |
| `make db-up` | Apply pending DB migrations |
| `make db-reset` | Drop and recreate the database (DESTRUCTIVE; requires `CONFIRM=yes`) |
| `make test-unit` | Run all unit tests |
| `make test-integration` | Run integration tests (requires DB up) |
| `make test-e2e` | Run end-to-end smoke test on the Pi |
| `make lint` | Run all linters |
| `make format` | Apply formatters |
| `make services-start` | systemctl start blesniff.target |
| `make services-stop` | systemctl stop blesniff.target |
| `make services-status` | systemctl status of all blesniff units |
| `make clean` | Remove build artifacts |
| `make logs` | tail -f journalctl for all blesniff units |

### 13.2 `deploy/setup.sh` (host bootstrap)

Idempotent. Safe to run multiple times. Phases:

1. Verify OS (`/etc/os-release` matches Raspberry Pi OS Trixie or compatible).
2. Create `tianer` user and group if missing.
3. Install apt packages from `deploy/apt-packages.txt` (idempotent via `apt-get install`).
4. Install PostgreSQL 17 + TimescaleDB extension (via official TimescaleDB apt repo).
5. Install Grafana (via official apt repo).
6. Install Node.js 24 LTS (via NodeSource).
7. Install Python 3.13 (default on Trixie) + uv.
8. Install Ubertooth tools (build from source; `deploy/scripts/install-ubertooth.sh`).
9. Install nrfutil (download official ARM64 binary).
10. Create directory tree (`/etc/tianer/`, `/var/lib/tianer/`, `/var/log/tianer/`, `/var/run/tianer/`).
11. Install udev rules.
12. Install systemd unit files.
13. Reload systemd, enable timers, do NOT start sniffers yet (operator must verify config).
14. Apply DB migrations.
15. Generate API key and DB password if missing.
16. Print "Setup complete. Next steps: ..." with the next commands.

### 13.3 Build Artifacts and Install Paths

| Component | Built where | Installed to |
|-----------|------------|--------------|
| ingest-bridge | `modules/bluetooth/ingest-bridge/build/blesniff-ingest` | `/usr/local/bin/blesniff-ingest` |
| deep-parser | `modules/bluetooth/deep-parser/build/blesniff-deep-parse` | `/usr/local/bin/blesniff-deep-parse` |
| sniffer-wrapper | `modules/bluetooth/sniffers/*.sh` | `/usr/local/lib/tianer/wrap/` |
| gap-detector | Python venv | `/opt/tianer/gap-detector/` |
| ml-enrichment | Python venv | `/opt/tianer/ml-enrichment/` |
| api | Python venv | `/opt/tianer/api/` |
| frontend | `platform/frontend/dist/` | `/usr/share/tianer/frontend/` |
| db migrations | `db/migrations/*.sql` | `/usr/share/tianer/migrations/` |
| systemd units | `deploy/systemd/*.service` | `/etc/systemd/system/` |

### 13.4 Adding a New Sniffer

Procedure (also in `docs/runbooks/add-new-sniffer.md`):

1. Edit `/etc/tianer/sniffers.yaml` to add the new sniffer block.
2. Run `make services-restart` (which validates the YAML and starts new per-sniffer units).
3. Verify via `make services-status` and `curl -H "X-API-Key: $KEY" http://127.0.0.1:8080/api/health`.

---

## 14. End-to-End Verification

### 14.1 Smoke Test (`tests/e2e/smoke.sh`)

Run by an autonomous agent after T22 to confirm the entire stack works.

```bash
#!/usr/bin/env bash
set -euo pipefail

step() { echo; echo "=== $1 ==="; }

step "1. Verify all systemd units are active"
for unit in postgresql grafana-server blesniff-api blesniff-rotate.timer blesniff-gap-detector.timer; do
    systemctl is-active --quiet "$unit" || { echo "FAIL: $unit not active"; exit 1; }
done

step "2. Verify DB schema"
psql -U tianer -d tianer -t -c "SELECT COUNT(*) FROM _migrations;" | grep -q '[1-9]' || exit 1

step "3. Verify API health endpoint"
api_key=$(cat /etc/tianer/secrets/api_key)
curl -sf -H "X-API-Key: $api_key" http://127.0.0.1:8080/api/health | jq -e '.status == "ok"'

step "4. Inject synthetic packets via mock-sniffer and verify they land in DB"
sniffer_id=99
psql -U tianer -d tianer -c \
  "INSERT INTO sniffers (sniffer_id, name, type, enabled) VALUES ($sniffer_id, 'mock99', 'mock', false)
   ON CONFLICT DO NOTHING;"

./tools/mock-sniffer.py --sniffer-id $sniffer_id --packets 500 --rate 100 \
  | /usr/local/bin/blesniff-ingest --sniffer-id $sniffer_id --input - &
INGEST_PID=$!
sleep 8
kill $INGEST_PID 2>/dev/null || true
wait $INGEST_PID 2>/dev/null || true

count=$(psql -U tianer -d tianer -t -c \
  "SELECT COUNT(*) FROM raw_packets WHERE sniffer_id = $sniffer_id AND ts > NOW() - INTERVAL '1 minute';")
[[ $count -ge 450 ]] || { echo "FAIL: only $count rows ingested (expected ~500)"; exit 1; }

step "5. Verify continuous aggregate populated"
sleep 70
agg_count=$(psql -U tianer -d tianer -t -c \
  "SELECT COUNT(*) FROM device_5min_buckets WHERE bucket > NOW() - INTERVAL '15 minutes';")
[[ $agg_count -gt 0 ]] || { echo "FAIL: continuous aggregate empty"; exit 1; }

step "6. Verify API returns devices"
device_count=$(curl -sf -H "X-API-Key: $api_key" http://127.0.0.1:8080/api/devices | jq '.total')
[[ $device_count -gt 0 ]] || { echo "FAIL: API returned 0 devices"; exit 1; }

step "7. Verify gap detector runs without error"
systemctl start blesniff-gap-detector.service
sleep 5
gap_status=$(systemctl show blesniff-gap-detector.service -p ExecMainStatus --value)
[[ $gap_status == "0" ]] || { echo "FAIL: gap detector exited $gap_status"; exit 1; }

step "8. Verify frontend serves"
curl -sf http://127.0.0.1:8080/ | grep -q 'BLE Sniffer' || { echo "FAIL: frontend not served"; exit 1; }

step "9. Verify Grafana is reachable and dashboards load"
curl -sf http://127.0.0.1:3000/api/health | jq -e '.database == "ok"'
curl -sf "http://127.0.0.1:3000/api/search?type=dash-db" | jq -e 'length >= 4'

step "10. Cleanup mock data"
psql -U tianer -d tianer -c "DELETE FROM raw_packets WHERE sniffer_id = $sniffer_id;"
psql -U tianer -d tianer -c "DELETE FROM sniffers WHERE sniffer_id = $sniffer_id;"

echo
echo "SMOKE TEST PASSED"
```

### 14.2 Health Endpoints

| Endpoint | Purpose | Expected |
|----------|---------|----------|
| `http://127.0.0.1:8080/api/health` | Application health | `{"status":"ok",...}` |
| `http://127.0.0.1:8080/metrics` | Prometheus metrics | text/plain |
| `http://127.0.0.1:3000/api/health` | Grafana health | `{"database":"ok",...}` |
| `psql -c "SELECT 1"` | DB reachability | `1` |

### 14.3 Smoke Test Acceptance

The agent reports the platform as deployed only after `make test-e2e` exits 0.

---

## 15. Implementation Task Breakdown

Tasks are dependency-ordered. Each task is a unit of work that produces a verifiable artifact.

**Task header format:**
* **ID, Title, Skill-set, Depends on, Estimated effort (S/M/L)**
* **Inputs:** what must exist before starting
* **Deliverables:** files/artifacts produced
* **Acceptance Criteria:** specific testable conditions
* **Verification Command(s):** exact commands an agent runs

---

### T01: Host Bootstrap and Prerequisites

**Skill-set:** Linux administration, systemd, udev, group/capability management, polkit basics.
**Depends on:** none.
**Effort:** M.

**Inputs:**
* Raspberry Pi CM5 with fresh Raspberry Pi OS 64-bit (Trixie) [^rpi-trixie].
* SSH access with sudo privileges for the operator account.
* Ubertooth and nRF dongles available (for udev validation).

**Deliverables:**
Per section 6 in full:
* `deploy/setup.sh` orchestrates the entire bootstrap.
* `deploy/scripts/create-user.sh` (section 6.1): creates the `tianer` system user, adds it to `plugdev`, `dialout`, and `wireshark` groups via `usermod -aG`.
* `deploy/scripts/setup-wireshark.sh` (section 6.3): installs tshark and configures `dumpcap` capabilities via `dpkg-reconfigure wireshark-common`.
* `deploy/scripts/create-dirs.sh` (section 6.4): creates `/var/lib/tianer`, `/etc/tianer`, etc., with correct ownership and mode.
* `deploy/udev/99-tianer.rules` (section 6.2): grants device access via the `plugdev` group with `MODE=0660`.
* `/etc/tmpfiles.d/tianer.conf`: declares FIFOs and runtime directories.
* `deploy/apt-packages.txt`: every apt dependency listed.
* `tests/integration/test_prereqs.sh` (section 6.9): end-to-end prereq verification.

**Acceptance Criteria:**
1. Running `sudo deploy/setup.sh` on a fresh host completes without error.
2. Re-running the same script produces no changes (idempotency).
3. `tests/integration/test_prereqs.sh` exits 0.
4. The `tianer` user can perform all platform operations without `sudo`:
   * Read Ubertooth device (`sudo -u tianer ubertooth-util -v`).
   * Read nRF serial port.
   * Invoke `dumpcap` / `tshark` for capture.
   * Connect to PostgreSQL.
5. No sudoers entries exist (`ls /etc/sudoers.d/tianer*` returns nothing).

**Verification:**
```bash
sudo deploy/setup.sh
sudo deploy/setup.sh  # second run; output should show no changes
tests/integration/test_prereqs.sh
```

---

### T02: PostgreSQL + TimescaleDB + Initial Migrations

**Skill-set:** PostgreSQL administration, TimescaleDB.
**Depends on:** T01.
**Effort:** M.

**Inputs:**
* PostgreSQL 17 and TimescaleDB packages available.

**Deliverables:**
* PostgreSQL 17 running with TimescaleDB extension enabled.
* `db/migrations/0001_init.sql` through `0004_residency_classifier.sql` applied.
* `db/apply-migrations.sh` script that is idempotent.
* `make db-up` works.

**Acceptance Criteria:**
1. `psql -U tianer -d tianer -c "SELECT default_version FROM pg_available_extensions WHERE name='timescaledb';"` shows >= 2.16.
2. `\d raw_packets` shows the hypertable with expected columns.
3. `SELECT * FROM _migrations;` lists all four migrations.
4. Inserting a row into `raw_packets` and querying via the continuous aggregate (after refresh) returns it.

**Verification:**
```bash
make db-up
make db-up  # idempotent re-run
psql -U tianer -d tianer -f db/tests/test_schema.sql
```

---

### T03: Ubertooth Toolchain

**Skill-set:** Ubertooth ecosystem, libbtbb build, BLE basics.
**Depends on:** T01.
**Effort:** S.

**Deliverables:**
* `ubertooth-btle`, `ubertooth-rx`, `ubertooth-util` installed.
* `deploy/scripts/install-ubertooth.sh` builds and installs from source against the firmware version on the dongle.
* A captured fixture PCAP at `tests/fixtures/pcap/ubertooth-sample-001.pcap`.

**Acceptance Criteria:**
1. `ubertooth-util -v` reports firmware version matching host tools.
2. `ubertooth-btle -A -c /tmp/test.pcap` runs for 30s and produces a non-empty valid PCAP.
3. `tshark -r /tmp/test.pcap -c 5` parses without errors.

**Verification:**
```bash
ubertooth-util -v
timeout 30 ubertooth-btle -A -c /tmp/test.pcap || true
tshark -r /tmp/test.pcap -c 5
```

---

### T04: Nordic nRF Toolchain

**Skill-set:** Nordic tooling, Python.
**Depends on:** T01.
**Effort:** S.

**Deliverables:**
* `nrfutil` ARM64 installed at `/usr/local/bin/nrfutil`.
* `nrfutil ble-sniffer` subcommand available.
* nRF dongle flashed with sniffer firmware.
* Fixture PCAP captured to `tests/fixtures/pcap/nrf-sample-001.pcap`.

**Acceptance Criteria:**
1. `nrfutil --version` succeeds.
2. `nrfutil ble-sniffer sniff --port /dev/tianer/nrf0 --output-pcap-file /tmp/nrf.pcap` runs for 30s and produces a valid PCAP.

**DECISION 5.1.2 Resolution:** If `nrfutil ble-sniffer` is not available for ARM64, fall back to the Python script from the Nordic SDK and document the alternative path in `docs/runbooks/nrf-fallback.md`.

**Verification:**
```bash
nrfutil --version
timeout 30 nrfutil ble-sniffer sniff --port /dev/tianer/nrf0 --output-pcap-file /tmp/nrf.pcap || true
tshark -r /tmp/nrf.pcap -c 5
```

---

### T05: Sniffer Wrapper Scripts

**Skill-set:** Linux shell, POSIX IPC, systemd templates.
**Depends on:** T03, T04.
**Effort:** M.

**Deliverables:**
* `modules/bluetooth/sniffers/ubertooth-wrap.sh`, `nrf-wrap.sh`.
* `modules/bluetooth/sniffers/tshark-wrap.sh` (per CONTRACT 8.4-A).
* `modules/bluetooth/sniffers/heartbeat.sh` (companion process).
* systemd templates `blesniff-sniffer@.service` and `blesniff-tshark@.service`.
* bats tests for the wrappers.

**Acceptance Criteria:** As in section 8.1, 8.4.

**Verification:**
```bash
bats modules/bluetooth/sniffers/tests/
./tests/integration/test_dual_write.sh
./tests/integration/test_tshark_format.sh
```

---

### T06: PCAP Rotation

**Skill-set:** Bash, systemd timers, zstd.
**Depends on:** T05.
**Effort:** S.

**Deliverables:**
* `deploy/scripts/rotate-pcap.sh`.
* `blesniff-rotate.service` + `blesniff-rotate.timer`.

**Acceptance Criteria:** As in section 8.2.

**Verification:**
```bash
./tests/integration/test_rotation_naming.sh
./tests/integration/test_compression.sh
./tests/integration/test_retention.sh
```

---

### T07: tshark Field Verification

**Skill-set:** Wireshark, BLE link-layer protocol.
**Depends on:** T03, T04.
**Effort:** S.

**Deliverables:**
* `docs/tshark-fields.md` documenting verified field names per DLT.
* Updated `tshark-wrap.sh` if any field name differs from default.
* Golden output file `tests/fixtures/expected/tshark-ubertooth-sample-001.txt`.

**Acceptance Criteria:**
1. Field list confirmed against `tshark -G fields` on the actual installation.
2. CONTRACT 8.4-A documents the final field names.

**Verification:**
```bash
tshark -G fields | grep -E 'btle|nordic_ble' > /tmp/all-fields.txt
# Manually verify the fields in tshark-wrap.sh appear in /tmp/all-fields.txt
diff <(./modules/bluetooth/sniffers/tshark-wrap.sh test < tests/fixtures/pcap/ubertooth-sample-001.pcap | head -20) \
     tests/fixtures/expected/tshark-ubertooth-sample-001.txt
```

---

### T08: Ingest Bridge (C++)

**Skill-set:** C++17, libpqxx, PostgreSQL COPY, CMake.
**Depends on:** T02, T07.
**Effort:** L.

**Deliverables:**
* `modules/bluetooth/ingest-bridge/` complete with `CMakeLists.txt`, sources, GoogleTest tests.
* Binary installed to `/usr/local/bin/blesniff-ingest`.
* systemd template `blesniff-ingest@.service`.

**Acceptance Criteria:** As in section 8.5.

**Verification:**
```bash
cd modules/bluetooth/ingest-bridge && cmake -B build && cmake --build build && ctest --test-dir build --output-on-failure
./tests/integration/test_ingest_e2e.sh
./tests/integration/test_pg_reconnect.sh
./tests/integration/test_ingest_throughput.sh  # >= 1500 packets/sec
```

---

### T09: Gap Detector (Python)

**Skill-set:** Python 3.13, asyncpg, pyshark, SQL, async.
**Depends on:** T06, T08.
**Effort:** L.

**Deliverables:**
* `modules/bluetooth/gap-detector/` Python package.
* Binary entry point `blesniff-gap-detect`.
* `blesniff-gap-detector.service` + `.timer`.

**Acceptance Criteria:** As in section 8.6.

**Verification:**
```bash
cd modules/bluetooth/gap-detector && uv run pytest
./tests/integration/test_gap_detection.sh
./tests/integration/test_backfill_idempotent.sh
```

---

### T10: Continuous Aggregates + Compression Validation

**Skill-set:** TimescaleDB.
**Depends on:** T02, T08.
**Effort:** S.

**Deliverables:**
* Verified continuous aggregate refresh behavior.
* Verified compression policy.
* Performance baseline in `docs/perf-baseline.md`.

**Acceptance Criteria:**
1. After seeding 1M rows across 7 days, queries against `device_5min_buckets` complete in under 50 ms.
2. After 7 days simulated, chunks older than the policy are compressed and a query still works.

**Verification:**
```bash
./tools/seed-test-data.py --packets-per-device 10000 --devices 100 --duration-hours 168
psql -c "EXPLAIN ANALYZE SELECT * FROM device_5min_buckets WHERE bucket > NOW() - INTERVAL '1 day';"
```

---

### T11: Residency Classifier Job

**Skill-set:** SQL, PL/pgSQL, systemd timers.
**Depends on:** T10.
**Effort:** S.

**Deliverables:**
* Migration `0004_residency_classifier.sql` (applied in T02 if pre-written).
* Scheduled job `blesniff-classify.service` + `.timer` running daily.

**Acceptance Criteria:**
1. After 7 days of synthetic data, `device_summary.residency_class` is populated.
2. Classes match seeded patterns.

**Verification:**
```bash
psql -c "SELECT residency_class, COUNT(*) FROM device_summary GROUP BY residency_class;"
```

---

### T12: MAC Randomization Handling

**Skill-set:** BLE Core Specification, SQL, frontend.
**Depends on:** T11.
**Effort:** S.

**Deliverables:**
* `device_summary.address_type` correctly populated.
* Residency classifier excludes or flags `address_type = 1` (random) appropriately.
* Frontend `ResidencyBadge.vue` shows a warning for random-MAC devices.

**Acceptance Criteria:**
1. A seeded random-MAC device with high observation count is shown as `transient` or with a `random` flag, not as `resident`.

**Verification:** Unit and component tests pass.

---

### T13: Deep Packet Parser (C++)

**Skill-set:** C++17, libpcap, BLE Core Specification, CMake.
**Depends on:** T03 (fixtures).
**Effort:** L.

**Deliverables:**
* `modules/bluetooth/deep-parser/` complete.
* Binary `/usr/local/bin/blesniff-deep-parse`.
* Golden fixture `tests/fixtures/expected/deep-parser-sample-001.jsonl`.

**Acceptance Criteria:** As in section 8.8.

**Verification:**
```bash
cd modules/bluetooth/deep-parser && cmake -B build && cmake --build build && ctest --test-dir build --output-on-failure
./tests/integration/test_deep_parser_golden.sh
```

---

### T14: ML Enrichment Scaffold

**Skill-set:** Python, JSON streaming.
**Depends on:** T13.
**Effort:** M.

**Deliverables:**
* `modules/bluetooth/ml-enrichment/` package.
* Rule-based classifier per section 8.9.

**Acceptance Criteria:** As in section 8.9.

**Verification:**
```bash
cd modules/bluetooth/ml-enrichment && uv run pytest
./tests/integration/test_ml_e2e.sh
```

---

### T15: FastAPI Backend

**Skill-set:** Python, FastAPI, asyncpg.
**Depends on:** T11.
**Effort:** L.

**Deliverables:**
* `platform/api/` package.
* All endpoints per CONTRACT 8.10-A.
* OpenAPI schema at `/docs`.
* `blesniff-api.service` systemd unit.

**Acceptance Criteria:** As in section 8.10.

**Verification:**
```bash
cd platform/api && uv run pytest
./tests/integration/test_api_endpoints.sh
curl -sf http://127.0.0.1:8080/openapi.json | jq -e '.paths | keys | length >= 8'
```

---

### T16: Frontend Scaffold

**Skill-set:** TypeScript, Vue 3, Vite.
**Depends on:** T15.
**Effort:** M.

**Deliverables:**
* `frontend/` with router, stores, API client.
* Vitest tests passing.
* `npm run build` produces `dist/`.

**Acceptance Criteria:**
1. `npm test` passes.
2. `npm run build` succeeds.
3. Dev server renders the device list with mocked API.

**Verification:**
```bash
cd platform/frontend && npm install && npm test && npm run build
```

---

### T17: Frontend Views

**Skill-set:** Vue 3, TypeScript, TailwindCSS.
**Depends on:** T16.
**Effort:** L.

**Deliverables:**
* DeviceList, DeviceDetail, Alerts, Health views.
* Pinia stores wired to API.
* Grafana panel embedding via `GrafanaPanel.vue`.

**Acceptance Criteria:**
1. Each view renders against a live API on the Pi.
2. Lighthouse accessibility score >= 90.

**Verification:** Manual visual + automated component tests.

---

### T18: Grafana Provisioning

**Skill-set:** Grafana, SQL.
**Depends on:** T11.
**Effort:** M.

**Deliverables:**
* `grafana/provisioning/` complete.
* Four dashboards in `grafana/dashboards/`.
* Anonymous read-only auth configured per DECISION 8.11.2.

**Acceptance Criteria:** As in section 8.12.

**Verification:**
```bash
curl -sf http://127.0.0.1:3000/api/health | jq -e '.database == "ok"'
curl -sf 'http://127.0.0.1:3000/api/search?type=dash-db' | jq 'length' | grep -E '^[4-9]|[1-9][0-9]+$'
```

---

### T19: systemd Wiring

**Skill-set:** systemd.
**Depends on:** T05, T06, T08, T09, T15.
**Effort:** M.

**Deliverables:**
* All systemd units finalized with correct ordering.
* `blesniff.target` aggregating everything enabled.
* `make services-start` brings up the entire stack in order.

**Acceptance Criteria:**
1. `systemctl start blesniff.target` succeeds, then `journalctl -u 'blesniff-*' --since "1 minute ago"` shows no errors.
2. systemd reports the dependency chain correctly with `systemctl list-dependencies blesniff.target`.

**Verification:**
```bash
make services-stop
make services-start
sleep 30
systemctl is-active blesniff.target
journalctl -u 'blesniff-*' --since "1 minute ago" | grep -iE 'error|fail' && exit 1 || true
```

---

### T20: Metrics and Health

**Skill-set:** FastAPI, Prometheus exposition format, Vue.
**Depends on:** T15, T17.
**Effort:** M.

**Deliverables:**
* `/metrics` endpoint exposing the catalogue in section 12.2.
* Frontend Health view consuming `/api/health`.

**Acceptance Criteria:**
1. All metrics in the catalogue are reported.
2. Health view shows live status for each sniffer.

**Verification:**
```bash
curl -sf http://127.0.0.1:8080/metrics | grep blesniff_packets_ingested_total
```

---

### T21: Backup

**Skill-set:** PostgreSQL, shell.
**Depends on:** T02.
**Effort:** S.

**Deliverables:**
* `deploy/scripts/backup.sh` running `pg_basebackup` nightly.
* `tianer-backup.service` + `.timer`.

**Acceptance Criteria:**
1. Backup runs nightly and produces a tarball at the configured destination.
2. Restoring the backup into an empty PG instance yields working data.

**DECISION 12.3.1 Resolution:** Backup is opt-in via `BLESNIFF_BACKUP_DEST` env var. Empty means skip (with warning).

---

### T22: Security Hardening

**Skill-set:** Linux admin, network security.
**Depends on:** T19, T20.
**Effort:** M.

**Deliverables:**
* All services bound to 127.0.0.1 or LAN-only interface.
* PostgreSQL `pg_hba.conf` restricted to local.
* API key required and validated.
* `ufw` rules applied if firewall is enabled.

**Acceptance Criteria:**
1. `nmap -sT 127.0.0.1` shows only expected ports.
2. From a remote host on the LAN, ports 8080 and 3000 are either reachable (if intended) or refused (if intended).
3. API request without `X-API-Key` returns 401.

**Verification:**
```bash
nmap -sT 127.0.0.1
curl -sf -o /dev/null -w '%{http_code}' http://127.0.0.1:8080/api/devices  # expect 401
curl -sf -H "X-API-Key: $(cat /etc/tianer/secrets/api_key)" http://127.0.0.1:8080/api/devices  # expect 200
```

---

### T23: End-to-End Smoke Test

**Skill-set:** Bash, end-to-end testing.
**Depends on:** All previous tasks.
**Effort:** S.

**Deliverables:**
* `tests/e2e/smoke.sh` passes per section 14.1.

**Acceptance Criteria:** Smoke test exits 0.

**Verification:**
```bash
make test-e2e
```

---

## 16. Open Decisions Summary

| ID | Topic | Recommended Default | Section |
|----|-------|---------------------|---------|
| 3.1 | OS version | Raspberry Pi OS 64-bit Trixie | 3 |
| 3.2 | USB port pinning | udev rules with vendor:product + port path | 3 |
| 5.1.2 | nrfutil ARM64 availability | Use binary if available; document Python fallback | 8.1 |
| 8.1.1 | Sniffer stdout PCAP support | Validate during T03/T04 | 8.1 |
| 8.2.1 | Rotation mechanism | Clean (wrapper restart), expect <500ms gap | 8.2 |
| 8.2.2 | Rotation trigger | Pure time-based | 8.2 |
| 8.4.1 | tshark field names per DLT | Verify during T07; document in `docs/tshark-fields.md` | 8.4 |
| 8.4.2 | Parse-failure behavior | Increment counter; alert at 100 consecutive | 8.4 |
| 8.5.1 | Ingest bridge language | C++17 with libpqxx | 8.5 |
| 8.5.2 | One bridge per sniffer | One per sniffer | 8.5 |
| 8.5.3 | Cross-sniffer duplicates | Store all observations | 8.5 |
| 8.6.1 | Heartbeat source | sniffer_heartbeat table, 30s | 8.6 |
| 8.7.1 | Residency thresholds | As in `classify_residency` v1 | 8.7 |
| 8.7.2 | BLE MAC randomization | v1: track address_type, warn in UI | 8.7 |
| 8.8.1 | BLE dissection lib | Roll our own | 8.8 |
| 8.8.2 | Field scope | Per CONTRACT 8.8-A; CONNECT_REQ deferred | 8.8 |
| 8.9.1 | ML scope | Rule-based only in v1 | 8.9 |
| 8.10.1 | API endpoint set | Per CONTRACT 8.10-A | 8.10 |
| 8.11.1 | UI library | TailwindCSS + headlessui/vue | 8.11 |
| 8.11.2 | Grafana embedding | iframe, anonymous LAN-only | 8.11 |
| 11.1.1 | PCAP fixture source | Capture during T03/T04, anonymize | 11.1 |
| 12.3.1 | Backup destination | External USB if present; else skip | 12.3 |
| 6.1 (legacy) | Deployment model | Rootless Podman + Quadlet (resolved by Q1) | 12.5 |

**Decision resolution protocol:** When an agent implements a default, they must:
1. Add a code comment with the decision ID at the implementation site.
2. Append a one-line entry to `docs/decisions-log.md`: `2026-MM-DD | DECISION X.Y | <agent-id> | <default-used> | <link-to-commit>`.

---

## 17. Out of Scope for v1

### Excluded from the v1 Bluetooth module

These are excluded from v1 specifically. They may appear in later versions of the same module (v1.1+) or are deferred indefinitely.

* Bluetooth Classic (BR/EDR) deep dissection. Captured but not enriched.
* Location triangulation across multiple Bluetooth sniffers.
* IRK-based BLE address resolution.
* Mobile or remote-access UI.
* Multi-Pi deployment.
* Full sklearn-based ML.
* CONNECT_REQ and connection-event parsing.

Candidates for v1.1+ within the Bluetooth module.

### Reserved for future Tian'er modules

These are not Bluetooth concerns at all; they are listed here as a reminder that the v1 architecture must remain extensible. Implementing them is the work of future modules per the Scope and Roadmap section. The v1 agent must not implement them, but must not make architectural decisions that would block them.

* GPS / GNSS reception.
* ADS-B flight telemetry (1090 MHz).
* Wi-Fi probe and management frame capture.
* Cellular passive signaling (2G/4G/5G broadcast channels).
* ISM-band telemetry (433 / 868 / 915 MHz).
* Microwave oven and consumer-appliance RF leakage signatures.
* Broadcast FM and DAB radio metadata.
* LoRaWAN.
* AIS marine vessel tracking.
* Acoustic and ultrasonic monitoring.
* Any other propagating signal that an interested operator decides is worth listening for.


---

## 18. Glossary

| Term | Meaning |
|------|---------|
| BLE | Bluetooth Low Energy. |
| BR/EDR | Bluetooth Classic (Basic Rate / Enhanced Data Rate). |
| Continuous aggregate | TimescaleDB materialized view with incremental refresh. |
| Contract | A documented interface between components (CONTRACT N.N-X). |
| DLT | Data Link Type, a libpcap identifier for the link-layer format. |
| FIFO | First-in-first-out named pipe (Unix IPC primitive). |
| GATT | Generic Attribute Profile, the BLE service/characteristic layer. |
| Hypertable | TimescaleDB-managed partitioned table. |
| IRK | Identity Resolving Key, used to resolve BLE Resolvable Private Addresses. |
| MAC | Media Access Control address (Bluetooth equivalent: BD_ADDR). |
| PCAP | Packet Capture file format. |
| PDU | Protocol Data Unit. |
| RPA | Resolvable Private Address (rotating BLE address). |
| RSSI | Received Signal Strength Indicator. |
| TLV | Type-Length-Value encoding (used in BLE AdvData). |

---

## 19. Document Maintenance

* Update this document via PRs that touch both the relevant section and the changelog at the bottom.
* Resolved decisions move to `docs/adr/` as Architecture Decision Records.
* Contract changes follow the change policy in section 9.

## Changelog

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-06-06 | 0.1 | Jean + Claude | Initial draft from voice design session |
| 2026-06-06 | 0.2 | Jean + Claude | Expanded for autonomous agent development: repository structure, pinned tech stack, configuration, contracts, testing strategy, fixtures, deployment automation, end-to-end verification, per-task acceptance criteria |
| 2026-06-07 | 0.3 | Jean + Claude | Updated versions to current releases (Raspberry Pi OS Trixie, PostgreSQL 17, Python 3.13, Node.js 24 LTS); added academic-style references; added comprehensive Section 6 covering users, groups, capabilities, file system layout, PostgreSQL roles, systemd privilege drop, and a runtime contract that no service requires sudo |
| 2026-06-07 | 0.5 | Jean + Claude | Reframed as Tian'er Signal Intelligence Platform with a roadmap of future sensor modules (GPS, ADS-B, Wi-Fi, cellular, ISM band, FM/DAB, LoRaWAN, AIS, acoustic, RF leakage); v1 is now explicitly scoped to the Bluetooth sensor module on shared platform infrastructure. Project-level identifier `tianer`; v1 module identifier remains `blesniff` for Bluetooth-specific components. Cross-module architectural commitments documented so v1 does not block future modules. |

---

## References

References use a Vancouver-style numeric citation format. All URLs were verified on 2026-06-07. Inline citations appear as superscript footnote markers throughout the document.

[^rpi-trixie]: Raspberry Pi Ltd. "Trixie - the new version of Raspberry Pi OS." Raspberry Pi Blog. October 2025. https://www.raspberrypi.com/news/trixie-the-new-version-of-raspberry-pi-os/

[^debian-trixie-release]: The Debian Project. "Debian 13 'trixie' released." Debian News. August 9, 2025. https://www.debian.org/News/2025/20250809

[^debian-pg17]: The Debian Project. "Package: postgresql-client-17 (trixie)." Debian Packages. https://packages.debian.org/trixie/postgresql-client-17

[^debian-py313]: The Debian Project. "Package: python3 (trixie) - depends on Debian's default Python 3 version (currently v3.13)." Debian Packages. https://packages.debian.org/trixie/python3

[^pg-support]: PostgreSQL Global Development Group. "Versioning Policy: each major version supported for 5 years from initial release." PostgreSQL.org. https://www.postgresql.org/support/versioning/

[^pg184]: PostgreSQL Global Development Group. "PostgreSQL 18.4, 17.10, 16.14, 15.18, and 14.23 Released!" PostgreSQL News. May 14, 2026. https://www.postgresql.org/about/news/postgresql-184-1710-1614-1518-and-1423-released-3297/

[^pg19-beta]: PostgreSQL Global Development Group. "PostgreSQL 19 Beta 1 Released!" PostgreSQL Release Notes. June 4, 2026. https://www.postgresql.org/docs/release/

[^ts-pg18]: Timescale Inc. "TimescaleDB CHANGELOG: TimescaleDB v2.23 introduces full PostgreSQL 18 support; available for PostgreSQL 15, 16, 17, and 18." GitHub. https://github.com/timescale/timescaledb/blob/main/CHANGELOG.md

[^ts-install]: Timescale Inc. "Upgrade PostgreSQL with TimescaleDB: PostgreSQL 15 support deprecated, removal June 2026; recommended PG 16+." Tiger Data Documentation. https://www.tigerdata.com/docs/deploy/self-hosted/upgrades/upgrade-pg

[^py3143]: Python Software Foundation. "The Latest Version of Python: Python 3.13.12, released February 3, 2026." phoenixNAP Knowledge Base. https://phoenixnap.com/kb/latest-python-version

[^py31312]: Python Software Foundation. "Python Release Python 3.13.12." Python.org. February 3, 2026. https://www.python.org/downloads/release/python-31312/

[^libpqxx]: libpqxx contributors. "libpqxx - Official C++ client API for PostgreSQL." GitHub. https://github.com/jtv/libpqxx

[^ubertooth-getting-started]: Great Scott Gadgets. "Getting Started - Ubertooth documentation; udev rule format: ACTION==\"add\" BUS==\"usb\" SYSFS{idVendor}==\"1d50\" SYSFS{idProduct}==\"6002\" GROUP:=\"plugdev\" MODE:=\"0660\". User must be member of plugdev group." Read the Docs. https://ubertooth.readthedocs.io/en/latest/getting_started.html

[^ubertooth-btle-output]: Great Scott Gadgets. "Capturing BLE in Wireshark - ubertooth-btle PCAP output via pipe or file." Ubertooth Documentation. https://ubertooth.readthedocs.io/en/latest/capturing_BLE_Wireshark.html

[^ubertooth-pcap]: Great Scott Gadgets. "Bluetooth Captures in PCAP - DLT allocation for BLE link types (DLT_BLUETOOTH_LE_LL_WITH_PHDR)." Ubertooth Documentation. https://ubertooth.readthedocs.io/en/latest/captures_pcap.html

[^nrfutil]: Nordic Semiconductor ASA. "ble-sniffer command quick guide." Nordic Semiconductor Technical Documentation. https://docs.nordicsemi.com/bundle/nrfutil/page/nrfutil-ble-sniffer/nrfutil-ble-sniffer.html

[^nrf-sniffer-guide]: Nordic Semiconductor ASA. "nRF Sniffer for Bluetooth LE User Guide v4.0.0." 2026. https://docs.nordicsemi.com/bundle/nrfutil_ble_sniffer_pdf/resource/nRF_Sniffer_BLE_UG_v4.0.0.pdf

[^node-lts]: OpenJS Foundation. "Node.js Release Working Group: Node.js 24.x is Active LTS (Krypton) until October 20, 2026; subsequent maintenance until April 30, 2028." GitHub. https://github.com/nodejs/release

[^wireshark-privileges]: Wireshark Foundation. "CaptureSetup/CapturePrivileges: Linux distributions provide a 'wireshark' group that limits dumpcap invocation. Add non-root users to that group to allow live capture." Wireshark Wiki. https://wiki.wireshark.org/CaptureSetup/CapturePrivileges

[^wireshark-blog]: Combs G. "Running Wireshark as You: dumpcap needs CAP_NET_RAW and CAP_NET_ADMIN via setcap." The Official Wireshark Blog. https://blog.wireshark.org/2010/02/running-wireshark-as-you/

[^wireshark-capture-privileges]: Wireshark Foundation. "Wireshark packaging notes: --enable-setcap-install installs dumpcap with cap_net_admin and cap_net_raw capabilities (Linux only); --with-dumpcap-group restricts dumpcap execution to the specified group." Wireshark source documentation.

[^udev-rules]: systemd / udev documentation. "udev rule syntax for hot-plugged USB devices: SUBSYSTEM, ATTRS{idVendor}, ATTRS{idProduct}, GROUP, MODE, TAG+=\"uaccess\"." Manual page udev(7). https://www.freedesktop.org/software/systemd/man/latest/udev.html

[^usermod-man]: Linux man-pages project. "usermod(8) - modify a user account. The -a (append) flag is required with -G to preserve existing supplementary groups." https://man7.org/linux/man-pages/man8/usermod.8.html

[^setcap-man]: Linux man-pages project. "setcap(8) - set file capabilities. The +eip flag set grants effective, inherited, and permitted bits." https://man7.org/linux/man-pages/man8/setcap.8.html

[^tmpfiles-d]: systemd documentation. "tmpfiles.d(5) - configuration for creation, deletion and cleaning of volatile and temporary files. Types include 'd' (directory) and 'p' (FIFO)." https://www.freedesktop.org/software/systemd/man/latest/tmpfiles.d.html

[^systemd-exec-user]: systemd documentation. "systemd.exec(5) - User= and Group= set the UNIX user/group the processes are executed as; SupplementaryGroups= sets supplementary Unix groups." https://www.freedesktop.org/software/systemd/man/latest/systemd.exec.html

[^systemd-exec-hardening]: systemd documentation. "systemd.exec(5) - sandboxing directives: NoNewPrivileges=, ProtectSystem=, ProtectHome=, PrivateTmp=, ReadWritePaths=." https://www.freedesktop.org/software/systemd/man/latest/systemd.exec.html

[^polkit-systemd]: freedesktop.org. "polkit(8) - Authorization Manager. systemd actions (org.freedesktop.systemd1.manage-units) can be authorized via polkit rules without sudo." https://www.freedesktop.org/software/polkit/docs/latest/

[^ble-core-spec]: Bluetooth SIG. "Bluetooth Core Specification version 5.4." Bluetooth Special Interest Group. https://www.bluetooth.com/specifications/specs/core-specification/

[^libpcap]: The Tcpdump Group. "libpcap - A portable C/C++ library for network traffic capture." https://www.tcpdump.org/

[^tshark-man]: Wireshark Foundation. "tshark(1) - Dump and analyze network traffic. The -l flag flushes the standard output after each packet is printed. The -T fields output format with -E separator produces structured per-packet output." https://www.wireshark.org/docs/man-pages/tshark.html
