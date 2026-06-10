# C05 — Ingest Bridge

**Status:** Phase A Design — guides Phase B implementation
**Author:** Code Architect
**Date:** 2026-06-09
**Dependencies:** C02 (Database), C03 (Capture Pipeline)
**Blocks:** C06 (Gap Detector) — transitively

---

## 1. Overview

### 1.1 Purpose

C05 Ingest Bridge is the **hot-path data ingest pipeline** for the Tian'er Signal Intelligence Platform. It reads normalized pipe-delimited tshark output from the capture pipeline (C03) via a named FIFO, parses each line into a structured `Packet` record, accumulates packets into a batch, and streams the batch to PostgreSQL using the `COPY` protocol. It is the bridge between the real-time capture layer and the persistent time-series storage layer.

The ingest bridge is written in **C++17 with libpqxx 7.8**, per D-8.5.1 [5]. It is deployed as one process per sniffer instance (D-8.5.2) inside the `tianer-platform` Podman pod. It authenticates to PostgreSQL as the `tianer_writer` role (INSERT-only, no SELECT), per Q9.

### 1.2 Scope

| In Scope | Out of Scope |
|----------|-------------|
| Parse pipe-delimited tshark lines (CONTRACT INGEST-1) into `Packet` structs | tshark invocation and field configuration (C03) |
| Time- and size-based batching per D-09 (300K rows / 5 min) | Sniffer operation and PCAP writing (C03) |
| Batch `COPY` to `bluetooth.raw_packets` via `pqxx::stream_to` | PCAP rotation (C04) |
| Reconnection to PostgreSQL with exponential backoff | Gap detection and PCAP backfill (C06) |
| Metric exposition via SIGUSR1 | Prometheus scraping / node_exporter textfile (C13) |
| Structured logging (JSON to stderr) | Log aggregation and Grafana dashboards (C13, C11) |
| Input validation, field range checks, length limits | Deep protocol dissection — AdvData is stored raw (C07 handles this) |
| Signal handling (SIGTERM graceful shutdown, SIGUSR1 metrics dump) | Heartbeat generation (C03) |

### 1.3 Boundaries

C05 lives between the capture pipeline (C03) and the database (C02). It reads from the FIFO at V03 and writes to PostgreSQL at V06. It does **not** own the FIFO creation (C01 tmpfiles.d), the tshark process (C03), the database schema (C02), or the pod definition (C14).

```
┌─────────────────────────────────────────────────────────────────┐
│                      tianer-platform pod                         │
│                                                                 │
│  ┌─────────────────┐     ┌─────────────────┐                   │
│  │ C03 Capture      │     │ C05 Ingest      │                   │
│  │ Pipeline         │     │ Bridge (C++)    │                   │
│  │                  │     │                 │                   │
│  │ tshark@ writes   │────▶│ parse_line()    │                   │
│  │ → V03            │     │   ↓             │                   │
│  │ (-ingest.fifo)   │     │ Batcher::add()  │                   │
│  │                  │     │   ↓             │                   │
│  │                  │     │ PgWriter::write │                   │
│  │                  │     │   ↓             │                   │
│  │                  │     │ pqxx::stream_to │                   │
│  └─────────────────┘     └────────┬────────┘                   │
│                                   │                             │
└───────────────────────────────────┼─────────────────────────────┘
                                    │ tianer_writer
                                    │ scram-sha-256
                                    │ COPY protocol
                                    ▼
                         ┌──────────────────────┐
                         │ C02 Database          │
                         │ tianer-postgres:5432  │
                         │                        │
                         │ bluetooth.raw_packets  │
                         │ (hypertable)           │
                         └──────────────────────┘
```

### 1.4 Position in the System

C05 is a **Layer 1 component** in the build sequence (component-breakdown.md §4.1). It must be built after C02 (database schema must exist) and C03 (FIFO must be available). It blocks C06 (gap detector depends on ingest being operational to validate its detection logic) and transitively blocks C07-C10.

C05 is part of the **v1 Bluetooth module** (`blesniff` prefix). The binary is named `blesniff-ingest`. Future sensor modules will add their own ingest bridges (e.g. `gpsrx-ingest`, `adsbrx-ingest`) following the same pattern.

---

## 2. High-Level Architecture (HLA)

### 2.1 Component Architecture

The ingest bridge is a single-threaded, event-driven pipeline with three sequential stages:

```
stdin (FIFO) ──▶  Parser  ──▶  Batcher  ──▶  PgWriter  ──▶ PostgreSQL
                 parse_line()   Batcher     PgWriter        bluetooth.raw_packets
                    │              │            │
                    ▼              ▼            ▼
               malformed       buffer       reconnect
               counter        depth         backoff
                              gauge
```

| Stage | Component | Input | Output | Failure Mode |
|-------|-----------|-------|--------|-------------|
| **Parse** | `parse_line(std::string_view)` | Pipe-delimited line (INGEST-1 format) | `std::optional<Packet>` | Malformed line → `std::nullopt`, increment counter |
| **Batch** | `Batcher` | `Packet` structs | `std::vector<Packet>` at flush time | Buffer overflow → oldest drop + counter increment |
| **Write** | `PgWriter` | `std::vector<Packet>` | `pqxx::stream_to` → `raw_packets` | Connection loss → reconnect with backoff, buffer held |

### 2.2 Data Flow

1. **FIFO read:** The process opens `/var/run/tianer/<name>-ingest.fifo` (V03) for reading. This is a blocking read — the process waits for tshark to produce data. The FIFO is created by C01 `tmpfiles.d` and written to by C03 tshark wrapper.

2. **Line read:** `std::getline()` reads one line at a time from stdin. tshark output is line-buffered per C03, so complete lines arrive without delay.

3. **Parse:** `parse_line()` tokenizes the pipe-delimited line against the INGEST-1 contract. Valid lines produce a `Packet` struct. Malformed lines produce `std::nullopt` and increment the `malformed_packets_total` counter. The `std::optional` pattern [4] provides a type-safe alternative to error codes or null pointers, making the "parse failure" path explicit and impossible to ignore.

4. **Batch:** `Batcher::add()` appends the `Packet` to an internal `std::vector`. When the batch reaches the size threshold (300K rows per D-09) or the time threshold (5 minutes per D-09), the batcher signals that a flush is due.

5. **Write:** `PgWriter::write()` consumes the batch via `drain()`, constructs an `pqxx::stream_to` against `bluetooth.raw_packets`, and streams the rows in a single `COPY` transaction. If the connection is down, `PgWriter` attempts reconnection with exponential backoff (1s, 2s, 4s, …, max 30s). During reconnection, the batcher continues accumulating up to its maximum buffer capacity.

### 2.3 Module Layout

```
modules/bluetooth/ingest-bridge/
├── CMakeLists.txt                    # CMake build for the ingest bridge
├── src/
│   ├── main.cpp                      # Entry point: arg parsing, signal handlers, main loop
│   ├── config.hpp                    # Configuration struct, BLESNIFF_* env var parsing
│   ├── parser.hpp                    # parse_line() declaration, Packet struct
│   ├── parser.cpp                    # parse_line() implementation
│   ├── batcher.hpp                   # Batcher class declaration
│   ├── batcher.cpp                   # Batcher implementation
│   └── pg_writer.hpp                 # PgWriter class declaration
│   └── pg_writer.cpp                 # PgWriter implementation (libpqxx COPY)
├── tests/
│   ├── CMakeLists.txt                # Test build configuration
│   ├── parser_test.cpp               # parse_line() unit tests (GoogleTest)
│   ├── batcher_test.cpp              # Batcher unit tests (GoogleTest)
│   └── pg_writer_test.cpp            # PgWriter integration tests (needs DB)
└── deploy/
    └── Containerfile.ingest          # Multi-stage build: builder stage + slim runtime
```

