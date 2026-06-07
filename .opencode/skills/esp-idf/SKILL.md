---
name: esp-idf
description: "ESP-IDF framework specifics, FreeRTOS, SPI bus, GPIO, build system, and ESP32 hardware traps. Triggered when working with ESP-IDF APIs, spi_bus_initialize, GPIO modes, xTaskCreatePinnedToCore, idf.py build/flash/monitor, or any ESP32 platform interaction."
---

# ESP-IDF Framework and ESP32 Hardware Knowledge

## Purpose

This skill consolidates ESP-IDF-specific knowledge that agents need when working with
the ESP32 platform. It covers build system commands, FreeRTOS dual-core usage, GPIO
caveats, SPI bus management, error handling patterns, RAII resource cleanup, serial
port setup, and VS Code diagnostics.

This skill is **complementary** to `nrf24l01plus` (chip-specific traps) and
`ble-protocol` (BLE protocol details). It focuses on **ESP-IDF platform behaviours
and APIs** that are easy to get wrong.

---

## 1. ESP-IDF Environment and Build System

### 1.1 Environment Activation

**ALWAYS** activate the ESP-IDF environment before running any `idf.py` command:

```bash
source ~/.espressif/tools/activate_idf_v6.0.1.sh
```

Without this, `idf.py` will not be found and the toolchain will not be in PATH.

### 1.2 Build Command

```bash
source ~/.espressif/tools/activate_idf_v6.0.1.sh && idf.py build
```

Must exit 0 with **zero warnings** (`-Werror` is active) before any commit.

### 1.3 Flash and Monitor

```bash
source ~/.espressif/tools/activate_idf_v6.0.1.sh
idf.py -p /dev/ttyUSB0 flash monitor
```

### 1.4 T1 Mechanical Compliance Check

```bash
bash docs/pipeline/scripts/t1-check.sh
```

Run between PAU units and at compliance gates.

### 1.5 CMake and Component Dependencies

ESP-IDF uses CMake with `idf_component_register()`:

```cmake
idf_component_register(
    SRCS "main.cpp"
    INCLUDE_DIRS "."
    REQUIRES nrf24l01plus    # library component dependency
)
```

**Rules:**
- Library components (`components/nrf24l01plus/`) must NOT depend on platform components (`main/`, driver components).
- The dependency graph must be unidirectional: `main` → `nrf24l01plus` (never the reverse).
- Library `CMakeLists.txt` uses `REQUIRES` only for other library components.
- Platform adapter code in `main/` handles all ESP-IDF includes and `REQUIRES`.

---

## 2. Serial Port and Monitor

### 2.1 Exiting Monitor Mode

| Shortcut | Action |
|----------|--------|
| `Ctrl`+`]` | Exit monitor (standard) |
| `Ctrl`+`T` then `X` | Alternative exit |
| `Ctrl`+`A` then `K` | If using `screen` backend |

> **Note:** `Ctrl+C` does **NOT** exit ESP-IDF monitor — it only interrupts the running app.

### 2.2 Finding the Serial Port

```bash
ls /dev/ttyUSB* /dev/ttyACM* 2>/dev/null
lsusb    # Look for "Silicon Labs CP210x", "QinHeng CH340", "FTDI"
sudo dmesg -w    # Watch kernel messages when plugging/unplugging
```

Before/after comparison method:

```bash
ls /dev/tty* > /tmp/before.txt
# Plug in ESP32
ls /dev/tty* > /tmp/after.txt
diff /tmp/before.txt /tmp/after.txt
```

### 2.3 Fixing Permission Issues

```bash
# Option 1: Add user to dialout group (permanent, requires logout)
sudo usermod -a -G dialout $USER

# Option 2: Quick fix (temporary, resets on unplug)
sudo chmod 666 /dev/ttyUSB0
```

---

## 3. FreeRTOS on ESP32

### 3.1 Dual-Core Architecture

- **Core 0** (`PRO_CPU`) — used by Wi-Fi/Bluetooth stack
- **Core 1** (`APP_CPU`) — free for application tasks
- Pin tasks to cores using `xTaskCreatePinnedToCore()`

### 3.2 Task Creation

```cpp
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

void my_task(void *pvParameters)
{
    while (1) {
        // task body
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
    vTaskDelete(NULL); // never reached — self-delete if loop exits
}

void app_main(void)
{
    xTaskCreatePinnedToCore(
        my_task,       // task function
        "my_task",     // name (debug only, max 16 chars)
        4096,          // stack size in BYTES (IDF-specific, not words)
        NULL,          // parameter passed to task
        5,             // priority (1–24 typical; higher = more urgent)
        NULL,          // task handle (NULL if not needed)
        1              // core: 0 = PRO_CPU, 1 = APP_CPU
    );
}
```

### 3.3 FreeRTOS Pitfalls

