# C10 — Frontend (Vue 3 SPA)

**Component:** Shared Platform — Vue 3 Single-Page Application  
**Language:** TypeScript 5.3+ (strict mode), Vue 3.4+ (Composition API, `<script setup>`), Vite 5.0+  
**Depends on:** C09 REST API  
**Status:** Design — Phase B Ready  

---

## 1. Overview

### 1.1 Purpose

The Tian'er frontend is a dark-themed Single-Page Application (SPA) that serves as the primary operator interface for the Signal Intelligence Platform. It presents captured BLE device data, pipeline health, and embedded Grafana dashboards through four views: Dashboard, Device Map, Packet Explorer, and Settings. The application is served as static assets from the C09 REST API container and consumed by a single persona: the **home lab operator** — a technically sophisticated user running this instrument at home out of curiosity [PD-01].

### 1.2 Scope

**In scope (MVP):**
- Four views: Dashboard (landing), Device Map, Packet Explorer, Settings
- Expandable side status panel (sniffer health, green/grey indicators)
- Client-side device table with sorting and filtering (<200 devices)
- Time-series charts via ECharts 5 with dark theme
- Anonymous Grafana iframe embedding (1 dashboard: pipeline-health)
- SSE-based live data updates from C09 REST API
- WCAG 2.2 AA-lite compliance (keyboard nav, ARIA landmarks, focus management, contrast ≥4.5:1)
- TypeScript strict mode + OpenAPI-generated API types
- 4-state model per view: loading, empty, error, normal

**Out of scope (MVP):**
- Alert/notification system (deferred to post-MVP) [PD-06]
- `vue-sonner` toast notifications (post-MVP) [UI-08]
- Multi-user authentication UI (API key is server-side injected)
- Mobile-native UX (tablet support only) [UXE-09]
- Any CRT/glitch visual effects (explicitly excluded by PD-03-theme)

### 1.3 Position in the System

```
┌─────────────────────────────────────────────────────────┐
│                     Operator Browser                     │
│               (LAN-connected, dark mode)                 │
└──────┬──────────────────────────────────┬───────────────┘
       │ HTTPS :8443                      │ HTTPS :3000
       ▼                                  ▼
┌──────────────┐                   ┌──────────────┐
│  C09 REST API │                   │    Grafana    │
│  (serves SPA) │                   │  (anonymous)  │
└──────┬───────┘                   └──────┬───────┘
       │                                  │
       │ PostgreSQL                       │ PostgreSQL
       ▼                                  ▼
┌──────────────┐                   ┌──────────────┐
│     C02      │                   │     C02      │
│  TimescaleDB │                   │  TimescaleDB │
└──────────────┘                   └──────────────┘
```

The frontend is served as static files from the API container. It communicates with C09 via REST + SSE, and embeds Grafana dashboards via anonymous iframe [UXE-05].

---

## 2. High-Level Architecture (HLA)

### 2.1 Data Flow

```
Browser
  │
  ├─► Vue Router ──► View Component
  │                      │
  │                      ├─► Pinia Store ──► API Client (openapi-fetch)
  │                      │                      │
  │                      │                      ├─► REST: GET/POST ──► C09 API
  │                      │                      │
  │                      │                      └─► SSE: EventSource ──► C09 /api/events
  │                      │
  │                      ├─► ECharts 5 (time-series charts)
  │                      ├─► TanStack Table v8 (device table)
  │                      └─► Grafana iframe (anonymous, 1 dashboard: pipeline-health)
  │
  └─► Error Boundary (global error handler)
```

### 2.2 Architectural Layers

| Layer | Technology | Responsibility |
|-------|-----------|---------------|
| **Routing** | Vue Router 4 | SPA navigation, route guards, 4 views |
| **State Management** | Pinia 2.1+ | Server state caching, UI state, SSE event buffering |
| **Data Fetching** | openapi-fetch + openapi-typescript | Type-safe API client generated from C09 OpenAPI spec |
| **Real-time Updates** | SSE (EventSource) | Live packet counts, sniffer status changes |
| **Presentation** | Vue 3.4+ SFC (`<script setup>`) | Component rendering, Composition API composables |
| **Styling** | Tailwind CSS 4 | Dark theme, BreachLab design tokens, responsive |
| **Charting** | ECharts 5 | Time-series charts with dark theme |
| **Table** | TanStack Table v8 | Client-side sorting, filtering, column visibility |
| **Icons** | Lucide Vue | Tree-shakeable icon components |
| **Dates** | date-fns 3 | Tree-shakeable date formatting |
| **Testing** | Vitest 1.2+ / Vue Test Utils / MSW | Unit + component + store tests |

### 2.3 Neighbouring Components

| Neighbour | Direction | Protocol | Contract |
|-----------|-----------|----------|----------|
| C09 REST API | Outbound (client→server) | HTTPS REST + SSE | API-1 (REST endpoint definitions) [1] |
| C11 Grafana | Outbound (browser→server) | HTTPS iframe embed | Anonymous access, CSP frame-src allowlist |
| C13 Observability | Indirect | Via C09 /api/health | Frontend surfaces health status from API |

### 2.4 Monorepo Location

