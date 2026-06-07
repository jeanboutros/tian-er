---
name: silent-failure
description: "Error path hunting patterns for embedded systems. Triggered when reviewing error handling, when compliance-gate runs architecture review, or when code uses try/catch, returns error codes, or calls functions that can fail. Catches swallowed exceptions, unchecked returns, void functions that could fail, and missing error propagation."
---

# Silent Failure Detection

## Purpose

In embedded systems, silent failures are catastrophic. A swallowed exception, an unchecked return value, or an error that degrades functionality without reporting can leave the system in an undefined state with no indication of what went wrong. This skill defines systematic patterns for detecting and preventing silent failures.

## When to Trigger

- **Any agent reviewing error handling** — in code reviews, architecture reviews, or compliance gates.
- **When compliance-gate runs T-ARCH or T2 review** — error handling is a structural concern.
- **When code uses try/catch** — especially empty catch blocks.
- **When code calls a function that returns `bool`, `esp_err_t`, or an error code** — and the return value is not checked.
- **When a function's signature is `void` but could fail** — the failure is silently discarded.
- **When reviewing FreeRTOS task functions** — infinite loops, watchdog triggers, missing error handling.
- **When reviewing HAL layer code** — error propagation gaps between hardware and application.

---

## Pattern Catalog

### Pattern 1: Swallowed Catch Blocks

**What it looks like:**
```cpp
try {
    radio.initialize();
} catch (...) {
    // nothing — exception silently swallowed
}

try {
    auto result = spi_transfer(data);
} catch (const std::exception& e) {
    // logged but not propagated — caller doesn't know it failed
}
```

**Why it's dangerous:**
- The caller believes the operation succeeded when it didn't.
- Subsequent code operates on incorrect assumptions (radio is initialized, data was transferred).
- No recovery is possible because no one knows there's a problem.
- In embedded systems, this can leave hardware in an undefined state.

**The correct pattern:**
```cpp
try {
    radio.initialize();
} catch (const HalError& e) {
    ESP_LOGE(TAG, "Radio initialization failed: %s", e.what());
    // Option 1: Propagate
    throw;
    // Option 2: Set error state and return
    state_ = State::Error;
    return false;
    // Option 3: Attempt specific recovery
    radio.reset();
    return radio.initialize();  // retry once
}
```

**How to detect in review:**
- Grep for `catch` blocks where the body is empty or only contains a comment.
- Grep for `catch` blocks where the body only logs but doesn't propagate or set error state.
- Verify that every `catch` block either: (a) rethrows, (b) returns an error, or (c) sets an error state that callers can check.

---

### Pattern 2: Unchecked Return Values

**What it looks like:**
```cpp
// bool return not checked
radio.write_reg(cfg);  // returns bool — failure is silently ignored

// esp_err_t return not checked
spi_device_transmit(spi_handle, &transaction);  // returns esp_err_t — ignored

// Error code return not checked
auto result = esp_wifi_start();  // returns ESP_OK on success — ignored
```

**Why it's dangerous:**
- The operation may have failed but the code continues as if it succeeded.
- In embedded systems, hardware operations fail frequently (SPI timeout, device not responding, power issues).
- Subsequent operations may depend on the success of the unchecked call.

**The correct pattern:**
```cpp
bool ok = radio.write_reg(cfg);
if (!ok) {
    ESP_LOGE(TAG, "Failed to write config register");
    return false;  // propagate the failure
}

esp_err_t ret = spi_device_transmit(spi_handle, &transaction);
if (ret != ESP_OK) {
    ESP_LOGE(TAG, "SPI transfer failed: %s", esp_err_to_name(ret));
    return false;  // propagate the failure
}
```

**Special note for ESP-IDF:**
- `ESP_ERROR_CHECK()` is appropriate **only in init functions** where failure is fatal.
- In runtime paths, `ESP_ERROR_CHECK()` calls `abort()` on failure — this is a **silent failure from the user's perspective** (device reboots without explanation).
- Use explicit `if (ret != ESP_OK)` checks in runtime code.

**How to detect in review:**
- Grep for function calls that return `bool` or `esp_err_t` where the return value is not captured or not checked.
- Grep for `spi_device_transmit`, `gpio_set_level`, `spi_bus_initialize`, and other ESP-IDF functions that return error codes.
- Verify that every call to `Hal::spi_xfer()`, `Driver::write_reg()`, etc. checks the return value.

