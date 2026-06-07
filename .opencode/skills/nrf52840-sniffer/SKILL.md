---
name: nrf52840-sniffer
description: "Nordic nRF52840 Dongle as an independent BLE packet sniffer for cross-validation of the ESP32 nRF24L01+ sniffer. Triggered when discussing nRF Sniffer, nRF52840, BLE packet capture with Wireshark, BLE cross-validation, or BLE advertising sniffing. Covers firmware setup, Wireshark extcap, capture workflow, and comparison with ESP32 output."
---

# nRF52840 Dongle — BLE Sniffer for Cross-Validation

## Purpose

The Nordic nRF52840 Dongle (USB VID:PID `1915:522A`) is used as an **independent
BLE packet sniffer** in this project.  Because it has a **native BLE radio**
(nRF52840 SoC), its on-air signal is 100% BLE-spec-compliant — unlike the
Ubertooth One's CC2400 transceiver, which only approximates BLE modulation.

The nRF Sniffer's role is to provide ground-truth BLE packet captures that can
be compared byte-for-byte with the ESP32 nRF24L01+ sniffer's dewhitened output.
If both devices decode the same over-the-air packet to the same PDU, the ESP32's
dewhitening and PDU parsing are confirmed correct.

---

## 1. Device Information

| Field | Value |
|-------|-------|
| USB Vendor ID | `1915` (Nordic Semiconductor) |
| USB Product ID | `522A` |
| Radio chip | nRF52840 (native BLE 5.0) |
| Interface | USB + Wireshark extcap plugin |
| Supported channels | All 40 BLE channels (37 data + 3 advertising) |

The device appears on the USB bus as:

```
Bus 003 Device 026: ID 1915:522a Nordic Semiconductor ASA nRF Sniffer for Bluetooth
```

---

## 2. Advantages Over Ubertooth for Validation

| Aspect | Ubertooth TX | nRF Sniffer | ESP32 nRF24 |
|--------|-------------|-------------|-------------|
| Radio type | CC2400 (general-purpose) | nRF52840 (BLE-native) | nRF24L01+ (Enhanced ShockBurst) |
| On-air signal | Approximates BLE spec | 100% BLE-spec compliant | nRF24-to-nRF24 optimized |
| Can transmit custom BLE | Yes (CC2400 GFSK) | No (capture only) | Yes (but not BLE-native) |
| Can receive BLE | Yes (passive sniff) | Yes (passive sniff) | Yes (BLE passive RX mode) |
| CRC verification | Ubertooth decodes CRC | Wireshark verifies CRC | Not available (CRC disabled) |
| Protocol decode | `ubertooth-btle` + `tshark` | Built-in Wireshark dissector | Custom firmware parsing |
| Maintenance status | Community-maintained, last release 2020 | Actively maintained by Nordic | Project-maintained |

**Key advantage:** The nRF52840's native BLE radio guarantees that any packet it
captures is a valid BLE packet — there is no risk of modulation-quality issues.
This makes it the ideal **second opinion** when validating ESP32 nRF24 sniffer output.

**Limitation for active TX testing:** The nRF Sniffer firmware is receive-only.
To transmit BLE advertising packets, use the Ubertooth (despite CC2400 modulation
issues) or a second nRF24L01+ module configured in BLE TX mode.

---

## 3. Setup

### 3.1 Download the nRF Sniffer package

Obtain the nRF Sniffer for Bluetooth LE from Nordic Semiconductor's developer
website. The package contains:
- Sniffer firmware hex file (`.hex`)
- Python capture script (`nrf_sniffer_ble.bat` / `.sh`)
- Wireshark extcap plugin files

> **Note:** The Nordic Semiconductor website blocks automated access. Visit the
> product page manually in a browser to download the package. Search for
> "nRF Sniffer for Bluetooth LE" on the Nordic Semiconductor website.

### 3.2 Install Python dependencies

```bash
pip3 install --break-system-packages pyserial pandas
```

Check the `requirements.txt` in the sniffer package for the authoritative list
of dependencies.

