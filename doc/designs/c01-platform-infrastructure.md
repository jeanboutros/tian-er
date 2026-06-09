# C01 — Platform Infrastructure

**Status:** Phase A Design — guides Phase B implementation  
**Author:** Code Architect  
**Date:** 2026-06-08  
**Dependencies:** None (Layer 0 root component)  
**Blocks:** C02, C03, C05, C13 (and transitively C04, C06, C07, C08, C09, C10, C11, C12)

---

## 1. Overview

### 1.1 Purpose

C01 Platform Infrastructure is the **host bootstrap layer** for the entire Tian'er Signal Intelligence Platform. It provisions the Raspberry Pi Compute Module 5 operating environment — the OS, users, groups, file capabilities, udev device rules, file system layout, container runtime, and configuration scaffolding — upon which every other component depends. Without C01, no other component can start.

### 1.2 Scope

C01 covers everything that must exist on the bare-metal host **before any container starts**:

| In Scope | Out of Scope |
|----------|-------------|
| OS verification and package installation | Database schema (C02) |
| System user and group creation | Sniffer wrapper scripts (C03) |
| file capabilities for `dumpcap` | PCAP rotation logic (C04) |
| udev rules for USB sniffer hardware | Ingest bridge build (C05) |
| File system layout, ownership, and `tmpfiles.d` | Container image builds (C14) |
| Rootless Podman installation and `systemd-linger` | systemd unit definitions (C12) |
| Quadlet host prerequisites (user lingering, volume directories) | Grafana provisioning (C11) |
| Secrets directory bootstrap and permissions | |
| Supply chain integrity configuration | |

### 1.3 Boundaries

C01 is the **only component that runs directly on the host** (not in a container). Everything it creates — users, directories, device nodes, capabilities, packages — is inherited by containers via bind-mounts, `--device`, and `--group-add`. C01 is idempotent: all scripts must be safe to re-run without side effects.

### 1.4 Position in the System

```
┌──────────────────────────────────────────────────────────────────┐
│                        C01 PLATFORM HOST                         │
│  Users/Groups → udev → tmpfiles → Filesystem → Podman + Quadlet  │
└──────┬───────┬───────┬───────┬───────┬───────┬──────────────────┘
       │       │       │       │       │       │
       ▼       ▼       ▼       ▼       ▼       ▼
    ┌──────┐┌──────┐┌──────┐┌──────┐┌──────┐┌──────┐
    │ C02  ││ C03  ││ C04  ││ C05  ││ C06  ││ C07+ │
    │  DB  ││Capture││Rotate││Ingest││ Gap  ││Deep  │
    └──────┘└──────┘└──────┘└──────┘└──────┘└──────┘
```

C01 sits at Layer 0 in the build sequence (see component-breakdown.md §4.1). It is the **root dependency** of the critical path `C01 → C02 → C03 → C05 → C09 → C10 → C12`.

---

## 2. High-Level Architecture (HLA)

### 2.1 Hardware Topology

```
┌─────────────────────────────────────────────────────────────────┐
│                   Raspberry Pi Compute Module 5                   │
│                         (CM5, 8 GB RAM)                           │
│                                                                   │
│  ┌──────────┐  ┌─────────────────────────────────────────────┐  │
│  │  eMMC    │  │              BCM2712 SoC                     │  │
│  │  32 GB   │  │  4× Cortex-A76 @ 2.4 GHz                    │  │
│  │ (OS +    │  │  VideoCore VII GPU                            │  │
│  │  system  │  │                                               │  │
│  │  images) │  └──────────────────┬──────────────────────────┘  │
│  └──────────┘                     │                              │
│                                   │ PCIe 2.0 x1                  │
│  ┌──────────┐                     │                              │
│  │ SD Card  │  ┌──────────────────┴──────────────────────────┐  │
│  │ (data    │  │              USB 2.0 Host                    │  │
│  │  volume) │  │  2× internal + 1× external (Type-C)         │  │
│  └──────────┘  └──────────────────┬──────────────────────────┘  │
│                                   │                              │
│                      ┌────────────┴────────────┐                │
│                      │    Powered USB 2.0 Hub   │                │
│                      │    (D-04 resolution)      │                │
│                      │    4× downstream ports     │                │
│                      └──┬──────┬──────┬──────┬───┘                │
│                         │      │      │      │                    │
│                    ┌────┴┐ ┌───┴──┐ ┌───┴──┐ ┌───┴──┐            │
│                    │Uber-│ │nRF   │ │nRF   │ │nRF   │            │
│                    │tooth│ │#0    │ │#1    │ │#2    │            │
│                    │One  │ │52840 │ │52840 │ │52840 │            │
│                    └─────┘ └──────┘ └──────┘ └──────┘            │
└─────────────────────────────────────────────────────────────────┘
```

**Key hardware decisions:**

| Item | Choice | Reference |
|------|--------|-----------|
| Compute module | Raspberry Pi CM5, 8 GB RAM | Inception §3 |
| Primary storage | eMMC 32 GB (OS, container images, PostgreSQL data) | Inception §3 |
| Secondary storage | SD card (PCAP data, `/var/lib/tianer/pcap/`) | Inception §3 |
| USB topology | Powered USB 2.0 hub for 4 dongles | D-04 |
| Dongle count | 1–4 configurable (1 Ubertooth One + up to 3× nRF52840) | Inception §3 |
| nRF PID | 1915:522A (sniffer) + 1915:520f (DFU/bootloader) | D-03 |

### 2.2 Storage Layout

```
eMMC 32 GB                              SD Card (data volume)
├── / (root filesystem)                 ├── /var/lib/tianer/
│   ├── /boot/                              ├── pcap/           (V02, ~25 GB allocated)
│   │   └── firmware/                       └── data/           (V05, ~2 GB allocated)
│   ├── /etc/
│   │   ├── tianer/           (V01, config)
│   │   │   ├── tianer.env
│   │   │   ├── blesniff.env
│   │   │   ├── secrets/      (mode 0700)
│   │   │   └── sniffers.yaml
│   │   ├── tmpfiles.d/
│   │   │   └── tianer.conf
│   │   ├── udev/rules.d/
│   │   │   └── 99-tianer.rules
│   │   └── containers/systemd/
│   │       └── *.container     (Quadlet files)
│   ├── /var/
│   │   ├── lib/tianer/         (tianer homedir)
│   │   ├── run/tianer/         (V03, tmpfs — FIFOs)
│   │   └── log/tianer/         (V04)
│   ├── /opt/tianer/            (Python venvs)
│   ├── /usr/share/tianer/      (static: frontend, migrations)
│   └── /usr/local/bin/         (compiled binaries)
│           ├── blesniff-ingest
│           └── blesniff-deep-parse
└── /var/lib/containers/         (Podman storage, images)
```

**Volume-to-path mapping** (from storage-strategy.md):

| Volume ID | Host Path | Content | Size Budget |
|-----------|-----------|---------|-------------|
| V01 | `/etc/tianer/` | Config files, TLS certs, secrets | < 10 MB |
| V02 | `/var/lib/tianer/pcap/` | Rotating PCAP files | 25 GB (14-day retention) |
| V03 | `/var/run/tianer/` | FIFOs (tmpfs, ephemeral) | < 100 MB in RAM |
| V04 | `/var/log/tianer/` | Structured logs | 5 GB |
| V05 | `/var/lib/tianer/data/` | Deep parser JSONL output | 2 GB |
| V06 | Podman volume (`tianer-postgres-data`) | PostgreSQL data | 5 GB (on eMMC) |
| V07 | Podman volume (`tianer-grafana-data`) | Grafana dashboards + sqlite | 1 GB |

### 2.3 USB Power Budget

The Raspberry Pi CM5 USB 2.0 host controller provides up to **500 mA per port** per the USB 2.0 specification. With 4 dongles requiring sustained power, a powered hub is mandatory (D-04).

| Device | Typical Draw | Peak Draw | Notes |
|--------|-------------|-----------|-------|
| Ubertooth One | ~150 mA | ~200 mA | LPC175x MCU + CC2400 radio |
| nRF52840 Dongle (PCA10059) | ~15 mA | ~30 mA | ARM Cortex-M4, BLE radio TX |
| 3× nRF52840 Dongles | ~45 mA | ~90 mA | Total for 3 dongles |
| **Total (4 dongles)** | **~195 mA** | **~290 mA** | Well within 500 mA per-port budget |

**Power hub requirement:** A powered USB 2.0 hub rated for at least 2.5W (500 mA at 5V) total downstream delivery. The hub must have its own 5V power supply, not drawing from the CM5. A 4-port hub with a 2A power adapter provides generous headroom.

### 2.4 Thermal Envelope

The Raspberry Pi CM5 is rated for **ambient temperature 0°C to 85°C** (commercial grade) with the SoC throttling beginning at **85°C** junction temperature.

| Scenario | SoC Load | Ambient | Est. SoC Temp | Risk |
|----------|---------|---------|---------------|------|
| Idle (no sniffers) | < 5% | 25°C | ~40°C | None |
| 4 sniffers + ingest | ~15–25% | 25°C | ~55°C | None |
| 4 sniffers + deep parse | ~40–60% | 25°C | ~70°C | Low |
| 4 sniffers + deep parse | ~40–60% | 40°C (enclosure) | ~82°C | Moderate — throttling possible |

**Mitigation:**
- The CM5 benefits from the carrier board's thermal design. If deploying in a sealed enclosure, a **passive heatsink** (≥15×15×15 mm aluminum finned) is recommended.
- The host exposes `/sys/class/thermal/thermal_zone0/temp` (SoC temperature in millidegrees C). This is monitored as part of C13 observability.
- Throttling events are logged by the kernel (`vcgencmd get_throttled`), inspectable via `journalctl`.

---

## 3. Data Model

### 3.1 User/Group/Capability Matrix