---

## 3. Data Model (ERD)

### 3.1 Packet Struct ↔ raw_packets Row Mapping

The `Packet` struct is the in-memory representation of a single tshark output line. It maps directly to a row in `bluetooth.raw_packets`.

```cpp
/**
 * @brief In-memory representation of one tshark output line.
 *
 * Maps 1:1 to a row in bluetooth.raw_packets. All fields use the
 * smallest type that can hold the legal value range.
 *
 * @code
 *   auto pkt = parse_line("1717689600.123456|aa:bb:cc:dd:ee:ff|1|-67|37|0|02011a");
 *   // pkt->ts == system_clock::from_time_t(1717689600) + 123456us
 *   // pkt->rssi == -67
 *   // pkt->mac has_value() == true
 * @endcode
 */
struct Packet {
    /// Timestamp from tshark frame.time_epoch (seconds since epoch, fractional).
    /// Unit: microseconds precision via system_clock::time_point.
    std::chrono::system_clock::time_point ts;

    /// Bluetooth device address (BD_ADDR), 6 bytes.
    /// std::nullopt when address is missing (non-ADV packets).
    std::optional<std::array<uint8_t, 6>> mac;

    /// Address type: true = random, false = public.
    /// std::nullopt when not determinable.
    std::optional<bool> address_random;

    /// Received Signal Strength Indicator in dBm.
    /// Legal range: -128 to +20 (BLE spec); typical range: -100 to -20.
    /// std::nullopt when RSSI is not reported.
    std::optional<int16_t> rssi;

    /// BLE advertising channel index.
    /// 37 (2402 MHz), 38 (2426 MHz), 39 (2480 MHz).
    /// std::nullopt when channel is not reported.
    std::optional<int16_t> channel;

    /// BLE advertising PDU type per Core Spec Vol 6, Part B, Section 2.3.
    /// Common values: 0=ADV_IND, 1=ADV_DIRECT_IND, 2=ADV_NONCONN_IND,
    ///               3=SCAN_REQ, 4=SCAN_RSP, 5=CONNECT_IND, 6=ADV_SCAN_IND.
    /// std::nullopt when PDU type is not reported.
    std::optional<int16_t> pdu_type;

    /// Raw advertising data payload bytes. Up to 31 bytes for legacy advertising.
    /// May be empty (zero-length vector) for PDUs without advertising data.
    std::vector<uint8_t> advdata;
};
```

### 3.2 Column Mapping Reference

The mapping from `Packet` fields to `raw_packets` columns follows the C02 database schema (§3.3.1, §3.3.2):

| `Packet` Field | `raw_packets` Column | SQL Type | Mapping |
|---------------|---------------------|----------|---------|
| `ts` | `ts` | `TIMESTAMPTZ NOT NULL` | `to_epoch_micros(ts)` via `pqxx::stream_to` field |
| `sniffer_id` (constructor param) | `sniffer_id` | `SMALLINT NOT NULL` | Injected by `PgWriter` constructor; not parsed from input |
| `mac` | `mac_address` | `BYTEA NOT NULL` | `mac.value()` → 6 raw bytes; `std::nullopt` → empty BYTEA (treated as 6 zero bytes by DB) |
| `address_random` | `address_type` | `SMALLINT` | `true` → `1`, `false` → `0`, `std::nullopt` → SQL NULL |
| `rssi` | `rssi` | `SMALLINT` | Direct value; `std::nullopt` → SQL NULL |
| `channel` | `channel` | `SMALLINT` | Direct value; `std::nullopt` → SQL NULL |
| `pdu_type` | `pdu_type` | `SMALLINT` | Direct value; `std::nullopt` → SQL NULL |
| `advdata` | `advdata` | `BYTEA` | Raw bytes; empty vector → empty BYTEA |
| (hardcoded)`false` | `backfilled` | `BOOLEAN NOT NULL DEFAULT FALSE` | Always `FALSE` for live ingest. Only gap detector sets `TRUE`. |

### 3.3 Deduplication (D-10)

The ingest bridge performs **no deduplication at insert time**. Per D-10, deduplication happens at **query time** via `DISTINCT ON (sniffer_id, ts, mac_address, pdu_type)`. This preserves all observations from different sniffers (needed for future triangulation, D-8.5.3) and allows the gap detector to backfill overlapping time windows without INSERT conflicts.

The `backfilled` column is always `false` for live-ingested rows. The gap detector sets it to `true` for backfilled rows. Query-time dedup with `ORDER BY backfilled DESC` prefers backfilled rows (which may carry richer metadata from C07 deep parsing).

### 3.4 Row Size Budget

| Field | Bytes (approx) |
|-------|---------------|
| `ts` (TIMESTAMPTZ) | 8 |
| `sniffer_id` (SMALLINT) | 2 |
| `mac_address` (BYTEA) | 6+1 (BYTEA header) |
| `address_type` (SMALLINT) | 2+nullable |
| `rssi` (SMALLINT) | 2+nullable |
| `channel` (SMALLINT) | 2+nullable |
| `pdu_type` (SMALLINT) | 2+nullable |
| `advdata` (BYTEA) | ~0–31+1 (typically ~15 average) |
| PostgreSQL tuple overhead | ~28 |
| **Total per row (average)** | **~60 bytes** |

At 300K rows per sniffer buffer, memory usage is approximately 300,000 × 60 = **~18 MB**. The D-09 budget of 60 MB per sniffer (originally estimated at 80 bytes/row) provides ample headroom.

---

## 4. Low-Level Architecture (LLA)

### 4.1 parse_line() State Machine

`parse_line()` implements a single-pass tokenizer over `std::string_view`, avoiding heap allocations for string splitting. The INGEST-1 contract defines exactly 7 pipe-delimited fields (see also C03 §4.3.1 and §5.2 — CAPTURE-2, which is the producer side of this same schema):

```
<epoch_sec>.<frac>|<mac>|<addr_type>|<rssi>|<channel>|<pdu_type>|<advdata_hex>
```

The sniffer name is not present in the line — it is injected as `sniffer_id` by the `PgWriter` constructor (§4.3) and mapped to the `raw_packets.sniffer_id` column (§3.2).

The parser walks the string character-by-character, extracting each field into the corresponding `Packet` member. A state machine ensures exactly 7 fields are present.

```
START → FIELD_TS → DELIM1 → FIELD_MAC → DELIM2 → FIELD_ADDR_TYPE
       → DELIM3 → FIELD_RSSI → DELIM4 → FIELD_CHANNEL
       → DELIM5 → FIELD_PDU_TYPE → DELIM6 → FIELD_ADVDATA → END
```

**Field parsing rules:**

| State | Validation | Error Handling |
|-------|-----------|---------------|
| FIELD_TS | Must parse as `double` (seconds.fractional). Fractional part converted to microseconds, added to `system_clock::from_time_t(integral_seconds)`. | Return `std::nullopt` |
| FIELD_MAC | Must be exactly 17 characters: `xx:xx:xx:xx:xx:xx` colon-hex format. Each pair validated as hex; colon at positions 2,5,8,11,14. | Return `std::nullopt`. Empty field (`||`) → `std::nullopt` for `mac` — field is allowed to be empty per INGEST-1. |
| FIELD_ADDR_TYPE | Must be `"0"` or `"1"`. Empty → `std::nullopt`. | Any other value → `std::nullopt` for `address_random` (not a parse failure) |
| FIELD_RSSI | Integer string. Empty → `std::nullopt`. Range: `[-128, 20]`. | Out of range → clamp to nearest bound and log WARNING. Non-integer → `std::nullopt` |
| FIELD_CHANNEL | Integer string. Empty → `std::nullopt`. Range: `[37, 39]` for BLE advertising channels (others accepted but logged). | Non-integer → `std::nullopt` |
| FIELD_PDU_TYPE | Integer string. Empty → `std::nullopt`. Range: `[0, 6]` for BLE advertising PDUs. | Non-integer → `std::nullopt` |
| FIELD_ADVDATA | Hex string (even length). Empty → empty vector. Length limit: 62 hex chars → 31 bytes (BLE legacy advertising max). | Non-hex characters → return `std::nullopt`. Length > 62 → truncate and log WARNING. |

