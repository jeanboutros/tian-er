---
name: pipeline
description: "The agent-facing state machine for the PSC validation pipeline. Defines phases, gates, state transitions, agent routing, dispatch envelope format, and skill loading rules. All agents must follow this state machine."
---

# Pipeline State Machine

## Purpose

This skill defines the complete pipeline workflow that all agents must follow. It replaces `docs/pipeline/agents.md` as the agent-facing state machine document — agents read this skill, not the docs file, for workflow rules.

## When to Trigger

- **Always loaded** for all agents as part of the core skill set.
- **Additionally triggered** when an agent needs to determine what phase it's in, what gate to run, or how to route work.

---

## Phase Definitions

### Phase A — Requirements & Design

**Goal:** Define and validate "What" and "How" before writing code.

**Sub-steps:**

| Step | Name | Description | Who |
|------|------|-------------|-----|
| A0 | Task Definition | Produce detailed task specification: acceptance criteria, files, constraints, test strategy, doc plan | All agents collaborate |
| A1 | Parallel Specialist Review | All 6 specialists review the proposal independently | SW Engineer, HW Engineer, Wireless Expert, Security Reviewer, Test Engineer, Docs Writer |
| A2 | Dual-Model Challenge | Two model passes review architecture: primary produces, challenger critiques | Supreme Leader orchestrates |
| A3 | A-GATE | T3 + T-ARCH compliance check | All 6 specialists (T3), SW Engineer (T-ARCH) |

**A-GATE pass criteria:** All 6 specialists issue APPROVED or CONDITIONAL PASS + T-ARCH passes.
**A-GATE fail:** Any REJECTED → producing agent runs `post-rejection-correction` protocol first, then loop back to A1 with specific critique (max 3 loops at T3).

**User-prompted-twice signal:** If the user provides clarification or requirements during Phase A that the agent should have asked for, this is a Phase A quality failure. The agent did not apply `assumption-trap` correctly or asked an insufficient range of questions. Treat it the same as a gate rejection: run the `post-rejection-correction` protocol (maps to RC-2: Missing Question Category) before continuing. The user should never need to volunteer requirements — the agent must ask.

### Phase B — Build (PAU Loop)

**Goal:** Implement incrementally with self-validation, enforced by compliance gates.

**Sub-steps:**

| Step | Name | Description | Who |
|------|------|-------------|-----|
| B1 | PLAN | Read task, identify files, list acceptance criteria, declare logical units | Code Architect |
| B2 | APPLY (per unit) | Implement one logical unit, run build | Code Architect |
| B2a | B-UNIT-GATE | T1 + T-ARCH compliance check after each unit | Code Architect (T1), SW Engineer (T-ARCH) |
| B3 | VALIDATE | Full build, optional flash | Code Architect |
| B3a | B-FINAL-GATE | T1 + T2 + T-ARCH compliance check after all units | Code Architect (T1), SW Engineer (T2 + T-ARCH) |

**B-UNIT-GATE pass criteria:** All 9 T1 checks pass + T-ARCH passes.
**B-FINAL-GATE pass criteria:** T1 passes + T2 passes + T-ARCH passes.
**Failure routing:** T1 → Code Architect fixes; T2 → Code Architect + Software Engineer input; T-ARCH → Software Engineer.

### Phase C — Multi-Agent Verify

**Goal:** Final check before commit. ALL specialist agents must approve.

**Sub-steps:**

| Step | Name | Description | Who |
|------|------|-------------|-----|
| C0 | T1 Re-run | Mechanical compliance re-check on final codebase | Code Architect |
| C1 | Dual-Model Challenge (Verification) | Primary verifier + challenger verifier | Supreme Leader orchestrates |
| C2 | Parallel Specialist Approval | All 6 specialists review independently | All 6 specialists |
| C3 | C-GATE | T1 re-run + T3 + T-ARCH | Code Architect (T1), Specialists (T3), SW Engineer (T-ARCH) |

**C-GATE pass criteria:** T1 passes + all 6 APPROVED + T-ARCH passes.

---

## State Machine

### Complete State Transition Diagram

