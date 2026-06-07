---
name: pau-loop
description: "Plan-Apply-Validate execution protocol for implementation work. Ensures every task follows a structured cycle: plan the work, execute incrementally with build validation, then verify against acceptance criteria."
---

# Plan-Apply-Validate (PAU) Loop Protocol

## Purpose

Every implementation task follows a mandatory three-phase cycle. No phase can be skipped. This ensures that acceptance criteria are verified, not assumed, and that every unit of work compiles cleanly.

**Loaded by:** Code Architect, Test Engineer.

---

## The Loop

```
PLAN ──→ APPLY ──→ VALIDATE
  ↑                    │
  └──── NOT MET ───────┘
```

---

## Phase 1: PLAN

**Goal:** Understand the task and define exactly what will change.

### Steps

1. **Read the task requirements:**
   - Acceptance criteria (binary pass/fail)
   - Dependencies (ensure prerequisites are complete)
   - Referenced specs (datasheet pages, BLE spec sections, architecture docs)

2. **Verify hardware details** (if applicable):
   - Check `docs/datasheets/` for register layouts
   - Verify bit positions and encodings against datasheet tables
   - If unclear, HALT with assumption-trap protocol

3. **Plan the implementation:**

```markdown
## Implementation Plan — [Task Name]

### Acceptance Criteria
1. [AC-1]
2. [AC-2]

### Logical Units
| # | Unit | Files | AC Satisfied |
|---|------|-------|-------------|
| 1 | [name] | [files] | [AC #s] |
| 2 | [name] | [files] | [AC #s] |

### Approach
[Brief description — max 3 paragraphs]

### Datasheet References
- [Register X]: page Y, table Z
- [Timing constraint]: page Y, figure Z

### Risks / Unknowns
[Any concerns — or "None identified"]
```

---

## Phase 2: APPLY (Per Unit)

**Goal:** Implement one logical unit at a time, validating after each.

### Per-Unit Cycle

1. Implement the unit
2. Run build validation:
   ```bash
   source ~/.espressif/tools/activate_idf_v6.0.1.sh && idf.py build
   ```
3. Must exit 0 with **zero warnings** (`-Werror` is active)
4. If build fails → fix before proceeding to next unit
5. Report: "Unit N complete. Build passes. Moving to unit N+1."

### Rules
- NEVER implement multiple units before validating
- NEVER skip the build check
- If a unit introduces a design question → HALT (assumption-trap)
- All new public symbols MUST have Doxygen before build

---

## Phase 3: VALIDATE

**Goal:** Confirm all acceptance criteria are met.

### Steps

1. Final build check: `idf.py build` exits 0
2. Walk through each acceptance criterion — cite the code that satisfies it
3. Verify Doxygen on all new public symbols
4. Verify no magic numbers in public API
5. Report completion with AC evidence

### Failure
If any AC is not met:
- Identify which unit needs rework
- Loop back to APPLY for that unit only
- Maximum 3 rework loops before escalation

---

## Anti-Patterns (NEVER DO THESE)

| Anti-Pattern | Why It Fails |
|-------------|-------------|
| Implement everything, then build | One typo invalidates the whole batch |
| Skip build between units | Errors compound silently |
| Assume AC is met without evidence | Challenger will reject |
| Guess at hardware values | Silent radio corruption |
| Write code before reading datasheet | Wrong from the start |

## Self-Reflection Clause

After fixing any bug or resolving any issue that required debugging, you MUST ask:
1. **Why was this bug missed?** — What review, test, or protocol gap allowed it through?
2. **What procedural safeguard would have caught it?** — What specific check, test, or verification step would have prevented it?
3. **Update the knowledge base** — Add the lesson to the relevant skill or learning doc in `docs/learning/` so the same class of bug is caught earlier next time.
