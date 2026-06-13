# C09 — REST API

**Status:** Phase A Design — guides Phase B implementation
**Author:** Code Architect
**Date:** 2026-06-09
**Dependencies:** C02 (Database), C13 (Observability)
**Blocks:** C10 (Frontend)

---

## 1. Overview

### 1.1 Purpose

C09 REST API is the **backend HTTP service** for the Tian'er Signal Intelligence Platform. Built with **Python 3.13 / FastAPI 0.110+** [1], it exposes a set of JSON REST endpoints that serve device observations, enrichment data, alerts, health status, and operational metrics to the Vue 3 frontend (C10) and Prometheus scrapers (C13). It also serves the pre-built frontend static bundle directly from the container image and provides a **Server-Sent Events (SSE)** endpoint for live data streaming to the browser. The endpoint contract is documented as an OpenAPI specification [2] available at `/docs`.

The API runs in the **tianer-platform pod** alongside the ingest bridge (C05), gap detector (C06), deep parser (C07), and ML enrichment (C08) containers. It accesses the PostgreSQL database via two separate connection pools — a read pool using the `tianer_ro` role and a write pool using the `tianer` role. The `tianer_writer` role is **not used by the API** — it is reserved exclusively for the ingest bridge, gap detector, and ML enrichment streams.

### 1.2 Scope

| In Scope | Out of Scope |
|----------|-------------|
| FastAPI application with asyncpg connection pools | Host OS package installation (C01) |
| `/api/*` REST endpoints (JSON) | Frontend component implementation (C10) |
| `/api/events` SSE endpoint for live data | Grafana data source configuration (C11) |
| `/api/metrics` Prometheus metrics endpoint | Pod/Quadlet unit file creation (C12/C14) |
| API key authentication middleware | Database schema definition (C02) |
| Rate limiting middleware | Continuous aggregate refresh execution (C02 owns the cagg; API calls `CALL refresh_continuous_aggregate()`) |
| CORS middleware | PCAP or ingest pipeline logic |
| TLS termination (self-signed cert per D-06) | Secrets generation (C01 `generate-secrets.sh`) |
| Frontend static file serving from container image | |
| Query builder with parameterized queries + pagination | |
| Health check endpoint with DB connectivity probe | |

### 1.3 Boundaries

C09 owns the **HTTP API surface**: the FastAPI app process, the route handlers, the auth/rate-limit/CORS middleware, the PostgreSQL connection pools, and the frontend static file mount. It does **not** own the TLS certificate (C01 manages `/etc/tianer/secrets/`), the Quadlet container definition (C14 defines `tianer-api.container`), the database schema (C02), the frontend source (C10 builds it), or the Prometheus scraping (C13 consumes `/api/metrics`).

```
┌──────────────────────────────────────────────────────────────────┐
│                     tianer-platform pod (bridge)                    │
│                                                                   │
│  ┌────────────────────┐    ┌───────────────────────────────────┐ │
│  │  C09 REST API       │    │  tianer-postgres (standalone)      │ │
│  │  (FastAPI + uvicorn)│    │  PostgreSQL 17 + TimescaleDB 2.23 │ │
│  │                     │    │                                   │ │
│  │  Port 8080 (TLS)    │    │  Role: tianer_ro (read pool)     │ │
│  │                     │◄───│  Role: tianer  (write pool)       │ │
│  │  Auth: X-API-Key    │    │                                   │ │
│  │  Rate limit: 100/s  │    │  NOT: tianer_writer (unused by   │ │
│  │  CORS: frontend only│    │        API — exclusive to C05,    │ │
│  │                     │    │        C06, C08)                  │ │
│  │  Volumes:            │    └───────────────────────────────────┘ │
│  │   V01 (:ro)          │                                         │
│  │   /etc/tianer/       │    ┌───────────────────────────────────┐ │
│  │   ├── secrets/       │    │  C10 Frontend (served from image) │ │
│  │   │   ├── api_key    │    │  /usr/share/tianer/frontend/      │ │
│  │   │   ├── tls.crt    │    │  Served by FastAPI at /           │ │
│  │   │   └── tls.key    │    └───────────────────────────────────┘ │
│  │   └── tianer.env     │                                         │
│  └────────────────────┘                                          │
└──────────────────────────────────────────────────────────────────┘
                                 │
                                 │ Host port forward
                                 │ 127.0.0.1:8080 → container:8080
                                 ▼
                        ┌────────────────┐
                        │  C13 Prometheus │
                        │  Scraper       │
                        │  /api/metrics   │
                        └────────────────┘
```

### 1.4 Position in the System

C09 is a **Layer 2 shared platform component** in the build sequence (component-breakdown.md §4.1). It depends on C02 (Database must be available with the `tianer_ro` and `tianer` roles provisioned and the `bluetooth` schema migrated). It blocks C10 (Frontend), which consumes the API to render device data, alerts, and health status.

---

## 2. High-Level Architecture (HLA)

### 2.1 Application Structure

```
platform/api/
├── pyproject.toml
├── src/tianer_api/
│   ├── __init__.py
│   ├── main.py              # FastAPI app creation, middleware stack, static mount, lifespan
│   ├── config.py            # Environment variable loading, settings dataclass
│   ├── db.py                # asyncpg connection pools (read + write)
│   ├── auth.py              # API key verification dependency
│   ├── rate_limit.py        # Token-bucket rate limiter
│   ├── models.py            # All Pydantic request/response models
│   ├── query_builder.py     # Typed query construction with parameterized SQL
│   ├── pagination.py        # Cursor-based and offset pagination helpers
│   ├── metrics.py           # Prometheus metrics registry + /api/metrics endpoint
│   ├── sse.py               # SSE stream manager
│   └── routers/
│       ├── __init__.py
│       ├── devices.py       # /api/devices, /api/devices/{mac}
│       ├── device_detail.py # /api/devices/{mac}/timeline, /api/devices/{mac}/enrichment
│       ├── alerts.py        # /api/alerts/new-devices
│       ├── events.py        # /api/events (SSE)
│       ├── health.py        # /api/health
│       └── metrics.py       # /api/metrics
└── tests/
    ├── conftest.py           # pytest fixtures: test DB, async client
    ├── test_health.py
    ├── test_devices.py
    ├── test_device_detail.py
    ├── test_alerts.py
    ├── test_auth.py
    ├── test_rate_limit.py
    ├── test_sse.py
    └── test_metrics.py
```

### 2.2 Request Flow

```
┌──────────┐    HTTPS (TLS)     ┌───────────────┐    scram-sha-256    ┌──────────────┐
│  C10     │───────────────────▶│  C09 FastAPI   │───────────────────▶│  C02 PG 17   │
│ Frontend │  X-API-Key header  │  uvicorn       │  asyncpg pool      │  +TimescaleDB│
│ (browser)│◀──────────────────│  :8080          │◀──────────────────│              │
└──────────┘    JSON response   └───────┬───────┘   query results     └──────────────┘
                                        │
                                        │ SSE (text/event-stream)
                                        │
                                        ▼
                                 ┌──────────────┐
                                 │  Browser      │
                                 │  EventSource  │
                                 └──────────────┘
```

**Per-request middleware pipeline:**

1. **TLS termination:** uvicorn handles at the transport layer using the self-signed cert from V01 [6].
2. **CORS check:** Validates `Origin` header against allowed origins (configured per §8.4).
3. **Rate limit check:** Consumes a token from the per-IP token bucket. Returns 429 if exhausted [3].
4. **Auth check:** All `/api/*` endpoints require `X-API-Key` header matching the value in `/etc/tianer/secrets/api_key`. Returns 401 on mismatch or absence (RFC 9110 Section 15.5.2) [3]. `/api/health` and `/api/metrics` are exempt (but `/api/metrics` is loopback-only).
5. **Route handler:** Validates query parameters via Pydantic models. Builds a parameterized SQL query via `query_builder`. Acquires a connection from the appropriate pool (read or write). Executes the query. Returns the JSON response.

### 2.3 Connection Pool Strategy

The API maintains **two separate asyncpg connection pools**, one per database role (psycopg 3 `AsyncConnectionPool`) [4]:

| Pool | Role | Max Connections | Purpose |
|------|------|----------------|---------|
| `read_pool` | `tianer_ro` | 10 | All read-only endpoints: `/api/devices`, `/api/devices/{mac}`, `/api/devices/{mac}/timeline`, `/api/devices/{mac}/enrichment`, `/api/alerts/new-devices`, `/api/packets`, `/api/sniffers`, `/api/stats` |
| `write_pool` | `tianer` | 5 | Write endpoints: continuous aggregate refresh, residency classification refresh. Also used by the SSE manager to poll for new devices. |

The `tianer_writer` role is **never used by the API**. It is exclusive to:

- C05 Ingest Bridge (INSERT via COPY protocol)
- C06 Gap Detector (INSERT via parameterized queries)
- C08 ML Enrichment (INSERT/UPDATE via parameterized queries)

This separation ensures that a compromise of the API (which has the widest attack surface) cannot directly write to tables (the `tianer_ro` role is SELECT-only) and cannot drop or alter tables unless the write pool is used — and the write pool has DDL privileges restricted to specific known operations.

**Pool lifecycle:** Pools are created in the FastAPI `lifespan` context manager [1]. On startup, connections are pre-warmed. On shutdown, all connections are gracefully closed.

### 2.4 Frontend Static Serving

