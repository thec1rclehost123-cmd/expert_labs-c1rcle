# 07. Reconciliation, repair and scheduling

## What runs automatically (in-process timers, Firestore driver only, `app.ts` ≈L228–275)
| Job | Cadence | Notes |
|---|---|---|
| `cartReservations.cleanupExpired(now)` and `scannerSessions.cleanupExpired()` | every 5 min | per instance, no leader election; stale-read batch-write race described in `03-…` |
| `refreshAnalyticsReadModel` | every 5 min | |
| `drainPersistedOutbox` | every 1 s (`v2-services.ts` ≈L637) | at-least-once delivery to consumers |
Timers are `unref`ed and cleared on close. **No scheduled fulfilment repair, reconciliation run, payout batch or journal job exists.** `.github/workflows/maintenance.yml` is a daily 02:00 IST health smoke test only. `docs/operations/fulfilment-repair.md` says the Render cron "is not configured". The Cloudflare plan proposes an hourly Worker cron to trigger reconcile and repair (planned, not built).

## Fulfilment repair (`repair:fulfilment`)
`pnpm --filter api-gateway repair:fulfilment [-- --apply --hours 48 --settle-minutes 10 --max 500]` runs `FulfilmentRepairService.sweep`. Default is a dry run; `--apply` repairs; refuses the memory driver; exits 1 on any `error` or `still_incomplete` finding. It scans newest first, only **`paid` orders with a `paymentId`** created between `now − lookback` and `now − settle`. Findings: `found | repaired | still_incomplete | error`. Gaps it detects: `hold`, `hold_conversion`, `tickets`, `ledger` (no legacy entries or no journals for a non-zero order). An order whose provider order id has no bound hold is `still_incomplete`. It is idempotent and safe during a webhook, and it can **not** repair a captured payment that has no order at all.

## Reconciliation runs (M2, on staging)
`ReconciliationService.run` loads **all** orders, provider payments, entitlements and payouts into memory (200 per page, unbounded), stale pending age 30 minutes, 14 discrepancy kinds, states `open | finance_review | support_required | resolved`. Manual and admin-triggered only. It will not scale with order volume, resolves nothing by itself, and a resolved item reappears in the next run if the data is still wrong. UI and routes: `../07-admin-console/08-flow-reconciliation.md`.

## Coverage gaps (what nothing catches automatically)
| Situation | Caught by |
|---|---|
| Payment captured with no hold (missing notes, other product) | only a manual reconciliation run (`captured_payment_without_order`); no auto-refund, no alert |
| Second payment against an already-converted hold | only a manual reconciliation run; captured, not refunded |
| Stalled fulfilment | webhook redelivery, redirect retry, or a human running `repair:fulfilment` |
| `payment.failed` | not handled |
| Missing provider fee on a redirect-only fulfilment | next `postTicketSale` after the webhook |
Recommended: schedule `repair:fulfilment` and a reconciliation run (cron), alert on `requires_support`, handle `payment.failed`, and chunk the cleanup batches.
