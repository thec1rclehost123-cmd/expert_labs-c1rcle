# Wire contracts: every exported zod schema

**Generated** on 2026-10-25 from `packages/contracts/src/contracts/*.ts` (the backend source of truth; the frontend holds a generated copy). For each exported `...Schema`: its kind (object, strict object, enum, array, other) and its top-level fields. A trailing `?` means the field text contains `.optional()`, `~` means `.nullable()`/`.nullish()`, `=` means a `.default(...)`. Field detail (min/max, regex) is in the source file named in each heading. Regenerate after a contract change. See `DEEP-DIVE-OTP-RSVP-CONTRACTS.md` for how the two repos stay in sync and why parity coverage is partial.


## `admin-analytics.ts` (2 schemas)

| Schema | Kind | Fields / values |
|---|---|---|
| `adminAnalyticsTopOrgDtoSchema` | object | `organizationId`, `name`, `revenuePaise` |
| `adminAnalyticsSummaryDtoSchema` | object | `totalRevenuePaise`, `ticketsSold`, `activeEventsCount`, `topOrganizations`, `scannedOrders`, `scannedEvents`, `truncated` |

## `admin-orders.ts` (3 schemas)

| Schema | Kind | Fields / values |
|---|---|---|
| `adminOrderContactDtoSchema` | object | `name`, `email`, `phone` |
| `adminOrderDtoSchema` | object | `id`, `eventId`, `organizationId`, `userId`~, `status`, `ticketCount`, `grandTotalPaise`, `refundedPaise`, `contact`, `paymentId`~, `paidAt`~, `createdAt` |
| `adminOrderListResponseSchema` | paginatedSchema |  |

## `admin-promoters.ts` (5 schemas)

| Schema | Kind | Fields / values |
|---|---|---|
| `adminPromoterAssignmentStatusSchema` | enum | `active`, `ended`, `suspended` |
| `adminPromoterAssignmentDtoSchema` | object | `id`, `eventId`, `promoterId`, `status`, `ratePercent`, `flatPaise`, `createdAt`, `endedAt`~, `suspendedAt`~ |
| `adminPromoterAssignmentListResponseSchema` | paginatedSchema |  |
| `adminPromoterSuspendRequestSchema` | object | `reason`? |
| `adminPromoterActionResponseSchema` | object | `promoterId`, `action`, `affectedAssignments`, `at` |

## `admin-promotions.ts` (4 schemas)

| Schema | Kind | Fields / values |
|---|---|---|
| `adminPromoTypeSchema` | enum | `public`, `private`, `single_use`, `multi_use` |
| `adminPromoDiscountTypeSchema` | enum | `percent`, `fixed` |
| `adminPromoDtoSchema` | object | `id`, `eventId`~, `organizationId`, `code`, `name`, `type`, `discountType`, `discountValue`, `discountFundingOwner`?~, `maxRedemptions`~, `redemptionCount`, `startsAt`~, `endsAt`~, `isActive`, `createdAt` |
| `adminPromoListResponseSchema` | paginatedSchema |  |

## `admin-settings.ts` (2 schemas)

| Schema | Kind | Fields / values |
|---|---|---|
| `platformSettingsDtoSchema` | object | `platformFeeRate`, `refundSingleApproverThresholdPaise`, `refundDualApproverThresholdPaise`, `maintenanceMode`, `featureFlags`, `accountingPolicy`?, `updatedAt` |
| `platformSettingsUpdateRequestSchema` | object | `platformFeeRate`?, `refundSingleApproverThresholdPaise`?, `refundDualApproverThresholdPaise`?, `maintenanceMode`?, `featureFlags`?, `accountingPolicy`? |

## `admin-tickets.ts` (3 schemas)

| Schema | Kind | Fields / values |
|---|---|---|
| `adminTicketStatusSchema` | enum | `valid`, `redeemed`, `void` |
| `adminTicketDtoSchema` | object | `id`, `orderId`, `eventId`, `organizationId`, `tierName`, `userId`~, `holderName`, `status`, `scanCountAllowed`, `scanCount`, `lastScannedAt`~, `createdAt` |
| `adminTicketListResponseSchema` | paginatedSchema |  |

## `admin.ts` (8 schemas)

| Schema | Kind | Fields / values |
|---|---|---|
| `adminVenueDtoSchema` | object | `id`, `organizationId`, `name`, `slug`, `city`~, `status`, `capacity`~, `createdAt`, `updatedAt` |
| `adminVenueListResponseSchema` | paginatedSchema |  |
| `adminEventDtoSchema` | object | `id`, `organizationId`, `venueId`~, `slug`, `title`, `status`, `isPublic`, `adminOverride`, `startAt`, `endAt`~, `startingPricePaise`~, `isFree`, `createdAt`, `updatedAt` |
| `adminEventListResponseSchema` | paginatedSchema |  |
| `adminHostDtoSchema` | object | `id`, `ownerId`, `name`, `slug`, `status`, `platformFeePercent`, `memberCount`, `createdAt`, `updatedAt` |
| `adminHostListResponseSchema` | paginatedSchema |  |
| `adminUserDtoSchema` | object | `id`, `email`, `name`, `image`~, `emailVerified`, `role`~, `isBanned`, `createdAt`, `updatedAt` |
| `adminUserListResponseSchema` | paginatedSchema |  |

## `auth.ts` (13 schemas)

| Schema | Kind | Fields / values |
|---|---|---|
| `roleSchema` | enum | `guest`, `partner`, `admin` |
| `userSchema` | object | `id`, `email`, `displayName`, `role`, `avatarUrl`~, `mustChangePassword`= |
| `sessionSchema` | object | `user`, `expiresAt` |
| `signupRequestSchema` | object | `email`, `password`, `displayName` |
| `loginRequestSchema` | object | `email`, `password` |
| `authBridgeResponseSchema` | object | `user`, `accessToken`, `expiresAt` |
| `changePasswordSchema` | object | `currentPassword`, `newPassword` |
| `otpSendRequestSchema` | object | `email` |
| `otpVerifyRequestSchema` | object | `email`, `code` |
| `otpAckResponseSchema` | object | `message` |
| `forgotPasswordRequestSchema` | object | `email` |
| `resetPasswordRequestSchema` | object | `newPassword`, `token` |
| `passwordResetAckSchema` | object | `status`, `message`? |

