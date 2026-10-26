# 02. Frontend architecture

Stack: Next.js 16.3 App Router, React 19.2, TypeScript 5.9, Tailwind **3.4** (other Next apps use v4), zod 4, Vitest 4 + Playwright, firebase 12 (phone OTP only), framer-motion + three.js (landing/onboarding). `next build --webpack`; dev uses Turbopack. **No TanStack Query, no react-hook-form usage** (both listed in dependencies but never imported in this app).

## 1. `src/` layout
```
src/
  app/                  App Router
    api/auth/*          auth BFF route handlers (login, signup, logout, refresh, session, change-password,
                        forgot-password, reset-password, otp/send|verify, phone-verification)
    api/bff/*           domain BFF handlers (organizations, organizations/[id]/access, onboarding/*,
                        slot-requests/*, notifications, support/tickets/*)
    partner/            V3 entry + [studio] shell  (THE live product)
    login signup onboard forgot-password reset-password change-password invitations/   public/auth screens
    venue host promoter partner-network/   LEGACY pages (redirected away; dead)
    layout.tsx page.tsx(landing) error.tsx loading.tsx not-found.tsx
  components/
    partner-v3/         live UI (about 162 files, 32.5k lines): shell, primitives, per-feature folders
                        (calendar door event-detail event-editor events finance marketing overview
                         partners promoter settings slot-requests)
    partner-support/    support screen (live)
    partner-shell/      LEGACY (only partner-role-routing.ts and types.ts still used by V3)
    venue/ host/ promoter/   LEGACY (dead, large; tests still run)
    providers/          DashboardAuthProvider, SessionProvider, PasswordChangeGuard,
                        session-context.tsx (DEAD)
    landing/ ui/        public landing page (three.js scene), WebGL error boundary
  data/                 partner-data-source.ts (types + 31-method interface),
                        fixture-partner-data-source.ts (3,136 lines), api-partner-data-source.ts (staff only)
  lib/                  access/ api/ bff/ calendar/ events/ finance/ firebase/ onboarding/ org/ overview/
                        partner/ rbac/ slot-requests/ support/ venue/  + safe-next-path.ts
  studios/studio-config.ts    nav per studio
  styles/partner-v3.css
  proxy.ts              Next 16 "proxy" (middleware)
```
The README mentions a `features/` folder; it does not exist. `src/lib/rbac/` holds only types; permission logic is spread across `lib/access/*`, `DashboardAuthProvider` and the `/access` DTO.

## 2. Provider tree
`app/layout.tsx` (server): awaits `getServerSession(cookies)` (calls gateway `GET /api/v2/auth/session`), renders `<html class="dark">` (dark is hard-coded) and wraps children in
`SessionProvider(initialUser)` → `DashboardAuthProvider` → `PasswordChangeGuard`.
No query provider, no theme provider. `partner/layout.tsx`, `onboard/layout.tsx` and the legacy layouts mount **another** `DashboardAuthProvider` inside the root one (duplicate org/access fetches).

- **`SessionProvider`**: seeds the in-memory store with the server user, calls `refresh()` via the BFF to obtain the access token, then `markHydrated`. Idle logout after **30 min** of no pointer/key activity; re-refresh on tab focus when the token expires within **5 min**.
- **Session store** (`@c1rcle/auth`, `packages/auth/src/session-store.ts`): module-level `useSyncExternalStore`; access token is **memory-only**; `status` = unknown | authenticated | anonymous; `hydrated` is a one-time flag that every data hook waits on to avoid pre-refresh 401s.
- **`DashboardAuthProvider`** (`components/providers/DashboardAuthProvider.tsx`): reads `activeOrgId` once from the cookie (lazy `useState`, so switching org needs a reload); on auth loads `getOrganizations()` and `getMyOnboardingRequest()` in parallel, then **N+1** `getPartnerAccess(org.id)` calls to build `memberships` (an org whose access call fails is silently dropped); auto-selects when exactly one org and no cookie; `isApproved = onboarding.status === 'approved' || memberships.length > 0`; `isBanned` always false; `signInWithGoogle` always rejects; shows a full-screen "Authorizing Access" spinner on every path except `/onboard*` and `/login*`; `user` typed `any`; context shape kept for about 41 consumers.
- **`PasswordChangeGuard`**: redirects to `/change-password?next=<path>` when `mustChangePassword`; exempt `/change-password`, `/login`, `/signup`.

