---
name: test-driven-development
description: "TDD loop: write test first, verify it catches the expected behaviour, then implement. Red-green-refactor for any language or framework. Load tdd-cpp alongside this skill when working in C++."
---

# Test-Driven Development

## Overview

Write the test first. Watch it fail. Write minimal code to pass.

Core principle: **If you didn't watch the test fail, you don't know if it tests the right thing.**

## When to Use

- New function, module, or component implementation
- New protocol or transformation logic
- Bug fixes (write the test that would have caught it)
- Refactoring (tests prove no regression)

## The TDD Cycle

```
RED ──→ GREEN ──→ REFACTOR
 ↑                    │
 └────────────────────┘
```

### RED: Write the Failing Test

Write a test for the behaviour you're about to implement — before writing any implementation code. Run it. It must fail.

If the test passes before you write the implementation, the test is wrong.

### GREEN: Implement Minimal Code

Write just enough code to make the test pass. No more. Resist the urge to generalise prematurely.

### REFACTOR: Clean Up

- Remove duplication
- Improve naming
- Add documentation
- Add edge case tests (boundary values, error paths)

The tests must still pass after refactoring.

## Test Categories

### 1. Happy Path Tests

Verify the function does what it should for valid inputs:

```
assert compute(known_input) == expected_output
```

### 2. Round-Trip / Encode-Decode Tests

For any serialise/deserialise or encode/decode pair:

```
assert decode(encode(value)) == value
assert encode(decode(bytes)) == bytes
```

### 3. Edge Case Tests

```
assert f(MAX_VALUE)   == expected
assert f(MIN_VALUE)   == expected
assert f(ZERO)        == expected
assert f(EMPTY)       == expected
```

### 4. Error Path Tests

```
assert f(invalid_input) raises Error
assert f(null_input)    returns default_or_error
```

### 5. Regression Tests

After every bug fix, write a test that would have caught it:

```
# This test documents the bug and proves it is fixed
assert f(previously_broken_input) == correct_output
```

## Rules

1. **Test file must exist before implementation** — even if empty
2. **The test must fail before implementation** — proves the test is real
3. **The test must pass after implementation** — proves correctness
4. **Never delete a test to make code pass** — fix the code instead
5. **Edge cases are not optional** — boundary values and error paths always tested
6. **One failing test at a time** — don't write multiple failing tests before implementing

## Self-Reflection Clause

After a bug that tests didn't catch:
1. **Why didn't a test catch this?** — Missing coverage, wrong assertion, or wrong test?
2. **Write the test now** — Add it to prevent regression.
3. **Update the knowledge base** — Add the pattern to `docs/learning/` if it's a recurring gap.
