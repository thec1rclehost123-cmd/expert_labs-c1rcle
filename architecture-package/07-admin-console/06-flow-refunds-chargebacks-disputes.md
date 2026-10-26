# 06. Flow: refunds, chargebacks and disputes

Code: FE `app/refunds`, `app/disputes`, `app/chargebacks`; BE `routes/v2/admin/{refunds,disputes,reconciliation}.ts`, `packages/core/src/domain/models/refund-request.ts`, `application/finance/{refund-service,admin-dispute-service,provider-payment-service,financial-posting-service}.ts`. Status: **Live on the backend checkout (M2, PR #136 merged)**. The decision doc `refund-and-chargeback-decisions.md` (BE PR #131) is **Unmerged** and its statement that refunds stop at "approved" is outdated.

## Refunds
### Model (`AdminRefundRequest`)
`id, orderId, organizationId, amountPaise, requestedBy, reason, scope (amount_only|ticket_lines), ticketLines[{tierId, quantity}], targetEntitlementIds, approversRequired, approvals[{adminId, approvedAt}], status, rejectedBy, rejectionReason, providerRefundId, providerStatus, executionIdempotencyKey ("refund:<id>"), executionAttempts, failureReason` + versioned fields. Collection `v2_refund_requests` (indexes `status+createdAt desc`, `orderId+createdAt desc`).
### FSM
`pending | approved | executing | rejected | settled | failed`.
- `create` → `approved` if `approversRequired === 0`, else `pending`.
- `approve` only from `pending`; refused if actor is the requester or a repeat approver; reaching `approversRequired` → `approved`.
- `reject` only from `pending`, reason required.
- `beginRefundExecution` from `approved` or `failed` → `executing` (`executionAttempts+1`).
- `markRefundSettled` / `markRefundFailed` only from `executing`; `markRefundExecutionUncertain` keeps `executing` and records the reason.
- **No cancel for `approved`, and no reject for `approved`/auto-approved**: an approved request that is never executed leaves the order locked in `refund_requested`.
### Approval tiers (`approversRequiredFor`, thresholds read live from platform settings at request time)
Amount below `refundSingleApproverThresholdPaise` (default 50,000 paise) → 0 approvers (1 if any target entitlement is `redeemed`); below `refundDualApproverThresholdPaise` (default 500,000) → 1; at or above → 2. Because settings have no tier gate (see `04-…`), anyone able to edit settings can lower these to 0 and auto-approve every refund.
### Request (`POST /admin/refunds`, body `requestRefundSchema {orderId, amountPaise>0, reason 1–2000, ticketLines?[≤100]}`, 201)
`authorize FINANCIAL_REFUND` (TIER2); order must be `paid`; amount ≤ refundable balance; for ticket lines it freezes the target entitlements (lowest ids not `void`); locks the order (`refund_requested`); saves the request; freezes tickets; audits `FINANCIAL_REFUND.request`. The three saves are not atomic (execute re-converges them). Effectively one in-flight refund per order. **The console has no UI to create a refund request** (API function exists, unused). `refund_requested` still counts as sold for inventory until `refunded`.
### Execute (`POST /admin/refunds/:id/execute`, command `admin.refund.execute`)
Does: authorise `FINANCIAL_REFUND`; return the existing result if already `settled`; throw `ConflictError` if `executing` with no provider id; **claim execution by compare-and-set** (one winner); re-lock the order if needed; call `paymentProvider.refundPayment` with the deterministic key `refund:<id>`; on a definite provider failure mark `failed`, release the tickets and restore the order; on an ambiguous error mark uncertain, throw 409 and keep the order locked for webhook/reconciliation; persist the provider fact before side effects; record a provider event (`refund_processed`); post refund journals (`FinancialPostingService.postRefund`); apply the order refund (deduped by `appliedRefundRequestIds`); void entitlements (all on a full refund, selected ones for ticket lines); mark `settled`; emit `finance.refund.settled`; audit `FINANCIAL_REFUND.execute`.
Does **not**: notify the buyer; touch the legacy partner ledger; audit failure/uncertain paths; poll the provider for a non-`processed` status (it checks the stored provider refunds and otherwise returns 409 "awaiting authoritative completion"); check who executes relative to who requested or approved.
### Other routes
`POST …/approve`, `…/reject` (body `{reason}`), `GET /admin/refunds` (`status`), `GET /admin/refunds/:id` (`{request, order}`; no FE caller), `GET /admin/refunds/export.csv` (10 columns; asks for 1,000 rows, capped at 100; unaudited). The static `export.csv` route is registered before `/:refundRequestId`.
### Console behaviour
Pending rows: Approve (single click), Reject (reason required). `approved`/`failed`: Execute / "Retry provider refund". Approvals column `n/required`.

## Disputes (partner-raised; `v2_disputes`)
`Dispute {id, organizationId, orderId, ledgerEntryId|null, raisedBy, reason, amount, status, resolutionNote, resolvedAt, resolution (upheld|denied)|null}`. FSM `open → under_review → resolved`; the admin resolve only blocks `resolved`, so it can go straight from `open`; the admin desk never sets `under_review`.
`POST /admin/disputes/:id/resolve` (`adminResolveDisputeSchema {outcome, resolutionNote 1–2000}`, `DISPUTE_RESOLVE` TIER2). **Upheld** writes a legacy `v2_partner_ledger` entry (`entryType:'refund'`, `status:'settled'`, key `dispute-resolve-<id>`); it does **not** touch journal v2, the order or any provider. **Denied** writes nothing. The ledger write precedes the dispute save; no check that the dispute amount ≤ the order amount. Partner side: owner/admin can raise **and** resolve their own dispute through partner routes (`organization.update`). No CSV export. `GET /admin/disputes/:id` exists (no FE caller).

## Chargebacks (provider disputes)
Razorpay events (`payment.dispute.*`) land in `v2_provider_payments.disputes` keyed by Razorpay ids (replays change nothing; a late "dispute opened" cannot undo a final outcome). `ProviderPaymentService` posts `chargeback_hold`, `chargeback_loss` and `chargeback_reversal` journals. `GET /admin/chargebacks` (in `reconciliation.ts`, `requireAdmin`) lists them with an N+1 order fetch per dispute; it is **read-only**: no admin action, no evidence submission.

## Open policy decisions (blocked on finance/legal, not code)
From the decision doc (PR #131): who funds a refund of a split order; what happens if the partner was already paid; who bears Razorpay's fee on a refunded payment; which tickets are refundable (scanned, windows, host-cancelled, partial); who bears a lost dispute and its fee, and whether payouts are held during an open dispute; tax on refunds (credit notes, TDS/TCS; the tax-rule scaffold has no rules enabled); whether approved refunds send automatically or by an explicit step (code chose the explicit execute step). Suggested defaults in that doc: proportional reversal netted against future payouts, platform bears the provider fee, separate send action. Next engineering steps named there: leg reversal as new ledger entries (never edits), payout holds for open disputes, credit notes/tax adjustments, reconciliation against provider refund and dispute records.