### 3.3 Flash the sniffer firmware

The dongle must be running the nRF Sniffer firmware, not the default bootloader.

**Method A — Using nrfutil:**

```bash
# Install nrfutil
pip3 install --break-system-packages nrfutil

# Put the dongle in DFU mode (press RESET while holding BOOT)
# Flash the sniffer firmware
nrfutil dfu serial -pkg nrf_sniffer_for_bluetooth_le_xxx.zip -p /dev/ttyACM0
```

**Method B — Using nRF Connect for Desktop (GUI):**

1. Install nRF Connect for Desktop from Nordic Semiconductor
2. Open the Programmer app
3. Select the nRF52840 Dongle
4. Load the sniffer firmware hex file
5. Click "Write" to flash

### 3.4 Install the Wireshark extcap plugin

The nRF Sniffer integrates with Wireshark as an extcap capture interface.

```bash
# Create the extcap directory
mkdir -p ~/.local/lib/wireshark/extcap/

# Copy the extcap scripts from the sniffer package
cp -r /path/to/nrf-sniffer/extcap/* ~/.local/lib/wireshark/extcap/
chmod +x ~/.local/lib/wireshark/extcap/nrf_sniffer_ble*

# Restart Wireshark to detect the new interface
```

After restarting Wireshark, the nRF Sniffer interface should appear in the
capture interfaces list.

### 3.5 Verify the setup

```bash
# Check that the dongle is detected
lsusb | grep "1915:522a"

# List Wireshark extcap interfaces
~/.local/lib/wireshark/extcap/nrf_sniffer_ble --extcap-interfaces

# Start a test capture (10 seconds on channel 37)
# In Wireshark: select "nRF Sniffer for Bluetooth LE" interface, start capture
```

---

## 4. Capture Workflow

### 4.1 Capturing BLE advertising packets

**Using Wireshark (GUI):**

1. Start Wireshark
2. Select "nRF Sniffer for Bluetooth LE" from the capture interfaces list
3. Configure the channel: in the capture options, set the advertising channel
   (37, 38, or 39) or select "All advertising channels"
4. Start capture
5. Filter for BLE advertising: `btle.advertising_address`

**Using tshark (CLI):**

```bash
# Capture on specific interface (use the extcap interface name from Wireshark)
tshark -i "nRF Sniffer for Bluetooth LE" -a duration:30 -w /tmp/nrf_capture.pcap
```

**Using the nRF Sniffer Python script directly:**

```bash
# The exact command depends on the sniffer package version
# Typically something like:
python3 /path/to/nrf-sniffer/capture.py --port /dev/ttyACM0 --channel 37 --out /tmp/ch37.pcap
```

### 4.2 BLE advertising channel selection

The three BLE advertising channels capture different traffic:

| Channel | Frequency | Typical traffic |
|---------|-----------|---------------|
| 37 | 2402 MHz | Most phones and peripherals |
| 38 | 2426 MHz | Some devices alternate 37/38 |
| 39 | 2480 MHz | Some devices alternate all three |

For validation, capture on **channel 37** first — it has the most traffic in
most environments. If the ESP32 nRF24L01+ is configured for a specific channel,
capture on the same channel for direct comparison.

### 4.3 Decoding captures with tshark

```bash
# Show all BLE advertising packets with MAC addresses
tshark -r /tmp/nrf_capture.pcap -T fields \
    -e btle.channel_idx \
    -e btle.pdu_type \
    -e btle.advertising_address \
    -e btle.advertising_data

# Filter for a specific MAC address
tshark -r /tmp/nrf_capture.pcap -Y "btle.advertising_address == ca:fe:ba:be:00:01" \
    -T fields -e btle.advertising_data

# Show detailed BLE layer decode
tshark -r /tmp/nrf_capture.pcap -V -O btle

# Count packets by PDU type
tshark -r /tmp/nrf_capture.pcap -T fields -e btle.pdu_type | sort | uniq -c | sort -rn
```

---

## 5. Cross-Validation with ESP32 nRF24L01+

### 5.1 Principle

