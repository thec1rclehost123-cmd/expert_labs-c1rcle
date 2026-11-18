# C1RCLE database schema list (Firestore)

> Generated from `origin/staging` @ `7c181a0` (2026-11-17). Datastore: **Cloud Firestore** via the Admin SDK (service account) with a `memory` driver for tests. `firestore.rules` is `allow read, write: if false` — no client SDK ever touches Firestore; everything goes through the api-gateway. All collections are top-level and prefixed `v2_`. Field lists are the TypeScript domain model each repository maps to (`packages/core/src/domain/models`); Firestore docs use the same names (entities that `extends VersionedEntity` also carry `version: number`, `createdAt: string` and `updatedAt: string` as ISO-8601 strings; a few read models use epoch-ms numbers instead). `Idx` = composite indexes declared in `firestore.indexes.json` (none = no composite index).

**95 collections** documented, **88 composite indexes** declared. Other stores: Redis (optional; shared rate-limit counters), Firebase Storage bucket (event/venue media, poster uploads; CORS via `configure:storage-cors`), Razorpay (payments), Resend (email).

## Conventions

- Aggregates carry a `version` and are written with compare-and-set transactions (`compare-and-set.ts`); a stale write returns 409 `version_conflict`.
- Money is integer paise (`Money`, `*Paise` fields); journals are double-entry with `AccountCode`/`JournalSide`.
- Uniqueness the database cannot enforce is done with claim/lookup docs keyed by the unique value (`*_active_keys`, `*_codes`, `*_vanity_aliases`, `v2_cover_wallet_by_event_user`).
- Side effects use the transactional outbox (`v2_outbox`), written inside the same Firestore transaction as the state change.
- Idempotency stores exist per command family (`v2_idempotency_records`, `*_idempotency`).

## Collection index

| Collection | Domain type | Composite indexes |
|---|---|---|
| `v2_auth_users` | PlatformUser | - |
| `v2_auth_sessions` | - | - |
| `v2_auth_accounts` | - | - |
| `v2_auth_verification_tokens` | - | - |
| `v2_staff_credentials` | - | - |
| `v2_email_otps` | EmailOtp | - |
| `v2_user_bans` | UserBan | - |
| `v2_admins` | PlatformAdmin | - |
| `v2_verification_attempts` | - | 2 |
| `v2_admin_audit_logs` | - | 1 |
| `v2_proposed_actions` | ProposedAction | 1 |
| `v2_organizations` | Organization | - |
| `v2_organization_invitations` | OrganizationInvitation | 1 |
| `v2_onboarding_requests` | OnboardingRequest | 4 |
| `v2_partnerships` | Partnership | 1 |
| `v2_promoter_connections` | PromoterConnection | 1 |
| `v2_platform_settings` | PlatformSettings | - |
| `v2_venues` | Venue | - |
| `v2_venue_slots` | VenueSlot | - |
| `v2_slot_requests` | SlotRequest | - |
| `v2_events` | Event | 1 |
| `v2_event_drafts` | EventDraft | - |
| `v2_event_catalog_tiers` | TicketTier | - |
| `v2_event_ticket_addons` | TicketAddon | - |
| `v2_event_catalog_promos` | PromoCode | - |
| `v2_event_catalog_tables` | TablePackage | - |
| `v2_event_catalog_promoter_assignments` | PromoterAssignment | 1 |
| `v2_promo_redemptions` | - | - |
| `v2_promo_redemption_locks` | - | - |
| `v2_event_social_policy` | - | - |
| `v2_promoter_links` | ReferralLink | 2 |
| `v2_promoter_tracking_codes` | - | - |
| `v2_promoter_codes` | - | - |
| `v2_promoter_assignment_codes` | - | - |
| `v2_promoter_event_code_claims` | - | - |
| `v2_promoter_vanity_aliases` | - | - |
| `v2_leaderboard_stats` | LeaderboardStat | 1 |
| `v2_cart_reservations` | CartReservation | 2 |
| `v2_hold_sequences` | - | - |
| `v2_orders` | Order | 4 |
| `v2_provider_payments` | ProviderPayment | - |
| `v2_provider_financial_events` | ProviderFinancialEvent | - |
| `v2_provider_financial_event_ids` | - | - |
| `v2_entitlements` | Entitlement | 4 |
| `v2_ticket_transfers` | TicketTransfer | 1 |
| `v2_ticket_access_codes` | TicketAccessCode | - |
| `v2_ticket_access_grants` | TicketAccessGrant | - |
| `v2_waitlist_entries` | WaitlistEntry | 3 |
| `v2_waitlist_sequences` | - | - |
| `v2_waitlist_active_keys` | - | - |
| `v2_complimentary_allocations` | ComplimentaryAllocation | - |
| `v2_complimentary_claims` | ComplimentaryAllocationClaim | - |
| `v2_refund_requests` | AdminRefundRequest | 2 |
| `v2_disputes` | Dispute | 3 |
| `v2_event_codes` | EventCode | 2 |
| `v2_scanner_sessions` | ScannerSession | 4 |
| `v2_scanner_session_tokens` | - | - |
| `v2_scanner_devices` | ScannerDevice | 1 |
| `v2_scan_ledger` | ScanLedger | 10 |
| `v2_door_sales` | DoorSale | 6 |
| `v2_door_sale_idempotency` | - | - |
| `v2_cover_wallets` | CoverWallet | 2 |
| `v2_cover_wallet_txns` | CoverWalletTxn | 4 |
| `v2_cover_wallet_by_event_user` | - | - |
| `v2_cover_wallet_idempotency` | - | - |
| `v2_cover_wallet_reconciliations` | CoverWalletReconciliation | 3 |
| `v2_event_cover_policies` | EventCoverPolicyProposal | - |
| `v2_event_cover_policy_acceptances` | EventCoverPolicyAcceptance | - |
| `v2_event_cover_accounts` | EventCoverAccount | - |
| `v2_event_cover_entries` | EventCoverEntry | - |
| `v2_event_cover_redemption_challenges` | EventCoverRedemptionChallenge | 1 |
| `v2_partner_ledger` | LedgerEntry | 1 |
| `v2_partner_ledger_idempotency` | - | - |
| `v2_financial_journals` | FinancialJournal | 1 |
| `v2_financial_journal_lines` | JournalLine | - |
| `v2_financial_journal_idempotency` | - | - |
| `v2_financial_journal_transition` | JournalTransitionState | - |
| `v2_payouts` | Payout | 2 |
| `v2_bank_accounts` | BankAccount | - |
| `v2_reconciliation_runs` | ReconciliationRun | - |
| `v2_reconciliation_discrepancies` | ReconciliationDiscrepancy | - |
| `v2_analytics_reads` | - | - |
| `v2_guest_profiles` | GuestProfile | - |
| `v2_guest_saved_events` | (subcollection) | - |
| `v2_follows` | Follow | 3 |
| `v2_guest_social_privacy` | GuestSocialPrivacy | - |
| `v2_guest_social_blocks` | - | - |
| `v2_guest_social_hidden_events` | - | - |
| `v2_social_notifications` | SocialNotification | 3 |
| `v2_notifications` | Notification | - |
| `v2_support_tickets` | SupportTicket | 4 |
| `v2_safety_reports` | SafetyReport | 4 |
| `v2_email_deliveries` | - | - |
| `v2_outbox` | - | 2 |
| `v2_idempotency_records` | - | - |