C01 defines exactly one service user with no login shell, one primary group, and three supplementary groups. No other runtime users are created.

| Entity | Type | Name | Purpose | Created By |
|--------|------|------|---------|-----------|
| System user | user | `tianer` | Runs all platform processes (UID allocated by `useradd --system`) | `deploy/scripts/create-user.sh` |
| Primary group | group | `tianer` | Owns service files, directories, and running processes | `useradd --user-group` |
| USB access | group | `plugdev` | Grants `/dev/tianer/ubertooth*` and nRF raw USB access via udev | `usermod -aG plugdev` |
| Serial access | group | `dialout` | Grants `/dev/ttyACM*` access for nRF CDC ACM serial | `usermod -aG dialout` |
| Capture capability | group | `wireshark` | Grants `dumpcap` invocation (file `cap_net_raw+cap_net_admin+eip`) | `usermod -aG wireshark` |
| Container socket | group | `tianer` | Rootless Podman socket (`/run/user/$(id -u tianer)/podman/podman.sock`) | Automatic via `loginctl enable-linger` |

**No sudoers entries are created.** The `tianer` user has no password, no login shell (`/usr/sbin/nologin`), and no `sudo` privileges. The only legitimate `sudo` is by the human operator at install time.

### 3.2 Entity Relationships

```
tianer (UID=system, homedir=/var/lib/tianer)
  │
  ├── Primary: group tianer (GID=system)
  │     └── Owns: /var/lib/tianer/, /var/run/tianer/, /var/log/tianer/
  │
  ├── Supplementary: plugdev (existing Debian group)
  │     └── Grants: /dev/tianer/ubertooth*, nRF USB raw devices
  │
  ├── Supplementary: dialout (existing Debian group)
  │     └── Grants: /dev/ttyACM* (nRF serial console)
  │
  └── Supplementary: wireshark (created by wireshark-common package)
        └── Grants: /usr/bin/dumpcap (cap_net_raw,cap_net_admin+eip)
```

### 3.3 Additional System Groups (Not Owned by tianer)

| Group | Package | Auto-Created | tianer Member? |
|-------|---------|-------------|----------------|
| `wireshark` | `wireshark-common` | Yes (via `dpkg-reconfigure`) | Yes |
| `plugdev` | `udev` (system default) | Yes | Yes |
| `dialout` | `systemd` (system default) | Yes | Yes |
| `render` | Mesa/gpu drivers | Yes | No (not needed) |
| `video` | Mesa/gpu drivers | Yes | No (not needed) |
| `docker` | — | No (Podman, not Docker) | N/A |

### 3.4 File Capability Bindings

| Binary | Capability Set | Set By | Purpose |
|--------|---------------|--------|---------|
| `/usr/bin/dumpcap` | `cap_net_raw,cap_net_admin+eip` | `dpkg-reconfigure wireshark-common` | Capture from FIFO (tshark delegates to dumpcap) |
| `/usr/bin/newuidmap` | (none, SUID-root by default) | `shadow` package | Used internally by rootless Podman for user namespace mapping |

No other binaries receive custom capabilities.

---

## 4. Low-Level Architecture (LLA)

### 4.1 Module Decomposition

C01 consists of **seven bootstrap phases**, each implemented as an idempotent script or command sequence:

```
setup.sh (orchestrator)
  │
  ├── Phase 1: OS Verification
  │     └── Verify /etc/os-release matches "Raspberry Pi OS" + "Debian GNU/Linux 13"
  │
  ├── Phase 2: Package Installation
  │     ├── deploy/apt-packages.txt → apt-get install (idempotent)
  │     └── GPG key + apt repo additions (PostgreSQL, Grafana, NodeSource, Podman)
  │
  ├── Phase 3: User and Group Creation
  │     └── deploy/scripts/create-user.sh
  │
  ├── Phase 4: Filesystem Layout
  │     ├── deploy/scripts/create-dirs.sh → install -d paths with ownership
  │     ├── /etc/tmpfiles.d/tianer.conf → systemd-tmpfiles --create
  │     └── deploy/scripts/generate-secrets.sh (first-run only)
  │
  ├── Phase 5: Hardware Access Setup
  │     ├── deploy/scripts/setup-wireshark.sh → dpkg-reconfigure + setcap
  │     └── udev rules install → 99-tianer.rules → udevadm control --reload
  │
  ├── Phase 6: Container Runtime
  │     └── Podman install + loginctl enable-linger tianer
  │
  └── Phase 7: Quadlet Deployment
        └── Copy *.container, *.pod, *.volume, *.network files to /etc/containers/systemd/
```

### 4.2 Phase 1: OS Verification

```bash
#!/usr/bin/env bash
set -euo pipefail

# Verify we're on Raspberry Pi OS 64-bit Trixie (Debian 13)
if ! grep -q 'Raspberry Pi OS' /etc/os-release; then
    echo "ERROR: This setup script requires Raspberry Pi OS (Trixie, Debian 13)." >&2
    exit 1
fi

if ! grep -q 'VERSION_ID="13"' /etc/os-release; then
    echo "ERROR: This setup script requires Debian 13 (Trixie). Found: $(grep VERSION_ID /etc/os-release)" >&2
    exit 1
fi

# Verify 64-bit
if [[ "$(uname -m)" != "aarch64" ]]; then
    echo "ERROR: This setup script requires ARM64 (aarch64). Found: $(uname -m)" >&2
    exit 1
fi

echo "OS verification passed: Raspberry Pi OS 64-bit Trixie (Debian 13) on aarch64"
```

### 4.3 Phase 2: Package Installation

#### 4.3.1 Core Package List (`deploy/apt-packages.txt`)

```
# System
systemd-container
dbus-user-session
uidmap
slirp4netns
fuse-overlayfs
catatonit

# Container runtime
podman

# C++ build chain
build-essential
g++-14
cmake
libpqxx-dev
libpcap-dev
nlohmann-json3-dev

# Python
python3
python3-dev
python3-venv
python3-pip

# Capture tools
tshark
wireshark-common

# Database client
postgresql-client-17

# Utilities
zstd
curl
jq
git
```

#### 4.3.2 External Apt Repositories

| Repository | Package | GPG Key URL | Purpose |
|-----------|---------|------------|---------|
| PostgreSQL PGDG | `postgresql-17`, `postgresql-server-dev-17` | `https://www.postgresql.org/media/keys/ACCC4CF8.asc` | Database server |
| TimescaleDB | `timescaledb-2-postgresql-17` | `https://packagecloud.io/timescale/timescaledb/gpgkey` | Time-series extension |
| Grafana | `grafana` | `https://apt.grafana.com/gpg.key` | Dashboard server |
| NodeSource | `nodejs` (v24.x) | `https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key` | Frontend build (dev only) |

**Supply chain integrity (Q10):** All GPG keys are verified against known fingerprints before import. `setup.sh` compares each downloaded GPG key against a hardcoded SHA256 fingerprint. If a key has changed, the script halts with an explicit warning.

#### 4.3.3 Installation Command Pattern

```bash
# Idempotent: apt-get install with --no-install-recommends to keep image small
xargs -a deploy/apt-packages.txt apt-get install -y --no-install-recommends
```

### 4.4 Phase 3: User and Group Creation

`deploy/scripts/create-user.sh` — full script as specified in inception.md §6.1:

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

**Idempotency guarantee:** `useradd` is guarded by `id -u`; `usermod -aG` (append flag) preserves existing memberships; `groupadd` is guarded by `getent group`.

### 4.5 Phase 4: Filesystem Layout

#### 4.5.1 Persistent Directories (`deploy/scripts/create-dirs.sh`)

```bash
#!/usr/bin/env bash
set -euo pipefail

# tianer-owned paths — persistent data
install -d -o tianer -g tianer -m 0750 /var/lib/tianer
install -d -o tianer -g tianer -m 0750 /var/lib/tianer/pcap
install -d -o tianer -g tianer -m 0750 /var/lib/tianer/data
install -d -o tianer -g tianer -m 0700 /etc/tianer
install -d -o tianer -g tianer -m 0700 /etc/tianer/secrets

# Root-owned paths — static content
install -d -o root -g root -m 0755 /opt/tianer
install -d -o root -g root -m 0755 /usr/share/tianer
install -d -o root -g root -m 0755 /usr/share/tianer/frontend
install -d -o root -g root -m 0755 /usr/share/tianer/migrations
install -d -o root -g root -m 0755 /usr/local/lib/tianer/wrap
```

#### 4.5.2 Runtime Directories and FIFOs (`/etc/tmpfiles.d/tianer.conf`)

```
# Type Path                    Mode UID    GID     Age Argument
d      /var/run/tianer         0750 tianer tianer  -   -
d      /var/log/tianer         0750 tianer tianer  -   -
p      /var/run/tianer/ut1.fifo        0660 tianer tianer  -   -
p      /var/run/tianer/nrf1.fifo       0660 tianer tianer  -   -
p      /var/run/tianer/nrf2.fifo       0660 tianer tianer  -   -
p      /var/run/tianer/nrf3.fifo       0660 tianer tianer  -   -
```

Applied immediately (no reboot needed) and at every boot:

```bash
systemd-tmpfiles --create /etc/tmpfiles.d/tianer.conf
```

#### 4.5.3 Secrets Generation (`deploy/scripts/generate-secrets.sh`)

