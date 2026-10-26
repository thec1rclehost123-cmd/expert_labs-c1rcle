# 06. Door sales and cover wallets

Code: BE `routes/v2/door/{door-ops-routes,door-sale-routes,cover-wallet-routes}.ts`, `application/door/{door-service,door-ticket-sale-service,door-ops-service}.ts`, `application/cover-wallet/cover-wallet-service.ts`, `domain/models/{door-sale,cover-wallet}.ts`; FE `app/(tabs)/door.tsx`, `app/wallet.tsx`.

## Paid door ticket sale (`POST /door/ticket-sale`)
Needs a **`full`** session (`canDoorEntry`), `STANDARD_COMMAND`, `runIdempotent` keyed by `body.idempotencyKey` (command `door.ticket-sale`); a replay returns 200 with `replayed: true`.
- Only `paymentMode === 'cash'` is accepted; any other mode throws 400 ("Electronic door sales require provider or terminal confirmation").
- Order id `DOOR-` + sha256(idempotencyKey)[0:32]; an existing order replays (`checkInIds: []`); not event- or tier-bound.
- Checks that the tier exists for the event and is `active`, and that `inventory.getAvailableQuantity` covers the quantity. **This is a read check, not an atomic reservation**, so concurrent sales can oversell.
- Pricing is face value × quantity; no fees or GST; `assertReconciles` applies. The order walks `pending → awaiting_payment → paid` (three saves); tickets are issued via `issueEntitlements({order})` and `saveMany`.
- **`admitsPerUnit` is not passed**: a couple tier sold at the door gets `scanCountAllowed: 1` per unit, while online checkout builds it from tier metadata.
- Each ticket is admitted immediately via `claimAdmission` (one seat) and a `consumed` ledger row is written per ticket; admissions count as `prebooked`, not `doorEntries`, in stats. Then `settleOrder` posts to the finance ledger and an audit record `door.ticket_sale` is written.
- The steps are sequential and **not atomic**: a mid-way failure can leave a paid order with un-admitted tickets.

## Walk-in and dine-in (headcount entries, `v2_door_sales`)
Routes `POST /door/walk-in`, `POST /door/dine-in` (`ticket.override`, **no scanner token**, body idempotency key, commands `door.walkin`/`door.dinein`), `GET /door/sales` (`venue.read`). The request contract is strict: `tierId`, `quantity` and price are rejected with 422.
- Price is the event's walk-in or dine-in tier (`findWalkInTier`/`findDineInTier`, `where eventId == and entryType == 'walkin'|'dinein'`, limit 1): `amountPaise = tier.priceInPaise`. **It is not multiplied by `totalGuests`** (verified: `door-service.ts:136, 203`), although route and schema comments say party size is priced. No tier-status, inventory or event-time check.
- Idempotency: `runIdempotent` plus the service's `doorSales.findByIdempotencyKey(key)` which reads `v2_door_sale_idempotency/{rawKey}`: a **global doc id with no organisation or event scope**, so a colliding key from another tenant would return that tenant's sale. The Firestore create is a transaction that checks the pointer first.
- Optional wallet debit: `coverWallets.findByEventAndUser(event.id, actor.userId)` looks up the **operator's own** wallet for the event, not the guest's; if it exists and covers the amount it debits with key `${key}-debit`. This looks like a leftover or a bug.
- Fields: `category walkin|dinein, guestName, guestPhone, guestAge, gender, guestEmail, totalGuests, tableNumber, gate, paymentMode cash|card|upi|other, amountPaise, paymentStatus ('collected' hard-coded), paymentRef (always null), createdBy/createdByName (both the user id), status active|voided|refunded, void/refund fields, idempotencyKey, version`. Id `DS-<venueId|unknown>-<Date.now()>-<Math.random base36>` (length borderline against the 64-char cap with long venue ids). Card, UPI and other modes are accepted with **no payment verification**.
- `voidSale` and `refundSale` exist in the service but **have no HTTP route**. `listSales` fetches up to 1000 rows and filters in memory (the `pageInfo` is fake).

