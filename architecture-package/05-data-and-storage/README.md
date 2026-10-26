# 05. Data and Storage

**Status:** Live on Firestore driver. Code: `packages/core/src/infrastructure/firestore/`, `firestore.rules`, `firestore.indexes.json`.

## Stores
| Store | Use | Notes |
|---|---|---|
| **Firestore** (project id `FIRESTORE_PROJECT_ID`; emulator 8081) | Primary database | Admin SDK only. `client.ts` is the only file importing `firebase-admin` |
| **Cloud Storage** (`FIREBASE_STORAGE_BUCKET`) | KYC documents, event posters | Pre-signed URLs via `FirebaseObjectStorage`; `configure:storage-cors` one-off script |
| **Redis** (`REDIS_URL`) | Rate-limit store only | Optional; degrades to memory; readiness reports `redis: down` |
| Postgres | Planned (D-002) | Not built |

**`firestore.rules` = `allow read, write: if false`** for everything. Clients never touch Firestore; rules are a fail-closed backstop. 77 composite indexes in `firestore.indexes.json`, validated by `scripts/check-firestore-indexes.mjs` in CI (heaviest: `v2_scan_ledger` 10, `v2_door_sales` 6).

## Collections (all prefixed `v2_`)
| Domain | Collections |
|---|---|
| Auth & identity | `auth_users`, `auth_sessions`, `auth_accounts`, `auth_verification_tokens`, `admins`, `staff_credentials`, `user_bans`, `guest_profiles`, `email_otps`, `verification_attempts` |
| Tenancy & venues | `organizations`, `organization_invitations`, `venues`, `venue_slots`, `slot_requests`, `partnerships`, `onboarding_requests` |
| Events & catalog | `events`, `event_catalog_tiers`, `event_catalog_promos`, `event_catalog_tables`, `event_catalog_promoter_assignments`, `promo_redemptions`, `promo_redemption_locks` |
| Promoters | `promoter_links`, `promoter_codes`, `promoter_tracking_codes`, `promoter_vanity_aliases`, `promoter_connections` |
| Checkout & tickets | `cart_reservations`, `hold_sequences`, `orders`, `entitlements`, `provider_payments`, `provider_financial_events`, `provider_financial_event_ids` |
| Door | `event_codes`, `scanner_sessions`, `scanner_session_tokens`, `scanner_devices`, `scan_ledger`, `door_sales`, `door_sale_idempotency`, `cover_wallets`, `cover_wallet_txns`, `cover_wallet_by_event_user`, `cover_wallet_idempotency`, `cover_wallet_reconciliations` |
| Finance | `partner_ledger`, `partner_ledger_idempotency`, `financial_journals`, `financial_journal_lines`, `financial_journal_idempotency`, `financial_journal_transition`, `payouts`, `bank_accounts`, `refund_requests`, `disputes`, `reconciliation_runs`, `reconciliation_discrepancies`, `leaderboard_stats` |
| Platform | `proposed_actions`, `admin_audit_logs`, `platform_settings`, `idempotency_records`, `outbox`, `analytics_reads`, `support_tickets`, `safety_reports`, `notifications`, `social_notifications`, `follows` |

`hold_sequences` holds the per-event hold counter used for transactional hold ordering (payment audit fix, **merged on staging via #135**). A generated list of all 69 collections with their source files is in `../13-reference/COLLECTIONS-INDEX.md`.

## Concurrency model
- Every mutable aggregate carries `version` + `updatedAt`; writes use compare-and-set (`compare-and-set.ts`, D-015).
- Admission uses one `runTransaction` (`claimAdmission`, D-025).
- Idempotency is durable (`v2_idempotency_records`) plus domain-specific idempotency collections.
- Known limitation: `cleanupExpired` uses one unbounded batch while Firestore caps at 500 ops (plan item A3.4, open).

## Encryption
Bank account numbers sealed with AES-256-GCM (`ENCRYPTION_KEY`, `ENCRYPTION_SALT`); legacy key fallback for old records. PII/KYC files sit in the storage bucket behind pre-signed URLs.

## Legacy
`v1-read-only-client.ts` and `migrate-and-seed-v1-sample.ts` read/migrate from the frozen V1 system.

## Cost/limit note
Backend runs on free tiers; Firestore quotas and per-request read cost matter because availability is computed at read time (`O(orders + holds)` per `GET /events/:id/tiers`, a documented concern).