## Identity & access

### `v2_auth_users`

better-auth user (+additionalFields: role, etc.)

Domain type `PlatformUser` (`domain/models/platform-user.ts`):

```ts
id: EntityId;
email: string;
name: string;
image: string | null;
emailVerified: boolean;
role: string | null;
createdAt: number;
updatedAt: number;
```

### `v2_auth_sessions`

Better Auth sessions (adapter-managed; token, userId, expiresAt, ipAddress, userAgent)


### `v2_auth_accounts`

Better Auth credential/OAuth accounts (providerId, accountId, password hash, tokens)


### `v2_auth_verification_tokens`

Better Auth verification tokens (identifier, value, expiresAt)


### `v2_staff_credentials`

Staff credential rotation record (forced password rotation state)


### `v2_email_otps`

Guest email OTP (hashed code, attempts, lockout)

Domain type `EmailOtp` (`domain/models/email-otp.ts`):

```ts
recipient: string;
userId?: string;
codeHash: string;
expiresAt: string;
lastSentAt: string;
attempts: number;
```

### `v2_user_bans`

Domain type `UserBan` (`domain/models/user-ban.ts`) extends VersionedEntity:

```ts
id: EntityId;
userId: EntityId;
isBanned: boolean;
bannedAt: string | null;
bannedBy: EntityId | null;
banReason: string | null;
```

### `v2_admins`

Platform admin authority records

Domain type `PlatformAdmin` (`domain/models/admin-authority.ts`) extends VersionedEntity:

```ts
id: EntityId;
email: string;
role: AdminRole;
isActive: boolean;
```

### `v2_verification_attempts`

Applicant KYC verification attempts (userId, attemptedAt)


Indexes: `(userId, attemptedAt)`; `(userId, attemptedAt DESC)`

### `v2_admin_audit_logs`

Append-only admin audit log (targetId, actor, action, occurredAt)


Indexes: `(targetId, occurredAt DESC)`

### `v2_proposed_actions`

Maker-checker proposals for sensitive admin actions

Domain type `ProposedAction` (`domain/models/admin-authority.ts`) extends VersionedEntity:

```ts
id: EntityId;
action: AdminAction;
proposedBy: EntityId;
reason: string;
payload: Record<string, unknown>;
status: ProposalStatus;
resolvedBy: EntityId | null;
resolvedAt: string | null;
resolutionReason: string | null;
```

Indexes: `(status, createdAt)`

## Organizations & partnerships

### `v2_organizations`

Tenant (venue/host/promoter); embeds members

Domain type `Organization` (`domain/models/organization.ts`) extends VersionedEntity:

```ts
id: EntityId;
name: string;
slug: string;
ownerId: EntityId;
members: OrganizationMember[];
settings: OrganizationSettings;
status: OrganizationStatus;
platformFeePercent: number;
```

### `v2_organization_invitations`

Domain type `OrganizationInvitation` (`domain/models/organization.ts`) extends VersionedEntity:

```ts
id: EntityId;
organizationId: EntityId;
email: string;
role: OrganizationRole;
capabilities: Capability[];
status: InvitationStatus;
invitedBy: EntityId;
expiresAt: string;
acceptedAt: string | null;
acceptedBy: EntityId | null;
```

Indexes: `(organizationId, createdAt)`

### `v2_onboarding_requests`

Partner onboarding/KYC application

Domain type `OnboardingRequest` (`domain/models/onboarding.ts`) extends VersionedEntity:

```ts
id: EntityId;
userId: EntityId;
status: OnboardingStatus;
requestedType: PartnerEntityType;
plan: OnboardingPlan;
profile: OnboardingProfile;
documents: OnboardingDocument[];
submittedAt: string | null;
reviewedBy: EntityId | null;
reviewedAt: string | null;
reviewNote: string | null;
provisionedOrganizationId: EntityId | null;
promoterAgreementAcceptances?: PromoterAgreementAcceptance[];
```

Indexes: `(status, createdAt)`; `(userId, requestedType, createdAt DESC)`; `(userId, requestedType, status)`; `(userId, createdAt DESC)`

### `v2_partnerships`

Org<->org commission agreement; `partyIds` array for membership queries

Domain type `Partnership` (`domain/models/partnership.ts`) extends VersionedEntity:

```ts
id: EntityId;
hostOrganizationId: EntityId;
venueOrganizationId: EntityId;
venueId: EntityId;
initiatedBy: PartnershipInitiator;
status: PartnershipStatus;
message: string | null;
venueShareRate: number | null;
resolutionReason: string | null;
resolvedAt: string | null;
```

Indexes: `(partyIds, createdAt)`

### `v2_promoter_connections`

Domain type `PromoterConnection` (`domain/models/promoter-connection.ts`) extends VersionedEntity:

```ts
id: EntityId;
promoterId: EntityId;
targetId: EntityId;
targetType: ConnectionTargetType;
initiatedBy: ConnectionInitiator;
status: PromoterConnectionStatus;
message: string | null;
resolutionReason: string | null;
resolvedAt: string | null;
```

Indexes: `(partyIds, createdAt)`

### `v2_platform_settings`

Singleton platform config (fees, accounting policy)

Domain type `PlatformSettings` (`domain/models/platform-settings.ts`):

```ts
revision?: number;
bookingFeePolicy?: BookingFeePolicy;
platformFeeRate: number;
refundSingleApproverThresholdPaise: number;
refundDualApproverThresholdPaise: number;
maintenanceMode: boolean;
featureFlags: Record<string, boolean>;
accountingPolicy: AccountingPolicyConfig;
updatedAt: string;
```

## Venues, events & catalog

### `v2_venues`

Domain type `Venue` (`domain/models/venue.ts`) extends VersionedEntity:

