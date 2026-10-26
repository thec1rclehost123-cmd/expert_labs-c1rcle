# 07. Flow: payouts, financial adjustments and commissions

Code: FE `app/payouts`, `app/adjustments`, `app/commissions`; BE `routes/v2/admin/{payouts,reconciliation,organization-actions}.ts`, `lib/payouts-gate.ts`, `domain/models/payout.ts`, `application/finance/{admin-payout-service,payout-service,financial-posting-service}.ts`.

## Payout FSM (`v2_payouts`)
`requested | processing | paid | failed | frozen`.
- `beginProcessing` only from `requested`.
- `markPayoutPaid` only from `processing`; requires transfer reference, method, valid timestamp, sha256 proof hash and proof metadata, and `approvedBy !== requestedBy` (the requester is a partner user, so this never constrains an admin).
- `markPayoutFailed` from `requested` or `processing`. **No admin route fails a payout** (a `postPayoutFailureReversal` posting exists).
- `freezePayout` is a no-op if already frozen; refused for `paid`/`failed`; stores `previousStatus`. `releasePayout` only from `frozen`, restoring exactly `previousStatus`.
- Payout `amount` in paise; partner request minimum ₹100. Bank account numbers are stored AES-256-GCM encrypted (AAD bound to the organisation); routes see only the masked value.

## Admin routes
| Route | Authority | Gate | Notes |
|---|---|---|---|
| `GET /admin/payouts`, `/admin/payouts/:id` | any admin | none | no CSV export; `:id` has no FE caller |
| `POST /admin/payouts/batch-run` | `PAYOUT_BATCH_RUN` (TIER2) | `requirePayoutsEnabled()` | body `runPayoutBatchSchema {payoutIds[1..200]}`; each payout must be `requested` **and** have a `payout_reservation` journal, else skipped (reported, not errored); saves `processing` one at a time. Console "Run" sends one id |
| `POST /admin/payouts/:id/complete` | `PAYOUT_COMPLETE` (TIER2) | **none** | body `completeManualPayoutSchema` (strict): `transferReference ≤120`, `transferredAt` ISO, `transferMethod bank_transfer\|neft\|rtgs\|imps\|upi`, `proof {sha256 hex64, contentType, sizeBytes}`. Marks paid, posts the completion journal **before** saving the payout (retry relies on journal idempotency), audits. The proof is metadata only (nothing verifies a file exists); console collects it by hand |
| `POST /admin/proposals/:id/freeze-payout` | `PAYOUT_FREEZE` (TIER3, dual control) | none (allowed even when the gate is off) | |
| `POST /admin/proposals/:id/release-payout` | `PAYOUT_RELEASE` (TIER3) | `requirePayoutsEnabled()` | |

## Payouts release gate (`lib/payouts-gate.ts`)
`PAYOUTS_ENABLED ?? NODE_ENV !== 'production'`; `render.yaml` sets `false`. When off, the gate returns 503 `server` **before the handler** (partner payout request, admin batch-run, release). Freeze, all reads, checkout and fulfilment stay available. **`complete` is not gated** (lines 124 and 175 of `payouts.ts` are the only gate uses; `complete` is at line 221), so with payouts "off" an admin can still mark an existing `processing` payout as paid and post its journal. `docs/operations/payments-release-gate.md` lists only three gated routes, consistent with the code. Prerequisites to enable in production (from the gate doc): lawyer-confirmed funds flow (RBI payment-aggregator question open), CA tax treatment (GST, TDS 194-O, TCS s.52), Razorpay written approval.

## Manual financial adjustment (`FINANCIAL_ADJUSTMENT`, TIER3)
1. `/adjustments` raises a proposal with payload `{organizationId, amountPaise, direction credit|debit}` + reason (organisation id typed by hand).
2. A different `super` approves on `/proposals`.
3. The operator pastes the approved proposal id into the Post form → `POST /admin/proposals/:id/post-financial-adjustment` (`reconciliation.ts` L71, route-inline authority): requires the proposal to be `FINANCIAL_ADJUSTMENT` and approved, requires `journalTransition.readModel === 'financial_journal_v2'` (refused while legacy reads are active), validates the payload with zod **here** (not at propose time), then `FinancialPostingService.postManualAdjustment` posts a balanced journal (`manual_adjustment_control` against `partner_payable`, source `{type:'manual_adjustment', id: proposalId}`, idempotency key `m2:manual-adjustment:<proposalId>`), audits `FINANCIAL_ADJUSTMENT` (target `financial_journal`), returns `financialAdjustmentResultSchema`.
4. Weakness: the proposal has no executed state, so re-posting is harmless for the journal (idempotent) but writes another audit row, and the Proposals page has **no executor** for this action.
A discrepancy can carry an `adjustmentPayload` that is meant to feed this flow (see `08-…`).

## Commission adjustment (`COMMISSION_ADJUST`, TIER3)
`/commissions` lists organisations with live `platformFeePercent` (integer 0–100; 0 is legal, so use `??` not `||`) and raises a proposal with `{organizationId, platformFeePercent}`; after a second `super` approves, `/proposals` shows "Adjust commission" → `POST /admin/proposals/:id/adjust-commission` (`adjustPlatformFeePercent`, no-op when unchanged). Note: `PlatformSettings.platformFeeRate` (0–1 fraction in Settings) is **never read**; checkout and onboarding use `platformFeePercentFor(plan)` and the per-organisation percent. Two different units and only one is live.

## Financial journal transition (`docs/operations/financial-journal-transition.md`)
The legacy settlement ledger is the read model until a deliberate cutover to `financial_journal_v2`; the app reads one model or the other and never sums both. Steps: dry-run preview; resumable backfill (journal keys derived from the legacy ledger-entry id); comparison with per-organisation differences persisted; repair and re-run; cut over only at zero discrepancies; rollback is a read-model switch, not a data reversal. **No HTTP route** runs backfill/compare/cutover/rollback, and the doc says not to run it against staging or production until the Milestone 2 approval gate authorises it.

## Fulfilment repair (operations)
`pnpm --filter api-gateway repair:fulfilment [-- --apply] [--hours 48 --settle-minutes 10 --max 500]` finishes already-`paid` orders whose multi-step fulfilment stopped part-way; needs real Firestore credentials; refuses the memory driver; intended to run on a Render cron that is not configured. The script is in the gateway's `package.json` and on staging (the work landed via #135; PR #130 itself was closed). Full detail: `../10-payments-and-finance/07-reconciliation-repair-scheduling.md`.
