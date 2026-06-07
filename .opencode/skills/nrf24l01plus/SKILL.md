---
name: nrf24l01plus
description: "nRF24L01+ chip-specific gotchas, traps, and diagnostic protocols for this project. Triggered when configuring, debugging, or reviewing code that touches the nRF24L01+ radio — register write order, SPI byte conventions, GPIO caveats, clone chip quirks, and the self-reflection clause for continuous learning."
---

# nRF24L01+ Hardware-Specific Knowledge

## Purpose

The nRF24L01+ has several hardware behaviours that violate reasonable assumptions.
Each one has caused **months of silent failure** in this project. This skill captures
chip-specific traps so agents can recognise and avoid them.

This skill is **complementary** to `datasheet-verification` (which covers general
register verification) and the `assumption-trap` protocol (which governs ambiguity
handling). It focuses on **gotchas already encountered and diagnosed**.

---

## 1. Critical Traps

### 1.1 EN_AA / EN_CRC Override Trap

**Rule: Write `EN_AA = 0x00` BEFORE writing `CONFIG` with `EN_CRC = 0`.**

The nRF24L01+ datasheet §CONFIG states: *"If the EN_AA is set for any pipe,
the EN_CRC bit in the CONFIG register is forced high."*

The power-on reset value of EN_AA is **0x3F** (all 6 pipes auto-ACK enabled). If you
write CONFIG with EN_CRC=0 while EN_AA is still 0x3F:

1. The hardware overrides EN_CRC to 1 internally
2. After clearing EN_AA=0x00, the stored CONFIG value **still has EN_CRC=1**
3. The nRF24 expects CRC in every received packet
4. BLE packets use CRC-24 (3 bytes), not nRF24 CRC-1/CRC-2 (1–2 bytes)
5. **Every BLE packet fails CRC check and is silently discarded**
6. The RX FIFO stays empty; `read_payload()` returns 0xFE (power-on default)

**Symptom pattern:**

| Symptom | Cause |
|---------|-------|
| RX FIFO always 0xFE | CRC rejection — no valid packets reach FIFO |
| RX_DR set but no real data | FIFO read from empty buffer returns 0xFE |
| RPD signal very rare (0.1%) | nRF24 can detect RF energy but can't demodulate with wrong CRC |
| CONFIG read-back shows 0x0B instead of 0x03 | EN_CRC forced on by EN_AA at write time |

**Correct code:**

```cpp
// CORRECT — EN_AA zeroed first
auto en_aa = nrf24::EnAa{};
for (int i = 0; i < 6; ++i) en_aa.pipe[i] = false;
radio.write_reg(en_aa);     // EN_AA=0x00 — no forcing condition exists
auto cfg = nrf24::Config{};
cfg.power_mode = nrf24::PowerMode::Up;
cfg.primary    = nrf24::PrimaryMode::RX;
cfg.crc_mode   = nrf24::CrcMode::Disabled;
radio.write_reg(cfg);       // EN_CRC=0 — accepted as-is
```

**Broken code (do NOT do this):**

```cpp
// BROKEN — CONFIG written before EN_AA
radio.write_reg(cfg);       // EN_CRC=0 — OVERRIDDEN to 1 because EN_AA=0x3F
radio.write_reg(en_aa);     // EN_AA=0x00 — too late, CONFIG already stored with EN_CRC=1
```

**Reference:** nRF24L01+ Product Specification v1.0 §CONFIG (page 54);
learning doc: `docs/learning/nrf24-enaa-encrc-override.md`

---

### 1.2 SPI Address Register Byte Order Trap

**Rule: Multi-byte address registers (RX_ADDR_P0, RX_ADDR_P1, TX_ADDR) use
LSByte-first SPI write order, per datasheet §8.3.1.**