```bash
#!/usr/bin/env bash
set -euo pipefail

SECRETS_DIR="/etc/tianer/secrets"
DB_PW_FILE="${SECRETS_DIR}/db_password"
API_KEY_FILE="${SECRETS_DIR}/api_key"
GRAFANA_DB_PW_FILE="${SECRETS_DIR}/grafana_db_password"

# Idempotent: only generate if file does not exist
if [[ ! -f "${DB_PW_FILE}" ]]; then
    openssl rand -base64 32 > "${DB_PW_FILE}"
    chmod 0600 "${DB_PW_FILE}"
    chown tianer:tianer "${DB_PW_FILE}"
fi

if [[ ! -f "${API_KEY_FILE}" ]]; then
    openssl rand -base64 32 > "${API_KEY_FILE}"
    chmod 0600 "${API_KEY_FILE}"
    chown tianer:tianer "${API_KEY_FILE}"
fi

if [[ ! -f "${GRAFANA_DB_PW_FILE}" ]]; then
    openssl rand -base64 32 > "${GRAFANA_DB_PW_FILE}"
    chmod 0600 "${GRAFANA_DB_PW_FILE}"
    chown tianer:tianer "${GRAFANA_DB_PW_FILE}"
fi
```

### 4.6 Phase 5: Hardware Access

#### 4.6.1 Wireshark / dumpcap Setup (`deploy/scripts/setup-wireshark.sh`)

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

# 3. Verify capabilities applied
getcap /usr/bin/dumpcap | grep -q 'cap_net_admin.*cap_net_raw\|cap_net_raw.*cap_net_admin'

# 4. tianer user is already in the wireshark group via create-user.sh.
```

#### 4.6.2 udev Rules (`deploy/udev/99-tianer.rules`)

```
# Ubertooth One - USB-only access via libusb. Group plugdev.
SUBSYSTEM=="usb", ATTRS{idVendor}=="1d50", ATTRS{idProduct}=="6002", \
    GROUP="plugdev", MODE="0660", \
    SYMLINK+="tianer/ubertooth%n"

# Nordic nRF52840 Dongle running nRF Sniffer firmware.
# Dual PID support per D-03: 522A (sniffer mode) + 520f (DFU/bootloader mode).
# Both provide raw USB (plugdev) and CDC ACM serial (dialout).
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

**Apply rules:**
```bash
sudo install -m 0644 deploy/udev/99-tianer.rules /etc/udev/rules.d/99-tianer.rules
sudo udevadm control --reload-rules
sudo udevadm trigger
```

**Persistent naming (Q4):** The `SYMLINK+="tianer/..."` directives create stable device paths:
- `/dev/tianer/ubertooth0` — the first (and typically only) Ubertooth One
- `/dev/tianer/nrf0` — first nRF52840 dongle
- `/dev/tianer/nrf1` — second nRF52840 dongle
- `/dev/tianer/nrf2` — third nRF52840 dongle

These names persist across reboots and USB port reconnections. The `%n` kernel device number suffix provides differentiation when multiple identical devices are connected. For deterministic ordering of multiple nRF dongles, consider `ATTRS{serial}` matching or port-path based rules in future refinement.

### 4.7 Phase 6: Container Runtime — Rootless Podman

#### 4.7.1 Installation

Podman is installed via apt from the Debian Trixie repository:

```bash
apt-get install -y podman slirp4netns fuse-overlayfs uidmap catatonit
```

No Docker packages are installed. No `docker` group is created.

#### 4.7.2 Rootless Configuration

All containers run as the `tianer` user — not as root, not as a privileged Podman daemon:

```bash
# Enable lingering for the tianer user so that systemd --user services
# (Quadlet-managed containers) start at boot and persist after logout.
loginctl enable-linger tianer
```

This creates `/var/lib/systemd/linger/tianer`, which tells systemd to start the `tianer` user's `systemd --user` instance at boot and keep it running indefinitely.

#### 4.7.3 User Namespace Prerequisites

Rootless Podman requires:
- `/etc/subuid` — subordinate UID range for user namespace mapping
- `/etc/subgid` — subordinate GID range for user namespace mapping

The `uidmap` package creates these on Debian. Verify:

```bash
grep tianer /etc/subuid /etc/subgid
# Expected output (example):
# /etc/subuid:tianer:165536:65536
# /etc/subgid:tianer:165536:65536
```

#### 4.7.4 Podman Storage

Rootless Podman stores its data under `/var/lib/containers/storage/` (system-level) or `$HOME/.local/share/containers/` (user-level). For the `tianer` user, the storage path is:

```
/var/lib/tianer/.local/share/containers/
```

