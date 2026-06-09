# C14 — Deployment Automation

**Status:** Phase A Design — guides Phase B implementation  
**Author:** Code Architect  
**Date:** 2026-06-09  
**Dependencies:** C01 (Platform Infrastructure) — C14 ships the scripts that C01's design specifies  
**Blocks:** None — C14 is a Layer 0 scaffold that grows alongside all other components

---

## 1. Overview

### 1.1 Purpose

C14 Deployment Automation is the **single-entry-point delivery system** for the entire Tian'er Signal Intelligence Platform. It provides:

- **`setup.sh`** — An idempotent, phase-based host bootstrap script that provisions the Raspberry Pi CM5 from a fresh OS image to a fully operational platform.
- `Makefile` — A top-level build, test, deploy, verify, backup, and restore orchestrator using `.PHONY` targets and pattern rules [1].
- **Containerfiles** — Per-component multi-stage container image build definitions.
- **Quadlet deployment** — systemd-integrated Podman container orchestration with automatic startup at boot.

C14 is the **mechanism**, not the policy. C01 (Platform Infrastructure) defines *what* must exist on the host; C14 defines *how* to get it there. C12 (Service Orchestration) defines *what* the running system looks like; C14 defines *how* to build and deploy it.

### 1.2 Scope

| In Scope | Out of Scope |
|----------|-------------|
| `setup.sh` — 16-phase idempotent host bootstrap | Per-component application logic |
| `Makefile` — build, test, deploy, verify, backup, restore targets | Container orchestration logic at runtime (C12) |
| Per-component `Containerfile` definitions (multi-stage builds) | Volume access matrix definitions (C01, storage-strategy.md) |
| Quadlet `.container`, `.pod`, `.volume`, `.network` file deployment | systemd unit definitions for application services (C12) |
| Supply chain integrity: GPG verification, SHA256 checksums, digest pinning, hash-pinned Python, `npm audit` | Secrets generation policy (C01) |
| Version manifest and deploy state file | Database migration content (C02) |
| Backup and restore procedures (`pg_basebackup`, volume snapshot) | Continuous backup scheduling (C12 timer) |
| Deployment logging and audit trail | Runtime metrics collection (C13) |
| Rollback procedure for failed deployments | Data migration rollback (C02) |

### 1.3 Boundaries

C14 operates at the **host level** (as `root` for setup, as `tianer` for builds). It produces:

- **On the host:** Installed packages, created users/groups/directories, deployed Quadlet files, applied DB migrations
- **On the host filesystem:** Container images in local Podman storage
- **In the deploy state:** A machine-readable JSON file tracking which phases completed, what versions were deployed, and what digests were verified

C14 is the **single entry point**. Every deployment operation — whether initial setup, upgrade, rollback, backup, or verification — is invoked through either `setup.sh` or `make`.

### 1.4 Position in the System

```
┌──────────────────────────────────────────────────────────────────────┐
│                          C14 DEPLOYMENT AUTOMATION                     │
│                                                                        │
│  setup.sh ──► Makefile ──► Containerfiles ──► Quadlet ──► systemd     │
│  (host)        (build)      (images)          (deploy)    (runtime)    │
└──────────────────────────────────────────────────────────────────────┘
        │              │               │               │
        ▼              ▼               ▼               ▼
   ┌─────────┐   ┌─────────┐   ┌──────────────┐  ┌──────────────┐
   │ C01 Host│   │ C05,C07 │   │ Containers    │  │ C12 systemd  │
   │ Boots.  │   │ Binaries│   │ in Podman     │  │ Orchestration│
   └─────────┘   └─────────┘   └──────────────┘  └──────────────┘
```

C14 sits at Layer 0 in the build sequence (component-breakdown.md §4.1), running alongside C01 and C02 as the scaffolding that enables all other components. It grows incrementally: the initial `setup.sh` phases bootstrap the host; later phases build and deploy individual components as they are implemented.

---

## 2. High-Level Architecture

### 2.1 `setup.sh` as Orchestrator

`deploy/setup.sh` is the master deployment script. It runs as `root` (via `sudo`) exactly once for initial deployment and is **safe to re-run** for remediation or upgrades. Every phase is independently idempotent.

```
setup.sh
  │
  ├── Phase  1: OS Verification
  ├── Phase  2: Package Installation (apt + GPG verification)
  ├── Phase  3: User and Group Creation
  ├── Phase  4: Filesystem Layout + Secrets
  ├── Phase  5: Hardware Access (Wireshark, udev)
  ├── Phase  6: Container Runtime (Podman + linger)
  ├── Phase  7: Quadlet Deployment
  ├── Phase  8: systemd daemon-reload
  ├── Phase  9: DB Migrations
  ├── Phase 10: Container Image Build (all components)
  ├── Phase 11: Container Services Enable
  ├── Phase 12: Container Services Start
  ├── Phase 13: Smoke Verification
  ├── Phase 14: Supply Chain Verification (post-deploy audit)
  ├── Phase 15: Backup Configuration
  └── Phase 16: Summary and Next Steps
```

**Idempotency guarantee:** Every phase checks whether its work is already done before acting. If `setup.sh` is interrupted at phase N and re-run, phases 1 through N-1 are no-ops, and phase N picks up where it left off (or retries safely).

**Phased execution:** `setup.sh` can be invoked with `--from-phase N` to start from a specific phase (for partial remediation). The `--dry-run` flag prints what each phase would do without executing.

**Exit codes:**
| Code | Meaning |
|------|---------|
| 0 | All phases completed successfully |
| 1 | Pre-flight check failed (wrong OS, architecture, or dependencies) |
| 2 | A phase failed (error message written to stderr; deploy state file records the failure) |
| 3 | Supply chain verification failed (GPG mismatch, checksum mismatch) |

### 2.2 Makefile Targets

The root `Makefile` is the developer's and operator's entry point for all build, test, deploy, verify, backup, and restore operations.

| Target | Purpose | Requires `root`? |
|--------|---------|-----------------|
| `make help` | List all targets with descriptions | No |
| `make setup` | Run `deploy/setup.sh` (full host bootstrap) | Yes |
| `make build` | Build all component container images | No (Podman rootless) |
| `make build-<component>` | Build a single component's image (e.g., `build-ingest`, `build-api`) | No |
| `make pull` | Pull pre-built container images from registry (digest-pinned) | No |
| `make install` | Install built artifacts (container images + Quadlet files) | Yes (for Quadlet files) |
| `make deploy` | `build` + `install` + `services-restart` | Yes |
| `make db-up` | Apply pending database migrations | No (needs `tianer` DB role) |
| `make db-reset` | Drop and recreate database (**DESTRUCTIVE**; requires `CONFIRM=yes`) | No |
| `make db-migrate-new` | Scaffold a new numbered migration file | No |
| `make test-unit` | Run all unit tests (C++, Python, TypeScript, bats) | No |
| `make test-integration` | Run all integration tests (requires test DB) | No |
| `make test-e2e` | Run end-to-end smoke test on the target | No |
| `make test-all` | Run unit + integration + e2e in sequence | No |
| `make lint` | Run all linters (clang-tidy, ruff, tsc, shellcheck) | No |
| `make format` | Apply formatters (clang-format, ruff format, prettier) | No |
| `make verify` | Run full supply chain audit (GPG keys, digests, hashes) | No |
| `make verify-version` | Check deployed versions against version manifest | No |
| `make verify-idempotency` | Run `setup.sh` twice and verify no changes on second run | Yes |
| `make backup` | Run database backup (`pg_basebackup` + config snapshot) | No |
| `make backup-pcap` | Backup PCAP files to external destination | No |
| `make backup-full` | `backup` + `backup-pcap` | No |
| `make restore` | Restore database from backup | Yes |
| `make restore-list` | List available backups with timestamps | No |
| `make restore-pcap` | Restore PCAP files from backup | Yes |
| `make services-start` | Start all containers via `systemctl --user start tianer.target` | No (as `tianer`) |
| `make services-stop` | Stop all containers | No (as `tianer`) |
| `make services-restart` | Restart all containers | No (as `tianer`) |
| `make services-status` | Show status of all containers | No |
| `make logs` | Tail logs from all containers (`journalctl` filtered) | No |
| `make logs-<component>` | Tail logs for a specific component | No |
| `make clean` | Remove build artifacts (compiled binaries, caches) | No |
| `make clean-all` | Remove container images + build artifacts (**DESTRUCTIVE**) | No |
| `make rollback` | Rollback to previous deployment version | Yes |
| `make rollback-list` | List available rollback versions | No |

### 2.3 Containerfile Build Chain

Every Tian'er component ships a `Containerfile` using multi-stage builds [2]. The build stage includes compilers, headers, and package managers. The runtime stage copies only the compiled artifact and its runtime dependencies — no shells, no package managers, no development headers.

