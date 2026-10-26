# 09. Backend API reference (door and scanner)

All paths under `/api/v2`. Route registration order (`route-manifest.ts` ≈L165-170): door codes, door ops, door sales, cover wallets, scanner routes, phase-5 (stats). Contracts: `packages/contracts/src/contracts/phase5.ts`. Rate classes per minute: `PUBLIC_READ` 120, `AUTH_READ` 240, `STANDARD_COMMAND` 60, `SENSITIVE_COMMAND` 10, `SCANNER_COMMAND` 300; bucket key `class|ip|userId|orgId` (memory by default, Redis if `RATE_LIMIT_STORE=redis`).

## Cross-cutting
- **Auth:** global `onRequest` hook resolves the Better Auth session (cookie or Bearer) and, with a valid `X-Organization-Id` the user belongs to, builds `request.actor`; otherwise a session-only actor with `organizationId: ''` and role `member`; no session → 401. The scanner token comes from `x-scanner-session-token` (`sessionTokenFrom()`), is redacted in logs (`logger-config.ts`) and allowed by CORS. `If-Match` is allowed by CORS but **no door route uses it**.
- **Idempotency:** `runIdempotent` (key `actor:command:key`, 24 h, hash of method + URL + path + body); with no key it simply runs. The scanner token is **not** part of the hash. Several routes make the header optional.
- **Errors:** scanner, door-ops and door-code routes use `mapDomainError` from `partner/events.ts` (not-found 404, unauthorized 401, `device_not_authorized` 403, forbidden 404 or 403, conflict/idempotency/state_transition/payment_refunded 409, invalid_operation 400, unmapped 500). `door-sale-routes.ts` and `cover-wallet-routes.ts` each carry **their own copy** that does not handle `device_not_authorized`, `conflict` or `payment_refunded`.
- **Dead contract items:** `phase5ErrorCodeSchema` codes (`SCANNER_SESSION_EXPIRED`, `VELOCITY_LIMIT_EXCEEDED`, …) are never emitted (generic codes are used); `magicQrRequestSchema` is unused; `sessionType` is stored but unused.

## Door codes (manager) , `event-code-routes.ts`
| Method + path | Rate | Perm | Idempotency | Schemas → service |
|---|---|---|---|---|
| `POST /events/:eventId/door-codes` | STANDARD | `door.manage` | optional header `door.code.create` | `eventCodeCreateBodySchema` → `eventCodeDtoSchema` (201) → `createEventCode` |
| `GET /events/:eventId/door-codes` | AUTH_READ | `door.manage` | none | `eventCodeListResponseSchema` → `listEventCodes` (active only, raw codes) |
| `POST /door-codes/:codeId/revoke` | SENSITIVE | `door.manage` | none | `revokeReasonBodySchema` → `revokeEventCode` (also revokes sessions) |
| `GET /door-codes/:codeId/sessions` | AUTH_READ | `door.manage` | none | `listSessionsForCode` |
| `POST /door/sessions/:sessionId/revoke` | SENSITIVE | `door.manage` | none | `revokeSession` |

## Scanner routes , `scanner-routes.ts`
| Method + path | Rate | Auth | Idempotency | Notes |
|---|---|---|---|---|
| `POST /door/sessions` | SENSITIVE | actor only, **no permission, no token** | optional `door.session.open` | `startShift` = `bindDevice` + `openSession`; 201; 404 wrong/foreign code, 400 revoked/expired/max devices, 403 geofence or unbound device |
| `GET /door/sessions/:sessionId` | AUTH_READ | actor + org check, no permission | none | `sessionToken` always null |
| `POST /door/check-ins` | SCANNER | actor + token | optional `door.check-in` | always 200: `consumed`, `denied`, `confirmation_required` |
| `POST /door/check-ins/verify`, `POST /door/lookup` | SCANNER | actor + token | none | non-mutating; `ticketLookupResponseSchema` (`valid|invalid`) |
| `GET /door/check-ins/:checkInId` | AUTH_READ | actor + org check, no permission | none | includes guest name |
| `POST /door/override` | SENSITIVE | `ticket.override`, no token, no device check | none | 409 if the scan is not `denied` |
| `GET /door/offline-manifest` | STANDARD | actor + token | none | route comment about sync verification is false on this checkout |
| `POST /door/offline-sync` | STANDARD | actor + token | optional `door.offline-sync` | 1–500 scans |
| `GET /tickets/:ticketId/qr` | AUTH_READ | actor only | none | holder or any org member; `no-store` |

