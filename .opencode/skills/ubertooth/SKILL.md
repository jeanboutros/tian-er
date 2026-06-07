---
name: ubertooth
description: "Ubertooth One as a BLE testing tool for cross-validation of the ESP32 nRF24L01+ sniffer. Triggered when discussing Ubertooth, BLE packet injection, BLE advertising transmission, cross-validation testing, or the ubertooth_btle_tx script. Covers firmware update, USB command protocol, BLE advertising TX, Ubertooth passive sniffing, dewhitening validation, and the CC2400 modulation limitation."
---

# Ubertooth One — BLE Testing and Cross-Validation

## Purpose

The Ubertooth One is an open-source 2.4 GHz USB software-defined radio (Great Scott
Gadgets) used in this project as an **independent ground-truth source** for validating
the ESP32 nRF24L01+ BLE sniffer firmware.  Its roles are:

1. **Passive BLE sniffer** — captures real BLE advertising packets, decoded and
   dewhitened independently of the nRF24. Compared with ESP32 output for byte-level
   PDU verification.
2. **BLE advertising transmitter** — injects known ADV_IND packets via
   `tools/ubertooth_btle_tx.py` so the ESP32 can validate dewhitening, address
   matching, and PDU parsing against a known payload.

**Important limitation:** The Ubertooth's CC2400 transceiver does **not** produce a
BLE-spec-compliant GFSK signal. The ESP32 nRF24L01+ sniffer receives real BLE
packets from phones and peripherals, but **cannot receive Ubertooth-transmitted packets**
reliably. See §5 for the root cause analysis and implications.

---

## 1. Firmware and Toolchain Update (Required)

The Debian-distributed Ubertooth packages are outdated (API 1.06) and incompatible
with the device firmware (API 1.07). All components must be built from source.

### 1.1 Version mismatch

| Component | Distro version | Source version | Issue |
|-----------|---------------|----------------|-------|
| libubertooth | 2018.12.R1-5.3 (API 1.06) | git-c9dfdbd (API 1.07) | Commands may fail silently |
| ubertooth-tools | 2018.12.R1-5.3 | git-c9dfdbd (all 12) | Missing ubertooth-dfu |
| libbtbb | not installed | git-f0fe176 | Required for packet decoding |
| pyubertooth | not installed | 0.2 | Required for Python scripts |

### 1.2 Build from source

All components install to `$HOME/.local/` to avoid conflicts with system packages.

```bash
# Prerequisites
sudo apt install cmake libusb-1.0-0-dev make gcc g++ pkg-config \
    python3 python3-pip python3-usb

# Clone
git clone https://github.com/greatscottgadgets/ubertooth.git /tmp/ubertooth

# Build libbtbb
cd /tmp/ubertooth/host/libbtbb && mkdir build && cd build
cmake .. -DCMAKE_INSTALL_PREFIX=$HOME/.local && make -j$(nproc) && make install

# Build libubertooth (add cmake_policy(SET CMP0005 NEW) to CMakeLists.txt first)
cd /tmp/ubertooth/host/libubertooth && mkdir build && cd build
cmake .. -DCMAKE_INSTALL_PREFIX=$HOME/.local -DBUILD_SHARED_LIB=1 && make -j$(nproc) && make install

# Build ubertooth-tools (add -lusb-1.0 if ubertooth-dfu link fails)
cd /tmp/ubertooth/host/ubertooth-tools && mkdir build && cd build
cmake .. -DCMAKE_INSTALL_PREFIX=$HOME/.local && make -j$(nproc) && make install

# Install Python bindings
pip3 install --break-system-packages pyubertooth
# OR: cd /tmp/ubertooth/host/python && pip3 install --break-system-packages .
```

### 1.3 Flash firmware (ARM toolchain required)

```bash
# Install xPack ARM toolchain
curl -SL https://github.com/xpack-dev-tools/arm-none-eabi-gcc-xpack/releases/v15.2.1-1/xpack-arm-none-eabi-gcc-15.2.1-1-linux-x64.tar.gz | tar xz -C /tmp/

# Build firmware
cd /tmp/ubertooth/firmware/bluetooth_rxtx
make CROSS=/tmp/xpack-arm/xpack-arm-none-eabi-gcc-15.2.1-1.1/bin/arm-none-eabi-

# Enter DFU mode
ubertooth-util -d

# Flash (may report I/O error at end — check version afterward)
$HOME/.local/bin/ubertooth-dfu -f /tmp/ubertooth/firmware/bluetooth_rxtx/bluetooth_rxtx.hex -r

# Verify
ubertooth-util -v
# Expected: Firmware version: git-c9dfdbd* (API:1.07)
```

