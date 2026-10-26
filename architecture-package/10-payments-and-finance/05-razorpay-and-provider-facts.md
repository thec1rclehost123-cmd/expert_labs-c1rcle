# 05. Razorpay and provider facts

Code: `apps/api-gateway/src/lib/payments/razorpay-adapter.ts` (338 lines), `unconfigured-provider.ts`, `packages/core/src/domain/ports/payment-provider.ts` (`MemoryPaymentProvider`), `application/finance/provider-payment-service.ts`, `domain/models/provider-payment.ts`.

## Adapter methods
All use `fetch` against `https://api.razorpay.com/v1` (overridable base URL) with Basic auth. **No timeout or `AbortSignal` on any call**: a hung provider holds a checkout or webhook request open.
| Method | Behaviour |
|---|---|
| `createOrder` | `POST /orders` with `{amount, currency, receipt: idempotencyKey, notes: metadata}`; verifies the response amount and currency equal the request, else throws |
| `verifyPayment` | HMAC of `order_id|payment_id` under the key secret (`safeHexEqual`, length-safe), then `getPayment` and requires `captured` |
| `getPayment` | `GET /payments/{id}` → `{verified, paymentId, amountPaise, captured, orderId, currency}`; 404 → "Payment not found" |
| `refundPayment` | `POST /payments/{id}/refund` with `{amount, receipt: idempotencyKey}`; any non-OK response throws `DefiniteProviderFailureError`; a malformed success body throws `InvalidOperationError` |
| `capturePayment` | `POST /payments/{id}/capture` (unused in the flow) |
| `generateSignature`, `verifyWebhookSignature` | implemented, but the webhook route does its own HMAC |
Ids are validated by `assertProviderId` (`/^[A-Za-z0-9_]{1,64}$/`), which closed an SSRF and path-injection hole (PR #92) and does not echo the input.

## Test vs live: there is no switch
Mode is whatever keys are loaded (`RAZORPAY_KEY_ID`, `RAZORPAY_KEY_SECRET`, `RAZORPAY_WEBHOOK_SECRET`, all optional in the schema). Wiring (`v2-services.ts` ≈L359–372): `STORAGE_DRIVER=memory` → `MemoryPaymentProvider(webhookSecret ?? 'test_webhook_secret')` (a public default secret; production refuses the memory driver); Firestore with all three keys → `RazorpayPaymentProvider`; otherwise `UnconfiguredPaymentProvider` (every method throws a 503). Readiness reports payment health when any Razorpay variable is set; a boot warning `payments_not_configured` fires under Firestore when fewer than three are set.

## Not implemented
RazorpayX payouts, Razorpay Route or linked-account transfers; payment links and QR codes; subscriptions; fetching refunds, disputes or settlements from the API; a Razorpay-side idempotency header (`receipt` is a label, not a dedup key); handling of `payment.failed`; settlement-report reconciliation against Razorpay settlements.

## Provider payments store (`v2_provider_payments`)
One document per Razorpay payment id: `id (pay_…), providerOrderId, orderId, processingState requires_support|processing|fulfilled|refunded, currency, capturedPaise, feePaise, taxOnFeePaise, capturedAt, refunds {refundId: paise}, disputes {disputeId: {amountPaise, amountDeductedPaise, status, reasonCode, reasonDescription, respondBy, phase, evidenceState, providerOccurredAt}}`. Raw webhook bodies are **not stored**, only a hash and ids.
`ProviderPaymentService.recordEvent`: ingests a minimised `providerFinancialEvent` first (provider event id, type, payment id, order id, refund id, dispute id, payload hash, trust level), deduped; `applyEvent` is version-checked and retries up to 5 times (a replay writes nothing); on exception the event is marked `support_required` and the error rethrown; after fulfilment a `captured` event is re-processed so the order id can be correlated; state becomes `refunded` when the refunds sum ≥ `capturedPaise`.
Dispute events call `applyDisputeFinancialPolicy`: `open` or `under_review` posts a `chargeback` hold; `lost`, or `closed` with `amountDeducted > 0`, posts `settleChargebackLoss`; otherwise `releaseChargebackHold`. **A dispute needs a correlated order** or it throws `NotFoundError('order_for_provider_dispute')`, marks the event `support_required` and the webhook returns an error (a retry loop) until support acts.
`reconcileOrder` compares the order with the provider record (missing record, amount or currency mismatch, refund mismatch, open dispute, provider net). The single-order HTTP endpoint is in open PR #129; the M2 runs are in `07-…`.

## Security notes
Signature checks are constant-time; the webhook fails closed; the order binding uses the **server-stored** provider order id; amounts and currency are re-verified against the provider (a provider that reports no currency skips that one check); payer data is kept out of logs (PR #126). Weak points: no fetch timeouts, no webhook rate limit, no webhook replay window, and refund idempotency relies on `receipt` plus Razorpay rejecting a duplicate full refund (the code deliberately does not auto-retry an ambiguous partial refund, which is the safe choice).
