# Firestore collections index (generated)

**Generated 2026-10-24** by scanning `packages/core/src/**` for `v2_*` collection names (backend checkout). **69 collections.** Access is through the Admin SDK only; `firestore.rules` deny every client read and write. The "Referenced in" column lists the source files that name the collection (repository adapters, ports, helpers). Fields and meaning are described in the per-area docs.

| Collection | Referenced in |
|---|---|
| `v2_admin_audit_logs` | `infrastructure/firestore/firestore-onboarding-repository.ts` |
| `v2_admins` | `infrastructure/firestore/firestore-onboarding-repository.ts` |
| `v2_analytics_reads` | `infrastructure/firestore/firestore-analytics-read-model-repository.ts` |
| `v2_auth_users` | `domain/models/platform-user.ts`, `domain/models/user-ban.ts`, `domain/ports/repositories.ts`, `domain/ports/user-directory.ts` (+2) |
| `v2_bank_accounts` | `infrastructure/firestore/firestore-bank-account-repository.ts` |
| `v2_cart_reservations` | `infrastructure/firestore/firestore-cart-reservation-repository.ts` |
| `v2_cover_wallet_by_event_user` | `infrastructure/firestore/firestore-cover-wallet-repository.ts` |
| `v2_cover_wallet_idempotency` | `infrastructure/firestore/firestore-cover-wallet-repository.ts` |
| `v2_cover_wallet_reconciliations` | `infrastructure/firestore/firestore-cover-wallet-repository.ts` |
| `v2_cover_wallet_txns` | `infrastructure/firestore/firestore-cover-wallet-repository.ts` |
| `v2_cover_wallets` | `infrastructure/firestore/firestore-cover-wallet-repository.ts` |
| `v2_disputes` | `infrastructure/firestore/firestore-dispute-repository.ts` |
| `v2_door_sale_idempotency` | `infrastructure/firestore/firestore-door-sale-repository.ts` |
| `v2_door_sales` | `infrastructure/firestore/firestore-door-sale-repository.ts` |
| `v2_email_otps` | `infrastructure/firestore/firestore-email-otp-repository.ts` |
| `v2_entitlements` | `infrastructure/firestore/firestore-entitlement-repository.ts` |
| `v2_event_catalog_promos` | `infrastructure/firestore/firestore-event-catalog-repository.ts` |
| `v2_event_catalog_promoter_assignments` | `infrastructure/firestore/firestore-event-catalog-repository.ts` |
| `v2_event_catalog_tables` | `infrastructure/firestore/firestore-event-catalog-repository.ts` |
| `v2_event_catalog_tiers` | `infrastructure/firestore/firestore-event-catalog-repository.ts` |
| `v2_event_codes` | `infrastructure/firestore/firestore-event-code-repository.ts` |
| `v2_events` | `infrastructure/firestore/firestore-event-repository.ts` |
| `v2_financial_journal_idempotency` | `infrastructure/firestore/firestore-financial-journal-repository.ts` |
| `v2_financial_journal_lines` | `infrastructure/firestore/firestore-financial-journal-repository.ts` |
| `v2_financial_journal_transition` | `infrastructure/firestore/firestore-journal-transition-repository.ts` |
| `v2_financial_journals` | `infrastructure/firestore/firestore-financial-journal-repository.ts` |
| `v2_follows` | `infrastructure/firestore/firestore-social-repositories.ts` |
| `v2_guest_profiles` | `infrastructure/firestore/firestore-guest-profile-repository.ts` |
| `v2_hold_sequences` | `infrastructure/firestore/firestore-cart-reservation-repository.ts` |
| `v2_idempotency_records` | `infrastructure/firestore/firestore-idempotency-store.ts` |
| `v2_leaderboard_stats` | `infrastructure/firestore/firestore-leaderboard-repository.ts` |
| `v2_notifications` | `infrastructure/firestore/firestore-notification-repository.ts`, `infrastructure/firestore/firestore-social-repositories.ts` |
| `v2_onboarding_requests` | `infrastructure/firestore/firestore-onboarding-repository.ts` |
| `v2_orders` | `infrastructure/firestore/firestore-order-repository.ts` |
| `v2_organization_invitations` | `infrastructure/firestore/firestore-invitation-repository.ts` |
| `v2_organizations` | `infrastructure/firestore/firestore-organization-repository.ts` |
| `v2_outbox` | `infrastructure/firestore/firestore-outbox-store.ts` |
| `v2_partner_ledger` | `infrastructure/firestore/firestore-ledger-repository.ts` |
| `v2_partner_ledger_idempotency` | `infrastructure/firestore/firestore-ledger-repository.ts` |
| `v2_partnerships` | `infrastructure/firestore/firestore-partnership-repository.ts` |
| `v2_payouts` | `infrastructure/firestore/firestore-payout-repository.ts` |
| `v2_platform_settings` | `domain/models/platform-settings.ts`, `infrastructure/firestore/firestore-platform-settings-repository.ts` |
| `v2_promo_redemption_locks` | `infrastructure/firestore/firestore-promo-redemption-repository.ts` |
| `v2_promo_redemptions` | `infrastructure/firestore/firestore-promo-redemption-repository.ts` |
| `v2_promoter_codes` | `infrastructure/firestore/firestore-referral-link-repository.ts` |
| `v2_promoter_connections` | `infrastructure/firestore/firestore-promoter-connection-repository.ts` |
| `v2_promoter_links` | `infrastructure/firestore/firestore-referral-link-repository.ts` |
| `v2_promoter_tracking_codes` | `infrastructure/firestore/firestore-referral-link-repository.ts` |
| `v2_promoter_vanity_aliases` | `infrastructure/firestore/firestore-referral-link-repository.ts` |
| `v2_proposed_actions` | `infrastructure/firestore/firestore-onboarding-repository.ts` |
| `v2_provider_financial_event_ids` | `infrastructure/firestore/firestore-provider-financial-event-repository.ts` |
| `v2_provider_financial_events` | `infrastructure/firestore/firestore-provider-financial-event-repository.ts` |
| `v2_provider_payments` | `infrastructure/firestore/firestore-provider-payment-repository.ts` |
| `v2_reconciliation_discrepancies` | `infrastructure/firestore/firestore-reconciliation-repository.ts` |
| `v2_reconciliation_runs` | `infrastructure/firestore/firestore-reconciliation-repository.ts` |
| `v2_refund_requests` | `infrastructure/firestore/firestore-refund-request-repository.ts` |
| `v2_safety_reports` | `infrastructure/firestore/firestore-safety-report-repository.ts` |
| `v2_scan_ledger` | `infrastructure/firestore/firestore-scan-ledger-repository.ts` |
| `v2_scanner_devices` | `infrastructure/firestore/firestore-scanner-device-repository.ts` |
| `v2_scanner_session_tokens` | `infrastructure/firestore/firestore-event-code-repository.ts` |
| `v2_scanner_sessions` | `infrastructure/firestore/firestore-event-code-repository.ts` |
| `v2_slot_requests` | `infrastructure/firestore/firestore-slot-request-repository.ts` |
| `v2_social_notifications` | `infrastructure/firestore/firestore-social-repositories.ts` |
| `v2_staff_credentials` | `infrastructure/firestore/firestore-staff-rotation-store.ts` |
| `v2_support_tickets` | `infrastructure/firestore/firestore-support-ticket-repository.ts` |
| `v2_user_bans` | `infrastructure/firestore/firestore-user-ban-repository.ts` |
| `v2_venue_slots` | `infrastructure/firestore/firestore-venue-slot-repository.ts` |
| `v2_venues` | `infrastructure/firestore/firestore-venue-repository.ts` |
| `v2_verification_attempts` | `infrastructure/firestore/firestore-onboarding-repository.ts` |
