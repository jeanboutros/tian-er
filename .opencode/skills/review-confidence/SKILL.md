---
name: review-confidence
description: "0-100 confidence scoring system for all review findings. When performing a Phase A or Phase C review, or when compliance-gate detects issues, every finding must include a confidence score. Scores 80 and above block (must fix); scores below 80 are advisory."
---

# Review Confidence Scoring

## Purpose

When multiple reviewers evaluate the same artifact, findings must be comparable. A confidence score system ensures that definitive problems are clearly distinguished from speculative concerns, and that blocking issues are addressed before advisory ones.

This system is inspired by Claude Code's code-review plugin.

## When to Trigger

- **Any agent performing a Phase A or Phase C review** — include confidence scores in all findings.
- **When compliance-gate skill detects issues** — each violation gets a confidence score.
- **When performing T-ARCH reviews** — structural and logical findings get confidence scores.
- **Any time an agent produces a verdict** (APPROVED, CONDITIONAL PASS, REJECTED).

---

## Confidence Scale

| Range | Level | Meaning | Action |
|-------|-------|---------|--------|
| 90-100 | Critical | Definitely real, must fix | **Blocks** — equivalent to REJECTED. Must be addressed before proceeding. |
| 80-89 | High | Very likely real, should fix | **Blocks** — equivalent to CONDITIONAL PASS with strong recommendation. Must be addressed or explicitly accepted by the user. |
| 70-79 | Moderate | Probably real, should consider | **Advisory** — does not block, but should be tracked and considered. |
| 0-69 | Low | Might be real, use judgment | **Informational** — noted for awareness, no action required. |

### Threshold: ≥80 Blocks

- **Any finding with confidence ≥80** MUST be addressed before the gate can pass.
- Findings with confidence <80 are advisory and do not block, but must be recorded.
- The overall verdict depends on whether any ≥80 finding exists.

---

## How to Score

### Scoring Factors

A finding's confidence score is based on three factors:

| Factor | What It Measures | High Score (>85) | Medium Score (70-85) | Low Score (<70) |
|--------|----------------|-------------------|----------------------|-----------------|
| **Evidence Strength** | How directly does the evidence support the finding? | Datasheet citation, build error, spec violation | Code inspection, design pattern concern | Theoretical concern, style preference |
| **Verification Level** | Has the finding been independently verified? | Build failure, test failure, automated check | Manual review, single reviewer | Unverified, speculative |
| **Spec Reference** | Is there a clear spec/standard/datasheet that defines the correct behaviour? | Exact section/page citation | General principle reference | No specific reference, subjective |

### Scoring Procedure

For each finding:

1. **Identify the evidence** — What specifically is wrong? Quote the file:line or datasheet section.
2. **Assess evidence strength** — Is this a provable violation or a design concern?
3. **Assess verification level** — Can this be mechanically verified, or is it judgement?
4. **Check spec reference** — Is there an objective standard that defines correct behaviour?
5. **Assign a preliminary score** — Start from the evidence strength, adjust up for verification and spec support, adjust down for subjectivity.
6. **Round to nearest 5** — Avoid false precision. Use 90, 85, 80, 75, 70, 65, 60, etc.

### Scoring Examples

| Finding | Evidence | Verification | Spec | Score | Level |
|---------|----------|-------------|------|-------|-------|
| Register field at wrong bit position | Datasheet §8.1 shows EN_CRC at bit 0, code has it at bit 1 | Build runs, code inspection | nRF24L01+ datasheet p.54 | 95 | Critical |
| Public `uint8_t` param with no typed overload | Grep finds `write_reg(uint8_t addr, uint8_t val)` in driver.h:45 | Automated T1.5 check | AGENTS.md typed vocabulary mandate | 90 | Critical |
| Missing Doxygen on `to_byte()` | Grep finds no `/** @brief */` above `to_byte()` in config.h:32 | Automated T1.2 check | Project rule: every public symbol | 90 | Critical |
| Likely buffer overflow in `read_payload()` | Size argument not validated before `memcpy` at driver.cpp:87 | Manual code review, no test failure | General secure coding, no project-specific rule | 75 | Moderate |
| Concern about future scalability | Current design couples BLE config to driver class | Design review, no current failure | No specific spec rule | 55 | Low |
| Style preference for different naming | Would prefer `TxAddr` over `TXAddr` | Opinion, no violation | No applicable rule | 20 | Low |

---

## Report Format

Every review or gate check that produces findings MUST use this format:

