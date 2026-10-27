# 11. Threat model and security findings

Source: `docs/architecture/scanner-threat-model.md` (reviewed 2026-09-15), plus verification in this session. Keep under NDA.

## Assets and boundaries
Assets: free entry, a guest's tab, door cash, the guest list, another club's data. The handset is **semi-trusted**: every value from it is attacker-controlled. Path: nginx → gateway (authn, RBAC, tenancy, validation, idempotency) → core → Firestore.

## Documented mitigations (each pinned by a named test, per the doc)
- **Forging entry:** rotating HMAC QR (30 s window, ±2 windows, `timingSafeEqual`); atomic `claimAdmission`; couple tickets with both seats in one claim and a `confirm:`-prefixed token bound to ticket, event, session and device; refunded tickets refused without spending a seat; override is its own terminal ledger state.
- **Stolen handset:** unbind takes effect on the next scan and revokes sessions; re-register cannot resurrect an unbound device (found live and fixed in D-029, moved to `/reauthorize`); the session token is stored only as a SHA-256 hash.
- **Money theft:** preset-item-only charges (`findChargeableItem` fails closed); only the guest can mint the tab QR (second D-029 hole); dual idempotency plus a velocity limit; price recalculated from the tier; session-type gating; refunds, top-ups and freezes absent from the scanner.
- **Cross-tenant:** 404 not 403; the code's organisation comes from the event; `wrong_event` answered before ticket state with no details; organisation checks on roster and stream.
- **DoS:** `SCANNER_COMMAND` 300/min, `SENSITIVE_COMMAND` 10/min; SSE budget (10 per actor, 500 global, 15-minute cap, backpressure closes); bounded roster with `truncated`; `count()`/`sum()` aggregates for stats.
- **Privacy and secrets:** `no-store` responses, pino redaction of `x-scanner-session-token`, door codes omitted from audit records, first name only on the charge view; production refuses to boot without a ≥ 32-character `MAGIC_TICKET_SECRET`.
- **Accepted residual risks (doc):** narrow `wrong_event` vs `invalid_signature` enumeration; door-code lookup not constant-time (about 2^37 codes, 10/min); any org member can read the roster; per-instance rate limits and stream budgets (a second instance needs Redis); the stream is poll-backed (≤ 3 s latency).

## Findings that contradict or extend the threat model
| # | Sev | Finding | Evidence |
|---|---|---|---|
| T1 | High | **Bare entitlement id accepted as a credential** online and in sync, defeating the rotating QR and screenshot protection for anyone who knows an id | `scanner-service.ts:474-477` (verified); doc line 73 says it is never accepted |
| T2 | Med | QR validity about **90–120 s**, not 30 s; wallet QR the same | `decodeQr` accepts claimed or claimed−30 and claim within ±60 s (verified) |
| T3 | High | **Offline sync on the checkout is unusable and its manifest unverifiable**; staging fixes freshness (PR #138) but still hands out all per-ticket keys in one response, relies on trust-on-first-use for the public key, has no device-binding check and no per-scan nonce | `08-…` |
| T4 | Med | Override unbound to shift/device; can override non-ticket denials (invalid signature, void) and an overridden staff-deny can double-count a guest | `05-…` |
| T5 | Med | Any org member (even read-only `member`) can open a shift with a door code, read any session and any check-in row (guest name), and mint a valid QR for any ticket of the org | `04-…` |
| T6 | Med | Scanner token not bound to the opening user; device binding not checked on resolve, manifest, sync, wallet and ticket-sale | `04-…` |
| T7 | Med | Global idempotency collections (`v2_door_sale_idempotency`, `v2_cover_wallet_idempotency`) keyed by raw client keys: cross-tenant replay would return another tenant's data | `06-…` |
| T8 | Med | Cover-wallet money creation: `door.manage` can issue and credit wallets for arbitrary user ids and amounts with no payment link; the offline-debit ban is a client-asserted `isOnline` flag | `06-…` |
| T9 | Med | Walk-in/dine-in price ignores party size; card/UPI/other accepted without verification; the walk-in debits the **operator's own** wallet | `06-…` (price verified) |
| T10 | Med | Wallet reconciliation reports a false mismatch for any wallet with a debit (negative amounts subtracted) | `06-…` (verified) |
| T11 | Low | Wallet txn id collision within the same millisecond; plain `Error` (500) on insufficient balance race; a zero-balance debit terminates the wallet and blocks further credit | `06-…` |
| T12 | Low–Med | Door-code and session lifetimes: session TTL fixed 12 h, not capped by code expiry or event end; `maxDevices` race; revoke loops sessions non-atomically | `04-…` |
| T13 | Low | Velocity limit 3 debits/min per device is low for a busy bar and blocks legitimate idempotent retries (checked before replay); `SENSITIVE_COMMAND` 10/min keyed by IP+user+org is easy to hit at door opening | reported |
| T14 | Med | Door ticket sale: inventory read check (oversell possible), no `admitsPerUnit` (couple tier gets 1 seat), non-atomic steps | `06-…` |
| T15 | Low | `check-ins`, `confirm`, `offline-sync`, `guests/check-in` accept calls without an `Idempotency-Key`; double-tap safety is `already_used` only | `05-…` |
| T16 | Low | Ledger write after `claimAdmission` is not atomic; manual check-in bypasses couple confirmation | `05-…` |
| T17 | Low | Stats cost grows with attendance (full door-sale read per SSE tick per stream), contradicting the doc | `07-…` |
| T18 | Med | **Frontend:** web build keeps the shift token in sessionStorage and the binding in localStorage (any script on the origin can read them): preview only; logout and shift end do **not** revoke the server session (token valid until expiry); staff token in memory only (forced re-login on every cold start); 60 s HTTP timeout on a door scanner | `02-…`, `03-…` |
| T19 | Low | Geofence is soft: client GPS is spoofable; no per-scan or IP check; silent no-op when the venue has no coordinates | doc + D-030 |

## Docs inside the threat model that are stale
Line 71 ("±2 windows") understates validity; line 73 (bare id never accepted) is wrong; the index list in §3.7 says "not committed" but all 8 indexes are committed; §3.6/§5 still describe the old HMAC manifest ("a device cannot admit anyone by itself"); no geofence, offline or key-exposure residual risks are recorded.