---

### Pattern 3: Void Functions That Could Fail

**What it looks like:**
```cpp
void radio_initialize();  // How do you know if it failed?
void configure_ble();     // Configuration can fail silently
void set_channel(int ch); // Invalid channel accepted without error
```

**Why it's dangerous:**
- Callers have no way to know if the operation succeeded.
- The function may set internal error state that no one checks.
- In embedded systems, initialization and configuration operations can fail (device not found, invalid parameters, hardware error).

**The correct pattern:**
```cpp
bool radio_initialize();       // returns true on success, false on failure
bool configure_ble(const BleConfig& cfg);  // returns false on invalid config
bool set_channel(Channel ch);  // typed enum prevents invalid values;
                                // returns false only on hardware failure
```

**Exception — truly infallible operations:**
```cpp
void set_ce_high();  // GPIO set — can't fail if GPIO is properly configured
void set_ce_low();   // GPIO clear — can't fail if GPIO is properly configured
```
These are acceptable as `void` because they cannot fail (assuming init succeeded). Document WHY they can't fail.

**How to detect in review:**
- List all `void` functions in the public API.
- For each, ask: "Can this operation fail?" If yes, change to `bool` or error code return.
- Look for functions that set internal error state but return `void` — callers can't check.

---

### Pattern 4: abort() in Library Code

**What it looks like:**
```cpp
void Hal::spi_xfer(...) {
    if (ret != ESP_OK) {
        abort();  // Library code calls abort — device reboots with no context
    }
}
```

**Why it's dangerous:**
- In a library, `abort()` is never appropriate for runtime failures.
- The calling application loses all context about what went wrong.
- On ESP32, `abort()` triggers a panic and reboot — the user sees "Guru Meditation Error" with no application-level context.

**When abort() IS acceptable:**
- In `app_main()` initialization for truly fatal setup failures (e.g. SPI bus initialization fails — the application literally cannot operate).
- Only when the alternative is operating in an undefined state.

**The correct pattern for library code:**
```cpp
bool Hal::spi_xfer(...) {
    esp_err_t ret = spi_device_transmit(spi_handle_, &transaction);
    if (ret != ESP_OK) {
        ESP_LOGE(TAG, "SPI transfer failed: %s", esp_err_to_name(ret));
        return false;  // let the caller decide what to do
    }
    return true;
}
```

**How to detect in review:**
- Grep for `abort()` calls in library code (anything under `components/`).
- Grep for `ESP_ERROR_CHECK()` in runtime paths (not init).
- Verify each `abort()` or `ESP_ERROR_CHECK()` is in `app_main()` init code only.

---

### Pattern 5: ESP_ERROR_CHECK in Runtime Paths

**What it looks like:**
```cpp
void process_packet() {
    ESP_ERROR_CHECK(spi_device_transmit(handle, &t));  // calls abort() on failure!
    // If this fails during packet processing, the entire device reboots
}
```

**Why it's dangerous:**
- `ESP_ERROR_CHECK()` is a macro that calls `abort()` on non-ESP_OK results.
- In init code, this is expected — if the SPI bus can't be set up, the device can't operate.
- In runtime code (packet processing, BLE reception, periodic tasks), this is a silent failure from the user's perspective — the device reboots with no application context.

**When ESP_ERROR_CHECK IS acceptable:**
- In `app_main()` before the main loop starts.
- In initialization functions that are called once at startup.
- When the alternative is continuing in a definitely-broken state.

**The correct pattern for runtime code:**
```cpp
void process_packet() {
    esp_err_t ret = spi_device_transmit(handle, &t);
    if (ret != ESP_OK) {
        ESP_LOGW(TAG, "SPI error in packet processing: %s", esp_err_to_name(ret));
        return;  // skip this packet, continue operating
    }
    // ... process the packet
}
```

**How to detect in review:**
- Grep for `ESP_ERROR_CHECK` in all `.cpp` files.
- For each occurrence, determine if it's in init code or runtime code.
- Flag any `ESP_ERROR_CHECK` in runtime code.

---

### Pattern 6: Silent Degradation

