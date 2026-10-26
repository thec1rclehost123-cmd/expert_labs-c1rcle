# 11. Data and collections (partner flows)

Store: Firestore via the Admin SDK only (`firestore.rules` denies all client access). All collections are prefixed `v2_`. Sources: `packages/core/src/infrastructure/firestore/*`, `firestore.indexes.json`.

## Collections by area
- **Org and staff:** `organizations` (members embedded plus a denormalised `memberIds` array for `array-contains` queries), `organization_invitations`, `auth_users`, `auth_sessions`, `auth_accounts`, `auth_verification_tokens`, `staff_credentials`, `email_otps`.
- **Venues and events:** `venues` (`public`/`private` split), `venue_slots`, `slot_requests`, `events`.
- **Catalog:** `event_catalog_tiers`, `event_catalog_promos`, `event_catalog_tables`, `event_catalog_promoter_assignments`, `promo_redemptions`, `promo_redemption_locks`.
- **Graph and promoters:** `partnerships` and `promoter_connections` (both with a denormalised `partyIds` array), `promoter_links`, `promoter_tracking_codes`, `promoter_codes`, `promoter_vanity_aliases`.
- **Onboarding/admin:** `onboarding_requests`, `admins`, `proposed_actions`, `verification_attempts`, `admin_audit_logs`.
- **Read models and infra:** `analytics_reads`, `notifications` (`recipientId` = organisation id), `idempotency_records`, `outbox`.
- **Finance:** `partner_ledger`, `payouts`, `bank_accounts`, `disputes`, `financial_journals`, `leaderboard_stats`, `orders`.
- **Door:** `event_codes`, `scanner_sessions`, `scanner_session_tokens`, `scanner_devices`, `scan_ledger`, `cover_wallets`. Also `entitlements`, `platform_settings`, `support_tickets`.

## Composite indexes relevant here (`firestore.indexes.json`)
`onboarding_requests (status, createdAt)` and `(userId, createdAt desc)`; `verification_attempts (userId, attemptedAt)` both orders; `admin_audit_logs (targetId, occurredAt desc)`; `organization_invitations (organizationId, createdAt)`; `partnerships (partyIds array-contains, createdAt)`; `promoter_connections (partyIds array-contains, createdAt)`; `promoter_links (eventId, createdAt)` and `(promoterId, createdAt)`; `events (isPublic, startAt)` (guest discovery, not the partner list); `event_codes (eventId, createdAt desc)` and `(organizationId, createdAt desc)`; several for `scanner_sessions`; `scanner_devices (organizationId, lastSeenAt desc)`; finance indexes for `payouts`, `disputes`, `partner_ledger`, `orders`, `leaderboard_stats`, `financial_journals`; `outbox (status, nextAttemptAt)` and `(status, leaseExpiresAt)`. `fieldOverrides` is empty. CI validates the file with `scripts/check-firestore-indexes.mjs`.

## Deliberately index-free queries (and the traps)
Notifications by `recipientId` (and `read`) with in-memory sorting; events, venues and slot requests use a single equality filter plus the repository paginator; catalog tiers/tables/assignments are equality-only; `findPendingByEmail` is a single filter plus in-code checks (a triple-where once needed a composite index that did not exist; see `docs/Intern/TEAM-MEMBER-ADD-VENUE-HOST-500-FIX.md`); venue slots use `venueId` equality, a 2000 cap and a code-side window filter; discovery uses `status == 'active'` with `limit`; organisations use `memberIds array-contains`.
**Trap:** no index is declared for `notifications`, `slot_requests`, `events` by `organizationId`, or `event_catalog_*`. Adding `orderBy` plus `where` on those would fail at runtime.

## Scale caveats
Notification list/mark-all read every notification for an org; discovery caps at 200; calendar caps at 2000 slots; analytics fallback scans up to 1000 per page; availability and tier reads are derived per request.
