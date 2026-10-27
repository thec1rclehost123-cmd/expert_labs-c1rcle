# 13. Status, gaps and roadmap (candid)

Snapshot 2026-10-24. **Keep under NDA.** "Verified" = checked in source this session; "reported" = from a code read, not independently re-checked.

## Branch and PR state
- **Frontend** (`C1RCLE-FRONTEND`): checked-out `fix/partner-connections-contracts` at `c1f4d54`; 0 ahead / 1 behind `origin/staging` (verified). PR #77 (`ca9a08d`: live profile, real entry QR, BFF onboarding, fixture disclosure) is merged on staging. Not on staging: `feat/dashboard-api-gateway-integration` (9 commits, reported: v2 gateway integration, onboard migration, login gate fix, api-client hardening, `activeMembership` schema, `refreshSession`) and `feat/partner-notifications-inbox` (1 commit `b7ebe48`). Merged earlier: Partners tab (#58–#61, #63, #64), staff roster/edit-role/reinvite (#66, #76, #80), GST on finance orders (#71).
- **Backend** (`C1RCLE-BACKEND`): checked-out `fix/partner-connections-and-events` at `4bcc15a`; 6 ahead / 9 behind `origin/staging` (verified). Its 6 commits: counterparty validation + notification target, tenant-scoped previews + unified create contract, route-manifest permission docs, analytics read-model projection (M4), door/scanner parity cases, rebase repair. Open and unmerged: #109 (business/signatory onboarding requirements), #110 (staff invitations; the routes already appear live, so the PR state is a doc inconsistency), #46 (calendar changes, 2 ahead). Payments PRs are tracked in `../12-status-and-known-issues`.
- PR numbers and counts move; re-run `git branch -a` and `git rev-list --left-right --count origin/staging...HEAD` before quoting.

## Task list (`task.md`, audit 2026-10-11): all ticked
A1 (`canDo` hard-coded true: fixed with `ACTION_PERMISSIONS`; verified in code), A2 (nav permission strings: fixed; verified), A3, A4 (door/cover-wallet mutations lacked `requirePermission`), A5 (venue MANAGER finance tab), A6 (`tenancyRoleToPartnerRole`), B1 (idempotency claim released on error → now FAILED + 409), B2 (fresh key per attempt → per intent), C1–C3 (contract parity, 186 checks then 226), D1–D4 (terminal events editable, door-sale race, same-state saves, stubs), E1–E3 (duplicate route, dead handlers kept after audit, dead v1 repository deleted: verified). Note A1/A2 fixed legacy code paths that V3 no longer uses.

## Frontend integration audit (2026-10-12) , current state
| ID | Finding | Now |
|---|---|---|
| H1 | Overview rendered fixtures; `loadOverviewData` dead | **Fixed** for venue/host; promoter stays fixture |
| H2 | Legacy fixture trees deployed | **Addressed** by 21 × 307 redirects; pages still in repo |
| H3 | Onboarding writes bypassed the BFF | **Fixed** except phone `verify-document` (direct by design) |
| H4 | No fixture disclosure | **Mostly fixed** (`FixtureBadge` on settings, event detail, marketing, events, promoter screens); not every screen verified |
| M1 | Shell chrome fixture-only | Open (inbox branch unmerged) |
| M2 | Guest `/profile` fixture | Resolved |
| M3 | Scanner schemas duplicated | Resolved (adapter over `@c1rcle/contracts`) |
| M4 | Analytics/door "not yet" surfaces | 3 of 5 closed; open on purpose: door composite indexes (`door-ops-service.ts:~546`), door `listSales` in-memory (D4) |
| M5 | Playwright only after merge; staging probe checks HTTP status only | Open |
| — | No true end-to-end test; all Playwright specs mock the backend | Open |

## Frontend defects (verified unless marked reported)
1. Overview event-card links lack the `/partner/<studio>` prefix (`overview-view-model.ts:170`); empty-state CTA hard-coded to venue (`:189`).
2. Events list/detail/edit are fixtures; real events missing; real ids 404; edit has no `onSubmit`.
3. Nav lists "Presence" with no route; Calendar and Door are not in the nav.
4. Block-date times built as UTC from local input (`venue-calendar-repository.ts:89-90`); default pricing-phase windows are UTC midnight (reported).
5. GET of the calendar can create a venue (auto-provision side effect).
6. Event publish is a ~6+N call browser workflow; partial drafts on failure; no cleanup.
7. **Open redirect:** `ChangePasswordClient.tsx:78` uses `router.push(next)` with no `safeNextPath` (`/login` does validate).
8. CSRF gaps: support POST routes have same-origin only and no `stripProtoKeys` (verified); login/signup/forgot/reset have none (anonymous, same-origin check only); slot-request routes take `organizationId` from the query without `assertPathId`.
9. `auth/login` and `auth/signup` handlers do not catch gateway network errors (unhandled 500) (reported).
10. Idempotency inconsistencies: fresh key per BFF call for support and slot-request actions; `staffApi` mints a new key per call.
11. Dead code/routes: `/api/bff/notifications`, `/api/bff/onboarding/verify-document`, `/api/auth/phone-verification`, `lib/bff/auth-otp.ts`, `lib/access/org-access-cache.ts`, `session-context.tsx`, `lib/venue/venue-repository.ts`, ~40k lines of legacy components.
12. Duplicate `DashboardAuthProvider` mounts; N+1 `/access` calls; orgs whose access call fails are silently dropped.
13. No in-app workspace switcher; active org read once from the cookie.
14. Hard-coded/placeholder values: fixture names, "Promoter xxxxxxxx", `MINIMUM_PAYOUT_PAISE`/INR, default `http://localhost:3001` in the poster URL builder (`venue-event-repository.ts:~295`), onboarding plan fixed to `basic`, `reviewNote` hard-coded `''`, staff-fetch error defaults to `canManage:true`.
15. reCAPTCHA disabled for every non-production environment including `preview`.
16. Silent error swallowing: `listPartnerTickets` returns `[]` on any error; autosave failures silent.
17. CSP: `style-src 'unsafe-inline'`; `img-src https:` wide; no `worker-src`/`manifest-src`; API routes get no CSP; HSTS declared twice; `Permissions-Policy` blocks camera.
18. Tooling inconsistencies: `eslint.config.ts` references non-existent Firebase allow-list files; README mentions a non-existent `features/` folder; `react-hook-form`, `NEXT_PUBLIC_SENTRY_DSN`, `NEXT_PUBLIC_GUEST_PORTAL_URL` unused here; Docker build passes no Firebase args.
19. Login with `?next=` bypasses the workspace check; server pages do not enforce studio/partner-type match.
20. `LoadingState` and route `loading.tsx` render nothing; "Authorizing Access" full-screen block on every non-login/onboard path.
21. 33 `eslint-disable` comments in non-test source; no TODO/FIXME/@ts-ignore.

