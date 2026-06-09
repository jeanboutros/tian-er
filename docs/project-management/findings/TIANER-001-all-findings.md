# TIANER-001 — Codified Findings (All Specialists)

**Compiled:** 2026-06-08
**Source:** 9 specialist Phase A reviews
**Status:** Open findings only — resolved items excluded

---

## Resolved (Not Listed Below)

| Domain | Items | Disposition |
|--------|-------|-------------|
| Hardware Engineer | 6 findings | APPROVED |
| Wireless Expert | 7 findings | APPROVED |
| Test Engineer | 9 findings | APPROVED |
| Software Engineer | F-NEW-1, F-NEW-2, F-NEW-3 | RESOLVED (5-fixes pass) |
| Security Reviewer | SX-F11 | RESOLVED (5-fixes pass) |
| Docs Writer | F11, F12 | RESOLVED (5-fixes pass) |
| Product Designer | F3 (design tone), F4 (frontend scope), F6 (alert UX) | ANSWERED by user |
| UX Engineer | F2 (notification system), F5 (Grafana iframe) | ANSWERED by user |

INFRA (SW/SX) and DOCS (DX) have **zero remaining open findings**.

---

## PRODUCT (PD) — Product Designer
### 5 open findings

| Code | Source | Confidence | Description | Type |
|------|--------|------------|-------------|------|
| PD-01 | Product Designer F1 | 90 | No user personas defined — who is this for? | DECISION |
| PD-02 | Product Designer F2 | 85 | No information architecture — 4 views + 4 dashboards, no hierarchy or primary workflow | DESIGN |
| PD-03 | Product Designer F5 | 75 | No operator workflows defined (advisory) | DESIGN |
| PD-04 | Product Designer F7 | 70 | Accessibility target underspecified — needs WCAG 2.2 AA commitment beyond "Lighthouse ≥ 90" (advisory) | DECISION |
| PD-05 | Product Designer F8 | 65 | No design system — tokens, color palette, typography undefined (advisory) | DESIGN |

---

## UX (UXE) — UX Engineer
### 12 open findings

| Code | Source | Confidence | Description | Type |
|------|--------|------------|-------------|------|
| UXE-01 | UX Engineer F1 | 95 | State management — 27% coverage, zero loading/empty/error/offline states | DESIGN |
| UXE-02 | UX Engineer F3 | 90 | No live data refresh mechanism defined (polling vs SSE vs WebSocket) | DECISION |
| UXE-03 | UX Engineer F4 | 88 | No accessibility architecture — one sentence about Lighthouse | DESIGN |
| UXE-04 | UX Engineer F6 | 85 | No information architecture beyond 4 flat routes — no hierarchy or primary workflow | DESIGN |
| UXE-05 | UX Engineer F7 | 82 | Device table interaction undefined for 100+ devices (pagination, sorting, filtering) | DESIGN |
| UXE-06 | UX Engineer F8 | 80 | Zero operator workflow interaction paths | DESIGN |
| UXE-07 | UX Engineer F9 | advisory | No Pinia store architecture | IMPLEMENTATION |
| UXE-08 | UX Engineer F10 | advisory | No state preservation strategy across route navigation | IMPLEMENTATION |
| UXE-09 | UX Engineer F11 | advisory | Charting library not selected | DECISION |
| UXE-10 | UX Engineer F12 | advisory | No responsive breakpoints defined | IMPLEMENTATION |
| UXE-11 | UX Engineer F13 | advisory | No design tokens for typography/color/spacing | DESIGN |
| UXE-12 | UX Engineer F14 | advisory | No settings view UX design | DESIGN |

---

## UI (UI) — UI Engineer
### 9 open findings

