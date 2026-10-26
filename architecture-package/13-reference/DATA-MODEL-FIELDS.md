# Data model: fields of every domain aggregate

**Generated** from `packages/core/src/domain/models/*.ts` on 2026-10-25 by scanning `export interface` blocks and `export type` unions. Field names, optionality and types are exact for the **domain objects**; the stored Firestore document normally has the same shape (timestamps are ISO strings, money is integer paise). Comments are the first line of the source doc comment. Nested object types are shown inline as written. For which collection stores what see `COLLECTIONS-INDEX.md`; for rules and state machines see the area docs. Regenerate when models change.


## `admin-authority.ts`

- **type `AdminRole`** = 'super' | 'admin' | 'ops' | 'finance' | 'support'
- **type `AuthorityTier`** = 1 | 2 | 3
- **type `ProposalStatus`** = 'pending' | 'approved' | 'rejected' | 'cancelled'

### `PlatformAdmin` (extends VersionedEntity)

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` | The Better Auth user id. |
| `email` | `string` |  |
| `role` | `AdminRole` |  |
| `isActive` | `boolean` | Revocation is a flag rather than a delete: audit records reference the |

### `CreatePlatformAdminInput`

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `email` | `string` |  |
| `role` | `AdminRole` |  |
| `now?` | `Date` |  |

### `ProposedAction` (extends VersionedEntity)

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `action` | `AdminAction` |  |
| `proposedBy` | `EntityId` |  |
| `reason` | `string` | Free-form justification, required — a TIER3 act must be explicable. |
| `payload` | `Record<string, unknown>` | The command payload the action will run with, frozen at proposal time. |
| `status` | `ProposalStatus` |  |
| `resolvedBy` | `EntityId \| null` |  |
| `resolvedAt` | `string \| null` |  |
| `resolutionReason` | `string \| null` |  |

### `ProposeActionInput`

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `action` | `AdminAction` |  |
| `proposedBy` | `EntityId` |  |
| `proposerRole` | `AdminRole` |  |
| `reason` | `string` |  |
| `payload?` | `Record<string, unknown>` |  |
| `now?` | `Date` |  |

### `ResolveActionInput`

| Field | Type | Note |
|---|---|---|
| `resolvedBy` | `EntityId` |  |
| `resolverRole` | `AdminRole` |  |
| `reason?` | `string` |  |
| `now?` | `Date` |  |

## `bank-account.ts`


### `BankAccount` (extends VersionedEntity)

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `organizationId` | `EntityId` |  |
| `bankName` | `string` |  |
| `accountHolder` | `string` |  |
| `last4` | `string` | Last 4 digits, plaintext (safe to display: `••1234`). |
| `encryptedAccountNumber` | `string` | AES-256-CBC envelope (`<ivHex>:<cipherHex>`) of the full account number. |
| `ifscCode` | `string` |  |
| `isDefault` | `boolean` |  |
| `verified` | `boolean` |  |

### `BankAccountCreateInput`

| Field | Type | Note |
|---|---|---|
| `organizationId` | `EntityId` |  |
| `bankName` | `string` |  |
| `accountHolder` | `string` |  |
| `last4` | `string` |  |
| `encryptedAccountNumber` | `string` |  |
| `ifscCode` | `string` |  |
| `isFirstAccount` | `boolean` | True if this is the org's first bank account (becomes default). |
| `now?` | `Date` |  |

## `cart-reservation.ts`

- **type `CartReservationStatus`** = 'active' | 'converted' | 'released'

### `CartReservation` (extends VersionedEntity)

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` | Deterministic id from idempotency key: `HOLD-{idempotencyKey}`. |
| `eventId` | `EntityId` |  |
| `organizationId` | `EntityId` |  |
| `userId` | `EntityId \| null` |  |
| `lines` | `{ tierId: EntityId; tierName: string; quantity: number; unitPricePaise: number; }[]` |  |
| `pricing` | `PricingBreakdown` |  |
| `appliedPromoCode` | `string \| null` |  |
| `attribution` | `OrderAttribution \| null` |  |
| `status` | `CartReservationStatus` |  |
| `expiresAt` | `string` | When this hold expires and inventory is returned. ISO-8601. |
| `convertedOrderId` | `EntityId \| null` | Set when converted to an order. |
| `providerOrderId?` | `string \| null` | The provider (Razorpay) order bound to this hold when the payment attempt |
| `attendee?` | `OrderContact \| null` | Who the tickets are for, captured at hold time; becomes the order contact. |
| `sequence?` | `number \| null` | Position in the event's strict order of hold creation, assigned by the |
| `idempotencyKey` | `string` | Idempotency key used to create this hold. |

### `CreateCartReservationInput`

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `eventId` | `EntityId` |  |
| `organizationId` | `EntityId` |  |
| `userId` | `EntityId \| null` |  |
| `lines` | `{ tierId: EntityId; tierName: string; quantity: number; unitPricePaise: number; }[]` |  |
| `pricing` | `PricingBreakdown` |  |
| `appliedPromoCode` | `string \| null` |  |
| `attribution` | `OrderAttribution \| null` |  |
| `reservationTtlMs?` | `number` | How long the hold lasts. v1 used ~10 minutes. |
| `idempotencyKey` | `string` |  |
| `now?` | `Date` |  |

## `cover-wallet-reconciliation.ts`

- **type `ReconciliationStatus`** = 'pending' | 'completed' | 'discrepancy' | 'resolved'
- **type `ReconciliationDiscrepancyType`** = | 'balance_mismatch' | 'missing_credit' | 'missing_debit' | 'extra_transaction' | 'duplicate_transaction' | 'offline_sync_gap'

### `CoverWalletReconciliation` (extends VersionedEntity)

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `eventId` | `EntityId` |  |
| `organizationId` | `EntityId` |  |
| `venueId` | `EntityId \| null` |  |
| `reconciliationDate` | `string` | Date being reconciled (YYYY-MM-DD in local timezone) |
| `walletId` | `EntityId \| null` | Wallet ID (if single wallet) or null for batch |
| `userId` | `EntityId \| null` | User ID (if single wallet) |
| `status` | `ReconciliationStatus` | Status |
| `expectedBalance` | `number` | Expected balance (computed) |
| `actualBalance` | `number` | Actual balance (from wallet doc) |
| `discrepancy` | `number` | Difference (actual - expected) |
| `periodCredits` | `number` | Total credits in period |
| `periodDebits` | `number` | Total debits in period |
| `periodRefunds` | `number` | Total refunds in period |
| `periodTxnCount` | `number` | Number of transactions in period |
| `discrepancies` | `CoverWalletReconciliationDiscrepancy[]` | Discrepancy details (if any) |
| `resolvedBy` | `EntityId \| null` | Resolved by |
| `resolvedAt` | `string \| null` | Resolution timestamp |
| `resolutionNotes` | `string \| null` | Resolution notes |

### `CoverWalletReconciliationDiscrepancy`

| Field | Type | Note |
|---|---|---|
| `type` | `ReconciliationDiscrepancyType` |  |
| `walletId` | `EntityId` |  |
| `userId` | `EntityId` |  |
| `expectedAmount` | `number` |  |
| `actualAmount` | `number` |  |
| `transactionId` | `EntityId \| null` |  |
| `description` | `string` |  |

### `CoverWalletReconciliationCreateInput`

| Field | Type | Note |
|---|---|---|
| `eventId` | `EntityId` |  |
| `organizationId` | `EntityId` |  |
| `venueId` | `EntityId \| null` |  |
| `reconciliationDate` | `string` |  |
| `walletId` | `EntityId \| null` |  |
| `userId` | `EntityId \| null` |  |
| `expectedBalance` | `number` |  |
| `actualBalance` | `number` |  |
| `periodCredits` | `number` |  |
| `periodDebits` | `number` |  |
| `periodRefunds` | `number` |  |
| `periodTxnCount` | `number` |  |
| `discrepancies` | `CoverWalletReconciliationDiscrepancy[]` |  |
| `now?` | `Date` |  |

## `cover-wallet.ts`

- **type `CoverWalletStatus`** = 'active' | 'frozen' | 'terminated' | 'closed'
- **type `CoverWalletTxnType`** = | 'credit' // top-up, referral bonus, promo | 'debit' // entry cover charge | 'refund' // refund of debit | 'adjustment'
- **type `CoverWalletTxnStatus`** = 'pending' | 'committed' | 'failed' | 'reversed'

### `CoverWalletPresetItem`

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `label` | `string` |  |
| `amountPaise` | `number` |  |
| `isAvailable` | `boolean` | Run out of a drink? Turn it off without editing the venue's price list. |

### `CoverWalletRules`

| Field | Type | Note |
|---|---|---|
| `presetItems` | `CoverWalletPresetItem[]` | The only things this wallet can be charged for. Empty = nothing. |
| `minChargePaise` | `number` | Belt-and-braces bounds on one charge, independent of the item list. |
| `maxChargePaise` | `number` |  |
| `showBalanceToGuest` | `boolean` | Whether the scanner may show the guest's balance on screen. |

### `CoverWallet` (extends VersionedEntity)

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `userId` | `EntityId` | User UID (guest or staff) |
| `eventId` | `EntityId` |  |
| `organizationId` | `EntityId` |  |
| `venueId` | `EntityId \| null` |  |
| `balance` | `number` | Current balance (integer paise) |
| `openingBalance` | `number` | Opening balance at activation |
| `totalCredits` | `number` | Total credits (paise) |
| `totalDebits` | `number` | Total debits (paise) |
| `totalRefunds` | `number` | Total refunds (paise) |
| `status` | `CoverWalletStatus` | Status |
| `terminatedAt` | `string \| null` | Termination timestamp (next day 05:00 local) |
| `terminationReason` | `string \| null` | Termination reason |
| `lastTxnAt` | `string \| null` | Last transaction timestamp |
| `lastCreditAt` | `string \| null` | Last credit timestamp |
| `lastDebitAt` | `string \| null` | Last debit timestamp |
| `metadata` | `Record<string, unknown>` | Metadata |
| `rules` | `CoverWalletRules` | What this wallet may be charged for, and within what bounds. |

### `CoverWalletTxn` (extends VersionedEntity)

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `walletId` | `EntityId` |  |
| `eventId` | `EntityId` |  |
| `organizationId` | `EntityId` |  |
| `venueId` | `EntityId \| null` |  |
| `userId` | `EntityId` |  |
| `type` | `CoverWalletTxnType` | Transaction type |
| `amount` | `number` | Amount in paise (positive for credit, negative for debit/refund) |
| `balanceAfter` | `number` | Balance after this transaction |
| `status` | `CoverWalletTxnStatus` | Status |
| `idempotencyKey` | `string` | Idempotency key (client-supplied, required for all mutations) |
| `referenceId` | `EntityId \| null` | Reference to source (doorSaleId, scanLedgerId, promoId, etc.) |
| `referenceType` | `string \| null` | Reference type |
| `deviceId` | `string \| null` | Device ID that initiated (for velocity limiting) |
| `operatorUid` | `EntityId \| null` | Operator UID (staff who processed) |
| `operatorName` | `string \| null` | Operator name |
| `description` | `string \| null` | Description |
| `failureReason` | `string \| null` | Failure reason (if failed) |
| `processedAt` | `string \| null` | Processed timestamp |

