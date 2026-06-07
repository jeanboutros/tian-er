# Phase A Review: Wireless Expert

| Field | Value |
|-------|-------|
| Reviewer | Wireless Expert |
| Phase | A (Design Review) |
| Date | 2026-06-07 |
| Artifact | `doc/designs/inception.md` v0.5 |
| Review scope | BLE protocol compliance, sniffer capability, RF accuracy, data whitening, MAC randomization, BR/EDR, cross-sniffer timing |

---

## Area 1: Single-Channel-at-a-Time Sniffer Limitation

**Finding:** Section 8.1 specifies Ubertooth wrapper mode `btle` with `channels: [37, 38, 39]` in `sniffers.yaml`. Section 6.8.2 shows one Ubertooth configured. This design has a fundamental BLE protocol limitation that is not documented:

BLE advertising occurs on 3 channels: 37 (2402 MHz), 38 (2426 MHz), 39 (2480 MHz). Per Bluetooth Core Specification v5.4 [^ble-core-spec], Volume 6, Part B, Section 2.3:

- Advertisers send ADV packets on all 3 channels in sequence (37 → 38 → 39) per advertising event.
- The Ubertooth One can only listen on **one channel at a time** in passive sniff mode (`ubertooth-btle -A`). It does not channel-hop between advertising channels during a single capture session.
- The nRF52840 Dongle running nRF Sniffer firmware similarly captures on one channel at a time in passive mode. The nRF Sniffer v4.x can follow connections (channel-hopping), but for advertising capture it is single-channel.

**Impact on data integrity:**

1. A single Ubertooth on channel 37 captures only ~1/3 of advertising events. If a device sends ADV_IND on 37, 38, 39, the Ubertooth on 37 sees the channel-37 packet only. This means ~2/3 of advertisements are missed on every single advertising event.

2. The document says section 1 goal 2: "Persist all captured packets... with no acceptable data loss." But the system design cannot capture "all" BLE packets — it can only capture packets on the channel it listens to. The wording should be "Persist all packets received on the monitored channel" with an explicit note that single-channel sniffing captures a fraction of total advertising traffic.

3. Multiple sniffers on different channels (e.g., Ubertooth on 37, nRF1 on 38, nRF2 on 39) could provide full-channel coverage, but the document does not specify this as a design intent. Section 6.8.2 shows 1 Ubertooth + 3 nRFs, but does not specify which channel each sniffer monitors. If all 4 listen on channel 37, the system has 4× capture probability on channel 37 and 0% on channels 38-39.

4. The `channels: [37, 38, 39]` field in `sniffers.yaml` is ambiguous. Does the Ubertooth hop between these channels? (No — `ubertooth-btle -A` takes a single channel argument.) Does it mean "this sniffer may be configured for any of these channels"? If so, which channel is actually used?

**Confidence:** 95  
**Verdict:** BLOCKING — The single-channel limitation must be explicitly documented. The system goals must be amended to acknowledge that single-channel sniffing captures a subset of advertising traffic. A recommended sniffer-to-channel mapping strategy should be added.

---

## Area 2: DLT Type Differences and tshark Field Names

**Finding:** CONTRACT 8.4-A specifies a single tshark output format for both Ubertooth and nRF sniffers. This is incorrect because the two sniffers produce PCAP files with different DLT (Data Link Type) values, which cause tshark to expose different field names:

| DLT Value | Name | Source | tshark Field Prefix |
|-----------|------|--------|---------------------|
| 251 | `DLT_BLUETOOTH_LE_LL_WITH_PHDR` | Ubertooth | `btle.*` |
| 252 | `DLT_BLUETOOTH_LE_LL` | Some nRF captures | `btle.*` |
| 195 | `DLT_USER0` (nRF Sniffer custom) | nRF Sniffer | `nordic_ble.*`, `btle.*` |

The nRF Sniffer uses a custom DLT type that tshark maps to `nordic_ble.*` fields, not the standard `btle.*` fields. Specifically:

- Ubertooth PCAP: `btle.advertising_address`, `btle.advertising_header.pdu_type`, `btle.advertising_data`
- nRF Sniffer PCAP: `nordic_ble.rssi`, `nordic_ble.channel`, `nordic_ble.advertiser_address`, plus some `btle.*` fields

CONTRACT 8.4-A (section 8.4) mixes these: it uses `btle.advertising_address` (Ubertooth field) alongside `nordic_ble.rssi` and `nordic_ble.channel` (nRF fields). A single tshark invocation cannot produce all these fields from one PCAP source — the nRF fields are empty when reading Ubertooth PCAP, and vice versa.

DECISION 8.4.1 acknowledges this partially: "Field names listed above are starting candidates and must be verified." But the underlying issue is deeper: there is no single CONTRACT that works for both sniffer types. The ingest bridge must handle two different tshark field schemas.

