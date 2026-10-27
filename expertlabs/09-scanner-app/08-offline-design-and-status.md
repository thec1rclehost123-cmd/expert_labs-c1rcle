# 08. Offline: design and status

Offline door scanning is a **stated business requirement** (owner decision 2026-10-18, `cloudflare_plan.md`). Today the app denies entry when offline. This page explains the old design, why it could not work, the new design, and exactly what is built where.

## Status at a glance (PR states from `gh`, 2026-10-24)
| Piece | State |
|---|---|
| App behaviour | **Fails closed**: offline card "Entry denied until connectivity returns"; Settings "Offline mode" row disabled |
| Backend old manifest + sync (this checkout and `origin/main`) | present, **unusable and unverifiable** (below) |
| Backend new design: derived keys, Ed25519 manifest, time-bounded sync | **PR #138 merged to staging 2026-10-22** (commit `15c18c8`: `offline-ticket-crypto.ts`, tests, `docs/operations/offline-scanning.md`). **Not** on the backend checkout `fix/partner-connections-and-events` or `origin/main` (verified: file absent) |
| Frontend contract mirror | **PR #81 merged 2026-10-22** |
| Frontend offline engine | **PR #82 open**: `apps/scanner-app/src/offline/*` (about 1,372 lines incl. tests), commit `5911aa0`, "not wired in yet" |
| Persistent store, API client calls, scan-screen branch, settings opt-in, sync trigger, conflict UI | **not built** |

## Old design and why it fails (this checkout)
`buildOfflineManifest` lists non-void, non-refund-locked, not-fully-scanned entitlements (500 per page, unpaginated response), each `{entitlementId, validFrom = signedAt, validTo = signedAt + 12 h, signature = HMAC(master, "manifest:eventId:entId:signedAt:expiresAt")}`.
- The signature uses the **master secret**, which a device cannot hold, so a device cannot verify it, and the server never verifies it either (`syncOfflineScans` never reads a manifest signature). The route comment claiming otherwise is false on this checkout.
- Sync re-decides each scan with `decodeQr` against **server now**, not the device's `scannedAt`, so any rotating QR scanned more than about 90–120 s before sync is rejected as `invalid_signature` and admits nothing. Bare ids are accepted. `scannedAt` is stored unvalidated, `syncedAt` is never set, `deviceId` per scan is not checked against the session, device binding is not checked, and re-sending a batch without an idempotency key records fresh `already_used` conflict rows.

## New design (backend staging, PR #138)
```
ticketKey  = HMAC-SHA256(MAGIC_TICKET_SECRET, "ticket-key|" + entitlementId)
QR payload = entitlementId : windowStart : HMAC(ticketKey, "entitlementId:windowStart")     (30 s window)
manifest   = GET /door/offline-manifest?eventId=   (needs a scanner session that may scan)
  { version:1, eventId, signedAt, expiresAt (+12 h), keyId, publicKey (base64 raw Ed25519),
    manifestSignature,
    manifest:[{ entitlementId, validFrom, validTo, seatsRemaining, ticketKey }] }
```
- The online QR now also signs with the per-ticket key, so **QRs minted before the deploy become invalid**.
- Manifest signed bytes: `c1rcle-offline-manifest-v1\n<eventId>\n<signedAt>\n<expiresAt>\n<sha256hex(sorted lines "id|validFrom|validTo|seatsRemaining|ticketKey\n")>`. The Ed25519 key is derived from the master secret by HKDF-SHA256 (salt `c1rcle`, info `offline-manifest-ed25519-v1`), so no extra secret is stored. Included: tickets not void, not refund-locked, not fully scanned. Excluded: tickets issued after `signedAt`.
- **Device verification:** check the rotating QR with `ticketKey` (current or previous window, claimed window within 2 windows of the device clock), look up the entry in the verified manifest, and spend from `seatsRemaining` locally.
- **Sync:** `POST /door/offline-sync` with `{payload, scannedAt, deviceId}`, 1–500 scans per batch; rotating codes verified **as of `scannedAt`**; `scannedAt` must be no more than 5 minutes in the future and no older than 12 h + 24 h grace (36 h) or the entry is `offline_expired` (ledger `denied` row, no seat spent); admission still goes through the atomic `claimAdmission`, so two devices that both admitted the same ticket produce one admission and one `already_used` conflict.
- **Tests added on staging:** a device verifies the server's QR with only its manifest key; an hours-old code is accepted at sync; a stale screenshot claimed as a fresh scan is refused; a tampered signature is refused without spending the ticket; the device clock is bounded (`offline_expired`); two offline devices admit once. Crypto tests include known-answer vectors, per-ticket isolation, window replay rejection, forged signature rejection and manifest signing independent of entry order.

