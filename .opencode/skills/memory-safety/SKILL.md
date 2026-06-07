---
name: memory-safety
description: "C++ memory safety, leak detection, and RAII best practices for embedded systems. Use this skill whenever discussing memory leaks, buffer overflows, stack overflows, heap fragmentation, smart pointers, RAII patterns, ASAN/Valgrind/MSAN, FreeRTOS heap analysis, or any question about C++ memory management in constrained environments. Also use when reviewing code for lifetime bugs, dangling references, use-after-free, double-free, or uninitialized memory access."
---

# Memory Safety Skill

## When to Use This Skill

Use this skill whenever you encounter any of the following situations:

- Reviewing code for memory leaks, buffer overflows, or use-after-free bugs
- Analyzing FreeRTOS heap usage, stack depth, or task memory allocation
- Designing RAII wrappers or smart pointer patterns for embedded C++
- Running or interpreting AddressSanitizer (ASAN), MemorySanitizer (MSAN), or Valgrind results
- Debugging heap fragmentation or memory exhaustion in constrained environments
- Evaluating whether `new`/`delete`, `malloc`/`free`, or static allocation is appropriate
- Reviewing DMA buffer alignment, ownership, and lifetime management
- Investigating stack overflow risks in embedded tasks
- Designing ownership models for shared peripherals (SPI bus, I2C bus, GPIO pins)
- Any question about "is this memory-safe?" or "could this leak?"

## Core Principles

### 1. RAII Over Manual Management

Every resource acquisition (memory, file handle, peripheral, mutex) must be wrapped in a class where:
- The **constructor** acquires the resource
- The **destructor** releases the resource (even on exception paths)
- **Copy/move semantics** are explicitly defined or deleted

```cpp
// BAD — manual management, leaks on early return or exception
void process() {
    auto* buf = new uint8_t[256];
    if (error_condition) return;  // LEAK!
    use(buf);
    delete[] buf;
}

// GOOD — RAII, exception-safe
void process() {
    std::unique_ptr<uint8_t[]> buf(new uint8_t[256]);
    if (error_condition) return;  // safe — unique_ptr destructs
    use(buf.get());
}
```

### 2. Stack-First Allocation for Embedded

In embedded systems, prefer stack allocation and static allocation over heap:

| Allocation Type | When to Use | Risk |
|---|---|---|
| **Static/global** | Constants, lookup tables, singleton peripherals | None (deterministic) |
| **Stack** | Small, scoped buffers (<1KB) in functions | Overflow if too large |
| **Heap** | Dynamic/large buffers, polymorphic objects | Fragmentation, leak, nondeterministic timing |
| **FreeRTOS heap** | `pvPortMalloc`/`vPortFree` for task stacks, queues | Fragmentation over time |

**Rules for ESP32/nRF24:**
- Buffers ≤64 bytes: stack allocation
- Buffers 64–256 bytes: stack with `std::array` or `std::unique_ptr<uint8_t[]>`
- Buffers >256 bytes: heap with `std::unique_ptr` or FreeRTOS `pvPortMalloc`
- **Never** use bare `new`/`malloc` without an owning smart pointer

### 3. Buffer Safety Checklist

For every buffer in the codebase, verify:

| Check | Criterion |
|---|---|
| **Bounds** | Buffer size is known at compile time or bounded by a validated length |
| **Null termination** | String buffers have room for null terminator |
| **Overflow** | `memcpy`, `snprintf`, `strncpy` destination size is checked |
| **Underflow** | Array indices are validated before access |
| **Lifetime** | Buffer outlives all pointers/references to it |
| **Alignment** | DMA buffers meet hardware alignment requirements (typically 4-byte or cache-line) |
| **Ownership** | Exactly one owner is responsible for freeing |

### 4. Smart Pointer Decision Tree

```
Need dynamic allocation?
├── NO → use stack/static allocation
└── YES
    ├── Shared ownership needed?
    │   ├── YES → std::shared_ptr (rare in embedded; consider redesign)
    │   └── NO → std::unique_ptr
    └── Array needed?
        ├── YES → std::unique_ptr<T[]> with new T[n]
        └── NO → std::unique_ptr<T> with std::make_unique<T>(...)
```

**Embedded-specific guidance:**
- `std::make_unique` is available in C++14 (use `std::unique_ptr<T>(new T(...))` for C++11)
- Avoid `std::shared_ptr` in embedded — the control block adds overhead and creates hidden coupling
- For peripheral handles (SPI bus, GPIO), use `std::unique_ptr` with custom deleters or reference-counted RAII wrappers

### 5. FreeRTOS Memory Safety

| Hazard | Detection | Prevention |
|---|---|---|
| **Stack overflow** | `uxTaskGetStackHighWaterMark()` | Allocate 1.5–2x estimated max usage; use `configCHECK_FOR_STACK_OVERFLOW=2` |
| **Heap fragmentation** | `xPortGetFreeHeapSize()`, `heap_caps_print_heap_info()` | Limit dynamic allocation to init phase; use `heap_caps_malloc()` with caps |
| **Task stack depth** | Static analysis of call chain depth | Document worst-case stack usage per task in comments |
| **Priority inversion** | Mutex priority inheritance | Use `xSemaphoreCreateMutex()` (not binary semaphores for mutual exclusion) |
| **Dangling timer callback** | `xTimerDelete()` before freeing timer data | Ensure timer callbacks don't access freed data |

