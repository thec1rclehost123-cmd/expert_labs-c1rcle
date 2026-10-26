# 08. Guest portal: in-depth documentation

**Frontend:** `C1RCLE-FRONTEND/apps/guest-portal` (`@c1rcle/app-guest-portal`, port 3000, Vercel project built from production branch `main`, canonical host `thec1rcle.com`)
**Backend pieces it depends on:** `apps/api-gateway/src/routes/v2/{public,checkout,rsvp,orders,tickets,wallet,profile,social,notifications,support,auth}` and the `door/scanner-routes.ts` ticket-QR route.
**Snapshot:** 2026-10-24. Frontend checkout `fix/partner-connections-contracts` (1 behind `origin/staging`; for `apps/guest-portal` there is **no diff** against staging). Backend checkout `fix/partner-connections-and-events`. Contracts are byte-identical between the repos apart from the generated header.

The guest portal is the consumer site: discover events, buy tickets (free RSVP or paid via Razorpay), see tickets with a rotating entry QR, get help. It is a Next.js app that calls the gateway through its own BFF.

## Reading order
**Prefer pictures? Start with `00-visual-guide.md` (16 diagrams). Open `visual-guide.html` for a rendered version (needs internet for the diagram library).**
| File | What you get |
|---|---|
| `00-visual-guide.md` / `visual-guide.html` | Page map, fixture vs live, sign-in, BFF path, checkout, recovery, tickets, referral link flow, defect map |
| `01-routes-and-screens.md` | Every page, data source, gating, fixture vs live matrix |
| `02-architecture-auth-and-bff.md` | Stack, config, proxy, headers, sessions, every BFF route and its checks |
| `03-discovery-seo-and-promoter-links.md` | Explore, event pages, SEO, sitemap, vanity links and why attribution is broken |
| `04-booking-and-checkout.md` | RSVP, paid flow, hold, Razorpay, verify, recovery, confirmation page |
| `05-tickets-wallet-profile-support.md` | Tickets and QR, cover wallet, profile, follows, notifications, help, legal |
| `06-backend-public-and-guest-api.md` | Public endpoints and the guest-side routes with limits |
| `07-status-gaps-and-roadmap.md` | Defects (frontend and backend), security findings, dead code, fix order |
| `08-testing-build-deploy.md` | Tests, CI, accessibility, performance, build and env names |

## Executive summary
- **Live:** event discovery and detail (`/explore`, `/event/*`), RSVP and **paid checkout with real Razorpay**, payment recovery, confirmation, ticket wallet with a rotating entry QR, password login/reset, help intake (in preview). Frontend PR **#72 is closed but its work (and more, including recovery) is on staging**: `a9e3061`, `034bdd8`, `dd71a17`. (Earlier pages of this package called the paid checkout UI "Unmerged": wrong.)
- **Fixtures in production:** the home hero and the app/marketing pages; `/checkout/<fixture slug>` and `/confirmation/preview-*` are reachable in production. `/hosts`, `/profile/[userId]` are fixture or empty in production; `/profile` and `/help` are **deliberately blocked in production** ("unavailable") and live only in preview and development.
- **No guest sign-up on the web:** `/signup` redirects to login, which tells people to use a mobile app. The gateway reports **every user as role `partner`** (signup hard-codes it and `toUserDto` always returns it), so guest vs partner is not enforced anywhere in the portal.
- **Features the backend supports that the portal does not use:** promo codes, referral/promoter attribution, attendee capture to the order, the 10-minute hold countdown, multiple-ticket QR, wallet QR for the cover tab, follows, guest notifications, profile save, order history list, refund status, venue/host list pages.
- **Broken promoter attribution:** `PromoterClickTracker` is imported nowhere, `/event/*` ignores `?ref=`, the checkout client never sends `referralCode` or `promoCode`: **no click is tracked and no order is attributed** from the portal.
- **Security findings (verified):** the help-ticket BFF routes splice an unvalidated `ticketId` into the gateway path; the support DTO returns `internalNotes` (and assignee, timeline, SLA) **to the requester**; the BFF forwards no client IP so anonymous auth calls share one rate-limit bucket per egress IP; no CSP, no HSTS, Razorpay script without SRI; malformed `%` in dynamic URLs raises a 500.
- **SEO bugs:** `/event/*` always emits `noindex`, so the sitemap lists pages search engines are told to skip, and the Event JSON-LD component is never rendered; venues and hosts are always absent from the sitemap.

## Conventions
Status labels: **Live**, **Partial**, **Unmerged**, **Planned**; "verified" = checked in source or with `gh`/`git` on 2026-10-24; "reported" = from a code read. Names only for environment variables.
