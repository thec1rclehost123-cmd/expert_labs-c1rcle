# 12. Status, gaps and roadmap (candid)

Snapshot 2026-10-24. **Keep under NDA.** PR states come from `gh` on 2026-10-24; "verified" = checked in source; "reported" = from a code read.

## Branch and PR state
- **Backend** (`C1RCLE-BACKEND`): checkout `fix/partner-connections-and-events` (6 ahead / 9 behind `origin/staging`). `origin/staging` has PR #138 (merged 2026-10-22, commit `15c18c8`): `offline-ticket-crypto.ts` (216 lines), its tests (183 lines), +184 lines of route tests and `docs/operations/offline-scanning.md`. **`origin/main` and the checkout do not.** Merged scanner-related: #138, #73 (event/door-code seeding scripts), #66 (password reset for partner/scanner staff), #58 (venue coordinates UI and geofence e2e), #50 (scanner backend unify). Edge work: #140 (multi-host allowlist), #141 (origin lock), #142 (sidecar logs); #143 open.
- **Frontend** (`C1RCLE-FRONTEND`): checkout `fix/partner-connections-contracts` (0 ahead / 1 behind local `origin/staging`; the local remote-tracking ref predates #81, so `git` shows the old contract file). **PR #81 merged 2026-10-22** (contract mirror); **PR #82 open** (offline engine, commit `5911aa0`); merged #21 (`feat/ci-pipeline`: full scanner app + venue coordinates UI).
- Both repos change quickly: run `git fetch` then `git rev-list --left-right --count origin/staging...HEAD` before quoting counts.

## Corrections to earlier pages of this package
`09-scanner-app/README.md` (first pass) and `12-status-and-known-issues` said PR #138, the manifest contract mirror and the "backend derived-key scanning" were **Unmerged**: they are merged on staging (not on the checked-out backend branch). Offline status in `12-status-and-known-issues/README.md` is updated accordingly.

## Gap register (IDs are this package's own, `SG-n`)
| ID | Sev | Gap | State |
|---|---|---|---|
| SG-1 | High | Offline engine not wired (no store, setting, sync trigger, conflict UI); offline is a business requirement | Open (PR #82) |
| SG-2 | High | `POST /door/devices` requires `door.manage`; pairing fails for `member`-role staff; contract/threat model say ungated | Open (verified) |
| SG-3 | Med | Walk-in/dine-in need `ticket.override`, app checks only the session flag | Open |
| SG-4 | Med | No revocation list in the manifest; refunds/voids invisible offline | Open (`cloudflare_plan.md` A4.3 partial) |
| SG-5 | Med | Two offline gates can both admit the same ticket; no per-gate partition or manifest pagination | Open, documented |
| SG-6 | Med | Couple tickets offline admit one seat per scan, no confirmation | Open, documented |
| SG-7 | Med | App-side Ed25519/HMAC verification plus local queue unchecked (A4.4) | Open (depends on #82) |
| SG-8 | Med | No native staff-token refresh; every cold start re-logs in; 401 just fails | Open |
| SG-9 | Med | Manual physical-device E2E never run (only a script-level HTTP E2E, 24/25 passed) | Open |
| SG-10 | Med | Web build cannot meet the secure-storage requirement; decision needed on whether it ships to real doors | Open (disclosed deviation) |
| SG-11 | Low–Med | `DoorSale` and `CoverWalletTxn` id schemes (random/timestamp ids; same-millisecond collision) | Deferred (needs migration plan) |
| SG-12 | Med | Stats lack hourly buckets, denied count, gender/age | Open (backend decision) |
| SG-13 | Low | Geofence soft layer, spoofable, no-op without venue coordinates | Partly closed (coordinates UI shipped) |
| SG-14 | Low | Door composite indexes for the gate/device/hour report unverified; door sales filtered in memory | Tracked (task D4, audit M4) |
| SG-15 | Med | No tests for the scanner app on this branch; Jest `--passWithNoTests` always passes; no EAS/native pipeline | Open (verified) |
| SG-16 | Low | Findings T1–T19 in `11-…` (bare id, QR lifetime, reconciliation sign, walk-in pricing, global idempotency, …) | Open |
| SG-17 | Info | No ADR for the offline model; D-030 label collision (geofence vs "partner-chain audit closes", duplicated in `decisions.md`); "A4" means two different tasks in `cloudflare_plan.md` and root `task.md` | Doc hygiene |

Closed during the earlier E2E run (per the task log): two missing ledger indexes; `GET /door/sales` 500 on empty results; `undefined` audit fields breaking walk-in/dine-in/charge; a ledger read-after-write that left door and online sales unsettled; `staffDenyResponseSchema` shape; a real double charge from missing idempotency wrapping on wallet-charge/ticket-sale; the CORS allow-list missing `X-Scanner-Session-Token` and the SSE bypassing `@fastify/cors`.

## Frontend-specific defects (reported unless noted)
No audio and decorative settings toggles (`soundOn`, `hapticOn`, `continuousScanning`, gate selector); dead login UI (remember, forgot, gate access code); hard-coded "v2.4" and "ALL GATES"; `clearDevicePairing` never called (the 403-unbound path is unhandled); confirm-couple errors unhandled; stats SSE reconnects every 2 s forever; `getSessionMeta` unvalidated cast; localhost API fallback in `app.config.ts`; no crash reporting; sparse accessibility labels; guest check-in has no per-row guard; the override modal does not clear its reason on close.

## Stale documentation
- `docs/scanner-app/README.md` (header "verified 2026-09-23"): says no scanner app exists yet and no `firestore.indexes.json`/dedicated cover-wallet service; all now exist.
- `sota-architecture.md`: indexes "prescribed, not committed" (they are); §7.3/§10/§11/SOTA-2/§14 describe the old manifest and forbid an offline queue.
- `06-v1-vs-v2-and-rollout.md`: "no v2 scanner app frontend anywhere". `05-cover-wallet-door-sales.md`: flows "deferred on the frontend" (Phases 2–3 are built). `07-storage-sizing-caching.md`: no attendance endpoint, one `cache-control` hit. `04-guest-roster-stats.md`: cursor on guests (real API uses `limit` and `truncated`), heartbeat interval.
- `api-contracts/scanner-app.md`: old offline shape; device-register permission; walk-in permission; no mention of `deviceLocation`, `/door/attendance-report`, `GET /door/devices`, `/cover-wallets/*`; timestamps "ISO except `Session.expiresAt` epoch ms" (the code uses ISO strings, `sessionExpiresAt`).
- `scanner-threat-model.md`: see `11-…`. `roadmap/phase-05-…`: early log entries (501 stubs) superseded; no mention of the Ed25519 redesign. `scanner-routes.ts:322-327` comment (manifest verified at sync) is false on the checkout; `door-sale-routes.ts` comment (party size priced) is false.
- `cloudflare_plan.md`: calls #138 a "draft PR" (merged) and says "PLAN ONLY" though edge work shipped. Frontend audit M3 / §1.3 still describes a local scanner schema duplicate (now an adapter).
- `docs/operations/offline-scanning.md` and `cloudflare-edge.md` are current but exist only on staging and some nginx branches.

## Suggested fix order
1. Decide and document the offline model (ADR), then wire PR #82: persistent encrypted store, manifest fetch with key pinning, opt-in setting, sync on reconnect, conflict UI.
2. Remove bare-id acceptance (or gate it behind a migration flag) and tighten QR validity to the documented window; add a QR-lifetime test.
3. Fix door permissions: align `POST /door/devices` and walk-in/dine-in with the documented session-based model (or update docs and app), and restrict session/check-in reads and ticket-QR minting to staff roles.
4. Bind the scanner token to the opening user and enforce device binding on all session routes; cap sessions by code expiry and event end; make `maxDevices` transactional.
5. Money: multiply walk-in/dine-in by party size (or define the tier as per-party), verify non-cash modes or reject them, scope idempotency keys per tenant, fix the reconciliation sign and txn id, wire void/refund routes, make ticket-sale atomic and set `admitsPerUnit`.
6. App: server-side logout/shift end, token refresh, scan-specific timeout, wire the preference toggles or remove them, handle the unbound-device 403, add accessibility labels, add tests, add an EAS/native pipeline and run the physical-device E2E.
7. Stats: aggregate-only reads, caching, hourly buckets; add a stats SSE backoff.
8. Refresh the stale docs listed above and merge the staging-only offline docs to main.
