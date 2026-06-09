---
name: pipeline-passport
description: "Pipeline Passport — a task card that travels with every task through the pipeline, tracking which steps have been executed, which gates have been passed, and which steps have been skipped with justification. No step may be skipped without written justification. An agent receiving a card with a missing previous step must reject the task and return failure to the dispatcher."
---

# Pipeline Passport

## Purpose

The Pipeline Passport is a **task card** that travels with every task through the pipeline. It is created by the PM when a ticket is opened, and it accumulates state as the task moves through each phase, step, and gate.

**Core principle: No step is skipped without written justification.** An agent that receives a task with a missing previous step must reject it and return failure to the Supreme Leader.

## When to Trigger

- **Always** — every task dispatched through the pipeline carries a passport.
- Loaded by the Supreme Leader before dispatching any task.
- Loaded by every specialist agent at the start of their work.

---

## Passport Format

The passport is stored as a markdown file in `docs/project-management/passports/<ticket-id>-passport.md`.

### Template

```markdown
# Pipeline Passport: <ticket-id>

## Task Identity

| Field | Value |
|-------|-------|
| Ticket | <ticket-id> |
| Title | <short description> |
| Created | <YYYY-MM-DD> |
| PM | <creator> |

## Required Steps

Every step the pipeline requires for this task. Steps are checked off sequentially. No step may be skipped without a written justification in the Skipped Steps section below.

### Phase A — Requirements & Design

- [ ] A0: Task Definition — acceptance criteria, files, constraints, test strategy, doc plan. **Domain classification:** ________ | **Roster:** ________
- [ ] A1: Specialist Review — all dispatched specialists review independently
  - [ ] A1-SW: Software Engineer
  - [ ] A1-TX: Test Engineer
  - [ ] A1-DX: Docs Writer
  - [ ] A1-HW: Hardware Engineer (conditional — if hardware in scope)
  - [ ] A1-WX: Wireless Expert (conditional — if wireless/RF in scope)
  - [ ] A1-SX: Security Reviewer (conditional — if security in scope)
  - [ ] A1-PD: Product Designer (conditional — if UI/UX in scope)
  - [ ] A1-UX: UX Engineer (conditional — if UI/UX in scope)
- [ ] A2: Dual-Model Challenge — primary pass + challenger pass
- [ ] A2a: ADR Creation — ADR file for every resolved design decision
- [ ] A3: A-GATE — T3 ✅/❌ | T-ARCH ✅/❌ | ADRs present ✅/❌ | Verdict: _______

### Phase B — Build (PAU Loop)

- [ ] B1: PLAN — identify files, acceptance criteria, logical units
- [ ] B2-1: APPLY (unit 1) — implement, run build
- [ ] B2a-1: B-UNIT-GATE — T1 ✅/❌ | T-ARCH ✅/❌ | Verdict: _______
- [ ] B2-N: APPLY (unit N) — implement, run build
- [ ] B2a-N: B-UNIT-GATE — T1 ✅/❌ | T-ARCH ✅/❌ | Verdict: _______
- [ ] B3: VALIDATE — full build, optional flash
- [ ] B3a: B-FINAL-GATE — T1 ✅/❌ | T2 ✅/❌ | T-ARCH ✅/❌ | Verdict: _______

### Phase C — Multi-Agent Verify

- [ ] C0: T1 Re-run — all T1 checks pass
- [ ] C1: Dual-Model Challenge (Verification) — primary + challenger
- [ ] C2: Specialist Approval — all dispatched specialists
- [ ] C3: C-GATE — T1 ✅/❌ | T3 ✅/❌ | T-ARCH ✅/❌ | Verdict: _______

### Commit

- [ ] COMMIT — all gates passed, all approvals issued

## Step Log

Every step execution is logged here with timestamp, agent, and result.

| Step | Agent | Timestamp | Result | Notes |
|------|-------|-----------|--------|-------|
| A0 | _______ | _______ | _______ | _______ |
| A1-SW | software-engineer | _______ | APPROVED/CONDITIONAL PASS/REJECTED | _______ |
| A1-TX | test-engineer | _______ | APPROVED/CONDITIONAL PASS/REJECTED | _______ |
| A1-DX | docs-writer | _______ | APPROVED/CONDITIONAL PASS/REJECTED | _______ |
| A1-HW | hardware-engineer | _______ | APPROVED/CONDITIONAL PASS/REJECTED | _______ |
| A1-WX | wireless-expert | _______ | APPROVED/CONDITIONAL PASS/REJECTED | _______ |
| A1-SX | security-reviewer | _______ | APPROVED/CONDITIONAL PASS/REJECTED | _______ |
| A1-PD | product-designer | _______ | APPROVED/CONDITIONAL PASS/REJECTED | _______ |
| A1-UX | ux-engineer | _______ | APPROVED/CONDITIONAL PASS/REJECTED | _______ |
| A2-Primary | _______ | _______ | _______ | _______ |
| A2-Challenger | _______ | _______ | _______ | _______ |
| A2a-ADR | _______ | _______ | _______ | _______ |
| A3-Gate | supreme-leader | _______ | PASS/FAIL | T3: __ T-ARCH: __ ADRs: __ |
| ... | ... | ... | ... | ... |

## Gate Results

| Gate | Tier | Attempt | Result | Retry Budget | Notes |
|------|------|---------|--------|---------------|-------|
| A-GATE | T3 | 1 | _______ | 0/3 | _______ |
| A-GATE | T-ARCH | 1 | _______ | 0/3 | _______ |
| B-UNIT-GATE-1 | T1 | 1 | _______ | 0/3 | _______ |
| B-UNIT-GATE-1 | T-ARCH | 1 | _______ | 0/3 | _______ |
| ... | ... | ... | ... | ... | ... |

## Skipped Steps

Any step that was skipped MUST have a written justification here. If this section is empty, no steps were skipped.

| Step | Justification | Authorised By |
|------|--------------|---------------|
| _______ | _______ | _______ |

## Loop History

Tracks all pipeline loops (A→B→A→B, B-unit retries, gate failures).

| Loop | From Step | To Step | Reason | Timestamp |
|------|-----------|---------|--------|-----------|
| _______ | _______ | _______ | _______ | _______ |

## Correction Records

Produced by the `post-rejection-correction` skill. One record per retry. Required for every gate failure before the retry is dispatched. Permanent — must not be edited after stamping.

| Retry | Gate | Tier | RC Category | Root cause (why missed) | Corrective action | Codified where |
|-------|------|------|-------------|------------------------|-------------------|----------------|
| _______ | _______ | _______ | _______ | _______ | _______ | _______ |
```