## `checkout.ts` (31 schemas)

| Schema | Kind | Fields / values |
|---|---|---|
| `pricingLineDtoSchema` | object | `tierId`, `tierName`, `quantity`, `unitPricePaise`, `subtotalPaise` |
| `pricingBreakdownDtoSchema` | object | `lines`, `subtotalPaise`, `discountPaise`, `discountedSubtotalPaise`, `platformFeePaise`, `paymentFeePaise`, `gstPaise`, `grandTotalPaise`, `appliedPromoCode`~, `currency` |
| `checkoutQuoteRequestSchema` | object | `eventId`, `lines`, `promoCode`?~, `referralCode`?~ |
| `checkoutQuoteResponseSchema` | pricingBreakdownDtoSchema |  |
| `cartReservationStatusSchema` | enum | `active`, `converted`, `released` |
| `checkoutAttendeeSchema` | object | `name`, `email`, `phone`?= |
| `checkoutHoldRequestSchema` | object | `eventId`, `lines`, `promoCode`?~, `referralCode`?~, `attendee`? |
| `checkoutHoldResponseSchema` | object | `holdId`, `expiresAt`, `status`, `pricing`, `convertedOrderId`~ |
| `paymentAttemptRequestSchema` | object | `holdId` |
| `paymentAttemptResponseSchema` | object | `paymentIntentId`, `amountPaise`, `keyId`? |
| `paymentRecoveryResponseSchema` | object | `holdId`, `eventId`, `status`, `orderId`~, `expiresAt`, `updatedAt` |
| `paymentConfirmRequestSchema` | object | `holdId`, `paymentIntentId`, `signature` |
| `checkoutOrderDtoSchema` | object | `id`, `eventId`, `organizationId`, `userId`~, `status`, `lines`, `currency`, `subtotalPaise`, `discountPaise`, `discountedSubtotalPaise`, `platformFeePaise`, `paymentFeePaise`, `gstPaise`, `grandTotalPaise`, `appliedPromoCode`~, `paymentIntentId`~, `paymentId`~, `paidAt`~, `reservationExpiresAt`, `failureReason`~, `refundedPaise`, `version`, `createdAt`, `updatedAt` |
| `orderStatusSchema` | checkoutOrderStatusSchema |  |
| `orderDtoSchema` | checkoutOrderDtoSchema |  |
| `ordersListResponseSchema` | paginatedSchema |  |
| `orderStatusResponseSchema` | object | `id`, `status`, `version`, `updatedAt` |
| `entitlementStatusSchema` | enum | `valid`, `redeemed`, `void` |
| `entitlementDtoSchema` | object | `id`, `orderId`, `eventId`, `organizationId`, `tierId`, `tierName`, `userId`~, `holderName`, `status`, `scanCountAllowed`, `scanCount`, `scannedAt`, `version`, `createdAt`, `updatedAt` |
| `entitlementsListResponseSchema` | paginatedSchema |  |
| `paymentConfirmResponseSchema` | object | `order`, `entitlements` |
| `walletSummaryDtoSchema` | object | `activeTicketCount`, `upcomingOrderCount` |
| `paymentProviderSchema` | enum | `razorpay`, `stripe` |
| `paymentOrderRequestSchema` | object | `amountPaise`, `currency`, `receipt`, `notes`? |
| `paymentOrderResponseSchema` | object | `id`, `amountPaise`, `currency`, `status` |
| `paymentVerificationRequestSchema` | object | `paymentId`, `orderId`, `signature` |
| `paymentVerificationResponseSchema` | object | `verified`, `paymentId`, `amountPaise`, `captured` |
| `refundRequestSchema` | object | `paymentId`, `amountPaise`?, `idempotencyKey` |
| `refundResponseSchema` | object | `id`, `status`, `amountPaise` |
| `rsvpRequestSchema` | object | `eventId`, `tierId` |
| `rsvpResponseSchema` | object | `order`, `entitlements` |

## `event.ts` (33 schemas)

