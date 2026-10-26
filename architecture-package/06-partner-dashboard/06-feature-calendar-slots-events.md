# 06. Calendar, slot requests and events

## Venue calendar
Code: FE `lib/calendar/venue-calendar-repository.ts`; BE `routes/v2/partner/venues.ts`, `VenueCalendarService` in `packages/core/src/application/venues/venue-service.ts`.
- Loads the org's active venues. **If none exist it auto-provisions one** (`POST /organizations/:id/venues`, name/slug from the org, stable idempotency key `venue-provision-<orgId>`): a read-only page can write. (Backend `VenueService.list` also auto-creates a venue named after the org when none exists and the actor or any member has `venue` capability.)
- Slots: `GET /venues/:id/calendar?from&to`; events: `GET /organizations/:id/events` filtered to the venue (limit 100, no paging). Window: 1 month back, 12 forward. Events in `draft`/`review` show as `pending`.
- **Block date:** `POST /venues/:id/calendar/blocks` (`venue.manage`, `Idempotency-Key` per intent). `venueBlockTimeRange` (`venue-calendar-repository.ts:89-90`) builds `new Date(`${date}T${from}:00.000Z`)`: **entered local times are treated as UTC.** Overnight ranges roll to the next day.
- Backend: only `blocked` slots are ever created. `createBlockIfFree` is a Firestore transaction with a half-open `[start,end)` overlap guard (concurrent blocks: exactly one wins). Unblock soft-cancels (status `cancelled`) and only works on `blocked` slots. Reads cap at **2000 slots** and filter in code; the window filter is `startTime >= from && startTime <= to`, so a block that starts before `from` but overlaps the window is omitted.
- **Availability** (`GET /venues/:id/availability`, cached 30 s) is derived: `fullyBooked = live.length > 0 && openSlots === 0`. Because **nothing ever creates `open` or `booked` slots**, a venue with any block always reports `fullyBooked: true`. A host partner sees redacted labels on non-open slots.
- `GET /venues/:venueId/calendar` is owner-only; a partner host reads availability instead.

## Host availability
Active partnerships where `hostOrganizationId === org`, then per venue `/venues/:id` and `/venues/:id/availability` (`host-availability-repository.test.ts`).

## Slot requests
FSM (`SLOT_REQUEST_TRANSITIONS`, `domain/models/venue.ts:328`, verified): `pending → accepted | rejected`; `accepted → cancelled`; rejected and cancelled are terminal. Same-state throws `InvalidOperationError` (400).
- Host creates `POST /venues/:venueId/slot-requests` (`slot-request.create`, body `{eventId?, message?}`, `hostId = actor.organizationId`). **No validation** that the venue exists/is active, that an active partnership exists, or that `eventId` belongs to the host.
- Venue accepts/rejects `POST /venues/slot-requests/:id/{accept,reject}` (`venue.schedule`), scoped to the venue's owning org (404 otherwise). **Accept writes nothing to the calendar, changes nothing on the event and notifies no one.**
- Host cancel `POST /venues/slot-requests/:id/cancel` works only from `accepted`; **a still-pending request cannot be withdrawn** (asserted by `venues.test.ts:~1022`), although the dashboard exposes cancel. The doc comment on `cancel` says pending/accepted → cancelled; the table disagrees.
- Reads: venue `GET /venues/:v/slot-requests` (+ detail), host `GET /organizations/:org/slot-requests`. No `slot_request.*` domain event exists, although the notification model has a `slot_request` action type.
- Dashboard: BFF composes reads (see `02-…`); the host editor's "Submit for approval" creates a draft event then the slot request (`${key}-event`, `${key}-slot-request`). UI maps wire `accepted` to `approved`; 404/409 error map in the review panel.

