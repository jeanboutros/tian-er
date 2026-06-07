---
name: assumption-trap
description: "Shared no-assumption protocol. Load this skill FIRST before any processing. Defines the structured format for halting on ambiguity — every agent must follow this protocol. Especially critical for hardware where guessing register values causes silent corruption."
---

# The Assumption Trap Protocol

## Purpose

You are bound by the **No Assumption** rule. When you encounter ANY gap, ambiguity, or unstated requirement, you MUST halt and signal rather than guess.

**This is especially critical in embedded/hardware work** — a wrong register value doesn't throw an exception, it silently corrupts the radio state.

## When to Trigger

Stop immediately if ANY of these apply:

- A register field's meaning is not clearly stated in the datasheet
- A bit position or encoding could be interpreted multiple ways
- A timing constraint is not specified (power-on delay, pulse width)
- A protocol detail (BLE spec) has edge cases not addressed
- A hardware behaviour depends on chip revision or errata
- You are about to write "typically", "usually", "by default", or "assuming"

## Output Format

When you hit a trap, return this EXACT structure and STOP processing:

```
STATUS: BLOCKED
CONTEXT: <What you were analyzing when you hit the gap>
QUESTION: <The specific question — one question only, be precise>
OPTIONS: <Suggested answers if applicable, as a numbered list>
IMPACT: <What downstream work depends on this answer>
```

## Rules

1. **Never guess.** Not even "reasonable" defaults. If the datasheet doesn't say it, you don't know it.
2. **Never infer.** A register field being "reserved" does not mean it's safe to write 0.
3. **Never fill gaps with training data.** The datasheet is the source of truth, not your training.
4. **One question at a time.** Report the FIRST blocking gap. The Director will re-invoke after resolution.
5. **Be specific.** "What is the reset value of RF_CH bit 7?" is better than "What are the register defaults?"
6. **Include options.** Help the user by listing concrete choices from the datasheet.

## Common Traps — Embedded/Hardware

### Register Design
- "This field is 3 bits" — What are ALL valid encodings? Are some reserved?
- "Power mode" — How long is the transition delay? What state are outputs during transition?
- "Address width" — Does 0x00 mean 3 bytes or illegal? Check the datasheet encoding table.

### RF Protocol
- "BLE channel 37" — Is the frequency 2402 MHz or something else? Verify against Core Spec.
- "Data whitening" — What's the initial seed? Is it channel-dependent? What polynomial?
- "Access address" — LSbit-first or MSbit-first? Per-byte or per-word reversal?

### Timing
- "After power-on" — How many ms before SPI is valid? Datasheet says 1.5ms? 10.3ms?
- "CE pulse" — Minimum width? What happens if too short?
