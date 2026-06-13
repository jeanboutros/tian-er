# Phase A Review: Test Engineer

| Field | Value |
|-------|-------|
| Reviewer | Test Engineer |
| Phase | A (Design Review) |
| Date | 2026-06-07 |
| Artifact | `docs/designs/inception.md` v0.5 |
| Review scope | Testability of acceptance criteria, test coverage gaps, integration test strategy, performance test feasibility, test data strategy, CI pipeline, smoke test comprehensiveness |

---

## Area 1: CI Infrastructure

**Finding:** Section 10.6 defines a CI pipeline (`ci/test-all.sh`) with 8 sequential phases. However, no CI infrastructure is defined:

1. **No CI runner specified.** Where does CI execute? The target platform is an ARM64 Raspberry Pi CM5. Standard CI providers (GitHub Actions, GitLab CI) provide x86-64 runners. ARM64 runners are available but at a premium. No runner configuration is mentioned.

2. **No CI trigger defined.** Section 10.1 says unit tests run "Every commit (pre-commit + CI)" but does not specify the CI event trigger (push to main? PR? merge queue?).

3. **Integration tests require PostgreSQL and TimescaleDB.** Section 10.6 step 6 says "against a docker-compose stack." But Docker on ARM64 (aarch64) has limited image availability. PostgreSQL + TimescaleDB Docker images for aarch64 exist but are larger and slower than on x86-64. The document does not specify how to run integration tests in CI.

4. **No artifact retention policy.** Build artifacts, test results, coverage reports — no retention policy.

5. **No CI failure notification.** Who is notified when CI fails? How?

6. **Pre-commit hooks (section 10.6 step 1) run on developer machines.** No guarantee that developers install pre-commit. CI must still run lint + format checks independently.

**Confidence:** 90  
**Verdict:** FAIL — CI infrastructure must be specified before implementation. At minimum: runner type, trigger event, Docker Compose configuration for integration tests, and failure notification.

---

## Area 2: CONTRACT 8.4-A End-to-End Integration Test

**Finding:** CONTRACT 8.4-A defines the interface between tshark and the ingest bridge. This is the most critical contract in the system — if the tshark output format changes (DECISION 8.4.1), the entire pipeline breaks. Despite this, there is no end-to-end integration test that validates the full path: real PCAP → tshark → ingest bridge → PostgreSQL.

The existing tests are:

| Test | Scope | Gap |
|------|-------|-----|
| `test_tshark_format.sh` | tshark output only | Does not feed output into ingest bridge |
| `test_ingest_e2e.sh` | Ingest bridge → DB | Uses synthetic input, not real tshark output |
| `parser_test.cpp` | C++ parser unit | 15 malformed cases, but not real tshark edge cases |

The gap: there is no test that takes a real PCAP fixture, runs it through `tshark-wrap.sh`, pipes the output to `blesniff-ingest`, and verifies the resulting PostgreSQL rows. This is the contract that everything else depends on.

**Impact:** If a tshark upgrade changes field formatting (e.g., RSSI from `-67 dBm` to `-67`), the ingest bridge parser fails silently (returns `std::nullopt` per CONTRACT 8.5-A), and the malformed_packets counter increments. No alarm fires until 100 consecutive malformed packets. In the meantime, all packets are lost.

**Confidence:** 95  
**Verdict:** BLOCKING — An end-to-end integration test from PCAP fixture through the full pipeline to PostgreSQL must be defined as a CI gate. This test must use real tshark output, not synthetic data.

---

## Area 3: Mock Sniffer Fidelity

**Finding:** Section 10.4 defines two test substitutes for real sniffers:

1. **PCAP replay** (`tools/pcap-replay.py`): Reads a fixture PCAP and writes it to a FIFO at a configurable rate. This is high-fidelity for testing the tshark → ingest path, but:
   - The replay tool simulates a pre-recorded capture, not a live capture with variable timing, dropped packets, and USB disconnects.
   - The replay rate is configurable, but the tool does not simulate the inter-packet timing distribution of real BLE advertising (which follows advertising intervals per BLE spec).
   - The replay tool does not simulate the FIFO buffer pressure that occurs when tshark is CPU-starved.