```
Containerfile Build Chain (Layer Dependency):

  ┌───────────────────┐
  │ Containerfile.base │  ← Shared base: debian:trixie-slim + runtime libs
  └────────┬──────────┘
           │ FROM tianer-base:latest
           ▼
  ┌───────────────────┐
  │ Containerfile.post │  ← PostgreSQL 17 + TimescaleDB
  │ gres               │
  └───────────────────┘

  ┌───────────────────┐
  │ Containerfile      │  ← FROM tianer-base:latest AS builder
  │ .ingest            │    Multi-stage: build + runtime
  └───────────────────┘

  ┌───────────────────┐
  │ Containerfile      │  ← FROM tianer-base:latest AS builder
  │ .deep-parse        │    Multi-stage: build + runtime
  └───────────────────┘

  ┌───────────────────┐
  │ Containerfile      │  ← FROM debian:trixie-slim
  │ .tshark            │    tshark + dumpcap in capture pod
  └───────────────────┘

  ┌───────────────────┐
  │ Containerfile      │  ← FROM python:3.13-slim
  │ .gap-detector      │    uv-based install
  └───────────────────┘

  ┌───────────────────┐
  │ Containerfile      │  ← FROM python:3.13-slim
  │ .ml-classify       │    uv-based install
  └───────────────────┘

  ┌───────────────────┐
  │ Containerfile      │  ← FROM python:3.13-slim
  │ .api               │    uv-based install + frontend bundle
  └───────────────────┘

  ┌───────────────────┐
  │ Containerfile      │  ← FROM node:24-slim AS builder
  │ .frontend          │    Builds to /dist; copied into API image
  └───────────────────┘

  ┌───────────────────┐
  │ Containerfile      │  ← FROM grafana/grafana:10.4.x (official)
  │ .grafana           │    Copies provisioning + dashboards
  └───────────────────┘

  ┌───────────────────┐
  │ Containerfile      │  ← FROM debian:trixie-slim
  │ .rotate            │    rotate-pcap.sh + zstd + cron/logrotate
  └───────────────────┘

  ┌───────────────────┐
  │ Containerfile      │  ← FROM debian:trixie-slim
  │ .sniffer           │    ubertooth-tools or nrfutil (two variants)
  └───────────────────┘
```

**Build ordering:** `Containerfile.base` must be built first. C++ components (ingest, deep-parse) depend on it. Python components are independent. Frontend is built first, then its output is copied into the API image. Build order is encoded in the `Makefile` as dependencies:

```makefile
.PHONY: build
build: build-base build-ingest build-deep-parse build-tshark build-gap-detector \
       build-ml-classify build-frontend build-api build-grafana build-rotate build-sniffer
```

### 2.4 Quadlet systemd Integration

Quadlet is a systemd generator (part of Podman ≥ 4.4) that converts declarative `.container`, `.pod`, `.volume`, and `.network` files into `systemd --user` service units. C14 deploys Quadlet files to `/etc/containers/systemd/` during `setup.sh` Phase 7.

```
/etc/containers/systemd/
  ├── tianer-net.network              # Internal bridge network
  ├── tianer-postgres-data.volume      # Persistent PostgreSQL volume (V06)
  ├── tianer-grafana-data.volume       # Persistent Grafana volume (V07)
  ├── tianer-capture.pod              # Capture pod (Network=none)
  ├── tianer-platform.pod             # Platform pod (bridge: tianer-net)
  ├── tianer-postgres.container       # PostgreSQL standalone
  ├── tianer-grafana.container        # Grafana standalone
  ├── tianer-sniffer@.container       # Sniffer wrapper (template)
  ├── tianer-tshark@.container        # tshark reader (template)
  ├── tianer-ingest@.container        # Ingest bridge (template)
  ├── tianer-rotate.container         # PCAP rotation
  ├── tianer-gap-detect.container     # Gap detector
  ├── tianer-deep-parse.container     # Deep parser (batch)
  ├── tianer-ml-classify.container    # ML enrichment
  ├── tianer-api.container            # FastAPI backend
  └── tianer-heartbeat.container      # Heartbeat companion (per-sniffer)
```

After deployment, `systemctl daemon-reload` triggers Quadlet to generate `systemd --user` units, which are started at boot via `systemd-linger` for the `tianer` user.

**Container control commands:**

```bash
# As the tianer user (or root controlling the tianer user instance):
sudo machinectl shell tianer@ /bin/bash -c 'systemctl --user start tianer.target'
sudo machinectl shell tianer@ /bin/bash -c 'systemctl --user status tianer-capture-pod.service'
```

---

## 3. Data Model

### 3.1 Version Manifest

Every deployment records a version manifest — a machine-readable JSON file at `/etc/tianer/version-manifest.json` that captures exactly what was deployed.

```jsonc
{
  "schema_version": 1,
  "deployed_at": "2026-06-09T14:30:00Z",
  "deployed_by": "setup.sh",
  "setup_script_version": "1.0.0",
  "project_commit": "abc1234def5678",
  "project_branch": "main",
  "components": {
    "tianer-base": {
      "image": "localhost/tianer-base",
      "tag": "1.0.0",
      "digest": "sha256:9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08",
      "built_at": "2026-06-09T14:25:00Z",
      "size_mb": 78
    },
    "tianer-ingest": {
      "image": "localhost/tianer-ingest",
      "tag": "1.0.0",
      "digest": "sha256:...",
      "built_at": "2026-06-09T14:26:00Z",
      "size_mb": 34
    },
    "tianer-api": {
      "image": "localhost/tianer-api",
      "tag": "1.0.0",
      "digest": "sha256:...",
      "built_at": "2026-06-09T14:28:00Z",
      "size_mb": 85
    }
  },
  "apt_packages": {
    "postgresql-17": "17.6-1",
    "podman": "5.4.2",
    "tshark": "4.2.12"
  },
  "python_packages": {
    "fastapi": "0.110.3",
    "psycopg": "3.1.20"
  },
  "npm_packages": {
    "vue": "3.4.21",
    "vite": "5.0.13"
  },
  "gpg_keys_verified": [
    "ACCC4CF8 (PostgreSQL)",
    "0E22EB88E39E60E7 (Grafana)",
    "9FD3B784BC1C6FC31A8A0A1C1655A0AB68576280 (NodeSource)"
  ],
  "external_binaries": {
    "nrfutil": {
      "version": "7.12.0",
      "sha256": "d7a8fbb307d7809469ca9abcb0082e4f8d5651e46d3cdb762d02d0bf37c9e592"
    }
  },
  "supply_chain_verified": true
}
```

**Location:** `/etc/tianer/version-manifest.json` — mode `0640`, owned by `root:tianer`.

**Update frequency:** Written fresh on every successful `make deploy` or `setup.sh` Phase 10 (container image build). Previous version is backed up as `version-manifest.json.<ISO8601>.bak`.

### 3.2 Deploy State File

The deploy state file tracks which `setup.sh` phases have completed and the status of the current deployment.

```jsonc
{
  "schema_version": 1,
  "last_run_at": "2026-06-09T14:30:00Z",
  "phases": {
    "1_os_verify": {"status": "completed", "at": "2026-06-09T14:30:01Z"},
    "2_packages": {"status": "completed", "at": "2026-06-09T14:32:00Z"},
    "3_user_group": {"status": "completed", "at": "2026-06-09T14:32:05Z"},
    "4_filesystem": {"status": "completed", "at": "2026-06-09T14:32:10Z"},
    "5_hardware": {"status": "completed", "at": "2026-06-09T14:32:15Z"},
    "6_container_runtime": {"status": "completed", "at": "2026-06-09T14:32:20Z"},
    "7_quadlet": {"status": "completed", "at": "2026-06-09T14:32:25Z"},
    "8_daemon_reload": {"status": "completed", "at": "2026-06-09T14:32:26Z"},
    "9_db_migrations": {"status": "completed", "at": "2026-06-09T14:32:30Z"},
    "10_image_build": {"status": "completed", "at": "2026-06-09T14:35:00Z"},
    "11_services_enable": {"status": "completed", "at": "2026-06-09T14:35:05Z"},
    "12_services_start": {"status": "completed", "at": "2026-06-09T14:35:10Z"},
    "13_smoke": {"status": "completed", "at": "2026-06-09T14:35:30Z"},
    "14_supply_chain": {"status": "completed", "at": "2026-06-09T14:35:35Z"},
    "15_backup_config": {"status": "completed", "at": "2026-06-09T14:35:36Z"},
    "16_summary": {"status": "completed", "at": "2026-06-09T14:35:37Z"}
  },
  "failed_phase": null,
  "last_error": null
}
```

**Location:** `/etc/tianer/deploy-state.json` — mode `0640`, owned by `root:tianer`.

**Update frequency:** Written at the start of each phase (status `in_progress`) and on completion (status `completed` or `failed`). If `setup.sh` is interrupted, the state file shows which phase was in progress and whether any phase failed.

### 3.3 SHA256 Manifest

Prior to deployment, a SHA256 manifest is generated for all externally sourced binaries and container images. This manifest is checked against the deployed artifacts during supply chain verification (Phase 14).

```jsonc
{
  "schema_version": 1,
  "generated_at": "2026-06-09T14:24:00Z",
  "source": "deploy/manifests/sha256-manifest.json",
  "entries": {
    "deploy/bundled/nrfutil-v7.12.0-linux-arm64": {
      "expected_sha256": "d7a8fbb307d7809469ca9abcb0082e4f8d5651e46d3cdb762d02d0bf37c9e592",
      "source_url": "https://github.com/NordicSemiconductor/pc-nrfutil/releases/download/v7.12.0/nrfutil-linux-arm64",
      "verified": false
    },
    "apt/gpg/postgresql.asc": {
      "expected_sha256": "b1e3d8e4a9f1f2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7",
      "source_url": "https://www.postgresql.org/media/keys/ACCC4CF8.asc",
      "verified": false
    }
  }
}
```

