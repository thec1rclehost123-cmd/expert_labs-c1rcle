# 06. Partner Dashboard: in-depth documentation

**Frontend:** `C1RCLE-FRONTEND/apps/partner-dashboard` (`@c1rcle/app-partner-dashboard`, port 3001)
**Backend:** `C1RCLE-BACKEND/apps/api-gateway` + `packages/core`, routes under `/api/v2`
**Snapshot:** 2026-10-24. Frontend branch `fix/partner-connections-contracts` (1 commit behind `origin/staging`, none ahead). Backend branch `fix/partner-connections-and-events` (6 ahead, 9 behind `origin/staging`). Branch counts move; re-check before relying on them.

The partner dashboard is the web app used by **venues, hosts (organisers) and promoters** to run events, calendars, partnerships, staff, finance and onboarding. It is one Next.js app with three "studios" (`/partner/venue`, `/partner/host`, `/partner/promoter`) sharing one shell.

## Reading order
**Prefer pictures? Start with `00-visual-guide.md` (19 flowcharts, decision trees, status maps and a defect map). Open `visual-guide.html` in a browser for a rendered version (needs internet for the diagram library).**

| File | What you get |
|---|---|
| `00-visual-guide.md` / `visual-guide.html` | 19 diagrams: system map, studios and navigation, live-vs-fixture map, sign-in flow, access derivation, security layers, request paths, onboarding, event publish, event lifecycle, slot requests, partnerships, promoter earnings, staff invites, finance, failure handling, roadmap, defect map |
| `01-product-model.md` | What each partner type can do, tabs, privacy rules, glossary |
| `02-frontend-architecture.md` | Source layout, providers, session, org resolution, network paths, proxy/CSP/CSRF, BFF handlers |
| `03-routes-and-screens.md` | Every route, live vs fixture, the data-source interface |
| `04-auth-rbac-and-access.md` | Login, roles, `/access`, permission matrices, gating, defects |
| `05-feature-onboarding-kyc.md` | Wizard, OTP, KYC upload, backend FSM, admin approval |
| `06-feature-calendar-slots-events.md` | Calendar, slot requests, event create/publish, event FSM |
| `07-feature-partners-staff-promoters.md` | Partnerships, promoter connections, referral links, staff |
| `08-feature-finance-overview-support.md` | Overview, finance, analytics, support, notifications |
| `09-backend-api-reference.md` | Endpoint tables and cross-cutting machinery |
| `10-domain-model-and-state-machines.md` | Aggregates and FSMs |
| `11-data-and-collections.md` | Firestore collections and indexes |
| `12-sequence-flows.md` | Mermaid sequence diagrams of the main flows |
| `13-status-gaps-and-roadmap.md` | Candid defect list, audits, branches, migration checklist |
| `14-testing-build-deploy.md` | Tests, build config, deployment, env var names |

## Executive summary
- **One supported UI:** the "V3" studio shell at `/partner/[studio]/…`. The older `/venue`, `/host`, `/promoter`, `/partner-network` trees are 307-redirected into it (`next.config.ts`). About 54 legacy pages and roughly 40k lines remain in the repo as dead code, and their tests still run.
- **Live against the real gateway:** auth, org list and per-org access, onboarding, venue and host overview, finance (all studios), venue calendar and host availability, slot requests, partners and promoter connections, staff and invitations, support tickets, the venue event publish workflow, the host "submit for approval" workflow.
- **Still fixtures (no env switch, decided by import):** events list, event detail and sub-tabs, event edit, door mode, marketing, settings, promoter overview/events/guests/analytics/leaderboard, topbar notifications and search.
- **Authorisation reality:** the gateway enforces role permissions on every call. The dashboard only hides tabs from a server-supplied `tabVisibility` map. The V3 UI does not use the per-action `canDo` gates (only dead legacy components do).
- **State:** no TanStack Query in this app. Data comes from server-component loaders, a same-origin BFF client, a direct-Bearer API client and hand-rolled hooks.
- **Biggest risks today** (details in `13-…`): promoter organisations receive empty permissions from `/access`; real events do not appear in the events list (fixtures) and event detail 404s for real ids; event edit is non-functional; event publish is a non-atomic multi-call workflow; events never leave `published` without an admin action; slot-request accept does not affect the calendar.

## What changed versus the first-pass overview
The earlier one-page `06-partner-dashboard/README.md` said legacy trees were fixture-backed and reachable, listed `canDo` as the gating mechanism, assumed TanStack Query, and described a 6-step onboarding. A source-level pass showed all four were wrong or stale; they are corrected here and in `04-auth-and-rbac/README.md` and `12-status-and-known-issues/README.md`.

## Conventions used in these docs
Status labels: **Live** (on `staging`/`main` and in use), **Partial**, **Unmerged** (open PR/branch only), **Planned**. File paths are repository-relative. "Verified" means checked in source on 2026-10-24; "reported" means taken from a code read and not independently re-checked. Only environment variable **names** appear; no secret values.

## Existing repo docs to read alongside
FE: `docs/partner-dashboard-backend-handoff.md`, `docs/partner-dashboard-role-integration.md`, `docs/frontend-backend-integration-audit.md`, `docs/superpowers/FRONTEND-GATEWAY-BACKEND-MAP.md`, `docs/superpowers/ONBOARDING-FLOW-SPEC-2026-09-01.md` (the first three carry dated 2026-10-02 status banners that are partly stale). BE: `docs/api-contracts/frontend-backend-matrix.md`, `docs/api-contracts/auth-and-permissions.md`, `docs/roadmap/phase-01-partner-dashboards.md` (stale on the admin→MANAGER mapping).