**Missing fields:** If fewer than 7 delimiters are found, return `std::nullopt`.

**Empty fields:** Per INGEST-1, empty fields between pipes are valid and result in `std::nullopt` for that field. An entirely empty line (tshark parse failure per D-8.4.2) produces `std::nullopt` with a `malformed_packets_total` increment.

### 4.2 Batcher Flush Policy (D-09)

The `Batcher` class implements dual-threshold flush logic. A flush is triggered when **either** condition is met:

```
FLUSH_CONDITION = (row_count >= MAX_ROWS) OR (elapsed >= MAX_LATENCY)
```

| Parameter | Value | Source | Rationale |
|-----------|-------|--------|-----------|
| `MAX_ROWS` | 300,000 | D-09 | Covers 5 minutes of DB downtime at 1000 pps per sniffer. The `Batcher` may grow beyond this during DB outage — see §6.1. |
| `MAX_LATENCY` | 5 minutes (300,000 ms) | D-09 | Ensures rows are visible in the database within 5 minutes even at low packet rates. |
| Overflow upper bound | ~500,000 rows | D-09 (implicit) | Hard cap: when the internal vector exceeds this during a prolonged outage, oldest rows are dropped with a counter increment. See §4.2.1. |

```cpp
/**
 * @brief Accumulates parsed Packets and flushes on dual-threshold policy.
 *
 * Flushes when either the row count or time threshold is reached.
 * During DB outages, continues accumulating beyond MAX_ROWS up to
 * the overflow bound; oldest rows are evicted beyond that.
 *
 * @code
 *   Batcher batcher(BATCH_MAX_ROWS, BATCH_MAX_LATENCY);
 *   while (auto pkt = parse_line(line)) {
 *       if (batcher.add(*pkt)) {
 *           auto batch = batcher.drain();
 *           pg_writer.write(batch);
 *       }
 *   }
 * @endcode
 */
class Batcher {
public:
    /**
     * @param max_rows     Flush threshold (row count). D-09: 300,000.
     * @param max_latency  Flush threshold (time since first buffered row). D-09: 5 min.
     */
    Batcher(size_t max_rows, std::chrono::milliseconds max_latency);

    /**
     * @brief Add a packet to the batch.
     * @return true if the batch should be flushed now.
     */
    bool add(Packet p);

    /**
     * @brief Drain the current batch and reset.
     * @return The accumulated packets, oldest first.
     */
    std::vector<Packet> drain();

    /// Current number of buffered rows (for metrics gauge).
    size_t depth() const;

    /// Total number of rows evicted due to overflow.
    uint64_t evicted_count() const;
};
```

#### 4.2.1 Overflow Eviction During DB Outage

When `PgWriter` cannot connect (PG outage), `add()` continues to be called. The batcher accumulates beyond `MAX_ROWS` up to a **hard upper bound** of ~500,000 rows (~30 MB). Beyond this bound, the oldest row in the buffer is evicted and `evicted_count()` is incremented.

This implements **graceful degradation** (PF-4): during a prolonged PG outage, the batcher preserves the most recent packets and drops the oldest, rather than crashing or growing unbounded. The eviction counter is exposed as a metric so the operator can quantify data loss. The **gap detector (C06)** identifies the outage window from heartbeat gaps and backfills from PCAP (the source of truth).

#### 4.2.2 Flush-on-Empty

When stdin closes (tshark exits / FIFO writer closes), `main()` calls `batcher.drain()` to flush the final partial batch to the database before exiting with code 0.

### 4.3 PgWriter — COPY Protocol & Reconnection

`PgWriter` manages the single `pqxx::connection` to PostgreSQL and provides a streaming `write()` interface for batched packets.

```cpp
/**
 * @brief Writes batches of Packets to bluetooth.raw_packets via COPY protocol.
 *
 * Uses tianer_writer role (INSERT-only, no SELECT per Q9).
 * No raw SQL string building — all values pass through pqxx::stream_to
 * which uses parameterized COPY internally.
 *
 * Reconnection uses exponential backoff during PG outages.
 *
 * @code
 *   PgWriter writer(cfg, sniffer_id);
 *   writer.write(batch);  // throws on unrecoverable failure
 * @endcode
 */
class PgWriter {
public:
    /**
     * @param cfg        Parsed configuration (PG host, port, credentials).
     * @param sniffer_id  Identifier for this sniffer instance.
     */
    explicit PgWriter(const Config& cfg, int16_t sniffer_id);

    /**
     * @brief Stream a batch of packets to PostgreSQL via COPY.
     *
     * Blocks until the entire batch is sent. On connection failure,
     * attempts reconnection with exponential backoff.
     *
     * @param batch  Span of packets to write. Must not be empty.
     * @throws std::runtime_error on unrecoverable failure after
     *         all reconnection attempts are exhausted.
     */
    void write(std::span<const Packet> batch);

    /// True if the connection is currently established.
    bool is_connected() const;

    /// Total number of successful flushes.
    uint64_t flush_count() const;

private:
    pqxx::connection conn_;
    int16_t sniffer_id_;

    /// Attempt to (re)connect. Returns true on success.
    bool connect();

    /// Compute the next backoff delay (1s, 2s, 4s, ..., max 30s).
    std::chrono::milliseconds next_backoff() const;

    int backoff_attempt_ = 0;
    static constexpr int MAX_BACKOFF_ATTEMPTS = 10;
};
```

#### 4.3.1 COPY Protocol Details

`PgWriter::write()` uses `pqxx::stream_to` to construct a `COPY bluetooth.raw_packets (ts, sniffer_id, mac_address, address_type, rssi, channel, pdu_type, advdata, backfilled) FROM STDIN` stream [3]. Each `Packet` in the batch is serialized into tab-separated field format (`\t`, the PostgreSQL COPY text format default delimiter [3]), with `\N` representing SQL NULL for optional fields (the PostgreSQL COPY default null string [3]). The `stream_to::complete()` call flushes the stream and returns the number of rows written.

**libpqxx handles all escaping internally** — there is no raw SQL string building, and thus no SQL injection surface.

#### 4.3.2 Reconnection with Exponential Backoff

```
BACKOFF_SEQUENCE: 1s → 2s → 4s → 8s → 16s → 30s → 30s → 30s → 30s → 30s (max)
```

The connection timeout behaviour and retry strategy follow best practices for PostgreSQL client resilience: the `connect_timeout` parameter at the libpq level provides a per-attempt timeout; application-level exponential backoff with a ceiling provides bounded retry without thundering-herd effects [1, §32.1.2].

| Attempt | Delay | Cumulative wait |
|---------|-------|-----------------|
| 1 | 1s | 1s |
| 2 | 2s | 3s |
| 3 | 4s | 7s |
| 4 | 8s | 15s |
| 5 | 16s | 31s |
| 6 | 30s | 61s |
| 7 | 30s | 91s |
| 8 | 30s | 121s |
| 9 | 30s | 151s |
| 10 | 30s | 181s |

After 10 failed attempts (approximately 3 minutes), the process logs a CRITICAL error and exits. systemd/Podman restarts the container, restarting the backoff sequence from 1. This prevents an infinite hang while still providing sustained retry coverage.

