---
name: type-design-review
description: "4-dimension type quality scoring for APIs, register structs, enum classes, and public interfaces. Triggered when adding or modifying types, during Phase C review, and during architecture gates. Enforces the typed vocabulary mandate: public API must use typed enums and struct wrappers, not raw integers."
---

# Type Design Review

## Purpose

This skill defines a systematic 4-dimension scoring system for evaluating the quality of type designs in your project's API. It enforces the project's core principle: **the public API must give users a vocabulary, not raw bytes.**

## When to Trigger

- **Adding or modifying types** — any new `enum class`, `struct`, `class`, `typedef`, or `using` in a public header.
- **Adding or modifying public methods** — any new method signature on a public class.
- **Adding or modifying register structs** — any `to_byte()`/`from_byte()` struct.
- **Phase C review** — when evaluating type quality as part of specialist verification.
- **Architecture gates (T-ARCH, T2)** — when checking API surface and type design principles.
- **When compliance-gate detects T1.5 or T2.4 violations** — raw `uint8_t` where typed vocabulary should exist.

---

## The Four Dimensions

### Dimension 1: Encapsulation (1-10)

**Question:** Are invariants protected? Are implementation details hidden? Can users create invalid states?

| Score | Description |
|-------|-------------|
| 10 | Users **cannot** create invalid states. All constructors enforce invariants. All fields are private with accessor methods. The type is immutable after construction. |
| 7-9 | Invalid states are difficult but not impossible to create. Most fields are private with typed accessors. Constructor validates input. |
| 5-6 | Some fields are public and can be set to invalid values. The type relies on documentation to prevent misuse. `from_byte()` may accept any `uint8_t` without validation. |
| 3-4 | Many fields are public raw integers. Users can easily set invalid combinations. No validation in constructor or setter. |
| 1-2 | All fields are public raw integers. Any value can be assigned. No encapsulation at all. |

**How to improve:**
- Make fields `private` with typed accessor methods.
- Validate input in constructors and `from_byte()` — reject or mask invalid values.
- Use `enum class` for fields with finite legal values so invalid values are compile-time errors.
- For reserved bits: mask them in `to_byte()`, ignore them in `from_byte()`.

**Example — 10/10:**
```cpp
class RfSetup {
    DataRate data_rate_;
    TxPower tx_power_;
    // no public raw fields — users can only set typed values
public:
    constexpr DataRate data_rate() const { return data_rate_; }
    constexpr void set_data_rate(DataRate dr) { data_rate_ = dr; }
    // from_byte() masks reserved bits, validates known encodings
};
```

**Example — 5/10:**
```cpp
struct RfSetup {
    uint8_t data_rate : 2;  // raw bitfield — user can set 0b11 (invalid)
    uint8_t tx_power : 2;   // raw bitfield — any value accepted
    uint8_t to_byte() const;
    static RfSetup from_byte(uint8_t);  // accepts any byte, no validation
};
```

**Example — 1/10:**
```cpp
uint8_t rf_setup_raw;  // just a raw byte, no type safety at all
```

### Dimension 2: Invariant Expression (1-10)

**Question:** Can the type system enforce valid states, or do you need runtime checks? Are invalid values rejected at compile time?

| Score | Description |
|-------|-------------|
| 10 | **All** invalid values are rejected at compile time. The type system makes it impossible to represent an illegal state. `enum class` with no raw escape hatch. Struct fields use typed enums with exhaustive `switch`. |
| 7-9 | Most invalid values are rejected at compile time. Enum classes for finite-valued fields. A few edge cases need runtime checks (e.g. reserved bit patterns). |
| 5-6 | Many invalid values require runtime checks. Some fields are `enum class` but others are `uint8_t` with named constants. A `from_byte()` that accepts any byte. |
| 3-4 | Most fields are raw integers with named `constexpr` constants. Invalid values are only caught at runtime. Users can pass any raw value where a typed parameter is expected because the type is not enforced. |
| 1-2 | Everything is raw `uint8_t` or `int`. No type enforcement at all. |

**How to improve:**
- Replace `uint8_t` parameters with typed structs when the parameter represents a register (e.g. `write_reg(Config{})` instead of `write_reg(0x00, value)`).
- Use `enum class` for every field with finite legal values — no "Raw = 0xFF" catch-all enumerator.
- Make raw `uint8_t` overloads `private` or `protected` — never `public`.
- For `from_byte()`: if the byte contains an illegal encoding, either return an error type or mask/map to the nearest valid value — never silently accept any byte.
- Use `static_assert` to verify round-trip properties at compile time.

**Example — 10/10:**
```cpp
enum class DataRate : uint8_t { Mbps1 = 0b00, Mbps2 = 0b01, Kbps250 = 0b10 };
// Cannot construct an invalid DataRate — only three values exist.
// No Raw=0xFF escape hatch.

class Driver {
public:
    void write_reg(Config cfg);           // typed overload — public
    void write_reg(RfSetup rf);           // typed overload — public
private:
    void write_reg(uint8_t addr, uint8_t val);  // raw overload — private
};
```

