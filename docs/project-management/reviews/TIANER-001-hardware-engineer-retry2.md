# Phase A Review — Hardware Engineer Specialist (Retry #2)

**Reviewer:** Hardware Engineer
**Date:** 2026-06-07
**Phase:** A (A1-HW, retry #2)

## VERDICT: REJECTED
**Blocker count:** 6 (all confidence ≥80)

### F1: nRF52840 Sniffer PID Wrong (Conf: 95)
udev rules use `1915:520f` but sniffer firmware PID is `522A` (verified in project's own nrf52840-sniffer skill). Will silently fail to enumerate hardware.

### F2: C01 Design Document Missing (Conf: 90)
`c01-platform-infrastructure.md` does not exist. Layer 0 dependency that blocks C02, C03, C05.

### F3: USB Symlinks Non-Deterministic (Conf: 85)
udev rules use `%n` (kernel device number) which changes across reboots. Must use physical port path binding (`KERNELS=="x-y.z"`) for deterministic `/dev/tianer/nrf0` mapping.

### F4: Storage Budget Missing (Conf: 85)
No formal storage calculation. 32 GB eMMC + SD — must verify: OS (~10GB) + DB (compressed chunks) + PCAP archives + logs doesn't exceed capacity.

### F5: USB Power Budget Missing (Conf: 80)
Required by component-breakdown §5.2 but absent. Cannot confirm 4-sniffer + hub within CM5 power budget.

### F6: Thermal Envelope Missing (Conf: 80)
Required by component-breakdown §5.2 but absent. CM5 throttles at ~80°C. Sustained 4-sniffer + DB + API load needs thermal analysis.

### Advisory (F7-F10)
F7: Udev symlink prefix conflict (/dev/blesniff/ vs /dev/tianer/). Standardize to /dev/tianer/. (Conf: 75)
F8: USB hotplug behavior not addressed. Add BindsTo= and udev TAG+="systemd". (Conf: 70)
F9: USB hub is single point of failure for all 4 sniffers. (Conf: 65)
F10: SD card class not specified. Minimum UHS-I Class 10 recommended. (Conf: 60)

## Hardware Readiness
- Most components are 90-100% buildable without hardware using mock/pcap-replay fixtures
- C03 most blocked (~60% buildable pre-HW — sniffer wrappers need hardware validation)
- MVP pipeline ~80% buildable without hardware
