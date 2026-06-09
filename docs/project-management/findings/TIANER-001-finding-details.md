# TIANER-001 — Specialist Finding Reference Table

**Date:** 2026-06-08
**Total findings:** 26 (PD-01 through UI-09)
**Phase:** A (Pre-Build)
**Reviews:** Product Designer (A1-PD), UX Engineer (A1-UXE), UI Engineer (A1-UIE)

---

## PD-01: No User Personas

| Field | Value |
|-------|-------|
| **Code** | PD-01 |
| **Raised by** | Product Designer |
| **Confidence** | 90 (BLOCKER) |
| **What's missing** | Zero user personas exist. Three rounds of engineering review never asked who the operators are, what their skills, goals, and contexts are. The spec assumes a generic "user" with no job-role differentiation. |
| **Why it matters** | Without personas, every UX decision (navigation hierarchy, information density, alert thresholds, responsive breakpoints, default views) is arbitrary. Frontend scope cannot be validated — you cannot know which features matter if you don't know who needs them. Risk: building a tool that works technically but no operator wants to use, or missing critical workflow shortcuts for the primary persona. |
| **Options** | 1. **Define 3 personas now (recommended):** Network Operator (primary — daily monitoring, triage, quick glance), Forensic Analyst (secondary — deep dive, export, time-range queries), System Administrator (tertiary — health dashboard, config, DB stats). Write one-page persona doc with job title, goals, tasks, frustration points, and daily workflow. 2. **Define only 1 persona (Network Operator) and defer others:** Reduces up-front cost. Risk: Forensic Analyst and Admin needs won't be designed for until post-MVP, causing rework. 3. **Defer all personas to Phase B:** Rely on implicit assumptions. Highest risk — rejected by UXE and PD, no basis for IA or navigation design. |
| **Type** | DESIGN — Product Designer must produce persona doc before Phase B |

---

## PD-02: No Information Architecture

| Field | Value |
|-------|-------|
| **Code** | PD-02 |
| **Raised by** | Product Designer |
| **Confidence** | 85 (BLOCKER) |
| **What's missing** | The design brief mentions 4 views (Live Monitor, Device Map, Packet Explorer, Settings) plus 4 Grafana dashboards, but there is no hierarchy, no primary workflow, no landing page definition, and no navigation structure. Information architecture is the skeleton the frontend hangs on — it determines component tree, route design, and state architecture. |
| **Why it matters** | Without IA, the component hierarchy (UI-03), state architecture (UXE-01), and navigation flows (UXE-08) cannot be designed. Every UX and UI finding below this one is downstream of this gap. Risk: flat-route anti-pattern where every view is a peer, no clear "home" page, context switching between views loses operator state, Grafana dashboards feel like a separate product. |
| **Options** | 1. **Landing page as Live Monitor with nested dashboard drill-down (recommended):** Operator lands on Live Monitor (primary workflow). Device Map and Packet Explorer are secondary navigation items reached from Live Monitor via contextual drill-down. Settings is a tertiary gear-icon route. Grafana dashboards are deep-linked from Live Monitor cards. Write a one-page IA diagram showing page hierarchy, primary and secondary flows, and cross-linking patterns. 2. **Tab-based flat navigation:** Four equal tabs, each independent. Simplest to build, worst UX — no workflow continuity, Grafana feels bolted on. 3. **Sidebar navigation with workspace metaphor:** Left sidebar with collapsible sections, a workspace area that remembers state per section. Most complex but best for multi-tasking operators. Defer to Phase B. |
| **Type** | DESIGN — Product Designer + UX Engineer must produce IA diagram before Phase B |

---

## PD-03: Design Tone Unresolved

| Field | Value |
|-------|-------|
| **Code** | PD-03 |
| **Raised by** | Product Designer |
| **Confidence** | 80 (BLOCKER) |
| **What's missing** | The founding principle "instrument, not toy" has never been translated into a design brief, visual direction, or design tokens (color palette, typography scale, spacing system, dark/light mode decision, corner radius, shadow language). The frontend spec describes a Vue 3 app with zero aesthetic constraints. |
| **Why it matters** | Without design tone, the UI Engineer cannot select a CSS approach (raw CSS, Tailwind, component library) or define the design token system that UXE-09 notes is missing. Risk: inconsistent visual language across components delivered by different sprints, or a generic "default Vue" look that undermines the instrument-grade credibility the project demands. |
| **Options** | 1. **Formal design brief with tokens (recommended):** Product Designer writes a 2-page brief covering: dark mode default (instrument-grade), monospace for data, sans-serif for controls, color palette (cybersecurity blue/amber), typography scale, spacing grid, and "no decoration" rule (no gradients, no box shadows beyond elevation, no rounded corners > 4px). Deliver as CSS custom properties + Figma frames. 2. **Adopt Tailwind CSS with custom config:** Use Tailwind's design token system; configure the palette and typography scale in tailwind.config.ts. Fastest to implement, constrains creative drift. 3. **Defer to Phase B:** UI Engineer picks defaults. Risk: visual rework later, no brand consistency. |
| **Type** | DESIGN — Product Designer must produce design token brief before Phase B |

---

## PD-04: Frontend MoSCoW Contradiction

| Field | Value |
|-------|-------|
| **Code** | PD-04 |
| **Raised by** | Product Designer |
| **Confidence** | 85 (BLOCKER) |
| **What's missing** | The project MVP definition explicitly excludes C10 (Frontend) and C11 (Grafana Dashboards) from MVP scope, yet the Phase A task list includes frontend implementation tasks as mandatory. This is a direct contradiction in scope: either frontend is in MVP or it's not. The ambiguity has cascaded — UI Engineer's F1 (UI-01) notes C10 design doc is missing because nobody committed to building it. |
| **Why it matters** | This contradiction must be resolved before any Phase B task can be dispatched to frontend agents. If frontend is truly out of MVP, then all PD/UXE/UI findings from this review are advisory (fix before Phase B of frontend, not before Phase B of MVP). If frontend is in MVP, then a C10 design document must be written, and all 26 findings here become blocking for Phase B entry. Risk: Phase B starts with ambiguous scope, agents receive contradictory instructions, frontend work is half-done and abandoned. |
| **Options** | 1. **Frontend is MVP — write C10 design doc now (recommended):** Accept that an instrument-grade platform needs a UI for operator trust. The Live Monitor view is essential for demonstrating the pipeline works. Scope to one view (Live Monitor) with Grafana embed; defer Device Map, Packet Explorer, and Settings to post-MVP. Write C10 doc this Phase A cycle. 2. **Frontend is post-MVP — make API the operator interface:** The C09 REST API is the only operator interface for MVP. Frontend findings become deferred (not blocking). All PD/UXE/UI findings are reclassified as post-MVP backlog. 3. **Hybrid: dashboard-only frontend (no C10 app):** Ship only C11 Grafana dashboards for MVP. The Live Monitor is a Grafana dashboard, not a Vue app. No C10 code. Simplest path but limits interaction patterns. |
| **Type** | DECISION — User must choose MVP scope boundary |