2. **Mock sniffer** (`tools/mock-sniffer.py`): "Emits synthetic packets with controllable parameters (rate, MAC pool, RSSI variance)." This is useful for volume testing but:
   - The mock sniffer produces data at the ingest bridge level (CONTRACT 8.4-A text format), not at the PCAP level. It bypasses tshark entirely. This means the mock sniffer cannot test the tshark → ingest path.
   - The smoke test (section 14.1 step 4) uses the mock sniffer to inject directly into the ingest bridge, completely bypassing the FIFO + tshark layers.

**Impact:** The mock sniffer gives false confidence in testing because it uses a simplified, controlled data format. Real tshark output may include edge cases (empty fields, unexpected encoding, multi-line output for single packets) that the mock sniffer never produces.

**Confidence:** 88  
**Verdict:** FAIL — The mock sniffer must be documented as testing only the ingest bridge, not the tshark → ingest path. A separate FIFO-level mock (writing PCAP data to a FIFO at realistic rates with realistic edge cases) should be added to the test strategy. The smoke test must include at least one end-to-end path through the full stack (PCAP → FIFO → tshark → ingest → DB).

---

## Area 4: Acceptance Criteria Testability

**Finding:** Several acceptance criteria are not testable or partially testable:

1. **Section 1, Goal 2:** "Persist all captured packets to disk in rotating PCAP files, with no acceptable data loss." "No acceptable data loss" is not quantified. What is the acceptable loss window? 0 packets? 0.001%? The dedup index (section 8.7) silently drops packets, FIFO edge cases may lose packets, and the single-channel limitation (see wireless review) means ~2/3 of BLE advertisements are never captured in the first place. The acceptance criterion is not falsifiable.

2. **Section 8.5, AC 4:** "After `kill -STOP $(pgrep postgres) && kill -CONT $(pgrep postgres)` simulating a brief PG outage, no packets are lost from a 60-second test stream." How is "no packets lost" verified? The test must compare the number of packets sent to the number of rows in the database. But the ingest bridge's dedup index may drop legitimate duplicates. The acceptance criterion must specify the comparison method.

3. **Section 8.2, AC 1:** "At each 30-minute boundary, a new PCAP file appears with name YYYYMMDD-HHMM.pcap." This requires a 30-minute wait per test iteration. No accelerated test mode is defined. DECISION 8.2.1 suggests `BLESNIFF_ROTATION_MINUTES=1` but this is not documented as a test configuration.

4. **Section 8.1, AC 3:** "Killing the FIFO reader does not drop bytes from the file." How is this verified? The test must kill tshark, wait, restart tshark, and then byte-compare the PCAP file against an expected output. But the PCAP file is being written to continuously — comparing a partial file is non-deterministic.

5. **Section 14.2, Smoke Test AC:** "The agent reports the platform as deployed only after `make test-e2e` exits 0." The smoke test (section 14.1) includes a 70-second sleep (step 5) waiting for continuous aggregate refresh. This makes the smoke test slow and potentially flaky on a loaded Pi.

**Confidence:** 92  
**Verdict:** FAIL — All acceptance criteria must be quantifiable and falsifiable. "No acceptable data loss" must be replaced with a specific threshold. Test verification commands must be deterministic or explicitly documented as probabilistic with expected pass rate.

---

## Area 5: Test Coverage Gaps

**Finding:** The test pyramid (section 10.1) defines coverage targets (80% C++ parser, 85% Python, 75% TS) but significant coverage gaps exist:

1. **No error path coverage for shell scripts.** The sniffer wrapper, rotation script, and tshark wrapper are bash scripts. `bats-core` tests are planned but only 4 test files are listed (section 8.1). The rotation script has 0 listed unit tests. The tshark wrapper has 1 malformed-input test but no tests for: tshark not found, FIFO not found, permission denied, tshark crash mid-stream, partial write.

2. **No negative testing for the ingest bridge.** The parser test covers 15 malformed cases, but what about:
   - Extremely long lines (>1 MB)
   - Non-UTF8 binary data in the FIFO stream
   - tshark emitting partial lines (write interrupted by signal)
   - SIGPIPE when stdout reader disconnects

3. **No database migration rollback testing.** If a migration fails partway through (e.g., `0003_compression_policies.sql` fails after `0002` succeeds), there is no test for recovery. Migration scripts are `IF NOT EXISTS` but `ALTER TABLE` and `SELECT add_compression_policy` are not idempotent in the same way.

