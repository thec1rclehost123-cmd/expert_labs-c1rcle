# 03. Routes and screens

## Public and auth routes
| Route | Renders | Source / API | Gate |
|---|---|---|---|
| `/` | Landing page (CTAs "Already a User" / "Apply for Partner Access"; three.js scene skipped for reduced-motion, saveData, viewports <768 px) | none | none |
| `/login` | Two-step: pick Venue/Host/Promoter, then credentials | `@c1rcle/auth login` → `/api/auth/login`; `getOrganizations` → BFF; `/access` per org; `listMyInvitations` direct | Public; honours `?next=` and `?callbackUrl=` via `safeNextPath`; `mustChangePassword` → `/change-password`; zero orgs → pending invite else `/onboard?type=`; chosen workspace mismatch shows an error and stays signed in |
| `/signup` | Simple form then `/onboard` | `/api/auth/signup` | Public |
| `/onboard` | The wizard (see `05-…`) | BFF + Firebase + gateway | Public by design |
| `/forgot-password`, `/reset-password` | Request/complete reset | `/api/auth/*` | Public |
| `/change-password` | First-login rotation | `/api/auth/change-password` | Needs a session; **`next` is not validated** (open-redirect: `ChangePasswordClient.tsx:78` does `router.push(next)`) |
| `/invitations/[invitationId]/accept` | Auto-accepts when signed in, else sign-in CTA | `POST /api/v2/invitations/:id/accept` direct | Not gated (handles anonymous itself) |
| `/partner` | Client redirect to `/partner/<role>/overview`, `/login` or the picker | `useDashboardAuth` | Gated (cookie presence) |
| `/partner/select-organization` | Workspace picker (one org auto-picks; zero → invite or `/onboard`) | BFF organizations + access | Waits for `session.hydrated` |

## Studio routes `/partner/[studio]/…`
`[studio]/layout.tsx`: `notFound()` for any studio other than venue/host/promoter; loads **fixture** `getPartnerShellInteractions`; wraps children in `PartnerStudioFrame`. Pages are dynamic (nonce + cookies).

| Route | Studios | Source | Notes |
|---|---|---|---|
| `overview` | all | **Venue/host: LIVE** `loadOverviewData` → `GET /organizations/:id/analytics/{overview, trends×3, calendar, events}` (server, cookie). **Promoter: fixture** | `OverviewLoadFailureState`; `network`/`activity` passed as empty arrays; "upcoming events" links broken (below) |
| `events` | all | **Fixture** (`getVenueEvents`, `getHostEvents`, `getPromoterEvents`) | query `status`, `party`, `view`, `search`, `tab`/`city` (promoter) |
| `events/create` | venue, host | Live calendar/availability and connected promoters; static option data from fixture; submit **live** (see `06-…`) | other studios 404 |
| `events/[eventId]` | venue, host (promoter skeleton) | **Fixture**; `notFound()` for any id not in the fixture | real ids 404 |
| `events/[eventId]/edit` | venue, host | **Fixture**, `onSubmit` never passed: **"Save changes" does nothing** | |
| `events/[eventId]/{guests,promoters,sales,tonight}` | venue, host | **Fixture** | |
| `events/[eventId]/analytics` | all | Skeleton ("unavailable until the data contract is approved") | |
| `calendar` | venue, host | **LIVE, client-side** `apiClient`: venue `/venues`, `/venues/:id/calendar`, `/events`, block `POST /venues/:id/calendar/blocks`; host `/partnerships`, `/events`, `/venues/:id`, `/venues/:id/availability` | not in nav |
| `slot-requests` | venue (incoming), host (outgoing) | **LIVE** via BFF | promoter 404 |
| `partners` | all | **LIVE, client-side** (`StudioPartnersClient`) | tabs/views from query params |
| `marketing` | venue, host | **Fixture**; pager disabled | |
| `door` | venue, host | **Fixture, read-only** (no scanning, no mutations) | not in nav |
| `finance` | all | **LIVE, server-side** `loadFinanceData` | query `view`, `range`, `status`, `sort`, `direction`, `search`; payout request is a client POST |
| `guests`, `analytics`, `leaderboard` | promoter only | **Fixture** | others skeleton |
| `settings` | venue, host | **Fixture** (`FixtureBadge`, disabled controls) | staff management is in Partners → staff |
| `support` | venue, host | **LIVE** via BFF | promoter `notFound`; Help icon hidden for promoter |
| `presence` | n/a | **No page** though nav lists it | 404 |

Boundary files: `app/error.tsx`, `app/loading.tsx` (use `@c1rcle/ui`); `partner/error.tsx`, `[studio]/error.tsx`, `not-found.tsx` (use `partner-v3/States`); `partner/loading.tsx`, `[studio]/loading.tsx` and `LoadingState` all **return `null`** (no spinner); `events/[eventId]/not-found.tsx`.

## Known broken links
- `overview-view-model.ts:170` sets event card `href: /events/<id>` with **no `/partner/<studio>` prefix**; no such route exists (404). The hero card's "Guests" link has the same problem. The empty-state link (`:189`) is hard-coded to `/partner/venue/events/create` even for hosts.
- Calendar links to `/partner/venue/events/<realId>` 404 because event detail is fixture-only.
- Nav "Presence" has no page.

## Legacy trees (unreachable; redirected 307)
`/venue` (22 pages: door, events, create, analytics, `[eventId]/…`, finance, orders, marketing, notifications, overview, partners, settings, slot-requests, support), `/host` (22 pages incl. invitations, requests, earnings, partner profiles), `/promoter` (9 pages), `/partner-network/promoters/[promoterId]` (1). They import fixtures through `lib/partner/repositories.ts` (`partnerRepositories` → `fixtureHostRepository`, `fixturePromoterRepository`), carry duplicate auth-provider layouts and large legacy test suites. `canDo` is used **only** here. Safe to delete after redirects are retired.

## The data-source interface (`src/data/partner-data-source.ts`)
31 methods; the **only implementation is `fixturePartnerDataSource`**. `api-partner-data-source.ts` exports a single function `getStaffFromApi(orgId)` (members failure → empty roster with `staffAccess.error`; invitations 403 → roster with `canManage:false`; success → `canManage:true`; maps `member` to "Staff").
- **Used by V3 (fixture):** `getStudioShell`, `getPromoterOverview`, `getVenueEvents`, `getHostEvents`, `getPromoterEvents`, `getVenueEventDetail`, `getHostEventDetail`, `getPromoterGuests`, `getPromoterAnalytics`, `getPromoterLeaderboard`, `getVenueCalendar` and `getHostAvailability` (edit page only), `getVenueEventEditor`, `getHostEventEditor`, `getVenueMarketing`, `getHostMarketing`, `getVenueDoorMode`, `getHostDoorMode`, `getVenueSettings`, `getHostSettings`, `getPartnerShellInteractions`.
- **No V3 caller (superseded by live loaders, dead fixture data):** `getVenueOverview`, `getHostOverview`, `getVenuePartners`, `getHostPartners`, `getPromoterPartners`, `getVenueFinance`, `getHostFinance`, `getPromoterFinance`, `getVenueSlotRequests`, `getHostSlotRequests`.
- **`dataStatus`:** a view-model label shown by `FixtureBadge` only when `=== 'fixture'`. Vocabulary is inconsistent: shell `'fixture'|'api'|'unavailable'`; overview/partners/finance `'fixture'|'api'`; slot requests/calendar/availability `'fixture'|'live'`. Live mappers hard-code the value, so the badge disappears. There is **no environment flag** choosing fixture vs API; liveness is decided by which function a page imports.
