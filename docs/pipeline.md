# Tian'er Pipeline — Full Specification

> This is the deep dive. For the overview, see [../README.md](../README.md).

## The Standing Committee Model

PSC draws its execution philosophy from the efficiency of standing committees: small groups with clear mandates, rapid decision-making, and enforced accountability. Every agent has exactly one job. Every gate passes or fails objectively. No infinite review cycles, no scope creep, no "looks good to me" rubber stamps.

The Supreme Leader is the orchestrator — it dispatches work, enforces protocol, and escalates when retries are exhausted. It never writes code, never designs, never decides technical questions. Those decisions belong to the specialist closest to the problem.

---

## Phase A — Requirements & Design

**Goal:** Define and validate "What" and "How" before writing a single line of code.

### Sub-steps

| Step | Name | Description | Who |
|------|------|-------------|-----|
| A0 | Task Definition | Produce detailed task specification: acceptance criteria, files, constraints, test strategy, doc plan | All agents collaborate |
| A1 | Parallel Specialist Review | All 6 specialists review the proposal independently | SW Engineer, HW Engineer, Wireless Expert, Security Reviewer, Test Engineer, Docs Writer |
| A2 | Dual-Model Challenge | Two model passes review architecture: primary produces, challenger critiques | Supreme Leader orchestrates |
| A3 | A-GATE | T3 + T-ARCH compliance check | All 6 specialists (T3), SW Engineer (T-ARCH) |

### A-GATE Pass Criteria

- All 6 specialists issue **APPROVED** or **CONDITIONAL PASS**
- T-ARCH passes
- On fail: loop back to A1 with specific critique (max 3 loops per tier)

---

## Phase B — Build (PAU Loop)

**Goal:** Implement incrementally with self-validation, enforced by compliance gates.

### Sub-steps

| Step | Name | Description | Who |
|------|------|-------------|-----|
| B1 | PLAN | Read task, identify files, list acceptance criteria, declare logical units | Code Architect |
| B2 | APPLY (per unit) | Implement one logical unit, run build verification | Code Architect |
| B2a | B-UNIT-GATE | T1 + T-ARCH compliance check after each unit | Code Architect (T1), SW Engineer (T-ARCH) |
| B3 | VALIDATE | Full build, optional flash test | Code Architect |
| B3a | B-FINAL-GATE | T1 + T2 + T-ARCH compliance check after all units | Code Architect (T1), SW Engineer (T2 + T-ARCH) |

### B-UNIT-GATE Pass Criteria

- All 9 T1 checks pass + T-ARCH passes
- On fail: fix and retry (max 3x per tier)

### B-FINAL-GATE Pass Criteria

- T1 passes + T2 passes + T-ARCH passes
- On fail: route to appropriate fixer (max 3x per tier)

### The PAU Loop

Each unit follows **Plan -> Apply -> Validate**:

1. **Plan** -- identify the unit, declare what changes are needed, list acceptance criteria
2. **Apply** -- implement the changes, run build verification
3. **Validate** -- run T1 checks, verify acceptance criteria, move to next unit or gate

---

## Phase C — Multi-Agent Verify

**Goal:** Final check before commit. ALL specialist agents must approve.

### Sub-steps

| Step | Name | Description | Who |
|------|------|-------------|-----|
| C0 | T1 Re-run | Mechanical compliance re-check on final codebase | Code Architect |
| C1 | Dual-Model Challenge (Verification) | Primary verifier + challenger verifier | Supreme Leader orchestrates |
| C2 | Parallel Specialist Approval | All 6 specialists review independently | All 6 specialists |
| C3 | C-GATE | T1 re-run + T3 + T-ARCH | Code Architect (T1), Specialists (T3), SW Engineer (T-ARCH) |

### C-GATE Pass Criteria

- T1 passes + all 6 APPROVED + T-ARCH passes

---

## Compliance Tiers

Every gate checks one or more compliance tiers. Each tier has an **independent retry budget of 3**.

### T1 -- Mechanical (Automated)

| # | Check | Criterion |
|---|-------|-----------|
| 1 | Build passes | Project build command exits 0 |
| 2 | No compiler warnings | `-Werror` is active; any warning is a failure |
| 3 | Doxygen on all public API | Every public function/class/struct has `/** ... */` Doxygen |
| 4 | No decision references in code | No `D-1:`, no "replaces the former..." |
| 5 | No raw integers in public API | Finite-value fields use `enum class`, not `uint8_t` |
| 6 | Reserved bits written as 0 | Register writes clear reserved bits |
| 7 | File placement | Library code in `components/`, app code in `main/` |
| 8 | Platform independence | Library headers include only `<cstdint>`, `<cstring>`, and own headers |
| 9 | No hardcoded secrets | No passwords, API keys, tokens in source files |