### `CoverWalletCreateInput`

| Field | Type | Note |
|---|---|---|
| `userId` | `EntityId` |  |
| `eventId` | `EntityId` |  |
| `organizationId` | `EntityId` |  |
| `venueId` | `EntityId \| null` |  |
| `openingBalance` | `number` |  |
| `metadata?` | `Record<string, unknown>` |  |
| `rules?` | `Partial<CoverWalletRules>` |  |
| `now?` | `Date` |  |

### `CoverWalletCreditInput`

| Field | Type | Note |
|---|---|---|
| `walletId` | `EntityId` |  |
| `amount` | `number; // paise, positive` |  |
| `referenceId` | `EntityId \| null` |  |
| `referenceType` | `string \| null` |  |
| `operatorUid` | `EntityId \| null` |  |
| `operatorName` | `string \| null` |  |
| `description` | `string \| null` |  |
| `idempotencyKey` | `string` |  |
| `now?` | `Date` |  |

### `CoverWalletDebitInput`

| Field | Type | Note |
|---|---|---|
| `walletId` | `EntityId` |  |
| `amount` | `number; // paise, positive (will be stored as negative)` |  |
| `referenceId` | `EntityId \| null` |  |
| `referenceType` | `string \| null` |  |
| `operatorUid` | `EntityId \| null` |  |
| `operatorName` | `string \| null` |  |
| `description` | `string \| null` |  |
| `idempotencyKey` | `string` |  |
| `deviceId` | `string \| null` |  |
| `now?` | `Date` |  |

## `dispute.ts`

- **type `DisputeStatus`** = 'open' | 'under_review' | 'resolved'
- **type `DisputeResolutionOutcome`** = 'upheld' | 'denied'

### `Dispute` (extends VersionedEntity)

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `organizationId` | `EntityId` |  |
| `orderId` | `EntityId` |  |
| `ledgerEntryId` | `EntityId \| null` |  |
| `raisedBy` | `EntityId` |  |
| `reason` | `string` |  |
| `amount` | `number; // paise` |  |
| `status` | `DisputeStatus` |  |
| `resolutionNote` | `string \| null` |  |
| `resolvedAt` | `string \| null` |  |
| `resolution` | `DisputeResolutionOutcome \| null` | Non-null only when resolved by an admin, not the partner-side `resolveDispute`. |

### `DisputeCreateInput`

| Field | Type | Note |
|---|---|---|
| `organizationId` | `EntityId` |  |
| `orderId` | `EntityId` |  |
| `ledgerEntryId?` | `EntityId \| null` |  |
| `raisedBy` | `EntityId` |  |
| `reason` | `string` |  |
| `amount` | `number` |  |
| `now?` | `Date` |  |

## `door-sale.ts`

- **type `DoorSaleCategory`** = 'walkin' | 'dinein'
- **type `DoorSaleStatus`** = 'active' | 'voided' | 'refunded'
- **type `DoorSalePaymentMode`** = 'cash' | 'card' | 'upi' | 'other'

### `DoorSale` (extends VersionedEntity)

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `eventId` | `EntityId` |  |
| `organizationId` | `EntityId` |  |
| `venueId` | `EntityId \| null` |  |
| `category` | `DoorSaleCategory` | Category: walkin or dinein |
| `guestName` | `string` | Guest name |
| `guestPhone` | `string \| null` | Guest phone (optional) |
| `guestAge` | `number \| null` | Guest age (optional) |
| `gender` | `string \| null` | Gender (optional) |
| `guestEmail` | `string \| null` | Email for the receipt (optional). |
| `totalGuests` | `number` | Total guests in party |
| `tableNumber` | `string \| null` | Category-specific fields |
| `gate` | `string \| null` | Gate identifier |
| `paymentMode` | `DoorSalePaymentMode` | Payment mode |
| `amountPaise` | `number` | Amount charged (paise) |
| `paymentStatus` | `'collected' \| 'pending' \| 'failed'` | Payment status |
| `paymentRef` | `string \| null` | Payment reference/transaction ID |
| `createdBy` | `EntityId` | Staff who created |
| `createdByName` | `string \| null` |  |
| `status` | `DoorSaleStatus` | Status |
| `voidedAt` | `string \| null` | Voided/refunded timestamp |
| `voidedBy` | `EntityId \| null` | Voided by |
| `voidReason` | `string \| null` | Voiding reason |
| `refundedAmountPaise` | `number \| null` | Refunded amount (if refunded) |
| `refundedAt` | `string \| null` | Refund timestamp |
| `refundedBy` | `EntityId \| null` | Refunded by |
| `idempotencyKey` | `string \| null` | Client-supplied idempotency key the sale was created under |

### `DoorSaleCreateInput`

| Field | Type | Note |
|---|---|---|
| `eventId` | `EntityId` |  |
| `organizationId` | `EntityId` |  |
| `venueId` | `EntityId \| null` |  |
| `category` | `DoorSaleCategory` |  |
| `guestName` | `string` |  |
| `guestPhone` | `string \| null` |  |
| `guestAge` | `number \| null` |  |
| `gender` | `string \| null` |  |
| `guestEmail` | `string \| null` |  |
| `totalGuests` | `number` |  |
| `tableNumber` | `string \| null` |  |
| `gate` | `string \| null` |  |
| `paymentMode` | `DoorSalePaymentMode` |  |
| `amountPaise` | `number` |  |
| `createdBy` | `EntityId` |  |
| `createdByName` | `string \| null` |  |
| `idempotencyKey` | `string \| null` |  |
| `now?` | `Date` |  |

## `email-otp.ts`


### `EmailOtp`

| Field | Type | Note |
|---|---|---|
| `recipient` | `string` | Normalized (lower-cased, trimmed) recipient email — the doc key. |
| `codeHash` | `string` |  |
| `expiresAt` | `string` |  |
| `lastSentAt` | `string` |  |
| `attempts` | `number` |  |

## `entitlement.ts`

- **type `EntitlementStatus`** = /** Issued and unused. */ | 'valid' /** Fully scanned in — `scanCount === scanCountAllowed`. */ | 'redeemed' /** Voided by a refund or an admin. */ | 'void'

