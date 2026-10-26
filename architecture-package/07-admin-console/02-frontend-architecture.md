# 02. Frontend architecture

Stack: Next.js 16.3.0 App Router, React 19.2.8, TypeScript 5.9.3, Tailwind **4.3.3** (`@tailwindcss/postcss`, theme from `@c1rcle/tailwind-config/base.css`, plus an `urgent` colour token in `globals.css`), zod 4.4.3, **TanStack Query 5.101.4** (server state), react-hook-form 7.84 (listed, not used by admin pages), lucide-react 1.28.0, Vitest 4.1.10, Playwright 1.62.1. `AGENTS.md` warns that this Next version differs from model training data.

## Layout of `src/`
| Path | Purpose |
|---|---|
| `app/<desk>/page.tsx` | one client page per desk (all `'use client'`, TanStack Query); plus `layout.tsx`, `loading.tsx`, `error.tsx`, `not-found.tsx`, `globals.css` |
| `app/api/auth/{login,logout,refresh,session}/route.ts` | auth BFF |
| `app/api/health/{readiness,version}/route.ts` | health BFF |
| `components/app-shell.tsx` | sidebar, header, auth redirect, theme toggle, sign-out |
| `components/providers/session-provider.tsx` | session bootstrap, 30-minute idle logout, refresh on tab focus |
| `components/admin/` | `page-header`, `status-filter`, `overview-desk`, `onboarding-application-detail` |
| `lib/admin/` | `admin-api.ts` (1,311 lines), `contract-types.ts`, `describe-error.ts`, `document-window.ts`, `format.tsx` (labels, tones, `StatusBadge`, `formatPaise`, `shortId`) |
| `lib/api.ts` | gateway client singleton, BFF client, idempotency helpers |
| `lib/bff/` | `auth-proxy.ts` (cookie, CSRF, origin helpers), `health-proxy.ts` |
| `lib/app-meta.ts` | `NAV_SECTIONS`, app title |
The README mentions a `src/features/` folder that does not exist.

## Build and config
Dev `next dev --turbopack --port 3002`; start `next start --port 3002`. `next.config.ts`: `output: 'standalone'` except on Vercel; `poweredByHeader: false`; `optimizePackageImports` for ui/icons; `ignoreBuildErrors: false`; headers on `/:path*`: `X-Content-Type-Options`, `Referrer-Policy`, `X-Frame-Options: DENY`, `Permissions-Policy`. **No CSP and no HSTS.** `vercel.json`: `cd ../.. && pnpm turbo run build --filter=@c1rcle/app-admin-console`, `pnpm install --frozen-lockfile`. `Dockerfile`: node 22.22.2-alpine, `turbo prune`, four `NEXT_PUBLIC_*` build args, non-root uid 1001, `PORT=3002`, `node apps/admin-console/server.js`.
Env names: client `NEXT_PUBLIC_API_BASE_URL`, `NEXT_PUBLIC_APP_NAME`, `NEXT_PUBLIC_APP_ID` (must be `admin`), `NEXT_PUBLIC_ENVIRONMENT`, optional `NEXT_PUBLIC_SENTRY_DSN`; server `GATEWAY_READINESS_TOKEN` (optional, health BFF) plus `NODE_ENV`, `VERCEL_ENV`, `VERCEL_URL`, `SITE_URL`, `ANALYZE`. The shared schema also declares guest-portal URL and Firebase variables that this app does not use. Schema: `packages/config/src/schema.ts`.

## Authentication flow
1. `/login` → `login()` in `packages/auth/src/auth-client.ts` validates with `loginRequestSchema` and POSTs same-origin `/api/auth/login`.
2. BFF (`app/api/auth/login/route.ts`): `assertSameOrigin`, strips prototype-pollution keys, forwards to `${API_BASE}/api/v2/auth/login`; errors pass through. On success: returns body (with `accessToken`, `expiresAt`), **re-scopes `Set-Cookie`** to the frontend origin (HttpOnly, SameSite=Lax, `Secure` only when `NEXT_PUBLIC_ENVIRONMENT=production`, `__Secure-` prefix stripped) and mints a double-submit CSRF cookie `admin.c1rcle.csrf` (non-httpOnly, SameSite=Strict). The access token lives **only in memory** in the `@c1rcle/auth` store. The login page shows a generic "Invalid email or password" for any 4xx and then does a hard `window.location.assign('/')`.
3. BFF routes: `login` (origin check, no CSRF since no cookie exists yet), `logout` (origin + CSRF; best-effort gateway call; always clears cookies, 204), `refresh` (origin + CSRF), `session` (GET, origin). `auth-proxy.ts` re-adds the `__Secure-` cookie twin when forwarding. Origin rule: a present `Origin` must equal the app origin and `Sec-Fetch-Site: cross-site` is rejected; a missing `Origin` is allowed.
4. `layout.tsx` calls `getServerSession(cookies)` (gateway session endpoint) and passes `initialUser` to `AppShell`. If null on any route except `/login`, the shell shows "Redirecting to sign in…" and `router.replace('/login')`. **Gating is client-side only: there is no `proxy.ts` or `middleware.ts`.**
5. `SessionProvider`: seeds the store, calls `refresh()` to get an access token (failure marks anonymous); **30-minute idle logout** (pointerdown/keydown/visibility reset the timer); refresh on tab focus when under 5 minutes remain. Token expiry comes from the gateway (`expiresAt`).
6. **401 handling** (`lib/api.ts`): the gateway client is cross-origin from the browser and sends `Authorization: Bearer <in-memory token>`; `reauth: refresh()` retries once; a terminal 401 calls `logout()` and hard-redirects to `/login`.