## 3. Studio shell
`PartnerStudioFrame` (client): no user → `/login?callbackUrl=…`; not approved → `/onboard`; no recognised role → `/partner/select-organization`; URL studio ≠ `activeMembership.partnerType` (normalised, `club`→`venue`) → "not available for this account"; tab key `false` in `tabVisibility` → "isn't available for your role". **These guards are client-only**: server pages do not verify the studio/partner-type match (e.g. `/partner/venue/finance` for a host org still renders that org's live finance).
`PartnerShell`: filters nav through `visibleStudioNavigation(config, tabVisibility)`; side or top navigation stored in `localStorage["c1rcle.partner.navigation-layout"]`; mobile drawer; sign-out. `PartnerTopbar`: search and notifications (both **fixture**), Help link (not for promoter), layout toggle, account menu (sign-out only).
**There is no org/workspace switcher in V3.** The only way to change org is `/partner/select-organization` (reached via login, onboarding or redirects).

`SEGMENT_VISIBILITY_KEYS` (`lib/access/studio-tab-access.ts`): overview, events, calendar, door, partners, presence, finance, support, settings, guests, analytics. `slot-requests`, `marketing`, `leaderboard` have **no key** and are never hidden. Rule is fail-open: unknown/`null` access shows everything.

## 4. Active organisation
- Cookie `c1rcle.active-org`: written client-side by `setActiveOrg` (`lib/org/active-org.ts`), `path=/; SameSite=Lax; max-age=1y`, `Secure` when `NEXT_PUBLIC_ENVIRONMENT === 'production'`; **not httpOnly**; an id hint only. `active-org-cookie.ts` is the RSC-safe reader (kept separate so server code does not import the `@c1rcle/auth` barrel).
- `login()`/`signup()` clear it so one account's id never leaks to another.
- `resolveLandingPath(orgId)` (`lib/org/route-after-auth.ts`): fetches `/access`, picks the studio from `partnerType`, lands on the first nav item not withheld by `tabVisibility`; fallback `/partner/select-organization`.
- `routeAfterAuth`: zero orgs → pending invite or `/onboard`; one org → set and go; several → picker. `/partner/select-organization` uses a **full page load** to `localStorage["partner:last-route:<orgId>"]` or the landing path.
- `resolveBrowserOrganizationId`: fallback picking the first org (sorted id) matching the partner type.

## 5. Two network paths from the browser
| | BFF path | Direct gateway path |
|---|---|---|
| Client | `bffClient` (`lib/bff/bff-client.ts`) | `apiClient` (`lib/api/client.ts`) |
| Auth | same-origin cookie + CSRF header, no bearer | `Authorization: Bearer <in-memory token>`; auto-adds `x-organization-id` when the path contains `organizations/<id>` |
| On 401 | clears session, hard-redirect to `/login` | one silent `refresh()` and replay, then clear and redirect |
| Used for | orgs, access, onboarding, slot requests, support | calendar, availability, partners/partnerships/connections, staff, event publish workflow, poster upload URL, payout request, phone `verify-document`, `listMyInvitations`, `acceptInvitation` |
The direct path requires CSP `connect-src` to include the gateway origin and **gateway CORS to allow the dashboard origin** (a hard dependency). Server components use `createServerApiClient(cookieHeader)` (`lib/api/server-client.ts`; `get`/`post` only). Core client (`packages/api-client`): 60 s timeout; GET retried at most once; writes never auto-retried; `x-request-id` per call; zod-validated responses; 401 re-auth replay; SSE helpers.

## 6. `proxy.ts`
- Edge **cookie-presence** check: a GET to a gated prefix without `better-auth.session_token` (or `__Secure-` twin) → 307 `/login?next=<pathname>` (query string dropped). Gated prefixes: `/venue /host /promoter /partner /partner-network` (legacy ones are redundant since redirects run first). `/onboard` and `/invitations` are intentionally public. This is UX only; the gateway re-validates every call.
- Mints a per-request nonce (`base64(randomUUID)`) as `x-nonce` request header and in the CSP response header (also on redirects). The nonce forces dynamic rendering app-wide (no static/ISR/PPR).
- CSP: `default-src 'self'`; `script-src 'self' 'nonce-…' 'strict-dynamic'` (+`'unsafe-eval'` in dev); `style-src 'self' 'unsafe-inline'`; `img-src 'self' blob: data: https:`; `font-src 'self'`; `connect-src 'self' blob: <gateway origin> https://storage.googleapis.com https://*.googleapis.com https://www.google.com https://www.gstatic.com`; `object-src 'none'`; `base-uri 'self'`; `form-action 'self'`; `frame-src 'self' https://www.google.com/recaptcha/ https://recaptcha.net/recaptcha/`; `frame-ancestors 'none'`; `upgrade-insecure-requests`.
- Matcher covers the gated prefixes, `/onboard`, and a catch-all excluding `api`, `_next/static`, `_next/image`, `favicon.ico`; so **API routes get no CSP**.

## 7. `next.config.ts`
Headers: `X-Frame-Options: DENY`, `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`, `Permissions-Policy: camera=(), microphone=(), geolocation=()` (blocks any future in-browser door scanning), HSTS (declared twice, identical). CSP deliberately lives only in the proxy to avoid two CSP headers intersecting. `poweredByHeader: false`; `typescript.ignoreBuildErrors: false`; `output: 'standalone'` only when `VERCEL` is unset; `optimizePackageImports` for `@c1rcle/ui` and `@c1rcle/icons`; images avif/webp with no `remotePatterns`.
`redirects()`: **21 entries, all 307** (`permanent: false`, kept reversible on purpose). Examples: `/venue/events/analytics`→`/partner/venue/analytics`, `/venue/events/:id/{finance,marketing}`→event detail, `/*/notifications`→overview, `/host/events/{invitations,requests}…`→`/partner/host/events`, `/promoter/links`→promoter overview, `/partner-network*`→`/partner`. Pinned by `legacy-redirects.test.ts`.

## 8. Cookies and CSRF (`lib/bff/auth-proxy.ts`)
- Session cookie `better-auth.session_token` is re-scoped to the frontend origin (`rescopeSessionCookies`): HttpOnly, SameSite=Lax, `Secure` in production, `Domain` dropped, `__Secure-` prefix stripped; decoded once to avoid double-encoding. `forwardToGateway` re-emits a `__Secure-` twin to the gateway; uses `redirect:'manual'`, `cache:'no-store'`.
- CSRF: double-submit. Cookie `<NEXT_PUBLIC_APP_ID>.c1rcle.csrf` (here `partner.c1rcle.csrf`), non-httpOnly, SameSite=Strict, `Secure` in production, 32 random bytes base64url; set on login, signup and `onboarding/me` if absent; `assertCsrf` compares cookie and `x-csrf-token` header with a timing-safe comparison; client helpers `csrfHeaders()` (`@c1rcle/auth`) and `lib/onboarding/csrf.ts`.
- `assertSameOrigin`: rejects a present `Origin` that differs, or `Sec-Fetch-Site: cross-site`; a missing `Origin` is allowed.
- Helpers: `assertPathId` (`^[A-Za-z0-9_-]{1,128}$`), `stripProtoKeys`, `ifMatchHeader` (positive integers only), `mintIdempotencyKey`, `passThroughGatewayError` (keeps the flat envelope and `Retry-After` on 429), `putToStorage` (server-side PUT to a signed URL; `memory://` is a no-op for dev).

## 9. BFF route handlers (`src/app/api`)
Same-origin check on everything; CSRF where marked.
| Route | Method | Gateway target | CSRF | Notes |
|---|---|---|---|---|
| `auth/login` | POST | `/api/v2/auth/login` | no | strips proto keys; rescopes cookies; mints CSRF; 200. Does not catch gateway network errors (unhandled 500) |
| `auth/signup` | POST | `/api/v2/auth/signup` | no | same; 201 |
| `auth/logout` | POST | `/api/v2/auth/logout` | yes | gateway failure tolerated; always 204; clears cookies |
| `auth/refresh` | POST | `/api/v2/auth/refresh` | yes | rescopes cookies |
| `auth/session` | GET | `/api/v2/auth/session` | n/a | 401 → `unauthorized` envelope |
| `auth/change-password` | POST | `/api/v2/auth/change-password` | yes | rescopes (gateway rotates sessions) |
| `auth/forgot-password` | POST | `/api/v2/auth/forgot-password` | no | constant 200 ack; only 400/422/429 keep detail, others become 502 |
| `auth/reset-password` | POST | `/api/v2/auth/reset-password` | no | returns `{status:true}` |
| `auth/otp/send`, `otp/verify` | POST | `/api/v2/auth/otp/{send,verify}` | yes | session-scoped email OTP |
| `auth/phone-verification` | POST | `/api/v2/onboarding/verify-document` (phone) | yes | **no UI calls it** (wizard goes direct) |
| `bff/organizations` | GET | `/api/v2/organizations` | n/a | forwards cookie and Authorization |
| `bff/organizations/[id]/access` | GET | `/organizations/:id/access` | n/a | `assertPathId`, sets `x-organization-id` |
| `bff/onboarding/me` | GET | `/onboarding/me` | n/a | mints CSRF if missing |
| `bff/onboarding/applications` | POST | `/onboarding/applications` | yes | forwards `Idempotency-Key`; 201 |
| `bff/onboarding/applications/[id]` | PATCH | `…/applications/:id` | yes | `If-Match` |
| `bff/onboarding/applications/[id]/submit` | POST | `…/submit` | yes | `Idempotency-Key`, `If-Match` |
| `bff/onboarding/applications/[id]/documents/upload` | POST | `…/documents/upload-url` → server PUT to storage → `…/documents` | yes | label validated; jpeg/png/webp else 415; max 5 MB (header and actual bytes, else 413); whole body buffered |
| `bff/onboarding/verify-document` | POST | `/onboarding/verify-document` | yes | **unused by UI** |
| `bff/slot-requests` | GET | composed reads (below) | n/a | requires `direction` (incoming/outgoing) and `organizationId` |
| `bff/slot-requests` | POST | `POST /organizations/:org/events` then `POST /venues/:venue/slot-requests` | yes | validates draft shape and `idempotencyKey` (`^[A-Za-z0-9_-]{8,128}$`); two writes use `${key}-event` and `${key}-slot-request` |
| `bff/slot-requests/[id]/{accept,reject,cancel}` | POST | `/venues/slot-requests/:id/{…}` | yes | `organizationId` query; a **fresh** idempotency key per BFF call |
| `bff/notifications` | GET | derived from slot-request statuses | n/a | **no UI caller** |
| `bff/support/tickets` | GET, POST | `/api/v2/support/tickets` | **POST: no** | POST has same-origin only, no CSRF, no `stripProtoKeys`; fresh idempotency key per request |
| `bff/support/tickets/[id]` | GET | `/support/tickets/:id` | n/a | `assertPathId` |
| `bff/support/tickets/[id]/messages` | POST | `…/messages` | **no** | no CSRF |

Slot-request read composition (`lib/bff/slot-request-gateway.ts`): **incoming** (venue): `GET /organizations/:org/venues` → per venue `GET /venues/:v/slot-requests` → detail per request (concurrency 8). **Outgoing** (host): `GET /organizations/:org/slot-requests` plus bulk `/venues` and `/events` enrichment. 10 s gateway timeout → 504. Wire status `accepted` maps to UI `approved`. Ticket tiers/promos/tables are intentionally excluded.

## 10. Shared UI and state
- Primitives (`partner-v3/index.ts`): Avatar, Badge, Button, Drawer, FixtureBadge, IconButton, PageContainer, PageHeader, PageSection, SearchInput, EmptyState, ErrorState, LoadingState (**renders nothing**; the `label` prop is ignored), Skeleton; plus MobileNavigation, PartnerGlobalSearch, PartnerNotifications, StudioRoutePlaceholder. Styling: CSS modules + `styles/partner-v3.css` + `globals.css`; legacy/auth pages use Tailwind with CSS variables. `@c1rcle/icons` is used heavily; `@c1rcle/ui` only in 3 root boundary files; `design-system`, `utils`, `providers`, `hooks` essentially unused.
- Data fetching: server pages read via loader functions (`loadOverviewData`, `loadFinanceData`) with typed failure objects; client screens use bespoke hooks (`useVenueCalendar`, `useHostAvailability`, `useConnectedPromoters`) built on `useState`+`useEffect`, `AbortController`, a request key and a retry counter, gated on `session.hydrated`. No caching, dedup or stale-while-revalidate. `router.refresh()` after finance writes.
- Errors: server loaders return typed failures (`signed-out`, `no-organization`, `forbidden`, `api`) rendered by `*LoadFailureState` with a retry (`router.refresh()`); client hooks return `{data, error, loading, retry}`; `describeApiError` appends up to 3 `fieldErrors` and reduces a `ZodError` to its first issue.
- **Idempotency pattern:** per-intent `Idempotency-Key` held in a ref and reused across retries (payout, partner actions, publish workflow, block date, onboarding). Exceptions that mint a fresh key per call: support tickets/messages, slot-request actions, `staffApi`.