| Schema | Kind | Fields / values |
|---|---|---|
| `eventStatusSchema` | enum | `draft`, `review`, `scheduled`, `published`, `sales_paused`, `started`, `ended`, `archived`, `cancelled` |
| `compensationModelSchema` | enum | `standard`, `custom`, `salary` |
| `salaryPeriodSchema` | enum | `per_event`, `per_day`, `per_month` |
| `eventCompensationSchema` | object | `model`, `globalRatePercent`~, `tierRates`, `salaryAmountPaise`~, `salaryPeriod`~, `salaryNotes`~ |
| `eventDtoSchema` | object | `id`, `organizationId`, `venueId`~, `slug`, `title`, `summary`=, `description`=, `imageUrl`~, `startAt`, `endAt`~, `status`, `isPublic`, `tags`=, `startingPricePaise`~, `isFree`, `cancellationReason`~, `compensation`?~, `version`, `createdAt`, `updatedAt` |
| `eventPreviewDtoSchema` | object | `event`, `isPublic` |
| `createEventSchema` | object | `venueId`, `title`, `summary`?, `description`?, `imageUrl`?~, `startAt`, `endAt`?~, `tags`=, `compensation`?~ |
| `posterContentTypeSchema` | enum | `image/jpeg`, `image/png`, `image/webp` |
| `posterUploadUrlRequestSchema` | object | `contentType` |
| `posterUploadUrlDtoSchema` | object | `uploadUrl`, `method`, `headers`, `storagePath`, `publicUrl`, `expiresAt` |
| `updateEventSchema` | object | `title`?, `summary`?, `description`?, `imageUrl`?~, `startAt`?, `endAt`?~, `tags`?, `startingPricePaise`?, `isFree`?, `compensation`?~ |
| `cancelEventSchema` | object | `reason` |
| `ticketTierStatusSchema` | enum | `active`, `paused`, `sold_out` |
| `ticketAccessTypeSchema` | enum | `ENTRY`, `VIP`, `VVIP`, `TABLE`, `PACKAGE`, `RSVP` |
| `ticketAudienceTypeSchema` | enum | `GENERAL`, `MALE`, `FEMALE`, `COUPLE`, `GROUP` |
| `ticketPricingPhaseSchema` | object | `id`, `name`, `priceInPaise`, `startsAt`, `endsAt`, `quantity`~ |
| `createTicketPricingPhaseSchema` | union |  |
| `ticketTierDtoSchema` | object | `id`, `eventId`, `organizationId`, `name`, `description`, `entryType`, `currency`, `priceInPaise`, `quantity`, `status`, `salesStartAt`~, `salesEndAt`~, `maxPerOrder`~, `accessType`?, `audienceType`?, `guestCount`?, `pricingPhases`?, `doorPriceInPaise`?~, `benefits`?, `minAge`?~, `maxAge`?~, `minPerOrder`?~, `maxPerUser`?~, `tableConfig`?~, `commissionEligible`?, `version`, `createdAt`, `update… |
| `createTicketTierSchema` | createTicketTierBaseSchema.superRefine |  |
| `promoTypeSchema` | enum | `public`, `private`, `single_use`, `multi_use` |
| `promoDiscountTypeSchema` | enum | `percent`, `fixed` |
| `discountFundingOwnerSchema` | enum | `platform`, `venue`, `host`, `promoter`, `shared` |
| `promoCodeDtoSchema` | object | `id`, `eventId`~, `organizationId`, `code`, `name`, `type`, `discountType`, `discountValue`, `discountFundingOwner`, `tierIds`, `maxRedemptions`~, `maxPerUser`~, `redemptionCount`, `startsAt`~, `endsAt`~, `isActive`, `version`, `createdAt`, `updatedAt` |
| `createPromoCodeSchema` | object | `code`, `name`?, `type`?, `discountType`, `discountValue`, `discountFundingOwner`, `tierIds`?, `maxRedemptions`?~, `maxPerUser`?~, `startsAt`?~, `endsAt`?~ |
| `tablePackageDtoSchema` | object | `id`, `eventId`, `organizationId`, `name`, `capacity`, `pricePaise`, `minSpendPaise`~, `isActive`, `version`, `createdAt`, `updatedAt` |
| `createTablePackageSchema` | object | `name`, `capacity`, `pricePaise`, `minSpendPaise`?~ |
| `commissionRateSchema` | object | `ratePercent`, `flatPaise` |
| `commissionTermsSchema` | object | `version`, `ratePercent`, `flatPaise`, `tierRates`? |
| `promoterAssignmentDtoSchema` | object | `id`, `eventId`, `promoterId`, `status`, `terms`, `endedAt`~, `version`, `createdAt`, `updatedAt` |
| `promoterAssignedEventDtoSchema` | object | `event`, `assignment` |
| `assignPromoterSchema` | object | `promoterId`, `ratePercent`?, `flatPaise`?, `tierRates`? |
| `publicTicketTierDtoSchema` | object | `id`, `eventId`, `name`, `description`, `priceInPaise`, `currency`, `availableQuantity` |
| `publicTicketTierListResponseSchema` | object | `items` |

## `guest-profile.ts` (2 schemas)

| Schema | Kind | Fields / values |
|---|---|---|
| `upsertGuestProfileSchema` | object | `displayName`, `dateOfBirth`, `city`, `tastes`, `intents` |
| `guestProfileDtoSchema` | object | `userId`, `displayName`, `dateOfBirth`, `city`, `tastes`, `intents`, `createdAt`, `updatedAt` |

## `notifications.ts` (10 schemas)

| Schema | Kind | Fields / values |
|---|---|---|
| `notificationRecipientTypeSchema` | enum | `venue`, `host`, `promoter` |
| `notificationActionTypeSchema` | enum | `promoter_connection`, `partnership`, `slot_request` |
| `notificationActionSchema` | object | `resourceType`, `resourceId` |
| `notificationPrioritySchema` | enum | `normal`, `high` |
| `notificationDtoSchema` | object | `id`, `recipientId`, `recipientType`, `type`, `title`, `body`, `read`, `readAt`~, `data`, `action`~, `priority`, `createdAt` |
| `notificationsListResponseSchema` | object | `items`, `pageInfo`, `unreadCount` |
| `notificationReadRequestSchema` | object |  |
| `markAllNotificationsReadResultSchema` | object | `selected` |
| `notificationDecisionSchema` | enum | `approve`, `reject` |
| `notificationActionRequestSchema` | object | `decision` |

## `onboarding.ts` (33 schemas)

| Schema | Kind | Fields / values |
|---|---|---|
| `onboardingStatusSchema` | enum | `draft`, `submitted`, `changes_requested`, `approved`, `rejected` |
| `onboardingPlanSchema` | enum | `basic`, `silver`, `diamond` |
| `onboardingProfileSchema` | object | `legalName`, `contactPerson`, `phone`, `city`, `area`?, `website`?, `capacity`?~, `instagram`?, `bio`?, `businessType`?, `registrationNumber`?, `entityType`? |
| `onboardingDocumentStatusSchema` | enum | `pending`, `verified`, `rejected` |
| `onboardingDocumentSchema` | object | `label`, `storagePath`, `uploadedAt`, `status`, `reviewedBy`~, `reviewedAt`~, `rejectionReason`~ |
| `onboardingRequestDtoSchema` | object | `id`, `userId`, `status`, `requestedType`, `plan`, `profile`, `documents`, `missingDocuments`, `submittedAt`~, `reviewedBy`~, `reviewedAt`~, `reviewNote`~, `provisionedOrganizationId`~, `version`, `createdAt`, `updatedAt` |
| `startOnboardingSchema` | object | `requestedType`, `plan`, `profile` |
| `saveOnboardingProgressSchema` | onboardingProfileSchema.partial |  |
| `addOnboardingDocumentSchema` | object | `label`, `storagePath` |
| `onboardingDocumentLabelSchema` | enum | `id_front`, `id_back`, `selfie`, `registration_certificate`, `sig_id_front`, `sig_id_back`, `sig_selfie` |
| `documentUploadUrlRequestSchema` | object | `label`, `contentType` |
| `documentUploadUrlDtoSchema` | object | `uploadUrl`, `method`, `headers`, `storagePath`, `expiresAt` |
| `documentReadUrlDtoSchema` | object | `readUrl`, `expiresAt` |
| `verifyDocumentSchema` | object | `documentType`, `documentNumber`, `holderName`?, `proofToken`? |
| `verificationResultDtoSchema` | object | `passed`, `provider`, `reason`~, `referenceId`~ |
| `reviewOnboardingSchema` | object | `note`? |
| `rejectKycDocumentSchema` | object | `reason` |
| `provisionedOrganizationDtoSchema` | object | `id`, `name`, `slug`, `ownerId`, `platformFeePercent`, `status`, `version`, `createdAt`, `updatedAt` |
| `approveOnboardingResultSchema` | object | `request`?~, `organization` |
| `adminRoleSchema` | enum | `super`, `admin`, `ops`, `finance`, `support` |
| `adminActionSchema` | enum | `EVENT_PAUSE`, `EVENT_RESUME`, `EVENT_FORCE_PAUSE`, `ONBOARDING_APPROVE`, `VENUE_SUSPEND`, `VENUE_REINSTATE`, `ORGANIZATION_SUSPEND`, `ORGANIZATION_REINSTATE`, `FINANCIAL_REFUND`, `PAYOUT_BATCH_RUN`, `PAYOUT_COMPLETE`, `DISPUTE_RESOLVE`, `USER_BAN`, `USER_UNBAN`, `ADMIN_PROVISION`, `ADMIN_ROLE_UPDATE`, `COMMISSION_ADJUST`, `FINANCIAL_ADJUSTMENT`, `PAYOUT_FREEZE`, `PAYOUT_RELEASE`, `PROMOTER_SUS… |
| `proposalStatusSchema` | enum | `pending`, `approved`, `rejected`, `cancelled` |
| `platformAdminDtoSchema` | object | `id`, `email`, `role`, `isActive`, `version`, `createdAt`, `updatedAt` |
| `proposedActionDtoSchema` | object | `id`, `action`, `proposedBy`, `reason`, `payload`, `status`, `resolvedBy`~, `resolvedAt`~, `resolutionReason`~, `version`, `createdAt`, `updatedAt` |
| `proposeActionSchema` | object | `action`, `reason`, `payload`? |
| `resolveProposalSchema` | object | `reason`? |
| `adminAuditRecordDtoSchema` | object | `id`, `adminId`, `adminRole`~, `action`, `targetType`, `targetId`, `targetName`~, `before`~, `after`~, `reason`~, `occurredAt` |
| `adminAlertCategoryKeySchema` | enum | `pending_proposals`, `pending_onboarding` |
| `adminAlertSeveritySchema` | enum | `normal`, `urgent` |
| `adminAlertCategorySchema` | object | `key`, `label`, `count`, `severity` |
| `adminAlertsResponseSchema` | object | `generatedAt`, `categories` |
| `adminLookupResultItemSchema` | object | `type`, `id`, `label` |
| `adminLookupResponseSchema` | object | `items` |