| Pitfall | Explanation |
|---------|-------------|
| **Stack size is in bytes, not words** | ESP-IDF FreeRTOS uses bytes; vanilla FreeRTOS docs use words |
| **Never return from a task function** | Always loop forever or call `vTaskDelete(NULL)` |
| **Watchdog resets** | Always yield with `vTaskDelay()` or blocking calls; never busy-spin |
| **Stack overflow** | `LoadProhibited` or watchdog → increase stack size (try 4096 or 8192) |
| **Cross-core `vTaskDelete()`** | Avoid; prefer self-deletion with `vTaskDelete(NULL)` |
| **`tskNO_AFFINITY`** | Use to let the scheduler choose the core |

---

## 4. GPIO Configuration

### 4.1 GPIO Mode Trap — CE Pin

**Rule: Use `GPIO_MODE_INPUT_OUTPUT` for pins that need readback, NOT `GPIO_MODE_OUTPUT`.**

On ESP32, `GPIO_MODE_OUTPUT` disables the input buffer. `gpio_get_level()` on an
output-only pin **always returns 0** regardless of the actual output level. This
makes CE readback diagnostics show LOW even when the pin is physically HIGH.

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

**Affected pins in this project:** CE (currently GPIO4, was GPIO5 — see §4.3).

### 4.2 SPI Pin Direction Trap

**Rule: Never manually change GPIO direction on SPI pins after `spi_bus_initialize()`.**

Setting `gpio_set_direction(mosi_pin, GPIO_MODE_INPUT)` after SPI initialization
breaks **all** subsequent SPI write operations. The MOSI pin disconnects from the
SPI peripheral, making all write commands silently fail.

The ESP-IDF SPI Master Driver manages pin direction automatically. After
`spi_bus_initialize()` and `spi_bus_add_device()`, the driver controls the IOMUX
routing and GPIO configuration for all SPI pins (MOSI, MISO, SCK, CSN).

```cpp
// REMOVED: gpio_set_direction(hal.mosi_pin(), GPIO_MODE_INPUT);
// The SPI driver manages MOSI pin direction automatically.
// Manually changing it disconnects the pin from the SPI peripheral.
```

**For reducing RF noise during RX-only operation:**
1. Use `spi_bus_remove_device()` + `gpio_set_direction()` only if the SPI bus is no longer needed — but this requires full re-initialization to resume.
2. Let the SPI driver manage the pins — the noise reduction of tri-stating MOSI is minimal compared to the risk of breaking SPI.
3. Add a series resistor on the MOSI trace — a hardware-level fix that doesn't affect the SPI driver.

### 4.3 GPIO5 Overlap (RESOLVED)

CE was migrated from GPIO5 to GPIO4. GPIO5 is SPI3_HOST's native IO_MUX CS0 pin,
and using it for CE created a latent conflict. **Do NOT move CE back to GPIO5.**

---

## 5. SPI Bus Management

### 5.1 Initialization

```cpp
#include "driver/spi_master.h"

spi_bus_config_t bus_cfg = {
    .mosi_io_num = MOSI_PIN,
    .miso_io_num = MISO_PIN,
    .sclk_io_num = SCK_PIN,
    .quadwp_io_num = -1,
    .quadhd_io_num = -1,
    .max_transfer_sz = 0,  // default 4096
};

spi_device_interface_config_t dev_cfg = {
    .mode = 0,                    // CPOL=0, CPHA=0
    .clock_speed_hz = 1000000,   // 1 MHz (clone chip compatibility)
    .spics_io_num = CSN_PIN,
    .queue_size = 1,
};

ESP_ERROR_CHECK(spi_bus_initialize(SPI3_HOST, &bus_cfg, SPI_DMA_CH_AUTO));
ESP_ERROR_CHECK(spi_bus_add_device(SPI3_HOST, &dev_cfg, &spi_handle));
```

### 5.2 SPI Transfer

```cpp
spi_transaction_t trans = {};
trans.flags = 0;
trans.length = 8;          // bits to send
trans.rxlength = 8;        // bits to receive (0 = same as length)
trans.tx_buffer = &tx_byte;
trans.rx_buffer = &rx_byte;

spi_device_polling_transmit(spi_handle, &trans);
```

### 5.3 RAII Cleanup

The ESP-IDF SPI API requires explicit cleanup. Use RAII in the HAL destructor:

```cpp
EspIdfHal::~EspIdfHal() {
    if (spi_handle_ != nullptr) {
        spi_bus_remove_device(spi_handle_);
    }
    spi_bus_free(SPI3_HOST);
}
```

### 5.4 SPI Clock Speed

Genuine nRF24L01+ supports up to 10 MHz. Clone chips (Si24R1, BK2425) typically
need **1–2 MHz maximum**. At 8 MHz (ESP-IDF default), clones silently ignore
register writes. The project uses 1 MHz for clone compatibility.

---

## 6. Error Handling Patterns

### 6.1 ESP_ERROR_CHECK — Init Only

`ESP_ERROR_CHECK()` aborts on failure. It is appropriate **only** in initialization,
where a failure means the device cannot function.

