# 02. Core Domain (`@c1rcle/core`)

**Path:** `C1RCLE-BACKEND/packages/core` · **Status:** Live · pure TypeScript, no HTTP.

## Layers
| Layer | Contains |
|---|---|
| `domain/` | ~40 aggregates (`models/`): organization, venue, event, event-catalog, order, entitlement, cart-reservation, scan-ledger, scanner-device, event-code, cover-wallet, door-sale, payout, ledger, financial-journal, refund-request, dispute, reconciliation, provider-payment, bank-account, admin-authority, onboarding, email-otp, support-ticket, social, notification, platform-settings. `ports/` (repository + service interfaces: audit, email-sender, idempotency, object-storage, outbox, payment-provider, staff-credentials, user-directory, verification). `fsm.ts` (explicit transition tables; same-state = idempotent no-op), `errors.ts` (typed `DomainError`), `events.ts`, `identity.ts` |
| `application/` | Services taking `ServiceDeps` (config, logger, repositories). `ActorContext` + `requireOrgAccess` enforce tenancy. Services throw domain errors, never HTTP |
| `infrastructure/` | `memory/` and `firestore/` adapters, `encryption.ts` (AES-256-GCM), `utils.ts` |

Exports: `.`, `./config`, `./domain`, `./domain/ports`, `./application`, `./infrastructure`, `./infrastructure/memory`, `./infrastructure/firestore`.

Rule enforced by `scripts/check-boundaries.mjs`: no database SDK imports in domain/application; no `.collection()` / `.doc()` in route files.

## Services (under `src/application/<dir>/`)
| Area | Services | Notes |
|---|---|---|
| Checkout | `checkout-service` (`quote`, `createHold`, `createRsvp`, `createPaymentIntent`, `verifyCheckoutCallback`, `confirmPayment`, `getPaymentRecoveryStatus`, `settleOrder`), `fulfilment-repair-service` | `confirmPayment` idempotent on provider `paymentId`; webhook and redirect-confirm converge |
| Inventory/pricing | `inventory-service`, `pricing/pricing-service` | availability = quantity - sold - activeHolds, derived (D-014). `refund_requested` still holds the seat until `refunded` |
| Orders/tickets | `order-service`, `ticket-service` | admission via `EntitlementRepository.claimAdmission` (single Firestore transaction, D-025); rotating QR = HMAC with `MAGIC_TICKET_SECRET`, +-2 window, `timingSafeEqual` |
| Door | `scanner-service`, `door-service`, `door-ops-service`, `door-stats-service`, `door-ticket-sale-service`, `cover-wallet-service` | see `../09-scanner-app` |
| Finance | `finance-service`, `financial-posting-service`, `journal-transition-service`, `payout-service`, `admin-payout-service`, `bank-account-service`, `refund-service`, `dispute-service`, `admin-dispute-service`, `reconciliation-service`, `provider-payment-service`, `leaderboard-service` | see `../10-payments-and-finance` |
| Orgs/venues/events | `organization-service`, `venue-service` (+calendar, slot requests), `event-service`, `event-catalog-service`, `partnerships/*`, `promoters/*`, `onboarding-service`, `public-service`, `analytics/*` | |
| Admin | `admin-authority-service` (single authorisation point for admin tiers), `admin-ops-service`, `safety/*`, `support/*` | |
| Notifications/social | `notification-service` + consumer, `social-service` + follower fan-out | email via `EmailSender` port (Resend) |
| Cross-cutting | `idempotency-service`, `events/event-bus` (`InProcessEventBus`) | idempotency durable on Firestore, request-hash for 409 key reuse, in-flight detection |

## Event state machine
`draft -> review -> scheduled -> published <-> sales_paused -> started -> ended -> archived`; any state can go to `cancelled`.

## Domain events and outbox
Events are appended to `v2_outbox`, then delivered to consumers: audit, analytics projection, notifications, follower fan-out. `OUTBOX_MAX_ATTEMPTS = 5`, then dead-letter. No external broker.

## Storage drivers
Selected by `STORAGE_DRIVER=memory|firestore`. Wiring in `apps/api-gateway/src/lib/v2-services.ts`.

| | Memory | Firestore |
|---|---|---|
| Use | default for `pnpm test`/CI, local dev | staging/production |
| Auth | not built; dev actor from `x-user-id` only when `NODE_ENV!=production` | real Better Auth |
| Payments | `MemoryPaymentProvider` | `RazorpayPaymentProvider` |
| Files | `EchoObjectStorage` | `FirebaseObjectStorage` |
| Verification | `FormatCheckVerificationProvider` | `FirebasePhoneVerificationProvider` |
| Concurrency | Map-based | `compare-and-set.ts` optimistic locking (D-015) + `runTransaction` for admission |

Memory in production is blocked unless `ALLOW_MEMORY_STORAGE_IN_PRODUCTION` is set.

## Status and caveats
- `docs/architecture/README.md` still says "no transactional compare-and-set yet": **stale** (D-015 and `claimAdmission` exist).
- Redis-backed inventory / circuit-breaker described in `inventory-service.ts` comments is **not wired**; Redis is only used for rate limiting.
- Postgres is a planned future store (D-002).
- Inventory correctness history: oversell, paid-without-ticket and late-capture defects found in the October audit were **fixed on staging** (via #135); see `../10-payments-and-finance/03-order-hold-inventory.md`. A full services, models and ports index with documentation gaps is in `../13-reference/SERVICES-INDEX.md`.
