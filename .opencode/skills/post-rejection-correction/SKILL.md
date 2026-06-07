---
name: post-rejection-correction
description: "Mandatory root-cause correction protocol. Triggered when an agent's output is rejected, receives CONDITIONAL PASS requiring rework, or the user had to prompt twice because the agent failed to gather enough requirements. The producing agent MUST classify the failure into a root-cause category and produce a Correction Record before any retry is dispatched."
---

# Post-Rejection Correction Protocol

## Purpose

Retry loops without learning just repeat the same class of mistake. When an agent's output is rejected or the user has to provide information the agent should have gathered, the producing agent MUST perform a structured root-cause analysis before retrying. The analysis is logged as a **Correction Record** in the passport — visible to all subsequent reviewers.

Fixing the symptom is necessary. Understanding why it was missed is mandatory.

## When to Trigger

This protocol is **mandatory** in three situations:

| Trigger | Signal | Who runs the analysis |
|---------|--------|-----------------------|
| **Gate REJECTED** | Any specialist issues REJECTED at A-GATE or C-GATE | The agent(s) whose output was rejected |
| **Gate CONDITIONAL PASS (rework required)** | Any specialist issues CONDITIONAL PASS where rework is required to proceed | The agent(s) responsible for the rework |
| **User prompted twice** | User provides clarification or requirements that the agent should have asked for during Phase A | The agent who ran Phase A (Supreme Leader / PM) |

### The User-Prompted-Twice Signal

If the user provides a follow-up that corrects or supplements something the agent should have caught or asked about during requirements gathering, that is a Phase A quality failure — the agent did not apply `assumption-trap` correctly or asked an insufficient range of questions.

**The user should never need to volunteer requirements. The agent must ask.**

This trigger maps to RC-2 (Missing Question Category) and sometimes RC-1 (Unverified Assumption). Run the full correction protocol before continuing with the task.

---

## The Five Root-Cause Categories

For each rejection finding, identify which category applies. More than one may apply.

### RC-1: Unverified Assumption

The agent assumed a value, behaviour, or constraint without verifying it against the source of truth.

- **Signal:** The reviewer found something that contradicts an assumption the agent made and didn't flag as a gap.
- **Corrective action:** Add the specific assumption class to the pre-flight checklist in `assumption-trap` or the relevant agent definition. Load `assumption-trap` at the earliest dispatch step on retry.
- **Passport action:** Record the specific assumption that was wrong and what the correct value is.

### RC-2: Missing Question Category

The agent didn't ask the right category of question — either during requirements gathering (Phase A) or during review.