**What it looks like:**
```cpp
bool read_ble_packet(uint8_t* buf, size_t len) {
    if (len > MAX_PAYLOAD) {
        // silently truncate instead of reporting error
        len = MAX_PAYLOAD;
    }
    // ... read payload
}

void on_rx_packet() {
    if (fifo_status.rx_empty()) {
        return;  // silently drop — no log, no error state
    }
    // ... process packet
}
```

**Why it's dangerous:**
- The system appears to work but produces incorrect results.
- Truncation silently changes behaviour — the caller gets fewer bytes than expected.
- Dropping events silently makes debugging impossible — the user has no indication that events are being lost.

**The correct pattern:**
```cpp
bool read_ble_packet(uint8_t* buf, size_t len) {
    if (len > MAX_PAYLOAD) {
        ESP_LOGW(TAG, "Requested %zu bytes, max is %d — request truncated", len, MAX_PAYLOAD);
        return false;  // or: process MAX_PAYLOAD bytes and set a status flag
    }
    // ... read payload
}
```

**How to detect in review:**
- Look for silent truncation: `len = MAX_`, `size = min(size, ...)`.
- Look for early returns without logging or error state: `if (condition) return;` with no side effects.
- Functions that modify inputs silently instead of reporting errors.

---

### Pattern 7: FreeRTOS Task Functions Without Error Handling

**What it looks like:**
```cpp
void ble_sniffer_task(void* arg) {
    while (true) {
        // no error handling — if radio_read fails, task spins infinitely
        // no watchdog feed — may trigger watchdog reset
        // no delay — may starve other tasks
        auto data = radio.read_fifo();
        process(data);
    }
}
```

**Why it's dangerous:**
- If `radio.read_fifo()` enters an error state and returns empty data indefinitely, the task spins forever with no indication of failure.
- If the task loop has no `vTaskDelay()`, it can starve lower-priority tasks.
- The ESP32 task watchdog may reboot the device if the task doesn't feed the watchdog within the timeout period.
- A panic in a FreeRTOS task kills the entire device, not just the task.

**The correct pattern:**
```cpp
void ble_sniffer_task(void* arg) {
    constexpr int MAX_CONSECUTIVE_ERRORS = 10;
    int consecutive_errors = 0;
    
    while (true) {
        bool ok = radio.read_fifo(buffer, sizeof(buffer));
        if (!ok) {
            consecutive_errors++;
            ESP_LOGW(TAG, "Read failed (%d/%d)", consecutive_errors, MAX_CONSECUTIVE_ERRORS);
            if (consecutive_errors >= MAX_CONSECUTIVE_ERRORS) {
                ESP_LOGE(TAG, "Too many consecutive errors — reinitializing radio");
                radio.reinitialize();
                consecutive_errors = 0;
            }
            vTaskDelay(pdMS_TO_TICKS(10));
            continue;
        }
        consecutive_errors = 0;
        
        process(buffer);
        vTaskDelay(pdMS_TO_TICKS(1));  // yield to other tasks
    }
}
```

**How to detect in review:**
- Grep for `xTaskCreate`, `xTaskCreatePinnedToCore` — find all FreeRTOS task functions.
- For each task function, verify: (a) error handling for operations that can fail, (b) `vTaskDelay` or blocking call to prevent starvation, (c) consecutive error limiting, (d) no `abort()` in the task loop.
- Check that task stack depth is adequate for the function's local variables and call depth.

---

### Pattern 8: Missing Error Propagation in HAL Layers

**What it looks like:**
```cpp
class Hal {
public:
    virtual void spi_xfer(const uint8_t* tx, uint8_t* rx, size_t len) = 0;
    // void return — caller cannot know if transfer failed
    // This is the hardware abstraction — errors MUST be propagatable
};
```

**Why it's dangerous:**
- The HAL is the boundary between the library and the platform.
- If the HAL can't propagate errors, the library above it has no way to detect hardware failures.
- This is the most critical layer for error propagation because all hardware operations go through it.

