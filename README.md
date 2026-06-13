# 天耳 (Tian'er) — Signal Intelligence Platform

## About

The project is named **天耳**, pronounced *Tian'er*, "Heavenly Ear." The name is a deliberate companion to **天眼** (*Tianyan*, "Heavenly Eye"), the spirit of which inspires this work. Tianyan looks outward at the universe through one giant aperture. Tian'er listens inward, to the sky directly overhead and the room directly around it, to anything that propagates through the air. The ambition is the same: build the instrument, point it at reality, see what is actually there.

Tian'er is a platform, not a single sensor. Wireless emissions are the first frontier because they are the easiest and the densest layer of ambient information around a modern household. But the architectural choice is deliberate: the same data pipeline that handles Bluetooth advertisements is designed to eventually carry GPS observations, ADS-B flight telemetry, broadcast radio metadata, Wi-Fi probe requests, cellular passive signaling, ISM-band telemetry from sensors and appliances, microwave oven leakage signatures, and whatever else turns out to be worth listening for. Under the sky, anything that propagates can find a home in this project.

The deeper inspiration is philosophical, not technical. There is a particular kind of innovation that emerges only under constraint, when a builder is told that a given system is closed to them. Denied the established path, the builder does not wait for permission; they construct their own, and they often construct it faster, more deliberately, and ultimately more capable than the original. The principle generalizes downward as well as upward: when you are denied the right to live inside someone else's reality, you build your own; and if you build with care and patience, the one you build is the better one.

Tian'er is the personal-scale expression of that idea. The available raw material is one small computer, a handful of hobbyist sensors, a deep stack of free software that the world's open-source community has already built, the time to learn what is needed, and an unreasonable enthusiasm for the work. The bet of this project is that those ingredients, combined with deliberate architectural design and patient iteration, are sufficient to produce a real instrument: not a demonstration, not a toy, but a tool that yields useful intelligence about the propagating environment it inhabits and that does so reliably enough to depend on.

## Status

| Component | Document | Status |
|-----------|----------|--------|
| Platform Infrastructure | [c01-platform-infrastructure.md](docs/designs/c01-platform-infrastructure.md) | ✅ Reviewed |
| Database | [c02-database.md](docs/designs/c02-database.md) | ⏳ Awaiting review |
| Capture Pipeline | [c03-capture-pipeline.md](docs/designs/c03-capture-pipeline.md) | ⏳ Awaiting review |
| PCAP Rotation | [c04-pcap-rotation.md](docs/designs/c04-pcap-rotation.md) | ⏳ Awaiting review |
| Ingest Bridge | [c05-ingest-bridge.md](docs/designs/c05-ingest-bridge.md) | ⏳ Awaiting review |
| Gap Detector | [c06-gap-detector.md](docs/designs/c06-gap-detector.md) | ⏳ Awaiting review |
| Deep Parser | [c07-deep-parser.md](docs/designs/c07-deep-parser.md) | ⏳ Awaiting review |
| ML Enrichment | [c08-ml-enrichment.md](docs/designs/c08-ml-enrichment.md) | ⏳ Awaiting review |
| REST API | [c09-rest-api.md](docs/designs/c09-rest-api.md) | ⏳ Awaiting review |
| Frontend | [c10-frontend.md](docs/designs/c10-frontend.md) | ⏳ Awaiting review |
| Grafana Dashboards | [c11-grafana-dashboards.md](docs/designs/c11-grafana-dashboards.md) | ⏳ Awaiting review |
| Service Orchestration | [c12-service-orchestration.md](docs/designs/c12-service-orchestration.md) | ⏳ Awaiting review |
| Observability | [c13-observability.md](docs/designs/c13-observability.md) | ⏳ Awaiting review |
| Deployment Automation | [c14-deployment-automation.md](docs/designs/c14-deployment-automation.md) | ⏳ Awaiting review |
| Inception Spec | [inception.md](docs/designs/inception.md) | ✅ Reviewed |
| Component Breakdown | [component-breakdown.md](docs/designs/component-breakdown.md) | ⏳ Awaiting review |
| Storage Strategy | [storage-strategy.md](docs/designs/storage-strategy.md) | ⏳ Awaiting review |
| Phase A Decisions | [ADR-0001](docs/adr/0001-phase-a-decisions.md) | ✅ Resolved |

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Target hardware | Raspberry Pi Compute Module 5, 8 GB RAM |
| OS | Raspberry Pi OS 64-bit (Trixie, Debian 13) |
| Database | PostgreSQL 17 + TimescaleDB 2.23+ |
| C++ | g++ 14.2, CMake 3.25+, GoogleTest 1.14, libpqxx 7.8, libpcap 1.10 |
| Python | CPython 3.13, uv, pytest 8.0+, FastAPI 0.110+ |
| Frontend | Node.js 24 LTS, Vue 3.4+, Pinia 2.1+, TypeScript 5.3+ (strict), Vite 5.0+ |
| Dashboards | Grafana 10.4+ |
| Process mgr | systemd 257 |
| Containers | rootless Podman + Quadlet |
| Sniffers | Ubertooth One, Nordic nRF52840 Dongle |

