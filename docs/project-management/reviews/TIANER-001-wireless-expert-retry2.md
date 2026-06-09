# Phase A Re-Review: Wireless Expert (A1-WX, Attempt 3)

**Reviewer:** Wireless Expert
**Date:** 2026-06-07
**Phase:** A (A1-WX, retry #2)

## VERDICT: CONDITIONAL PASS
**Blocker count:** 7 findings at confidence ≥80

### WX-F1: INGEST-1 Ambiguity at C03→C05 Boundary (Conf: 88)
CAPTURE-2 is parameterized per DLT but INGEST-1 doesn't specify whether the output is normalized to a single schema or whether the ingest bridge handles multiple schemas. C03 may produce different column orders for Ubertooth vs nRF.

### WX-F2: Concrete Single-Dongle Channel Strategy Missing (Conf: 85)
C03 lists "BLE Protocol Notes" but doesn't specify the actual strategy. D-05 says "Strategy A/B hybrid" — need concrete default: suggest "default to channel 37; rotation via timer-based restart (30 min per channel) is post-MVP."

### WX-F3: CRC-24 Never Verified in Pipeline (Conf: 90)
No component verifies CRC-24. C07 assumes "CRC already verified" but nRF Sniffer firmware may not verify CRC for advertising packets. This is a single point of trust with no cross-validation. Suggest: C07 should have a configurable CRC-24 validation step.

### WX-F4: Deep Parser DLT-Aware Parsing (Conf: 82)
C07 reads rotated PCAP files but ROTATION-1 filename convention doesn't encode sniffer type. Parser needs to detect DLT from pcap header and handle different PDU layering (DLT 195 for nRF vs 251 for Ubertooth).

### WX-F5: Missing BLE Protocol Golden Test Vectors (Conf: 85)
No pre-computed golden vectors for CRC-24, dewhitening, or AdvData TLV parsing. C07 dissector testing needs these.

### WX-F6: Missing Cross-Sniffer Correlation Fixture (Conf: 80)
ubertooth-sample-001.pcap and nrf-sample-001.pcap are independent captures. Need simultaneous capture on same channel.

### WX-F7: DEEP-1 JSONL Missing CRC Status and Sniffer-Type Fields (Conf: 87)
JSONL output doesn't include `crc_valid` or `sniffer_type`/`dl_type` fields. Downstream ML can't distinguish verified from unverified data.

### Also Unresolved from Original Review
- MAC sub-typing (address_type still SMALLINT with 2 values)
- RSSI accuracy (continuous aggregate AVG(rssi) without sniffer_id grouping)

## Resolution Path
1. C07: Specify CRC-24 verification strategy per DLT; DLT-aware parsing; extend DEEP-1 with CRC/sniffer fields
2. C03: Specify concrete channel strategy; document dewhitening per sniffer; ensure CAPTURE-2 normalization
3. C05: Document INGEST-1 receives normalized schema
4. C02: Address cross-sniffer RSSI averaging
5. Test fixtures: Add golden test vectors and cross-sniffer capture