**Location:** `deploy/manifests/sha256-manifest.json` (committed to repo). A copy is placed at `/etc/tianer/sha256-manifest.json` with `verified` flags updated post-deployment.

### 3.4 Backup Index

Backups are tracked in a local index file.

```jsonc
{
  "schema_version": 1,
  "backups": [
    {
      "id": "20260609-143000",
      "created_at": "2026-06-09T14:30:00Z",
      "type": "full",
      "components": ["database", "config", "pcap"],
      "db_backup_path": "/mnt/backup/tianer-db-20260609-143000.tar.gz",
      "db_backup_size_mb": 342,
      "config_backup_path": "/mnt/backup/tianer-config-20260609-143000.tar.gz",
      "version_manifest": "/mnt/backup/tianer-version-20260609-143000.json",
      "sha256": "d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2c3d4",
      "verified": true
    }
  ]
}
```

**Location:** `/etc/tianer/backup-index.json` — mode `0640`, owned by `tianer:tianer`.

---

## 4. Low-Level Architecture

### 4.1 `setup.sh` Phases — Detailed

#### Phase 1: OS Verification

```bash
#!/usr/bin/env bash
# Phase 1: Verify the host meets minimum requirements.
# Idempotent: yes (read-only check)
# Failure: exits 1 with diagnostic message

set -euo pipefail

verify_os() {
    log "Phase 1: Verifying OS..."
    if ! grep -q 'Raspberry Pi OS' /etc/os-release 2>/dev/null; then
        log ERROR "Not Raspberry Pi OS. Found: $(grep PRETTY_NAME /etc/os-release || echo 'unknown')"
        exit 1
    fi
    if ! grep -q 'VERSION_ID="13"' /etc/os-release 2>/dev/null; then
        log ERROR "Not Debian 13 (Trixie). Found: $(grep VERSION_ID /etc/os-release || echo 'unknown')"
        exit 1
    fi
    if [[ "$(uname -m)" != "aarch64" ]]; then
        log ERROR "Not ARM64. Found: $(uname -m)"
        exit 1
    fi
    log "OS verification passed: Raspberry Pi OS 64-bit Trixie (Debian 13) on aarch64"
    mark_phase_complete "1_os_verify"
}
```

#### Phase 2: Package Installation

```bash
# Phase 2: Install system packages from apt-packages.txt
# Idempotent: yes — apt-get install on already-installed packages is a no-op
# Supply chain: every third-party GPG key is verified against hardcoded SHA256 before import

install_packages() {
    log "Phase 2: Installing system packages..."

    # Step 2a: Verify and import GPG keys for third-party repositories
    verify_gpg_keys

    # Step 2b: Install core packages from apt-packages.txt [4]
    xargs -a "${SCRIPT_DIR}/apt-packages.txt" apt-get install -y --no-install-recommends

    # Step 2c: Install PostgreSQL from PGDG repo
    apt-get install -y --no-install-recommends postgresql-17 postgresql-server-dev-17

    # Step 2d: Install TimescaleDB
    apt-get install -y --no-install-recommends timescaledb-2-postgresql-17

    # Step 2e: Install Grafana
    apt-get install -y --no-install-recommends grafana

    # Step 2f: Install Node.js 24 LTS (for frontend build only; not needed for runtime)
    apt-get install -y --no-install-recommends nodejs

    # Step 2g: Install Python 3.13 + uv
    apt-get install -y --no-install-recommends python3 python3-dev python3-venv
    python3 -m pip install uv  # uv is bootstrapped, pinned

    mark_phase_complete "2_packages"
}
```

**GPG key verification (Phase 2a detail):**

```bash
# Hardcoded known-good GPG key fingerprints.
# These are verified against the project's official documentation before committing.
declare -A GPG_KEY_FINGERPRINTS=(
    ["postgresql"]="B97B0AFCAA1A47F044F244A07FCC7D46ACCC4CF8"
    ["grafana"]="4E40D0CEDE1C9A1FA4DE5C9C0E22EB88E39E60E7"
    ["nodesource"]="9FD3B784BC1C6FC31A8A0A1C1655A0AB68576280"
)

verify_gpg_keys() {
    for keyfile in "${SCRIPT_DIR}/gpg-keys/"*.asc; do
        local key_name=$(basename "$keyfile" .asc)
        local expected="${GPG_KEY_FINGERPRINTS[$key_name]:-}"
        if [[ -z "$expected" ]]; then
            log ERROR "Unknown GPG key: $key_name"
            exit 3
        fi
        local actual=$(gpg --dry-run --import --import-options show-only \
            "$keyfile" 2>&1 | grep -oP '[A-F0-9]{40}' | head -1)
        if [[ "$actual" != "$expected" ]]; then
            log ERROR "GPG key fingerprint mismatch for $key_name"
            log ERROR "Expected: $expected"
            log ERROR "Got:      $actual"
            exit 3
        fi
        log INFO "GPG key verified: $key_name ($expected)"
    done
}
```

#### Phase 3: User and Group Creation

```bash
# Phase 3: Create tianer system user and required groups.
# Idempotent: yes — checks user existence; usermod -aG preserves existing groups.
# Script: deploy/scripts/create-user.sh

create_user_and_groups() {
    log "Phase 3: Creating tianer user and groups..."
    "${SCRIPT_DIR}/scripts/create-user.sh"
    mark_phase_complete "3_user_group"
}
```

#### Phase 4: Filesystem Layout + Secrets

```bash
# Phase 4: Create directory structure, FIFOs, and secrets.
# Idempotent: yes — install -d is idempotent; generate-secrets.sh checks file existence.
# Scripts: deploy/scripts/create-dirs.sh, deploy/scripts/generate-secrets.sh

setup_filesystem() {
    log "Phase 4: Creating filesystem layout..."
    "${SCRIPT_DIR}/scripts/create-dirs.sh"
    systemd-tmpfiles --create /etc/tmpfiles.d/tianer.conf
    "${SCRIPT_DIR}/scripts/generate-secrets.sh"
    mark_phase_complete "4_filesystem"
}
```

#### Phase 5: Hardware Access

```bash
# Phase 5: Configure Wireshark capabilities and install udev rules.
# Idempotent: yes — dpkg-reconfigure is idempotent; install overwrites rules file.
# Scripts: deploy/scripts/setup-wireshark.sh

setup_hardware_access() {
    log "Phase 5: Configuring hardware access..."
    "${SCRIPT_DIR}/scripts/setup-wireshark.sh"
    install -m 0644 "${SCRIPT_DIR}/udev/99-tianer.rules" /etc/udev/rules.d/99-tianer.rules
    udevadm control --reload-rules
    udevadm trigger
    mark_phase_complete "5_hardware"
}
```

#### Phase 6: Container Runtime

```bash
# Phase 6: Install and configure rootless Podman.
# Idempotent: yes — apt-get install on installed packages is no-op;
# loginctl enable-linger is idempotent.

setup_container_runtime() {
    log "Phase 6: Setting up container runtime..."
    apt-get install -y --no-install-recommends podman slirp4netns fuse-overlayfs uidmap catatonit
    loginctl enable-linger tianer
    mark_phase_complete "6_container_runtime"
}
```

#### Phase 7: Quadlet Deployment

```bash
# Phase 7: Deploy Quadlet .container, .pod, .volume, .network files.
# Idempotent: yes — overwrites existing files.

deploy_quadlet_files() {
    log "Phase 7: Deploying Quadlet unit files..."
    install -d -o root -g root -m 0755 /etc/containers/systemd
    install -m 0644 "${SCRIPT_DIR}/containers/"*.container /etc/containers/systemd/
    install -m 0644 "${SCRIPT_DIR}/containers/"*.pod       /etc/containers/systemd/
    install -m 0644 "${SCRIPT_DIR}/containers/"*.volume     /etc/containers/systemd/
    install -m 0644 "${SCRIPT_DIR}/containers/"*.network    /etc/containers/systemd/
    mark_phase_complete "7_quadlet"
}
```

#### Phase 8: systemd daemon-reload

```bash
# Phase 8: Reload systemd to pick up Quadlet units.
# Idempotent: yes.

reload_systemd() {
    log "Phase 8: Reloading systemd..."
    systemctl daemon-reload
    mark_phase_complete "8_daemon_reload"
}
```

#### Phase 9: DB Migrations

```bash
# Phase 9: Apply database migrations.
# Idempotent: yes — migration tracking table prevents re-application.
# Requires: PostgreSQL running (Phase 2 installed the packages; operator starts the service
# or the postgres container takes over via Quadlet).

apply_migrations() {
    log "Phase 9: Applying database migrations..."
    make -C "${PROJECT_ROOT}" db-up
    mark_phase_complete "9_db_migrations"
}
```

#### Phase 10: Container Image Build

