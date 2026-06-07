---
name: ble-protocol
description: "Bluetooth Low Energy protocol details for nRF24L01+ sniffing and transmission — advertising channels, access addresses, PDU types and header format, data whitening (Galois LFSR), bit order conventions (LSbit-first vs MSbit-first), and CRC-24. Triggered when working with BLE advertising, PDU types, data whitening, access addresses, channel mapping, dewhitening, or any Bluetooth Low Energy protocol detail."
---

# BLE Protocol Knowledge for nRF24L01+ Projects

## Purpose

This skill captures BLE protocol details that are essential for configuring the
nRF24L01+ to sniff or transmit BLE advertising packets. It covers the exact byte
layouts, bit order conventions, and mathematical transformations required — all of
which have caused bugs in this project.

This skill is **complementary** to `nrf24l01plus` (chip-specific traps) and
`esp-idf` (platform APIs). It focuses on **BLE protocol specifics** that must be
implemented correctly in software.

---

## 1. BLE Advertising Channels

BLE uses 40 channels in the 2.4 GHz ISM band, divided into two categories:

| Channel Type | Channel Indices | Frequency Range | Sniffable with nRF24L01+? |
|--------------|----------------|-----------------|---------------------------|
| Advertising  | 37, 38, 39      | 2402, 2426, 2480 MHz | **Yes** (1 Mbps GFSK) |
| Data         | 0–36            | 2404–2478 MHz    | **No** (uses adaptive frequency hopping, CRC, and data whitening that require full BLE stack) |

### 1.1 Channel-to-Frequency Mapping

| BLE Channel | Frequency (MHz) | nRF24 Channel | nRF24 Register Value |
|-------------|-----------------|---------------|---------------------|
| 37          | 2402            | 2             | RF_CH = 2           |
| 38          | 2426            | 26            | RF_CH = 26          |
| 39          | 2480            | 80            | RF_CH = 80          |

**Derivation:** BLE frequency = 2402 + 2 × channel_index.

- Channel 37: 2402 + 2 × 0... no. The BLE spec defines:
  - Channel 37 = 2402 MHz
  - Channel 38 = 2426 MHz
  - Channel 39 = 2480 MHz

These are NOT a simple linear formula from the channel index. The mapping is
defined by Table 2.3 in Bluetooth Core Spec Vol 6 Part A.

The nRF24 `RF_CH` register value = (frequency_MHz - 2400) / 1_MHz_step.
So: ch37→RF_CH=2, ch38→RF_CH=26, ch39→RF_CH=80.

### 1.2 Data Channels (Not Accessible to nRF24L01+)

Data channels 0–36 use frequency hopping, CRC-24, and data whitening managed by
the full BLE link layer. The nRF24L01+ in ShockBurst/Enhanced ShockBurst mode
cannot participate in BLE data connections. Only advertising channel sniffing
(or advertising packet injection) is feasible.

---

## 2. Access Address

### 2.1 Advertising Access Address

The BLE advertising access address is **fixed** at `0x8E89BED6` for all three
advertising channels. Every advertising packet on air begins with this 4-byte
value (after the 1-byte preamble).

### 2.2 nRF24 SPI Byte Order (CRITICAL TRAP)

The nRF24L01+ uses **LSByte-first** SPI write order for multi-byte address
registers (datasheet §8.3.1). This means the BLE access address must be written
in **reversed byte order** compared to the human-readable form.

**Transformation chain for the advertising access address:**

```
BLE Access Address: 0x8E89BED6

Step 1: BLE on-air (LSByte-first):   D6 BE 89 8E
Step 2: Per-byte bit reversal (swapbits):  6B 7D 91 71  (MSByte-first on air)
Step 3: nRF24 SPI write (LSByte-first):      71 91 7D 6B  (FIRST byte = LSByte)
```

**Correct constant (LSByte-first SPI order):**

```cpp
inline constexpr uint8_t ADV_ACCESS_ADDR[4] = {0x71, 0x91, 0x7D, 0x6B};
```

**Why the double reversal:**
1. BLE transmits LSbit-first per byte → bits within each byte are reversed
   relative to the nRF24's MSbit-first convention.
