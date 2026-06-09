# TIANER-001 — Specialist Finding Resolutions

**Date:** 2026-06-08
**Source:** User answers to 26 codified findings from PD, UXE, UIE reviews

## DECIDED (15 items — definitive answers)

| Code | Resolution |
|------|-----------|
| PD-01 | User persona: home lab operator — technically sophisticated, running this at home out of curiosity. Single persona for MVP. |
| PD-02 | Landing page: dashboard of analytics + side status box (expandable, green/grey for device status). Navigation hierarchy to be designed by UXE. |
| PD-03 | Design tone: cyberpunk, dark mode, inspired by https://breachlab.org/ |
| PD-03-theme | Theme selected: BreachLab (moderated). Keep amber accent (#f5a00c), dark background (#0a0a0f), monospace data. Tone down CRT/glitch effects for professional instrument feel. |
| PD-04 | Frontend IS in MVP (not excluded). C10 design doc must be written. |
| PD-06 | Alert UX: deferred to post-MVP. No alerts for MVP. |
| PD-07 | Accessibility: user requests UX Engineer recommendation (WCAG 2.2 AA vs A). UXE to propose with rationale. |
| PD-08 | Design system: user requests Product Designer + UI Engineer proposal. Lightweight design system with tokens (colors, spacing, type scale). |
| UXE-01 | States: simple MVP model that can evolve. UXE to propose. |
| UXE-02 | No notification system for MVP. Confirmed. |
| UXE-03 | Live data: SSE (Server-Sent Events) for real-time updates. C09 API must add SSE endpoint. |
| UXE-04 | Accessibility architecture: UXE to propose simple model (basic keyboard nav, ARIA landmarks, focus management). |
| UXE-05 | Grafana integration: anonymous access for start. Embedded iframe in Vue app. |
| UXE-06 | Information architecture: UXE to design navigation hierarchy with dashboard landing page + side status box. |
| UXE-07 | Device table: simpler client-side sorting/filtering (not server-side). Suitable for <200 devices. |
| UXE-09 | ECharts 5 selected for charting. Tablet support included. UXE to suggest CSS tokens. Settings view: simple. |
| UI-01 | C10 design document: full document required. All 8 sections per component-breakdown template. |
| UI-02 | Vite config: standard with proxy + aliases (recommended from finding details). |
| UI-03 | MVP component tree: full 4 views (Dashboard, Device Map, Packet Explorer, Settings). |
| UI-04 | API client: OpenAPI-generated types (recommended). |
| UI-05 | Bundle budget: JS<500KB gzipped, CSS<100KB, load<3s at 10Mbps, memory<300MB with 2 Grafana iframes. Confirmed. |
| UI-06 | TypeScript: strict mode + generated API types (recommended). |
| UI-07 | Test architecture: Vitest + MSW + 80% store/composable coverage (recommended). |
| UI-08 | Dependencies: ECharts 5 (charting), TanStack Table v8 (table), date-fns 3 (dates), vue-sonner (notifications, deferred to post-MVP), Tailwind CSS 4 (CSS), Lucide Vue (icons), @/ + vite-tsconfig-paths (aliases). |
| UI-09 | Grafana security: anonymous access for MVP. CSP allowlist for frame-src. |

## DESIGN TASKS (7 items — specialist to propose)

| Code | Task | Assigned To |
|------|------|-------------|
| PD-05 | Define 3 operator workflows (daily monitoring, gap investigation, device forensics) | Product Designer (propose → user approve) |
| PD-07 | Recommend WCAG 2.2 AA vs A with rationale | UX Engineer |
| PD-08 | Propose lightweight design system (tokens, 8 core components) | Product Designer + UI Engineer |
| UXE-01 | Propose minimal state coverage model for MVP (loading/empty/error/normal states) | UX Engineer |
| UXE-04 | Propose simple accessibility architecture (keyboard nav, ARIA landmarks, focus) | UX Engineer |
| UXE-06 | Design navigation hierarchy (dashboard landing + side status box + nav structure) | UX Engineer |
| UXE-08 | Propose interaction paths for 3 workflows | UX Engineer (after PD-05) |
| UXE-09 | Specify CSS design tokens (typography, spacing, colors in cyberpunk style) | UX Engineer |
| PD-03-theme | Theme selected: BreachLab | PD, UXE, UI approved design proposals |
