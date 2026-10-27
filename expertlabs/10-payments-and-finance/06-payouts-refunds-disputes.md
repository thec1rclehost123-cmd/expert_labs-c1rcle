# 06. Payouts, refunds, disputes and chargebacks

Admin-side mechanics (approval tiers, execute, freeze/release, reconciliation UI) are documented in `../07-admin-console/06-…` and `07-…`. This page covers the partner and guest side and the money rules.

## Partner payouts (`finance-routes.ts`, `payout-service.ts`)
- `POST /organizations/:organizationId/payouts` (≈L163): `requirePayoutsEnabled()`, `STANDARD_COMMAND`, permission `finance.payouts.manage` (owner/admin). Idempotent: payout id `payout-{sha256(orgId:key)[0:32]}`; a replay with a different org, bank account, amount or requester → `IdempotencyConflictError`.
- Reads: `GET …/finance/balance|ledger|orders` (`finance.read`, `finance.orders.read`), `GET …/payouts[/:id]` (`finance.payouts.read`). Bank accounts: `POST|GET …/bank-accounts`, `POST …/:id/default`, `DELETE …/:id`.
- Rules: minimum ₹100 (`MINIMUM_PAYOUT_PAISE = 10_000`); amount ≤ `available + reserved + paid` (enforced atomically in `createWithinLimit`); reservation journal posted at request time; bank account must belong to the same organisation (else `ForbiddenError`).
- FSM: `requested → processing → paid`; `requested|processing → failed`; any non-terminal → `frozen` (records `previousStatus`); `frozen → previousStatus` (dual control).
- **Payouts are manual bank transfers recorded as evidence:** `markPayoutPaid` needs a transfer reference, method, valid timestamp, sha256 proof hash plus proof metadata, and `approvedBy ≠ requestedBy`; the completion journal credits `cash_bank_clearing` ("Manual bank transfer completed"). **Nothing calls a payout API.**
- **Release gate** (`lib/payouts-gate.ts`): `enabled = PAYOUTS_ENABLED ?? (NODE_ENV !== 'production')`; off → 503 "Payouts are not enabled on this deployment." before any service runs; `render.yaml` sets it false. Gated: partner payout request, admin batch-run, release-payout. **Not gated:** freeze, reads, and admin `complete`.
- A payout request on today's default read model cannot succeed in practice: available is always 0 (see `04-…`).

## Bank accounts
AES-256-GCM, scrypt-derived key (`ENCRYPTION_KEY`, optional `ENCRYPTION_SALT`), envelope `iv:tag:cipher`, with the organisation id bound as AAD. `last4` is plaintext; the route shows `••1234`; `getFullAccountNumber` is the only decrypt path (operator flow). Legacy fallback: records written before `ENCRYPTION_KEY` was wired decrypt with published constants; production refuses to boot without a real key; PR #120 re-encrypts existing rows. `verified` is **never set true** anywhere; ids use `Date.now()` (collision risk for two creates in one millisecond). The `bank-account.ts` comment still says CBC while the code is GCM.

## Refunds: who can start one
- **There is no guest or partner refund request API.** Request, approve, reject and execute are admin only (`routes/v2/admin/refunds.ts`).
- Guests get only **automatic full refunds** inside `confirmPayment`: late capture with the seats gone (`autorefund:{paymentId}`) and promo limit reached at capture (`promo-limit-refund:{paymentId}`). The buyer sees a 409 `payment_refunded` outcome; the guest portal shows the refund message and clears its recovery cookie.
- Partners can raise a **dispute** against an order; an admin resolves it (`upheld` writes a legacy `refund` ledger entry that no balance code reads and that moves no money; `denied` writes nothing).
- Admin refund essentials: statuses `pending | approved | executing | rejected | settled | failed`; approvers by amount under ₹500 zero, up to ₹5,000 one, above two (thresholds from platform settings; a scanned ticket forces at least one); partial refund of a partly scanned order is **allowed** (it only raises approvals) and a `ticket_lines` refund selects the first N non-void entitlements sorted by id, **not necessarily the unscanned ones**; a full refund voids every entitlement including scanned ones. Execute: claim, call provider with `refund:{id}`, record provider id/status; definite failure → request `failed`, tickets released, order restored; ambiguous (timeout) → stays `executing`, order locked, `ConflictError`; success → `refund_processed` event, three refund journals, order refund applied, entitlements voided, request `settled`.

## Chargebacks
Provider dispute events become journals: open → `chargeback` hold, lost (or closed with deduction) → loss, otherwise hold released. After a payout, exposure above the partner's allocation goes to `chargeback_loss_receivable`. **Who bears a lost dispute is an open decision** (#131 suggests the partner bears a lost dispute, the platform bears the fee, and payouts are held while a dispute is open); there is no automatic recovery from the partner and no blocking of an in-flight payout beyond the balance hold. The admin console lists chargebacks read-only.

## Guest-facing refund UX
None beyond the outcome messages above; an `order.refunded` status is not surfaced in the wallet. Refund policy (which tickets are refundable, host-cancelled events) is undecided (see `09-…`).
