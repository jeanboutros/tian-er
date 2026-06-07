---
name: cpp-embedded
description: "C++ patterns and rules for embedded hardware register libraries. Triggered when writing or reviewing C++ code for any embedded project — enum class mandates, Doxygen rules, register struct design, platform independence, HAL interfaces, and vocabulary dogfooding. Always loaded for code-architect, software-engineer, hardware-engineer, and test-engineer agents when the tech stack includes C++."
---

# C++ Embedded Library Design Principles

## Purpose

This skill defines the mandatory C++ coding standards for modelling hardware registers,
peripherals, and protocol fields in reusable embedded libraries. These rules ensure
that the library API is **self-documenting, type-safe, and platform-independent**.

This skill is **complementary** to `datasheet-verification` (which governs where
register values come from) and `nrf24l01plus` (which captures chip-specific traps).
It focuses on **how C++ types encode hardware knowledge** once the datasheet has been
verified.

---

## 1. Doxygen Documentation Rules

### 1.1 Mandatory Structure

Every new **function**, **struct**, **class**, **enum**, or **macro group** must be
preceded by a Doxygen `/** ... */` block with this structure:

```cpp
/**
 * @brief One-sentence summary of what this does
 *
 * Longer explanation if needed — describe the protocol, bit layout,
 * algorithm, or non-obvious design decision here.
 *
 * Use @code / @endcode blocks for:
 *   - Bit/byte layout diagrams
 *   - Arithmetic walkthroughs
 *   - Usage examples
 *
 * @code
 *   // Example: show inputs, intermediate steps, and result
 *   foo(REG_STATUS);   // 0x07 & 0x1F = 0x07 → cmd = 0x07
 * @endcode
 *
 * @param name   Description (include units, valid range, or bit meaning)
 * @return       What is returned and under what conditions
 */
```

### 1.2 Rules

| Element | Rule |
|---------|------|
| `@brief` | Always present; one sentence, no period at end |
| `@code` | Use for any non-obvious bit manipulation, protocol encoding, or illustrative call site. Include an edge-case example (e.g. `0xFF` input) when the masking/clamping behaviour matters |
| `@param` | One line per parameter; note which bits are used if the full width is not consumed |
| `@return` | Always present for non-void functions |
| Inline comments | `/* short inline notes */` inside function bodies — do NOT duplicate the Doxygen block content |

### 1.3 Examples in @code Blocks Must Use Typed Vocabulary

```cpp
// BAD — raw hex in an @code block, even though nrf24::reg::CONFIG exists
radio.write_reg(0x00, 0x03);

// GOOD — uses the library's own vocabulary
radio.write_reg(nrf24::Config{...});
```

All `@code` examples must use library typed constants and struct forms. Raw literals
are acceptable only in comments marked `// internal use`.

---

## 2. Typed Enum Mandate

### 2.1 Every Register Field → `enum class`

Every register field that has a **finite set of legal values** must be an `enum class`,
not an `int` or `uint8_t` literal.

```cpp
// BAD — plain enum, names leak, implicit conversion to int
enum DataRate { Mbps1 = 0, Kbps250 = 1 };

// GOOD — enum class, no leakage, explicit underlying type
enum class DataRate : uint8_t {
    Kbps250 = 0b00,  // RF_DR_HIGH=0, RF_DR_LOW=1
    Mbps1    = 0b01,  // RF_DR_HIGH=0, RF_DR_LOW=0
    Mbps2    = 0b10,  // RF_DR_HIGH=1, RF_DR_LOW=0
    // 0b11 is reserved per datasheet — no enumerator for it
};
```

### 2.2 Enum Names from Datasheet

Enumerator names must come **verbatim from the datasheet symbol names** wherever
possible (e.g. `RF_DR_LOW`, `RF_PWR`). Use CamelCase only when the datasheet uses
all-caps abbreviations that would be unreadable as-is.

### 2.3 No Catch-All or Raw Value Enumerators

```cpp
// BAD — catch-all enumerator opens the type to invalid values
enum class TxPower : uint8_t {
    Minus18dBm = 0b00,
    Minus12dBm = 0b01,
    Minus6dBm  = 0b10,
    dBm0       = 0b11,
    Raw        = 0xFF    // ← NEVER DO THIS
};

// GOOD — no catch-all; use from_byte() for raw access
enum class TxPower : uint8_t {
    Minus18dBm = 0b00,
    Minus12dBm = 0b01,
    Minus6dBm  = 0b10,
    dBm0       = 0b11,
};
// Raw access through struct method, not enum value:
// TxPower p = TxPower::from_byte(raw_byte);  // validates and returns
```

### 2.4 Method Parameters Must Use Typed Enums