4. **No Grafana dashboard query testing.** Section 8.12 acceptance criteria say "All four dashboards load without query errors." But dashboards contain SQL queries that may break after schema changes. No test validates that dashboard queries still produce results after a migration.

5. **No performance regression testing.** Section 10.5 defines performance targets but no mechanism to detect regression. "Manual + weekly on Pi" is unreliable. If a code change reduces ingest throughput from 1500 to 800 packets/sec, it will not be detected until the weekly manual test.

6. **No test for the `classify_residency` function.** The PL/pgSQL function (section 8.7, migration 0004) has complex branching logic but no unit tests are defined. Only an integration test (section 8.7 AC 4) says "returns valid classes."

**Confidence:** 85  
**Verdict:** FAIL — Coverage gaps must be addressed for: shell error paths, ingest bridge negative cases, migration rollback, Grafana queries, performance regression, and SQL function unit tests.

---

## Area 6: Integration Test Strategy

**Finding:** Section 10.6 step 6 says integration tests run "against a docker-compose stack." The design does not specify:

1. **What services are in the Docker Compose stack?** The integration tests need at least PostgreSQL + TimescaleDB. Do they also need Grafana? The API? The ingest bridge?

2. **How is the Docker Compose stack provisioned?** Is it started once per CI run? Persists between runs? How is data cleaned between tests?

3. **How are C++ binaries available in Docker?** The ingest bridge and deep parser are compiled for ARM64 aarch64. If CI runs on x86-64, the binaries won't execute. Cross-compilation or multi-arch Docker images are needed.

4. **No test isolation between integration tests.** If `test_ingest_e2e.sh` inserts rows into `raw_packets`, those rows persist and may affect `test_gap_detection.sh`. No test fixture cleanup strategy is defined.

5. **No test ordering defined.** Can integration tests run in parallel? Are there dependencies between them? If `test_gap_detection.sh` requires data from `test_ingest_e2e.sh`, this must be documented.

**Confidence:** 82  
**Verdict:** FAIL — Integration test infrastructure and isolation must be specified. Each integration test must document its prerequisites, expected state, and cleanup procedure.

---

## Area 7: Performance Test Feasibility

**Finding:** Section 10.5 defines 5 performance targets. Feasibility issues:

1. **"1500 packets/sec sustained for 5 min, no drops"** — This requires a test data source that produces 1500 pkts/sec. The PCAP replay tool (`tools/pcap-replay.py`) can accelerate playback, but accelerating a 30-second PCAP to 1500 pkts/sec produces unrealistic inter-packet timing. The performance may not reflect real-world conditions.

2. **"Ingest P95 latency < 1500 ms"** — This requires measuring the time from "line received by ingest bridge" to "row visible in PostgreSQL." There is no instrumentation for this measurement. The SIGUSR1 metrics dump (section 8.5) reports `last_flush_ms` and `batch_size_avg` but not end-to-end latency.

3. **"API P95 latency < 200 ms on aggregated endpoints"** — This requires a load testing tool (`wrk`, `hey`, `k6`). No tool is specified. The test requires the continuous aggregate to be populated, which requires seeded data and waiting for the aggregate refresh policy.

4. **"30-min PCAP file processed in < 5 min"** — This requires a 30-minute PCAP fixture. Section 11.1 lists `ubertooth-large.pcap.zst` (30m, ~200k packets) but it is listed as a fixture to be captured during T03. If hardware is not available during CI, this test cannot run.

5. **No thermal consideration.** Performance targets are specified for "the target Pi" but performance degrades under thermal throttling (see hardware review). No test specifies ambient temperature conditions.

**Confidence:** 80  
**Verdict:** ADVISORY — Performance tests are feasible but require additional tooling and instrumentation. The 1500 pkts/sec target may need a dedicated load generation tool. Latency measurement requires adding instrumentation to the ingest bridge.

---

## Area 8: Test Data Strategy

**Finding:** Section 11 defines fixtures and synthetic data generators. Issues:

1. **PCAP fixture provenance.** DECISION 11.1.1 says fixtures are captured during T03/T04 "with real hardware." This means fixtures are not available until hardware is procured and set up. No fixture PCAPs exist at design time. Development and testing before T03/T04 can only use synthetic data.