```markdown
## Review Findings

**Reviewer:** [Agent role]
**Phase:** [A / B / C]
**Artifact:** [File(s) or design doc reviewed]
**Date:** [ISO 8601 date]

### Findings

| ID | Confidence | Severity | File:Line | Description | Suggested Fix |
|----|-----------|----------|-----------|-------------|---------------|
| F1 | 95 | Critical | driver.h:142 | Public `uint8_t` param `reg_addr` has typed vocabulary but no typed overload | Add `write_reg(Config{})` overload, make raw overload private |
| F2 | 90 | Critical | config.h:7 | EN_CRC at bit 0 in code, datasheet says bit 0 — PASS (verified) | N/A |
| F3 | 80 | High | driver.cpp:87 | `read_payload()` doesn't validate size before memcpy | Add size check: `if (len > MAX_PAYLOAD) return false;` |
| F4 | 70 | Moderate | ble.h:23 | Whitening seed is channel-dependent but not clearly documented | Add Doxygen explaining seed derivation |
| F5 | 55 | Low | driver.h:200 | Consider using `std::array` instead of C array for payload | Future refactor suggestion |

### Blocking Findings (confidence ≥80)

- F1: Must add typed overload before C-GATE.
- F3: Must validate size before memcpy before C-GATE.

### Advisory Findings (confidence <80)

- F4: Should document whitening seed derivation.
- F5: Consider for future refactoring.

### Verdict

[APPROVED / CONDITIONAL PASS / REJECTED]

**Rationale:** [Explain why this verdict, referencing blocking findings if any]
```

---

## Integration with Compliance Gates

### T1 Mechanical Checks

T1 checks are **mechanically verifiable** — they either pass or fail. T1 findings are always scored **90-95 (Critical)** because they are automated and objective:

- T1.1 (Build): 95 — build failure is definitive
- T1.2 (Doxygen): 90 — grep is definitive
- T1.3 (Decision refs): 90 — grep is definitive
- T1.4 (Changelog comments): 90 — grep is definitive
- T1.5 (Raw uint8_t): 90 — grep is definitive
- T1.6 (Magic numbers): 85 — requires checking named constant existence
- T1.7 (Constants in module): 90 — grep is definitive
- T1.8 (Reserved bits): 85 — requires code analysis

### T2 Architectural Checks

T2 checks require judgement. T2 findings vary:

- T2.1 (Platform boundary): 90 — definitive grep
- T2.2 (Namespace hygiene): 85 — definitive but requires understanding intent
- T2.3 (File placement): 80 — requires design judgement
- T2.4 (API surface): 85 — requires understanding type alternatives
- T2.5 (No mutable globals): 90 — definitive grep

### T3 Semantic Checks

T3 checks require deep expertise. T3 findings vary widely:

- Datasheet deviation: 90-100 (if directly cited against datasheet)
- Protocol error: 85-95 (if directly cited against BLE spec)
- Security vulnerability: 80-95 (depends on exploitability)
- Test gap: 75-90 (depends on likelihood of the edge case)
- Documentation gap: 70-85 (depends on whether the user needs the info)

### T-ARCH Checks

T-ARCH findings are typically structural:

- Logical inconsistency: 85-95
- Missing section in output: 80-90
- Principle violation: 85-95 (if principle is clearly defined in AGENTS.md)
- Routing error: 90-100 (routing is deterministic)

---

## Confidence and Gate Verdicts

| All Findings <80 | Result |
|-----------------|--------|
| No findings at all | APPROVED |
| Only advisory findings | CONDITIONAL PASS (advisory findings logged) |

| Any Finding ≥80 | Result |
|-----------------|--------|
| One or more Critical/High findings | REJECTED (must fix blocking findings) |

| After Fixing | Result |
|-------------|--------|
| All previously ≥80 findings addressed | Re-evaluate. If no new ≥80 findings, APPROVED/CONDITIONAL PASS |

### Multiple Reviewers

When multiple reviewers produce findings on the same artifact:

1. **Collect all findings** from all reviewers.
2. **Deduplicate** — if two reviewers find the same issue, merge and take the higher confidence score.
3. **Dedup scoring** — when merging, add 5 to the confidence if two independent reviewers agree (cap at 100). Independent agreement is strong evidence.
4. **Address all ≥80 findings** from any reviewer.
5. **Record all findings** in the review persistence directory.

---

## Self-Reflection Clause

After any review that produces findings, especially blocking findings (≥80), ask:

1. **Why was this issue not caught earlier?** — What review, test, or protocol gap allowed it through?
2. **What procedural safeguard would have caught it?** — What specific check, test, or verification step would have prevented it?
3. **Update the knowledge base** — Add the lesson to the relevant skill or learning doc.

### Scoring Self-Reflection

After fixing a blocking finding, the reviewer should also self-reflect on the **confidence score itself**:
- Was the confidence score accurate? Did an 85-rated finding turn out to be trivial, or did a 70-rated finding turn out to be critical?
- What evidence did I over- or under-weight?
- Should future reviews adjust their scoring for similar findings?

Record these reflections in the review round file.
