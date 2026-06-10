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

The Raspberry Pi CM5 is rated for **ambient temperature 0°C to 85°C** (commercial grade) with the SoC throttling beginning at **85°C** junction temperature [7].

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
| System user | user | `tianer` | Runs all platform processes (UID allocated by `useradd --system` [8]) | `deploy/scripts/create-user.sh` |
| Primary group | group | `tianer` | Owns service files, directories, and running processes | `useradd --user-group` |
| USB access | group | `plugdev` | Grants `/dev/tianer/ubertooth*` and nRF raw USB access via udev [5] | `usermod -aG plugdev` |
| Serial access | group | `dialout` | Grants `/dev/ttyACM*` access for nRF CDC ACM serial | `usermod -aG dialout` |
| Capture capability | group | `wireshark` | Grants `dumpcap` invocation (file `cap_net_raw+cap_net_admin+eip` [4]) | `usermod -aG wireshark` |
| Container socket | group | `tianer` | Rootless Podman socket (`/run/user/$(id -u tianer)/podman/podman.sock`) | Automatic via `loginctl enable-linger` [2] |

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

Per the capabilities(7) man page [4], file capabilities allow assigning specific kernel capabilities to executables without granting full root privileges. The `eip` suffix denotes effective, inheritable, and permitted sets.

| Binary | Capability Set | Set By | Purpose |
|--------|---------------|--------|---------|
| `/usr/bin/dumpcap` | `cap_net_raw,cap_net_admin+eip` [4] | `dpkg-reconfigure wireshark-common` | Capture from FIFO (tshark delegates to dumpcap). `CAP_NET_RAW` permits RAW and PACKET sockets; `CAP_NET_ADMIN` permits promiscuous mode [4]. |
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
#    Per useradd(8) [8]: --system creates a system account with UID in
#    SYS_UID_MIN–SYS_UID_MAX range; --home-dir sets the login directory;
#    --shell /usr/sbin/nologin disables interactive login.
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

Per the tmpfiles.d(5) specification [3], the `p` type line creates a named pipe (FIFO) if it does not exist yet. Directories are created with `d` type lines. The format is: `Type Path Mode User Group Age Argument`.

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

# 3. Verify capabilities applied. Per capabilities(7) [4], CAP_NET_RAW
#    permits RAW and PACKET sockets; CAP_NET_ADMIN permits promiscuous mode.
getcap /usr/bin/dumpcap | grep -q 'cap_net_admin.*cap_net_raw\|cap_net_raw.*cap_net_admin'

# 4. tianer user is already in the wireshark group via create-user.sh.
```

#### 4.6.2 udev Rules (`deploy/udev/99-tianer.rules`)

Per the udev(7) specification [5], udev rules match device attributes and assign properties. The `SYMLINK+=` operator appends to the symlink list; `GROUP=` sets the device node group; `MODE=` sets permissions. Rules apply in lexicographic filename order.

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
# Per loginctl(1) [2], enable-linger spawns a user manager at boot and
# keeps it running after logouts.
loginctl enable-linger tianer
```

This creates `/var/lib/systemd/linger/tianer`, which tells systemd to start the `tianer` user's `systemd --user` instance at boot and keep it running indefinitely [2].

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
2. Container Quadlet files include `--group-add keep-groups` so that the container process inherits the `plugdev` and `dialout` group memberships needed to access the device. Per the podman-run(1) documentation [6], `keep-groups` passes the user's supplementary group access into the container when running rootless.
3. If a device is not present at container start, the container must gracefully degrade (log warning, pause, retry) — not crash-loop.
4. Adding a new sniffer type (future module) requires a new udev rule line with a new symlink under `/dev/tianer/`.

### 5.3 Container Runtime Contract (CONTAINER-RUNTIME)

All components (C02–C14) assume the container runtime environment defined by C01:

| Property | Value |
|----------|-------|
| Container engine | Podman (rootless, user `tianer`) [6] |
| Orchestration | Quadlet `.container` files → systemd `--user` units |
| User lingering | `loginctl enable-linger tianer` (enabled) [2] |
| Network mode | `tianer-capture` pod: `Network=none`; `tianer-platform` pod: bridge network `tianer-net` |
| Image pull policy | Digest-pinned (`@sha256:`); never `:latest` |
| Volume mounts | Per storage-strategy.md access matrix |
| USB passthrough | `--device /dev/tianer/<name>` + `--group-add keep-groups` [6] |
| Capabilities | `--cap-drop ALL` default; only C03 containers get `CAP_NET_RAW` + `CAP_NET_ADMIN` [4] |
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
| F-C01-5 | **Thermal throttling** | SoC temp > 85°C [7]; kernel log `throttled=0x...`; `vcgencmd get_throttled` | CPU clocks reduced; ingest latency increases; deep parser throughput drops | Passive heatsink; active cooling if sustained; reduce concurrent deep parser jobs |
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

The `tianer` user is a **system user** (UID in system range, typically < 1000) [8]:
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
- `NoNewPrivileges=true` in generated Quadlet systemd units. Per systemd.exec(5) [1], `NoNewPrivileges=yes` ensures the service process and all its children never gain new privileges, preventing privilege escalation via setuid/setgid or file capabilities.
- `--cap-drop ALL` as default in Quadlet `.container` files
- Only C03 capture containers receive `CAP_NET_RAW` + `CAP_NET_ADMIN` [1]
- `ReadWritePaths=` restricts writable paths per container to `/var/log/tianer/`, `/var/lib/tianer/pcap/`, `/var/lib/tianer/data/`
- `ProtectSystem=strict` in systemd units. Per systemd.exec(5) [1], `ProtectSystem=strict` mounts `/usr/`, `/etc/` (but not `/etc/tianer/` which is whitelisted via `ReadWritePaths=`), and `/boot` read-only, preventing container processes from modifying system files.
- All systemd hardening directives follow the systemd.exec(5) specification for sandboxing [1].

### 8.7 Container Privilege Inheritance

Containers running under rootless Podman with the `tianer` user map their internal `root` (UID 0) to the host UID of `tianer`. This means:
- Even if a container process runs as UID 0 inside the container, it has the privileges of the `tianer` user on the host.
- File capabilities set on host binaries (e.g., `dumpcap`) are inherited by containers when the binary is bind-mounted.
- The `--group-add keep-groups` flag passes supplementary group memberships into the container [6].
- No container ever runs as host `root`.

---

## 9. Configuration

### 9.1 Environment Variables

| Variable | File | Purpose |
|----------|------|---------|
| `TIANER_DB_HOST` | `/etc/tianer/tianer.env` | PostgreSQL host (default: `tianer-postgres` on `tianer-net`) |
| `TIANER_DB_PORT` | `/etc/tianer/tianer.env` | PostgreSQL port (default: `5432`) |
| `TIANER_DB_NAME` | `/etc/tianer/tianer.env` | Database name (default: `tianer`) |
| `TIANER_DB_USER` | per-service `.env` | Role name (`tianer_writer`, `tianer_ro`, etc.) |
| `TIANER_DB_PASSWORD` | `/etc/tianer/secrets/db_password` | Database password (base64, `openssl rand`) |
| `TIANER_API_KEY` | `/etc/tianer/secrets/api_key` | API authentication key |
| `BLESNIFF_PCAP_DIR` | `/etc/tianer/blesniff.env` | PCAP output directory |
| `BLESNIFF_ROTATION_MINUTES` | `/etc/tianer/blesniff.env` | PCAP file rotation interval |

### 9.2 Configuration Files

| File | Format | Purpose |
|------|--------|---------|
| `/etc/tianer/sniffers.yaml` | YAML | Sniffer definitions and enable/disable |
| `/etc/tianer/alerts.yaml` | YAML | Alert thresholds (tunable without rebuilding) |
| `/etc/tianer/tianer.env` | KEY=value | Shared platform environment variables |
| `/etc/tianer/blesniff.env` | KEY=value | Bluetooth module specific variables |
| `/etc/tmpfiles.d/tianer.conf` | tmpfiles.d [3] | Runtime directories and FIFO provisioning |
| `/etc/udev/rules.d/99-tianer.rules` | udev rules [5] | Device symlinks and permissions |

