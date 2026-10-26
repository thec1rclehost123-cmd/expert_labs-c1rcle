# 08. Overview, finance, analytics, support and notifications

## Overview (venue/host: Live; promoter: fixture)
- `lib/overview/load-overview-data.ts` (server, cookie auth) issues six reads: `GET /organizations/:id/analytics/overview`, hourly trends for today, 30-day daily trends (serves both the 1W and 1M ranges), 24-month monthly trends, the month calendar, and event cards (limit 4). View-models: `overview-view-model.ts`.
- Failure taxonomy (`OverviewLoadError` → `OverviewLoadFailureState`): `signed-out`, `no-organization`, `forbidden`, `api`; only `api` offers retry. It is handled in the page rather than `error.tsx` because Next strips server error messages in production.
- `network` and `activity` are passed as empty arrays. Event-card links are broken (no `/partner/<studio>` prefix; see `03-…`).
- Promoter overview stays on `fixturePartnerDataSource.getPromoterOverview()` with an in-code reason.

## Backend analytics (`partner/analytics.ts`, `application/analytics/*`)
| Endpoint | Query | Response |
|---|---|---|
| `GET /organizations/:org/analytics/overview` | none | `organizationOverviewDtoSchema` |
| `…/analytics/trends` | `from`, `to` (YYYY-MM-DD), `granularity` hour/day/month | `organizationTrendsDtoSchema` (dense buckets, max 400; bounds clamped or swapped) |
| `…/analytics/events` | `limit` 1–50 (default 10) | `organizationEventCardListResponseSchema` (soonest first; events already started omitted) |
| `…/analytics/calendar` | `month` YYYY-MM | `organizationCalendarDtoSchema` |
| `GET /events/:id/analytics` | none | `eventAnalyticsDtoSchema` (404 for a foreign event) |
All org routes: `organization.read` + `cached('ORGANIZATION')`; event route `event.read` + `cached('EVENT_PREVIEW')`.
- **Read model** `v2_analytics_reads`: written by a projection consumer on `event.{created,updated,published,cancelled}` and a 5-minute full sweep (revenue/check-ins move on writes that emit no event). Reads use the read model first, then a compute-on-request fallback (bounded scan, `COMPUTE_SCAN_LIMIT = 1000` per page). Trends, event cards and calendar are always computed per request.
- Gaps: `views`, `guestlistSignups`, `conversionRate` hard-coded 0 (no source aggregate); event `capacity` uses the **venue's** capacity (default 0), not `event.capacity`; `repeatGuests` = distinct entitled users; `totals` are lifetime for the scanned data, not the sum of the buckets. The 5-minute route cache sits on top of the projection, so partner data can be up to about **10 minutes stale** (and the cache has no invalidation on writes).
- Venue/host studio analytics pages and event analytics are skeletons; promoter analytics is fixture.

## Finance (all studios: Live, server-side)
Code: `lib/finance/load-finance-data.ts`, `api-finance-repository.ts`, `finance-view-model.ts` (the only place paise become rupees), `PayoutRequestAction`.
- `loadFinanceData` issues five concurrent reads (page size 50, no pagination UI): `GET /organizations/:id/finance/balance`, `…/finance/ledger`, `…/payouts`, `…/bank-accounts`, `…/finance/orders`. It **throws rather than falling back to fixtures**. Failure kinds `signed-out | no-organization | forbidden | api`, rendered in the page.
- Balance fields: available, pending, reserved, held (with reasons), paid, lifetime, plus `readModel`. Money is integer paise; INR/`₹` hard-coded.
- Bank accounts are masked only; **"Save bank details" is permanently disabled** ("requires the payout account mutation API"), although backend `POST /bank-accounts` exists (`finance.bank_accounts.manage`, owner/admin; no idempotency key; also `/:id/default`, `DELETE`).
- **Payout request:** client `POST /organizations/:id/payouts` with `Idempotency-Key` held per intent; minimum ₹100 (`MINIMUM_PAYOUT_PAISE = 10000`); validates 2-decimal rupees and `≤ available`; then `router.refresh()`. Backend: `finance.payouts.manage` (owner/admin) + `requirePayoutsEnabled()` (503 when `PAYOUTS_ENABLED` is false; defaults on outside production, **off in production unless set**). The request body also carries `idempotencyKey` (the payout command uses the body field, not the header). Not gated by `MANAGE_PAYOUTS` in the V3 UI.
- Backend finance routes: `GET finance/balance|ledger` (`finance.read`), `finance/orders` (`finance.orders.read`), payouts GET (`finance.payouts.read`), disputes (`organization.read` for reads; raise/review/resolve need `organization.update`), `GET /leaderboard` (public, no auth) and `…/leaderboard/me`. The dashboard does **not** call disputes or leaderboard.
- Settlement/ledger semantics (platform fee, venue share, promoter commission, host payout) are in `../10-payments-and-finance`. Open legal/tax decisions (hard-coded 18% GST on guest fees, funds-flow, refund and chargeback loss rules) apply here too. **Important for partners:** on the default read model a partner's available balance is always 0 (the legacy ledger never advances) and the journal cutover has no operator path, so payout requests cannot succeed in practice; see `../10-payments-and-finance/04-ledger-and-journal.md`.

## Support (Live via BFF)
`PartnerSupportScreen` + `lib/support/support-repository.ts` → `/api/bff/support/tickets*` → gateway `POST|GET /support/tickets`, `GET /support/tickets/:id`, `POST …/messages` (session-scoped, no org; `SENSITIVE_COMMAND` + `Idempotency-Key`). The subject is encoded as `[<Department> · <Subcategory>] <subject>` and parsed back out; 9 departments (`support-departments.ts`) each map to a gateway category. **`listPartnerTickets` swallows every error and returns `[]`** (`support-repository.ts` ~L97–108), so a failed load looks like an empty inbox. The support POST BFF routes have **no CSRF check and no `stripProtoKeys`**, and mint a fresh idempotency key per request (retries are not deduped). Promoter has no support tab. Admin handling is in `admin/support.ts` (assign, priority, reply, notes, link, resolve, merge, delete).

## Notifications
- **Backend inbox exists:** `GET /organizations/:orgId/notifications` (`unreadCount`), `PATCH …/read-all`, `PATCH …/:id/read`, `POST …/:id/actions` (approve/reject the linked resource as the acting org, then mark read; idempotent `notifications.action`). Producers: `promoter_connection.requested`, `partnership.requested` (to the other side), `event.published` (to the publishing org itself, `recipientType` always `venue`), `promoter_assignment.created`. No `slot_request.*` producer. List/markAllRead read **every** notification for the org and sort/slice in memory (unbounded).
- **Dashboard does not use it.** The topbar notifications and global search are **fixture** (`getPartnerShellInteractions`); promoter notifications are always an empty list. `/api/bff/notifications` derives notifications from slot-request statuses and has no UI caller. The frontend branch `feat/partner-notifications-inbox` (1 commit, `b7ebe48`, "restore live notification inbox (lost in merge)") is **Unmerged**.
- The guest follow-graph inbox (`/notifications/me*`, `v2_social_notifications`, `v2_follows`) is a different, session-scoped system.
