# C1RCLE API route list (api-gateway, `/api/v2`)

> **Update 2026-11-18:** added `POST /invitations/:id/resend`. Roles are now `owner`, `admin` (Ops lead), `manager` (Venue manager), `door_supervisor`, `gate_staff`, `member`. `/door/events`, `/door/guests`, `/door/sales` and `POST /door/devices` now need the narrow `door.floor` permission instead of `venue.read` / `door.manage`. The generated tables below were not regenerated; treat per-route access labels for those routes as superseded by this note.

> Generated from `origin/staging` @ `7c181a0` (2026-11-17). Method: the app was booted with the in-memory driver and every registered route was captured through Fastify `onRoute`; per-route guards were then read from each route file's `preHandler`. HEAD/OPTIONS auto-routes are omitted. **404 routes.**

Column legend: **RL** = rate-limit class (`PUBLIC_READ` 120/min, `AUTH_READ` 240/min, `STANDARD_COMMAND` 60/min, `SENSITIVE_COMMAND` 10/min, `OTP_SEND` 5/min, `OTP_VERIFY` 10/min, `SCANNER_COMMAND` 300/min). **Access** = `perm:` is an RBAC permission enforced in the route `preHandler` (`requirePermission`); `session` means a signed-in session is resolved globally and fine-grained authorization happens in the service layer (not visible at route level); `public` = no session required. **Validation** = Zod schemas from `@c1rcle/contracts` (B body, Q query, P params). Unlisted guard = none declared at route level.

## Cross-cutting