### 1.4 Environment configuration

Add to `~/.bashrc`:

```bash
export PATH="$HOME/.local/bin:$PATH"
export LD_LIBRARY_PATH="$HOME/.local/lib${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}"
```

Verify:

```bash
which ubertooth-util          # → $HOME/.local/bin/ubertooth-util
ubertooth-util -v             # → API:1.07
ldd $(which ubertooth-btle)   # → links libubertooth from $HOME/.local/lib
```

---

## 2. USB Command Protocol

The Ubertooth communicates via USB vendor-specific control transfers (endpoint 0).

### 2.1 Key commands

| Command | Code | Direction | wValue | Data | Purpose |
|---------|------|-----------|--------|------|---------|
| `UBERTOOTH_STOP` | 21 | CTRL_OUT | 0 | none | Stop current mode |
| `UBERTOOTH_SET_CHANNEL` | 12 | CTRL_OUT | freq_MHz | none | Set RF channel |
| `UBERTOOTH_BTLE_SLAVE` | 54 | CTRL_OUT | 0 | 6-byte MAC | Start ADV_IND TX |
| `UBERTOOTH_LE_SET_ADV_DATA` | 71 | CTRL_OUT | 0 | adv_data bytes | Set advertising payload |
| `UBERTOOTH_GET_API_VERSION` | 9 | CTRL_IN | 0 | (2 bytes) | Read API version |

### 2.2 Firmware automatic processing in BTLE_SLAVE mode

When in `MODE_BT_SLAVE_LE`, the firmware automatically:

| Task | Who does it | Can host override? |
|------|------------|-------------------|
| CRC-24 computation | Firmware (init=0x555555) | No |
| Data whitening (x^7+x^4+1 LFSR) | Firmware (channel-dependent seed) | No |
| Access address insertion | Firmware (0x8E89BED6) | No |
| PDU header construction | Firmware (always ADV_IND type=0x00) | No |
| ~100ms packet interval | Firmware timer | No |

The host controls only:
1. MAC address (6 bytes, via `UBERTOOTH_BTLE_SLAVE`)
2. Advertising data payload (N bytes, via `UBERTOOTH_LE_SET_ADV_DATA`)
3. RF channel/frequency (via `UBERTOOTH_SET_CHANNEL`)

### 2.3 Python — raw USB commands

```python
import usb.core, usb.util

CTRL_OUT = usb.util.build_request_type(
    usb.util.CTRL_OUT, usb.util.CTRL_TYPE_VENDOR, usb.util.CTRL_RECIPIENT_DEVICE)

dev = usb.core.find(idVendor=0x1d50, idProduct=0x6002)
assert dev is not None, "Ubertooth not found"
if dev.is_kernel_driver_active(0):
    dev.detach_kernel_driver(0)
dev.set_configuration()
usb.util.claim_interface(dev, 0)

# Stop any ongoing operation
dev.ctrl_transfer(CTRL_OUT, 21, 0, 0, b'', 1000)

# Set channel to 2402 MHz (BLE advertising channel 37)
dev.ctrl_transfer(CTRL_OUT, 12, 2402, 0, b'', 1000)

# Set advertising data (Flags + Complete Local Name "TEST")
adv_data = bytes.fromhex('0201060B0954455354')
dev.ctrl_transfer(CTRL_OUT, 71, 0, 0, adv_data, 1000)

# Start BLE advertising TX with MAC address
mac = bytes.fromhex('CAFEBABE0001')
dev.ctrl_transfer(CTRL_OUT, 54, 0, 0, mac, 1000)
```

### 2.4 BLE channel mapping

| BLE Channel | Frequency | nRF24 `RF_CH` | Ubertooth `wValue` | LFSR Seed |
|-------------|-----------|---------------|---------------------|-----------|
| 37 | 2402 MHz | 2 | 2402 | `swapbits(37) \| 2 = 0xA6` |
| 38 | 2426 MHz | 26 | 2426 | `swapbits(38) \| 2 = 0x66` |
| 39 | 2480 MHz | 80 | 2480 | `swapbits(39) \| 2 = 0xE6` |

