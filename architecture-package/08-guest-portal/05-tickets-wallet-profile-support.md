# 05. Tickets, wallet, profile, follows, notifications and help

## Tickets and the entry QR
- `/tickets` (signed in): `TicketsClient` GETs `/api/wallet/tickets?limit=50`, then for each distinct event resolves event and venue **from the browser** with `apiClient` (a cross-origin call to `NEXT_PUBLIC_API_BASE_URL` sending the in-memory Bearer token to the public routes). Entitlements are grouped by `orderId::tierId` into one card with `ticketCount`; statuses: all `void` → cancelled, all `redeemed` → used, otherwise active; past = used, cancelled or ended. Only the first 50 entitlements are fetched and there is no pagination UI.
- **QR** (`TicketDetailModal.tsx`): for active tickets it GETs `/api/tickets/:firstEntitlementId/qr` (gateway `door/scanner-routes.ts` ≈L414) returning `{qrPayload, expiresAt, refreshIntervalSec}`. Payload `ticketId:windowStart:HMAC`, 30 s window (`MAGIC_WINDOW_SEC`); only the holder or same-organisation staff get it, anyone else 404. The UI refetches every `refreshIntervalSec` ("Refreshes every 30s"); failure shows "Entry code unavailable" with Retry; `used`, `cancelled` and `pending` show a "No entry code" message; rendered with `qrcode.react` (level M).
- **Limitation:** a multi-quantity card shows "TICKET 1 OF n" but only the **first** entitlement's QR: tickets 2..n are unreachable (no per-ticket pager).
- **Offline:** none. No service worker, no cached pass; a failed fetch shows the unavailable message. The rotating 30 s HMAC makes a cached QR useless anyway. (Note: the backend accepts a bare ticket id as a credential; see `../09-scanner-app/05-…`.)

## Cover wallet
Backend: a guest can mint a rotating tab QR via `GET /api/v2/cover-wallets/:walletId/qr` (`cover-wallet-routes.ts:184-212`, `AUTH_READ` + `venue.read`, which the guest `member` role holds); the service allows **only the wallet's own `userId`** (staff get 404); format `cw:<walletId>:<window>:<hmac>`, 30 s window, `no-store`. There is **no guest "my wallets" endpoint**, so a guest cannot discover a wallet id, and the portal has **no cover-wallet UI and no BFF**: a guest cannot show a wallet QR from the portal.

## Ticket transfer and refunds
Transfer, claim and cancel-transfer are **absent end to end** (the backend deliberately has none: `ticket-routes.ts` exposes only `GET /tickets/:id`). There is no guest-initiated refund; the guest sees only the auto-refund outcome message and the recovery message "The previous payment was refunded. No tickets were issued." An `order.refunded` status is not surfaced in the wallet.

## Profile
`/profile` (`ProfileClient`) reads `/api/profile/me` (404 = no profile yet, falling back to the session identity) plus the wallet to build upcoming and attended events; sections via `?view=`, `?filter=`, `?section=`. Fields without endpoints (phone, instagram, gender, sign-in method) are omitted; the badge is hard-coded "Member". **Blocked in production** (`PrivateDataUnavailable`).
**Editing is UI-only:** `PersonalProfileFormClient` validates and tracks changes but has **no save button, no submit handler and no network call**. The gateway `PUT /api/v2/profile/me` (`profile.ts:52-80`, upsert `{displayName, dateOfBirth, city, tastes, intents}`) is unused and there is no BFF write route; the form also offers fields the gateway does not store. The Account and security tab has no change-password UI wired.

## Follows and guest notifications
Backend routes exist (`social/social-routes.ts`): `POST /follows`, `DELETE /follows/:targetType/:targetId` (`venue|host`), `GET /follows/me`, `GET /follows/:targetType/:targetId/status`; guest notifications `GET /notifications/me`, `/unread-count`, `POST /read`, `/read-all`; the only notification type is `event.new_from_followed`. **No portal UI or BFF for either** (the directory "follow" buttons are fixture and disabled). `routes/v2/notifications/notifications-routes.ts` is the **partner organisation** inbox, a different system.

## Help and support intake
BFF `/api/help/tickets*` maps to the gateway `support/intake-routes.ts`: `POST` (`SENSITIVE_COMMAND`, Idempotency-Key required, body `{subject 3-200, description 10-5000, category account|billing|order|event|technical|other, priority default medium, email?, organizationId?}`, 201), list, detail, `POST /:ticketId/messages` (content 1–5000); everything is scoped to `requester.userId` (404 for others). UI `HelpClient` (create, list, detail, reply) works in preview and dev; **the page is blocked in production**.
**Security (verified):** `toSupportTicketDto` (`intake-routes.ts` ≈L45, 70) returns **`internalNotes`**, `assignee`, `timeline`, `sla` and the merge and delete audit fields to the requester, and `SupportService.getMyTicket`/`sendMessage` return the raw aggregate with no redaction (the contract schema includes them, `phase7.ts:116`). Internal admin notes therefore reach the guest.

## Legal
`/privacy` and `/terms` are title-only stubs (`indexable:false`): no policy text, no cookie banner, no consent UI.

## Backend guest reads not wired into the portal
`GET /orders` (list), `/orders/:id/status`, `/wallet` (summary), `/wallet/orders`; all are `AUTH_READ` 240/min and return 404 for strangers (IDOR-safe). Only `/orders/:id` and `/wallet/tickets` are used.