## Cover wallets
Collections `v2_cover_wallets`, `v2_cover_wallet_txns`, `v2_cover_wallet_idempotency`, `v2_cover_wallet_reconciliations`. None of the `/cover-wallets/*` routes use a scanner token (login + permission).
| Route | Permission | Notes |
|---|---|---|
| `POST /cover-wallets` | `door.manage` | issue a wallet for any `userId` string with any opening balance; **no user existence or payment check**; opening credit txn `wallet-open-<id>` |
| `GET /cover-wallets/:id` | `venue.read` | |
| `GET /cover-wallets/:id/qr` | `venue.read` | the service lets only the owning guest mint |
| `POST …/debit` | `ticket.override` | body `idempotencyKey` + `isOnline` (client-asserted; `false` → 400 `InvalidOperationError`) |
| `POST …/credit` | `door.manage` | arbitrary amounts; wallet must be active |
| `POST …/freeze`, `…/unfreeze`, `…/terminate`, `…/reconcile` | `door.manage` | header idempotency commands |
Not routed: refund, adjust, close, transactions, list, resolve-reconciliation, organisation stats.
- **Wallet:** `id CW-<hash>, userId, eventId, organizationId, venueId, balance, openingBalance, totalCredits, totalDebits, totalRefunds, status, terminatedAt, terminationReason, lastTxnAt, lastCreditAt, lastDebitAt, metadata, rules {presetItems[{id,label,amountPaise,isAvailable}], minChargePaise (1), maxChargePaise (5,000,000), showBalanceToGuest}`. Status `active|frozen|terminated|closed`: freeze only from active, unfreeze only from frozen (transactional); terminate and close are unconditional repo updates (the service blocks terminate on already terminated/closed). A debit that brings the balance to exactly 0 auto-terminates with `balance_depleted`, and a terminated wallet **cannot be credited**. `computeTerminationTime` (next day 05:00 IST) is computed but discarded, so there is no scheduled termination.
- **Txn:** id `txn-<walletId>-<Date.now()>` (two debits committing in the same millisecond on one wallet could overwrite each other), `type credit|debit|refund|adjustment`, `amount`, `balanceAfter`, `status pending|committed|failed|reversed`, plus references, device and operator fields. **Debit txns store a negative `amount`** (`firestore-cover-wallet-repository.ts:202`: `amount: -input.amount`).
- **`chargePreset`:** price = preset item price × quantity (1–20) within min/max; the caller names an item, never an amount. `POST /door/wallet-charge` needs a `charge` session and is wrapped in `runIdempotent` (command `door.wallet.charge`) because the inner check is read-then-write. Velocity: `countRecentDebits(deviceId, 1 min) >= 3` → 400, checked **before** the idempotency replay, so a legitimate retry of the 4th call in a minute is blocked. Insufficient balance in the repo throws a plain `Error` (500) if it races past the service pre-check.
- **Reconciliation bug (verified):** the service does `expectedBalance -= txn.amount` for debits (`cover-wallet-service.ts:820`), but debit amounts are stored negative, so subtracting **adds**: any wallet with a debit shows a false `balance_mismatch` of about twice its debits. The only route test covers the no-debit path. Also: one reconciliation per event/date (second → 409), up to 10,000 txns per wallet and 1,000 wallets, `getOrganizationStats` passes an organisation id where an event id is expected.
- Idempotency doc ids (`v2_cover_wallet_idempotency`, `v2_door_sale_idempotency`) are **global raw client keys**: a cross-tenant collision would return another tenant's data.

## App behaviour (`wallet.tsx`)
`canCharge === false` → "This shift has no charging permission"; camera permission gate; scan → `resolveWalletQr` (`POST /door/wallet-qr`, read-only; session meta re-read at scan and at charge time); the card shows first name, balance (null → "Balance hidden by venue", never 0) and status; items come from `presetItems` where `isAvailable`; there is **no free-amount input**; items are disabled if the price exceeds the balance or the wallet is not active; `quantity` is hard-coded 1; one UUID idempotency key per tap, refreshed after success or when abandoned; a failed charge keeps the same key so a retry replays safely; a local velocity gate of 3 charges per 60 s; the wallet is cleared after each charge (a fresh QR scan is needed every time).