```
platform/frontend/
├── package.json
├── tsconfig.json
├── vite.config.ts
├── index.html
├── src/
│   ├── main.ts                    # App bootstrap
│   ├── App.vue                    # Root component, error boundary
│   ├── router/
│   │   └── index.ts               # Vue Router config (4 routes)
│   ├── stores/                    # Pinia stores
│   │   ├── devices.ts             # Device list + summary
│   │   ├── packets.ts             # Packet explorer state
│   │   ├── health.ts              # Sniffer health, SSE-connected
│   │   └── settings.ts            # UI preferences (localStorage-backed)
│   ├── views/
│   │   ├── DashboardView.vue      # Landing: stats + charts + status
│   │   ├── DeviceMapView.vue      # Device table with filters
│   │   ├── PacketExplorerView.vue # Packet detail browser
│   │   └── SettingsView.vue       # API key, Grafana URLs, display prefs
│   ├── components/
│   │   ├── layout/
│   │   │   ├── AppLayout.vue      # Shell: NavBar + SideStatus + <router-view>
│   │   │   ├── NavBar.vue         # Top nav with view tabs
│   │   │   └── SideStatusPanel.vue # Expandable sniffer health panel
│   │   ├── common/
│   │   │   ├── StatCard.vue       # Metric card with icon + value + delta
│   │   │   ├── DataTable.vue      # TanStack Table wrapper
│   │   │   ├── Badge.vue          # Status badge (green/grey/amber)
│   │   │   ├── ChartCard.vue      # ECharts wrapper in a card
│   │   │   ├── PageHeader.vue     # View title + description
│   │   │   ├── TimeRangeSelector.vue # Predefined range picker
│   │   │   ├── EmptyState.vue     # Empty state placeholder
│   │   │   ├── ErrorState.vue     # Error state with retry button
│   │   │   └── LoadingState.vue   # Skeleton/spinner placeholder
│   │   └── grafana/
│   │       └── GrafanaEmbed.vue   # Sandboxed iframe wrapper
│   ├── composables/
│   │   ├── useSSE.ts              # SSE connection management
│   │   ├── useApi.ts              # API client instance + auth header
│   │   ├── useTimeRange.ts        # Time range state + format helpers
│   │   └── useBreakpoint.ts       # Responsive breakpoint detection
│   ├── api/
│   │   └── client.ts              # Generated openapi-fetch client
│   ├── types/
│   │   └── api.ts                 # Generated openapi-typescript types
│   └── assets/
│       └── style.css               # Tailwind imports + theme tokens
└── tests/
    ├── setup.ts                    # MSW server + Vue Test Utils setup
    ├── mocks/
    │   └── handlers.ts             # MSW request handlers
    ├── stores/
    │   ├── devices.test.ts
    │   ├── packets.test.ts
    │   └── health.test.ts
    └── components/
        ├── StatCard.test.ts
        ├── DataTable.test.ts
        └── SideStatusPanel.test.ts
```

---

## 3. Data Model (ERD)

### 3.1 Client-Side State Entities

The frontend does not own a persistent data model — all data is transient, fetched from C09 REST API. However, it models the following entities in Pinia stores:

```
┌─────────────────────┐     ┌──────────────────────────┐
│    SnifferHealth    │     │        DeviceSummary       │
├─────────────────────┤     ├──────────────────────────┤
│ sniffer_id: number  │     │ device_address: string    │
│ name: string        │     │ address_type: enum        │
│ type: enum          │     │ last_seen: Date           │
│ status: enum        │     │ observation_count: number │
│ last_heartbeat: Date│     │ first_seen: Date          │
│ packets_total: number│    │ avg_rssi: number          │
│ channel: number     │     │ manufacturer: string?     │
│ uptime_seconds: num │     │ classification: enum      │
└─────────────────────┘     └──────────────────────────┘

┌──────────────────────────┐   ┌──────────────────────────┐
│       RawPacket           │   │      DashboardStats       │
├──────────────────────────┤   ├──────────────────────────┤
│ sniffer_id: number       │   │ total_devices: number     │
│ ts: Date                 │   │ active_devices_1h: number │
│ device_address: string   │   │ new_devices_24h: number   │
│ rssi: number             │   │ packets_today: number     │
│ pdu_type: enum           │   │ packet_rate_current: num  │
│ channel: number          │   │ sniffers_online: number   │
│ raw_data_hex: string?    │   │ sniffers_total: number    │
└──────────────────────────┘   │ gaps_open: number         │
                               │ disk_usage_pct: number    │
                               │ uptime_seconds: number    │
                               └──────────────────────────┘

┌──────────────────────────┐   ┌──────────────────────────┐
│      TimeRangeState       │   │      UISettingsState       │
├──────────────────────────┤   ├──────────────────────────┤
│ start: Date               │   │ theme: "dark" (fixed)     │
│ end: Date                 │   │ sidebarCollapsed: boolean │
│ preset: enum              │   │ grafanaBaseUrl: string    │
│ (1h, 6h, 24h, 7d, custom)│   │ timeRange: TimeRangeState │
└──────────────────────────┘   │ refreshInterval: number   │
                               └──────────────────────────┘
```

### 3.2 State Lifecycle

All data stores follow the 4-state model:

```
FETCH ──► loading ──► normal       (happy path: data returned)
  │           │
  │           ├──► empty            (no data returned)
  │           │      │
  │           │      └──► normal    (SSE delivers first data)
  │           │
  │           ├──► error            (fetch fails)
  │           │      │
  │           │      └──► loading   (retry)
  │           │
  │           └──► normal ──► error (polling/refresh failure)
  │
  └── error ──► loading             (retry from error state)
```

| State | Trigger | UI Rendering |
|-------|---------|-------------|
| `loading` | Initial fetch in progress | `<LoadingState>` skeleton/spinner |
| `empty` | Fetch returned zero results | `<EmptyState>` with descriptive message |
| `error` | Fetch failed (network, 4xx, 5xx) | `<ErrorState>` with retry button |
| `normal` | Data available | Actual component content |

### 3.3 SSE Event Types

The frontend subscribes to C09 SSE endpoint `/api/events` [UXE-03]. Event types (aligned with C09 §4.5 / API-1.7):

| Event | Payload | Updates Store |
|-------|---------|---------------|
| `device:new` | `{"mac":"aa:bb:cc:dd:ee:ff","first_seen":"ISO8601","rssi":-52}` | `devicesStore.addNewDevice(data)` |
| `packet:batch` | `{"sniffer_id":1,"count":127,"window_start":"ISO8601","window_end":"ISO8601"}` | `packetsStore.addBatch(data)`, `devicesStore.incrementCounters(data)` |
| `health:change` | `{"component":"sniffer","sniffer_id":1,"status":"online"\|"offline","ts":"ISO8601"}` | `healthStore.updateComponentStatus(data)` |
| `gap:detected` | `{"sniffer_id":1,"gap_start":"ISO8601","gap_end":"ISO8601","bucket_count":3}` | `healthStore.addGap(data)` |

**Event naming convention:** Events use the `domain:action` pattern (colon-separated). This is consistent with the W3C SSE recommendation [12] for namespacing events without conflict. The C09 API defines and emits these events; the C10 frontend consumes them. The SSE connection also receives periodic keep-alive comments (`: keepalive\n\n`) from C09 to prevent proxy timeout.

---

## 4. Low-Level Architecture (LLA)

### 4.1 Module Decomposition

#### 4.1.1 App Shell (`AppLayout.vue`)

The root layout is a CSS Grid shell:

```
┌────────────────────────────────────────────────────┐
│ NavBar (fixed top, h-14)                           │
├──────────┬─────────────────────────────────────────┤
│ Side     │ <router-view>                           │
│ Status   │                                         │
│ Panel    │  ┌──────────────────────────────────┐   │
│ (collaps-│  │ PageHeader                        │   │
│ ible,    │  │ (view title + description)         │   │
│  w-64    │  ├──────────────────────────────────┤   │
│  expand- │  │                                   │   │
│  ed)     │  │ View Content                      │   │
│          │  │ (StatCards / DataTable / Charts)   │   │
│          │  │                                   │   │
│          │  └──────────────────────────────────┘   │
└──────────┴─────────────────────────────────────────┘
```

