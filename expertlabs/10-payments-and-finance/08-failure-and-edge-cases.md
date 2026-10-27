# 08. Failure and edge cases

| Case | Behaviour | Where / test |
|---|---|---|
| **Double webhook** | `confirmPayment` finds the order by `paymentId` and re-runs `completeFulfilment` (ensure-steps, no duplicates); provider events are deduped by `providerFinancialEvents.ingest` | `checkout-service.ts:680-688`; `webhook-routes.test.ts:261,577` |
| **Webhook before redirect** | the webhook creates the order (system actor); the later redirect finds it, requires the same user, event and hold, and returns it with tickets | `payment-routes.test.ts:241,804` |
| **Redirect before webhook** | the redirect creates the order; the webhook re-runs fulfilment and records correlated provider facts including fee, so the provider-fee journal can post later | `webhook-routes.ts:212` |
| **Both race** | `VersionConflictError` on `orders.save`; the loser converges on the winner | `payment-routes.test.ts:784` |
| **Late payment after hold expiry** | verified first (signature, order binding, amount, currency); hold revived with a new sequence; seats free → order built; seats gone → full refund (`autorefund:{paymentId}`), hold released, 409 `payment_refunded`. A webhook redelivery repeats the refund call, which Razorpay should reject as already refunded (not safe by key) | `checkout-service.ts:899-928`; `payment-routes.test.ts:564-640` |
| **Capture with no order or hold** (missing notes, other product, payment link) | provider capture fact recorded (`requires_support`/`processing`), webhook answers 200 with an ids-only error log; reconciliation raises `captured_payment_without_order`; **no automatic refund or alert** | `webhook-routes.ts:184-196` |
| **Capture bound to a different order id** | rejected "Payment was not made against this hold"; non-2xx so Razorpay retries | `checkout-service.ts:716-742` |
| **Crash mid-fulfilment** | retried by webhook redelivery or redirect; otherwise `repair:fulfilment` (manual) | `payment-routes.test.ts:452,488` |
| **Two payments against one hold** | the second is rejected ("belongs to a different hold") because the hold is converted; the second payment is **captured with no order and not refunded** | `checkout-service.ts:682-686` |
| **Refund of a partially scanned order** | allowed; raises approvers to at least 1; `ticket_lines` selection is the first N by id and can pick a scanned ticket; a full refund voids scanned tickets too | `refund-request.ts:109`; `refund-service.ts:112-137`; `order.test.ts:275` |
| **Refund provider timeout** | order stays `refund_requested`, request `executing` with no provider id; needs reconciliation; `ConflictError` | `refund-service.ts:324-329` |
| **Chargeback after payout** | hold journal at dispute open; if the payable was already paid out, exposure above the partner allocation goes to `chargeback_loss_receivable`; who bears it is open | `financial-posting-service.ts:107-245` |
| **Currency mismatch** | rejected "Payment currency X does not match hold currency Y"; the adapter's `createOrder` also rejects a response with a different amount or currency; a provider reporting no currency skips the check | `checkout-service.ts:748`; `payment-routes.test.ts:396` |
| **Amount below ₹1** | rejected at quote | `payment-audit.test.ts:352` |
| **Event unpublished after hold** | new payment attempt refused; an in-flight payment is still honoured or refunded | `payment-audit.test.ts:264` |
| **Webhook secret missing** | 503, nothing processed | `webhook-routes.ts:113` |
| **Dispute with no correlated order** | throws, event marked `support_required`, webhook returns an error (Razorpay retries) until support acts | `provider-payment-service.ts:123` |
| **Promo limit hit at capture** | atomic claim fails → hold released, full refund `promo-limit-refund:{paymentId}`, 409 | `checkout-service.ts` |
| **Hold sweep vs conversion** | a conversion between the sweep's read and its batch commit could be overwritten to `released` | `firestore-cart-reservation-repository.ts:141-160` |
| **Free ticket double tap** | converges on one order; second RSVP → 409 | `createRsvp` |