### T2 -- Architectural (Software Engineer)

| # | Check | Criterion |
|---|-------|-----------|
| 1 | Platform boundary | All hardware access through `Hal` interface |
| 2 | Namespace hygiene | Clean hierarchy, no pollution |
| 3 | Typed enums | Every field with finite legal values uses `enum class` |
| 4 | No mutable globals | Stateful singletons forbidden |
| 5 | Build dependencies | Component dependencies correct and minimal |

### T3 -- Semantic (All 6 Specialists)

All 6 specialist agents must issue APPROVED or CONDITIONAL PASS:
- Software Engineer -- architecture, API surface, SOLID
- Hardware Engineer -- datasheet fidelity, register correctness, timing
- Wireless Expert -- protocol compliance, channel mapping, modulation
- Security Reviewer -- attack surfaces, buffer safety, secrets handling
- Test Engineer -- test coverage, edge cases, static assertions
- Docs Writer -- documentation completeness, reference accuracy

### T-ARCH -- Structural & Principles

| # | Check | Criterion |
|---|-------|-----------|
| 1 | Logical consistency | No contradictions within the design |
| 2 | Structural soundness | No circular dependencies, clean layering |
| 3 | Principle alignment | Follows project principles (typed API, RAII, etc.) |
| 4 | Completeness | All requirements covered, no orphaned code |

---

## Compliance Gate State Machine

```
                                    ┌───────────────────────────────────────────────┐
                                    |                                               |
                                    v                                               |
┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐|
│  A0:Task │───>│A1:Review│───>│A2:Dual │───>│A3:A-GATE│───>│ B1:PLAN │───>│B2:APPLY │|
│  Def     │    │Parallel │    │Challenge│   │T3+T-ARCH│    │         │    │ (unit)  │|
└─────────┘    └─────────┘    └─────────┘    └────┬────┘    └─────────┘    └────┬────┘|
                                                     │                              │     │
                                                     │ FAIL (3x T3 or T-ARCH)        │     │
                                                     │ ┌───────────────────────────┘  │     │
                                                     │ │  PASS                         │     │
                                                     v v                              v     │
                                               ┌──────────┐                    ┌──────────┐ │
                                               │A1:Review │<──── 3xT3 ────   │B2a:UNIT  │ │
                                               │(loop back│                   │GATE      │ │
                                               │ with cri-│                   │T1+T-ARCH │ │
                                               │ tique)    │                   └────┬─────┘ │
                                               └──────────┘                        │       │
                                                                                   │       │
                                                         PASS ────────────────────┘       │
                                                                                           │
                                                                                   ┌───────v──────┐
                                                                                   │More units?   │
                                                                                   └──┬────────┬─┘
                                                                                      │YES     │NO
                                                                                      │        │
                                                                                      │        v
                                                                                      │  ┌──────────┐
                                                                                      │  │B3a:FINAL │
                                                                                      │  │GATE      │
                                                                                      │  │T1+T2+ARCH│
                                                                                      │  └────┬─────┘
                                                                                      │       │
                                                                                      │  FAIL (3x any tier)
                                                                                      │  ┌─────│─────┐
                                                                                      │  │ LOOP BACK │
                                                                                      │  └─────│─────┘
                                                                                      │       │ PASS
                                                                                      │       v
                                                                                      │  ┌──────────┐
                                                                                      │  │C0:T1 re-  │
                                                                                      │  │run        │
                                                                                      │  └────┬─────┘
                                                                                      │       │
                                                                                      │       v
                                                                                      │  ┌──────────┐
                                                                                      │  │C1:Dual   │
                                                                                      │  │Challenge │
                                                                                      │  │(Verify)  │
                                                                                      │  └────┬─────┘
                                                                                      │       │
                                                                                      │       v
                                                                                      │  ┌──────────┐
                                                                                      │  │C2:Special-│
                                                                                      │  │ist Appro-│
                                                                                      │  │val (T3)  │
                                                                                      │  └────┬─────┘
                                                                                      │       │
                                                                                      │       v
                                                                                      │  ┌──────────┐
                                                                                      │  │C3:C-GATE │
                                                                                      │  │T1+T3+ARCH│
                                                                                      │  └────┬─────┘
                                                                                      │       │
                                                                                      │  FAIL (3x any tier)
                                                                                      │  ┌─────│─────┐
                                                                                      │  │ LOOP BACK│
                                                                                      │  └─────│─────┘
                                                                                      │       │ PASS
                                                                                      │       v
                                                                                      │  ┌──────────┐
                                                                                      │  │ COMMIT   │
                                                                                      │  └──────────┘
```