---

## PD-05: No Operator Workflows

| Field | Value |
|-------|-------|
| **Code** | PD-05 |
| **Raised by** | Product Designer |
| **Confidence** | 75 (ADVISORY) |
| **What's missing** | No task-flow diagrams for any operator scenario. Key workflows like "operator detects a gap and initiates backfill," "operator triages a new device alert," or "operator exports a PCAP fragment for forensics" have no documented interaction paths. The design brief mentions features but not how they're connected. |
| **Why it matters** | Workflows are the bridge between personas and IA — they validate that the navigation structure supports real tasks. Without them, the component tree might group functions that are never used together, or split functions that must be co-located. |
| **Options** | 1. **Define 3 critical-path workflows now (recommended):** (a) Daily monitoring: operator opens dashboard, scans alerts, drills into device details, returns to overview. (b) Gap investigation: operator sees gap alert, opens gap details, initiates manual backfill, verifies backfill complete. (c) Device forensics: operator searches device, views packet timeline, exports PCAP slice. Write as swimlane diagrams. 2. **Define only the daily monitoring workflow:** Covers MVP Live Monitor scope. Defer others to post-MVP. 3. **Defer all workflows to Phase B:** Accept risk that IA may need redesign if workflows reveal structural issues. |
| **Type** | DESIGN — Product Designer + UX Engineer should produce workflow diagrams |

---

## PD-06: Alert UX Underspecified

| Field | Value |
|-------|-------|
| **Code** | PD-06 |
| **Raised by** | Product Designer |
| **Confidence** | 70 (ADVISORY) |
| **What's missing** | The concept of "alerts" appears in the design brief but has no UX specification: alert severity levels, visual treatment per severity, acknowledgment flow, escalation rules, alert grouping/batching, or alert history UI. Without this, the notification system UXE-02 identifies cannot be designed. |
| **Why it matters** | Alerts are the primary operator-facing output — if they're noisy, confusing, or missing critical visual cues (color, icon, priority), operator trust erodes. Silent failure detection (PF-3) requires that alert states are visually distinct. |
| **Options** | 1. **Three-tier alert model (recommended):** Critical (red, requires acknowledgment, persists until resolved), Warning (amber, auto-dismisses after timeout if transient), Info (blue, informational, no acknowledgment). Specify visual treatment, grouping rules (same-source alerts coalesce), and acknowledgment API contract. 2. **Two-tier (Critical + Info only):** Simpler model, defers Warning tier. Risk: operators learn to ignore "Info" alerts. 3. **Defer to Phase B:** UI Engineer and UX Engineer design alert system ad-hoc. Risk: inconsistency, rework. |
| **Type** | DESIGN — Product Designer + UX Engineer should specify alert model |

---

## PD-07: Accessibility — WCAG 2.2 AA Target

| Field | Value |
|-------|-------|
| **Code** | PD-07 |
| **Raised by** | Product Designer |
| **Confidence** | 70 (ADVISORY) |
| **What's missing** | The spec says "Lighthouse ≥ 90" as the accessibility target. This is a tool score, not a standard. Lighthouse 90 can be achieved with automated audits that miss keyboard navigation, focus order, screen reader labels, color contrast exceptions, and reduced-motion preferences. A real accessibility target requires a named conformance level (WCAG 2.2 A/AA/AAA) with specific success criteria commitments. |
| **Why it matters** | "Instrument, not toy" implies the tool must be usable under stress — operators with vision impairments, color blindness, or motor limitations must not be locked out. WCAG 2.2 AA is the legal baseline for government and enterprise tools. Meeting it at MVP avoids expensive retrofits later. Post-MVP, if the tool is deployed in regulated environments, AA may be mandatory. |
| **Options** | 1. **Target WCAG 2.2 AA (recommended):** Commit to all Level A and AA success criteria. This means: keyboard-navigable all views, focus indicators, color contrast ≥ 4.5:1 (normal text) and 3:1 (large text), screen reader labels on all interactive elements, ARIA roles on dynamic content regions (live alerts, table updates), reduced-motion support, and 200% zoom without horizontal scroll. Audit with axe-core + manual keyboard test. 2. **Target WCAG 2.2 A only:** Minimum viable accessibility. Covers basic keyboard navigation and semantic HTML but not contrast or screen reader completeness. Lower bar, lower effort. Risk: not sufficient for regulated environments. 3. **Keep Lighthouse ≥ 90 as target:** Defers real accessibility to post-MVP. Risk: significant rework if WCAG compliance is later required. |
| **Type** | DECISION — User must choose accessibility conformance target |

---

## PD-08: No Design System

| Field | Value |
|-------|-------|
| **Code** | PD-08 |
| **Raised by** | Product Designer |
| **Confidence** | 65 (ADVISORY) |
| **What's missing** | No design system exists — no component library specification, no token definitions, no pattern library, no Figma component frames. Every frontend decision from button styling to table density would be invented during implementation. UXE-09 also notes missing design tokens. |
| **Why it matters** | A design system enforces consistency across components built by different sprints or different agents. Without it, the frontend accumulates visual debt — two different table implementations with different row heights, inconsistent color usage, ad-hoc spacing. For an instrument-grade tool, visual consistency is a trust signal. |
| **Options** | 1. **Lightweight design system with 8 core components (recommended):** Define tokens (colors, spacing, type scale) + 8 reusable components: Button, Badge (alert severity), DataTable (sortable, filterable, virtual-scroll), StatCard (metric display), PageHeader (breadcrumb + actions), SearchBar, TimeRangePicker, NotificationToast. Specify variants and states per component. 2. **Adopt an existing library (e.g., PrimeVue, Naive UI) and customize:** Faster start, but carries unused code and limits visual control. Requires CSS override for "instrument" aesthetic. 3. **Defer to Phase B:** Build components ad-hoc, extract into design system post-MVP. Risk: high rework cost, inconsistent UI. |
| **Type** | DESIGN — Product Designer + UI Engineer should produce design system spec |

