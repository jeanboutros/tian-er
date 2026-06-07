---
name: datasheet-verification
description: "Mandatory verification protocol for hardware register details and RF protocol parameters. Every claim about hardware behaviour MUST be verified against the datasheet before coding."
---

# Datasheet Verification Protocol

## Purpose

In embedded development, wrong register values don't throw exceptions — they silently corrupt device state. Every hardware detail MUST be verified against the datasheet before it enters code.

## When to Use

- **Before coding any register struct** — verify bit layout
- **Before coding any RF parameter** — verify frequency, data rate, modulation
- **Before coding any timing** — verify delays, pulse widths
- **During review** — verify implementation matches datasheet

## The Protocol

### Step 1: Locate the Source

1. Check `docs/datasheets/` for the relevant datasheet
2. Find the specific register table, timing diagram, or protocol section
3. Note the page number and table/figure reference

### Step 2: Verify Field-by-Field

For each register field in the code:

```markdown
| Field | Code Value | Datasheet Value | Page/Table | Match? |
|-------|-----------|----------------|-----------|--------|
| [name] | [bit pos] | [bit pos] | p.X, Table Y | ✓/✗ |
```

### Step 3: Check Encodings

For multi-bit fields with specific encodings:
- List ALL valid values from the datasheet encoding table
- Verify the enum class enumerators match exactly
- Flag any "reserved" or "do not use" values

### Step 4: If Unclear

1. **Check the web** — application notes, errata, community reports
2. **Use `fetch_webpage`** on official sources (Nordic Semi, Espressif)
3. **If STILL unclear** → HALT with assumption-trap protocol

## Verification Checklist

| Item | Verified? |
|------|-----------|
| Register address matches datasheet | |
| Bit positions match register layout diagram | |
| Field encodings match encoding table | |
| Reset/default values match datasheet | |
| Reserved bits identified and handled | |
| Non-contiguous fields have explicit encoding logic | |
| Timing constraints noted and respected | |

## Key Datasheets for This Project

List your project-specific datasheets here (e.g. chip reference manuals, protocol specs).
Remove this section if your project has no hardware datasheets.

## Anti-Patterns

| Don't | Why |
|-------|-----|
| Rely on training data for register values | Training data may be from a different chip revision |
| Copy values from example code without verifying | Example code may be wrong or for a different variant |
| Assume "reserved" bits are zero | Some chips have reserved bits that must be written as 1 |
| Skip timing verification | Violating timing causes intermittent hard-to-debug failures |

## Self-Reflection Clause

After fixing any bug or resolving any issue that required debugging, you MUST ask:
1. **Why was this bug missed?** — What review, test, or protocol gap allowed it through?
2. **What procedural safeguard would have caught it?** — What specific check, test, or verification step would have prevented it?
3. **Update the knowledge base** — Add the lesson to the relevant skill or learning doc in `docs/learning/` so the same class of bug is caught earlier next time.
