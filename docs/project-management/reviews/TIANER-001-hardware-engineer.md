# Phase A Review: Hardware Engineer

| Field | Value |
|-------|-------|
| Reviewer | Hardware Engineer |
| Phase | A (Design Review) |
| Date | 2026-06-07 |
| Artifact | `docs/designs/inception.md` v0.5 |
| Review scope | Hardware compatibility, USB device mapping, thermal/power constraints, hardware failure modes, storage constraints, firmware version management |

---

## Area 1: USB Device Mapping and nRF52840 Product ID

**Finding:** Section 6.2 specifies the Nordic nRF52840 Dongle USB product ID as `1915:520f`. This is incorrect for the nRF Sniffer firmware configuration.

The Nordic nRF52840 Dongle has multiple USB product IDs depending on firmware state:

| State | Vendor ID | Product ID | Notes |
|-------|-----------|------------|-------|
| Bootloader (OpenSK / default) | `1915` | `522A` | Most common out-of-box state |
| nRF Sniffer firmware (CDC ACM) | `1915` | `520F` | Some older sniffer firmware versions |
| nRF Sniffer firmware (v4.x+) | `1915` | `522A` | Newer sniffer firmware uses same PID as bootloader |
| Development Kit (Segger OB) | `1366` | `1015` | Separate product entirely |

The PID `520f` appears in some Nordic documentation for older sniffer firmware versions, but the current nRF Sniffer for Bluetooth LE v4.0.0 (referenced in [^nrf-sniffer-guide]) typically enumerates with PID `522A` when running sniffer firmware. The actual PID depends on which firmware version is flashed and whether the dongle enumerates via the OpenSK bootloader or native USB.

Section 6.2 also includes a catch-all for `1366` (Segger OB on DK boards) but does not distinguish between DK and Dongle. The udev rule as written would match any Segger device, including J-Link debuggers that happen to be plugged in.

**Impact:** If the udev rule specifies `520f` but the dongle enumerates as `522A`, the dongle will not get the `plugdev` group assignment, and the sniffer wrapper will fail to access the device. The operator will see a permissions error with no clear diagnosis.

**Confidence:** 88 (PID values vary by firmware version; must verify against actual hardware)  
**Verdict:** FAIL — Product ID must be verified against actual hardware during T04 before any udev rule is committed. The document should list both `520f` and `522A` as candidate PIDs with a DECISION REQUIRED marker.

---

## Area 2: USB Port Count on Raspberry Pi CM5

**Finding:** Section 3 specifies "1 to 4 concurrent sniffers, configurable." The design in section 6.8.2 configures 4 sniffers (1 Ubertooth + 3 nRF). However, the Raspberry Pi Compute Module 5 has limited external USB connectivity.

The CM5 exposes USB via the CM5 I/O board, which provides:

- 2 × USB 2.0 ports (Type-A, via PCIe-to-USB controller on the I/O board)
- 1 × USB 3.0 port (Type-A, native from BCM2712, shared with PCIe)

Total: 3 externally accessible USB ports on the standard CM5 I/O board.

Additionally, the CM5 exposes a single USB-C port used for power/data in USB boot mode, but this is not available for peripheral connections in normal operation.

The design requires 4 USB dongles (1 Ubertooth + 3 nRF). With only 3 available USB ports, a USB hub is required. The document does not mention:

1. Whether a USB hub is required.
2. If a hub is used, whether it is powered or bus-powered (4 dongles may exceed the 500 mA per-port specification of USB 2.0).
3. USB hub introducing additional latency and potential packet loss at high data rates.
4. Udev rules for port-path identification become more complex with a hub in the path.

**Confidence:** 92  
**Verdict:** FAIL — The hardware configuration must be reconciled. Either reduce to 3 sniffers or document the USB hub requirement with power and latency implications.

---

## Area 3: Power Budget

**Finding:** No power budget is documented. The Raspberry Pi CM5 I/O board power supply must be evaluated:

| Component | Typical Current | Voltage | Power |
|-----------|----------------|---------|-------|
| CM5 + 8 GB RAM (idle) | ~300 mA | 5V | 1.5 W |
| CM5 + 8 GB RAM (loaded: PG, Grafana, API, sniffers) | ~800-1200 mA | 5V | 4-6 W |
| Ubertooth One | ~200 mA | 5V (USB) | 1.0 W |
| nRF52840 Dongle (each) | ~50 mA | 5V (USB) | 0.25 W |
| 3 × nRF52840 | ~150 mA | 5V | 0.75 W |
| USB hub (if powered) | ~200 mA | 5V | 1.0 W |
| USB hub (if bus-powered) | 0 (draws from host) | 5V | 0 |
| External storage (SD card) | ~100 mA | 3.3V | 0.33 W |
| **Total (without powered hub)** | ~1750 mA | 5V | **8.75 W** |
| **Total (with powered hub)** | ~1550 mA from Pi | 5V | **7.75 W from Pi** |