**The correct pattern:**
```cpp
class Hal {
public:
    virtual bool spi_xfer(const uint8_t* tx, uint8_t* rx, size_t len) = 0;
    // Returns true on success, false on failure.
    // The caller (Driver) can decide what to do with the failure.
};

// Concrete implementation:
bool EspIdfHal::spi_xfer(const uint8_t* tx, uint8_t* rx, size_t len) {
    spi_transaction_t trans = {};
    trans.tx_buffer = tx;
    trans.rx_buffer = rx;
    trans.length = len * 8;
    
    esp_err_t ret = spi_device_transmit(spi_handle_, &trans);
    if (ret != ESP_OK) {
        ESP_LOGE(TAG, "SPI transfer failed: %s", esp_err_to_name(ret));
        return false;
    }
    return true;
}
```

**Error propagation chain:**
```
Hardware → HAL (bool) → Driver (bool) → Application (bool or error handling)
```
Every layer must propagate errors upward. No layer should swallow errors silently.

**How to detect in review:**
- Check the `Hal` abstract class: does every method that can fail return `bool` or an error code?
- Check the `EspIdfHal` implementation: does every ESP-IDF call check its return value?
- Check the `Driver` class: does every method propagate HAL errors upward?
- Check the application: does it handle `false` returns from the Driver?

---

## Review Checklist

When reviewing error handling, use this checklist:

```markdown
### Silent Failure Review Checklist

| # | Pattern | Checked? | Findings |
|---|---------|----------|----------|
| 1 | Swallowed catch blocks | yes/no | [list empty or log-only catch blocks] |
| 2 | Unchecked return values | yes/no | [list calls where bool/esp_err_t is ignored] |
| 3 | Void functions that could fail | yes/no | [list void functions that should return bool] |
| 4 | abort() in library code | yes/no | [list abort() calls outside app_main init] |
| 5 | ESP_ERROR_CHECK in runtime paths | yes/no | [list ESP_ERROR_CHECK outside init] |
| 6 | Silent degradation | yes/no | [list truncations/drops without logging] |
| 7 | FreeRTOS task error handling | yes/no | [list tasks without error handling, delays, limits] |
| 8 | HAL error propagation gaps | yes/no | [list HAL methods that suppress errors] |
```

For each finding, assign a confidence score using the review-confidence skill:
- **90-100 (Critical):** Will definitely cause silent failure in production
- **80-89 (High):** Very likely to cause silent failure under error conditions
- **70-79 (Moderate):** Could cause issues but might be rare or have mitigations
- **0-69 (Low):** Theoretical concern, unlikely in practice

---

## Integration with Compliance Gates

### T-ARCH (Architecture + Principles Review)

Silent failure detection is part of T-ARCH.3 (Principle Alignment) and T-ARCH.1 (Logical Consistency):

- **T-ARCH.1:** A system that silently swallows errors is logically inconsistent — the caller believes an operation succeeded when it didn't.
- **T-ARCH.3:** The project principle is that error paths must be explicit and propagated. Silent failure violates this principle.

### T2.4 (API Surface Audit)

A void function that could fail is an API design violation — the API surface should expose failure modes to callers.

### T3.3 (Security Review)

- Swallowed exceptions can hide security-relevant errors.
- Unchecked return values from SPI operations can allow injected data to bypass validation.
- FreeRTOS tasks without error handling can be exploited for denial of service.

---

## Self-Reflection Clause

After finding a silent failure pattern in review, ask:

1. **Why was this not caught during implementation?** — What review, test, or protocol gap allowed the silent failure to be written?
2. **What procedural safeguard would have caught it?** — What specific check or test would have prevented it?
   - Static analysis: Can we add a lint rule for unchecked return values?
   - Code review: Can we add "check all return values" as a mandatory review item?
   - Testing: Can we add a test that injects HAL failures and verifies error propagation?
3. **Update the knowledge base** — Add the lesson to `docs/learning/silent-failure-patterns.md` or the relevant learning doc.

### Common Root Causes

| Root Cause | Safeguard |
|-----------|-----------|
| Developer assumed operation always succeeds | Add error path unit tests that inject failures |
| Empty catch block during development left in | Grep for empty catch blocks in T1 checks |
| `void` return by convention | Review all `void` functions for "can this fail?" |
| `ESP_ERROR_CHECK` in runtime by habit | Add lint rule: no `ESP_ERROR_CHECK` outside init functions |
| Hal method returns void | Design rule: all Hal methods that can fail must return bool or error code |
| Silent truncation for "convenience" | Design rule: never silently modify inputs; return error or log warning |