**Grid template:** `grid-cols-[auto_1fr]` with a CSS transition on the sidebar column. NavBar is `position: sticky; top: 0; z-index: 50`. SideStatusPanel is `position: sticky; top: 3.5rem; height: calc(100vh - 3.5rem); overflow-y: auto`.

**Accessibility — skip-navigation link:**

AppLayout must include a skip-navigation link as the first focusable element, visible on keyboard focus:

```html
<a href="#main-content" class="sr-only focus:not-sr-only focus:absolute focus:top-2 focus:left-2 focus:z-[100] focus:px-4 focus:py-2 focus:bg-bg-card focus:text-accent focus:rounded focus:ring-2 focus:ring-accent">
  Skip to main content
</a>
```

The `<router-view>` wrapper must carry `id="main-content"`:

```html
<main id="main-content">
  <router-view />
</main>
```

**Accessibility — ARIA live regions for SSE-driven updates:**

SSE events drive dynamic content updates throughout the app. Screen reader users must be informed of these changes without disruption. The following ARIA live regions are required:

| Region | Attribute | Location | Purpose |
|--------|-----------|----------|---------|
| General announcements | `aria-live="polite"` | AppLayout (invisible container adjacent to `<router-view>`) | Announces new devices detected, significant packet count milestones, gap detection alerts |
| Sniffer status badges | `role="status"` | SideStatusPanel badge elements | Announces sniffer online/offline transitions as they occur via `health:change` SSE events |
| StatCard values | `aria-atomic="true"` | DashboardView StatCard components | Screen readers re-read the entire StatCard (label + value) when values change, ensuring value-only updates are not missed |

The general announcement region uses `aria-live="polite"` so announcements queue behind current screen reader speech. An invisible `<div>` with `aria-live="polite"` is maintained by the `useSSE` composable, which pushes human-readable messages (e.g., "New device detected: AA:BB:CC:DD:EE:FF") as `textContent` updates. The composable deduplicates rapid-fire updates by batching announcements within a 500ms debounce window.

**Accessibility — route-change focus management:** After route navigation completes, keyboard focus must be programmatically moved to the `<h1>` heading of the new view. This ensures screen reader and keyboard users are not stranded at the navigation element after a route change (WCAG 2.2 SC 2.4.3 Focus Order [2]). Implementation uses Vue Router's `afterEach` guard:

```typescript
// src/router/index.ts
router.afterEach(() => {
  // Focus the page heading after route transition completes
  // nextTick ensures the new view has rendered
  import('vue').then(({ nextTick }) => {
    nextTick(() => {
      document.getElementById('page-heading')?.focus();
    });
  });
});
```

Each view's `<h1>` must carry `id="page-heading"` and `tabindex="-1"` (to allow programmatic focus without adding it to the natural tab order). Example from `PageHeader.vue`:

```html
<h1 id="page-heading" tabindex="-1">{{ title }}</h1>
```

#### 4.1.2 Router Configuration

```typescript
// src/router/index.ts
const routes = [
  { path: '/',            name: 'dashboard',       component: () => import('@/views/DashboardView.vue') },
  { path: '/devices',     name: 'device-map',      component: () => import('@/views/DeviceMapView.vue') },
  { path: '/packets',     name: 'packet-explorer', component: () => import('@/views/PacketExplorerView.vue') },
  { path: '/settings',    name: 'settings',        component: () => import('@/views/SettingsView.vue') },
];
```

All routes use lazy loading (dynamic imports) for code splitting. Dashboard is the landing page (`/`).

#### 4.1.3 Pinia Store Design

Each store follows this pattern:

```typescript
// Example: src/stores/devices.ts
import { defineStore } from 'pinia';
import { ref, computed } from 'vue';
import type { DeviceSummary } from '@/types/api';
import { apiClient } from '@/api/client';

export const useDevicesStore = defineStore('devices', () => {
  // State
  const devices = ref<DeviceSummary[]>([]);
  const loading = ref(false);
  const error = ref<string | null>(null);
  const lastFetched = ref<Date | null>(null);

  // Getters
  const isEmpty = computed(() => !loading.value && devices.value.length === 0);
  const onlineCount = computed(() => devices.value.filter(d => d.classification === 'resident').length);

  // Actions
  async function fetchDevices() {
    loading.value = true;
    error.value = null;
    try {
      const { data } = await apiClient.GET('/api/devices');
      devices.value = data ?? [];
      lastFetched.value = new Date();
    } catch (e) {
      error.value = e instanceof Error ? e.message : 'Unknown error';
    } finally {
      loading.value = false;
    }
  }

  return { devices, loading, error, lastFetched, isEmpty, onlineCount, fetchDevices };
});
```

#### 4.1.4 SSE Composable