**Example — 5/10:**
```cpp
enum class DataRate : uint8_t { Mbps1 = 0, Mbps2 = 1, Kbps250 = 2 };
// Good: enum class for DataRate
// Problem: Driver still has public write_reg(uint8_t, uint8_t)
// Problem: from_byte() accepts 0b11 as DataRate without error
```

**Example — 1/10:**
```cpp
void write_reg(uint8_t addr, uint8_t val);  // anything goes, no typing at all
```

### Dimension 3: Usefulness (1-10)

**Question:** Does the type make the API self-documenting? Would a newcomer understand the intent without looking at the datasheet?

| Score | Description |
|-------|-------------|
| 10 | The API reads like documentation. `radio.set_rate(DataRate::Mbps1)` is immediately clear. Named typed constants are used everywhere. Doc examples use the library's own vocabulary. A newcomer can write correct code without reading external docs. |
| 7-9 | The API is mostly self-documenting. Most operations use named types and enumerators. A few operations still require looking up the meaning of parameters. |
| 5-6 | The API is partially self-documenting. Some operations use typed enums, others use raw integers with named constants. A newcomer needs to read docs for roughly half the API. |
| 3-4 | The API is mostly opaque. Named constants exist but are `constexpr uint8_t` values, not type-safe. A newcomer must read the datasheet to understand most operations. |
| 1-2 | The API is completely opaque. `write_reg(0x00, 0x03)` — requires memorizing the datasheet. |

**How to improve:**
- Every `@code` example in documentation must use typed vocabulary first: `radio.write_reg(cfg)` not `radio.write_reg(0x00, 0x03)`. For C/C++ projects, this applies to Doxygen `@code` blocks.
- Parameters should have types that document their purpose: `DataRate` not `uint8_t rate`.
- Use aggregate initialization for register structs: `Config{.en_crc = true}` not constructor with positional args.
- Provide `format()` methods that produce human-readable debug output.
- Learning docs must use the typed API with a note for educational raw examples.

**Example — 10/10:**
```
// Example — 10/10 (self-documenting typed API):
Config cfg;
cfg.enable_crc = true;
cfg.power_up = true;
radio.write_reg(cfg);  // Self-documenting. No datasheet lookup needed.
```

**Example — 5/10:**
```
// Readable but requires knowing the register address:
radio.write_reg(reg::CONFIG, Config{.en_crc = true}.to_byte());
// Better than raw values, but the typed overload (write_reg(Config{}))
// would be even clearer.
```

**Example — 1/10:**
```cpp
radio.write_reg(0x00, 0x03);  // What does this do? Need datasheet.
```

### Dimension 4: Enforcement (1-10)

**Question:** Are invalid values rejected at compile time? Are raw types hidden from the public API?

| Score | Description |
|-------|-------------|
| 10 | **All** public API surface uses typed enums and struct wrappers. Raw `uint8_t` overloads exist but are `private`. Invalid register addresses, field values, and command codes are compile-time errors. Users simply cannot write `radio.write_reg(0xFF, 0x00)` because the types prevent it. |
| 7-9 | Most public API is typed. A few raw overloads are still `public` but deprecated or clearly documented as internal. Most invalid values are compile-time errors. |
| 5-6 | Raw `uint8_t` overloads are `public` alongside typed overloads. Users can bypass typing. Named constants exist as `constexpr uint8_t` values (not type-safe). |
| 3-4 | The API is mostly raw `uint8_t`. A few `enum class` values exist for the most common parameters. Most invalid values are silently accepted at runtime. |
| 1-2 | Everything is `uint8_t` or `int`. No type enforcement. Any value compiles and may silently fail at runtime. |

**How to improve:**
- Move all raw `uint8_t` overloads to `private` or `protected` sections.
- Provide typed struct overloads that deduce the register/command from the struct type: `write_reg(Config{})` instead of `write_reg(reg::CONFIG, val)`.
- If both raw and typed overloads exist, the raw overload MUST be private. The presence of a typed overload does not excuse making the raw overload public.
- Add `static_assert` tests that verify round-trip properties: `static_assert(Config{}.to_byte() == 0x08, "default config");`
- In code reviews: grep for `public.*uint8_t` in headers. Every match must have a typed alternative or be made private.

**Example — 10/10:**
```cpp
class Driver {
public:
    // Typed overloads — the ONLY public API
    void write_reg(Config cfg);
    void write_reg(RfSetup rf);
    void write_reg(SetupAw aw);
    Config read_reg(Config);
    RfSetup read_reg(RfSetup);
    
private:
    // Raw overload — internal use only
    void write_reg(uint8_t addr, uint8_t val);
    uint8_t read_reg(uint8_t addr);
};
```

**Example — 5/10:**
```cpp
class Driver {
public:
    void write_reg(Config cfg);              // typed — good
    void write_reg(uint8_t addr, uint8_t val); // raw — PUBLIC — bad
    // User can write radio.write_reg(0xFF, 0x00) — no type safety
};
```