The official Raspberry Pi CM5 I/O board uses a USB-C power input. The Raspberry Pi 27W USB-C Power Supply provides 5V/5A (25W), which is sufficient. However, if a bus-powered USB hub is used, the 4 dongles + hub may exceed the USB 2.0 specification of 500 mA per port × 2 ports = 1000 mA total.

**The document does not specify:**
- The power supply model/wattage.
- Whether a powered USB hub is required.
- What happens to system stability when all services are under load + all dongles active.

**Confidence:** 85  
**Verdict:** FAIL — Power budget must be documented before hardware procurement. Insufficient power causes USB brownouts that manifest as random dongle disconnects — extremely difficult to debug.

---

## Area 4: Thermal Constraints

**Finding:** No thermal analysis is provided. The CM5 with 8 GB RAM under sustained load (PostgreSQL + Grafana + FastAPI + 4 sniffer processes + ingest bridges + tshark instances) may approach thermal limits, especially in the default passive-cooled CM5 I/O board enclosure.

Key concerns:

1. The CM5 I/O board ships with a small passive heatsink. Under sustained 6W load, junction temperature may approach 80°C without active cooling or additional heatsinking.
2. USB dongles in adjacent USB ports may mutually heat each other, especially if a hub concentrates them physically.
3. Thermal throttling on the BCM2712 reduces CPU clock from 2.4 GHz to 1.0 GHz, drastically reducing ingest throughput and potentially causing FIFO overflow.
4. The Pi is likely deployed in an enclosed space (rack, cabinet, desk with limited airflow).

**Confidence:** 75 (thermal behavior is environment-dependent)  
**Verdict:** ADVISORY — Thermal budget should be measured during T01 with all services running. Add a temperature monitoring alert to the Grafana pipeline-health dashboard.

---

## Area 5: Storage Constraints

**Finding:** Section 3 specifies:

- Primary storage: eMMC 32 GB
- Secondary storage: SanDisk SD card (data partition)

32 GB eMMC is tight for the system + data workload:

| Use | Estimated Size |
|-----|---------------|
| Raspberry Pi OS + packages | ~6 GB |
| PostgreSQL 17 + TimescaleDB | ~1 GB |
| Grafana | ~300 MB |
| Node.js + build artifacts | ~500 MB |
| Python + venvs | ~500 MB |
| Ubertooth + nRF tools | ~200 MB |
| System overhead + swap | ~2 GB |
| **Remaining for PCAP + DB** | **~21.5 GB** |

With 4 sniffers at ~135 MB per 30-min file per sniffer (uncompressed), raw PCAP grows at ~540 MB per 30 minutes = ~1.08 GB/hour. With 14-day retention and zstd compression at ~5:1, storage needed is ~1.08 × 24 × 14 / 5 ≈ 72.6 GB.

This exceeds the 32 GB eMMC even with aggressive compression. The SD card is specified as secondary storage but the document does not state what goes on it. If PCAP files go to SD, SD card endurance (limited write cycles) becomes a concern under continuous write load.

**Confidence:** 90  
**Verdict:** FAIL — A storage sizing calculation must be added to the design. The current hardware specification cannot support the stated retention policy with 4 concurrent sniffers.

---

## Area 6: Firmware Version Management

**Finding:** Section 5 specifies `ubertooth-tools: matching firmware release` and Section 8.1 references `deploy/scripts/install-ubertooth.sh` that "builds from source against the firmware version on the dongle." This is vague and problematic:

1. **Ubertooth firmware vs. tools version mismatch** is a common failure mode. The Ubertooth Getting Started guide [^ubertooth-getting-started] explicitly warns that firmware and tools must match. If the dongle has firmware v1.3 but tools are built from a newer commit, the SPI protocol may be incompatible, causing capture corruption.

2. **nRF dongle firmware flashing** is mentioned in T04 but no procedure is documented. The nRF52840 Dongle requires entering bootloader mode and flashing via `nrfutil device dfu`. The sniffer firmware download URL, version, and flashing procedure must be documented.

3. **No firmware version validation step.** There is no acceptance criterion that verifies firmware-tool version compatibility. T03 says `ubertooth-util -v` reports firmware version but does not specify what to do if it doesn't match.

**Confidence:** 90  
**Verdict:** FAIL — Firmware version requirements must be explicitly pinned. A pre-flight check comparing firmware version to expected tools version must be added to the sniffer wrapper or setup script.

---

## Area 7: Hardware Failure Modes

**Finding:** The document's failure mode tables (sections 8.1-8.2) cover software-level failures (process exit, disk full). Hardware-level failures are not addressed:

1. **USB dongle disconnect/reconnect:** A dongle that physically disconnects due to vibration, bad contact, or USB controller reset will cause the sniffer wrapper to exit. The udev rule will reassign the device on reconnect, but the systemd unit is bound to a template name, not a device path. The wrapper must re-associate with the reconnected dongle.

