# API route index (generated)

**Generated from source on 2026-10-24** by scanning `apps/api-gateway/src/routes/v2/**` on backend checkout `fix/partner-connections-and-events` at `4bcc15a`. It lists **275 route registrations**. Staging may differ (it is 9 commits ahead of this checkout; the door routes are unchanged but PR #138 modifies the offline manifest and sync handlers). Treat the route files as authoritative.

Columns: **Rate** = rate-limit class (`PUBLIC_READ` 120/min, `AUTH_READ` 240, `STANDARD_COMMAND` 60, `SENSITIVE_COMMAND` 10, `OTP_SEND` 5, `OTP_VERIFY` 10, `SCANNER_COMMAND` 300). **Permission** = `requirePermission` on the route (organisation RBAC); `-` means none on the route, so authorisation, if any, happens in the service (session actor, scanner token, platform-admin authority, or HMAC). **Cache** = response cache class. **Idem** = the handler takes an `Idempotency-Key` or wraps `runIdempotent`; **Command** = its idempotency command name. Loop-registered routes (`${action}`) are shown with the placeholder.

## Summary by area

| Area (route directory) | Routes |
|---|---|
| `admin` | 84 |
| `partner` | 66 |
| `door` | 41 |
| `finance` | 17 |
| `auth` | 11 |
| `public` | 11 |
| `onboarding` | 8 |
| `social` | 8 |
| `checkout` | 6 |
| `notifications` | 4 |
| `support` | 4 |
| `internal` | 3 |
| `orders` | 3 |
| `wallet` | 3 |
| `phase5-routes` | 2 |
| `profile` | 2 |
| `rsvp` | 1 |
| `tickets` | 1 |

**Routes with no `requirePermission` (excluding `/public` and `/internal`): 155.** They are protected by something else (session, scanner token, admin authority, HMAC) or are deliberately open (signup, login). Review list at the end of this file.

By method: DELETE 5, GET 130, PATCH 8, POST 129, PUT 3.


## admin

| Method | Path | Rate | Permission | Cache | Idem | Command | Source |
|---|---|---|---|---|---|---|---|
| GET | `/api/v2/admin/analytics` | AUTH_READ | - |  |  |  | `admin/analytics.ts:21` |
| GET | `/api/v2/admin/venues` | AUTH_READ | - |  |  |  | `admin/directory.ts:117` |
| GET | `/api/v2/admin/events` | AUTH_READ | - |  |  |  | `admin/directory.ts:146` |
| GET | `/api/v2/admin/hosts` | AUTH_READ | - |  |  |  | `admin/directory.ts:175` |
| GET | `/api/v2/admin/users` | AUTH_READ | - |  |  |  | `admin/directory.ts:204` |
| GET | `/api/v2/admin/lookup` | AUTH_READ | - |  |  |  | `admin/directory.ts:233` |
| GET | `/api/v2/admin/venues/export.csv` | AUTH_READ | - |  |  |  | `admin/directory.ts:263` |
| GET | `/api/v2/admin/events/export.csv` | AUTH_READ | - |  |  |  | `admin/directory.ts:308` |
| GET | `/api/v2/admin/hosts/export.csv` | AUTH_READ | - |  |  |  | `admin/directory.ts:361` |
| GET | `/api/v2/admin/users/export.csv` | AUTH_READ | - |  |  |  | `admin/directory.ts:411` |
| GET | `/api/v2/admin/audit/export.csv` | AUTH_READ | - |  |  |  | `admin/directory.ts:446` |
| POST | `/api/v2/admin/disputes/:disputeId/resolve` | SENSITIVE_COMMAND | - |  | yes | admin.dispute.resolve | `admin/disputes.ts:62` |
| GET | `/api/v2/admin/disputes` | AUTH_READ | - |  |  |  | `admin/disputes.ts:117` |
| GET | `/api/v2/admin/disputes/:disputeId` | AUTH_READ | - |  |  |  | `admin/disputes.ts:152` |
| POST | `/api/v2/admin/events/:eventId/pause` | SENSITIVE_COMMAND | - |  | yes | admin.event.pause | `admin/event-actions.ts:52` |
| POST | `/api/v2/admin/events/:eventId/resume` | SENSITIVE_COMMAND | - |  | yes | admin.event.resume | `admin/event-actions.ts:96` |
| POST | `/api/v2/admin/events/:eventId/force-complete` | SENSITIVE_COMMAND | - |  | yes | admin.event.force_complete | `admin/event-actions.ts:140` |
| GET | `/api/v2/admin/onboarding/applications` | AUTH_READ | - |  |  |  | `admin/onboarding-review.ts:77` |
| GET | `/api/v2/admin/onboarding/applications/:requestId` | AUTH_READ | - |  | yes |  | `admin/onboarding-review.ts:112` |
| GET | `/api/v2/admin/onboarding/applications/:requestId/documents/:label/read-url` | AUTH_READ | - |  |  |  | `admin/onboarding-review.ts:144` |
| POST | `/api/v2/admin/onboarding/applications/:requestId/documents/:label/verify` | SENSITIVE_COMMAND | - |  | yes | admin.onboarding.document.verify | `admin/onboarding-review.ts:173` |
| POST | `/api/v2/admin/onboarding/applications/:requestId/documents/:label/reject` | SENSITIVE_COMMAND | - |  | yes | admin.onboarding.document.reject | `admin/onboarding-review.ts:223` |
| POST | `/api/v2/admin/onboarding/applications/:requestId/approve` | SENSITIVE_COMMAND | - |  | yes | admin.onboarding.approve | `admin/onboarding-review.ts:283` |
| GET | `/api/v2/admin/proposals` | AUTH_READ | - |  |  |  | `admin/onboarding-review.ts:356` |
| POST | `/api/v2/admin/proposals` | SENSITIVE_COMMAND | - |  | yes |  | `admin/onboarding-review.ts:391` |
| GET | `/api/v2/admin/admins` | AUTH_READ | - |  |  |  | `admin/onboarding-review.ts:422` |
| POST | `/api/v2/admin/proposals/:proposalId/provision-admin` | SENSITIVE_COMMAND | - |  | yes | admin.provision | `admin/onboarding-review.ts:460` |
| POST | `/api/v2/admin/proposals/:proposalId/update-admin-role` | SENSITIVE_COMMAND | - |  | yes | admin.role_update | `admin/onboarding-review.ts:512` |
| POST | `/api/v2/admin/admins/:adminId/revoke` | SENSITIVE_COMMAND | - |  | yes | admin.revoke | `admin/onboarding-review.ts:560` |
| GET | `/api/v2/admin/audit` | AUTH_READ | - |  |  |  | `admin/onboarding-review.ts:610` |
| POST | `/api/v2/admin/onboarding/applications/:requestId/${action}` | SENSITIVE_COMMAND | - |  | yes |  | `admin/onboarding-review.ts:650` |
| POST | `/api/v2/admin/proposals/:proposalId/${action}` | SENSITIVE_COMMAND | - |  | yes |  | `admin/onboarding-review.ts:710` |
| GET | `/api/v2/admin/orders` | AUTH_READ | - |  |  |  | `admin/orders.ts:60` |
| GET | `/api/v2/admin/orders/export.csv` | AUTH_READ | - |  |  |  | `admin/orders.ts:89` |
| POST | `/api/v2/admin/proposals/:proposalId/adjust-commission` | SENSITIVE_COMMAND | - |  | yes | admin.organization.adjust_commission | `admin/organization-actions.ts:46` |
| POST | `/api/v2/admin/organizations/:organizationId/suspend` | SENSITIVE_COMMAND | - |  | yes | admin.organization.suspend | `admin/organization-actions.ts:89` |
| POST | `/api/v2/admin/organizations/:organizationId/reinstate` | SENSITIVE_COMMAND | - |  | yes | admin.organization.reinstate | `admin/organization-actions.ts:132` |
| POST | `/api/v2/admin/proposals/:proposalId/freeze-payout` | SENSITIVE_COMMAND | - |  | yes | admin.payout.freeze | `admin/payouts.ts:72` |
| POST | `/api/v2/admin/proposals/:proposalId/release-payout` | SENSITIVE_COMMAND | - |  | yes | admin.payout.release | `admin/payouts.ts:120` |
| POST | `/api/v2/admin/payouts/batch-run` | SENSITIVE_COMMAND | - |  | yes | admin.payout.batch_run | `admin/payouts.ts:171` |
| POST | `/api/v2/admin/payouts/:payoutId/complete` | SENSITIVE_COMMAND | - |  | yes | admin.payout.complete | `admin/payouts.ts:220` |
| GET | `/api/v2/admin/payouts` | AUTH_READ | - |  |  |  | `admin/payouts.ts:271` |
| GET | `/api/v2/admin/payouts/:payoutId` | AUTH_READ | - |  |  |  | `admin/payouts.ts:306` |
| GET | `/api/v2/admin/promoters` | AUTH_READ | - |  |  |  | `admin/promoters.ts:69` |
| GET | `/api/v2/admin/promoters/export.csv` | AUTH_READ | - |  |  |  | `admin/promoters.ts:98` |
| POST | `/api/v2/admin/promoters/:promoterId/suspend` | SENSITIVE_COMMAND | - |  | yes | admin.promoter.suspend | `admin/promoters.ts:141` |
| POST | `/api/v2/admin/promoters/:promoterId/reinstate` | SENSITIVE_COMMAND | - |  | yes | admin.promoter.reinstate | `admin/promoters.ts:189` |
| GET | `/api/v2/admin/promotions` | AUTH_READ | - |  |  |  | `admin/promotions.ts:59` |
| GET | `/api/v2/admin/promotions/export.csv` | AUTH_READ | - |  |  |  | `admin/promotions.ts:88` |
| POST | `/api/v2/admin/proposals/:proposalId/post-financial-adjustment` | SENSITIVE_COMMAND | - |  | yes | admin.finance.adjustment | `admin/reconciliation.ts:71` |
| GET | `/api/v2/admin/chargebacks` | AUTH_READ | - |  |  |  | `admin/reconciliation.ts:150` |
| POST | `/api/v2/admin/reconciliation/runs` | SENSITIVE_COMMAND | - |  | yes | admin.reconciliation.run | `admin/reconciliation.ts:198` |
| GET | `/api/v2/admin/reconciliation/runs` | AUTH_READ | - |  |  |  | `admin/reconciliation.ts:245` |
| GET | `/api/v2/admin/reconciliation/runs/:runId` | AUTH_READ | - |  |  |  | `admin/reconciliation.ts:266` |
| GET | `/api/v2/admin/reconciliation/discrepancies` | AUTH_READ | - |  |  |  | `admin/reconciliation.ts:280` |
| GET | `/api/v2/admin/reconciliation/discrepancies/:discrepancyId` | AUTH_READ | - |  |  |  | `admin/reconciliation.ts:301` |
| POST | `/api/v2/admin/reconciliation/discrepancies/:discrepancyId/retry` | SENSITIVE_COMMAND | - |  | yes | admin.reconciliation.recheck | `admin/reconciliation.ts:332` |
| POST | `/api/v2/admin/reconciliation/discrepancies/:discrepancyId/resolve` | SENSITIVE_COMMAND | - |  | yes | admin.reconciliation.resolve | `admin/reconciliation.ts:386` |
| GET | `/api/v2/admin/reconciliation/export.csv` | - | - |  |  |  | `admin/reconciliation.ts:443` |
| POST | `/api/v2/admin/refunds` | SENSITIVE_COMMAND | - |  | yes | admin.refund.request | `admin/refunds.ts:92` |
| POST | `/api/v2/admin/refunds/:refundRequestId/approve` | SENSITIVE_COMMAND | - |  | yes | admin.refund.approve | `admin/refunds.ts:147` |
| POST | `/api/v2/admin/refunds/:refundRequestId/reject` | SENSITIVE_COMMAND | - |  | yes | admin.refund.reject | `admin/refunds.ts:195` |
| POST | `/api/v2/admin/refunds/:refundRequestId/execute` | SENSITIVE_COMMAND | - |  | yes | admin.refund.execute | `admin/refunds.ts:251` |
| GET | `/api/v2/admin/refunds` | AUTH_READ | - |  |  |  | `admin/refunds.ts:301` |
| GET | `/api/v2/admin/refunds/export.csv` | AUTH_READ | - |  |  |  | `admin/refunds.ts:336` |
| GET | `/api/v2/admin/refunds/:refundRequestId` | AUTH_READ | - |  |  |  | `admin/refunds.ts:385` |
| GET | `/api/v2/admin/settings/platform` | AUTH_READ | - |  |  |  | `admin/settings.ts:60` |
| PUT | `/api/v2/admin/settings/platform` | SENSITIVE_COMMAND | - |  | yes | admin.platform-settings.update | `admin/settings.ts:85` |
| GET | `/api/v2/admin/support/tickets` | AUTH_READ | - |  |  |  | `admin/support.ts:76` |
| GET | `/api/v2/admin/support/tickets/:ticketId` | AUTH_READ | - |  | yes |  | `admin/support.ts:120` |
| POST | `/api/v2/admin/support/tickets/:ticketId/assign` | SENSITIVE_COMMAND | - |  | yes | admin.support.assign | `admin/support.ts:152` |
| POST | `/api/v2/admin/support/tickets/:ticketId/priority` | SENSITIVE_COMMAND | - |  | yes | admin.support.change-priority | `admin/support.ts:206` |
| POST | `/api/v2/admin/support/tickets/:ticketId/reply` | SENSITIVE_COMMAND | - |  | yes | admin.support.reply | `admin/support.ts:260` |
| POST | `/api/v2/admin/support/tickets/:ticketId/notes` | SENSITIVE_COMMAND | - |  | yes | admin.support.note | `admin/support.ts:314` |
| POST | `/api/v2/admin/support/tickets/:ticketId/link` | SENSITIVE_COMMAND | - |  | yes | admin.support.link | `admin/support.ts:368` |
| POST | `/api/v2/admin/support/tickets/:ticketId/resolve` | SENSITIVE_COMMAND | - |  | yes | admin.support.resolve | `admin/support.ts:428` |
| POST | `/api/v2/admin/support/tickets/:ticketId/merge` | SENSITIVE_COMMAND | - |  | yes | admin.support.merge | `admin/support.ts:482` |
| DELETE | `/api/v2/admin/support/tickets/:ticketId` | SENSITIVE_COMMAND | - |  | yes | admin.support.delete | `admin/support.ts:608` |
| GET | `/api/v2/admin/tickets` | AUTH_READ | - |  |  |  | `admin/tickets.ts:57` |
| GET | `/api/v2/admin/tickets/export.csv` | AUTH_READ | - |  |  |  | `admin/tickets.ts:86` |
| POST | `/api/v2/admin/users/:userId/ban` | SENSITIVE_COMMAND | - |  | yes | admin.user.ban | `admin/user-actions.ts:45` |
| POST | `/api/v2/admin/users/:userId/unban` | SENSITIVE_COMMAND | - |  | yes | admin.user.unban | `admin/user-actions.ts:94` |
| POST | `/api/v2/admin/venues/:venueId/suspend` | SENSITIVE_COMMAND | - |  | yes | admin.venue.suspend | `admin/venue-actions.ts:50` |
| POST | `/api/v2/admin/venues/:venueId/reinstate` | SENSITIVE_COMMAND | - |  | yes | admin.venue.reinstate | `admin/venue-actions.ts:94` |

## auth

| Method | Path | Rate | Permission | Cache | Idem | Command | Source |
|---|---|---|---|---|---|---|---|
| POST | `/api/v2/auth/signup` | SENSITIVE_COMMAND | - |  |  |  | `auth/index.ts:59` |
| POST | `/api/v2/auth/login` | SENSITIVE_COMMAND | - |  |  |  | `auth/index.ts:106` |
| POST | `/api/v2/auth/forgot-password` | SENSITIVE_COMMAND | - |  |  |  | `auth/index.ts:156` |
| POST | `/api/v2/auth/reset-password` | SENSITIVE_COMMAND | - |  |  |  | `auth/index.ts:191` |
| POST | `/api/v2/auth/refresh` | SENSITIVE_COMMAND | - |  |  |  | `auth/index.ts:230` |
| POST | `/api/v2/auth/logout` | - | - |  |  |  | `auth/index.ts:258` |
| GET | `/api/v2/auth/session` | AUTH_READ | - |  |  |  | `auth/index.ts:268` |
| POST | `/api/v2/auth/change-password` | SENSITIVE_COMMAND | - |  |  |  | `auth/index.ts:295` |
| GET | `/api/v2/auth/invitations/mine` | AUTH_READ | - |  |  |  | `auth/index.ts:373` |
| POST | `/api/v2/auth/otp/send` | OTP_SEND | - |  |  |  | `auth/otp-routes.ts:32` |
| POST | `/api/v2/auth/otp/verify` | OTP_VERIFY | - |  |  |  | `auth/otp-routes.ts:72` |

## checkout

| Method | Path | Rate | Permission | Cache | Idem | Command | Source |
|---|---|---|---|---|---|---|---|
| POST | `/api/v2/checkout/quote` | STANDARD_COMMAND | - |  |  |  | `checkout/checkout-routes.ts:48` |
| POST | `/api/v2/checkout/holds` | STANDARD_COMMAND | - |  | yes | checkout.hold | `checkout/checkout-routes.ts:93` |
| GET | `/api/v2/payments/attempts/:holdId/status` | AUTH_READ | - |  |  |  | `checkout/payment-routes.ts:56` |
| POST | `/api/v2/payments/attempts` | STANDARD_COMMAND | - |  | yes | payment.attempt | `checkout/payment-routes.ts:85` |
| POST | `/api/v2/payments/:id/verify` | STANDARD_COMMAND | - |  | yes | payment.verify | `checkout/payment-routes.ts:156` |
| POST | `/api/v2/webhooks/payments/razorpay` | - | - |  |  |  | `checkout/webhook-routes.ts:109` |

## door

| Method | Path | Rate | Permission | Cache | Idem | Command | Source |
|---|---|---|---|---|---|---|---|
| POST | `/api/v2/cover-wallets` | STANDARD_COMMAND | door.manage |  | yes | cover_wallet.create | `door/cover-wallet-routes.ts:78` |
| GET | `/api/v2/cover-wallets/:walletId` | AUTH_READ | venue.read |  |  |  | `door/cover-wallet-routes.ts:137` |
| GET | `/api/v2/cover-wallets/:walletId/qr` | AUTH_READ | venue.read |  |  |  | `door/cover-wallet-routes.ts:184` |
| POST | `/api/v2/cover-wallets/:walletId/debit` | STANDARD_COMMAND | ticket.override |  | yes | cover_wallet.debit | `door/cover-wallet-routes.ts:215` |
| POST | `/api/v2/cover-wallets/:walletId/credit` | STANDARD_COMMAND | door.manage |  | yes | cover_wallet.credit | `door/cover-wallet-routes.ts:283` |
| POST | `/api/v2/cover-wallets/:walletId/freeze` | STANDARD_COMMAND | door.manage |  | yes | cover_wallet.freeze | `door/cover-wallet-routes.ts:345` |
| POST | `/api/v2/cover-wallets/:walletId/unfreeze` | STANDARD_COMMAND | door.manage |  | yes | cover_wallet.unfreeze | `door/cover-wallet-routes.ts:395` |
| POST | `/api/v2/cover-wallets/:walletId/terminate` | STANDARD_COMMAND | door.manage |  | yes | cover_wallet.terminate | `door/cover-wallet-routes.ts:443` |
| POST | `/api/v2/cover-wallets/:walletId/reconcile` | STANDARD_COMMAND | door.manage |  | yes | cover_wallet.reconcile | `door/cover-wallet-routes.ts:499` |
| GET | `/api/v2/door/events` | AUTH_READ | venue.read |  |  |  | `door/door-ops-routes.ts:70` |
| POST | `/api/v2/door/devices` | STANDARD_COMMAND | door.manage |  |  |  | `door/door-ops-routes.ts:98` |
| GET | `/api/v2/door/devices` | AUTH_READ | door.manage |  |  |  | `door/door-ops-routes.ts:127` |
| POST | `/api/v2/door/devices/:deviceId/unbind` | SENSITIVE_COMMAND | door.manage |  |  |  | `door/door-ops-routes.ts:155` |
| POST | `/api/v2/door/devices/:deviceId/reauthorize` | SENSITIVE_COMMAND | door.manage |  |  |  | `door/door-ops-routes.ts:189` |
| POST | `/api/v2/door/heartbeat` | SCANNER_COMMAND | - |  |  |  | `door/door-ops-routes.ts:222` |
| POST | `/api/v2/door/check-ins/confirm` | SCANNER_COMMAND | - |  | yes | door.check-in.confirm | `door/door-ops-routes.ts:260` |
| POST | `/api/v2/door/staff-deny` | SCANNER_COMMAND | - |  |  |  | `door/door-ops-routes.ts:320` |
| GET | `/api/v2/door/guests` | AUTH_READ | venue.read |  |  |  | `door/door-ops-routes.ts:360` |
| GET | `/api/v2/door/attendance-report` | AUTH_READ | venue.read |  |  |  | `door/door-ops-routes.ts:401` |
| POST | `/api/v2/door/guests/check-in` | STANDARD_COMMAND | ticket.override |  | yes | door.guest.check-in | `door/door-ops-routes.ts:431` |
| POST | `/api/v2/door/wallet-qr` | SCANNER_COMMAND | - |  |  |  | `door/door-ops-routes.ts:480` |
| POST | `/api/v2/door/wallet-charge` | SCANNER_COMMAND | - |  | yes | door.wallet.charge | `door/door-ops-routes.ts:519` |
| POST | `/api/v2/door/ticket-sale` | STANDARD_COMMAND | - |  | yes | door.ticket-sale | `door/door-ops-routes.ts:582` |
| POST | `/api/v2/door/walk-in` | STANDARD_COMMAND | ticket.override |  | yes | door.walkin | `door/door-sale-routes.ts:67` |
| POST | `/api/v2/door/dine-in` | STANDARD_COMMAND | ticket.override |  | yes | door.dinein | `door/door-sale-routes.ts:130` |
| GET | `/api/v2/door/sales` | AUTH_READ | venue.read |  |  |  | `door/door-sale-routes.ts:195` |
| POST | `/api/v2/events/:eventId/door-codes` | STANDARD_COMMAND | door.manage |  | yes | door.code.create | `door/event-code-routes.ts:45` |
| GET | `/api/v2/events/:eventId/door-codes` | AUTH_READ | door.manage |  |  |  | `door/event-code-routes.ts:104` |
| POST | `/api/v2/door-codes/:codeId/revoke` | SENSITIVE_COMMAND | door.manage |  |  |  | `door/event-code-routes.ts:134` |
| GET | `/api/v2/door-codes/:codeId/sessions` | AUTH_READ | door.manage |  |  |  | `door/event-code-routes.ts:167` |
| POST | `/api/v2/door/sessions/:sessionId/revoke` | SENSITIVE_COMMAND | door.manage |  |  |  | `door/event-code-routes.ts:199` |
| POST | `/api/v2/door/sessions` | SENSITIVE_COMMAND | - |  | yes | door.session.open | `door/scanner-routes.ts:74` |
| GET | `/api/v2/door/sessions/:sessionId` | AUTH_READ | - |  |  |  | `door/scanner-routes.ts:126` |
| POST | `/api/v2/door/check-ins` | SCANNER_COMMAND | - |  | yes | door.check-in | `door/scanner-routes.ts:154` |
| POST | `/api/v2/door/check-ins/verify` | SCANNER_COMMAND | - |  |  |  | `door/scanner-routes.ts:211` |
| POST | `/api/v2/door/lookup` | SCANNER_COMMAND | - |  |  |  | `door/scanner-routes.ts:228` |
| GET | `/api/v2/door/check-ins/:checkInId` | AUTH_READ | - |  |  |  | `door/scanner-routes.ts:269` |
| POST | `/api/v2/door/override` | SENSITIVE_COMMAND | ticket.override |  |  |  | `door/scanner-routes.ts:293` |
| GET | `/api/v2/door/offline-manifest` | STANDARD_COMMAND | - |  |  |  | `door/scanner-routes.ts:328` |
| POST | `/api/v2/door/offline-sync` | STANDARD_COMMAND | - |  | yes | door.offline-sync | `door/scanner-routes.ts:362` |
| GET | `/api/v2/tickets/:ticketId/qr` | AUTH_READ | - |  |  |  | `door/scanner-routes.ts:413` |

## finance

| Method | Path | Rate | Permission | Cache | Idem | Command | Source |
|---|---|---|---|---|---|---|---|
| GET | `/api/v2/organizations/:organizationId/finance/balance` | AUTH_READ | finance.read |  |  |  | `finance/finance-routes.ts:58` |
| GET | `/api/v2/organizations/:organizationId/finance/ledger` | AUTH_READ | finance.read |  |  |  | `finance/finance-routes.ts:81` |
| GET | `/api/v2/organizations/:organizationId/finance/orders` | AUTH_READ | finance.orders.read |  |  |  | `finance/finance-routes.ts:128` |
| POST | `/api/v2/organizations/:organizationId/payouts` | STANDARD_COMMAND | finance.payouts.manage |  | yes | finance.payout.request | `finance/finance-routes.ts:163` |
| GET | `/api/v2/organizations/:organizationId/payouts` | AUTH_READ | finance.payouts.read |  |  |  | `finance/finance-routes.ts:214` |
| GET | `/api/v2/organizations/:organizationId/payouts/:payoutId` | AUTH_READ | finance.payouts.read |  |  |  | `finance/finance-routes.ts:246` |
| POST | `/api/v2/organizations/:organizationId/bank-accounts` | STANDARD_COMMAND | finance.bank_accounts.manage |  |  |  | `finance/finance-routes.ts:280` |
| GET | `/api/v2/organizations/:organizationId/bank-accounts` | AUTH_READ | finance.read |  |  |  | `finance/finance-routes.ts:318` |
| POST | `/api/v2/organizations/:organizationId/bank-accounts/:bankAccountId/default` | STANDARD_COMMAND | finance.bank_accounts.manage |  |  |  | `finance/finance-routes.ts:343` |
| DELETE | `/api/v2/organizations/:organizationId/bank-accounts/:bankAccountId` | STANDARD_COMMAND | finance.bank_accounts.manage |  |  |  | `finance/finance-routes.ts:373` |
| POST | `/api/v2/organizations/:organizationId/disputes` | STANDARD_COMMAND | organization.update |  |  |  | `finance/finance-routes.ts:400` |
| GET | `/api/v2/organizations/:organizationId/disputes` | AUTH_READ | organization.read |  |  |  | `finance/finance-routes.ts:438` |
| GET | `/api/v2/organizations/:organizationId/disputes/:disputeId` | AUTH_READ | organization.read |  |  |  | `finance/finance-routes.ts:470` |
| POST | `/api/v2/organizations/:organizationId/disputes/:disputeId/review` | STANDARD_COMMAND | organization.update |  |  |  | `finance/finance-routes.ts:500` |
| POST | `/api/v2/organizations/:organizationId/disputes/:disputeId/resolve` | STANDARD_COMMAND | organization.update |  |  |  | `finance/finance-routes.ts:528` |
| GET | `/api/v2/leaderboard` | PUBLIC_READ | - |  |  |  | `finance/leaderboard-routes.ts:50` |
| GET | `/api/v2/organizations/:organizationId/leaderboard/me` | AUTH_READ | organization.read |  |  |  | `finance/leaderboard-routes.ts:76` |

## internal

| Method | Path | Rate | Permission | Cache | Idem | Command | Source |
|---|---|---|---|---|---|---|---|
| GET | `/api/v2/internal/health` | - | - |  |  |  | `internal/index.ts:28` |
| GET | `/api/v2/internal/version` | - | - |  |  |  | `internal/index.ts:36` |
| GET | `/api/v2/internal/readiness` | - | - |  |  |  | `internal/index.ts:47` |

## notifications

| Method | Path | Rate | Permission | Cache | Idem | Command | Source |
|---|---|---|---|---|---|---|---|
| GET | `/api/v2/organizations/:organizationId/notifications` | AUTH_READ | organization.read |  |  |  | `notifications/notifications-routes.ts:48` |
| PATCH | `/api/v2/organizations/:organizationId/notifications/read-all` | STANDARD_COMMAND | organization.read |  | yes |  | `notifications/notifications-routes.ts:91` |
| PATCH | `/api/v2/organizations/:organizationId/notifications/:notificationId/read` | STANDARD_COMMAND | organization.read |  |  |  | `notifications/notifications-routes.ts:116` |
| POST | `/api/v2/organizations/:organizationId/notifications/:notificationId/actions` | STANDARD_COMMAND | organization.read |  | yes | notifications.action | `notifications/notifications-routes.ts:148` |

## onboarding

| Method | Path | Rate | Permission | Cache | Idem | Command | Source |
|---|---|---|---|---|---|---|---|
| GET | `/api/v2/onboarding/me` | AUTH_READ | - |  |  |  | `onboarding.ts:49` |
| GET | `/api/v2/onboarding/applications` | AUTH_READ | - |  |  |  | `onboarding.ts:73` |
| POST | `/api/v2/onboarding/applications` | STANDARD_COMMAND | - |  | yes |  | `onboarding.ts:105` |
| PATCH | `/api/v2/onboarding/applications/:requestId` | STANDARD_COMMAND | - |  | yes |  | `onboarding.ts:132` |
| POST | `/api/v2/onboarding/applications/:requestId/documents/upload-url` | STANDARD_COMMAND | - |  |  |  | `onboarding.ts:169` |
| POST | `/api/v2/onboarding/applications/:requestId/documents` | STANDARD_COMMAND | - |  | yes |  | `onboarding.ts:194` |
| POST | `/api/v2/onboarding/applications/:requestId/submit` | STANDARD_COMMAND | - |  | yes |  | `onboarding.ts:225` |
| POST | `/api/v2/onboarding/verify-document` | SENSITIVE_COMMAND | - |  | yes |  | `onboarding.ts:257` |

## orders

| Method | Path | Rate | Permission | Cache | Idem | Command | Source |
|---|---|---|---|---|---|---|---|
| GET | `/api/v2/orders` | AUTH_READ | - |  |  |  | `orders/orders-routes.ts:29` |
| GET | `/api/v2/orders/:id` | AUTH_READ | - |  |  |  | `orders/orders-routes.ts:59` |
| GET | `/api/v2/orders/:id/status` | AUTH_READ | - |  |  |  | `orders/orders-routes.ts:75` |

## partner

| Method | Path | Rate | Permission | Cache | Idem | Command | Source |
|---|---|---|---|---|---|---|---|
| GET | `/api/v2/organizations/:organizationId/analytics/overview` | AUTH_READ | organization.read | ORGANIZATION |  |  | `partner/analytics.ts:72` |
| GET | `/api/v2/organizations/:organizationId/analytics/trends` | AUTH_READ | organization.read | ORGANIZATION |  |  | `partner/analytics.ts:101` |
| GET | `/api/v2/organizations/:organizationId/analytics/events` | AUTH_READ | organization.read | ORGANIZATION |  |  | `partner/analytics.ts:131` |
| GET | `/api/v2/organizations/:organizationId/analytics/calendar` | AUTH_READ | organization.read | ORGANIZATION |  |  | `partner/analytics.ts:166` |
| GET | `/api/v2/events/:eventId/analytics` | AUTH_READ | event.read | EVENT_PREVIEW |  |  | `partner/analytics.ts:196` |
| GET | `/api/v2/organizations/:organizationId/discover-partners` | AUTH_READ | organization.read |  |  |  | `partner/discover.ts:31` |
| GET | `/api/v2/events/:eventId/ticket-tiers` | AUTH_READ | event.read |  |  |  | `partner/event-catalog.ts:60` |
| POST | `/api/v2/events/:eventId/ticket-tiers` | STANDARD_COMMAND | event.update |  | yes | event-ticket-tiers.create | `partner/event-catalog.ts:85` |
| GET | `/api/v2/events/:eventId/promo-codes` | AUTH_READ | event.read |  |  |  | `partner/event-catalog.ts:135` |
| POST | `/api/v2/events/:eventId/promo-codes` | STANDARD_COMMAND | event.update |  | yes | event-promo-codes.create | `partner/event-catalog.ts:174` |
| GET | `/api/v2/events/:eventId/table-packages` | AUTH_READ | event.read |  |  |  | `partner/event-catalog.ts:224` |
| POST | `/api/v2/events/:eventId/table-packages` | STANDARD_COMMAND | event.update |  | yes | event-table-packages.create | `partner/event-catalog.ts:249` |
| GET | `/api/v2/events/:eventId/promoter-assignments` | AUTH_READ | event.read |  |  |  | `partner/event-catalog.ts:299` |
| POST | `/api/v2/events/:eventId/promoter-assignments` | STANDARD_COMMAND | event.update |  | yes | event-promoter-assignments.create | `partner/event-catalog.ts:329` |
| POST | `/api/v2/promoter-assignments/:assignmentId/end` | STANDARD_COMMAND | event.update |  | yes |  | `partner/event-catalog.ts:391` |
| GET | `/api/v2/promoters/:promoterId/events` | AUTH_READ | event.read |  |  |  | `partner/events.ts:84` |
| GET | `/api/v2/organizations/:organizationId/events` | AUTH_READ | event.read |  |  |  | `partner/events.ts:130` |
| GET | `/api/v2/events/:eventId` | AUTH_READ | event.read |  |  |  | `partner/events.ts:169` |
| POST | `/api/v2/organizations/:organizationId/events` | STANDARD_COMMAND | event.create |  | yes | event.create | `partner/events.ts:195` |
| POST | `/api/v2/organizations/:organizationId/poster/upload-url` | STANDARD_COMMAND | event.create |  |  |  | `partner/events.ts:252` |
| PATCH | `/api/v2/events/:eventId` | STANDARD_COMMAND | event.update |  | yes | event.update | `partner/events.ts:281` |
| GET | `/api/v2/events/:eventId/previews` | AUTH_READ | event.read |  |  |  | `partner/events.ts:336` |
| POST | `/api/v2/events/:eventId/${action}` | STANDARD_COMMAND | - |  | yes |  | `partner/events.ts:393` |
| POST | `/api/v2/events/:eventId/cancel` | STANDARD_COMMAND | event.cancel |  | yes | event.cancel | `partner/events.ts:438` |
| GET | `/api/v2/organizations` | AUTH_READ | organization.read |  |  |  | `partner/organizations.ts:82` |
| GET | `/api/v2/organizations/:organizationId` | AUTH_READ | organization.read | ORGANIZATION |  |  | `partner/organizations.ts:115` |
| POST | `/api/v2/organizations` | STANDARD_COMMAND | - |  | yes | organizations.create | `partner/organizations.ts:146` |
| PATCH | `/api/v2/organizations/:organizationId` | STANDARD_COMMAND | organization.update |  | yes | organizations.update | `partner/organizations.ts:199` |
| GET | `/api/v2/organizations/:organizationId/members` | AUTH_READ | organization.read |  |  |  | `partner/organizations.ts:282` |
| POST | `/api/v2/organizations/:organizationId/members` | STANDARD_COMMAND | staff.manage |  | yes | organizations.inviteMember | `partner/organizations.ts:320` |
| PATCH | `/api/v2/organizations/:organizationId/members/:userId` | STANDARD_COMMAND | staff.manage |  | yes | organizations.changeRole | `partner/organizations.ts:370` |
| DELETE | `/api/v2/organizations/:organizationId/members/:userId` | STANDARD_COMMAND | staff.manage |  | yes | organizations.removeMember | `partner/organizations.ts:438` |
| GET | `/api/v2/organizations/:organizationId/access` | AUTH_READ | organization.read |  |  |  | `partner/organizations.ts:482` |
| GET | `/api/v2/organizations/:organizationId/invitations` | AUTH_READ | staff.manage |  |  |  | `partner/organizations.ts:525` |
| POST | `/api/v2/organizations/:organizationId/invitations` | STANDARD_COMMAND | staff.manage |  | yes | organization-invitations.create | `partner/organizations.ts:565` |
| POST | `/api/v2/invitations/:invitationId/revoke` | STANDARD_COMMAND | staff.manage |  | yes |  | `partner/organizations.ts:621` |
| POST | `/api/v2/invitations/:invitationId/accept` | STANDARD_COMMAND | - |  |  |  | `partner/organizations.ts:655` |
| GET | `/api/v2/organizations/:organizationId/partnerships` | AUTH_READ | organization.read |  |  |  | `partner/partnerships.ts:49` |
| POST | `/api/v2/partnerships` | STANDARD_COMMAND | venue.manage |  | yes | partnerships.request | `partner/partnerships.ts:90` |
| POST | `/api/v2/partnerships/:partnershipId/venue-share` | STANDARD_COMMAND | venue.manage |  | yes | partnerships.venue-share | `partner/partnerships.ts:165` |
| POST | `/api/v2/partnerships/:partnershipId/${action}` | STANDARD_COMMAND | - |  | yes |  | `partner/partnerships.ts:228` |
| GET | `/api/v2/organizations/:organizationId/promoter-connections` | AUTH_READ | organization.read |  |  |  | `partner/promoter-connections.ts:42` |
| POST | `/api/v2/promoter-connections` | STANDARD_COMMAND | venue.manage |  | yes | promoter-connections.request | `partner/promoter-connections.ts:82` |
| POST | `/api/v2/promoter-connections/:connectionId/${action}` | STANDARD_COMMAND | - |  | yes |  | `partner/promoter-connections.ts:169` |
| POST | `/api/v2/promoter-assignments/:assignmentId/referral-links` | STANDARD_COMMAND | event.read |  | yes | promoter-assignment.referral-link.create | `partner/referral-links.ts:59` |
| GET | `/api/v2/events/:eventId/referral-links` | AUTH_READ | event.read |  |  |  | `partner/referral-links.ts:115` |
| POST | `/api/v2/events/:eventId/referral-links` | STANDARD_COMMAND | event.update |  | yes | referral-links.create | `partner/referral-links.ts:154` |
| POST | `/api/v2/referral-links/:referralLinkId/deactivate` | STANDARD_COMMAND | event.update |  | yes |  | `partner/referral-links.ts:210` |
| GET | `/api/v2/organizations/:organizationId/venues` | AUTH_READ | venue.read |  |  |  | `partner/venues.ts:126` |
| POST | `/api/v2/organizations/:organizationId/venues` | STANDARD_COMMAND | venue.create |  | yes | venue.create | `partner/venues.ts:165` |
| GET | `/api/v2/venues/:venueId` | AUTH_READ | venue.read |  |  |  | `partner/venues.ts:217` |
| PATCH | `/api/v2/venues/:venueId` | STANDARD_COMMAND | venue.manage |  | yes | venue.update | `partner/venues.ts:242` |
| GET | `/api/v2/venues/:venueId/profile` | - | - |  |  |  | `partner/venues.ts:297` |
| PATCH | `/api/v2/venues/:venueId/profile` | STANDARD_COMMAND | venue.manage |  | yes | venue.profile.update | `partner/venues.ts:318` |
| GET | `/api/v2/venues/:venueId/calendar` | AUTH_READ | venue.read |  |  |  | `partner/venues.ts:373` |
| POST | `/api/v2/venues/:venueId/calendar/blocks` | STANDARD_COMMAND | venue.manage |  | yes | venue-calendar.block | `partner/venues.ts:402` |
| DELETE | `/api/v2/venues/:venueId/calendar/blocks/:blockId` | STANDARD_COMMAND | venue.manage |  | yes | venue-calendar.unblock | `partner/venues.ts:446` |
| GET | `/api/v2/venues/:venueId/menu` | AUTH_READ | venue.read | VENUE_PROFILE |  |  | `partner/venues.ts:488` |
| PUT | `/api/v2/venues/:venueId/menu` | STANDARD_COMMAND | venue.manage |  | yes | venue-menu.update | `partner/venues.ts:513` |
| GET | `/api/v2/venues/:venueId/availability` | AUTH_READ | venue.read | AVAILABILITY |  |  | `partner/venues.ts:571` |
| GET | `/api/v2/venues/:venueId/slot-requests` | AUTH_READ | venue.read |  |  |  | `partner/venues.ts:607` |
| GET | `/api/v2/organizations/:organizationId/slot-requests` | AUTH_READ | slot-request.list |  |  |  | `partner/venues.ts:646` |
| GET | `/api/v2/venues/:venueId/slot-requests/:slotRequestId` | AUTH_READ | venue.read |  |  |  | `partner/venues.ts:684` |
| POST | `/api/v2/venues/:venueId/slot-requests` | STANDARD_COMMAND | slot-request.create |  | yes | venue.slotRequest.create | `partner/venues.ts:717` |
| POST | `/api/v2/venues/slot-requests/:slotRequestId/${action}` | STANDARD_COMMAND | venue.schedule |  | yes |  | `partner/venues.ts:771` |
| POST | `/api/v2/venues/slot-requests/:slotRequestId/cancel` | STANDARD_COMMAND | slot-request.create |  | yes | venue.slotRequest.cancel | `partner/venues.ts:817` |

## phase5-routes

| Method | Path | Rate | Permission | Cache | Idem | Command | Source |
|---|---|---|---|---|---|---|---|
| GET | `/api/v2/door/stats` | AUTH_READ | - |  |  |  | `phase5-routes.ts:75` |
| GET | `/api/v2/door/stats/stream` | AUTH_READ | - |  |  |  | `phase5-routes.ts:100` |

## profile

| Method | Path | Rate | Permission | Cache | Idem | Command | Source |
|---|---|---|---|---|---|---|---|
| GET | `/api/v2/profile/me` | AUTH_READ | - |  |  |  | `profile.ts:30` |
| PUT | `/api/v2/profile/me` | STANDARD_COMMAND | - |  |  |  | `profile.ts:52` |

## public

| Method | Path | Rate | Permission | Cache | Idem | Command | Source |
|---|---|---|---|---|---|---|---|
| GET | `/api/v2/public/promoter-links/:handle/:slug` | PUBLIC_READ | - |  |  |  | `public/discovery.ts:85` |
| POST | `/api/v2/public/promoter-links/click` | PUBLIC_READ | - |  |  |  | `public/discovery.ts:105` |
| GET | `/api/v2/public/events` | PUBLIC_READ | - |  |  |  | `public/discovery.ts:125` |
| GET | `/api/v2/public/events/:idOrSlug` | PUBLIC_READ | - |  |  |  | `public/discovery.ts:155` |
| GET | `/api/v2/public/venues/by-id/:venueId` | PUBLIC_READ | - |  |  |  | `public/discovery.ts:182` |
| GET | `/api/v2/public/venues/:slug` | PUBLIC_READ | - |  |  |  | `public/discovery.ts:200` |
| GET | `/api/v2/public/hosts/by-id/:organizationId` | PUBLIC_READ | - |  |  |  | `public/discovery.ts:225` |
| GET | `/api/v2/public/hosts/:slug` | PUBLIC_READ | - |  |  |  | `public/discovery.ts:246` |
| GET | `/api/v2/public/discovery` | PUBLIC_READ | - |  |  |  | `public/discovery.ts:264` |
| GET | `/api/v2/public/search` | PUBLIC_READ | - |  |  |  | `public/discovery.ts:277` |
| GET | `/api/v2/public/events/:idOrSlug/tiers` | PUBLIC_READ | - |  |  |  | `public/discovery.ts:311` |

## rsvp

| Method | Path | Rate | Permission | Cache | Idem | Command | Source |
|---|---|---|---|---|---|---|---|
| POST | `/api/v2/rsvp` | STANDARD_COMMAND | - |  | yes | rsvp.create | `rsvp/rsvp-routes.ts:38` |

## social

| Method | Path | Rate | Permission | Cache | Idem | Command | Source |
|---|---|---|---|---|---|---|---|
| POST | `/api/v2/follows` | STANDARD_COMMAND | - |  |  |  | `social/social-routes.ts:52` |
| DELETE | `/api/v2/follows/:targetType/:targetId` | STANDARD_COMMAND | - |  |  |  | `social/social-routes.ts:81` |
| GET | `/api/v2/follows/me` | AUTH_READ | - |  |  |  | `social/social-routes.ts:103` |
| GET | `/api/v2/follows/:targetType/:targetId/status` | AUTH_READ | - |  |  |  | `social/social-routes.ts:136` |
| GET | `/api/v2/notifications/me` | AUTH_READ | - |  |  |  | `social/social-routes.ts:160` |
| GET | `/api/v2/notifications/me/unread-count` | AUTH_READ | - |  |  |  | `social/social-routes.ts:193` |
| POST | `/api/v2/notifications/me/read` | STANDARD_COMMAND | - |  |  |  | `social/social-routes.ts:211` |
| POST | `/api/v2/notifications/me/read-all` | STANDARD_COMMAND | - |  |  |  | `social/social-routes.ts:235` |

## support

| Method | Path | Rate | Permission | Cache | Idem | Command | Source |
|---|---|---|---|---|---|---|---|
| POST | `/api/v2/support/tickets` | SENSITIVE_COMMAND | - |  | yes | support.ticket.submit | `support/intake-routes.ts:88` |
| GET | `/api/v2/support/tickets` | AUTH_READ | - |  |  |  | `support/intake-routes.ts:138` |
| GET | `/api/v2/support/tickets/:ticketId` | AUTH_READ | - |  |  |  | `support/intake-routes.ts:169` |
| POST | `/api/v2/support/tickets/:ticketId/messages` | SENSITIVE_COMMAND | - |  | yes | support.ticket.message | `support/intake-routes.ts:194` |

## tickets

| Method | Path | Rate | Permission | Cache | Idem | Command | Source |
|---|---|---|---|---|---|---|---|
| GET | `/api/v2/tickets/:id` | AUTH_READ | - |  |  |  | `tickets/ticket-routes.ts:26` |

## wallet

| Method | Path | Rate | Permission | Cache | Idem | Command | Source |
|---|---|---|---|---|---|---|---|
| GET | `/api/v2/wallet` | AUTH_READ | - |  |  |  | `wallet/wallet-routes.ts:28` |
| GET | `/api/v2/wallet/tickets` | AUTH_READ | - |  |  |  | `wallet/wallet-routes.ts:50` |
| GET | `/api/v2/wallet/orders` | AUTH_READ | - |  |  |  | `wallet/wallet-routes.ts:80` |

## Routes with no `requirePermission` (review list)

| Method | Path | Rate | Source |
|---|---|---|---|
| GET | `/api/v2/admin/analytics` | AUTH_READ | `admin/analytics.ts:21` |
| GET | `/api/v2/admin/venues` | AUTH_READ | `admin/directory.ts:117` |
| GET | `/api/v2/admin/events` | AUTH_READ | `admin/directory.ts:146` |
| GET | `/api/v2/admin/hosts` | AUTH_READ | `admin/directory.ts:175` |
| GET | `/api/v2/admin/users` | AUTH_READ | `admin/directory.ts:204` |
| GET | `/api/v2/admin/lookup` | AUTH_READ | `admin/directory.ts:233` |
| GET | `/api/v2/admin/venues/export.csv` | AUTH_READ | `admin/directory.ts:263` |
| GET | `/api/v2/admin/events/export.csv` | AUTH_READ | `admin/directory.ts:308` |
| GET | `/api/v2/admin/hosts/export.csv` | AUTH_READ | `admin/directory.ts:361` |
| GET | `/api/v2/admin/users/export.csv` | AUTH_READ | `admin/directory.ts:411` |
| GET | `/api/v2/admin/audit/export.csv` | AUTH_READ | `admin/directory.ts:446` |
| POST | `/api/v2/admin/disputes/:disputeId/resolve` | SENSITIVE_COMMAND | `admin/disputes.ts:62` |
| GET | `/api/v2/admin/disputes` | AUTH_READ | `admin/disputes.ts:117` |
| GET | `/api/v2/admin/disputes/:disputeId` | AUTH_READ | `admin/disputes.ts:152` |
| POST | `/api/v2/admin/events/:eventId/pause` | SENSITIVE_COMMAND | `admin/event-actions.ts:52` |
| POST | `/api/v2/admin/events/:eventId/resume` | SENSITIVE_COMMAND | `admin/event-actions.ts:96` |
| POST | `/api/v2/admin/events/:eventId/force-complete` | SENSITIVE_COMMAND | `admin/event-actions.ts:140` |
| GET | `/api/v2/admin/onboarding/applications` | AUTH_READ | `admin/onboarding-review.ts:77` |
| GET | `/api/v2/admin/onboarding/applications/:requestId` | AUTH_READ | `admin/onboarding-review.ts:112` |
| GET | `/api/v2/admin/onboarding/applications/:requestId/documents/:label/read-url` | AUTH_READ | `admin/onboarding-review.ts:144` |
| POST | `/api/v2/admin/onboarding/applications/:requestId/documents/:label/verify` | SENSITIVE_COMMAND | `admin/onboarding-review.ts:173` |
| POST | `/api/v2/admin/onboarding/applications/:requestId/documents/:label/reject` | SENSITIVE_COMMAND | `admin/onboarding-review.ts:223` |
| POST | `/api/v2/admin/onboarding/applications/:requestId/approve` | SENSITIVE_COMMAND | `admin/onboarding-review.ts:283` |
| GET | `/api/v2/admin/proposals` | AUTH_READ | `admin/onboarding-review.ts:356` |
| POST | `/api/v2/admin/proposals` | SENSITIVE_COMMAND | `admin/onboarding-review.ts:391` |
| GET | `/api/v2/admin/admins` | AUTH_READ | `admin/onboarding-review.ts:422` |
| POST | `/api/v2/admin/proposals/:proposalId/provision-admin` | SENSITIVE_COMMAND | `admin/onboarding-review.ts:460` |
| POST | `/api/v2/admin/proposals/:proposalId/update-admin-role` | SENSITIVE_COMMAND | `admin/onboarding-review.ts:512` |
| POST | `/api/v2/admin/admins/:adminId/revoke` | SENSITIVE_COMMAND | `admin/onboarding-review.ts:560` |
| GET | `/api/v2/admin/audit` | AUTH_READ | `admin/onboarding-review.ts:610` |
| POST | `/api/v2/admin/onboarding/applications/:requestId/${action}` | SENSITIVE_COMMAND | `admin/onboarding-review.ts:650` |
| POST | `/api/v2/admin/proposals/:proposalId/${action}` | SENSITIVE_COMMAND | `admin/onboarding-review.ts:710` |
| GET | `/api/v2/admin/orders` | AUTH_READ | `admin/orders.ts:60` |
| GET | `/api/v2/admin/orders/export.csv` | AUTH_READ | `admin/orders.ts:89` |
| POST | `/api/v2/admin/proposals/:proposalId/adjust-commission` | SENSITIVE_COMMAND | `admin/organization-actions.ts:46` |
| POST | `/api/v2/admin/organizations/:organizationId/suspend` | SENSITIVE_COMMAND | `admin/organization-actions.ts:89` |
| POST | `/api/v2/admin/organizations/:organizationId/reinstate` | SENSITIVE_COMMAND | `admin/organization-actions.ts:132` |
| POST | `/api/v2/admin/proposals/:proposalId/freeze-payout` | SENSITIVE_COMMAND | `admin/payouts.ts:72` |
| POST | `/api/v2/admin/proposals/:proposalId/release-payout` | SENSITIVE_COMMAND | `admin/payouts.ts:120` |
| POST | `/api/v2/admin/payouts/batch-run` | SENSITIVE_COMMAND | `admin/payouts.ts:171` |
| POST | `/api/v2/admin/payouts/:payoutId/complete` | SENSITIVE_COMMAND | `admin/payouts.ts:220` |
| GET | `/api/v2/admin/payouts` | AUTH_READ | `admin/payouts.ts:271` |
| GET | `/api/v2/admin/payouts/:payoutId` | AUTH_READ | `admin/payouts.ts:306` |
| GET | `/api/v2/admin/promoters` | AUTH_READ | `admin/promoters.ts:69` |
| GET | `/api/v2/admin/promoters/export.csv` | AUTH_READ | `admin/promoters.ts:98` |
| POST | `/api/v2/admin/promoters/:promoterId/suspend` | SENSITIVE_COMMAND | `admin/promoters.ts:141` |
| POST | `/api/v2/admin/promoters/:promoterId/reinstate` | SENSITIVE_COMMAND | `admin/promoters.ts:189` |
| GET | `/api/v2/admin/promotions` | AUTH_READ | `admin/promotions.ts:59` |
| GET | `/api/v2/admin/promotions/export.csv` | AUTH_READ | `admin/promotions.ts:88` |
| POST | `/api/v2/admin/proposals/:proposalId/post-financial-adjustment` | SENSITIVE_COMMAND | `admin/reconciliation.ts:71` |
| GET | `/api/v2/admin/chargebacks` | AUTH_READ | `admin/reconciliation.ts:150` |
| POST | `/api/v2/admin/reconciliation/runs` | SENSITIVE_COMMAND | `admin/reconciliation.ts:198` |
| GET | `/api/v2/admin/reconciliation/runs` | AUTH_READ | `admin/reconciliation.ts:245` |
| GET | `/api/v2/admin/reconciliation/runs/:runId` | AUTH_READ | `admin/reconciliation.ts:266` |
| GET | `/api/v2/admin/reconciliation/discrepancies` | AUTH_READ | `admin/reconciliation.ts:280` |
| GET | `/api/v2/admin/reconciliation/discrepancies/:discrepancyId` | AUTH_READ | `admin/reconciliation.ts:301` |
| POST | `/api/v2/admin/reconciliation/discrepancies/:discrepancyId/retry` | SENSITIVE_COMMAND | `admin/reconciliation.ts:332` |
| POST | `/api/v2/admin/reconciliation/discrepancies/:discrepancyId/resolve` | SENSITIVE_COMMAND | `admin/reconciliation.ts:386` |
| GET | `/api/v2/admin/reconciliation/export.csv` | - | `admin/reconciliation.ts:443` |
| POST | `/api/v2/admin/refunds` | SENSITIVE_COMMAND | `admin/refunds.ts:92` |
| POST | `/api/v2/admin/refunds/:refundRequestId/approve` | SENSITIVE_COMMAND | `admin/refunds.ts:147` |
| POST | `/api/v2/admin/refunds/:refundRequestId/reject` | SENSITIVE_COMMAND | `admin/refunds.ts:195` |
| POST | `/api/v2/admin/refunds/:refundRequestId/execute` | SENSITIVE_COMMAND | `admin/refunds.ts:251` |
| GET | `/api/v2/admin/refunds` | AUTH_READ | `admin/refunds.ts:301` |
| GET | `/api/v2/admin/refunds/export.csv` | AUTH_READ | `admin/refunds.ts:336` |
| GET | `/api/v2/admin/refunds/:refundRequestId` | AUTH_READ | `admin/refunds.ts:385` |
| GET | `/api/v2/admin/settings/platform` | AUTH_READ | `admin/settings.ts:60` |
| PUT | `/api/v2/admin/settings/platform` | SENSITIVE_COMMAND | `admin/settings.ts:85` |
| GET | `/api/v2/admin/support/tickets` | AUTH_READ | `admin/support.ts:76` |
| GET | `/api/v2/admin/support/tickets/:ticketId` | AUTH_READ | `admin/support.ts:120` |
| POST | `/api/v2/admin/support/tickets/:ticketId/assign` | SENSITIVE_COMMAND | `admin/support.ts:152` |
| POST | `/api/v2/admin/support/tickets/:ticketId/priority` | SENSITIVE_COMMAND | `admin/support.ts:206` |
| POST | `/api/v2/admin/support/tickets/:ticketId/reply` | SENSITIVE_COMMAND | `admin/support.ts:260` |
| POST | `/api/v2/admin/support/tickets/:ticketId/notes` | SENSITIVE_COMMAND | `admin/support.ts:314` |
| POST | `/api/v2/admin/support/tickets/:ticketId/link` | SENSITIVE_COMMAND | `admin/support.ts:368` |
| POST | `/api/v2/admin/support/tickets/:ticketId/resolve` | SENSITIVE_COMMAND | `admin/support.ts:428` |
| POST | `/api/v2/admin/support/tickets/:ticketId/merge` | SENSITIVE_COMMAND | `admin/support.ts:482` |
| DELETE | `/api/v2/admin/support/tickets/:ticketId` | SENSITIVE_COMMAND | `admin/support.ts:608` |
| GET | `/api/v2/admin/tickets` | AUTH_READ | `admin/tickets.ts:57` |
| GET | `/api/v2/admin/tickets/export.csv` | AUTH_READ | `admin/tickets.ts:86` |
| POST | `/api/v2/admin/users/:userId/ban` | SENSITIVE_COMMAND | `admin/user-actions.ts:45` |
| POST | `/api/v2/admin/users/:userId/unban` | SENSITIVE_COMMAND | `admin/user-actions.ts:94` |
| POST | `/api/v2/admin/venues/:venueId/suspend` | SENSITIVE_COMMAND | `admin/venue-actions.ts:50` |
| POST | `/api/v2/admin/venues/:venueId/reinstate` | SENSITIVE_COMMAND | `admin/venue-actions.ts:94` |
| POST | `/api/v2/auth/signup` | SENSITIVE_COMMAND | `auth/index.ts:59` |
| POST | `/api/v2/auth/login` | SENSITIVE_COMMAND | `auth/index.ts:106` |
| POST | `/api/v2/auth/forgot-password` | SENSITIVE_COMMAND | `auth/index.ts:156` |
| POST | `/api/v2/auth/reset-password` | SENSITIVE_COMMAND | `auth/index.ts:191` |
| POST | `/api/v2/auth/refresh` | SENSITIVE_COMMAND | `auth/index.ts:230` |
| POST | `/api/v2/auth/logout` | - | `auth/index.ts:258` |
| GET | `/api/v2/auth/session` | AUTH_READ | `auth/index.ts:268` |
| POST | `/api/v2/auth/change-password` | SENSITIVE_COMMAND | `auth/index.ts:295` |
| GET | `/api/v2/auth/invitations/mine` | AUTH_READ | `auth/index.ts:373` |
| POST | `/api/v2/auth/otp/send` | OTP_SEND | `auth/otp-routes.ts:32` |
| POST | `/api/v2/auth/otp/verify` | OTP_VERIFY | `auth/otp-routes.ts:72` |
| POST | `/api/v2/checkout/quote` | STANDARD_COMMAND | `checkout/checkout-routes.ts:48` |
| POST | `/api/v2/checkout/holds` | STANDARD_COMMAND | `checkout/checkout-routes.ts:93` |
| GET | `/api/v2/payments/attempts/:holdId/status` | AUTH_READ | `checkout/payment-routes.ts:56` |
| POST | `/api/v2/payments/attempts` | STANDARD_COMMAND | `checkout/payment-routes.ts:85` |
| POST | `/api/v2/payments/:id/verify` | STANDARD_COMMAND | `checkout/payment-routes.ts:156` |
| POST | `/api/v2/webhooks/payments/razorpay` | - | `checkout/webhook-routes.ts:109` |
| POST | `/api/v2/door/heartbeat` | SCANNER_COMMAND | `door/door-ops-routes.ts:222` |
| POST | `/api/v2/door/check-ins/confirm` | SCANNER_COMMAND | `door/door-ops-routes.ts:260` |
| POST | `/api/v2/door/staff-deny` | SCANNER_COMMAND | `door/door-ops-routes.ts:320` |
| POST | `/api/v2/door/wallet-qr` | SCANNER_COMMAND | `door/door-ops-routes.ts:480` |
| POST | `/api/v2/door/wallet-charge` | SCANNER_COMMAND | `door/door-ops-routes.ts:519` |
| POST | `/api/v2/door/ticket-sale` | STANDARD_COMMAND | `door/door-ops-routes.ts:582` |
| POST | `/api/v2/door/sessions` | SENSITIVE_COMMAND | `door/scanner-routes.ts:74` |
| GET | `/api/v2/door/sessions/:sessionId` | AUTH_READ | `door/scanner-routes.ts:126` |
| POST | `/api/v2/door/check-ins` | SCANNER_COMMAND | `door/scanner-routes.ts:154` |
| POST | `/api/v2/door/check-ins/verify` | SCANNER_COMMAND | `door/scanner-routes.ts:211` |
| POST | `/api/v2/door/lookup` | SCANNER_COMMAND | `door/scanner-routes.ts:228` |
| GET | `/api/v2/door/check-ins/:checkInId` | AUTH_READ | `door/scanner-routes.ts:269` |
| GET | `/api/v2/door/offline-manifest` | STANDARD_COMMAND | `door/scanner-routes.ts:328` |
| POST | `/api/v2/door/offline-sync` | STANDARD_COMMAND | `door/scanner-routes.ts:362` |
| GET | `/api/v2/tickets/:ticketId/qr` | AUTH_READ | `door/scanner-routes.ts:413` |
| GET | `/api/v2/leaderboard` | PUBLIC_READ | `finance/leaderboard-routes.ts:50` |
| GET | `/api/v2/onboarding/me` | AUTH_READ | `onboarding.ts:49` |
| GET | `/api/v2/onboarding/applications` | AUTH_READ | `onboarding.ts:73` |
| POST | `/api/v2/onboarding/applications` | STANDARD_COMMAND | `onboarding.ts:105` |
| PATCH | `/api/v2/onboarding/applications/:requestId` | STANDARD_COMMAND | `onboarding.ts:132` |
| POST | `/api/v2/onboarding/applications/:requestId/documents/upload-url` | STANDARD_COMMAND | `onboarding.ts:169` |
| POST | `/api/v2/onboarding/applications/:requestId/documents` | STANDARD_COMMAND | `onboarding.ts:194` |
| POST | `/api/v2/onboarding/applications/:requestId/submit` | STANDARD_COMMAND | `onboarding.ts:225` |
| POST | `/api/v2/onboarding/verify-document` | SENSITIVE_COMMAND | `onboarding.ts:257` |
| GET | `/api/v2/orders` | AUTH_READ | `orders/orders-routes.ts:29` |
| GET | `/api/v2/orders/:id` | AUTH_READ | `orders/orders-routes.ts:59` |
| GET | `/api/v2/orders/:id/status` | AUTH_READ | `orders/orders-routes.ts:75` |
| POST | `/api/v2/events/:eventId/${action}` | STANDARD_COMMAND | `partner/events.ts:393` |
| POST | `/api/v2/organizations` | STANDARD_COMMAND | `partner/organizations.ts:146` |
| POST | `/api/v2/invitations/:invitationId/accept` | STANDARD_COMMAND | `partner/organizations.ts:655` |
| POST | `/api/v2/partnerships/:partnershipId/${action}` | STANDARD_COMMAND | `partner/partnerships.ts:228` |
| POST | `/api/v2/promoter-connections/:connectionId/${action}` | STANDARD_COMMAND | `partner/promoter-connections.ts:169` |
| GET | `/api/v2/venues/:venueId/profile` | - | `partner/venues.ts:297` |
| GET | `/api/v2/door/stats` | AUTH_READ | `phase5-routes.ts:75` |
| GET | `/api/v2/door/stats/stream` | AUTH_READ | `phase5-routes.ts:100` |
| GET | `/api/v2/profile/me` | AUTH_READ | `profile.ts:30` |
| PUT | `/api/v2/profile/me` | STANDARD_COMMAND | `profile.ts:52` |
| POST | `/api/v2/rsvp` | STANDARD_COMMAND | `rsvp/rsvp-routes.ts:38` |
| POST | `/api/v2/follows` | STANDARD_COMMAND | `social/social-routes.ts:52` |
| DELETE | `/api/v2/follows/:targetType/:targetId` | STANDARD_COMMAND | `social/social-routes.ts:81` |
| GET | `/api/v2/follows/me` | AUTH_READ | `social/social-routes.ts:103` |
| GET | `/api/v2/follows/:targetType/:targetId/status` | AUTH_READ | `social/social-routes.ts:136` |
| GET | `/api/v2/notifications/me` | AUTH_READ | `social/social-routes.ts:160` |
| GET | `/api/v2/notifications/me/unread-count` | AUTH_READ | `social/social-routes.ts:193` |
| POST | `/api/v2/notifications/me/read` | STANDARD_COMMAND | `social/social-routes.ts:211` |
| POST | `/api/v2/notifications/me/read-all` | STANDARD_COMMAND | `social/social-routes.ts:235` |
| POST | `/api/v2/support/tickets` | SENSITIVE_COMMAND | `support/intake-routes.ts:88` |
| GET | `/api/v2/support/tickets` | AUTH_READ | `support/intake-routes.ts:138` |
| GET | `/api/v2/support/tickets/:ticketId` | AUTH_READ | `support/intake-routes.ts:169` |
| POST | `/api/v2/support/tickets/:ticketId/messages` | SENSITIVE_COMMAND | `support/intake-routes.ts:194` |
| GET | `/api/v2/tickets/:id` | AUTH_READ | `tickets/ticket-routes.ts:26` |
| GET | `/api/v2/wallet` | AUTH_READ | `wallet/wallet-routes.ts:28` |
| GET | `/api/v2/wallet/tickets` | AUTH_READ | `wallet/wallet-routes.ts:50` |
| GET | `/api/v2/wallet/orders` | AUTH_READ | `wallet/wallet-routes.ts:80` |