```bash
# Phase 10: Build all container images using multi-stage Containerfiles.
# Idempotent: yes — Podman caches layers; rebuilds only changed layers.
# Runs as tianer user (rootless Podman).

build_images() {
    log "Phase 10: Building container images..."
    sudo -u tianer make -C "${PROJECT_ROOT}" build
    mark_phase_complete "10_image_build"
}
```

#### Phase 11: Container Services Enable

```bash
# Phase 11: Enable systemd --user services so containers auto-start at boot.
# Idempotent: yes — systemctl enable is a no-op on already-enabled units.

enable_services() {
    log "Phase 11: Enabling container services..."
    sudo machinectl shell tianer@ /bin/bash -c '
        systemctl --user enable tianer-postgres.service
        systemctl --user enable tianer-grafana.service
        systemctl --user enable tianer-capture-pod.service
        systemctl --user enable tianer-platform-pod.service
    '
    mark_phase_complete "11_services_enable"
}
```

#### Phase 12: Container Services Start

```bash
# Phase 12: Start all containers.
# Starts in dependency order: postgres → pods → grafana.

start_services() {
    log "Phase 12: Starting container services..."
    sudo machinectl shell tianer@ /bin/bash -c 'systemctl --user start tianer.target'
    sleep 5  # Allow containers to initialize
    mark_phase_complete "12_services_start"
}
```

#### Phase 13: Smoke Verification

```bash
# Phase 13: Run a quick smoke test.
# Verifies: PostgreSQL accepting connections, API responding, containers running.
# Script: tests/e2e/smoke.sh (subset — fast checks only).

smoke_verify() {
    log "Phase 13: Running smoke verification..."
    "${PROJECT_ROOT}/tests/e2e/smoke.sh" --quick
    mark_phase_complete "13_smoke"
}
```

#### Phase 14: Supply Chain Verification

```bash
# Phase 14: Post-deployment supply chain audit.
# Verifies: all container image digests match the manifest,
# all Python packages match uv.lock hashes, npm audit passes.
# Script: deploy/scripts/verify-supply-chain.sh

verify_supply_chain() {
    log "Phase 14: Verifying supply chain integrity..."
    "${SCRIPT_DIR}/scripts/verify-supply-chain.sh"
    mark_phase_complete "14_supply_chain"
}
```

#### Phase 15: Backup Configuration

```bash
# Phase 15: Configure and schedule backups (if destination configured).
# Reads TIANER_BACKUP_DEST from config; skips gracefully if not set.

configure_backup() {
    log "Phase 15: Configuring backup..."
    source /etc/tianer/tianer.env 2>/dev/null || true
    if [[ -n "${TIANER_BACKUP_DEST:-}" ]]; then
        # Enable backup timer
        sudo machinectl shell tianer@ /bin/bash -c '
            systemctl --user enable tianer-backup.timer
            systemctl --user start tianer-backup.timer
        '
        log INFO "Backup configured: destination=${TIANER_BACKUP_DEST}"
    else
        log WARN "No backup destination configured (TIANER_BACKUP_DEST not set). Skipping."
    fi
    mark_phase_complete "15_backup_config"
}
```

#### Phase 16: Summary and Next Steps

```bash
# Phase 16: Write version manifest, print summary.
# Records what was deployed, how to access it, and next steps.

print_summary() {
    log ""
    log "============================================"
    log "  Tian'er Deployment Complete"
    log "============================================"
    log ""
    log "Deploy state: /etc/tianer/deploy-state.json"
    log "Version manifest: /etc/tianer/version-manifest.json"
    log ""
    log "Services:"
    log "  PostgreSQL:  127.0.0.1:5432 (tianer database)"
    log "  API:         127.0.0.1:8080"
    log "  Grafana:     127.0.0.1:3000"
    log ""
    log "Useful commands:"
    log "  make services-status    — Check all container statuses"
    log "  make logs               — Tail all logs"
    log "  make test-e2e           — Full end-to-end verification"
    log "  make backup             — Run manual backup"
    log ""
    log "API key: $(cat /etc/tianer/secrets/api_key 2>/dev/null || echo 'not found — check /etc/tianer/secrets/')"
    log ""

    # Write version manifest
    write_version_manifest
    mark_phase_complete "16_summary"

    log "Setup complete."
}
```

### 4.2 `setup.sh` — Full Orchestrator Logic

The main `setup.sh` script dispatches to each phase function. It supports:

```bash
#!/usr/bin/env bash
set -euo pipefail

# CLI flags
FROM_PHASE=1
DRY_RUN=false
SKIP_SMOKE=false
CONFIG_FILE="/etc/tianer/tianer.env"

# Parse arguments
while [[ $# -gt 0 ]]; do
    case "$1" in
        --from-phase) FROM_PHASE="$2"; shift 2 ;;
        --dry-run)    DRY_RUN=true; shift ;;
        --skip-smoke) SKIP_SMOKE=true; shift ;;
        --help)       print_help; exit 0 ;;
        *)            log ERROR "Unknown flag: $1"; exit 1 ;;
    esac
done

# Run all phases in sequence
run_phase 1  verify_os               "$FROM_PHASE"
run_phase 2  install_packages        "$FROM_PHASE"
run_phase 3  create_user_and_groups  "$FROM_PHASE"
run_phase 4  setup_filesystem        "$FROM_PHASE"
run_phase 5  setup_hardware_access   "$FROM_PHASE"
run_phase 6  setup_container_runtime "$FROM_PHASE"
run_phase 7  deploy_quadlet_files    "$FROM_PHASE"
run_phase 8  reload_systemd          "$FROM_PHASE"
run_phase 9  apply_migrations        "$FROM_PHASE"
run_phase 10 build_images            "$FROM_PHASE"
run_phase 11 enable_services         "$FROM_PHASE"
run_phase 12 start_services          "$FROM_PHASE"
run_phase 13 "$SKIP_SMOKE && echo skip || smoke_verify" "$FROM_PHASE"
run_phase 14 verify_supply_chain     "$FROM_PHASE"
run_phase 15 configure_backup        "$FROM_PHASE"
run_phase 16 print_summary           "$FROM_PHASE"
```

### 4.3 Makefile Architecture

The `Makefile` is structured in sections:

```makefile
# Makefile — Tian'er Signal Intelligence Platform
# ================================================

PROJECT_ROOT := $(shell pwd)
TIANER_USER  := tianer

# --- Discovery: find all component directories ---
PYTHON_COMPONENTS := $(wildcard modules/bluetooth/*/) $(wildcard platform/*/)
CPP_COMPONENTS    := $(filter-out %/sniffers/ %/tests/,$(wildcard modules/bluetooth/*/))

# --- Help ---
.PHONY: help
help:
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort | \
		awk 'BEGIN {FS = ":.*?## "}; {printf "  \033[36m%-24s\033[0m %s\n", $$1, $$2}'

# --- Setup ---
.PHONY: setup
setup: ## Run full host bootstrap (requires root)
	@sudo deploy/setup.sh

# --- Build ---
.PHONY: build build-base
build: build-base build-images ## Build all container images

build-base: ## Build the tianer base image (required by C++ components)
	@podman build -t tianer-base:latest -f deploy/containers/Containerfile.base .

build-images: build-ingest build-deep-parse build-tshark build-gap-detector \
              build-ml-classify build-frontend build-api build-grafana \
              build-rotate build-sniffer

build-ingest: ## Build ingest bridge image
	@podman build -t tianer-ingest:latest \
		-f deploy/containers/Containerfile.ingest \
		modules/bluetooth/ingest-bridge/

# ... (one target per component)

# --- Test ---
.PHONY: test-unit test-integration test-e2e test-all
test-unit: ## Run all unit tests
	@./ci/test-all.sh --unit

test-integration: ## Run all integration tests
	@./ci/test-all.sh --integration

test-e2e: ## Run end-to-end smoke test
	@./tests/e2e/smoke.sh

test-all: test-unit test-integration test-e2e ## Run all tests

# --- Deploy ---
.PHONY: deploy install services-restart
deploy: build install services-restart ## Full deploy: build, install, restart

install: ## Install artifacts (Quadlet files, container images)
	@sudo deploy/scripts/install-quadlet.sh
	@make db-up

services-start: ## Start all containers
	@sudo machinectl shell $(TIANER_USER)@ /bin/bash -c 'systemctl --user start tianer.target'

services-stop: ## Stop all containers
	@sudo machinectl shell $(TIANER_USER)@ /bin/bash -c 'systemctl --user stop tianer.target'

services-restart: services-stop services-start ## Restart all containers

services-status: ## Show container status
	@sudo machinectl shell $(TIANER_USER)@ /bin/bash -c 'systemctl --user status tianer-*.service'

# --- Backup & Restore ---
.PHONY: backup backup-full restore restore-list
backup: ## Run database backup
	@deploy/scripts/backup.sh

backup-pcap: ## Backup PCAP files
	@echo "TODO: tianer-backup-pcap.service integration"

backup-full: backup backup-pcap ## Full backup (DB + config + PCAP)

restore: ## Restore database from backup (interactive)
	@deploy/scripts/restore.sh

restore-list: ## List available backups
	@deploy/scripts/restore.sh --list

# --- Verification ---
.PHONY: verify verify-version verify-idempotency
verify: ## Run supply chain verification
	@deploy/scripts/verify-supply-chain.sh

verify-version: ## Check deployed versions against manifest
	@deploy/scripts/verify-version.sh

verify-idempotency: ## Run setup.sh twice, confirm no changes on second run
	@deploy/scripts/verify-idempotency.sh

# --- Rollback ---
.PHONY: rollback rollback-list
rollback: ## Roll back to previous deployment
	@deploy/scripts/rollback.sh

rollback-list: ## List available rollback versions
	@deploy/scripts/rollback.sh --list

# --- Database ---
.PHONY: db-up db-reset db-migrate-new
db-up: ## Apply pending DB migrations
	@db/apply-migrations.sh

db-reset: ## Drop and recreate database (DESTRUCTIVE; CONFIRM=yes required)
	@test "$(CONFIRM)" = "yes" || { echo "Set CONFIRM=yes to proceed."; exit 1; }
	@db/reset.sh

# --- Logs ---
.PHONY: logs
logs: ## Tail container logs
	@sudo machinectl shell $(TIANER_USER)@ /bin/bash -c \
		'journalctl --user -u tianer-*.service -f'

# --- Clean ---
.PHONY: clean clean-all
clean: ## Remove build artifacts
	@find . -type d -name build -prune -exec rm -rf {} +
	@find . -type d -name __pycache__ -prune -exec rm -rf {} +
	@find . -type d -name node_modules -prune -exec rm -rf {} +

clean-all: clean ## Remove build artifacts AND container images
	@podman system prune -af --volumes
```

