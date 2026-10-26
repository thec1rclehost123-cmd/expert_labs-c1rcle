# 01. API Gateway

**Path:** `C1RCLE-BACKEND/apps/api-gateway` · **Status:** Live (staging) · the only backend deployable.

## Purpose
HTTP surface of the platform under `/api/v2`. It authenticates, authorises, validates, rate-limits, then delegates to services in `packages/core`. Handlers are deliberately thin.

## Tech
Fastify 5, ESM, zod 4, Better Auth 1.x, `firebase-admin`, `ioredis`, `@fastify/cors`, pino. Runs via `node --import tsx src/server.ts` because `@c1rcle/core` ships as TypeScript source (only `contracts` has a `dist`). CI `pnpm build` is therefore the only place a core/gateway `tsc` break is caught.

## Key files
| File | Role |
|---|---|
| `src/server.ts` | Reads config, listens (`PORT`/`HOST`, default 8080), SIGTERM/SIGINT shutdown |
| `src/app.ts` | Pure factory `buildApp(options)`; tests use `app.inject` |
| `src/config/index.ts` | **Only** reader of `process.env`; zod-validated, fails fast |
| `src/plugins/{auth,rbac,rate-limit,cache,validate-v2,error-handler}.ts` | Cross-cutting plugins |
| `src/lib/v2-services.ts` | Singleton service container `createV2Services()` |
| `src/routes/v2/route-manifest.ts` | Registers every route group under `/api/v2` |
| `src/lib/{v2-idempotency,v2-response-validation,razorpay-webhook,stream-limiter,payouts-gate,readiness,redis,csv}.ts` | Helpers |

Production config refuses to boot without real `MAGIC_TICKET_SECRET`, `ENCRYPTION_KEY`, `EMAIL_OTP_SECRET`, and a non-localhost `REDIS_URL` when the Redis rate-limit store is on.

## Plugins
- **CORS**: exact origins, credentials on. Allowed headers: `Authorization`, `Content-Type`, `X-Organization-Id`, `X-Request-Id`, `X-Client-Request-Id`, `Idempotency-Key`, `If-Match`, `X-Scanner-Session-Token`.
- **validate-v2**: `validateV2({body, querystring, params, headers})`; failure is 422 with `fieldErrors`.
- **auth**: Better Auth + global `onRequest` session/membership hook (see `../04-auth-and-rbac`).
- **rbac**: `requirePermission(permission)`, RBAC + ABAC, default deny.
- **rate-limit**: sliding window keyed `class|ip|user|org`; memory store by default, Redis with `RATE_LIMIT_STORE=redis` (falls back to memory if Redis down).

| Class | /60 s |
|---|---|
| PUBLIC_READ | 120 |
| AUTH_READ | 240 |
| STANDARD_COMMAND | 60 |
| SENSITIVE_COMMAND | 10 |
| OTP_SEND | 5 |
| OTP_VERIFY | 10 |
| SCANNER_COMMAND | 300 |

- **cache**: in-memory, tenant-scoped GET cache; org taken from verified actor, never the header. TTLs: org/venue profile 5 min, event preview 60 s, availability 30 s.
- **error-handler**: `DomainError` to flat envelope; 5xx messages redacted; unknown route returns 404 envelope (never 501).

Policy order per route (D-012): `rateLimit -> validateV2 -> requirePermission -> cached`.

## Endpoint groups (`/api/v2`)
Route files: `src/routes/v2/<group>/`. Paths built from `${action}` templates are shown with placeholders.