## `organization.ts` (30 schemas)

| Schema | Kind | Fields / values |
|---|---|---|
| `organizationRoleSchema` | enum | `owner`, `admin`, `manager`, `member` |
| `organizationStatusSchema` | enum | `active`, `suspended`, `archived` |
| `organizationDtoSchema` | object | `id`, `name`, `slug`, `role`, `status`, `version`, `createdAt`, `updatedAt` |
| `createOrganizationSchema` | object | `name`, `slug` |
| `venueStatusSchema` | enum | `active`, `suspended` |
| `venueDtoSchema` | object | `id`, `organizationId`, `name`, `slug`, `status`, `description`=, `capacity`~, `city`~, `version`, `createdAt`, `updatedAt` |
| `createVenueSchema` | object | `name`, `slug`, `description`?, `capacity`?~, `city`?~ |
| `organizationMemberDtoSchema` | object | `userId`, `role`, `capabilities`, `joinedAt`, `invitedBy`?, `name`?~, `email`?~ |
| `updateMemberRoleSchema` | object | `role` |
| `inviteMemberSchema` | object | `userId`, `role`, `capabilities`? |
| `venueAddressSchema` | object | `street`?, `city`?, `state`?, `zip`?, `country`?, `lat`?, `lng`? |
| `venuePublicProfileSchema` | object | `name`, `slug`, `description`, `shortDescription`?, `photoUrl`~, `address`, `facilities`, `capacity`~, `settings` |
| `venuePrivateProfileSchema` | object | `contactEmail`~, `contactPhone`~, `socials`?, `internalNotes` |
| `venueProfileDtoSchema` | object | `public`, `private` |
| `venueSlotDtoSchema` | object | `id`, `venueId`, `label`, `startTime`, `endTime`, `recurring`, `status`, `capacityFor`~, `version`, `createdAt`, `updatedAt` |
| `createVenueBlockSchema` | object | `label`, `startTime`, `endTime` |
| `slotRequestDtoSchema` | object | `id`, `venueId`, `eventId`~, `hostId`, `status`, `message`?, `version`, `createdAt`, `updatedAt` |
| `createSlotRequestSchema` | object | `eventId`?~, `message`? |
| `slotRequestListResponseSchema` | paginatedSchema |  |
| `slotRequestActorSchema` | object | `id`, `name` |
| `slotRequestDetailDtoSchema` | object | `request`, `event`~, `venue`, `host`~ |
| `invitationStatusSchema` | enum | `pending`, `accepted`, `revoked`, `expired` |
| `invitationDtoSchema` | object | `id`, `organizationId`, `email`, `role`, `capabilities`, `status`, `invitedBy`, `expiresAt`, `acceptedAt`~, `version`, `createdAt`, `updatedAt` |
| `createInvitationSchema` | object | `email`, `role`, `capabilities`? |
| `venueAvailabilitySlotSchema` | object | `id`, `label`, `startTime`, `endTime`, `status`, `capacityFor`~ |
| `venueAvailabilityDtoSchema` | object | `venueId`, `from`, `to`, `openSlots`, `bookedSlots`, `blockedSlots`, `openMinutes`, `fullyBooked`, `slots` |
| `venueMenuItemSchema` | object | `name`, `description`?, `pricePaise`~, `tags`= |
| `venueMenuSectionSchema` | object | `name`, `items` |
| `venueMenuDtoSchema` | object | `sections`, `updatedAt`~ |
| `updateVenueMenuSchema` | object | `sections` |

## `partner.ts` (29 schemas)