---

## Mandatory Rules

**These rules are non-negotiable. They are enforced by the Supreme Leader's Pre-Dispatch Gate (see `skills/core/pipeline/SKILL.md § Pipeline Enforcement Protocol`). Any agent receiving a dispatch without a valid passport MUST return `STATUS: BLOCKED`. No work proceeds without a passport.**

### Rule 1: No Step Without a Stamp

Every step in the Required Steps section must be checked off with a timestamp and result before the next step can begin. An agent receiving a task where a previous step is un-checked must:

1. **STOP** — do not proceed.
2. **Return STATUS: BLOCKED** with the message: `Pipeline passport step <step-id> is not completed. Previous step must be stamped before proceeding.`
3. **Report to the Supreme Leader** — the dispatcher decides whether to route to the missing step's agent or to request a justified skip.

### Rule 2: No Skip Without Justification

A step may be skipped ONLY if:

1. A written justification is provided in the Skipped Steps section.
2. The justification includes: what was skipped, why it's safe to skip, and who authorised the skip.
3. The Supreme Leader approves the skip before dispatching to the next step.

Example justifications:
- "A2: Dual-Model Challenge skipped for trivial refactor (rename only). No architectural change, no protocol change — Dual-Model Challenge is not required for rename-only tasks per pipeline SKILL.md §Dual-Model Challenge."
- "B2a-3: B-UNIT-GATE for unit 3 skipped — unit 3 is a documentation-only change (Doxygen comments) with no functional code. T-ARCH review covered in B2a-2 which included the same file."

