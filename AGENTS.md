# AGENTS.md — Project Configuration

## Project Identity

| Field | Value |
|-------|-------|
| Name | 天耳 (Tian'er) — Signal Intelligence Platform |
| Repository | tian-er |
| License | MIT |

---

## Project Principles

### Engineering for Failure

**This project designs for failure, recovery, and observability from the first commit.**

Tian'er is an instrument that must be reliable enough to depend on — not a demo, not a toy. Every component is designed as if it will fail in production, because it will. These principles are non-negotiable and apply to every design document, every code review, and every deployment.

1. **Design for Failure.** Every component must document what fails, how it's detected, and how it recovers. No component may have an undocumented failure mode. If you are implementing a component, you must write the Failure Modes & Recovery section before you write the happy-path code.

2. **Recovery by Design.** Every data path must have a documented recovery mechanism. If data can be lost, the loss must be detectable and quantifiable. "No data loss" must be precisely defined — where loss is possible, the acceptable window and detection mechanism must be stated in the design document.

3. **Redundant Detection.** Critical failure conditions must be detectable by at least two independent mechanisms. If the primary detector fails, a secondary must exist or an alert must fire immediately. No single point of recovery.

4. **Graceful Degradation.** When a component fails, the system degrades gracefully — partial function is better than total function. One sniffer crashing must not take down the other three. The API being down must not prevent Grafana from reading the database directly.

5. **Observability by Design.** Every failure mode must produce an observable signal. Silent failures are the highest severity class — worse than a crash, because a crash is detectable. Every component exposes health metrics, structured logs, and failure counters. If a recovery-critical service (e.g., gap detector) is down, an alert must fire immediately.

6. **Blast Radius Minimization.** A compromise or failure in one component must not automatically cascade to others. Per-service sandboxing, least-privilege access, and network isolation enforce blast radius boundaries.

7. **Defense in Depth.** Security controls at every layer, not just the perimeter. TLS on the API, API key authentication, parameterized queries, input validation, file permissions, database access restrictions, and systemd sandboxing — all must be present, not any single one.

8. **Crash-Only Design.** Components must be safe to kill at any time. Recovery on restart must be automatic and correct. No component may require a "clean shutdown" for data integrity — if it does, the design is wrong.

9. **Airplane-Grade Reliability Post-MVP.** The MVP delivers a working pipeline with basic failure detection and restart. Post-MVP, we add redundant detection, alerting, structured logging, TLS, secrets rotation, disk encryption, and the full observability stack. The MVP is the first flight; post-MVP is the airline certification.

10. **Quantified SLA.** Every data integrity claim is quantified. "No data loss" means PCAP files are the source of truth. DB ingest latency is measured. Gap recovery windows are timed. Performance targets have data volumes attached. If you cannot measure it, it does not exist.

---

## Design Document Structure

All design documents in `doc/designs/` must contain these sections:

1. **Overview** — Purpose, scope, boundaries, position in the system
2. **High-Level Architecture (HLA)** — Component diagram, data flow, neighbouring components
3. **Data Model (ERD)** — Entities, relationships, fields, constraints (where applicable)
4. **Low-Level Architecture (LLA)** — Module decomposition, algorithms, error handling (where applicable)
5. **Inter-Component Contracts** — Input/output schemas, message formats
6. **Failure Modes & Recovery** — Failure catalog, detection, propagation, recovery, SLA
7. **Observability** — Metrics exposed, logging format, health checks, alert rules
8. **Security Considerations** — Attack surface, authentication, input validation, data sensitivity
9. **Configuration** — Environment variables, config files, defaults, tuning
10. **Test Plan** — Unit, integration, performance, acceptance criteria
11. **Deployment Notes** — Build steps, install paths, systemd units, dependencies

---

## Online Validation Rule

**Agents MUST verify facts, specifications, and conventions against their online sources before acting.** Do not rely on training data or memory for anything that has an authoritative URL.

Examples:
- Before writing a commit → check [conventionalcommits.org](https://www.conventionalcommits.org/en/v1.0.0/) for the current spec
- Before using an API → check the library's official docs
- Before citing a standard (WCAG, OWASP, RFC) → fetch the canonical page
- Before installing a package → verify it exists and check the current version
- Before claiming a hardware register value → verify against the datasheet

If a source is unreachable, state that explicitly and ask the user rather than guessing from memory.

### Context7 — Library Documentation Lookup

When agents need library or API documentation, they SHOULD use [Context7](https://github.com/upstash/context7) to fetch up-to-date, version-specific docs directly into the prompt. This prevents hallucinated APIs, outdated code examples, and generic answers based on stale training data.

**Setup (one-time, requires Node.js 18+):**

```bash
npx ctx7 setup
```

**Available tools:**

| Mode | Command / Tool | Purpose |
|------|---------------|---------|
| CLI | `ctx7 library <name> <query>` | Search the Context7 index for a library and get its ID |
| CLI | `ctx7 docs <libraryId> <query>` | Fetch documentation for a library by its Context7 ID |
| MCP | `resolve-library-id` | Resolve a library name into a Context7-compatible ID |
| MCP | `query-docs` | Retrieve documentation by library ID (e.g. `/vercel/next.js`) |

**Rule for all agents:** Always use Context7 when needing library/API documentation, code generation, setup, or configuration steps — without the user having to explicitly ask. If Context7 is not available, fall back to fetching the library's official documentation URL directly.

---

### Cross-Document Consistency Rule

**When any document is updated, all affected documents must be re-visited for consistency.** No document may carry a "staleness banner" or disclaimer pointing readers to "newer" documents. If a document is out of date, it must be updated — not flagged as stale. A stale document is a defect, not a feature.

This means:
- After any design decision is resolved, all documents referencing that decision must be updated immediately.
- After any code change, affected design documents, runbooks, and README must be checked for consistency.
- No document may describe a state that contradicts another document. Contradiction = bug.
- The definitive source of truth is the latest commit on `main`. Every document in that commit must be internally and mutually consistent.

**Enforcement:** The Documentation-Update Rule (below) already requires agents to update affected documentation after every change. This rule adds the cross-document consistency check: after updating one document, the agent must also check all other documents that reference the same concept and bring them into alignment.

---

## Commit Rules

### Conventional Commits v1.0.0 (Mandatory)

Every commit MUST follow the [Conventional Commits v1.0.0](https://www.conventionalcommits.org/en/v1.0.0/) specification.

```
<type>[optional scope]: <description>

[mandatory body]

[optional footer(s)]
```

### Types

| Type | When to Use |
|------|-------------|
| `feat` | New feature — component, service, script, or capability |
| `fix` | Bug fix in an existing file |
| `docs` | Documentation-only changes (README, design docs, AGENTS.md) |
| `refactor` | Code restructuring with no behaviour change |
| `chore` | Maintenance (dependency updates, file moves, counter resets) |
| `style` | Formatting, whitespace — no logic change |
| `test` | Adding or updating tests |
| `build` | Build system or external dependency changes |
| `ci` | CI configuration changes |
| `perf` | Performance improvement with no functional change |
| `revert` | Reverts a previous commit (body SHOULD reference the reverted SHA) |

### Scope (Optional)

Use the component or directory name:

- `feat(c05-ingest-bridge): add reconnection logic to PgWriter`
- `fix(c03-capture-pipeline): parameterize tshark fields per DLT type`
- `docs(c02-database): add ERD for raw_packets hypertable`
- `refactor(c09-rest-api): extract auth middleware`

### Body (Mandatory)

Every commit MUST include a body that describes:
1. **What** changed
2. **Why** it changed

### Commit Granularity

- **One file per commit** is the default.
- **Bundle only when tightly coupled** — files that are part of the same logical change.
- **Never bundle unrelated changes.**
- **When in doubt, split.**

---

## Documentation-Update Rule

**After any change to the project, agents MUST update all affected documentation before considering the task complete.**

This includes:
- `doc/designs/` design documents — update when components change
- `AGENTS.md` — update when project configuration changes
- `README.md` — update when project status changes
- `docs/adr/` — add ADR when a decision is resolved
- `docs/runbooks/` — update when operational procedures change
- **Agent permission blocks** — validate `edit`/`bash` against agent role per the Permission Validation Rule below

### Post-Change Verification Checklist

After every change, before considering the task complete, the agent MUST run this checklist:

- [ ] **AGENTS.md** — Did I add or remove a skill? Update the Skill Registry. Did I create or change an agent? Run the Permission Validation Rule check.
- [ ] **docs/pipeline.md** — Did I change a gate, tier, dispatch rule, or enforcement protocol? Update this doc.
- [ ] **.opencode/skills/pipeline/SKILL.md** — Did I add or change an agent intent? Update the routing table. Did pipeline steps change? Update the passport template reference.
- [ ] **.opencode/skills/pipeline-passport/SKILL.md** — Did I add or remove a pipeline step? Update the Required Steps template.
- [ ] **README.md** — Did I create, delete, or rename a file? Update the Review & Test Status tables.
- [ ] **Agent `permission:` block** — Did I create or change an agent? Validate `edit`/`bash` against the Permission Validation Rule. This is not optional — dispatch-only agents with `edit: allow` are a structural defect.
- [ ] **Cross-document consistency** — Did I change a concept, name, path, or version that appears in another document? Search all docs and update them. If I found a stale reference I couldn't fix, that's a bug — flag it rather than adding a staleness banner.

If a checklist item doesn't apply, mark it `N/A`. If you skip an item without justification, the change is incomplete.

---

## Agent Permission Validation Rule

**Every agent's `permission:` block must match its declared role. Mismatches are a structural defect.**

When creating or modifying an agent file, validate these constraints:

| If the agent's Role says... | Then `permission.edit` must be... | And `permission.bash` must be... |
|-----------------------------|-----------------------------------|----------------------------------|
| Dispatch-only / orchestrator / "never executes work" / "never writes code" | `deny` | `deny` |
| Read-only reviewer (does not produce code) | `deny` | `allow` (for building/documenting) |
| Task management only (PM) | `allow` (only for management files) | `allow` |
| Code-producing agent (Code Architect, UI Engineer) | `allow` | `allow` |
| Security reviewer / test engineer | `allow` | `allow` |

**Default denial for dispatch-only agents:** Any agent with "DISPATCH-ONLY" in its role description, or whose constraints say "Can edit code: No", MUST have `permission.edit: deny` and `permission.bash: deny` in its YAML frontmatter.

**The check:** After any agent file change, verify:
1. The permission block is present in the YAML frontmatter
2. `edit` matches the agent's declared capabilities (if "never writes code" → `deny`)
3. `bash` matches the agent's declared capabilities (if "coordination only" → `deny`)
4. The `Constraints` section in the body does not contradict the YAML permissions

This rule exists because advisory text ("I should not write code") is insufficient — the YAML permission block is the runtime enforcement. A dispatch-only orchestrator with `edit: allow` is a breach vector for bypassing the pipeline.

---

## Naming Conventions

| Context | Convention | Example |
|---------|-----------|---------|
| Project namespace | `tianer` | Database `tianer`, Python packages `tianer_*` |
| v1 module namespace | `blesniff` | `blesniff-ingest` binary, `blesniff-sniffer@.service` |
| Shared environment variables | `TIANER_*` | `TIANER_DB_HOST`, `TIANER_API_PORT` |
| Module environment variables | `BLESNIFF_*` | `BLESNIFF_ROTATION_MINUTES`, `BLESNIFF_PCAP_DIR` |
| Database | `tianer` | `CREATE DATABASE tianer` |
| DB roles | `tianer`, `tianer_ro`, `tianer_grafana` | Module-specific table ownership in `bluetooth` schema |
| systemd targets | `tianer.target` (shared), `blesniff.target` (module) | |
| Device symlinks | `/dev/tianer/ubertooth0`, `/dev/tianer/nrf0` | udev rules |
| Design documents | `c##-component-name.md` | `c05-ingest-bridge.md` |

---

## Tech Stack

| Component | Value |
|-----------|-------|
| Target hardware | Raspberry Pi Compute Module 5, 8 GB RAM |
| OS | Raspberry Pi OS 64-bit (Trixie, Debian 13) |
| Database | PostgreSQL 17 + TimescaleDB 2.23+ |
| C++ compiler | g++ 14.2 (C++17 baseline, C++20 features allowed) |
| C++ build | CMake 3.25+ |
| C++ test | GoogleTest 1.14 |
| C++ Postgres | libpqxx 7.8 |
| C++ PCAP | libpcap 1.10 |
| C++ JSON | nlohmann/json 3.11+ |
| Python | CPython 3.13 (Trixie default) |
| Python pkg mgr | uv 0.4+ |
| Python test | pytest 8.0+ |
| Async PG | psycopg[binary] 3.1 |
| FastAPI | 0.110+ |
| Wireshark | tshark 4.2+ |
| Ubertooth | ubertooth-tools (pinned to firmware release tag) |
| Nordic | nrfutil 7.x with ble-sniffer subcommand |
| Node.js | 24 LTS |
| Frontend build | Vite 5.0+ |
| Frontend | Vue 3.4+ |
| Frontend state | Pinia 2.1+ |
| TypeScript | tsc 5.3+ (strict mode) |
| Frontend test | Vitest 1.2+ |
| Dashboards | Grafana 10.4+ |
| Process mgr | systemd 257 (Trixie default) |
| Compression | zstd 1.5+ |
| Pre-commit | pre-commit 3.6+ |
| Linter (C++) | clang-tidy 19 |
| Formatter (C++) | clang-format 19 |
| Linter (Python) | ruff 0.3+ |
| Type check (Python) | mypy 1.8+ (strict) |
| Shell test | bats-core + kcov |

---

## Design Documents

All design documents are in `doc/designs/`. See `doc/designs/component-breakdown.md` for the full component inventory, dependency map, and per-component artifact requirements.

| Document | Component |
|----------|-----------|
| `inception.md` | Original v0.5 inception spec |
| `component-breakdown.md` | Phase A synthesis: review findings, component breakdown, sequencing, artifacts |
| `c01-platform-infrastructure.md` | OS, users, groups, capabilities, udev, filesystem, config |
| `c02-database.md` | PostgreSQL + TimescaleDB schema, roles, migrations |
| `c03-capture-pipeline.md` | Sniffer wrappers, FIFO, tshark, heartbeat |
| `c04-pcap-rotation.md` | File rotation, compression, retention |
| `c05-ingest-bridge.md` | C++ ingest pipeline (parser, batcher, writer) |
| `c06-gap-detector.md` | Python gap detection and backfill |
| `c07-deep-parser.md` | C++ BLE dissector, AdvData parser, JSONL output |
| `c08-ml-enrichment.md` | Python rule-based device classification |
| `c09-rest-api.md` | FastAPI backend, auth, endpoints |
| `c10-frontend.md` | Vue 3 + TypeScript web UI |
| `c11-grafana-dashboards.md` | Grafana provisioning and dashboards |
| `c12-service-orchestration.md` | systemd units, dependency chain, Makefile |
| `c13-observability.md` | Metrics catalogue, structured logging, alerts |
| `c14-deployment-automation.md` | setup.sh, build targets, install paths, rollback |
| `storage-strategy.md` | Container persistent storage: volume inventory, access matrix, container image policy |

---

## Skill Registry

### Core Skills (Always Loaded)

| Skill | Purpose |
|-------|---------|
| assumption-trap | Halt on ambiguity — never guess |
| brainstorming | Phase A creative exploration |
| clarification-table | Batch blocked questions into interactive HTML table for user to answer |
| compliance-gate | Tiered gate system (T1/T2/T3/T-ARCH) |
| context7-docs | Fetch up-to-date library/API docs |
| cross-document-consistency | Grep-based cross-document checks (DC-1 through DC-4) |
| datasheet-verification | Verify claims against source documents |
| doxygen-cpp | Doxygen documentation standard for C/C++ projects |
| flag-protocol | Structured request format for non-PM agents |
| grill-me | Adversarial design review |
| incremental-execution | Unit-by-unit implementation |
| memory-safety | Memory safety review (C/C++ projects) |
| pau-loop | Plan → Apply → Validate loop |
| pipeline | Full pipeline state machine |
| pipeline-passport | Task tracking card |
| post-rejection-correction | Root-cause analysis before retry |
| review-confidence | 0-100 confidence scoring |
| self-audit-checklist | Mandatory pre-verdict checklist |
| silent-failure | Detect silent failure modes |
| systematic-debugging | Structured debugging protocol |
| tdd-cpp | Test-driven development for C++ |
| test-driven-development | Generic TDD loop |
| type-design-review | Type system and API review |
| verification-before-completion | Final verification before marking done |

### Domain Skills (Project-Specific)

| Skill | Purpose |
|-------|---------|
| ble-protocol | BLE protocol compliance — advertising channels, PDU types, data whitening, CRC-24 |
| cpp-embedded | Embedded C++ patterns — enum class mandates, register struct design |
| nrf52840-sniffer | nRF52840 dongle as BLE sniffer — firmware, DLT types, field names |
| ubertooth | Ubertooth One platform — firmware version matching, PCAP output, channel selection |