| Schema | Kind | Fields / values |
|---|---|---|
| `partnershipStatusSchema` | enum | `pending`, `active`, `rejected`, `blocked`, `ended` |
| `partnershipDtoSchema` | object | `id`, `hostOrganizationId`, `venueOrganizationId`, `venueId`, `initiatedBy`, `status`, `message`~, `venueShareRate`~, `resolutionReason`~, `resolvedAt`~, `version`, `createdAt`, `updatedAt`, `hostName`~, `hostSlug`~, `venueName`~, `venueSlug`~, `venueCity`~ |
| `requestPartnershipSchema` | object | `venueId`, `initiatedBy`, `hostOrganizationId`?, `message`?, `venueShareRate`? |
| `resolvePartnershipSchema` | object | `reason`? |
| `setVenueShareRequestSchema` | object | `venueShareRate`~ |
| `partnerTypeSchema` | enum | `venue`, `host`, `promoter` |
| `partnerPermissionSchema` | enum | `VIEW_FINANCIALS`, `MANAGE_STAFF`, `MANAGE_EVENTS`, `EDIT_EVENT_RULES`, `MANAGE_TABLES`, `VIEW_GUESTLIST`, `SCAN_ENTRY`, `LOG_INCIDENTS`, `VIEW_ANALYTICS`, `MANAGE_SETTINGS`, `MANAGE_PROMOTERS`, `MANAGE_PAYOUTS`, `MANAGE_PARTNERSHIPS`, `MANAGE_PAGE_CONTENT`, `VIEW_REAL_TIME_SCANS`, `MANAGE_GUEST_OPS`, `CHARGE_COVER_WALLETS`, `EXPORT_GUESTS` |
| `partnerAccessDtoSchema` | object | `organizationId`, `userId`, `partnerType`, `role`, `permissions`, `tabVisibility`~ |
| `topEventDtoSchema` | object | `eventId`, `title`, `revenuePaise`, `tickets`, `date` |
| `organizationOverviewDtoSchema` | object | `organizationId`, `totalEvents`, `publishedEvents`, `totalRevenuePaise`, `totalTicketsSold`, `totalCheckIns`, `topEvents`, `lastEventAt`~ |
| `trendGranularitySchema` | enum | `hour`, `day`, `month` |
| `trendBucketDtoSchema` | object | `key`, `revenuePaise`, `tickets`, `checkIns` |
| `organizationTrendsDtoSchema` | object | `organizationId`, `granularity`, `from`, `to`, `buckets`, `totals` |
| `organizationEventCardDtoSchema` | object | `eventId`, `title`, `startAt`, `status`, `venueId`~, `venueName`~, `imageUrl`~, `ticketsSold`, `capacity`~ |
| `organizationEventCardListResponseSchema` | object | `organizationId`, `items` |
| `organizationCalendarDtoSchema` | object | `organizationId`, `month`, `firstDayOffset`, `days` |
| `eventAnalyticsDtoSchema` | object | `eventId`, `totalRevenuePaise`, `ticketsSold`, `totalCheckIns`, `capacity`, `views`, `guestlistSignups`, `avgTicketPricePaise`, `occupancyRate`, `sellThroughRate`, `refundAmountPaise`, `refundRate`, `noShowRate`, `repeatGuests`, `conversionRate` |
| `referralLinkDtoSchema` | object | `id`, `eventId`, `promoterId`, `organizationId`, `assignmentId`~, `assignmentVersion`~, `termsSnapshot`?~, `attributionSignature`~, `eventTitle`, `campaignLabel`, `vanityPrefix`, `vanitySlug`~, `code`, `label`, `isActive`, `clicks`, `conversions`, `revenuePaise`, `commissionPaise`, `version`, `createdAt`, `updatedAt` |
| `createReferralLinkSchema` | object | `promoterId`, `code`?, `label`? |
| `promoterRateTypeSchema` | enum | `percentage`, `fixed` |
| `promoterCompensationModelSchema` | enum | `standard`, `custom`, `salary` |
| `promoterCompensationRateSchema` | object | `rateType`, `rateValue` |
| `promoterCompensationConfigSchema` | object | `model`, `globalCommission`, `tierCommissions`?, `salaryAmount`?, `salaryNotes`? |
| `promoterConnectionStatusSchema` | enum | `pending`, `active`, `rejected`, `blocked`, `revoked` |
| `promoterConnectionDtoSchema` | object | `id`, `promoterId`, `targetId`, `targetType`, `initiatedBy`, `status`, `message`~, `resolutionReason`~, `resolvedAt`~, `version`, `createdAt`, `updatedAt`, `promoterName`~, `promoterSlug`~, `targetName`~, `targetSlug`~, `targetCity`~ |
| `requestConnectionSchema` | object | `counterpartyId`, `targetType`, `initiatedBy`, `message`? |
| `discoverPartnerKindSchema` | enum | `host`, `venue`, `promoter` |
| `discoverPartnerDtoSchema` | object | `id`, `kind`, `name`, `slug`, `city`~, `verified`, `organizationId`~, `venueId`~ |
| `discoverPartnersQuerySchema` | object | `type`?, `q`?, `cursor`?, `limit`= |

## `phase5.ts` (63 schemas)

