# 03. Screens and the scan flow

## Screens
| Route | What it does | Calls | Notes |
|---|---|---|---|
| `index` | boot spinner | none | |
| `login` | email + password; org discovery; venue picker | `POST /auth/login`, `GET /organizations` | Dead UI: "Remember this device", "Forgot?", "gate access code" button (fake message). Field label says "STAFF ID OR PHONE" but the API takes an email. If the build is pinned to a venue the account cannot access: "This build is pinned to a venue your account cannot access". A manual "venue id" field is honoured only if the account is a member |
| `pairing` | device name → register handset | `POST /door/devices` | Body copy "only a manager can restore it". **Server needs `door.manage`** (see `04-…`) |
| `redeem` | tonight's events → door code → open shift | `GET /door/events?date=today`, `POST /door/sessions` | Cards cycle 4 themes, LIVE badge, DOOR CODE input (`C1R-XXXXXXXX`, auto-capitalised). Sends an optional GPS fix (`requestForegroundPermissionsAsync`, `getCurrentPositionAsync(Balanced)`; any denial returns undefined and never blocks). "ALL GATES" chip is hard-coded text |
| `profile` | back + `SettingsPanel` | none | reachable before a shift exists |
| `(tabs)/_layout` | floating pill tab bar; `useHeartbeat()`; `ToastHost`; `AppScreenHeader` (event title, gate or "All Gates", pulsing LIVE, back = end shift) | `POST /door/heartbeat` every 60 s | heartbeat failures are swallowed; no AppState handling; nothing sent if meta is null |
| `(tabs)/scan` | the scanner | see below | |
| `(tabs)/door` | ENTRY FORM / WALK-INS / DINE-IN | `POST /door/ticket-sale|walk-in|dine-in`, `GET /door/sales` | see `06-…` |
| `(tabs)/guests` | roster search + manual check-in | `GET /door/guests`, `POST /door/guests/check-in` | see `07-…` |
| `(tabs)/stats` | occupancy + door sales | `GET /door/stats`, SSE, `GET /door/sales` | see `07-…` |
| `(tabs)/settings` | `SettingsPanel` | none | |
| `wallet` | cover tab: scan wallet QR, charge preset items | `POST /door/wallet-qr`, `POST /door/wallet-charge` | see `06-…` |

## The scan screen (`app/(tabs)/scan.tsx`)

### Camera and decoding
`useCameraPermissions`: null renders blank; not granted shows "Camera access is required" + GRANT ACCESS (no "open Settings" deep link). `CameraView` with `barcodeTypes: ['qr']`, torch ("FLASH"), animated scanline. `onBarcodeScanned` is active only while `uiState.kind === 'idle'`. **The app does no client-side QR parsing**: the raw string goes to the server as `qrPayload`. Payload formats the server understands: the rotating `entitlementId:windowStart:signature` (30 s window) or a bare entitlement id (see `05-…`).

### Manual entry
"CODE" reveals a text field + CHECK, running the same `runScan(manualCode)`. The "TAP TO SCAN" button only works with typed text (odd affordance). "Deny without a ticket" opens a reason field → `POST /door/staff-deny` → adds an "Unknown" denied recent entry and the toast "Entry denied & logged".

### `runScan(qrPayload)` step by step
1. Return early if already checking, or if `qrPayload === lastScannedRef.current` (de-dup; cleared only by DISMISS / TRY AGAIN, so the same ticket cannot be re-read until the sheet is dismissed; a different ticket can be scanned immediately).
2. `getSessionMeta()`; null → idle with no message.
3. `Network.getNetworkStateAsync()`; not connected → the **offline card**: "SCANNER OFFLINE: Entry denied until connectivity returns." + TRY AGAIN (fails closed).
4. `lookupTicket` → `POST /door/lookup` (non-mutating).
5. If lookup is `invalid` **or** `autoAdmit` is on → `checkIn` (`POST /door/check-ins`, with `gate`) immediately → `handleScanResult`.
6. Otherwise show the "Valid Ticket" sheet (guest, tier, PAX) with DISMISS / ADMIT GUEST; `confirmAdmit` calls `checkIn` with an `admitBusy` double-tap guard.
7. Errors: `isAuthFailure` → toast "session expired" + notify; `network`/`timeout` → the offline card; anything else → a toast with the message.

### Results
`CheckInResult` is a discriminated union with three statuses:
- `consumed`: green "Admitted", haptic Success, a recent entry, toast "{name} admitted". Keeps the last 5 recents and bumps the "inside" counter.
- `denied`: "Entry Denied" with the reason as a pill, haptic Warning, a recent entry carrying `checkInId`, toast "Entry denied & logged". The holder shows as "Unknown" (the deny payload has no holder). There is **no separate "duplicate" status**: an already-used ticket is `denied` with a server `denyReason`.
- `confirmation_required`: the couple flow.
Denied recents show an OVERRIDE button when `canOverride(role)` (any role except `member`): modal with a mandatory reason → `POST /door/override` → marks "OVERRIDDEN". The override modal's `onRequestClose` does not clear the reason. Staff-deny entries pass the deny response id to the override call (whether the backend accepts that is unconfirmed; the backend does allow override of any `denied` row, including staff-deny rows).
No audio. The `soundOn` preference is not wired.

### Couple tickets (`features/scan/coupleConfirm.ts`)
`confirmation_required` returns `{token, expiresAt, seats}`. `useCoupleConfirm` runs a 250 ms ticker against the **server** `expiresAt` (robust to backgrounding). Card: "COUPLE TICKET - N SEATS", "Confirm both guests present (Ns)", NO / YES. After expiry: "Confirmation window expired - re-scan." `confirm(bool)` → `POST /door/check-ins/confirm {eventId, confirmationToken, confirmed}`.
Gaps: a confirm error is an **unhandled promise rejection**; the confirm result bypasses `handleScanResult` (no haptic, no recent entry, no inside-count bump); `ResultSheet` shows "Couple Ticket" while the confirmation card is up; no idempotency key (a repeat returns denied by server rules).

## Door entry screen (`door.tsx`, 932 lines) in short
Fields: name, 10-digit phone (`^\d{10}$`), gender (male/female only though the API accepts more), age ≥ 18, optional email; entry type walk, dine or ticket (ticket needs a tier from `meta.tiers`; sold-out tiers disabled); `totalGuests` stepper; payment mode cash/card/upi/other. `canWalkIn === false` only shows a warning (the form still submits and the server refuses). `canCharge` shows a "COVER TAB" button to `/wallet`. One `crypto.randomUUID()` key per form with an `attemptSignature` over all fields: same fields reuse the key on retry, edited fields mint a new key, reset after success; a replayed ticket sale shows "Already sold - do not collect again". Lists for WALK-INS and DINE-IN come from `GET /door/sales`.

## UX and accessibility gaps (reported)
Very few `accessibilityLabel`/`accessibilityRole` props (only the tab bar); icon-only buttons (back, FLASH, brand mark) unlabelled; result colour is the main signal (text title and haptics also present, no audio); floating tab bar may cover content on short screens; errors show raw `cause.message`; phone validation is India-specific; no pull-to-refresh or loading indicator on guests; the guests check-in button has no per-row busy state.