```ts
id: EntityId;
organizationId: EntityId;
ownerId: EntityId;
status: VenueStatus;
public: VenuePublicProfile;
private: VenuePrivateProfile;
```

### `v2_venue_slots`

Domain type `VenueSlot` (`domain/models/venue.ts`) extends VersionedEntity:

```ts
id: EntityId;
venueId: EntityId;
label: string;
startTime: string;
endTime: string;
recurring: boolean;
status: VenueSlotStatus;
capacityFor: number | null;
timezone?: string | null;
notes?: string | null;
visibleToHostIds?: EntityId[] | null;
slotRequestId?: EntityId | null;
requestingHostId?: EntityId | null;
```

### `v2_slot_requests`

Domain type `SlotRequest` (`domain/models/venue.ts`) extends VersionedEntity:

```ts
id: EntityId;
venueId: EntityId;
slotId?: EntityId | null;
slotStartTime?: string | null;
slotEndTime?: string | null;
timezone?: string | null;
eventId: EntityId | null;
hostId: EntityId;
status: SlotRequestStatus;
message?: string;
eventDraft?: { event: Record<string, unknown>; tiers: { localId: EntityId; command: Record<string, unknown> }[]; promotions: Record<string, unknown>[]; promoters: Record<string, unknown>[]; };
```

### `v2_events`

Aggregate root with version for compare-and-set

Domain type `Event` (`domain/models/event.ts`) extends VersionedEntity:

```ts
id: EntityId;
organizationId: EntityId;
venueId: EntityId | null;
externalLocation?: ExternalEventLocation;
slug: string;
title: string;
summary: string;
description: string;
imageUrl: string | null;
privateImageStoragePath?: string | null;
startAt: string;
endAt: string | null;
lastMaterialChangeAt?: string;
status: EventStatus;
visibilityMode?: EventVisibilityMode;
addressDisclosure?: EventAddressDisclosure;
isPublic: boolean;
tags: string[];
startingPricePaise: number | null;
isFree: boolean;
cancellationReason: string | null;
postponement?: { reason: string; initiatedBy: 'host' | 'venue'; organizationId: EntityId; at: string; } | null;
refundPolicy?: EventRefundPolicy;
compensation?: EventCompensation | null;
entryRules?: EventEntryRules;
media?: EventMedia;
capacity: number | null;
adminOverride: boolean;
```

Indexes: `(isPublic, startAt)`

### `v2_event_drafts`

Domain type `EventDraft` (`domain/models/event-draft.ts`):

```ts
readonly id: string;
readonly userId: EntityId;
readonly organizationId: EntityId;
readonly schemaVersion: 1;
readonly data: Record<string, unknown>;
readonly updatedAt: string;
```

### `v2_event_catalog_tiers`

Domain type `TicketTier` (`domain/models/event-catalog.ts`) extends VersionedEntity:

```ts
id: EntityId;
eventId: EntityId;
organizationId: EntityId;
name: string;
description: string;
entryType: string;
currency: string;
priceInPaise: number;
coverAmountPaise?: number;
quantity: number;
status: TicketTierStatus;
salesStartAt: string | null;
salesEndAt: string | null;
maxPerOrder: number | null;
accessMode?: TicketTierAccessMode;
accessType?: TicketAccessType;
audienceType?: TicketAudienceType;
guestCount?: number;
admissionMode?: TicketAdmissionMode;
pricingPhases?: TicketPricingPhase[];
doorPriceInPaise?: number | null;
benefits?: string[];
minAge?: number | null;
maxAge?: number | null;
minPerOrder?: number | null;
maxPerUser?: number | null;
tableConfig?: { capacity: number; minimumSpendPaise: number; redeemableAmountPaise: number; tableCount: number; } | null;
commissionEligible?: boolean;
```

### `v2_event_ticket_addons`

Domain type `TicketAddon` (`domain/models/ticket-addon.ts`) extends VersionedEntity:

```ts
id: EntityId;
eventId: EntityId;
organizationId: EntityId;
name: string;
description: string;
kind: TicketAddonKind;
currency: 'INR';
priceInPaise: number;
quantity: number;
maxPerEntitlement: number;
salesStartAt: string | null;
salesEndAt: string | null;
status: TicketAddonStatus;
```

### `v2_event_catalog_promos`

Domain type `PromoCode` (`domain/models/event-catalog.ts`) extends VersionedEntity:

```ts
id: EntityId;
eventId: EntityId | null;
organizationId: EntityId;
code: string;
name: string;
type: PromoType;
discountType: PromoDiscountType;
discountValue: number;
discountFundingOwner: DiscountFundingOwner;
tierIds: EntityId[];
maxRedemptions: number | null;
maxPerUser: number | null;
redemptionCount: number;
startsAt: string | null;
endsAt: string | null;
isActive: boolean;
```

### `v2_event_catalog_tables`

Domain type `TablePackage` (`domain/models/event-catalog.ts`) extends VersionedEntity:

```ts
id: EntityId;
eventId: EntityId;
organizationId: EntityId;
name: string;
minSpendPaise: number | null;
capacity: number;
pricePaise: number;
isActive: boolean;
```

### `v2_event_catalog_promoter_assignments`

Domain type `PromoterAssignment` (`domain/models/event-catalog.ts`) extends VersionedEntity:

```ts
id: EntityId;
eventId: EntityId;
promoterId: EntityId;
status: PromoterAssignmentStatus;
terms: CommissionTerms;
createdAt: string;
endedAt: string | null;
suspendedAt: string | null;
suspendedFromStatus: 'invited' | 'accepted' | 'active' | null;
invitedAt: string | null;
invitedBy: EntityId | null;
respondedAt: string | null;
respondedBy: EntityId | null;
acceptedAt: string | null;
rejectedAt: string | null;
revokedAt: string | null;
revokedBy: EntityId | null;
revocationReason: string | null;
startsAt: string | null;
expiresAt: string | null;
eligibleTierIds: EntityId[] | null;
```

Indexes: `(status, expiresAt)`

### `v2_promo_redemptions`

Promo redemption counters per user/code


### `v2_promo_redemption_locks`

Lock docs guarding concurrent promo redemption


### `v2_event_social_policy`

`{ enabled: boolean }` per event


## Promoters

### `v2_promoter_links`

Domain type `ReferralLink` (`domain/models/referral-link.ts`) extends VersionedEntity:

