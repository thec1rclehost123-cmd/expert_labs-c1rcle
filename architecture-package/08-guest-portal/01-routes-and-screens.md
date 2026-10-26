# 01. Routes and screens

Legend: **LIVE** = real gateway data, **FIXTURE** = static data, **DEAD** = unreachable or unused.

## Pages
| Route | Purpose and data | Auth gating | Caching and states |
|---|---|---|---|
| `/` | Marketing hero (`homeFixture.hero`, always fixture). Production: "Upcoming events" is LIVE (first 3 of `getPublicEventsForSitemap()`, title and summary only). Non-production: `HomeFeaturedDropsClient` and `HomeFeaturedEvents` use `homeFixture` and `exploreFixture`. Organization and WebSite JSON-LD | public | `revalidate=300` but the root layout calls `cookies()`, so the page is dynamic and the value is moot; production with no events omits the section |
| `/explore` | LIVE: `GET /api/v2/public/discovery` plus a per-venue fan-out `GET /public/venues/by-id/:id`, mapped by `toExploreEvent`. Filtering is **client-side** over the feed (search, city, date, sort trending/soonest/newest/price-low) | public | `loading.tsx`, `error.tsx`; any API error returns `[]` (empty state, never a fixture); no pagination; the discovery feed is capped at 20 server-side; the canonical URL is hard-coded |
| `/event/[eventId]` | LIVE: `GET /public/events/:idOrSlug` (id or slug), then venue and host by id; mapped by `toEventDetailFixture` | public | `not-found.tsx`, `error.tsx`; `notFound()` on any fetch error; CDN header `s-maxage=300` from `next.config` |
| `/checkout/[id]` | Fixture slugs first (`neon-nights`, `rooftop-jazz`, `techno-bunker`, `art-collective`, `indie-jam`, `sunday-soul`, …), else LIVE (event, venue, `/events/:id/tiers`); `purchasable = tiers loaded` | proxy-gated (cookie presence only, not validity) | `loading`, `error`, `not-found`; noindex; **fixture ids are reachable in production** |
| `/confirmation/[id]` | Fixture ids `preview-<fixture slug>` first, else LIVE `GET /orders/:id` with the forwarded cookie plus event and venue; `generateStaticParams` returns the fixture ids | proxy-gated; the order read is buyer-scoped (a stranger gets 404) | `notFound()` on any failure |
| `/tickets` | `TicketsClient`: authenticated → LIVE wallet (`/api/wallet/tickets?limit=50`, then public event and venue reads **from the browser**); anonymous → FIXTURE showcase (`TicketsGuestView`) | proxy-gated, so the guest view is normally unreachable (a stale cookie can reach it) | a wallet failure renders as empty, never a fixture |
| `/profile` | `requireGuestSession('/profile')` (server session) then `ProfileClient` reading `/api/profile/me` and the wallet | proxy plus server redirect to `/login?next=` | `force-dynamic`; **production returns `PrivateDataUnavailable` ("Profile unavailable")** (`profile/page.tsx:54`), live only in preview and dev |
| `/profile/[userId]` | FIXTURE public profile; 404 in production | public | `force-dynamic` |
| `/help` | `requireGuestSession`, then `HelpClient` talking to the help BFF; **production returns "Help unavailable"** | proxy plus server | `force-dynamic` |
| `/hosts` | FIXTURE directory in non-production; **empty lists in production** (`hosts/page.tsx:21-28`); noindex | public | no data in production |
| `/host` | `permanentRedirect('/hosts')` | | |
| `/host/[hostId]`, `/venue/[venueId]` | Production: LIVE `GET /public/hosts/:slug` and `/public/venues/:slug` (return null when not production SEO) rendered by `AuthoritativeHostView` and `AuthoritativeVenueView` (name and description only); non-production: FIXTURE | public | `force-dynamic`; production 404 if the API fails; `indexable` is false for both |
| `/[promoterHandle]/[vanitySlug]` | LIVE: `GET /public/promoter-links/:handle/:slug`, then `redirect('/event/<slug>?ref=<code>')` (307); 404 on failure | public | any other unknown two-segment path hits this route and costs one API call before 404 |
| `/login` | email and password; `?next` through `safeNextPath`; `?reason=expired` banner | public | |
| `/signup` | `redirect('/login?mode=signup')`; the login UI says "Guest sign-up is not available on the web yet… create your account in the mobile app" | | |
| `/forgot-password`, `/reset-password` | forms; reset sets `referrer: no-referrer` and strips the token from the URL (e2e verified) | public | |
| `/app`, `/about`, `/privacy`, `/terms` | `/app` is a marketing page (phone story, fixture); `/about`, `/privacy`, `/terms` are **title-only stubs** with no policy text, `indexable:false` | public | |
| `not-found.tsx`, `error.tsx`, `manifest.ts`, `robots.ts`, `sitemap.ts` | global states via `@c1rcle/ui`; PWA manifest (standalone, black, a single 128 px webp icon, no service worker) | | |

## Fixture vs live matrix
| Surface | Production | Preview / dev |
|---|---|---|
| `/` | hero fixture + live events | all fixture |
| `/explore`, `/event/*` | live | live |
| `/hosts` | empty, noindex | fixture |
| `/host/:slug`, `/venue/:slug` | live (minimal) | fixture |
| `/profile/:userId` | 404 | fixture |
| `/profile`, `/help` | **blocked ("unavailable")** | live |
| `/tickets` (signed in) | live | live |
| `/tickets` (anonymous) | fixture showcase, normally unreachable via the proxy | same |
| `/checkout/<fixture slug>`, `/confirmation/preview-*` | **fixtures reachable (no production gate)** | same |
| `/checkout/<real>`, `/confirmation/<orderId>` | live | live |
| `/app`, `/about`, `/privacy`, `/terms` | static marketing or stubs | same |

## Feature folders (`src/features`)
`auth`, `booking`, `directory`, `event-detail`, `explore`, `home`, `profile`, `tickets` (each with `components`, `fixtures`, `types`); `src/components/{layout,private,providers,seo}`, `app-shell.tsx`, `session-bootstrap.tsx`; `src/lib/{api,auth,bff,seo}`. A stale `docs/frontend-backend-handoff.md` still claims checkout, payments, orders and wallet are fixtures: **it is out of date**.
