# 02. Architecture, auth and the BFF

## Stack (`package.json`)
Next 16.3.0 (App Router, Turbopack dev), React / React DOM 19.2.8, Tailwind 4.3.3 (`@tailwindcss/postcss`), zod 4.4.3, `qrcode.react` 4.2.0, `lucide-react` 1.28.0; **`@tanstack/react-query` and `react-hook-form` are declared but not used** by the portal; tests Vitest 4.1.10 (jsdom), Playwright 1.62.1, Testing Library. Workspace packages: `@c1rcle/{api-client,auth,config,contracts,design-system,hooks,icons,providers,types,ui,utils}`. `AGENTS.md` warns that this is "not the Next.js you know".

## Config
- `next.config.ts`: `poweredByHeader:false`; `htmlLimitedBots:/.*/` (metadata resolves before streaming so a real HTTP 404 is returned); `output:'standalone'` unless `VERCEL` is set; `optimizePackageImports` for ui and icons; images avif/webp, `minimumCacheTTL` 86400, **no `remotePatterns`** (remote event images use `unoptimized`, 7 places).
- Headers: global `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`, `X-Frame-Options: DENY`, `Permissions-Policy: camera=(), microphone=(), geolocation=()`; `/home/*` and `/app/screens/*` `public, max-age=604800, swr=2592000`; `/event/*` `public, s-maxage=300, stale-while-revalidate=3600`. **No CSP, no HSTS, no `redirects()`** (branch `origin/feat/app-shell-csp` exists, unmerged).
- `src/proxy.ts` (Next 16 proxy; matcher excludes `api`, `_next/static`, `_next/image`, `favicon.ico`): production **308** canonical redirect (`www.thec1rcle.com` or non-https → `https://thec1rcle.com`); private prefixes `/profile /tickets /help /checkout /confirmation`: GET/HEAD without cookie `better-auth.session_token` (or `__Secure-` twin) → **307 to `/login?next=<pathname only>`** (the query string is dropped, so `?tier=` is lost) with `Cache-Control: private, no-store` and `X-Robots-Tag: noindex, follow, noarchive`; with a cookie it sets private/no-store and noindex; non-production responses get `X-Robots-Tag: noindex, nofollow, noarchive`. The proxy trusts cookie **presence** only; real checks happen at the server or the API.
- `vercel.json`: framework nextjs, `cd ../.. && pnpm turbo run build --filter=@c1rcle/app-guest-portal`, `pnpm install --frozen-lockfile`. `Dockerfile`: node 22.22.2-alpine, turbo prune, standalone, non-root `nextjs`; build args `NEXT_PUBLIC_API_BASE_URL`, `NEXT_PUBLIC_APP_NAME`, `NEXT_PUBLIC_APP_ID=guest`, `NEXT_PUBLIC_ENVIRONMENT`, `SITE_URL`.
- Env names: client `NEXT_PUBLIC_API_BASE_URL`, `NEXT_PUBLIC_GUEST_PORTAL_URL` (default `https://thec1rcle.com`), `NEXT_PUBLIC_APP_NAME`, `NEXT_PUBLIC_ENVIRONMENT`, `NEXT_PUBLIC_APP_ID`, `NEXT_PUBLIC_SENTRY_DSN` (declared, no SDK), Firebase names (onboarding only); server `NODE_ENV`, `SITE_URL`, `VERCEL_ENV`, `VERCEL_URL`, `ANALYZE`. `.env.example` holds only the four `NEXT_PUBLIC_*` names.
- **Fonts:** the CSS names `'Satoshi','Inter'` (`globals.css:135`) but there is no `@font-face`, `next/font` or webfont link, so the UI falls back to system fonts. **Analytics:** none (no GA, PostHog or Sentry). **PWA:** manifest only.

## Sessions and auth
- **Login:** `login()` (`@c1rcle/auth`) POSTs `/api/auth/login`, returning `{user, accessToken, expiresAt}`. The BFF re-scopes the Better Auth cookies (`rescopeSessionCookies`, `auth-proxy.ts` ≈L282-312): strips `__Secure-`, drops Domain, imposes HttpOnly, SameSite=Lax and Secure (production only); the name is always `better-auth.session_token`; `forwardToGateway` re-adds the `__Secure-` twin; a decode-then-re-encode avoids double encoding. A CSRF cookie `guest.c1rcle.csrf` (JS-readable, SameSite=Strict, Secure in production, 32 random bytes base64url) is minted on login, signup or session fetch. The access token is **in memory only**; private BFF calls authenticate with the httpOnly cookie, not the token. Errors: 400/401 → "Incorrect email or password", 429 → "Too many attempts", 5xx → "temporarily unavailable".
- **Next-param safety** (`lib/auth/safe-next-path.ts`): rejects non-`/` starts, `//`, backslash, control characters, cross-origin parses, length > 512 and `/login*`, falling back to `/profile`; 10 unit tests; **no open redirect found**.
- **Signup:** no web UI (see README). The BFF `POST /api/auth/signup` still works and the gateway signup hard-codes `role: 'partner'`. `toUserDto` (`auth/index.ts` ≈L573) returns `role: 'partner'` for **every** user (comment: guest/admin auth are later phases), so the portal session user always says partner; a partner or admin can log in at the portal; a guest actor is `role:'member'` with an empty organisation. `SENSITIVE_COMMAND` is 10/min per IP+user+org.
- **OTP:** BFF routes `/api/auth/otp/send|verify` and gateway routes exist (limits 5 and 10 per minute), but **no portal UI uses them** (DEAD).
- **Forgot/reset:** forgot returns a constant ack (no account oracle); reset passes the gateway envelope through; the token travels only in the body.
- **Server bootstrap:** the root layout calls `getServerSession(cookies())` on every request (making all pages dynamic and adding a gateway round trip when a cookie exists). `SessionProvider` seeds the session, calls `refresh()` for a token and runs a **30-minute idle logout** to `/login`. `SessionBootstrap` (renew timer, expiry redirect with `reason=expired`) is **dead code**. `bffClient` on 401 clears the session and does `window.location.assign('/login')` without `next`.
- **Logout:** `/api/auth/logout` always expires the session and CSRF cookies; the server-side revoke is best effort.