- Global hooks (`apps/api-gateway/src/app.ts`): request tracing, CORS allow-list (exact origins, credentials), unsafe-method CSRF origin check (403 on untrusted `Origin`/cross-site), body limit (1 MiB -> 413), error redaction, rate limit (memory or Redis store), auth-context plugin (Better Auth session cookie -> `request.actor`), forced password-rotation gate (allow-listed paths), RBAC (default deny), response cache classes, `validateV2` + `validateV2Response`.
- Command idempotency: mutating guest/partner commands take an `Idempotency-Key` header (`v2_idempotency_records`).
- Errors use the v2 error contract (`status`, `code`, `message`, `requestId`).
- Better Auth itself is mounted behind `/api/v2/auth/*` (routes below are the gateway's wrappers) and Google/native callbacks live in `routes/v2/auth/native-callbacks.ts`.

## Counts by area

| Area | Routes |
|---|---|
| Internal / health | 3 |
| Auth & sessions | 12 |
| Public discovery (unauthenticated) | 14 |
| Guest: checkout, payments, RSVP, orders | 12 |
| Guest: tickets, waitlist, wallet | 25 |
| Guest: profile, social, saved events, support intake | 29 |
| Partner dashboard (org-scoped) | 132 |
| Door, scanner & cover wallet | 49 |
| Finance, ledger & payouts | 17 |
| Platform admin console | 111 |
| **Total** | **404** |

## 01 Internal / health

### `internal/index.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| GET | `/internal/health` | - | public | - |  |
| GET | `/internal/readiness` | - | public | - |  |
| GET | `/internal/version` | - | public | - |  |

## 02 Auth & sessions

### `auth/index.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| POST | `/auth/change-password` | SENSITIVE_COMMAND | session | B:changePasswordSchema |  |
| POST | `/auth/forgot-password` | SENSITIVE_COMMAND | public | B:forgotPasswordRequestSchema |  |
| GET | `/auth/invitations/mine` | AUTH_READ | session | - |  |
| POST | `/auth/login` | SENSITIVE_COMMAND | public | B:loginRequestSchema |  |
| POST | `/auth/logout` | - | session | - |  |
| POST | `/auth/partner/signup` | SENSITIVE_COMMAND | public | B:signupRequestSchema | public; partner signup (role fixed server-side) |
| POST | `/auth/refresh` | SENSITIVE_COMMAND | public | - |  |
| POST | `/auth/reset-password` | SENSITIVE_COMMAND | public | B:resetPasswordRequestSchema |  |
| GET | `/auth/session` | AUTH_READ | session | - |  |
| POST | `/auth/signup` | SENSITIVE_COMMAND | public | B:signupRequestSchema | public; guest signup |

### `auth/otp-routes.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| POST | `/auth/otp/send` | OTP_SEND | public | B:otpSendRequestSchema |  |
| POST | `/auth/otp/verify` | OTP_VERIFY | public | B:otpVerifyRequestSchema |  |

## 03 Public discovery (unauthenticated)

### `guest-social.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| GET | `/public/events/:eventId/social-policy` | AUTH_READ | public | P:eventParams |  |

### `public/discovery.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| GET | `/public/discovery` | PUBLIC_READ | public | - |  |
| GET | `/public/events` | PUBLIC_READ | public | Q:paginationQuerySchema |  |
| GET | `/public/events/:idOrSlug` | PUBLIC_READ | public | P:idOrSlugParam |  |
| GET | `/public/events/:idOrSlug/ticket-addons` | PUBLIC_READ | public | P:idOrSlugParam |  |
| GET | `/public/events/:idOrSlug/tiers` | PUBLIC_READ | public | P:idOrSlugParam |  |
| GET | `/public/hosts/:slug` | PUBLIC_READ | public | P:slugParam |  |
| GET | `/public/hosts/by-id/:organizationId` | PUBLIC_READ | public | P:organizationIdParam |  |
| GET | `/public/promoter-links/:handle/:slug` | PUBLIC_READ | public | P:vanityParamsSchema |  |
| POST | `/public/promoter-links/click` | PUBLIC_READ | public | B:promoterClickBodySchema |  |
| GET | `/public/search` | PUBLIC_READ | public | Q:searchQuerySchema |  |
| POST | `/public/ticket-access-codes/redeem` | SENSITIVE_COMMAND | public | B:redeemTicketAccessCodeSchema |  |
| GET | `/public/venues/:slug` | PUBLIC_READ | public | P:slugParam |  |
| GET | `/public/venues/by-id/:venueId` | PUBLIC_READ | public | P:venueIdParam |  |

## 10 Guest: checkout, payments, RSVP, orders

### `checkout/checkout-routes.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| POST | `/checkout/holds` | STANDARD_COMMAND | session | B:checkoutHoldRequestSchema |  |
| POST | `/checkout/quote` | STANDARD_COMMAND | session | B:checkoutQuoteRequestSchema |  |

### `checkout/payment-routes.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| POST | `/payments/:id/verify` | STANDARD_COMMAND | session | P:paymentIdParam B:paymentConfirmRequestSchema |  |
| POST | `/payments/attempts` | STANDARD_COMMAND | session | B:paymentAttemptRequestSchema |  |
| POST | `/payments/attempts/:holdId/reconcile` | STANDARD_COMMAND | session | P:holdIdParam |  |
| GET | `/payments/attempts/:holdId/status` | AUTH_READ | session | P:holdIdParam |  |

### `checkout/webhook-routes.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| POST | `/webhooks/payments/razorpay` | - | provider signature | - |  |

### `orders/orders-routes.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| GET | `/orders` | AUTH_READ | session | Q:paginationQuerySchema |  |
| GET | `/orders/:id` | AUTH_READ | session | P:orderIdParam |  |
| GET | `/orders/:id/status` | AUTH_READ | session | P:orderIdParam |  |

### `rsvp/rsvp-routes.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| POST | `/rsvp` | STANDARD_COMMAND | session | B:rsvpRequestSchema |  |
| POST | `/rsvp/:orderId/cancel` | STANDARD_COMMAND | session | P:z.object |  |

## 11 Guest: tickets, waitlist, wallet

### `tickets/ticket-routes.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| GET | `/ticket-transfers/:transferId` | AUTH_READ | session | P:transferIdParam |  |
| POST | `/ticket-transfers/:transferId/accept` | STANDARD_COMMAND | session | - | guest transfer resolution |
| POST | `/ticket-transfers/:transferId/cancel` | STANDARD_COMMAND | session | - | guest transfer resolution |
| POST | `/ticket-transfers/:transferId/decline` | STANDARD_COMMAND | session | - | guest transfer resolution |
| POST | `/ticket-transfers/:transferId/resend` | STANDARD_COMMAND | session | P:transferIdParam |  |
| GET | `/tickets/:id` | AUTH_READ | session | P:ticketIdParam |  |
| POST | `/tickets/:id/event-changes/:eventVersion/respond` | STANDARD_COMMAND | session | P:eventChangeResponseParams B:eventChangeResponseRequestSchema |  |
| GET | `/tickets/:id/management` | AUTH_READ | session | P:ticketIdParam |  |
| POST | `/tickets/:id/refund-review` | STANDARD_COMMAND | session | P:ticketIdParam B:guestRefundReviewRequestSchema |  |
| GET | `/tickets/:id/transfers` | AUTH_READ | session | P:ticketIdParam |  |
| POST | `/tickets/:id/transfers` | STANDARD_COMMAND | session | P:ticketIdParam B:createTransferBody |  |

### `tickets/waitlist-routes.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| GET | `/admin/events/:eventId/tiers/:tierId/waitlist` | AUTH_READ | platform admin | P:offerParams Q:adminQueueQuery | 410 in V1 (unavailableInV1) |
| POST | `/admin/events/:eventId/tiers/:tierId/waitlist/offer-next` | SENSITIVE_COMMAND | platform admin | P:offerParams | 410 in V1 (unavailableInV1) |
| GET | `/events/:eventId/tiers/:tierId/waitlist` | AUTH_READ | perm: event.read | P:offerParams Q:adminQueueQuery | 410 in V1 (unavailableInV1) |
| POST | `/events/:eventId/tiers/:tierId/waitlist/offer-next` | SENSITIVE_COMMAND | perm: event.update | P:offerParams | 410 in V1 (unavailableInV1) |
| GET | `/waitlist` | AUTH_READ | session | Q:paginationQuerySchema | 410 in V1 (unavailableInV1) |
| POST | `/waitlist` | STANDARD_COMMAND | session | B:joinBody | 410 in V1 (unavailableInV1) |
| DELETE | `/waitlist/:entryId` | SENSITIVE_COMMAND | session | P:entryParams | 410 in V1 (unavailableInV1) |
| GET | `/waitlist/:entryId` | AUTH_READ | session | P:entryParams | 410 in V1 (unavailableInV1) |
| POST | `/waitlist/:entryId/accept` | SENSITIVE_COMMAND | session | P:entryParams | 410 in V1 (unavailableInV1) |

### `wallet/wallet-routes.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| GET | `/wallet` | AUTH_READ | session | - |  |
| GET | `/wallet/booking-tickets` | AUTH_READ | session | Q:paginationQuerySchema |  |
| GET | `/wallet/event-covers` | AUTH_READ | session | - |  |
| GET | `/wallet/orders` | AUTH_READ | session | Q:paginationQuerySchema |  |
| GET | `/wallet/tickets` | AUTH_READ | session | Q:paginationQuerySchema |  |

## 12 Guest: profile, social, saved events, support intake

### `guest-onboarding.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| GET | `/profile/onboarding` | AUTH_READ | session | - |  |
| PUT | `/profile/onboarding` | STANDARD_COMMAND | session | B:guestOnboardingPatchSchema |  |
| POST | `/profile/onboarding/phone` | OTP_VERIFY | session | B:guestPhoneProofSchema |  |

### `guest-social.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| GET | `/organizations/:organizationId/social-events` | AUTH_READ | perm: event.update | P:orgParams |  |
| PATCH | `/organizations/:organizationId/social-events/:eventId` | STANDARD_COMMAND | perm: event.update | P:orgParams.extend |  |
| PUT | `/social/blocks/:userId` | STANDARD_COMMAND | session | P:userParams B:socialBlockSchema |  |
| GET | `/social/events/:eventId/guestlist` | AUTH_READ | session | P:eventParams Q:socialListQuerySchema |  |
| PUT | `/social/events/:eventId/visibility` | STANDARD_COMMAND | session | P:eventParams B:socialHiddenEventSchema |  |
| GET | `/social/privacy` | AUTH_READ | session | B:guestSocialPrivacySchema |  |
| PUT | `/social/privacy` | STANDARD_COMMAND | session | B:guestSocialPrivacySchema |  |
| GET | `/social/profiles/:userId` | AUTH_READ | session | P:userParams |  |
| POST | `/social/reports/:userId` | SENSITIVE_COMMAND | session | P:userParams |  |

### `profile.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| GET | `/profile/me` | AUTH_READ | session | - |  |
| PUT | `/profile/me` | STANDARD_COMMAND | session | B:upsertGuestProfileSchema |  |

### `saved-events.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| GET | `/saved-events` | AUTH_READ | session | Q:savedEventListQuerySchema |  |
| GET | `/saved-events/:eventId` | AUTH_READ | session | - |  |
| PUT | `/saved-events/:eventId` | STANDARD_COMMAND | session | B:savedEventStateSchema |  |

### `social/social-routes.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| POST | `/follows` | STANDARD_COMMAND | session | B:createFollowSchema |  |
| DELETE | `/follows/:targetType/:targetId` | STANDARD_COMMAND | session | P:followTargetParamsSchema |  |
| GET | `/follows/:targetType/:targetId/status` | AUTH_READ | session | P:followTargetParamsSchema |  |
| GET | `/follows/me` | AUTH_READ | session | Q:listMyFollowsQuerySchema |  |
| GET | `/notifications/me` | AUTH_READ | session | Q:listNotificationsQuerySchema |  |
| POST | `/notifications/me/read` | STANDARD_COMMAND | session | B:markNotificationsReadSchema |  |
| POST | `/notifications/me/read-all` | STANDARD_COMMAND | session | - |  |
| GET | `/notifications/me/unread-count` | AUTH_READ | session | - |  |

### `support/intake-routes.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| GET | `/support/tickets` | AUTH_READ | session | Q:paginationQuerySchema |  |
| POST | `/support/tickets` | SENSITIVE_COMMAND | session | B:submitSupportTicketSchema |  |
| GET | `/support/tickets/:ticketId` | AUTH_READ | session | P:supportIdParam |  |
| POST | `/support/tickets/:ticketId/messages` | SENSITIVE_COMMAND | session | P:supportIdParam B:supportTicketMessageSchema |  |

## 20 Partner dashboard (org-scoped)

### `notifications/notifications-routes.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| GET | `/organizations/:organizationId/notifications` | AUTH_READ | perm: organization.read | P:organizationIdParam Q:paginationQuerySchema |  |
| POST | `/organizations/:organizationId/notifications/:notificationId/actions` | STANDARD_COMMAND | perm: organization.read | P:notificationIdParam B:notificationActionRequestSchema |  |
| PATCH | `/organizations/:organizationId/notifications/:notificationId/read` | STANDARD_COMMAND | perm: organization.read | P:notificationIdParam B:notificationReadRequestSchema |  |
| PATCH | `/organizations/:organizationId/notifications/read-all` | STANDARD_COMMAND | perm: organization.read | P:organizationIdParam |  |

### `onboarding.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| GET | `/onboarding/applications` | AUTH_READ | session | Q:paginationQuerySchema |  |
| POST | `/onboarding/applications` | STANDARD_COMMAND | session | B:startOnboardingSchema |  |
| PATCH | `/onboarding/applications/:requestId` | STANDARD_COMMAND | session | P:requestIdParam B:saveOnboardingProgressSchema |  |
| POST | `/onboarding/applications/:requestId/documents` | STANDARD_COMMAND | session | P:requestIdParam B:addOnboardingDocumentSchema |  |
| POST | `/onboarding/applications/:requestId/documents/upload-url` | STANDARD_COMMAND | session | P:requestIdParam B:documentUploadUrlRequestSchema |  |
| POST | `/onboarding/applications/:requestId/promoter-agreement/accept` | STANDARD_COMMAND | session | P:requestIdParam B:promoterAgreementAcceptanceRequestSchema.omit |  |
| POST | `/onboarding/applications/:requestId/submit` | STANDARD_COMMAND | session | P:requestIdParam |  |
| GET | `/onboarding/me` | AUTH_READ | session | - |  |
| GET | `/onboarding/promoter-agreement` | AUTH_READ | session | - |  |
| POST | `/onboarding/verify-document` | SENSITIVE_COMMAND | session | B:verifyDocumentSchema |  |
| POST | `/onboarding/verify-phone` | SENSITIVE_COMMAND | session | B:verifyPhoneSchema |  |

### `partner/analytics.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| GET | `/events/:eventId/analytics` | AUTH_READ | perm: event.read | P:eventIdParam | cache: EVENT_PREVIEW |
| GET | `/events/:eventId/analytics/booking-heatmap` | AUTH_READ | perm: event.read | P:eventIdParam | cache: EVENT_PREVIEW |
| GET | `/events/:eventId/promoter-analytics` | AUTH_READ | perm: event.read | P:eventIdParam |  |
| GET | `/organizations/:organizationId/analytics/calendar` | AUTH_READ | perm: organization.read | P:organizationIdParam Q:calendarQuery | cache: ORGANIZATION |
| GET | `/organizations/:organizationId/analytics/events` | AUTH_READ | perm: organization.read | P:organizationIdParam Q:cardQuery | cache: ORGANIZATION |
| GET | `/organizations/:organizationId/analytics/overview` | AUTH_READ | perm: organization.read | P:organizationIdParam | cache: ORGANIZATION |
| GET | `/organizations/:organizationId/analytics/sales-breakdown` | AUTH_READ | perm: organization.read | P:organizationIdParam | cache: ORGANIZATION |
| GET | `/organizations/:organizationId/analytics/trends` | AUTH_READ | perm: organization.read | P:organizationIdParam Q:trendsQuery | cache: ORGANIZATION |

### `partner/discover.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| GET | `/organizations/:organizationId/discover-partners` | AUTH_READ | perm: organization.read | P:organizationIdParam Q:discoverPartnersQuerySchema |  |

### `partner/event-catalog.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| GET | `/events/:eventId/access-codes` | AUTH_READ | perm: event.read | P:eventIdParam Q:paginationQuerySchema |  |
| POST | `/events/:eventId/access-codes` | STANDARD_COMMAND | perm: event.update | P:eventIdParam B:createTicketAccessCodeSchema |  |
| POST | `/events/:eventId/access-codes/:codeId/revoke` | STANDARD_COMMAND | perm: event.update | P:accessCodeIdParam B:z.object |  |
| POST | `/events/:eventId/access-invitations` | STANDARD_COMMAND | perm: event.update | P:eventIdParam B:createTicketAccessInvitationSchema |  |
| GET | `/events/:eventId/cover-policy` | AUTH_READ | perm: event.read | P:eventIdParam |  |
| POST | `/events/:eventId/cover-policy` | STANDARD_COMMAND | perm: event.update | P:eventIdParam B:proposeEventCoverPolicySchema |  |
| POST | `/events/:eventId/cover-policy/:proposalId/accept` | STANDARD_COMMAND | perm: event.update | P:eventIdParam.extend |  |
| GET | `/events/:eventId/promo-codes` | AUTH_READ | perm: event.read | P:eventIdParam Q:paginationQuerySchema |  |
| POST | `/events/:eventId/promo-codes` | STANDARD_COMMAND | perm: event.update | P:eventIdParam B:createPromoCodeSchema |  |
| GET | `/events/:eventId/promoter-assignments` | AUTH_READ | perm: event.read | P:eventIdParam |  |
| POST | `/events/:eventId/promoter-assignments` | STANDARD_COMMAND | perm: event.update | P:eventIdParam B:assignPromoterSchema |  |
| GET | `/events/:eventId/promoter-candidates` | AUTH_READ | perm: event.read | P:eventIdParam |  |
| GET | `/events/:eventId/table-packages` | AUTH_READ | perm: event.read | P:eventIdParam |  |
| POST | `/events/:eventId/table-packages` | STANDARD_COMMAND | perm: event.update | P:eventIdParam B:createTablePackageSchema |  |
| GET | `/events/:eventId/ticket-addons` | AUTH_READ | perm: event.read | P:eventIdParam |  |
| POST | `/events/:eventId/ticket-addons` | STANDARD_COMMAND | perm: event.update | P:eventIdParam B:createTicketAddonSchema |  |
| GET | `/events/:eventId/ticket-tiers` | AUTH_READ | perm: event.read | P:eventIdParam |  |
| POST | `/events/:eventId/ticket-tiers` | STANDARD_COMMAND | perm: event.update | P:eventIdParam B:createTicketTierSchema |  |
| PATCH | `/events/:eventId/ticket-tiers/:tierId` | STANDARD_COMMAND | perm: event.update | P:ticketTierIdParam |  |
| POST | `/promoter-assignments/:assignmentId/end` | STANDARD_COMMAND | perm: event.update | P:assignmentIdParam |  |
| POST | `/promoter-assignments/:assignmentId/respond` | STANDARD_COMMAND | perm: event.read | P:assignmentIdParam B:promoterAssignmentActionSchema |  |
| POST | `/promoter-assignments/:assignmentId/revoke` | STANDARD_COMMAND | perm: event.update | P:assignmentIdParam B:promoterAssignmentRevokeSchema |  |
| GET | `/promoter-candidates` | AUTH_READ | perm: event.create | - |  |

### `partner/event-comparison.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| GET | `/events/:eventId/comparison` | AUTH_READ | perm: event.read | - |  |

### `partner/events.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| GET | `/events/:eventId` | AUTH_READ | perm: event.read | P:eventIdParam |  |
| PATCH | `/events/:eventId` | STANDARD_COMMAND | perm: event.update | P:eventIdParam B:updateEventSchema |  |
| POST | `/events/:eventId/cancel` | STANDARD_COMMAND | perm: event.cancel | P:eventIdParam B:cancelEventSchema |  |
| POST | `/events/:eventId/duplicate` | STANDARD_COMMAND | perm: event.create | - | idempotent lifecycle action (Idempotency-Key) |
| GET | `/events/:eventId/external-location/contact` | AUTH_READ | perm: event.read | P:eventIdParam |  |
| POST | `/events/:eventId/external-location/resubmit` | SENSITIVE_COMMAND | perm: event.update | P:eventIdParam B:externalEventLocationInputSchema |  |
| POST | `/events/:eventId/lifecycle` | STANDARD_COMMAND | perm: event.update | P:eventIdParam B:lifecycleBody |  |
| POST | `/events/:eventId/pause-sales` | STANDARD_COMMAND | perm: event.update | - | idempotent lifecycle action (Idempotency-Key) |
| GET | `/events/:eventId/previews` | AUTH_READ | perm: event.read | P:eventIdParam |  |
| POST | `/events/:eventId/publish` | STANDARD_COMMAND | perm: event.publish | - | idempotent lifecycle action (Idempotency-Key) |
| POST | `/events/:eventId/resume-sales` | STANDARD_COMMAND | perm: event.update | - | idempotent lifecycle action (Idempotency-Key) |
| POST | `/events/:eventId/review` | STANDARD_COMMAND | perm: event.update | - | idempotent lifecycle action (Idempotency-Key) |
| DELETE | `/organizations/:organizationId/event-draft` | STANDARD_COMMAND | perm: event.create | P:orgIdParam |  |
| GET | `/organizations/:organizationId/event-draft` | AUTH_READ | perm: event.create | P:orgIdParam |  |
| PUT | `/organizations/:organizationId/event-draft` | STANDARD_COMMAND | perm: event.create | P:orgIdParam B:eventDraftRequestSchema |  |
| GET | `/organizations/:organizationId/events` | AUTH_READ | perm: event.read | P:orgIdParam Q:paginationQuerySchema |  |
| POST | `/organizations/:organizationId/events` | STANDARD_COMMAND | perm: event.create | P:orgIdParam B:createEventBody |  |
| POST | `/organizations/:organizationId/poster/upload-url` | STANDARD_COMMAND | perm: event.create | P:orgIdParam B:posterUploadUrlRequestSchema |  |
| GET | `/organizations/:organizationId/venue-events` | AUTH_READ | perm: venue.read | P:orgIdParam Q:paginationQuerySchema |  |
| GET | `/organizations/:organizationId/venue-events/:eventId` | AUTH_READ | perm: venue.read | P:orgIdParam.extend |  |
| GET | `/promoters/:promoterId/assignment-invitations` | AUTH_READ | perm: event.read | P:z.object |  |
| GET | `/promoters/:promoterId/events` | AUTH_READ | perm: event.read | P:z.object |  |

### `partner/marketing.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| GET | `/organizations/:organizationId/analytics/ticket-tiers` | AUTH_READ | perm: organization.read | P:z.object |  |
| GET | `/organizations/:organizationId/marketing/audience` | AUTH_READ | perm: finance.orders.read | P:z.object |  |

### `partner/organizations.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| POST | `/invitations/:invitationId/accept` | STANDARD_COMMAND | session | P:invitationIdParam |  |
| POST | `/invitations/:invitationId/revoke` | STANDARD_COMMAND | perm: staff.manage | P:invitationIdParam |  |
| POST | `/invitations/:invitationId/resend` | SENSITIVE_COMMAND | perm: staff.manage | P:invitationIdParam | new 2026-11-18: re-emails a pending invite; reissues a 2-day temporary password only if the invitee never chose their own; only for roles below the caller |
| GET | `/organizations` | AUTH_READ | perm: organization.read | Q:paginationQuerySchema |  |
| POST | `/organizations` | STANDARD_COMMAND | session | B:createOrganizationBody |  |
| GET | `/organizations/:organizationId` | AUTH_READ | perm: organization.read | P:organizationIdParam | cache: ORGANIZATION |
| PATCH | `/organizations/:organizationId` | STANDARD_COMMAND | perm: organization.update | P:organizationIdParam |  |
| GET | `/organizations/:organizationId/access` | AUTH_READ | perm: organization.read | P:organizationIdParam |  |
| GET | `/organizations/:organizationId/invitations` | AUTH_READ | perm: staff.manage | P:organizationIdParam Q:paginationQuerySchema |  |
| POST | `/organizations/:organizationId/invitations` | STANDARD_COMMAND | perm: staff.manage | P:organizationIdParam |  |
| GET | `/organizations/:organizationId/members` | AUTH_READ | perm: organization.read | P:organizationIdParam Q:paginationQuerySchema |  |
| POST | `/organizations/:organizationId/members` | STANDARD_COMMAND | perm: staff.manage | P:organizationIdParam B:inviteMemberSchema |  |
| DELETE | `/organizations/:organizationId/members/:userId` | STANDARD_COMMAND | perm: staff.manage | P:memberIdParam |  |
| PATCH | `/organizations/:organizationId/members/:userId` | STANDARD_COMMAND | perm: staff.manage | P:memberIdParam B:updateMemberRoleSchema |  |

### `partner/partnerships.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| GET | `/organizations/:organizationId/partnerships` | AUTH_READ | perm: organization.read | P:organizationIdParam Q:paginationQuerySchema |  |
| POST | `/partnerships` | STANDARD_COMMAND | perm: venue.manage | B:requestPartnershipSchema |  |
| POST | `/partnerships/:partnershipId/approve` | STANDARD_COMMAND | perm: venue.manage | - | resolution action |
| POST | `/partnerships/:partnershipId/block` | STANDARD_COMMAND | perm: venue.manage | - | resolution action |
| POST | `/partnerships/:partnershipId/end` | STANDARD_COMMAND | perm: venue.manage | - | resolution action |
| POST | `/partnerships/:partnershipId/reject` | STANDARD_COMMAND | perm: venue.manage | - | resolution action |
| POST | `/partnerships/:partnershipId/venue-share` | STANDARD_COMMAND | perm: venue.manage | P:partnershipIdParam B:setVenueShareRequestSchema |  |

### `partner/profiles.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| GET | `/organizations/:organizationId/partner-profiles/:partnerId` | AUTH_READ | perm: organization.read | Q:query |  |

### `partner/promoter-connections.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| GET | `/organizations/:organizationId/promoter-connections` | AUTH_READ | perm: organization.read | P:organizationIdParam Q:paginationQuerySchema |  |
| POST | `/promoter-connections` | STANDARD_COMMAND | perm: venue.manage | B:requestConnectionSchema |  |
| POST | `/promoter-connections/:connectionId/approve` | STANDARD_COMMAND | perm: venue.manage (default; per-action option) | - | resolution action |
| POST | `/promoter-connections/:connectionId/block` | STANDARD_COMMAND | perm: venue.manage (default; per-action option) | - | resolution action |
| POST | `/promoter-connections/:connectionId/reject` | STANDARD_COMMAND | perm: venue.manage (default; per-action option) | - | resolution action |
| POST | `/promoter-connections/:connectionId/revoke` | STANDARD_COMMAND | perm: venue.manage (default; per-action option) | - | resolution action |

### `partner/referral-links.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| GET | `/events/:eventId/promoters/:promoterId/attributed-orders` | AUTH_READ | perm: event.read | P:eventPromoterParam Q:eventAttributedOrdersQuerySchema |  |
| GET | `/events/:eventId/referral-links` | AUTH_READ | perm: event.read | P:eventIdParam Q:paginationQuerySchema |  |
| POST | `/events/:eventId/referral-links` | STANDARD_COMMAND | perm: event.update | P:eventIdParam B:createReferralLinkSchema |  |
| POST | `/promoter-assignments/:assignmentId/referral-links` | STANDARD_COMMAND | perm: event.read | P:assignmentIdParam B:promoterCreateLinkSchema |  |
| GET | `/promoters/:promoterId/attributed-orders` | AUTH_READ | perm: event.read | P:promoterIdParam Q:paginationQuerySchema |  |
| GET | `/promoters/:promoterId/campaign-summary` | AUTH_READ | perm: event.read | P:promoterIdParam |  |
| GET | `/promoters/:promoterId/referral-links` | AUTH_READ | perm: event.read | P:promoterIdParam Q:referralLinkPaginationQuerySchema |  |
| POST | `/referral-links/:referralLinkId/deactivate` | STANDARD_COMMAND | perm: event.update | P:linkIdParam |  |

### `partner/search.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| GET | `/organizations/:organizationId/search` | AUTH_READ | perm: organization.read | Q:partnerGlobalSearchQuerySchema |  |

### `partner/settings.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| POST | `/organizations/:organizationId/partner-profile-media/upload-url` | STANDARD_COMMAND | perm: partner.profile.update | P:organizationParams B:uploadRequest |  |
| GET | `/organizations/:organizationId/partner-settings` | AUTH_READ | perm: organization.read | P:organizationParams |  |
| PATCH | `/organizations/:organizationId/partner-settings` | STANDARD_COMMAND | perm: partner.profile.update | P:organizationParams B:updatePartnerSettingsSchema |  |

### `partner/venues.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| GET | `/organizations/:organizationId/slot-requests` | AUTH_READ | perm: slot-request.list | P:orgIdParam Q:paginationQuerySchema |  |
| GET | `/organizations/:organizationId/venues` | AUTH_READ | perm: venue.read | P:orgIdParam Q:paginationQuerySchema |  |
| POST | `/organizations/:organizationId/venues` | STANDARD_COMMAND | perm: venue.create | P:orgIdParam B:createVenueBody |  |
| GET | `/venues/:venueId` | AUTH_READ | perm: venue.read | P:venueIdParam |  |
| PATCH | `/venues/:venueId` | STANDARD_COMMAND | perm: venue.manage | P:venueIdParam B:updateVenueBody |  |
| GET | `/venues/:venueId/availability` | AUTH_READ | perm: venue.read | P:venueIdParam Q:calendarQuery |  |
| GET | `/venues/:venueId/calendar` | AUTH_READ | perm: venue.read | P:venueIdParam Q:calendarQuery |  |
| POST | `/venues/:venueId/calendar/availability` | STANDARD_COMMAND | perm: venue.manage | P:venueIdParam B:publishVenueAvailabilitySchema |  |
| POST | `/venues/:venueId/calendar/blocks` | STANDARD_COMMAND | perm: venue.manage | P:venueIdParam B:createVenueBlockSchema |  |
| DELETE | `/venues/:venueId/calendar/blocks/:blockId` | STANDARD_COMMAND | perm: venue.manage | P:venueBlockParam |  |
| GET | `/venues/:venueId/menu` | AUTH_READ | perm: venue.read | P:venueIdParam | cache: VENUE_PROFILE |
| PUT | `/venues/:venueId/menu` | STANDARD_COMMAND | perm: venue.manage | P:venueIdParam B:updateVenueMenuSchema |  |
| GET | `/venues/:venueId/profile` | - | session | P:venueIdParam |  |
| PATCH | `/venues/:venueId/profile` | STANDARD_COMMAND | perm: venue.manage | P:venueIdParam B:updateVenueBody |  |
| GET | `/venues/:venueId/slot-requests` | AUTH_READ | perm: venue.read | P:venueIdParam Q:paginationQuerySchema |  |
| POST | `/venues/:venueId/slot-requests` | STANDARD_COMMAND | perm: slot-request.create | P:venueIdParam B:createSlotRequestSchema |  |
| GET | `/venues/:venueId/slot-requests/:slotRequestId` | AUTH_READ | perm: venue.read | P:slotRequestDetailParam |  |
| POST | `/venues/slot-requests/:slotRequestId/accept` | STANDARD_COMMAND | perm: venue.schedule | - | resolve slot request |
| POST | `/venues/slot-requests/:slotRequestId/cancel` | STANDARD_COMMAND | perm: slot-request.create | P:slotRequestIdParam |  |
| POST | `/venues/slot-requests/:slotRequestId/cancel-by-venue` | STANDARD_COMMAND | perm: venue.schedule | P:slotRequestIdParam |  |
| POST | `/venues/slot-requests/:slotRequestId/reject` | STANDARD_COMMAND | perm: venue.schedule | - | resolve slot request |

## 40 Door, scanner & cover wallet

### `door/cover-wallet-routes.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| POST | `/cover-wallets` | STANDARD_COMMAND | perm: door.manage | B:coverWalletIssueRequestSchema |  |
| GET | `/cover-wallets/:walletId` | AUTH_READ | perm: venue.read | P:walletIdParam |  |
| POST | `/cover-wallets/:walletId/credit` | STANDARD_COMMAND | perm: door.manage | P:walletIdParam B:coverWalletCreditRequestSchema |  |
| POST | `/cover-wallets/:walletId/debit` | STANDARD_COMMAND | perm: ticket.override | P:walletIdParam B:coverWalletDebitBody |  |
| POST | `/cover-wallets/:walletId/freeze` | STANDARD_COMMAND | perm: door.manage | P:walletIdParam |  |
| GET | `/cover-wallets/:walletId/qr` | AUTH_READ | perm: venue.read | P:walletIdParam |  |
| POST | `/cover-wallets/:walletId/reconcile` | STANDARD_COMMAND | perm: door.manage | P:walletIdParam B:coverWalletReconciliationRequestSchema |  |
| POST | `/cover-wallets/:walletId/terminate` | STANDARD_COMMAND | perm: door.manage | P:walletIdParam B:coverWalletTerminateBody |  |
| POST | `/cover-wallets/:walletId/unfreeze` | STANDARD_COMMAND | perm: door.manage | P:walletIdParam |  |

### `door/door-ops-routes.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| GET | `/door/attendance-report` | AUTH_READ | perm: venue.read | Q:attendanceReportQuerySchema |  |
| POST | `/door/check-ins/confirm` | SCANNER_COMMAND | scanner/door session | B:confirmCoupleBodySchema |  |
| GET | `/door/devices` | AUTH_READ | perm: door.manage | Q:paginationQuerySchema |  |
| POST | `/door/devices` | STANDARD_COMMAND | perm: door.manage | B:scannerDeviceBindBodySchema |  |
| POST | `/door/devices/:deviceId/reauthorize` | SENSITIVE_COMMAND | perm: door.manage | P:deviceIdParam B:scannerDeviceReauthorizeBodySchema |  |
| POST | `/door/devices/:deviceId/unbind` | SENSITIVE_COMMAND | perm: door.manage | P:deviceIdParam B:revokeReasonBodySchema |  |
| GET | `/door/events` | AUTH_READ | perm: venue.read | Q:doorEventListQuerySchema |  |
| GET | `/door/guests` | AUTH_READ | perm: venue.read | Q:doorGuestListQuerySchema |  |
| POST | `/door/guests/check-in` | STANDARD_COMMAND | perm: ticket.override | B:manualCheckInBodySchema |  |
| POST | `/door/heartbeat` | SCANNER_COMMAND | scanner/door session | B:scannerHeartbeatBodySchema |  |
| POST | `/door/staff-deny` | SCANNER_COMMAND | scanner/door session | B:staffDenyBodySchema |  |
| POST | `/door/ticket-sale` | STANDARD_COMMAND | scanner/door session | B:doorTicketSaleBodySchema |  |
| POST | `/door/wallet-charge` | SCANNER_COMMAND | scanner/door session | B:walletChargeBodySchema |  |
| POST | `/door/wallet-qr` | SCANNER_COMMAND | scanner/door session | B:walletQrResolveBodySchema |  |

### `door/door-sale-routes.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| POST | `/door/dine-in` | STANDARD_COMMAND | perm: ticket.override | B:doorDineInRequestSchema |  |
| GET | `/door/sales` | AUTH_READ | perm: venue.read | Q:doorSalesQuerySchema |  |
| POST | `/door/walk-in` | STANDARD_COMMAND | perm: ticket.override | B:doorWalkInRequestSchema |  |

### `door/event-code-routes.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| POST | `/door-codes/:codeId/revoke` | SENSITIVE_COMMAND | perm: door.manage | P:codeIdParam B:revokeReasonBodySchema |  |
| GET | `/door-codes/:codeId/sessions` | AUTH_READ | perm: door.manage | P:codeIdParam Q:paginationQuerySchema |  |
| POST | `/door/sessions/:sessionId/revoke` | SENSITIVE_COMMAND | perm: door.manage | P:sessionIdParam B:revokeReasonBodySchema |  |
| GET | `/events/:eventId/door-codes` | AUTH_READ | perm: door.manage | P:eventIdParam |  |
| POST | `/events/:eventId/door-codes` | STANDARD_COMMAND | perm: door.manage | P:eventIdParam B:eventCodeCreateBodySchema |  |

### `door/scanner-routes.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| POST | `/door/check-ins` | SCANNER_COMMAND | scanner/door session | B:scanRequestSchema |  |
| GET | `/door/check-ins/:checkInId` | AUTH_READ | scanner/door session | P:checkInIdParam |  |
| POST | `/door/check-ins/verify` | SCANNER_COMMAND | scanner/door session | B:scanRequestSchema |  |
| POST | `/door/lookup` | SCANNER_COMMAND | scanner/door session | B:scanRequestSchema |  |
| GET | `/door/offline-manifest` | STANDARD_COMMAND | scanner/door session | Q:offlineManifestRequestSchema |  |
| POST | `/door/offline-sync` | STANDARD_COMMAND | scanner/door session | B:offlineSyncRequestSchema |  |
| POST | `/door/override` | SENSITIVE_COMMAND | perm: ticket.override | B:overrideRequestSchema |  |
| POST | `/door/sessions` | SENSITIVE_COMMAND | scanner/door session | B:scannerSessionCreateBodySchema |  |
| GET | `/door/sessions/:sessionId` | AUTH_READ | scanner/door session | P:sessionIdParam |  |
| POST | `/door/ticket-addons/redeem` | SCANNER_COMMAND | scanner/door session | B:ticketAddonRedemptionRequestSchema |  |
| GET | `/tickets/:ticketId/qr` | AUTH_READ | scanner/door session | P:ticketIdParam |  |

### `event-cover/event-cover-redemption-routes.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| POST | `/event-cover/redemption-challenges` | STANDARD_COMMAND | perm: door.manage | B:createEventCoverRedemptionChallengeSchema |  |
| GET | `/event-cover/redemption-challenges/:challengeId` | AUTH_READ | perm: door.manage | P:challengeParams |  |
| POST | `/event-cover/redemption-challenges/:challengeId/consume` | STANDARD_COMMAND | perm: door.manage | P:challengeParams B:z.object |  |
| GET | `/wallet/event-cover-redemption-challenges/:challengeId` | AUTH_READ | session | P:challengeParams |  |
| POST | `/wallet/event-cover-redemption-challenges/:challengeId/confirm` | STANDARD_COMMAND | session | P:challengeParams B:z.object |  |

### `phase5-routes.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| GET | `/door/stats` | AUTH_READ | scanner/door session | Q:doorStatsQuerySchema |  |
| GET | `/door/stats/stream` | AUTH_READ | scanner/door session | Q:doorStatsQuerySchema |  |

## 50 Finance, ledger & payouts

### `finance/finance-routes.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| GET | `/organizations/:organizationId/bank-accounts` | AUTH_READ | perm: finance.read | P:organizationIdParam |  |
| POST | `/organizations/:organizationId/bank-accounts` | STANDARD_COMMAND | perm: finance.bank_accounts.manage | P:organizationIdParam B:bankAccountRequestSchema |  |
| DELETE | `/organizations/:organizationId/bank-accounts/:bankAccountId` | STANDARD_COMMAND | perm: finance.bank_accounts.manage | P:bankAccountIdParam |  |
| POST | `/organizations/:organizationId/bank-accounts/:bankAccountId/default` | STANDARD_COMMAND | perm: finance.bank_accounts.manage | P:bankAccountIdParam |  |
| GET | `/organizations/:organizationId/disputes` | AUTH_READ | perm: organization.read | P:organizationIdParam Q:paginationQuerySchema |  |
| POST | `/organizations/:organizationId/disputes` | STANDARD_COMMAND | perm: organization.update | P:organizationIdParam B:raiseDisputeRequestSchema |  |
| GET | `/organizations/:organizationId/disputes/:disputeId` | AUTH_READ | perm: organization.read | P:disputeIdParam |  |
| POST | `/organizations/:organizationId/disputes/:disputeId/resolve` | STANDARD_COMMAND | perm: organization.update | P:disputeIdParam B:resolveDisputeRequestSchema |  |
| POST | `/organizations/:organizationId/disputes/:disputeId/review` | STANDARD_COMMAND | perm: organization.update | P:disputeIdParam |  |
| GET | `/organizations/:organizationId/finance/balance` | AUTH_READ | perm: finance.read | P:organizationIdParam |  |
| GET | `/organizations/:organizationId/finance/ledger` | AUTH_READ | perm: finance.read | P:organizationIdParam Q:paginationQuerySchema |  |
| GET | `/organizations/:organizationId/finance/orders` | AUTH_READ | perm: finance.orders.read | P:organizationIdParam Q:paginationQuerySchema |  |
| GET | `/organizations/:organizationId/payouts` | AUTH_READ | perm: finance.payouts.read | P:organizationIdParam Q:paginationQuerySchema |  |
| POST | `/organizations/:organizationId/payouts` | STANDARD_COMMAND | perm: finance.payouts.manage | P:organizationIdParam B:payoutRequestSchema |  |
| GET | `/organizations/:organizationId/payouts/:payoutId` | AUTH_READ | perm: finance.payouts.read | P:payoutIdParam |  |

### `finance/leaderboard-routes.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| GET | `/leaderboard` | PUBLIC_READ | session | Q:leaderboardQuerySchema |  |
| GET | `/organizations/:organizationId/leaderboard/me` | AUTH_READ | perm: organization.read | P:organizationIdParam Q:leaderboardQuerySchema |  |

## 90 Platform admin console

### `admin/analytics.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| GET | `/admin/analytics` | AUTH_READ | platform admin | - |  |

### `admin/communications.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| GET | `/admin/communications/email-deliveries/:aggregateId` | AUTH_READ | platform admin | P:paramsSchema |  |

### `admin/complimentary-allocations.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| POST | `/admin/complimentary-allocations/:allocationId/claims` | SENSITIVE_COMMAND | platform admin | P:allocationParams B:issueBody |  |
| POST | `/admin/complimentary-claims/:claimId/revoke` | SENSITIVE_COMMAND | platform admin | P:claimParams B:revokeBody |  |
| GET | `/admin/events/:eventId/complimentary-allocations` | AUTH_READ | platform admin | P:eventParams Q:paginationQuerySchema |  |
| POST | `/admin/events/:eventId/complimentary-allocations` | SENSITIVE_COMMAND | platform admin | P:eventParams B:createBody |  |

### `admin/directory.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| GET | `/admin/audit/export.csv` | AUTH_READ | platform admin | - |  |
| GET | `/admin/events` | AUTH_READ | platform admin | Q:directoryQuerySchema |  |
| GET | `/admin/events/export.csv` | AUTH_READ | platform admin | - |  |
| GET | `/admin/hosts` | AUTH_READ | platform admin | Q:directoryQuerySchema |  |
| GET | `/admin/hosts/export.csv` | AUTH_READ | platform admin | - |  |
| GET | `/admin/lookup` | AUTH_READ | platform admin | Q:lookupQuerySchema |  |
| GET | `/admin/users` | AUTH_READ | platform admin | Q:directoryQuerySchema |  |
| GET | `/admin/users/export.csv` | AUTH_READ | platform admin | - |  |
| GET | `/admin/venues` | AUTH_READ | platform admin | Q:directoryQuerySchema |  |
| GET | `/admin/venues/export.csv` | AUTH_READ | platform admin | - |  |

### `admin/disputes.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| GET | `/admin/disputes` | AUTH_READ | platform admin | Q:disputeQuerySchema |  |
| GET | `/admin/disputes/:disputeId` | AUTH_READ | platform admin | P:disputeIdParam |  |
| POST | `/admin/disputes/:disputeId/resolve` | SENSITIVE_COMMAND | platform admin | P:disputeIdParam B:adminResolveDisputeSchema |  |

### `admin/event-actions.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| POST | `/admin/events/:eventId/external-venue-review` | SENSITIVE_COMMAND | platform admin | P:eventIdParam B:externalVenueReviewDecisionSchema |  |
| POST | `/admin/events/:eventId/force-complete` | SENSITIVE_COMMAND | platform admin | P:eventIdParam |  |
| POST | `/admin/events/:eventId/pause` | SENSITIVE_COMMAND | platform admin | P:eventIdParam |  |
| POST | `/admin/events/:eventId/resume` | SENSITIVE_COMMAND | platform admin | P:eventIdParam |  |
| GET | `/admin/external-venue-requests` | AUTH_READ | platform admin | - |  |

### `admin/onboarding-review.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| GET | `/admin/admins` | AUTH_READ | platform admin | Q:paginationQuerySchema |  |
| POST | `/admin/admins/:adminId/revoke` | SENSITIVE_COMMAND | platform admin | P:adminIdParam |  |
| GET | `/admin/audit` | AUTH_READ | platform admin | Q:auditQuerySchema |  |
| GET | `/admin/onboarding/applications` | AUTH_READ | platform admin | Q:queueQuerySchema |  |
| GET | `/admin/onboarding/applications/:requestId` | AUTH_READ | platform admin | P:requestIdParam |  |
| POST | `/admin/onboarding/applications/:requestId/approve` | SENSITIVE_COMMAND | platform admin | P:requestIdParam B:reviewOnboardingSchema.optional |  |
| GET | `/admin/onboarding/applications/:requestId/documents/:label/read-url` | AUTH_READ | platform admin | P:requestDocumentParam |  |
| POST | `/admin/onboarding/applications/:requestId/documents/:label/reject` | SENSITIVE_COMMAND | platform admin | P:requestDocumentParam B:rejectKycDocumentSchema |  |
| POST | `/admin/onboarding/applications/:requestId/documents/:label/verify` | SENSITIVE_COMMAND | platform admin | P:requestDocumentParam |  |
| POST | `/admin/onboarding/applications/:requestId/reject` | SENSITIVE_COMMAND | platform admin | - | platform-admin review action |
| POST | `/admin/onboarding/applications/:requestId/request-changes` | SENSITIVE_COMMAND | platform admin | - | platform-admin review action |
| GET | `/admin/proposals` | AUTH_READ | platform admin | Q:proposalQuerySchema |  |
| POST | `/admin/proposals` | SENSITIVE_COMMAND | platform admin | B:proposeActionSchema |  |
| POST | `/admin/proposals/:proposalId/approve` | SENSITIVE_COMMAND | platform admin | - | platform-admin proposed-action resolution |
| POST | `/admin/proposals/:proposalId/cancel` | SENSITIVE_COMMAND | platform admin | - | platform-admin proposed-action resolution |
| POST | `/admin/proposals/:proposalId/provision-admin` | SENSITIVE_COMMAND | platform admin | P:proposalIdParam |  |
| POST | `/admin/proposals/:proposalId/reject` | SENSITIVE_COMMAND | platform admin | - | platform-admin proposed-action resolution |
| POST | `/admin/proposals/:proposalId/update-admin-role` | SENSITIVE_COMMAND | platform admin | P:proposalIdParam |  |

### `admin/orders.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| GET | `/admin/orders` | AUTH_READ | platform admin | Q:ordersQuerySchema |  |
| GET | `/admin/orders/by-booking-reference/:bookingReference` | AUTH_READ | platform admin | P:z.object |  |
| GET | `/admin/orders/export.csv` | AUTH_READ | platform admin | - |  |

### `admin/organization-actions.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| POST | `/admin/organizations/:organizationId/reinstate` | SENSITIVE_COMMAND | platform admin | P:organizationIdParam |  |
| POST | `/admin/organizations/:organizationId/suspend` | SENSITIVE_COMMAND | platform admin | P:organizationIdParam |  |
| POST | `/admin/proposals/:proposalId/adjust-commission` | SENSITIVE_COMMAND | platform admin | P:proposalIdParam |  |

### `admin/payouts.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| GET | `/admin/payouts` | AUTH_READ | platform admin | Q:payoutQuerySchema |  |
| GET | `/admin/payouts/:payoutId` | AUTH_READ | platform admin | P:payoutIdParam |  |
| POST | `/admin/payouts/:payoutId/complete` | SENSITIVE_COMMAND | platform admin | P:payoutIdParam B:completeManualPayoutSchema |  |
| POST | `/admin/payouts/batch-run` | SENSITIVE_COMMAND | platform admin | B:runPayoutBatchSchema |  |
| POST | `/admin/proposals/:proposalId/freeze-payout` | SENSITIVE_COMMAND | platform admin | P:proposalIdParam |  |
| POST | `/admin/proposals/:proposalId/release-payout` | SENSITIVE_COMMAND | platform admin | P:proposalIdParam |  |

### `admin/promoters.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| POST | `/admin/promoter-links/:linkId/deactivate` | SENSITIVE_COMMAND | platform admin | P:promoterLinkIdParam B:adminPromoterSuspendRequestSchema |  |
| GET | `/admin/promoters` | AUTH_READ | platform admin | Q:promotersQuerySchema |  |
| GET | `/admin/promoters/:promoterId/referral-links` | AUTH_READ | platform admin | P:promoterIdParam Q:promotersQuerySchema |  |
| POST | `/admin/promoters/:promoterId/reinstate` | SENSITIVE_COMMAND | platform admin | P:promoterIdParam B:adminPromoterSuspendRequestSchema |  |
| GET | `/admin/promoters/:promoterId/risk-flags` | AUTH_READ | platform admin | P:promoterIdParam |  |
| POST | `/admin/promoters/:promoterId/suspend` | SENSITIVE_COMMAND | platform admin | P:promoterIdParam B:adminPromoterSuspendRequestSchema |  |
| GET | `/admin/promoters/export.csv` | AUTH_READ | platform admin | - |  |

### `admin/promotions.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| GET | `/admin/promotions` | AUTH_READ | platform admin | Q:promotionsQuerySchema |  |
| GET | `/admin/promotions/export.csv` | AUTH_READ | platform admin | - |  |

### `admin/reconciliation.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| GET | `/admin/chargebacks` | AUTH_READ | platform admin | Q:paginationQuerySchema |  |
| POST | `/admin/proposals/:proposalId/post-financial-adjustment` | SENSITIVE_COMMAND | platform admin | P:z.object |  |
| GET | `/admin/reconciliation/discrepancies` | AUTH_READ | platform admin | Q:discrepancyQuery |  |
| GET | `/admin/reconciliation/discrepancies/:discrepancyId` | AUTH_READ | platform admin | P:discrepancyParam |  |
| POST | `/admin/reconciliation/discrepancies/:discrepancyId/resolve` | SENSITIVE_COMMAND | platform admin | P:discrepancyParam B:resolveReconciliationDiscrepancySchema |  |
| POST | `/admin/reconciliation/discrepancies/:discrepancyId/retry` | SENSITIVE_COMMAND | platform admin | P:discrepancyParam |  |
| GET | `/admin/reconciliation/export.csv` | - | platform admin | - |  |
| GET | `/admin/reconciliation/runs` | AUTH_READ | platform admin | Q:paginationQuerySchema |  |
| POST | `/admin/reconciliation/runs` | SENSITIVE_COMMAND | platform admin | - |  |
| GET | `/admin/reconciliation/runs/:runId` | AUTH_READ | platform admin | P:runParam |  |

### `admin/refunds.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| GET | `/admin/refunds` | AUTH_READ | platform admin | Q:refundQuerySchema |  |
| POST | `/admin/refunds` | SENSITIVE_COMMAND | platform admin | B:requestRefundSchema |  |
| GET | `/admin/refunds/:refundRequestId` | AUTH_READ | platform admin | P:refundIdParam |  |
| POST | `/admin/refunds/:refundRequestId/approve` | SENSITIVE_COMMAND | platform admin | P:refundIdParam |  |
| POST | `/admin/refunds/:refundRequestId/execute` | SENSITIVE_COMMAND | platform admin | P:refundIdParam |  |
| POST | `/admin/refunds/:refundRequestId/policy-remedy` | SENSITIVE_COMMAND | platform admin | P:refundIdParam |  |
| POST | `/admin/refunds/:refundRequestId/reconcile` | SENSITIVE_COMMAND | platform admin | P:refundIdParam B:reconcileRefundRequestSchema |  |
| POST | `/admin/refunds/:refundRequestId/reject` | SENSITIVE_COMMAND | platform admin | P:refundIdParam B:rejectRefundRequestSchema |  |
| GET | `/admin/refunds/export.csv` | AUTH_READ | platform admin | - |  |

### `admin/settings.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| GET | `/admin/settings/platform` | AUTH_READ | platform admin | - |  |
| PUT | `/admin/settings/platform` | SENSITIVE_COMMAND | platform admin | B:platformSettingsUpdateRequestSchema |  |

### `admin/support.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| GET | `/admin/support/tickets` | AUTH_READ | platform admin | Q:supportTicketQuerySchema |  |
| DELETE | `/admin/support/tickets/:ticketId` | SENSITIVE_COMMAND | platform admin | P:supportIdParam |  |
| GET | `/admin/support/tickets/:ticketId` | AUTH_READ | platform admin | P:supportIdParam |  |
| POST | `/admin/support/tickets/:ticketId/assign` | SENSITIVE_COMMAND | platform admin | P:supportIdParam B:assignSupportTicketSchema |  |
| POST | `/admin/support/tickets/:ticketId/close` | - | platform admin | - | body-free support transition |
| POST | `/admin/support/tickets/:ticketId/escalate` | - | platform admin | - | body-free support transition |
| POST | `/admin/support/tickets/:ticketId/link` | SENSITIVE_COMMAND | platform admin | P:supportIdParam B:supportTicketLinkSchema |  |
| POST | `/admin/support/tickets/:ticketId/merge` | SENSITIVE_COMMAND | platform admin | P:supportIdParam B:mergeSupportTicketSchema |  |
| POST | `/admin/support/tickets/:ticketId/notes` | SENSITIVE_COMMAND | platform admin | P:supportIdParam B:supportTicketMessageSchema |  |
| POST | `/admin/support/tickets/:ticketId/priority` | SENSITIVE_COMMAND | platform admin | P:supportIdParam B:changeSupportTicketPrioritySchema |  |
| POST | `/admin/support/tickets/:ticketId/reopen` | - | platform admin | - | body-free support transition |
| POST | `/admin/support/tickets/:ticketId/reply` | SENSITIVE_COMMAND | platform admin | P:supportIdParam B:supportTicketMessageSchema |  |
| POST | `/admin/support/tickets/:ticketId/resolve` | SENSITIVE_COMMAND | platform admin | P:supportIdParam B:resolveSupportTicketSchema |  |
| POST | `/admin/support/tickets/:ticketId/restore` | - | platform admin | - | body-free support transition |

### `admin/tickets.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| GET | `/admin/events/:eventId/cover-reconciliation` | AUTH_READ | platform admin | P:eventCoverReconciliationParamsSchema |  |
| POST | `/admin/ticket-transfers/:transferId/freeze` | STANDARD_COMMAND | platform admin | P:freezeTransferParamsSchema B:freezeTransferBodySchema |  |
| POST | `/admin/ticket-transfers/:transferId/release` | STANDARD_COMMAND | platform admin | P:freezeTransferParamsSchema B:freezeTransferBodySchema |  |
| GET | `/admin/tickets` | AUTH_READ | platform admin | Q:ticketsQuerySchema |  |
| GET | `/admin/tickets/:ticketId/cover-reconciliation` | AUTH_READ | platform admin | P:ticketTransferParamsSchema |  |
| POST | `/admin/tickets/:ticketId/cover-redemptions/:captureEntryId/reverse` | SENSITIVE_COMMAND | platform admin | P:coverReversalParamsSchema B:coverReversalBodySchema |  |
| GET | `/admin/tickets/:ticketId/event-change-refund-reviews` | AUTH_READ | platform admin | P:ticketTransferParamsSchema |  |
| GET | `/admin/tickets/:ticketId/transfers` | AUTH_READ | platform admin | P:ticketTransferParamsSchema |  |
| GET | `/admin/tickets/export.csv` | AUTH_READ | platform admin | - |  |

### `admin/user-actions.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| POST | `/admin/users/:userId/ban` | SENSITIVE_COMMAND | platform admin | P:userIdParam B:banBodySchema.optional |  |
| POST | `/admin/users/:userId/unban` | SENSITIVE_COMMAND | platform admin | P:userIdParam |  |

### `admin/venue-actions.ts`

| Method | Path | RL | Access | Validation | Notes |
|---|---|---|---|---|---|
| POST | `/admin/venues/:venueId/reinstate` | SENSITIVE_COMMAND | platform admin | P:venueIdParam |  |
| POST | `/admin/venues/:venueId/suspend` | SENSITIVE_COMMAND | platform admin | P:venueIdParam |  |
