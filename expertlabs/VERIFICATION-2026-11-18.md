# Verification report, 2026-11-18

Everything in this package was checked against the **current** `staging` of both source repositories (backend `3802889`, frontend `8532a73`) and live GitHub state. This file says what was checked, how, and what is still unverified. Nothing here was assumed: each verdict has the evidence it rests on.

## 1. Summary

| Area | Method | Result |
|---|---|---|
| File paths named in the package | Every backticked repo path resolved against both source trees (202 distinct paths) | 185 exist; the 17 that do not are triaged in section 2 (all documented absences, globs or false positives, 1 real typo, fixed) |
| API routes named in the package | Every backticked route matched against the registered-route list captured from the running gateway (838 mentions) | 825 match; 13 do not, triaged in section 2 (frontend routes, provider API paths, removed routes the text says are gone) |
| Collection names | Every `v2_*` name matched against the 95 collections found in the Firestore repositories (222 mentions) | 220 match; 2 are a documented naming discrepancy and a truncated token |
| Env var names | Every name in `ENV-MATRIX.md` found in code (109) | all 109 are read somewhere in the code |
| Generated reference files | Recomputed from code and diffed (`ENV-MATRIX`, `DATA-MODEL-FIELDS`, `CONTRACT-SCHEMAS`, `SERVICES-INDEX`, `ERROR-CODES`, `ADR-INDEX`) | each now ends with a "Changes since this file was generated" section: 20 new env vars, 69 new domain types and 77 interfaces with changed fields, 131 new contract schemas, 37 services not previously indexed, 13 error codes (one more than listed: `event_not_on_sale`) |
| Route and collection indexes | Regenerated from the running app and the repositories | exact (405 routes, 95 collections, 88 composite indexes) |
| Pull request statuses | Every PR number cited, against GitHub for both repositories | many have changed; see `13-reference/PR-LEDGER-2026-11-18.md` |
| Known-issues / defect register | Targeted code probes for each row that can be tested by reading code | section 3 |

## 2. Non-resolving references: verdicts

| Reference (page) | Verdict |
|---|---|
| `apps/partner-dashboard/server.js` (06/14) | False positive: produced by the Next.js standalone build, not committed |
| `/admins`, `/admins/{id}/revoke`, `/admin/admins` (07/03, 07/11) | Shorthand for the admin authority routes; the registered paths are under `/admin/...` (see `API-ROUTE-INDEX.md`) |
| `v2_platform_admins` (07/14) | Documented naming discrepancy: the real collection is `v2_admins` |
| `docs/admin-dashboard/0*`, `docs/frontend-backend-handoff.md`, `docs/security.md`, `docs/payments-design`, `docs/operations/refund-and-chargeback-decisions.md` | Documents the text says are stale, absent or only on an unmerged branch; correct as stated |
| `/profile/:userId`, `/guests` (08/01, 09/02) | Frontend routes, not gateway routes |
| `/venues/by-id/:venueId`, `/menu` | Shorthand: the registered paths are `/public/venues/by-id/:venueId` and `GET/PUT /venues/:venueId/menu` |
| `/door/stats/ws` (09/07, 09/13) | The text says it is gone; the gateway serves `GET /door/stats/stream` (SSE) |
| `/payments/{id}/capture`, `/payments/{id}/refund` (10/05) | Razorpay API paths, not gateway routes |
| `/admin/finance/reconciliation/:orderId` (10/09) | Listed as "not on staging" in an unmerged PR; still true |
| `apps/scanner-app/src/offline/` (09/08) | PR #82 (frontend) is still open and the directory still does not exist on staging; correct as stated |
| `apps/api-gateway/src/plugins/auth.firestore.integration.test.ts` (04, 13) | The file does not exist (the PR that would have added it is closed); the text describes the gap correctly |
| `scripts/backfill-analytics.ts` (13/DEEP-DIVE-OTP) | **Fixed**: the real path is `apps/api-gateway/src/scripts/backfill-analytics.ts` |
| `v2_provider_` | Truncated token in prose |

## 3. Defect register: re-verification

Verdicts: **still true**, **fixed**, **changed**, **not re-verified**.