During the reconnection attempt, the batcher continues buffering from the FIFO. The FIFO provides backpressure: when it fills (64KB kernel pipe buffer per the Linux default since 2.6.11 [2]), `tail -f` (C03) blocks on write, which pauses tshark output, which pauses the FIFO reader. The PCAP capture (V02) continues unaffected per the decoupled capture architecture (storage-strategy.md §Backpressure Isolation).

### 4.4 Main Loop

```cpp
// Pseudocode — actual implementation in main.cpp
int main(int argc, char* argv[]) {
    Config cfg = parse_args_and_env(argc, argv);
    PgWriter writer(cfg, cfg.sniffer_id);
    Batcher batcher(MAX_ROWS, MAX_LATENCY);

    register_signal_handlers();  // SIGTERM → graceful shutdown, SIGUSR1 → metrics dump

    std::string line;
    while (std::getline(std::cin, line)) {
        auto pkt = parse_line(line);
        if (!pkt) {
            malformed_count++;
            continue;
        }

        if (batcher.add(std::move(*pkt))) {
            writer.write(batcher.drain());
        }
    }

    // stdin closed — flush final batch
    if (batcher.depth() > 0) {
        writer.write(batcher.drain());
    }

    return EXIT_SUCCESS;
}
```

### 4.5 Signal Handling

| Signal | Behavior |
|--------|----------|
| `SIGTERM` | Drain final batch to DB (with timeout), exit 0. If DB is unreachable, log ERROR and exit 0 (data is recoverable from PCAP via gap detector). |
| `SIGUSR1` | Write one-line Prometheus-format metric snapshot to stderr (see §7.2). Does not interrupt processing. |
| `SIGINT` | Same as SIGTERM. |

### 4.6 Graceful Shutdown Sequence

1. `SIGTERM` received → set `shutdown_requested` flag.
2. `std::getline()` returns EOF due to tshark exiting (C03 orchestrates shutdown order).
3. Main loop exits. Final batch drained to DB.
4. `PgWriter` destructor closes `pqxx::connection`.
5. Metrics summary logged to stderr.
6. Exit code 0.

---

## 5. Inter-Component Contracts

### 5.1 INGEST-1: Normalized tshark Input Schema

**Source:** C03 Capture Pipeline (tshark wrapper, §4.3.1 and §5.2 — CAPTURE-2 contract)
**Consumer:** C05 Ingest Bridge
**Transport:** Named FIFO at `/var/run/tianer/<name>-ingest.fifo` (V03), one per sniffer

This schema is the consumer-side specification of the CAPTURE-2 / INGEST-1 contract. The producer side is documented in C03 §4.3.1 and §5.2. Both sides agree on exactly 7 pipe-delimited fields.

```
<epoch_seconds>.<fractional>|<mac_address>|<address_type>|<rssi>|<channel>|<pdu_type>|<advdata_hex>
```

| Position | Field | Type | Empty Allowed | Example | Notes |
|----------|-------|------|---------------|---------|-------|
| 1 | `ts` | Float string (seconds.fractional) | No | `1717689600.123456` | `frame.time_epoch` from tshark |
| 2 | `mac_address` | Colon-hex string (17 chars) | Yes | `aa:bb:cc:dd:ee:ff` | BD_ADDR. Empty for non-ADV packets. |
| 3 | `address_type` | `"0"` or `"1"` | Yes | `"1"` | `0` = public, `1` = random. From tshark `btle.advertising_address.address_type`. |
| 4 | `rssi` | Integer string | Yes | `-67` | dBm. From tshark `btle.rssi` or `radiotap.dbm_antsignal`. |
| 5 | `channel` | Integer string | Yes | `37` | BLE advertising channel: 37, 38, or 39 (or other for future modules). |
| 6 | `pdu_type` | Integer string | Yes | `0` | BLE PDU type per Core Spec: 0=ADV_IND, 1=ADV_DIRECT_IND, etc. |
| 7 | `advdata` | Hex string (even length) | Yes | `02011a0303aafe17...` | Raw advertising data payload, hex-encoded. Up to 62 hex chars (31 bytes) for legacy BLE. |

**Line terminators:** `\n` (LF). tshark output is line-buffered per C03.

**Empty-field convention:** When a field is not available (e.g. `rssi` for a non-ADV packet), it appears as an empty string between pipes: `||`. All fields except `ts` allow empty.

### 5.2 INGEST-2: COPY Protocol

**Source:** C05 Ingest Bridge (PgWriter)
**Consumer:** C02 Database (PostgreSQL)
**Transport:** TCP to `tianer-postgres:5432` on `tianer-net` bridge network

| Property | Value |
|----------|-------|
| Protocol | PostgreSQL `COPY ... FROM STDIN` |
| Target table | `bluetooth.raw_packets` |
| Authentication | SCRAM-SHA-256 as `tianer_writer` role |
| Columns | `(ts, sniffer_id, mac_address, address_type, rssi, channel, pdu_type, advdata, backfilled)` |
| Field delimiter | Tab (`\t`) — pqxx default for `stream_to`; matches PostgreSQL COPY text format default [3] |
| NULL representation | `\N` — pqxx default; matches PostgreSQL COPY text format default null string [3] |
| Batch size | Up to 300K rows per flush per D-09 |
| Transaction | Implicit within `stream_to::complete()` |
| Error handling | `pqxx::stream_to` throws `pqxx::sql_error` on constraint violation or connection loss |

**Role privilege:** The `tianer_writer` role has INSERT and COPY privileges on `bluetooth.raw_packets` but **no SELECT, UPDATE, DELETE, TRUNCATE, or DDL** (per Q9). This means:
- `COPY ... FROM STDIN` succeeds (INSERT equivalent).
- If the ingest bridge were compromised, the attacker cannot read sensitive data or destroy existing rows.
- The `backfilled` column is hardcoded to `FALSE` — the ingest bridge cannot alter backfill metadata.

---

## 6. Failure Modes & Recovery

### 6.1 Failure Catalog

| ID | Failure | Detection | Propagation | Recovery | SLA |
|----|---------|-----------|-------------|----------|-----|
| **F-C05-1** | **PostgreSQL outage** | `pqxx::stream_to` throws `pqxx::broken_connection`. `PgWriter::write()` catches and enters reconnect loop. | Batcher continues buffering. FIFO backpressure pauses tshark. Sniffer continues capturing to PCAP unaffected. | Exponential backoff reconnect (1s–30s). On reconnect, buffered batch is flushed. Gap detector identifies outage window and backfills from PCAP. | No PCAP data loss. Ingest gap recovered within gap detector cycle (5 min). |
| **F-C05-2** | **Buffer overflow during prolonged PG outage** | Batcher row count exceeds overflow bound (~500K rows). Oldest rows evicted with `evicted_count` increment. | `blesniff_packets_dropped_total` metric increments. WARNING log emitted. | Gap detector identifies outage window from heartbeat gap. Backfills all packets from PCAP. Evicted rows lost from real-time path but recovered from PCAP — no permanent loss. | Zero permanent data loss. Eviction window is recoverable from PCAP within gap detector cycle. |
| **F-C05-3** | **Malformed tshark line** | `parse_line()` returns `std::nullopt`. `malformed_packets_total` counter incremented. | Individual line skipped. Subsequent lines processed normally. | After 100 consecutive malformed lines, log ERROR and the alert `TianerMalformedPacketsHigh` fires (C13). This threshold prevents log spam from a broken tshark configuration while detecting systemic issues. | Single malformed lines: no impact. Systemic malformation: alert within ~0.1s at 1000 pps. |
| **F-C05-4** | **FIFO reader blocks indefinitely** | `std::getline()` blocks when FIFO writer pauses (e.g., tshark crashes). | No data loss — PCAP capture continues. | C03 heartbeat detects tshark crash. Gap detector identifies window. No action needed in C05. | No data loss. Recovery window: gap detector cycle. |
| **F-C05-5** | **stdin closes** (tshark exits gracefully) | `std::getline()` returns EOF. | Remaining batch drained to DB. Process exits 0. | Podman/systemd restarts ingest bridge when tshark is restarted. | No data loss. Partial batch flushed before exit. |
| **F-C05-6** | **OOM kill** | Kernel OOM killer sends SIGKILL. Process dies without draining. | All in-flight packets in the batcher are lost from the real-time path. | systemd/Podman restarts the container. Gap detector identifies gap from heartbeat and backfills from PCAP. | Zero permanent data loss. In-flight loss bounded to one batch (~300K rows). Recovered from PCAP by gap detector within its next cycle. |
| **F-C05-7** | **Partial batch loss due to crash during COPY** | `pqxx::stream_to` partially completes before crash. PostgreSQL may commit the partial COPY if the TCP session breaks mid-stream. | A fraction of the batch (~0–300K rows) makes it to `raw_packets`. The remainder is lost from the real-time path. | Gap detector compares PCAP row counts to DB row counts during secondary verification. Identifies the gap window and backfills. | Zero permanent data loss. Recovery from PCAP via gap detector. |