See [AGENTS.md](AGENTS.md) for the full pinned version table.

## Architecture

```
Sniffer → splitter → PCAP file (rotating)
                 └→ FIFO → tshark → ingest bridge → PostgreSQL/TimescaleDB
                                                           │
                                            gap detector ←─┘
                                            deep parser → ML enrichment → device_enrichment
                                                           │
                                        Grafana ←──────────┘
                                        FastAPI + Vue.js ←─┘
```

- **Sniffer wrappers** dual-write to rotating PCAP files and named pipes (FIFOs)
- **tshark** reads from FIFOs and outputs parsed fields to stdout
- **Ingest bridge** (C++17) batches and writes to PostgreSQL via COPY
- **Gap detector** (Python) monitors for data gaps and backfills from PCAP files
- **Deep parser** (C++17) performs full BLE advertisement dissection on rotated PCAP files
- **ML enrichment** (Python) classifies devices and enriches records
- **FastAPI** serves data via REST API with SSE for live updates
- **Vue.js** frontend provides real-time dashboard with Grafana iframe embeds
- **Grafana** provides observability dashboards and historical analytics

## Documentation

| Document | Description |
|----------|-------------|
| [inception.md](docs/designs/inception.md) | Original v0.5 inception spec — goals, hardware, architecture, contracts |
| [component-breakdown.md](docs/designs/component-breakdown.md) | Phase A synthesis: review findings, component breakdown, sequencing |
| [c01–c14](docs/designs/) | Per-component design documents (14 total) |
| [storage-strategy.md](docs/designs/storage-strategy.md) | Container persistent storage: volume inventory, access matrix, pod topology |
| [ADR-0001](docs/adr/0001-phase-a-decisions.md) | All 24 Phase A architectural decisions |

## Collaborate

Tian'er is a hobby signal intelligence platform built for learning, tinkering, and discovery. If any of these interest you, come join in:

- **Linux** — systemd, Podman, udev, capabilities, rootless containers on ARM
- **C++** — embedded systems, BLE protocol parsing, PCAP handling, libpqxx
- **PostgreSQL** — TimescaleDB hypertables, continuous aggregates, compression policies
- **TypeScript & Vue.js** — real-time dashboards, SSE, reactive state management
- **Python** — FastAPI, async database drivers, ML enrichment pipelines
- **Bluetooth & Wireless** — BLE advertising, BR/EDR survey, nRF52840, Ubertooth One
- **SDR & Amateur Radio** — RTL-SDR, HackRF, signal processing, RF analysis
- **Real-time Analytics** — streaming data pipelines, time-series databases, live dashboards

The first Bluetooth sub-project is in progress: [april-brother-at-absniffer](https://github.com/jeanboutros/april-brother-at-absniffer)

This project's agentic workflow is powered by [PSC](https://github.com/jeanboutros/psc).

**Pull requests, issues, and ideas welcome.** Let's make this a fun project together.

## License

MIT