Use `nrf24::ble::channel_to_rf_ch()` or `nrf24::ble::ADV_CHANNELS[]` for the
nRF24 side. For the Ubertooth, pass the frequency in MHz to `UBERTOOTH_SET_CHANNEL`.

---

## 3. TX Script: `tools/ubertooth_btle_tx.py`

### 3.1 Overview

The custom `tools/ubertooth_btle_tx.py` script sends USB commands directly to the
firmware's BTLE_SLAVE mode, bypassing the useless `ubertooth-tx` stub. It handles
device discovery, channel cycling, signal handling, and verbose output.

### 3.2 Common usage

```bash
# Install dependency
pip3 install pyusb

# Transmit ADV_IND on channel 37 with MAC and advertising data
python3 tools/ubertooth_btle_tx.py \
    -m CA:FE:BA:BE:00:01 \
    -d 0201060B0954455354 \
    -c 37 -v

# Cycle through all three advertising channels (5s per channel)
python3 tools/ubertooth_btle_tx.py \
    -m DE:AD:BE:EF:00:01 \
    -d 020105 \
    -C 37,38,39 -i 5 -v

# Transmit for 30 seconds then stop
python3 tools/ubertooth_btle_tx.py \
    -m CA:FE:BA:BE:00:01 \
    -d 020106050948454C4C4F \
    -c 37 -t 30 -v
```

### 3.3 Command-line options

| Option | Description | Default |
|--------|-------------|---------|
| `-m MAC` | MAC address (e.g. `AA:BB:CC:DD:EE:FF`) | Required |
| `-d DATA` | Advertising data as hex string | Required |
| `-c CH` | Single advertising channel (37, 38, or 39) | 37 |
| `-C CHS` | Cycle through comma-separated channels | — |
| `-i SEC` | Seconds between channel switches when cycling | 5 |
| `-n COUNT` | Number of packets (0=infinite) | 0 |
| `-t SEC` | Total transmit time in seconds (0=infinite) | 0 |
| `-v` | Verbose output | Off |

### 3.4 What the firmware transmits

For `-m CA:FE:BA:BE:00:01 -d 0201060B0954455354 -c 37`:

| Field | Value |
|-------|-------|
| Access Address | 0x8E89BED6 (auto) |
| PDU type | ADV_IND (0x00, auto) |
| PDU length | 6 + len(AdvData) (auto) |
| AdvA | CA:FE:BA:BE:00:01 (from `-m`) |
| AdvData | 02 01 06 0B 09 54 45 53 54 (from `-d`) |
| CRC-24 | Computed by firmware (init=0x555555) |
| Whitening | Applied by firmware (seed from channel) |
| Interval | ~100ms (firmware-controlled) |

---

## 4. Cross-Validation Workflow

### 4.1 End-to-end test script

`tools/e2e_crossval_test.sh` orchestrates the full cross-validation:

1. Flash ESP32 firmware
2. Start ESP32 serial capture
3. Start Ubertooth TX with known MAC and AdvData
4. Optionally start nRF52840 Sniffer capture (`--nrf`)
5. Wait for test duration
6. Analyze ESP32 log and nRF PCAP for the known MAC
7. Report PASS/FAIL

```bash
# Basic test (ESP32 + Ubertooth only)
./tools/e2e_crossval_test.sh

# With nRF Sniffer cross-validation
./tools/e2e_crossval_test.sh --nrf -t 60

# Skip flash, single channel, verbose
./tools/e2e_crossval_test.sh --no-flash -c 37 -t 10 -v
```

**Exit codes:** 0=PASS, 1=nRF sees packets but ESP32 doesn't, 2=neither sees packets, 3=error.

### 4.2 Ubertooth passive sniffing as ground truth

Instead of transmitting, use the Ubertooth as a **passive sniffer** to capture
real BLE traffic alongside the ESP32:

```bash
# Capture on channel 37
ubertooth-btle -c 37 -q /tmp/ch37.pcap &

# Decode with tshark
tshark -r /tmp/ch37.pcap -T fields \
    -e btle.advertising_address \
    -e btle.advertising_data
```

