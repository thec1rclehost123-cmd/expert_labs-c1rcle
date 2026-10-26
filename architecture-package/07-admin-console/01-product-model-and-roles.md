# 01. Product model, roles and navigation

## Design principles (from the plan docs, confirmed in code)
- **Thin console, thick gateway.** The Next.js app only calls `/api/v2/admin/*` via `@c1rcle/api-client`. Mistakes of the V1 console deliberately avoided: console stores that hit Firestore directly, split-brain authority between console and gateway, role claims inside a JWT, a dev-mode bypass.
- **Authority is not an org role.** A platform admin is a record in `v2_admins` keyed by the Better Auth user id. Every admin action runs `requireAdmin` → `authorize(action)` → `record` (audit). An org `owner` is not an admin.
- **Tier by consequence, not job title.** TIER1: any active admin (logged). TIER2: `super|admin|ops|finance`. TIER3: `super` only **and** dual control.
- **Deactivated admin = non-admin.** Both get 401 "Platform admin authority required" (no oracle about former admins). A role too low for a tier gets 403.
- **Frozen wire contract:** bare DTO for a single resource; lists `{items, pageInfo}`; flat errors `{code, message, status, requestId, fieldErrors?}`; money in integer paise; ISO timestamps (one exception: `AdminAuditRecord.occurredAt` is epoch ms); a request body never carries a `role`. The frontend contract copy is generated (`scripts/export-contracts.mjs`) and guarded by `contract-parity` (186 checks, 226 with the scanner).

## Roles (backend schema: five)
`super | admin | ops | finance | support`. V1's `content` and `readonly` were dropped on purpose.
| Role | Reads | Can act on |
|---|---|---|
| `super` | everything | TIER1 + TIER2; raise/approve/execute TIER3 (provision admin, role update, commission adjust, financial adjustment, payout freeze/release); revoke an admin (no self-revoke); sees real emails in the users CSV and audit target names |
| `admin`, `ops` | everything | TIER1 + TIER2 |
| `finance` | everything | TIER1 + TIER2; sees real emails (users CSV, audit names) |
| `support` | everything | TIER1 only: support desk, event pause/resume/force-complete, audit read |
`ops` and `finance` are indistinguishable at TIER2: any of `super|admin|ops|finance` can request, approve and execute refunds, run payout batches, resolve reconciliation and disputes, suspend venues/organisations/promoters and ban users. The only role-specific behaviour is email visibility (`super` and `finance`). There is **no per-domain role split and no read matrix** (deliberately not ported from V1). Because every read is `requireAdmin` only, `support` can read refunds, payouts, reconciliation evidence, KYC documents and user lookups.

## Action tiers (`packages/core/src/domain/models/admin-authority.ts`)
| Tier | Actions |
|---|---|
| **TIER1** (any admin) | `EVENT_PAUSE`, `EVENT_RESUME`, `EVENT_FORCE_PAUSE` (this is force-complete, not pause); all `SUPPORT_*` and `SAFETY_*` are audit strings only |
| **TIER2** (`super\|admin\|ops\|finance`) | `ONBOARDING_APPROVE` (also covers per-document verify/reject, reject, request-changes), `VENUE_SUSPEND/REINSTATE`, `ORGANIZATION_SUSPEND/REINSTATE`, `PROMOTER_SUSPEND/REINSTATE`, `FINANCIAL_REFUND`, `PAYOUT_BATCH_RUN`, `PAYOUT_COMPLETE`, `DISPUTE_RESOLVE`, `RECONCILIATION_RUN`, `RECONCILIATION_RESOLVE`, `USER_BAN`, `USER_UNBAN` |
| **TIER3** (`super` + dual control) | `ADMIN_PROVISION`, `ADMIN_ROLE_UPDATE`, `COMMISSION_ADJUST`, `FINANCIAL_ADJUSTMENT`, `PAYOUT_FREEZE`, `PAYOUT_RELEASE` |
Not tiered at all: platform settings update (see `14-…`, no authority check beyond "is an active admin"). The wire `adminActionSchema` omits `RECONCILIATION_RUN`, `RECONCILIATION_RESOLVE`, `PAYOUT_COMPLETE` (harmless: only TIER3 actions can be proposed).

## Departments, desks and navigation
`src/lib/app-meta.ts` defines 6 sections and 24 items:
- **Overview:** Overview, Analytics, Lookup.
- **Approvals:** Proposals, Onboarding, KYC review.
- **Commerce:** Orders, Tickets, Promotions, Promoters, Commissions.
- **Care:** Support, Refunds, Disputes, Chargebacks, Payouts, Reconciliation, Adjustments.
- **Directory:** Venues, Events, Users, Hosts.
- **Platform:** Admins, Audit, Health, Settings.
Business view: Platform Governance (login, admins, audit, proposals), Finance (commissions, refunds, payouts, disputes, chargebacks, reconciliation, adjustments, analytics), Operations (venues, events, hosts, users, orders, tickets, promotions, promoters, settings, onboarding, KYC), Customer Support (support desk; guest intake lives in the guest portal at `/help`).

## Explicitly deferred / out of scope (recorded decisions)
Fresh login on every entry, vision/CDN health nodes, content boosts and curation, invite-by-email for admins (role collision with the Better Auth admin plugin; only `sendOtpEmail` exists), ledger viewer, security dashboard, raw logs viewer, elevated-risk gate, announcements, `WEBHOOK_RETRY`, `ADMIN_ACCESS_REVOKE`, verification and warning actions, per-role read matrix, safety reports with moderation (service exists, no route). Partner "reprovision" is won't-build (unified org model).

## What each product surface is for
- **Dashboard:** five counters: pending proposals, pending refunds (count and capped value), submitted onboarding applications, active admins, plus static principle cards.
- **Onboarding vs KYC review:** Onboarding is the application decision desk (approve/reject/request changes); KYC review is a card view of the same queue (statuses `submitted` + `changes_requested`) for per-document verify/reject. No extra backend route.
- **Disputes vs Chargebacks:** Disputes are **partner-raised** challenges to ledger entries (resolve upheld/denied). Chargebacks are **provider (Razorpay) disputes** recorded from webhooks; the console shows them read-only.
- **Proposals:** the dual-control inbox for TIER3 actions; Adjustments, Commissions, Payouts (freeze/release) and Admins (role change) all raise proposals into it.