```
                                    ┌───────────────────────────────────────────────┐
                                    │                                               │
                                    ▼                                               │
┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐│
│  A0:Task │───▶│A1:Review│───▶│A2:Dual │───▶│A3:A-GATE│───▶│ B1:PLAN │───▶│B2:APPLY ││
│  Def     │    │Parallel │    │Challenge│   │T3+T-ARCH│    │         │    │ (unit)  ││
└─────────┘    └─────────┘    └─────────┘    └────┬────┘    └─────────┘    └────┬────┘│
                                                     │                              │     │
                                                     │ FAIL (3× T3 or T-ARCH)        │     │
                                                     │ ┌───────────────────────────┘  │     │
                                                     │ │  PASS                         │     │
                                                     ▼ ▼                              ▼     │
                                               ┌──────────┐                    ┌──────────┐ │
                                               │A1:Review │◀──── 3×T3 ────   │B2a:UNIT  │ │
                                               │(loop back│                   │GATE      │ │
                                               │ with cri-│                   │T1+T-ARCH │ │
                                               │ tique)    │                   └────┬─────┘ │
                                               └──────────┘                        │       │
                                                                                   │       │
                                                         PASS ────────────────────┘       │
                                                                                           │
                                                                                   ┌───────▼──────┐
                                                                                   │More units?   │
                                                                                   └──┬────────┬─┘
                                                                                      │YES     │NO
                                                                                      │        │
                                                                                      │        ▼
                                                                                      │  ┌──────────┐
                                                                                      │  │B3a:FINAL │
                                                                                      │  │GATE      │
                                                                                      │  │T1+T2+ARCH│
                                                                                      │  └────┬─────┘
                                                                                      │       │
                                                                                      │  FAIL (3× any tier)
                                                                                      │  ┌─────│─────┐
                                                                                      │  │ LOOP BACK │
                                                                                      │  └─────│─────┘
                                                                                      │       │ PASS
                                                                                      │       ▼
                                                                                      │  ┌──────────┐
                                                                                      │  │C0:T1 re-  │
                                                                                      │  │run        │
                                                                                      │  └────┬─────┘
                                                                                      │       │
                                                                                      │       ▼
                                                                                      │  ┌──────────┐
                                                                                      │  │C1:Dual   │
                                                                                      │  │Challenge │
                                                                                      │  │(Verify)  │
                                                                                      │  └────┬─────┘
                                                                                      │       │
                                                                                      │       ▼
                                                                                      │  ┌──────────┐
                                                                                      │  │C2:Special-│
                                                                                      │  │ist Appro-│
                                                                                      │  │val (T3)  │
                                                                                      │  └────┬─────┘
                                                                                      │       │
                                                                                      │       ▼
                                                                                      │  ┌──────────┐
                                                                                      │  │C3:C-GATE │
                                                                                      │  │T1+T3+ARCH│
                                                                                      │  └────┬─────┘
                                                                                      │       │
                                                                                      │  FAIL (3× any tier)
                                                                                      │  ┌─────│─────┐
                                                                                      │  │ LOOP BACK│
                                                                                      │  └─────│─────┘
                                                                                      │       │ PASS
                                                                                      │       ▼
                                                                                      │  ┌──────────┐
                                                                                      │  │ COMMIT   │
                                                                                      │  └──────────┘
```

### State Transition Table

| From State | Event | To State | Condition |
|-----------|-------|----------|-----------|
| A0 | Task defined | A1 | All agents have task spec |
| A1 | Reviews complete | A2 | All 6 specialists reviewed |
| A2 | Challenge complete | A3 | Synthesis produced |
| A3 | A-GATE passes | B1 | All specialists APPROVED/CONDITIONAL PASS + T-ARCH passes |
| A3 | A-GATE fails | A1 | REJECTED or T-ARCH fail; loop back with critique (max 3×) |
| B1 | Plan complete | B2 | Logical units identified |
| B2 | Unit implemented | B2a | Build passes locally |
| B2a | B-UNIT-GATE passes | B2 (next unit) | T1 + T-ARCH pass |
| B2a | B-UNIT-GATE fails | B2 (fix) | T1 or T-ARCH fail; fix and retry (max 3× per tier) |
| B2 | All units done | B3a | All units pass B-UNIT-GATE |
| B3a | B-FINAL-GATE passes | C0 | T1 + T2 + T-ARCH pass |
| B3a | B-FINAL-GATE fails | B2 (fix) | Any tier fails; route to appropriate fixer (max 3× per tier) |
| C0 | T1 re-run passes | C1 | All T1 checks pass |
| C0 | T1 re-run fails | B2 (fix) | Code Architect fixes; re-run T1 (max 3×) |
| C1 | Challenge complete | C2 | Synthesis produced |
| C2 | Reviews complete | C3 | All 6 specialists reviewed |
| C3 | C-GATE passes | COMMIT | All APPROVED + T1 pass + T-ARCH pass |
| C3 | C-GATE fails | C0 or C2 or B2 | T1 fail → C0 (Code Architect fixes, re-run T1); T3 fail → C2 (specialist re-review); T-ARCH fail → Software Engineer (architectural fix) |
| Any | 3 retries exhausted at any tier | ESCALATE | Supreme Leader presents full violation report to user |

