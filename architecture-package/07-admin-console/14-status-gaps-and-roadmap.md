# 14. Status, gaps and roadmap (candid)

Snapshot 2026-10-24. **Keep under NDA.** Items marked (verified) were checked in source in this session; (reported) come from a code read and were not independently re-checked.

## Branch and PR state
- **Backend:** checked-out `fix/partner-connections-and-events` at `4bcc15a`; 6 ahead / 9 behind `origin/staging`. Staging-only commits include #137 (staff revoke/reinvite), #114, #138 (offline-scan keys), #139–#142, #143 is open. The M2 financial engine (#136, merged) is on this checkout (`78982fc`, merged by `d991299`): refund execute, reconciliation, chargebacks, financial adjustment, journal transition, payouts gate (#125 merged).
- **Frontend:** `fix/partner-connections-contracts` at `c1f4d54`; 0 ahead / 1 behind `origin/staging`. Merged: FE #79 (M2 refund contracts), #74 (admin tax-rule/reconciliation contract mirror), #41–#48 admin fixes (audit null role, orders empty email, health token proxy, secure-cookie prefix, KYC document review, console deploy verification). Local branch `feat/mirror-admin-finance-contracts` is stale (3 ahead / 4 behind, duplicate-export residue).
- **Open backend PRs touching this area:** #127 (provider reconciliation record, tax rules), #129 (tax-rule routes and a single-order reconciliation endpoint; **not** superseded by M2, which has no tax rules and no single-order endpoint, verified in the payments review), #131 (refund/chargeback decisions doc, 36 commits behind staging), #119 (settlement legs), #116 (payments design docs), #110, #109. Closed: #130, #133, #134.
- Counts and PR states move; re-run `git rev-list --left-right --count origin/staging...HEAD` before quoting.

## Security and authorization defects
| # | Sev | Finding | Evidence |
|---|---|---|---|
| S1 | High | **Platform settings update has no tier gate**; any active admin (incl. `support`) can change refund thresholds (set to 0 = every refund auto-approved), maintenance mode, flags, accounting policy | `admin-ops-service.ts:271-276` calls only `requireAdmin` (verified); `PLATFORM_SETTINGS_UPDATE` is not an `AdminAction` |
| S2 | High | **Approved proposals can be re-executed** repeatedly; no executed state, no expiry; executor unconstrained | `ProposalStatus = pending|approved|rejected|cancelled`, `isExecutable = status==='approved'` (verified) |
| S3 | High | **Ban, maintenance mode, `platformFeeRate` are not enforced** anywhere; org/venue suspension only affects public discovery, connections and partnership | repo-wide grep (verified for ban and maintenance mode) |
| S4 | Med | `POST /admin/payouts/:id/complete` is **not behind `requirePayoutsEnabled()`** | `payouts.ts` gate at L124 and L175 only; `complete` at L221 (verified) |
| S5 | Med | Console has **no admin-role awareness and no middleware**: a signed-in non-admin sees the full shell; support-role admins see forbidden buttons | no `proxy.ts`/`middleware.ts`; role schema `guest|partner|admin` (verified) |
| S6 | Med | **Forced password change unhandled** in the console | no `mustChangePassword`/change-password code (verified) |
| S7 | Med | Omnibox lookup returns user emails to any admin (bypasses redaction) | reported |
| S8 | Med | All reads open to every role incl. KYC document read URLs (unaudited) | reported |
| S9 | Med | Zero-approver refunds: one admin can request and execute alone; settings (S1) can force this | reported |
| S10 | Low | `/api/health/*` BFF routes unauthenticated, no origin check (exposes dependency status, version, build SHA) | reported |
| S11 | Low | No CSP, no HSTS in `next.config.ts` headers | reported |
| S12 | Low | `readUrl` navigated without scheme validation (trusted gateway) | reported |
| S13 | Low | Token held in memory per tab; logout in one tab does not invalidate others until refresh/401 | reported |
| S14 | Med | `revokeAdmin` single-actor, no reason, no last-super guard; seed roster check reads only the first page | reported |
| S15 | Low | Reconciliation retry checks existence before authority (404 vs 403 leak) and stores a non-standard 409 body | reported |

## Audit completeness
Proposal approve/reject/cancel drop IP/user agent (verified: `onboarding-review.ts` L410-418 closures take three arguments); no reason on suspend/reinstate/ban/pause/resume/force-complete; no-ops unaudited; settings audit incomplete; `executeRefund` failure paths unaudited (reported); document read-URL and 8 of 10 CSV exports unaudited; audit write non-atomic and `append` swallows duplicate ids; audit read has no cursor and is capped at 200; the console does not display before/after payloads.

