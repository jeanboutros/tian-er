# Phase A Security Review — Component Decomposition (Retry #2)
**Reviewer:** Security Reviewer
**Date:** 2026-06-07
**Phase:** A (A1-SX, retry #2)

## VERDICT: CONDITIONAL PASS
**Blocker count:** 10 blocking findings (confidence ≥80), 6 advisory

### Blocking Findings
SX-F1 (92): Podman+Quadlet container security model completely absent — container escape, image integrity, network isolation, volume mount security all undesigned.
SX-F2 (90): Supply chain integrity gaps — no GPG verification, commit-hash pinning, checksum validation for any dependency.
SX-F3 (88): Secrets rotation still not designed — 3 secrets generated once, no rotation procedure.
SX-F4 (88): Hybrid model (native + containers) creates undesigned security boundary. Every component must answer "containerized or native?"
SX-F5 (85): DB role `tianer` has full DDL privs. Need write-only `tianer_writer` role for ingest bridge and gap detector.
SX-F6 (85): `generate-secrets.sh` never specified — no idempotency contract, no `--rotate` flag.
SX-F7 (82): No container image integrity — no digest pinning, no signing, no policy.json.
SX-F8 (82): Parameterized queries mandated for API only — gap detector, deep parser, ML enrichment not covered.
SX-F9 (80): No PostgreSQL audit logging (pgaudit). BLE data may constitute PII.
SX-F10 (80): Grafana + PostgreSQL connections use plain HTTP. D-06 covers API but not Grafana or PG.

### Advisory (SX-A1 to SX-A6)
Process isolation refinements, fuzz testing, SBOM/dependency scanning, Grafana file permissions bug, PCAP retention privacy impact, CORS config.

### OWASP Coverage (average: ~60% across A01-A09)
Worst gaps: A08 (Integrity, 40%), A06 (Vulnerable Components, 50%), A09 (Security Logging, 55%), A04 (Insecure Design, 60%), A07 (Auth Failures, 60%).

### Key Recommendation
Every resolved decision must trigger a corresponding design document update BEFORE the decision is marked resolved.
