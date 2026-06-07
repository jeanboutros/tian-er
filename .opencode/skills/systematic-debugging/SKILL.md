---
name: systematic-debugging
description: "Hypothesis-driven debugging for embedded systems. NO FIXES WITHOUT ROOT CAUSE. Especially important for hardware where symptoms are often misleading (SPI timing, RF interference, register state corruption)."
---

# Systematic Debugging — Embedded Systems

## Overview

Random fixes waste time and create new bugs. In embedded systems, symptoms are especially misleading — a corrupted register might manifest as "no packets received" when the real cause is a wrong bit in CONFIG.

Core principle: **ALWAYS find root cause before attempting fixes. Symptom fixes are failure.**

## The Iron Law

```
NO FIXES WITHOUT ROOT CAUSE INVESTIGATION FIRST
```

If you haven't completed Phase 1, you cannot propose fixes.

## When to Use

- Build fails with unclear error
- Hardware not responding (STATUS = 0xFF)
- No packets received despite "correct" configuration
- Intermittent failures
- Unexpected register values after configuration

## Phase 1: Observe & Hypothesize

1. **Reproduce** — What exact command/action triggers the bug?
2. **Observe** — What is the actual output vs expected?
3. **Hypothesize** — List 3-5 possible root causes, ordered by likelihood:

```markdown
### Hypotheses

| # | Hypothesis | Evidence For | Evidence Against | Test |
|---|-----------|-------------|-----------------|------|
| 1 | SPI not connected | STATUS=0xFF | — | Read STATUS, expect != 0xFF |
| 2 | Wrong register value | No RX packets | SPI works | Read back CONFIG, compare |
| 3 | CE not asserted | Config correct, no RX | — | Measure CE pin with scope |
```

## Phase 2: Test Hypotheses

For each hypothesis (highest likelihood first):
1. Design a **minimal test** that confirms or eliminates it
2. Execute the test
3. Record the result
4. If confirmed → proceed to Phase 3
5. If eliminated → next hypothesis

### Embedded-Specific Tests

| Symptom | Test |
|---------|------|
| STATUS = 0xFF | Check SPI wiring, CSN timing, power |
| STATUS = 0x0E (reset) | Check if configure_rx() ran, read back registers |
| No RX_DR interrupt | Check RF_CH, address, payload width, CE state |
| Garbled data | Check data rate mismatch, whitening, byte order |
| Intermittent | Check timing (CE pulse width, power-on delay) |

## Phase 3: Fix Root Cause

1. Fix ONLY the root cause identified in Phase 2
2. Run `idf.py build` — must pass
3. If hardware connected: flash and verify the fix resolves the symptom
4. Verify no regression in other functionality

## Phase 4: Prevent Recurrence

1. Add a `static_assert` or diagnostic check if applicable
2. Document the finding in `docs/learning/`
3. Update `nrf24::diag::verify_ble_rx()` if a new check would catch this

## Anti-Patterns

| Don't | Why |
|-------|-----|
| Change multiple things at once | Can't tell which fixed it |
| Assume "it works on other projects" | Your wiring/config may differ |
| Trust the code without reading registers back | Silent write failures exist |
| Skip the hypothesis phase | Random edits waste more time than thinking |
| Fix the symptom instead of the cause | It will come back |

## Self-Reflection Clause

After fixing any bug or resolving any issue that required debugging, you MUST ask:
1. **Why was this bug missed?** — What review, test, or protocol gap allowed it through?
2. **What procedural safeguard would have caught it?** — What specific check, test, or verification step would have prevented it?
3. **Update the knowledge base** — Add the lesson to the relevant skill or learning doc in `docs/learning/` so the same class of bug is caught earlier next time.
