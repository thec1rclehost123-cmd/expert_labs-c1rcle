# 06. Backend: public and guest-side API

All paths under `/api/v2`. Full generated index: `../13-reference/API-ROUTE-INDEX.md`. Guest actors are session-only (`role: member`, empty organisation) so org-scoped permissions do not apply; guest routes authenticate with the Better Auth session and scope every read to the buyer (404 for strangers).

| Area | Endpoints | Rate | Notes |
|---|---|---|---|
| Public discovery | see `03-…` (`/public/*`) | PUBLIC_READ 120/min | unauthenticated; no cache headers |
| Auth | `POST /auth/signup|login|forgot-password|reset-password|refresh|logout`, `GET /auth/session`, `POST /auth/otp/send|verify` | SENSITIVE_COMMAND 10, OTP_SEND 5, OTP_VERIFY 10 | signup hard-codes role `partner`; `toUserDto` always returns `partner`; OTP generic acks |
| Checkout | `POST /checkout/quote|holds`, `POST /rsvp`, `POST /payments/attempts`, `POST /payments/:id/verify`, `GET /payments/attempts/:holdId/status` | STANDARD_COMMAND 60 (status AUTH_READ) | `../10-payments-and-finance/02-…` |
| Orders and wallet | `GET /orders`, `/orders/:id`, `/orders/:id/status`, `/wallet`, `/wallet/tickets`, `/wallet/orders`, `GET /tickets/:id` | AUTH_READ 240 | IDOR-safe 404; transfer, claim, cancel-transfer not registered |
| Ticket QR | `GET /tickets/:ticketId/qr` | AUTH_READ | holder or same-org staff; `no-store` |
| Profile | `GET|PUT /profile/me` | AUTH_READ / STANDARD_COMMAND | upsert `{displayName, dateOfBirth, city, tastes, intents}` (18+ rule in the model) |
| Social | `POST /follows`, `DELETE /follows/:targetType/:targetId`, `GET /follows/me`, `GET /follows/:targetType/:targetId/status` | STANDARD / AUTH_READ | targets `venue|host`; idempotent follow |
| Guest notifications | `GET /notifications/me`, `GET /notifications/me/unread-count`, `POST /notifications/me/read|read-all` | AUTH_READ / STANDARD | type `event.new_from_followed` (bus subscriber) |
| Support intake | `POST|GET /support/tickets`, `GET /support/tickets/:id`, `POST /support/tickets/:id/messages` | SENSITIVE_COMMAND 10 for writes | requester from the session; **DTO leaks internal notes** |
| Cover-wallet QR | `GET /cover-wallets/:walletId/qr` | AUTH_READ | owner only; guest cannot list wallets |

Backend findings for the guest surface (verified or reported): `toUserDto` always `partner` and no guest role exists in practice; the support DTO leaks `internalNotes`, `assignee`, `timeline`, `sla` to the requester; the referral attribution HMAC falls back to `'dev-secret'` when `magicTicketSecret` is empty; the promoter click endpoint is anonymous with no dedupe (limited by 120/min/IP); the public tiers response is open and uncached; public discovery is capped at 20 and unsorted by editorial rules; there is no public venue or host list.
Rate-limit keying caveat: the key is `class|ip|userId|orgId`; the portal's BFF does not forward the visitor's IP (see `02-…`), so anonymous BFF auth calls share **one bucket per egress IP** at 10/min and server-side public reads share 120/min per egress IP; a handful of users, or an attacker, can lock out logins for everyone.
