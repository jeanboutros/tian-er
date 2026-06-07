---
name: tdd-cpp
description: "TDD patterns specific to C++: static_assert for compile-time tests, register struct round-trip tests, protocol logic tests, and edge case testing for embedded C++ APIs. Load alongside test-driven-development when working in C++."
---

# Test-Driven Development — C++ Patterns

## Overview

Write the test first. Watch it fail. Write minimal code to pass.

Core principle: **If you didn't watch the test fail, you don't know if it tests the right thing.**

## When to Use

- New register struct implementation
- New protocol function (whitening, channel mapping)
- Bug fixes (write the test that would have caught it)
- Refactoring (tests prove no regression)

## The TDD Cycle

```
RED ──→ GREEN ──→ REFACTOR
 ↑                    │
 └────────────────────┘
```

### RED: Write the Failing Test

```cpp
// test_registers.cpp — write this FIRST
#include "nrf24l01plus/registers/rf_setup.h"

// Test: RfSetup encodes 1Mbps correctly
static_assert(nrf24::RfSetup{.data_rate = nrf24::DataRate::Mbps1}.to_byte() == 0x06);

// Test: Round-trip preserves all fields
static_assert(nrf24::RfSetup::from_byte(0x26).data_rate == nrf24::DataRate::Kbps250);
```

Build → should fail (struct doesn't exist yet or encoding is wrong).

### GREEN: Implement Minimal Code

Write just enough code to make the test pass:
- Implement the struct with correct bit positions
- Implement `to_byte()` and `from_byte()`
- Build → all static_asserts pass

### REFACTOR: Clean Up

- Add Doxygen documentation
- Ensure naming matches datasheet
- Add edge case tests (0xFF, 0x00, reserved bits)

## Test Categories for This Project

### 1. Register Round-Trip Tests
```cpp
// Every register struct needs these:
static_assert(T::from_byte(T{...}.to_byte()) == expected);
static_assert(T::from_byte(RESET_VALUE).field == expected_default);
```

### 2. Encoding Tests
```cpp
// Verify specific encodings match datasheet
static_assert(nrf24::RfSetup{.data_rate = nrf24::DataRate::Mbps2}.to_byte() & 0x08);
```

### 3. Edge Case Tests
```cpp
// 0xFF input — reserved bits must be masked
static_assert(nrf24::RfCh::from_byte(0xFF).channel == 127); // bit 7 is reserved

// 0x00 input — all fields at minimum
static_assert(nrf24::Config::from_byte(0x00).power_mode == nrf24::PowerMode::Down);
```

### 4. Protocol Logic Tests
```cpp
// Channel mapping
static_assert(nrf24::ble::channel_to_rf_ch(37) == 2);
static_assert(nrf24::ble::channel_to_rf_ch(0) == 4);
static_assert(nrf24::ble::channel_to_rf_ch(39) == 80);
```

## Rules

1. **Test file must exist before implementation** — even if empty
2. **Build must fail before implementation** — proves the test is real
3. **Build must pass after implementation** — proves correctness
4. **Never delete a test to make code pass** — fix the code instead
5. **Edge cases are not optional** — 0xFF and reserved bits always tested