### 6.2 Blast Radius Analysis

| Failure | Blast Radius | Justification |
|---------|-------------|---------------|
| F-C05-1 (PG outage) | **One sniffer's real-time ingest** | One process per sniffer means each bridge fails independently. Other sniffers' bridges (separate processes) continue ingesting. |
| F-C05-2 (buffer overflow) | **Oldest ~200K rows evicted** | Dropped rows are recoverable from PCAP. Gap detector backfills. |
| F-C05-6 (OOM kill) | **One batch (~300K rows) lost from real-time path** | Recoverable from PCAP within 5 minutes. Other sniffer bridges unaffected. |
| F-C05-7 (partial batch) | **Subset of one batch** | Recoverable from PCAP. Gap detector's secondary verification catches this. |

### 6.3 Crash-Only Design (PF-8)

The ingest bridge can be killed at any time (SIGKILL, OOM, power loss) without permanent data loss:
- **PCAP is the source of truth.** Every packet the sniffer captures is written to disk before reaching the ingest bridge.
- **No clean shutdown required.** The gap detector identifies the outage window from heartbeat data and backfills from PCAP.
- **No in-memory state that cannot be recovered.** The batcher's in-flight packets are the only transient state, and they are recoverable from PCAP.
- **COPY is transactional per batch.** If the COPY partially completes before a crash, the committed rows are a valid subset. The gap detector backfills any that were missed.

---

## 7. Observability

### 7.1 Metrics Catalogue

All metrics are exposed as file-based Prometheus text format, consistent with D-08. On `SIGUSR1`, the process writes the current metric snapshot to stderr. C13 (Observability) collects stderr output and writes to the Prometheus textfile directory.

| Metric | Type | Description | Labels |
|--------|------|-------------|--------|
| `blesniff_packets_ingested_total` | Counter | Total packets successfully written to PostgreSQL | `sniffer_id`, `sniffer_name` |
| `blesniff_packets_received_total` | Counter | Total packets parsed from tshark input (valid + malformed) | `sniffer_id`, `sniffer_name` |
| `blesniff_malformed_packets_total` | Counter | Packets rejected by `parse_line()` | `sniffer_id`, `sniffer_name` |
| `blesniff_batches_flushed_total` | Counter | Number of successful COPY operations | `sniffer_id`, `sniffer_name` |
| `blesniff_packets_dropped_total` | Counter | Packets evicted from buffer during overflow | `sniffer_id`, `sniffer_name` |
| `blesniff_reconnect_attempts_total` | Counter | Number of reconnection attempts | `sniffer_id`, `sniffer_name` |
| `blesniff_ingest_latency_seconds` | Histogram | Time from line read to COPY completion (per batch flush) | `sniffer_id`, `sniffer_name` |
| `blesniff_buffer_depth` | Gauge | Current number of buffered rows awaiting flush | `sniffer_id`, `sniffer_name` |
| `blesniff_connection_state` | Gauge | 1 if connected to PG, 0 if disconnected | `sniffer_id`, `sniffer_name` |

**Example SIGUSR1 dump:**
```
# HELP blesniff_packets_ingested_total Total packets written to PostgreSQL.
# TYPE blesniff_packets_ingested_total counter
blesniff_packets_ingested_total{sniffer_id="1",sniffer_name="ubertooth0"} 458320
blesniff_malformed_packets_total{sniffer_id="1",sniffer_name="ubertooth0"} 23
blesniff_batches_flushed_total{sniffer_id="1",sniffer_name="ubertooth0"} 156
blesniff_packets_dropped_total{sniffer_id="1",sniffer_name="ubertooth0"} 0
blesniff_reconnect_attempts_total{sniffer_id="1",sniffer_name="ubertooth0"} 0
blesniff_buffer_depth{sniffer_id="1",sniffer_name="ubertooth0"} 12450
blesniff_connection_state{sniffer_id="1",sniffer_name="ubertooth0"} 1
```

### 7.2 Structured Logging

All log output goes to stderr in JSON Lines format, consistent with the C13 observability standard:

```json
{"ts":"2026-06-09T12:34:56.789Z","level":"INFO","component":"blesniff-ingest","sniffer_id":1,"sniffer_name":"ubertooth0","msg":"Batch flushed","rows_flushed":285000,"flush_count":156,"latency_ms":312}
{"ts":"2026-06-09T12:35:01.123Z","level":"WARN","component":"blesniff-ingest","sniffer_id":1,"sniffer_name":"ubertooth0","msg":"Reconnecting to PostgreSQL","attempt":3,"backoff_ms":4000}
{"ts":"2026-06-09T12:35:05.456Z","level":"ERROR","component":"blesniff-ingest","sniffer_id":1,"sniffer_name":"ubertooth0","msg":"Consecutive malformed lines threshold exceeded","consecutive":100}
{"ts":"2026-06-09T12:35:10.789Z","level":"CRITICAL","component":"blesniff-ingest","sniffer_id":1,"sniffer_name":"ubertooth0","msg":"Max reconnection attempts exhausted","total_attempts":10,"exiting":true}
```

**Log level usage:**

| Level | When |
|-------|------|
| `INFO` | Batch flushed, startup, graceful shutdown, connection established |
| `WARN` | Single malformed line, RSSI/channel out of expected range, reconnection attempt in progress, buffer approaching capacity (>80%) |
| `ERROR` | Consecutive malformed threshold met, COPY failure after reconnect, unexpected field value |
| `CRITICAL` | Max reconnection attempts exhausted, process exiting |

### 7.3 Health Check