### Role handling, a major gap
- The session user's `role` is `guest|partner|admin` (`packages/contracts/src/contracts/auth.ts`). The five admin roles exist only in `platformAdminDtoSchema` from `listAdmins`.
- **No nav item, page or action is gated by admin role.** `NAV_SECTIONS` is static; the overview card even states "No browser-side role checks decide what you can do".
- Consequence: any signed-in user (including a guest or partner account) passes the shell gate and sees the whole console; calls fail and pages show generic errors; there is no "not an admin" screen. A `support` admin sees buttons for refund approval and payout freeze and learns the refusal only from the failed call.
- `/settings` finds the admin's own role by scanning `listAdmins(100)`; this breaks past 100 admins and for roles refused the list.
- **Forced password rotation is unhandled.** `userSchema.mustChangePassword` exists and the gateway returns 403 on every non-auth route until rotation, but the console has no `/api/auth/change-password`, no change-password page, no forgot/reset routes, and never reads the flag. An admin provisioned with a temporary password would be stuck on 403s.

## API client layer
- `lib/api.ts`: `getAdminApiClient()` (singleton `createApiClient({getToken, reauth, onUnauthorized})`), `getAdminBffClient()` (same-origin, for `/api/health/*`), `newIdempotencyKey()` = `crypto.randomUUID()`.
- `withIntentKeys()` is a Proxy meant to reuse one key per method+path+body until success; it is exported and unit-tested but **never applied**. Every `admin-api.ts` mutation calls `newIdempotencyKey()` per invocation, so a click-retry sends a **new key**. UI copy such as "It is safe to retry: the request is idempotency-keyed" overstates the guarantee.
- `packages/api-client/src/client.ts` (the only module calling `fetch`): `x-request-id`, Bearer token, `x-organization-id` inferred from `organizations/<id>` in the path (so org suspend/reinstate carry it), 60 s timeout; GETs retry once with backoff and `Retry-After` (≤30 s); writes never auto-retried; zod-validates every response (a `parse` error mentions frontend/backend being out of sync and logs the raw payload to `console.error`); `fetchText` for CSV; SSE helper unused here.
- Errors: `describeAdminError` surfaces the gateway message and up to three field errors for status < 500, else a fallback. **Only onboarding, KYC and KYC-review use it**; other pages show hard-coded sentences, so role refusals, self-approval refusals and dual-control messages are lost.
- Contracts: `admin-api.ts` imports about 60 schemas from the generated `@c1rcle/contracts`; request bodies are `.parse()`d before sending (e.g. `proposeActionSchema`, `rejectRefundRequestSchema`, `requestRefundSchema`); responses validated with DTO/list schemas; enums derived from schemas (e.g. `ORDER_STATUSES`) except a few hand-written lists (`DisputeStatus`, `DisputeResolutionOutcome`, `PROMOTER_ASSIGNMENT_STATUSES`; `ADMIN_AUDIT_ACTIONS` lists 19 of 22 `AdminAction` members and is unused). Locally defined schemas: `refundOutcomeSchema`, `adminAuditPageSchema` (items only, no `pageInfo`), health `readinessResponseSchema` and `versionResponseSchema`.
- **Pagination/sort/search:** none. Every list uses `limit` (default 100; audit 200); `pageInfo.hasNextPage` is never read; no sort params. Server-side filters exist only for `status`/`state` (proposals, onboarding, refunds, disputes, payouts, reconciliation), support filters and audit `targetId`. Orders and tickets filter **client-side over the first 100 rows**. Support search and the audit target filter fire a request per keystroke (no debounce).

## Shared UI
`@c1rcle/ui`: `Button` (primary/outline/ghost/destructive, `aria-busy` when loading), `Card` family, `TextField` (label, hint, error, `aria-invalid`, `aria-describedby`), `LoadingState` (`role=status`), `EmptyState`, `ErrorState` (`role=alert`, retry, request id). Local: `PageHeader`, `StatusFilter` (native labelled `<select>`), `StatusBadge`, `OnboardingApplicationDetail`, `OverviewDesk`. Many pages hand-roll native `<select>`/`<textarea>` with divergent classes. State: TanStack Query keys `['admin', <desk>, …]` with `invalidateQueries` on mutation success; local `useState` per page; session store via `useSyncExternalStore`; theme from `@c1rcle/providers` (dark by default via a static inline script).
Tables are hand-written per page (`overflow-x-auto rounded-lg border`), mostly `<th scope="col">` (not on chargebacks and reconciliation); no sorting, pagination, sticky headers or bulk actions.
Accessibility: skip link, labelled primary `<nav>`, `aria-current`, `aria-expanded` mobile menu, `role=alert` banners. Weaknesses: colour-only status, inline confirm panels without focus management, mobile nav backdrop without focus trap/Escape, row buttons named only by document label, no `aria-live` on success messages.

## Health BFF
`app/api/health/{readiness,version}` call the gateway `/api/v2/internal/{readiness,version}` with the server-held `GATEWAY_READINESS_TOKEN` (`x-readiness-token`). These routes have **no session check and no origin check**: anyone who can reach the app can read dependency status, version and build SHA (operational detail, not secrets). The Health page refreshes every 30 s; checks: configuration, gateway, firestore, storage, redis, paymentProvider.
