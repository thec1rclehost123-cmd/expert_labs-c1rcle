# 04. Auth, RBAC and access

## Login and workspace selection
1. `/login` (pick studio, then credentials) → BFF `POST /api/auth/login` → gateway Better Auth; the BFF re-scopes the session cookie and mints the CSRF cookie. `login()` clears any previous active-org hint.
2. The client loads `GET /api/bff/organizations` and, per org, `GET /api/bff/organizations/:id/access`. Zero orgs → pending invitation (`GET /auth/invitations/mine`) or `/onboard?type=`. One org → set active and land. Several → `/partner/select-organization`.
3. `resolveLandingPath` lands on the first tab not withheld by `tabVisibility`.
4. `mustChangePassword` (staff created with a temporary password) sends the user to `/change-password` via `PasswordChangeGuard`; the gateway also returns 403 `passwordChangeRequired` on every non-allowlisted path until rotated.
5. Selecting a workspace that does not match the studio chosen on `/login` shows an error and leaves the user signed in. When `?next=` is present, login redirects there **without** verifying the workspace match.

## Gateway authorisation (what is actually enforced)
- Actor resolution (`apps/api-gateway/src/plugins/auth.ts`): Better Auth session, then `X-Organization-Id` (regex-checked) matched to a membership → `request.actor = {userId, organizationId, role, capabilities}`. No membership → session-only actor with `organizationId = ''`.
- `requirePermission(p)` (`plugins/rbac.ts`): 401 without actor, 403 without the permission, and **ABAC**: a `:organizationId` path param must equal `actor.organizationId` (path is authoritative). Routes addressed by `/events/:id` or `/venues/:id` rely on services (`requireOrgAccess`, `fetchOwned`) and return 404 for foreign resources.
- Authorisation is **role-only**: capability (host/venue/promoter) is not checked by any route (a host-only org can still `POST /organizations/:id/venues`).

### Organisation role → permission (backend `ROLE_PERMISSIONS`)
| Permission | owner | admin | manager | member |
|---|---|---|---|---|
| organization.read | ✓ | ✓ | ✓ | ✓ |
| organization.update | ✓ | ✓ | – | – |
| organization.create | ✓ | – | – | – |
| staff.manage | ✓ | ✓ | – | – |
| venue.read | ✓ | ✓ | ✓ | ✓ |
| venue.create | ✓ | ✓ | – | – |
| venue.manage / venue.schedule | ✓ | ✓ | ✓ | – |
| event.read | ✓ | ✓ | ✓ | ✓ |
| event.create / update / publish | ✓ | ✓ | ✓ | – |
| event.cancel | ✓ | ✓ | – | – |
| slot-request.create | ✓ | ✓ | ✓ | – |
| slot-request.list | ✓ | ✓ | ✓ | ✓ |
| ticket.override, door.manage | ✓ | ✓ | ✓ | – |

Finance: owner/admin hold `finance.read`, `.orders.read`, `.refunds.request`, `.payouts.read`, `.payouts.manage`, `.bank_accounts.manage`, `.reconciliation.read`, `.audit.read`. Manager holds read, orders.read, refunds.request, payouts.read. Member none. Never granted to any org role (platform-admin only): refunds approve/execute, reconciliation resolve, adjustments create. Granted but unused by any partner route: `finance.refunds.request`, `finance.reconciliation.read`, `finance.audit.read`.
Notable: `member` can read partnerships, connections, analytics, notifications and **answer** partnership/connection requests (those use `organization.read`); disputes use `organization.update`, so an owner/admin can both raise and resolve their own dispute through partner routes (platform has separate admin dispute routes).

## `GET /organizations/:id/access` (the dashboard hint)
`organizations.ts` (~l.482): `requirePermission('organization.read')`, `organizations.get` (forbidden hidden as 404), find the member, then
- `partnerType = primaryPartnerType(capabilities)` (venue > host > promoter; empty → venue),
- `role = tenancyRoleToPartnerRole(membership.role)`: owner/admin → `OWNER`, manager → `MANAGER`, member/none → `STAFF`,
- `permissions = permissionsFor(type, role)`, `tabVisibility = tabVisibilityFor(type, role)`.
Response `partnerAccessDtoSchema {organizationId, userId, partnerType, role: string, permissions[], tabVisibility: Record<string,boolean> | null}`; `null` = unrestricted. Not cached.

