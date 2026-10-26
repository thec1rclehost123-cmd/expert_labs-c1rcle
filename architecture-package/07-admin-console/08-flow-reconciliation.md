# 08. Flow: reconciliation

Code: FE `app/reconciliation`; BE `routes/v2/admin/reconciliation.ts` (authority inline, no service delegation for the routes), `packages/core/src/application/finance/reconciliation-service.ts`, `domain/models/reconciliation.ts`, `provider-payment-service.ts`. Status: **Live on the backend checkout (M2)**. Backend PR #127 part 1 (the provider reconciliation record) is already on staging; #127 part 2 (tax-rules store) and #129 (tax-rule routes and a single-order reconciliation endpoint) are **open and not superseded** by M2.

## Provider facts store
`v2_provider_payments` records Razorpay's side of each payment (captured amount, fee, tax on the fee, refunds, disputes), keyed by Razorpay ids so replays change nothing. It is a **fact store**: it moves no balance and decides nobody's loss. `reconcileOrder(orderId)` compares an order with the provider record and reports: missing provider record, amount mismatch, currency mismatch, a provider-side refund the order lacks, an open dispute; it also returns the provider net (captured − fee − tax − refunds − disputes).

## Models
`ReconciliationRun {id, status running|completed|failed, requestedBy, startedAt, completedAt, scannedCounts, discrepancyCount, failureReason}`; `ReconciliationDiscrepancy {id, runId, kind, state, organizationId, orderId, paymentId, payoutId, amountPaise, currency, evidence, repair ('recheck_evidence'|null), resolutionReason, resolvedBy, resolvedAt}`. Collections `v2_reconciliation_runs`, `v2_reconciliation_discrepancies` (no declared composite index; equality filters `runId`/`state`/`organizationId` only).
**States:** `open | finance_review | support_required | resolved`. The only transition code is to `resolved` (no-op if already resolved); the other states are set at creation by `kind`.
**14 kinds:** `captured_payment_without_order`, `paid_order_without_capture`, `paid_order_without_ticket`, `ticket_without_paid_order`, `missing_journal`, `duplicate_journal`, `unbalanced_journal`, `amount_mismatch`, `currency_mismatch`, `provider_order_mismatch`, `refund_mismatch`, `chargeback_mismatch`, `stale_pending_payment` (30 minutes), `payout_mismatch`.

## Routes
| Route | Authority | Notes |
|---|---|---|
| `POST /admin/reconciliation/runs` (201) | `RECONCILIATION_RUN` (TIER2) | `run` loads orders, provider payments, entitlements and payouts **into memory** (200 per page, unbounded) with an N+1 journal lookup per order; on exception the run is marked failed and rethrown |
| `GET /admin/reconciliation/runs`, `/runs/:runId` | any admin | `:runId` has no FE caller |
| `GET /admin/reconciliation/discrepancies`, `/:id` | any admin | filters `runId`, `state` |
| `POST …/discrepancies/:id/retry` | TIER2 (checked **after** `getDiscrepancy`, so a low-role admin sees 404 vs 403) | only when `repair === 'recheck_evidence'`; otherwise 409 with a **non-standard** `{error:{code,message}}` body that is then stored for idempotent replay; re-runs the **entire** reconciliation and does not touch the targeted discrepancy |
| `POST …/discrepancies/:id/resolve` | `RECONCILIATION_RESOLVE` (TIER2) | body reason 1–2000; changes only the discrepancy (no repair, no journal, no order change); audit with before/after state |
| `GET /admin/reconciliation/export.csv` | `requireAdmin` | **no rate limit and no `validateV2` preHandler**; asks for 10,000 rows, capped at 100; unaudited; no FE caller |
| `GET /admin/chargebacks` | `requireAdmin` | see `06-…` |
| `POST /admin/proposals/:id/post-financial-adjustment` | TIER3 | see `07-…` |

Every discrepancy is created with `repair: 'recheck_evidence'`; the only allow-listed "repair" is re-running the scan.

## Behavioural weaknesses (verified or reported in source)
- **Resolve-then-rediscover:** runs scan whole collections with no cross-run de-duplication and no run-level concurrency guard; each run creates new discrepancy rows, so a `resolved` item reappears as `open` in the next run unless the underlying data is fixed.
- A discrepancy's `adjustmentPayload` is meant to raise a `FINANCIAL_ADJUSTMENT` proposal, but the console has no UI wiring between the two (Adjustments is manual).
- The console's state badge is always `warning` tone regardless of state; evidence shows the first four key/value pairs; "Run reconciliation" has no confirm.
- Reconciliation is manual only; there is no schedule. `docs/operations/payment-reconciliation.md` still says "no HTTP endpoint" (stale).
- Provider fee/chargeback loss policy is undefined (see `06-…`), so a discrepancy's financial remedy is a human decision today.