```typescript
// src/composables/useSSE.ts
import { ref, onMounted, onUnmounted } from 'vue';

/** Exponential backoff reconnect configuration. */
const RECONNECT_DELAYS = [1000, 2000, 4000, 8000, 16000, 30000]; // ms
const STALE_THRESHOLD_MS = 60_000; // 60s without events = stale

export interface SSEHandlers {
  onDeviceNew?: (data: { mac: string; first_seen: string; rssi: number }) => void;
  onPacketBatch?: (data: { sniffer_id: number; count: number; window_start: string; window_end: string }) => void;
  onHealthChange?: (data: { component: string; sniffer_id: number; status: 'online' | 'offline'; ts: string }) => void;
  onGapDetected?: (data: { sniffer_id: number; gap_start: string; gap_end: string; bucket_count: number }) => void;
}

export function useSSE(url: string, handlers: SSEHandlers) {
  const connected = ref(false);
  const stale = ref(false);
  const lastEventAt = ref<Date | null>(null);
  let eventSource: EventSource | null = null;
  let reconnectAttempt = 0;
  let reconnectTimer: ReturnType<typeof setTimeout> | null = null;
  let staleTimer: ReturnType<typeof setInterval> | null = null;

  function connect() {
    // Use withCredentials: true so the browser sends the tianer_auth cookie
    // set by C09 on first valid API key validation.
    eventSource = new EventSource(url, { withCredentials: true });

    eventSource.onopen = () => {
      connected.value = true;
      stale.value = false;
      reconnectAttempt = 0;
    };

    eventSource.addEventListener('device:new', (e: MessageEvent) => {
      lastEventAt.value = new Date();
      handlers.onDeviceNew?.(JSON.parse(e.data));
    });

    eventSource.addEventListener('packet:batch', (e: MessageEvent) => {
      lastEventAt.value = new Date();
      handlers.onPacketBatch?.(JSON.parse(e.data));
    });

    eventSource.addEventListener('health:change', (e: MessageEvent) => {
      lastEventAt.value = new Date();
      handlers.onHealthChange?.(JSON.parse(e.data));
    });

    eventSource.addEventListener('gap:detected', (e: MessageEvent) => {
      lastEventAt.value = new Date();
      handlers.onGapDetected?.(JSON.parse(e.data));
    });

    eventSource.onerror = () => {
      connected.value = false;
      eventSource?.close();
      scheduleReconnect();
    };

    // Stale detection: if no events for >60s, mark as stale
    staleTimer = setInterval(() => {
      if (lastEventAt.value && (Date.now() - lastEventAt.value.getTime() > STALE_THRESHOLD_MS)) {
        stale.value = true;
      }
    }, 10_000);
  }

  function scheduleReconnect() {
    const delay = RECONNECT_DELAYS[Math.min(reconnectAttempt, RECONNECT_DELAYS.length - 1)];
    reconnectAttempt++;
    reconnectTimer = setTimeout(() => connect(), delay);
  }

  function disconnect() {
    if (reconnectTimer) clearTimeout(reconnectTimer);
    if (staleTimer) clearInterval(staleTimer);
    eventSource?.close();
    eventSource = null;
    connected.value = false;
    stale.value = false;
    reconnectAttempt = 0;
  }

  onMounted(() => connect());
  onUnmounted(() => disconnect());

  return { connected, stale, lastEventAt, connect, disconnect };
}
```

#### 4.1.5 Core Components

| Component | Props | Emits | 4-State | Notes |
|-----------|-------|-------|---------|-------|
| `StatCard` | `icon`, `label`, `value`, `delta?`, `trend?` | — | Shows value or `--` if undefined | Amber accent for value, monospace font |
| `DataTable` | `columns[]`, `rows[]`, `loading`, `error`, `emptyMessage` | `sort`, `filter`, `row-click` | All 4 states handled internally | TanStack Table v8 wrapper |
| `Badge` | `variant: 'success'\|'warning'\|'danger'\|'neutral'`, `label` | — | — | Green/grey/amber variants |
| `NavBar` | `currentRoute` | `navigate` | — | Tab-style navigation |
| `SideStatusPanel` | `sniffers[]`, `collapsed` | `toggle` | `loading`/`empty`/`error`/`normal` | Expandable, green/grey indicators. Data source: GET /api/sniffers. Polls at 15s interval (or receives SSE `health:change` events). `empty` state: "No sniffers configured. Add a sniffer in Settings to begin capturing." |
| `ChartCard` | `title`, `option`, `loading`, `error` | — | `loading`/`error`/`normal`/`empty` | ECharts wrapper with dark theme |
| `PageHeader` | `title`, `description`, `actions?` | — | — | Page-level heading |
| `TimeRangeSelector` | `modelValue` | `update:modelValue` | — | Preset buttons + custom range |

#### 4.1.6 View Compositions

**DashboardView:**
```
PageHeader("Dashboard")
├── StatCard row (4 cards: devices seen, packets today, sniffers online, gaps open)
│     └── Data source: GET /api/stats → DashboardStats
│     │   total_devices → "Devices Seen" card
│     │   packets_today → "Packets Today" card
│     │   sniffers_online → "Sniffers Online" card (with sniffers_total as context)
│     │   gaps_open → "Open Gaps" card
│     │   packet_rate_current → displayed as subtitle/trend on packets card
│     │   disk_usage_pct → displayed in SideStatusPanel (via health store)
│     │   uptime_seconds → displayed in SideStatusPanel (via health store)
├── ChartCard (packets over time, line chart)
│     └── Data source: GET /api/devices/{mac}/timeline aggregated across all devices
│         (or a future /api/stats/timeseries endpoint; for MVP, derived from
│         device timeline queries or the Grafana pipeline-health dashboard)
├── ChartCard (top devices by observations, bar chart)
│     └── Data source: GET /api/devices?sort=-total_count&limit=10
└── GrafanaEmbed (pipeline-health dashboard, iframe)
```

**DeviceMapView:**
```
PageHeader("Device Map")
├── TimeRangeSelector
├── Search bar + filter chips (address type, classification, manufacturer)
└── DataTable (columns: address, last seen, count, RSSI, manufacturer, class)
    └── Expandable row: per-device detail (first seen, timeline, raw packets link)
```

**PacketExplorerView:**
```
PageHeader("Packet Explorer")
├── TimeRangeSelector
├── Filter bar (sniffer, PDU type, channel, address)
│     └── Filter values populated from GET /api/sniffers (for sniffer dropdown)
└── DataTable (columns: time, sniffer, address, type, RSSI, channel, raw hex)
      └── Data source: GET /api/packets?sniffer_id=&mac=&pdu_type=&channel=&since=&limit=&offset=
      │   Paginated via limit/offset. Sort: -ts (newest first).
      │   Total row count from response.total for pagination controls.
      └── Expandable row: full packet detail with raw hex dump
```

**SettingsView:**
```
PageHeader("Settings")
├── Section: Display (refresh interval, Grafana base URL)
├── Section: API (connection status, test button)
└── Section: About (version, build date)
```

### 4.2 API Client Architecture

The API client is generated from C09's OpenAPI spec using `openapi-typescript` to produce type definitions and `openapi-fetch` for the runtime client [9]:

```typescript
// Generated: src/types/api.ts
// Contains all request/response types from C09 OpenAPI spec

// src/api/client.ts
import createClient from 'openapi-fetch';
import type { paths } from '@/types/api';

export const apiClient = createClient<paths>({
  baseUrl: import.meta.env.VITE_API_BASE_URL || '/api',
  headers: {
    'Content-Type': 'application/json',
    // API key injected at build time or via env; never in client source
  },
});
```

**API key handling:** The API key is never stored in client-side JavaScript source. Two authentication mechanisms are used:

1. **REST requests (openapi-fetch):** The API key is injected by C09 into `index.html` as a `<meta name="tianer-api-key" content="...">` tag at serve time. The `useApi` composable reads it from the DOM on mount and sets it as the `X-API-Key` header on all REST requests.