```ts
id: EntityId;
eventId: EntityId;
promoterId: EntityId;
organizationId: EntityId;
assignmentId: EntityId | null;
assignmentVersion: number | null;
termsSnapshot: CommissionTerms | null;
commissionPolicyVersion: string | null;
commercialPolicyVersion: string | null;
eligibleTierIds: EntityId[] | null;
attributionSignature: string | null;
eventTitle: string;
campaignLabel: string;
vanityPrefix: string;
vanitySlug: string | null;
code: string;
label: string;
isActive: boolean;
clicks: number;
lastClickedAt: string | null;
conversions: number;
revenuePaise: number;
commissionPaise: number;
```

Indexes: `(eventId, createdAt)`; `(promoterId, createdAt)`

### `v2_promoter_tracking_codes`

Unique-code claim docs for referral tracking codes


### `v2_promoter_codes`

Per-promoter code registry


### `v2_promoter_assignment_codes`

Code claim docs per promoter assignment


### `v2_promoter_event_code_claims`

Per-event code claims


### `v2_promoter_vanity_aliases`

Vanity alias claims (uniqueness guard)


### `v2_leaderboard_stats`

Read model keyed by period/city

Domain type `LeaderboardStat` (`domain/models/leaderboard.ts`):

```ts
promoterId: string;
periodType: LeaderboardPeriodType;
periodValue: string;
city: string;
totalCommissionEarnedPaise: number;
updatedAt: string;
```

Indexes: `(periodType, periodValue, city, totalCommissionEarnedPaise DESC)`

## Commerce, tickets & refunds

### `v2_cart_reservations`

Inventory holds; expire via expiresAt

Domain type `CartReservation` (`domain/models/cart-reservation.ts`) extends VersionedEntity:

```ts
saleAllocation?: SaleAllocationSnapshot;
id: EntityId;
eventId: EntityId;
organizationId: EntityId;
userId: EntityId | null;
lines: { tierId: EntityId; tierName: string; pricingPhaseId?: string | null; quantity: number; unitPricePaise: number; admissionUnitPricePaise?: number; coverUnitPricePaise?: number; coverPolicy?: { proposalId: EntityId; policyVersion: string; termsDigest: string }; ticketAddons?: TicketAddonSelection[]; accessGrantId?: EntityId; guestCount?: number; admissionMode?: TicketAdmissionMode; eligibility?: { audienceType: string; minAge: number | null; maxAge: number | null; attested: true; attestedAt: string; }; }[];
pricing: PricingBreakdown;
appliedPromoCode: string | null;
attribution: OrderAttribution | null;
status: CartReservationStatus;
expiresAt: string;
convertedOrderId: EntityId | null;
providerOrderId?: string | null;
providerOrderCreation?: { receipt: string; amountPaise: number; currency: string; holdId: EntityId; eventId: EntityId; organizationId: EntityId; claimedAt: string; state: 'creating' | 'requires_support' | 'bound'; } | null;
attendee?: OrderContact | null;
upgrade?: TicketUpgradeSnapshot;
sequence?: number | null;
idempotencyKey: string;
```

Indexes: `(status, expiresAt)`; `(eventId, status, expiresAt)`

### `v2_hold_sequences`

Per-event hold sequence counter (doc id = eventId)


### `v2_orders`

Domain type `Order` (`domain/models/order.ts`) extends VersionedEntity:

```ts
saleAllocation?: SaleAllocationSnapshot;
bookedAt?: string;
fulfillmentState?: 'pending' | 'complete';
compensationRequired?: { reason: 'capacity_unavailable' | 'promotion_unavailable' | 'event_unavailable'; requestedAt: string; };
fulfillmentCompletedAt?: string;
receiptFacts?: OrderReceiptFacts;
feePolicy?: BookingFeePolicy;
accountingPolicy?: AccountingPolicySnapshot;
refundSnapshot?: OriginalPurchaseRefundSnapshot;
id: EntityId;
bookingReference?: string;
eventId: EntityId;
organizationId: EntityId;
userId: EntityId | null;
contact: OrderContact;
status: OrderStatus;
lines: OrderLine[];
currency: string;
subtotalPaise: number;
discountPaise: number;
discountFundingOwner: DiscountFundingOwner | null;
discountedSubtotalPaise: number;
platformFeePaise: number;
paymentFeePaise: number;
gstPaise: number;
grandTotalPaise: number;
admissionSubtotalPaise?: number;
coverSubtotalPaise?: number;
addOnSubtotalPaise?: number;
discountedAdmissionSubtotalPaise?: number;
appliedPromoCode: string | null;
attribution: OrderAttribution | null;
paymentIntentId: string | null;
paymentId: string | null;
paidAt: string | null;
reservationExpiresAt: string;
failureReason: string | null;
eventCancellation?: OrderEventCancellation | null;
refundedPaise: number;
appliedRefundRequestIds: EntityId[];
refundedTicketClaims?: RefundedTicketClaim[];
```

Indexes: `(userId, createdAt DESC)`; `(organizationId, createdAt DESC)`; `(eventId, createdAt DESC)`; `(eventId, attribution.promoterId)`

### `v2_provider_payments`

Razorpay payment mirror

Domain type `ProviderPayment` (`domain/models/provider-payment.ts`) extends VersionedEntity:

```ts
id: EntityId;
provider: 'razorpay';
providerOrderId: string | null;
orderId: EntityId | null;
processingState: 'requires_support' | 'processing' | 'fulfilled' | 'refunded';
currency: string;
capturedPaise: number;
feePaise: number;
taxOnFeePaise: number;
capturedAt: string | null;
refunds: Record<string, number>;
disputes: Record<string, ProviderDispute>;
```

### `v2_provider_financial_events`

Normalised webhook events

Domain type `ProviderFinancialEvent` (`domain/models/provider-financial-event.ts`):

```ts
readonly id: EntityId;
readonly provider: 'razorpay';
readonly providerEventId: string;
readonly eventType: string;
readonly paymentId: string;
readonly providerOrderId: string | null;
readonly refundId: string | null;
readonly disputeId: string | null;
readonly receivedAt: string;
readonly providerOccurredAt: string | null;
readonly payloadHash: string;
readonly trustedIngestion: ProviderEventTrust;
readonly replayOfEventId: EntityId | null;
readonly processingState: ProviderEventProcessingState;
readonly processingAttempts: number;
readonly finalResult: string | null;
readonly processedAt: string | null;
readonly version: number;
```

### `v2_provider_financial_event_ids`

Provider event id dedupe docs


### `v2_entitlements`

One doc per ticket seat/unit

Domain type `Entitlement` (`domain/models/entitlement.ts`) extends VersionedEntity:

```ts
fulfillmentRequired?: boolean;
id: EntityId;
orderId: EntityId;
eventId: EntityId;
organizationId: EntityId;
tierId: EntityId;
tierName: string;
userId: EntityId | null;
unitKind?: EntitlementUnitKind;
currentHolderUserId?: EntityId | null;
credentialVersion?: number;
upgradeFromEntitlementId?: EntityId;
upgradedToEntitlementId?: EntityId;
upgradeRefundRequestId?: EntityId;
pendingTransferId?: EntityId | null;
eligibility?: { audienceType: string; minAge: number | null; maxAge: number | null; attested: true; attestedAt: string; };
holderName: string;
status: EntitlementStatus;
scanCountAllowed: number;
scanCount: number;
scannedAt: string[];
refundLock?: { refundRequestId: EntityId; frozenAt: string; } | null;
ticketAddons?: TicketAddonEntitlement[];
```

Indexes: `(userId, eventId)`; `(userId, createdAt DESC)`; `(eventId, createdAt DESC)`; `(organizationId, createdAt DESC)`

### `v2_ticket_transfers`

Domain type `TicketTransfer` (`domain/models/ticket-transfer.ts`) extends VersionedEntity:

```ts
id: EntityId;
entitlementId: EntityId;
eventId: EntityId;
senderUserId: EntityId;
recipientUserId: EntityId | null;
recipientEmailHash?: string | null;
recipientEmail?: string | null;
status: TicketTransferStatus;
expiresAt: string;
acceptedAt: string | null;
resolvedAt: string | null;
```

Indexes: `(status, expiresAt)`

### `v2_ticket_access_codes`

Domain type `TicketAccessCode` (`domain/models/ticket-access.ts`) extends VersionedEntity:

```ts
id: EntityId;
kind: 'code' | 'invitation';
scope?: TicketAccessScope;
eventId: EntityId;
organizationId: EntityId;
tierIds: EntityId[];
recipientEmailHash: string | null;
createdBy: EntityId;
status: TicketAccessCodeStatus;
startsAt: string;
expiresAt: string;
maxRedemptions: number;
redemptionCount: number;
revokedAt: string | null;
revokedBy: EntityId | null;
revokedReason: string | null;
```

### `v2_ticket_access_grants`

Domain type `TicketAccessGrant` (`domain/models/ticket-access.ts`) extends VersionedEntity:

```ts
id: EntityId;
codeId: EntityId;
scope?: TicketAccessScope;
eventId: EntityId;
organizationId: EntityId;
userId: EntityId;
tierIds: EntityId[];
expiresAt: string;
```

### `v2_waitlist_entries`

Domain type `WaitlistEntry` (`domain/models/waitlist-entry.ts`) extends VersionedEntity:

```ts
id: EntityId;
eventId: EntityId;
organizationId: EntityId;
tierId: EntityId;
userId: EntityId;
accessGrantId: EntityId | null;
eligibilityAttested: boolean;
eligibilityAttestedAt: string | null;
sequence: number;
status: WaitlistStatus;
offerReservationId: EntityId | null;
offeredAt: string | null;
offerExpiresAt: string | null;
resolvedAt: string | null;
```

Indexes: `(status, offerExpiresAt)`; `(eventId, tierId, sequence)`; `(eventId, tierId, status, sequence)`

### `v2_waitlist_sequences`

Per-tier waitlist sequence counter


### `v2_waitlist_active_keys`

Uniqueness guard: one active waitlist entry per user+tier


### `v2_complimentary_allocations`

Domain type `ComplimentaryAllocation` (`domain/models/complimentary-allocation.ts`) extends VersionedEntity:

```ts
id: EntityId;
eventId: EntityId;
organizationId: EntityId;
tierId: EntityId;
category: ComplimentaryCategory;
reason: string;
quota: number;
claimedCount: number;
status: 'active' | 'closed';
expiresAt: string | null;
createdBy: EntityId;
```

### `v2_complimentary_claims`

Domain type `ComplimentaryAllocationClaim` (`domain/models/complimentary-allocation.ts`) extends VersionedEntity:

```ts
id: EntityId;
allocationId: EntityId;
eventId: EntityId;
organizationId: EntityId;
tierId: EntityId;
recipientUserId: EntityId;
holderName: string;
status: 'issued' | 'revoked';
orderId: EntityId;
issuedBy: EntityId;
issuedAt: string;
revokedBy: EntityId | null;
revokedAt: string | null;
revocationReason: string | null;
```

### `v2_refund_requests`

Domain type `AdminRefundRequest` (`domain/models/refund-request.ts`) extends VersionedEntity:

```ts
executionActor?: { adminId: EntityId; adminRole: AdminRole; source: 'execution_claim' | 'legacy_recovery'; authorizedAt: string; ipAddress?: string; userAgent?: string; };
settlementAudit?: { requestStatusBefore: string; orderStatusBefore: string; orderStatusAfter: string; occurredAt: number; };
id: EntityId;
readonly intakeFingerprint?: string;
readonly sourceEventVersion?: number;
readonly remedyForRefundRequestId?: EntityId;
readonly refundCause?: RefundCause;
readonly refundComponents?: Readonly<RefundComponents>;
readonly purchasePolicyVersion?: string;
orderId: EntityId;
organizationId: EntityId;
amountPaise: number;
requestedBy: EntityId;
reason: string;
scope: RefundScope;
ticketLines: RefundTicketLine[];
targetEntitlementIds: EntityId[];
approversRequired: number;
approvals: AdminRefundApproval[];
status: AdminRefundRequestStatus;
rejectedBy: EntityId | null;
rejectionReason: string | null;
providerRefundId: string | null;
providerStatus: string | null;
executionIdempotencyKey: string;
executionAttempts: number;
failureReason: string | null;
```

Indexes: `(status, createdAt DESC)`; `(orderId, createdAt DESC)`

### `v2_disputes`

Domain type `Dispute` (`domain/models/dispute.ts`) extends VersionedEntity:

```ts
id: EntityId;
organizationId: EntityId;
orderId: EntityId;
ledgerEntryId: EntityId | null;
journalId?: EntityId | null;
resolutionIntent?: { outcome: DisputeResolutionOutcome; resolutionNote: string; claimedBy: EntityId; claimedRole: string; ipAddress?: string; userAgent?: string; claimedAt: string; journalId: EntityId | null; };
raisedBy: EntityId;
reason: string;
amount: number;
status: DisputeStatus;
resolutionNote: string | null;
resolvedAt: string | null;
resolution: DisputeResolutionOutcome | null;
```

Indexes: `(organizationId, createdAt DESC)`; `(organizationId, status, createdAt DESC)`; `(status, createdAt DESC)`

