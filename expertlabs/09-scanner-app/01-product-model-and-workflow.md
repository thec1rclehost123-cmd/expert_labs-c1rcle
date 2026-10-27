# 01. Product model and event-day workflow

## Who uses it
Door and entry staff of a venue, on a handset (Expo / React Native; a web build exists as a preview target). The platform is multi-tenant: competing venues hold real logins, so cross-tenant isolation is a core requirement (cross-tenant lookups answer 404, not 403).

Roles that touch the door:
- Organisation roles `owner|admin|manager|member` and permissions `door.manage`, `ticket.override`, `venue.read` (owner/admin/manager hold the first two; `member` is read-only).
- Scanner **session permissions** (booleans on the shift): `canScan`, `canDoorEntry`, `canWalkIn`, `canCharge`.
- A guest owns the ticket QR and the wallet QR. A supervisor (partner or admin console) handles refunds, top-ups and freezes; **the scanner never refunds, tops up or freezes**.

## The three credentials
| Credential | How sent | Lifetime | Storage |
|---|---|---|---|
| Staff session | `Authorization: Bearer` + `X-Organization-Id` | 7 days (contract says refreshable; the app has **no refresh flow**) | memory only |
| Device identity | `deviceId` (`scanner_` + 32 hex, generated once on the handset, never hardware-derived) | permanent until a manager unbinds it | Keychain/Keystore |
| Scanner session | `X-Scanner-Session-Token` (`scn_…`) | 12 h fixed | Keychain/Keystore; returned once by `POST /door/sessions` |

## Event-day workflow
1. **Before doors open (manager):** mint a door code `C1R-XXXXXXXX` for the event (`POST /events/:eventId/door-codes`), choosing a type (`full`, `scan_only`, `charge`), an optional gate, `maxDevices` (default 5) and an optional expiry. Revoking a code kills every live session it opened.
2. **Staff sign in** (`POST /auth/login`). The app finds the venue (build pin, on-device binding, or `GET /organizations`). `GET /door/events?date=today` lists tonight's events, resolved in IST so a 1 a.m. shift belongs to the previous night.
3. **Pair the handset** (first launch): the app sends `POST /door/devices`. (See the permission problem in `04-…`.)
4. **Open a shift:** staff pick the event and enter the door code → `POST /door/sessions`. The response carries the session token (shown once), event, permissions, gate, tiers and opening stats. A best-effort GPS fix is sent; the server may refuse if the handset is more than 500 m from the venue (soft geofence, D-030; spoofable, and silently skipped when the venue has no coordinates).
5. **Scan guests:** `POST /door/check-ins` always returns HTTP 200 with `consumed`, `denied` or `confirmation_required`. Couple tickets use a two-step confirm. Manual lookup uses `/door/lookup` (read-only).
6. **Roster and manual entry:** `GET /door/guests` (server-side search); `POST /door/guests/check-in` (needs `ticket.override`) covers a cracked guest phone.
7. **Escalations:** `POST /door/staff-deny` refuses someone without spending the ticket; `POST /door/override` (needs `ticket.override`) admits a denied scan and is its own ledger state.
8. **Door money:** paid door ticket sale, walk-in and dine-in headcount entries, and cover-wallet charges (see `06-…`).
9. **Monitoring:** heartbeat every 60 s from the app; stats polled and streamed (see `07-…`).
10. **Lost handset:** a manager unbinds the device (`/door/devices/:id/unbind`); effective on the next scan and it revokes the device's sessions. Only `/reauthorize` (`door.manage`) restores it.
11. **End of shift:** the session expires after 12 h; the app's "end shift" and "log out" are **client-only** (no server revoke).

## Gates, devices and shifts
A *gate* is an optional string pinned by the door code; the session and device carry `gate`, `deviceId`, `deviceName`. A *shift* is one scanner session. Sizing assumptions in the docs: 300–500 guests per event, 2–3 devices, one event per venue at a time, about 0.2 scans per second per device at peak.

## Permissions by door code type
| Code type | canScan | canDoorEntry | canWalkIn | canCharge |
|---|---|---|---|---|
| `full` | yes | yes | yes | no |
| `scan_only` | yes | no | yes | no |
| `charge` | no | no | no | yes |

## Online versus offline
- **Product rule today (SOTA-2):** offline always denies, with no queue. The Settings "Offline mode" row is disabled for this reason.
- **Owner decision 2026-10-18:** "Offline door scanning is required." A new opt-in "pre-authorised offline" design exists on backend staging (see `08-…`). `docs/architecture/decisions.md` has **no ADR** for it, although the SOTA doc says an offline admission queue needs a new ADR and contract version.

## Cover wallet and walk-ins (product intent)
A prepaid, per-event, per-guest tab in integer paise; an emptied wallet terminates; the guest shows a rotating `cw:` QR; the operator picks a venue-defined **preset item** (the client never names a price); 3 charges per device per minute. Walk-ins and dine-ins are headcount entries priced by the event's walk-in/dine-in tier; a paid door ticket sale creates a real order (cash only).