### Rule 3: Loop Tracking

When the pipeline loops (e.g., A-GATE fails and loops back to A1, or C-GATE fails and routes to B2), the Loop History section MUST be updated with:

- The step being looped from
- The step being looped to
- The reason (e.g., "A-GATE T3 FAIL: hardware-engineer REJECTED — register bit position incorrect")
- The timestamp

This ensures that loops are visible and auditable. An infinite loop is prevented by the 3-retry budget per tier per gate.

### Rule 4: Passport Travels With Dispatch

The passport file path MUST be included in every dispatch envelope:

```yaml
ticket: "<ticket-id>"
passport: "docs/project-management/passports/<ticket-id>-passport.md"
phase: "<A|B|C>"
step: "<A0|A1|...>"
# ... rest of envelope fields ...
```

### Rule 5: Agent Responsibilities

| Agent | Responsibility |
|-------|---------------|
| **PM** | Creates the passport when opening a ticket. Fills in Task Identity and Required Steps. |
| **Supreme Leader** | Stamps the passport with gate results. Updates Loop History on retries. Authorises skips. |
| **Specialist Agent** | Stamps the passport with their step result before returning to Supreme Leader. |
| **Code Architect** | Stamps B-step results (B1, B2, B3) and unit-gate results. Updates Required Steps with actual unit count after B1. |
| **Any Agent** | Must check that previous steps are stamped before starting work. If missing, return STATUS: BLOCKED. |

### Rule 6: Missing Previous Step Detection

Every agent MUST check the passport before starting work:

```
1. Read passport file from dispatch envelope.
2. Find the step you are responsible for in Required Steps.
3. Check that ALL previous steps are checked off (✅) or listed in Skipped Steps with justification.
4. If any previous step is missing:
   a. Do NOT start work.
   b. Return STATUS: BLOCKED with message: "Pipeline passport step <missing-step-id> not completed."
   c. Supreme Leader will route to the appropriate agent to complete the missing step.
5. If all previous steps are stamped, proceed with work.
6. After completing work, stamp your step with result, timestamp, and notes.
7. Return passport with the dispatch envelope.
```

---

## How Loops Are Tracked

When the pipeline loops (e.g., A-GATE fails → back to A1), the passport records this in Loop History:

```
| Loop 1 | A3 (A-GATE) | A1 (Review) | T3 FAIL: hardware-engineer REJECTED — register bit position wrong | 2026-06-07T14:30Z |
| Loop 2 | A3 (A-GATE) | A1 (Review) | T-ARCH FAIL: logical inconsistency in state machine | 2026-06-07T15:00Z |
```

The Required Steps section accumulates multiple check marks for looped steps:

```
- [x] A1: Specialist Review (attempt 1) — CONDITIONAL PASS
- [x] A1: Specialist Review (attempt 2) — APPROVED
- [x] A2: Dual-Model Challenge (attempt 1) — contradictions found, requires synthesis
- [x] A2: Dual-Model Challenge (attempt 2) — synthesis complete
- [ ] A3: A-GATE — T3: ___ | T-ARCH: ___ | Verdict: _______
```

---

## Self-Reflection Clause

After any passport violation (missing step, unjustified skip, stale passport), the responsible agent MUST ask:

1. **Why was this violation not caught earlier?** — Was the passport check not performed, or was the check insufficient?
2. **What procedural safeguard would have caught it?** — Would a stricter check, an automated gate, or a different passport format have caught it?
3. **Update the knowledge base** — Add the lesson to this skill or the pipeline skill.