The Ubertooth outputs dewhitened PDU bytes. Compare directly with
`nrf24::ble::dewhiten()` output from the ESP32 — both should be byte-identical
for the same over-the-air packet.

### 4.3 Dewhitening validation techniques

| Technique | Description | Requires Ubertooth TX? |
|-----------|-------------|----------------------|
| 1. Crafted packet injection | Ubertooth TX → ESP32 RX, compare known payload | Yes |
| 2. Passive capture comparison | Ubertooth + ESP32 both sniff real BLE | No (passive) |
| 3. Round-trip self-test | Whiten→dewhiten in unit test | No hardware |
| 4. CRC-24 check | After dewhiten, verify CRC matches | Yes or passive |
| 5. Multi-channel consistency | Same packet on ch37/38/39 — proves all seeds | Yes |
| 6. Edge-case packets | Empty, max-length, all-zero, all-0xFF AdvData | Yes |

**Key point:** Technique 2 (passive sniffing) is the most reliable because real
BLE devices produce spec-compliant GFSK that both receivers can decode.

---

## 5. CC2400 TX Limitation (CRITICAL)

### 5.1 The problem

The Ubertooth's CC2400 transceiver approximates BLE GFSK modulation but
**does not produce a fully BLE-spec-compliant signal**. The nRF24L01+'s
demodulator, optimized for nRF24-to-nRF24 communication, is sensitive to
deviations from the BLE spec's BT=0.5 GFSK profile.

**Observed result:** ~1,359 Ubertooth-transmitted packets, ZERO received
by the ESP32 nRF24L01+. Meanwhile, the ESP32 successfully receives real BLE
packets from 6+ devices on all three advertising channels.

### 5.2 Root cause

| Hypothesis | Confidence | Fixability |
|-----------|------------|------------|
| CC2400 GFSK modulation quality (BT product, deviation accuracy) | 80% | Not fixable without different TX hardware |
| TX power/antenna coupling | 75% | Testable by rotating, but RPD testing suggests signal is received |
| CC2400 FIFO/timing issues | 55% | Not addressable from host |

### 5.3 What works and what doesn't

| Use | Works? | Notes |
|-----|--------|-------|
| Ubertooth **passive sniffing** | ✅ Yes | Independent ground-truth capture |
| Ubertooth **TX → Ubertooth RX** | ✅ Yes | Two Ubertooths can talk to each other |
| Ubertooth TX → **real BLE device** | ✅ Yes | Phones/tablets receive Ubertooth ADV_IND |
| Ubertooth TX → **ESP32 nRF24L01+** | ❌ No | CC2400 modulation not nRF24-compatible |
| Real BLE → **ESP32 nRF24L01+** | ✅ Yes | This is the validated use case |

### 5.4 Implications for testing

- **Do NOT rely on Ubertooth TX for ESP32 nRF24 validation.** It does not work.
- **Use Ubertooth for passive sniffing** (ground-truth comparison with real BLE traffic).
- **For active TX testing**, use a Nordic nRF52840 Dongle or a second nRF24L01+
  module instead.
- **The dewhitening and PDU parsing code IS correct** — validated by receiving
  real BLE packets from production devices.

### 5.5 Diagnosing "no packets received"

If the ESP32 sniffer reports zero packets during Ubertooth TX:

1. Check RPD register — if RPD=0 always, signal strength issue; if RPD=1 but no
   packets, modulation quality issue.
2. Print raw FIFO bytes **before** dewhitening — any bytes at all prove RF
   reception is working.
3. Verify channel mapping: `RF_CH` must be 2/26/80, not 37/38/39.
4. Verify `RX_ADDR_P0` matches `nrf24::ble::ADV_ACCESS_ADDR` in LSByte-first
   SPI order (`{0x71, 0x91, 0x7D, 0x6B}`).
5. Verify EN_AA=0x00 and EN_CRC=0 in CONFIG.

---

## 6. Common Issues

### 6.1 "Ubertooth not found"

```bash
lsusb | grep 1d50                         # verify device connected
groups $USER                              # must include 'plugdev'
# Create udev rule if missing:
sudo tee /etc/udev/rules.d/40-ubertooth.rules << 'EOF'
SUBSYSTEM=="usb", ATTR{idVendor}=="1d50", ATTR{idProduct}=="6002", MODE="0666"
EOF
sudo udevadm control --reload-rules && sudo udevadm trigger
```

