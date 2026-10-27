# 09. Backend API reference (partner-facing)

All paths under `/api/v2` (no `/partner` prefix). Registered in `apps/api-gateway/src/routes/v2/route-manifest.ts` (order: organizations, venues, events, event-catalog, partnerships, discover, analytics, referral-links, promoter-connections, then onboarding, door, finance, notifications). Legend: **H** = `x-organization-id` header required and must equal the path org; **IK** = `Idempotency-Key` required; **IM** = `If-Match` (positive integer version) required. Rate classes: PUBLIC_READ 120, AUTH_READ 240, STANDARD_COMMAND 60, SENSITIVE_COMMAND 10, OTP_SEND 5, OTP_VERIFY 10, SCANNER_COMMAND 300 (per minute).

## Cross-cutting machinery
- **Actor + RBAC + ABAC:** see `04-…`. On the memory driver `actorFromRequest` (`lib/v2-services.ts`) fabricates an actor from headers (non-production only).
- **Cache** (`plugins/cache.ts`): in-memory `Map` keyed by class, org, user, URL; only 200s; **no invalidation on writes**; per process. TTLs: ORGANIZATION and VENUE_PROFILE 5 min, EVENT_PREVIEW 60 s, AVAILABILITY 30 s. Used on `GET /organizations/:id`, all `analytics/*`, `GET /venues/:id/menu`, `/availability`, `GET /events/:id/analytics`. A venue menu read can be stale up to 5 min after `PUT /menu`; an org read up to 5 min after PATCH.
- **Idempotency** (`lib/v2-idempotency.ts` `runIdempotent`): hash of method + URL + path params + body; replay returns the stored response; same key with a different body → 409; in-flight duplicate → 409; TTL 24 h; a failed attempt is kept as `FAILED`, so a retry returns 409 `idempotency_failed`. If no key is passed to `runIdempotent` it simply runs. Door-code create does not validate the header (effectively optional); payout request uses `body.idempotencyKey`; `POST/DELETE bank-accounts`, `default` and all dispute writes have no idempotency at all; invitation `revoke` and assignment `end` validate the header but do not enforce it.
- **Concurrency:** `If-Match` required on `PATCH /organizations/:id`, `PATCH /events/:id` and the venue PATCH/PUT routes. Lifecycle actions accept `If-Match` but ignore it; the FSM and repository compare-and-set return 409 `version_conflict` instead.
- **Error envelope:** `{code, message, status, requestId, fieldErrors?, details?}`; zod failures → 422 `validation` with `fieldErrors`; lists `{items, pageInfo:{page,pageSize,total,hasNextPage}}`; success bodies are bare DTOs.
| Domain code | HTTP | Envelope code |
|---|---|---|
| `*_not_found`, `not_found` | 404 | not_found |
| `unauthorized` | 401 | unauthorized |
| `forbidden` | 403 (or 404 with `hideForbidden`) | forbidden / not_found |
| `device_not_authorized` | 403 | forbidden |
| `version_conflict` | 409 (details: expected and current version) | conflict |
| `conflict`, `payment_refunded`, `idempotency_*`, `state_transition` | 409 | conflict |
| `invalid_operation` | 400 | validation |
| anything else | 500 | server (logged as `unmapped_domain_error`) |
`mapDomainError` lives in `partner/events.ts` (~l.495–695); `finance-routes.ts` has its own copy that does not handle `state_transition` or `conflict`.
- **Domain events** (`lib/v2-services.ts`): in-process bus over a durable outbox, drained every 1 s on Firestore, at-least-once. `event.published`/`event.updated` → audit consumer (writes to a `MemoryAuditRepository`: **not durable even on Firestore**); `event.published` → follower fan-out + notifications; `promoter_connection.requested`, `partnership.requested` → notifications; `event.created|updated|published|cancelled` → analytics projection.
- **Background jobs** (Firestore driver): every 5 min sweep expired cart reservations and scanner sessions; every 5 min refresh the analytics read model for every org and event.

## Organizations, members, invitations (`partner/organizations.ts`)
| Method + path | Pre-handlers | Notes |
|---|---|---|
| `GET /organizations` | AUTH_READ, `organization.read` (header optional) | active memberships; paginated |
| `GET /organizations/:org` | AUTH_READ, `organization.read`, cached ORGANIZATION | forbidden hidden as 404; DTO omits settings and `platformFeePercent` |
| `POST /organizations` | STANDARD_COMMAND, no permission (first-org bootstrap), IK | owner gets all 3 capabilities; emits `organization.created`; body accepts `settings` though the contract schema does not list it |
| `PATCH /organizations/:org` | STANDARD_COMMAND, `organization.update`, IK, IM | strict body; emits `organization.updated` |
| `GET\|POST …/members`, `PATCH\|DELETE …/members/:userId` | `staff.manage` for writes; `organization.read` list | POST adds an existing user by id (no event); role in admin/manager/member; owner not removable/demotable |
| `GET /organizations/:org/access` | AUTH_READ, `organization.read` (not cached) | see `04-…` |
| `GET\|POST …/invitations` | `staff.manage`; POST IK | `invitation.create` idempotency |
| `POST /invitations/:id/revoke` | `staff.manage` | key validated, not enforced |
| `POST /invitations/:id/accept` | STANDARD_COMMAND, no permission, no org header | no email check |
| `GET /auth/invitations/mine` | AUTH_READ | invitations for the session email |