---

## UXE-01: State Management — 27% Coverage

| Field | Value |
|-------|-------|
| **Code** | UXE-01 |
| **Raised by** | UX Engineer |
| **Confidence** | 95 (BLOCKER) |
| **What's missing** | Only 27% of necessary application states are addressed. Missing entirely: loading states (skeleton/spinner for every data fetch), empty states (no devices, no packets, no alerts — each needs bespoke messaging), error states (API down, DB unreachable, data stale), offline states (network loss detection, reconnection UX), partial-data states (some data loaded, some pending). |
| **Why it matters** | PF-1 (Design for Failure) and PF-9 (Observability by Design) require that every failure mode produces an observable signal. If the frontend doesn't render loading, error, and offline states, the operator cannot distinguish "no data because nothing is happening" from "no data because the backend is down." This is a silent failure mode at the UI layer — the most dangerous kind. |
| **Options** | 1. **Full state matrix per view (recommended):** For each view (Live Monitor, Device Map, etc.), define the render tree for all 6 states: Loading, Empty, Error, Offline, Stale, and Normal. Write as a state table with mockups for each. 2. **Generic fallback states only:** One loading spinner, one error banner, one "no data" message — reused across all views. Faster but less helpful (operators can't tell which component failed). 3. **Defer error/offline states to Phase B:** Ship with loading + empty only. Risk: PF-1 violation — failure modes that aren't rendered are silent failures. |
| **Type** | DESIGN — UX Engineer must produce state matrix per view before Phase B |

---

## UXE-02: Notification / Alert System

| Field | Value |
|-------|-------|
| **Code** | UXE-02 |
| **Raised by** | UX Engineer |
| **Confidence** | 92 (BLOCKER) |
| **What's missing** | No notification or alert system exists in the design. The spec mentions "alerts" as a concept but defines no interaction model, no visual specification, no persistence semantics, no acknowledgment flow, no aggregation rules, and no API contract. This directly violates PF-5 (Graceful Degradation — if the API is down, the operator must still see the last-known alert state) and PF-9 (Observability by Design — every failure must produce an observable signal). |
| **Why it matters** | The notification system is the operator's primary awareness channel. Without it: (1) gap detection alerts have nowhere to surface, (2) device-classification results have no delivery channel, (3) backend failures are invisible to the operator. If the notifier itself fails, there's no secondary detection mechanism — a single-point-of-observability failure that violates PF-3. |
| **Options** | 1. **Server-Sent Events (SSE) with Pinia store persistence (recommended):** Backend pushes events over SSE (gap detected, device classified, health change). Frontend stores in Pinia, persists across route changes. Toast notifications for real-time, notification center (bell icon) for historical. Critical alerts persist until acknowledged; warnings auto-dismiss; info is ephemeral. Acknowledgment writes to API. Unacknowledged count in badge. 2. **Polling-based (no SSE):** Frontend polls `/api/alerts` every N seconds. Simpler backend (no SSE), higher latency, wasted bandwidth. Acceptable for MVP if N is configurable. 3. **WebSocket:** Bidirectional, lowest latency, most complex (reconnection, auth, message ordering). Overkill for this use case — alerts are server-to-client only. |
| **Type** | DESIGN — UX Engineer + API Designer must specify notification system contract before Phase B |

---

## UXE-03: Live Data Refresh Mechanism

| Field | Value |
|-------|-------|
| **Code** | UXE-03 |
| **Raised by** | UX Engineer |
| **Confidence** | 90 (BLOCKER) |
| **What's missing** | No mechanism is specified for updating the Live Monitor view with real-time data. The design brief says "live view" but doesn't say how data gets from the database to the browser. Options (polling, SSE, WebSocket) have fundamentally different architectural implications — they determine the API contract, the backend infrastructure, the frontend state model, and the failure-recovery patterns. |
| **Why it matters** | The Live Monitor is the primary operator view — its refresh behavior defines the perceived responsiveness of the entire platform. Wrong choice here means architectural rework. Polling is simple but wastes DB connections at scale. SSE is server-to-client and reconnects natively. WebSocket is bidirectional but complex. The decision also affects C09 API design: SSE requires a streaming endpoint that doesn't exist in the current API contract. |
| **Options** | 1. **SSE for Live Monitor, polling fallback (recommended):** One SSE endpoint (`/api/events`) pushes device appearance, packet counts, gap alerts, and health changes. Frontend reconnects on disconnect with Last-Event-ID. If SSE fails (proxy doesn't support it), fall back to polling at configurable interval. 2. **Polling only:** Simpler, no SSE infrastructure. `GET /api/live` returns snapshot. Poll every 5s. Add conditional GET (ETag/If-None-Match) to reduce DB load. 3. **WebSocket:** Full duplex, most real-time. Requires WS-aware proxy, auth per-message, and reconnection logic. Overhead not justified for this use case. |
| **Type** | DECISION — User must choose data refresh mechanism (determines C09 API contract) |

---

## UXE-04: Accessibility Architecture

| Field | Value |
|-------|-------|
| **Code** | UXE-04 |
| **Raised by** | UX Engineer |
| **Confidence** | 88 (BLOCKER) |
| **What's missing** | One sentence about Lighthouse scores is the entirety of the accessibility specification. There is no: keyboard navigation plan, focus management strategy (especially for dynamic content updates), screen reader announcements for live regions, ARIA role assignment for complex widgets (data tables, charts, Grafana iframes), color contrast plan, reduced-motion support, or zoom/reflow strategy. The accessibility architecture must define how the frontend framework (Vue 3) will implement these — not just what the audit tool will score. |
| **Why it matters** | The Live Monitor is a dynamic, data-dense interface. Without a focus management strategy, screen readers will not announce new alerts or table updates. Without keyboard navigation, operators who don't use a mouse cannot triage alerts. Without ARIA live regions, critical alerts may go unannounced. A Lighthouse audit score of 90 cannot detect any of these gaps — only manual testing and architectural planning can. |
| **Options** | 1. **WCAG 2.2 AA architecture spec (recommended):** Define: (a) Keyboard navigation tree — Tab order, skip links, focus trapping in modals. (b) ARIA live regions — `<aria-live="polite">` for table updates, `<aria-live="assertive">` for critical alerts. (c) Focus management — auto-focus new alert card, restore focus after route change. (d) Color — palette with ≥ 4.5:1 contrast, non-color indicators for alert severity (icons). (e) Motion — `prefers-reduced-motion` media query disables transitions. (f) Screen reader text — all icons have `aria-label`, all charts have text fallback. Write as a 3-page spec. 2. **WCAG 2.2 A only:** Basic keyboard + semantic HTML. Less effort. Accept for Phase B entry; close AA gaps in post-MVP. 3. **Defer to Phase B:** Implement with `vue-axe` and fix issues as found. Risk: architecture-level gaps (live regions, focus management) are hard to retrofit. |
| **Type** | DESIGN — UX Engineer must produce accessibility architecture spec |

---

## UXE-05: Grafana Integration UX

| Field | Value |
|-------|-------|
| **Code** | UXE-05 |
| **Raised by** | UX Engineer |
| **Confidence** | 85 (BLOCKER) |
| **What's missing** | The design says Grafana dashboards are embedded via iframe — no specification exists for: embedding method (direct iframe, Grafana's public-embed API, or reverse-proxy), deep-linking (passing time range / device filter to Grafana URL), failure states (Grafana down renders a broken iframe), loading states (iframe load is invisible to the parent page), theme synchronization (parent is dark, Grafana may be light), authentication (anonymous vs forwarded credentials), and cross-origin communication (postMessage for bi-directional interaction). |
| **Why it matters** | "Grafana via iframe" is a 4-word spec for a component that carries high architectural complexity. A broken iframe with no loading/error states violates PF-5 (Graceful Degradation). Anonymous Grafana on a LAN exposes dashboards to anyone on the network — a security concern. Without deep-linking, the operator must manually navigate Grafana to find relevant data, breaking the integrated-instrument experience. |
| **Options** | 1. **Authenticated reverse-proxy with deep-link support (recommended):** C09 API proxies `/grafana/*` to Grafana with C09 auth token. Grafana runs with `allow_embedding = true` and anonymous access disabled. Frontend constructs deep-link URLs with `?from=...&to=...&var-device=...` query params. Render an `<iframe>` with loading skeleton and error fallback (Grafana down → "Dashboard unavailable" card with retry button). Post-message bridge for bi-directional time-range sync. 2. **Anonymous Grafana with direct iframe:** Simpler, no auth proxy. Grafana has `allow_embedding = true`, `auth.anonymous = true`. No deep-linking. Accept risk of LAN exposure. 3. **No Grafana in iframe — open in new tab:** The "Dashboards" nav item opens Grafana in a new browser tab. Simplest, but completely breaks the integrated-tool experience — Grafana is a separate product, not part of the instrument. |
| **Type** | DESIGN — UX Engineer + API Designer must produce Grafana integration spec |

---

## UXE-06: Information Architecture (UX Perspective)

| Field | Value |
|-------|-------|
| **Code** | UXE-06 |
| **Raised by** | UX Engineer |
| **Confidence** | 85 (BLOCKER) |
| **What's missing** | Beyond the Product Designer's IA finding (PD-02), the UX Engineer identifies that 4 flat routes with no hierarchy fail to encode the operator's mental model. The IA must define: landing page identity (what the operator sees first), navigation persistence across route changes, contextual transitions (clicking a device in Live Monitor → Device Detail), breadcrumb trails, and the relationship between app views and Grafana dashboards. |
| **Why it matters** | PD-02 and UXE-06 are the same gap seen from different angles — PD sees missing structure, UXE sees missing interaction. If both are unsatisfied, the frontend will ship with 4 disconnected pages and no workflow continuity. |
| **Options** | 1. **Same as PD-02 Option 1 — co-designed IA:** Product Designer and UX Engineer collaborate on a single IA document covering both structure (PD) and interaction (UXE). The recommended model is: Live Monitor as landing, secondary navigation to Device Map and Packet Explorer, Settings as tertiary, Grafana as deep-linked embed. 2. **Accept PD-02 resolution and defer UXE interaction design to Phase B:** IA is defined now, but contextual transitions and breadcrumbs are designed during implementation. 3. **Defer all IA to Phase B:** Rejected by both PD and UXE. |
| **Type** | DESIGN — Co-owned by Product Designer and UX Engineer; must align with PD-02 |

---

## UXE-07: Device Table Interaction at Scale

| Field | Value |
|-------|-------|
| **Code** | UXE-07 |
| **Raised by** | UX Engineer |
| **Confidence** | 82 (BLOCKER) |
| **What's missing** | The Device Map view implies a table of discovered devices. At 100+ devices (realistic for urban BLE environments), the interaction model is undefined. No specification exists for: sorting (by RSSI, first seen, last seen, device class), filtering (by device type, MAC prefix, time window), pagination vs virtual/infinite scroll, row actions (select, drill-down, export), batch operations, or performance budget (DOM nodes, render time). |
| **Why it matters** | A static `<table>` with 100+ rows will degrade browser performance, especially with live updates (UXE-03). Without filtering, operators drown in noise — a busy BLE environment may show hundreds of transient devices. Without sorting, operators can't find the nearest/strongest device. The table component choice (UI-08) depends on these interaction requirements. |
| **Options** | 1. **Virtual scroll with server-side pagination, sort, and filter (recommended):** Table renders only visible rows (virtual DOM). Sorting and filtering are server-side queries (C09 API supports `?sort=...&filter=...&page=...`). Client maintains a Pinia store of visible devices. Row click drills into Device Detail. Export selected rows to CSV/PCAP. 2. **Client-side with 200-device limit:** Load all devices, sort/filter in browser. Paginate client-side. Simpler API, risky at scale. 3. **Defer to Phase B:** Build simple table first; add virtual scroll when performance degrades. Reasonable if MVP environment has < 50 devices. |
| **Type** | DESIGN — UX Engineer + UI Engineer must co-specify table interaction model |

---

## UXE-08: Operator Workflow Interaction Paths

| Field | Value |
|-------|-------|
| **Code** | UXE-08 |
| **Raised by** | UX Engineer |
| **Confidence** | 80 (BLOCKER) |
| **What's missing** | Zero operator workflow interaction paths are documented. This is the UX-side counterpart of PD-05 (no operator workflows). PD asks "what does the operator do?" UXE asks "how does the operator do it?" — click paths, keyboard shortcuts, multi-step sequences, undo/redo, confirmation dialogs, form validation patterns. |
| **Why it matters** | Without interaction paths, the implementation will produce features that exist but cannot be connected into tasks. The operator will see a device list but won't know how to drill into it. They'll see a gap alert but won't know how to trigger backfill. These paths must be designed — they won't emerge naturally from component implementation. |
| **Options** | 1. **Define interaction paths for 3 critical workflows (recommended):** Same workflows as PD-05, but specified as click-path diagrams with wireframes: screen sequence, button labels, confirmation states, error feedback, and success states. Deliver as 3 annotated wireframe flows (Figma or hand-drawn). 2. **Define 1 workflow (daily monitoring) now, defer others:** Minimum path to validate IA and state architecture. Others designed in Phase B. 3. **Defer all to Phase B:** Implementer defines interaction ad-hoc. Risk: wrong interaction model baked into architecture, costly to change. |
| **Type** | DESIGN — UX Engineer must produce interaction path diagrams |

---

## UXE-09: Advisory Gaps (Store Architecture, State Preservation, Charting, Responsive, Tokens, Settings)

| Field | Value |
|-------|-------|
| **Code** | UXE-09 |
| **Raised by** | UX Engineer |
| **Confidence** | 75-78 (ADVISORY — composite finding) |
| **What's missing** | Six secondary gaps identified in the UX review advisory section: (1) No Pinia store architecture — store modules, their responsibilities, and cross-store dependencies are undefined. (2) No state preservation across routes — if operator navigates from Live Monitor to Device Detail and back, does Live Monitor state (filters, scroll position, selected alert) survive? (3) No charting library selected — blocks Live Monitor data visualization. (4) No responsive breakpoints — the design assumes a desktop monitor; tablet/mobile behavior is unplanned. (5) No design tokens — colors, spacing, type scale, corner radius are all unspecified (overlaps PD-03 and PD-08). (6) No Settings view design — what is configurable, how is it presented? |
| **Why it matters** | These are implementation-blocking gaps: the UI Engineer (UI-03, UI-08) cannot select a charting library without the UX Engineer specifying what charts are needed. The state architecture defines the Pinia store hierarchy which the component tree depends on. Responsive breakpoints affect every component. These are "advisory" only because they can be resolved in Phase B — but every one left unresolved increases implementation risk and rework probability. |
| **Options** | 1. **Resolve all 6 in C10 design doc (recommended):** Write a UX specification section in C10 that covers: store module map, route-state preservation strategy (keep-alive or store snapshots), charting library requirements (time-series, bar, scatter), responsive strategy (desktop-first, min-width 1024px), design token file (CSS custom properties), and Settings view wireframe. 2. **Resolve top 3 (store, state preservation, charting) now, defer bottom 3 to Phase B:** Reduces up-front spec cost. Risk: inconsistent tokens, no mobile fallback. 3. **Defer all 6 to Phase B:** Implementer decides ad-hoc. Highest risk — these are architectural, not cosmetic, decisions. |
| **Type** | DESIGN — UX Engineer + UI Engineer co-specify in C10 document |

---

## UI-01: C10 Design Document Missing

| Field | Value |
|-------|-------|
| **Code** | UI-01 |
| **Raised by** | UI Engineer |
| **Confidence** | 95 (BLOCKER) |
| **What's missing** | The `component-breakdown.md` mandates a C10 Frontend design document as a pre-Phase B deliverable. It does not exist. The frontend section of the inception spec is a feature list, not a design document — it lacks component decomposition, build configuration, API client architecture, TypeScript types, state management plan, and test architecture. |
| **Why it matters** | Without C10, every Phase B frontend task is dispatched with undocumented assumptions. Agents will invent their own conventions for component structure, API calling, type definitions, and store organization — leading to a patchwork codebase. The C10 doc is the blueprint: without it, Phase B builds without a plan. |
| **Options** | 1. **Write C10 design doc now (recommended):** Produce a full C10 document covering: component tree and hierarchy, Vite configuration, API client architecture, TypeScript type generation, Pinia store architecture, test strategy, build pipeline, dependency manifest, and design token system. Use `doc/designs/c10-frontend.md` template from `component-breakdown.md`. Estimated 8-12 pages. 2. **Write a minimal C10 covering only blocking decisions:** Vite config, component tree (Live Monitor only), API client, and dependencies (the 7 from UI-08). Defer full doc to post-MVP. Risk: conventions diverge as more components are added. 3. **Start Phase B without C10 — document as-you-build:** Agents write sections of C10 as they implement. Risk: no upfront design review, architecture decisions made in isolation. |
| **Type** | DECISION — User must authorize C10 document creation and assign specialists |

---

## UI-02: No Vite Configuration

| Field | Value |
|-------|-------|
| **Code** | UI-02 |
| **Raised by** | UI Engineer |
| **Confidence** | 93 (BLOCKER) |
| **What's missing** | Zero Vite/build configuration exists. Undefined: base path (production subdirectory or root), proxy configuration (C09 API dev proxy), environment variable schema (.env files and their keys), module resolution aliases (path shortcuts), build output directory, asset handling (static files, fonts, images), and dev server settings (port, HMR, CORS). |
| **Why it matters** | The Vite configuration determines the entire frontend build pipeline. Wrong base path → broken production routing. No dev proxy → manual CORS configuration in every API call, broken in development. No path aliases → deep relative imports (`../../../stores/device.ts`). This must be designed, not discovered during `npm create vue`. |
| **Options** | 1. **Standard Vite config with proxy and aliases (recommended):** `base: '/'` (nginx reverse-proxy handles subdirectory). `server.proxy: { '/api': 'http://localhost:8000' }` for C09 dev. `resolve.alias: { '@': '/src' }`. `.env.development` and `.env.production` with `VITE_API_BASE_URL`. Output to `dist/`. Document all env var keys in C10. 2. **Minimal config (no proxy, no aliases):** Accept relative imports and CORS headaches during dev. Simplest, lowest up-front cost. Risk: developer friction, inconsistent import paths. 3. **Defer to Phase B:** Implementer writes Vite config as first task. Risk: no review of proxy/CORS/alias decisions. |
| **Type** | DESIGN — UI Engineer must specify Vite config in C10 document |

---

## UI-03: Component Tree Infeasible

| Field | Value |
|-------|-------|
| **Code** | UI-03 |
| **Raised by** | UI Engineer |
| **Confidence** | 90 (BLOCKER) |
| **What's missing** | The current component hierarchy (Live Monitor, Device Map, Packet Explorer, Settings, + Grafana dashboards) cannot be implemented because three essential component categories have no selected library or design: (1) No charting library — Live Monitor requires time-series and bar charts. (2) No table component — Device Map and Packet Explorer require sortable, filterable data tables. (3) Grafana integration is "iframe" — no component design for the iframe wrapper, deep-linking, or error states. |
| **Why it matters** | The component tree is the implementation roadmap. If 3 of its 5 branches can't be built, the tree is not a plan — it's aspirational. The UI-08 dependency finding is downstream of this gap. |
| **Options** | 1. **Reduce MVP component tree to Live Monitor only (recommended):** Implement one view with: stat cards (device count, packet rate, gaps), time-series chart (packet volume over time), alert feed (notification list), and Grafana iframe embed. Defer Device Map, Packet Explorer, and Settings to post-MVP. This makes the tree feasible with 3 dependency decisions (charting, notifications, iframe) instead of 7+. 2. **Full tree, but accept placeholder components:** Implement all views as skeletons with "coming soon" — only Live Monitor is functional. This validates IA/routing but provides no operator value beyond the first view. 3. **Defer component tree definition to Phase B:** Highest risk — UI-08 dependencies remain unresolved, implementer must make all library decisions. |
| **Type** | DECISION — User must choose MVP component scope (aligns with PD-04) |

---

## UI-04: API Client Architecture Unresolved

| Field | Value |
|-------|-------|
| **Code** | UI-04 |
| **Raised by** | UI Engineer |
| **Confidence** | 88 (BLOCKER) |
| **What's missing** | The frontend must call the C09 REST API, but the client architecture is undefined: (1) OpenAPI client generation vs hand-written TypeScript types — if C09 produces an OpenAPI spec, should the frontend use `openapi-typescript` + `openapi-fetch` or hand-write fetch wrappers? (2) CORS configuration — C09 must expose appropriate CORS headers for the frontend origin. (3) Authentication token management — how does the frontend obtain, store, refresh, and attach the API key? (4) Error handling strategy — per-endpoint vs global error handler, retry logic, exponential backoff. |
| **Why it matters** | The API client is the nervous system of the frontend — every component depends on it. Wrong choice here means every data-fetching composable must be rewritten. OpenAPI gen gives type safety but introduces a build dependency on the C09 spec. Hand-written types give control but drift from the backend. CORS misconfiguration blocks all dev work. |
| **Options** | 1. **OpenAPI-generated client with typed fetch wrapper (recommended):** If C09 exposes `/openapi.json`, use `openapi-typescript` to generate types and `openapi-fetch` for the typed client. Pinia stores wrap the client. API key in `Authorization: Bearer <key>` header, stored in Pinia + localStorage, refreshed via `/api/auth/refresh`. Global Axios-like interceptor for 401/403/5xx errors. Retry with exponential backoff for 503. 2. **Hand-written typed client:** Define TypeScript interfaces manually, write fetch wrappers per endpoint. No build-time dependency on C09 spec. Risk: drift between backend types and frontend types. 3. **Defer to Phase B:** Implementer chooses during first API call task. Risk: no architectural review, inconsistent patterns across stores. |
| **Type** | DESIGN — UI Engineer + API Designer must specify API client architecture |

---

## UI-05: Bundle / Performance Budget

| Field | Value |
|-------|-------|
| **Code** | UI-05 |
| **Raised by** | UI Engineer |
| **Confidence** | 85 (BLOCKER) |
| **What's missing** | No performance budget exists for the frontend: bundle size (JS + CSS), initial load time, time-to-interactive, browser memory budget, or render performance (FPS during live updates). Specifically, the Grafana iframe adds an entire second web application inside the page — its memory and CPU cost must be quantified. A single Grafana dashboard can consume 50-100 MB of browser memory. |
| **Why it matters** | The target hardware (Raspberry Pi CM5) is not the browser host, but operators may access from constrained devices (thin clients, older laptops). A page that consumes 300+ MB with Grafana iframes open will be unusable on 4 GB machines. Without a budget, performance issues are discovered after launch. |
| **Options** | 1. **Define tiered budget (recommended):** JS bundle < 500 KB (gzipped), CSS < 100 KB, initial load < 3s on 10 Mbps, time-to-interactive < 5s, idle browser memory < 150 MB (no Grafana) / < 300 MB (with 2 Grafana iframes). Test with Lighthouse CI in CI pipeline. Grafana iframes loaded lazily (not on initial page load). 2. **Lighthouse score target only:** "Performance ≥ 90." Coarser but simpler. Doesn't catch memory issues. 3. **Defer to Phase B:** Measure after build, fix what's slow. Risk: architectural decisions (e.g., heavy charting library) lock in poor performance. |
| **Type** | DESIGN — UI Engineer must define performance budget in C10 |

---

## UI-06: TypeScript Strict Mode

| Field | Value |
|-------|-------|
| **Code** | UI-06 |
| **Raised by** | UI Engineer |
| **Confidence** | 82 (BLOCKER) |
| **What's missing** | The AGENTS.md tech stack mandates "TypeScript tsc 5.3+ (strict mode)" but no `tsconfig.json` exists, no type generation pipeline is defined, and no typed API contracts are specified. Strict mode means `strict: true` in tsconfig — which enables `noImplicitAny`, `strictNullChecks`, `strictFunctionTypes`, and 5 other flags that catch entire classes of bugs at compile time. |
| **Why it matters** | "Instrument, not toy" means type safety is mandatory, not optional. Without strict mode and typed API contracts, the frontend accumulates `any` types, unchecked null accesses, and mismatched API shapes — exactly the kind of silent failures PF-9 requires us to detect. Strict mode is cheap to enable at project start and expensive to retrofit. |
| **Options** | 1. **Strict mode + generated API types (recommended):** `tsconfig.json` with `strict: true`, `noUncheckedIndexedAccess: true`, `exactOptionalPropertyTypes: false`. API types generated from C09 OpenAPI spec via `openapi-typescript`. Component props fully typed. Vue SFC `<script setup lang="ts">`. CI runs `vue-tsc --noEmit`. 2. **Strict mode with hand-written types:** Same strictness, but types are maintained manually. Accepts drift risk. 3. **Non-strict mode (deferred):** `strict: false` during Phase B, enable strict in post-MVP. Risk: large type debt, breaking changes when strict is enabled. |
| **Type** | DESIGN — UI Engineer must produce tsconfig.json and type generation plan |

---

## UI-07: Test Architecture Incomplete

| Field | Value |
|-------|-------|
| **Code** | UI-07 |
| **Raised by** | UI Engineer |
| **Confidence** | 80 (BLOCKER) |
| **What's missing** | The AGENTS.md tech stack mandates Vitest 1.2+ but no test architecture is designed: no store tests, no composable tests, no component tests, no E2E tests, no error state tests, no vitest config, no test directory structure, no mock strategy for API calls, no CI integration. The existing test plan covers C++ and Python but not Vue/TypeScript. |
| **Why it matters** | PF-1 (Design for Failure) applies to the frontend — every component's error state must be tested. Without store tests, the state management layer (UXE-01) has no verification. Without composable tests, the API client layer (UI-04) has no safety net. A frontend with 0 tests violates the project's reliability principles. |
| **Options** | 1. **Vitest unit + component tests with MSW for API mocking (recommended):** `vitest` for unit tests (stores, composables, utility functions). `@vue/test-utils` for component tests. `msw` (Mock Service Worker) for API mocking — intercepts fetch at the network level, works in both unit and component tests. Test directory: `src/__tests__/` mirroring source structure. CI: `vitest run --coverage`. Target: 80% line coverage for stores and composables. 2. **Unit tests only (no component tests):** Test stores and composables; skip Vue component rendering tests. Faster, simpler, but no UI-level verification. 3. **Defer to Phase B:** Write tests after implementation. Risk: tests never get written, refactoring is unsafe. |
| **Type** | DESIGN — UI Engineer must produce test architecture section in C10 |

---

## UI-08: Seven Technical Dependencies Unresolved

| Field | Value |
|-------|-------|
| **Code** | UI-08 |
| **Raised by** | UI Engineer |
| **Confidence** | 80 (BLOCKER) |
| **What's missing** | Seven technical decisions that are prerequisites for writing any frontend code are unresolved. Until these are chosen, `package.json` cannot be finalized, the build pipeline cannot be configured, and no component can be implemented with its actual dependencies. |
| **Why it matters** | Each dependency choice carries architectural weight — bundle size, tree-shaking support, TypeScript types, Vue 3 integration, and long-term maintenance. Choosing wrong means mid-Phase B library swaps, which are among the most expensive changes to make. These 7 decisions collectively determine the frontend's technical foundation. |

### Dependency 1: Charting Library

| Aspect | Detail |
|--------|--------|
| **What it must support** | Time-series line charts (packet volume over time), bar charts (device count by type), responsive resizing, dark theme, TypeScript types, Vue 3 composable API, tree-shaking |
| **Options** | 1. **Chart.js 4 + vue-chartjs wrapper (recommended):** Mature, 60 KB gzipped, excellent time-series support, Canvas-based (no DOM overhead for large datasets), Vue 3 wrapper maintained. Cons: no built-in crosshair/tooltip sync, less "pretty" defaults than ECharts. 2. **ECharts 5 + vue-echarts:** Most feature-rich, built-in dark theme, best for complex dashboards. 170 KB gzipped. Overkill for this project's chart needs. 3. **D3.js:** Maximum flexibility, no chart abstractions — you build everything. Steep learning curve, high maintenance cost. Not recommended for this project. |
| **Recommended** | Chart.js 4 + vue-chartjs |

### Dependency 2: Table Component

| Aspect | Detail |
|--------|--------|
| **What it must support** | Sortable columns (ASC/DESC toggle), filterable by column value, virtual scroll for 100+ rows, row selection, responsive columns, TypeScript types, Vue 3 composition API |
| **Options** | 1. **TanStack Table v8 + custom Vue wrapper (recommended):** Headless — you control rendering. Full TypeScript, virtual scroll support via `@tanstack/virtual`, sort/filter/pagination built in. 15 KB gzipped (core). Requires writing your own markup (no pre-styled cells). 2. **AG Grid Community:** Most feature-rich, virtual scroll, sort, filter, export built-in. 200+ KB gzipped. Heavy. 3. **Vxe Table:** Good Vue 3 support, virtual scroll, sort, filter. Moderate bundle size. Smaller community than TanStack. |
| **Recommended** | TanStack Table v8 + @tanstack/vue-virtual |

### Dependency 3: Date Handling

| Aspect | Detail |
|--------|--------|
| **What it must support** | Parse ISO 8601 timestamps from API, format for display (relative "2 min ago" + absolute "2026-06-08 14:32:00"), timezone awareness, duration calculations ("gap lasted 45s"), lightweight (tree-shakeable) |
| **Options** | 1. **date-fns 3 (recommended):** Functional, tree-shakeable, immutable, 10-20 KB per function imported, full TypeScript. Lightest option. No chainable API — each function is standalone. 2. **Luxon 3:** DateTime wrapper with timezone and duration support. Heavier (60 KB). Only needed if timezone manipulation is complex. 3. **day.js:** Moment-compatible API, 2 KB core. Plugin-based (timezone, duration are plugins). Smaller than date-fns but plugin ecosystem is less TypeScript-native. |
| **Recommended** | date-fns 3 |

### Dependency 4: Notifications / Toasts

| Aspect | Detail |
|--------|--------|
| **What it must support** | Toast notifications (auto-dismiss, manual dismiss), severity levels (info, warning, error/critical), stacking behavior, persistence for critical alerts, programmatic API (call from Pinia actions), Vue 3 composable, TypeScript |
| **Options** | 1. **vue-sonner (recommended):** Vue 3 port of Sonner — beautiful, animated, accessible (ARIA live region), 5 KB gzipped. Rich customization, promise support. Severity levels via variants. Cons: newer library, smaller community. 2. **vue-toastification:** Mature, well-tested, 10 KB. Supports severity, stacking, custom components. Less visually polished than Sonner. 3. **Build custom with Pinia:** Full control, zero dependency. Cost: design, build, test, maintain your own toast system. Risk: accessibility gaps if not built carefully. |
| **Recommended** | vue-sonner |

### Dependency 5: CSS Framework / Version

| Aspect | Detail |
|--------|--------|
| **What it must support** | Utility-first or component CSS, dark mode default, design token integration (CSS custom properties), tree-shaking (purge unused CSS in production), responsive utilities, Vue 3 SFC compatibility |
| **Options** | 1. **Tailwind CSS 4 (recommended):** Utility-first, CSS-first configuration (no JS config file in v4), automatic dark mode via `@media (prefers-color-scheme: dark)`, JIT compiler purges unused styles, 4 KB gzipped in production. Perfect for "instrument" aesthetic — constrain visual choices to a defined token set. 2. **Tailwind CSS 3:** Proven, stable, larger ecosystem. JS-based config. Slightly larger build. Good if v4 migration risk is a concern. 3. **Vanilla CSS with CSS Modules:** Zero dependency, full control. Highest maintenance cost — you build your own spacing, color, typography system. Not recommended for team consistency. |
| **Recommended** | Tailwind CSS 4 |

### Dependency 6: Icon Library

| Aspect | Detail |
|--------|--------|
| **What it must support** | SVG icons, tree-shakeable (import only used icons), consistent sizing, accessible (aria-label support), Vue 3 component or composable, dark-theme-appropriate (stroke/fill adaptable) |
| **Options** | 1. **Lucide Vue (recommended):** 1000+ icons, 1 KB per icon, tree-shakeable, Vue 3 components, customizable (size, color, stroke-width via props), MIT licensed. Clean, consistent, modern aesthetic. 2. **Heroicons (Vue port):** 300 icons, Tailwind-aligned, 2 styles (outline, solid). Smaller set, but designed to work perfectly with Tailwind. 3. **@iconify/vue:** Largest set (100,000+ icons from multiple packs), on-demand loading, tree-shakeable. More complex setup, larger ecosystem. Overkill unless you need specific icon packs. |
| **Recommended** | Lucide Vue |

### Dependency 7: Vite Config — Path Aliases

| Aspect | Detail |
|--------|--------|
| **What it must support** | TypeScript path aliases (`@/` → `src/`), Vite resolve alias (mirrors tsconfig paths), sub-aliases for stores, composables, components |
| **Options** | 1. **Single `@/` alias + `vite-tsconfig-paths` plugin (recommended):** Define `@/*` → `src/*` in `tsconfig.json` paths. `vite-tsconfig-paths` reads tsconfig and auto-configures Vite resolve aliases. Single source of truth — no duplication. Sub-imports as `@/stores/device`, `@/components/DeviceTable.vue`. 2. **Multiple aliases with manual Vite config:** Define `@store/`, `@comp/`, `@api/` in both `tsconfig.json` and `vite.config.ts` manually. More granular, but higher maintenance. 3. **No aliases:** Use relative imports (`../../../stores/device.ts`). Fragile on refactor, hard to read. Not recommended. |
| **Recommended** | Single `@/` alias + vite-tsconfig-paths plugin |
| **Type** | DECISION — User must approve 7 dependency choices (each is individually a DECISION; collectively they form a dependency lock manifest for C10) |

---

## UI-09: Grafana Embedding Security

| Field | Value |
|-------|-------|
| **Code** | UI-09 |
| **Raised by** | UI Engineer |
| **Confidence** | 70 (ADVISORY) |
| **What's missing** | Grafana dashboards embedded via iframe introduce security concerns that are unaddressed: (1) CSP headers — the parent page's Content-Security-Policy must allow `frame-src` to the Grafana origin. (2) X-Frame-Options — Grafana must send `X-Frame-Options: SAMEORIGIN` or the embed will be blocked by the browser. (3) Anonymous exposure — if Grafana allows anonymous access (common for embedded dashboards), anyone on the LAN who knows the Grafana port can view dashboards without authentication. |
| **Why it matters** | PF-7 (Defense in Depth) requires security at every layer. An unauthenticated Grafana on a LAN is a direct violation — it exposes operational data (device counts, packet volumes, gap statistics) to anyone on the network. The CSP misconfiguration is a silent failure — the iframe simply renders blank, and the operator doesn't know why. |
| **Options** | 1. **Authenticated reverse-proxy with restrictive CSP (recommended):** C09 reverse-proxies `/grafana/*` → Grafana backend, enforcing C09 API key authentication. CSP header: `frame-src 'self'` (Grafana served from same origin via proxy). Grafana config: `allow_embedding = true`, `auth.anonymous = false`. Grafana dashboards accessible only through C09 auth. 2. **Anonymous Grafana with CSP allowlist:** Grafana on separate port (e.g., 3000), anonymous access enabled for embedded dashboards. CSP: `frame-src http://localhost:3000`. Accepts LAN exposure risk. 3. **No iframe — open Grafana in new tab:** Sidesteps CSP and X-Frame-Options entirely. Grafana remains independently authenticated. Tradeoff: broken "integrated instrument" experience (see UXE-05). |
| **Type** | DESIGN — API Designer + UI Engineer must specify embedding security in C09/C10 documents |

---

## Summary: Finding Type Distribution

| Type | Count | Codes |
|------|-------|-------|
| **DECISION** | 5 | PD-04, PD-07, UXE-03, UI-01, UI-03, UI-08 (7 sub-decisions) |
| **DESIGN** | 16 | PD-01, PD-02, PD-03, PD-05, PD-06, PD-08, UXE-01, UXE-02, UXE-04, UXE-05, UXE-06, UXE-07, UXE-08, UXE-09, UI-02, UI-04, UI-05, UI-06, UI-07, UI-09 |
| **Total** | 26 | PD-01 through UI-09 |

*Note: Some findings span categories. PD-04 and UI-03 are both scope decisions (DECISION) and design triggers (DESIGN). UI-08 is classified as DECISION because the user must explicitly approve the dependency manifest, but each dependency also triggers DESIGN work (dependency spec in C10).*

---

## Summary: Blocking vs Advisory

| Severity | Count | Codes |
|----------|-------|-------|
| **BLOCKER (≥80)** | 19 | PD-01, PD-02, PD-03, PD-04, UXE-01, UXE-02, UXE-03, UXE-04, UXE-05, UXE-06, UXE-07, UXE-08, UI-01, UI-02, UI-03, UI-04, UI-05, UI-06, UI-07, UI-08 |
| **ADVISORY (<80)** | 7 | PD-05, PD-06, PD-07, PD-08, UXE-09, UI-09 |

*Note: PD-07 (70) is ADVISORY but elevated here — WCAG 2.2 AA is recommended as a project-level commitment that affects every component. UI-09 (70) is security-related and should be elevated if LAN exposure is a concern.*

---

## Next Steps

1. **Resolve 5 DECISION-type findings:** User answers PD-04 (MVP scope), PD-07 (accessibility target), UXE-03 (live data mechanism), UI-01 (C10 document), UI-03 (MVP component tree), and UI-08 (7 dependency choices).
2. **Dispatch 16 DESIGN-type findings to specialists:** Product Designer (PD-01 through PD-08), UX Engineer (UXE-01 through UXE-09), UI Engineer (UI-01 through UI-09). Some are co-owned — see individual entries.
3. **Write C10 Frontend Design Document:** After decisions are resolved and designs are produced, consolidate into a single C10 document following the `doc/designs/c10-frontend.md` template.
4. **Re-run Phase A gate:** After all 26 findings are resolved, re-submit TIANER-001 for Phase A re-review by all three specialists.