2. **SSE connection (EventSource):** The native `EventSource` API [12] cannot send custom HTTP headers (no `Authorization`, no `X-API-Key`). Therefore, SSE authentication uses a **cookie-based approach**:
   - On first valid API key validation (via REST), C09 sets a `tianer_auth` cookie with attributes `Secure; SameSite=Strict; HttpOnly; Path=/api`.
   - The SSE `EventSource` connects with `withCredentials: true`, causing the browser to automatically send the `tianer_auth` cookie.
   - The C09 SSE endpoint validates either the `X-API-Key` header OR the `tianer_auth` cookie for authentication.
   - This is the recommended MVP approach because it requires zero client-side cryptographic handling for SSE.

**Rationale for cookie over `fetch()` polyfill:** A `fetch()`-based SSE implementation using `ReadableStream` could carry the `X-API-Key` header but introduces complexity (manual line parsing, reconnection logic, browser compatibility concerns). The cookie approach leverages `EventSource`'s built-in reconnection and is well-tested across browsers. On a LAN-only deployment with TLS, the `HttpOnly` cookie is an acceptable auth mechanism for the SSE stream.

### 4.3 Dark Theme — BreachLab Design Tokens

Per PD-03-theme, the visual design follows a moderated BreachLab aesthetic [2]:

| Token | CSS Variable | Value |
|-------|-------------|-------|
| Background | `--color-bg-primary` | `#0a0a0f` |
| Background elevated | `--color-bg-secondary` | `#12121a` |
| Background card | `--color-bg-card` | `#16161f` |
| Accent primary | `--color-accent` | `#f5a00c` (amber) |
| Accent hover | `--color-accent-hover` | `#d4890a` |
| Text primary | `--color-text-primary` | `#e0e0e0` |
| Text secondary | `--color-text-secondary` | `#888899` |
| Text muted | `--color-text-muted` | `#7a7a8c` |
| Status online | `--color-status-online` | `#22c55e` (green-500) |
| Status offline | `--color-status-offline` | `#6e6e80` |
| Border | `--color-border` | `#222233` |
| Font mono | `--font-mono` | `'JetBrains Mono', 'Fira Code', monospace` |
| Font sans | `--font-sans` | `'Inter', system-ui, sans-serif` |
| Focus ring | `--color-focus-ring` | `#f5a00c` (amber, 2px, for WCAG 2.4.7) |

**Advisory note — SC 1.4.1 Use of Color:** The status-offline indicator uses a distinct shape (dashed border) alongside colour to ensure accessibility for colour-blind operators. This satisfies WCAG 2.2 SC 1.4.1 (Use of Color) by conveying information through more than one sensory channel [2].

### 4.4 4-State Rendering Pattern

Every view implements the 4-state model using a composable:

```typescript
// src/composables/useAsyncState.ts
import { ref, computed } from 'vue';

export function useAsyncState<T>() {
  const data = ref<T | null>(null);
  const loading = ref(false);
  const error = ref<string | null>(null);

  const state = computed(() => {
    if (loading.value) return 'loading';
    if (error.value) return 'error';
    if (!data.value || (Array.isArray(data.value) && data.value.length === 0)) return 'empty';
    return 'normal';
  });

  return { data, loading, error, state };
}
```

Each view renders conditionally based on `state`:

```vue
<template>
  <LoadingState v-if="state === 'loading'" />
  <ErrorState v-else-if="state === 'error'" :message="error" @retry="fetchData" />
  <EmptyState v-else-if="state === 'empty'" message="No devices detected yet." />
  <!-- Normal content -->
  <div v-else>...</div>
</template>
```

---

## 5. Inter-Component Contracts

### 5.1 Contract FRONT-1: Frontend ↔ C09 REST API

| Field | Value |
|-------|-------|
| **Contract ID** | FRONT-1 (extends API-1) |
| **Protocol** | HTTPS REST (JSON) |
| **Base URL** | `VITE_API_BASE_URL` (default `/api`) |
| **Auth** | `X-API-Key` header (injected by C09 at serve time) |
| **Type safety** | Generated from C09 OpenAPI spec via `openapi-typescript` |

**Endpoints consumed by frontend:**

| Method | Path | Purpose | Polling/SSE | Notes |
|--------|------|---------|-------------|-------|
| GET | `/api/devices` | Device summary list | Poll 30s | Paginated; consumed by DeviceMapView + DashboardView top-devices chart |
| GET | `/api/devices/{address}` | Single device detail | On demand | Expandable row drill-down in DeviceMapView |
| GET | `/api/devices/{address}/timeline` | Device timeline data | On demand | Consumed by device detail expandable row |
| GET | `/api/packets` | Raw packet list (paginated) | Poll 15s | Consumed by PacketExplorerView DataTable. Filters: sniffer_id, mac, pdu_type, channel, since |
| GET | `/api/sniffers` | Sniffer health status | Poll 15s (initial load) + SSE updates | Consumed by SideStatusPanel. SSE `health:change` events provide live updates between polls |
| GET | `/api/stats` | Aggregate dashboard statistics | Poll 60s | Consumed by DashboardView StatCards. C09 caches response for 30s (`Cache-Control: max-age=30`). Frontend respects this with `If-None-Match` / 304 handling |
| GET | `/api/health` | API health check | Poll 30s | Used for connection status indicator; also consumed by SideStatusPanel for degraded/error state |
| GET | `/api/events` | SSE event stream | Persistent SSE | Live `health:change`, `device:new`, `packet:batch` events update stores without polling |

### 5.2 Contract FRONT-2: Frontend ↔ Grafana (iframe)

| Field | Value |
|-------|-------|
| **Protocol** | HTTPS iframe embed |
| **Auth** | Anonymous access [UXE-05] |
| **CSP** | `frame-src https://grafana.tianer.local:3000` (CSP Level 2 `frame-ancestors` on Grafana side) [8] |
| **Sandbox** | `sandbox="allow-scripts allow-same-origin"` (not `allow-top-navigation`) |
| **Dashboards** | `pipeline-health` |

### 5.3 Contract FRONT-3: Vite Dev Server ↔ C09 API (Proxy)

| Field | Value |
|-------|-------|
| **Dev proxy** | `/api` → `http://localhost:8080` |
| **Dev proxy** | `/events` → `http://localhost:8080` |
| **HMR** | `ws://localhost:5173` (Vite default) |

---

## 6. Failure Modes & Recovery

### 6.1 Failure Catalog