| Claim | Verdict | Evidence |
|---|---|---|
| `POST /admin/payouts/:payoutId/complete` bypasses the payout gate | **still true** | route has no `requirePayoutsEnabled`; the gate is on `batch-run`, `release-payout` and org payout creation (`routes/v2/admin/payouts.ts`) |
| Support DTO leaks internal notes to the guest | **fixed** | `toSupportTicketDto` returns `internalNotes: []` and redacts timeline detail and actor (`routes/v2/support/intake-routes.ts`) |
| Invitation accept has no email check | **fixed** | `acceptInvitationForRecipient` requires the verified recipient whose email equals the invitation's (`domain/models/organization.ts`) |
| Approved admin proposals can be re-executed | **likely fixed, not re-tested** | a `status !== 'pending'` guard exists (`application/admin/admin-ops-service.ts`) |
| Users who are banned stay signed in / bans unenforced | **changed** | bans are enforced at session creation and on every request (`plugins/auth.ts`); `maintenanceMode` is still only a stored setting with no enforcement code |
| Admin settings update has no authority tier | **still true** (weak evidence) | `routes/v2/admin/settings.ts` has no tier or proposal logic, only a rate limit |
| `platformFeeRate` not enforced | **not re-verified** | checkout derives the fee from the plan and a fee-policy snapshot (`checkout-service.ts`); the admin setting's effect was not traced |
| Guest portal: no CSP or HSTS | **changed** | a Content-Security-Policy in Report-Only mode exists (`apps/guest-portal/next.config.ts`); no HSTS header in app config (the host may add one: not verified); event-page `noindex` not re-checked |
| Email verification is not enforced by the server | **changed** | checkout and RSVP require a verified guest phone (`requireVerifiedGuestPhone`) and check `emailVerified` for the buyer (`checkout-service.ts`) |
| Signup reveals whether an email is registered | **still true** | a duplicate email returns 409 "Email already registered." (`routes/v2/auth/index.ts`) |
| Events never reach `ended` | **not re-verified** | the state machine allows `started -> ended -> archived` (`domain/models/event.ts`); whether anything triggers it was not traced |
| Scanner: device registration needs `door.manage` but the app asks every staff member | **fixed** | `POST /door/devices` needs `door.floor`; listing and re-authorizing stay `door.manage` |
| Scanner: no way to complete the forced first-login password change | **fixed** | "Set your password" step in the scanner app; the gateway answers 403 `passwordChangeRequired` until done |
| Scanner: QR valid 90 to 120 seconds | **still true** | 30 s windows, two windows of drift accepted (`application/door/offline-ticket-crypto.ts`) |
| Edge: webhook shares the general rate zone | **still true** | `c1rcle_edge_general` 10 r/s per IP on all `/api/v2/`; no webhook location (`deploy/nginx/snippets/rate-limits.conf`) |
| Edge: a single origin secret (rotation causes an outage) | **still true** | entrypoint reads one `NGINX_ORIGIN_VERIFY_SECRET` (`deploy/docker/nginx-entrypoint.sh`) |
| Safety reports are built but never wired | **still true** | no safety routes are registered |
| Rate limiting is in memory | **still true** | `RATE_LIMIT_STORE` defaults to `memory` and the example blueprint sets it |
| `zod` high-severity advisory | **fixed** | lockfile resolves `zod@4.6.7` |
| `renovate.json` invalid | **fixed** | repaired on staging |
| Frontend `pnpm audit`: 10 critical and 24 high | **changed** | run 2026-11-18: 73 findings, 0 critical, 22 high, 39 moderate, 12 low |
| Backend `pnpm audit` | **changed** | CI 2026-11-17: 3 findings (1 low, 1 moderate, 1 high: picomatch ReDoS) |
| Staging backend stuck on an old deploy | **fixed** | Render is live on `3802889`, the same commit this package was verified against (2026-11-18) |
| Invite emails had no link or credentials | **fixed** | `PARTNER_DASHBOARD_URL` is declared in the blueprint and set on the service |
| Free hosting plan hibernates | **still true** | blueprint plan is free; a keep-alive Worker exists |

All other rows in `12-status-and-known-issues/README.md` (guest portal placeholders, partner dashboard fixtures, leaderboard idempotency, RSVP oversell, wallet reconciliation sign, walk-in party size, frontend CI details and others) were **not re-verified**; treat them as the 2026-10-24 to 2026-11-04 snapshot.

## 4. Still unverified

- The per-feature "Live / Partial / Unmerged / Planned" status of each product area, beyond the rows above.
- The narrative pages in folders 00 to 11 apart from the structural check in section 1 (paths, routes, collections, env names).
- Behaviour that needs a running system: end-to-end payments, webhook timing, offline scanning on a real handset, the CDN and DNS setup of any specific deployment.
- Legal and commercial items (licence, NDA, funds-flow decisions) are outside what code can show.