## Door & scanning

### `v2_event_codes`

Domain type `EventCode` (`domain/models/event-code.ts`) extends VersionedEntity:

```ts
id: EntityId;
code: string;
eventId: EntityId;
organizationId: EntityId;
venueId: EntityId | null;
type: EventCodeType;
gate: string | null;
createdBy: EntityId | null;
createdByName: string | null;
status: EventCodeStatus;
revokedAt: string | null;
revokedReason: string | null;
expiresAt: string | null;
stats: EventCodeStats;
maxDevices: number;
allowReuse: boolean;
```

Indexes: `(eventId, createdAt DESC)`; `(organizationId, createdAt DESC)`

### `v2_scanner_sessions`

Domain type `ScannerSession` (`domain/models/event-code.ts`) extends VersionedEntity:

```ts
id: EntityId;
sessionToken: string | null;
codeId: EntityId;
eventId: EntityId;
organizationId: EntityId;
venueId: EntityId | null;
type: ScannerSessionType;
deviceId: string | null;
deviceName: string | null;
expiresAt: string;
lastUsedAt: string | null;
revokedAt: string | null;
revokedReason: string | null;
permissions: SessionPermissions;
createdBy: EntityId;
createdByName: string | null;
```

Indexes: `(codeId, createdAt DESC)`; `(codeId, revokedAt, expiresAt)`; `(deviceId, createdAt DESC)`; `(revokedAt, expiresAt)`

### `v2_scanner_session_tokens`

Hashed scanner session tokens


### `v2_scanner_devices`

Domain type `ScannerDevice` (`domain/models/scanner-device.ts`) extends VersionedEntity:

```ts
id: EntityId;
organizationId: EntityId;
venueId: EntityId | null;
deviceId: string;
deviceName: string;
status: ScannerDeviceStatus;
boundBy: EntityId;
boundAt: string;
unboundAt: string | null;
unboundReason: string | null;
lastSeenAt: string;
lastEventId: EntityId | null;
lastGate: string | null;
scanCount: number;
lastScanAt: string | null;
lastScanResult: string | null;
```

Indexes: `(organizationId, lastSeenAt DESC)`

### `v2_scan_ledger`

Append-only admission/deny record

Domain type `ScanLedger` (`domain/models/scan-ledger.ts`) extends VersionedEntity:

```ts
id: EntityId;
eventId: EntityId;
organizationId: EntityId;
venueId: EntityId | null;
entitlementId: EntityId | null;
doorSaleId: EntityId | null;
entryType: string | null;
tierName: string | null;
tierId: EntityId | null;
operatorUid: string | null;
operatorName: string | null;
operatorRole: string | null;
gate: string | null;
deviceId: string | null;
deviceName: string | null;
deviceBound: boolean;
status: ScanLedgerStatus;
denyReason: ScanDenyReason | null;
denyMessage: string | null;
guestName: string | null;
guestEmail: string | null;
guestPhone: string | null;
scannedAt: string;
admittedCount: number;
scanCountUsed: number | null;
scanCountAllowed: number | null;
isOffline: boolean;
syncedAt: string | null;
offlineDeviceId: string | null;
overriddenBy: string | null;
overrideReason: string | null;
```

Indexes: `(eventId, scannedAt DESC)`; `(eventId, admittedCount)`; `(eventId, tierName, admittedCount)`; `(organizationId, scannedAt DESC)`; `(deviceId, scannedAt DESC)`; `(operatorUid, scannedAt DESC)`; `(eventId, isOffline, scannedAt)`; `(eventId, entitlementId)`; `(eventId, status)`; `(entitlementId, status)`

### `v2_door_sales`

Domain type `DoorSale` (`domain/models/door-sale.ts`) extends VersionedEntity:

```ts
id: EntityId;
eventId: EntityId;
organizationId: EntityId;
venueId: EntityId | null;
category: DoorSaleCategory;
guestName: string;
guestPhone: string | null;
guestAge: number | null;
gender: string | null;
guestEmail: string | null;
totalGuests: number;
tableNumber: string | null;
gate: string | null;
paymentMode: DoorSalePaymentMode;
amountPaise: number;
paymentStatus: 'collected' | 'pending' | 'failed';
paymentRef: string | null;
createdBy: EntityId;
createdByName: string | null;
status: DoorSaleStatus;
voidedAt: string | null;
voidedBy: EntityId | null;
voidReason: string | null;
refundedAmountPaise: number | null;
refundedAt: string | null;
refundedBy: EntityId | null;
idempotencyKey: string | null;
```

Indexes: `(eventId, createdAt DESC)`; `(organizationId, createdAt DESC)`; `(venueId, createdAt DESC)`; `(category, createdAt DESC)`; `(createdBy, createdAt DESC)`; `(organizationId, createdAt)`

### `v2_door_sale_idempotency`

Door-sale idempotency records


## Cover wallet

### `v2_cover_wallets`

Domain type `CoverWallet` (`domain/models/cover-wallet.ts`) extends VersionedEntity:

```ts
id: EntityId;
userId: EntityId;
eventId: EntityId;
organizationId: EntityId;
venueId: EntityId | null;
balance: number;
openingBalance: number;
totalCredits: number;
totalDebits: number;
totalRefunds: number;
status: CoverWalletStatus;
terminatedAt: string | null;
terminationReason: string | null;
lastTxnAt: string | null;
lastCreditAt: string | null;
lastDebitAt: string | null;
metadata: Record<string, unknown>;
rules: CoverWalletRules;
```

Indexes: `(eventId, createdAt DESC)`; `(organizationId, createdAt DESC)`

### `v2_cover_wallet_txns`

Domain type `CoverWalletTxn` (`domain/models/cover-wallet.ts`) extends VersionedEntity:

```ts
id: EntityId;
walletId: EntityId;
eventId: EntityId;
organizationId: EntityId;
venueId: EntityId | null;
userId: EntityId;
type: CoverWalletTxnType;
amount: number;
balanceAfter: number;
status: CoverWalletTxnStatus;
idempotencyKey: string;
referenceId: EntityId | null;
referenceType: string | null;
deviceId: string | null;
operatorUid: EntityId | null;
operatorName: string | null;
description: string | null;
failureReason: string | null;
processedAt: string | null;
```

Indexes: `(deviceId, type, createdAt)`; `(walletId, createdAt DESC)`; `(eventId, createdAt DESC)`; `(type, createdAt DESC)`

### `v2_cover_wallet_by_event_user`

Lookup doc (eventId+userId -> walletId) enforcing one wallet per guest per event