- **Signal:** The rejection finding covers a whole category of concern the agent never considered (e.g., timing constraints, reserved bit behaviour, error paths, edge cases at boundary conditions).
- **User-prompted-twice maps here:** The follow-up question reveals a category the agent should have covered during Phase A.
- **Corrective action:** Add the missing question category to the relevant skill (`grill-me` question categories, `self-audit-checklist` rows, domain skill checklist, or the agent's own "What to Check" section).
- **Passport action:** Record the category missed and the exact skill file and section where it has been added.

### RC-3: Unknown Fault Pattern

The agent didn't recognise a fault pattern that was present in the output — the anti-pattern existed but wasn't on the agent's radar.

- **Signal:** The reviewer identified a known anti-pattern (e.g., raw integer in public API, reserved bits not masked, unbounded copy, magic number in doc example) that the agent's self-audit failed to catch.
- **Corrective action:** Add the fault pattern to the relevant checklist — a new T1 check row in `compliance-gate`, a row in `self-audit-checklist`, or a "Common Faults" entry in the domain skill. The pattern must be codified so it's caught mechanically next time.
- **Passport action:** Record the pattern and where it has been codified.

### RC-4: Missing Skill / Domain Expertise

The agent lacked the domain knowledge required to produce a correct output.

- **Signal:** The rejection requires specialist knowledge the agent did not apply (e.g., a datasheet register table, protocol spec section, memory safety rule, security constraint).
- **Corrective action:** Add the required skill as a mandatory load in the Agent Routing Table in `pipeline` SKILL.md for this intent type. Update the dispatch envelope's `skills_loaded` for the retry immediately.
- **Passport action:** Record the missing skill and the dispatch rule that has been updated.
- **Supreme Leader action (immediate):** Update `skills_loaded` in the retry dispatch envelope before sending. Do not wait for the next task.

### RC-5: Wrong Review Scope

The agent reviewed the wrong thing — wrong files, wrong layer, wrong tier, or a subset of what the gate required.

- **Signal:** The reviewer found issues in areas the agent didn't examine; the gate scope defined in `compliance-gate` was not fully covered.
- **Corrective action:** Before the retry, explicitly re-read the gate scope from `compliance-gate` and confirm every file, layer, and check category is included. Update the agent's pre-work checklist if the scope confusion was structural.
- **Passport action:** Record what scope was missed and what the correct scope is per `compliance-gate`.

---

## Correction Record Format

Before any retry is dispatched, the producing agent MUST output a Correction Record in this exact format and append it to the passport's `## Correction Records` section.

```markdown
## Correction Record — <gate> Retry <n>/<budget>

| Field | Value |
|-------|-------|
| Gate | [A-GATE / B-UNIT-GATE / B-FINAL-GATE / C-GATE / User-Prompted-Twice] |
| Attempt | [n of 3] |
| Tier failed | [T1 / T2 / T3 / T-ARCH / Phase-A] |
| Producing agent | [agent role] |

### Rejection Findings and Root Causes

| Finding (from violation report) | Why the agent missed it | RC category |
|---------------------------------|------------------------|-------------|
| [finding 1] | [specific reason] | [RC-1 to RC-5] |
| [finding 2] | [specific reason] | [RC-1 to RC-5] |

### Corrective Actions

| RC category | Action taken | Where codified |
|-------------|-------------|----------------|
| RC-N | [what was changed] | [skill file, section, or checklist row] |

### Pre-Retry Confirmation

- [ ] Correction Record appended to passport
- [ ] Missing skills added to `skills_loaded` in retry dispatch envelope (if RC-4)
- [ ] Missing question category added to relevant skill (if RC-2)
- [ ] Fault pattern added to relevant checklist (if RC-3)
- [ ] All rejection findings are addressed — root cause, not just symptom
```

---

## Protocol Steps

1. **Receive** rejection / conditional-pass / user-prompted-twice signal.
2. **Read** the rejection findings from the Gate Violation Report or user prompt.
3. **For each finding**, classify into one or more RC categories (RC-1 through RC-5).
4. **For each RC category**, take the corrective action defined above.
5. **Produce a Correction Record** using the format above.
6. **Append** the Correction Record to the passport's `## Correction Records` section.
7. **If RC-4:** Notify Supreme Leader to update `skills_loaded` for the retry dispatch — this happens *before* the retry is sent.
8. **Only then** proceed with the retry.

---

## Rules

1. **No retry without a Correction Record.** An agent that retries without producing a Correction Record is in protocol violation. The Supreme Leader must block the retry and require the analysis first.
2. **Correct the root cause, not just the symptom.** Fixing the specific line that was rejected is necessary but not sufficient. The Correction Record must explain why the agent missed it and how the process is improved so it won't be missed again.
3. **Correction Records are permanent.** They are part of the passport audit trail and must not be deleted or edited after being stamped, even after the gate passes.
4. **User-prompted-twice requires RC-2 analysis.** The agent must identify the specific question or question category that should have been asked in Phase A, and add it to the relevant skill or checklist before continuing.
5. **RC-4 triggers immediate skill loading.** Missing skills must be added to the retry dispatch envelope before the retry is sent, not deferred to a future task.
6. **Phase C reviewers read the Correction Records.** If an agent repeated a class of failure across multiple retries without updating their process, Phase C reviewers should flag this as a structural quality failure.

---

## Self-Reflection Clause

After completing a Correction Record:

1. **Why was the root cause not caught before the output was submitted?** — What self-audit step was skipped or insufficient?
2. **Is this a one-off or a pattern?** — Does this RC-2 or RC-3 finding point to a whole category of questions or fault patterns that should be systematically added to core skills?
3. **Update the knowledge base** — Ensure every corrective action is codified in the right skill so the same class of failure is caught earlier on future tasks.
