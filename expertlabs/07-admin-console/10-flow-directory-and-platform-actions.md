# 10. Directory, platform actions, lookup, analytics, exports and settings

Code: BE `routes/v2/admin/{directory,orders,tickets,promotions,promoters,analytics,settings,venue-actions,organization-actions,user-actions,event-actions}.ts`, `packages/core/src/application/admin/admin-ops-service.ts`; FE `app/{venues,events,hosts,users,promoters,orders,tickets,promotions,analytics,lookup,settings,audit}`.

## Directory reads (any admin, `AUTH_READ`)
`GET /admin/venues|events|hosts|users` (`directoryQuerySchema`; offset pagination with `count()`+`offset()`, so O(offset) per page; no search or filter parameters; `listUsers` joins ban status with an N+1 `userBans.getByUserId`). `hosts` lists **organisations**. `GET /admin/orders`, `/tickets`, `/promotions`, `/promoters`, `/analytics`, `/lookup`, `/audit`. Offset pagination and `paginateQuery` do `Math.min(limit, 100)`.

## Lookup (omnibox)
`GET /admin/lookup?q=` (≥3 characters, else `[]`): exact id (or email) match in parallel across venue, event, organisation, user. **PII bypass:** the user label is the email, so a `support` admin who cannot see emails in the users CSV or audit names can still resolve one by typing 3+ characters.

## Direct commands
| Action | Route | Tier | Behaviour |
|---|---|---|---|
| Suspend/reinstate venue | `POST /admin/venues/:id/suspend\|reinstate` (`venue-actions.ts`, **no test file**) | TIER2 | idempotent no-op if already in state (no audit row); no reason recorded |
| Suspend/reinstate organisation | `POST /admin/organizations/:id/suspend\|reinstate` | TIER2 | same |
| Suspend/reinstate promoter | `POST /admin/promoters/:id/suspend\|reinstate` | TIER2 | acts on **all** the promoter's assignments (`active ↔ suspended`; `ended` terminal); returns `{affected, at}`; audits only if `affected > 0`; each assignment saved separately, so a mid-loop failure leaves a partial result |
| Pause/resume event | `POST /admin/events/:id/pause\|resume` | TIER1 (any admin incl. `support`) | domain `adminPauseEvent`/`adminResumeEvent` set `adminOverride` |
| Force-complete event | `POST /admin/events/:id/force-complete` | TIER1 (`EVENT_FORCE_PAUSE`) | `published\|sales_paused\|started → ended`, `adminOverride:true`, clears `isPublic`; **no refund, cancellation or event emission** |
| Ban/unban user | `POST /admin/users/:id/ban\|unban` (`user-actions.ts`, optional body reason) | TIER2 | `UserBan` aggregate (`id = userId`, one record upserted, no history); a ban for a non-existent user id succeeds with a placeholder `@unknown.c1rcle.internal` email |
All are `SENSITIVE_COMMAND` and idempotent (`admin.venue.suspend`, `admin.organization.suspend`, `admin.promoter.suspend`, `admin.event.pause|resume|force_complete`, `admin.user.ban`, …).

### Enforcement reality (verified in source)
- **User ban is stored but not enforced:** `userBans` is read only by the admin listing/CSV and plumbing (`context.ts`, `infrastructure/utils.ts`); no check in auth, the session plugin or checkout.
- **Maintenance mode** is never read outside the settings code (the model comment mentions an ops-middleware gate that does not exist).
- **`platformFeeRate`** is never used (see `07-…`).
- **Organisation and venue `suspended`** is honoured only for public discovery (`public-service.ts` ≈L106-107), promoter connections and venue partnership; not in checkout, partner writes or login.
So most "suspend/ban/maintenance" buttons record intent and audit, but do not (yet) stop the affected party transacting.

## Analytics
`GET /admin/analytics` (`adminAnalyticsSummaryDtoSchema`): scans the latest "1000" orders/events/orgs and reports `truncated`, `scannedOrders`, `scannedEvents`; on Firestore `listAll` is capped at 100, so the totals are a lower bound. The page warns when truncated.

## CSV exports (`lib/csv.ts`)
`csvEscape` does RFC 4180 quoting and prefixes `'` to cells starting with `= + - @ TAB CR` (formula-injection guard). Routes: `/admin/{venues,events,hosts,users,audit,promoters,orders,tickets,promotions,refunds,reconciliation}/export.csv`. Audited as `ADMIN_EXPORT` (with row count): **users and audit only**. The users CSV writes `[redacted]` for email unless the viewer is `super` or `finance`; the audit CSV resolves `platform_user` target names only for `super`/`finance`. Orders CSV includes `userId` and `paymentId`; tickets CSV includes `holderName` (PII); both unaudited. Requested limits of 1,000/10,000 rows are **silently capped at 100** on Firestore. The reconciliation CSV route has no rate limit and no `validateV2`, and no FE function calls it.

## Platform settings (`GET|PUT /admin/settings/platform`)
`PlatformSettings` (`v2_platform_settings/singleton`): `platformFeeRate` (0–1, default 0.15), `refundSingleApproverThresholdPaise` (50,000), `refundDualApproverThresholdPaise` (500,000), `maintenanceMode`, `featureFlags`, `accountingPolicy {version, settlementDelayDays, reserveBasisPoints, requiresEventCompletion, holdsOnCancellation, holdsOnDispute, holdsOnUnresolvedReconciliation}`, `updatedAt`. Default policy version is `'m2-test-policy-v1'`, a test-grade policy that is the production default.
- **No authority tier and no dual control on update** (`updatePlatformSettings` calls only `requireAdmin`): any active admin, including `support`, can change refund thresholds, maintenance mode, flags and accounting policy. `settings.test.ts` only tests "refuses a non-admin".
- Body `platformSettingsUpdateRequestSchema` is strict, but there is no cross-field validation (single threshold can exceed dual), no `If-Match`, last-write-wins merge, and the audit omits `featureFlags` and most policy fields.
- Consumers: refund thresholds (`RefundService.requestRefund`), accounting policy (`CheckoutService` ≈L1160, `payout-service.ts` ≈L127). `platformFeeRate` and `maintenanceMode` have no consumers.
- Console: fee rate shown as a fraction (Commissions shows a whole percent).