| Group | Representative endpoints |
|---|---|
| internal | `GET /internal/health`, `/internal/version`, `/internal/readiness` (last two token-gated at nginx) |
| auth (OTP and RSVP detail: `../13-reference/DEEP-DIVE-OTP-RSVP-CONTRACTS.md`) | `POST /auth/signup｜login｜forgot-password｜reset-password｜refresh｜logout｜change-password`, `GET /auth/session`, `GET /auth/invitations/mine`, `POST /auth/otp/send｜verify` |
| public (no auth) | `/public/events`, `/events/:idOrSlug`, `/events/:idOrSlug/tiers`, `/venues/:slug`, `/hosts/:slug`, `/discovery`, `/search`, `/promoter-links/:handle/:slug`, `POST /promoter-links/click` |
| organizations | org CRUD, members, `/access`, invitations (`revoke`, `accept`) |
| venues | CRUD, profile, calendar blocks, menu, availability, slot requests (accept/reject/cancel) |
| events | CRUD, lifecycle `review｜publish｜pause｜resume｜duplicate`, `cancel`, `previews`, poster `upload-url` |
| event catalog | `ticket-tiers`, `promo-codes`, `table-packages`, `promoter-assignments`, `referral-links` |
| partner network | `/partnerships`, `/promoter-connections`, `discover-partners`, `analytics/*`, org `notifications` |
| onboarding/KYC | `/onboarding/me`, `/onboarding/applications` (+ document upload/verify/submit), `/onboarding/verify-document` |
| guest profile | `GET/PUT /profile/me` |
| checkout & payments | `POST /checkout/quote｜holds`, `/rsvp`, `POST /payments/attempts`, `GET /payments/attempts/:holdId/status`, `POST /payments/:id/verify`, `POST /webhooks/payments/razorpay` |
| orders/tickets/wallet | `/orders`, `/orders/:id`, `/tickets/:id`, `/tickets/:ticketId/qr`, `/wallet/*` (ticket transfer deliberately unregistered) |
| door & scanner | door codes, `/door/sessions`, `/door/check-ins` (+verify/confirm), `/door/lookup`, `/door/override`, `/door/staff-deny`, `/door/offline-manifest`, `/door/offline-sync`, `/door/devices`, `/door/heartbeat`, `/door/guests`, `/door/walk-in`, `/door/dine-in`, `/door/sales`, `/door/ticket-sale`, `/door/wallet-qr`, `/door/wallet-charge`, `/door/stats`, SSE `/door/stats/stream`, `/cover-wallets/*` |
| finance (org) | `/organizations/:org/finance/*`, `payouts`, `bank-accounts`, `disputes`, `leaderboard` |
| social/notifications | `/follows*`, `/notifications/me*` |
| support (guest) | `/support/tickets*` |
| admin | ~17 route files under `/admin/*`: onboarding review, `proposals` (dual control), `admins`, `audit`, `refunds`, `payouts`, `disputes`, `chargebacks`, `reconciliation/*`, directory lists (`venues｜events｜hosts｜users｜orders｜tickets｜promoters｜promotions`), suspend/ban desks, `settings/platform`, support desk, `*/export.csv` |

`docs/api-contracts/openapi.yaml` covers only ~11 paths; **the zod contracts and route files are authoritative**.

## Background work
Firestore driver only, every 5 minutes: sweep expired cart reservations and scanner sessions; refresh the analytics read model. Domain events go through an outbox (`v2_outbox`) with leases, retry backoff, 5 attempts then dead-letter; delivered in-process (no external queue).

## Realtime
Server-Sent Events only (`/door/stats/stream`, decision D-028). nginx serves it unbuffered with a 20-minute read timeout. No WebSocket.

## Testing
Vitest 4; each route test builds the full app via `buildApp()` + `app.inject()` on the memory driver. Scenario suite `pnpm test:scenarios` (merge-only). Firestore integration tests run against the emulator in CI. Coverage ratchet (about 54% lines baseline).

## Status
| Item | Status |
|---|---|
| Routes listed above | Live on `staging`, including the payments correctness stack and the financial engine (merged via #135 and #136); see `../10-payments-and-finance/09-…`. A **generated index of all 275 route registrations** with permissions and idempotency is in `../13-reference/API-ROUTE-INDEX.md` |
| Rate limiter / cache | Live, **per-process memory** unless Redis configured; single free-tier instance today |
| Chat/DM | Planned (not started) |
| Ticket transfer | Deliberately not registered |

Further reading: `docs/architecture/README.md`, `docs/architecture/decisions.md`, `docs/api-contracts/*`.
