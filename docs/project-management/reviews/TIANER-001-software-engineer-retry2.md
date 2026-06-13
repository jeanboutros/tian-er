# Phase A Review — Software Engineer Specialist (Retry #2)

## Review of: `inception.md` + `component-breakdown.md`

**Reviewer:** Software Engineer  
**Date:** 2026-06-07  
**Phase:** A (A1-SW, retry #2)  
**Artifacts reviewed:**
- `docs/designs/inception.md`
- `docs/designs/component-breakdown.md`
- `docs/project-management/passports/TIANER-001-passport.md`

## VERDICT: REJECTED
**Blocker count:** 10 (3 critical blockers, 7 high-confidence findings ≥80)

### Critical Blocker #1: Container Orchestration Decision Not Reflected (Conf: 95)
Passport resolves Podman + Quadlet hybrid model. Neither inception.md nor component-breakdown.md mentions containers. C12 and C14 are undesignable. Every service deployment doc must answer "containerized or native?"

### Critical Blocker #2: inception.md Contains Stale Version References (Conf: 90)
OS "Bookworm" → should be Trixie. Python "3.12" → 3.13. Node.js "20" → 24 LTS. Database name "blesniff" → "tianer". DB roles "blesniff*" → "tianer*". Config path wrong. Document is actively misleading for implementers.

### Critical Blocker #3: Resolved Decisions D-09/D-10/D-11 Missing (Conf: 88)
D-09 (300K row buffer, 5 min, 60MB RAM), D-10 (pdu_type in dedup index), D-11 (local heartbeat file primary, DB secondary) — none appear in component-breakdown.

### Finding #4: No Contract for DB ↔ Grafana (Conf: 82)
Grafana connects directly to DB bypassing API. No contract for read-only access scope, query complexity limits, connection pooling.

### Finding #5: No `bluetooth` Schema Specification (Conf: 85)
inception.md says `tianer` DB with `bluetooth` schema, but all table creation uses `public`. Future modules need schema isolation.

### Finding #6: C13 Layer Placement Ambiguity (Conf: 78)
C13 at Layer 1 references C05 metrics that don't exist yet. Needs split into "catalogue" (Layer 1) and "wiring" (distributed).

### Finding #7: Missing Contract for C07 → C02 (Conf: 80)
No contract specifying that C08 owns the DB write path, not C07. No idempotency contract (ON CONFLICT DO NOTHING vs UPDATE).

### Finding #8: Contract ID Restructuring Without ADR (Conf: 75)
inception.md's CONTRACT 8.4-A became CAPTURE-2 with no ADR linking them.

### Finding #9: D-05 "Dedup at Query Time" Not Reflected (Conf: 82)
Resolved decision says dedup at query time, but inception.md still has unique index — implies insert-time dedup. C02 schema inconsistent.

### Finding #10: Environment File Path Inconsistency (Conf: 78)
Naming convention says `/etc/tianer/blesniff.env` but inception.md says `/etc/blesniff/blesniff.env`.

### Advisory Findings (A1-A4)
A1: CAPTURE-2 contract lacks C++ type mapping for Packet struct (Conf: 70)
A2: C12 "final wiring" at Layer 4 — suggest noting systemd units can be written per-component (Conf: 65)
A3: `blesniff-classify.service` appears but isn't assigned to any component (Conf: 60)
A4: No `modules/shared/` for common code between future modules (Conf: 55)

## Component Design Doc Readiness
- C01: YES (with container decision caveat)
- C02: YES (needs `bluetooth` schema decision)
- C03: YES
- C04: YES
- C05: PARTIALLY (needs D-09 buffer, D-10 dedup)
- C06: PARTIALLY (needs D-11 heartbeat fallback)
- C07: YES
- C08: YES
- C09: YES
- C10: PARTIALLY (needs C09 API schema)
- C11: YES
- C12: NO (needs container orchestration)
- C13: YES (catalogue only)
- C14: NO (needs container orchestration)

## T-ARCH Review
- T-ARCH.1 (Logical consistency): FAIL — contradictions between docs and decisions
- T-ARCH.2 (Structural soundness): FAIL — missing container orchestration component
- T-ARCH.3 (Principle alignment): FAIL — PF-1 violated (new deployment model has no failure modes)
- T-ARCH.4 (Completeness): FAIL — 12 resolved decisions not all reflected
- T-ARCH.5 (Correct agent routing): PASS

## Recommendation
1. Add inception.md staleness warning banner
2. Incorporate Podman+Quadlet decision into component-breakdown
3. Document D-09, D-10, D-11 explicitly in artifacts
4. Add bluetooth schema specification
5. Add DB↔Grafana contract
6. Clarify D-05 "dedup at query time" impact on schema
