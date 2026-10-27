# 05. QR codes and admission

Code: `packages/core/src/application/door/scanner-service.ts` (`decodeQr` ≈L474, `scan` ≈L800, `overrideScan` ≈L999, `recordStaffDeny` ≈L1316), `domain/models/{entitlement,scan-ledger}.ts`, `infrastructure/firestore/firestore-entitlement-repository.ts` (`claimAdmission` ≈L100).

## Rotating ticket QR (this checkout)
- Format `entitlementId:windowStart:hex(HMAC-SHA256(MAGIC_TICKET_SECRET, "entitlementId:windowStart"))`, `windowStart = floor(unixSec/30)*30` (`MAGIC_WINDOW_SEC = 30`). `GET /tickets/:id/qr` returns `{qrPayload, expiresAt = windowStart+30 s, refreshIntervalSec: 30}`; the guest portal refetches every 30 s. `Cache-Control: no-store`.
- Who can mint: the ticket holder (`entitlement.userId === actor.userId`) **or any member of the ticket's organisation, any role, no permission check**; anyone else gets 404.
- Entitlement ids are deterministic: `ENT-` + sha256(`orderId:tierId:index`)[0:32]; they are visible to the holder in `GET /tickets/:id`, `/wallet/tickets` and scan responses.
- `MAGIC_TICKET_SECRET` is optional in the schema; production requires ≥ 32 characters (`config/index.ts`). The same secret signs couple-confirmation tokens (`confirm:`), wallet QRs (`wallet:`) and the offline manifest.

### Verification (`decodeQr`, verified in source)
```
parts = payload.split(':')
if not exactly 3 non-empty parts  ->  treat the WHOLE payload as an entitlement id, no signature   <-- accepted as a credential
else verify HMAC against claimed window OR claimed window minus 30 s (constant-time)
     and require the claimed window within 2 windows (60 s) of now
```
- **Bare entitlement ids are accepted** with no signature (`return { entitlementId: qrPayload, magic: false }`), online and in offline sync. The threat model (line 73) says a bare id is never accepted; it only means a malformed 3-part payload fails. Anyone who knows or sees an id can present it; it defeats the rotating-QR and screenshot protection. The offline doc admits "legacy static QR still accepted".
- **Effective lifetime is about 90–120 s, not 30 s:** the signature may cover the claimed or the previous window, and the claimed window may be up to 60 s off the current one. The same applies to wallet QRs. The threat model's "±2 windows" line understates it.
- Replay and screenshot defence is therefore: the rotating signature (about 2 minutes) plus the atomic single spend (`already_used`).
- A mismatching signature on a 3-part payload records a ledger `denied` row with `invalid_signature`.

## Wallet QR
`cw:<walletId>:<windowStart>:<hmac("wallet:<walletId>:<windowStart>")>`, 30 s window, same current-or-previous rule and ±2 window check, so also about 90–120 s. `verifyWalletQr` never falls back to a bare id. Minting is **guest-only** (`wallet.userId === actor.userId`, else 404; a deliberate D-029 fix). A route comment in `cover-wallet-routes.ts` still says staff may read it; the service and doc say guest only.

## Couple confirmation token
`base64url(entitlementId|eventId|sessionId|deviceId|expectedScansUsed|seats|expiresAtMs).hmac("confirm:"+body)`, TTL 30 s; bound to the session, device and scan count and checked at confirm.

## Admission, in order (`scan()`)
1. `requireEvent` (tenant). 2. `authenticateSession`. 3. `requireOrgAccess(session)`. 4. `canSessionScan`. 5. `requireBoundDevice`. 6. Gate: if the code pins a gate, a different client gate → 403.
7. `decodeQr`; failure writes a `denied` ledger row (`invalid_signature`).
8. **Couple gate:** if the ticket exists, belongs to the event, is not void or refund-locked, has `scanCountAllowed === 2` and `scanCount === 0`, and the session has a device, return `confirmation_required` (signed token, nothing written). Only an **exactly-2-seat** ticket takes this path; 3+ seat tickets admit one seat per scan.
9. `entitlements.claimAdmission(id, eventId)`.
10. `recordScan` (ledger row), `bumpCodeUsage`, `touchDevice`, audit on success.