> **All FAIL transitions require a Correction Record before retry.** For every row above where the event is a gate or check failure, the producing agent MUST complete the `post-rejection-correction` protocol and stamp a Correction Record in the passport before the Supreme Leader dispatches the retry. The Supreme Leader must verify the Correction Record is present; if it is missing, block the dispatch and route to the producing agent first.

---

## Pipeline Enforcement Protocol

The pipeline is **not advisory**. It is **mandatory**. The Supreme Leader MUST enforce these rules before every dispatch. No exceptions.

### Pre-Dispatch Gate (Non-Skippable)

Before the Supreme Leader classifies intent or routes to any agent, it MUST execute this gated sequence:

| Gate Step | Check | Failure Action |
|-----------|-------|----------------|
| **PM Gate** | If new task: passport must be created by PM before any routing. Supreme Leader dispatches to `@pm` with `trigger: "create-passport"` and waits. | BLOCKED — no routing until passport exists. |
| **Passport Exists** | Passport file at `docs/project-management/passports/<ticket-id>-passport.md` exists on disk. | BLOCKED — dispatch to PM for passport creation. |
| **Prior Steps Stamped** | All steps before target step have timestamps and results in Step Log. | BLOCKED — route to missing step's agent. |
| **Gate Results Recorded** | If at a gate, Gate Results table has current attempt entries. | BLOCKED — run gate first. |
| **Skips Justified** | Any unchecked Required Step has corresponding Skipped Steps entry with authorisation. | BLOCKED — require authorisation. |
| **Correction Records** | If retry_count > 0 for any tier, Correction Record exists in passport. | BLOCKED — dispatch to producing agent for post-rejection-correction. |

### Role Separation Rules

| Rule | Enforcement |
|------|-------------|
| Only PM creates passports | Supreme Leader MUST NOT create passport files. If none exists, dispatch to PM and wait. |
| Only PM creates tickets | Supreme Leader MUST NOT create task entries in TODO.md or ticket files. |
| Supreme Leader is dispatch-only | Supreme Leader MUST NOT perform specialist work. If a specialist fails, report to user — do not fill in. |
| No combined PM + Supreme Leader | These roles operate at different steps. The envelope must go PM → Supreme Leader, never both at once. |

### Status Protocol

When any enforcement check fails:

```
STATUS: BLOCKED
Reason: <which check failed and why>
Action Required: <what must happen to unblock>
```

The Supreme Leader must return this to the user immediately. Do NOT proceed to routing. Do NOT attempt to self-resolve.

---

## Dispatch Envelope Format

Every agent dispatch carries a structured envelope. This ensures context is preserved across handoffs.

```yaml
ticket: "<task-id>"
phase: "<A|B|C>"
step: "<A0|A1|A2|A3|B1|B2|B2a|B3|B3a|C0|C1|C2|C3>"
trigger: "<reason for this dispatch>"
agent: "<agent-role>"
passport: "docs/project-management/passports/<ticket-id>-passport.md"
skills_loaded:
  - "assumption-trap"
  - "compliance-gate"
  - "pipeline"
  - "pau-loop"
  - "<domain-specific-skills>"
  # On retry dispatches, also load:
  # - "post-rejection-correction"  ← required when retry_count > 0 on any tier
expected_outcomes:
  - "<specific deliverable 1>"
  - "<specific deliverable 2>"
next_agent: "<agent-role or 'user' for escalation>"
retry_count:
  T1: <number>
  T2: <number>
  T3: <number>
  T-ARCH: <number>
OWASP_expansion: "<none | list of added compliance categories>"
```

### Field Definitions

| Field | Description |
|-------|-------------|
| `ticket` | Unique task identifier from `docs/pipeline/TODO.md` |
| `phase` | Current pipeline phase (A, B, or C) |
| `step` | Current step within the phase |
| `trigger` | Why this dispatch occurred (e.g. "A-GATE failed: T3.1 datasheet fidelity") |
| `agent` | The agent being dispatched to |
| `passport` | Path to the pipeline passport file tracking completed steps for this task |
| `skills_loaded` | List of skills loaded for this dispatch (always includes core skills) |
| `expected_outcomes` | Concrete, verifiable deliverables expected |
| `next_agent` | Who receives the output next |
| `retry_count` | Current retry count for each tier at the current gate |
| `OWASP_expansion` | Any OWASP compliance categories added for this task |

---

## Agent Routing Table

Which agent handles which intent:

| Intent | Agent | Skills to Load |
|--------|-------|---------------|
| Architecture design | Software Engineer | assumption-trap, compliance-gate, type-design-review |
| Register model design | Hardware Engineer | assumption-trap, datasheet-verification, domain |
| RF protocol design | Wireless Expert | assumption-trap, datasheet-verification, domain |
| Security analysis | Security Reviewer | assumption-trap, silent-failure, memory-safety |
| Test strategy | Test Engineer | assumption-trap, test-driven-development, tdd-cpp (C++ projects) |
| Documentation plan | Docs Writer | assumption-trap, verification-before-completion |
| Implementation | Code Architect | pau-loop, incremental-execution, compliance-gate |
| T1 compliance check | Code Architect | compliance-gate, verification-before-completion |
| T2 architectural review | Software Engineer | compliance-gate, type-design-review |
| T3 semantic review | All 6 specialists | compliance-gate, domain-specific skills |
| T-ARCH review | Software Engineer | compliance-gate, type-design-review |
| Memory safety review | Memory Safety | assumption-trap, memory-safety |
| Gate orchestration | Supreme Leader | pipeline, compliance-gate, flag-protocol |
| Dispatch/routing only | Supreme Leader | pipeline, flag-protocol |
| Task creation | PM | pipeline, flag-protocol |
| Debugging | Code Architect | systematic-debugging, domain |
| Product vision / requirements discovery | Product Designer | assumption-trap, design-taste, ux-patterns |
| Interaction design / UX review | UX Engineer | assumption-trap, ux-patterns, design-taste |
| UI implementation | UI Engineer | pau-loop, incremental-execution, design-taste, ux-patterns |

---

## Dual-Model Challenge Protocol

Used in **Phase A** (architecture) and **Phase C** (verification).

### How It Works

1. **Primary pass** — First model produces the output (architecture proposal or verification).
2. **Challenger pass** — Second model independently reviews, looking for:
   - Contradictions with datasheet/spec
   - Missed edge cases
   - Unsupported assumptions
   - Security gaps
   - Protocol non-compliance
   - T-ARCH violations (logical errors, structural issues, principle misalignment)
3. **Synthesis** — Supreme Leader merges findings:
   - Agreements → accepted
   - Contradictions → presented to user for decision
   - One-sided findings → accepted if well-evidenced, otherwise flagged

### When to Invoke Dual-Model Challenge

| Scenario | Use Dual-Model? |
|----------|-----------------|
| New register implementation | Yes |
| New protocol feature (whitening, CRC, etc.) | Yes |
| HAL interface change | Yes |
| Architecture change | Yes |
| Bug fix in existing code | No (single pass sufficient) |
| Documentation-only change | No |
| Trivial refactor (rename, move) | No |

---

## How to Read AGENTS.md and Load Domain Skills

### Tech Stack Reference

Before starting any task, read `AGENTS.md` for:
- **Multi-Agent Validation Pipeline** (Mandatory) — the 3-phase pipeline summary
- **Key Rules** — no-assumption, PAU loop, quality gate, incremental execution, flag protocol, compliance gates
- **Documentation Rules** — doc format requirements for the project
- **Design Principles** — typed vocabulary, module boundaries, structural rules
- **Domain-Specific Traps** — critical known pitfalls for the project's tech stack
- **Knowledge Management Rules** — learning doc creation requirements

### Skill Loading Rules

1. **Always loaded (core skills):** assumption-trap, compliance-gate, pipeline, pau-loop, verification-before-completion, self-audit-checklist, review-confidence, type-design-review, silent-failure
2. **Domain skills (load based on task):** any domain skills listed in AGENTS.md for the project's tech stack; datasheet-verification, memory-safety, systematic-debugging, test-driven-development as needed
3. **Phase skills (load based on phase):** brainstorming (Phase A), incremental-execution (Phase B), grill-me (Phase A or C dual-model challenge)
4. **Compliance expansion (load based on OWASP triggers):** Review task for new concern categories and load additional compliance checks as needed

### Mandatory Skill Loading Order

When a task is dispatched, skills must be loaded in this order:
1. `assumption-trap` — FIRST, always
2. `compliance-gate` — tiered checks, OWASP expansion
3. `pipeline` — this skill, state machine
4. `pau-loop` — for Phase B work
5. Domain-specific skills as needed

---

## Self-Reflection Clause

After any pipeline violation or gate failure, the responsible agent MUST ask:

1. **Why was this not caught earlier?** — What review, test, or protocol gap allowed it through?
2. **What procedural safeguard would have caught it?** — What specific check, test, or verification step would have prevented it?
3. **Update the knowledge base** — Add the lesson to the relevant skill or learning doc.

Violations in the pipeline process itself (wrong routing, missed gate, skipped step) should be logged as flags and added to the pipeline skill's lessons learned.