Both the nRF52840 Sniffer and the ESP32 nRF24L01+ receive the **same over-the-air
BLE packets**. After dewhitening, both should produce identical PDU bytes. Comparing
their outputs validates the ESP32's dewhitening and PDU parsing.

### 5.2 Side-by-side capture

1. **Start nRF Sniffer** capturing on the target channel (e.g., ch37)
2. **Start ESP32** nRF24L01+ sniffer on the same channel using `nrf24::ble::switch_channel(radio, 37)`
3. **Let both run for 30+ seconds** capturing real BLE traffic
4. **Decode nRF Sniffer PCAP** with tshark
5. **Compare**: PDU types, MAC addresses, and advertising data must match byte-for-byte

### 5.3 End-to-end test with Ubertooth TX

When using `tools/e2e_crossval_test.sh`, add `--nrf` to include nRF Sniffer
capture alongside the ESP32:

```bash
# Full cross-validation: Ubertooth TX → ESP32 RX + nRF Sniffer RX
./tools/e2e_crossval_test.sh --nrf -t 60 -v

# Result interpretation:
#   PASS = Both ESP32 and nRF Sniffer see the Ubertooth MAC
#   FAIL (nRF sees, ESP32 doesn't) = ESP32 configuration issue
#   FAIL (neither sees) = Ubertooth TX issue (likely CC2400 modulation)
```

**Note:** Due to the CC2400 TX limitation (see Ubertooth skill §5), the nRF
Sniffer may see Ubertooth packets while the ESP32 nRF24 does not. This is
expected — the nRF52840's BLE-native receiver tolerates the CC2400's imperfect
GFSK better than the nRF24L01+.

### 5.4 Comparing PDU bytes

The nRF Sniffer outputs fully decoded BLE PDU data (dewhitened, CRC-verified).
The ESP32 outputs dewhitened PDU bytes from `nrf24::ble::dewhiten()`.

For a given advertising packet, the bytes should match:

| Field | nRF Sniffer (tshark) | ESP32 serial output |
|-------|----------------------|---------------------|
| PDU type | `btle.pdu_type` | Parsed from buf[0] lower 4 bits |
| AdvA (MAC) | `btle.advertising_address` | buf[2]..buf[7] (6 bytes) |
| AdvData | `btle.advertising_data` | buf[8]..buf[8+len-1] |

### 5.5 What a match proves

**If both sniffers decode the same packets to the same bytes:**

- ✅ `nrf24::ble::dewhiten()` is correct for that channel's whitening seed
- ✅ `nrf24::ble::swapbits()` bit-reversal is correct
- ✅ `nrf24::ble::channel_to_rf_ch()` frequency mapping is correct
- ✅ `nrf24::ble::ADV_ACCESS_ADDR` byte order is correct
- ✅ ESP32 SPI communication with nRF24 is working
- ✅ nRF24 register configuration is correct

**If the nRF Sniffer sees packets but the ESP32 doesn't:**

- ❌ Check nRF24 register configuration (EN_AA, EN_CRC, RX_ADDR_P0, RF_CH)
- ❌ Check wiring (CE, CSN, MOSI, MISO, SCK)
- ❌ Check CE pin is HIGH (use `GPIO_MODE_INPUT_OUTPUT`)
- ❌ Check MOSI pin is not set to `GPIO_MODE_INPUT`
- ❌ Run `nrf24::diag::spi_comm_test()` and `nrf24::diag::verify_ble_rx()`

**If neither sniffer sees Ubertooth-transmitted packets:**

- ⚠️ Ubertooth may not be transmitting (check firmware, USB, channel settings)
- ⚠️ Ubertooth and receivers may be too far apart
- ⚠️ See Ubertooth skill §5 for the CC2400 TX limitation

---

## 6. Common Issues

### 6.1 "Dongle not detected by Wireshark"