### 6.2 "Cannot claim USB interface"

Another process is using the device:

```bash
ubertooth-util -S                         # stop any running mode
killall ubertooth-btle ubertooth-specan    # kill stale processes
```

### 6.3 "API version mismatch"

The host library reports a different API version than the firmware:
- Ensure you built all components from the same Git source (tag `2020-12-R1` or newer).
- Ensure `PATH` includes `$HOME/.local/bin` **before** `/usr/bin`.
- Verify with `ubertooth-util -v`.

### 6.4 "ubertooth-tx does nothing"

`ubertooth-tx` is a **stub** that prints "WARNING: This tool currently does nothing!"
and exits. It is for classic Bluetooth, not BLE. Use `tools/ubertooth_btle_tx.py`
or the raw USB commands in §2.3.

### 6.5 "ubertooth-dfu link error"

If `ubertooth-dfu` fails to link with `undefined reference to 'libusb_init'`, add
`-lusb-1.0` to the link step in `ubertooth-tools/CMakeLists.txt`.

### 6.6 "DFU I/O error at end of flash"

The DFU flash may report `"libUSB Error: Input/Output Error"` at the very end of
writing. This is a known issue with the verification step — the flash itself
succeeded. Verify with `ubertooth-util -v`. Do not re-flash unless the version
is wrong.

### 6.7 "PATH takes wrong version"

Old distro packages in `/usr/bin` take priority if `$HOME/.local/bin` is not
first in `PATH`:

```bash
# CORRECT — local takes priority
export PATH="$HOME/.local/bin:$PATH"

# WRONG — system paths checked first
export PATH="$PATH:$HOME/.local/bin"

# Verify
which ubertooth-util   # should show $HOME/.local/bin/ubertooth-util
```

---

## 7. Self-Reflection Clause

After any Ubertooth-related debugging or testing session, agents MUST answer:

1. **Why did this issue occur?** Was it an API mismatch, a firmware limitation,
   a configuration error, or a hardware incompatibility?
2. **What procedural safeguard would have caught it earlier?** Could version
   verification, a connectivity test, or a RPD check have prevented wasted time?
3. **Does this lesson need to be captured?** If yes, update this skill or the
   relevant learning doc in `docs/learning/` with the finding.

---

## 8. References

- Ubertooth GitHub: https://github.com/greatscottgadgets/ubertooth (verified 2026-06-06)
- Ubertooth firmware `bluetooth_rxtx.c`: https://github.com/greatscottgadgets/ubertooth/blob/2020-12-R1/firmware/bluetooth_rxtx/bluetooth_rxtx.c
- Ubertooth API `ubertooth_interface.h`: https://github.com/greatscottgadgets/ubertooth/blob/2020-12-R1/host/libubertooth/src/ubertooth_interface.h
- Great Scott Gadgets Ubertooth One product page: https://greatscottgadgets.com/ubertoothone/ (retired)
- Ubertooth docs on ReadTheDocs: https://ubertooth.readthedocs.io/en/latest/
- Bluetooth Core Spec Vol 6 Part B §3.2 (data whitening): https://www.bluetooth.com/specifications/specs/core-specification/
- Dmitry Grinberg "Bit-banging BLE": http://dmitry.gr/?r=05.Projects&proj=11.%20Bluetooth%20LE%20fakery
- Project learning docs:
  - `docs/learning/ubertooth-ble-testing.md` — full USB protocol, packet injection, dewhitening validation
  - `docs/learning/ubertooth-update-and-diagnostics.md` — firmware update, TX failure diagnosis, nRF Sniffer setup
  - `docs/learning/ble-data-whitening-nrf24.md` — dewhitening bugs and correct implementation
- Project tools:
  - `tools/ubertooth_btle_tx.py` — custom BLE advertising transmitter
  - `tools/e2e_crossval_test.sh` — end-to-end cross-validation test
- Project library:
  - `nrf24::ble::configure_rx()`, `nrf24::ble::switch_channel()`, `nrf24::ble::dewhiten()` from `ble_config.h` / `ble.h`