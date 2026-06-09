---
name: doxygen-cpp
description: "Doxygen documentation standard for C and C++ projects. Defines the mandatory structure, tag coverage rules, @code example conventions, and verification checklist. Loaded by Docs Writer when the project language is C or C++."
---

# Doxygen-CPP — C/C++ Documentation Standard

## Purpose

Defines the mandatory Doxygen documentation format for C and C++ projects. This is the language-specific doc-standard skill for C/C++ — loaded by Docs Writer during Phase A and Phase C when the project's tech stack includes C or C++.

## When to Trigger

- Loaded by Docs Writer when the project language is C or C++
- Loaded by Code Architect and Test Engineer during Phase B to ensure documentation compliance

---

## Mandatory Structure

Every new **function**, **struct**, **class**, **enum**, or **public typedef** must be preceded by a Doxygen `/** ... */` block:

```cpp
/**
 * @brief One-sentence summary of what this does.
 *
 * Longer explanation if needed — describe the protocol, bit layout,
 * algorithm, or non-obvious design decision here.
 *
 * @code
 *   // Example using library vocabulary
 *   MyType obj;
 *   obj.field = MyEnum::Value;
 * @endcode
 *
 * @param name   Description (include units, valid range).
 * @return       What is returned and under what conditions.
 */
```

## Tag Coverage Rules

| Element | Rule |
|---------|------|
| `@brief` | Always present; one sentence, no period at end |
| `@code` | Use for any non-obvious bit manipulation, protocol encoding, or illustrative call site |
| `@param` | One line per parameter; include units, valid range, or bit meaning |
| `@return` | Always present for non-void functions |
| Block comments | `/* short inline notes */` inside function bodies — do NOT duplicate Doxygen content |

## @code Example Conventions

All `@code` examples must use the library's own typed vocabulary:

```cpp
// BAD — raw hex in an @code block
radio.write_reg(0x00, 0x03);

// GOOD — uses the library's own vocabulary
radio.write_reg(nrf24::Config{...});
```

Raw literals are acceptable only in comments marked `// internal use`.

## Verification Checklist

| # | Check | Criterion |
|---|-------|-----------|
| 1 | Doxygen presence | Every new public function/struct/enum has `/** @brief */` |
| 2 | @param coverage | Every parameter documented with units/range/meaning |
| 3 | @return coverage | Every non-void function documents return value |
| 4 | @code examples | Use library vocabulary (no raw hex, no magic numbers) |

## Mechanical Check (T1.2)

Greppable: every new/changed function, struct, enum, method must have a `/**` block with `@brief`. Missing `@brief` on any public symbol is a T1.2 failure.

## Self-Reflection Clause

After fixing any documentation issue, ask:
1. **Why was this doc gap missed?** — What review or gate allowed it through?
2. **What procedural safeguard would have caught it?** — Would a grep check or pre-commit hook have prevented it?
3. **Update the knowledge base** — Add the lesson to this skill or the docs-writer agent definition.
