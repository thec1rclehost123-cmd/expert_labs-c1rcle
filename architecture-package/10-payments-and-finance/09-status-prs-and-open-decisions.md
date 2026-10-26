# 09. Status, PRs and open decisions

## What is merged where (verified with `gh` and `git merge-base --is-ancestor`, 2026-10-24)
The payments and finance code on the backend checkout is **identical to `origin/staging`**. The checkout is 6 commits ahead of and 9 behind staging; none of the differences touch payments.
| PR | State | Outcome |
|---|---|---|
| #113 | merged 10-11 | fail-closed payments (`UnconfiguredPaymentProvider`), production secret hardening, Redis rate limiting |
| #115, #117, #118, #124, #126, #130, #133 | **closed, but landed in staging via #135** | real checkout signature and payment-to-hold binding, late-capture honoured or refunded, attendee on the hold, signature checked against the hold's stored order, webhook accepts empty-array notes and keeps payer data out of logs, resumable fulfilment and `repair:fulfilment`, the payment audit fixes (commits `6457988`, `2cc597f`, `4fa64e4`, `ad9e57f`, `86bf8f3`, `91269d7`, `cc74e4c`…) |
| #134 | closed | duplicate of #135 |
| **#135** | **merged 10-18** | "M1 test-mode checkout": the clean branch carrying the whole stack above |
| **#136** | **merged 10-18** | "M2 financial engine": journal v2, refund executor, disputes and chargebacks, reconciliation runs, journal transition |
| #125 | merged 10-18 | `PAYOUTS_ENABLED` release gate |
| #120 | merged into `fix/staging-checks` | bank-account re-encrypt migration |
| #128 | merged | `render.yaml` matches the live service |
| #127 | **open** | part 1 (provider reconciliation record) is on staging; **part 2, the tax-rules store, is not on staging or main** (grep finds only docs and a config comment) |
| #129 | open (stacked on #127) | `GET/POST /admin/tax-rules`, `GET /admin/finance/reconciliation/:orderId`; not on staging |
| #119 | open | settlement legs and GST as its own ledger line (DP-01); not on staging (`computeSettlementLegs` only in `dist`) |
| #121 | open | Firestore `bindProviderOrder` race test (tests only, stale) |
| #116 | open | the three payments design docs |
| #131 | open | refund and chargeback decisions doc (`docs/operations/refund-and-chargeback-decisions.md`) |
| #143 | open | nginx anonymised client IP |
Remote branches still present: `feat/settlement-legs` (6 ahead), `feat/ledger-refund-chargeback-fee` (11), `feat/admin-reconciliation-tax-routes` (13), `fix/payments-correctness` (3), `feat/checkout-attendee` (4).
**Correction to earlier pages:** this package previously listed #115, #124, #126, #130 and #133 as unmerged and said refunds "never settle". Both statements are wrong for staging. Frontend PR #72 (guest paid checkout) is **closed**; the same work (`a9e3061`, `034bdd8`, `dd71a17`) is on staging.

## Open decisions (people, not code)
`payments-release-gate.md` (all must be on file before `PAYOUTS_ENABLED=true`, recording who approved and when):
1. **Funds flow.** Collecting into THEC1RCLE's own account and paying partners later raises an RBI 2025 Payment Aggregator Directions question that is **unanswered**, as is whether Razorpay Route supports the model.
2. **Tax.** Ticket GST, GST on the convenience fee, TDS s.194-O, GST TCS s.52 (a CA decision). Today GST is charged on guest fees only, hard-coded 18%.
3. **Razorpay's written approval** of the configuration.
Refund and chargeback record (#131, seven decisions, each with a suggested default): (1) who funds a refund of a split order (reverse each leg pro rata); (2) a partner already paid out (net it against the next payout); (3) who bears Razorpay's fee on a refunded payment (the platform); (4) which tickets are refundable (never after scan; host-cancelled gets a full refund); (5) who bears a lost chargeback and the dispute fee (partner bears a lost dispute, platform the fee, payouts held while open); (6) tax on refunds (credit notes; CA only); (7) approval thresholds and whether sending the refund is a separate action.
Other: the hold length of 10 minutes (`DEFAULT_HOLD_TTL_MS`); accounting policy defaults are test values (`m2-test-policy-v1`, 3-day delay, 0% reserve); #119 notes "plan commission" is 0 for every plan and rates are not snapshotted onto the hold at purchase; the design docs (#116) flag Indian payout tax and Firestore per-document write limits as gaps.

## Defects and risks (verified unless marked reported)
| # | Sev | Finding |
|---|---|---|
| P1 | **High** | **Legacy ledger never advances** `pending` entries, so partner available balance is always 0 in the default read model |
| P2 | **High** | **Journal cutover, backfill, compare and rollback have no route or script** (code-callable only); partners cannot be paid until someone runs a cutover |
| P3 | High | A second payment against an already-converted hold is captured with no order and not refunded; only a manual reconciliation run catches it |
| P4 | Med | No `payment.failed` handling; no scheduled repair or reconciliation |
| P5 | Med | Webhook route has no rate limit and no replay window; adapter `fetch` calls have no timeout |
| P6 | Med | Quote-time promoter guard weaker than the settlement constraint: settlement can fail after the guest paid |
| P7 | Med | Placeholder buyer contact `guest@example.com` (and `RSVP Guest`) when no attendee is sent; the guest portal never sends one |
| P8 | Med | Events never reach `ended` without an admin force-complete, so the default `requiresEventCompletion` policy keeps partner money `pending` indefinitely even in the journal model |
| P9 | Med | `cleanupExpired` race (stale read, batch overwrite) and 500-op batch cap |
| P10 | Low–Med | Dispute event without a correlated order loops (webhook error) until support acts |
| P11 | Low | Bank accounts: `verified` never set, `Date.now()` ids, CBC comment; admin dispute "upheld" writes an ignored legacy refund entry |
| P12 | Low | `MemoryPaymentProvider` defaults the webhook secret to the public string `test_webhook_secret` (memory driver is refused in production) |
| P13 | Low | Referral attribution HMAC falls back to `'dev-secret'` when `magicTicketSecret` is empty; click tracking is anonymous with no dedupe |
| P14 | Info | Phase-06 and payment-reconciliation docs are stale; "platform fee" naming conflates two different fees |
Suggested order: advance or retire the legacy read model and give the journal cutover an operator path (P1, P2); close the orphan-capture gaps and add `payment.failed` plus scheduled repair/reconcile (P3, P4); harden the webhook and adapter (P5); fix the promoter guard and attendee capture (P6, P7); decide the open policies; only then consider turning payouts on.