| # | Failure Mode | Detection | UX Impact | Recovery | SLA |
|---|-------------|-----------|-----------|----------|-----|
| FM-1 | **API unreachable** (C09 down, network) | HTTP fetch timeout (>5s), or 502/503/504 | `<ErrorState>` with retry; SideStatusPanel shows all sniffers grey | Exponential backoff retry (1s, 2s, 4s, 8s, max 30s); user can manually retry | UI recovers within 5s of API recovery |
| FM-2 | **API returns 401/403** | HTTP status 401/403 | `<ErrorState>` with "Authentication failed" message | No auto-recovery; operator must check API key in Settings | Manual |
| FM-3 | **API returns 5xx** | HTTP status 500-599 | `<ErrorState>` with generic message | Exponential backoff retry (same as FM-1) | UI recovers within 5s of API recovery |
| FM-4 | **SSE connection lost** | `EventSource.onerror` fires | `connected` ref = false; SideStatusPanel shows "Disconnected" badge; stale data warning after 60s | Auto-reconnect with exponential backoff; lastEventId sent for catch-up | Reconnect within 10s |
| FM-5 | **Grafana iframe fails to load** | iframe `onerror`, 4xx/5xx status | iframe shows "Grafana unavailable" placeholder | No auto-recovery; operator must verify Grafana is running | Manual |
| FM-6 | **Stale data** (SSE connected but no events) | Timer: last event age > 2x poll interval | Data rows show "stale" indicator (muted opacity); last-updated timestamp visible | Re-fetch via REST as fallback | Detected within 2x poll interval |
| FM-7 | **Browser tab backgrounded** | `document.visibilityState` change | SSE paused; REST polling paused | Resume on foreground; catch-up fetch on return | Data current within 5s of foregrounding |
| FM-8 | **Bundle load failure** (chunk 404) | Vite dynamic import rejection | Global error boundary catches; shows "Application failed to load" full-screen error | User refreshes page | Manual |
| FM-9 | **Memory pressure** (>300MB with 2 Grafana iframes) | `performance.memory` API (Chrome) | Degraded rendering (throttle chart animations, reduce table rows) | GC eventually; if persistent, operator closes a Grafana iframe | Mitigated below 300MB threshold |

### 6.2 Detection Mechanisms

**Primary:** HTTP response status codes, `EventSource.onerror`, `iframe.onerror`  
**Secondary (redundant):** Stale data timers comparing `lastEvent.timestamp` vs `Date.now()`, periodic `/api/health` check from a `setInterval` independent of data fetching

### 6.3 Graceful Degradation

| Scenario | Degradation |
|----------|------------|
| API down | All views show ErrorState; Grafana iframes still work (read DB directly) |
| SSE down | Switch to REST polling fallback at 15s intervals; SideStatusPanel shows "polling" indicator |
| Grafana down | Single iframe shows placeholder; rest of app works |
| High memory usage | Reduce table page size to 25 rows; throttle chart re-renders to 5s intervals |

---

## 7. Observability

### 7.1 Frontend Metrics (Reported to C13 via C09)

| Metric | Type | Description |
|--------|------|-------------|
| `tianer_frontend_page_load_ms` | Histogram | Time from navigation to interactive (TTI) |
| `tianer_frontend_api_error_total` | Counter | Count of API call failures by endpoint + status |
| `tianer_frontend_sse_disconnect_total` | Counter | SSE disconnection events |
| `tianer_frontend_view_render_total` | Counter | View renders by route name |
| `tianer_frontend_js_error_total` | Counter | Unhandled JS exceptions caught by error boundary |

### 7.2 Error Boundary

A global `onErrorCaptured` hook on `App.vue` catches all unhandled errors and:
1. Logs to console with `console.error`
2. Increments `tianer_frontend_js_error_total` (via `navigator.sendBeacon` to `/api/metrics/frontend`)
3. Renders a full-screen error state if the error is fatal (e.g., router failure)

### 7.3 Structured Logging

Client-side errors are reported as structured JSON to `/api/metrics/frontend`:

```json
{
  "timestamp": "2026-06-09T12:00:00Z",
  "level": "error",
  "message": "API fetch failed",
  "context": {
    "url": "/api/devices",
    "status": 502,
    "view": "DeviceMapView",
    "user_agent": "Mozilla/5.0 ..."
  }
}
```

---

## 8. Security Considerations

### 8.1 Attack Surface

| Vector | Risk | Mitigation |
|--------|------|-----------|
| **API key in client code** | HIGH — key exposure in bundled JS | API key injected by C09 server into `index.html` at serve time via `<meta>` tag or server-side template; never in source files |
| **XSS via device data** | MEDIUM — malicious BLE advertisement name could inject script | Vue's template interpolation auto-escapes HTML; `v-html` is banned project-wide via ESLint rule `vue/no-v-html` |
| **Clickjacking of Grafana iframe** | LOW — Grafana is LAN-only | `frame-ancestors 'self'` header on Grafana; CSP `frame-src` allowlist on frontend [8] |
| **SSE connection hijacking** | LOW — LAN-only deployment | SSE endpoint on same origin; no CORS concerns for SPA |
| **Dependency supply chain** | MEDIUM — npm packages | `npm audit` in CI; lockfile committed; `--omit=dev` for production install |
| **Open redirect via settings** | LOW — Grafana URL is operator-configurable | Input validation: only allow `https://` URLs; block `javascript:` and `data:` schemes |
| **Information disclosure via error messages** | LOW — stack traces in ErrorState | Production build strips source maps; error messages are generic |
| **CSRF** | LOW — read-only SPA, no state-changing operations from UI in MVP | `SameSite=Lax` on session cookie; future: CSRF token for settings save |

### 8.2 Content Security Policy

The frontend is served with the following CSP header (set by C09 API):

```
Content-Security-Policy:
  default-src 'self';
  script-src 'self';
  style-src 'self' 'unsafe-inline';
  img-src 'self' data:;
  connect-src 'self' https://tianer.local:8443;
  frame-src https://grafana.tianer.local:3000;
  font-src 'self';
  object-src 'none';
  base-uri 'self';
  form-action 'none';
```

This policy aligns with CSP Level 2 directives [8]. `frame-src` is explicitly allowlisted for the Grafana host.

### 8.3 Input Validation

All user inputs are validated at the component level:

| Input | Validation |
|-------|-----------|
| Search queries | Max 100 chars, alphanumeric + colon (MAC format) |
| Time range custom dates | ISO 8601 format, `start < end`, `end <= now` |
| Settings URLs | `URL` constructor parsing, must be `https:` scheme |
| Numeric inputs (refresh interval) | Integer 5–300 seconds |

---

## 9. Configuration

### 9.1 Build-Time Environment Variables (Vite)