Every **method parameter** that accepts one of a finite set of values must use a
typed enum or a struct with `ADDRESS` constant, not `uint8_t`.

```cpp
// BAD — uint8_t accepts any value, including 0xFF (invalid register address)
void write_reg(uint8_t addr, uint8_t val);

// GOOD — typed struct deduces address, public API is type-safe
void write_reg(const Config& cfg);  // deduces Config::ADDRESS
```

If a named-constant vocabulary exists for a parameter (e.g. `nrf24::reg::CONFIG`),
the parameter type must enforce it at compile time. `constexpr uint8_t` namespace
constants are **documentation aids, not type safety** — a `uint8_t` parameter that
accepts `nrf24::reg::CONFIG` also accepts `0xFF`. The type system must reject invalid
values at compile time.

Convenience `uint8_t` overloads must be `private` or `protected` — never `public`.

---

## 3. Library API Design — Give Users a Vocabulary

### 3.1 One Enum Class Per Field

```cpp
enum class CrcMode : uint8_t { Disabled = 0b00, Crc1Byte = 0b01, Crc2Byte = 0b10 };
enum class PowerMode : uint8_t { Down = 0, Up = 1 };
enum class PrimaryMode : uint8_t { TX = 0, RX = 1 };
```

### 3.2 One Aggregate Struct Per Register

```cpp
struct Config {
    static constexpr uint8_t ADDRESS = 0x00;  // register address

    PowerMode power_mode = PowerMode::Down;
    PrimaryMode primary   = PrimaryMode::RX;
    CrcMode crc_mode      = CrcMode::Disabled;

    uint8_t to_byte() const;
    static Config from_byte(uint8_t raw);
    std::string format() const;
};
```

Every struct MUST have:
- `ADDRESS` — `static constexpr uint8_t` with the register address
- `to_byte()` — serialise fields to a raw byte
- `from_byte()` — deserialise from a raw byte
- `format()` — human-readable string for diagnostics

### 3.3 Typed Overloads on the Driver Class

```cpp
// Public — typed API, deduces address from struct type
template<typename Reg>
Reg read_reg();                    // returns Reg, deduces ADDRESS

template<typename Reg>
void write_reg(const Reg& reg);    // deduces ADDRESS from Reg::ADDRESS

// Private — raw API, only accessible internally
bool write_reg(uint8_t addr, uint8_t val);
uint8_t read_reg(uint8_t addr);
```

The typed API must be the **ONLY public API**. If both raw and typed overloads
exist, the raw overloads must be `private` or `protected`.

### 3.4 Free Functions for Individual Fields

```cpp
// to_reg() overloads — compose individual fields without a full struct
constexpr uint8_t to_reg(DataRate dr);
constexpr uint8_t to_reg(TxPower pwr);
```

### 3.5 Detail Sub-Namespace

Keep internal helpers (bit-splitting, encoding tables, mask constants) in a
`detail` sub-namespace so they do not pollute the public API:

```cpp
namespace nrf24::detail {
    constexpr uint8_t RF_DR_LOW_BIT = 5;
    constexpr uint8_t RF_DR_HIGH_BIT = 3;
    // ...
}
```

### 3.6 Headers Are a Library Contract

Headers must contain **only** Doxygen `/** */` blocks and code declarations. No
learning notes, no TODO comments, no design rationale. That material belongs in
`docs/learning/`.

---

## 4. Completeness — Model the Full Register

### 4.1 Every Bit Accounted For

Every bit in a register must be accounted for, even reserved bits.

```cpp
struct RfSetup {
    // Bits [7:6] — reserved, must write 0
    // Bit 5 — RF_DR_LOW (see DataRate)
    // Bit 4 — PLL_LOCK (read-only)
    // Bit 3 — RF_DR_HIGH (see DataRate)
    // Bits [2:1] — RF_PWR (see TxPower)
    // Bit 0 — Obsolete, write 0 (nRF24L01+ only; was RSSI in nRF24L01)

    DataRate data_rate = DataRate::Mbps1;
    TxPower  tx_power  = TxPower::Minus18dBm;

    uint8_t to_byte() const {
        // reserved bits [7:6] = 0, obsolete bit 0 = 0
        return detail::encode_rate(data_rate) | detail::encode_power(tx_power);
    }
};
```

Reserved bits are **never silently ignored** in `to_byte()` or `from_byte()`.
The `to_byte()` implementation must explicitly mask/write reserved bits to their
required values (typically 0).

### 4.2 Non-Contiguous Fields

If a multi-bit field has a non-contiguous encoding (e.g. nRF24 `DataRate` spans
bits 5 and 3), the encoding logic must follow the datasheet's table exactly.
Use a helper in `detail::` rather than inventing a compact representation.

