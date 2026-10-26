# 10. Payments and finance: in-depth documentation

**Backend:** `C1RCLE-BACKEND` (`packages/core/src/application/{checkout,pricing,inventory,finance}`, `domain/models/{pricing,order,cart-reservation,ledger,financial-journal,journal-transition,payout,provider-payment,reconciliation,bank-account}`, `apps/api-gateway/src/routes/v2/{checkout,finance,admin}`, `lib/payments/`).
**Frontend callers:** guest portal checkout (`08-guest-portal/`), partner finance screens (`06-partner-dashboard/08-…`), admin money desks (`07-admin-console/06-…` to `08-…`).
**Snapshot:** 2026-10-24. Backend checkout `fix/partner-connections-and-events` (6 ahead / 9 behind `origin/staging`). **The payments and finance code on this checkout is identical to staging** (a diff of checkout, payment, pricing, finance, ledger, journal, payout, refund, razorpay and order paths is empty). PR states below come from `gh`.

## What this covers
How a guest's money enters (checkout, Razorpay, webhook), how it is split (platform fee, venue share, promoter commission, host), where it is recorded (legacy partner ledger and journal v2), how partners get paid (manual payouts behind a release gate), how it goes back (refunds, chargebacks), and how mistakes are found (reconciliation and repair).

## Reading order
**Prefer pictures? Start with `00-visual-guide.md` (18 diagrams). Open `visual-guide.html` for a rendered version (needs internet for the diagram library).**
| File | What you get |
|---|---|
| `00-visual-guide.md` / `visual-guide.html` | Money flow, price calculation, checkout sequence, state machines, decision trees, ledger vs journal, failure cases, defect map |
| `01-money-model-and-pricing.md` | Constants, price formula, fees, GST, promo, commission tiers, settlement split, rounding |
| `02-checkout-and-payment-flow.md` | Quote, hold, payment intent, verify, confirm, webhook, RSVP, recovery |
| `03-order-hold-inventory.md` | Order and hold state machines, availability, oversell protection |
| `04-ledger-and-journal.md` | Legacy partner ledger, journal v2 (accounts, postings), balances, journal transition |
| `05-razorpay-and-provider-facts.md` | Adapter, modes, provider payments store, webhook ingestion, what is not implemented |
| `06-payouts-refunds-disputes.md` | Payout flow and gate, bank accounts, refunds, disputes, chargebacks (guest and partner side) |
| `07-reconciliation-repair-scheduling.md` | What runs automatically, scripts, reconciliation runs |
| `08-failure-and-edge-cases.md` | Case table: double webhook, late payment, orphan capture, partial scans, chargeback after payout |
| `09-status-prs-and-open-decisions.md` | What is merged where (corrected), open PRs, legal/tax/finance decisions, defects |
| `10-testing.md` | Tests that document behaviour, docs-vs-code mismatches |

## Executive summary
- **Live and tested on staging:** online paid checkout with server-side pricing, a 10-minute hold with oversell protection, Razorpay order binding and signature check, idempotent confirmation shared by the browser redirect and the webhook, late-capture revive-or-refund, resumable fulfilment, journal v2 postings, provider fact ingestion, admin refund executor, reconciliation runs, a payout release gate. The correctness PRs (#115, #117, #118, #124, #126, #130, #133) landed in staging through the merged clean branch **#135**; the financial engine is **#136**; the payout gate is **#125**. (Earlier pages of this package called these "Unmerged": that was wrong.)
- **Not on staging or main:** the tax-rule store (#127 part 2), tax and reconciliation admin routes (#129), settlement legs / GST as its own ledger line (#119), the design docs (#116) and the refund/chargeback decision record (#131).
- **Money cannot reach partners yet, by design and by gaps:** payouts are gated off in production; **and** on the default (legacy) read model a partner's available balance is always zero because nothing advances legacy ledger entries from `pending`; the only way out is the journal cutover, which has **no route or script**.
- **Two different "platform fees":** the guest pays 5% platform fee + 2.5% payment fee + 18% GST on those fees **on top** of the ticket; separately partners lose 15/12/10% (basic/silver/diamond) of net ticket revenue as the platform's commission. Docs often conflate them.
- **Tax is not implemented:** GST is charged on guest fees only (hard-coded 18%), not on the ticket price; TDS 194-O and GST TCS are absent; funds-flow structure (RBI payment aggregator question) is undecided.
- **Biggest risks (details in `09-…`):** a second payment against an already-converted hold is captured with no order and no refund; no handling of `payment.failed`; no scheduled repair or reconciliation (manual only); webhook route has no rate limit and the adapter's `fetch` calls have no timeout; promoter-quote guard weaker than the real constraint (settlement can fail after the guest paid); placeholder buyer contact `guest@example.com` when no attendee is sent.

## Conventions
Status labels: **Live** (on staging/main), **Partial**, **Unmerged** (open PR/branch only), **Planned**. "Verified" = checked in source or with `gh`/`git` on 2026-10-24; "reported" = from a code read, not independently re-checked. Money is always integer paise; examples show rupees for readability. Only variable **names** appear.