---

## 10. Test Plan

### 10.1 Unit Tests

All unit tests use bats-core [9] for Bash test automation. Test files in `deploy/scripts/tests/`.

| Test | Description | Acceptance Criteria |
|------|-------------|---------------------|
| `test-create-user.sh` | Run `create-user.sh` on a clean system | `id tianer` succeeds; user has correct groups |
| `test-create-user-idempotent.sh` | Re-run `create-user.sh` | Success; no errors; group memberships unchanged |
| `test-create-dirs.sh` | Run `create-dirs.sh` | All directories exist with correct permissions |
| `test-tmpfiles.sh` | Run `systemd-tmpfiles --create` | FIFOs exist at expected paths |
| `test-udev-rules-syntax.sh` | Validate `99-tianer.rules` syntax | `udevadm verify` passes |
| `test-secrets-generate.sh` | Run `generate-secrets.sh` | Secret files exist with mode 0600 |
| `test-secrets-idempotent.sh` | Re-run `generate-secrets.sh` | Files unchanged; no regeneration |
| `test-dumpcap-capabilities.sh` | Run `setup-wireshark.sh` | `getcap /usr/bin/dumpcap` shows `cap_net_raw,cap_net_admin+eip` |
| `test-podman-rootless.sh` | Install Podman; verify rootless | `podman run --rm docker.io/library/alpine:latest echo hello` succeeds as `tianer` |
| `test-linger.sh` | Run `loginctl enable-linger tianer` | `loginctl show-user tianer -p Linger` returns `yes` |
| `test-quadlet-syntax.sh` | Validate all `.container` files | `podman systemd generate` succeeds |

### 10.2 Integration Tests

Integration tests use bash [10] scripting with `set -euo pipefail` strict mode.

| Test | Description | Acceptance Criteria |
|------|-------------|---------------------|
| `test-full-bootstrap.sh` | Run `setup.sh` on a clean Raspberry Pi OS system | All phases complete; `tianer-check-prereqs.sh` returns 0 |
| `test-bootstrap-idempotent.sh` | Re-run `setup.sh` | No errors; all checks pass; no duplicate operations |
| `test-usb-device-access.sh` | Plug in an nRF52840 dongle | `/dev/tianer/nrf0` exists; `stat` shows correct permissions |
| `test-container-start.sh` | Start all Quadlet containers with `systemctl --user` | All containers healthy; `pg_isready` returns accepting |
| `test-fifo-persistence.sh` | Reboot; verify FIFOs after restart | All 4 FIFOs exist after boot |

### 10.3 Acceptance Criteria

1. **System user:** `tianer` user exists with UID < 1000, shell `/usr/sbin/nologin`, groups `plugdev,dialout,wireshark`.
2. **File capabilities:** `dumpcap` has `cap_net_raw,cap_net_admin+eip`.
3. **udev rules:** Devices appear at `/dev/tianer/<name>` with correct permissions.
4. **tmpfiles:** FIFOs created at `/var/run/tianer/*.fifo` with mode 0660.
5. **Podman rootless:** Containers run as `tianer`, not as root.
6. **Linger:** `tianer` user's systemd --user instance starts at boot.
7. **Secrets:** Password files exist with mode 0600.
8. **Idempotent:** All bootstrap scripts safe to re-run.
9. **Health check:** `tianer-check-prereqs.sh` returns 0 after bootstrap.
10. **No sudo:** `tianer` user has no sudoers entry.

---

## 11. Deployment Notes

### 11.1 Bootstrap Order

`deploy/setup.sh` orchestrates all C01 phases. Phases are executed in order; each phase is idempotent.

### 11.2 Manual Steps

Outside the scope of `setup.sh`, the following steps require human intervention:
- SD card formatting and mounting
- Raspberry Pi OS installation on eMMC
- Physical USB hub and dongle connection
- SSH key configuration for operator access