### 4.4 Containerfile Structure

Each component's `Containerfile` follows a consistent multi-stage pattern.

#### 4.4.1 Base Image (`deploy/containers/Containerfile.base`)

```dockerfile
# Stage 1: Build environment (used by C++ components)
FROM debian:trixie-slim AS builder
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    g++-14 \
    cmake \
    libpqxx-dev \
    libpcap-dev \
    nlohmann-json3-dev \
    pkg-config \
    && rm -rf /var/lib/apt/lists/*
# Pin cmake version for reproducibility
RUN cmake --version | head -1

# Stage 2: Runtime environment (minimal — no compilers, no headers)
FROM debian:trixie-slim AS runtime
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpqxx-7.8 \
    libpcap0.8 \
    libstdc++6 \
    ca-certificates \
    zstd \
    && rm -rf /var/lib/apt/lists/*
```

#### 4.4.2 Ingest Bridge (`deploy/containers/Containerfile.ingest`)

```dockerfile
ARG BASE_IMAGE=localhost/tianer-base:latest

FROM ${BASE_IMAGE} AS builder
COPY . /src/
WORKDIR /src
RUN cmake -B build -DCMAKE_BUILD_TYPE=Release && cmake --build build -- -j$(nproc)

FROM ${BASE_IMAGE} AS runtime
COPY --from=builder /src/build/blesniff-ingest /usr/local/bin/blesniff-ingest
USER tianer
ENTRYPOINT ["/usr/local/bin/blesniff-ingest"]
```

#### 4.4.3 Python API (`deploy/containers/Containerfile.api`)

```dockerfile
# Stage 1: Frontend build
FROM node:24-slim AS frontend-builder
COPY platform/frontend/ /src/frontend/
WORKDIR /src/frontend
RUN npm ci [6] && npm run build

# Stage 2: Python venv build
FROM python:3.13-slim AS python-builder
RUN python3 -m pip install uv
COPY platform/api/ /src/api/
WORKDIR /src/api
RUN uv sync --frozen --no-dev [5]

# Stage 3: Runtime (minimal)
FROM python:3.13-slim AS runtime
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq5 ca-certificates \
    && rm -rf /var/lib/apt/lists/*
COPY --from=python-builder /src/api/.venv /opt/tianer/api/.venv
COPY --from=frontend-builder /src/frontend/dist /usr/share/tianer/frontend
ENV PATH="/opt/tianer/api/.venv/bin:$PATH"
USER tianer
EXPOSE 8080
ENTRYPOINT ["uvicorn", "tianer_api.main:app", "--host", "0.0.0.0", "--port", "8080"]
```

### 4.5 Quadlet Files

Each Quadlet file is a declarative container specification. See `deploy/containers/` in the repository. Key template pattern:

**`tianer-sniffer@.container` (template):**

```ini
[Unit]
Description=Tian'er Sniffer %i Wrapper
After=tianer-tshark@%i.service

[Container]
Image=localhost/tianer-sniffer:latest
EnvironmentFile=/etc/tianer/tianer.env
EnvironmentFile=/etc/tianer/blesniff.env
Environment=SNIFFER_NAME=%i
Volume=/etc/tianer:/etc/tianer:ro
Volume=/var/lib/tianer/pcap:/var/lib/tianer/pcap:rw
Volume=/var/lib/tianer/heartbeat:/var/lib/tianer/heartbeat:rw
Volume=/var/log/tianer:/var/log/tianer:rw
Volume=/var/run/tianer:/var/run/tianer:rw
AddDevice=/dev/tianer/%i
Pod=tianer-capture.pod
User=tianer
Group=tianer
GroupAdd=keep-groups
SecurityLabelDisable=true
NoNewPrivileges=true
```

---

## 5. Inter-Component Contracts

### 5.1 Setup Contract (SETUP-1)

**From:** C14 (setup.sh)  
**To:** C01 (Platform Infrastructure)  
**Format:** Host state verification

C14 guarantees that after `setup.sh` Phase 4 completes, all paths listed in C01's `FS-LAYOUT` contract exist with correct ownership and permissions. C01's `tianer-check-prereqs.sh` can be run to verify.

### 5.2 Build Contract (BUILD-1)

**From:** C14 (Makefile)  
**To:** Per-component `Containerfile`  
**Format:** Container image + digest

Every component's `Containerfile` must:
1. Build to a tagged image (`localhost/tianer-<component>:<tag>`)
2. Accept an `ARG BASE_IMAGE` for the base image
3. Be a multi-stage build with no development tools in the runtime stage
4. Produce an image with a `USER tianer` directive
5. Not include any secrets in image layers

### 5.3 Deploy Contract (DEPLOY-1)

**From:** C14 (Quadlet deployment)  
**To:** C12 (Service Orchestration)  
**Format:** Generated systemd `--user` units from Quadlet `.container` files

C14 deploys Quadlet files; C12's runtime orchestration depends on the generated systemd units. The unit names must be predictable: `tianer-<component>.service` for singletons, `tianer-<component>@<instance>.service` for templates.

### 5.4 Backup Contract (BACKUP-1)

**From:** C14 (`make backup`)  
**To:** C02 (Database)  
**Format:** `pg_basebackup` tarball + config tarball

C14 runs `pg_basebackup -Ft -z -D <dest>` against the PostgreSQL container. The backup includes the entire `tianer` database. Config files from `/etc/tianer/` are backed up separately (excluding secrets).

### 5.5 Supply Chain Contract (SUPPLY-1)

**From:** C14 (verify-supply-chain.sh)  
**To:** All externally sourced artifacts  
**Format:** SHA256 comparison against committed manifest

Every external binary, container image, and GPG key referenced in the project must have a SHA256 entry in `deploy/manifests/sha256-manifest.json`. The `verify-supply-chain.sh` script checks all entries and exits 0 only when every hash matches.

---

## 6. Failure Modes & Recovery

### 6.1 Failure Catalog

| # | Failure Mode | Detection | Impact | Recovery |
|---|-------------|-----------|--------|----------|
| F-C14-1 | **Partial setup — interrupted mid-phase** | Deploy state file shows phase status `in_progress`; no `completed` marker | Host in partially configured state; containers may not start | Re-run `setup.sh` — detects already-completed phases and resumes from the failed phase |
| F-C14-2 | **Version mismatch — wrong OS version** | Phase 1: `grep VERSION_ID /etc/os-release` | setup.sh exits 1 immediately; nothing modified | Install correct OS (Raspberry Pi OS 64-bit Trixie) |
| F-C14-3 | **Version mismatch — wrong image tag pulled** | Phase 14: `verify-supply-chain.sh` compares digests | Container may run wrong code version | `make rollback` to previous digest; investigate registry |
| F-C14-4 | **Disk full during image build** | `df -h` during Phase 10; Podman build error | Image build fails mid-way; partial layers left | Free space; `make clean-all`; retry. C04 emergency purge of old PCAP files |
| F-C14-5 | **Disk full during backup** | `pg_basebackup` error; disk space check | Backup incomplete; previous backup may be corrupted | Free space; retry backup. Alert fires at 80% disk usage |
| F-C14-6 | **Container image pull failure** | Podman pull error in Phase 10 or service start | Services cannot start on that image | Fall back to local build (`make build` instead of `make pull`); retry registry; alert if pull fails 3 consecutive times |
| F-C14-7 | **GPG key changed upstream** | Phase 2a: fingerprint comparison fails | setup.sh exits 3; no packages installed | Verify with project maintainers; if legitimate, update SHA256 in manifest and commit |
| F-C14-8 | **SHA256 checksum mismatch (binary)** | Phase 14: `sha256sum` against manifest | Downloaded binary may be tampered | Re-download from official source; if persistent, escalate; do not deploy |
| F-C14-9 | **Database migration failure** | Phase 9: migration script exits non-zero | Schema may be partially migrated; DB in unknown state | `_migrations` table tracks which applied; manual investigation; restore from backup if needed |
| F-C14-10 | **Container image build failure (compile error)** | Phase 10: `podman build` exits non-zero | That component's image not built; others may be fine | Fix source; re-run `make build-<component>`; redeploy |
| F-C14-11 | **Backup destination unreachable** | Phase 15: mount check fails | Backups not configured; alert | Mount external drive; re-run `make backup` |
| F-C14-12 | **Rollback to version with different schema** | Migration mismatch: rollback target has fewer migrations | DB may be in state incompatible with old code | Rollback should be code-only (container images, config); DB must be restored separately from a backup taken at that version |
| F-C14-13 | **`npm audit` failure (vulnerabilities)** | Phase 14: `npm audit --audit-level=high` | Vulnerable frontend dependencies; may deploy with warning | Fix or override; if override, document in `docs/decisions-log.md` |
| F-C14-14 | **uv.lock hash mismatch** | Phase 14: `uv sync --frozen` fails | Python dependency tree changed without lock update | Regenerate lock, review dependency changes, commit; or revert |
| F-C14-15 | **sudo required but operator not in sudoers** | Phase 2–5: `sudo` commands fail | Bootstrap cannot proceed | Operator must have sudo access for install-time. This is documented as a prerequisite. |