`PartnerPermission` (18): VIEW_FINANCIALS, MANAGE_STAFF, MANAGE_EVENTS, EDIT_EVENT_RULES, MANAGE_TABLES, VIEW_GUESTLIST, SCAN_ENTRY, LOG_INCIDENTS, VIEW_ANALYTICS, MANAGE_SETTINGS, MANAGE_PROMOTERS, MANAGE_PAYOUTS, MANAGE_PARTNERSHIPS, MANAGE_PAGE_CONTENT, VIEW_REAL_TIME_SCANS, MANAGE_GUEST_OPS, CHARGE_COVER_WALLETS, EXPORT_GUESTS. Source: `packages/core/src/domain/models/partner-access.ts`.

### Permission matrices
**Venue:** OWNER = 16 permissions (all except MANAGE_PROMOTERS, VIEW_REAL_TIME_SCANS). MANAGER = VIEW_FINANCIALS, MANAGE_EVENTS, EDIT_EVENT_RULES, MANAGE_TABLES, VIEW_GUESTLIST, SCAN_ENTRY, LOG_INCIDENTS, VIEW_ANALYTICS, MANAGE_PAGE_CONTENT, MANAGE_GUEST_OPS, CHARGE_COVER_WALLETS. FINANCE_ADMIN = VIEW_FINANCIALS, MANAGE_PAYOUTS, VIEW_ANALYTICS. STAFF = VIEW_GUESTLIST, MANAGE_TABLES, LOG_INCIDENTS. SECURITY = VIEW_GUESTLIST, SCAN_ENTRY, LOG_INCIDENTS. DOOR = SECURITY + CHARGE_COVER_WALLETS.
**Host:** OWNER = VIEW_FINANCIALS, MANAGE_STAFF, MANAGE_EVENTS, MANAGE_PROMOTERS, MANAGE_PAGE_CONTENT, VIEW_ANALYTICS, MANAGE_PAYOUTS, MANAGE_PARTNERSHIPS, VIEW_REAL_TIME_SCANS (no door/scan). COHOST = MANAGE_EVENTS, MANAGE_PROMOTERS, VIEW_ANALYTICS, VIEW_REAL_TIME_SCANS, VIEW_GUESTLIST. MANAGER = MANAGE_EVENTS, VIEW_ANALYTICS, VIEW_GUESTLIST, VIEW_REAL_TIME_SCANS. STAFF = VIEW_GUESTLIST, VIEW_REAL_TIME_SCANS.
**Promoter:** PROMOTER = VIEW_ANALYTICS, MANAGE_PAGE_CONTENT, VIEW_GUESTLIST, VIEW_FINANCIALS, MANAGE_PAYOUTS. TEAM_LEAD = VIEW_ANALYTICS, MANAGE_STAFF, VIEW_GUESTLIST, VIEW_FINANCIALS.

### `tabVisibility`
`null` (all tabs) for venue/host OWNER and promoter TEAM_LEAD. Venue MANAGER: everything except `settings`. Venue STAFF: `walk_ins, guest_ops, registers, door`. Venue SECURITY: `guest_ops, walk_ins, door`. Venue FINANCE_ADMIN: `finance, analytics, overview`. Other venue roles: `door` only. Host COHOST: all except `finance` and `settings`; host MANAGER also loses `network`; host STAFF: `events, calendar, audience, team`. Promoter (non-TEAM_LEAD): `overview, links, events, calendar, partners, analytics, finance, guests`, `settings` hidden. Backend tab keys differ from frontend labels (`walk_ins, guest_ops, registers, door, staff, partnerships, page_management, crm, presence, network`, …).

