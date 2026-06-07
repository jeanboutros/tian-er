---
name: compliance-gate
description: "Tiered compliance gate system for the PSC pipeline. Defines T1 (Mechanical), T2 (Architectural), T3 (Semantic), and T-ARCH (Architecture + Principles) checks. Every agent output passes through these gates. Includes OWASP compliance expansion and self-reflection on violations."
---

# Compliance Gate System

## Purpose

Every piece of work — code, documentation, reviews, and agent outputs — must pass through compliance gates before proceeding. This skill defines the complete tiered check system, gate placement, retry protocols, and the architectural review that runs on ALL agent outputs, not just formal phase transitions.

## When to Trigger

- **Always loaded** for all agents as part of the core skill set.
- **Additionally triggered** when any agent produces output that needs a gate check.
- **Triggered during B-UNIT-GATE, B-FINAL-GATE, A-GATE, and C-GATE** execution.
- **Triggered after every agent output** for T-ARCH review.

---

## Tier Definitions

### T1 — Mechanical (Automated)

Checks that can be verified by automated tooling. No human judgement required.

| # | Check | Method | Fail Condition |
|---|-------|--------|----------------|
| T1.1 | Build passes | Project build command | Exit code ≠ 0 or any warning |
| T1.2 | Doxygen on public symbols | Grep every new/changed function, struct, enum, method for `/**` block with `@brief` | Missing `@brief` on any public symbol |
| T1.3 | No decision references | Grep for `D-\d`, `F-\d`, `(decision` patterns in source files | Any match |
| T1.4 | No changelog-style comments | Grep for `replaces the`, `was previously`, `formerly`, `old`, `refactored from` in source files | Any match |
| T1.5 | No raw integers in public API where typed vocabulary exists | Every parameter in public API with a finite set of legal values must use a typed enum or struct | Public raw-type param with no typed alternative |
| T1.6 | No magic numbers in doc examples | Grep for unexplained literals in doc/code examples within public headers | Raw literal where named constant/enum exists |
| T1.7 | Constants in correct module | Grep for domain-level constants defined outside their module | Constant defined in the wrong module |
| T1.8 | Reserved/padding fields handled | Check serialisation implementations for reserved bit/field masking | Reserved fields not masked or zeroed |
| T1.9 | No hardcoded secrets | Grep for common secret patterns (password, api_key, secret, token, credential, Bearer) in source files | Any match that isn't a test fixture or explanatory comment |

**Who runs T1:** Code Architect (automated checks); Supreme Leader orchestrates retry loops.

### T2 — Architectural (Agent Delta Review)

Checks that require design-level judgement about structure, boundaries, and API surface.

| # | Check | Method | Fail Condition |
|---|-------|--------|----------------|
| T2.1 | Module/platform boundary | Grep for platform-specific includes in shared library public headers | Platform headers leaking into portable/shared module |
| T2.2 | Namespace/module structure | All public symbols in correct module namespace | Symbol in wrong namespace or module |
| T2.3 | File placement | Domain constants in their module, task functions in separate files, platform adapters isolated | Constant or function in wrong module |
| T2.4 | API surface audit | Every public method parameter is maximally restrictive type | Public raw-type param where typed vocabulary exists |
| T2.5 | No mutable globals in library | Grep for file-scope mutable globals in shared modules | Global mutable state found |

**Who runs T2:** Software Engineer spot-checks with delta review; Code Architect implements fixes.

### T3 — Semantic (Full Specialist Review)

Checks that require deep domain expertise — datasheet fidelity, protocol correctness, security, test coverage, documentation completeness.

| # | Check | Method | Fail Condition |
|---|-------|--------|----------------|
| T3.1 | Datasheet fidelity | Field names, bit positions, encodings compared to datasheet tables | Any deviation from datasheet |
| T3.2 | Protocol correctness | BLE channel mapping, whitening, CRC, PDU format verified | Protocol violation |
| T3.3 | Security review | Buffer bounds, stack depth, secrets, input validation | Any vulnerability |
| T3.4 | Test coverage | static_assert round-trips, edge cases, host-side unit tests | Uncovered public function or missing edge case |
| T3.5 | Documentation completeness | Learning docs, INDEX.md, verified references | Missing or unverified doc |
| T3.6 | Full architecture review | SOLID, HAL sufficiency, namespace hygiene, component CMake | Architecture violation |