### 6.2 Idempotency as Recovery

The primary recovery mechanism for partial failures is **idempotency**:

- Every `setup.sh` phase can be re-run safely, regardless of whether it previously completed or failed.
- `make db-up` tracks applied migrations in the `_migrations` table — re-running does not double-apply.
- `make build` uses Podman's layer cache — rebuilding only recompiles changed layers.
- `systemctl --user enable` is a no-op on already-enabled units.
- `systemctl --user start` is safe on already-running units.

The principle: **re-running the same command with the same inputs produces the same state.** No phase requires "clean undo" before retry. If a phase fails mid-way, re-running it from the start is safe.

### 6.3 Recovery SLA

| Scenario | Detection | Recovery Time | Data Loss |
|----------|-----------|---------------|-----------|
| Partial setup (interrupted) | Deploy state file | < time to re-run remaining phases | None — setup-only, no data |
| Image build failure | Build log | Developer fixes source; rebuilds in < 5 min | None — no running service affected |
| Deployment (image pull) failure | Phase 10/12 error | Rollback to previous digest < 2 min | None — previous images still present |
| Disk full during build | Build error | Free space + retry < 10 min | None if freeable; otherwise operator intervention |
| GPG key change | Phase 2a check | Investigation + manifest update < 1 hour | None — no installation proceeds until verified |
| DB migration failure | Phase 9 error | Restore from backup + retry < 30 min | Last backup's data window |

---

## 7. Observability

### 7.1 Deployment Logging

Every `setup.sh` phase and `make` target writes structured log output:

```
TIANER | {"ts":"2026-06-09T14:30:01Z","level":"INFO","component":"deploy","phase":"1_os_verify","msg":"OS verification passed","os":"Raspberry Pi OS 64-bit Trixie Debian 13","arch":"aarch64"}

TIANER | {"ts":"2026-06-09T14:32:00Z","level":"INFO","component":"deploy","phase":"2_packages","msg":"Packages installed","count":47}

TIANER | {"ts":"2026-06-09T14:35:30Z","level":"INFO","component":"deploy","phase":"13_smoke","msg":"Smoke test passed","duration_s":20}

TIANER | {"ts":"2026-06-09T14:35:37Z","level":"INFO","component":"deploy","phase":"16_summary","msg":"Deployment complete","total_duration_s":337}
```

**Log destination:** `journald` (via stderr) and `/var/log/tianer/deploy.log` (dual-write).

### 7.2 Version and Digest Logging

After every deployment, the version manifest is written with:
- Every deployed container image digest (`@sha256:...`)
- Every apt package version installed
- Every Python package version from `uv.lock`
- Every npm package version from `package-lock.json`
- Every externally downloaded binary SHA256
- Every GPG key fingerprint verified
- Whether supply chain verification passed

This manifest is a permanent audit record. It is backed up alongside the database.

### 7.3 Health Checks (Post-Deployment)

C14's smoke verification (Phase 13) checks:

| Check | Command | Pass Condition |
|-------|---------|----------------|
| PostgreSQL accepting connections | `psql -h 127.0.0.1 -U tianer -c 'SELECT 1'` | Returns `1` |
| API health endpoint | `curl -sf http://127.0.0.1:8080/api/health` | Returns `{"status":"ok"}` |
| Grafana health endpoint | `curl -sf http://127.0.0.1:3000/api/health` | Returns `{"database":"ok"}` |
| All containers running | `systemctl --user is-active tianer-*.service` | All report `active` |
| DB migrations applied | `psql -c "SELECT COUNT(*) FROM _migrations"` | >= 5 |

### 7.4 Metrics

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `tianer_deploy_phase_duration_seconds` | Gauge | `phase` | Duration of each setup phase |
| `tianer_deploy_total` | Counter | `status` | Number of deployments (success/fail/rollback) |
| `tianer_build_duration_seconds` | Gauge | `component` | Image build time per component |
| `tianer_backup_size_bytes` | Gauge | `type` | Size of last backup |
| `tianer_backup_duration_seconds` | Gauge | `type` | Duration of last backup |
| `tianer_supply_chain_verified` | Gauge (0/1) | — | 1 if last supply chain check passed |
| `tianer_image_digest_count` | Gauge | — | Number of tracked image digests |

### 7.5 Alerts

| Alert | Condition | Severity |
|-------|-----------|----------|
| `TianerDeployFailed` | Deploy state file has `failed_phase` not null | Critical |
| `TianerSupplyChainFail` | `tianer_supply_chain_verified == 0` | Critical |
| `TianerBackupFail` | Last backup older than 25 hours | Warning |
| `TianerImageDigestMismatch` | Running image digest != version manifest digest | Critical |
| `TianerDiskFullWarning` | Disk usage > 80% on `/` or `/var/lib/tianer` | Warning |

---

## 8. Security Considerations

### 8.1 Supply Chain Integrity (Q10 — Mandatory for MVP)

Supply chain integrity is **mandatory for MVP** per ADR-0001 (Q10). C14 implements five layers of verification:

| Layer | Artifact | Verification | Failure Action |
|-------|----------|-------------|----------------|
| 1. GPG keys | apt repository signing keys | Hardcoded SHA256 fingerprint comparison before import | Halt with exit 3 |
| 2. External binaries | nrfutil, ubertooth firmware | SHA256 checksum against committed manifest | Halt; do not deploy |
| 3. Container images | All images (built or pulled) | Digest pinning (`@sha256:`); Phase 14 verifies | Halt deployment; alert |
| 4. Python packages | All `uv.lock` dependencies | `uv sync --frozen` enforces hash-pinned packages | Build fails; review and update lock |
| 5. npm packages | Frontend dependencies | `npm ci` (clean install from lockfile) + `npm audit --audit-level=high` in CI | Audit failure blocks deployment |

**GPG key fingerprint verification (Phase 2a):**

Every third-party apt GPG key has a hardcoded expected SHA256 fingerprint committed to the repository. Before importing any key, `setup.sh` computes the actual fingerprint and compares. A mismatch indicates either:
- The upstream project rotated their signing key (legitimate — verify via official channels, then update the manifest)
- A man-in-the-middle attack or repository compromise (halt immediately)

**Container image digest pinning:**

All Quadlet `.container` files use digest-pinned image references:
```ini
Image=localhost/tianer-ingest@sha256:9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08
```

This is enforced by CI: any Quadlet file containing `:latest` (without `@sha256:`) [3] is rejected. The digest is written into the version manifest and verified at startup.

**Python hash pinning:**

`uv.lock` contains SHA256 hashes for every dependency. `uv sync --frozen` enforces these hashes — if any package hash changes upstream, the build fails. This prevents:
- Dependency confusion attacks (a package name collision in a public index)
- Malicious package updates (an attacker publishing a modified version of a dependency)
- Accidental breaking changes from unpinned transitive dependencies

**npm integrity:**

`npm ci` uses `package-lock.json` with integrity hashes. `npm audit --audit-level=high` fails the CI build on high-severity vulnerabilities.

### 8.2 No Secrets in Scripts

C14 enforces the following rules:
- **No secrets in `setup.sh`:** All credentials are generated by `generate-secrets.sh` (C01) and stored in `/etc/tianer/secrets/` with mode `0600`.
- **No secrets in Containerfiles:** Secrets are injected at runtime via `EnvironmentFile` in Quadlet units, never baked into image layers.
- **No secrets in environment files committed to repo:** `deploy/config/tianer.env.example` is a template with placeholder values. The actual `tianer.env` is never committed.
- **No secrets in Makefile:** Database passwords are never hardcoded in Makefile targets.

### 8.3 Root-Only Privileges