## Event create, edit, publish
### Dashboard flow
`events/create` (`VenueEventEditorRoute`): venue steps `basics → promoters → review`; host `venue → basics → promoters → review`; step and `venue/date/slot` live in the URL. Validation is hand-written (`validateDraft`, `validateCompensation`); the zod `venue-event-schema.ts` is used only by legacy code. Connected promoters show as `"Promoter <last 8 chars of id>"` (names are not fetched).
### Publish workflow (`publishVenueEvent`, `lib/events/venue-event-repository.ts`, browser-side, **not atomic**)
1. Poster: `POST /organizations/:id/poster/upload-url` then a **browser PUT** to the signed URL (CSP allows `storage.googleapis.com`); max 5 MB.
2. `POST /organizations/:id/events` (compensation included for paid tiers + promoters).
3. `POST /events/:id/ticket-tiers` once per tier, sequentially (RSVP tiers no price; pricing phases from `DD-MM` entries, end-exclusive).
4. If compensation is `custom`: `PATCH /events/:id` with `If-Match` to remap local tier ids to server ids.
5. `POST /events/:id/promoter-assignments` per promoter.
6. `POST /events/:id/review`, then `POST /events/:id/publish`.
Each call uses `Idempotency-Key = <workflowId>-<suffix>`; the workflow id is minted once per intent in a ref and cleared on success. A mid-way failure leaves a partial draft (tiers, no publish); a retry replays idempotently but there is **no cleanup**. On success it routes to `/partner/venue/calendar?month&date`.
### Edit
`events/[eventId]/edit` loads fixture data and never passes `onSubmit`: **non-functional**. Backend `PATCH /events/:id` (strict: title, summary, description, imageUrl, startAt, endAt, tags, startingPricePaise, isFree, compensation; `If-Match` required) works but is not wired. Not settable via API: `capacity` (model field only), `venueId`, `slug`. `eventDtoSchema` omits `capacity` and `adminOverride`.
### Event list/detail
Fixture: real events published through the editor **do not appear in `/events`**, and `/events/<realId>` 404s.

## Event model and FSM (`domain/models/event.ts`, `event-service.ts`)
Fields: `id, organizationId, venueId|null, slug, title, summary, description, imageUrl, startAt, endAt, status, isPublic, tags[], startingPricePaise, isFree, cancellationReason, compensation?, capacity|null, adminOverride, version, createdAt, updatedAt`.
| From | To |
|---|---|
| draft | review, cancelled |
| review | scheduled, draft, cancelled |
| scheduled | published, started, cancelled |
| published | sales_paused, started, cancelled |
| sales_paused | published, started, cancelled |
| started | ended |
| ended | archived |
| archived, cancelled | terminal |
Same-state is a no-op; terminal events cannot be updated (`assertEventMutable`); `isPublic` true for published, sales_paused, started.
- **Publish rules:** `draft → published` is rejected (409; review is not skippable). From `review`, `scheduled` is transient and only `published` persists (one version bump). `validateCompensationForPublish` over compensation-eligible tiers (non-RSVP and `commissionEligible !== false`): `standard` needs global rate 0–100; `salary` needs positive integer amount + period; `custom` needs an integer 0–100 rate for every eligible tier. Then it refreshes `startingPricePaise` (minimum paid tier price, else 0) and `isFree`. **Publish does not check** that a venue exists, that any tier exists, that `startAt` is in the future, or that a poster is set.
- **Lifecycle gap (verified in source):** within `event-service.ts` there is no path to `started`, `ended` or `archived`; the only reference is the admin force-complete in `application/admin/admin-ops-service.ts` (`adminForceCompleteEvent`, sets `adminOverride`), and I found no scheduler calling it. Finance and analytics treat `ended`/`archived` as "complete" (`finance-service.ts:~304`). Also `started → cancelled` is not in the table, so `cancel` fails once started.
- `review` only goes `draft → review` (no-op from review); there is no route for `review → draft`. `duplicate` copies title, description, tags, compensation, start/end and venue as a new draft (same title and `startAt`; tiers not copied). `resume-sales` re-emits `event.published`, re-triggering notification and fan-out consumers (dedupe key `event_published:<id>`).
- `create`: `assertVenueAccess` allows the owner, or a host with an active partnership on an active venue; an unknown venue id silently passes (legacy drafts); a suspended venue or foreign non-partner venue gives 404.
- Events emitted: `event.created` (create, duplicate), `event.updated` (update, pause), `event.published` (publish, resume), `event.cancelled`. `review` emits nothing.
- Poster URL: key `posters/<org>/<uuid>.<ext>`, public-read, 5 MB, 15-minute TTL.

## Ticket tiers, promo codes, table packages
See `07-…` and `10-…` for fields and invariants. Only **create** and **list** routes exist (no update, pause, delete for tiers; no update/deactivate for promo codes; no update for table packages), even though a domain `updateTicketTier` exists.