### State Transition Table

| From | Event | To | Condition |
|------|-------|----|-----------|
| A0 | Task defined | A1 | All agents have task spec |
| A1 | Reviews complete | A2 | All 6 specialists reviewed |
| A2 | Challenge complete | A3 | Synthesis produced |
| A3 | A-GATE passes | B1 | All APPROVED/CONDITIONAL PASS + T-ARCH passes |
| A3 | A-GATE fails | A1 | REJECTED or T-ARCH fail; loop back (max 3x) |
| B1 | Plan complete | B2 | Logical units identified |
| B2 | Unit implemented | B2a | Build passes locally |
| B2a | B-UNIT-GATE passes | B2 (next unit) | T1 + T-ARCH pass |
| B2a | B-UNIT-GATE fails | B2 (fix) | T1 or T-ARCH fail; retry (max 3x per tier) |
| B2 | All units done | B3a | All units pass B-UNIT-GATE |
| B3a | B-FINAL-GATE passes | C0 | T1 + T2 + T-ARCH pass |
| B3a | B-FINAL-GATE fails | B2 (fix) | Any tier fails (max 3x per tier) |
| C0 | T1 re-run passes | C1 | All T1 checks pass |
| C0 | T1 re-run fails | B2 (fix) | Code Architect fixes (max 3x) |
| C1 | Challenge complete | C2 | Synthesis produced |
| C2 | Reviews complete | C3 | All 6 specialists reviewed |
| C3 | C-GATE passes | COMMIT | All APPROVED + T1 pass + T-ARCH pass |
| C3 | C-GATE fails | C0 or C2 or B2 | T1 fail -> C0 (Code Architect fixes, re-run T1); T3 fail -> C2 (specialist re-review); T-ARCH fail -> Software Engineer (architectural fix) |
| Any | 3 retries exhausted at any tier | ESCALATE | Supreme Leader presents violation report to user |

---

## Dispatch Envelope

Every agent dispatch carries a structured envelope:

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
| `ticket` | Unique task identifier |
| `phase` | Current pipeline phase (A, B, or C) |
| `step` | Current step within the phase |
| `trigger` | Why this dispatch occurred |
| `agent` | The agent being dispatched to |
| `passport` | Path to the pipeline passport file tracking completed steps for this task |
| `skills_loaded` | Skills loaded for this dispatch (always includes core) |
| `expected_outcomes` | Concrete, verifiable deliverables expected |
| `next_agent` | Who receives the output next |
| `retry_count` | Current retry count for each tier at the current gate |
| `OWASP_expansion` | Any OWASP compliance categories added for this task |

---

## Agent Routing

| Intent | Agent | Key Skills |
|--------|-------|------------|
| Architecture design | Software Engineer | assumption-trap, compliance-gate, type-design-review |
| Register model design | Hardware Engineer | assumption-trap, datasheet-verification, domain |
| RF protocol design | Wireless Expert | assumption-trap, datasheet-verification, domain |
| Security analysis | Security Reviewer | assumption-trap, silent-failure, memory-safety |
| Test strategy | Test Engineer | assumption-trap, test-driven-development |
| Documentation plan | Docs Writer | assumption-trap, verification-before-completion |
| Implementation | Code Architect | pau-loop, incremental-execution, compliance-gate |
| T1 compliance check | Code Architect | compliance-gate, verification-before-completion |
| T2 architectural review | Software Engineer | compliance-gate, type-design-review |
| T3 semantic review | All 6 specialists | compliance-gate, domain-specific skills |
| T-ARCH review | Software Engineer | compliance-gate, type-design-review |
| Memory safety review | Memory Safety | assumption-trap, memory-safety |
| Gate orchestration | Supreme Leader | pipeline, compliance-gate, flag-protocol |
| Dispatch/routing | Supreme Leader | pipeline, flag-protocol |
| Task creation | PM | pipeline, flag-protocol |
| Debugging | Code Architect | systematic-debugging, domain |

---

## Dual-Model Challenge Protocol

Used in **Phase A** (architecture) and **Phase C** (verification).

### How It Works

1. **Primary pass** -- First model produces the output (architecture proposal or verification).
2. **Challenger pass** -- Second model independently reviews, looking for:
   - Contradictions with datasheet/spec
   - Missed edge cases
   - Unsupported assumptions
   - Security gaps
   - Protocol non-compliance
   - T-ARCH violations (logical errors, structural issues, principle misalignment)
3. **Synthesis** -- Supreme Leader merges findings:
   - Agreements -> accepted
   - Contradictions -> presented to user for decision
   - One-sided findings -> accepted if well-evidenced, otherwise flagged

### When to Invoke