| Code | Source | Confidence | Description | Type |
|------|--------|------------|-------------|------|
| UI-01 | UI Engineer F1 | 95 | C10 frontend design document does not exist — required by component-breakdown | DESIGN |
| UI-02 | UI Engineer F2 | 93 | No Vite configuration — base path, proxy, env vars, module resolution | IMPLEMENTATION |
| UI-03 | UI Engineer F3 | 90 | Component tree infeasible — no charting library, no table component, Grafana iframe undesigned | DESIGN |
| UI-04 | UI Engineer F4 | 88 | API client architecture unresolved — OpenAPI gen vs hand-written types, CORS unconfigured | DESIGN |
| UI-05 | UI Engineer F5 | 85 | No bundle/performance budget — browser memory with Grafana iframes uncalculated | DESIGN |
| UI-06 | UI Engineer F6 | 82 | TypeScript strict mode unconfigured — no tsconfig.json, no typed API contracts | IMPLEMENTATION |
| UI-07 | UI Engineer F7 | 80 | Test architecture incomplete — no store/composable/error-state tests, no vitest config | IMPLEMENTATION |
| UI-08 | UI Engineer F8 | 80 | Seven technical dependencies unresolved — charting, table, date, notifications, CSS, icons, Vite | DECISION |
| UI-09 | UI Engineer F9 | 70 | Grafana embedding security — CSP headers, X-Frame-Options, anonymous exposure on LAN (advisory) | IMPLEMENTATION |

---

## INFRA (SW/SX) — Software Engineer + Security Reviewer
### 0 open findings
All findings resolved in 5-fixes pass or earlier retries.

---

## DOCS (DX) — Docs Writer
### 0 open findings
All findings resolved in 5-fixes pass.

---

## Summary

| Category | Open | DECISION | DESIGN | IMPLEMENTATION | Blocking (≥80) |
|----------|------|----------|--------|----------------|-----------------|
| PRODUCT  | 5    | 2        | 3      | 0              | 2               |
| UX       | 12   | 2        | 7      | 3              | 5               |
| UI       | 9    | 1        | 5      | 3              | 7               |
| INFRA    | 0    | 0        | 0      | 0              | 0               |
| DOCS     | 0    | 0        | 0      | 0              | 0               |
| **TOTAL** | **26** | **5**  | **15** | **6**          | **14**          |

Blocking findings (confidence ≥80): **14**

---

## Answer Template (copy and paste)

```
PD-01: [DECISION — who are the users?]
PD-02: [DECISION — information architecture approval?]
PD-03: [DESIGN — define operator workflows, or defer?]
PD-04: [DECISION — commit to WCAG 2.2 AA?]
PD-05: [DESIGN — define design system, or defer?]

UXE-01: [DESIGN — state coverage for all views/states?]
UXE-02: [DECISION — polling vs SSE vs WebSocket for live data?]
UXE-03: [DESIGN — accessibility architecture plan?]
UXE-04: [DESIGN — information architecture redesign?]
UXE-05: [DESIGN — device table pagination/sort/filter spec?]
UXE-06: [DESIGN — operator workflow interaction paths?]
UXE-07: [IMPLEMENTATION — Pinia store architecture, Phase B?]
UXE-08: [IMPLEMENTATION — state preservation, Phase B?]
UXE-09: [DECISION — which charting library?]
UXE-10: [IMPLEMENTATION — responsive breakpoints, Phase B?]
UXE-11: [DESIGN — design tokens specification?]
UXE-12: [DESIGN — settings view UX, or defer?]

UI-01: [DESIGN — create C10 frontend design document?]
UI-02: [IMPLEMENTATION — Vite config, Phase B?]
UI-03: [DESIGN — component tree feasibility plan?]
UI-04: [DESIGN — API client architecture decision?]
UI-05: [DESIGN — bundle/performance budget targets?]
UI-06: [IMPLEMENTATION — tsconfig.json, Phase B?]
UI-07: [IMPLEMENTATION — test architecture, Phase B?]
UI-08: [DECISION — resolve 7 technical dependencies?]
UI-09: [IMPLEMENTATION — Grafana CSP headers, Phase B?]
```