**Who runs T3:** All six specialist agents (Software Engineer, Hardware Engineer, Wireless Expert, Security Reviewer, Test Engineer, Docs Writer).

### T-ARCH — Architecture + Principles Review

T-ARCH checks structural and logical correctness of **every agent output** — not just code. This is the gate that ensures every output adheres to project principles regardless of whether it's code, documentation, or a review.

T-ARCH is **always applied** alongside whichever tier checks are running. It is not a separate gate placement — it is a cross-cutting concern.

| # | Check | Applies To | Fail Condition |
|---|-------|------------|----------------|
| T-ARCH.1 | Logical consistency | All outputs | Internal contradictions, claims that conflict with evidence |
| T-ARCH.2 | Structural soundness | All outputs | Missing sections, incomplete analysis, skip in reasoning |
| T-ARCH.3 | Principle alignment | All outputs | Output violates established project principles (typed API, HAL decoupling, datasheet fidelity, no raw integers) |
| T-ARCH.4 | Completeness | All outputs | Required sections or checks omitted without justification |
| T-ARCH.5 | Correct agent routing | Dispatches | Task routed to wrong agent type (e.g. implementation sent to reviewer) |

**T-ARCH is evaluated by:**
- The **Software Engineer** for code and design outputs
- The **Supreme Leader** for routing and dispatch decisions
- Any specialist reviewing a document or plan for their domain

---

## OWASP Compliance Expansion

When a ticket, task, or feature introduces new types of concerns, the compliance-gate skill must detect this and require additional compliance checks to be loaded.

### Detection Rules

The Supreme Leader or the reviewing agent must check whether the work introduces any of the following concern categories:

| Concern Category | Trigger Signals | Required Skill/Check |
|-----------------|-----------------|---------------------|
| **Secrets/Credentials** | Passwords, API keys, tokens, certificates, encryption keys | Security Reviewer full checklist + OWASP A07 (Identification and Authentication Failures) |
| **PII/User Data** | User identifiers, device IDs, location data, personal data | OWASP A01 (Broken Access Control) + privacy compliance review |
| **Network Communication** | WiFi, BLE, HTTP, MQTT, TCP/UDP sockets | OWASP A02 (Cryptographic Failures) + A03 (Injection) review |
| **Firmware Updates** | OTA, flash writes, bootloader | OWASP A08 (Software and Data Integrity Failures) |
| **Input from External Sources** | SPI data, RF packets, UART, sensor data | OWASP A03 (Injection) — treat all external input as untrusted |
| **Logging/Debug Output** | Serial output, debug prints, telemetry | OWASP A09 (Security Logging and Monitoring Failures) |

### Expansion Protocol

1. **Identify** new concern categories from the task description.
2. **Load** the relevant compliance check (OWASP category or project-specific skill).
3. **Add** the expanded checks to the gate tier (T3 for semantic, T-ARCH for structural).
4. **Document** the expansion in the gate results: `OWASP Expansion: [category] → [added checks]`.
5. **Re-run** the gate with the expanded check set.

### Common OWASP Categories for Embedded/IoT

| OWASP ID | Category | Relevant to This Project |
|----------|----------|--------------------------|
| A01 | Broken Access Control | BLE pairing, debug interface access |
| A02 | Cryptographic Failures | CRC configuration, whitening integrity |
| A03 | Injection | SPI input parsing, RF payload handling |
| A04 | Insecure Design | HAL abstraction boundaries |
| A05 | Security Misconfiguration | Register defaults, power-on state |
| A06 | Vulnerable Components | Third-party libraries, ESP-IDF version |
| A07 | Auth Failures | If auth is added to BLE pairing |
| A08 | Software/Data Integrity | OTA updates, firmware verification |
| A09 | Security Logging | Debug output, error reporting |
| A10 | SSRF | N/A for this project (no HTTP client) |