2. **MAC anonymization.** The document says "MACs anonymized via deterministic hash before commit." But if the hash is deterministic, any tester who knows the hash function can reverse it given the salt. The anonymization method must be documented and its irreversibility justified.

3. **`seed-test-data.py` schema coupling.** The synthetic data generator inserts rows directly into `raw_packets`. If the schema changes (new required field, different column type), the generator breaks. No schema version coupling test exists.

4. **No multi-sniffer test data.** The synthetic generator produces data for a single sniffer. No tool produces realistic multi-sniffer data that would test cross-sniffer scenarios (same device seen by multiple sniffers, different RSSI, slightly different timestamps).

5. **No edge-case PCAP fixtures.** Section 11.1 lists `ubertooth-malformed.pcap` (10 hand-crafted packets) but does not specify what malformed conditions it covers. No fixture for: encrypted BLE traffic, BLE 5.0 2M PHY packets, BLE 5.2 LE Isochronous channels, or other protocol edge cases.

**Confidence:** 78  
**Verdict:** ADVISORY — The test data strategy is reasonable for v1 but has gaps that should be addressed: multi-sniffer test data, edge-case PCAP fixtures, and a pre-T03 synthetic data path.

---

## Area 9: Smoke Test Comprehensiveness

**Finding:** The smoke test (section 14.1) covers 10 steps. Analysis:

| Step | Coverage | Gap |
|------|----------|-----|
| 1. systemd units active | Services up | No check for service _health_ (a service can be "active" but not functioning) |
| 2. DB schema | Migrations applied | No check for TimescaleDB extension or hypertable |
| 3. API health | DB reachable | No check for specific health indicators (sniffer count, last heartbeat recency) |
| 4. Ingest via mock-sniffer | Synthetic → DB | Bypasses FIFO + tshark (see Area 3) |
| 5. Continuous aggregate | Rows present | 70-second wait is slow; no check for aggregate correctness |
| 6. API returns devices | Endpoint works | No check for device count accuracy or field correctness |
| 7. Gap detector runs | Process exits 0 | No check that gap detector actually found or didn't find gaps |
| 8. Frontend serves | HTML returned | No check for JavaScript bundle loading or API connectivity |
| 9. Grafana reachable | Health + dashboards | No check for data in dashboard queries |
| 10. Cleanup | Removes test data | Good |

Missing from the smoke test:
- **No PCAP → tshark → ingest path validation.** Step 4 bypasses this.
- **No rotation test.** Does PCAP rotation work?
- **No reboot test.** Do FIFOs survive reboot? Do services come back up in the right order?
- **No database connectivity loss test.** Does ingest bridge reconnect after PG restart?
- **No disk-full test.** What happens when `/var/lib/blesniff/pcap` fills up?

**Confidence:** 88  
**Verdict:** FAIL — The smoke test must include at least one end-to-end path through the full stack (PCAP → FIFO → tshark → ingest → DB → API → frontend). Reboot resilience and disk-full scenarios should be added.

---

## Area 10: Test Environment Parity

**Finding:** The design specifies a single Raspberry Pi CM5 as both the production and test environment. This raises concerns:

1. **No isolation between test and production data.** Running integration tests on the Pi writes to the same PostgreSQL instance. The smoke test (section 14.1) creates a sniffer with ID 99 to avoid collision with production IDs, but what if production uses ID 99? The `raw_packets` hypertable has no row-level security.

2. **Test databases.** Section 10.3 says "a separate test database `blesniff_test` per run." Good, but this is only for database tests. Integration tests that span the full stack (sniffer → ingest → DB → API) use the production database because the ingest bridge and API are configured to connect to `blesniff`, not `blesniff_test`.

3. **CI on non-Pi hardware.** The CI pipeline (section 10.6) must run on CI runners that are likely x86-64. The C++ binaries compile differently on x86-64 vs aarch64 (endianness, pointer size, atomic operations). Integration tests that pass on x86-64 may fail on the Pi.

4. **No staging environment.** There is no concept of a staging Pi that mirrors production. Any change deployed to the production Pi has no pre-deployment validation on equivalent hardware.

