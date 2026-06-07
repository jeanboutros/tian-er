---
name: incremental-execution
description: "Protocol for breaking tasks into logical units and validating after each. Every implementation agent MUST follow this — no implementing an entire task at once."
---

# Incremental Execution Protocol

## Purpose

Prevent the failure mode where an agent implements an entire task in one batch, only to discover fundamental issues at the end. Every implementation MUST be broken into small, validatable units.

## When to Use

- **Code Architect** — Before and during every implementation task
- **Test Engineer** — Before and during test writing

## The Protocol

### Step 1: Identify Logical Units

Before writing any code, break the task into units:

```markdown
### Logical Units — [Task Name]

| # | Unit | Files | Validation |
|---|------|-------|-----------|
| 1 | [name] | [files] | build passes |
| 2 | [name] | [files] | build passes |
| 3 | [name] | [files] | build passes + tests pass |
```

### Step 2: Execute Unit-by-Unit

For each unit:
1. Implement the unit (and ONLY that unit)
2. Run the project build/test command
3. Confirm: exit 0, zero warnings
4. Report: "Unit N/M complete. Build passes."
5. Proceed to next unit

### Step 3: Final Validation

After all units, run the full build and test suite. Must exit 0.

## Rules

1. **Maximum unit size:** One register struct, one function, one test file section
2. **Never skip validation:** Even for "trivial" changes — -Werror catches everything
3. **Report between units:** State what completed and what's next
4. **Fix before proceeding:** If build fails, fix the current unit first
5. **Never bundle unrelated changes:** Each unit serves one purpose