---

## Gate Placement

Gates are mandatory checkpoints at phase transitions. Work cannot proceed past a gate until all required tiers pass.

### A-GATE (Exits Phase A)

| Property | Value |
|----------|-------|
| Location | Between Phase A and Phase B |
| Tiers | T3 + T-ARCH |
| Who runs | All 6 specialists (T3), Software Engineer (T-ARCH) |
| Pass | All 6 issue APPROVED or CONDITIONAL PASS + T-ARCH passes |
| Fail | Any REJECTED → loop back to Phase A with critique |
| Retry budget | 3 loops at T3, 3 loops at T-ARCH (independent) |

### B-UNIT-GATE (After Each PAU Unit)

| Property | Value |
|----------|-------|
| Location | After each logical unit in the PAU loop |
| Tiers | T1 + T-ARCH |
| Who runs | Code Architect (T1 automated), Software Engineer (T-ARCH) |
| Pass | All 9 T1 checks pass + T-ARCH passes |
| Fail | T1 fail → Code Architect fixes; T-ARCH fail → route appropriately |
| Retry budget | 3 loops per unit at T1, 3 loops at T-ARCH (independent) |

### B-FINAL-GATE (Exits Phase B)

| Property | Value |
|----------|-------|
| Location | Between Phase B and Phase C |
| Tiers | T1 + T2 + T-ARCH |
| Who runs | Code Architect (T1), Software Engineer (T2 + T-ARCH) |
| Pass | T1 passes AND T2 passes AND T-ARCH passes |
| Fail | Tier-specific routing, independent retry counters |
| Retry budget | 3 loops per tier (independent) |

### C-GATE (Enters Commit)

| Property | Value |
|----------|-------|
| Location | After Phase C specialist review, before git commit |
| Tiers | T1 re-run + T3 + T-ARCH |
| Who runs | Code Architect (T1), All 6 specialists (T3), Software Engineer (T-ARCH) |
| Pass | T1 passes AND all 6 APPROVED AND T-ARCH passes |
| Fail | Tier-specific routing, independent retry counters |
| Retry budget | 3 loops per tier (independent) |

### T-ARCH on Every Agent Output

T-ARCH runs on **every agent output**, not just at formal gate transitions:

- **After Code Architect:** T-ARCH checks logical consistency and principle alignment of implementation.
- **After Software Engineer:** T-ARCH checks structural soundness of design/output.
- **After Hardware Engineer:** T-ARCH checks datasheet citations are complete and logical.
- **After any review:** T-ARCH checks the review itself is structurally sound (findings have evidence, no gaps in reasoning).

This ensures no output — whether code, design document, or review — escapes principle-aligned scrutiny.

---

## Retry Protocol

Each compliance tier has an **independent** 3-retry budget at each gate.

### Independent Counters

| Tier | Retry budget per gate | Who handles failures |
|------|----------------------|---------------------|
| T1 (Mechanical) | 3 retries per gate | Code Architect (automated fixes) |
| T2 (Architectural) | 3 retries per gate | Code Architect with Software Engineer input |
| T3 (Semantic) | 3 retries per gate | Relevant specialist(s) |
| T-ARCH (Architecture + Principles) | 3 retries per gate | Software Engineer for code; Supreme Leader for routing |

**Key rule:** T1, T2, T3, and T-ARCH retry counters are independent. A T1 failure does not consume the T2, T3, or T-ARCH budget. Worst case per gate: 3×T1 + 3×T2 + 3×T3 + 3×T-ARCH = 12 loops.

### Retry Routing

| Failure tier | Routes to | What happens |
|-------------|-----------|--------------|
| T1 | Code Architect | Fix mechanical violations |
| T2 | Code Architect + Software Engineer | Fix architectural violations |
| T3 | Relevant specialist(s) | Fix semantic violations |
| T-ARCH | Software Engineer (code) or Supreme Leader (routing) | Fix logical/structural issues |

### Correction Analysis — Before Every Retry

Before any retry is dispatched, the **producing agent** MUST load the `post-rejection-correction` skill and produce a **Correction Record**:

1. Read the rejection findings from the Gate Violation Report.
2. Classify each finding into a root-cause category (RC-1 through RC-5).
3. Take the corrective action for each category (update a skill, add a question, add a checklist row).
4. Produce a Correction Record and append it to the passport's `## Correction Records` section.
5. If RC-4 (missing skill): Supreme Leader updates `skills_loaded` in the retry dispatch envelope before sending.

**No retry dispatch may proceed without a Correction Record stamped in the passport.**

See `post-rejection-correction` skill for the full root-cause taxonomy, the five RC categories, and the Correction Record format.

### Escalation

After 3 loops at the same tier → **ESCALATE to user** with:
- Full violation log (what failed, which check, which tier)
- Number of retry attempts
- Specific files and lines that failed
- Suggested remediation if known

### Gate-by-Gate Summary

| Gate | Tiers | Retry budget | Escalation trigger |
|------|-------|-------------|-------------------|
| A-GATE | T3 + T-ARCH | 3×T3, 3×T-ARCH | 3 loops at any tier |
| B-UNIT-GATE | T1 + T-ARCH | 3×T1, 3×T-ARCH | 3 loops at any tier |
| B-FINAL-GATE | T1 + T2 + T-ARCH | 3×T1, 3×T2, 3×T-ARCH | 3 loops at any tier |
| C-GATE | T1 + T3 + T-ARCH | 3×T1, 3×T3, 3×T-ARCH | 3 loops at any tier |

---

## Running the Checks

### T1 Automated Checks

T1 checks can be automated with grep/lint scripts appropriate for your project's language and build system. Run the project's build command and check exit code, then grep for the banned patterns listed in the T1 table above.

### Manual T2 and T-ARCH Checks

T2 and T-ARCH require human/agent judgement. Use the checklists above and document findings per file:line.

### T3 Specialist Review

T3 requires domain expertise. Route to the relevant specialist agent per the routing table in the pipeline SKILL.md.

---

## Violation Report Format

When a gate fails, generate a structured violation report:

```markdown
## Gate Violation Report

**Gate:** [A-GATE / B-UNIT-GATE / B-FINAL-GATE / C-GATE]
**Tier:** [T1 / T2 / T3 / T-ARCH]
**Retry:** [1/3 / 2/3 / 3/3]

### Violations

| # | Check | File:Line | Description | Suggested Fix |
|---|-------|-----------|-------------|---------------|
| 1 | T1.5 | driver.h:142 | Public `uint8_t` param `reg_addr` has no typed overload | Add `write_reg(Config{})` overload, make raw overload private |
| 2 | T-ARCH.3 | driver.h:142 | Violates typed API principle | See T1.5 fix |

### Self-Reflection

1. **Why was this violation not caught during implementation?** [Analysis]
2. **What procedural safeguard would prevent recurrence?** [Specific check or tool]
3. **Knowledge update:** [Location of updated doc/skill]
```

---

## Self-Reflection Clause

When a compliance violation is caught at any tier, the responsible agent MUST ask:

1. **Why was this violation not caught earlier?** — What review, test, or protocol gap allowed it through?
2. **What procedural safeguard would have caught it?** — What specific check, test, or verification step would have prevented it?
3. **Update the knowledge base** — Add the lesson to the relevant skill or learning doc so the same class of violation is caught earlier next time.

### Where to Log Lessons

| Violation type | Log location |
|---------------|-------------|
| Domain-specific hardware bugs | Relevant domain skill |
| General compliance process | `SKILL.md` compliance-gate skill |
| Learning about tech stack | `docs/learning/` |
| Pipeline process improvement | `docs/pipeline/pipeline.md` |

### Self-Reflection Format (added to violation reports)

```
SELF-REFLECTION:
  Violation: [T1.5 — raw uint8_t in public API]
  Why missed: [No grep check in editor; typed overload was planned but forgotten]
  Safeguard: [Add pre-commit hook that greps for public uint8_t params]
  Knowledge update: [Added to docs/learning/typed-api-enforcement.md]
```
