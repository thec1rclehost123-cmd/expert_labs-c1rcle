# 09. Flow: support desk

Code: FE `app/support/page.tsx` (1,118 lines); BE `routes/v2/admin/support.ts`, `routes/v2/support/intake-routes.ts` (guest side), `packages/core/src/domain/models/support-ticket.ts`, `application/support/*`. Guest intake: guest-portal `/help` with BFF `app/api/help/*` (`gateway-proxy.ts`). Status: **Live** (shipped 2026-09-20, `cec80b7`). Safety reports/moderation: **Paused** (`SafetyService`/`AdminSafetyService` exist in core; no `/safety` route is registered; indexes exist; contract `adminAlertCategoryKeySchema` has no handler).

## Aggregate (`SupportTicket`, `v2_support_tickets`)
`id, subject, description, category (account|billing|order|event|technical|other), status, priority (low|medium|high|urgent), requester {userId, email, organizationId}, assignee {userId, name}|null, messages[], internalNotes[], timeline[], links {venueId, eventId, orderId, organizationId, userId}, sla {responseDueAt, resolutionDueAt, responseBreachedAt, resolutionBreachedAt}, mergedInto, mergedFrom[]`, resolved/closed/deleted attribution. Soft delete is an overlay (`deletedAt/deletedBy`), not a status.

## Status FSM and SLA
`open → in_progress | waiting_on_customer`; `in_progress → waiting_on_customer | escalated | resolved`; `waiting_on_customer → in_progress`; `resolved → closed`; `closed → open` (requester follow-up or admin reopen).
SLA hours (response / resolution): urgent 1/4, high 4/24, medium 24/72, low 72/168. Breach stamps (`*BreachedAt`) are set once; first breach wins. A priority change resets the SLA from now; reopen gives a fresh SLA.
Rules: `assign` moves `open → in_progress`; refused for terminal or deleted tickets. `reply` sets `waiting_on_customer` (refused for terminal/deleted); a customer reply moves it back to `in_progress`. `escalate` refused for terminal/deleted. `resolve` requires a reason. `reopen` only from a terminal state, refused for deleted or merged tickets. `merge`: the duplicate becomes `closed` with `mergedInto`; its messages are copied into the primary, time-sorted, prefixed `[Merged from ticket …]`. `note` allowed in any non-deleted state.

## Admin routes (all `SENSITIVE_COMMAND`, idempotent, authority **`requireAdmin` only = TIER1, any role**)
Reads: `GET /admin/support/tickets` (filters `status|priority|category|assigneeUserId|requesterUserId|search|includeDeleted`), `GET /admin/support/tickets/:ticketId`. Mutations: `POST …/{assign, priority, reply, notes, link, resolve, merge, escalate, close, reopen, restore}` and `DELETE /admin/support/tickets/:ticketId` (soft). Command names `admin.support.<verb>`. Audit actions `SUPPORT_ASSIGN|REPLY|NOTE|PRIORITY|LINK|ESCALATE|RESOLVE|CLOSE|REOPEN|MERGE|DELETE|RESTORE` (free strings, not `AdminAction` entries). Pattern: `requireAdmin`, load, pure domain function, save, `authority.record` with before/after JSON, IP and user agent; wrapped in `runIdempotent`.

## Guest intake (requester side)
Routes `POST /support/tickets`, `GET /support/tickets`, `GET /support/tickets/:id`, `POST /support/tickets/:id/messages` (`intake-routes.ts`). The requester is derived from the session, never from client input; the guest view never exposes `internalNotes`; `requester.email` stays null unless the session verifies it. The guest-portal BFF checks same-origin, forwards the session cookie, mints the `Idempotency-Key` server-side and calls the gateway. Partner dashboards use the same routes (see `../06-partner-dashboard/08-…`). Full guest-origin login wiring is deferred.

## Console behaviour
Queue (filters, SLA columns) + detail pane (timeline, SLA, merged-into/from, conversation). Actions: reply, internal note, assign (picker over `listAdmins(100)`), change priority, link order/event/venue/user id, resolve with reason, merge a duplicate by id, escalate, close, reopen, restore, soft-delete. Delete, merge, close and escalate have **no confirm**; all mutation errors share one generic banner; search is not debounced (a request per keystroke); draft state resets on ticket switch (`selectTicket`).

## Weaknesses (verified or reported)
- `assign` accepts an arbitrary `userId` and `name` from the caller: it does not verify the user is an admin and the display name is client-supplied.
- `link` does not verify the linked entity exists; `merge` saves primary then duplicate non-atomically.
- **Firestore list applies only one equality filter** (status, else priority, else category) at query time; assignee, requester, search and `includeDeleted` are filtered **after pagination** in code, so filtered pages can be short and `total` counts the unfiltered set.
- No outbound email to the requester on reply (none found); audit `reason` for replies/notes is the first 80 characters of the content.
- Indexes: `status`, `priority`, `category` (each with `createdAt desc`) and `requesterUserId+deletedAt+createdAt`; `listByMergedInto` (`mergedInto+deletedAt`) is equality-only and needs none.
- Docs say support has 11 routes; the code has 15 (escalate, close, reopen, restore added).