## BFF routes (`src/app/api/**`)
All are no-store and forward the browser cookie through `forwardToGateway` (`redirect:'manual'`, `cache:'no-store'`). "same-origin" = `assertSameOrigin` (rejects a mismatched `Origin` or `Sec-Fetch-Site: cross-site`; a **missing Origin is allowed**). "CSRF" = double-submit cookie vs `x-csrf-token`.
| Method and path | Gateway target | Checks |
|---|---|---|
| POST `/api/auth/login` | `/api/v2/auth/login` | same-origin; `stripProtoKeys`; re-scopes cookies, mints CSRF; relays 429 Retry-After |
| POST `/api/auth/signup` | `/api/v2/auth/signup` | same-origin; no CSRF; 201 (UI never calls it) |
| POST `/api/auth/otp/send`, `/otp/verify` | `/api/v2/auth/otp/*` | same-origin + CSRF; **no UI caller (DEAD)** |
| POST `/api/auth/refresh` | `/api/v2/auth/refresh` | same-origin + CSRF; 401 clears cookies |
| POST `/api/auth/logout` | `/api/v2/auth/logout` | same-origin + CSRF; always 204 |
| GET `/api/auth/session` | `/api/v2/auth/session` | same-origin; 401 clears cookies; mints CSRF if absent |
| POST `/api/auth/forgot-password`, `/reset-password` | `/api/v2/auth/*` | same-origin; constant ack / pass-through; reset has no CSRF |
| POST `/api/rsvp` | `/api/v2/rsvp` | same-origin + session cookie + CSRF; **fresh random Idempotency-Key (the client key is not honoured)**; 201 |
| POST `/api/checkout/quote` | `/api/v2/checkout/quote` | `proxyGatewayWrite`: same-origin, cookie, CSRF, sanitised body; 200 |
| POST `/api/checkout/holds` | `/api/v2/checkout/holds` | same; honours a client `Idempotency-Key` matching `^[A-Za-z0-9_-]{8,128}$`, else random; 201; sets httpOnly cookie `c1rcle.payment-recovery-hold=<holdId>` (15 min, Lax) |
| POST `/api/payments/attempts` | `/api/v2/payments/attempts` | same; returns Razorpay order id, amount, keyId; re-sets the recovery cookie |
| POST `/api/payments/[id]/verify` | `/api/v2/payments/:id/verify` | id must match `^[A-Za-z0-9_-]{1,64}$`; same proxy; idempotent |
| GET and DELETE `/api/payments/recovery` | GET `/api/v2/payments/attempts/:holdId/status` (holdId from the cookie, validated) | same-origin only (**no CSRF even on DELETE**); DELETE clears the cookie; GET 404 `not_found` without a cookie |
| GET `/api/orders/[id]` | `/api/v2/orders/:id` | same-origin; id regex `^[A-Za-z0-9][A-Za-z0-9_-]*$`, max 64 |
| GET `/api/tickets/[ticketId]/qr` | `/api/v2/tickets/:id/qr` | same-origin; id regex; response always no-store |
| GET `/api/wallet/tickets` | `/api/v2/wallet/tickets?limit=` | same-origin; limit clamped 1–100 (default 50); no cursor passthrough |
| GET `/api/profile/me` | `/api/v2/profile/me` | same-origin; 404 passed through as `not_found` ("no profile yet") |
| GET and POST `/api/help/tickets` | `/api/v2/support/tickets` | GET forwards the **raw query string**; POST same-origin only; random Idempotency-Key; **no CSRF** |
| GET `/api/help/tickets/[ticketId]` | `/api/v2/support/tickets/:id` | **no same-origin check and no id validation** |
| POST `/api/help/tickets/[ticketId]/messages` | `/api/v2/support/tickets/:id/messages` | same-origin only; **no id validation**; no CSRF |
**Not present in the BFF:** profile write (`PUT /profile/me`), follows, notifications, cover wallets, orders list, refund requests; the promoter click is posted by the browser straight to the gateway (and nothing posts it, see `03-…`).