### `v2_cover_wallet_idempotency`

Cover-wallet idempotency records


### `v2_cover_wallet_reconciliations`

Domain type `CoverWalletReconciliation` (`domain/models/cover-wallet-reconciliation.ts`) extends VersionedEntity:

```ts
id: EntityId;
eventId: EntityId;
organizationId: EntityId;
venueId: EntityId | null;
reconciliationDate: string;
walletId: EntityId | null;
userId: EntityId | null;
status: ReconciliationStatus;
expectedBalance: number;
actualBalance: number;
discrepancy: number;
periodCredits: number;
periodDebits: number;
periodRefunds: number;
periodTxnCount: number;
discrepancies: CoverWalletReconciliationDiscrepancy[];
resolvedBy: EntityId | null;
resolvedAt: string | null;
resolutionNotes: string | null;
```

Indexes: `(eventId, createdAt DESC)`; `(organizationId, createdAt DESC)`; `(organizationId, createdAt)`

### `v2_event_cover_policies`

Domain type `EventCoverPolicyProposal` (`domain/models/event-cover.ts`) extends VersionedEntity:

```ts
id: EntityId;
hostOrganizationId: EntityId;
venueOrganizationId: EntityId;
proposedByUserId: EntityId;
policyVersion: string;
terms: EventCoverPolicyTerms;
termsDigest: string;
```

### `v2_event_cover_policy_acceptances`

Domain type `EventCoverPolicyAcceptance` (`domain/models/event-cover.ts`) extends VersionedEntity:

```ts
id: EntityId;
proposalId: EntityId;
eventId: EntityId;
hostOrganizationId: EntityId;
venueOrganizationId: EntityId;
acceptedByUserId: EntityId;
proposalVersion: string;
termsDigest: string;
```

### `v2_event_cover_accounts`

Domain type `EventCoverAccount` (`domain/models/event-cover.ts`) extends VersionedEntity:

```ts
id: EntityId;
entitlementId: EntityId;
orderId: EntityId;
eventId: EntityId;
guestUserId: EntityId | null;
hostOrganizationId: EntityId;
venueOrganizationId: EntityId;
currency: 'INR';
issuedPaise: number;
remainingPaise: number;
reservedPaise: number;
redeemedPaise: number;
refundedPaise: number;
status: EventCoverStatus;
refundRequestId?: EntityId | null;
refundPolicy: EventCoverRefundPolicy;
policyVersion: string;
```

### `v2_event_cover_entries`

Domain type `EventCoverEntry` (`domain/models/event-cover.ts`) extends VersionedEntity:

```ts
id: EntityId;
accountId: EntityId;
entitlementId: EntityId;
orderId: EntityId;
eventId: EntityId;
venueOrganizationId: EntityId;
type: EventCoverEntryType;
amountPaise: number;
operationId: EntityId;
actorUserId: EntityId | null;
terminalId: EntityId | null;
receiptReference: string | null;
policyVersion: string;
reversesEntryId: EntityId | null;
reason?: string;
```

### `v2_event_cover_redemption_challenges`

Domain type `EventCoverRedemptionChallenge` (`domain/models/event-cover.ts`) extends VersionedEntity:

```ts
id: EntityId;
accountId: EntityId;
entitlementId: EntityId;
eventId: EntityId;
venueOrganizationId: EntityId;
guestUserId: EntityId;
initiatedByUserId: EntityId;
terminalId: EntityId;
amountPaise: number;
grossSpendPaise: number;
categories: readonly string[];
containsAlcohol: boolean;
receiptReference: string;
status: EventCoverRedemptionChallengeStatus;
expiresAt: string;
confirmedAt: string | null;
consumedAt: string | null;
```

Indexes: `(status, expiresAt)`

## Finance

### `v2_partner_ledger`

Per-org ledger

Domain type `LedgerEntry` (`domain/models/ledger.ts`) extends VersionedEntity:

```ts
id: EntityId;
organizationId: EntityId;
orderId: EntityId;
eventId: EntityId;
entryType: LedgerEntryType;
amount: number;
status: LedgerEntryStatus;
idempotencyKey: string;
createdAt: string;
updatedAt: string;
```

Indexes: `(organizationId, createdAt DESC)`

### `v2_partner_ledger_idempotency`

Ledger posting idempotency


### `v2_financial_journals`

Double-entry journal header

Domain type `FinancialJournal` (`domain/models/financial-journal.ts`):

```ts
readonly id: EntityId;
readonly organizationId: EntityId;
readonly currency: CurrencyCode;
readonly lines: readonly JournalLine[];
readonly source: PostingSource;
readonly idempotencyKey: string;
readonly accountingPolicy: AccountingPolicySnapshot;
readonly components: FinancialComponentBreakdown | null;
readonly reversalOfJournalId: EntityId | null;
readonly evidenceOnly: boolean;
readonly status: 'posted';
readonly postedAt: string;
readonly createdAt: string;
```

Indexes: `(organizationId, postedAt DESC)`

### `v2_financial_journal_lines`

Journal lines (debit/credit)

Domain type `JournalLine` (`domain/models/financial-journal.ts`):

```ts
readonly id: EntityId;
readonly account: AccountCode;
readonly side: JournalSide;
readonly amountPaise: number;
readonly currency: CurrencyCode;
readonly organizationId: EntityId | null;
readonly component: keyof FinancialComponentBreakdown | null;
readonly memo: string;
```

### `v2_financial_journal_idempotency`

Journal posting idempotency


### `v2_financial_journal_transition`

Read-model cutover state (legacy ledger -> journal)

Domain type `JournalTransitionState` (`domain/models/journal-transition.ts`):

```ts
readonly id: 'singleton';
readonly phase: JournalTransitionPhase;
readonly readModel: FinanceReadModelVersion;
readonly checkpointOrganizationId: EntityId | null;
readonly backfilledEntries: number;
readonly backfillCompletedAt: string | null;
readonly comparisonCompletedAt: string | null;
readonly discrepancies: readonly JournalTransitionDiscrepancy[];
readonly cutoverAt: string | null;
readonly rollbackAt: string | null;
readonly updatedAt: string;
readonly version: number;
```

### `v2_payouts`

Domain type `Payout` (`domain/models/payout.ts`) extends VersionedEntity:

```ts
id: EntityId;
organizationId: EntityId;
bankAccountId: EntityId;
amount: number;
status: PayoutStatus;
failureReason: string | null;
requestedBy: EntityId;
processedAt: string | null;
transferReference: string | null;
transferredAt: string | null;
transferMethod: string | null;
transferProof: PayoutTransferProof | null;
approvedBy: EntityId | null;
previousStatus: PayoutStatus | null;
```