## Door ops , `door-ops-routes.ts`
| Method + path | Rate | Auth | Idempotency |
|---|---|---|---|
| `GET /door/events` | AUTH_READ | `venue.read` | none (query `date`, IST) |
| `POST /door/devices` | STANDARD | **`door.manage`** | none |
| `GET /door/devices` | AUTH_READ | `door.manage` | none |
| `POST /door/devices/:id/unbind` | SENSITIVE | `door.manage` | none |
| `POST /door/devices/:id/reauthorize` | SENSITIVE | `door.manage` | none |
| `POST /door/heartbeat` | SCANNER | actor + token | none |
| `POST /door/check-ins/confirm` | SCANNER | actor + token | optional `door.check-in.confirm` |
| `POST /door/staff-deny` | SCANNER | actor + token (bound device) | none (201) |
| `GET /door/guests`, `GET /door/attendance-report` | AUTH_READ | `venue.read` | none (`no-store`) |
| `POST /door/guests/check-in` | STANDARD | `ticket.override` | optional `door.guest.check-in` (201) |
| `POST /door/wallet-qr` | SCANNER | actor + token (`charge` session) | none (`no-store`) |
| `POST /door/wallet-charge` | SCANNER | actor + token (`charge` session) | body key `door.wallet.charge` (`no-store`) |
| `POST /door/ticket-sale` | STANDARD | actor + token (`full` session) | body key `door.ticket-sale` (201, or 200 `replayed:true`) |

## Door sales , `door-sale-routes.ts` (all require `X-Organization-Id`)
`POST /door/walk-in`, `POST /door/dine-in` (STANDARD, `ticket.override`, body key `door.walkin` / `door.dinein`, strict schema, 201, **no scanner token**); `GET /door/sales` (AUTH_READ, `venue.read`; up to 1000 rows filtered in memory).

## Stats , `phase5-routes.ts`
`GET /door/stats` and SSE `GET /door/stats/stream` (AUTH_READ; organisation check in the service; no permission, no token). See `07-…`.

## Cover wallets , `cover-wallet-routes.ts`
See the table in `06-…` (`door.manage` for issue/credit/freeze/unfreeze/terminate/reconcile; `venue.read` for GETs; `ticket.override` for debit; no scanner token anywhere).

## Guest and wallet read routes
`GET /tickets/:id` (own ticket), `GET /wallet`, `/wallet/tickets`, `/wallet/orders` (read projections). Transfer, claim and cancel-transfer **are not registered** (design decision pending).

## Frontend function map (`scannerApiClient.ts`)
`login` → `POST /auth/login`; `fetchMyOrganizations` → `GET /organizations`; `fetchTodaysEvents` → `GET /door/events?date=today`; `registerDevice` → `POST /door/devices`; `redeemDoorCode` → `POST /door/sessions`; `checkIn` → `POST /door/check-ins`; `lookupTicket` → `POST /door/lookup`; `confirmCheckIn` → `POST /door/check-ins/confirm`; `fetchStats` → `GET /door/stats`; `openStatsStream` → SSE; `fetchGuests` → `GET /door/guests`; `manualCheckIn` → `POST /door/guests/check-in`; `staffDeny` → `POST /door/staff-deny`; `overrideCheckIn` → `POST /door/override`; `submitWalkIn|submitDineIn` → `POST /door/walk-in|dine-in`; `submitTicketSale` → `POST /door/ticket-sale`; `resolveWalletQr` → `POST /door/wallet-qr`; `chargeWallet` → `POST /door/wallet-charge`; `fetchDoorSales` → `GET /door/sales`; `sendHeartbeat` → `POST /door/heartbeat`. **Backend routes with no app caller:** `POST /door/check-ins/verify`, `GET /door/check-ins/:id`, `GET /door/attendance-report`, `GET /door/devices`, unbind/reauthorize, offline manifest/sync, all door-code and cover-wallet admin routes. Local schemas in `schemas.ts` (can drift): `loginResponseSchema`, `checkInResultSchema`, `staffDenyResponseSchema`.