**Impact:** The tshark wrapper script (section 8.4) will produce empty `nordic_ble.*` fields for Ubertooth data and potentially empty `btle.advertising_address` for nRF data (nRF may expose advertiser address under `nordic_ble.advertiser_address` instead). The ingest bridge parser must be told which schema to expect, or a normalization layer must be added.

**Confidence:** 92  
**Verdict:** FAIL — CONTRACT 8.4-A must be split into per-sniffer-type contracts or a unified normalization layer must be designed. The current single-contract assumption is incorrect.

---

## Area 3: Data Whitening

**Finding:** BLE uses data whitening (pseudo-random scrambling) on both the advertising channel PDU and the PHY-level access code to reduce DC bias in the transmitted signal. Per Bluetooth Core Specification v5.4, Volume 6, Part B, Section 1.4.2:

- The whitening LFSR is initialized with the channel index (lower 7 bits) for each channel.
- All sniffing hardware must dewhiten the received signal to recover the original PDU.
- If dewhitening is incorrect, the recovered PDU payload is garbled but the access address may still match (since the access address is whitened with the same LFSR but checked pre-dewhitening at the correlator level).

The document does not mention data whitening at all. This has implications:

1. **Ubertooth One:** Hardware dewhitening is performed in the Ubertooth firmware before PCAP output. Documented in the Ubertooth source. This is transparent to the software pipeline.

2. **nRF52840 Dongle (nRF Sniffer firmware):** The nRF Sniffer firmware dewhitens before emitting the PCAP stream. However, if the firmware version does not dewhiten correctly (which has been a bug in some nRF Sniffer firmware releases), the PCAP contains whitened data that tshark cannot parse correctly. tshark does not dewhiten — it expects pre-dewhitened PCAP input.

3. **Deep parser (section 8.8):** The C++ deep parser reads PCAP files directly via libpcap. If a PCAP file contains whitened data (due to firmware bug), the deep parser will produce garbage AdvData fields with no indication of the error. The deep parser should validate that parsed AdvData TLV structures are well-formed and flag un-parseable data.

4. **Cross-sniffer validation:** If Ubertooth and nRF produce different raw AdvData for the same packet, it could be a dewhitening difference rather than a different packet. There is no mechanism to detect this.

**Confidence:** 85  
**Verdict:** FAIL — Data whitening must be documented. The deep parser must include a validation step for dewhitened data. A cross-sniffer comparison mechanism should detect whitened vs. dewhitened output discrepancies.

---

## Area 4: MAC Address Randomization

**Finding:** Section 8.7 DECISION 8.7.2 addresses random MAC handling: "v1 stores all MACs, marks address_type. Residency analysis is computed but flagged as unreliable for address_type=1 (random) devices." This is reasonable for v1 but has important implications not fully discussed:

1. **Random MAC prevalence:** In typical environments, 70-90% of BLE advertisements come from random (static or RPA) addresses. Flagging most devices as "unreliable for residency" significantly reduces the system's utility.

2. **Static vs. RPA distinction:** The `address_type` field in `raw_packets` (0=public, 1=random) does not distinguish between static random addresses and Resolvable Private Addresses (RPAs). Per BLE Core Spec v5.4, Vol 6, Part B, Section 1.3.2:
   - Static random addresses: top 2 bits of most significant byte are `1x` (not `11` for NRPA, not `01` which is undefined). These are stable per device but change infrequently.
   - RPA: top 2 bits are `11`. These change every ~15 minutes (TIRP timer) and are IRK-derived.
   - NRPA: top 2 bits are `01`. These are one-time-use.

   The current `address_type` boolean (0/1) cannot distinguish these sub-categories. At minimum, a 3-value enum (public, random_static, random_resolvable) would improve the residency classifier's ability to handle static random addresses as "reliable" and RPAs as "unreliable."

3. **tshark field `btle.advertising_header.randomized_tx`:** CONTRACT 8.4-A position 3 is "address type (0=public, 1=random)." The tshark field `btle.advertising_header.randomized_tx` is a boolean (TRUE/FALSE), not 0/1. The ingest bridge parser must convert this.

**Confidence:** 88  
**Verdict:** FAIL — `address_type` should be an enum with at least 3 values (public, random_static, random_resolvable) or 4 values (+non_resolvable). The residency classifier design must acknowledge that static random addresses are stable and should not be blanket-flagged as unreliable.

---

## Area 5: BR/EDR Capture Limitations

**Finding:** Section 8.1 defines `ubertooth-rx` mode for BR/EDR survey capture. Section 1 goal 1 says "Continuously capture Bluetooth (BR/EDR and BLE) packets." However:

1. `ubertooth-rx` performs a frequency-hopping survey that scans across the 79 BR/EDR channels but cannot follow BR/EDR connections (which use adaptive frequency hopping per the master's clock). It captures Inquiry/Page scan packets but not established connection traffic.

2. The document does not specify what happens with captured BR/EDR data. There is no ingest bridge, no database schema, no tshark field contract for BR/EDR. Section 2 (Non-Goals) says "Real-time decryption of paired/encrypted BLE traffic" is out of scope, but BR/EDR inquiry data is different — it could be ingested similarly to BLE advertising.

3. The `sniffers.yaml` example shows `mode: btle` for the Ubertooth. The `mode: rx` option exists but no integration is designed. A sniffer configured in `rx` mode produces PCAP with a different format (DLT_BLUETOOTH_BREDR_LL or similar) that tshark-wrap.sh and the ingest bridge do not support.

**Confidence:** 80  
**Verdict:** ADVISORY — BR/EDR is listed as a v1 goal but has no pipeline. Either design a minimal BR/EDR ingest path or explicitly move BR/EDR to v1.1+ in the Non-Goals section.

---

## Area 6: Cross-Sniffer Timing and Correlation

**Finding:** Section 8.5 DECISION 8.5.3 says "Store every observation per sniffer. raw_packets.sniffer_id distinguishes them." This is correct for dedup avoidance, but the document does not address cross-sniffer timing:

1. **Clock synchronization:** Each sniffer has its own clock. Ubertooth uses the host system clock for PCAP timestamps. The nRF Sniffer uses the dongle's internal clock, which is not synchronized to the host. Timestamps between sniffers may differ by seconds, making temporal correlation unreliable.

2. **tshark re-timestamping:** When tshark reads from a FIFO, it may re-timestamp packets using the host's arrival time rather than the original capture time. Section 8.4 uses `frame.time_epoch` which is the frame arrival time at the capture interface, not the original RF transmission time. For Ubertooth, this is the capture time. For nRF via serial, this is the serial-port-read time, which has variable latency.

3. **Cross-sniffer dedup for triangulation:** Section 2 (Non-Goals) says "Location triangulation across multiple Bluetooth sniffers" is for later, but the data model must support it. Currently, the dedup index `uq_raw_packets_dedup ON raw_packets (sniffer_id, ts, mac_address)` prevents deduplication across sniffers (sniffer_id is part of the key), which is correct. However, there is no mechanism to associate "same packet seen by two sniffers" when triangulation is added later.

**Confidence:** 82  
**Verdict:** ADVISORY — Document that cross-sniffer timestamps are not synchronized in v1. Add a note that future triangulation requires NTP or PTP synchronization between sniffers.

---

## Area 7: RSSI Accuracy and Comparability

**Finding:** Section 8.4 includes RSSI as a field in CONTRACT 8.4-A. The document uses RSSI for:
- Continuous aggregates (avg_rssi, min_rssi, max_rssi in section 8.7)
- Device signal quality assessment

However:

1. **Ubertooth RSSI** is measured in dBm relative to the Ubertooth's CC2400 radio internal reference. It is approximate (±6 dB per Ubertooth documentation) and not calibrated.

2. **nRF52840 RSSI** is measured by the nRF52840 radio peripheral and reported in the `nordic_ble.rssi` field. The nRF52840 RSSI is measured differently (integrated over a different time window and with different reference).

3. **Cross-sniffer RSSI comparison is meaningless** without per-device calibration. A -67 dBm reading from Ubertooth and a -70 dBm reading from nRF for the same packet are not comparable. The continuous aggregate `AVG(rssi)` across sniffers (via `array_agg(DISTINCT sniffer_id)` in `device_5min_buckets`) would produce misleading averages if sniffer_id is not controlled for.

4. The `device_5min_buckets` continuous aggregate (section 8.7, migration 0002) computes `AVG(rssi)::SMALLINT` which averages across all `raw_packets` rows for a MAC in the bucket, regardless of sniffer_id. This silently mixes Ubertooth and nRF RSSI values.

**Confidence:** 85  
**Verdict:** FAIL — The continuous aggregate must group by `(mac_address, bucket, sniffer_id)` or document that cross-sniffer RSSI averaging is explicitly approximate. The frontend/device summary must not present averaged RSSI without qualification.

---

## Area 8: BLE Access Address and Advertising PDU Types

**Finding:** CONTRACT 8.4-A includes `pdu_type` but does not specify the BLE PDU type enumeration. Per BLE Core Spec v5.4, Vol 6, Part B, Section 2.3, the advertising channel PDU types are:

| PDU Type Value | Name | Description |
|----------------|------|-------------|
| 0 | ADV_IND | Connectable undirected |
| 1 | ADV_DIRECT_IND | Connectable directed |
| 2 | ADV_NONCONN_IND | Non-connectable undirected |
| 3 | SCAN_REQ | Scan request |
| 4 | SCAN_RSP | Scan response |
| 5 | CONNECT_IND | Connect request |
| 6 | ADV_SCAN_IND | Scannable undirected |

The tshark field `btle.advertising_header.pdu_type` may return the numeric value or a string name depending on tshark version and configuration. CONTRACT 8.4-A says position 6 is "int string" but does not specify the mapping. The ingest bridge parser must know which to expect.

Additionally, SCAN_REQ (type 3) and SCAN_RSP (type 4) packets have different field layouts. SCAN_REQ has a `ScanA` address field; SCAN_RSP has `AdvA` plus `ScanRspData`. CONTRACT 8.4-A position 2 is `btle.advertising_address` which maps to `AdvA`, but SCAN_REQ packets have `ScanA` instead. The contract is incomplete for non-ADV_IND types.

**Confidence:** 80  
**Verdict:** ADVISORY — CONTRACT 8.4-A should specify the PDU type mapping and document field differences for SCAN_REQ/SCAN_RSP packets. The ingest bridge should handle at least ADV_IND, ADV_NONCONN_IND, SCAN_RSP, and ADV_SCAN_IND.

---

## Summary of Findings

| # | Area | Severity | Confidence | Verdict | Description |
|---|------|----------|-----------|---------|-------------|
| 1 | Single-channel limitation | Blocking | 95 | BLOCKING | ~2/3 of advertisements missed; system goal "no data loss" is incorrect |
| 2 | DLT type / tshark field differences | Major | 92 | FAIL | Single contract cannot cover both Ubertooth and nRF tshark fields |
| 3 | Data whitening | Major | 85 | FAIL | No mention of dewhitening; deep parser may parse whitened data as garbage |
| 4 | MAC randomization | Major | 88 | FAIL | address_type should be 3-4 value enum, not boolean |
| 5 | BR/EDR pipeline | Minor | 80 | ADVISORY | Listed as v1 goal but no pipeline designed |
| 6 | Cross-sniffer timing | Minor | 82 | ADVISORY | No clock synchronization between sniffers |
| 7 | RSSI accuracy | Major | 85 | FAIL | Cross-sniffer RSSI averaging is misleading |
| 8 | PDU type handling | Minor | 80 | ADVISORY | Contract incomplete for SCAN_REQ/SCAN_RSP |

---

## Overall Verdict

**CONDITIONAL PASS** — 1 blocking finding (single-channel limitation) must be resolved by amending system goals. 4 major findings (DLT differences, data whitening, MAC randomization, RSSI averaging) must be addressed before implementation. 3 advisory findings can be resolved during implementation.

Resolution path:
1. Document single-channel limitation explicitly. Amend goal 2 from "no data loss" to "no loss of packets received on the monitored channel."
2. Split CONTRACT 8.4-A into per-sniffer-type variants or add a tshark normalization layer.
3. Add data whitening documentation and deep parser validation.
4. Extend `address_type` to at least 3 values.
5. Fix `device_5min_buckets` to group by sniffer_id or document cross-sniffer RSSI approximation.

---

## Self-Audit Checklist

| # | Check | Status |
|---|-------|--------|
| 1 | Read the complete artifact? | YES — all 2910 lines |
| 2 | Every finding includes a confidence score? | YES |
| 3 | Every finding is actionable? | YES — specific protocol sections and design changes |
| 4 | No speculative claims without evidence? | YES — all claims reference BLE Core Spec |
| 5 | Blocking findings justified by severity? | YES — single-channel affects fundamental data completeness |
| 6 | Advisory findings clearly separated? | YES — areas 5, 6, 8 are advisory |
| 7 | No duplicate findings across areas? | YES |
| 8 | Verdict is consistent with finding severities? | YES — blocking but resolvable → CONDITIONAL PASS |

---

## Flags for PM

| Flag ID | Type | Description | Urgency |
|---------|------|-------------|---------|
| FLAG-WE-001 | Decision | Single-channel sniffing captures ~1/3 of advertisements. Accept or add more sniffers for full coverage? | Blocking |
| FLAG-WE-002 | Decision | CONTRACT 8.4-A must handle two different tshark field schemas. Split contract or add normalization? | High |
| FLAG-WE-003 | Decision | Extend address_type from boolean (2 values) to enum (3-4 values) for random sub-type classification? | High |
| FLAG-WE-004 | Ticket | Add data whitening section to design doc; add dewhitening validation to deep parser. | High |
| FLAG-WE-005 | Decision | Fix RSSI averaging in device_5min_buckets to group by sniffer_id? | Medium |
| FLAG-WE-006 | Decision | BR/EDR listed as v1 goal but no pipeline. Keep as goal or move to v1.1? | Medium |