## Frontend engine (PR #82, `apps/scanner-app/src/offline/`)
Pure-JS `@noble/curves` and `@noble/hashes` (exact-pinned 2.4.0), plus Jest config and `@jest/globals`.
| Module | Role |
|---|---|
| `crypto.ts` | `parseRotatingQr`, `checkRotatingQr` (HMAC-SHA256 with the key as the UTF-8 of the hex string; current or previous window; claimed window within 2 windows; constant-time compare), `manifestSigningBytes`, `verifyManifestSignature` (Ed25519; never throws) |
| `manifest.ts` | local zod schema (deliberately not importing contracts, so it can drift); `loadManifest` rejects `unsupported_version \| malformed \| wrong_event \| key_changed \| bad_signature \| already_expired`; **pinned key id** guards against a key swap mid-event |
| `decision.ts` | pure `decideOfflineScan`: `duplicate_read` (same payload within 2 s), `not_rotating_code` (bare id), `manifest_expired`, `clock_behind` (device clock > 5 min before `signedAt`), `not_in_manifest`, `invalid_signature`, `already_used` (seats exhausted); admit returns `seatsLeft` |
| `scanLog.ts` | persistent `OfflineScanLog` over a `KeyValueStore`; statuses `pending|synced|conflict`; writes serialised and persisted before resolving; a corrupt file **throws** instead of being discarded; synced records keep counting against the seat budget until `settleForNewManifest()` |
| `scanner.ts` | `OfflineScanner` serialises scans through a promise queue so two reads cannot both take the last seat |
| `sync.ts` | sends pending scans oldest-first in batches of 500; idempotency key `offline-sync-` + first 32 hex of sha256 over the joined record ids; a batch failure stops the run and keeps everything pending; duplicate payloads mark the later scans as the refused ones |
Tests: `crypto.test.ts` (server known-answer vectors), `manifest.test.ts`, `scanner.test.ts`, `sync.test.ts` (tamper, wrong event, smuggled ticket, pinned-key change, expiry, replay window, bare id, couple seats, concurrency, restart durability, corrupt file, batching).

### Not wired (per the module README and checks)
- No persistent `KeyValueStore`: SecureStore is too small for a manifest; the README says use a file or SQLite store with an encryption key held in SecureStore, and treat the manifest like a session token (it holds per-ticket keys).
- No API client functions for `GET /door/offline-manifest` or `POST /door/offline-sync` (which expects an `Idempotency-Key` **header**, unlike the app's body-field convention).
- `scan.tsx` is untouched (still denies); the Settings toggle is still disabled; no default-off venue opt-in (a venue risk decision); no sync-on-reconnect trigger; no conflict UI; no manifest refresh or `settleForNewManifest` call site.

## Risks of the new design
- **Key exposure:** the manifest hands every admissible ticket's key to any scan-capable session in one unpaginated response (about 250 bytes per entry, multi-MB for large events); a compromised handset can mint valid rotating QRs for all listed tickets for up to 12 h. The shipped manifest does **not** encrypt keys to the device session, and does **not** carry the planned revocation-list version (`cloudflare_plan.md` A4.3 is partial).
- **Trust anchor:** the public key arrives in the same response as the signature, so the signature detects tampering of a stored manifest, not a hostile server; trust relies on pinning `keyId` from an earlier fetch (trust on first use).
- **Revocation:** refunds and voids after the snapshot are invisible offline until the next manifest fetch or sync.
- **Double admission:** two offline gates can both admit the same ticket; the second is flagged only at sync, after the guest is in. Couple tickets admit one seat per scan offline (no confirmation step).
- **Clock:** `scannedAt` is the device's claim; a back-dated clock can make an old screenshot verify within the 36 h bound; the engine tolerates a device clock up to 5 minutes behind `signedAt`.
- **Other:** bare-id acceptance is retained in sync; rotating `MAGIC_TICKET_SECRET` invalidates all QRs, manifests and keys at once; tickets issued after `signedAt` must be scanned online; device binding is still not checked in manifest or sync; no per-scan nonce (a replayed batch cannot be told apart without the batch key).
- **Process:** no ADR in `decisions.md` records the offline admission model.