| Schema | Kind | Fields / values |
|---|---|---|
| `eventCodeTypeSchema` | enum | `full`, `scan_only`, `charge` |
| `eventCodeCreateBodySchema` | object | `type`, `gate`~=, `maxDevices`?, `allowReuse`?, `expiresAt`~= |
| `eventCodeDtoSchema` | object | `id`, `code`, `eventId`, `organizationId`, `venueId`~, `type`, `gate`~, `status`, `maxDevices`, `allowReuse`, `expiresAt`~, `revokedAt`~, `revokedReason`~, `stats`~, `createdAt` |
| `eventCodeListResponseSchema` | object | `items` |
| `revokeReasonBodySchema` | object | `reason` |
| `scannerSessionCreateBodySchema` | object | `eventId`, `code`, `deviceId`, `deviceName`, `sessionType`, `deviceLocation`? |
| `scannerSessionTokenHeaderSchema` | loose object | `x-scanner-session-token` |
| `scannerSessionDtoSchema` | object | `id`, `eventId`, `codeId`, `sessionToken`~, `sessionExpiresAt`, `deviceId`~, `deviceName`~, `permissions`, `status`, `createdAt` |
| `scanRequestSchema` | object | `eventId`, `qrPayload`, `operatorName`?, `operatorRole`?, `gate`? |
| `scanDenyReasonSchema` | enum | `invalid_signature`, `already_used`, `expired`, `wrong_event`, `device_invalid`, `void_ticket`, `capacity_exceeded`, `wrong_gate`, `offline_expired`, `override_required`, `promoter_not_authorized` |
| `scanResponseSchema` | object | `status`, `checkInId`?, `confirmation`?, `denyReason`?~, `denyMessage`?~, `entitlement`? |
| `ticketLookupResponseSchema` | object | `status`, `denyReason`~, `denyMessage`~, `entitlement` |
| `checkInDtoSchema` | object | `id`, `eventId`, `status`, `denyReason`~, `denyMessage`~, `entitlementId`~, `tierId`~, `tierName`~, `operatorUid`~, `operatorName`~, `operatorRole`~, `gate`~, `deviceId`~, `guestName`~, `scannedAt`, `admittedCount`, `scanCountUsed`~, `scanCountAllowed`~, `isOffline`, `overriddenBy`~, `overrideReason`~, `createdAt` |
| `checkInListResponseSchema` | paginatedSchema |  |
| `overrideRequestSchema` | object | `checkInId`, `reason` |
| `overrideResponseSchema` | object | `checkInId`, `status`, `overriddenBy`, `overrideReason` |
| `doorStatsQuerySchema` | object | `eventId` |
| `doorStatsDtoSchema` | object | `eventId`, `occupancy`~, `byEntryType`, `scans`, `doorSales`, `coverWallet`, `generatedAt` |
| `magicQrRequestSchema` | object | `entitlementId`, `tierPricePaise` |
| `magicQrResponseSchema` | object | `qrPayload`, `expiresAt`, `refreshIntervalSec`= |
| `offlineManifestRequestSchema` | object | `eventId` |
| `offlineManifestResponseSchema` | object | `manifest`, `signedAt`, `expiresAt` |
| `offlineSyncRequestSchema` | object | `eventId`, `scans` |
| `offlineSyncResponseSchema` | object | `synced`, `conflicts` |
| `doorWalkInRequestSchema` | object | `eventId`, `guestName`, `guestPhone`?~, `guestAge`?~, `gender`?~, `guestEmail`?~, `totalGuests`=, `gate`?~, `paymentMode`=, `idempotencyKey` |
| `doorDineInRequestSchema` | extended object |  |
| `doorSaleResponseSchema` | object | `id`, `eventId`, `category`, `guestName`, `totalGuests`, `amountPaise`, `paymentMode`, `status`, `createdAt` |
| `doorSalesListResponseSchema` | paginatedSchema |  |
| `coverWalletIssueRequestSchema` | object | `eventId`, `userId`, `openingBalancePaise`, `metadata`? |
| `coverWalletDebitRequestSchema` | object | `walletId`, `amountPaise`, `referenceId`?, `referenceType`?, `description`?, `idempotencyKey`, `deviceId`? |
| `coverWalletCreditRequestSchema` | object | `walletId`, `amountPaise`, `referenceId`?, `referenceType`?, `description`?, `idempotencyKey` |
| `coverWalletResponseSchema` | object | `id`, `eventId`, `userId`, `balancePaise`, `status`, `openingBalancePaise`, `totalCreditsPaise`, `totalDebitsPaise`, `terminatedAt`~, `createdAt` |
| `coverWalletReconciliationRequestSchema` | object | `eventId`, `reconciliationDate`, `walletId`?, `userId`? |
| `coverWalletReconciliationResponseSchema` | object | `id`, `eventId`, `reconciliationDate`, `expectedBalancePaise`, `actualBalancePaise`, `discrepancyPaise`, `status`, `discrepancies`~, `createdAt` |
| `phase5ErrorCodeSchema` | enum | `SCANNER_SESSION_EXPIRED`, `VELOCITY_LIMIT_EXCEEDED`, `WALLET_TERMINATED`, `OFFLINE_MANIFEST_EXPIRED`, `MAGIC_TICKET_SIGNATURE_INVALID`, `SCANNER_SESSION_REVOKED`, `COVER_WALLET_INSUFFICIENT_BALANCE`, `DOOR_SALE_PRICE_MISMATCH` |
| `doorEventListQuerySchema` | object | `date`= |
| `doorEventSummarySchema` | object | `id`, `title`, `slug`, `venueId`~, `startAt`, `endAt`~, `status`, `capacity`~ |
| `doorEventListResponseSchema` | object | `items` |
| `doorTierSummarySchema` | object | `id`, `name`, `entryType`, `pricePaise`, `available` |
| `startShiftResponseSchema` | object | `sessionId`, `sessionToken`, `sessionExpiresAt`, `event`, `permissions`, `gate`~, `tiers`, `stats`, `device` |
| `scannerDeviceBindBodySchema` | object | `deviceId`, `deviceName` |
| `scannerDeviceReauthorizeBodySchema` | object | `deviceName` |
| `scannerDeviceDtoSchema` | object | `id`, `deviceId`, `deviceName`, `venueId`~, `status`, `boundAt`, `unboundAt`~, `unboundReason`~, `lastSeenAt`, `lastEventId`~, `lastGate`~, `scanCount`, `lastScanAt`~, `lastScanResult`~ |
| `scannerDeviceListResponseSchema` | object | `items` |
| `scannerHeartbeatBodySchema` | object | `eventId`, `gate`? |
| `confirmCoupleBodySchema` | object | `eventId`, `confirmationToken`, `confirmed`, `operatorName`?, `operatorRole`?, `gate`? |
| `staffDenyBodySchema` | object | `eventId`, `qrPayload`?, `reason`, `gate`? |
| `doorGuestSchema` | object | `id`, `name`, `ticketType`, `entryType`, `quantity`, `source`, `status`, `enteredAt`~, `scansUsed`~, `scansAllowed`~ |
| `doorGuestListQuerySchema` | object | `eventId`, `status`?, `source`?, `search`?, `limit`= |
| `doorGuestListResponseSchema` | object | `items`, `truncated` |
| `attendanceReportQuerySchema` | object | `eventId` |
| `attendanceReportGuestSchema` | object | `entitlementId`, `holderName`, `tierName`, `status`, `scanCount`, `scanCountAllowed`, `scannedAt` |
| `attendanceReportTierBreakdownSchema` | object | `tierName`, `entered`, `notEntered` |
| `attendanceReportDtoSchema` | object | `eventId`, `totalEntitlements`, `enteredEntitlements`, `admittedCount`, `notEntered`, `voided`, `byTier`, `guests`, `truncated` |
| `manualCheckInBodySchema` | object | `eventId`, `entitlementId` |
| `manualCheckInResponseSchema` | object | `guest`, `checkInId` |
| `coverWalletPresetItemSchema` | object | `id`, `label`, `amountPaise`, `isAvailable` |
| `walletChargeViewSchema` | object | `walletId`, `eventId`, `guestFirstName`, `status`, `balancePaise`~, `presetItems`, `minChargePaise`, `maxChargePaise` |
| `walletQrResolveBodySchema` | object | `eventId`, `qrPayload` |
| `walletChargeBodySchema` | object | `eventId`, `qrPayload`, `presetItemId`, `quantity`=, `idempotencyKey` |
| `walletChargeResponseSchema` | object | `wallet`, `charged`, `balancePaise` |
| `doorTicketSaleBodySchema` | object | `eventId`, `tierId`, `quantity`=, `paymentMode`, `guestName`, `guestPhone`?~, `guestEmail`?~, `guestAge`?~, `gender`?~, `gate`?~, `idempotencyKey` |
| `doorTicketSaleResponseSchema` | object | `orderId`, `amountPaise`, `quantity`, `paymentMode`, `ticketIds`, `checkInIds`, `replayed` |