## Defects in the access derivation (verified in source 2026-10-24)
1. **Promoter orgs get empty permissions.** `tenancyRoleToPartnerRole` can only emit OWNER, MANAGER, STAFF, and `PROMOTER_PERMISSIONS` has only PROMOTER and TEAM_LEAD. A promoter owner/admin therefore maps to `OWNER`, `permissionsFor('promoter','OWNER')` returns `[]`, and tabs fall to the non-TEAM_LEAD branch (settings hidden). Tests cover PROMOTER/TEAM_LEAD directly but never via the `/access` path.
2. **Unreachable roles:** FINANCE_ADMIN, SECURITY, DOOR, COHOST, PROMOTER, TEAM_LEAD cannot be produced by `/access`. Staff have no per-person partner role; capabilities are the only per-person axis, so a door-only user cannot be expressed (a `member` lands on venue STAFF tabs).
3. A member with empty `capabilities` (the default when an invite omits them) is treated as **venue** even inside a host org.
4. **Invitation accept has no email check.** `OrganizationService.acceptInvitation` (`organization-service.ts:334`) loads the invitation by id and calls the domain `acceptInvitation(org, invitation, command.userId, now)`; the session user's email is never compared with `invitation.email`. Any signed-in user holding the `invitationId` can accept.
5. `changeRole` has no hierarchy check (an admin can demote or remove other admins; only the owner is protected). `inviteMember` (direct add by `userId`) bypasses invitations and credential provisioning.
6. `GET /venues/:venueId/profile` has no rate limit and no `requirePermission`; `GET /door/stats` and `/door/stats/stream` have no `requirePermission` (tenancy is enforced in the service).
7. `docs/roadmap/phase-01-partner-dashboards.md` says admin maps to MANAGER; the code (and `partner-access.test.ts`) map admin to OWNER.

## Frontend gating
- **V3** uses only `tabVisibility`: nav filtering plus the client-only frame guard; `isOrgAdmin = role in {admin, owner}` in `StudioPartnersClient`; and `staffAccess.canManage`, derived from whether `GET …/invitations` succeeded (default `true` when absent, and `true` on staff-fetch exception: a UI hint only, since the gateway enforces `staff.manage`). **V3 never calls `canDo`, `hasPermission` or `grantedPermissions`.**
- **`canDo(action)`** (`DashboardAuthProvider.tsx` ~L246, fail-closed; `'*'` grants all) with `ACTION_PERMISSIONS`: canEditEvent/canPublishEvent→MANAGE_EVENTS, canSendMarketing→MANAGE_PAGE_CONTENT, canManageDoorMode→SCAN_ENTRY, canSendTicket→MANAGE_GUEST_OPS, canApprovePromoter→MANAGE_PARTNERSHIPS, canManageFinance→MANAGE_PAYOUTS, canManageSettings→MANAGE_SETTINGS, canManageStaff and canManageSecurity→MANAGE_STAFF. Call sites exist only in unreachable legacy components (host event detail, venue settings/event-detail/finance/slot-requests/create-event/marketing/guests/finance/door screens).
- `useOrgAccess` (`lib/access/use-org-access.ts`): waits for `hydrated`; 403 → `isSuspended`; other error → `error`; stale results from a previous org are treated as loading. `lib/access/org-access-cache.ts` (shared in-flight cache) is unused.
- Consequence: the **payout-request button is not gated by `MANAGE_PAYOUTS`** in V3; it is enabled whenever a bank account exists and balance ≥ ₹100. The gateway (`finance.payouts.manage`, owner/admin only) is the real control.
- Server pages do not verify studio/partner-type match; only the client frame does.
- Types (`lib/rbac/types.ts`): `PartnerType = venue|host|promoter|club`, `StaffRole = owner|admin|manager|staff|promoter`. No `isBanned` concept; `mustChangePassword` is on the session user.

## Platform admin authority (separate; see `../04-auth-and-rbac`)
Onboarding approval and document review are platform-admin actions (`v2_admins`, TIER2: support cannot approve), not partner permissions.