### `claimAdmission` (the keystone)
A single `db.runTransaction`: read `v2_entitlements/{id}`, run the pure domain `admitSeats()`, write the new entitlement if admitted. The memory adapter does the same with no `await` between read and write. Concurrent scans of one ticket admit exactly once.

### Rules (`evaluateAdmission`, `entitlement.ts`), checked in order
1. Missing entitlement → `invalid_signature` ("Ticket not found").
2. `eventId` mismatch → `wrong_event` (answered before any state, so it leaks nothing).
3. `status === 'void'` → `void_ticket`.
4. `refundLock` set → `void_ticket` ("temporarily unavailable").
5. `scanCount >= scanCountAllowed` → `already_used`.
`admitSeats(seats, expectedScansUsed)`: a stale `expectedScansUsed` → `already_used` ("changed since confirmation"); `scanCount + seats > allowed` → `already_used`; otherwise `scanCount++` per seat, append to `scannedAt[]`, and status `redeemed` on the last seat.
**Not implemented on this checkout** (despite the docs' wording): entry windows (no validFrom/validTo or time-of-day check at admission; only the offline manifest carries them) and seat/capacity budgeting (`capacity_exceeded` is in the enum but never emitted; capacity is only shown in stats). Also never emitted: `expired`, `device_invalid`, `wrong_gate`, `promoter_not_authorized`. Deny reasons that do occur: `invalid_signature`, `already_used`, `wrong_event`, `void_ticket`, `override_required`.

## Scan ledger (`v2_scan_ledger`)
Id `SCAN-` + sha256(eventId, entitlementId, ts, randomBytes(12))[0:32]: random on purpose, so every attempt gets its own row (no ledger-level dedupe). Fields: `eventId, organizationId, venueId, entitlementId, doorSaleId (always null), entryType, tierName, tierId, operatorUid (from the session actor), operatorName/operatorRole (client labels only), gate, deviceId, deviceName, deviceBound, status, denyReason, denyMessage, guestName/Email/Phone, scannedAt, admittedCount, scanCountUsed, scanCountAllowed, isOffline, syncedAt (never set by sync), offlineDeviceId, overriddenBy, overrideReason`.
Status FSM: `pending → consumed|denied|cancelled|expired`; `consumed → revoked`; `denied → revoked|overridden`; `cancelled|expired → revoked`; `revoked` and `overridden` terminal. Scans are written directly as `consumed` or `denied`.
**The ledger write after `claimAdmission` is not in the same transaction**: a failure between them spends a seat with no ledger row (and skips the audit).
**Double-tap:** a second scan of the same ticket records a second `denied`/`already_used` row. With the optional `Idempotency-Key` header the same body replays the stored 200; without it, the app just sees "already used". The request hash includes the rotating `qrPayload`.

## Override, staff-deny and manual check-in
- **Override** (`POST /door/override`, `ticket.override`, no session token, no device check): the ledger FSM allows only `denied → overridden` (else 409). It sets `overriddenBy`, `overrideReason`, `admittedCount = max(existing, 1)` and **does not touch the entitlement**. It can override **any** denied row, including `invalid_signature`, `void_ticket` and staff-deny rows, never re-validating the ticket. An overridden `already_used` ticket stays spent. An overridden staff-deny or `override_required` row counts as an admission in stats while the ticket stays unspent, so the same guest can later scan in and be counted twice.
- **Staff-deny** (`POST /door/staff-deny`, session + bound device, `SCANNER_COMMAND`): writes a `denied` row with `denyReason: override_required` and `denyMessage: "Refused by staff: <reason>"`, `admittedCount: 0`, an audit record; does not spend the ticket; does not require `canScan`.
- **Manual check-in** (`POST /door/guests/check-in`, `ticket.override`): same `claimAdmission` for one seat; writes a `consumed` row with `deviceId: null`, `deviceName: 'manual check-in'`, `deviceBound: false`; failure → `InvalidOperationError` → 400; **bypasses the couple confirmation**, so it admits only one seat of a couple ticket.

## Entitlement model
Fields: `id, orderId, eventId, organizationId, tierId, tierName, userId (nullable), holderName, status valid|redeemed|void, scanCountAllowed, scanCount, scannedAt[], refundLock {refundRequestId, frozenAt}, version, createdAt, updatedAt`. FSM: `valid → redeemed|void`, `redeemed → void`, `void` terminal. A couple ticket has `scanCountAllowed: 2`.