The pre-built frontend bundle (produced by C10's `npm run build`) is embedded in the container image at `/usr/share/tianer/frontend/`. FastAPI mounts two static routes:

| Path | Behaviour |
|------|-----------|
| `/*` (catch-all) | Serves files from `/usr/share/tianer/frontend/`. Falls back to `index.html` for SPA client-side routing. |
| `/assets/*` | Serves static assets with immutable `Cache-Control: public, max-age=31536000` (1 year). |

This is a standard single-page application deployment pattern. The Vue Router handles client-side routing; any path not matching an `/api/*` route is served `index.html`.

---

## 3. Data Model (Pydantic Models)

All response models use **Pydantic v2** with `model_config = {"from_attributes": False}` (manual construction from DB rows). Field names use camelCase in JSON (via `alias_generator` or explicit `alias`).

### 3.1 DeviceSummary

Represents a device in the device list view.

```python
from pydantic import BaseModel, Field
from datetime import datetime
from typing import Literal, Optional

class DeviceSummary(BaseModel):
    """A device in the aggregated device list view.

    Maps to bluetooth.device_summary with computed fields from the
    device_enrichment table where available.
    """
    mac: str = Field(
        ...,
        description="Bluetooth device address encoded as hex colon-separated (e.g. 'aa:bb:cc:dd:ee:ff')",
        examples=["aa:bb:cc:dd:ee:ff"],
        pattern=r"^([0-9a-fA-F]{2}:){5}[0-9a-fA-F]{2}$",
    )
    address_type: Literal["public", "random", "unknown"] = Field(
        ...,
        description="Bluetooth address type. 'unknown' when not determinable.",
    )
    first_seen: datetime = Field(
        ...,
        description="Earliest observation timestamp for this device (ISO 8601).",
    )
    last_seen: datetime = Field(
        ...,
        description="Most recent observation timestamp for this device (ISO 8601).",
    )
    total_count: int = Field(
        ...,
        ge=0,
        description="Cumulative observation count across all sniffers.",
    )
    distinct_days: int = Field(
        ...,
        ge=0,
        description="Number of distinct calendar days this device has been observed.",
    )
    residency_class: Optional[str] = Field(
        None,
        description="Residency classification: 'unknown', 'new', 'lost', 'resident', 'frequent', 'transient'. See C02 §3.12.",
    )
    classes: list[str] = Field(
        default_factory=list,
        description="Device classes from ML enrichment (e.g. ['apple_continuity', 'battery_service_device']).",
    )
```

**DB mapping:** `mac` is decoded from `BYTEA` using `encode(mac_address, 'hex')` in SQL. `address_type` is decoded from `SMALLINT` (0→public, 1→random, NULL→unknown). `classes` comes from `device_summary.enrichment_data->'classes'`.

### 3.2 DeviceDetail

Full detail view for a single device.

```python
class DeviceDetail(DeviceSummary):
    """Full device detail including all enrichment metadata."""
    enrichment_records: int = Field(
        ...,
        ge=0,
        description="Total enrichment records available for this device.",
    )
    sniffers: list[int] = Field(
        default_factory=list,
        description="List of sniffer IDs that have observed this device.",
    )
    last_classified: Optional[datetime] = Field(
        None,
        description="When the residency classifier was last run for this device.",
    )
```

### 3.3 TimelineBucket

A single time bucket in the device timeline.

```python
class TimelineBucket(BaseModel):
    """A 5-minute aggregate bucket from device_5min_buckets continuous aggregate."""
    ts: datetime = Field(..., description="Bucket start timestamp.")
    count: int = Field(..., ge=0, description="Packet count in this bucket.")
    avg_rssi: Optional[int] = Field(None, description="Average RSSI in dBm for this bucket.")
    min_rssi: Optional[int] = Field(None)
    max_rssi: Optional[int] = Field(None)
    sniffers: list[int] = Field(default_factory=list, description="Sniffer IDs that contributed to this bucket.")
```

### 3.4 EnrichmentRecord

A single enrichment row from `device_enrichment`.

```python
class EnrichmentRecord(BaseModel):
    """A single deep packet enrichment record from device_enrichment table."""
    observed_ts: datetime = Field(..., description="Timestamp from the original packet.")
    sniffer_id: int = Field(..., ge=1, le=4)
    local_name: Optional[str] = Field(None, description="BLE device name from AD type 0x08/0x09.")
    service_uuids_16: list[str] = Field(default_factory=list)
    service_uuids_128: list[str] = Field(default_factory=list)
    manufacturer_id: Optional[int] = Field(None, description="Bluetooth SIG Company Identifier.")
    tx_power: Optional[int] = Field(None, description="TX Power Level in dBm.")
    flags: Optional[int] = Field(None, description="BLE Flags from AD type 0x01.")
```

### 3.5 HealthStatus

Response for `/api/health`.

```python
class SnifferHealth(BaseModel):
    """Health status of a single sniffer."""
    id: int = Field(..., ge=1, le=4)
    name: str = Field(..., examples=["ut1"])
    type: str = Field(..., examples=["ubertooth"])
    enabled: bool
    last_heartbeat: Optional[datetime] = Field(None)
    running: bool = Field(..., description="Whether the sniffer was alive within the last 60 seconds.")

class HealthStatus(BaseModel):
    """Overall system health response."""
    status: Literal["ok", "degraded", "error"] = Field(
        ...,
        description="Aggregate health: 'ok' (all healthy), 'degraded' (some components unhealthy), 'error' (critical failure).",
    )
    db: Literal["ok", "error"] = Field(..., description="Database connectivity status.")
    db_migration: Optional[str] = Field(None, description="Latest applied migration name.")
    db_connections: int = Field(..., ge=0, description="Current active database connections.")
    db_size_mb: float = Field(..., ge=0, description="Approximate database size in megabytes.")
    sniffers: list[SnifferHealth] = Field(default_factory=list)
    ingest_gaps_open: int = Field(..., ge=0, description="Number of currently open ingest gaps.")
```

**Health logic:**
- `status = "ok"` when `db == "ok"` AND all enabled sniffers have `running == True` AND `ingest_gaps_open == 0`.
- `status = "degraded"` when `db == "ok"` but some sniffers are not running or gaps exist.
- `status = "error"` when `db == "error"`.

### 3.6 AlertInfo

An alert notification.

```python
class AlertInfo(BaseModel):
    """A single alert notification."""
    alert_type: Literal["new_device", "device_lost", "gap_detected", "gap_failed"] = Field(...)
    severity: Literal["info", "warning", "critical"] = Field(...)
    message: str = Field(...)
    ts: datetime = Field(...)
    data: dict = Field(default_factory=dict, description="Alert-specific payload.")
```

### 3.7 MetricsSnapshot

Prometheus metric family wrapper.

```python
class MetricsSnapshot(BaseModel):
    """Collection of Prometheus metric families in OpenMetrics text format."""
    # Not a traditional JSON model — the /api/metrics endpoint returns
    # Prometheus text format (Content-Type: text/plain; version=0.0.4).
    # This model exists only for documentation and test validation.
    pass
```

### 3.8 Pagination Models

```python
class PaginatedResponse(BaseModel):
    """Wrapper for paginated list responses."""
    total: int = Field(..., ge=0, description="Total number of items matching the query.")
    limit: int = Field(..., gt=0, le=500, description="Requested page size (max 500).")
    offset: int = Field(..., ge=0, description="Requested page offset.")
    items: list = Field(default_factory=list)

class TimelineResponse(BaseModel):
    """Response for the device timeline endpoint."""
    mac: str
    bucket: Literal["5m", "1h", "1d"] = Field("5m")
    from_ts: Optional[datetime] = Field(None, alias="from")
    to_ts: Optional[datetime] = Field(None, alias="to")
    buckets: list[TimelineBucket] = Field(default_factory=list)

class ErrorResponse(BaseModel):
    """Standardized error response for non-2xx status codes."""
    error: str = Field(..., description="Machine-readable error code.")
    message: str = Field(..., description="Human-readable error description.")
    detail: Optional[str] = Field(None, description="Optional detailed information.")
```

### 3.9 PacketItem

Represents a single raw packet in the Packet Explorer view.

```python
class PacketItem(BaseModel):
    """A single raw BLE packet from the capture pipeline.

    Maps to bluetooth.raw_packets with decoded fields.
    """
    ts: datetime = Field(
        ...,
        description="Packet capture timestamp (ISO 8601).",
    )
    sniffer_id: int = Field(
        ...,
        ge=1,
        le=4,
        description="Sniffer that captured this packet.",
    )
    mac: str = Field(
        ...,
        description="Bluetooth device address encoded as hex colon-separated.",
        pattern=r"^([0-9a-fA-F]{2}:){5}[0-9a-fA-F]{2}$",
    )
    rssi: Optional[int] = Field(
        None,
        description="Received Signal Strength Indicator in dBm.",
    )
    pdu_type: str = Field(
        ...,
        description="BLE PDU type (e.g. 'ADV_IND', 'SCAN_RSP', 'ADV_NONCONN_IND').",
    )
    channel: int = Field(
        ...,
        ge=37,
        le=39,
        description="BLE advertising channel (37, 38, or 39).",
    )
    raw_data_hex: Optional[str] = Field(
        None,
        description="Hex-encoded raw PDU payload bytes.",
    )
```

**DB mapping:** `ts` maps to the `ts` column in the `raw_packets` hypertable. `mac` is decoded from `BYTEA mac_address` using `encode(mac_address, 'hex')`. `pdu_type` is decoded from `SMALLINT pdu_type` per the BLE Core Specification PDU type encoding. There is no synthetic `id` column on `raw_packets` — packets are identified by the composite key `(sniffer_id, ts, mac_address, pdu_type)`.

### 3.10 PacketListResponse

```python
class PacketListResponse(BaseModel):
    """Paginated raw packet list response."""
    packets: list[PacketItem] = Field(
        default_factory=list,
        description="List of packets for the current page.",
    )
    total: int = Field(
        ...,
        ge=0,
        description="Total number of packets matching the query.",
    )
    limit: int = Field(
        ...,
        gt=0,
        le=500,
        description="Requested page size.",
    )
    offset: int = Field(
        ...,
        ge=0,
        description="Requested page offset.",
    )
```

### 3.11 SnifferStatus

Represents a single sniffer's operational status for the SideStatusPanel.

```python
class SnifferStatus(BaseModel):
    """Operational status of a single sniffer."""
    sniffer_id: int = Field(
        ...,
        ge=1,
        le=4,
        description="Sniffer identifier.",
    )
    name: str = Field(
        ...,
        description="Human-readable sniffer name (e.g. 'ut1').",
    )
    type: str = Field(
        ...,
        description="Sniffer hardware type: 'ubertooth', 'nrf52840', or 'hci'.",
    )
    mode: str = Field(
        ...,
        description="Current operating mode (e.g. 'btle').",
    )
    channel: int = Field(
        ...,
        ge=37,
        le=39,
        description="Currently tuned BLE advertising channel.",
    )
    enabled: bool = Field(
        ...,
        description="Whether this sniffer is enabled in the pipeline configuration.",
    )
    last_heartbeat: Optional[datetime] = Field(
        None,
        description="Timestamp of the most recent heartbeat from this sniffer (ISO 8601).",
    )
    status: Literal["running", "stopped", "error", "unknown"] = Field(
        ...,
        description="Current operational status. 'running': heartbeat within 60s. 'stopped': enabled but no recent heartbeat. 'error': sniffer process exited abnormally. 'unknown': sniffer has not reported status.",
    )
```

**DB mapping:** Queries the `bluetooth.sniffers` table joined with `bluetooth.sniffer_heartbeat` for the most recent heartbeat timestamp. `status` is derived: `running` when `enabled = TRUE` AND `last_heartbeat` is within 60 seconds; `stopped` when `enabled = TRUE` but heartbeat is stale; `error` when the process is not running; `unknown` when no heartbeat has ever been recorded.

### 3.12 SnifferListResponse

```python
class SnifferListResponse(BaseModel):
    """Sniffer status list response for the SideStatusPanel."""
    sniffers: list[SnifferStatus] = Field(
        default_factory=list,
        description="List of all configured sniffers with their current status.",
    )
```

### 3.13 DashboardStats

Aggregate overview statistics for the Dashboard StatCards.

```python
class DashboardStats(BaseModel):
    """Aggregate dashboard overview statistics.

    All counts are computed from live database queries. This model
    combines data from multiple tables (device_summary, raw_packets,
    sniffers, ingest_gaps, and system metrics) into a single response.
    """
    total_devices: int = Field(
        ...,
        ge=0,
        description="Total distinct devices ever observed.",
    )
    active_devices_1h: int = Field(
        ...,
        ge=0,
        description="Devices with last_seen within the past hour.",
    )
    new_devices_24h: int = Field(
        ...,
        ge=0,
        description="Devices with first_seen within the past 24 hours.",
    )
    packets_today: int = Field(
        ...,
        ge=0,
        description="Total packets captured since midnight UTC.",
    )
    packet_rate_current: int = Field(
        ...,
        ge=0,
        description="Estimated current packet rate (packets per second, averaged over last 60 seconds).",
    )
    sniffers_online: int = Field(
        ...,
        ge=0,
        description="Number of sniffers with a heartbeat within the last 60 seconds.",
    )
    sniffers_total: int = Field(
        ...,
        ge=0,
        description="Total number of configured sniffers.",
    )
    gaps_open: int = Field(
        ...,
        ge=0,
        description="Number of currently open ingest gaps.",
    )
    disk_usage_pct: float = Field(
        ...,
        ge=0,
        le=100,
        description="PCAP storage disk usage percentage.",
    )
    uptime_seconds: int = Field(
        ...,
        ge=0,
        description="System uptime in seconds (from /proc/uptime on the host or container start time).",
    )
```

**Caching:** This endpoint response is cached for 30 seconds. Stats do not require per-request freshness — the DashboardView polls at 60-second intervals, so a 30-second cache TTL prevents redundant database queries during rapid page loads or SSE-triggered re-fetches without meaningfully delaying data freshness.

---

## 4. Low-Level Architecture (LLA)

### 4.1 Auth Middleware

The API key authentication is implemented as a FastAPI dependency [1], not as Starlette middleware. This allows selective application — health and metrics endpoints are exempt.

```python
# auth.py

from fastapi import Depends, HTTPException, Security
from fastapi.security import APIKeyHeader
import secrets

_api_key_header = APIKeyHeader(name="X-API-Key", auto_error=False)

class ApiKeyAuth:
    """API key authentication dependency.

    Reads the expected key from the filesystem once at startup.
    Uses constant-time comparison to prevent timing attacks.
    """

    def __init__(self, key_path: str):
        with open(key_path, "r") as f:
            self._expected = f.read().strip()

    async def __call__(self, api_key: str | None = Security(_api_key_header)) -> str:
        if api_key is None:
            raise HTTPException(status_code=401, detail="Missing X-API-Key header")
        if not secrets.compare_digest(api_key.encode(), self._expected.encode()):
            raise HTTPException(status_code=401, detail="Invalid API key")
        return api_key

# Used as a dependency in router files:
# router = APIRouter(dependencies=[Depends(auth)])
```

**Key properties:**
- The API key is a 32-byte random string generated by `openssl rand -base64 32` (C01 `generate-secrets.sh`).
- The key is stored at `/etc/tianer/secrets/api_key` (mode 0600, owner `tianer:tianer`), mounted into the container via V01.
- The key is read once at startup. A filesystem watcher could be added post-MVP for hot-reload but is not required for v1 (restart the container to rotate the key).
- `secrets.compare_digest` is used to prevent timing side-channel attacks on the key comparison [5].
- The `/api/health` and `/api/metrics` endpoints are explicitly excluded from the dependency. `/api/events` (SSE) requires authentication.

**Rationale for dependency over middleware:** Middleware runs before routing and would require a path exclusion list. FastAPI dependencies [1] are applied at the router level, giving precise control over which endpoints require authentication.

### 4.2 Rate Limiting Middleware

Rate limiting uses a **token bucket algorithm** implemented as Starlette middleware. It applies to `/api/*` endpoints only (excluding `/api/health` and `/api/metrics`).

```python
# rate_limit.py

import time
import asyncio
from collections import defaultdict
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.responses import JSONResponse

class TokenBucket:
    """Thread-unsafe token bucket for single-consumer use (backed by asyncio.Lock)."""

    def __init__(self, rate: float, burst: int):
        self._rate = rate          # tokens per second
        self._burst = burst        # max tokens
        self._tokens = float(burst)
        self._last = time.monotonic()
        self._lock = asyncio.Lock()

    async def consume(self, tokens: int = 1) -> bool:
        async with self._lock:
            now = time.monotonic()
            elapsed = now - self._last
            self._tokens = min(self._burst, self._tokens + elapsed * self._rate)
            self._last = now
            if self._tokens >= tokens:
                self._tokens -= tokens
                return True
            return False

class RateLimitMiddleware(BaseHTTPMiddleware):
    """Per-IP rate limiting middleware.

    Limits each client IP to 100 requests per second with a burst of 150.
    Only applies to paths starting with /api/ (excluding /api/health and /api/metrics).
    """

    def __init__(self, app, rate: float = 100.0, burst: int = 150):
        super().__init__(app)
        self._rate = rate
        self._burst = burst
        self._buckets: dict[str, TokenBucket] = defaultdict(
            lambda: TokenBucket(rate, burst)
        )

    async def dispatch(self, request, call_next):
        path = request.url.path
        if path.startswith("/api/") and path not in ("/api/health", "/api/metrics"):
            client_ip = request.client.host if request.client else "unknown"
            bucket = self._buckets[client_ip]
            if not await bucket.consume():
                return JSONResponse(
                    status_code=429,
                    content={"error": "rate_limit_exceeded",
                             "message": "Too many requests. Retry after 1 second.",
                             "detail": f"Rate limit: {self._rate:.0f} req/s"},
                    headers={"Retry-After": "1"},
                )
        return await call_next(request)
```

**Design decisions:**
- **In-memory buckets:** Tokens are stored in a `defaultdict` in the API process. They are lost on restart — acceptable for v1 (burst of requests after restart is bounded by the client's retry logic).
- **Per-IP, not per-key:** Rate limiting by client IP prevents a single misbehaving client from overwhelming the API. If multiple users share an API key behind NAT, they share the rate limit — acceptable for a single-user LAN deployment.
- **No external Redis dependency:** Keeping rate limits in-memory avoids introducing a Redis dependency for v1 MVP. Post-MVP can add Redis-backed rate limiting for persistence and consistency.
- **Burst of 150:** Allows the frontend to make a burst of requests on page load (device list + health + alerts) without being throttled.
- **Cleanup:** A background task periodically evicts buckets older than 5 minutes to prevent unbounded memory growth (not shown; lightweight for v1 since the number of unique client IPs is small).

### 4.3 Query Builder

All SQL is constructed using a **typed query builder** that prevents string concatenation and enforces parameterized queries. This is a defense-in-depth measure: even if `asyncpg` already parameterizes `$1, $2, ...` placeholders, the query builder prevents developers from accidentally introducing SQL injection through string interpolation.

```python
# query_builder.py

from typing import Any
from dataclasses import dataclass, field

@dataclass
class Query:
    """A parameterized SQL query ready for execution."""
    sql: str
    params: list[Any] = field(default_factory=list)

class DeviceQueryBuilder:
    """Builds parameterized queries for the /api/devices endpoint."""

    SORT_MAP = {
        "last_seen": "ds.last_seen",
        "first_seen": "ds.first_seen",
        "total_count": "ds.total_count",
        "-last_seen": "ds.last_seen",
        "-first_seen": "ds.first_seen",
        "-total_count": "ds.total_count",
    }

    @staticmethod
    def build_list(
        query: str | None = None,
        residency_class: str | None = None,
        seen_after: str | None = None,
        sort: str = "-last_seen",
        limit: int = 50,
        offset: int = 0,
    ) -> Query:
        """Build a parameterized query for the device list.

        All user-supplied values become $N parameters. Column names and
        sort directions are validated against a whitelist (SORT_MAP).
        """
        clauses = ["WHERE 1=1"]
        params: list[Any] = []

        if query:
            # Search by hex MAC prefix (e.g. 'aa:bb' matches 'aa:bb:*')
            mac_prefix = query.lower().replace(":", "")
            if len(mac_prefix) <= 12:
                clauses.append(
                    "AND encode(ds.mac_address, 'hex') LIKE $1"
                )
                params.append(f"{mac_prefix}%")
                param_idx = 2
            else:
                param_idx = 1
        else:
            param_idx = 1

        if residency_class:
            clauses.append(f"AND ds.residency_class = ${param_idx}")
            params.append(residency_class)
            param_idx += 1

        if seen_after:
            clauses.append(f"AND ds.last_seen > ${param_idx}")
            params.append(seen_after)
            param_idx += 1

        sort_col = DeviceQueryBuilder.SORT_MAP.get(sort, "ds.last_seen DESC")
        if sort_col.startswith("-"):
            sort_col = sort_col[1:] + " DESC"
        else:
            sort_col = sort_col + " ASC"

        clauses.append(f"ORDER BY {sort_col}")
        clauses.append(f"LIMIT ${param_idx}")
        params.append(min(limit, 500))  # Enforce max page size
        param_idx += 1
        clauses.append(f"OFFSET ${param_idx}")
        params.append(max(offset, 0))

        sql = f"""
        SELECT
            encode(ds.mac_address, 'hex') AS mac,
            CASE ds.address_type
                WHEN 0 THEN 'public'
                WHEN 1 THEN 'random'
                ELSE 'unknown'
            END AS address_type,
            ds.first_seen,
            ds.last_seen,
            ds.total_count,
            ds.distinct_days,
            ds.residency_class,
            COALESCE(ds.enrichment_data->'classes', '[]'::jsonb) AS classes
        FROM bluetooth.device_summary ds
        {' '.join(clauses)}
        """
        return Query(sql=sql, params=params)

    @staticmethod
    def build_count(
        query: str | None = None,
        residency_class: str | None = None,
        seen_after: str | None = None,
    ) -> Query:
        """Build the count query for pagination total."""
        # Same WHERE clauses as build_list, without ORDER/LIMIT/OFFSET
        clauses = ["WHERE 1=1"]
        params: list[Any] = []
        param_idx = 1

        if query:
            mac_prefix = query.lower().replace(":", "")
            if len(mac_prefix) <= 12:
                clauses.append(
                    "AND encode(ds.mac_address, 'hex') LIKE $1"
                )
                params.append(f"{mac_prefix}%")
                param_idx = 2

        if residency_class:
            clauses.append(f"AND ds.residency_class = ${param_idx}")
            params.append(residency_class)
            param_idx += 1

        if seen_after:
            clauses.append(f"AND ds.last_seen > ${param_idx}")
            params.append(seen_after)
            param_idx += 1

        sql = f"""
        SELECT COUNT(*) AS total
        FROM bluetooth.device_summary ds
        {' '.join(clauses)}
        """
        return Query(sql=sql, params=params)
```

**Key safety properties:**
1. **Column name whitelist:** The `SORT_MAP` dictionary maps user-facing sort keys to exact column references. User input never directly becomes a column name.
2. **All values parameterized:** Every user-supplied value becomes a `$N` parameter passed to `asyncpg.execute()`. Never string-interpolated.
3. **Limit clamping:** The `limit` parameter is clamped to a maximum of 500. A `LIMIT 0` query returns an empty list but does not error.
4. **Offset clamping:** Negative offsets are clamped to 0.
5. **No dynamic table names:** All table references are hard-coded. The user cannot influence which table is queried.

### 4.4 Pagination

The API uses **offset-based pagination** for all list endpoints. Cursor-based pagination is deferred to post-MVP.

```
GET /api/devices?limit=50&offset=0    → items 0-49,  total=N
GET /api/devices?limit=50&offset=50   → items 50-99, total=N
```

**Implementation:** The `DeviceQueryBuilder` constructs both the main query (with `LIMIT`/`OFFSET`) and a count query (with only `WHERE` clauses). Both queries run inside a single database transaction with `repeatable_read` isolation level to ensure consistent `total` + page. If the transaction isolation overhead is too high, the count and data queries can run in separate short-lived transactions — a small inconsistency in `total` during concurrent modifications is acceptable for a read-heavy dashboard.

**Edge cases:**
- `offset` ≥ `total` → returns empty `items` list with accurate `total`.
- `limit = 0` → returns empty `items` list. Useful for checking `total` without data.
- `limit > 500` → clamped to 500. The frontend should request additional pages.
- `offset` and `limit` both 0 → empty list (validated by Pydantic).

### 4.5 SSE Implementation

The Server-Sent Events endpoint (`GET /api/events`) provides a unidirectional stream of live events from the server to the browser. It uses an in-memory publish-subscribe pattern backed by `asyncio.Queue` [7]. The endpoint returns a FastAPI `StreamingResponse` [1] with `media_type="text/event-stream"`.

```python
# sse.py

import asyncio
import json
from collections.abc import AsyncGenerator

class SSEManager:
    """Manages SSE client subscriptions and event broadcasting.

    Each connected client gets an asyncio.Queue. Events are broadcast
    to all connected clients by pushing to each queue.
    """

    def __init__(self):
        self._queues: list[asyncio.Queue[str]] = []
        self._broadcast_task: asyncio.Task | None = None
        self._counter = 0

    async def subscribe(self) -> AsyncGenerator[str, None]:
        """Register a new SSE client. Yields keep-alive comments and events."""
        queue: asyncio.Queue[str] = asyncio.Queue(maxsize=100)
        self._queues.append(queue)
        self._counter += 1
        print(f"TIANER | {{\"ts\":\"...\",\"level\":\"INFO\",\"component\":\"api\",\"msg\":\"SSE client connected\",\"total_clients\":{self._counter}}}")
        try:
            while True:
                try:
                    # Wait up to 15 seconds for an event; send keep-alive comment on timeout
                    event = await asyncio.wait_for(queue.get(), timeout=15.0)
                    yield event
                except asyncio.TimeoutError:
                    yield ": keepalive\n\n"
        except asyncio.CancelledError:
            pass
        finally:
            self._queues.remove(queue)
            self._counter -= 1

    async def broadcast(self, event_type: str, data: dict) -> None:
        """Send an event to all connected clients."""
        payload = json.dumps(data)
        message = f"event: {event_type}\ndata: {payload}\n\n"
        stale: list[asyncio.Queue] = []
        for q in self._queues:
            try:
                q.put_nowait(message)
            except asyncio.QueueFull:
                # Client is too slow — drop the queue
                stale.append(q)
        for q in stale:
            self._queues.remove(q)
            self._counter -= 1

    async def start_polling(self, db_pool, interval: float = 2.0) -> None:
        """Background task: poll DB for new devices and broadcast events."""
        last_seen_max = None
        while True:
            try:
                async with db_pool.acquire() as conn:
                    # Query for devices seen since the last poll
                    if last_seen_max is None:
                        # First run: get the latest seen timestamp
                        row = await conn.fetchrow(
                            "SELECT MAX(first_seen) AS ts FROM bluetooth.device_summary"
                        )
                        last_seen_max = row["ts"]
                    else:
                        rows = await conn.fetch(
                            """SELECT encode(mac_address, 'hex') AS mac,
                                      first_seen, address_type
                               FROM bluetooth.device_summary
                               WHERE first_seen > $1
                               ORDER BY first_seen DESC
                               LIMIT 50""",
                            last_seen_max,
                        )
                        for row in rows:
                            await self.broadcast("device:new", {
                                "mac": row["mac"],
                                "first_seen": row["first_seen"].isoformat(),
                                "rssi": row.get("rssi", -100),
                            })
                        if rows:
                            last_seen_max = rows[0]["first_seen"]
            except Exception as e:
                print(f"TIANER | {{\"ts\":\"...\",\"level\":\"ERROR\",\"component\":\"api\",\"msg\":\"SSE poll error\",\"error\":\"{e}\"}}")
            await asyncio.sleep(interval)

# Singleton instance
sse_manager = SSEManager()
```

**SSE endpoint (`routers/events.py`):**

```python
from fastapi import APIRouter, Request, Depends
from sse_stream import sse_manager
from auth import auth

router = APIRouter()

@router.get("/api/events")
async def event_stream(request: Request, _=Depends(auth)):
    """Stream live events via Server-Sent Events.

    Requires authentication (cookie-based for browser EventSource, or
    X-API-Key header for programmatic clients). Returns text/event-stream.
    Events:
        - device:new: A device was seen for the first time in the current session.
        - packet:batch: Batched packet count per sniffer, emitted every ~2 seconds.
        - health:change: A sniffer's status changed (online/offline).
        - gap:detected: The gap detector found a missing time window.
    """
    return StreamingResponse(
        sse_manager.subscribe(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
            "X-Accel-Buffering": "no",  # Disable nginx buffering if behind proxy
        },
    )
```

**Event types (aligned with C10 §3.3):**

| Event Type | Trigger | Data Fields |
|------------|---------|-------------|
| `device:new` | A device with `first_seen` newer than the last poll | `mac` (hex colon-separated string), `first_seen` (ISO 8601), `rssi` (int, dBm) |
| `packet:batch` | Every ~2 seconds for each active sniffer | `sniffer_id` (int), `count` (int, packets in this window), `window_start` (ISO 8601), `window_end` (ISO 8601) |
| `health:change` | Sniffer heartbeat transitions between online/offline state | `component` (string, always `"sniffer"`), `sniffer_id` (int), `status` (`"online"` or `"offline"`), `ts` (ISO 8601) |
| `gap:detected` | A new row appears in `ingest_gaps` with status `open` | `sniffer_id` (int), `gap_start` (ISO 8601), `gap_end` (ISO 8601), `bucket_count` (int) |

**Design notes:**
- SSE is unidirectional (server → client). The browser can close the connection and the server detects it via `CancelledError`.
- Each client has its own `asyncio.Queue` with a max size of 100. Slow clients that don't read events fast enough are silently disconnected.
- **Event source expansion (post-MVP):** The MVP SSE polling task only broadcasts `device:new` events. The `packet:batch`, `health:change`, and `gap:detected` event types are defined in the inter-component contract (§5.1 API-1.7) so C10 can implement the receiving side. These are broadcast by separate polling tasks or trigger-based listeners:
  - `packet:batch` — polled from `raw_packets` per-sniffer every 2 seconds (`COUNT(*)` grouped by sniffer, windowed).
  - `health:change` — polled from `sniffer_heartbeat`, broadcast on status transition (online→offline or offline→online).
  - `gap:detected` — polled from `ingest_gaps` with `status = 'open'`.
- **Cookie-based auth for browser SSE:** The native `EventSource` API cannot send custom HTTP headers. The SSE endpoint supports cookie-based authentication: on first valid API key validation, C09 sets a `tianer_auth` cookie (Secure, SameSite=Strict, HttpOnly). The SSE endpoint accepts either the `X-API-Key` header OR the `tianer_auth` cookie for authentication. The browser's `EventSource` connects with `withCredentials: true` to send the cookie automatically.
- The polling interval is 2 seconds. This trades freshness for simplicity. A post-MVP improvement could use PostgreSQL `LISTEN/NOTIFY` to eliminate polling.
- The SSE manager is a module-level singleton. In a multi-worker deployment (multiple uvicorn workers), each worker would have its own SSE manager — acceptable for a single-user LAN deployment. Post-MVP could use Redis pub/sub for cross-worker broadcast.

### 4.6 Application Lifespan

```python
# main.py

from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi.staticfiles import StaticFiles
from db import create_pools, close_pools
from config import settings
from auth import ApiKeyAuth
from rate_limit import RateLimitMiddleware
from metrics import setup_metrics
from sse import sse_manager

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    read_pool = await create_pools(settings)
    app.state.read_pool = read_pool
    app.state.write_pool = write_pool  # (same create_pools returns both)
    setup_metrics(app)
    # Start SSE polling in background
    app.state.sse_task = asyncio.create_task(
        sse_manager.start_polling(read_pool, interval=2.0)
    )
    yield
    # Shutdown
    app.state.sse_task.cancel()
    await close_pools(app.state.read_pool, app.state.write_pool)

app = FastAPI(
    title="Tian'er REST API",
    version="1.0.0",
    lifespan=lifespan,
    docs_url="/docs",
    redoc_url=None,  # Only Swagger UI, no ReDoc
)

# Middleware order matters: CORS → Rate Limit → (Auth is per-router)
app.add_middleware(CORSMiddleware, ...)  # See §8.4
app.add_middleware(RateLimitMiddleware, rate=100.0, burst=150)

# Auth
auth = ApiKeyAuth(key_path=settings.api_key_file)

# Routers
from routers import devices, device_detail, alerts, events, health, metrics_router
app.include_router(devices.router, dependencies=[Depends(auth)])
app.include_router(device_detail.router, dependencies=[Depends(auth)])
app.include_router(alerts.router, dependencies=[Depends(auth)])
app.include_router(events.router)  # Auth applied inside the router
app.include_router(health.router)  # No auth
app.include_router(metrics_router.router)  # No auth

# Static files (frontend) — must be LAST to avoid shadowing API routes
app.mount("/", StaticFiles(directory="/usr/share/tianer/frontend", html=True), name="frontend")
```

---

## 5. Inter-Component Contracts

### 5.1 Contract API-1: REST Endpoint Definitions

This is the definitive endpoint contract between C09 REST API and C10 Frontend. All endpoint paths, methods, query parameters, request headers, response statuses, and response bodies are specified here. This contract replaces CONTRACT 8.10-A from inception.md with a more complete specification including MAC encoding details, SSE, and `/api/metrics`.

#### API-1.1: GET /api/health

| Property | Value |
|----------|-------|
| **Method** | GET |
| **Path** | `/api/health` |
| **Auth** | None (public health check) |
| **Query Params** | None |
| **Success Response** | `200 OK`, `Content-Type: application/json` |
| **Response Model** | `HealthStatus` (§3.5) |
| **Error Responses** | `503 Service Unavailable` if DB is unreachable (still returns `HealthStatus` with `status: "error"` and `db: "error"`) |
| **Rate Limited** | No |
| **Caching** | `Cache-Control: no-store` |

**Example response:**
```json
{
    "status": "ok",
    "db": "ok",
    "db_migration": "0005_device_enrichment",
    "db_connections": 23,
    "db_size_mb": 1250.5,
    "sniffers": [
        {
            "id": 1,
            "name": "ut1",
            "type": "ubertooth",
            "enabled": true,
            "last_heartbeat": "2026-06-09T14:30:00Z",
            "running": true
        }
    ],
    "ingest_gaps_open": 0
}
```

#### API-1.2: GET /api/devices

| Property | Value |
|----------|-------|
| **Method** | GET |
| **Path** | `/api/devices` |
| **Auth** | `X-API-Key` header required |
| **Query Params** | `q` (string, optional) — hex MAC prefix search (e.g. `aa:bb`). `class` (string, optional) — filter by `residency_class`. `seen_after` (ISO 8601, optional) — only devices last seen after this timestamp. `sort` (string, optional, default `-last_seen`) — sort field with `-` prefix for descending. Valid: `last_seen`, `-last_seen`, `first_seen`, `-first_seen`, `total_count`, `-total_count`. `limit` (int, optional, default 50, max 500). `offset` (int, optional, default 0). |
| **Success Response** | `200 OK`, `Content-Type: application/json` |
| **Response Model** | `PaginatedResponse[DeviceSummary]` |
| **Error Responses** | `400 Bad Request` — invalid query parameter. `401 Unauthorized` — missing/invalid API key. `429 Too Many Requests` — rate limit exceeded. `503 Service Unavailable` — database down (RFC 9110 Section 15.6.4) [3]. |
| **Rate Limited** | Yes (100 req/s) |

**Example request:**
```
GET /api/devices?q=aa:bb&class=resident&sort=-total_count&limit=20&offset=0
X-API-Key: dGhpc2lzYXRlc3RrZXk=
```

**Example response:**
```json
{
    "total": 42,
    "limit": 20,
    "offset": 0,
    "items": [
        {
            "mac": "aa:bb:cc:dd:ee:ff",
            "address_type": "public",
            "first_seen": "2026-06-01T00:00:00Z",
            "last_seen": "2026-06-09T14:25:00Z",
            "total_count": 15234,
            "distinct_days": 9,
            "residency_class": "resident",
            "classes": ["apple_continuity"]
        }
    ]
}
```

#### API-1.3: GET /api/devices/{mac}

| Property | Value |
|----------|-------|
| **Method** | GET |
| **Path** | `/api/devices/{mac}` |
| **Path Param** | `mac` — Bluetooth device address, hex colon-separated. Case-insensitive. Must match regex `^([0-9a-fA-F]{2}:){5}[0-9a-fA-F]{2}$`. |
| **Auth** | `X-API-Key` header required |
| **Query Params** | None |
| **Success Response** | `200 OK`, `Content-Type: application/json` |
| **Response Model** | `DeviceDetail` (§3.2) |
| **Error Responses** | `400 Bad Request` — invalid MAC format. `401 Unauthorized`. `404 Not Found` — no device with this MAC. `429 Too Many Requests`. `503 Service Unavailable`. |

#### API-1.4: GET /api/devices/{mac}/timeline

| Property | Value |
|----------|-------|
| **Method** | GET |
| **Path** | `/api/devices/{mac}/timeline` |
| **Path Param** | `mac` (as above) |
| **Auth** | `X-API-Key` header required |
| **Query Params** | `from` (ISO 8601, optional, default: 24 hours ago). `to` (ISO 8601, optional, default: now). `bucket` (string, optional, default `5m`). Valid: `5m`, `1h`, `1d`. |
| **Success Response** | `200 OK`, `Content-Type: application/json` |
| **Response Model** | `TimelineResponse` |
| **Error Responses** | `400`, `401`, `404`, `429`, `503` |

**Implementation notes:**
- `bucket=5m` uses `device_5min_buckets` continuous aggregate.
- `bucket=1h` wraps `device_5min_buckets` into 1-hour buckets with `time_bucket('1 hour', bucket)` in a subquery.
- `bucket=1d` wraps into daily buckets.
- `from`/`to` range is clamped to 90 days maximum (the `raw_packets` retention window).

#### API-1.5: GET /api/devices/{mac}/enrichment

| Property | Value |
|----------|-------|
| **Method** | GET |
| **Path** | `/api/devices/{mac}/enrichment` |
| **Path Param** | `mac` (as above) |
| **Auth** | `X-API-Key` header required |
| **Query Params** | `limit` (int, optional, default 100, max 500). `offset` (int, optional, default 0). |
| **Success Response** | `200 OK`, `Content-Type: application/json` |
| **Response Model** | `PaginatedResponse[EnrichmentRecord]` |
| **Error Responses** | `400`, `401`, `404`, `429`, `503` |

#### API-1.6: GET /api/alerts/new-devices

| Property | Value |
|----------|-------|
| **Method** | GET |
| **Path** | `/api/alerts/new-devices` |
| **Auth** | `X-API-Key` header required |
| **Query Params** | `since` (ISO 8601, optional, default: 24 hours ago). `limit` (int, optional, default 100, max 500). `offset` (int, optional, default 0). |
| **Success Response** | `200 OK`, `Content-Type: application/json` |
| **Response Model** | `PaginatedResponse[DeviceSummary]` |
| **Error Responses** | `400`, `401`, `429`, `503` |

**Query logic:**
```sql
SELECT encode(mac_address, 'hex') AS mac, address_type, first_seen, last_seen,
       total_count, distinct_days, residency_class,
       COALESCE(enrichment_data->'classes', '[]'::jsonb) AS classes
FROM bluetooth.device_summary
WHERE first_seen > $1
ORDER BY first_seen DESC
LIMIT $2 OFFSET $3;
```

#### API-1.7: GET /api/events (SSE)

| Property | Value |
|----------|-------|
| **Method** | GET |
| **Path** | `/api/events` |
| **Auth** | Cookie-based (`tianer_auth`) OR `X-API-Key` header. Browser clients use `EventSource` with `withCredentials: true`; the cookie is set by C09 on first API key validation via `POST /api/auth/login` or on first valid `X-API-Key` request. |
| **Query Params** | None |
| **Response Content-Type** | `text/event-stream` |
| **Connection** | Long-lived (keep-alive). Server sends events as they occur. |
| **Event Types** | `device:new`, `packet:batch`, `health:change`, `gap:detected` (§4.5) |
| **Error Responses** | `401 Unauthorized`. `429 Too Many Requests`. Connection dropped → client should reconnect after 5 seconds with exponential backoff. |
| **Rate Limited** | Yes (counts as 1 token when connection established; SSE data frames do not consume tokens) |

**Client reconnection:** The `EventSource` API automatically reconnects on connection loss. The server does not maintain client state — reconnecting clients simply start receiving events from the current point. The `Last-Event-ID` header is not required for v1 (all events are stateless).

#### API-1.8: GET /api/metrics

| Property | Value |
|----------|-------|
| **Method** | GET |
| **Path** | `/api/metrics` |
| **Auth** | None (bound to loopback only) |
| **Query Params** | None |
| **Response Content-Type** | `text/plain; version=0.0.4` |
| **Rate Limited** | No |

This endpoint exposes the Prometheus metrics registry in OpenMetrics text format. It is consumed by C13 (Prometheus scraper). The endpoint is **only accessible from localhost** — the uvicorn server is configured with `--host=0.0.0.0` for the API but an additional middleware check ensures `/api/metrics` returns 403 for non-loopback requests.

**Metrics exposed (§7.1):**
- `tianer_api_requests_total{method,path,status_code}` — counter
- `tianer_api_request_duration_seconds{method,path}` — histogram
- `tianer_api_active_connections` — gauge
- `tianer_api_errors_total{method,path,error_type}` — counter
- `tianer_api_db_pool_size{pool}` — gauge (read/write pool sizes)
- `tianer_api_db_pool_available{pool}` — gauge
- `tianer_api_sse_clients` — gauge (number of connected SSE clients)
- `tianer_api_rate_limit_hits_total` — counter

#### API-1.9: GET /api/packets

| Property | Value |
|----------|-------|
| **Method** | GET |
| **Path** | `/api/packets` |
| **Auth** | `X-API-Key` header required |
| **Query Params** | `sniffer_id` (int, optional, 1–4) — filter by sniffer. `mac` (string, optional) — hex colon-separated MAC address filter (exact match on `encode(mac_address, 'hex')`). `pdu_type` (string, optional) — filter by PDU type (e.g. `ADV_IND`, `SCAN_RSP`, `ADV_NONCONN_IND`). `channel` (int, optional, 37–39) — filter by BLE advertising channel. `since` (ISO 8601, optional, default: 24 hours ago) — only packets after this timestamp. `limit` (int, optional, default 50, max 500). `offset` (int, optional, default 0). |
| **Sort** | `-ts` (default, newest first). No alternate sort options in v1 — packet lists are always reverse-chronological. |
| **Success Response** | `200 OK`, `Content-Type: application/json` |
| **Response Model** | `PacketListResponse` (§3.10) |
| **Error Responses** | `400 Bad Request` — invalid query parameter. `401 Unauthorized` — missing/invalid API key. `429 Too Many Requests`. `503 Service Unavailable` — database down. |
| **Rate Limited** | Yes (100 req/s) |
| **DB Role** | `tianer_ro` |

**Example request:**
```
GET /api/packets?sniffer_id=1&pdu_type=ADV_IND&channel=37&limit=50&offset=0
X-API-Key: dGhpc2lzYXRlc3RrZXk=
```

**Example response:**
```json
{
    "packets": [
        {
            "ts": "2026-06-09T14:30:05Z",
            "sniffer_id": 1,
            "mac": "aa:bb:cc:dd:ee:ff",
            "rssi": -62,
            "pdu_type": "ADV_IND",
            "channel": 37,
            "raw_data_hex": "0201060303e0ff0c094d79446576696365"
        }
    ],
    "total": 15234,
    "limit": 50,
    "offset": 0
}
```

**Implementation notes:**
- The query targets `bluetooth.raw_packets` hypertable. Since this is a time-series table, the `since` parameter is used for partition pruning.
- `mac` filtering uses exact match: `WHERE encode(mac_address, 'hex') = $N` (lowercased).
- `pdu_type` filtering maps the string to the `SMALLINT` encoding from the deep parser (C07). The query builder maintains a `PDU_TYPE_MAP` dictionary for this conversion.
- If `sniffer_id`, `mac`, `pdu_type`, and `channel` are all omitted, returns all packets in the time window — this can be a heavy query. The frontend should always provide at least one filter or a narrow `since` window.

#### API-1.10: GET /api/sniffers

| Property | Value |
|----------|-------|
| **Method** | GET |
| **Path** | `/api/sniffers` |
| **Auth** | `X-API-Key` header required |
| **Query Params** | None |
| **Success Response** | `200 OK`, `Content-Type: application/json` |
| **Response Model** | `SnifferListResponse` (§3.12) |
| **Error Responses** | `401 Unauthorized` — missing/invalid API key. `429 Too Many Requests`. `503 Service Unavailable` — database down. |
| **Rate Limited** | Yes (100 req/s) |
| **DB Role** | `tianer_ro` |

**Example response:**
```json
{
    "sniffers": [
        {
            "sniffer_id": 1,
            "name": "ut1",
            "type": "ubertooth",
            "mode": "btle",
            "channel": 37,
            "enabled": true,
            "last_heartbeat": "2026-06-09T14:30:05Z",
            "status": "running"
        },
        {
            "sniffer_id": 2,
            "name": "nrf1",
            "type": "nrf52840",
            "mode": "btle",
            "channel": 38,
            "enabled": true,
            "last_heartbeat": "2026-06-09T14:29:55Z",
            "status": "running"
        }
    ]
}
```

**Implementation notes:**
- The query joins `bluetooth.sniffers` with the most recent row from `bluetooth.sniffer_heartbeat` per sniffer.
- `status` is derived at the API layer: `running` if heartbeat is within 60 seconds and `enabled = TRUE`; `stopped` if `enabled = TRUE` but heartbeat is stale; `error` if the sniffer process is not running; `unknown` if no heartbeat has ever been recorded.
- This endpoint is consumed primarily by the `SideStatusPanel` component. It is also polled by the SSE polling task to detect sniffer state transitions.
- No pagination — the number of sniffers is small (≤ 4 in the MVP architecture, max 8 with future expansion).

#### API-1.11: GET /api/stats

| Property | Value |
|----------|-------|
| **Method** | GET |
| **Path** | `/api/stats` |
| **Auth** | `X-API-Key` header required |
| **Query Params** | None |
| **Success Response** | `200 OK`, `Content-Type: application/json` |
| **Response Model** | `DashboardStats` (§3.13) |
| **Error Responses** | `401 Unauthorized` — missing/invalid API key. `429 Too Many Requests`. `503 Service Unavailable` — database down. |
| **Rate Limited** | Yes (100 req/s) |
| **DB Role** | `tianer_ro` |
| **Caching** | `Cache-Control: public, max-age=30`. Stats do not require per-request freshness — the DashboardView polls at 60-second intervals. A 30-second cache TTL prevents redundant database queries during rapid page loads while keeping data adequately fresh. |

**Example response:**
```json
{
    "total_devices": 42,
    "active_devices_1h": 15,
    "new_devices_24h": 3,
    "packets_today": 125000,
    "packet_rate_current": 127,
    "sniffers_online": 2,
    "sniffers_total": 4,
    "gaps_open": 0,
    "disk_usage_pct": 62.0,
    "uptime_seconds": 302400
}
```

**Implementation notes:**
- `total_devices`: `SELECT COUNT(*) FROM bluetooth.device_summary`
- `active_devices_1h`: `SELECT COUNT(*) FROM bluetooth.device_summary WHERE last_seen > NOW() - INTERVAL '1 hour'`
- `new_devices_24h`: `SELECT COUNT(*) FROM bluetooth.device_summary WHERE first_seen > NOW() - INTERVAL '24 hours'`
- `packets_today`: Queries the `raw_packets` hypertable with `WHERE ts >= CURRENT_DATE`. TimescaleDB chunk exclusion optimises this query.
- `packet_rate_current`: `SELECT COUNT(*) FROM bluetooth.raw_packets WHERE ts > NOW() - INTERVAL '60 seconds'`, divided by 60. If zero packets in the window, returns 0.
- `sniffers_online`: derived from the same logic as `SnifferStatus.status = 'running'`.
- `sniffers_total`: `SELECT COUNT(*) FROM bluetooth.sniffers`.
- `gaps_open`: `SELECT COUNT(*) FROM bluetooth.ingest_gaps WHERE status = 'open'`.
- `disk_usage_pct`: Queried from the host filesystem (PCAP directory usage) or from a pre-computed metric. In the MVP, this is read from the `bluetooth.system_metrics` table populated by C13 Observability. Falls back to `-1` if the metric is unavailable.
- `uptime_seconds`: Read from `/proc/uptime` on the host. In the container environment, this requires a bind-mount of `/proc/uptime` (read-only). Falls back to 0 if the file is unavailable.

**Query optimisation:** All sub-queries run in parallel using `asyncio.gather()` to minimise total response latency. The endpoint is cached for 30 seconds to avoid redundant computation.

#### API-1.12: Standard Error Responses

All endpoints return these standardized error bodies:

```json
// 400 Bad Request (validation error)
{
    "error": "validation_error",
    "message": "Invalid query parameter",
    "detail": "'limit' must be between 1 and 500"
}

// 401 Unauthorized
{
    "error": "unauthorized",
    "message": "Invalid API key",
    "detail": null
}

// 404 Not Found
{
    "error": "not_found",
    "message": "Device not found",
    "detail": "No device with MAC aa:bb:cc:dd:ee:ff in device_summary"
}

// 429 Too Many Requests
{
    "error": "rate_limit_exceeded",
    "message": "Too many requests. Retry after 1 second.",
    "detail": "Rate limit: 100 req/s"
}

// 503 Service Unavailable
{
    "error": "database_unavailable",
    "message": "Database connection failed",
    "detail": "Could not connect to tianer-postgres:5432 after 3 retries"
}
```

### 5.2 Contract DB-API (referenced from C02 §5.3)

The REST API connects to C02 Database under the terms of contract DB-API:

| Property | Value |
|----------|-------|
| **Read role** | `tianer_ro` — for all `/api/devices`, `/api/alerts`, `/api/devices/{mac}/*` read endpoints |
| **Write role** | `tianer` — for continuous aggregate refresh, residency classification, and any future write operations |
| **Unused role** | `tianer_writer` — NOT used by API (exclusive to C05, C06, C08) |
| **Auth** | scram-sha-256 |
| **Connection** | TCP to `tianer-postgres:5432` on `tianer-net` bridge |
| **Pooling** | `asyncpg` connection pool: 5 write, 10 read |
| **Query pattern** | All queries use parameterized `$N` placeholders via `query_builder` |

---

## 6. Failure Modes & Recovery

### 6.1 Failure Catalog

| # | Failure Mode | Detection | Impact | Recovery |
|---|-------------|-----------|--------|----------|
| **F-C09-1** | **Database unreachable** (container down, network partition, exhausted connections) | `asyncpg.exceptions.ConnectionDoesNotExistError` or `ConnectionRefusedError` on `pool.acquire()`. `/api/health` reports `db: "error"`. | All endpoints return `503 Service Unavailable`. SSE drops all clients. Frontend shows error state. Other platform components (ingest, gap detector) unaffected. | asyncpg pool auto-retries failed connection acquisitions with exponential backoff (1s, 2s, 4s, cap 30s). When PostgreSQL restarts, connections are re-established automatically. `/api/health` transitions `db` back to `"ok"` on the next probe. No persistent API state is lost. |
| **F-C09-2** | **Slow or hung query** | Query execution exceeds `statement_timeout`. asyncpg raises `TimeoutError`. Request metric records high duration. | Affected endpoint returns `503`. Other endpoints continue to function (per-pool connection isolation). | `statement_timeout` is set to **10 seconds** per connection via `SET statement_timeout = '10s'` on pool acquire. The slow query is cancelled. The connection is returned to the pool. The client may retry the request. |
| **F-C09-3** | **Invalid or missing API key** | Auth middleware check fails. | All `/api/*` endpoints return `401 Unauthorized`. `/api/health` and `/api/metrics` continue to work (no auth required). | Operator verifies the correct API key is configured in the frontend or provides it in the request header. Key file at `/etc/tianer/secrets/api_key` is readable by the container (mounted via V01). |
| **F-C09-4** | **Rate limit exceeded** | Token bucket exhausted for client IP. | Requests return `429 Too Many Requests`. Client should back off. | Client-side retry with backoff. Frontend implements debouncing on user-driven requests. The rate limit resets continuously (100 tokens/s refill). |
| **F-C09-5** | **API process crash (OOM, unhandled exception)** | uvicorn exits with non-zero code. `systemctl --user is-failed tianer-api`. | All endpoints unreachable. Frontend shows connection error. SSE connections dropped. Podman auto-restarts the container. | Podman restart policy: `Restart=on-failure`, `RestartSec=5s`. Container restarts in < 3 seconds. asyncpg pools are recreated in `lifespan` startup. SSE clients reconnect via `EventSource` auto-reconnect. **No in-flight request state persists** — all state is in the database (C02). |
| **F-C09-6** | **Disk full on V01 or container filesystem** | uvicorn log writes fail. `tianer_disk_usage_percent > 80%` metric. | API may continue to serve cached responses but cannot write logs. TLS cert/key still readable (mounted `:ro`). | C01 disk space alert fires at 80%. C04 emergency PCAP purge frees space. API container itself has minimal disk footprint — logs are written to journald (V04), not container disk. |
| **F-C09-7** | **TLS certificate expired or invalid** | Browser shows certificate warning. uvicorn starts but clients reject the connection. | API unreachable over HTTPS. Self-signed cert cannot be auto-renewed — it must be manually re-generated. | Operator runs `openssl req -x509 -newkey rsa:4096 -keyout /etc/tianer/secrets/tls.key -out /etc/tianer/secrets/tls.crt -days 3650 -nodes -subj "/CN=tianer"`. Restart the API container to load the new cert. |
| **F-C09-8** | **SSE polling task crashes** | `asyncio.CancelledError` or unhandled exception in `start_polling`. | SSE clients stop receiving events. REST endpoints are unaffected. | The `lifespan` context manager wraps the SSE task in a try/except. On exception, the task is logged and a new polling task is created. |

### 6.2 Connection Recovery

The API implements the same exponential backoff reconnect strategy as other consumers (C02 §6.2):

```
Attempt 1: wait 1s  →  retry acquire from pool
Attempt 2: wait 2s  →  retry
Attempt 3: wait 4s  →  retry
Attempt 4: wait 8s  →  retry
Attempt 5: wait 16s →  retry
Attempt 6+: wait 30s (cap) → retry indefinitely
```

asyncpg's built-in connection retry handles this at the pool level. The API does not give up on the database.

### 6.3 Statement Timeout

A per-connection `statement_timeout` of **10 seconds** is set on every connection acquired from the pool:

```python
async def on_acquire(connection):
    await connection.execute("SET statement_timeout = '10s'")
```

This prevents a single slow query (e.g., unoptimized full-table scan on `raw_packets`) from holding a connection indefinitely. The query is cancelled, the connection returns to the pool, and the endpoint returns a `503 Service Unavailable` with `{"error": "database_unavailable", "message": "Query timed out", "detail": "..."}`.

### 6.4 Graceful Degradation

When the API is down:
- **C10 Frontend** is served from browser cache (if previously loaded) or shows a connection error with a "Retry" button.
- **C11 Grafana** continues reading directly from the database (independent connection path).
- **C03 Capture Pipeline** continues writing PCAP and ingesting to DB.
- **C06 Gap Detector** continues detecting and backfilling gaps.

The API's unavailability does not cascade to other components (PF-4, PF-6).

---

## 7. Observability

### 7.1 Metrics

All metrics are exposed via `/api/metrics` in Prometheus text format. The API also logs request metrics in structured JSON to journald.

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `tianer_api_requests_total` | Counter | `method`, `path`, `status_code` | Total API requests served. |
| `tianer_api_request_duration_seconds` | Histogram | `method`, `path` | Request duration buckets: 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10. |
| `tianer_api_active_connections` | Gauge | — | Number of in-flight requests being processed. |
| `tianer_api_errors_total` | Counter | `method`, `path`, `error_type` | Error responses grouped by type: `auth_failure`, `validation_error`, `db_error`, `timeout`, `rate_limit`, `internal_error`. |
| `tianer_api_db_pool_size` | Gauge | `pool` (`read`, `write`) | Configured pool size. |
| `tianer_api_db_pool_available` | Gauge | `pool` | Available (idle) connections in pool. |
| `tianer_api_db_pool_checked_out` | Gauge | `pool` | Connections currently in use. |
| `tianer_api_sse_clients` | Gauge | — | Number of connected SSE clients. |
| `tianer_api_rate_limit_hits_total` | Counter | — | Number of requests rejected due to rate limiting. |

**Metrics implementation (`metrics.py`):**

```python
from prometheus_client import Counter, Histogram, Gauge, generate_latest, REGISTRY

REQUEST_COUNT = Counter(
    "tianer_api_requests_total",
    "Total API requests",
    ["method", "path", "status_code"],
)
REQUEST_DURATION = Histogram(
    "tianer_api_request_duration_seconds",
    "Request duration in seconds",
    ["method", "path"],
    buckets=[0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10],
)
ACTIVE_CONNECTIONS = Gauge("tianer_api_active_connections", "Active in-flight requests")
ERROR_COUNT = Counter(
    "tianer_api_errors_total",
    "Error responses",
    ["method", "path", "error_type"],
)
SSE_CLIENTS = Gauge("tianer_api_sse_clients", "Connected SSE clients")
RATE_LIMIT_HITS = Counter(
    "tianer_api_rate_limit_hits_total",
    "Rate limit rejections",
)
```

### 7.2 Structured Logging

All API logs are written to stderr in structured JSON format, consumed by journald:

```
TIANER | {"ts":"2026-06-09T14:30:00.123Z","level":"INFO","component":"api","msg":"Request completed","method":"GET","path":"/api/devices","status":200,"duration_ms":23,"client_ip":"127.0.0.1"}
TIANER | {"ts":"2026-06-09T14:30:01.456Z","level":"WARN","component":"api","msg":"Slow query detected","method":"GET","path":"/api/devices/aa:bb:cc:dd:ee:ff/timeline","duration_ms":4500,"query":"SELECT ..."}
TIANER | {"ts":"2026-06-09T14:30:02.789Z","level":"ERROR","component":"api","msg":"Database connection failed","pool":"read","error":"ConnectionRefusedError","retry_attempt":3}
```

**Log levels:**
- `INFO`: Normal request/response lifecycle, pool status changes, SSE client connect/disconnect.
- `WARN`: Slow queries (> 1 second), approaching pool limits (> 80% consumption), rate limit hits.
- `ERROR`: Database connection failures, unhandled exceptions, SSE polling failures.

### 7.3 Health Check

`/api/health` (§5.1 API-1.1) is the primary health endpoint. It is consumed by:

| Consumer | Frequency | Purpose |
|----------|-----------|---------|
| C11 Grafana | Every 30s | Show health status on pipeline-health dashboard |
| C10 Frontend | On page load | Display health panel |
| C13 Prometheus | Every 15s | Alert if `status != "ok"` for > 30s |
| Podman health check | Every 30s | `--health-cmd` for auto-restart (C14) |

**Podman health check command (C14 Quadlet):**
```ini
HealthCmd=curl -sf -o /dev/null http://127.0.0.1:8080/api/health
HealthInterval=30s
HealthTimeout=5s
HealthRetries=2
HealthStartPeriod=10s
```

### 7.4 Alert Summary

| Alert Name | Trigger | Severity | Action |
|------------|---------|----------|--------|
| `TianerAPIDown` | `tianer_api_requests_total` stops incrementing (via `absent()` query) for 60s | **Critical** | Check container status; restart if needed |
| `TianerAPIDBConnectionError` | `tianer_api_db_pool_available{pool="read"} == 0` for 2m | **Critical** | Check database availability; check connection limit |
| `TianerAPIHighLatency` | P95 `tianer_api_request_duration_seconds` > 2s for 5m | **Warning** | Check for slow queries in `pg_stat_statements`; check DB load |
| `TianerAPIHighErrorRate` | `rate(tianer_api_errors_total[5m]) / rate(tianer_api_requests_total[5m]) > 0.05` | **Warning** | Check error logs; investigate root cause |
| `TianerAPIRateLimitExceeded` | `rate(tianer_api_rate_limit_hits_total[1m]) > 10` | **Warning** | Check for misbehaving client; adjust rate limit if legitimate |
| `TianerAPISSEStale` | `tianer_api_sse_clients == 0` for > 5m while frontend is known to be loaded | **Info** | SSE polling task may have crashed; check logs |

---

## 8. Security Considerations

### 8.1 TLS (D-06)

All API traffic uses **mandatory TLS** with a **self-signed certificate**. The certificate and private key are stored at:

- `/etc/tianer/secrets/tls.crt` — Certificate (mode 0600, owner `tianer:tianer`)
- `/etc/tianer/secrets/tls.key` — Private key (mode 0600, owner `tianer:tianer`)

Both files are mounted into the API container via V01 (`:ro`). uvicorn is configured with:

```python
# In the container startup command (uvicorn):
# uvicorn tianer_api.main:app \
#   --host 0.0.0.0 \
#   --port 8080 \
#   --ssl-keyfile /etc/tianer/secrets/tls.key \
#   --ssl-certfile /etc/tianer/secrets/tls.crt
```

**Self-signed cert rationale:** The API is LAN-accessible only (bound to the Pi's internal IP). A self-signed cert provides encryption-in-transit without the operational complexity of Let's Encrypt on a device that may not have reliable internet access. The browser will show a certificate warning on first access — the operator must accept the exception. Post-MVP may add a proper CA-signed cert if the device has a public DNS name.

**Certificate generation (C01 `generate-secrets.sh`):**
```bash
openssl req -x509 -newkey rsa:4096 \
    -keyout /etc/tianer/secrets/tls.key \
    -out /etc/tianer/secrets/tls.crt \
    -days 3650 -nodes \
    -subj "/CN=tianer"
chmod 600 /etc/tianer/secrets/tls.key
chmod 600 /etc/tianer/secrets/tls.crt
chown tianer:tianer /etc/tianer/secrets/tls.*
```

### 8.2 API Key Authentication

- **Header:** `X-API-Key: <32-byte base64 string>`
- **Generation:** `openssl rand -base64 32` (C01 `generate-secrets.sh`)
- **Storage:** `/etc/tianer/secrets/api_key` (mount V01 `:ro`)
- **Comparison:** Constant-time via `secrets.compare_digest` (Python stdlib)
- **Scope:** All `/api/*` endpoints except `/api/health`
- **Rotation:** Restart the API container after replacing the key file. The key is read once at startup. Post-MVP: hot-reload via filesystem watcher.

**Frontend key handling:** The API key is **never stored in client-side JavaScript code**. In the MVP deployment, the key is injected into the frontend build as a build-time variable, or the frontend server-side rendering injects it into the initial HTML. Alternatively, the browser can prompt the user for the key on first load and store it in `sessionStorage` (lost on browser close). The exact mechanism is defined in C10 (Frontend) design document. The API does not support cookie-based sessions or OAuth in v1.

### 8.3 Parameterized Queries

**Every SQL query executed by the API uses parameterized `$N` placeholders.** The `query_builder` module (§4.3) enforces this at the type level. String interpolation into SQL is prohibited by design:

```python
# ALLOWED — parameterized query
await conn.fetch("SELECT * FROM bluetooth.device_summary WHERE mac_address = $1", mac_bytes)

# NEVER — string interpolation (would fail code review)
await conn.fetch(f"SELECT * FROM bluetooth.device_summary WHERE mac = '{mac}'")
```

Additional safeguards:
- MAC addresses are decoded from hex to `BYTEA` before being used in queries (not passed as hex strings).
- Column names in `ORDER BY` and `GROUP BY` are validated against a whitelist (`SORT_MAP` in `DeviceQueryBuilder`).
- The `limit` and `offset` values are clamped to safe ranges before interpolation into SQL (integers, no injection possible).
- Table names are hard-coded — no dynamic table selection.

### 8.4 CORS

Cross-Origin Resource Sharing is configured to allow only the frontend origin:

```python
from starlette.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        "https://tianer.local",
        "https://localhost:8080",
        "https://127.0.0.1:8080",
    ],
    allow_credentials=False,
    allow_methods=["GET", "OPTIONS"],
    allow_headers=["X-API-Key", "Content-Type", "Accept"],
    max_age=600,  # 10 minutes
)
```

**Rationale:** The API does not use cookies, so `allow_credentials=False` is safe. Only `GET` is needed (the API has no POST/PUT/DELETE endpoints in v1 — all writes go through PostgreSQL directly). `OPTIONS` is required for CORS preflight. The `X-API-Key` header is explicitly listed.

### 8.5 Input Validation

All input is validated at the API layer before any database query is constructed:

| Input Source | Validation |
|-------------|------------|
| Path parameter `{mac}` | Regex `^([0-9a-fA-F]{2}:){5}[0-9a-fA-F]{2}$`. Converted to `BYTEA` with `bytes.fromhex(mac.replace(":", ""))`. |
| Query parameter `q` | String, length ≤ 20 characters. Stripped of non-hex/non-colon characters. |
| Query parameter `class` | Enumerated: must be one of `unknown`, `new`, `lost`, `resident`, `frequent`, `transient`. |
| Query parameter `sort` | Whitelist: one of `last_seen`, `-last_seen`, `first_seen`, `-first_seen`, `total_count`, `-total_count`. |
| Query parameter `limit` | Integer, 1–500. Clamped. |
| Query parameter `offset` | Integer, ≥ 0. Clamped. |
| Query parameter `since`, `seen_after`, `from`, `to` | ISO 8601 datetime string. Validated by Pydantic `datetime` type. |
| Query parameter `bucket` | Enumerated: `5m`, `1h`, `1d`. |
| Header `X-API-Key` | String. Constant-time comparison. |

All validation is performed by Pydantic models at the FastAPI layer. Invalid input returns `400 Bad Request` with a descriptive error message before any database interaction occurs.

### 8.6 Loopback Binding for Metrics

The `/api/metrics` endpoint is accessible only from `127.0.0.1` (localhost). This is enforced by a middleware check:

```python
# In metrics_router.py
from fastapi import Request

@router.get("/api/metrics")
async def metrics(request: Request):
    if request.client and request.client.host not in ("127.0.0.1", "::1"):
        raise HTTPException(status_code=403, detail="Metrics only available from localhost")
    return Response(content=generate_latest(REGISTRY), media_type="text/plain; version=0.0.4")
```

This prevents external clients from scraping metrics that might reveal sensitive operational data without requiring authentication on the metrics endpoint.

### 8.7 Container Security

The API container runs with minimal privileges:
- `--cap-drop ALL` — no kernel capabilities
- `--read-only` root filesystem (except `/tmp` for uvicorn's temporary needs)
- `V01` mounted `:ro` — config and secrets cannot be modified
- No host network access — uses `tianer-net` bridge
- `PublishPort=127.0.0.1:8080:8080` in Quadlet — only loopback on host
- `MemoryMax=256M` — resource limit
- `CPUQuota=150%` — 1.5 CPU cores max

---

## 9. Configuration

### 9.1 Environment Variables

| Variable | Default | Purpose | Source |
|----------|---------|---------|--------|
| `TIANER_DB_HOST` | `tianer-postgres` | PostgreSQL hostname | C14 Quadlet `Environment=` |
| `TIANER_DB_PORT` | `5432` | PostgreSQL port | C14 Quadlet `Environment=` |
| `TIANER_DB_NAME` | `tianer` | Database name | C14 Quadlet `Environment=` |
| `TIANER_DB_RO_USER` | `tianer_ro` | Read-only role name | C14 Quadlet `Environment=` |
| `TIANER_DB_RO_PASSWORD_FILE` | `/etc/tianer/secrets/ro_db_password` | File containing read-only password | C14 Quadlet `Environment=` |
| `TIANER_DB_RW_USER` | `tianer` | Read-write role name | C14 Quadlet `Environment=` |
| `TIANER_DB_RW_PASSWORD_FILE` | `/etc/tianer/secrets/db_password` | File containing read-write password | C14 Quadlet `Environment=` |
| `TIANER_API_KEY_FILE` | `/etc/tianer/secrets/api_key` | File containing API key | C14 Quadlet `Environment=` |
| `TIANER_API_HOST` | `0.0.0.0` | uvicorn bind address | C14 Quadlet `Environment=` |
| `TIANER_API_PORT` | `8080` | uvicorn listen port | C14 Quadlet `Environment=` |
| `TIANER_API_RATE_LIMIT` | `100` | Requests per second per IP | C14 Quadlet `Environment=` |
| `TIANER_API_RATE_BURST` | `150` | Maximum burst size | C14 Quadlet `Environment=` |
| `TIANER_API_CORS_ORIGINS` | `https://tianer.local,https://localhost:8080` | Comma-separated allowed origins | C14 Quadlet `Environment=` |
| `TIANER_API_LOG_LEVEL` | `info` | Log level: debug, info, warn, error | C14 Quadlet `Environment=` |
| `TIANER_API_DB_POOL_READ_SIZE` | `10` | Read connection pool size | C14 Quadlet `Environment=` |
| `TIANER_API_DB_POOL_WRITE_SIZE` | `5` | Write connection pool size | C14 Quadlet `Environment=` |
| `TIANER_API_STATEMENT_TIMEOUT_S` | `10` | Per-query statement timeout (seconds) | C14 Quadlet `Environment=` |
| `TIANER_API_SSE_POLL_INTERVAL_S` | `2` | SSE new-device poll interval (seconds) | C14 Quadlet `Environment=` |
| `TIANER_TLS_CERT_FILE` | `/etc/tianer/secrets/tls.crt` | TLS certificate file | C14 Quadlet `Environment=` |
| `TIANER_TLS_KEY_FILE` | `/etc/tianer/secrets/tls.key` | TLS private key file | C14 Quadlet `Environment=` |

### 9.2 Config Loading (`config.py`)

```python
# config.py

import os
from dataclasses import dataclass, field

@dataclass
class Settings:
    db_host: str = os.getenv("TIANER_DB_HOST", "tianer-postgres")
    db_port: int = int(os.getenv("TIANER_DB_PORT", "5432"))
    db_name: str = os.getenv("TIANER_DB_NAME", "tianer")

    db_ro_user: str = os.getenv("TIANER_DB_RO_USER", "tianer_ro")
    db_ro_password_file: str = os.getenv(
        "TIANER_DB_RO_PASSWORD_FILE", "/etc/tianer/secrets/ro_db_password"
    )
    db_rw_user: str = os.getenv("TIANER_DB_RW_USER", "tianer")
    db_rw_password_file: str = os.getenv(
        "TIANER_DB_RW_PASSWORD_FILE", "/etc/tianer/secrets/db_password"
    )

    api_key_file: str = os.getenv(
        "TIANER_API_KEY_FILE", "/etc/tianer/secrets/api_key"
    )
    api_host: str = os.getenv("TIANER_API_HOST", "0.0.0.0")
    api_port: int = int(os.getenv("TIANER_API_PORT", "8080"))
    api_rate_limit: float = float(os.getenv("TIANER_API_RATE_LIMIT", "100"))
    api_rate_burst: int = int(os.getenv("TIANER_API_RATE_BURST", "150"))
    cors_origins: list[str] = field(default_factory=lambda: os.getenv(
        "TIANER_API_CORS_ORIGINS",
        "https://tianer.local,https://localhost:8080"
    ).split(","))
    log_level: str = os.getenv("TIANER_API_LOG_LEVEL", "info")

    db_pool_read_size: int = int(os.getenv("TIANER_API_DB_POOL_READ_SIZE", "10"))
    db_pool_write_size: int = int(os.getenv("TIANER_API_DB_POOL_WRITE_SIZE", "5"))
    statement_timeout_s: int = int(os.getenv("TIANER_API_STATEMENT_TIMEOUT_S", "10"))
    sse_poll_interval_s: float = float(os.getenv("TIANER_API_SSE_POLL_INTERVAL_S", "2"))

    tls_cert_file: str = os.getenv(
        "TIANER_TLS_CERT_FILE", "/etc/tianer/secrets/tls.crt"
    )
    tls_key_file: str = os.getenv(
        "TIANER_TLS_KEY_FILE", "/etc/tianer/secrets/tls.key"
    )

    @property
    def db_ro_password(self) -> str:
        with open(self.db_ro_password_file, "r") as f:
            return f.read().strip()

    @property
    def db_rw_password(self) -> str:
        with open(self.db_rw_password_file, "r") as f:
            return f.read().strip()

    @property
    def api_key(self) -> str:
        with open(self.api_key_file, "r") as f:
            return f.read().strip()

settings = Settings()
```

### 9.3 Tuning Guidelines

| Scenario | Parameter | Suggested Value |
|----------|-----------|-----------------|
| High frontend concurrency (multiple browser tabs) | `TIANER_API_DB_POOL_READ_SIZE` | 20 |
| Database on fast NVMe | `TIANER_API_STATEMENT_TIMEOUT_S` | 5 (lower is more aggressive) |
| Slow Raspberry Pi with heavy query workload | `TIANER_API_STATEMENT_TIMEOUT_S` | 15 |
| Aggressive rate limiting for shared deployment | `TIANER_API_RATE_LIMIT` | 30 |
| Single-user deployment (generous) | `TIANER_API_RATE_LIMIT` | 200 |
| Latency-sensitive timeline queries | `TIANER_API_SSE_POLL_INTERVAL_S` | 1 |

---

## 10. Test Plan

### 10.1 Unit Tests (pytest + httpx)

All tests use **pytest 8.0+** [7] with **pytest-asyncio 0.23+** and **httpx** [8] for async HTTP client testing. Tests run against a real PostgreSQL test database (not mocks).

| Test File | What It Tests | Key Assertions |
|-----------|---------------|----------------|
| `test_health.py` | `/api/health` endpoint | Returns 200 with `status` field. DB `ok` when connected. Returns `status: "error"` when DB down. Sniffers list populated from `sniffers` table. |
| `test_devices.py` | `/api/devices` (list) | Returns `PaginatedResponse` with correct `total`, `limit`, `offset`. `q` filter matches MAC prefix. `class` filter matches residency. `sort` orders correctly. `limit` and `offset` paginate. Invalid `class` returns 400. |
| `test_device_detail.py` | `/api/devices/{mac}`, `/api/devices/{mac}/timeline`, `/api/devices/{mac}/enrichment` | Detail returns `DeviceDetail` with correct fields. 404 for unknown MAC. Timeline returns buckets from cagg. Enrichment returns records with correct structure. `bucket` parameter works for `5m`/`1h`/`1d`. |
| `test_alerts.py` | `/api/alerts/new-devices` | Returns only devices with `first_seen > since`. Pagination works. |
| `test_auth.py` | API key authentication | Missing header → 401. Invalid key → 401. Valid key → 200. Health endpoint accessible without key. Metrics endpoint accessible without key (but loopback only). |
| `test_rate_limit.py` | Rate limiting middleware | Requests within limit → 200. Requests exceeding limit → 429 with `Retry-After` header. Rate resets after wait period. Health/metrics endpoints not rate-limited. |
| `test_validation.py` | Input validation | Invalid MAC format → 400. Negative limit → 400. Limit > 500 → clamped to 500. Invalid sort field → 400. Invalid bucket → 400. Missing required params → 422. |
| `test_sse.py` | SSE streaming endpoint | Connection returns `text/event-stream`. Keep-alive comments arrive within 16 seconds. `device:new` event fires when test inserts device. Client disconnect is clean. Cookie auth works with `withCredentials`. |  |
| `test_metrics.py` | `/api/metrics` endpoint | Returns `text/plain`. Contains expected metric names. Non-loopback request → 403. |

### 10.2 Integration Tests

| Test | Description | Acceptance Criteria |
|------|-------------|---------------------|
| `test_api_to_db.sh` (from `tests/integration/`) | End-to-end: seed DB via `seed-test-data.py`, query all API endpoints, validate responses | All endpoints return expected schemas. Data matches seed input. |
| `test_api_pagination.sh` | Seed 500 devices, paginate through in 50-device pages | All 500 devices seen. No duplicates. Total consistent across pages. |
| `test_api_concurrent.sh` | 10 concurrent clients hitting `/api/devices` for 30 seconds | Zero 503 errors. P95 latency < 200ms. Rate limiter not triggered. |
| `test_api_db_down.sh` | Stop PostgreSQL, verify API degrades gracefully | `/api/health` returns `status: "error"`, `db: "error"`. `/api/devices` returns 503. API recovers automatically when PostgreSQL restarts. |
| `test_api_slow_query.sh` | Inject artificial delay via `pg_sleep(15)` into a query | Query cancelled after `statement_timeout`. Endpoint returns 503 with timeout detail. Connection returned to pool. |

### 10.3 Performance Tests

| Test | Target | Measurement |
|------|--------|-------------|
| `/api/devices` (default params) | P95 < 200ms | `tianer_api_request_duration_seconds` histogram |
| `/api/devices/{mac}/timeline` (24h, 5m buckets) | P95 < 500ms | Same |
| `/api/health` with 4 sniffers | P95 < 50ms | Same |
| Concurrent requests: 10 clients, 100 req/s total | P95 < 500ms, 0 errors | Same + error counter |
| SSE latency: `new_device` event | < 3 seconds after DB insert | Manual timing |

### 10.4 Test Database Setup

Tests use a **dedicated test database** `tianer_test` created via `pytest fixtures` [7] with asyncpg [9]:

```python
# conftest.py

@pytest.fixture(scope="session")
def anyio_backend():
    return "asyncio"

@pytest.fixture(scope="session")
async def test_db():
    """Create a test database, apply migrations, seed data, yield connection."""
    # Connect to postgres default DB, create tianer_test
    sys_conn = await asyncpg.connect(
        host=os.getenv("TIANER_DB_HOST", "tianer-postgres"),
        port=int(os.getenv("TIANER_DB_PORT", "5432")),
        user="postgres",
        database="postgres",
    )
    await sys_conn.execute("DROP DATABASE IF EXISTS tianer_test")
    await sys_conn.execute("CREATE DATABASE tianer_test OWNER tianer")
    await sys_conn.close()

    # Apply migrations to tianer_test
    # ... (run apply-migrations.sh against tianer_test)

    # Create API test app with settings pointing to tianer_test
    # ...

    yield
    # Teardown: DROP DATABASE tianer_test

@pytest.fixture
async def client(test_db):
    """Return an httpx.AsyncClient for the test app."""
    transport = httpx.ASGITransport(app=test_db.app)
    async with httpx.AsyncClient(transport=transport, base_url="http://test") as ac:
        yield ac

@pytest.fixture
async def auth_client(client):
    """Return a client with X-API-Key header set."""
    client.headers["X-API-Key"] = "test-api-key"
    return client
```

### 10.5 Acceptance Criteria

1. **All endpoints respond per API-1 contract:** Paths, methods, status codes, response schemas match the specification in §5.1.
2. **OpenAPI schema** available at `/docs` and passes schema validation [2].
3. **Auth:** Missing `X-API-Key` returns 401. Invalid key returns 401. Valid key allows access. Health and metrics endpoints exempt.
4. **Pagination:** Large datasets paginate correctly with consistent totals.
5. **Input validation:** All invalid inputs return 400 or 422 with descriptive errors.
6. **Rate limiting:** Requests exceeding the limit receive 429 with `Retry-After` header.
7. **Graceful degradation:** When database is unreachable, `/api/health` reports `db: "error"` and data endpoints return 503.
8. **SSE:** Events arrive within the polling interval. Client disconnect is clean.
9. **Metrics:** `/api/metrics` returns valid Prometheus text format only to loopback.
10. **Performance:** P95 latency targets met (§10.3).

---

## 11. Deployment Notes

### 11.1 Container Image

The API container image is built from a minimal Python 3.13 base image using multi-stage builds:

```dockerfile
# Stage 1: Build (install dependencies)
FROM python:3.13-slim-bookworm AS builder
RUN pip install --no-cache-dir uv
COPY platform/api/pyproject.toml /app/
COPY platform/api/uv.lock /app/
WORKDIR /app
RUN uv sync --frozen --no-dev

# Stage 2: Runtime (minimal)
FROM python:3.13-slim-bookworm
COPY --from=builder /app/.venv /app/.venv
COPY platform/api/src/tianer_api /app/tianer_api
COPY platform/frontend/dist /usr/share/tianer/frontend/
ENV PATH="/app/.venv/bin:$PATH"
EXPOSE 8080
CMD ["uvicorn", "tianer_api.main:app", "--host", "0.0.0.0", "--port", "8080", \
     "--ssl-keyfile", "/etc/tianer/secrets/tls.key", \
     "--ssl-certfile", "/etc/tianer/secrets/tls.crt", \
     "--no-access-log"]
```

**Image size target:** Under 200 MB uncompressed. Python 3.13 slim is ~130 MB; FastAPI + uvicorn + asyncpg + dependencies add ~40 MB; application code is negligible.

### 11.2 Quadlet Unit

`deploy/containers/tianer-api.container` (Quadlet file, deployed by C14):

```ini
[Unit]
Description=Tian'er REST API (FastAPI)
After=tianer-net.network tianer-postgres.service
Requires=tianer-platform.pod
PartOf=tianer-platform.pod

[Container]
Image=localhost/tianer-api:latest
ContainerName=tianer-api
Pod=tianer-platform.pod
Network=tianer-net
PublishPort=127.0.0.1:8080:8080

# Volumes — V01 :ro for config and secrets
Volume=/etc/tianer/secrets/api_key:/etc/tianer/secrets/api_key:ro,Z
Volume=/etc/tianer/secrets/ro_db_password:/etc/tianer/secrets/ro_db_password:ro,Z
Volume=/etc/tianer/secrets/db_password:/etc/tianer/secrets/db_password:ro,Z
Volume=/etc/tianer/secrets/tls.crt:/etc/tianer/secrets/tls.crt:ro,Z
Volume=/etc/tianer/secrets/tls.key:/etc/tianer/secrets/tls.key:ro,Z
# tianer.env not needed — env vars injected via Environment below

Environment=TIANER_DB_HOST=tianer-postgres
Environment=TIANER_DB_PORT=5432
Environment=TIANER_DB_NAME=tianer
Environment=TIANER_DB_RO_USER=tianer_ro
Environment=TIANER_DB_RW_USER=tianer
Environment=TIANER_DB_RO_PASSWORD_FILE=/etc/tianer/secrets/ro_db_password
Environment=TIANER_DB_RW_PASSWORD_FILE=/etc/tianer/secrets/db_password
Environment=TIANER_API_KEY_FILE=/etc/tianer/secrets/api_key
Environment=TIANER_API_HOST=0.0.0.0
Environment=TIANER_API_PORT=8080
Environment=TIANER_API_CORS_ORIGINS=https://tianer.local,https://localhost:8080
Environment=TIANER_API_DB_POOL_READ_SIZE=10
Environment=TIANER_API_DB_POOL_WRITE_SIZE=5
Environment=TIANER_API_LOG_LEVEL=info

# Security hardening
NoNewPrivileges=true
DropCapability=ALL
ReadOnly=true
Tmpfs=/tmp:rw,noexec,nosuid,size=64M

# Resource limits
MemoryMax=256M
CPUQuota=150%

# Health check
HealthCmd=curl -sf -o /dev/null http://127.0.0.1:8080/api/health
HealthInterval=30s
HealthTimeout=5s
HealthRetries=2
HealthStartPeriod=10s

[Service]
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=default.target
```

### 11.3 Volume Mounts

| Volume | Path in Container | Access | Contents |
|--------|------------------|--------|----------|
| V01 | `/etc/tianer/secrets/api_key` | `:ro` | API key (single file bind-mount) |
| V01 | `/etc/tianer/secrets/ro_db_password` | `:ro` | Read-only DB password |
| V01 | `/etc/tianer/secrets/db_password` | `:ro` | Read-write DB password |
| V01 | `/etc/tianer/secrets/tls.crt` | `:ro` | TLS certificate |
| V01 | `/etc/tianer/secrets/tls.key` | `:ro` | TLS private key |
| (Image) | `/usr/share/tianer/frontend/` | N/A | Frontend static bundle (embedded in image) |
| (Tmpfs) | `/tmp` | `:rw` | Ephemeral temp files (tmpfs, 64 MB) |

The frontend static files are **embedded in the container image** — no separate volume mount is needed. This keeps the deployment simple: updating the frontend requires a new container image build, but frontend updates are expected to be infrequent in production. The `PublishPort` mapping of `127.0.0.1:8080:8080` ensures the API is only accessible on the host's loopback interface, but this can be changed to `0.0.0.0:8080:8080` for LAN access.

### 11.4 uvicorn Configuration

uvicorn is started with the following settings (see uvicorn deployment documentation [6]):

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| `--host` | `0.0.0.0` | Listen on all interfaces (container network isolation already restricts access) |
| `--port` | `8080` | Standard alternative HTTP port (8080 avoids conflict with potential host nginx on 80/443) |
| `--ssl-keyfile` | `/etc/tianer/secrets/tls.key` | Mandatory TLS (D-06) |
| `--ssl-certfile` | `/etc/tianer/secrets/tls.crt` | Mandatory TLS (D-06) |
| `--no-access-log` | (flag) | Access logs are handled by structured logging middleware, not uvicorn's default |
| `--workers` | `1` | Single worker for simplicity. asyncpg pools are not shared across workers. Post-MVP can add `gunicorn` with `uvicorn` workers. |
| `--timeout-keep-alive` | `5` | Close idle keep-alive connections after 5 seconds (SSE uses its own timeout) |
| `--limit-concurrency` | `200` | Maximum concurrent connections (SSE + REST). 200 is generous for a single-user Pi. |

### 11.5 Container Boot Order

1. `tianer-net.network` — bridge network created
2. `tianer-postgres.service` — PostgreSQL starts, runs init scripts, applies migrations
3. `tianer-platform.pod` — platform pod created
4. `tianer-api.service` — API container starts (waits for DB via `After=` dependency)
5. API runs `lifespan` startup: creates pools, warms connections, starts SSE polling
6. API returns 200 on `/api/health` within 10 seconds (HealthStartPeriod)
7. `tianer-capture.pod` — capture pod starts (independent of API)

### 11.6 Deployment Checklist

Before considering C09 deployed, verify:

- [ ] Container image builds successfully: `podman build -t localhost/tianer-api:latest -f deploy/containers/Dockerfile.api .`
- [ ] Container starts and passes health check within HealthStartPeriod
- [ ] `curl -k https://127.0.0.1:8080/api/health` returns 200 with `status: "ok"`
- [ ] `curl -k https://127.0.0.1:8080/api/health -H "X-API-Key: test"` still returns 200 (health is exempt from auth)
- [ ] `curl -k https://127.0.0.1:8080/api/devices` returns 401 (missing API key)
- [ ] `curl -k https://127.0.0.1:8080/api/devices -H "X-API-Key: $(cat /etc/tianer/secrets/api_key)"` returns 200 with empty device list
- [ ] `/api/metrics` accessible from localhost, returns Prometheus text format
- [ ] `/api/metrics` returns 403 from non-loopback IP
- [ ] `/docs` loads Swagger UI with all endpoints documented
- [ ] Frontend loads at `https://tianer.local:8080/` (served from container image)
- [ ] TLS certificate warning appears in browser (expected for self-signed cert)
- [ ] Structured logs appear in `journalctl CONTAINER_NAME=tianer-api` with `TIANER |` prefix
- [ ] Container restart on `kill` (send SIGTERM, verify auto-restart)

---

## Appendix A: Pydantic Model Summary

| Model | Endpoint(s) | DB Source Table(s) |
|-------|------------|-------------------|
| `DeviceSummary` | `/api/devices`, `/api/alerts/new-devices` | `device_summary` |
| `DeviceDetail` | `/api/devices/{mac}` | `device_summary` + `device_enrichment` (count) |
| `TimelineBucket` | `/api/devices/{mac}/timeline` | `device_5min_buckets` (cagg) |
| `EnrichmentRecord` | `/api/devices/{mac}/enrichment` | `device_enrichment` |
| `HealthStatus` | `/api/health` | `sniffers`, `sniffer_heartbeat`, `ingest_gaps`, `_migrations` |
| `SnifferHealth` | (part of `HealthStatus`) | `sniffers`, `sniffer_heartbeat` |
| `AlertInfo` | (SSE event) | `device_summary`, `ingest_gaps` |
| `PacketItem` | `/api/packets` | `raw_packets` |
| `PacketListResponse` | `/api/packets` | `raw_packets` |
| `SnifferStatus` | `/api/sniffers` | `sniffers`, `sniffer_heartbeat` |
| `SnifferListResponse` | `/api/sniffers` | `sniffers`, `sniffer_heartbeat` |
| `DashboardStats` | `/api/stats` | `device_summary`, `raw_packets`, `sniffers`, `sniffer_heartbeat`, `ingest_gaps`, `system_metrics` |
| `PaginatedResponse` | (wraps list endpoints) | — |
| `TimelineResponse` | (wraps timeline) | — |
| `ErrorResponse` | (all error responses) | — |

## Appendix B: Endpoint Quick Reference

| Method | Path | Auth | Rate Limited | DB Role | Purpose |
|--------|------|------|-------------|---------|---------|
| GET | `/api/health` | No | No | `tianer` | System health check |
| GET | `/api/devices` | Yes | Yes | `tianer_ro` | Paginated device list with filters |
| GET | `/api/devices/{mac}` | Yes | Yes | `tianer_ro` | Single device detail |
| GET | `/api/devices/{mac}/timeline` | Yes | Yes | `tianer_ro` | Device timeline aggregates |
| GET | `/api/devices/{mac}/enrichment` | Yes | Yes | `tianer_ro` | Device enrichment records |
| GET | `/api/alerts/new-devices` | Yes | Yes | `tianer_ro` | Recently first-seen devices |
| GET | `/api/packets` | Yes | Yes | `tianer_ro` | Paginated raw packet list |
| GET | `/api/sniffers` | Yes | Yes | `tianer_ro` | Sniffer status list |
| GET | `/api/stats` | Yes | Yes | `tianer_ro` | Aggregate dashboard stats |
| GET | `/api/events` | Yes | Yes (connect) | `tianer_ro` | SSE live event stream |
| GET | `/api/metrics` | No | No | N/A (in-memory) | Prometheus metrics |
| GET | `/*` (frontend) | No | No | N/A | SPA static files |

## Appendix C: DB Role Usage Summary

| Role | Used by C09 | Purpose | Privileges |
|------|-------------|---------|------------|
| `tianer_ro` | **Yes** | All read-only endpoints (`/api/devices`, `/api/devices/{mac}`, `/api/devices/{mac}/timeline`, `/api/devices/{mac}/enrichment`, `/api/alerts/new-devices`, SSE polling) | SELECT only |
| `tianer` | **Yes** | `/api/health` (writes health metadata to `sniffer_health_snapshot`), continuous aggregate refresh (calls `CALL refresh_continuous_aggregate()`), residency classification (calls `classify_residency()`) | ALL (DDL + DML) |
| `tianer_writer` | **No** | NOT used by the API. Exclusive to C05 Ingest Bridge, C06 Gap Detector, C08 ML Enrichment. | INSERT only |

---

## References

[1] S. Ramírez. "FastAPI — Advanced User Guide." https://fastapi.tiangolo.com/advanced/, 2024. Sections: Middleware (https://fastapi.tiangolo.com/advanced/middleware/), Custom Response — StreamingResponse (https://fastapi.tiangolo.com/advanced/custom-response/), Lifespan Events (https://fastapi.tiangolo.com/advanced/events/).

[2] OpenAPI Initiative. "OpenAPI Specification v3.2.0." https://spec.openapis.org/oas/v3.2.0.html, September 2025.

[3] IETF. "RFC 9110 — HTTP Semantics." https://www.rfc-editor.org/rfc/rfc9110, June 2022. Sections: 15.5.2 (401 Unauthorized), 15.5.19 (429 Too Many Requests — defined in RFC 6585, incorporated), 15.6.4 (503 Service Unavailable).

[4] Psycopg Development Team. "psycopg 3 — Connection Pools." https://www.psycopg.org/psycopg3/docs/advanced/pool.html, 2024.

[5] Python Software Foundation. "secrets — Generate secure random numbers for managing secrets." https://docs.python.org/3/library/secrets.html, Python 3.14 Documentation. `secrets.compare_digest()` constant-time comparison function.

[6] Encode. "Uvicorn — Settings." https://www.uvicorn.org/settings/, 2024. Deployment documentation covering `--workers`, `--proxy-headers`, `--ssl-keyfile`, `--ssl-certfile`.

[7] pytest. "pytest: helps you write better programs." https://docs.pytest.org/en/stable/, v8.0+. — Documents pytest testing framework: test discovery, fixture system (`@pytest.fixture` with session scope for test database setup), `pytest-asyncio` plugin for async test support, and `conftest.py` shared fixtures (§10.1, §10.4).

[8] httpx. "httpx — A next-generation HTTP client for Python." https://www.python-httpx.org/ — Documents httpx async HTTP client: `httpx.AsyncClient` with `ASGITransport` for in-process ASGI testing, cookie and header management, and timeout configuration — used for all API endpoint tests (§10.1, §10.2).

[9] MagicStack. "asyncpg — A fast PostgreSQL Database Client Library for Python/asyncio." https://github.com/MagicStack/asyncpg — Documents asyncpg's async connection API: `asyncpg.connect()`, connection pool (`asyncpg.create_pool()`), and parameterized query execution — used by C09 for database connections in test fixtures (§10.4) and production connection pools (§4.2).

[10] Python Software Foundation. "asyncio — Queues." https://docs.python.org/3/library/asyncio-queue.html, Python 3.14 Documentation. `asyncio.Queue` class used for SSE event distribution.

---

*End of C09 REST API Design Document*