### `Entitlement` (extends VersionedEntity)

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` | `entitlementId(orderId, tierId, index)` — deterministic, see above. |
| `orderId` | `EntityId` |  |
| `eventId` | `EntityId` |  |
| `organizationId` | `EntityId` |  |
| `tierId` | `EntityId` |  |
| `tierName` | `string` |  |
| `userId` | `EntityId \| null` | Null for a guest checkout with no account. |
| `holderName` | `string` | Name on the ticket, for door staff. |
| `status` | `EntitlementStatus` |  |
| `scanCountAllowed` | `number` | How many people this one entitlement admits. 1 for a normal ticket, 2 for |
| `scanCount` | `number` |  |
| `scannedAt` | `string[]` | Timestamps of each scan, newest last. Door audit trail. |
| `refundLock?` | `{ refundRequestId: EntityId; frozenAt: string; } \| null` | A provider refund may be in flight while the ticket still exists. The |

### `IssueEntitlementsInput`

| Field | Type | Note |
|---|---|---|
| `order` | `Order` |  |
| `admitsPerUnit?` | `Record<EntityId, number>` | How many people each tier admits per unit, keyed by tier id. Absent means |
| `now?` | `Date` |  |

### `AdmissionDecision`

| Field | Type | Note |
|---|---|---|
| `admitted` | `boolean` |  |
| `denyReason` | `ScanDenyReason \| null` |  |
| `denyMessage` | `string \| null` |  |
| `scansUsed` | `number` |  |
| `scansAllowed` | `number` |  |

### `AdmissionClaim`

| Field | Type | Note |
|---|---|---|
| `admitted` | `boolean` |  |
| `denyReason` | `ScanDenyReason \| null` |  |
| `denyMessage` | `string \| null` |  |
| `entitlement` | `Entitlement \| null` |  |
| `scansUsed` | `number` |  |
| `scansAllowed` | `number` |  |

### `AdmitSeatsOptions`

| Field | Type | Note |
|---|---|---|
| `seats?` | `number` |  |
| `expectedScansUsed?` | `number` |  |
| `now?` | `Date` |  |

## `event-catalog.ts`

- **type `TicketTierStatus`** = 'active' | 'paused' | 'sold_out'
- **type `TicketAccessType`** = 'ENTRY' | 'VIP' | 'VVIP' | 'TABLE' | 'PACKAGE' | 'RSVP'
- **type `TicketAudienceType`** = 'GENERAL' | 'MALE' | 'FEMALE' | 'COUPLE' | 'GROUP'
- **type `CreateTicketPricingPhaseInput`** = PartialDateTicketPricingPhaseInput | TicketPricingPhase
- **type `PromoDiscountType`** = 'percent' | 'fixed'
- **type `PromoType`** = 'public' | 'private' | 'single_use' | 'multi_use'
- **type `PromoterAssignmentStatus`** = 'active' | 'ended' | 'suspended'

### `TicketPricingPhase`

| Field | Type | Note |
|---|---|---|
| `id` | `string` |  |
| `name` | `string` |  |
| `priceInPaise` | `number` |  |
| `startsAt` | `string` |  |
| `endsAt` | `string` |  |
| `quantity` | `number \| null` |  |

### `PartialDateTicketPricingPhaseInput`

| Field | Type | Note |
|---|---|---|
| `id` | `string` |  |
| `name` | `string` |  |
| `priceInPaise` | `number` |  |
| `startDate` | `string` |  |
| `endDate` | `string` |  |
| `quantity` | `number \| null` |  |

### `TicketTier` (extends VersionedEntity)

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `eventId` | `EntityId` |  |
| `organizationId` | `EntityId` |  |
| `name` | `string` |  |
| `description` | `string` |  |
| `entryType` | `string` | Entry class shown to guests (V1 `entryType`, default 'general'). |
| `currency` | `string` | ISO 4217; V1 default 'INR'. |
| `priceInPaise` | `number` | Price in paise. |
| `quantity` | `number` | Live sellable quantity; runs through inventory-service at sell time. |
| `status` | `TicketTierStatus` |  |
| `salesStartAt` | `string \| null` |  |
| `salesEndAt` | `string \| null` |  |
| `maxPerOrder` | `number \| null` | Maximum tickets that can be purchased per order. |
| `accessType?` | `TicketAccessType` |  |
| `audienceType?` | `TicketAudienceType` |  |
| `guestCount?` | `number` |  |
| `pricingPhases?` | `TicketPricingPhase[]` |  |
| `doorPriceInPaise?` | `number \| null` |  |
| `benefits?` | `string[]` |  |
| `minAge?` | `number \| null` |  |
| `maxAge?` | `number \| null` |  |
| `minPerOrder?` | `number \| null` |  |
| `maxPerUser?` | `number \| null` |  |
| `tableConfig?` | `{ capacity: number; minimumSpendPaise: number; redeemableAmountPaise: number; tableCount: number; } \| null` |  |
| `commissionEligible?` | `boolean` |  |

### `CreateTicketTierInput`

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `eventId` | `EntityId` |  |
| `organizationId` | `EntityId` |  |
| `name` | `string` |  |
| `description?` | `string` |  |
| `entryType?` | `string` |  |
| `currency?` | `string` |  |
| `priceInPaise?` | `number` |  |
| `quantity` | `number` |  |
| `salesStartAt?` | `string \| null` |  |
| `salesEndAt?` | `string \| null` |  |
| `maxPerOrder?` | `number \| null` |  |
| `accessType?` | `TicketAccessType` |  |
| `audienceType?` | `TicketAudienceType` |  |
| `guestCount?` | `number` |  |
| `pricingPhases?` | `CreateTicketPricingPhaseInput[]` |  |
| `doorPriceInPaise?` | `number \| null` |  |
| `benefits?` | `string[]` |  |
| `minAge?` | `number \| null` |  |
| `maxAge?` | `number \| null` |  |
| `minPerOrder?` | `number \| null` |  |
| `maxPerUser?` | `number \| null` |  |
| `tableConfig?` | `TicketTier['tableConfig']` |  |
| `commissionEligible?` | `boolean` |  |
| `now?` | `Date` |  |

### `PromoCode` (extends VersionedEntity)

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `eventId` | `EntityId \| null` |  |
| `organizationId` | `EntityId` |  |
| `code` | `string` | Normalized uppercase — unique per (organization, event). |
| `name` | `string` | Display name; defaults to the code when not provided (V1 behavior). |
| `type` | `PromoType` |  |
| `discountType` | `PromoDiscountType` |  |
| `discountValue` | `number` | Percent (0–100) when `percent`; paise when `fixed`. |
| `discountFundingOwner` | `DiscountFundingOwner` | Commercial owner that absorbs this discount. |
| `tierIds` | `EntityId[]` | Empty = applies to all tiers (V1 semantics). |
| `maxRedemptions` | `number \| null` | Total allowed redemptions; null = unlimited. |
| `maxPerUser` | `number \| null` | Per-user redemption cap; null = unlimited. |
| `redemptionCount` | `number` | Cumulative redemptions (incremented by the redemption consumer). |
| `startsAt` | `string \| null` |  |
| `endsAt` | `string \| null` |  |
| `isActive` | `boolean` |  |

### `CreatePromoCodeInput`

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `eventId` | `EntityId \| null` |  |
| `organizationId` | `EntityId` |  |
| `code` | `string` |  |
| `name?` | `string` |  |
| `type?` | `PromoType` |  |
| `discountType` | `PromoDiscountType` |  |
| `discountValue` | `number` |  |
| `discountFundingOwner` | `DiscountFundingOwner` |  |
| `tierIds?` | `EntityId[]` |  |
| `maxRedemptions?` | `number \| null` |  |
| `maxPerUser?` | `number \| null` |  |
| `startsAt?` | `string \| null` |  |
| `endsAt?` | `string \| null` |  |
| `now?` | `Date` |  |

### `TablePackage` (extends VersionedEntity)

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `eventId` | `EntityId` |  |
| `organizationId` | `EntityId` |  |
| `name` | `string` |  |
| `minSpendPaise` | `number \| null` | Minimum spend to reserve (paise). |
| `capacity` | `number` |  |
| `pricePaise` | `number` |  |
| `isActive` | `boolean` |  |

### `CreateTablePackageInput`

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `eventId` | `EntityId` |  |
| `organizationId` | `EntityId` |  |
| `name` | `string` |  |
| `minSpendPaise?` | `number \| null` |  |
| `capacity` | `number` |  |
| `pricePaise` | `number` |  |
| `now?` | `Date` |  |

### `CommissionTerms`

| Field | Type | Note |
|---|---|---|
| `version` | `number` | Which number was in force at assignment time. |
| `ratePercent` | `number` | Percentage of eligible sales paid to the promoter. |
| `flatPaise` | `number` | Optional fixed fee (paise). |
| `tierRates?` | `Record<string, { ratePercent: number; flatPaise: number }>` |  |

### `PromoterAssignment` (extends VersionedEntity)

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `eventId` | `EntityId` |  |
| `promoterId` | `EntityId` |  |
| `status` | `PromoterAssignmentStatus` |  |
| `terms` | `CommissionTerms` | Commission terms frozen at assignment time. |
| `createdAt` | `string` |  |
| `endedAt` | `string \| null` | When the assignment was revoked/unlinked, if ever. |
| `suspendedAt` | `string \| null` | When the assignment was suspended by an admin, if ever. |

### `CreatePromoterAssignmentInput`

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `eventId` | `EntityId` |  |
| `promoterId` | `EntityId` |  |
| `terms` | `CommissionTerms` |  |
| `now?` | `Date` |  |

## `event-code.ts`

- **type `EventCodeType`** = 'full' | 'scan_only' | 'charge'
- **type `EventCodeStatus`** = 'active' | 'revoked' | 'expired'
- **type `ScannerSessionType`** = 'staff' | 'device'

### `EventCode` (extends VersionedEntity)

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` | Deterministic: `CODE-${randomHex}` |
| `code` | `string` | Human-readable code: `C1R-XXXXXX` |
| `eventId` | `EntityId` |  |
| `organizationId` | `EntityId` |  |
| `venueId` | `EntityId \| null` |  |
| `type` | `EventCodeType` | Code type determines permissions |
| `gate` | `string \| null` | Optional gate restriction |
| `createdBy` | `EntityId \| null` | Creator UID |
| `createdByName` | `string \| null` | Human-readable creator name |
| `status` | `EventCodeStatus` | Current status |
| `revokedAt` | `string \| null` | Revocation timestamp |
| `revokedReason` | `string \| null` | Revocation reason |
| `expiresAt` | `string \| null` | Expiry timestamp (ISO-8601) |
| `stats` | `EventCodeStats` | Usage statistics |
| `maxDevices` | `number` | Max concurrent devices (default 5) |
| `allowReuse` | `boolean` | Whether multiple devices can use same code simultaneously |

### `EventCodeStats`

| Field | Type | Note |
|---|---|---|
| `scansCount` | `number` | Total scans performed with this code |
| `doorEntriesCount` | `number` | Door entries processed |
| `doorRevenue` | `number` | Revenue from door entries (paise) |
| `lastUsedAt` | `string \| null` | Last used timestamp |
| `activeSessions` | `number` | Active session count |

### `EventCodeCreateInput`

| Field | Type | Note |
|---|---|---|
| `eventId` | `EntityId` |  |
| `organizationId` | `EntityId` |  |
| `venueId` | `EntityId \| null` |  |
| `type` | `EventCodeType` |  |
| `gate` | `string \| null` |  |
| `createdBy` | `EntityId` |  |
| `createdByName` | `string` |  |
| `maxDevices?` | `number` |  |
| `allowReuse?` | `boolean` |  |
| `expiresAt` | `string \| null` |  |
| `now?` | `Date` |  |

