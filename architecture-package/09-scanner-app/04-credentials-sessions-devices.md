# 04. Credentials, sessions and devices

Code: `packages/core/src/domain/models/{event-code,scanner-device}.ts`, `application/door/scanner-service.ts` (door codes ≈L591, `openSession` ≈L674), `routes/v2/door/{event-code-routes,scanner-routes,door-ops-routes}.ts`, `plugins/{auth,rbac}.ts`.

## Door codes (`v2_event_codes`)
- Format `C1R-` + 8 characters from `ACDEFGHJKLMNPQRTUVWXYZ2346789` via `crypto.randomInt` (about 2^37 combinations). Id `CODE-<32 hex>`. Raw code strings are never written to audit records.
- Types and permissions: see `01-…`. Fields: `gate` (nullable, pins the gate), `maxDevices` (default 5, max 100), `allowReuse` (default false), `expiresAt`, `status active|revoked|expired`, revoke info, stats `{scansCount, doorEntriesCount, doorRevenue, lastUsedAt, activeSessions}`. `doorEntriesCount` and `doorRevenue` are never incremented by any caller found (`incrementDoorEntry` unused).
- Created by `door.manage` (`POST /events/:eventId/door-codes`); organisation, venue and event come from the event, not the request. `GET` lists **active codes only** and returns raw code strings (gated like minting). Revoke sets the code revoked and revokes every live session in a non-atomic loop.
- **Expiry is only checked at session open.** A session opened before expiry keeps working for its 12 h; a session is not capped by event end.

## Opening a shift (`POST /door/sessions`)
Better Auth actor only: **no `requirePermission` and no scanner token** (this route mints the token). Rate class `SENSITIVE_COMMAND` (10/min per IP+user+org: a shared NAT or shared staff login can hit it at door opening).
1. `findByCode` is a **global** query (`where code ==`, limit 1). A wrong code, another organisation's code, or a code for a different event all throw the **same** NotFound (no oracle).
2. A non-active code → 400; an expired code is lazily flipped to `expired` → 400.
3. Device cap is counted from live sessions (`findActiveByCode`). Reopening on the same device revokes the old session; otherwise with `!allowReuse` and `live.length >= maxDevices` → 400. This is **read-then-write with no transaction**, so a race can exceed the cap.
4. `startShift` auto-binds the device (`bindDevice`) before `openSession`, so there is no mandatory pre-registration step.
5. Geofence (D-030): 500 m, shift-open only, client-reported GPS, skipped when the venue has no coordinates.

## Scanner session (`v2_scanner_sessions`)
Id `SESS-<32 hex>`; fixed **12 h** TTL (`SESSION_TTL_MS`), no refresh, no sliding window. The raw token is `scn_` + `randomBytes(32)` base64url and is returned **once**; only its SHA-256 is stored, as the doc id of `v2_scanner_session_tokens` (`{sessionId}`). `authenticateSession(token, eventId)` returns one generic 401 "Scanner session is not valid for this event" for unknown, revoked, expired or wrong-event tokens. A `cleanupExpired` timer runs every 5 min (Firestore driver only). Status is derived (active, expired, revoked).
**The token is not bound to the user who opened it**: a leaked token plus any organisation member's own login works. The session `sessionType` (`staff|device`) is stored but never used for permissions.

## Devices (`v2_scanner_devices`)
Id `SDEV-` + first 32 hex of sha256(`orgId:deviceId`). The device id is client-generated (16–128 chars) and explicitly **not** a security boundary. Fields: `deviceId, deviceName, status active|unbound, boundBy, boundAt, unboundAt, unboundReason, lastSeenAt, lastEventId, lastGate, scanCount, lastScanAt, lastScanResult`. FSM: `active → unbound` via `unbindDevice` (also revokes live sessions, capped at 100), `unbound → active` only via `reauthorizeDevice` (`door.manage`). `bindDevice` refuses to reactivate an unbound device (403; found live and fixed in D-029).
`requireBoundDevice` is called in **scan, confirm, staff-deny and heartbeat**. It is **not** called in `resolve` (verify/lookup), `buildOfflineManifest`, `syncOfflineScans`, wallet QR/charge or ticket-sale, so an unbound device can still do those while its session lives (unbind revokes its sessions, which mitigates it). `touch` is an unversioned `update` with `FieldValue.increment`.

## Which credentials each route needs (verified from route files)
| Needs login + `X-Scanner-Session-Token` | Needs login + role permission only | Needs login only |
|---|---|---|
| check-ins, verify, lookup, confirm, staff-deny, heartbeat, wallet-qr, wallet-charge, ticket-sale, offline-manifest, offline-sync | door-code CRUD, device CRUD (`door.manage`); override, manual guest check-in, walk-in, dine-in, cover-wallet debit (`ticket.override`); events, guests, attendance-report, sales, wallet GETs (`venue.read`) | `POST /door/sessions`, `GET /door/sessions/:id`, `GET /door/check-ins/:id`, `GET /tickets/:id/qr`, `GET /door/stats`, `GET /door/stats/stream` (stats check organisation in the service) |
Session permissions are checked per route: `canScan` (scan, resolve, confirm, manifest, sync), `canCharge` (wallet-qr, wallet-charge), `canDoorEntry` (ticket-sale). Staff-deny and heartbeat need only a valid session, so a `charge` session can record staff denials.

## Role permission mapping (`plugins/rbac.ts`)
`owner|admin|manager` hold `ticket.override` and `door.manage`; `member` holds read-only permissions (`venue.read`, `event.read`, …) and **neither**.

## Findings (verified unless marked reported)
1. **`POST /door/devices` requires `door.manage`** (`door-ops-routes.ts:104`), contradicting the contract, SOTA doc and threat model, which say it is deliberately ungated so staff can register a phone. The app's `pairing` screen calls it for every staff member, so a `member`-role scanner user gets 403 (and the app's guidance for 403, "unbound by a manager", would misdescribe it). Cause: audit task A4 (2026-10-11) added guards without updating the docs.
2. Walk-in and dine-in need `ticket.override` (not the session `canWalkIn`), while the app only checks the session flag; `member` door workers see the form and get 403.
3. Any organisation member, including read-only `member`, can open a shift if they have a door code, read any session (`GET /door/sessions/:id`), read any check-in row including the guest name (`GET /door/check-ins/:id`), and mint a valid rotating QR for any ticket of their organisation (`GET /tickets/:id/qr`).
4. Scanner token not bound to the opening user; device binding not enforced on several paths; `maxDevices` race; session outlives code expiry and event end.