## Backend defects (verified unless marked reported)
1. Promoter orgs get `permissions: []` from `/access`; roles FINANCE_ADMIN/SECURITY/DOOR/COHOST/PROMOTER/TEAM_LEAD unreachable (`partner-access.ts:186-210`).
2. Nothing moves an event to `started/ended/archived` except admin force-complete; `started → cancelled` not allowed.
3. Slot-request accept does not touch the calendar; no `open`/`booked` slots exist, so availability is `fullyBooked` whenever any block exists; create does not validate venue, partnership or event.
4. A host cannot cancel a pending slot request (FSM `pending: ['accepted','rejected']`).
5. Invitation accept lacks an email-match check (`organization-service.ts:334`); `changeRole` lacks a hierarchy check.
6. `promoterAssignmentDtoSchema.status` omits `suspended` (`contracts/event.ts:375`).
7. `GET /venues/:id/profile` lacks rate limit and `requirePermission`; `GET /door/stats(+/stream)` lack `requirePermission`.
8. Idempotency gaps: bank-account writes, dispute writes, invitation revoke, assignment end, door-code create (reported).
9. Calendar window filter uses `startTime` only (overlapping earlier blocks omitted); 2000-slot cap.
10. Discovery capped at first 200 orgs/200 venues.
11. `eventDtoSchema` omits `capacity`/`adminOverride`; `capacity` not settable; event analytics uses venue capacity.
12. Domain-event audit goes to `MemoryAuditRepository` (lost on restart).
13. `organizationDtoSchema` omits settings and `platformFeePercent`; create accepts `settings` that the contract does not list.
14. No `slot_request.*` notifications; partnership/connection answers do not notify requesters.
15. Response cache has no invalidation; stale up to 5 min (plus up to ~10 min for analytics with the projection).
16. Org owner/admin can raise and resolve their own dispute via partner routes.
17. Invitation email contains a plaintext temporary password (forced rotation on first login).
18. Referral attribution HMAC falls back to `'dev-secret'` if `magicTicketSecret` is empty (production is intended to fail to boot without it).

## Endpoints the dashboard needs that do not exist
Ticket tier update/pause/delete; promo code update/deactivate; table package update; event delete/archive/revert to draft; partner refund request route (permission granted, no route); partner reads of reconciliation and audit (permissions exist, no routes); per-role overview DTOs (`HostOverview`, `PromoterOverview`), `PartnerEventSummary`, `PartnerEventDetail`, `PartnerAnalyticsSummary`, `PromoterNetworkProfileData`; ticket transfer/claim; chat/DM and safety reports (BLOCKED, 404 by absence); payout bank-account save is **not missing server-side** but is disabled in the UI.

## Fixture → live migration checklist (ordered by user impact)
1. **Events list and detail** from `GET /organizations/:id/events` + `GET /events/:id` (fixes 404s for real events and the broken overview links).
2. **Event edit** wired to `PATCH /events/:id` with `If-Match`.
3. **Overview links** (add `/partner/<studio>` prefix; studio-aware empty-state CTA).
4. **Notifications and search:** merge `feat/partner-notifications-inbox`, switch the topbar to `/organizations/:id/notifications`, add a `slot_request.*` producer.
5. **Promoter studio** screens (overview, events via `GET /promoters/:id/events`, guests, analytics, leaderboard) and fix promoter `/access`.
6. **Settings, marketing, door mode:** need endpoints or fixtures labelled via `FixtureBadge`.
7. **Atomic publish:** a server-side workflow (single endpoint) or compensation/cleanup.
8. **Calendar semantics:** create `open`/`booked` slots on accept; fix the window overlap filter and UTC handling.
9. Delete the legacy trees and dead BFF routes; add a real-gateway e2e smoke.

## Operational and product caveats that apply here
Backend runs on a Render free instance (sleeps, 30–40 s cold start; per-process cache and rate limiter). Payouts are gated off in production by default. Hard-coded 18% GST, funds-flow legal structure, and refunds never settling affect partner finance (see `../10-payments-and-finance` and `../12-status-and-known-issues`). Doc staleness: the 2026-10-02 banners in the FE handoff docs and the backend roadmap statements "No equivalent contract exists for Venue" and "HostRepository is dead code" are historical.