The nRF24L01+ datasheet §8.3.1 states: *"LSByte is written first"* for multi-byte
address registers. On-air, the nRF24 transmits the MSByte first (§7.3: "MSB to the
left"). This reversal creates a common and devastating trap.

**The BLE advertising access address transformation chain:**

```
BLE Access Address: 0x8E89BED6

Step 1: BLE on-air (LSByte-first): D6 BE 89 8E
Step 2: Per-byte bit reversal (swapbits): 6B 7D 91 71  (MSByte-first on air)
Step 3: nRF24 SPI write (LSByte-first): 71 91 7D 6B  (FIRST byte = LSByte)
```

**Correct constant (LSByte-first SPI order):**

```cpp
inline constexpr uint8_t ADV_ACCESS_ADDR[4] = {0x71, 0x91, 0x7D, 0x6B};
```

**Wrong constant (MSByte-first on-air order — the original bug):**

```cpp
// BROKEN — bytes in MSByte-first on-air order, not LSByte-first SPI order
inline constexpr uint8_t ADV_ACCESS_ADDR[4] = {0x6B, 0x7D, 0x91, 0x71};
```

**Why this is hard to catch:**

1. **Self-consistent readback:** Writing the wrong byte sequence and reading the same
   register back returns the same wrong sequence — comparison always passes.
2. **RPD works but FIFO empty:** The nRF24 still detects RF energy (RPD=1), but
   address matching fails silently.
3. **Two independent inversions compose:** BLE LSBit-first + nRF24 LSByte-first;
   getting one right but not the other produces plausible-looking but wrong results.

**Symptom pattern:**

| Symptom | Cause |
|---------|-------|
| RPD=1 occasionally | RF energy detected (physical layer works) |
| FIFO always empty | Address never matches — no packets enter FIFO |
| 0xFE FIFO reads | Power-on default, not real data |

**Prevention:**

- Always verify multi-byte register SPI byte order against the datasheet AND an
  independent reference implementation (e.g., Dmitry Grinberg's reference at
  `dmitry.gr`).
- Never trust readback verification alone for self-consistent errors.
- Derive expected values independently from the known BLE AA `0x8E89BED6`.

**Reference:** nRF24L01+ Product Specification §8.3.1 and §7.3;
learning doc: `docs/learning/nrf24-spi-address-byte-order.md`;
cross-verified against Dmitry Grinberg's implementation at `dmitry.gr`

---

### 1.3 CE GPIO Configuration Trap

**Rule: Use `GPIO_MODE_INPUT_OUTPUT` for the CE pin, not `GPIO_MODE_OUTPUT`.**

On ESP32, `GPIO_MODE_OUTPUT` disables the input buffer. `gpio_get_level()` on an
output-only pin **always returns 0** regardless of the actual output level. This makes
CE readback diagnostics show LOW even when the pin is physically HIGH.

```cpp
// BROKEN — output-only, gpio_get_level() always returns 0
gpio_config_t gpio_cfg = {
    .mode = GPIO_MODE_OUTPUT,  // ← input buffer disabled
};

// CORRECT — input-output, gpio_get_level() returns actual pad level
gpio_config_t gpio_cfg = {
    .mode = GPIO_MODE_INPUT_OUTPUT,  // ← input buffer enabled
};
```

**Additional note:** CE has been migrated from GPIO5 to GPIO4. GPIO5 is SPI3_HOST's
native IO_MUX CS0 pin, and using it for CE created a latent conflict. Do NOT move CE
back to GPIO5.

**Reference:** ESP-IDF GPIO & RTC GPIO documentation,
learning doc: `docs/learning/mosi-spi-register-verification.md`

---

### 1.4 MOSI GPIO_MODE_INPUT Trap

**Rule: Never manually change GPIO direction on SPI pins after `spi_bus_initialize()`.**

Setting `gpio_set_direction(mosi_pin, GPIO_MODE_INPUT)` after SPI initialization
breaks **all** subsequent SPI write operations. The MOSI pin disconnects from the
SPI peripheral, making all write commands silently fail.

The ESP-IDF SPI Master Driver manages pin direction automatically. After
`spi_bus_initialize()` and `spi_bus_add_device()`, the driver controls the IOMUX
routing and GPIO configuration for all SPI pins (MOSI, MISO, SCK, CSN). Calling
`gpio_set_direction()` on any of these pins overwrites the driver's configuration.

**Symptoms:**

| Symptom | Explanation |
|---------|-------------|
| `switch_channel()` appears to work, but nRF24 doesn't change channel | Channel-switch command sent on MOSI, which the radio never receives |
| `clear_irq_flags()` doesn't clear IRQ flags | STATUS register write command never received |
| Any register write after `gpio_set_direction()` is silently dropped | MOSI is disconnected from SPI peripheral |
| Read operations may still return valid data | nRF24 always outputs STATUS on MISO during command byte phase |

**Fix:** Remove ALL `gpio_set_direction()` calls on SPI pins. Do not set MOSI to
`GPIO_MODE_INPUT` even during RX-only operation.

```cpp
// REMOVED: gpio_set_direction(hal.mosi_pin(), GPIO_MODE_INPUT);
// The SPI driver manages MOSI pin direction automatically.
// Manually changing it disconnects the pin from the SPI peripheral.
```

**For reducing RF noise during RX-only operation**, the correct approaches are:

1. Use `spi_bus_remove_device()` + `gpio_set_direction()` only if the SPI bus is
   no longer needed — but this requires full re-initialization to resume.
2. Let the SPI driver manage the pins — the noise reduction of tri-stating MOSI
   is minimal compared to the risk of breaking SPI.
3. Add a series resistor on the MOSI trace — a hardware-level fix that doesn't
   affect the SPI driver.

**Reference:** ESP-IDF SPI Master Driver documentation;
learning doc: `docs/learning/mosi-spi-register-verification.md`

---

### 1.5 Clone Chip Detection (Si24R1 / BK2425)

**Rule: Always run `nrf24::diag::spi_comm_test()` before configuration.
Always verify CONFIG read-back after writing.**

On genuine nRF24L01+, writing EN_AA=0x00 then CONFIG with EN_CRC=0 works correctly.
On Si24R1 clone chips, the CRC forcing may persist even after EN_AA=0x00.

**Clone chip symptoms:**

| Symptom | Meaning |
|---------|---------|
| STATUS reads 0xFF | MISO stuck high — no device, or MISO not connected |
| STATUS reads 0x0E | SPI reads working, device is alive (POR default) |
| POR values all wrong but STATUS is OK | SPI writes not accepted — clock too fast for clone |
| EN_CRC forced on despite EN_AA=0x00 | Si24R1 clone chip |
| FIFO always returns 0xFE | Reading from empty FIFO — CRC rejection or no RX |

**SPI clock speed:** Genuine nRF24L01+ supports up to 10 MHz. Clone chips typically
need **1–2 MHz maximum**. At 8 MHz (ESP-IDF default), clones silently ignore register
writes. The project uses 1 MHz for clone compatibility.

**Detection procedure** (built into `nrf24::diag::spi_comm_test()`):

```cpp
nrf24::EspIdfHal hal;
hal.init(pins);
nrf24::Driver radio(hal);
bool spi_ok = nrf24::diag::spi_comm_test(radio);
if (!spi_ok) {
    // SPI failed — check wiring, power, module type
    return;
}
// SPI confirmed working, proceed with configuration
nrf24::ble::configure_rx(radio);
```

**Reference:** learning doc: `docs/learning/nrf24-spi-clone-chip-diagnostic.md`

---

## 2. Register Write Order — Correct Initialization Sequence

The correct order for configuring the nRF24L01+ for BLE passive reception is:

1. **Run `spi_comm_test()`** — validates SPI, detects clones, leaves EN_AA=0x00
2. **Write EN_AA = 0x00** — disable auto-ACK on all pipes (CRITICAL: before CONFIG)
3. **Write CONFIG** — PWR_UP=1, PRIM_RX=1, EN_CRC=0 (CRC disabled for BLE)
4. **Write EN_RXADDR** — enable only the pipes needed (pipe 0 for BLE advertising)
5. **Write SETUP_AW** — set address width (4 bytes for BLE access address)
6. **Write RF_CH** — set channel (2 for BLE ch37, 26 for ch38, 80 for ch39)
7. **Write RF_SETUP** — set data rate (1 Mbps for BLE) and TX power
8. **Write RX_ADDR_P0** — BLE access address in LSByte-first SPI order
9. **Write RX_PW_P0** — set payload width (32 for BLE)
10. **Write DYNPD = 0x00** — disable dynamic payload
11. **Write FEATURE = 0x00** — disable all features
12. **Set CE HIGH** — enter RX mode

**The ONLY ordering constraint that matters:** EN_AA must be 0x00 before CONFIG
is written with EN_CRC=0. All other registers can be written in any order, but
EN_AA before CONFIG is mandatory.

**Reference:** nRF24L01+ Product Specification §6.1.2 (state machine);
learning doc: `docs/learning/nrf24-enaa-encrc-override.md`

---

## 3. Diagnostic Protocol

### 3.1 Three-Stage SPI Communication Test (`nrf24::diag::spi_comm_test`)

Run BEFORE any register configuration. Must be called on a fresh power-on state.

| Stage | Purpose | Method |
|-------|---------|--------|
| 1. POR check | Verify SPI reads work | Read registers, compare against datasheet POR values. Warm-boot detection: if STATUS != 0xFF but CONFIG != 0x08, skip remaining POR checks (module was previously programmed) |
| 2. Write-verify | Verify SPI writes work | Write known patterns (0x00, 0x3F, 0x55, 0x2A) to EN_AA, read back each. Mask reserved bits [7:6] (write 0x55, read back 0x15) |
| 3. Clone detection | Detect Si24R1 clones | Set EN_AA=0x00, write CONFIG with EN_CRC=0, check if EN_CRC stays forced on |

**Warm boot handling:** After an ESP32 software reset, the nRF24 module stays powered
and retains its previously programmed values. The POR check will falsely fail even
though SPI is working perfectly. If STATUS reads correctly (not 0xFF) but CONFIG is
not at its POR value (0x08), assume warm boot and skip directly to Stage 2
(write-verify).

**Stage 2 detail — EN_AA has 6 valid bits [5:0]:**

- `0x00` — all pipes disabled (important for BLE CRC=0 setup)
- `0x3F` — all pipes enabled (POR default)
- `0x55` — alternating 0/1 pattern (masked: `0x15` because bits [7:6] are reserved)
- `0x2A` — complementary alternating pattern

**After spi_comm_test:** EN_AA is left at 0x00. This is compatible with
`configure_rx()` which also writes EN_AA=0x00 as its first step.

### 3.2 Register Verification (`nrf24::diag::verify_ble_rx`)

Run AFTER `configure_rx()`. Reads back all programmed registers and compares against
expected values from the RxConfig.

**Critical checks:**

| Register | Expected | Failure Mode |
|----------|----------|-------------|
| CONFIG | EN_CRC=0 | If EN_CRC=1: EN_AA was non-zero when CONFIG was written, or clone chip |
| EN_AA | 0x00 | If non-zero: EN_CRC will be forced on |
| RF_SETUP | DataRate=1Mbps | If wrong: BLE advertising won't be demodulated |
| RX_ADDR_P0 | BLE AA in LSByte-first order | If wrong: no packets will match the address |
| FIFO_STATUS | rx_empty after flush | If rx_empty=0 with no traffic: stale data from previous run |

### 3.3 Zero-Packets Checklist

When all register read-backs PASS but `pkts=0` continuously:

| # | Check | Why It Matters |
|---|-------|---------------|
| 1 | CONFIG has EN_CRC=0 | nRF24 CRC rejects BLE packets |
| 2 | EN_AA = 0x00 | Non-zero EN_AA forces EN_CRC=1 |
| 3 | RX_ADDR_P0 = correct LSByte-first BLE AA | Wrong byte order = no address match |
| 4 | CE is HIGH during RX mode | nRF24 requires CE HIGH to stay in RX |
| 5 | MOSI pin is NOT set to GPIO_MODE_INPUT | Disconnects MOSI from SPI peripheral |
| 6 | No SPI pin conflicts with ESP32 GPIO routing | Strapping pins, flash pins, etc. |
| 7 | RF_CH matches expected BLE frequency | ch37→2, ch38→26, ch39→80 |
| 8 | RX_PW_P0 = 32 | Capture full BLE packets |
| 9 | BLE devices are actually advertising nearby | Use Ubertooth or phone BLE scanner to confirm |
| 10 | nRF24L01+ antenna and power supply (3.3V, stable) | Add 10µF + 100nF decoupling caps near VCC/GND |

### 3.4 Self-Consistent Error Detection

A self-consistent error is one where the wrong value is both written and used as the
expected value, so readback verification always passes. The SPI address byte order
bug was exactly this kind of error.

**Prevention:**

1. **Derive expected values independently.** Compute expected register bytes from the
   known BLE AA value `0x8E89BED6` using a separate code path from the constant used
   for writing.
2. **Test against real traffic.** Transmit a known BLE packet (e.g., via Ubertooth)
   and verify the nRF24 receives it. Register readback alone cannot catch
   self-consistent byte-order errors.
3. **Trace the full transformation chain.** Write out every step of byte/bit
   transformation and verify the final SPI bytes produce the correct on-air address.

**Reference:** learning doc: `docs/learning/nrf24-spi-address-byte-order.md` §5

---

## 4. Self-Reflection Clause

After fixing any nRF24L01+ bug, agents MUST answer these three questions:

1. **Why was this bug missed?** What assumption led to the incorrect code?
2. **What procedural safeguard would have caught it?** What test, review step,
   or verification protocol should be added?
3. **Does this lesson need to be captured?** If yes, update this skill or the
   relevant learning doc in `docs/learning/` with the lesson learned.

### Examples from This Project

| Bug | Why Missed | Safeguard Added |
|-----|-----------|-----------------|
| EN_AA/EN_CRC override | Didn't know about the hardware forcing rule | Write EN_AA=0x00 BEFORE CONFIG; verify read-back |
| SPI address byte order | Stopped transformation chain one step short; readback verification was self-consistent | Derive expected values independently from BLE AA; test against real traffic |
| MOSI GPIO_MODE_INPUT | Assumed GPIO direction was independent of SPI peripheral | Never change GPIO direction on SPI pins after spi_bus_initialize() |
| CE GPIO_MODE_OUTPUT | Assumed gpio_get_level() works on output-only pins | Use GPIO_MODE_INPUT_OUTPUT for pins that need readback |
| Si24R1 clone EN_CRC | Assumed all nRF24 modules are genuine Nordic parts | Run spi_comm_test() Stage 3; verify CONFIG read-back |

### Process for Updating This Skill

When a new nRF24L01+ trap is discovered:

1. Add it as a new subsection under §1 (Critical Traps) with:
   - A one-line **Rule** in bold
   - The mechanism that makes it a trap
   - Correct code example (using library vocabulary, no magic numbers)
   - Broken code example (commented out, marked `// BROKEN`)
   - Symptom pattern table
   - Reference to datasheet section and/or learning doc
2. Add the safeguard to the relevant section (§2, §3, or §4)
3. Commit with: `docs(skills): add nRF24L01+ trap — [trap name]`

---

## 5. Quick Reference — Common Pitfalls Table

| # | Pitfall | One-Line Rule | Symptom |
|---|---------|--------------|---------|
| 1 | EN_AA forces EN_CRC | Write EN_AA=0x00 BEFORE CONFIG | CONFIG=0x0B instead of 0x03; RX FIFO empty |
| 2 | SPI byte order | Multi-byte addr regs: LSByte first (§8.3.1) | RPD=1 but FIFO empty; readback passes |
| 3 | CE readback | Use GPIO_MODE_INPUT_OUTPUT, not GPIO_MODE_OUTPUT | gpio_get_level() returns 0 always |
| 4 | MOSI direction | Never change SPI pin GPIO after bus init | All SPI writes silently fail; reads work |
| 5 | Clone chips | Run spi_comm_test(); check EN_CRC read-back | Registers stuck at POR; EN_CRC forced on |
| 6 | Empty FIFO reads | Reading empty RX FIFO returns 0xFE, not 0x00 | 0xFE pattern interpreted as valid data |
| 7 | Warm boot | After ESP32 reset, nRF24 retains state | POR check fails; STATUS OK but CONFIG wrong |
| 8 | Dewhitening | Must bit-swap then Galois LFSR; seed = swapbits(ch) \| 0x02 | Decoded data is garbage |

---

## 6. References

- nRF24L01+ Product Specification v1.0 — All register definitions, byte order, state machine
  - §CONFIG (page 54): EN_CRC forcing by EN_AA
  - §8.3.1: LSByte-first multi-byte register write order
  - §7.3: MSByte-first on-air transmission ("MSB to the left")
  - Table 28: Register Map with POR default values
- Bluetooth Core Specification Vol 6 Part B — BLE physical layer
  - §1.3.1: LSBit-first transmission per byte
  - §3.2: Data whitening
- ESP-IDF SPI Master Driver: https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/peripherals/spi_master.html
- ESP-IDF GPIO & RTC GPIO: https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/peripherals/gpio.html
- Dmitry Grinberg, "Bit-banging Bluetooth Low Energy": http://dmitry.gr/?r=05.Projects&proj=11.%20Bluetooth%20LE%20fakery

### Learning Docs

- `docs/learning/nrf24-enaa-encrc-override.md` — EN_AA/EN_CRC register override trap
- `docs/learning/nrf24-spi-address-byte-order.md` — LSByte-first SPI write order
- `docs/learning/mosi-spi-register-verification.md` — MOSI GPIO bug, register verification, review findings
- `docs/learning/nrf24-spi-clone-chip-diagnostic.md` — SPI comm test, clone chip detection
- `docs/learning/ble-data-whitening-nrf24.md` — Dewhitening algorithm (Galois LFSR, swapbits)
- `docs/learning/nrf24-spi-basics.md` — SPI wiring and register map
- `docs/learning/nrf24l01plus-register-map.md` — Full register reference