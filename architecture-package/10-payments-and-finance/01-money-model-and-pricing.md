# 01. Money model and pricing

Code: `packages/core/src/domain/models/{pricing,ledger,partnership,onboarding,event-catalog}.ts`, `application/pricing/pricing-service.ts`, `application/finance/finance-service.ts`.

## Units and rounding
Everything is **integer paise** (`assertPaise` requires a safe integer ≥ 0). Guest-price math uses half-up rounding: `applyPercent(paise, pct) = Math.round(paise * round(pct*10) / 1000)`. Partner split uses `Math.round` per leg with the **host as the remainder**; promoter commission rounds **down**. Currency is hard-coded `INR`; a mixed-currency cart is refused.

## What the guest pays (`pricing.ts`, verified constants)
| Constant | Value |
|---|---|
| `PLATFORM_FEE_PERCENT` | 5 |
| `PAYMENT_FEE_PERCENT` | 2.5 |
| `GST_PERCENT` | 18 |
Order of operations:
```
subtotal     = sum(unitPrice x qty)
discount     = promo ? min(max(0, raw), eligibleSubtotal) : 0
               percent: applyPercent(eligible, value)   fixed: value
               (empty tierIds = all tiers, scoped promos discount only the named tiers)
discounted   = max(0, subtotal - discount)
platformFee  = applyPercent(discounted, 5)
paymentFee   = applyPercent(discounted, 2.5)
gst          = applyPercent(platformFee + paymentFee, 18)     // on the FEES only, NOT on the ticket price
grandTotal   = discounted + platformFee + paymentFee + gst
```
`assertReconciles` runs on every calculation. Worked example: a ₹1,500 ticket gives platform fee 75.00, payment fee 37.50, GST 20.25, **total ₹1,632.75**.
- **The 18% is a constant, not data.** The payment audit lists this as open item 1: pricing must read dated, approved tax rules, but the tax-rule store (#127 part 2) is **not on staging**. Changing it changes every price, so it needs the CA's decision first.
- GST on the ticket price itself is **not** charged; **TCS/TDS are absent from code** (only mentioned as open CA questions in `payments-release-gate.md`).

## How a line is priced (`PricingService.calculate`)
Loads the event's tiers. Per line it checks: tier exists and `status === 'active'`; `salesStartAt`/`salesEndAt` window; `minPerOrder`; `maxPerUser` against the quantity; `tier.maxPerOrder`. **Pricing phases:** the phase where `startsAt <= now < endsAt` overrides `priceInPaise`; otherwise the base price (phases are not otherwise validated for overlap at this point). Promo: `getPromoByCode`, then `assertPromoUsable` (active flag and date window); percent and fixed discounts as above; a promo with a discount must have a `discountFundingOwner` (`platform|venue|host|promoter|shared`).
Other rules at quote time (`checkout-service.ts`): the event must be `published`; `maxRedemptions` and `maxPerUser` for the promo are checked (counting redemptions by promo id **and** by code, to cover history); a total strictly between 0 and 100 paise is refused ("Online payments must be at least ₹1"); a total of exactly 0 is allowed (free). `effectiveTierPricePaise` falls back to the legacy `doorPriceInPaise` for old tiers (used by RSVP).
**Door sales are priced differently:** `tier.priceInPaise × quantity`, no fees, no GST, no phases, no promos, cash only (`door-ticket-sale-service.ts`).

## What partners get (settlement split)
Base for the split is **net ticket revenue** = `order.discountedSubtotalPaise` (guest fees and GST are not in the base). `computeSettlementSplit` (`ledger.ts`):
```
platformFee = round(gross x feeRate)
venueShare  = round(gross x venueRate)
promoter    = frozen promoterCommissionPaise  or  round(gross x promoterRate)  (0 when none)
host        = gross - platformFee - venueShare - promoter
```
A guard throws if `promoter > gross - platformFee - venueShare` (`finance-service.ts` ≈L127).
| Leg | Source of the rate |
|---|---|
| **Platform fee (partner side)** | the organisation's onboarding plan: **basic 15%, silver 12%, diamond 10%** (`platformFeePercentFor(plan)`, `onboarding.ts`). There is no per-organisation override field. `PLAN_TIER_PLATFORM_FEE_RATE` in `ledger.ts` also has a `gold` 0.12 that no plan produces; `DEFAULT_PLATFORM_SETTINGS.platformFeeRate = 0.15` exists but is **not read here** |
| **Venue share** | `partnership.venueShareRate` (0–50%, whole number) found by `partnerships.findByPair(host, event.venueId)`; **null means 0%** (fail-safe); with no venue or no partnership everything goes to the host |
| **Promoter commission** | the commission snapshotted on the order's attribution at quote time (`commissionForLines`: per line `floor(line.subtotal × ratePercent / 100) + flatPaise × qty`, using `terms.tierRates[tierId]` when present); otherwise the volume tier below |
| **Host** | the remainder |
Promoter volume tiers (`commissionTierFor`, ticketsSold = sum of the promoter's link conversions, limit 1000 links):
| Tickets sold | Rate |
|---|---|
| 0 | 10% |
| 10 | 12% |
| 25 | 15% |
| 50 | 18% |
| 100 | 20% |
Example (illustration only): net ₹1,500, plan basic (15%), venue share 20%, promoter 10%: platform ₹225, venue ₹300, promoter ₹150, host ₹825.

### Two things called "platform fee"
1. **Guest fee:** `platformFeePaise` = 5% of the discounted subtotal, charged **on top** of the ticket (plus 2.5% payment fee and GST on both).
2. **Partner commission:** the `platform_fee` ledger leg = 15/12/10% of net ticket revenue, **deducted from partners**.
They are separate money. Several older docs treat them as one.

## Quote-time promoter guard (weak)
The quote rejects when commission > `grandTotal - platformFeePaise`, but settlement allocates on the **discounted subtotal**. The guard is weaker than the real constraint, so `recordTicketSale` can throw later (after the guest has paid) if commission exceeds distributable revenue. Fix: compare against the same base the split uses.

## Accounting policy (platform settings)
`DEFAULT_ACCOUNTING_POLICY`: version `m2-test-policy-v1`, `settlementDelayDays = 3`, `reserveBasisPoints = 0`, `requiresEventCompletion = true`, `holdsOnCancellation = true`, `holdsOnDispute = true`, `holdsOnUnresolvedReconciliation = true`. The version name says "test policy"; it is snapshotted into every journal and used in balance classification. It needs business sign-off. (Editing it has no authority tier: see `07-admin-console/10-…`.)