The ingest bridge has no HTTP health endpoint (it's a C++ process, not a web server per D-08). Health is determined by:
- **Process liveness:** systemd/Podman monitors the process. If it exits with non-zero, auto-restart.
- **Connection state:** `blesniff_connection_state` gauge. If 0 for >5 minutes, alert `TianerIngestDisconnected` fires.
- **Butter depth:** If `blesniff_buffer_depth` > 90% of max capacity for >1 minute, alert fires.
- **Gap detection:** C06 independently verifies ingest is producing data via heartbeat and row-count comparisons.

### 7.4 Alert Rules

| Alert | Condition | Severity | Action |
|-------|-----------|----------|--------|
| `TianerIngestDisconnected` | `blesniff_connection_state == 0` for 5 minutes | Critical | Investigate PostgreSQL health; check network; check credentials |
| `TianerIngestMalformedHigh` | 100 consecutive `blesniff_malformed_packets_total` increments | Warning | Check tshark configuration; verify DLT type matches |
| `TianerIngestDroppingPackets` | `blesniff_packets_dropped_total` increases | Warning | PG outage or throughput bottleneck — gap detector will backfill |
| `TianerIngestBufferFull` | `blesniff_buffer_depth` > 90% capacity for 60s | Warning | Investigate PG write latency; may need to scale sniffer instances |

---

## 8. Security Considerations

### 8.1 Attack Surface

| Surface | Risk | Mitigation |
|---------|------|-----------|
| **FIFO input** (V03) | Malicious tshark output (compromised C03 container) could craft lines to exploit parser bugs. | `parse_line()` has strict field validation, character-by-character parsing, length limits, and range checks. All fields use bounded types. |
| **PostgreSQL connection** | Network eavesdropping on `tianer-net` bridge. | SCRAM-SHA-256 authentication. tianer-net is host-local bridge, no external access. TLS deferred to post-MVP (D-06). |
| **Environment variables** | `BLESNIFF_DB_PASSWORD` in process memory. | Environment variable injected via `EnvironmentFile`. Process runs as unprivileged user in rootless Podman container. |
| **Signal handlers** | SIGUSR1 triggers metadata dump to stderr. | No privileged operations. Stderr captured by C13 collector. |

### 8.2 Input Validation (INGEST-1)

Every field from the tshark line is validated before being stored:

| Field | Validation |
|-------|-----------|
| `ts` | Must parse as IEEE 754 double. Integral seconds must be in range [0, 2^32-1]. Microseconds in [0, 999999]. |
| `mac_address` | Must match `/^([0-9a-fA-F]{2}:){5}[0-9a-fA-F]{2}$/`. 17 characters exactly. Valid hex in all 6 bytes. |
| `address_type` | Must be `"0"` or `"1"`. |
| `rssi` | Integer. Clamped to `[-128, 20]`. |
| `channel` | Integer in `[0, 255]`. BLE advertising channels [37,39] are expected; others accepted for forward-compatibility. |
| `pdu_type` | Integer in `[0, 255]`. BLE advertising PDUs [0,6] expected; others accepted for forward-compatibility. |
| `advdata` | Even-length hex string, max 62 characters (31 bytes). Each character must be `[0-9a-fA-F]`. |

### 8.3 Field Length Limits

| Field | Max Length (chars) | Rationale |
|-------|-------------------|-----------|
| Full line | 256 | 17 (mac) + 4 (rssi) + 2 (channel) + 3 (pdu_type) + 62 (advdata) + 15 (ts) + 7 delimiters ≈ 110. 256 provides generous headroom. |
| `advdata` hex | 62 | BLE legacy advertising max 31 bytes → 62 hex chars. |
| `mac_address` | 17 | Fixed format `xx:xx:xx:xx:xx:xx`. |

Lines exceeding 256 characters are truncated at 256 and parsed — this handles well-formed lines while bounding memory usage. Truncation at 256 may corrupt the last field (advdata), which will be rejected by `parse_line()` as invalid hex. The line is skipped and `malformed_packets_total` is incremented.

### 8.4 No Raw SQL String Building

The ingest bridge uses `pqxx::stream_to` which constructs parameterized COPY streams internally. At no point does the application code concatenate user input into a SQL string. The `COPY` protocol itself is a binary/structured protocol that separates data from commands — it is not susceptible to SQL injection.

```cpp
// The COPY stream is constructed like this — no string concatenation:
pqxx::stream_to stream{
    tx,
    pqxx::table_path{"bluetooth", "raw_packets"},
    std::vector<std::string_view>{"ts", "sniffer_id", "mac_address",
                                   "address_type", "rssi", "channel",
                                   "pdu_type", "advdata", "backfilled"}
};
```

### 8.5 Least-Privilege Database Role (Q9)

The ingest bridge connects as `tianer_writer`, which has:
- `CONNECT` on `tianer` database
- `USAGE` on `bluetooth` schema
- `INSERT` on all tables under `bluetooth` schema

It does **not** have `SELECT`, `UPDATE`, `DELETE`, `TRUNCATE`, or `DDL` privileges. If the ingest bridge container is compromised:
- The attacker **cannot** read data from `raw_packets` (no SELECT).
- The attacker **cannot** modify or delete existing data (no UPDATE, DELETE, TRUNCATE).
- The attacker **cannot** drop tables or alter schema (no DDL).
- The attacker **can only** insert new rows — which are tagged with the sniffer_id and distinguishable.

---

## 9. Configuration

### 9.1 Environment Variables

All configuration is read from environment variables, per the Tian'er configuration convention (D-02: `BLESNIFF_*` for Bluetooth module components).

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `BLESNIFF_DB_HOST` | No | `tianer-postgres` | PostgreSQL hostname (container name on tianer-net) |
| `BLESNIFF_DB_PORT` | No | `5432` | PostgreSQL port |
| `BLESNIFF_DB_NAME` | No | `tianer` | Database name |
| `BLESNIFF_DB_USER` | No | `tianer_writer` | Database role for COPY operations |
| `BLESNIFF_DB_PASSWORD` | **Yes** | — | SCRAM-SHA-256 password for `tianer_writer` |
| `BLESNIFF_SNIFFER_ID` | **Yes** | — | Sniffer ID matching `sniffers.sniffer_id` in the database |
| `BLESNIFF_SNIFFER_NAME` | No | (derived from sniffer_id) | Human-readable sniffer name for metrics/labels |
| `BLESNIFF_INGEST_FIFO` | No | `/var/run/tianer/${name}-ingest.fifo` | Path to the input FIFO |
| `BLESNIFF_BATCH_MAX_ROWS` | No | `300000` | Max rows per batch (D-09) |
| `BLESNIFF_BATCH_TIMEOUT_MS` | No | `300000` | Max milliseconds before forced flush (D-09) |
| `BLESNIFF_BATCH_OVERFLOW_LIMIT` | No | `500000` | Hard cap on buffered rows before eviction |
| `BLESNIFF_RECONNECT_MAX_INTERVAL_MS` | No | `30000` | Max backoff interval in milliseconds |
| `BLESNIFF_RECONNECT_MAX_ATTEMPTS` | No | `10` | Max reconnection attempts before exit |

### 9.2 Default Values Rationale

| Parameter | Value | Source |
|-----------|-------|--------|
| `BATCH_MAX_ROWS` | 300,000 | D-09 (ADR-0001) |
| `BATCH_TIMEOUT_MS` | 300,000 | D-09 — 5 minutes |
| `BATCH_OVERFLOW_LIMIT` | 500,000 | Derived: ~30 MB at 60 bytes/row. Leaves headroom below 60 MB per-sniffer budget. |
| `RECONNECT_MAX_INTERVAL_MS` | 30,000 | inception.md §8.5: "max 30s" |
| `RECONNECT_MAX_ATTEMPTS` | 10 | 10 × 30s max interval = ~180 seconds of retry. Longer outages are handled by gap detector backfill. |

### 9.3 Quadlet EnvironmentFile

The container receives its configuration via a Quadlet `EnvironmentFile` directive in the `.container` definition (C14):

```ini
# deploy/containers/blesniff-ingest@.container
[Container]
Image=localhost/blesniff-ingest:latest
Exec=/usr/local/bin/blesniff-ingest
EnvironmentFile=/etc/tianer/defaults/blesniff.conf
EnvironmentFile=/etc/tianer/secrets/db_writer_password.env
Volume=/var/run/tianer/%i-ingest.fifo:/var/run/tianer/%i-ingest.fifo:rw
Network=tianer-net
Pod=tianer-platform.pod
```

The `%i` template expands to the sniffer name (e.g. `ubertooth0`), enabling per-sniffer FIFO paths.

---

## 10. Test Plan

### 10.1 Unit Tests (GoogleTest [6])

| Test Suite | Test Case | File | Acceptance Criteria |
|-----------|-----------|------|---------------------|
| `ParserTest` | `HappyPath_CompleteLine` | `tests/parser_test.cpp` | Parses all fields correctly from a complete line |
| `ParserTest` | `HappyPath_MinimalLine` | `tests/parser_test.cpp` | Parses line where all optional fields are empty |
| `ParserTest` | `HappyPath_NoAdvData` | `tests/parser_test.cpp` | Empty advdata produces empty vector |
| `ParserTest` | `Malformed_MissingDelimiter` | `tests/parser_test.cpp` | Fewer than 6 delimiters → `std::nullopt` |
| `ParserTest` | `Malformed_BadTimestamp` | `tests/parser_test.cpp` | Non-float ts → `std::nullopt` |
| `ParserTest` | `Malformed_BadMAC_Short` | `tests/parser_test.cpp` | MAC too short → `std::nullopt` |
| `ParserTest` | `Malformed_BadMAC_NonHex` | `tests/parser_test.cpp` | MAC has non-hex chars → `std::nullopt` |
| `ParserTest` | `Malformed_BadMAC_NoColons` | `tests/parser_test.cpp` | MAC missing colons → `std::nullopt` |
| `ParserTest` | `Malformed_BadAddrType` | `tests/parser_test.cpp` | Address type not 0 or 1 → `address_random` is `std::nullopt` (not parse failure) |
| `ParserTest` | `Malformed_BadRSSI_NonInteger` | `tests/parser_test.cpp` | Non-integer RSSI → `std::nullopt` |
| `ParserTest` | `Malformed_BadChannel_NonInteger` | `tests/parser_test.cpp` | Non-integer channel → `std::nullopt` |
| `ParserTest` | `Malformed_BadPDUType_NonInteger` | `tests/parser_test.cpp` | Non-integer PDU type → `std::nullopt` |
| `ParserTest` | `Malformed_BadAdvData_OddLength` | `tests/parser_test.cpp` | Odd-length hex → `std::nullopt` |
| `ParserTest` | `Malformed_BadAdvData_NonHex` | `tests/parser_test.cpp` | Non-hex char in advdata → `std::nullopt` |
| `ParserTest` | `Boundary_RSSIClamping` | `tests/parser_test.cpp` | RSSI -200 clamped to -128, +100 clamped to +20 |
| `ParserTest` | `Boundary_AdvDataTruncation` | `tests/parser_test.cpp` | advdata > 62 chars truncated to 62, WARNING logged |
| `ParserTest` | `Boundary_EmptyLine` | `tests/parser_test.cpp` | Empty line → `std::nullopt` |
| `ParserTest` | `Boundary_LongLine` | `tests/parser_test.cpp` | Line > 256 chars truncated |
| `ParserTest` | `Timestamp_MicrosecondPrecision` | `tests/parser_test.cpp` | Fractional seconds correctly converted to microseconds |
| `BatcherTest` | `FlushOnRowCount` | `tests/batcher_test.cpp` | `Batcher::add()` returns true when `MAX_ROWS` reached |
| `BatcherTest` | `FlushOnLatency` | `tests/batcher_test.cpp` | `Batcher::add()` returns true when `MAX_LATENCY` elapsed (using mock clock) |
| `BatcherTest` | `NoFlushUntilThreshold` | `tests/batcher_test.cpp` | `Batcher::add()` returns false before threshold |
| `BatcherTest` | `DrainEmptiesBuffer` | `tests/batcher_test.cpp` | After `drain()`, `depth()` is zero |
| `BatcherTest` | `OverflowEviction` | `tests/batcher_test.cpp` | Adding beyond overflow limit evicts oldest rows |
| `BatcherTest` | `EvictionCounter` | `tests/batcher_test.cpp` | `evicted_count()` matches number of evicted rows |

### 10.2 Integration Tests

| Test | Type | File | Acceptance Criteria |
|------|------|------|---------------------|
| `PgWriter_BasicWrite` | Integration | `tests/pg_writer_test.cpp` | Batch of N packets → N rows in `raw_packets` with correct column values |
| `PgWriter_NullableFields` | Integration | `tests/pg_writer_test.cpp` | `std::nullopt` fields appear as SQL NULL |
| `PgWriter_Reconnect` | Integration | `tests/pg_writer_test.cpp` | Kill and restart PG container mid-write; all packets eventually delivered |
| `PgWriter_BackoffExhausted` | Integration | `tests/pg_writer_test.cpp` | After 10 failed attempts, `std::runtime_error` thrown |
| `EndToEnd_FixtureToDB` | Integration | `tests/integration/test_ingest_e2e.sh` | Pipe fixture lines through `blesniff-ingest`; verify DB row count = input line count |
| `EndToEnd_MalformedInput` | Integration | `tests/integration/test_ingest_e2e.sh` | Feed lines with malformed entries; verify correct rows inserted + malformed counter incremented |
| `EndToEnd_Throughput` | Performance | `tests/integration/test_ingest_e2e.sh` | Sustained ≥ 1000 rows/sec on Raspberry Pi CM5 without OOM |
| `EndToEnd_PGOutage` | Integration | `tests/integration/test_pg_reconnect.sh` | `kill -STOP`/`kill -CONT` postgres — no packet loss from 60s stream |

### 10.3 Test Fixtures

```
tests/fixtures/
├── ingest/
│   ├── valid-complete.txt           # 100 lines, all fields present
│   ├── valid-partial.txt            # 100 lines, some optional fields empty
│   ├── malformed-bad-mac.txt        # Lines with various MAC errors
│   ├── malformed-bad-ts.txt         # Lines with bad timestamps
│   ├── malformed-bad-advdata.txt    # Lines with bad advdata hex
│   ├── boundary-clamping.txt        # Lines with out-of-range RSSI/channel
│   ├── empty-lines.txt              # Lines with blank/malformed content
│   └── large-sample.txt             # 10,000 lines for throughput tests
└── expected/
    └── (golden DB state for each fixture)
```

### 10.4 Build Verification (T1.1)

```bash
cd modules/bluetooth/ingest-bridge
cmake -B build -DCMAKE_BUILD_TYPE=Debug -DBUILD_TESTING=ON
cmake --build build
# Must exit 0 with zero warnings (-Werror enabled)
```

### 10.5 Test Execution

```bash
# Unit tests
cd modules/bluetooth/ingest-bridge
ctest --test-dir build --output-on-failure

# Integration tests (requires running PostgreSQL)
./tests/integration/test_ingest_e2e.sh
./tests/integration/test_pg_reconnect.sh
```

---

## 11. Deployment Notes

### 11.1 Build System (CMake)

`modules/bluetooth/ingest-bridge/CMakeLists.txt`:

```cmake
cmake_minimum_required(VERSION 3.25)
project(blesniff-ingest VERSION 1.0.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# Warnings as errors
add_compile_options(-Wall -Wextra -Wpedantic -Werror)

find_package(libpqxx REQUIRED)
find_package(nlohmann_json REQUIRED)   # For structured logging (JSON output)

add_executable(blesniff-ingest
    src/main.cpp
    src/parser.cpp
    src/batcher.cpp
    src/pg_writer.cpp
)

target_include_directories(blesniff-ingest PRIVATE src)
target_link_libraries(blesniff-ingest PRIVATE
    libpqxx::pqxx
    nlohmann_json::nlohmann_json
)

install(TARGETS blesniff-ingest
    RUNTIME DESTINATION /usr/local/bin
)

# Tests
option(BUILD_TESTING "Build tests" ON)
if(BUILD_TESTING)
    enable_testing()
    find_package(GTest REQUIRED)
    add_subdirectory(tests)
endif()
```

### 11.2 Multi-Stage Containerfile

`modules/bluetooth/ingest-bridge/deploy/Containerfile.ingest`:

```dockerfile
# Stage 1: Build
FROM debian:trixie-slim AS builder

RUN apt-get update && apt-get install -y --no-install-recommends \
    g++-14 cmake make libpqxx-dev nlohmann-json3-dev \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /build
COPY modules/bluetooth/ingest-bridge/ .
RUN cmake -B build -DCMAKE_BUILD_TYPE=Release -DBUILD_TESTING=OFF \
    && cmake --build build --parallel $(nproc)

# Stage 2: Runtime
FROM debian:trixie-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
    libpqxx-7.8 libstdc++6 \
    && rm -rf /var/lib/apt/lists/*

COPY --from=builder /build/build/blesniff-ingest /usr/local/bin/blesniff-ingest

USER tianer
ENTRYPOINT ["/usr/local/bin/blesniff-ingest"]
```

**Image properties:**
- Runtime image < 100 MB (slim base + libpqxx + libstdc++ only)
- No shells, no package managers, no compilers, no development headers in runtime stage
- Runs as `tianer` user (UID created in C01)
- Startup time < 1 second (single compiled binary)

### 11.3 Quadlet Container Unit

`deploy/containers/blesniff-ingest@.container` (template, deployed by C14):

```ini
[Unit]
Description=blesniff-ingest — BLE ingest bridge for %i
After=tianer-postgres.service
BindsTo=tianer-platform.pod
Before=tianer-gap-detector.service

[Container]
Image=localhost/blesniff-ingest:latest
ContainerName=blesniff-ingest-%i
Exec=/usr/local/bin/blesniff-ingest

# Configuration
EnvironmentFile=/etc/tianer/defaults/blesniff.conf
EnvironmentFile=/etc/tianer/secrets/db_writer_password.env
Environment=BLESNIFF_SNIFFER_NAME=%i

# Volume mounts
Volume=/var/run/tianer/%i-ingest.fifo:/var/run/tianer/%i-ingest.fifo:rw
Volume=/etc/tianer/:/etc/tianer/:ro
Volume=/var/log/tianer/:/var/log/tianer/:rw

# Networking
Network=tianer-net
Pod=tianer-platform.pod

# Resource limits
MemoryMax=128M
MemoryHigh=96M

# Security
NoNewPrivileges=yes
ReadOnlyPaths=/
ReadWritePaths=/var/log/tianer/
DropCapability=ALL

[Install]
WantedBy=tianer-platform.pod
```

**Key properties:**
- Template unit (`@`) — one instance per sniffer: `blesniff-ingest@ubertooth0`
- V03 mounted `:rw` for FIFO read access
- V01 (`/etc/tianer/`) mounted `:ro` — configuration and secrets
- V04 (`/var/log/tianer/`) mounted `:rw` — structured log output
- Memory cap: 128 MB (D-09 budget of 60 MB per sniffer + runtime overhead)
- `ReadOnlyPaths=/` with selective `ReadWritePaths` — defense in depth
- `DropCapability=ALL` — no elevated privileges
- Runs in `tianer-platform` pod, sharing the pod's network namespace with C06, C07, C08, C09

### 11.4 Install Path

```
/usr/local/bin/blesniff-ingest                        # Compiled binary
/etc/containers/systemd/blesniff-ingest@.container     # Quadlet unit file
/etc/tianer/defaults/blesniff.conf                     # Default env config
/etc/tianer/secrets/db_writer_password.env             # Database credentials (0600)
```

### 11.5 Build & Deploy

```bash
# Build
cd modules/bluetooth/ingest-bridge
cmake -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build

# Build container image
podman build -t localhost/blesniff-ingest:latest \
    -f modules/bluetooth/ingest-bridge/deploy/Containerfile.ingest .

# Deploy Quadlet unit
sudo install -m 0644 deploy/containers/blesniff-ingest@.container \
    /etc/containers/systemd/
systemctl --user daemon-reload

# Start per-sniffer instances
systemctl --user start blesniff-ingest@ubertooth0
systemctl --user start blesniff-ingest@nrf0
```

### 11.6 Deployment Checklist

Before considering C05 deployed, verify:

- [ ] `blesniff-ingest` binary builds with zero warnings (`-Werror`)
- [ ] All unit tests pass: `ctest --test-dir build --output-on-failure` exits 0
- [ ] Integration tests pass with running PostgreSQL
- [ ] Container image builds successfully, < 200 MB
- [ ] Quadlet unit file installed at `/etc/containers/systemd/blesniff-ingest@.container`
- [ ] Per-sniffer instances start and connect to PostgreSQL: `blesniff_connection_state == 1`
- [ ] Packets are ingested: `SELECT COUNT(*) FROM bluetooth.raw_packets WHERE sniffer_id = <id>` returns non-zero after ingestion starts
- [ ] `tianer_writer` role privileges verified: cannot SELECT, UPDATE, DELETE, TRUNCATE
- [ ] SIGUSR1 produces valid Prometheus text format on stderr
- [ ] SIGTERM triggers graceful shutdown with final batch drain
- [ ] Memory usage stays below 128 MB under sustained load

---

*End of C05 Ingest Bridge Design Document.*

---

## References

[1] PostgreSQL Global Development Group. "PostgreSQL 17 Documentation: 32.1. Database Connection Control Functions — libpq." https://www.postgresql.org/docs/17/libpq-connect.html, 2024\. §32.1.2 documents `connect_timeout` and other connection parameter key words used by the ingest bridge's reconnection logic.

[2] Linux man-pages. "pipe(7) — Overview of Pipes and FIFOs." https://man7.org/linux/man-pages/man7/pipe.7.html, 2026. Documents the default pipe capacity (16 pages, i.e., 65,536 bytes on a 4096-byte page system since Linux 2.6.11), `F_SETPIPE_SZ`/`F_GETPIPE_SZ` operations, and `FIONREAD` ioctl — the kernel mechanisms underlying the FIFO backpressure design in §4.3.2.

[3] PostgreSQL Global Development Group. "PostgreSQL 17 Documentation: COPY." https://www.postgresql.org/docs/17/sql-copy.html, 2024\. Documents the `COPY ... FROM STDIN` protocol, the default `\t` (tab) field delimiter and `\N` null string in text format, and the binary vs. text format distinction — the protocol used by `pqxx::stream_to` in §4.3.1 and §5.2.

[4] cppreference.com. "std::optional — C++17." https://en.cppreference.com/w/cpp/utility/optional, 2024. Documents the C++17 `std::optional<T>` class template — a type-safe vocabulary type for "value or absence" that is the primary pattern used in `parse_line()` return values, `Packet` struct nullable fields (§3.1), and column mappings (§3.2).

[5] J. T. Vermeulen. "libpqxx — The Official C++ Client API for PostgreSQL, Release 7.8.1." https://github.com/jtv/libpqxx/releases/tag/7.8.1, 2024. The `pqxx::stream_to` class, used in `PgWriter` (§4.3), provides a stream-based interface for the PostgreSQL COPY protocol, handling escaping internally and eliminating SQL injection surface.

[6] Google. "GoogleTest — Google Testing and Mocking Framework." https://google.github.io/googletest/, v1.14. — Documents GoogleTest C++ testing framework: `TEST()` and `TEST_F()` macros, `EXPECT_*` and `ASSERT_*` assertion families, test fixture setup/teardown, and `ctest` integration via CMake's `enable_testing()` and `add_test()` (§10.1).