| Scenario | Use Dual-Model? |
|----------|-----------------|
| New register implementation | Yes |
| New protocol feature | Yes |
| HAL interface change | Yes |
| Architecture change | Yes |
| Bug fix in existing code | No (single pass) |
| Documentation-only change | No |
| Trivial refactor (rename, move) | No |

---

## Skill Loading Protocol

### Mandatory Loading Order

When a task is dispatched, skills must be loaded in this order:

1. `assumption-trap` -- FIRST, always
2. `compliance-gate` -- tiered checks, OWASP expansion
3. `pipeline` -- this skill, state machine
4. `pau-loop` -- for Phase B work
5. Domain-specific skills as needed

### Skill Categories

1. **Core skills** (always loaded): assumption-trap, compliance-gate, pipeline, pau-loop, verification-before-completion, self-audit-checklist, review-confidence, type-design-review, silent-failure
2. **Domain skills** (loaded based on task): project-specific skills matching the tech stack
3. **Phase skills** (loaded based on phase): brainstorming (Phase A), incremental-execution (Phase B), grill-me (Phase A or C Dual-Model Challenge)
4. **Compliance expansion** (loaded based on OWASP triggers): review task for new concern categories and load additional compliance checks as needed

---

## Pipeline Passport

Every task carries a **passport** that tracks which pipeline steps have been completed. No step may be skipped without written justification. An agent receiving a task with a missing previous step must reject it and return STATUS: BLOCKED to the Supreme Leader.

Passports are stored in `docs/project-management/passports/<ticket-id>-passport.md` and are created by the PM when a ticket is opened. For the full passport format and rules, see `.opencode/skills/pipeline-passport/SKILL.md`.

Key passport rules:

1. **No step without a stamp** -- every step must be checked off before the next step begins
2. **No skip without justification** -- a written justification and Supreme Leader authorisation are required
3. **Loops are tracked** -- every A->B->A loop is recorded in the passport's Loop History section
4. **Passport travels with dispatch** -- the passport file path is included in every dispatch envelope

---

## Pipeline Enforcement Protocol

The pipeline is **not advisory**. It is **mandatory**. The Supreme Leader MUST enforce these rules before every dispatch. No exceptions.

### Pre-Dispatch Gate (Non-Skippable)

Before the Supreme Leader classifies intent or routes to any agent, it MUST execute this gated sequence:

| Gate Step | Check | Failure Action |
|-----------|-------|----------------|
| **PM Gate** | If new task: passport must be created by PM before any routing. Supreme Leader dispatches to PM with `trigger: "create-passport"` and waits. | BLOCKED -- no routing until passport exists. |
| **Passport Exists** | Passport file at `docs/project-management/passports/<ticket-id>-passport.md` exists on disk. | BLOCKED -- dispatch to PM for passport creation. |
| **Prior Steps Stamped** | All steps before target step have timestamps and results in Step Log. | BLOCKED -- route to missing step's agent. |
| **Gate Results Recorded** | If at a gate, Gate Results table has current attempt entries. | BLOCKED -- run gate first. |
| **Skips Justified** | Any unchecked Required Step has corresponding Skipped Steps entry with authorisation. | BLOCKED -- require authorisation. |
| **Correction Records** | If retry_count > 0 for any tier, Correction Record exists in passport. | BLOCKED -- dispatch to producing agent for post-rejection-correction. |

### Role Separation Rules

| Rule | Enforcement |
|------|-------------|
| Only PM creates passports | Supreme Leader MUST NOT create passport files. If none exists, dispatch to PM and wait. |
| Only PM creates tickets | Supreme Leader MUST NOT create task entries in TODO.md or ticket files. |
| Supreme Leader is dispatch-only | Supreme Leader MUST NOT perform specialist work. If a specialist fails, report to user -- do not fill in. |
| No combined PM + Supreme Leader | These roles operate at different steps. The envelope must go PM -> Supreme Leader, never both at once. |

### Status Protocol

When any enforcement check fails:

```
STATUS: BLOCKED
Reason: <which check failed and why>
Action Required: <what must happen to unblock>
```

The Supreme Leader must return this to the user immediately. Do NOT proceed to routing. Do NOT attempt to self-resolve.

---

## Self-Reflection Clause

After any pipeline violation or gate failure, the responsible agent MUST ask:

1. **Why was this not caught earlier?** -- What review, test, or protocol gap allowed it through?
2. **What procedural safeguard would have caught it?** -- What specific check, test, or verification step would have prevented it?
3. **Update the knowledge base** -- Add the lesson to the relevant skill or learning doc.

Violations in the pipeline process itself (wrong routing, missed gate, skipped step) should be logged as flags and added to the pipeline skill's lessons learned.