```cpp
// GOOD — init code, abort is acceptable
ESP_ERROR_CHECK(spi_bus_initialize(SPI3_HOST, &bus_cfg, SPI_DMA_CH_AUTO));

// BAD — runtime code, abort kills the entire firmware
ESP_ERROR_CHECK(spi_device_polling_transmit(handle, &trans));
```

### 6.2 Error Codes — Runtime Code

Runtime code (SPI transfers, register writes) must propagate errors gracefully,
not abort:

```cpp
// GOOD — runtime code returns bool for success/failure
bool Driver::write_reg(uint8_t addr, uint8_t val) {
    spi_transaction_t trans = {};
    trans.length = 16;
    trans.tx_buffer = &tx_buf;
    esp_err_t ret = spi_device_polling_transmit(handle_, &trans);
    return ret == ESP_OK;
}
```

### 6.3 Hal::spi_xfer() Returns bool

In this project, `EspIdfHal::spi_xfer()` returns `false` on SPI errors instead of
calling `abort()`. All `Driver` methods propagate the error:

- `read_reg(uint8_t)` returns `0xFF` on failure
- `write_reg()`/`write_reg_multi()`/`read_reg_multi()`/`read_payload()`/`flush_rx()`/`flush_tx()` return `bool`
- Typed template overloads (`read_reg<T>()`, `write_reg<T>()`) retain their original void/T return types for API stability
- `EspIdfHal` has an RAII destructor that removes the SPI device and frees the bus

---

## 7. VS Code Diagnostics

### 7.1 Required Settings

If VS Code shows errors such as `cannot open source file "freertos/FreeRTOS.h"`,
verify `.vscode/settings.json` includes:

```jsonc
{
    "C_Cpp.default.configurationProvider": "espressif.esp-idf-extension",
    "C_Cpp.default.compileCommands": "${workspaceFolder}/build/compile_commands.json",
    "C_Cpp.default.compilerPath": "<Xtensa toolchain GCC path>",
    "idf.port": "/dev/ttyUSB0",
    "idf.flashType": "UART"
}
```

### 7.2 IDF_PYTHON_ENV_PATH Fix

The EIM-installed activate script does NOT export `IDF_PYTHON_ENV_PATH` or
`IDF_TOOLS_PATH`. The ESP-IDF extension's `idf.currentSetup` must match a key
under `idfInstalled` in `~/.espressif/idf-env.json`.

Fix in `.vscode/settings.json`:

```jsonc
{
    "idf.currentSetup": "$HOME/.espressif/<version>/esp-idf-<version>",
    "idf.customExtraVars": {
        "IDF_PATH": "$HOME/.espressif/<version>/esp-idf",
        "IDF_TOOLS_PATH": "$HOME/.espressif",
        "IDF_PYTHON_ENV_PATH": "$HOME/.espressif/tools/python/<version>/venv",
        "ESP_IDF_VERSION": "<version>"
    }
}
```

Verify the exact `idfInstalled` key:

```bash
python3 -c "import json; print(list(json.load(open('$HOME/.espressif/idf-env.json'))['idfInstalled']))"
```

For CLI shell (outside the activate script):

```bash
export IDF_TOOLS_PATH=$HOME/.espressif
export IDF_PYTHON_ENV_PATH=$HOME/.espressif/tools/python/v6.0.1/venv
```

---

## 8. Project Structure

- Framework: ESP-IDF v6.0.1
- ESP-IDF location: `~/.espressif/v6.0.1/esp-idf/`
- Library component: `components/nrf24l01plus/`
- Platform adapter: `main/` (ESP-IDF specific code, `EspIdfHal`)
- Build output: `build/`

---

## 9. Self-Reflection Clause

After fixing any ESP-IDF related bug, agents MUST answer these three questions:

1. **Why was this bug missed?** What assumption about the ESP-IDF API or ESP32
   hardware led to the incorrect code?
2. **What procedural safeguard would have caught it?** What test, code review
   pattern, or documentation check should be added?
3. **Does this lesson need to be captured?** If yes, update this skill or the
   relevant learning doc in `docs/learning/` with the lesson learned.

### Process for Updating This Skill

When a new ESP-IDF or ESP32 trap is discovered:

1. Add it as a new subsection under the relevant section with:
   - A one-line **Rule** in bold
   - The mechanism that makes it a trap
   - Correct code example (using library vocabulary, no magic numbers)
   - Broken code example (commented out, marked `// BROKEN`)
   - Symptom pattern table
   - Reference to ESP-IDF documentation or learning doc
2. Commit with: `docs(skills): add ESP-IDF trap — [trap name]`

---

## 10. References

- ESP-IDF SPI Master Driver: https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/peripherals/spi_master.html
- ESP-IDF GPIO & RTC GPIO: https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/peripherals/gpio.html
- ESP-IDF FreeRTOS SMP: https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/system/freertos_idf.html
- ESP-IDF Application Startup: https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-guides/startup.html
- Learning doc: `docs/learning/freertos-task-pinning.md`
- Learning doc: `docs/learning/esp32-app-main-basics.md`
- Learning doc: `docs/learning/mosi-spi-register-verification.md`