### 6. AddressSanitizer (ASAN) Integration

For host-side unit tests, enable ASAN to catch memory errors:

```cmake
# In tests/CMakeLists.txt
target_compile_options(test_rf_setup PRIVATE -fsanitize=address -fno-omit-frame-pointer -g)
target_link_options(test_rf_setup PRIVATE -fsanitize=address)
```

ASAN detects:
- Use-after-free
- Heap buffer overflow
- Stack buffer overflow
- Global buffer overflow
- Use-after-return (with `ASAN_OPTIONS=detect_stack_use_after_return=1`)
- Memory leaks (with `ASAN_OPTIONS=detect_leaks=1`)

### 7. ESP-IDF Heap Analysis

```c
// In app_main() or diagnostic task:
#include "esp_heap_caps.h"
#include "esp_system.h"

void print_heap_info() {
    printf("Free heap: %lu bytes\n", esp_get_free_heap_size());
    printf("Min free heap: %lu bytes\n", esp_get_minimum_free_heap_size());
    heap_caps_print_heap_info(MALLOC_CAP_8BIT);
}

// Enable heap poisoning for debug builds:
//.menuconfig → Component Config → Heap Memory Debugging → Comprehensive
```

### 8. Lifetime and Ownership Patterns for SPI/I2C Peripherals

For shared peripherals (like the nRF24 SPI bus):

```cpp
// BAD — raw pointer, no ownership tracking
class Driver {
    Hal* hal_;  // who owns this? who frees it?
};

// GOOD — reference semantics (non-owning, lifetime guaranteed by scope)
class Driver {
    Hal& hal_;  // caller guarantees lifetime — documented in @param
};

// GOOD — unique_ptr for transferable ownership
class Driver {
    std::unique_ptr<Hal> hal_;  // driver owns the HAL, moves with driver
};

// BEST for embedded — static allocation with reference
static EspIdfHal hal;  // static lifetime, never freed
static Driver radio(hal);  // reference to static, lifetime = program
```

### 9. Common Embedded Memory Anti-Patterns

| Anti-Pattern | Problem | Fix |
|---|---|---|
| `new` without `delete` | Memory leak | Use `std::unique_ptr` or stack allocation |
| `malloc` without `free` | Leak in C code | Use `std::unique_ptr` with custom deleter |
| Large stack arrays in tasks | Stack overflow | Use `std::unique_ptr<uint8_t[]>` or static buffers |
| Unbounded `printf` to fixed buffer | Buffer overflow | Use `snprintf` with size check |
| `vTaskDelay(1)` with large stack | Wasted stack | Profile with `uxTaskGetStackHighWaterMark()` |
| Dynamic allocation in ISR | Non-deterministic, can crash | Pre-allocate, use queues |
| Missing virtual destructor | Undefined behavior on delete | Always `virtual ~Base() = default;` |
| Returning reference to local | Dangling reference | Return by value or use `std::unique_ptr` |

### 10. Diagnostic Techniques

| Technique | When to Use | How |
|---|---|---|
| **ASAN** | Host-side unit tests | `-fsanitize=address` in CMake |
| **Valgrind** | Host-side integration tests | `valgrind --leak-check=full ./test_binary` |
| **ESP-IDF heap tracing** | Target-side leak detection | `CONFIG_HEAP_TRACING_STANDALONE=y` + `heap_caps_check_integrity()` |
| **Stack watermark** | Target-side stack overflow detection | `uxTaskGetStackHighWaterMark(NULL)` after heavy usage |
| **Free heap monitoring** | Runtime memory pressure | `esp_get_free_heap_size()` periodic logging |
| **Canary values** | Stack/heap corruption detection | `CONFIG_STACK_CHECK_ALL=y` in menuconfig |

## Verification Protocol

When reviewing code for memory safety, follow this checklist:

1. **Buffer bounds**: Every buffer access is within declared bounds
2. **Lifetime**: No dangling references or use-after-free
3. **Ownership**: Every allocation has exactly one responsible owner
4. **RAII**: All resources (memory, SPI bus, GPIO) are RAII-wrapped
5. **Stack depth**: FreeRTOS task stacks are ≥1.5x estimated peak usage
6. **Heap fragmentation**: Dynamic allocation limited to init phase
7. **ASAN/Valgrind**: Host-side tests compiled with sanitizers
8. **DMA safety**: Buffer alignment, cache coherency, ownership transfer documented
9. **Printf format**: No format string injection, all format strings are compile-time constants
10. **Smart pointers**: `std::unique_ptr` for exclusive ownership, never bare `new`/`delete`

## References

- ESP-IDF Heap Debugging: https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/system/mem_alloc_debug.html
- ASAN Documentation: https://clang.llvm.org/docs/AddressSanitizer.html
- C++ Core Guidelines — Resource Management: https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#S-resource
- FreeRTOS Stack Overflow Detection: https://www.freertos.org/Stacks-and-stack-overflow-checking.html