This is on the eMMC by default. Container images consume disk space here; images are built once during deployment and pulled only on updates. Total image storage budget: **under 600 MB** for all 11 container images (each under 200 MB, but image layers are deduplicated by Podman's overlay storage driver).

### 4.8 Phase 7: Quadlet Deployment

#### 4.8.1 Quadlet Overview

Quadlet is a systemd generator that converts `.container` files (and `.pod`, `.volume`, `.network` files) into systemd units at boot. Quadlet is part of Podman ≥ 4.4 and is the recommended mechanism for running Podman containers under systemd on the host.

#### 4.8.2 Deployment

Place Quadlet unit files in `/etc/containers/systemd/` (system-wide, not per-user):

```bash
install -m 0644 deploy/containers/*.container /etc/containers/systemd/
install -m 0644 deploy/containers/*.pod       /etc/containers/systemd/
install -m 0644 deploy/containers/*.volume     /etc/containers/systemd/
install -m 0644 deploy/containers/*.network    /etc/containers/systemd/
systemctl daemon-reload
```

Quadlet automatically picks up these files on `daemon-reload` and generates corresponding `systemd --user` units for the `tianer` user (via linger).

#### 4.8.3 Rootless Podman Systemd Registration

Per the container security recommendation, all containers run as the `tianer` user via rootless Podman. The Quadlet files specify `User=tianer` in the generated systemd units, which means `systemctl --user` under the `tianer` user controls the containers:

```bash
# Start all containers as tianer user
machinectl shell tianer@ /bin/bash -c 'systemctl --user start tianer.target'

# Or, from root:
systemctl --user -M tianer@ start tianer.target
```

### 4.9 Container Host Prerequisites — Summary Table

| Prerequisite | Provided By | Verified By |
|-------------|------------|-------------|
| `tianer` user with `plugdev,dialout,wireshark` | `create-user.sh` | `id tianer` |
| `/etc/subuid` + `/etc/subgid` with tianer row | `uidmap` package | `grep tianer /etc/subuid` |
| `loginctl enable-linger tianer` | Phase 6 | `loginctl show-user tianer -p Linger` |
| `dbus-user-session` package | `apt-packages.txt` | `dpkg -l dbus-user-session` |
| `slirp4netns` or `pasta` for rootless networking | `apt-packages.txt` | `which slirp4netns` |
| `fuse-overlayfs` for rootless storage driver | `apt-packages.txt` | `which fuse-overlayfs` |
| Host directories owned by `tianer:tianer` | `create-dirs.sh` | `stat -c '%U:%G' /var/lib/tianer` |
| USB devices with correct udev permissions | `99-tianer.rules` | `ls -l /dev/tianer/` |
| Secrets files (mode 0600) | `generate-secrets.sh` | `stat -c '%a' /etc/tianer/secrets/*` |

---

## 5. Inter-Component Contracts

### 5.1 File System Layout Contract (FS-LAYOUT)

All other components depend on the paths provisioned by C01. These paths are **immutable contracts** — changing any path requires coordinated updates to C02 through C14.

| Path | Owner:Group | Mode | Purpose | Consumers |
|------|------------|------|---------|-----------|
| `/etc/tianer/` | `tianer:tianer` | 0700 | Config root (V01) | All containers (:ro bind-mount) |
| `/etc/tianer/tianer.env` | `root:tianer` | 0640 | Shared environment variables | All systemd units (EnvironmentFile) |
| `/etc/tianer/blesniff.env` | `root:tianer` | 0640 | Bluetooth module config | C03, C04, C05, C06 |
| `/etc/tianer/sniffers.yaml` | `root:tianer` | 0640 | Sniffer definitions | C03, C12 |
| `/etc/tianer/secrets/` | `tianer:tianer` | 0700 | Secret files | C02, C05, C06, C09, C11 |
| `/etc/tianer/secrets/db_password` | `tianer:tianer` | 0600 | PostgreSQL main password | C02, C05, C09 |
| `/etc/tianer/secrets/api_key` | `tianer:tianer` | 0600 | API authentication key | C09, C10 |
| `/etc/tianer/secrets/grafana_db_password` | `tianer:tianer` | 0600 | Grafana read-only password | C11 |
| `/var/lib/tianer/` | `tianer:tianer` | 0750 | Persistent data root | C02(via volume), C04, C06, C07, C08 |
| `/var/lib/tianer/pcap/` | `tianer:tianer` | 0750 | PCAP file storage (V02) | C03(w), C04(rw), C06(ro), C07(ro) |
| `/var/lib/tianer/data/` | `tianer:tianer` | 0750 | Deep parser output (V05) | C07(w), C08(ro) |
| `/var/lib/tianer/heartbeat/` | `tianer:tianer` | 0750 | Heartbeat timestamp files | C03, C06 |
| `/var/run/tianer/` | `tianer:tianer` | 0750 | Runtime FIFOs + sockets (V03, tmpfs) | C03, C05 |
| `/var/run/tianer/ut1.fifo` | `tianer:tianer` | 0660 | Ubertooth PCAP stream FIFO | C03(w), C03(tshark r) |
| `/var/run/tianer/nrf1.fifo` | `tianer:tianer` | 0660 | nRF #1 PCAP stream FIFO | C03(w), C03(tshark r) |
| `/var/run/tianer/nrf2.fifo` | `tianer:tianer` | 0660 | nRF #2 PCAP stream FIFO | C03(w), C03(tshark r) |
| `/var/run/tianer/nrf3.fifo` | `tianer:tianer` | 0660 | nRF #3 PCAP stream FIFO | C03(w), C03(tshark r) |
| `/var/log/tianer/` | `tianer:tianer` | 0750 | Structured logs (V04) | All containers (:rw bind-mount) |
| `/opt/tianer/` | `root:root` | 0755 | Python virtual environments | C06, C08, C09 |
| `/usr/share/tianer/` | `root:root` | 0755 | Static artifacts (frontend, migrations) | C02, C09, C10 |
| `/usr/local/bin/blesniff-ingest` | `root:root` | 0755 | Ingest bridge binary | C05 |
| `/usr/local/bin/blesniff-deep-parse` | `root:root` | 0755 | Deep parser binary | C07 |

**Contract ID:** `FS-LAYOUT` is the informal contract name used throughout component design documents. Any change to the above table must update the component that creates the path (C01) and all listed consumers.

### 5.2 Device Naming Contract (DEV-NAMES)

Persistent udev device symlinks are the **only supported mechanism** for container USB passthrough. Components reference devices by their `/dev/tianer/` symlink, never by raw `/dev/bus/usb/` or `/dev/ttyACM*` paths.

| Symlink | Hardware | USB VID:PID | Container Pass-through | Consumer |
|---------|----------|------------|------------------------|----------|
| `/dev/tianer/ubertooth0` | Ubertooth One | `1d50:6002` | `--device /dev/tianer/ubertooth0` | C03 (ubertooth-wrap) |
| `/dev/tianer/nrf0` | nRF52840 Dongle #1 | `1915:522a` or `1915:520f` | `--device /dev/tianer/nrf0` | C03 (nrf-wrap) |
| `/dev/tianer/nrf1` | nRF52840 Dongle #2 | `1915:522a` or `1915:520f` | `--device /dev/tianer/nrf1` | C03 (nrf-wrap) |
| `/dev/tianer/nrf2` | nRF52840 Dongle #3 | `1915:522a` or `1915:520f` | `--device /dev/tianer/nrf2` | C03 (nrf-wrap) |

**Contract ID:** `DEV-NAMES`

**Rules:**
1. All container `--device` mounts reference the `/dev/tianer/*` symlink, not the raw device node.
2. Container Quadlet files include `--group-add keep-groups` so that the container process inherits the `plugdev` and `dialout` group memberships needed to access the device.
3. If a device is not present at container start, the container must gracefully degrade (log warning, pause, retry) — not crash-loop.
4. Adding a new sniffer type (future module) requires a new udev rule line with a new symlink under `/dev/tianer/`.

### 5.3 Container Runtime Contract (CONTAINER-RUNTIME)

All components (C02–C14) assume the container runtime environment defined by C01:

| Property | Value |
|----------|-------|
| Container engine | Podman (rootless, user `tianer`) |
| Orchestration | Quadlet `.container` files → systemd `--user` units |
| User lingering | `loginctl enable-linger tianer` (enabled) |
| Network mode | `tianer-capture` pod: `Network=none`; `tianer-platform` pod: bridge network `tianer-net` |
| Image pull policy | Digest-pinned (`@sha256:`); never `:latest` |
| Volume mounts | Per storage-strategy.md access matrix |
| USB passthrough | `--device /dev/tianer/<name>` + `--group-add keep-groups` |
| Capabilities | `--cap-drop ALL` default; only C03 containers get `CAP_NET_RAW` + `CAP_NET_ADMIN` |
| Privileged mode | Never — zero `--privileged` containers |
| Secrets | EnvironmentFile from host; never baked into images |

---

## 6. Failure Modes & Recovery

### 6.1 Failure Catalog

| # | Failure Mode | Detection | Impact | Recovery |
|---|-------------|-----------|--------|----------|
| F-C01-1 | **USB dongle disconnected** | `udevadm monitor`; sniffer process exits; heartbeat stops | One sniffer channel lost; others continue (graceful degradation, PF-4) | Reconnect dongle; udev recreates symlink; sniffer container auto-restarts via systemd `Restart=on-failure` |
| F-C01-2 | **Powered USB hub power loss** | All USB devices disappear simultaneously; all sniffer heartbeats stop within 1 minute | All capture pipelines halted; gap detector records gaps; PCAP rotation stops | Restore hub power; all devices re-enumerate; udev recreates symlinks; all sniffer containers auto-restart; gap detector backfills from last PCAP file |
| F-C01-3 | **eMMC disk full** | `df -h /` monitored by host-level metric; `BLESNIFF_EMERGENCY_PURGE` trigger in C04 | Container images cannot update; PostgreSQL may refuse writes; new logs dropped | C04 emergency purge of oldest uncompressed PCAP files; operator frees space; alert fires |
| F-C01-4 | **SD card disk full** | `df -h /var/lib/tianer/pcap` monitored; rotation script exits non-zero | PCAP rotation fails; oldest PCAP files cannot be rotated or deleted; new capture may lose data if tee blocks on full FS | C04 emergency purge; operator replaces SD card if full; PCAP files are source of truth — never delete older than retention without operator confirmation |
| F-C01-5 | **Thermal throttling** | SoC temp > 85°C; kernel log `throttled=0x...`; `vcgencmd get_throttled` | CPU clocks reduced; ingest latency increases; deep parser throughput drops | Passive heatsink; active cooling if sustained; reduce concurrent deep parser jobs |
| F-C01-6 | **Power loss / unclean shutdown** | Kernel boot log shows unexpected restart; systemd-journald recovers | tmpfs `/var/run/tianer/` lost (FIFOs); in-flight container state lost | On reboot: `systemd-tmpfiles` recreates FIFOs; `systemd --user` starts via linger; containers auto-restart; gap detector backfills missed buckets from PCAP; PostgreSQL WAL recovery; no clean shutdown required (PF-8) |
| F-C01-7 | **Container startup failure** | `systemctl --user is-failed <unit>`; container exit code ≠ 0 | That component unavailable; gap detector marks buckets as gaps; alerts fire (PF-5) | systemd restarts per `Restart=on-failure` + `RestartSec=5s`; if persistent, operator investigates via `podman logs` |
| F-C01-8 | **Podman storage corruption** | `podman system info` fails; image pull fails; containers won't start | All containers affected; complete platform outage | `podman system reset` (destructive — clears all images and containers); rebuild and redeploy from CI artifacts; PostgreSQL data in Podman volume (V06) should be backed up first |
| F-C01-9 | **udev rule not loaded** | `ls -l /dev/tianer/` returns empty; sniffer process gets EACCES | Sniffer containers cannot access USB devices | `udevadm control --reload-rules && udevadm trigger`; verify rule file exists; reinstall from `deploy/udev/99-tianer.rules` |
| F-C01-10 | **wireshark group missing / dumpcap capabilities lost** | `getcap /usr/bin/dumpcap` returns empty | tshark in containers gets EPERM; no capture possible | Re-run `deploy/scripts/setup-wireshark.sh`; verify `dpkg-reconfigure wireshark-common` was run |
| F-C01-11 | **Secrets file missing** | Container startup fails with "password file not found" | Service cannot authenticate to DB or serve API | Re-run `generate-secrets.sh`; if lost entirely, regenerate and update all consumers |

### 6.2 Host Reboot Recovery Sequence

The recovery sequence is fully automatic (PF-8: Crash-Only Design):

```
Boot
  │
  ├── kernel boots → mounts rootfs (eMMC) + SD card
  │
  ├── systemd starts
  │     │
  │     ├── systemd-tmpfiles --create (recreates /var/run/tianer/* FIFOs)
  │     │
  │     ├── loginctl enable-linger tianer → starts tianer user's systemd --user
  │     │     │
  │     │     ├── Quadlet generator reads /etc/containers/systemd/*.container
  │     │     ├── Converts to systemd --user units
  │     │     └── Starts tianer-postgres.service (standalone)
  │     │     └── Starts tianer-capture pod (sniffer + tshark + ingest)
  │     │     └── Starts tianer-platform pod (gapdetect, api, etc.)
  │     │     └── Starts tianer-grafana.service (standalone)
  │     │
  │     └── udev re-enumerates USB devices → creates /dev/tianer/* symlinks
  │
  └── ~15–30 seconds to full readiness (PF-10: quantified SLA)
```

**Acceptable recovery window:** Under 30 seconds from kernel boot to all containers ready. Containers with `Restart=on-failure` retry after `RestartSec=5s` if PostgreSQL is not yet accepting connections.

### 6.3 Recovery Service Dependency

| Recovery Agent | Failure It Handles | Is It Available After Reboot? |
|---------------|-------------------|-------------------------------|
| `systemd-tmpfiles` | FIFO loss | Yes — built-in systemd component |
| `systemd --user` (linger) | Container crash | Yes — starts at boot |
| `udev` | USB re-enumeration | Yes — kernel subsystem |
| Gap detector (C06) | Missed ingest buckets | Yes — auto-starts, backfills from PCAP |
| `create-user.sh` (re-run) | Corrupted user/group | Manual — operator intervention |

---

## 7. Observability

### 7.1 Host-Level Metrics

C01 exposes health signals that are consumable by C13 (Observability) and C11 (Grafana dashboards). These are host-level metrics, collected outside containers.

| Metric | Source | Collection Method | Labels |
|--------|--------|------------------|--------|
| `tianer_host_disk_usage_bytes` | `statvfs` on `/`, `/var/lib/tianer/pcap` | Host script / cron every 60s | `mountpoint` |
| `tianer_host_disk_usage_percent` | `df` | Same script | `mountpoint` |
| `tianer_host_memory_available_bytes` | `/proc/meminfo` | Same script | — |
| `tianer_host_cpu_temp_celsius` | `/sys/class/thermal/thermal_zone0/temp` | Same script | — |
| `tianer_host_cpu_throttled` | `vcgencmd get_throttled` | Host script | `reason` (undervoltage, arm_freq_cap, throttled, soft_temp_limit) |
| `tianer_usb_device_present` | `ls /dev/tianer/*` | Host script every 30s | `device` (ubertooth0, nrf0, nrf1, nrf2) |
| `tianer_fifo_present` | `test -p /var/run/tianer/*.fifo` | Host script at boot + every 60s | `name` |

### 7.2 Structured Logging

All Tian'er components use a consistent log format with the project keyword `TIANER` as a mandatory prefix. This enables single-pattern grep across all services and simple routing to Grafana Loki.

**Format:**
```
TIANER | {"ts":"ISO8601","level":"LEVEL","component":"NAME",...}
```

**Keyword:** Every log line emitted by any Tian'er component (host scripts, containers, C++ ingest bridge, Python gap detector, API server, frontend builds) MUST begin with the literal string `TIANER`. No exceptions. The keyword is followed by a pipe separator and a JSON object for structured field extraction.

**Log destination per component type:**

| Source | Destination | Collection |
|--------|------------|------------|
| Host bootstrap scripts (C01) | journald via stderr | journald → Loki via promtail |
| Container stdout/stderr | journald via Podman conmon | Same path |
| C++ services (C05, C07) | `/var/log/tianer/<component>.jsonl` (file) + stderr (journald) | Dual-write; file for local inspection, journald for Loki |
| Python services (C06, C08, C09) | journald via stderr | Python `logging` configured with journald handler |

**Structured fields:**

| Field | Required | Description |
|-------|----------|-------------|
| `ts` | Yes | ISO 8601 timestamp with timezone |
| `level` | Yes | One of: DEBUG, INFO, WARN, ERROR, FATAL |
| `component` | Yes | Component name: host, sniffer, tshark, ingest, gapdetect, deepparse, ml, api, grafana |
| `sniffer` | If applicable | Sniffer instance name (ut1, nrf1, nrf2, nrf3) |
| `msg` | Yes | Human-readable message |
| Additional fields | Optional | Component-specific key=value pairs |

**Examples:**
```
TIANER | {"ts":"2026-06-09T14:32:01Z","level":"INFO","component":"host","msg":"Created tianer system user","uid":999}
TIANER | {"ts":"2026-06-09T14:32:02Z","level":"INFO","component":"host","msg":"dumpcap capabilities verified","caps":"cap_net_raw,cap_net_admin+eip"}
TIANER | {"ts":"2026-06-09T14:32:15Z","level":"ERROR","component":"host","msg":"Failed to create directory","path":"/var/lib/tianer/pcap","error":"Permission denied"}
TIANER | {"ts":"2026-06-09T14:35:00Z","level":"INFO","component":"sniffer","sniffer":"ut1","msg":"Capture started","channel":37,"packets_per_sec":127}
TIANER | {"ts":"2026-06-09T14:40:00Z","level":"WARN","component":"ingest","sniffer":"ut1","msg":"DB connection lost","retry_in_s":5}
TIANER | {"ts":"2026-06-09T14:45:00Z","level":"INFO","component":"host","msg":"Host metrics collected","cpu_temp_c":48.8,"disk_usage_pct":62,"mem_avail_mb":3200}
```

**Grep usage:**
```bash
# All Tian'er logs across all components
grep '^TIANER ' /var/log/tianer/*.jsonl

# Only errors
grep '^TIANER |.*"level":"ERROR"' /var/log/tianer/*.jsonl

# From journald (all containers + host)
journalctl | grep '^TIANER '

# Specific component from journald
journalctl CONTAINER_NAME=tianer-ingest | grep '^TIANER '
```

**Loki integration:**
Promtail (deployed as a container in the platform pod) tails `/var/log/tianer/*.jsonl` and the journal. The pipeline stage extracts the JSON after `TIANER |` into structured labels:

```yaml
# promtail pipeline stage
selector:
    '{app="tianer"}'
    stages:
      - regex:
          expression: '^TIANER \| (?P<json>.*)'
      - json:
          expressions:
            level: level
            component: component
            msg: msg
            sniffer: sniffer
```

This gives Grafana Loki dashboards the ability to filter by `{component="ingest", level="ERROR"}` across all services with one query.

### 7.3 Health Checks

C01 provides one explicit health check script consumed by other components:

`/usr/local/bin/tianer-check-prereqs.sh`:
```bash
#!/usr/bin/env bash
set -euo pipefail

# Returns 0 if all prerequisites are met.

# User and groups
id tianer >/dev/null 2>&1 || { echo "FAIL: tianer user missing"; exit 1; }
for grp in plugdev dialout wireshark; do
    id -nG tianer | grep -qw "$grp" || { echo "FAIL: tianer not in $grp"; exit 1; }
done

# Capabilities
getcap /usr/bin/dumpcap 2>/dev/null | grep -q 'cap_net_raw' || { echo "FAIL: dumpcap capabilities missing"; exit 1; }

# Paths
for path in /var/lib/tianer /var/run/tianer /var/log/tianer /etc/tianer; do
    [[ -d "$path" ]] || { echo "FAIL: $path missing"; exit 1; }
    [[ "$(stat -c '%U' "$path")" == "tianer" ]] || { echo "FAIL: $path wrong owner"; exit 1; }
done

# udev rule installed
[[ -f /etc/udev/rules.d/99-tianer.rules ]] || { echo "FAIL: udev rule missing"; exit 1; }

# Podman installed and rootless working
which podman >/dev/null 2>&1 || { echo "FAIL: podman not installed"; exit 1; }

# systemd-linger enabled
loginctl show-user tianer -p Linger 2>/dev/null | grep -q 'Linger=yes' || { echo "FAIL: linger not enabled"; exit 1; }

# FIFOs created
for fifo in ut1 nrf1 nrf2 nrf3; do
    [[ -p "/var/run/tianer/${fifo}.fifo" ]] || { echo "FAIL: /var/run/tianer/${fifo}.fifo missing"; exit 1; }
done

echo "PREREQUISITES OK"
```

### 7.4 Alert Rules

Alert rules and thresholds are externalized to a configuration file, not hardcoded in scripts. This allows operators to tune alert sensitivity without modifying code or rebuilding containers.

**Configuration file:** `/etc/tianer/alerts.yaml`

```yaml
# /etc/tianer/alerts.yaml — Tian'er alert thresholds
# Mounted :ro into Prometheus/Grafana containers (V01)
# Changes take effect on next Prometheus reload (no restart needed)

alerts:
  disk:
    warning_percent: 80
    critical_percent: 90
    evaluation_interval: 60s
  
  thermal:
    throttle_duration_threshold: 60s        # CPU must be throttled for this long before alerting
    evaluation_interval: 30s
  
  usb:
    missing_duration_threshold: 30s         # Device must be absent for this long before alerting
    evaluation_interval: 30s
  
  fifo:
    missing_duration_threshold: 30s
    evaluation_interval: 60s
  
  memory:
    warning_percent: 85
    critical_percent: 95
    evaluation_interval: 60s
```

**Alert definitions** (Prometheus rules, referencing the config values):

| Alert | PromQL Expression | Severity | Action |
|-------|------------------|----------|--------|
| `TianerHostDiskWarning` | `tianer_host_disk_usage_percent > 80` | **Warning** | Notify operator |
| `TianerHostDiskCritical` | `tianer_host_disk_usage_percent > 90` for any mountpoint | **Critical** | Stop non-essential containers (deep parser); notify operator; emergency PCAP purge |
| `TianerHostThermalThrottle` | `tianer_host_cpu_throttled{reason="throttled"} == 1` for > 60s | **Warning** | Check cooling; reduce deep parser load |
| `TianerUSBDeviceMissing` | `tianer_usb_device_present == 0` for any device for > 30s | **Critical** | Check USB hub power; check dongle connection; gap detector will backfill |
| `TianerFifoMissing` | `tianer_fifo_present == 0` for any FIFO for > 30s | **Critical** | Re-run `systemd-tmpfiles --create`; sniffer will fail to start |
| `TianerHostMemoryWarning` | `(1 - tianer_host_memory_available_bytes / tianer_host_memory_total_bytes) * 100 > 85` | **Warning** | Check for memory leak; consider restarting containers |
| `TianerHostMemoryCritical` | `(1 - tianer_host_memory_available_bytes / tianer_host_memory_total_bytes) * 100 > 95` | **Critical** | OOM killer imminent; stop non-essential containers |

**Rationale for externalized config:**
- Thresholds are environment-specific (a Pi in a hot attic needs different thermal thresholds than a basement deployment)
- Changing a threshold should not require a code change or container rebuild
- The file is in V01 (`/etc/tianer/`), mounted `:ro` into the Prometheus container
- Prometheus reloads rules on `SIGHUP` or `POST /-/reload` — no restart needed
- The Prometheus rules file (`.yml`) references the alert thresholds; the `alerts.yaml` provides the tunable values that operators edit

---

## 8. Security Considerations

### 8.1 Disk Encryption (D-07)

**Decision:** Disk encryption (LUKS) is an **accepted risk for the MVP**. The Raspberry Pi CM5 has no TPM, making unattended boot with full disk encryption infeasible without an external key source (USB key, network unlock). For the MVP, the following compensating controls are in place:

- Physical security: the device is a physically secured appliance in a controlled environment.
- Database passwords are stored in mode 0600 files, never in world-readable locations.
- PCAP data (containing BLE MACs) is on the SD card — if physical access is gained, the attacker can read the card directly regardless of filesystem encryption.
- Post-MVP: evaluate LUKS with a TPM add-on board (e.g., LetsTrust TPM for Raspberry Pi) or an external keyfile on a USB drive that must be present at boot.

The risk is documented and accepted. The acceptance is logged in ADR-0001 (D-07). No disk encryption configuration is implemented in C01 for the MVP.

### 8.2 Physical Security

The device is a **physically secured appliance**:
- Located in a controlled-access environment.
- USB ports accessible only via the powered hub (dongles), not exposed to user plugging.
- SD card slot concealed or epoxied if physical tampering is a concern.
- No exposed network services beyond the API (bound to 127.0.0.1 by default).

### 8.3 Sudo Policy

| Rule | Rationale |
|------|-----------|
| No `tianer` user sudoers entry | The runtime service user must never escalate privileges |
| No `NOPASSWD` rules for any user | Install-time sudo requires explicit password entry by operator |
| No service calls `sudo` in `ExecStart=` | All required capabilities are granted via file capabilities, not sudo |
| `root` account disabled for SSH | Only the operator account (with SSH key) can log in; `sudo` from operator account for install |

**Verification:** `grep -r sudo deploy/containers/ deploy/systemd/` must return no results in runtime unit files.

### 8.4 User Privilege

The `tianer` user is a **system user** (UID in system range, typically < 1000):
- No login shell (`/usr/sbin/nologin`)
- No password (no entry in `/etc/shadow` or locked password)
- No `sudo` or `su` access
- Only supplementary groups: `plugdev`, `dialout`, `wireshark` — none are administrative groups
- Home directory is `/var/lib/tianer` (data directory, not `/home/tianer`)

### 8.5 File Permissions

| Path | Mode | Owner:Group | Justification |
|------|------|------------|---------------|
| `/etc/tianer/` | 0700 | `tianer:tianer` | Configuration data — root cannot read secrets dir? Actually root can always read. 0700 prevents other users from listing. |
| `/etc/tianer/secrets/` | 0700 | `tianer:tianer` | Prevents listing by non-tianer users |
| `/etc/tianer/secrets/db_password` | 0600 | `tianer:tianer` | Credential — only the service user can read |
| `/var/lib/tianer/pcap/` | 0750 | `tianer:tianer` | PCAP data — contains BLE MACs; group-readable for operator in `tianer` group |
| `/var/run/tianer/*.fifo` | 0660 | `tianer:tianer` | FIFO — writable by tianer and anyone in tianer group |
| `/var/log/tianer/` | 0750 | `tianer:tianer` | Logs — may contain device MACs |

### 8.6 Container Security Inheritance

C01 configures the **host-level security baseline** that containers inherit:
- `NoNewPrivileges=true` in generated Quadlet systemd units
- `--cap-drop ALL` as default in Quadlet `.container` files
- Only C03 containers receive `--cap-add CAP_NET_RAW,CAP_NET_ADMIN`
- `--group-add keep-groups` ensures `plugdev`, `dialout`, `wireshark` are available in container
- `--security-opt no-new-privileges` per container
- All container images digest-pinned (`podman pull image@sha256:...`)

### 8.7 Supply Chain Integrity (Q10)

All externally sourced software is verified:

| Artifact | Verification Method | Implementation |
|----------|-------------------|----------------|
| apt packages (Debian repos) | GPG apt repository signing | Standard Debian mechanism; GPG keys verified against hardcoded SHA256 before adding |
| apt packages (3rd-party repos) | GPG key fingerprint verification | `gpg --fingerprint` compared to expected fingerprint in `setup.sh` |
| nrfutil binary | SHA256 checksum | Downloaded from Nordic's official GitHub releases; checksum verified against published SHA256SUMS file |
| Ubertooth tools (from source) | Git tag + commit hash | `git checkout` specific verified tag; source built locally |
| Container images | Digest pinning (`@sha256:`) | All Quadlet `.container` files specify `Image=...@sha256:...`; never `:latest` |
| Python packages | Hash-pinned with `uv` | `uv lock` generates `uv.lock` with hashes; `uv sync --frozen` enforces |
| Node.js packages | `npm audit` + lockfile | `package-lock.json` with integrity hashes; `npm ci` for clean install |

---

## 9. Configuration

### 9.1 Environment Files

C01 creates the config directory and a shared environment template:

**`/etc/tianer/tianer.env`** (shared across all components):

```bash
# Tian'er Platform — Shared Environment
# Sourced by all systemd units via EnvironmentFile=

# Database (shared by all modules)
TIANER_DB_HOST=127.0.0.1
TIANER_DB_PORT=5432
TIANER_DB_NAME=tianer

# Paths (shared)
TIANER_CONFIG_DIR=/etc/tianer
TIANER_DATA_DIR=/var/lib/tianer
TIANER_RUN_DIR=/var/run/tianer
TIANER_LOG_DIR=/var/log/tianer

# API (shared)
TIANER_API_HOST=127.0.0.1
TIANER_API_PORT=8080

# Grafana (shared)
TIANER_GRAFANA_URL=http://127.0.0.1:3000
```

**`/etc/tianer/blesniff.env`** (Bluetooth module override):

```bash
# Tian'er v1 Bluetooth Module — BLESNIFF_* settings
# Inherits TIANER_* from tianer.env

# Database credentials
BLESNIFF_DB_USER=tianer_writer
BLESNIFF_DB_PASSWORD_FILE=/etc/tianer/secrets/db_password

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

# Paths
BLESNIFF_PCAP_DIR=/var/lib/tianer/pcap
BLESNIFF_FIFO_DIR=/var/run/tianer
BLESNIFF_LOG_DIR=/var/log/tianer
```

### 9.2 Sniffer Configuration Template

**`/etc/tianer/sniffers.yaml`** (deployed as template; operator edits for their hardware):

```yaml
# Tian'er v1 Bluetooth Sniffer Configuration
# Edit this file to match your hardware setup.
# Changes take effect on next service restart.

sniffers:
  - id: 1
    name: ut1
    type: ubertooth
    device: /dev/tianer/ubertooth0
    mode: btle           # btle (BLE advertising) | rx (Classic BR/EDR survey)
    channels: [37]       # Single channel at a time (hardware limitation)
    enabled: true

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
    enabled: false      # Disabled by default — enable when hardware available

  - id: 4
    name: nrf3
    type: nrf
    device: /dev/tianer/nrf2
    mode: ble
    enabled: false
```

### 9.3 tmpfiles.d Configuration

**`/etc/tmpfiles.d/tianer.conf`** — deployed once by C01, managed by systemd on every boot:

```
# Tian'er Signal Intelligence Platform — Runtime Paths & FIFOs
# Managed by systemd-tmpfiles. Do not edit manually.

d      /var/run/tianer                 0750 tianer    tianer    -   -
d      /var/log/tianer                 0750 tianer    tianer    -   -
p      /var/run/tianer/ut1.fifo        0660 tianer    tianer    -   -
p      /var/run/tianer/nrf1.fifo       0660 tianer    tianer    -   -
p      /var/run/tianer/nrf2.fifo       0660 tianer    tianer    -   -
p      /var/run/tianer/nrf3.fifo       0660 tianer    tianer    -   -
```

### 9.4 udev Rules

**`/etc/udev/rules.d/99-tianer.rules`** — deployed once:

```
# Tian'er Signal Intelligence Platform — USB Sniffer Device Rules
# Grants plugdev access to sniffer hardware; creates /dev/tianer/* symlinks.

# Ubertooth One (1d50:6002)
SUBSYSTEM=="usb", ATTRS{idVendor}=="1d50", ATTRS{idProduct}=="6002", \
    GROUP="plugdev", MODE="0660", \
    SYMLINK+="tianer/ubertooth%n"

# Nordic nRF52840 Dongle — Sniffer firmware (1915:522A) — Primary
SUBSYSTEM=="usb", ATTRS{idVendor}=="1915", ATTRS{idProduct}=="522a", \
    GROUP="plugdev", MODE="0660"
SUBSYSTEM=="tty", ATTRS{idVendor}=="1915", ATTRS{idProduct}=="522a", \
    SYMLINK+="tianer/nrf%n", \
    GROUP="dialout", MODE="0660"

# Nordic nRF52840 Dongle — DFU/Bootloader firmware (1915:520F) — Secondary
SUBSYSTEM=="usb", ATTRS{idVendor}=="1915", ATTRS{idProduct}=="520f", \
    GROUP="plugdev", MODE="0660"
SUBSYSTEM=="tty", ATTRS{idVendor}=="1915", ATTRS{idProduct}=="520f", \
    SYMLINK+="tianer/nrf%n", \
    GROUP="dialout", MODE="0660"

# Nordic nRF52840 DK (development kit) — Alternative ID (1366:*) 
SUBSYSTEM=="usb", ATTRS{idVendor}=="1366", \
    GROUP="plugdev", MODE="0660"
SUBSYSTEM=="tty", ATTRS{idVendor}=="1366", \
    SYMLINK+="tianer/nrf%n", \
    GROUP="dialout", MODE="0660"
```

### 9.5 Quadlet Host Prerequisites

C01 ensures the host is ready for Quadlet deployment:

| File/Directory | Purpose | Created By |
|---------------|---------|-----------|
| `/etc/containers/systemd/` | Quadlet `.container`, `.pod`, `.volume`, `.network` files | `install -d` in Phase 7 |
| `/var/lib/tianer/.config/systemd/user/` | User systemd directory (`systemctl --user`) | Automatic by systemd when linger is enabled |
| `/etc/subuid` + `/etc/subgid` | Subordinate UID/GID mapping | `uidmap` package (Phase 2) |
| `$XDG_RUNTIME_DIR` | `/run/user/$(id -u tianer)/` | Automatic by systemd-logind when linger starts |

---

## 10. Test Plan

### 10.1 Unit Tests (bash — bats-core)

| Test | File | Purpose |
|------|------|---------|
| `create-user.sh` creates user when absent | `deploy/scripts/tests/test_create_user.bats` | Verify `useradd` is invoked correctly |
| `create-user.sh` is idempotent | `deploy/scripts/tests/test_create_user.bats` | Re-running does not error |
| `create-dirs.sh` creates all paths with correct ownership | `deploy/scripts/tests/test_create_dirs.bats` | `stat` checks on every path |
| `generate-secrets.sh` is idempotent | `deploy/scripts/tests/test_generate_secrets.bats` | Does not overwrite existing secrets |
| `setup-wireshark.sh` applies capabilities | `deploy/scripts/tests/test_setup_wireshark.bats` | `getcap /usr/bin/dumpcap` check |

### 10.2 Integration Tests (bash)

| Test | File | Purpose |
|------|------|---------|
| Full prerequisite verification | `tests/integration/test_prereqs.sh` | Run `tianer-check-prereqs.sh` and verify exit 0 |
| OS verification rejects wrong distro | `tests/integration/test_os_verify.sh` | Mock `/etc/os-release` with wrong values |
| udev rules loaded correctly | `tests/integration/test_udev.sh` | `udevadm test` on a simulated device |
| Podman rootless smoke test | `tests/integration/test_podman_rootless.sh` | `sudo -u tianer podman run --rm hello-world` |
| systemd-linger active | `tests/integration/test_linger.sh` | `loginctl show-user tianer -p Linger` |
| tmpfiles creates FIFOs | `tests/integration/test_tmpfiles.sh` | Verify FIFOs exist after `systemd-tmpfiles --create` |
| Container with USB device access | `tests/integration/test_container_usb.sh` | `--device /dev/tianer/nrf0` passes through correctly |

### 10.3 Acceptance Criteria

| # | Criterion | Verification Command | Pass Condition |
|---|-----------|---------------------|----------------|
| AC-C01-1 | `tianer` system user exists with no login shell | `getent passwd tianer \| awk -F: '{print $7}'` | Returns `/usr/sbin/nologin` |
| AC-C01-2 | `tianer` is in `plugdev`, `dialout`, `wireshark` | `id -nG tianer` | Contains all three group names |
| AC-C01-3 | All mandatory paths exist with correct ownership | `stat -c '%U %G %a' /var/lib/tianer /var/run/tianer /var/log/tianer /etc/tianer` | Each line is `tianer tianer 750` (except `/etc/tianer` which is `tianer tianer 700`) |
| AC-C01-4 | udev rules installed | `test -f /etc/udev/rules.d/99-tianer.rules` | File exists |
| AC-C01-5 | `dumpcap` has file capabilities | `getcap /usr/bin/dumpcap` | Output contains `cap_net_raw` and `cap_net_admin` |
| AC-C01-6 | Podman installed | `which podman` | Returns a path |
| AC-C01-7 | Rootless Podman works as `tianer` | `sudo -u tianer podman run --rm hello-world` | Exits 0 |
| AC-C01-8 | `systemd-linger` enabled for `tianer` | `loginctl show-user tianer -p Linger` | Returns `Linger=yes` |
| AC-C01-9 | FIFOs created at boot | `ls -la /var/run/tianer/*.fifo` | Shows 4 FIFO files |
| AC-C01-10 | Secrets directory has correct permissions | `stat -c '%a' /etc/tianer/secrets` | Returns `700` |
| AC-C01-11 | No `tianer` sudoers entry | `test -f /etc/sudoers.d/tianer` | File does not exist (exits 1) |
| AC-C01-12 | Quadlet prerequisites directory exists | `test -d /etc/containers/systemd` | Directory exists |
| AC-C01-13 | Container image pull works | `sudo -u tianer podman pull docker.io/library/hello-world:latest` | Exits 0 |

### 10.4 End-to-End Test

`tests/integration/test_prereqs.sh` — full acceptance validation (runs on the target Pi after `setup.sh`):

```bash
#!/usr/bin/env bash
set -euo pipefail

FAILURES=0

check() {
    local desc="$1"; shift
    if "$@"; then
        echo "PASS: $desc"
    else
        echo "FAIL: $desc"
        FAILURES=$((FAILURES + 1))
    fi
}

# User
check "tianer user exists" id tianer >/dev/null
check "tianer has no login shell" bash -c '[[ "$(getent passwd tianer | cut -d: -f7)" == "/usr/sbin/nologin" ]]'

# Groups
for grp in plugdev dialout wireshark; do
    check "tianer in group $grp" id -nG tianer | grep -qw "$grp"
done

# Paths
check "/var/lib/tianer exists with correct owner" bash -c '[[ "$(stat -c "%U:%G:%a" /var/lib/tianer)" == "tianer:tianer:750" ]]'
check "/var/run/tianer exists" test -d /var/run/tianer
check "/var/log/tianer exists" test -d /var/log/tianer
check "/etc/tianer exists with correct perms" bash -c '[[ "$(stat -c "%U:%G:%a" /etc/tianer)" == "tianer:tianer:700" ]]'
check "/etc/tianer/secrets perms" bash -c '[[ "$(stat -c "%a" /etc/tianer/secrets)" == "700" ]]'

# udev
check "udev rules installed" test -f /etc/udev/rules.d/99-tianer.rules

# Capabilities
check "dumpcap has capabilities" getcap /usr/bin/dumpcap | grep -q 'cap_net_raw'

# Podman
check "podman installed" which podman >/dev/null
check "rootless podman works" sudo -u tianer podman run --rm hello-world >/dev/null 2>&1

# Linger
check "systemd-linger enabled" loginctl show-user tianer -p Linger | grep -q 'Linger=yes'

# FIFOs (may not exist if no services started yet; tmpfiles can be triggered manually)
check "FIFOs created by tmpfiles" bash -c '
  systemd-tmpfiles --create /etc/tmpfiles.d/tianer.conf
  for f in ut1 nrf1 nrf2 nrf3; do test -p "/var/run/tianer/${f}.fifo" || exit 1; done
'

# Secrets
check "secrets generated" test -f /etc/tianer/secrets/db_password
check "secrets permissions" bash -c '[[ "$(stat -c "%a" /etc/tianer/secrets/db_password)" == "600" ]]'

# No sudoers
check "no tianer sudoers" bash -c '! test -f /etc/sudoers.d/tianer'

# Quadlet
check "Quadlet directory exists" test -d /etc/containers/systemd

# Subordinate IDs
check "subuid has tianer" grep -q tianer /etc/subuid
check "subgid has tianer" grep -q tianer /etc/subgid

echo
if [[ $FAILURES -eq 0 ]]; then
    echo "PREREQUISITES OK ($FAILURES failures)"
    exit 0
else
    echo "PREREQUISITES FAILED ($FAILURES failures)"
    exit 1
fi
```

---

## 11. Deployment Notes

### 11.1 `setup.sh` Phases

`deploy/setup.sh` is the master orchestrator. It runs as `root` (via `sudo`) once at initial deployment and is **idempotent** — safe to re-run for remediation or upgrades.

| Phase | Step | Script/Command | Idempotent? |
|-------|------|---------------|-------------|
| 1 | Verify OS | `grep /etc/os-release` | Yes — read-only |
| 2a | Install system packages | `apt-get install -y $(cat apt-packages.txt)` | Yes — apt-get install on already-installed packages is a no-op |
| 2b | Add external apt repos | `apt-add-repository` + GPG key import | Yes — checks if repo already present |
| 2c | Install repo-specific packages | `apt-get install -y postgresql-17 ...` | Yes |
| 3 | Create tianer user + groups | `create-user.sh` | Yes — guarded by `id -u tianer` and `-aG` |
| 4a | Create filesystem paths | `create-dirs.sh` | Yes — `install -d` is idempotent |
| 4b | Apply tmpfiles.d | `systemd-tmpfiles --create` | Yes — `-p` type is idempotent |
| 4c | Generate secrets | `generate-secrets.sh` | Yes — checks file existence first |
| 5a | Setup Wireshark capabilities | `setup-wireshark.sh` | Yes — `dpkg-reconfigure` is idempotent |
| 5b | Install udev rules | `install -m 0644 99-tianer.rules` + `udevadm control --reload` + `udevadm trigger` | Yes — overwrites existing rules file |
| 6a | Install Podman | `apt-get install -y podman slirp4netns ...` | Yes |
| 6b | Enable lingering | `loginctl enable-linger tianer` | Yes — `enable-linger` is idempotent |
| 7 | Deploy Quadlet files | `install -m 0644` to `/etc/containers/systemd/` | Yes — overwrites existing |
| 8 | Reload systemd | `systemctl daemon-reload` | Yes |
| 9 | Apply DB migrations | `make db-up` | Yes — migration tracking table prevents re-application |
| 10 | Print completion summary | echo with next-steps instructions | N/A |

### 11.2 apt Package List

`deploy/apt-packages.txt` — full list (see §4.3.1)

### 11.3 Containerfile for Base Image

C01 provides a **base container image** used as the foundation for other service images (C05, C07). It is a minimal Debian Trixie slim image with runtime dependencies pre-installed:

**`deploy/containers/Containerfile.tianer-base`:**

```dockerfile
# Stage 1: Build stage (used only in multi-stage builds by C05/C07)
FROM debian:trixie-slim AS builder
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    g++-14 \
    cmake \
    libpqxx-dev \
    libpcap-dev \
    nlohmann-json3-dev \
    && rm -rf /var/lib/apt/lists/*

# Stage 2: Runtime stage (minimal — no compilers, no headers)
FROM debian:trixie-slim AS runtime
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpqxx-7.8 \
    libpcap0.8 \
    libstdc++6 \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*
```

**Size target:** Under 80 MB (runtime stage only).

**Policy:** No shells, no package managers, no development headers in the runtime stage. Each component's `Containerfile` inherits from this base using `FROM tianer-base:latest AS runtime` and copies its compiled binary.

### 11.4 Quadlet Unit Deployment

Quadlet files are deployed to `/etc/containers/systemd/` from `deploy/containers/`. The inventory (per storage-strategy.md):

| File | Type | Purpose |
|------|------|---------|
| `tianer-net.network` | Network | Internal bridge network for platform pod |
| `tianer-postgres-data.volume` | Volume | Persistent PostgreSQL data (V06) |
| `tianer-grafana-data.volume` | Volume | Persistent Grafana data (V07) |
| `tianer-capture.pod` | Pod | Container group for sniffer + tshark + ingest (Network=none) |
| `tianer-platform.pod` | Pod | Container group for API, gap-detect, deep-parser, ML (bridge) |
| `tianer-postgres.container` | Container | PostgreSQL 17 in standalone container |
| `tianer-sniffer@.container` | Container (template) | Per-sniffer wrapper |
| `tianer-tshark@.container` | Container (template) | Per-sniffer tshark |
| `tianer-ingest@.container` | Container (template) | Per-sniffer ingest bridge |
| `tianer-rotate.container` | Container | PCAP rotation |
| `tianer-gap-detect.container` | Container | Gap detector |
| `tianer-deep-parse.container` | Container | Deep parser (batch) |
| `tianer-ml-classify.container` | Container | ML enrichment |
| `tianer-api.container` | Container | FastAPI backend |
| `tianer-grafana.container` | Container | Grafana |

After deployment, containers are managed via `systemctl --user`:

```bash
# Enable auto-start at boot (as tianer user)
sudo machinectl shell tianer@ /bin/bash -c '
  systemctl --user enable tianer-postgres.service
  systemctl --user enable tianer-grafana.service
  systemctl --user enable tianer-capture-pod.service
  systemctl --user enable tianer-platform-pod.service
'

# Start all
sudo machinectl shell tianer@ /bin/bash -c 'systemctl --user start tianer.target'
```

---

## 12. Container Integration

### 12.1 Quadlet Unit Verification

Each Quadlet `.container` file must be validated before deployment:

```bash
# Validate Quadlet syntax (dry-run generation)
/usr/libexec/podman/quadlet --dryrun --user

# Check generated systemd units
ls ~/.config/systemd/user/tianer-*.service
```

### 12.2 Volume Mount Correctness

Verify that volumes declared in storage-strategy.md are correctly mounted in containers:

| Container | Expected Mounts | Access Mode |
|-----------|----------------|-------------|
| `tianer-postgres` | `tianer-postgres-data.volume:/var/lib/postgresql/data` | :rw |
| `tianer-sniffer@` | `/etc/tianer:/etc/tianer` (V01), `/var/lib/tianer/pcap:/var/lib/tianer/pcap` (V02), `/var/run/tianer:/var/run/tianer` (V03), `/var/log/tianer:/var/log/tianer` (V04) | V01:ro, V02:rw, V03:rw, V04:rw |
| `tianer-tshark@` | V01:ro, V03:rw, V04:rw | (no V02 access) |
| `tianer-ingest@` | V01:ro, V03:rw, V04:rw | (no V02 access) |
| `tianer-rotate` | V01:ro, V02:rw, V04:rw | |
| `tianer-gap-detect` | V01:ro, V02:ro, V04:rw | V02 read-only for backfill |
| `tianer-deep-parse` | V01:ro, V02:ro, V05:rw, V04:rw | V02 read-only |
| `tianer-ml-classify` | V01:ro, V05:ro, V04:rw | V05 read-only |
| `tianer-api` | V01:ro, V04:rw | No data volumes — reads DB only |
| `tianer-grafana` | `tianer-grafana-data.volume:/var/lib/grafana`, V01:ro, V04:rw | |

### 12.3 Pod Networking

```
┌─────────────────────────────────────────┐
│           tianer-capture pod             │
│           (Network=none)                 │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ │
│  │ sniffer@ │→│ tshark@  │→│ ingest@  │ │
│  │          │ │          │ │          │ │
│  └──────────┘ └──────────┘ └────┬─────┘ │
│                                  │       │
└──────────────────────────────────┼───────┘
                                   │ localhost:5432
                                   ▼
┌─────────────────────────────────────────┐
│          tianer-platform pod             │
│          (bridge: tianer-net)            │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ │
│  │gap-detect│ │deep-parse│ │  api     │ │
│  │          │ │          │ │          │ │
│  └──────────┘ └──────────┘ └──────────┘ │
└─────────────────────────────────────────┘
         │                              │
         │ localhost:5432               │ localhost:5432
         ▼                              ▼
┌─────────────────┐       ┌─────────────────┐
│ tianer-postgres │       │ tianer-grafana  │
│ (standalone)    │       │ (standalone)    │
└─────────────────┘       └─────────────────┘
```

- **tianer-capture pod:** Network=none — sniffer containers talk only to USB and FIFOs. tshark reads from FIFOs. Ingest writes to PostgreSQL via shared `localhost` network (the ingest container connects to PostgreSQL on the host's loopback).
- **tianer-platform pod:** Bridge network (`tianer-net`) — allows HTTP between API and Grafana if needed in future.
- **PostgreSQL + Grafana:** Standalone containers on `localhost` for database connections.

### 12.4 Container Boot Order

```
1. tianer-postgres.service        (standalone, must be ready first)
2. tianer-capture-pod.service     (depends on postgres)
     ├─ tianer-tshark@.service    (must start before sniffer)
     ├─ tianer-ingest@.service    (must start before sniffer)
     └─ tianer-sniffer@.service   (starts last in pod)
3. tianer-platform-pod.service    (depends on postgres)
     ├─ tianer-gap-detect.service
     ├─ tianer-deep-parse.service (batched, not always running)
     ├─ tianer-ml-classify.service
     └─ tianer-api.service
4. tianer-grafana.service         (depends on postgres)
```

---

## Appendix A: Storage Budget

| Storage Item | Location | Space Allocation | Notes |
|-------------|----------|-----------------|-------|
| OS + system packages | eMMC (`/`) | ~4 GB | Raspberry Pi OS Lite base |
| Container images | eMMC (`/var/lib/containers/`) | ~600 MB | 11 images, layer-deduplicated |
| PostgreSQL data (V06) | eMMC (Podman volume) | 5 GB | TimescaleDB hypertable + chunks |
| Grafana data (V07) | eMMC (Podman volume) | 1 GB | sqlite + dashboard cache |
| PCAP files (V02) | SD card (`/var/lib/tianer/pcap/`) | 25 GB | 14-day retention, compressed |
| JSONL output (V05) | SD card (`/var/lib/tianer/data/`) | 2 GB | Deep parser output, rotated |
| Logs (V04) | eMMC (`/var/log/tianer/`) | 5 GB | Structured logs, 30-day retention |
| **Total eMMC** | | ~16 GB | ~50% of 32 GB eMMC |
| **Total SD card** | | ~27 GB | Depends on SD card size; 32 GB minimum |
| **Free eMMC buffer** | | ~16 GB | Headroom for growth, temp files, package updates |

---

## Appendix B: Container Image Policy

Per storage-strategy.md and the container security recommendation:

| Requirement | Implementation |
|------------|---------------|
| Base image | `debian:trixie-slim` for C++/system services; `python:3.13-slim` for Python services |
| Multi-stage builds | Mandatory — build stage has compilers; runtime stage copies only artifacts |
| No shells in runtime | Remove `/bin/sh` and `/bin/bash` from runtime stage (use `scratch` or `gcr.io/distroless` where feasible) |
| No package managers | `apt`, `pip`, `npm` removed from runtime stage |
| No dev headers | `*-dev` packages only in build stage |
| Image size target | Under 200 MB uncompressed per image |
| Boot time target | Under 3 seconds to container ready |
| Digest pinning | `podman pull image@sha256:...` only; CI enforces no `:latest` tags |
| CVE scanning | `podman image inspect` + periodic `trivy` scan in CI |

---

## Appendix C: Cross-Reference to Other Components

| Component | Depends on C01 for... | C01 Section |
|-----------|----------------------|-------------|
| C02 (Database) | User `tianer`, PostgreSQL installed, secrets generated, config env file | §3, §4.4, §4.5.3, §9.1 |
| C03 (Capture Pipeline) | udev device symlinks, FIFOs, Wireshark capabilities, sniffer config | §4.5.2, §4.6, §9.2 |
| C05 (Ingest Bridge) | FIFO paths, DB credentials env file, libpqxx installed | §5.1, §9.1 |
| C06 (Gap Detector) | PCAP directory, heartbeat directory, DB credentials | §5.1 |
| C09 (REST API) | Secrets (API key, DB password), TLS certs directory | §5.1, §4.5.3 |
| C12 (Service Orchestration) | Quadlet directory, Podman runtime, user linger, all systemd units | §4.7, §4.8, §11.4 |
| C13 (Observability) | Host-level metrics paths, log directory | §7, §5.1 |
| C14 (Deployment Automation) | Entire `setup.sh` phase sequence | §11.1 |

---

## Appendix D: Glossary

| Term | Definition |
|------|-----------|
| **Rootless Podman** | Podman running as an unprivileged user (`tianer`), using user namespaces — no `root` involved |
| **Quadlet** | systemd generator that converts `.container` files into `systemd --user` service units |
| **systemd-linger** | Mechanism allowing a user's `systemd --user` instance to start at boot and persist across login sessions (`loginctl enable-linger`) |
| **tmpfiles.d** | systemd mechanism for creating/cleaning files, directories, and FIFOs at boot (`/etc/tmpfiles.d/`) |
| **tmpfs** | RAM-backed filesystem — used for `/var/run/tianer/` (V03) so FIFOs are in-memory and recreated fresh at each boot |
| **udev** | Linux device manager that creates device nodes, applies permissions, and creates symlinks when hardware is connected |
| **file capability** | Linux kernel feature granting specific privileges to a binary without `setuid root` (e.g., `cap_net_raw+eip` on `/usr/bin/dumpcap`) |
| **D-03, D-04, D-07** | Decision IDs from Phase A — see `docs/adr/0001-phase-a-decisions.md` |
| **PF-4, PF-5, PF-8, PF-10** | Engineering-for-Failure principle IDs — see component-breakdown.md §6.1 |