```cpp
namespace nrf24::detail {
    constexpr uint8_t encode_rate(DataRate dr) {
        switch (dr) {
            case DataRate::Kbps250: return 0x20;  // RF_DR_LOW=1, RF_DR_HIGH=0
            case DataRate::Mbps1:   return 0x00;  // RF_DR_LOW=0, RF_DR_HIGH=0
            case DataRate::Mbps2:   return 0x08;  // RF_DR_LOW=0, RF_DR_HIGH=1
            default:                return 0x00;
        }
    }
}
```

---

## 5. Platform Independence

### 5.1 No Platform Headers in Library Public API

A reusable library must not `#include` platform-specific headers in any public
header under `include/`.

**Allowed in library headers:**
```cpp
#include <cstdint>    // standard C++
#include <cstring>    // standard C++
#include <cstdio>     // standard C++
#include "nrf24l01plus/registers/config.h"  // own headers
```

**Forbidden in library headers:**
```cpp
#include "driver/spi_master.h"   // ESP-IDF
#include "driver/gpio.h"         // ESP-IDF
#include "Arduino.h"             // Arduino
#include "stm32f4xx_hal.h"       // STM32 HAL
```

### 5.2 HAL Abstract Interface

Define a pure-virtual `Hal` class in the library. Platform-specific implementations
live **outside** the library.

```cpp
// In the library (components/nrf24l01plus/include/nrf24l01plus/hal.h)
namespace nrf24 {

class Hal {
public:
    virtual ~Hal() = default;

    /** Transfer bytes over SPI. Returns false on error. */
    virtual bool spi_xfer(const uint8_t* tx, uint8_t* rx, size_t len) = 0;

    /** Set CE pin HIGH. */
    virtual void ce_high() = 0;

    /** Set CE pin LOW. */
    virtual void ce_low() = 0;
};

} // namespace nrf24
```

```cpp
// In the platform adapter (main/esp_idf_hal.h)
#include "nrf24l01plus/hal.h"
#include "driver/spi_master.h"
#include "driver/gpio.h"

class EspIdfHal : public nrf24::Hal {
    // ESP-IDF specific implementation
};
```

### 5.3 Library Source Files

Library source files (`.cpp` under `components/nrf24l01plus/`) must only include
library headers and standard C/C++ headers. No platform headers allowed.

---

## 6. Dogfood Your Own Vocabulary

### 6.1 All Documentation Uses Typed Constants

When the library defines named constants or typed enums, all documentation,
`@code` examples, and internal usage must use them:

```cpp
// BAD — raw hex, user must look up what 0x00 and 0x03 mean
radio.write_reg(0x00, 0x03);

// GOOD — self-documenting, impossible to put a field in the wrong position
nrf24::Config cfg;
cfg.power_mode = nrf24::PowerMode::Up;
cfg.primary    = nrf24::PrimaryMode::RX;
cfg.crc_mode   = nrf24::CrcMode::Disabled;
radio.write_reg(cfg);
```

### 6.2 @param Docs Reference the Header

```cpp
/**
 * @param addr  Register address. Use constants from nrf24l01plus/registers/addresses.h
 */
```

### 6.3 Learning Docs Use Typed Vocabulary

Learning docs in `docs/learning/` must use the library's typed vocabulary in all
`@code` blocks and inline code. If a doc needs to show the low-level raw API for
educational purposes, it must prominently note:

> *"This shows the low-level SPI API for educational purposes. Production code
> should use the typed struct API — see [cpp-enum-class-and-struct.md]."*

---

## 7. Self-Reflection Clause

After fixing any C++ API design bug (wrong enum value, missing typed overload,
raw integer leak, reserved bit mishandling), agents MUST answer:

1. **Why was this bug missed?** What review gap or design principle was violated?
2. **What procedural safeguard would have caught it?** What specific check should
   be added to the self-audit checklist or T1/T2 gate?
3. **Update the knowledge base** — add the lesson to this skill or the relevant
   learning doc in `docs/learning/`.

### Process for Updating This Skill

When a new C++ pattern issue is discovered:

1. Add it as a new subsection with:
   - A one-line **Rule** in bold
   - Correct code example
   - Broken code example (commented out, marked `// BAD`)
   - Explanation of why the correct form matters
2. Commit with: `docs(skills): add C++ embedded rule — [rule name]`

---

## 8. References

- Learning doc: `docs/learning/cpp-enum-class-and-struct.md`
- ESP-IDF C++ style: https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-guides/cplusplus.html
- C++ `enum class`: https://en.cppreference.com/w/cpp/language/enum
- Project register design: `components/nrf24l01plus/include/nrf24l01plus/registers/`
- Project HAL interface: `components/nrf24l01plus/include/nrf24l01plus/hal.h`