## `phase6.ts` (43 schemas)

| Schema | Kind | Fields / values |
|---|---|---|
| `ledgerEntryDtoSchema` | object | `id`, `orderId`, `eventId`, `entryType`, `amountPaise`, `status`, `createdAt` |
| `ledgerEntryListResponseSchema` | paginatedSchema |  |
| `balanceSummaryResponseSchema` | object | `availablePaise`, `pendingPaise`, `reservedPaise`, `heldPaise`, `paidPaise`, `lifetimePaise`, `holds`, `readModel` |
| `financeOrderDtoSchema` | object | `id`, `eventId`, `eventName`, `buyerName`, `ticketCount`, `currency`, `grandTotalPaise`, `gstPaise`?=, `refundedPaise`, `status`, `paidAt`~, `createdAt` |
| `financeOrderListResponseSchema` | paginatedSchema |  |
| `payoutRequestSchema` | object | `amountPaise`, `bankAccountId`?, `idempotencyKey` |
| `payoutResponseSchema` | object | `id`, `organizationId`, `bankAccountId`, `amountPaise`, `status`, `failureReason`~, `processedAt`~, `transferReference`~, `transferredAt`~, `transferMethod`~, `transferProof`~, `approvedBy`~, `createdAt` |
| `payoutListResponseSchema` | paginatedSchema |  |
| `adminPayoutStatusSchema` | enum | `requested`, `processing`, `paid`, `failed`, `frozen` |
| `runPayoutBatchSchema` | object | `payoutIds` |
| `completeManualPayoutSchema` | object | `transferReference`, `transferredAt`, `transferMethod`, `proof` |
| `payoutBatchResultSchema` | object | `processed`, `skipped` |
| `bankAccountRequestSchema` | object | `bankName`, `accountHolder`, `accountNumber`, `ifscCode` |
| `bankAccountResponseSchema` | object | `id`, `bankName`, `accountHolder`, `maskedAccountNumber`, `ifscCode`, `isDefault`, `verified`, `createdAt` |
| `bankAccountListResponseSchema` | object | `items` |
| `raiseDisputeRequestSchema` | object | `orderId`, `ledgerEntryId`?, `reason`, `amountPaise` |
| `resolveDisputeRequestSchema` | object | `resolutionNote` |
| `disputeResolutionOutcomeSchema` | enum | `upheld`, `denied` |
| `disputeResponseSchema` | object | `id`, `organizationId`, `orderId`, `ledgerEntryId`~, `raisedBy`, `reason`, `amountPaise`, `status`, `resolutionNote`~, `resolvedAt`~, `resolution`~, `createdAt` |
| `disputeListResponseSchema` | paginatedSchema |  |
| `adminDisputeStatusSchema` | enum | `open`, `under_review`, `resolved` |
| `adminResolveDisputeSchema` | object | `outcome`, `resolutionNote` |
| `requestRefundSchema` | object | `orderId`, `amountPaise`, `reason`, `ticketLines`? |
| `rejectRefundRequestSchema` | object | `reason` |
| `adminRefundApprovalDtoSchema` | object | `adminId`, `approvedAt` |
| `adminRefundRequestStatusSchema` | enum | `pending`, `approved`, `executing`, `rejected`, `settled`, `failed` |
| `adminRefundRequestDtoSchema` | object | `id`, `orderId`, `organizationId`, `amountPaise`, `requestedBy`, `reason`, `scope`, `ticketLines`, `targetEntitlementIds`, `approversRequired`, `approvals`, `status`, `rejectedBy`~, `rejectionReason`~, `providerRefundId`~, `providerStatus`~, `executionIdempotencyKey`, `executionAttempts`, `failureReason`~, `createdAt`, `updatedAt` |
| `adminRefundRequestListResponseSchema` | paginatedSchema |  |
| `reconciliationRunStatusSchema` | enum | `running`, `completed`, `failed` |
| `reconciliationDiscrepancyKindSchema` | enum | `captured_payment_without_order`, `paid_order_without_capture`, `paid_order_without_ticket`, `ticket_without_paid_order`, `missing_journal`, `duplicate_journal`, `unbalanced_journal`, `amount_mismatch`, `currency_mismatch`, `provider_order_mismatch`, `refund_mismatch`, `chargeback_mismatch`, `stale_pending_payment`, `payout_mismatch` |
| `reconciliationDiscrepancyStateSchema` | enum | `open`, `finance_review`, `support_required`, `resolved` |
| `reconciliationRunDtoSchema` | object | `id`, `status`, `requestedBy`, `startedAt`, `completedAt`~, `scannedCounts`, `discrepancyCount`, `failureReason`~ |
| `reconciliationDiscrepancyDtoSchema` | object | `id`, `runId`, `kind`, `state`, `organizationId`~, `orderId`~, `paymentId`~, `payoutId`~, `amountPaise`~, `currency`~, `evidence`, `repair`~, `resolutionReason`~, `resolvedBy`~, `resolvedAt`~, `createdAt`, `updatedAt` |
| `reconciliationRunListResponseSchema` | paginatedSchema |  |
| `reconciliationDiscrepancyListResponseSchema` | paginatedSchema |  |
| `resolveReconciliationDiscrepancySchema` | object | `reason` |
| `providerChargebackDtoSchema` | object | `disputeId`, `paymentId`, `orderId`~, `organizationId`~, `amountPaise`, `amountDeductedPaise`, `currency`, `status`, `reasonCode`~, `reasonDescription`~, `respondBy`~, `phase`~, `evidenceState`, `providerOccurredAt`~ |
| `providerChargebackListResponseSchema` | paginatedSchema |  |
| `financialAdjustmentResultSchema` | object | `adjustmentId`, `organizationId`, `amountPaise`, `direction`, `journalId`, `postedAt` |
| `leaderboardPeriodTypeSchema` | enum | `all_time`, `month`, `week` |
| `leaderboardQuerySchema` | object | `periodType`=, `periodValue`?, `city`?, `limit`= |
| `leaderboardStatResponseSchema` | object | `promoterId`, `periodType`, `periodValue`, `city`, `totalCommissionEarnedPaise`, `updatedAt` |
| `leaderboardTopResponseSchema` | object | `items` |

