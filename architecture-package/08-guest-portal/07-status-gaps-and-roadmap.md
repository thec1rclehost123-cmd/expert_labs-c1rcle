# 07. Status, gaps and roadmap (candid)

Snapshot 2026-10-24. **Keep under NDA.** "Verified" = checked in source or with `gh` on 2026-10-24; "reported" = from a code read.

## State and branches
Frontend: nothing differs between the checked-out branch and `origin/staging` for `apps/guest-portal`. PR #72 (guest paid checkout) is **closed**; its work and more (`a9e3061` real Razorpay, `034bdd8` tell a refunded buyer, `dd71a17` recover in-flight checkout) are on staging, so #72 adds nothing beyond staging. Open frontend items: PR #70 (checkout attendee contract mirror) is open; `origin/feat/app-shell-csp` (CSP) is unmerged. Backend payments are on staging (`../10-payments-and-finance/09-…`).

## Security findings (frontend)
| # | Sev | Finding | Evidence |
|---|---|---|---|
| G-S1 | High | **Help-ticket BFF path injection:** `/api/help/tickets/[ticketId]` (GET) and `…/messages` (POST) interpolate the decoded `ticketId` into the gateway path with **no regex check** (unlike the orders, qr and verify routes), so a crafted segment (`..%2F` style) can steer an authenticated, cookie-bearing request to other gateway paths as the user. The GET also lacks `assertSameOrigin` | `help/tickets/[ticketId]/route.ts` L12–13 (verified) |
| G-S2 | Med | The BFF does **not forward the client IP**: all anonymous auth calls (login, signup, forgot, reset) share one rate-limit bucket per egress IP at 10/min and server-side public reads share 120/min; a few users or an attacker can lock out logins for everyone | `forwardToGateway` sends only accept, content-type, cookie, idempotency (reported) |
| G-S3 | Med | No CSP, no HSTS; the Razorpay script loads without SRI | `next.config.ts` (verified absence) |
| G-S4 | Med | Help POST routes have no CSRF header check (origin check only, and a **missing Origin passes**); help GET forwards the raw query string; the recovery DELETE has no CSRF (low impact) | reported |
| G-S5 | Low–Med | `decodeURIComponent` without try/catch in the event, host, venue, checkout and confirmation pages: a malformed `%` raises a URIError → 500 instead of 404 | reported |
| G-S6 | Low | Proxy gating trusts cookie presence only (private pages fail later at the server or API) | proxy |
No open redirect, no unescaped HTML (the only `dangerouslySetInnerHTML` is the escaped JSON-LD), and no IDOR on orders, tickets, wallet, QR or support (all 404 for strangers).

## Security findings (backend, guest surface)
| # | Sev | Finding |
|---|---|---|
| G-B1 | High | **Support DTO leaks `internalNotes`, `assignee`, `timeline`, `sla` and merge/delete audit fields to the requester** (`intake-routes.ts` ≈L45, 70; service returns the raw aggregate; contract schema includes them) |
| G-B2 | Med | `toUserDto` returns `role:'partner'` for every user and signup hard-codes it; **guest vs partner is not enforced** anywhere in the portal |
| G-B3 | Low | Referral attribution verification falls back to `'dev-secret'` if `magicTicketSecret` is empty |
| G-B4 | Low | Promoter click inflation: anonymous, no dedupe |

## Functional defects
1. **Promoter attribution broken** end to end (`03-…`). 2. **No promo-code or attendee capture**: every order carries `Guest / guest@example.com`; RSVP orders `RSVP Guest`. 3. **Confirmation page shows `Guest` and `RSVP-…` even for paid orders** and only the first line. 4. **SEO:** `/event/*` always `noindex`; Event JSON-LD never ships; venues and hosts never in the sitemap; canonical hard-coded in event and explore metadata. 5. **`/profile` and `/help` are blocked in production**, `/hosts` empty, `/signup` absent: the production surface is mostly discovery plus checkout plus tickets. 6. **Profile editing does nothing** (no save). 7. **Multi-ticket orders** expose only the first ticket's QR. 8. **Wallet:** only 50 entitlements, N+1 browser round trips, no pagination. 9. **Fixture routes reachable in production:** `/checkout/<fixture slug>`, `/confirmation/preview-*`; the preview path on a live event links to a 404. 10. **Tier selector** links a derived tier id that does not exist; label says "checkout preview" for live events. 11. **Hold countdown** missing; `PREVIEW_MAX_QUANTITY=10`; fallbacks `Venue TBA`, `India`, `Events`, `Host TBA`; accent tone derived from `title.length`. 12. **No guest cover-wallet UI**, no follows UI, no notifications inbox, no order history, no refund status, no change-password. 13. **Fonts never load** (Satoshi/Inter referenced, no loader). 14. **Cold start and caching:** the root layout reads cookies so every page is dynamic and `revalidate=300` is moot; the gateway sends no Cache-Control on public GETs; explore fan-out is 1 + N; event pages make 3 reads although the event embeds venue and organiser; large `.mp4/.m4v` assets in `public/`.

## Dead and unreachable code
`PromoterClickTracker`; `AuthoritativeEventView` (the production page uses `EventDetailView`) and `getPublicEventForSeo`; `SessionBootstrap`; `lib/app-meta.ts` `NAV_ITEMS` (`/bookings`, `/account` do not exist); OTP BFF routes; the BFF signup route (UI absent); `TicketsGuestView` in practice; the `PendingReservationAlert` type; `features/**/fixtures` outside non-production; duplicate `csrfHeader()` helper (in `checkout-api.ts` and `CheckoutFlowClient.tsx`); `FIXTURE_ONLY` comments on layout components that are the live shell; stale `coverage/` folders in the working tree; stale `docs/frontend-backend-handoff.md`; unused dependencies (react-query, react-hook-form).

## Gaps versus backend capability
Promo-code and referral input, attendee capture to the order, hold countdown, multi-ticket QR, wallet pagination, wallet QR UI, follows, notification inbox, profile save, change password, order history list, refund status display, venue and host list endpoints and pages, use of the search endpoint, real privacy and terms content, a guest role and web sign-up, and production enablement of `/profile` and `/help`.

## Suggested order
1. Fix G-B1 (redact internal fields for requesters) and G-S1 (validate path ids; add same-origin and CSRF to every BFF route). 2. Forward the client IP from the BFF (and the origin-proof header per the nginx plan). 3. Send attendee, promo code and referral code; wire `PromoterClickTracker` and read `?ref`. 4. Real guest role and sign-up, or document partner-only auth. 5. Fix event SEO (index, JSON-LD, canonical) and the sitemap. 6. Enable `/profile` and `/help` in production after the above; add profile save. 7. Per-ticket QR pager, wallet pagination, order history. 8. CSP and HSTS, SRI or self-hosting for the Razorpay script. 9. Remove fixture routes from production, delete dead code, refresh the handoff doc. 10. Cover-wallet, follows and notification UIs when product wants them.
