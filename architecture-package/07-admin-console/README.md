# 07. Admin Console: in-depth documentation

**Frontend:** `C1RCLE-FRONTEND/apps/admin-console` (`@c1rcle/app-admin-console`, port 3002)
**Backend:** `C1RCLE-BACKEND/apps/api-gateway/src/routes/v2/admin/*` (17 route files) + `packages/core` (`application/admin`, `application/finance`, `application/onboarding`, `application/support`)
**Snapshot:** 2026-10-24. Backend checkout `fix/partner-connections-and-events` (6 ahead / 9 behind `origin/staging`); frontend checkout `fix/partner-connections-contracts` (1 behind `origin/staging`). The M2 financial engine (refund execute, reconciliation, chargebacks, financial adjustment, journal transition, payouts gate) is on the backend checkout.

The admin console is the **internal platform-operations UI**: approve partners (onboarding and KYC), run money desks (refunds, disputes, chargebacks, payouts, reconciliation, adjustments, commissions), manage directories (venues, events, hosts, users), run the support desk, and read the audit trail. It is a **thin client**: every call goes to `/api/v2/admin/*`; the console holds no Firestore access and no policy logic.

## Reading order
**Prefer pictures? Start with `00-visual-guide.md` (18 flowcharts, decision trees, enforcement map and a defect map). Open `visual-guide.html` in a browser for a rendered version (needs internet for the diagram library).**

| File | What you get |
|---|---|
| `00-visual-guide.md` / `visual-guide.html` | 18 diagrams: system map, navigation, roles and tiers, gateway decision flow, dual control, onboarding review, refunds, disputes vs chargebacks, payouts and release gate, adjustments, reconciliation, support lifecycle, what each action really does, settings effects, sign-in gaps, audit coverage, defect map, fix order |
| `01-product-model-and-roles.md` | Principles, roles, tiers, desks and navigation |
| `02-frontend-architecture.md` | Source layout, auth BFF, session, API client, idempotency, error handling |
| `03-desks-and-screens.md` | Every page: endpoints, filters, mutations, confirmation UX |
| `04-authority-dual-control-audit.md` | Authority model, tiers, proposals, audit trail, separation of duties |
| `05-flow-onboarding-kyc.md` | Application review, document verification, approval and provisioning |
| `06-flow-refunds-chargebacks-disputes.md` | Refund FSM and execute semantics, disputes, provider chargebacks |
| `07-flow-payouts-and-adjustments.md` | Payout FSM, release gate, freeze/release, financial adjustments, commissions |
| `08-flow-reconciliation.md` | Runs, discrepancy kinds, retry/resolve, journal transition |
| `09-flow-support-desk.md` | Ticket model, FSM, SLA, actions, guest intake |
| `10-flow-directory-and-platform-actions.md` | Suspend/ban/pause, lookup, analytics, CSV exports, platform settings |
| `11-backend-api-reference.md` | Endpoint tables, cross-cutting machinery, FE↔BE map |
| `12-domain-model-and-state-machines.md` | Aggregates and FSM diagrams |
| `13-sequence-flows.md` | Mermaid sequence diagrams |
| `14-status-gaps-and-roadmap.md` | Candid defect list (security and workflow), stale docs, branch/PR state |
| `15-testing-build-deploy.md` | Tests, build, deploy, env var names |

## Executive summary
- **Live and fixture-free:** all 27 pages read live from the gateway; there is no fixture data in app code (unlike the partner dashboard).
- **The console does not know who the admin is.** The browser session user has role `guest|partner|admin`; the five admin roles (`super|admin|ops|finance|support`) appear only in the admin roster. No nav item, page or button is gated by role, there is no middleware, and a signed-in non-admin sees the full shell and discovers refusals only from failed calls. Enforcement is entirely at the gateway.
- **Forced password rotation is not handled.** The gateway returns 403 on every non-auth route while `mustChangePassword` is set, but the console has no change-password page or BFF route; an admin created with a temporary password would be stuck.
- **Authority model (backend):** tiers by consequence: TIER1 any active admin; TIER2 `super|admin|ops|finance`; TIER3 `super` only **plus dual control** (a different `super` approves; the execute route reads the stored payload). Admin identity lives in `v2_admins`, never inferred from org roles.
- **Top backend findings (verified in source 2026-10-24):**
  - `PUT /admin/settings/platform` has **no tier gate** (any active admin, including `support`, can change refund thresholds, maintenance mode, feature flags, accounting policy).
  - Approved proposals have **no executed state and no expiry**; an approved freeze/release/role-update/commission/adjustment can be executed again.
  - **User bans, maintenance mode and `platformFeeRate` are stored but not enforced** anywhere outside admin code.
  - `POST /admin/payouts/:id/complete` is **not behind the payouts release gate**.
  - Audit gaps: proposal approve/reject/cancel drop IP/user-agent; most CSV exports and KYC document-URL minting are unaudited; suspend/ban/pause actions record no reason.
  - Lists and CSV exports are **silently capped at 100 rows** on Firestore.
- **Money desks work end to end** (M2): refund request → tiered approval → explicit execute against Razorpay (convergent, idempotent), reconciliation runs and discrepancies, chargeback listing, manual financial adjustment via dual control. Open **policy** decisions (who bears refund/chargeback loss and provider fees, tax on refunds) and the payouts release gate still block production money movement.

## Conventions
Status labels: **Live**, **Partial**, **Unmerged**, **Planned**. "Verified" means checked in source on 2026-10-24; "reported" means taken from a code read and not independently re-checked. Only environment variable **names** appear; no secret values. Paths are repository-relative.

## Existing repo docs to read alongside (and which are stale)
FE `docs/admin-console/ADMIN-CONSOLE-PLAN-2026-09-11.md` and `ADMIN-CONSOLE-GAP-CLOSURE-2026-09-11.md` (says 11 nav items; the console has 24). BE `docs/admin-dashboard/01..07-*.md`, `docs/roadmap/ADMIN-DASHBOARD-GAPS.md`, `docs/roadmap/phase-07-*.md`. **Stale versus code:** `docs/operations/payment-reconciliation.md` ("no refund executor", "no HTTP endpoint"), `docs/operations/payment-audit-2026-10.md` open item 3 ("refunds never settle"), the dual-control doc's `proposed` state (code uses `pending`), "If-Match on versioned PATCH/PUT" (admin has none), route-file counts. Details in `14-…`.
