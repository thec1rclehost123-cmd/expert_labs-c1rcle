# 01. Product model

## One shell, three studios
Every partner account has a `partnerType`: `venue`, `host` or `promoter`. All three use the same shell (`components/partner-v3/PartnerShell.tsx`), configured per type by `src/studios/studio-config.ts`. Routes are `/partner/[studio]/…` where `studio ∈ {venue, host, promoter}`; any other value is a 404.

- Partner type is **not** the user role. Signup always creates a user with platform role `partner` (set server-side). A venue/host/promoter **organisation** exists only after an admin approves an onboarding application; approval grants exactly the one capability the applicant chose.
- Partner type comes from the server (`GET /api/v2/organizations/:id/access` → `partnerType`), derived from the member's capabilities with precedence venue > host > promoter (default venue if none).
- Staff roles inside an org are `owner | admin | manager | member`; the dashboard labels `member` as "Staff".

## What each type does
| | Venue | Host | Promoter |
|---|---|---|---|
| Owns | A physical venue: profile, menu, calendar, availability | No venue | No venue |
| Core job | Accept/reject host slot requests, block calendar time, create events directly, run the door | Request a slot at a venue ("Start event request"), build events, manage promoters | Connect to hosts/venues, get referral/tracking links, track performance and commission |
| Primary action | Create event | Start event request | Get link |
| Nav (studio-config) | Overview, Events, Slot requests, Partners, Marketing, Presence*, Finance, Support, Settings | same as venue (host hrefs) | Overview, Events, Guests, Analytics, Finance, Partners, Leaderboard |
| Finance | Payouts after fees | Remainder after platform fee, venue share, promoter commission | Commission earnings |

\* "Presence" is in the nav but **no page exists** (404). Calendar and Door are real routes but not in the nav (reached from links). Slot requests appear in the nav for venue and host; promoter has none.

## Money model (settlement split)
`hostPayout = gross − platformFee − venueShare − promoterCommission`.
- Platform fee % comes from the plan chosen at onboarding: basic 15, silver 12, diamond 10 (`platformFeePercent` on the organisation; partners cannot edit it).
- `venueShareRate` is a whole-number percent 0–50 on a partnership, set only by a party to an **active** partnership (`POST /partnerships/:id/venue-share`). An un-negotiated rate settles to 0.
- Promoter commission terms are frozen on the promoter assignment at creation (`terms.version`; default 15% from the route; per-tier rates possible). Commission rounds down; none on free orders.
- Promoter commission tiers (v1 reference): 0→10% Base, 10→12% Silver, 25→15% Gold, 50→18% Platinum, 100→20% Diamond (threshold unit not stated in the doc).

## Privacy rules that must hold
- The promoter network profile is visible to verified venue/host users only and must never show revenue, earnings, commission, payout, bank or settlement fields. There is deliberately no guest-portal promoter profile.
- Venue/host analytics receive **aggregates only**, never raw guest records.
- Promoter "recent orders" expose time, event, ticket count, channel and status only.
- Organisation switching must invalidate role-specific cached data. (Not verified in code.)
- UI for not-yet-live mutations must show "prepared preview" states and never claim success.

## Authority split (the "two vocabularies")
| | Gateway `Permission` | Dashboard `PartnerPermission` |
|---|---|---|
| Keyed by | `OrganizationRole` owner/admin/manager/member | `PartnerRole` OWNER/MANAGER/STAFF/… |
| Examples | `venue.manage`, `event.create`, `finance.read` | `VIEW_FINANCIALS`, `MANAGE_EVENTS` (18 values) |
| Purpose | **What the API enforces** | Dashboard hint (`/access` response); never an authorisation boundary |
Details and defects: `04-auth-rbac-and-access.md`.

## Glossary
- **Studio**: the role-specific dashboard under `/partner/[studio]`.
- **Slot request**: a host's request to book a venue slot for an event; statuses pending, accepted, rejected, cancelled. The host creates it (`slot-request.create`); the venue accepts/rejects (`venue.schedule`).
- **Calendar / calendar block**: a venue's calendar of slots; a *block* marks time unavailable. Availability is derived, never stored.
- **Partnership**: venue↔host relationship (requested → active, or rejected/blocked/ended). Only the counterparty approves/rejects; either side may block or end.
- **Promoter connection**: promoter↔host/venue link, separate from partnerships. Recipient approves/rejects; either party blocks; only the promoter revokes.
- **Promoter assignment**: a promoter attached to one event, with commission terms frozen at creation; ended, never deleted.
- **Referral / tracking link**: per-event link carrying attribution for a promoter; deactivated, never deleted; carries attribution, never money.
- **Venue share**: percentage of gross a venue takes on events a host runs there (0–50).
- **Door code**: `C1R-XXXXXX` code that authenticates a scanner device for an event (`full`, `scan_only`, `charge`); sessions 12 h.
- **Cover wallet**: prepaid credit used at the door (credit, debit, freeze, reconcile, terminate).
- **tabVisibility**: server map of tab keys to booleans; `null` means show all; only an explicit `false` hides a tab.
- **Fixture / `dataStatus`**: sample data bound to the UI; the `FixtureBadge` component discloses it when `dataStatus === 'fixture'`.
- **BFF**: the thin Next.js route-handler layer that owns cookies and CSRF.
- **Org scoping**: the `X-Organization-Id` header must equal the `:organizationId` path segment; the path is authoritative; a mismatch is 403; foreign resources read as 404.