## Workflow and UX defects (frontend, reported unless noted)
1. No confirm on high-impact single clicks: event force-complete (irreversible), pause/resume, refund approve and execute, payout Run, reconciliation Run, support delete/merge/close/escalate, maintenance toggle, unban, reinstates.
2. `withIntentKeys` is dead code; each click mints a fresh idempotency key; "safe to retry" copy is misleading (verified: exported, never applied).
3. No pagination or sort anywhere; hard cap 100 (200 audit); dashboard refund value and admin count wrong above 100; orders/tickets status filters client-side over the first 100 rows; Firestore silently caps CSV "1,000/10,000" at 100.
4. No UI to **create a refund request**, to raise `ADMIN_PROVISION`, or to execute `FINANCIAL_ADJUSTMENT` from Proposals (paste-id workaround); provision failures silent; no organisation picker on Adjustments or commission.
5. Generic error messages hide gateway reasons (role refusal, self-approval, dual-control) on all desks except onboarding/KYC.
6. Lookup and tables truncate ids to 12 characters; audit before/after not shown; audit/ support filters fire per keystroke.
7. Unit mismatch: Commissions percent (0–100) vs Settings fraction (0–1).
8. Payout proof entry is manual hash/type/size.
9. Accessibility: colour-only status, no focus management in inline confirms, no focus trap on mobile nav, no `aria-live` for success.
10. Dead code: `withIntentKeys`, `requestRefund`, `ADMIN_AUDIT_ACTIONS`, `NAV_ITEMS`; `playwright.prod.config.ts` references a CI job (`verify-admin-console-deploy`) and script (`wait-for-vercel-deploy.mjs`) that do not exist.

## Backend functional defects
Proposal-execution semantics (S2); reconciliation resolve-then-rediscover and full-collection in-memory scans with N+1 lookups; **support list filters applied after pagination**; support `assign` takes caller-supplied identity; offset pagination O(offset) and the 100-row cap; the last-super guard reads one page of admins; settings accept inconsistent thresholds and default policy `'m2-test-policy-v1'`; dispute upheld writes only the legacy ledger; no admin action to fail a payout, cancel an approved refund or act on a chargeback; no route for journal backfill/cutover; safety reports and alerts have no routes; approve of onboarding not atomic; provider chargeback fee/loss policy undefined; no email to the requester on support replies.

## Open policy and legal decisions (not code)
Funds-flow structure (RBI payment-aggregator question), CA treatment (ticket GST, TDS 194-O, GST TCS s.52; guest-fee GST is hard-coded 18% in `pricing.ts`), refund and chargeback loss rules (decision doc PR #131), Razorpay written approval for payouts. These block turning `PAYOUTS_ENABLED` on in production.

## Stale documentation (verified or strongly indicated)
- `docs/operations/payment-reconciliation.md` ("no HTTP endpoint", "refund executor not built") and `payment-audit-2026-10.md` open item 3 ("refunds never settle"): **stale**; `executeRefund` and `/admin/refunds/:id/execute`, `/admin/reconciliation/*`, `/admin/chargebacks` exist.
- `refund-and-chargeback-decisions.md` says refunds stop at "approved on purpose" (outdated), and exists only on `origin/docs/refund-chargeback-decisions`.
- `docs/admin-dashboard/02` draws a `proposed` state (code: `pending`); README tier diagram omits `PAYOUT_COMPLETE`, `RECONCILIATION_*`, `FINANCIAL_ADJUSTMENT`; README §7 "If-Match on versioned PATCH/PUT" (admin has none); docs say 15 admin route files + 14 tests (now 17 + 15), 11 support routes (now 15), refunds 5 routes (now 7), "176" or "186" parity checks (226 with scanner), "11 nav items" (24).
- `phase-07-admin-console.md` ("Phase C partial"; `v2_admins` vs `v2_platform_admins` naming; local Windows path) and `phase-06-…` (predates M2); `admin-ops-service.ts` header mentions a removed `suspendVenueFromProposal`; idempotency doc `07` does not mention the FAILED state (task B1).
- The integration audit's H1–H4, M1, M5 concern the partner dashboard and guest portal, not this console.

## Roadmap suggestions (ordered by risk)
1. Gate settings update (TIER3 dual control or at least TIER2), validate threshold ordering and add `If-Match`.
2. Add `executed` status/`executedAt`/`executedBy` and an expiry to proposals; make execution single-use.
3. Enforce or remove: user ban (auth + checkout), maintenance mode, org/venue suspension in checkout and partner writes, `platformFeeRate`.
4. Put `payouts/:id/complete` behind the release gate; add a payout-fail and a refund-cancel path.
5. Console: expose the admin role in the session, gate nav/actions by role, add a not-an-admin screen, handle `mustChangePassword`, add confirms and a reason field on destructive actions, add pagination, show audit before/after.
6. Audit hardening: forward `meta` for approve/reject/cancel, audit document read-URLs and all exports, make audit atomic with the mutation or outbox-based.
7. Fix support list filtering (server-side, with indexes), verify assignee is an admin.
8. Reconciliation: scheduled runs, cross-run de-duplication, concurrency guard, targeted retry.
9. Resolve the policy decisions in `06-…` and `07-…`, then reconsider enabling payouts.
