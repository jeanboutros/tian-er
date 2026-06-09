---
name: cross-document-consistency
description: "Grep-based cross-document consistency checks: ADR cross-reference (DC-1), schema consistency (DC-2), decision-to-document trace (DC-3), SQL-vs-decision validation (DC-4). Loaded by Docs Writer during Phase C verification."
---

# Cross-Document Consistency

## Purpose

Prevent documentation drift by checking that documents agree with each other and with actual code. These are grep-based, deterministic checks — no judgement required. The skill codifies the four cross-document consistency checks (DC-1 through DC-4) that the Docs Writer runs during Phase C verification.

## When to Trigger

- Loaded by Docs Writer during Phase C verification
- Triggered when ADRs exist or when schema/documentation cross-references need validation
- May be loaded by other agents when verifying document consistency

---

## DC-1 — ADR Cross-Reference

**Check:** Every resolved decision has an ADR file, and every ADR is referenced from at least one other document.

**Procedure:**

1. Read the ADR directory (`docs/adr/`).
2. For each ADR file, extract its title and key terms.
3. Grep the documentation tree (`docs/`, `README.md`, design docs) for references to each ADR (by filename, title, or decision ID).
4. Flag any orphaned ADR — no cross-reference found in any document.
5. Flag any resolved decision from the A2 Dual-Model Challenge synthesis that has no corresponding ADR file.

**Pass criterion:** Every ADR is referenced in at least one other document. Every resolved A2 decision has an ADR.

---

## DC-2 — Schema Consistency

**Check:** SQL schemas, API contracts, data models match between documentation files. No contradictory field names, types, or constraints across files.

**Procedure:**

1. Grep for schema-defining patterns across all documentation files:
   - `CREATE TABLE`, `ALTER TABLE`
   - Interface/type definitions in design docs
   - API endpoint descriptions with request/response shapes
2. For each entity mentioned in multiple files, compare:
   - Field names (case-sensitive)
   - Field types (`INT` vs `VARCHAR`, `boolean` vs `bool`, etc.)
   - Constraints (`NOT NULL`, `UNIQUE`, `PRIMARY KEY`)
3. Flag any contradiction between two documentation files for the same entity.

**Pass criterion:** No contradictory field definitions across files for the same entity.

---

## DC-3 — Decision-to-Document Trace

**Check:** Every ADR title or key decision is referenced in at least one implementation document. No orphaned decisions.

**Procedure:**

1. Extract key terms from each ADR title (e.g., "Use PostgreSQL for persistence" → terms: `PostgreSQL`, `persistence`).
2. Grep implementation docs (`docs/learning/`, `docs/designs/`, `README.md`) for those terms.
3. Flag any ADR whose decisions are not traceable to at least one implementation document.

**Pass criterion:** Every ADR has at least one traceable reference in implementation documentation.

---

## DC-4 — SQL-vs-Decision Validation

**Check:** Any SQL schema claimed in documentation matches actual `.sql` files in the repository.

**Procedure:**

1. Find all `.sql` files in the repository (`find . -name '*.sql'`).
2. For each `.sql` file, extract table names, column names, and constraints.
3. Compare against documentation claims that reference SQL schemas.
4. Flag any mismatch between actual SQL and documented schemas.

**Pass criterion:** All documentation claims about SQL schemas match the actual `.sql` files.

---

## Output Format

```
CROSS-DOCUMENT CONSISTENCY REPORT

DC-1 (ADR Cross-Reference):
  ADRs found: N
  Referenced: M
  Orphaned: K
  [List orphaned ADRs with paths]

DC-2 (Schema Consistency):
  Contradictions found: N
  [List contradictions with file:line pairs]

DC-3 (Decision-to-Document Trace):
  ADRs checked: N
  Traced: M
  Untraced: K
  [List untraced ADRs with paths]

DC-4 (SQL-vs-Decision Validation):
  SQL files found: N
  Mismatches: K
  [List mismatches with paths]

VERDICT: [ALL CLEAR / ISSUES FOUND]
```

---

## Self-Reflection Clause

After any cross-document inconsistency is found, ask:

1. **Why was this inconsistency not caught earlier?** — Which phase or gate should have detected it?
2. **What procedural safeguard would have caught it?** — Which DC check caught it, and should other DC checks be expanded?
3. **Update the knowledge base** — Add the lesson to this skill or to the pipeline skill.