## Venues, calendar, slot requests (`partner/venues.ts`)
| Method + path | Permission | Notes |
|---|---|---|
| `GET /organizations/:org/venues` | `venue.read` + path-org guard | paginated |
| `POST /organizations/:org/venues` | `venue.create`, IK | emits `venue.created`; body not strict |
| `GET /venues/:id` | `venue.read` | owner or active host partner |
| `PATCH /venues/:id` | `venue.manage`, IK, IM | strict; shallow merge of `public`/`private` (sending `address` replaces the whole address) |
| `GET\|PATCH /venues/:id/profile` | GET: **no rate limit, no requirePermission**; PATCH `venue.manage`, IK, IM | owner only |
| `GET /venues/:id/calendar` | `venue.read`; `from`,`to` ISO | owner only |
| `POST /venues/:id/calendar/blocks`, `DELETE …/blocks/:blockId` | `venue.manage`, IK | transactional overlap guard; soft-cancel |
| `GET\|PUT /venues/:id/menu` | `venue.read` cached 5 min / `venue.manage` IK IM | wholesale replace; negative prices rejected |
| `GET /venues/:id/availability` | `venue.read`, cached 30 s | derived |
| `GET /venues/:id/slot-requests` (+ `/:slotRequestId`), `GET /organizations/:org/slot-requests` | `venue.read` / `slot-request.list` | owner-scoped / host-scoped |
| `POST /venues/:id/slot-requests` | `slot-request.create`, IK | see `06-…` |
| `POST /venues/slot-requests/:id/{accept,reject}` | `venue.schedule`, IK | |
| `POST /venues/slot-requests/:id/cancel` | `slot-request.create`, IK | host only; only from accepted |

## Events (`partner/events.ts`)
`GET /promoters/:promoterId/events` (`event.read`; active assignments only); `GET /organizations/:org/events` (`event.read`); `GET /events/:id`; `POST /organizations/:org/events` (`event.create`, IK, `createEventSchema.strict()`, emits `event.created`); `POST /organizations/:org/poster/upload-url` (`event.create`, H only, **no IK by design**; 5 MB; 15-min TTL; public-read); `PATCH /events/:id` (`event.update`, IK, IM); `GET /events/:id/previews`; lifecycle `POST /events/:id/{review (event.update), publish (event.publish), pause-sales, resume-sales (event.update), duplicate (event.create, 201), cancel (event.cancel, body {reason})}` registered in a loop; idempotency commands `event.<action>`.

## Event catalog and referral links
`GET|POST /events/:id/{ticket-tiers, promo-codes, table-packages, promoter-assignments}` (reads `event.read`, writes `event.update`, IK; commands `event-ticket-tiers.create`, `event-promo-codes.create`, `event-table-packages.create`, `event-promoter-assignments.create`); `POST /promoter-assignments/:id/end` (`event.update`); referral links `POST /promoter-assignments/:id/referral-links`, `GET|POST /events/:id/referral-links`, `POST /referral-links/:id/deactivate`.

## Partnerships, connections, discovery
`GET /organizations/:org/partnerships`; `POST /partnerships` (`venue.manage`); `POST /partnerships/:id/{approve,reject}` (`organization.read`), `{block,end,venue-share}` (`venue.manage`); `GET /organizations/:org/promoter-connections`; `POST /promoter-connections` (`venue.manage`); `POST /promoter-connections/:id/{approve,reject}` (`organization.read`), `{block,revoke}` (`venue.manage`); `GET /organizations/:org/discover-partners` (`organization.read`; `type,q,cursor,limit≤100`).

## Analytics, notifications, onboarding, finance, door, support
Covered in `08-…` (analytics, notifications, finance, support) and `05-…` (onboarding). Door (dashboard-facing subset): door codes `POST|GET /events/:id/door-codes` (`door.manage`; IK optional; response contains the code credential), `POST /door-codes/:id/revoke` (SENSITIVE_COMMAND), `GET /door-codes/:id/sessions`, `POST /door/sessions/:id/revoke`; `venue.read`: `GET /door/events|guests|attendance-report`; `door.manage`: `/door/devices` (+unbind, reauthorize) and cover-wallet credit/freeze/unfreeze/terminate/reconcile; `ticket.override`: `/door/guests/check-in`, `/door/walk-in`, `/door/dine-in`, `/cover-wallets/:id/debit`, `/door/override`; `GET /door/sales` (`venue.read`); `GET /door/stats` and SSE `/door/stats/stream` (AUTH_READ, no requirePermission; heartbeat 15 s, max 15 min). The **dashboard Door screen is fixture** and calls none of these; the scanner app is the real client.