## `phase7.ts` (21 schemas)

| Schema | Kind | Fields / values |
|---|---|---|
| `supportTicketStatusSchema` | enum | `open`, `in_progress`, `waiting_on_customer`, `escalated`, `resolved`, `closed` |
| `supportTicketPrioritySchema` | enum | `low`, `medium`, `high`, `urgent` |
| `supportTicketCategorySchema` | enum | `account`, `billing`, `order`, `event`, `technical`, `other` |
| `supportTicketMessageDtoSchema` | object | `id`, `senderRole`, `senderId`, `senderName`, `content`, `createdAt` |
| `supportInternalNoteDtoSchema` | object | `id`, `authorId`, `authorName`, `content`, `createdAt` |
| `supportTimelineEventDtoSchema` | object | `id`, `type`, `message`, `detail`~, `actorId`~, `at` |
| `supportTicketLinksDtoSchema` | object | `venueId`~, `eventId`~, `orderId`~, `organizationId`~, `userId`~ |
| `supportSlaDtoSchema` | object | `responseDueAt`, `resolutionDueAt`, `responseBreachedAt`~, `resolutionBreachedAt`~ |
| `supportTicketRequesterDtoSchema` | object | `userId`, `email`~, `organizationId`~ |
| `supportTicketAssigneeDtoSchema` | object | `userId`, `name` |
| `supportTicketDtoSchema` | object | `id`, `subject`, `description`, `category`, `status`, `priority`, `requester`, `assignee`~, `messages`, `internalNotes`, `timeline`, `links`, `sla`, `mergedInto`~, `mergedFrom`, `resolvedAt`~, `resolvedBy`~, `closedAt`~, `closedBy`~, `deletedAt`~, `deletedBy`~, `createdAt`, `updatedAt` |
| `supportTicketListResponseSchema` | paginatedSchema |  |
| `submitSupportTicketSchema` | object | `subject`, `description`, `category`, `priority`=, `email`~, `organizationId`~ |
| `supportTicketMessageSchema` | object | `content` |
| `assignSupportTicketSchema` | object | `userId`, `name` |
| `changeSupportTicketPrioritySchema` | object | `priority` |
| `supportTicketLinkSchema` | object | `venueId`?, `eventId`?, `orderId`?, `organizationId`?, `userId`? |
| `resolveSupportTicketSchema` | object | `reason` |
| `mergeSupportTicketSchema` | object | `duplicateTicketId` |
| `supportTicketQuerySchema` | extended object |  |
| `supportTicketIdParamSchema` | object | `ticketId` |

## `public.ts` (6 schemas)

| Schema | Kind | Fields / values |
|---|---|---|
| `hostPublicDtoSchema` | object | `id`, `name`, `slug` |
| `venuePublicDetailDtoSchema` | extended object |  |
| `eventVenuePublicDtoSchema` | object | `id`, `name`, `slug`, `photoUrl`~, `address` |
| `eventOrganizerPublicDtoSchema` | hostPublicDtoSchema |  |
| `eventPublicDetailDtoSchema` | extended object |  |
| `discoveryFeedDtoSchema` | object | `items` |

## `shared.ts` (8 schemas)

| Schema | Kind | Fields / values |
|---|---|---|
| `pageInfoSchema` | object | `page`, `pageSize`, `total`, `hasNextPage` |
| `noContentSchema` | z.void |  |
| `opaqueIdSchema` | z
  .string |  |
| `cursorSchema` | z.string |  |
| `paginationQuerySchema` | object | `cursor`?, `limit`= |
| `idempotencyKeySchema` | z
  .string |  |
| `versionHeaderSchema` | z
  .string |  |
| `organizationIdSchema` | opaqueIdSchema |  |

## `social.ts` (14 schemas)

| Schema | Kind | Fields / values |
|---|---|---|
| `followTargetTypeSchema` | enum | `venue`, `host` |
| `createFollowSchema` | object | `targetType`, `targetId` |
| `followTargetParamsSchema` | object | `targetType`, `targetId` |
| `followDtoSchema` | object | `id`, `targetType`, `targetId`, `createdAt` |
| `listMyFollowsQuerySchema` | object | `cursor`?, `limit`=, `targetType`? |
| `followListResponseSchema` | object | `items`, `pageInfo`, `nextCursor`~ |
| `followStatusDtoSchema` | object | `following`, `followerCount` |
| `notificationTypeSchema` | enum | `event.new_from_followed` |
| `notificationDtoSchema` | object | `id`, `type`, `title`, `body`, `link`~, `subjectId`, `createdAt`, `readAt`~ |
| `listNotificationsQuerySchema` | object | `cursor`?, `limit`=, `unreadOnly`? |
| `notificationListResponseSchema` | object | `items`, `pageInfo`, `nextCursor`~ |
| `unreadCountDtoSchema` | object | `count` |
| `markNotificationsReadSchema` | object | `ids` |
| `markReadResultDtoSchema` | object | `updated` |
