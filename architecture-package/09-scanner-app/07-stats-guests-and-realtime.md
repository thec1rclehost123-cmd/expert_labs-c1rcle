# 07. Guest roster, stats, realtime and heartbeat

Code: BE `routes/v2/door/door-ops-routes.ts` (guests, attendance, heartbeat), `routes/v2/phase5-routes.ts` (stats, SSE), `lib/stream-limiter.ts`, `application/door/{door-ops-service,door-stats-service}.ts`; FE `app/(tabs)/{guests,stats}.tsx`, `src/features/heartbeat/useHeartbeat.ts`.

## Guest roster
- `GET /door/guests?eventId&search&status` (`venue.read`, `no-store`): server-side search, `truncated` flag, not-entered guests first. Bounded reads (up to 20k entitlements and 2k door sales). The roster includes refunded/void tickets as `not_entered`.
- App (`guests.tsx`): search debounced 300 ms; filters All / Entered / Not entered; shows "More results not shown - refine your search." when truncated and "No guests match." when empty; the "{n} on list" counter counts the filtered results, not the roster. Tapping the check button calls `POST /door/guests/check-in` if `canOverride(role)` (any role except `member`); for `member` it shows "Your role can't manually check guests in - tap does nothing." No loading indicator, no pull-to-refresh, no per-row busy guard and no idempotency key on the check button. Errors are inline; there is no 401 redirect here (the 60 s watchdog covers it).
- Manual check-in admits one seat via the same atomic claim and **bypasses the couple confirmation** (see `05-…`).
- `GET /door/attendance-report` (`venue.read`, `no-store`): attendance counts (couple headcount handled); exists on the backend, not used by the app.

## Stats
`GET /door/stats?eventId` and the SSE stream use the Better Auth actor and an organisation check **in the service**: no `requirePermission` and **no scanner token**.
- Computed per call: seven parallel `count()` queries by scan status; admissions via `getAdmissionStats` (`sum(admittedCount)` where `admittedCount > 0`, plus one sum per tier name from the catalog, remainder as `other`); door-sale headcount as the sum of `totalGuests` over active sales (`findByEvent`, limited to 1000); `inside = admitted + doorHeadcount`; occupancy returns capacity and remaining (null when no capacity configured), prebooked and door entries; `byEntryType` merges tier names with `walk-in` and `dine-in`; door-sale count/revenue from `doorSales.getEventStats`; wallet aggregates.
- **Cost and accuracy issues (reported):** `doorSales.getEventStats` reads every door-sale document and sums in Node, does not exclude voided/refunded sales, and contradicts the service's own "never a full document read" claim; at the SSE tick (3 s) that is a full scan per tick per open stream; stats are not cached; `admittedCount` includes override rows (`max(…,1)`), overridden staff-deny rows and door ticket sales; `doorHeadcount` caps at 1000 sales.

## SSE stream (`GET /door/stats/stream`)
`reply.hijack()` with manual CORS headers. Tick 3 s, keep-alive comment every 15 s, hard lifetime 15 minutes then `closed:{reason:'expired'}`; frames are sent only when content changes (`generatedAt` excluded); each tick re-authorises by calling `getStats` again; on failure `closed:{reason:'unavailable'}`. Budget (`stream-limiter.ts`): max **10 per actor** (key `orgId:userId`) and **500 global**, in-memory and per instance; over budget → 429 with `retry-after: 30`; a slow consumer whose `write()` returns false is dropped. nginx serves it unbuffered with a 20-minute read timeout. There is no WebSocket (a test asserts `/door/stats/ws` is gone).

## App stats screen (`stats.tsx`)
Two effects. (1) Poll `GET /door/sales` (walkin and dinein) every 15 s to derive door heads, tables and taken amount. (2) Prime with `fetchStats`, then open the stream (`openStatsStream`, `timeoutMs` 20 min, org header only); on close it reconnects after a **fixed 2 s, forever, with no backoff**. Subtitle shows the error or "Live" / "Connecting…". Hero card: inside / capacity / percent. "Tickets scanned" is derived as `inside - doorHeads`. Honest "not available" cards for entries per hour, gender split, average age and rejected.
**Not a bug (verified):** the stream call omits `X-Scanner-Session-Token`; the backend stats routes require none, so the explorer's concern that it would fail is moot. If those routes later add a token requirement, the stream will break.

## Heartbeat
`POST /door/heartbeat` (`SCANNER_COMMAND`, session + bound device) every **60 s** from the tabs layout (the docs say about 30 s); updates `lastSeenAt`, `lastEventId`, `lastGate`. Failures are swallowed; no AppState handling. The heartbeat is the only way a manager can see device liveness; the backend has `GET /door/devices` (`door.manage`) but the app and consoles do not surface it yet.

## Gaps
No hourly buckets, denied count or gender/age data in `GET /door/stats` (the app shows "not available"); no stats cache; SSE limits are per instance (a second instance needs Redis); the roster is readable by any organisation member (no per-guest permission); `GET /door/sales` has no real pagination.
