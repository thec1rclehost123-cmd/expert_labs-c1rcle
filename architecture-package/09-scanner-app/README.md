# 09. Scanner App: in-depth documentation

**Frontend:** `C1RCLE-FRONTEND/apps/scanner-app` (`@c1rcle/app-scanner-app` v0.1.0, Expo)
**Backend:** `C1RCLE-BACKEND/apps/api-gateway/src/routes/v2/door/*`, `routes/v2/phase5-routes.ts`, `packages/core/src/application/{door,cover-wallet}`, `packages/contracts/src/contracts/phase5.ts`
**Snapshot:** 2026-10-24. Backend checkout `fix/partner-connections-and-events` (6 ahead / 9 behind `origin/staging`); frontend checkout `fix/partner-connections-contracts` (1 behind `origin/staging`). The offline backend work (PR #138, merged 2026-10-22) is on `origin/staging` but **not** on the backend checkout or `origin/main`. Frontend PR #81 (contract mirror) is merged; frontend PR #82 (offline engine) is **open**. Local remote-tracking refs can lag; PR states below come from `gh` on 2026-10-24.

The scanner app is the **door staff handset app**: staff sign in, pair the device, open a shift with an event door code, then scan guest QR codes, check guests in by hand, sell door tickets, log walk-ins and dine-ins, charge cover wallets, and watch live occupancy. It is the only client that runs at the venue door, so correctness at the door (no double entry, no free entry, no double charge) is the core promise.

## Reading order
**Prefer pictures? Start with `00-visual-guide.md` (diagrams of the shift lifecycle, scan decision, couple tickets, credentials, wallet charge and offline design). Open `visual-guide.html` in a browser for a rendered version (needs internet for the diagram library).**

| File | What you get |
|---|---|
| `00-visual-guide.md` / `visual-guide.html` | All the main flows as diagrams |
| `01-product-model-and-workflow.md` | Who uses it, event-day workflow, gates, devices, shifts, permissions by code type |
| `02-app-architecture.md` | Expo app layout, config, routing guard, auth state machine, storage, API client |
| `03-screens-and-scan-flow.md` | Every screen and the scan flow step by step |
| `04-credentials-sessions-devices.md` | Door codes, scanner sessions, devices, the three credentials, who may call what |
| `05-qr-and-admission.md` | QR formats, verification, atomic admission, ledger, override, staff-deny, couples |
| `06-door-sales-and-cover-wallet.md` | Door ticket sale, walk-in, dine-in, cover wallets |
| `07-stats-guests-and-realtime.md` | Guest roster, stats, SSE stream, heartbeat |
| `08-offline-design-and-status.md` | Offline today, the new derived-key design, status per branch |
| `09-backend-api-reference.md` | Endpoint tables, rate classes, error mapping, frontend function map |
| `10-domain-model-and-state-machines.md` | Aggregates and FSMs |
| `11-threat-model-and-security-findings.md` | Threat model plus new verified findings |
| `12-status-gaps-and-roadmap.md` | Defect list, stale docs, branches and PRs, fix order |
| `13-testing-build-deploy.md` | Tests, CI, build, env var names |

## Executive summary
- **Online scanning works end to end.** Login, device pairing, shift open, scan (single and couple tickets), manual lookup, staff-deny, override, guest roster, door sales, wallet charges and live stats are implemented against the real gateway. There are no fixtures.
- **Offline: the app still fails closed.** If the device is offline the scan screen shows "Scanner offline, entry denied until connectivity returns". The backend half of a new offline design (per-ticket derived keys, signed Ed25519 manifest, time-bounded sync) is merged on staging (#138). The frontend engine (#82) is open and **not wired into any screen**. Offline door scanning is a stated business requirement (owner decision 2026-10-18).
- **Three credentials, none sufficient alone:** staff login (7-day, memory only), device id (client-generated), scanner session token (12 h, from a door code). Most door routes need login **and** the scanner token; manager actions need login and a role permission only.
- **Admission is atomic.** One Firestore transaction (`claimAdmission`) spends a seat, so concurrent scans of the same ticket admit exactly once.
- **Top findings (verified in source 2026-10-24):**
  - A **bare entitlement id is accepted as a ticket credential** (online and in sync): anyone who knows a ticket id can present it without the rotating signature.
  - Rotating QR codes are valid for about **90–120 s**, not the documented 30 s.
  - **`POST /door/devices` needs `door.manage`**, but the app's pairing screen calls it for every staff member; a `member`-role scanner user cannot pair. Walk-in and dine-in need `ticket.override` too, while the app only checks the session flag.
  - Walk-in and dine-in price **ignores party size**; wallet **reconciliation reports a false mismatch** for any wallet with a debit.
  - The app has **no tests on this branch**, no EAS/native build pipeline, and logout does **not** revoke the server session.
- **Corrections to earlier pages:** PR #138 and PR #81 are merged (earlier pages said "Unmerged"); the app's stats SSE call sends no scanner token and that is **correct** (the backend stats routes do not use one).

## Conventions
Status labels: **Live**, **Partial**, **Unmerged**, **Planned**. "Verified" = checked in source or with `gh`/`git` on 2026-10-24; "reported" = from a code read, not independently re-checked. Only environment variable **names** appear. Paths are repository-relative.

## Existing repo docs to read alongside (several stale, see `12-…`)
Backend: `docs/scanner-app/` (SOTA blueprint, topics 01–07), `docs/api-contracts/scanner-app.md` (frozen contract, "the code wins"), `docs/architecture/scanner-threat-model.md`, `docs/roadmap/phase-05-door-scanner-cover-wallet.md`, and on staging only `docs/operations/offline-scanning.md`. Frontend: the offline module README on the PR #82 branch.