### `ScannerSession` (extends VersionedEntity)

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` | Hash of session token: SHA256(token) |
| `sessionToken` | `string \| null` | The raw session token (only returned on creation) |
| `codeId` | `EntityId` | The event code this session belongs to |
| `eventId` | `EntityId` | The event this session is for |
| `organizationId` | `EntityId` | Organization context |
| `venueId` | `EntityId \| null` | Venue context (optional) |
| `type` | `ScannerSessionType` | Session type |
| `deviceId` | `string \| null` | Device identifier |
| `deviceName` | `string \| null` | Device name (for audit) |
| `expiresAt` | `string` | Session expiry (ISO-8601) |
| `lastUsedAt` | `string \| null` | Last activity timestamp |
| `revokedAt` | `string \| null` | Revocation timestamp |
| `revokedReason` | `string \| null` | Revocation reason |
| `permissions` | `SessionPermissions` | Permissions derived from code type |
| `createdBy` | `EntityId` | Session creator (staff UID) |
| `createdByName` | `string \| null` | Human-readable creator name |

### `SessionPermissions`

| Field | Type | Note |
|---|---|---|
| `canScan` | `boolean` | Can scan tickets |
| `canDoorEntry` | `boolean` | Can process door entry (walk-in/dine-in) |
| `canWalkIn` | `boolean` | Can process walk-in sales |
| `canCharge` | `boolean` | Can process cover wallet charges |

### `ScannerSessionCreateInput`

| Field | Type | Note |
|---|---|---|
| `codeId` | `EntityId` |  |
| `organizationId` | `EntityId` | The organization the session belongs to — the event code's owner, NOT the |
| `codeData` | `{ id: EntityId; code: string; eventId: EntityId; venueId: EntityId \| null; type: EventCodeType; gate: stri…` |  |
| `deviceId` | `string` |  |
| `deviceName` | `string` |  |
| `createdBy` | `EntityId` |  |
| `createdByName` | `string` |  |
| `sessionType` | `ScannerSessionType` |  |
| `now?` | `Date` |  |

## `event.ts`

- **type `EventStatus`** = | 'draft' | 'review' | 'scheduled' | 'published' | 'sales_paused' | 'started' | 'ended' | 'archived' | 'cancelled'
- **type `CompensationModel`** = 'standard' | 'custom' | 'salary'
- **type `SalaryPeriod`** = 'per_event' | 'per_day' | 'per_month'

### `TicketTierRef`

| Field | Type | Note |
|---|---|---|
| `tierId` | `EntityId` |  |
| `priceInPaise` | `number` |  |
| `quantity` | `number` |  |

### `EventCompensation`

| Field | Type | Note |
|---|---|---|
| `model` | `CompensationModel` |  |
| `globalRatePercent` | `number \| null` |  |
| `tierRates` | `Record<string, number>` |  |
| `salaryAmountPaise` | `number \| null` |  |
| `salaryPeriod` | `SalaryPeriod \| null` |  |
| `salaryNotes` | `string \| null` |  |

### `Event` (extends VersionedEntity)

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `organizationId` | `EntityId` |  |
| `venueId` | `EntityId \| null` |  |
| `slug` | `string` | URL-safe public identifier — V1 events always carried one and the public |
| `title` | `string` |  |
| `summary` | `string` | Public blurb shown to guests. |
| `description` | `string` | Full public description. |
| `imageUrl` | `string \| null` |  |
| `startAt` | `string` | ISO-8601 timestamps. |
| `endAt` | `string \| null` |  |
| `status` | `EventStatus` |  |
| `isPublic` | `boolean` | Caching visibility derived from status (public vs internal). |
| `tags` | `string[]` | Opaque tags — presentation only. |
| `startingPricePaise` | `number \| null` | Base price when a single price is offered. |
| `isFree` | `boolean` |  |
| `cancellationReason` | `string \| null` | Reason/meta recorded when CANCELLED. |
| `compensation?` | `EventCompensation \| null` |  |
| `capacity` | `number \| null` | Total people the room legally holds, for the door's occupancy gauge. |
| `adminOverride` | `boolean` | True while the current `sales_paused` state was forced by a platform |

### `CreateEventInput`

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `organizationId` | `EntityId` |  |
| `venueId` | `EntityId` |  |
| `title` | `string` |  |
| `summary?` | `string` |  |
| `description?` | `string` |  |
| `imageUrl?` | `string \| null` |  |
| `startAt` | `string` |  |
| `endAt?` | `string \| null` |  |
| `tags?` | `string[]` |  |
| `compensation?` | `EventCompensation \| null` |  |
| `capacity?` | `number \| null` |  |
| `now?` | `Date` |  |

## `financial-journal.ts`

- **type `DiscountFundingOwner`** = 'platform' | 'venue' | 'host' | 'promoter' | 'shared'
- **type `JournalSide`** = 'debit' | 'credit'
- **type `PostingSourceType`** = | 'online_payment_capture' | 'free_ticket' | 'door_cash_sale' | 'door_electronic_sale' | 'promo_discount' | 'complimentary_ticket' | 'void' | 'refund' | 'chargeback' | 'chargeback_loss' | 'chargeback_reversal' | 'reserve' | 'reserve_release' | 'payout_reservation' | 'payout_completion' | 'payout_failure_reversal' | 'manual_adjustment' | 'legacy_backfill'

### `Money`

| Field | Type | Note |
|---|---|---|
| `amountPaise` | `number` |  |
| `currency` | `CurrencyCode` |  |

### `FinancialComponentBreakdown`

| Field | Type | Note |
|---|---|---|
| `ticketFaceValuePaise` | `number` |  |
| `discountPaise` | `number` |  |
| `discountFundingOwner` | `DiscountFundingOwner \| null` |  |
| `netTicketRevenuePaise` | `number` |  |
| `gstPaise` | `number` |  |
| `platformFeePaise` | `number` |  |
| `paymentFeePaise` | `number` |  |
| `providerFeePaise` | `number` |  |
| `platformAllocationPaise` | `number` |  |
| `hostAllocationPaise` | `number` |  |
| `venueAllocationPaise` | `number` |  |
| `promoterAllocationPaise` | `number` |  |
| `refundPaise` | `number` |  |
| `chargebackPaise` | `number` |  |
| `reservePaise` | `number` |  |
| `payoutPaise` | `number` |  |
| `roundingAdjustmentPaise` | `number` |  |

### `AccountingPolicySnapshot`

| Field | Type | Note |
|---|---|---|
| `version` | `string` |  |
| `capturedAt` | `string` |  |
| `settlementDelayDays` | `number` |  |
| `reserveBasisPoints` | `number` |  |
| `requiresEventCompletion` | `boolean` |  |
| `holdsOnCancellation` | `boolean` |  |
| `holdsOnDispute` | `boolean` |  |
| `holdsOnUnresolvedReconciliation` | `boolean` |  |

### `JournalLine`

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `account` | `AccountCode` |  |
| `side` | `JournalSide` |  |
| `amountPaise` | `number` |  |
| `currency` | `CurrencyCode` |  |
| `organizationId` | `EntityId \| null` |  |
| `component` | `keyof FinancialComponentBreakdown \| null` |  |
| `memo` | `string` |  |

### `PostingSource`

| Field | Type | Note |
|---|---|---|
| `type` | `PostingSourceType` |  |
| `id` | `EntityId` |  |
| `orderId` | `EntityId \| null` |  |
| `paymentId` | `string \| null` |  |
| `eventId` | `EntityId \| null` |  |

### `FinancialJournal`

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `organizationId` | `EntityId` |  |
| `currency` | `CurrencyCode` |  |
| `lines` | `readonly JournalLine[]` |  |
| `source` | `PostingSource` |  |
| `idempotencyKey` | `string` |  |
| `accountingPolicy` | `AccountingPolicySnapshot` |  |
| `components` | `FinancialComponentBreakdown \| null` |  |
| `reversalOfJournalId` | `EntityId \| null` |  |
| `evidenceOnly` | `boolean` |  |
| `status` | `'posted'` |  |
| `postedAt` | `string` |  |
| `createdAt` | `string` |  |

### `CreateJournalInput`

| Field | Type | Note |
|---|---|---|
| `organizationId` | `EntityId` |  |
| `currency` | `CurrencyCode` |  |
| `lines` | `readonly Omit<JournalLine, 'id' \| 'currency'>[]` |  |
| `source` | `PostingSource` |  |
| `idempotencyKey` | `string` |  |
| `accountingPolicy` | `AccountingPolicySnapshot` |  |
| `components?` | `FinancialComponentBreakdown \| null` |  |
| `reversalOfJournalId?` | `EntityId \| null` |  |
| `evidenceOnly?` | `boolean` |  |
| `postedAt?` | `Date` |  |

## `guest-profile.ts`


### `GuestProfile` (extends VersionedEntity)

| Field | Type | Note |
|---|---|---|
| `userId` | `EntityId` |  |
| `displayName` | `string` |  |
| `dateOfBirth` | `string` | Calendar date, `YYYY-MM-DD`. |
| `city` | `string` |  |
| `tastes` | `string[]` |  |
| `intents` | `string[]` |  |

### `UpsertGuestProfileInput`

| Field | Type | Note |
|---|---|---|
| `displayName` | `string` |  |
| `dateOfBirth` | `string` |  |
| `city` | `string` |  |
| `tastes` | `string[]` |  |
| `intents` | `string[]` |  |
| `now?` | `Date` |  |

## `journal-transition.ts`

- **type `FinanceReadModelVersion`** = 'legacy_ledger_v1' | 'financial_journal_v2'
- **type `JournalTransitionPhase`** = 'legacy_reads' | 'backfill' | 'comparison' | 'journal_reads'

### `JournalTransitionDiscrepancy`

| Field | Type | Note |
|---|---|---|
| `organizationId` | `EntityId` |  |
| `legacyPayablePaise` | `number` |  |
| `journalPayablePaise` | `number` |  |
| `differencePaise` | `number` |  |
| `comparedAt` | `string` |  |

### `JournalTransitionState`

| Field | Type | Note |
|---|---|---|
| `id` | `'singleton'` |  |
| `phase` | `JournalTransitionPhase` |  |
| `readModel` | `FinanceReadModelVersion` |  |
| `checkpointOrganizationId` | `EntityId \| null` |  |
| `backfilledEntries` | `number` |  |
| `backfillCompletedAt` | `string \| null` |  |
| `comparisonCompletedAt` | `string \| null` |  |
| `discrepancies` | `readonly JournalTransitionDiscrepancy[]` |  |
| `cutoverAt` | `string \| null` |  |
| `rollbackAt` | `string \| null` |  |
| `updatedAt` | `string` |  |
| `version` | `number` |  |

## `leaderboard.ts`

- **type `LeaderboardPeriodType`** = 'all_time' | 'month' | 'week'

### `LeaderboardStat`

| Field | Type | Note |
|---|---|---|
| `promoterId` | `string` |  |
| `periodType` | `LeaderboardPeriodType` |  |
| `periodValue` | `string` | 'all' for all_time, 'YYYY-MM' for month, 'YYYY-Www' (ISO week) for week. |
| `city` | `string` | `GLOBAL_CITY` or a normalized (lower-cased, trimmed) city name. |
| `totalCommissionEarnedPaise` | `number` |  |
| `updatedAt` | `string` |  |

### `LeaderboardBucket`

| Field | Type | Note |
|---|---|---|
| `periodType` | `LeaderboardPeriodType` |  |
| `periodValue` | `string` |  |
| `city` | `string` |  |

## `ledger.ts`

- **type `LedgerEntryType`** = | 'ticket_revenue' | 'platform_fee' | 'venue_share' | 'host_payout' | 'promoter_commission' | 'refund'
- **type `LedgerEntryStatus`** = 'pending' | 'settled' | 'paid_out'

### `LedgerEntry` (extends VersionedEntity)

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `organizationId` | `EntityId` |  |
| `orderId` | `EntityId` |  |
| `eventId` | `EntityId` |  |
| `entryType` | `LedgerEntryType` |  |
| `amount` | `number` | Amount in paise. Always non-negative; direction is implied by `entryType`. |
| `status` | `LedgerEntryStatus` |  |
| `idempotencyKey` | `string` | Idempotency guard: one ledger write per (orderId, entryType). |
| `createdAt` | `string` |  |
| `updatedAt` | `string` |  |

### `LedgerSettlementSplit`

| Field | Type | Note |
|---|---|---|
| `platformFee` | `number` |  |
| `venueShare` | `number` |  |
| `promoterCommission` | `number` |  |
| `hostPayout` | `number` |  |

## `notification.ts`

- **type `NotificationRecipientType`** = 'venue' | 'host' | 'promoter'
- **type `NotificationActionType`** = 'promoter_connection' | 'partnership' | 'slot_request'
- **type `NotificationPriority`** = 'normal' | 'high'

### `NotificationAction`

| Field | Type | Note |
|---|---|---|
| `resourceType` | `NotificationActionType` |  |
| `resourceId` | `EntityId` |  |

### `Notification`

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `recipientId` | `EntityId` |  |
| `recipientType` | `NotificationRecipientType` |  |
| `type` | `string` | Domain-local type, e.g. `promoter_connection.requested`. |
| `title` | `string` |  |
| `body` | `string` |  |
| `read` | `boolean` |  |
| `readAt` | `string \| null` | ISO-8601; null until the notification is read. |
| `data` | `Record<string, unknown>` |  |
| `action` | `NotificationAction \| null` |  |
| `priority` | `NotificationPriority` |  |
| `dedupeKey` | `string \| null` |  |
| `createdAt` | `string` | ISO-8601 (injected clock). |

### `CreateNotificationInput`

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `recipientId` | `EntityId` |  |
| `recipientType` | `NotificationRecipientType` |  |
| `type` | `string` |  |
| `title` | `string` |  |
| `body` | `string` |  |
| `data?` | `Record<string, unknown>` |  |
| `action?` | `NotificationAction` |  |
| `priority?` | `NotificationPriority` |  |
| `dedupeKey?` | `string` |  |
| `now` | `Date` |  |

## `onboarding.ts`

- **type `OnboardingStatus`** = 'draft' | 'submitted' | 'changes_requested' | 'approved' | 'rejected'
- **type `PartnerEntityType`** = 'venue' | 'host' | 'promoter'
- **type `OnboardingPlan`** = 'basic' | 'silver' | 'diamond'
- **type `OnboardingDocumentStatus`** = 'pending' | 'verified' | 'rejected'

### `OnboardingProfile`

| Field | Type | Note |
|---|---|---|
| `legalName` | `string` |  |
| `contactPerson` | `string` |  |
| `phone` | `string` |  |
| `city` | `string` |  |
| `area?` | `string` |  |
| `website?` | `string` |  |
| `capacity?` | `number \| null` |  |
| `instagram?` | `string` |  |
| `bio?` | `string` |  |
| `businessType?` | `string` |  |
| `registrationNumber?` | `string` |  |
| `entityType?` | `string` |  |

### `OnboardingDocument`

| Field | Type | Note |
|---|---|---|
| `label` | `string` | v1 label vocabulary: id_front, id_back, selfie, cheque, registration_certificate… |
| `storagePath` | `string` |  |
| `uploadedAt` | `string` |  |
| `status` | `OnboardingDocumentStatus` |  |
| `reviewedBy` | `EntityId \| null` |  |
| `reviewedAt` | `string \| null` |  |
| `rejectionReason` | `string \| null` |  |

### `OnboardingRequest` (extends VersionedEntity)

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `userId` | `EntityId` |  |
| `status` | `OnboardingStatus` |  |
| `requestedType` | `PartnerEntityType` | What the applicant ASKED to be. Never itself a grant. |
| `plan` | `OnboardingPlan` |  |
| `profile` | `OnboardingProfile` |  |
| `documents` | `OnboardingDocument[]` |  |
| `submittedAt` | `string \| null` |  |
| `reviewedBy` | `EntityId \| null` | Admin decision trail. |
| `reviewedAt` | `string \| null` |  |
| `reviewNote` | `string \| null` |  |
| `provisionedOrganizationId` | `EntityId \| null` | Set once approved — the organization this request produced. |

### `CreateOnboardingRequestInput`

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `userId` | `EntityId` |  |
| `requestedType` | `PartnerEntityType` |  |
| `plan` | `OnboardingPlan` |  |
| `profile` | `OnboardingProfile` |  |
| `now?` | `Date` |  |

### `ReviewOnboardingInput`

| Field | Type | Note |
|---|---|---|
| `reviewedBy` | `EntityId` |  |
| `note?` | `string` |  |
| `now?` | `Date` |  |

## `order.ts`

- **type `OrderStatus`** = /** Cart held, inventory reserved, nothing charged yet. */ | 'pending' /** Payment intent created with the provider

### `OrderLine`

| Field | Type | Note |
|---|---|---|
| `tierId` | `EntityId` |  |
| `tierName` | `string` |  |
| `quantity` | `number` |  |
| `unitPricePaise` | `number` |  |
| `subtotalPaise` | `number` |  |

### `OrderContact`

| Field | Type | Note |
|---|---|---|
| `name` | `string` |  |
| `email` | `string` |  |
| `phone` | `string` |  |

### `OrderAttribution`

| Field | Type | Note |
|---|---|---|
| `referralLinkId` | `EntityId` |  |
| `promoterId` | `EntityId` |  |
| `code` | `string` |  |
| `assignmentId` | `EntityId` |  |
| `assignmentVersion` | `number` |  |
| `termsSnapshot` | `CommissionTerms` |  |
| `attributionSignature` | `string` |  |
| `promoterCommissionPaise` | `number` |  |

### `Order` (extends VersionedEntity)

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `eventId` | `EntityId` |  |
| `organizationId` | `EntityId` |  |
| `userId` | `EntityId \| null` | Null for a guest checkout with no account. |
| `contact` | `OrderContact` |  |
| `status` | `OrderStatus` |  |
| `lines` | `OrderLine[]` |  |
| `currency` | `string` |  |
| `subtotalPaise` | `number` |  |
| `discountPaise` | `number` |  |
| `discountFundingOwner` | `DiscountFundingOwner \| null` |  |
| `discountedSubtotalPaise` | `number` |  |
| `platformFeePaise` | `number` |  |
| `paymentFeePaise` | `number` |  |
| `gstPaise` | `number` |  |
| `grandTotalPaise` | `number` |  |
| `appliedPromoCode` | `string \| null` |  |
| `attribution` | `OrderAttribution \| null` |  |
| `paymentIntentId` | `string \| null` | Provider's order id (Razorpay `order_...`), set at intent. |
| `paymentId` | `string \| null` | Provider's payment id, set on capture. The idempotency anchor. |
| `paidAt` | `string \| null` |  |
| `reservationExpiresAt` | `string` | When the inventory hold lapses. ISO-8601. |
| `failureReason` | `string \| null` | Why the order ended, when it ended unhappily. |
| `refundedPaise` | `number` | Cumulative amount refunded so far (Phase 6 admin refunds). A partial |
| `appliedRefundRequestIds` | `EntityId[]` | Refund request ids already reflected in `refundedPaise` (retry fence). |

### `CreateOrderInput`

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `eventId` | `EntityId` |  |
| `organizationId` | `EntityId` |  |
| `userId?` | `EntityId \| null` |  |
| `contact` | `OrderContact` |  |
| `pricing` | `PricingBreakdown` |  |
| `attribution?` | `OrderAttribution \| null` |  |
| `reservationTtlMs?` | `number` | How long the cart hold lasts. v1 used ~10 minutes. |
| `now?` | `Date` |  |

## `organization.ts`

- **type `Capability`** = 'host' | 'venue' | 'promoter'
- **type `OrganizationRole`** = 'owner' | 'admin' | 'manager' | 'member'
- **type `InvitationStatus`** = 'pending' | 'accepted' | 'revoked' | 'expired'
- **type `OrganizationStatus`** = 'active' | 'suspended' | 'archived'

### `OrganizationMember`

| Field | Type | Note |
|---|---|---|
| `userId` | `EntityId` |  |
| `role` | `OrganizationRole` |  |
| `capabilities` | `Capability[]` |  |
| `joinedAt` | `string` |  |
| `invitedBy?` | `EntityId` |  |
| `name?` | `string \| null` |  |
| `email?` | `string \| null` |  |

### `OrganizationInvitation` (extends VersionedEntity)

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `organizationId` | `EntityId` |  |
| `email` | `string` | Lower-cased on creation so `A@x.com` and `a@x.com` cannot both be pending. |
| `role` | `OrganizationRole` |  |
| `capabilities` | `Capability[]` |  |
| `status` | `InvitationStatus` |  |
| `invitedBy` | `EntityId` |  |
| `expiresAt` | `string` | ISO-8601. Past this, the invitation is expired even if still `pending`. |
| `acceptedAt` | `string \| null` |  |
| `acceptedBy` | `EntityId \| null` |  |

### `Organization` (extends VersionedEntity)

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `name` | `string` |  |
| `slug` | `string` |  |
| `ownerId` | `EntityId` | Owner of the organization — the top-lead account. |
| `members` | `OrganizationMember[]` | Memberships keyed by userId (bounded within MemoryRepository at T08). |
| `settings` | `OrganizationSettings` | Internal / non-public settings. |
| `status` | `OrganizationStatus` |  |
| `platformFeePercent` | `number` | Commission the platform takes, whole-number percent. Set once at |

### `OrganizationSettings`

| Field | Type | Note |
|---|---|---|
| `name?` | `string` | Owned/branding info shared with partner surfaces. |
| `timezone?` | `string` | IANA timezone; V1 proven default 'Asia/Kolkata'. |
| `defaultCurrency?` | `string` | ISO 4217 default currency for pricing; V1 proven default 'INR'. |

### `CreateOrganizationInput`

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `name` | `string` |  |
| `slug` | `string` |  |
| `ownerId` | `EntityId` |  |
| `settings?` | `OrganizationSettings` |  |
| `capabilities?` | `Capability[]` | Owner's capabilities. Defaults to all three for a self-serve org. |
| `platformFeePercent?` | `number` | Defaults to the `basic` plan's 15% until an approval says otherwise. |
| `now?` | `Date` |  |

### `OrganizationProps`

| Field | Type | Note |
|---|---|---|
| `name?` | `string` |  |
| `slug?` | `string` |  |
| `settings?` | `OrganizationSettings` |  |

### `InviteMemberInput`

| Field | Type | Note |
|---|---|---|
| `userId` | `EntityId` |  |
| `role` | `OrganizationRole` |  |
| `capabilities?` | `Capability[]` |  |
| `invitedBy` | `EntityId` |  |
| `now?` | `Date` |  |

### `CreateInvitationInput`

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `organizationId` | `EntityId` |  |
| `email` | `string` |  |
| `role` | `OrganizationRole` |  |
| `capabilities?` | `Capability[]` |  |
| `invitedBy` | `EntityId` |  |
| `ttlDays?` | `number` |  |
| `now?` | `Date` |  |

## `partner-access.ts`

- **type `PartnerType`** = 'venue' | 'host' | 'promoter'
- **type `PartnerRole`** = | 'OWNER' | 'MANAGER' | 'FINANCE_ADMIN' | 'STAFF' | 'SECURITY' | 'DOOR' | 'COHOST' | 'PROMOTER' | 'TEAM_LEAD'
- **type `PartnerPermission`** = | 'VIEW_FINANCIALS' | 'MANAGE_STAFF' | 'MANAGE_EVENTS' | 'EDIT_EVENT_RULES' | 'MANAGE_TABLES' | 'VIEW_GUESTLIST' | 'SCAN_ENTRY' | 'LOG_INCIDENTS' | 'VIEW_ANALYTICS' | 'MANAGE_SETTINGS' | 'MANAGE_PROMOTERS' | 'MANAGE_PAYOUTS' | 'MANAGE_PARTNERSHIPS' | 'MANAGE_PAGE_CONTENT' | 'VIEW_REAL_TIME_SCANS' | 'MANAGE_GUEST_OPS' | 'CHARGE_COVER_WALLETS' | 'EXPORT_GUESTS'

### `PartnerAccessContext`

| Field | Type | Note |
|---|---|---|
| `partnerType` | `PartnerType` |  |
| `role` | `PartnerRole` |  |
| `permissions` | `readonly PartnerPermission[]` |  |
| `tabVisibility` | `TabVisibility \| null` | `null` = no restriction; the caller sees every tab. |

## `partnership.ts`

- **type `PartnershipStatus`** = 'pending' | 'active' | 'rejected' | 'blocked' | 'ended'
- **type `PartnershipInitiator`** = 'host' | 'venue'

### `Partnership` (extends VersionedEntity)

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `hostOrganizationId` | `EntityId` | The host organization. |
| `venueOrganizationId` | `EntityId` | The venue organization (the venue's owning tenant, not the venue id). |
| `venueId` | `EntityId` |  |
| `initiatedBy` | `PartnershipInitiator` |  |
| `status` | `PartnershipStatus` |  |
| `message` | `string \| null` |  |
| `venueShareRate` | `number \| null` | The venue's agreed split of each settlement's gross, as a whole-number |
| `resolutionReason` | `string \| null` | Set when rejected or blocked, so the other party learns why. |
| `resolvedAt` | `string \| null` |  |

### `CreatePartnershipInput`

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `hostOrganizationId` | `EntityId` |  |
| `venueOrganizationId` | `EntityId` |  |
| `venueId` | `EntityId` |  |
| `initiatedBy` | `PartnershipInitiator` |  |
| `message?` | `string` |  |
| `venueShareRate?` | `number` | Optional at request time; both parties can set it later on the live deal. |
| `now?` | `Date` |  |

### `CommissionTier`

| Field | Type | Note |
|---|---|---|
| `threshold` | `number` | Minimum tickets sold to reach this tier. |
| `rate` | `number` | Whole-number percentage. |
| `label` | `string` |  |

## `payout.ts`

- **type `PayoutStatus`** = 'requested' | 'processing' | 'paid' | 'failed' | 'frozen'

### `Payout` (extends VersionedEntity)

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `organizationId` | `EntityId` |  |
| `bankAccountId` | `EntityId` |  |
| `amount` | `number; // paise` |  |
| `status` | `PayoutStatus` |  |
| `failureReason` | `string \| null` |  |
| `requestedBy` | `EntityId` |  |
| `processedAt` | `string \| null` |  |
| `transferReference` | `string \| null` |  |
| `transferredAt` | `string \| null` |  |
| `transferMethod` | `string \| null` |  |
| `transferProof` | `PayoutTransferProof \| null` |  |
| `approvedBy` | `EntityId \| null` |  |
| `previousStatus` | `PayoutStatus \| null` | The status this payout held before an admin froze it. Set only while |

### `PayoutTransferProof`

| Field | Type | Note |
|---|---|---|
| `sha256` | `string` | SHA-256 of the immutable proof object; the proof body is kept out of the payout row. |
| `contentType` | `string` |  |
| `sizeBytes` | `number` |  |

### `PayoutTransferEvidence`

| Field | Type | Note |
|---|---|---|
| `transferReference` | `string` |  |
| `transferredAt` | `string` |  |
| `transferMethod` | `string` |  |
| `proof` | `PayoutTransferProof` |  |
| `approvedBy` | `EntityId` |  |

### `PayoutCreateInput`

| Field | Type | Note |
|---|---|---|
| `id?` | `EntityId` |  |
| `organizationId` | `EntityId` |  |
| `bankAccountId` | `EntityId` |  |
| `amount` | `number` |  |
| `requestedBy` | `EntityId` |  |
| `now?` | `Date` |  |

## `platform-settings.ts`


### `PlatformSettings`

| Field | Type | Note |
|---|---|---|
| `platformFeeRate` | `number` | Default platform fee rate (decimal, e.g. 0.15 = 15%). |
| `refundSingleApproverThresholdPaise` | `number` | Refund threshold below which zero approvers are required (paise, default ₹500 = 50_000). |
| `refundDualApproverThresholdPaise` | `number` | Refund threshold requiring two approvers at or above (paise, default ₹5_000 = 500_000). |
| `maintenanceMode` | `boolean` | When true, the platform enters maintenance mode (ops-middleware gate). |
| `featureFlags` | `Record<string, boolean>` | Arbitrary feature flags for gradual rollouts. |
| `accountingPolicy` | `AccountingPolicyConfig` |  |
| `updatedAt` | `string` |  |

### `AccountingPolicyConfig`

| Field | Type | Note |
|---|---|---|
| `version` | `string` |  |
| `settlementDelayDays` | `number` |  |
| `reserveBasisPoints` | `number` |  |
| `requiresEventCompletion` | `boolean` |  |
| `holdsOnCancellation` | `boolean` |  |
| `holdsOnDispute` | `boolean` |  |
| `holdsOnUnresolvedReconciliation` | `boolean` |  |

### `PlatformSettingsInput`

| Field | Type | Note |
|---|---|---|
| `platformFeeRate?` | `number` |  |
| `refundSingleApproverThresholdPaise?` | `number` |  |
| `refundDualApproverThresholdPaise?` | `number` |  |
| `maintenanceMode?` | `boolean` |  |
| `featureFlags?` | `Record<string, boolean>` |  |
| `accountingPolicy?` | `AccountingPolicyConfig` |  |

## `platform-user.ts`


### `PlatformUser`

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `email` | `string` |  |
| `name` | `string` |  |
| `image` | `string \| null` |  |
| `emailVerified` | `boolean` |  |
| `role` | `string \| null` | Platform role (e.g. `partner`, `admin`, `guest`) — Better Auth additional field. |
| `createdAt` | `number` |  |
| `updatedAt` | `number` |  |

## `pricing.ts`


### `PricingLineInput`

| Field | Type | Note |
|---|---|---|
| `tier` | `TicketTier` |  |
| `quantity` | `number` |  |

### `PricingLine`

| Field | Type | Note |
|---|---|---|
| `tierId` | `EntityId` |  |
| `tierName` | `string` |  |
| `quantity` | `number` |  |
| `unitPricePaise` | `number` | The tier's price at quote time, captured so a later edit cannot rewrite it. |
| `subtotalPaise` | `number` |  |

### `PricingBreakdown`

| Field | Type | Note |
|---|---|---|
| `lines` | `PricingLine[]` |  |
| `subtotalPaise` | `number` | Σ(unit × qty), before any discount. |
| `discountPaise` | `number` |  |
| `discountFundingOwner?` | `DiscountFundingOwner \| null` |  |
| `discountedSubtotalPaise` | `number` | `subtotal - discount`, never below zero. |
| `platformFeePaise` | `number` |  |
| `paymentFeePaise` | `number` |  |
| `gstPaise` | `number` | GST on (platform + payment) fees only — not on the ticket price. |
| `grandTotalPaise` | `number` |  |
| `appliedPromoCode` | `string \| null` | The promo actually applied, if any. Null when none applied. |
| `currency` | `string` |  |

### `CalculatePricingInput`

| Field | Type | Note |
|---|---|---|
| `lines` | `PricingLineInput[]` |  |
| `promo?` | `PromoCode \| null` | Already validated as redeemable by the caller; this only does the maths. |
| `currency?` | `string` | ISO 4217. All lines must agree — a mixed-currency cart is not a cart. |

## `promoter-connection.ts`

- **type `PromoterConnectionStatus`** = 'pending' | 'active' | 'rejected' | 'blocked' | 'revoked'
- **type `ConnectionTargetType`** = 'host' | 'venue'
- **type `ConnectionInitiator`** = 'promoter' | 'target'

### `PromoterConnection` (extends VersionedEntity)

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `promoterId` | `EntityId` |  |
| `targetId` | `EntityId` |  |
| `targetType` | `ConnectionTargetType` |  |
| `initiatedBy` | `ConnectionInitiator` |  |
| `status` | `PromoterConnectionStatus` |  |
| `message` | `string \| null` |  |
| `resolutionReason` | `string \| null` |  |
| `resolvedAt` | `string \| null` |  |

### `CreatePromoterConnectionInput`

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `promoterId` | `EntityId` |  |
| `targetId` | `EntityId` |  |
| `targetType` | `ConnectionTargetType` |  |
| `initiatedBy` | `ConnectionInitiator` |  |
| `message?` | `string` |  |
| `now?` | `Date` |  |

## `provider-financial-event.ts`

- **type `ProviderEventTrust`** = 'signature_verified' | 'internal_provider_response'
- **type `ProviderEventProcessingState`** = 'pending' | 'processed' | 'support_required'

### `ProviderFinancialEvent`

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `provider` | `'razorpay'` |  |
| `providerEventId` | `string` |  |
| `eventType` | `string` |  |
| `paymentId` | `string` |  |
| `providerOrderId` | `string \| null` |  |
| `refundId` | `string \| null` |  |
| `disputeId` | `string \| null` |  |
| `receivedAt` | `string` |  |
| `providerOccurredAt` | `string \| null` |  |
| `payloadHash` | `string` |  |
| `trustedIngestion` | `ProviderEventTrust` |  |
| `replayOfEventId` | `EntityId \| null` |  |
| `processingState` | `ProviderEventProcessingState` |  |
| `processingAttempts` | `number` |  |
| `finalResult` | `string \| null` |  |
| `processedAt` | `string \| null` |  |
| `version` | `number` |  |

### `CreateProviderFinancialEventInput`

| Field | Type | Note |
|---|---|---|
| `providerEventId?` | `string \| null` |  |
| `eventType` | `string` |  |
| `paymentId` | `string` |  |
| `providerOrderId?` | `string \| null` |  |
| `refundId?` | `string \| null` |  |
| `disputeId?` | `string \| null` |  |
| `receivedAt` | `Date` |  |
| `providerOccurredAt?` | `string \| null` |  |
| `payloadHash` | `string` |  |
| `trustedIngestion` | `ProviderEventTrust` |  |
| `replayOfEventId?` | `EntityId \| null` |  |

## `provider-payment.ts`

- **type `ProviderDisputeStatus`** = 'open' | 'under_review' | 'won' | 'lost' | 'closed'
- **type `ProviderPaymentEvent`** = | { type: 'captured'
- **type `ReconciliationFinding`** = | { kind: 'provider_record_missing' } | { kind: 'amount_mismatch'

### `ProviderDispute`

| Field | Type | Note |
|---|---|---|
| `amountPaise` | `number` |  |
| `amountDeductedPaise` | `number` |  |
| `status` | `ProviderDisputeStatus` |  |
| `reasonCode` | `string \| null` |  |
| `reasonDescription` | `string \| null` |  |
| `respondBy` | `string \| null` |  |
| `phase` | `'fraud' \| 'retrieval' \| 'chargeback' \| 'pre_arbitration' \| 'arbitration' \| null` |  |
| `evidenceState` | `'none' \| 'submitted'` |  |
| `providerOccurredAt` | `string \| null` |  |

### `ProviderPayment` (extends VersionedEntity)

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` | The provider's payment id (Razorpay `pay_...`). |
| `provider` | `'razorpay'` |  |
| `providerOrderId` | `string \| null` | The provider's order id, once known. |
| `orderId` | `EntityId \| null` | Our order, once the capture has been tied to one. |
| `processingState` | `'requires_support' \| 'processing' \| 'fulfilled' \| 'refunded'` | Durable reconciliation state; safe for operators and recovery reads. |
| `currency` | `string` |  |
| `capturedPaise` | `number` | Zero until the provider reports the capture. |
| `feePaise` | `number` | The provider's own fee on this payment, in paise. |
| `taxOnFeePaise` | `number` | Tax the provider charged on its fee, in paise. |
| `capturedAt` | `string \| null` |  |
| `refunds` | `Record<string, number>` | Provider refund id -> refunded paise. Only refunds the provider has processed. |
| `disputes` | `Record<string, ProviderDispute>` | Provider dispute id -> amount and current status. |

### `ReconciliationResult`

| Field | Type | Note |
|---|---|---|
| `orderId` | `EntityId` |  |
| `paymentId` | `string \| null` |  |
| `reconciled` | `boolean` | True only when there is nothing to look at. |
| `findings` | `ReconciliationFinding[]` |  |
| `providerNetPaise` | `number \| null` | The provider's net for this payment; null when there is no provider record. |

## `reconciliation.ts`

- **type `ReconciliationRunStatus`** = 'running' | 'completed' | 'failed'
- **type `ReconciliationDiscrepancyKind`** = | 'captured_payment_without_order' | 'paid_order_without_capture' | 'paid_order_without_ticket' | 'ticket_without_paid_order' | 'missing_journal' | 'duplicate_journal' | 'unbalanced_journal' | 'amount_mismatch' | 'currency_mismatch' | 'provider_order_mismatch' | 'refund_mismatch' | 'chargeback_mismatch' | 'stale_pending_payment' | 'payout_mismatch'
- **type `ReconciliationDiscrepancyState`** = 'open' | 'finance_review' | 'support_required' | 'resolved'

### `ReconciliationRun` (extends VersionedEntity)

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `status` | `ReconciliationRunStatus` |  |
| `requestedBy` | `EntityId` |  |
| `startedAt` | `string` |  |
| `completedAt` | `string \| null` |  |
| `scannedCounts` | `Record<string, number>` |  |
| `discrepancyCount` | `number` |  |
| `failureReason` | `string \| null` |  |

### `ReconciliationDiscrepancy` (extends VersionedEntity)

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `runId` | `EntityId` |  |
| `kind` | `ReconciliationDiscrepancyKind` |  |
| `state` | `ReconciliationDiscrepancyState` |  |
| `organizationId` | `EntityId \| null` |  |
| `orderId` | `EntityId \| null` |  |
| `paymentId` | `string \| null` |  |
| `payoutId` | `EntityId \| null` |  |
| `amountPaise` | `number \| null` |  |
| `currency` | `string \| null` |  |
| `evidence` | `Record<string, string \| number \| boolean \| null>` |  |
| `repair` | `'recheck_evidence' \| null` |  |
| `resolutionReason` | `string \| null` |  |
| `resolvedBy` | `EntityId \| null` |  |
| `resolvedAt` | `string \| null` |  |

## `referral-link.ts`


### `ReferralLink` (extends VersionedEntity)

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `eventId` | `EntityId` |  |
| `promoterId` | `EntityId` |  |
| `organizationId` | `EntityId` |  |
| `assignmentId` | `EntityId \| null` | Assignment whose approved compensation was frozen when this link was made. |
| `assignmentVersion` | `number \| null` |  |
| `termsSnapshot` | `CommissionTerms \| null` |  |
| `attributionSignature` | `string \| null` |  |
| `eventTitle` | `string` |  |
| `campaignLabel` | `string` |  |
| `vanityPrefix` | `string` |  |
| `vanitySlug` | `string \| null` |  |
| `code` | `string` | Short, case-insensitive, URL-safe. Unique per event. |
| `label` | `string` |  |
| `isActive` | `boolean` |  |
| `clicks` | `number` | Click counter — cheap vanity metric, never an attribution source. |
| `conversions` | `number` | Orders attributed to this link. Incremented by the checkout path, which |
| `revenuePaise` | `number` |  |
| `commissionPaise` | `number` |  |

### `CreateReferralLinkInput`

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `eventId` | `EntityId` |  |
| `promoterId` | `EntityId` |  |
| `organizationId` | `EntityId` |  |
| `assignmentId?` | `EntityId` |  |
| `assignmentVersion?` | `number` |  |
| `termsSnapshot?` | `CommissionTerms` |  |
| `attributionSignature?` | `string` |  |
| `eventTitle?` | `string` |  |
| `vanityPrefix?` | `string` |  |
| `vanitySlug?` | `string` |  |
| `code?` | `string` | Omit to generate one. |
| `label?` | `string` |  |
| `random?` | `() => number` | Injected randomness — the domain never reaches for a global RNG. |
| `now?` | `Date` |  |

## `refund-request.ts`

- **type `AdminRefundRequestStatus`** = 'pending' | 'approved' | 'executing' | 'rejected' | 'settled' | 'failed'
- **type `RefundScope`** = 'amount_only' | 'ticket_lines'

### `AdminRefundApproval`

| Field | Type | Note |
|---|---|---|
| `adminId` | `EntityId` |  |
| `approvedAt` | `string` |  |

### `RefundTicketLine`

| Field | Type | Note |
|---|---|---|
| `tierId` | `EntityId` |  |
| `quantity` | `number` |  |

### `AdminRefundRequest` (extends VersionedEntity)

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `orderId` | `EntityId` |  |
| `organizationId` | `EntityId` |  |
| `amountPaise` | `number` | Paise. Must not exceed the order's refundable balance at request time. |
| `requestedBy` | `EntityId` |  |
| `reason` | `string` |  |
| `scope` | `RefundScope` | `amount_only` adjusts money without selectively revoking admission rights. |
| `ticketLines` | `RefundTicketLine[]` |  |
| `targetEntitlementIds` | `EntityId[]` |  |
| `approversRequired` | `number` | 0 (auto), 1, or 2 — fixed at creation from the amount tier. |
| `approvals` | `AdminRefundApproval[]` |  |
| `status` | `AdminRefundRequestStatus` |  |
| `rejectedBy` | `EntityId \| null` |  |
| `rejectionReason` | `string \| null` |  |
| `providerRefundId` | `string \| null` | The payment provider's refund id, set once settlement succeeds. |
| `providerStatus` | `string \| null` |  |
| `executionIdempotencyKey` | `string` |  |
| `executionAttempts` | `number` |  |
| `failureReason` | `string \| null` | Why settlement failed, if it did — surfaced for a human to retry or escalate. |

### `CreateAdminRefundRequestInput`

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `orderId` | `EntityId` |  |
| `organizationId` | `EntityId` |  |
| `amountPaise` | `number` |  |
| `requestedBy` | `EntityId` |  |
| `reason` | `string` |  |
| `scope?` | `RefundScope` |  |
| `ticketLines?` | `RefundTicketLine[]` |  |
| `targetEntitlementIds?` | `EntityId[]` |  |
| `hasRedeemedEntitlement` | `boolean` | Whether any entitlement on this order has already been scanned. |
| `thresholds?` | `RefundThresholds` | Platform-wide refund thresholds (fetched from PlatformSettings). |
| `now?` | `Date` |  |

### `RefundThresholds`

| Field | Type | Note |
|---|---|---|
| `singleApproverCeilingPaise` | `number` |  |
| `dualApproverCeilingPaise` | `number` |  |

## `safety-report.ts`

- **type `SafetyReportStatus`** = 'open' | 'dismissed' | 'actioned'
- **type `SafetyReportCategory`** = 'harassment' | 'spam' | 'inappropriate' | 'safety' | 'other'
- **type `SafetyReportPriority`** = 'critical' | 'normal'
- **type `SafetyReportTargetType`** = 'user' | 'event' | 'venue' | 'organization'

### `SafetyReportReporter`

| Field | Type | Note |
|---|---|---|
| `userId` | `EntityId` |  |
| `email` | `string \| null` | Captured on the guest intake path when disclosed; null otherwise. |

### `SafetyReportTarget`

| Field | Type | Note |
|---|---|---|
| `type` | `SafetyReportTargetType` |  |
| `id` | `EntityId` |  |

### `SafetyReportContext`

| Field | Type | Note |
|---|---|---|
| `eventId` | `EntityId \| null` |  |
| `messageId` | `EntityId \| null` |  |

### `SafetyReportAction`

| Field | Type | Note |
|---|---|---|
| `type` | `string` | The discipline verb the operator states resolved this report. |
| `ref` | `EntityId` | The ban / target id the discipline was applied against. |

### `SafetyReport` (extends VersionedEntity)

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `reporter` | `SafetyReportReporter` |  |
| `target` | `SafetyReportTarget` |  |
| `category` | `SafetyReportCategory` |  |
| `priority` | `SafetyReportPriority` | `critical` when the category is `safety`, else `normal` (v1's derivation). |
| `details` | `string \| null` |  |
| `context` | `SafetyReportContext` |  |
| `status` | `SafetyReportStatus` |  |
| `reviewDueAt` | `string \| null` | 24h from creation while open — the review promise v1 never recorded. |
| `dismissedAt` | `string \| null` |  |
| `dismissedBy` | `EntityId \| null` |  |
| `actionedAt` | `string \| null` |  |
| `actionedBy` | `EntityId \| null` |  |
| `action` | `SafetyReportAction \| null` |  |
| `deletedAt` | `string \| null` | Soft-delete attribution — never a hard delete (see `event.ts`). |
| `deletedBy` | `EntityId \| null` |  |

### `CreateSafetyReportInput`

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `reporter` | `SafetyReportReporter` |  |
| `target` | `SafetyReportTarget` |  |
| `category` | `SafetyReportCategory` |  |
| `details?` | `string \| null` |  |
| `context?` | `Partial<SafetyReportContext>` |  |
| `now?` | `Date` |  |

## `scan-ledger.ts`

- **type `ScanDenyReason`** = | 'invalid_signature' | 'already_used' | 'expired' | 'wrong_event' | 'device_invalid' | 'void_ticket' | 'capacity_exceeded' | 'wrong_gate' | 'offline_expired' | 'override_required' | 'promoter_not_authorized'

### `ScanLedger` (extends VersionedEntity)

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` | Deterministic: `${eventId}_${entitlementId}_${scanIndex}` or `${eventId}_${entitlementId}` for single-scan tic |
| `eventId` | `EntityId` |  |
| `organizationId` | `EntityId` |  |
| `venueId` | `EntityId \| null` |  |
| `entitlementId` | `EntityId \| null` | The entitlement that was scanned (for ticket scans) |
| `doorSaleId` | `EntityId \| null` | For walk-in/dine-in sales |
| `entryType` | `string \| null` | The entitlement type (general, couple, vip, etc.) |
| `tierName` | `string \| null` | Ticket tier name at time of scan |
| `tierId` | `EntityId \| null` | Ticket tier ID at time of scan |
| `operatorUid` | `string \| null` | Who scanned (operator UID) |
| `operatorName` | `string \| null` | Operator display name |
| `operatorRole` | `string \| null` | Operator role |
| `gate` | `string \| null` | Gate identifier |
| `deviceId` | `string \| null` | Device ID used for scan |
| `deviceName` | `string \| null` | Device name (for audit) |
| `deviceBound` | `boolean` | Whether device was registered/bound |
| `status` | `ScanLedgerStatus` | Scan result |
| `denyReason` | `ScanDenyReason \| null` | If denied, why |
| `denyMessage` | `string \| null` | Human-readable deny message |
| `guestName` | `string \| null` | Guest name (for audit) |
| `guestEmail` | `string \| null` | Guest email (if available) |
| `guestPhone` | `string \| null` | Guest phone (if available) |
| `scannedAt` | `string` | Scan timestamp (ISO-8601) |
| `admittedCount` | `number` | Number of people admitted by this scan |
| `scanCountUsed` | `number \| null` | For entitlement scans: how many scans used vs allowed |
| `scanCountAllowed` | `number \| null` |  |
| `isOffline` | `boolean` | Offline scan flag |
| `syncedAt` | `string \| null` | Offline sync timestamp (if applicable) |
| `offlineDeviceId` | `string \| null` | Offline device ID (for audit) |
| `overriddenBy` | `string \| null` | Actor uid who overrode a denied entry. Null unless `status === 'overridden'`. |
| `overrideReason` | `string \| null` | Why staff overrode the denial. Null unless `status === 'overridden'`. |

### `ScanLedgerCreateInput`

| Field | Type | Note |
|---|---|---|
| `eventId` | `EntityId` |  |
| `organizationId` | `EntityId` |  |
| `venueId` | `EntityId \| null` |  |
| `entitlementId` | `EntityId \| null` |  |
| `doorSaleId` | `EntityId \| null` |  |
| `entryType` | `string \| null` |  |
| `tierName` | `string \| null` |  |
| `tierId` | `EntityId \| null` |  |
| `operatorUid` | `string \| null` |  |
| `operatorName` | `string \| null` |  |
| `operatorRole` | `string \| null` |  |
| `gate` | `string \| null` |  |
| `deviceId` | `string \| null` |  |
| `deviceName` | `string \| null` |  |
| `deviceBound` | `boolean` |  |
| `guestName` | `string \| null` |  |
| `guestEmail` | `string \| null` |  |
| `guestPhone` | `string \| null` |  |
| `scannedAt` | `string` |  |
| `admittedCount` | `number` |  |
| `scanCountUsed` | `number \| null` |  |
| `scanCountAllowed` | `number \| null` |  |
| `isOffline` | `boolean` |  |
| `offlineDeviceId` | `string \| null` |  |
| `status?` | `ScanLedgerStatus` | Terminal status to record at creation. A scan is decided before it is |
| `denyReason?` | `ScanDenyReason \| null` |  |
| `denyMessage?` | `string \| null` |  |
| `now?` | `Date` |  |

## `scanner-device.ts`

- **type `ScannerDeviceStatus`** = 'active' | 'unbound'

### `ScannerDevice` (extends VersionedEntity)

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` | Hash of `organizationId` + `deviceId` — see `scannerDeviceId`. |
| `organizationId` | `EntityId` |  |
| `venueId` | `EntityId \| null` |  |
| `deviceId` | `string` | Opaque, client-generated, stable for the life of the install. |
| `deviceName` | `string` | Human label staff can recognize in a device list ("Gate iPad 1"). |
| `status` | `ScannerDeviceStatus` |  |
| `boundBy` | `EntityId` | Who authorized it, and when. |
| `boundAt` | `string` |  |
| `unboundAt` | `string \| null` |  |
| `unboundReason` | `string \| null` |  |
| `lastSeenAt` | `string` | Liveness — how the dashboard knows which scanners are on the door now. |
| `lastEventId` | `EntityId \| null` |  |
| `lastGate` | `string \| null` |  |
| `scanCount` | `number` | Rolling per-device counters, for "this phone is denying everything". |
| `lastScanAt` | `string \| null` |  |
| `lastScanResult` | `string \| null` |  |

### `BindScannerDeviceInput`

| Field | Type | Note |
|---|---|---|
| `organizationId` | `EntityId` |  |
| `venueId` | `EntityId \| null` |  |
| `deviceId` | `string` |  |
| `deviceName` | `string` |  |
| `boundBy` | `EntityId` |  |
| `now?` | `Date` |  |

## `social.ts`


### `Follow`

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `followerId` | `EntityId` |  |
| `targetType` | `FollowTargetType` |  |
| `targetId` | `EntityId` |  |
| `createdAt` | `string` |  |

### `SocialNotification`

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `userId` | `EntityId` |  |
| `type` | `NotificationType` |  |
| `title` | `string` |  |
| `body` | `string` |  |
| `link` | `string \| null` | Relative guest-portal path the notification opens, e.g. `/event/<slug>`. |
| `subjectId` | `EntityId` | The aggregate the notification is about (event id for new-event). |
| `createdAt` | `string` |  |
| `readAt` | `string \| null` |  |

## `support-ticket.ts`

- **type `SupportTicketStatus`** = 'open' | 'in_progress' | 'waiting_on_customer' | 'escalated' | 'resolved' | 'closed'
- **type `SupportTicketPriority`** = 'low' | 'medium' | 'high' | 'urgent'
- **type `SupportTicketCategory`** = 'account' | 'billing' | 'order' | 'event' | 'technical' | 'other'
- **type `SupportTimelineEventType`** = | 'created' | 'reply' | 'internal_note' | 'assignment' | 'priority_change' | 'link' | 'escalation' | 'merge' | 'status_change' | 'deleted'

### `SupportTicketRequester`

| Field | Type | Note |
|---|---|---|
| `userId` | `EntityId` |  |
| `email` | `string \| null` | Set on the guest intake path; null when an admin opens a ticket directly. |
| `organizationId` | `EntityId \| null` |  |

### `SupportTicketMessage`

| Field | Type | Note |
|---|---|---|
| `id` | `string` |  |
| `senderRole` | `'customer' \| 'admin'` |  |
| `senderId` | `EntityId` |  |
| `senderName` | `string` |  |
| `content` | `string` |  |
| `createdAt` | `string` |  |

### `SupportInternalNote`

| Field | Type | Note |
|---|---|---|
| `id` | `string` |  |
| `authorId` | `EntityId` |  |
| `authorName` | `string` |  |
| `content` | `string` |  |
| `createdAt` | `string` |  |

### `SupportTimelineEvent`

| Field | Type | Note |
|---|---|---|
| `id` | `string` |  |
| `type` | `SupportTimelineEventType` |  |
| `message` | `string` |  |
| `detail` | `string \| null` |  |
| `actorId` | `EntityId \| null` |  |
| `at` | `string` |  |

### `SupportTicketLinks`

| Field | Type | Note |
|---|---|---|
| `venueId` | `EntityId \| null` |  |
| `eventId` | `EntityId \| null` |  |
| `orderId` | `EntityId \| null` |  |
| `organizationId` | `EntityId \| null` |  |
| `userId` | `EntityId \| null` |  |

### `SupportSla`

| Field | Type | Note |
|---|---|---|
| `responseDueAt` | `string` |  |
| `resolutionDueAt` | `string` |  |
| `responseBreachedAt` | `string \| null` |  |
| `resolutionBreachedAt` | `string \| null` |  |

### `SupportTicket` (extends VersionedEntity)

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `subject` | `string` |  |
| `description` | `string` |  |
| `category` | `SupportTicketCategory` |  |
| `status` | `SupportTicketStatus` |  |
| `priority` | `SupportTicketPriority` |  |
| `requester` | `SupportTicketRequester` |  |
| `assignee` | `{ userId: EntityId; name: string } \| null` |  |
| `messages` | `SupportTicketMessage[]` |  |
| `internalNotes` | `SupportInternalNote[]` |  |
| `timeline` | `SupportTimelineEvent[]` |  |
| `links` | `SupportTicketLinks` |  |
| `sla` | `SupportSla` |  |
| `mergedInto` | `EntityId \| null` |  |
| `mergedFrom` | `EntityId[]` | Clients the primary absorbed — the mirror of `mergedInto` (v2 addition). |
| `resolvedAt` | `string \| null` |  |
| `resolvedBy` | `EntityId \| null` |  |
| `closedAt` | `string \| null` |  |
| `closedBy` | `EntityId \| null` |  |
| `deletedAt` | `string \| null` | Soft-delete attribution — never a hard delete (see `event.ts`). |
| `deletedBy` | `EntityId \| null` |  |

### `CreateSupportTicketInput`

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `subject` | `string` |  |
| `description` | `string` |  |
| `category` | `SupportTicketCategory` |  |
| `priority` | `SupportTicketPriority` |  |
| `requester` | `SupportTicketRequester` |  |
| `now?` | `Date` |  |

## `user-ban.ts`


### `UserBan` (extends VersionedEntity)

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` | Keyed by the user id — one ban record per user. |
| `userId` | `EntityId` |  |
| `isBanned` | `boolean` |  |
| `bannedAt` | `string \| null` |  |
| `bannedBy` | `EntityId \| null` |  |
| `banReason` | `string \| null` |  |

### `BanUserInput`

| Field | Type | Note |
|---|---|---|
| `bannedBy` | `EntityId` |  |
| `reason?` | `string` |  |
| `now?` | `Date` |  |

## `venue.ts`

- **type `VenueStatus`** = 'active' | 'suspended'
- **type `VenueSlotStatus`** = 'open' | 'booked' | 'blocked' | 'cancelled'
- **type `SlotRequestStatus`** = 'pending' | 'accepted' | 'rejected' | 'cancelled'

### `VenuePublicProfile`

| Field | Type | Note |
|---|---|---|
| `name` | `string` |  |
| `slug` | `string` |  |
| `description` | `string` |  |
| `shortDescription?` | `string` |  |
| `photoUrl` | `string \| null` |  |
| `address` | `VenueAddress` |  |
| `facilities` | `string[]` |  |
| `capacity` | `number \| null` |  |
| `menu` | `VenueMenu` | Food/drink offering shown to guests. Public by nature: it is menu copy, |
| `settings` | `VenuePublicSettings` | Public-facing config (guest list toggle etc.). |

### `VenueMenu`

| Field | Type | Note |
|---|---|---|
| `sections` | `VenueMenuSection[]` |  |
| `updatedAt` | `string \| null` | ISO-8601; lets a client show "updated on …" without another read. |

### `VenueMenuSection`

| Field | Type | Note |
|---|---|---|
| `name` | `string` |  |
| `items` | `VenueMenuItem[]` |  |

### `VenueMenuItem`

| Field | Type | Note |
|---|---|---|
| `name` | `string` |  |
| `description?` | `string` |  |
| `pricePaise` | `number \| null` | Integer paise, matching every other money field in the system. |
| `tags` | `string[]` | Dietary/allergen tags — presentation only, never a business rule. |

### `UpdateMenuInput`

| Field | Type | Note |
|---|---|---|
| `sections` | `VenueMenuSection[]` |  |
| `now?` | `Date` |  |

### `VenuePrivateProfile`

| Field | Type | Note |
|---|---|---|
| `contactEmail` | `string \| null` |  |
| `contactPhone` | `string \| null` |  |
| `socials` | `{ instagram?: string; website?: string; facebook?: string; }` |  |
| `internalNotes` | `string` | Internal notes — never exposed over public APIs. |

### `VenuePublicSettings`

| Field | Type | Note |
|---|---|---|
| `showGuestList` | `boolean` |  |
| `activityEnabled` | `boolean` |  |

### `VenueAddress`

| Field | Type | Note |
|---|---|---|
| `street?` | `string` |  |
| `city?` | `string` |  |
| `state?` | `string` |  |
| `zip?` | `string` |  |
| `country?` | `string` |  |
| `lat?` | `number` |  |
| `lng?` | `number` |  |

### `Venue` (extends VersionedEntity)

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `organizationId` | `EntityId` |  |
| `ownerId` | `EntityId` |  |
| `status` | `VenueStatus` |  |
| `public` | `VenuePublicProfile` |  |
| `private` | `VenuePrivateProfile` |  |

### `CreateVenueInput`

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `organizationId` | `EntityId` |  |
| `ownerId` | `EntityId` |  |
| `name` | `string` |  |
| `slug` | `string` |  |
| `description?` | `string` | V1-proven create fields (previous backend always wrote these at create). |
| `capacity?` | `number \| null` |  |
| `city?` | `string \| null` |  |
| `now?` | `Date` |  |

### `VenueUpdate`

| Field | Type | Note |
|---|---|---|
| `public?` | `Partial<VenuePublicProfile>` |  |
| `private?` | `Partial<VenuePrivateProfile>` |  |

### `VenueSlot` (extends VersionedEntity)

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `venueId` | `EntityId` |  |
| `label` | `string` |  |
| `startTime` | `string` |  |
| `endTime` | `string` |  |
| `recurring` | `boolean` |  |
| `status` | `VenueSlotStatus` |  |
| `capacityFor` | `number \| null` |  |

### `SlotRequest` (extends VersionedEntity)

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `venueId` | `EntityId` |  |
| `eventId` | `EntityId \| null` |  |
| `hostId` | `EntityId` |  |
| `status` | `SlotRequestStatus` |  |
| `message?` | `string` |  |

### `VenueAvailability`

| Field | Type | Note |
|---|---|---|
| `venueId` | `EntityId` |  |
| `from` | `string` |  |
| `to` | `string` |  |
| `openSlots` | `number` |  |
| `bookedSlots` | `number` |  |
| `blockedSlots` | `number` |  |
| `openMinutes` | `number` | Total minutes across open slots — the practical "how much can I book". |
| `fullyBooked` | `boolean` | True when nothing in the window can be booked. |
| `slots` | `VenueAvailabilitySlot[]` |  |

### `VenueAvailabilitySlot`

| Field | Type | Note |
|---|---|---|
| `id` | `EntityId` |  |
| `label` | `string` |  |
| `startTime` | `string` |  |
| `endTime` | `string` |  |
| `status` | `Exclude<VenueSlotStatus, 'cancelled'>` |  |
| `capacityFor` | `number \| null` |  |