**Example — 1/10:**
```cpp
class Driver {
public:
    void write_reg(uint8_t addr, uint8_t val);  // only raw API exists
};
```

---

## Scoring a Type Design

### Overall Score

The overall type design score is the average of the four dimensions:

```
Overall = (Encapsulation + Invariant Expression + Usefulness + Enforcement) / 4
```

### Scoring Thresholds

| Overall | Meaning |
|---------|---------|
| 8.0-10.0 | Excellent — type design fully enforces the typed vocabulary mandate |
| 6.0-7.9 | Acceptable — mostly typed, some improvement needed |
| 4.0-5.9 | Needs improvement — significant raw integer exposure |
| 0.0-3.9 | Unacceptable — does not meet the project's API standards |

### Minimum Thresholds for Gate Pass

| Gate | Minimum Overall | Minimum per Dimension |
|------|----------------|----------------------|
| A-GATE | 7.0 | No dimension below 5 |
| B-FINAL-GATE | 8.0 | No dimension below 6 |
| C-GATE | 8.0 | No dimension below 6 |

If a type design scores below the minimum, the gate fails for T2.4 (API surface audit) or T-ARCH.3 (principle alignment).

---

## Review Template

When evaluating a type design, produce this assessment:

```markdown
## Type Design Review

**Type:** [class/struct/enum name]
**File:** [header file:line]
**Reviewer:** [Agent role]
**Date:** [ISO 8601]

### Dimension Scores

| Dimension | Score | Justification |
|-----------|-------|---------------|
| Encapsulation | [1-10] | [Specific observations about invariant protection, field visibility] |
| Invariant Expression | [1-10] | [Specific observations about compile-time vs runtime enforcement] |
| Usefulness | [1-10] | [Specific observations about self-documentation, newcomer comprehension] |
| Enforcement | [1-10] | [Specific observations about raw type exposure, private overloads] |

**Overall Score:** [average] / 10

### Findings

| ID | Confidence | Dimension | Description | Suggested Fix |
|----|-----------|-----------|-------------|---------------|
| F1 | [0-100] | [Enf/Inv/Enc/Use] | [specific observation] | [concrete fix] |

### Verdict

[APPROVED / CONDITIONAL PASS / REJECTED]
**Rationale:** [Why this verdict, referencing minimum thresholds]
```

---

## The Typed Vocabulary Mandate

This project has a hard requirement: **the public API must use typed enums and struct wrappers, not raw integers.**

### The Mandate (from AGENTS.md)

1. Every register field with finite legal values **must be an `enum class`**.
2. Every method parameter accepting a finite set of values **must use a typed enum or struct with ADDRESS constant**.
3. Convenience `uint8_t` overloads **must be `private` or `protected`** — never `public`.
4. Named raw-typed constants (e.g. `constexpr uint8_t reg::CONFIG`) are **documentation aids, NOT type safety**. If a raw-typed parameter accepts these constants, it also accepts any invalid value.
5. The typed API must be the **ONLY public API**.

### What This Means in Practice

```cpp
// BAD — public raw overload allows invalid values
class Driver {
public:
    void write_reg(uint8_t addr, uint8_t val);  // 0xFF is accepted but invalid
};

// GOOD — typed overloads only, raw overload is private
class Driver {
public:
    void write_reg(Config cfg);     // Cannot pass invalid type
    void write_reg(RfSetup rf);     // Cannot pass invalid type
private:
    void write_reg(uint8_t addr, uint8_t val);  // internal use only
};
```

### How This Skill Enforces the Mandate

- **T1.5** checks for public `uint8_t` parameters where typed vocabulary exists.
- **T2.4** checks that every public method parameter is maximally restrictive.
- **T-ARCH.3** checks principle alignment with the typed vocabulary mandate.
- **This skill** provides the scoring rubric to evaluate whether a type design meets the mandate.

---

## Self-Reflection Clause

After reviewing a type design and finding issues, ask:

1. **Why was this not caught during design?** — What review, protocol, or automated check gap allowed a raw-type API to be proposed?
2. **What procedural safeguard would have caught it?** — What specific check in the pipeline would have prevented it?
3. **Update the knowledge base** — Add the lesson to `docs/learning/` if the pattern is new, or update the type-design-review skill if the scoring criteria need refinement.

### Common Patterns and Lessons

| Pattern | Why It's Wrong | Lesson |
|---------|---------------|--------|
| Public `uint8_t` overload alongside typed overload | Users bypass typing, pass `0xFF` | Raw overloads must be private |
| `constexpr uint8_t` for register addresses | Not type-safe — any byte accepted | Use struct with `ADDRESS` constant instead |
| `from_byte()` that accepts any `uint8_t` | Invalid encodings silently accepted | Validate or mask in `from_byte()` |
| Enum class with `Raw = 0xFF` catch-all | Defeats type safety entirely | No catch-all enumerator; use separate error path |
| Public field `uint8_t` in register struct | Users can set invalid values | Use enum class fields with accessor methods |