### 11.3 Rollback

C01 does not support automated rollback since it modifies the host OS. Manual rollback steps:
1. Remove `tianer` user: `userdel -r tianer`
2. Remove groups (if no longer needed): `groupdel wireshark`
3. Remove udev rule: `rm /etc/udev/rules.d/99-tianer.rules && udevadm control --reload`
4. Remove directories: `rm -rf /etc/tianer /var/lib/tianer /var/run/tianer /var/log/tianer /opt/tianer /usr/share/tianer`
5. Remove tmpfiles config: `rm /etc/tmpfiles.d/tianer.conf`
6. Uninstall packages (optional): review `apt-packages.txt` and remove unnecessary packages

---

## References

[1] Freedesktop.org. "systemd.exec(5) — Execution Environment Configuration." https://man7.org/linux/man-pages/man5/systemd.exec.5.html, systemd 261~rc1 — Documents `User=`, `Group=`, `NoNewPrivileges=`, `ReadWritePaths=`, `ProtectSystem=strict`, and all sandboxing directives referenced in §8.6.

[2] Freedesktop.org. "loginctl(1) — Control the systemd Login Manager." https://man7.org/linux/man-pages/man1/loginctl.1.html, systemd 261~rc1 — Documents `enable-linger` semantics: spawns a user manager at boot and persists after logout (§4.7.2, §5.3).

[3] Freedesktop.org. "tmpfiles.d(5) — Configuration for Creation, Deletion, and Cleaning of Files and Directories." https://man7.org/linux/man-pages/man5/tmpfiles.d.5.html, systemd 261~rc1 — Documents the `p` type for FIFO creation and `d` type for directory creation (§4.5.2).

[4] Linux man-pages. "capabilities(7) — Overview of Linux Capabilities." https://man7.org/linux/man-pages/man7/capabilities.7.html — Documents `CAP_NET_RAW` (permits RAW and PACKET sockets), `CAP_NET_ADMIN` (permits promiscuous mode), and file capability semantics (`eip` = effective, inheritable, permitted) referenced in §3.4, §4.6.1, §5.3.

[5] Freedesktop.org. "udev(7) — Dynamic Device Management." https://man7.org/linux/man-pages/man7/udev.7.html, systemd 261~rc1 — Documents `SYMLINK+=` (append symlink), `GROUP=` (set device group), `MODE=` (set permissions) in udev rules referenced in §4.6.2.

[6] Podman Project. "podman-run(1) — Run a Command in a New Container." https://docs.podman.io/en/latest/markdown/podman-run.1.html — Documents rootless Podman, `--device` for host device passthrough, `--group-add keep-groups` for supplementary group access in rootless containers (§5.2, §5.3).

[7] Raspberry Pi Ltd. "Raspberry Pi Computer Hardware — Frequency Management and Thermal Control." https://www.raspberrypi.com/documentation/computers/raspberry-pi.html#frequency-management-and-thermal-control — Documents CM5 SoC thermal throttling threshold at 85°C junction temperature (§2.4).

[8] Shadow-utils project. "useradd(8) — Create a New User or Update Default New User Information." https://man7.org/linux/man-pages/man8/useradd.8.html, shadow-utils 4.19.0 — Documents `--system` (system account), `--home-dir` (login directory), `--shell` (login shell), and `--user-group` (§4.4, §8.4).

[9] bats-core. "Bash Automated Testing System." https://github.com/bats-core/bats-core, v1.11+. — Documents bats-core test framework for Bash script testing: `.bats` test file format, `@test` declarations, `run` helper, status/output assertions, and `setup()`/`teardown()` hooks (§10.1).

[10] Free Software Foundation. "GNU Bash Reference Manual." https://www.gnu.org/software/bash/manual/, 2024. — Documents bash scripting: `set -euo pipefail` strict mode, variable expansion, process substitution, and exit status conventions used in C01 scripts (§10.1, §10.2).

---

*End of C01 Platform Infrastructure Design Document.*