```bash
# Check USB enumeration
lsusb | grep "1915:522a"

# If not visible, try:
# 1. Unplug and replug the dongle
# 2. Press the RESET button on the dongle
# 3. Flash the sniffer firmware (the bootloader mode shows a different PID)

# Verify extcap script is executable
chmod +x ~/.local/lib/wireshark/extcap/nrf_sniffer_ble*

# List extcap interfaces
~/.local/lib/wireshark/extcap/nrf_sniffer_ble --extcap-interfaces
```

### 6.2 "Dongle stuck in bootloader mode"

After power-on, the nRF52840 Dongle enters the bootloader (indicated by the red
LED). It will not function as a sniffer until the sniffer firmware is flashed.

1. Press RESET while holding BOOT to enter DFU mode
2. Flash the sniffer firmware hex file using `nrfutil` or nRF Connect
3. After flashing, the dongle should enumerate with PID `522A`

### 6.3 "nRF Sniffer capture is empty"

- Verify the dongle is running sniffer firmware, not the bootloader
- Confirm the channel selection matches real BLE traffic (try channel 37)
- Check antenna orientation — the dongle's PCB antenna is directional
- Increase distance from the BLE device (too close can cause overload)
- In Wireshark, check the capture options: "Promiscuous mode" should be enabled

### 6.4 "Wireshark doesn't show the nRF Sniffer interface"

```bash
# Check extcap directory permissions
ls -la ~/.local/lib/wireshark/extcap/

# Verify the extcap scripts work standalone
~/.local/lib/wireshark/extcap/nrf_sniffer_ble --extcap-interfaces

# If standalone works but Wireshark doesn't see it:
# 1. Check Wireshark's extcap path in Edit → Preferences → Extcap
# 2. Restart Wireshark completely
# 3. Check for Python import errors:
python3 ~/.local/lib/wireshark/extcap/nrf_sniffer_ble --extcap-interfaces 2>&1
```

### 6.5 "Python dependency errors"

The nRF Sniffer capture script requires specific Python packages:

```bash
# Install required packages
pip3 install --break-system-packages pyserial pandas

# If the extcap script has import errors, check its shebang line
# It may reference a different Python interpreter
head -1 ~/.local/lib/wireshark/extcap/nrf_sniffer_ble*
```

### 6.6 "Firmware flashing fails"

If `nrfutil dfu serial` fails:

1. Verify the dongle is in DFU mode (red LED pulsing)
2. Try a different USB port (some USB hubs interfere)
3. Check the serial port: `ls /dev/ttyACM*`
4. Try with explicit baud rate: `nrfutil dfu serial -pkg firmware.zip -p /dev/ttyACM0 -b 115200`

---

## 7. Self-Reflection Clause

After any nRF Sniffer debugging or cross-validation session, agents MUST answer:

1. **Why was this cross-validation needed?** What nRF24 behaviour was being
   validated — dewhitening, address matching, channel mapping, or something else?
2. **Did the nRF Sniffer capture confirm or contradict the ESP32 output?** If
   they matched, the ESP32 firmware is validated for that aspect. If they differed,
   the ESP32's dewhitening or PDU parsing has a bug.
3. **Does this lesson need to be captured?** If yes, update this skill or the
   relevant learning doc in `docs/learning/` with the finding.

---

## 8. References

- Nordic Semiconductor nRF Sniffer for Bluetooth LE product page
  (visit browser: https://www.nordicsemi.com/Products/Development-tools/nRF-Sniffer-for-Bluetooth-LE)
- nRF52840 Dongle product page: https://www.nordicsemi.com/Products/Development-hardware/nRF52840-Dongle
- Wireshark extcap documentation: https://www.wireshark.org/docs/wsug_html_chunked/ChCaptureAddCaptureInterfaceSection.html
- Bluetooth Core Specification Vol 6 Part B (BLE physical layer):
  https://www.bluetooth.com/specifications/specs/core-specification/
- Project learning docs:
  - `docs/learning/ubertooth-update-and-diagnostics.md` §9 — nRF Sniffer setup
  - `docs/learning/ubertooth-ble-testing.md` — BLE testing and dewhitening validation
- Project tools:
  - `tools/e2e_crossval_test.sh` — end-to-end cross-validation with `--nrf` option