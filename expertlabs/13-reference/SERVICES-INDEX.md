# Backend services, models and ports index

Source: `C1RCLE-BACKEND/packages/core/src` (application services, domain models, ports) and `contracts`. "Documented in" points at the in-depth set that explains the area. **Areas with no in-depth doc** are marked ⚠ and listed again at the end.

## Application services (`application/<dir>/<file>`)
| Service | Purpose | Documented in |
|---|---|---|
| `organizations/organization-service` | organisation CRUD, membership, invitations, org-scoped access, staff credential provisioning | `06-partner-dashboard/07-…` |
| `onboarding/onboarding-service` | KYC onboarding requests, documents, approval into an organisation | `06-…/05-…`, `07-admin-console/05-…` |
| `venues/venue-service` | venues, slot requests, calendar blocks (`VenueService`, `VenueCalendarService`, `VenueSlotRequestService`) | `06-…/06-…` |
| `events/event-service`, `event-catalog/event-catalog-service` | event lifecycle; tiers, promos, tables, promoter assignments | `06-…/06-…`, `07-…` |
| `partnerships/partnership-service`, `partner-discovery-service` | venue↔host partnerships, discovery browse | `06-…/07-…` |
| `promoters/promoter-connection-service`, `referral-link-service` | promoter connections, referral links, attribution, commission snapshotting | `06-…/07-…`, `08-guest-portal/03-…` |
| `analytics/analytics-service`, `analytics-projection` | partner analytics read model and its projection | `06-…/08-…` |
| `notifications/notification-service`, `notification-consumer` | partner organisation inbox | `06-…/08-…` |
| `admin/admin-authority-service`, `admin-ops-service` | dual control, directories, lookup, exports, platform actions, settings | `07-admin-console/04-…`, `10-…` |
| `support/support-service`, `admin-support-service` | guest intake and the admin support desk | `07-…/09-…`, `08-guest-portal/05-…` |
| `checkout/checkout-service`, `fulfilment-repair-service` | quote, hold, intent, confirm, settlement, repair | `10-payments-and-finance/02-…`, `07-…` |
| `orders/order-service`, `tickets/ticket-service`, `inventory/inventory-service`, `pricing/pricing-service` | orders read model, ticket reads, availability and holds, pricing | `10-…/01-…`, `03-…` |
| `public/public-service` | public discovery feed | `08-guest-portal/03-…` |
| `finance/*`: `finance-service`, `financial-posting-service`, `journal-transition-service`, `payout-service`, `admin-payout-service`, `bank-account-service`, `dispute-service`, `admin-dispute-service`, `refund-service`, `provider-payment-service`, `reconciliation-service` | ledger and journal, payouts, refunds, disputes, provider facts, reconciliation | `10-…/04-…`–`07-…`, `07-admin-console/06-…`–`08-…` |
| `door/door-service`, `scanner-service`, `door-ops-service`, `door-stats-service`, `door-ticket-sale-service` | admission, devices and sessions, shift ops and roster, live stats, paid door sales | `09-scanner-app/*` |
| `cover-wallet/cover-wallet-service` | prepaid bar tab | `09-…/06-…` |
| `idempotency/idempotency-service`, `events/event-bus`, `events/audit-consumers`, `auth/email-otp-service` | idempotency, outbox bus, event→audit records, signup OTP | `06-…/09-…`, `04-auth-and-rbac` |
| `social/social-service`, `notification-consumers`, `social-notification-repository` | follow graph and guest in-app notifications (Phase 8) | `DEEP-DIVE-SOCIAL-SAFETY-LEADERBOARD-PROFILE.md` section 1 |
| `finance/leaderboard-service` | public promoter leaderboard ranked by commission | same page, section 3 |
| `safety/safety-service`, `admin-safety-service` | guest safety reports and admin desk. **Not wired: never instantiated, no routes exist** | same page, section 4 |
| `guest-profile/guest-profile-service` | guest onboarding profile (date of birth, city, tastes) | same page, section 2 |
| ⚠ `rsvp` route family | free-ticket RSVP (handled by `checkout-service.createRsvp`) | `10-…/02-…` |

## Domain models (`domain/models/`, 40 files)
`admin-authority`, `bank-account`, `cart-reservation`, `cover-wallet`, `cover-wallet-reconciliation`, `dispute`, `door-sale`, `email-otp`, `entitlement`, `event-catalog`, `event-code`, `event`, `financial-journal`, `guest-profile`, `journal-transition`, `leaderboard`, `ledger`, `notification`, `onboarding`, `order`, `organization`, `partner-access`, `partnership`, `payout`, `platform-settings`, `platform-user`, `pricing`, `promoter-connection`, `provider-financial-event`, `provider-payment`, `reconciliation`, `referral-link`, `refund-request`, `safety-report`, `scan-ledger`, `scanner-device`, `social`, `support-ticket`, `user-ban`, `venue`; plus `errors.ts`, `events.ts`, `fsm.ts`, `identity.ts`.

## Ports (`domain/ports/`)
Services: `audit` (`AuditRepository`, `AdminAuditRepository`), `email-sender`, `idempotency`, `object-storage`, `outbox`, `payment-provider`, `staff-credentials`, `user-directory`, `verification`. `repositories.ts` interfaces (about 50): organisation, user account, user ban, partnership, referral link, promoter connection, invitation, venue, slot request, venue slot, event, event catalog, analytics read model, onboarding, platform admin, proposed action, verification attempt, cart reservation, order, entitlement, promo redemption, scan ledger, scanner device, event code, scanner session, door sale, cover wallet (+ txn, reconciliation), ledger, financial journal, journal transition, payout, bank account, dispute, provider payment, reconciliation, provider financial event, admin refund request, safety report, support ticket, leaderboard, email OTP, notification, guest profile (**declared twice**, lines 1406 and 1466; TypeScript merges them), follow, social notification, platform settings.
Adapters: about 50 in `infrastructure/firestore/`, about 35 in `infrastructure/memory/`; also `compare-and-set.ts`, `pagination.ts` (caps `limit` at 100 with offset cursors), `client.ts` (the only `firebase-admin` import), `firebase-object-storage.ts`, an outbox store, `encryption.ts` (AES-256-GCM).

## Contracts (`packages/contracts/src/contracts/`, 21 files)
`admin`, `admin-analytics`, `admin-orders`, `admin-promoters`, `admin-promotions`, `admin-settings`, `admin-tickets`, `auth`, `checkout`, `event`, `guest-profile`, `notifications`, `onboarding`, `organization`, `partner`, `phase5` (door), `phase6` (finance), `phase7` (support, admin), `public`, `shared`, `social`; parity fixtures `parity/cases.mjs`.

## Route families (`apps/api-gateway/src/routes/v2/`)
admin, auth, checkout, door, finance, internal, notifications, onboarding, orders, partner, public, rsvp, social, support, tickets, wallet, plus `profile.ts` and `phase5-routes.ts`: generated index in `API-ROUTE-INDEX.md` (275 route registrations).

## Coverage gaps in this package
The social, leaderboard, safety and guest-profile areas now have a dedicated deep dive. RSVP and the analytics backfill are covered in `DEEP-DIVE-OTP-RSVP-CONTRACTS.md`, and every model's fields are in `DATA-MODEL-FIELDS.md`; nothing in this table lacks a page now.