2. nRF24 multi-byte address registers are written LSByte-first in SPI order.
3. These two inversions compose, and getting only one right produces a
   plausible-looking but wrong value.

**Self-consistent error detection:** Writing the wrong byte sequence and reading
the same register back returns the same wrong sequence — comparison always passes.
Always derive expected values independently from `0x8E89BED6`.

### 2.3 Data Channel Access Address

Data channel access addresses are connection-specific (assigned during
`CONNECT_IND`). The nRF24L01+ cannot track data channel connections. This is
listed only for completeness — do NOT attempt to sniff data channels.

---

## 3. PDU Types and Header Format

### 3.1 Advertising PDU Types

| PDU Type | Value (bits [3:0]) | Name | Description |
|----------|--------------------|------|-------------|
| 0x0 | ADV_IND | Connectable undirected advertising |
| 0x1 | ADV_DIRECT_IND | Connectable directed advertising |
| 0x2 | ADV_NONCONN_IND | Non-connectable advertising |
| 0x3 | SCAN_REQ | Scan request |
| 0x4 | SCAN_RSP | Scan response |
| 0x5 | CONNECT_IND | Connection indication |
| 0x6 | ADV_EXT_IND | Extended advertising (BLE 5.0+) |

The PDU type field is 4 bits (bits [3:0] of the first header byte).

### 3.2 PDU Header Format (2 bytes)

```
Byte 0:                              Byte 1:
┌───┬───┬───┬───┬───┬───┬───┬───┐    ┌───┬───┬───┬───┬───┬───┬───┬───┐
│  PDU Type  │RFU│ChS│TxA│RxA│    │          Length           │RFU│
│   [3:0]    │[4]│[5]│[6]│[7]│    │          [7:0]             │   │
└───┴───┴───┴───┴───┴───┴───┴───┘    └───┴───┴───┴───┴───┴───┴───┴───┘
```

| Field | Bits | Description |
|-------|------|-------------|
| PDU Type | [3:0] | See table above |
| RFU | [4] | Reserved for future use, must be 0 |
| ChSel | [5] | Channel selection algorithm (0 or 1) |
| TxAdd | [6] | 1 = advertiser address is random; 0 = public |
| RxAdd | [7] | 1 = target address is random; 0 = public |
| Length | [15:8] | Payload length in bytes (6–37 for advertising) |

**Note on bit order:** BLE transmits LSbit-first within each byte. The PDU type
bits [3:0] are transmitted first (lowest bit first). When reading from the nRF24
RX FIFO (MSbit-first bytes), the header bytes must be bit-reversed before
interpreting the fields.

### 3.3 ADV_NONCONN_IND PDU Structure (Most Common for Beacons)

```
┌──────────────┬────────────────┬─────────────────────────┐
│ Header (2B)  │ AdvA (6 bytes) │ AdvData (0–31 bytes)     │
└──────────────┴────────────────┴─────────────────────────┘
```

- **Header byte 0**: PDU type = 0x2, TxAdd = 1 (random address) → `0x42` in
  LSbit-first BLE order, which becomes `0x42` after swapbits.
- **AdvA**: 6-byte advertiser address, LSByte-first on air.
- **AdvData**: Length-Type-Value (LTV) entries, max 31 bytes.

### 3.4 Example PDU Header Encoding

For `ADV_NONCONN_IND` with a random address:

```
BLE on-air (LSbit-first):
  Byte 0: bits [7:0] = RxAdd=0, TxAdd=1, ChSel=0, RFU=0, Type=0010
         = 0b01000010 = 0x42

After swapbits for nRF24 (MSbit-first):
  swapbits(0x42) = 0x42  (bit-palindromic value)
```

Always verify PDU header encoding with both the BLE spec bit order AND the
nRF24 swapbits transformation.

---

## 4. Data Whitening

### 4.1 Overview

BLE applies **data whitening** to every on-air packet (PDU + CRC) before
transmission. Whitening XORs the payload with a pseudo-random bit sequence
generated by a **7-bit Galois LFSR**. The receiver applies the identical sequence
to recover the original data.

**Purpose:** reduce DC bias, improve clock recovery, spread spectral energy.

**Polynomial:** x^7 + x^4 + 1