All prefixed with `VITE_` to be exposed to client bundle per Vite convention [3]:

| Variable | Default | Description |
|----------|---------|-------------|
| `VITE_API_BASE_URL` | `/api` | Base URL for REST API calls (absolute path in prod, proxied in dev) |
| `VITE_GRAFANA_BASE_URL` | `https://grafana.tianer.local:3000` | Base URL for Grafana iframe embeds |
| `VITE_APP_NAME` | `Tian'er` | Application display name |
| `VITE_POLL_INTERVAL_DEVICES` | `30000` | Device list polling interval (ms) |
| `VITE_POLL_INTERVAL_PACKETS` | `15000` | Packet list polling interval (ms) |
| `VITE_POLL_INTERVAL_STATS` | `60000` | Statistics polling interval (ms) |
| `VITE_SSE_URL` | `/api/events` | SSE endpoint URL |

### 9.2 Vite Configuration

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import vue from '@vitejs/plugin-vue';
import tailwindcss from '@tailwindcss/vite';
import tsconfigPaths from 'vite-tsconfig-paths';

export default defineConfig({
  plugins: [
    tailwindcss(),
    vue(),
    tsconfigPaths(),       // Path aliases from tsconfig.json
  ],
  resolve: {
    alias: {
      '@': '/src',          // Import alias
    },
  },
  server: {
    port: 5173,
    proxy: {
      '/api': {
        target: 'http://localhost:8080',
        changeOrigin: true,
      },
      '/events': {
        target: 'http://localhost:8080',
        ws: true,            // SSE needs WebSocket-capable proxy
      },
    },
  },
  build: {
    target: 'esnext',       // Modern browser baseline
    outDir: 'dist',
    sourcemap: false,        // No source maps in production
    rollupOptions: {
      output: {
        manualChunks: {
          'vendor-vue': ['vue', 'vue-router', 'pinia'],
          'vendor-echarts': ['echarts'],
          'vendor-table': ['@tanstack/vue-table'],
        },
      },
    },
  },
});
```

### 9.3 Tailwind CSS 4 Configuration

Tailwind CSS 4 uses CSS-first configuration [4]. Custom theme values are defined in `src/assets/style.css`:

```css
@import "tailwindcss";