**Confidence:** 80  
**Verdict:** ADVISORY — The `blesniff_test` database approach is reasonable. The design should document that full-stack integration tests on the Pi use the production database and recommend a `blesniff_test` configuration for the API and ingest bridge during testing.

---

## Summary of Findings

| # | Area | Severity | Confidence | Verdict | Description |
|---|------|----------|-----------|---------|-------------|
| 1 | CI infrastructure | Major | 90 | FAIL | No CI runner, trigger, or Docker stack defined |
| 2 | CONTRACT 8.4-A e2e test | Blocking | 95 | BLOCKING | No end-to-end integration test from PCAP to DB |
| 3 | Mock sniffer fidelity | Major | 88 | FAIL | Mock bypasses tshark; fidelity unverifiable |
| 4 | Acceptance criteria testability | Major | 92 | FAIL | "No data loss" unquantifiable; AC 4 unverifiable |
| 5 | Test coverage gaps | Major | 85 | FAIL | Shell error paths, negative tests, migration rollback, perf regression |
| 6 | Integration test strategy | Major | 82 | FAIL | No Docker spec, no test isolation, no ordering |
| 7 | Performance test feasibility | Minor | 80 | ADVISORY | Feasible with additional tooling and instrumentation |
| 8 | Test data strategy | Minor | 78 | ADVISORY | No multi-sniffer data, limited edge-case fixtures |
| 9 | Smoke test comprehensiveness | Major | 88 | FAIL | Bypasses FIFO+tshark, no rotation/reboot/disk-full tests |
| 10 | Test environment parity | Minor | 80 | ADVISORY | Production DB used for integration; no staging |

---

## Overall Verdict

**CONDITIONAL PASS** — 1 blocking finding (missing e2e integration test for CONTRACT 8.4-A) must be resolved before the contract is implemented. 6 major findings (CI infrastructure, mock fidelity, AC testability, coverage gaps, integration strategy, smoke test) must be addressed during implementation. 3 advisory findings can be addressed post-MVP.

Resolution path:
1. Add an end-to-end integration test that validates PCAP → tshark → ingest bridge → DB as a CI gate.
2. Define CI infrastructure (runner, triggers, Docker Compose config).
3. Quantify all "no data loss" acceptance criteria with specific thresholds.
4. Document mock sniffer limitations and add a FIFO-level mock.
5. Add shell error path tests, negative ingest tests, migration rollback tests.
6. Expand smoke test to cover the full pipeline path.

---

## Self-Audit Checklist

| # | Check | Status |
|---|-------|--------|
| 1 | Read the complete artifact? | YES — all 2910 lines |
| 2 | Every finding includes a confidence score? | YES |
| 3 | Every finding is actionable? | YES — specific tests and infrastructure to add |
| 4 | No speculative claims without evidence? | YES — all findings reference specific sections |
| 5 | Blocking findings justified by severity? | YES — missing e2e test for critical contract |
| 6 | Advisory findings clearly separated? | YES — areas 7, 8, 10 are advisory |
| 7 | No duplicate findings across areas? | YES |
| 8 | Verdict is consistent with finding severities? | YES — 1 blocking, 6 major, 3 advisory → CONDITIONAL PASS |

---

## Flags for PM

| Flag ID | Type | Description | Urgency |
|---------|------|-------------|---------|
| FLAG-TE-001 | Decision | Add e2e integration test: PCAP fixture → tshark → ingest → DB. Block on this before T08. | Blocking |
| FLAG-TE-002 | Decision | CI infrastructure: GitHub Actions ARM64 runner or self-hosted Pi runner? | High |
| FLAG-TE-003 | Decision | Quantify "no acceptable data loss" threshold. 0 rows? 0.01%? Per-sniffer? | High |
| FLAG-TE-004 | Ticket | Add FIFO-level mock sniffer (writes PCAP to FIFO, not text to stdin). | High |
| FLAG-TE-005 | Ticket | Add shell error path tests for rotation, tshark wrapper, and sniffer wrapper. | Medium |
| FLAG-TE-006 | Ticket | Add smoke test steps for: full pipeline path, PCAP rotation, and reboot resilience. | Medium |
| FLAG-TE-007 | Decision | Performance regression: manual weekly or automated with baseline comparison? | Low |