This defines the LFSR feedback taps. In the 7-bit Galois LFSR state, when the
shift-out bit is 1, XOR the state with bits at positions 4 and 0 before the next
shift. The encoded tap mask is `0x11` (bit 4 = x^4, bit 0 = 1).

### 4.2 Seed Derivation from Channel Index

The Bluetooth Core Spec (Vol 6 Part B §3.2) states the 7-bit LFSR seed is
"the value of the channel index plus 64." In the spec's LSB-first bit ordering,
position 1 is always 1, and positions 6:1 carry the 6-bit channel index value.

For the nRF24L01+, which delivers bytes MSbit-first, the seed formula is:

```
seed = swapbits(channel_idx) | 0x02
```

- `swapbits(channel_idx)` — bit-reverses the channel index, placing its bits
  in the MSbit-first positions of the LFSR.
- `| 0x02` — sets bit 1, which is position 1 in the Galois register (the
  constant-1 part of the spec's "channel index + 64" formula).

**Seed values for advertising channels:**

| Channel | `channel_idx` (hex) | `swapbits(ch)` | Seed (`swapbits|2`) | Type | Freq (MHz) |
|---------|---------------------|-----------------|----------------------|------|------------|
| 37 | 0x25 | 0xA4 | 0xA6 | Adv | 2402 |
| 38 | 0x26 | 0x64 | 0x66 | Adv | 2426 |
| 39 | 0x27 | 0xE4 | 0xE6 | Adv | 2480 |

### 4.3 Galois LFSR Algorithm (Correct Implementation)

```cpp
void dewhiten(uint8_t *data, uint8_t len, uint8_t channel_idx)
{
    // Step 1: bit-swap each byte (nRF24 MSbit-first → BLE LSbit-first)
    for (uint8_t i = 0; i < len; i++)
        data[i] = nrf24::ble::swapbits(data[i]);

    // Step 2: Galois LFSR dewhitening
    uint8_t lfsr = nrf24::ble::swapbits(channel_idx) | 0x02;
    for (uint8_t i = 0; i < len; i++) {
        for (uint8_t m = 1; m; m <<= 1) {
            if (lfsr & 0x80) {
                lfsr ^= 0x11;      // Galois feedback: XOR polynomial taps
                data[i] ^= m;       // XOR whitening bit into data
            }
            lfsr <<= 1;             // left-shift (Galois form)
        }
    }
}
```

**Critical implementation notes:**

1. **Galois form, NOT Fibonacci.** The same polynomial produces different
   sequences in Galois vs Fibonacci form. BLE uses Galois (left-shifting).
2. **Bit-swap comes FIRST.** The nRF24 delivers MSbit-first bytes; BLE
   processes LSbit-first. The swap must happen before LFSR dewhitening.
3. **XOR is symmetric.** The same function whitens and dewhitens.
4. **The seed formula `swapbits(ch) | 2` is NOT the same as `(ch & 0x3F) | 0x40`.**
   These produce different seeds (except coincidentally for channel 38).

### 4.4 Common Whitening Bugs

| Bug | Symptom | Root Cause |
|-----|---------|-----------|
| Missing swapbits before dewhitening | Every byte wrong (except bit-palindromic values like 0x00, 0xFF, 0x42) | nRF24 delivers MSbit-first; BLE expects LSbit-first |
| Fibonacci LFSR instead of Galois | Every byte wrong, different XOR masks | Same polynomial, different feedback topology |
| Seed formula `(ch & 0x3F) | 0x40` instead of `swapbits(ch) | 2` | Wrong LFSR initial state for all channels except 38 (coincidence) | Wrong interpretation of spec's "channel index + 64" |
| Wrong feedback XOR value (e.g. `0x48` instead of `0x11`) | Every byte wrong | Fibonacci feedback mask ≠ Galois feedback mask |

### 4.5 Verification

Run `docs/pipeline/scripts/ble_whiten_reference.py` to generate known vectors.
Compare byte-by-byte against `nrf24::ble::dewhiten()` output. Must be 100% match.

Cross-validate on hardware with Ubertooth One or nRF52840 Dongle capturing the
same BLE advertisements.

---

## 5. Bit Order Conventions

### 5.1 The Three Representations

BLE and nRF24 use different bit order conventions. Every byte in a BLE packet
goes through these transformations:

| Representation | Bit Order | Where Used |
|----------------|-----------|-----------|
| BLE on-air | LSbit-first per byte, LSByte-first per word | Bluetooth Core Spec |
| nRF24 SPI | MSbit-first per byte, LSByte-first for address registers | nRF24 hardware |
| Host (C/C++) | MSbit-first (= natural binary) | Application code |

### 5.2 swapbits Utility

The `swapbits()` function reverses all 8 bits of a byte. It is the bridge between
BLE LSbit-first and nRF24/host MSbit-first:

```cpp
inline uint8_t swapbits(uint8_t v)
{
    uint8_t r = 0;
    if (v & 0x80) r |= 0x01;  // bit 7 → bit 0
    if (v & 0x40) r |= 0x02;  // bit 6 → bit 1
    if (v & 0x20) r |= 0x04;  // bit 5 → bit 2
    if (v & 0x10) r |= 0x08;  // bit 4 → bit 3
    if (v & 0x08) r |= 0x10;  // bit 3 → bit 4
    if (v & 0x04) r |= 0x20;  // bit 2 → bit 5
    if (v & 0x02) r |= 0x40;  // bit 1 → bit 6
    if (v & 0x01) r |= 0x80;  // bit 0 → bit 7
    return r;
}
```

Uses the if-cascade form from Dmitry Grinberg's reference — most portable
across embedded toolchains, no table lookups, no architecture-specific intrinsics.

### 5.3 Where swapbits is Required

| Context | Before | After | Why |
|---------|--------|-------|-----|
| RX: bytes from nRF24 FIFO | MSbit-first | LSbit-first | BLE protocol processing |
| RX: before dewhitening | MSbit-first | LSbit-first | LFSR operates in BLE bit order |
| TX: before loading into TX FIFO | LSbit-first | MSbit-first | nRF24 expects MSbit-first |
| Access address (multi-byte) | MSByte-first | LSByte-first | nRF24 address registers are LSByte-first in SPI |
| CRC bytes (TX encoding) | LSbit-first (computed) | MSbit-first | nRF24 transmits MSbit-first |

### 5.4 Common Bit Order Bugs

| Bug | Symptom | Root Cause |
|-----|---------|-----------|
| Not swapping RX bytes before dewhitening | All bytes wrong except bit-palindromes | Assuming nRF24 and BLE have "same bit order" |
| Using `(ch & 0x3F) | 0x40` for seed | Wrong dewhitening on all channels except 38 | Literal interpretation of spec without bit-order conversion |
| MSByte-first access address in nRF24 | RPD=1 but FIFO empty | nRF24 expects LSByte-first for address registers |

---

## 6. CRC-24

### 6.1 BLE CRC Parameters

| Parameter | Value |
|-----------|-------|
| Polynomial | x^24 + x^10 + x^9 + x^6 + x^4 + x^3 + x + 1 = `0x00065B` |
| Initial value (advertising) | `0x555555` |
| Initial value (data) | `0x555555` (reset for each connection) |
| CRC length | 3 bytes (24 bits) |

### 6.2 nRF24L01+ Cannot Verify BLE CRC

The nRF24L01+ uses its own CRC-1 or CRC-2 (1 or 2 bytes, polynomial `0x07` or
`0x8005`). It cannot compute or verify BLE CRC-24. When sniffing BLE:

1. **Disable nRF24 CRC** — set `EN_CRC=0` in CONFIG (after setting `EN_AA=0x00`).
2. **The 3 CRC bytes are received as part of the payload** — the nRF24 payload
   width must be set to capture them (32 bytes captures PDU header + AdvA + data + CRC).
3. **CRC-24 must be verified in software** if needed.

### 6.3 CRC-24 Computation (for reference)

```cpp
// BLE CRC-24 computation (for TX packet crafting)
uint32_t crc24(const uint8_t* data, size_t len, uint32_t init = 0x555555)
{
    uint32_t crc = init;
    for (size_t i = 0; i < len; i++) {
        uint8_t byte = data[i];
        for (int bit = 0; bit < 8; bit++) {
            uint32_t msb = (crc >> 23) & 1;
            crc <<= 1;
            if (msb ^ ((byte >> bit) & 1)) {
                crc ^= 0x00065B;
            }
        }
        crc &= 0xFFFFFF;
    }
    return crc;
}
```

**Note:** The CRC is computed in LSbit-first order over the PDU. The resulting
3 CRC bytes must be bit-swapped before transmission via nRF24 TX FIFO.

---

## 7. Preamble

### 7.1 BLE Preamble

The BLE preamble is a single byte: `0xAA` if the LSB of the access address is `0`
(advertising), or `0x55` if the LSB is `1` (data channels).

Since the advertising access address is `0x8E89BED6` (LSB of the first on-air byte
is `0xD6`, which has LSB `0`), the advertising preamble is `0xAA`.

The nRF24L01+ automatically prepends the preamble based on the first byte of the
TX address. No manual preamble insertion is needed.

---

## 8. Full RX and TX Pipelines

### 8.1 RX Pipeline (nRF24 Sniffer → BLE PDU)

```
nRF24 RX FIFO (32 bytes, MSbit-first)
    │
    ▼
swapbits(each byte)           ← MSbit-first → LSbit-first
    │
    ▼
dewhiten(swapped, len, ch)    ← Galois LFSR with seed = swapbits(ch)|2
    │
    ▼
PDU (header + AdvA + AdvData)  ← LSbit-first, unwhitened
    │
    ▼
Extract header fields          ← PDU type, length, TxAdd, RxAdd
    │
    ▼
Verify CRC-24 (optional)       ← Over PDU bytes, init=0x555555
```

### 8.2 TX Pipeline (BLE PDU → nRF24 TX FIFO)

```
Plaintext PDU (header + AdvA + AdvData)
    │
    ▼
Compute CRC-24                  ← Polynomial 0x00065B, init 0x555555
    │
    ▼
swapbits(CRC bytes)             ← CRC computed LSbit-first, sent MSbit-first
    │
    ▼
Whiten(PDU + CRC, ch)           ← Galois LFSR with seed = swapbits(ch)|2
    │
    ▼
swapbits(each byte)             ← LSbit-first → MSbit-first for nRF24
    │
    ▼
nRF24 TX FIFO (MSbit-first)
```

---

## 9. Self-Reflection Clause

After fixing any BLE protocol bug, agents MUST answer:

1. **Why was this bug missed?** What assumption about BLE bit order, LFSR form,
   or byte ordering led to the incorrect code?
2. **What procedural safeguard would have caught it?** What test, cross-validation,
   or reference implementation check should be added?
3. **Update the knowledge base** — add the lesson to this skill or the relevant
   learning doc in `docs/learning/`.

### Process for Updating This Skill

When a new BLE protocol issue is discovered:

1. Add it as a new subsection with:
   - A one-line **Rule** in bold
   - The protocol specification reference (Bluetooth Core Spec section)
   - Correct code or transformation example
   - Broken code example (commented out, marked `// BROKEN`)
   - Symptom pattern table
2. Commit with: `docs(skills): add BLE protocol rule — [rule name]`

---

## 10. References

- **Bluetooth Core Specification** Vol 6 Part B — Link Layer
  - §1.3.1: LSBit-first transmission per byte
  - §2.1: Advertising physical channel PDU
  - §3.2: Data whitening
  - Table 2.3: Channel index to frequency mapping
- **Bluetooth Core Specification** Vol 6 Part A — Physical Layer
  - Channel frequencies and modulation
- **nRF24L01+ Product Specification** v1.0
  - §7.3: Packet format "MSB to the left"
  - §8.3.1: LSByte-first multi-byte register write order
- **Dmitry Grinberg**, "Bit-banging Bluetooth Low Energy": http://dmitry.gr/?r=05.Projects&proj=11.%20Bluetooth%20LE%20fakery
- Learning doc: `docs/learning/ble-data-whitening-nrf24.md`
- Learning doc: `docs/learning/nrf24-ble-packet-crafting.md`
- Learning doc: `docs/learning/rca-adv-access-addr-byte-order.md`
- Library reference: `nrf24::ble::dewhiten()`, `nrf24::ble::swapbits()`