@theme {
  --color-bg-primary: #0a0a0f;
  --color-bg-secondary: #12121a;
  --color-bg-card: #16161f;
  --color-accent: #f5a00c;
  --color-accent-hover: #d4890a;
  --color-text-primary: #e0e0e0;
  --color-text-secondary: #888899;
  --color-text-muted: #7a7a8c;
  --color-status-online: #22c55e;
  --color-status-offline: #6e6e80;
  --color-border: #222233;
  --font-mono: 'JetBrains Mono', 'Fira Code', monospace;
  --font-sans: 'Inter', system-ui, sans-serif;
}
```

### 9.4 TypeScript Configuration

```json
{
  "compilerOptions": {
    "target": "ESNext",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "jsx": "preserve",
    "noEmit": true,
    "isolatedModules": true,
    "skipLibCheck": true,
    "paths": {
      "@/*": ["./src/*"]
    },
    "types": ["vitest/globals"]
  },
  "include": ["src/**/*.ts", "src/**/*.vue", "tests/**/*.ts"]
}
```

### 9.5 Runtime Settings (localStorage)

User-configurable settings persisted to `localStorage`:

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `tianer:refreshInterval` | number | 30 | Data refresh interval (seconds) |
| `tianer:grafanaUrl` | string | `VITE_GRAFANA_BASE_URL` | Grafana base URL override |
| `tianer:sidebarCollapsed` | boolean | false | Sidebar collapse state |
| `tianer:tablePageSize` | number | 50 | Default table rows per page |
| `tianer:timeRangePreset` | string | `'24h'` | Default time range preset |

---

## 10. Test Plan

### 10.1 Testing Stack

| Layer | Tool | Purpose |
|-------|------|---------|
| Unit tests | Vitest 1.2+ [10] | Composables, utility functions, store logic |
| Component tests | Vitest + Vue Test Utils | Component rendering, props, emits, 4-state rendering |
| Store tests | Vitest + Pinia | Store actions, getters, state transitions |
| API mocking | MSW (Mock Service Worker) | Intercept fetch/SSE at the network level |
| Coverage | v8 provider (Vitest built-in) | 80% line coverage target for stores + composables |

### 10.2 Test Coverage Matrix

| Module | Target | Tests |
|--------|--------|-------|
| Stores (`stores/*.ts`) | 80% lines | Action success/error paths, state transitions, SSE event handling |
| Composables (`composables/*.ts`) | 80% lines | useSSE connection lifecycle, useTimeRange preset logic, useApi error handling |
| Components (`components/common/*.vue`) | Visibility + states | Render each of 4 states, prop variations, emit verification |
| Views (`views/*.vue`) | Smoke | Mount with MSW mock data, verify key elements present |

### 10.3 Example Test: Device Store

```typescript
// tests/stores/devices.test.ts
import { describe, it, expect, beforeEach } from 'vitest';
import { setActivePinia, createPinia } from 'pinia';
import { useDevicesStore } from '@/stores/devices';
import { server } from '../mocks/server';

describe('devices store', () => {
  beforeEach(() => {
    setActivePinia(createPinia());
  });

  it('starts in loading state', () => {
    const store = useDevicesStore();
    store.loading = true; // simulate fetch start
    expect(store.loading).toBe(true);
  });

  it('fetches devices and transitions to normal', async () => {
    const store = useDevicesStore();
    // MSW intercepts GET /api/devices
    await store.fetchDevices();
    expect(store.loading).toBe(false);
    expect(store.error).toBeNull();
    expect(store.devices.length).toBeGreaterThan(0);
  });

  it('transitions to empty when no devices returned', async () => {
    // MSW returns empty array
    const store = useDevicesStore();
    await store.fetchDevices();
    expect(store.isEmpty).toBe(true);
  });

  it('transitions to error on fetch failure', async () => {
    // MSW returns 500
    const store = useDevicesStore();
    await store.fetchDevices();
    expect(store.error).not.toBeNull();
  });
});
```

### 10.4 MSW Handler Example

```typescript
// tests/mocks/handlers.ts
import { http, HttpResponse } from 'msw';

export const handlers = [
  http.get('/api/devices', () => {
    return HttpResponse.json([
      {
        device_address: 'AA:BB:CC:DD:EE:FF',
        address_type: 'public',
        last_seen: '2026-06-09T12:00:00Z',
        observation_count: 1500,
        first_seen: '2026-06-08T00:00:00Z',
        avg_rssi: -65,
        classification: 'resident',
      },
    ]);
  }),

  http.get('/api/health', () => {
    return HttpResponse.json({ status: 'ok' });
  }),
];
```

### 10.5 Bundle Budget Verification

Per UI-05 [3]:

| Metric | Budget | Verification |
|--------|--------|-------------|
| JS total (gzipped) | <500 KB | `vite build && gzip -c dist/assets/*.js \| wc -c` |
| CSS total (gzipped) | <100 KB | `vite build && gzip -c dist/assets/*.css \| wc -c` |
| Page load (10 Mbps) | <3 seconds | Lighthouse audit (desktop, throttled) |
| Memory (with 2 Grafana iframes) | <300 MB | Chrome DevTools Memory panel, measure after 5 min idle |

### 10.6 Accessibility Testing

**WCAG 2.2 AA-lite compliance checks [2]:**

| # | Criterion | Check Method |
|---|-----------|-------------|
| 1 | Color contrast ≥4.5:1 for normal text | axe DevTools automated scan |
| 2 | All interactive elements keyboard-accessible (Tab/Enter/Escape) | Manual keyboard audit |
| 3 | ARIA landmarks present (banner, navigation, main, complementary) | axe DevTools |
| 4 | Focus visible on all interactive elements (amber outline, 2px) | Manual Tab-through |
| 5 | `alt` text on all non-decorative icons | axe DevTools |
| 6 | Form inputs have associated `<label>` elements | axe DevTools |
| 7 | Page `<title>` updates per route | Manual navigation test |
| 8 | `lang="en"` on `<html>` element | HTML validation |
| 9 | No content flashes >3 times/second (WCAG 2.3.1) | Manual review (no animations in MVP) |
| 10 | `prefers-reduced-motion` respected (disable chart animations) | System setting toggle |

---

## 11. Deployment Notes

### 11.1 Build Command

```bash
# Development
cd platform/frontend
npm install
npm run dev          # Vite dev server at :5173 with HMR + API proxy

# Production build
npm run build        # vite build → dist/

# Test
npm run test         # vitest run
npm run test:coverage # vitest run --coverage
npm run lint         # eslint src/
npm run typecheck    # vue-tsc --noEmit
```

### 11.2 Production Deployment

The frontend is served as static files from the C09 REST API container. The build output (`dist/`) is copied into the C09 container image at build time:

```dockerfile
# In C09 container Dockerfile:
COPY --from=frontend-builder /app/platform/frontend/dist /app/static
```

C09's FastAPI serves static files from `/app/static` at the root path. Vue Router is in `createWebHistory` mode, so all non-API routes must fall back to `index.html` — C09 must implement a catch-all route for SPA client-side routing.

### 11.3 Container Integration

| Aspect | Detail |
|--------|--------|
| Container | None — frontend runs in the user's browser, not in a container |
| Static assets | Served from C09 API container (tianer-platform pod) |
| Volume mounts | None |
| Port exposure | Via C09 API reverse proxy (port 8443 with TLS) |

### 11.4 Performance Budget Enforcement

Add to `package.json`:

```json
{
  "scripts": {
    "build": "vite build",
    "size-check": "vite build && node scripts/check-bundle-size.js"
  }
}
```

The size check script verifies JS <500KB gzipped and CSS <100KB gzipped, failing CI if exceeded.

### 11.5 Dependencies

```json
{
  "dependencies": {
    "vue": "^3.4.0",
    "vue-router": "^4.3.0",
    "pinia": "^2.1.0",
    "echarts": "^5.5.0",
    "@tanstack/vue-table": "^8.17.0",
    "date-fns": "^3.6.0",
    "lucide-vue-next": "^0.400.0",
    "openapi-fetch": "^0.9.0"
  },
  "devDependencies": {
    "@vitejs/plugin-vue": "^5.0.0",
    "@tailwindcss/vite": "^4.0.0",
    "tailwindcss": "^4.0.0",
    "typescript": "^5.3.0",
    "vite": "^5.4.0",
    "vitest": "^1.2.0",
    "@vue/test-utils": "^2.4.0",
    "msw": "^2.2.0",
    "openapi-typescript": "^7.0.0",
    "eslint": "^8.57.0",
    "eslint-plugin-vue": "^9.0.0",
    "vite-tsconfig-paths": "^4.3.0",
    "vue-tsc": "^2.0.0"
  }
}
```

---

## References

[1] Vue.js. "Vue 3 Documentation — Composition API, Single-File Components, `<script setup>`." https://vuejs.org/guide/introduction.html, accessed 2026-06-09.

[2] W3C. "Web Content Accessibility Guidelines (WCAG) 2.2 — W3C Recommendation 12 December 2024." https://www.w3.org/TR/WCAG22/, 2024.

[3] VoidZero Inc. "Vite 5 Documentation — Build Config, Env Variables, Proxy." https://vitejs.dev/guide/, accessed 2026-06-09.

[4] Tailwind Labs Inc. "Tailwind CSS 4 Documentation — CSS-first Configuration, Dark Mode, Custom Theme." https://tailwindcss.com/docs/installation, accessed 2026-06-09.

[5] Apache Software Foundation. "Apache ECharts 5 Documentation — Chart Configuration, Dark Theme." https://echarts.apache.org/en/option.html, accessed 2026-06-09.

[6] TanStack. "TanStack Table v8 Documentation — Client-side Sorting, Filtering, Pagination." https://tanstack.com/table/v8, accessed 2026-06-09.

[7] date-fns. "date-fns v3 Documentation — Tree-shakeable Date Formatting." https://date-fns.org/docs/Getting-Started, accessed 2026-06-09.

[8] W3C. "Content Security Policy Level 2 — W3C Recommendation 15 December 2016." https://www.w3.org/TR/CSP2/, 2016.

[9] openapi-ts. "OpenAPI TypeScript — Generate TypeScript Types from OpenAPI Schemas." https://openapi-ts.dev/, accessed 2026-06-09.

[10] VoidZero Inc. "Vitest Documentation — Vue Test Utils Integration, MSW Mocking." https://vitest.dev/guide/, accessed 2026-06-09.

[11] IETF. "RFC 6455 — The WebSocket Protocol." https://www.rfc-editor.org/rfc/rfc6455, December 2011. (Referenced for SSE-vs-WebSocket comparison; SSE is preferred for unidirectional server→client data.)

[12] W3C. "Server-Sent Events — W3C Recommendation." https://html.spec.whatwg.org/multipage/server-sent-events.html, accessed 2026-06-09.