| Operation | Requires `root`? | Justification |
|-----------|-----------------|---------------|
| `apt-get install` | Yes | System package manager requires root |
| `useradd` / `usermod` | Yes | User management requires root |
| `install` to `/etc/`, `/usr/` | Yes | System directories owned by root |
| `systemctl daemon-reload` | Yes | systemd control requires root |
| `udevadm control --reload` | Yes | udev management requires root |
| `loginctl enable-linger` | Yes | Enabling linger for another user requires root |
| `podman build` / `podman pull` | **No** | Runs as `tianer` (rootless Podman) |
| `systemctl --user start` | **No** | Runs as `tianer` user instance |
| `make db-up` (migrations) | **No** | Connects to PostgreSQL as `tianer` DB role |
| `make backup` | **No** | Runs `pg_basebackup` as `tianer` role |
| `make test-*` | **No** | Development activities |

The principle: root is required **only for provisioning the host**, not for runtime operations. Once C01 is complete, all remaining operations (including C14's build and deploy phases) run as the `tianer` user.

### 8.4 Backup Encryption

Backups written to external destinations (USB drive, network share) may contain sensitive data (BLE MAC addresses, device metadata). C14 supports optional GPG encryption of backups:

```bash
# In deploy/config/tianer.env.example:
# TIANER_BACKUP_GPG_KEY=0xABCD1234  # Optional: GPG public key for backup encryption
```

If `TIANER_BACKUP_GPG_KEY` is set, `backup.sh` encrypts the backup tarball before writing to the destination. Decryption requires the corresponding private key, which is stored separately (not on the Pi).

---

## 9. Configuration

### 9.1 Environment Variables

All C14 configuration is read from `/etc/tianer/tianer.env`:

| Variable | Default | Description |
|----------|---------|-------------|
| `TIANER_BACKUP_DEST` | (empty) | Path or URL for backup destination. Empty = automatic backup disabled |
| `TIANER_BACKUP_RETENTION_DAYS` | `14` | Number of daily backups to retain |
| `TIANER_BACKUP_GPG_KEY` | (empty) | GPG key fingerprint for backup encryption |
| `TIANER_REGISTRY` | `localhost` | Container image registry (local for on-device builds) |
| `TIANER_IMAGE_TAG` | `latest` | Default image tag for builds |
| `TIANER_BUILD_PARALLEL` | `$(nproc)` | Parallel build jobs for C++ components |
| `TIANER_DEPLOY_LOG_LEVEL` | `info` | Log level for deployment scripts (debug/info/warn/error) |
| `TIANER_VERIFY_STRICT` | `true` | If true, supply chain verification failure halts deployment |

### 9.2 Deploy State File

Location: `/etc/tianer/deploy-state.json`  
Schema: See §3.2  
Purpose: Track setup phase completion and deployment status

### 9.3 Version Manifest

Location: `/etc/tianer/version-manifest.json`  
Schema: See §3.1  
Purpose: Audit record of what was deployed

### 9.4 SHA256 Manifest

Location: `deploy/manifests/sha256-manifest.json` (committed)  
Schema: See §3.3  
Purpose: Known-good checksums for external artifacts

### 9.5 Quadlet Configuration

Files: `/etc/containers/systemd/*.container`, `*.pod`, `*.volume`, `*.network`  
Source: `deploy/containers/` in repository  
Update mechanism: `make install` copies files; `systemctl daemon-reload` applies

---

## 10. Test Plan

### 10.1 Unit Tests (bats-core)

| Test | File | Purpose |
|------|------|---------|
| GPG key verification — known key passes | `deploy/scripts/tests/test_gpg_verify.bats` | Valid key fingerprint matches |
| GPG key verification — wrong key fails | `deploy/scripts/tests/test_gpg_verify.bats` | Invalid key exits non-zero |
| SHA256 verification — known binary matches | `deploy/scripts/tests/test_sha256_verify.bats` | Checksum comparison correct |
| SHA256 verification — tampered binary fails | `deploy/scripts/tests/test_sha256_verify.bats` | Mismatch detected |
| Version manifest generation | `deploy/scripts/tests/test_version_manifest.bats` | JSON schema valid |
| Deploy state file — phase tracking | `deploy/scripts/tests/test_deploy_state.bats` | Phase transitions correct |

### 10.2 Integration Tests (bash)

| Test | File | Purpose |
|------|------|---------|
| `setup.sh` idempotency — two runs produce same state | `tests/integration/test_setup_idempotency.sh` | Second run is a no-op |
| `setup.sh --from-phase N` resumes correctly | `tests/integration/test_setup_partial.sh` | Skips completed phases |
| `make build` produces all images | `tests/integration/test_build.sh` | All 11 images tagged |
| `make deploy` smoke test | `tests/integration/test_deploy.sh` | Services start and respond |
| Container image size under target | `tests/integration/test_image_sizes.sh` | Each image < 200 MB uncompressed |
| Container boot time under 3 seconds | `tests/integration/test_boot_time.sh` | `podman start` to ready < 3s |
| Multi-stage build — no compilers in runtime | `tests/integration/test_multi_stage.sh` | `which g++` fails in runtime container |
| `make backup` + `make restore` round-trip | `tests/integration/test_backup_restore.sh` | Data survives backup/restore cycle |
| Supply chain verification — all passes | `tests/integration/test_supply_chain.sh` | `verify-supply-chain.sh` exits 0 |
| Supply chain verification — tampered image detected | `tests/integration/test_supply_chain_tamper.sh` | Digest mismatch detected |
| `npm audit` clean | `tests/integration/test_npm_audit.sh` | No high/critical vulnerabilities |
| Python hash pinning enforced | `tests/integration/test_python_hash.sh` | `uv sync --frozen` enforces hashes |
| Quadlet syntax validation | `tests/integration/test_quadlet_syntax.sh` | `quadlet --dryrun` succeeds |
| Container volume mounts correct | `tests/integration/test_volume_mounts.sh` | Each mount from storage-strategy.md present |
| Rollback restores previous images | `tests/integration/test_rollback.sh` | Old image digests restored |
| Backup encryption round-trip | `tests/integration/test_backup_gpg.sh` | Encrypted backup decryptable |

### 10.3 End-to-End Tests

| Test | File | Purpose |
|------|------|---------|
| Full clean deployment from OS image | `tests/e2e/test_full_deploy.sh` | `setup.sh` from scratch on test VM |
| Deploy → upgrade → rollback cycle | `tests/e2e/test_upgrade_rollback.sh` | Full lifecycle |
| Power-loss recovery (kill -9 during deploy) | `tests/e2e/test_power_loss.sh` | Partial deploy state, re-run recovers |

### 10.4 Acceptance Criteria

| # | Criterion | Verification | Pass Condition |
|---|-----------|-------------|----------------|
| AC-C14-1 | `setup.sh` completes all 16 phases on fresh OS | `sudo deploy/setup.sh` on test VM | Exits 0; deploy-state.json shows all `completed` |
| AC-C14-2 | `setup.sh` is idempotent | Run twice; second run produces no changes | Second run output shows "already completed" for all phases |
| AC-C14-3 | `make build` produces all 11 container images | `podman images localhost/tianer-*` | All images listed |
| AC-C14-4 | Each runtime image under 200 MB | `podman image inspect --format '{{.Size}}'` | Each < 209715200 bytes |
| AC-C14-5 | Supply chain verification passes on clean deploy | `make verify` | Exits 0 |
| AC-C14-6 | `make test-all` passes (unit + integration + e2e) | `make test-all` | Exits 0 |
| AC-C14-7 | Backup + restore round-trip preserves data | Seed test DB, backup, restore, verify | Row counts match |
| AC-C14-8 | Rollback to previous version works | Deploy A, deploy B, rollback to A | Running containers match version A digests |
| AC-C14-9 | No secrets in committed files | `grep -r 'password\|api_key\|secret' deploy/ --include='*.sh' --include='Containerfile*' --exclude='generate-secrets.sh'` | No matches (except generate-secrets.sh) |
| AC-C14-10 | GPG key mismatch halts setup | Tamper with a GPG key; run setup | Phase 2 exits 3 |
| AC-C14-11 | SHA256 mismatch halts verification | Tamper with a downloaded binary; run verify | Phase 14 exits 3 |

---

## 11. Deployment Notes

### 11.1 Prerequisites for Deployment

Before running `setup.sh`, the host must have:
- Raspberry Pi OS 64-bit (Trixie, Debian 13) installed and bootable
- SSH access for the operator account with `sudo` privileges
- Internet connectivity for package downloads
- Sufficient disk space: 16 GB free on eMMC, 32 GB SD card mounted

### 11.2 `setup.sh` as Root

```bash
# Clone the repository
git clone https://github.com/jean/tian-er.git /opt/tian-er
cd /opt/tian-er

# Run setup (requires sudo password)
sudo deploy/setup.sh

# For a dry run:
sudo deploy/setup.sh --dry-run

# To resume from a specific phase:
sudo deploy/setup.sh --from-phase 5

# To skip smoke tests (for headless deployments):
sudo deploy/setup.sh --skip-smoke
```

### 11.3 `make deploy` — Incremental Updates

After initial setup, use `make deploy` for incremental updates:

```bash
cd /opt/tian-er
git pull origin main
make deploy    # build → install → services-restart
```

This rebuilds only changed components (Podman layer caching), reinstalls Quadlet files, and restarts services.

### 11.4 Quadlet Deployment

Quadlet unit files are located in `deploy/containers/`. After any change to a Quadlet file:

```bash
make install              # Copies Quadlet files to /etc/containers/systemd/
sudo systemctl daemon-reload  # Quadlet regenerates systemd units
make services-restart     # Restart affected containers
```

### 11.5 Backup Procedure

**Manual backup:**
```bash
make backup       # Database + config backup
make backup-full  # Database + config + PCAP backup
```

**Automatic backup:** If `TIANER_BACKUP_DEST` is configured in `/etc/tianer/tianer.env`, nightly backups run via `tianer-backup.timer` (C12).

**Backup contents:**
- PostgreSQL full backup (`pg_basebackup -Ft -z`)
- Configuration snapshot (`/etc/tianer/` excluding `secrets/`)
- Version manifest at time of backup
- PCAP files (if `backup-full`)

### 11.6 Restore Procedure

```bash
# List available backups
make restore-list

# Restore database from most recent backup
make restore

# Restore a specific backup
make restore BACKUP_ID=20260609-143000

# Restore PCAP files from backup
make restore-pcap
```

**Restore steps (automated by `deploy/scripts/restore.sh`):**

1. Stop all containers except PostgreSQL
2. Drop existing `tianer` database (requires confirmation)
3. Create fresh `tianer` database
4. Restore from `pg_basebackup` tarball
5. Re-apply any migrations that were applied after the backup was taken
6. Restart all containers
7. Run smoke verification

### 11.7 Rollback Procedure

```bash
# List available rollback versions
make rollback-list

# Roll back to previous version
make rollback

# Roll back to a specific version manifest
make rollback VERSION=20260609-143000
```

**Rollback steps (automated by `deploy/scripts/rollback.sh`):**

1. Identify the version manifest for the target version (from `/etc/tianer/version-manifest.json.<ISO8601>.bak`)
2. Stop all containers
3. Tag current images as `tianer-<component>:previous`
4. Pull or re-tag images to match the target version manifest digests
5. Update Quadlet files to reference target digests (if digest changed)
6. Restart containers
7. Write new version manifest recording the rollback
8. Run smoke verification

**Rollback limitations:**
- Rollback is **code + config only**. If the target version has a different database schema, the operator must also restore the database from a backup taken at that version.
- Rollback does not downgrade system packages (apt). Container images are the deployment unit.
- Rollback preserves current data; it only changes running code.

### 11.8 Adding a New Component

To add a new component to the deployment pipeline:

1. Create `deploy/containers/Containerfile.<component>` (multi-stage build)
2. Create `deploy/containers/tianer-<component>.container` (Quadlet unit)
3. Add `build-<component>` target to `Makefile`
4. Add `build-<component>` to `build-images` dependency list
5. Add entry to `deploy/manifests/sha256-manifest.json` if the component downloads external artifacts
6. Update `setup.sh` Phases 10–12 if the component needs auto-start
7. Update `deploy/containers/tianer.target` if using a target unit

### 11.9 Image Registry Configuration

By default, images are built locally and tagged `localhost/tianer-<component>:<tag>`. To use a remote registry:

```bash
# In /etc/tianer/tianer.env:
TIANER_REGISTRY=registry.example.com/tianer
```

The `Makefile` then tags images as `registry.example.com/tianer/<component>:<tag>` and `make pull` pulls from the registry instead of building locally.

For offline/disconnected deployments, images can be built on a connected machine, exported with `podman save`, transferred via USB drive, and loaded with `podman load`. The `make build-offline` target handles this workflow.

---

## Appendix A: File Inventory

| File | Location | Purpose |
|------|----------|---------|
| `deploy/setup.sh` | Repository root | Master host bootstrap script |
| `deploy/scripts/create-user.sh` | Repository | Create `tianer` user and groups |
| `deploy/scripts/create-dirs.sh` | Repository | Create filesystem layout |
| `deploy/scripts/generate-secrets.sh` | Repository | Generate DB password, API key |
| `deploy/scripts/setup-wireshark.sh` | Repository | Configure dumpcap capabilities |
| `deploy/scripts/verify-supply-chain.sh` | Repository | Post-deployment supply chain audit |
| `deploy/scripts/verify-version.sh` | Repository | Version manifest validation |
| `deploy/scripts/verify-idempotency.sh` | Repository | Idempotency verification |
| `deploy/scripts/backup.sh` | Repository | Database + config backup |
| `deploy/scripts/restore.sh` | Repository | Database + config restore |
| `deploy/scripts/rollback.sh` | Repository | Version rollback |
| `deploy/scripts/install-quadlet.sh` | Repository | Install Quadlet files |
| `deploy/apt-packages.txt` | Repository | System package dependency list |
| `deploy/udev/99-tianer.rules` | Repository | USB device permissions |
| `deploy/config/tianer.env.example` | Repository | Configuration template |
| `deploy/manifests/sha256-manifest.json` | Repository | Known-good checksums |
| `deploy/containers/Containerfile.base` | Repository | Base container image |
| `deploy/containers/Containerfile.*` | Repository | Per-component container images (11 files) |
| `deploy/containers/tianer-*.container` | Repository | Quadlet container definitions (11 files) |
| `deploy/containers/tianer-*.pod` | Repository | Quadlet pod definitions (2 files) |
| `deploy/containers/tianer-*.volume` | Repository | Quadlet volume definitions (2 files) |
| `deploy/containers/tianer-*.network` | Repository | Quadlet network definition (1 file) |
| `deploy/gpg-keys/*.asc` | Repository | Third-party GPG keys (committed) |
| `Makefile` | Repository root | Build, test, deploy, backup, restore |

---

## Appendix B: Cross-References

| Component | References C14 for... | C14 Section |
|-----------|----------------------|-------------|
| C01 (Platform Infrastructure) | `setup.sh` phase sequence, script names, file paths | §4.1, Appendix A |
| C02 (Database) | `make db-up`, migration application, backup/restore | §4.2, §11.5–11.6 |
| C05 (Ingest Bridge) | `Containerfile.ingest`, `tianer-ingest.container`, `make build-ingest` | §4.4.2, §4.5 |
| C07 (Deep Parser) | `Containerfile.deep-parse`, `tianer-deep-parse.container`, `make build-deep-parse` | §4.4 |
| C09 (REST API) | `Containerfile.api`, `tianer-api.container`, `make build-api` | §4.4.3 |
| C10 (Frontend) | `Containerfile.frontend`, frontend build stage in API Containerfile | §4.4.3 |
| C12 (Service Orchestration) | Quadlet unit deployment, systemd integration | §2.4, §4.5 |

---

## Appendix C: Glossary

| Term | Definition |
|------|-----------|
| **Containerfile** | Podman-compatible container build file (equivalent to Dockerfile) |
| **Digest pinning** | Referencing container images by their content-addressable SHA256 digest (`@sha256:...`) rather than mutable tags |
| **GPack** | A package manager's GPG-signed repository metadata |
| **Idempotent** | Safe to run multiple times — produces the same result regardless of how many times it's executed |
| **Multi-stage build** | A Containerfile with multiple `FROM` statements — build tools exist only in earlier stages; runtime stage copies only artifacts |
| **Quadlet** | systemd generator that converts declarative `.container`/`.pod`/`.volume`/`.network` files into `systemd --user` service units |
| **Rootless Podman** | Podman running as an unprivileged user (not root), using Linux user namespaces for container isolation |
| **systemd-linger** | Mechanism allowing a user's `systemd --user` instance to persist across login sessions and start at boot |
| **uv** | Fast Python package manager — replaces pip, pip-tools, virtualenv |
| **uv.lock** | Hash-pinned dependency lock file produced by `uv lock` |

## References

[1] Free Software Foundation. "GNU Make Manual — Phony Targets." https://www.gnu.org/software/make/manual/html_node/Phony-Targets.html, 2024. Defines `.PHONY` target semantics and pattern rules.

[2] Podman Project. "podman-build(1) — Build a Container Image Using a Containerfile." https://docs.podman.io/en/latest/markdown/podman-build.1.html, 2024. Describes multi-stage Containerfile builds with `FROM ... AS` stages.

[3] Podman Project. "podman-run(1) — Run a Command in a New Container." https://docs.podman.io/en/latest/markdown/podman-run.1.html, 2024. Describes image digest pinning (`@sha256:`) and `--pull` policies.

[4] Debian Wiki. "SourcesList — Debian Wiki." https://wiki.debian.org/SourcesList, 2024. Documents `apt-get` package management, `sources.list` format, and GPG key verification for Debian repositories.

[5] Astral. "uv sync — Update the Project's Environment." https://docs.astral.sh/uv/reference/cli/#uv-sync, 2024. Documents `uv sync --frozen` (install from lockfile without updating) and `uv lock` (generate hash-pinned lockfile).

[6] npm, Inc. "npm-ci — Clean Install a Project." https://docs.npmjs.com/cli/v11/commands/npm-ci, 2024. Documents `npm ci` (clean install from lockfile) vs `npm install`, requiring `package-lock.json`.