2. **SD card corruption:** SD cards under continuous write load fail faster than eMMC. No checksum or integrity verification is specified for the SD card data partition.

3. **Ubertooth antenna failure:** The Ubertooth One has an external antenna connector. If the antenna disconnects, the device still enumerates on USB but receives no RF signal. The heartbeat mechanism reports the sniffer as "running" but the data stream is empty. This is a silent failure.

4. **nRF dongle CDC ACM serial lockup:** Under certain conditions (buffer overflow, firmware bug), the nRF52840 CDC ACM serial device can stop responding while remaining enumerated. The wrapper script won't detect this — it will appear to be writing successfully to the serial device while no packets flow.

**Confidence:** 80  
**Verdict:** FAIL — Hardware failure modes must be documented. At minimum: USB disconnect recovery, antenna-failure detection (zero-packet alert while heartbeat is running), and SD card integrity must be addressed.

---

## Area 8: eMMC vs SD Card Architecture

**Finding:** The document specifies eMMC as primary and SD as secondary, but does not define what lives where:

- Should PostgreSQL data directory (with TimescaleDB hypertable chunks) go on eMMC or SD?
- Should PCAP files go on eMMC or SD?
- Should the swap file go on eMMC (fast) or SD (slow but spares eMMC writes)?

PostgreSQL's random-write pattern is brutal on SD card endurance. PCAP sequential writes are more SD-friendly. The document needs a partition layout that places each workload on the appropriate medium.

**Confidence:** 78  
**Verdict:** ADVISORY — Recommend adding a partition layout diagram and a rationale for data placement.

---

## Summary of Findings

| # | Area | Severity | Confidence | Verdict | Description |
|---|------|----------|-----------|---------|-------------|
| 1 | nRF52840 USB PID | Blocking | 88 | FAIL | `520f` may be wrong; `522A` is more likely for current firmware |
| 2 | USB port count | Blocking | 92 | FAIL | CM5 has 3 USB ports, design needs 4 dongles |
| 3 | Power budget | Major | 85 | FAIL | No power budget documented |
| 4 | Thermal constraints | Major | 75 | ADVISORY | No thermal analysis, risk of throttling |
| 5 | Storage constraints | Blocking | 90 | FAIL | 32 GB eMMC insufficient for 4 sniffer × 14-day retention |
| 6 | Firmware version management | Major | 90 | FAIL | No explicit version pin, no version compatibility check |
| 7 | Hardware failure modes | Major | 80 | FAIL | 4 hardware failure modes undocumented |
| 8 | eMMC/SD architecture | Minor | 78 | ADVISORY | No partition layout or data placement rationale |

---

## Overall Verdict

**CONDITIONAL PASS** — 3 blocking findings (USB PID, port count, storage) must be resolved before hardware procurement. The remaining findings are major but can be resolved during implementation with design updates.

Resolution path:
1. Verify nRF52840 PID against actual hardware (T04).
2. Document USB hub requirement or reduce to 3 concurrent sniffers.
3. Add storage sizing calculation; consider NVMe (mentioned as optional in section 3 but not provisioned).
4. Add power and thermal budget sections.
5. Document firmware version pins and compatibility checks.

---

## Self-Audit Checklist

| # | Check | Status |
|---|-------|--------|
| 1 | Read the complete artifact? | YES — all 2910 lines |
| 2 | Every finding includes a confidence score? | YES |
| 3 | Every finding is actionable? | YES — specific hardware specs and actions |
| 4 | No speculative claims without evidence? | YES — PID claim flagged as requiring verification |
| 5 | Blocking findings justified by severity? | YES — wrong PID = no access; 3 ports < 4 dongles; storage overflow |
| 6 | Advisory findings clearly separated? | YES — areas 4 and 8 are advisory |
| 7 | No duplicate findings across areas? | YES |
| 8 | Verdict is consistent with finding severities? | YES — blocking but resolvable → CONDITIONAL PASS |

---

## Flags for PM

| Flag ID | Type | Description | Urgency |
|---------|------|-------------|---------|
| FLAG-HE-001 | Decision | nRF52840 PID: verify `520f` vs `522A` against actual hardware. Update udev rule accordingly. | Blocking |
| FLAG-HE-002 | Decision | USB port count: 3 available, 4 required. Add USB hub or reduce sniffers? | Blocking |
| FLAG-HE-003 | Decision | Storage budget: 32 GB eMMC insufficient for designed workload. Add NVMe or SD-only PCAP storage? | Blocking |
| FLAG-HE-004 | Decision | Power supply wattage: specify and document. Is 27W sufficient with USB hub? | High |
| FLAG-HE-005 | Ticket | Add firmware version pin and pre-flight compatibility check to setup script. | High |
| FLAG-HE-006 | Decision | Antenna-failure detection: add "zero packets while heartbeat running" alert. | Medium |