Indexes: `(organizationId, createdAt DESC)`; `(status, createdAt DESC)`

### `v2_bank_accounts`

Account number sealed with ENCRYPTION_KEY

Domain type `BankAccount` (`domain/models/bank-account.ts`) extends VersionedEntity:

```ts
id: EntityId;
organizationId: EntityId;
bankName: string;
accountHolder: string;
last4: string;
encryptedAccountNumber: string;
ifscCode: string;
isDefault: boolean;
verified: boolean;
```

### `v2_reconciliation_runs`

Domain type `ReconciliationRun` (`domain/models/reconciliation.ts`) extends VersionedEntity:

```ts
id: EntityId;
status: ReconciliationRunStatus;
requestedBy: EntityId;
startedAt: string;
completedAt: string | null;
scannedCounts: Record<string, number>;
discrepancyCount: number;
failureReason: string | null;
```

### `v2_reconciliation_discrepancies`

Domain type `ReconciliationDiscrepancy` (`domain/models/reconciliation.ts`) extends VersionedEntity:

```ts
id: EntityId;
runId: EntityId;
kind: ReconciliationDiscrepancyKind;
state: ReconciliationDiscrepancyState;
organizationId: EntityId | null;
orderId: EntityId | null;
paymentId: string | null;
payoutId: EntityId | null;
amountPaise: number | null;
currency: string | null;
evidence: Record<string, string | number | boolean | null>;
repair: 'recheck_evidence' | null;
resolutionReason: string | null;
resolvedBy: EntityId | null;
resolvedAt: string | null;
```

### `v2_analytics_reads`

Denormalised analytics read model (backfill-analytics script)


## Guest, social, support & platform

### `v2_guest_profiles`

Domain type `GuestProfile` (`domain/models/guest-profile.ts`) extends VersionedEntity:

```ts
userId: EntityId;
displayName: string;
dateOfBirth: string;
city: string;
tastes: string[];
intents: string[];
phoneNumber?: string;
phoneVerifiedAt?: string;
```

### `v2_guest_saved_events`

`v2_guest_saved_events/{userId}/events/{eventId}`

Domain type: (subcollection)

### `v2_follows`

Domain type `Follow` (`domain/models/social.ts`):

```ts
id: EntityId;
followerId: EntityId;
targetType: FollowTargetType;
targetId: EntityId;
createdAt: string;
```

Indexes: `(followerId, createdAt DESC)`; `(followerId, targetType, createdAt DESC)`; `(targetType, targetId, createdAt DESC)`

### `v2_guest_social_privacy`

Domain type `GuestSocialPrivacy` (`domain/models/guest-social.ts`):

```ts
profileVisible: boolean;
showOnGuestlists: boolean;
showUpcomingEvents: boolean;
```

### `v2_guest_social_blocks`

Block edges, doc id = `{blocker}:{blocked}` (userId, targetId)


### `v2_guest_social_hidden_events`

Hidden-event edges (userId, eventId)


### `v2_social_notifications`

Guest-facing inbox

Domain type `SocialNotification` (`domain/models/social.ts`):

```ts
id: EntityId;
userId: EntityId;
type: NotificationType;
title: string;
body: string;
link: string | null;
subjectId: EntityId;
createdAt: string;
readAt: string | null;
eventChange?: { eventVersion: number; entitlementId: EntityId; canAccept: boolean; canRequestRefund: boolean; };
eventChangeResponse?: { response: EventMaterialChangeResponse; respondedAt: string; respondedBy: EntityId; } | null;
```

Indexes: `(eventChange.entitlementId, eventChange.eventVersion DESC)`; `(userId, createdAt DESC)`; `(userId, readAt, createdAt DESC)`

### `v2_notifications`

Partner/org-tenant inbox

Domain type `Notification` (`domain/models/notification.ts`):

```ts
id: EntityId;
recipientId: EntityId;
recipientType: NotificationRecipientType;
type: string;
title: string;
body: string;
read: boolean;
readAt: string | null;
data: Record<string, unknown>;
action: NotificationAction | null;
priority: NotificationPriority;
dedupeKey: string | null;
createdAt: string;
```

### `v2_support_tickets`

Domain type `SupportTicket` (`domain/models/support-ticket.ts`) extends VersionedEntity:

```ts
refundReview?: { orderId: EntityId; entitlementIds: EntityId[]; eventVersion: number; policyKind: string; requestedAt: string; };
id: EntityId;
subject: string;
description: string;
category: SupportTicketCategory;
status: SupportTicketStatus;
priority: SupportTicketPriority;
requester: SupportTicketRequester;
assignee: { userId: EntityId; name: string } | null;
messages: SupportTicketMessage[];
internalNotes: SupportInternalNote[];
timeline: SupportTimelineEvent[];
links: SupportTicketLinks;
sla: SupportSla;
mergedInto: EntityId | null;
mergedFrom: EntityId[];
resolvedAt: string | null;
resolvedBy: EntityId | null;
closedAt: string | null;
closedBy: EntityId | null;
deletedAt: string | null;
deletedBy: EntityId | null;
```

Indexes: `(status, createdAt DESC)`; `(priority, createdAt DESC)`; `(category, createdAt DESC)`; `(requesterUserId, deletedAt, createdAt DESC)`

### `v2_safety_reports`

Domain type `SafetyReport` (`domain/models/safety-report.ts`) extends VersionedEntity:

```ts
id: EntityId;
reporter: SafetyReportReporter;
target: SafetyReportTarget;
category: SafetyReportCategory;
priority: SafetyReportPriority;
details: string | null;
context: SafetyReportContext;
status: SafetyReportStatus;
reviewDueAt: string | null;
dismissedAt: string | null;
dismissedBy: EntityId | null;
actionedAt: string | null;
actionedBy: EntityId | null;
action: SafetyReportAction | null;
deletedAt: string | null;
deletedBy: EntityId | null;
```

Indexes: `(status, createdAt DESC)`; `(category, createdAt DESC)`; `(priority, createdAt DESC)`; `(reporterUserId, deletedAt, createdAt DESC)`

### `v2_email_deliveries`

Transactional email delivery log (Resend)


### `v2_outbox`

Transactional outbox (status, nextAttemptAt, leaseExpiresAt, payload) written in the same transaction as domain changes


Indexes: `(status, nextAttemptAt)`; `(status, leaseExpiresAt)`

### `v2_idempotency_records`

HTTP command idempotency records (key, actor, command, request hash, stored response)

