# Glossary

Terms as used in the code and docs. For acronyms from payments and edge see the area docs.

**Organisation / host / venue / promoter / partner.** An *organisation* is the tenant (`v2_organizations`). Its **capabilities** are `host` (runs events), `venue` (owns a physical venue) and `promoter` (sells through links). *Partner* is the dashboard's word for any of the three; the user's platform role is `guest|partner|admin`. In the admin console "Hosts" lists organisations.
**Studio.** A role-specific section of the partner dashboard (`/partner/venue|host|promoter`).
**Partner role / permission.** Dashboard hints (`OWNER`, `MANAGER`, `STAFF`, … and 18 `PartnerPermission` values) returned by `/access`; not what the API enforces. **Organisation role** (`owner|admin|manager|member`) and **gateway permission** (`venue.manage`, `event.create`, `ticket.override`, `door.manage`, `finance.read`, …) are what the API enforces.
**tabVisibility.** Server map of dashboard tab keys to booleans (`null` = all visible; only `false` hides).
**Platform admin / TIER1, TIER2, TIER3.** A record in `v2_admins` with role `super|admin|ops|finance|support`. Tier 1 any admin; tier 2 `super|admin|ops|finance`; tier 3 `super` only with **dual control**.
**Dual control / proposal.** Two-person rule: one `super` proposes, a different `super` approves, an execute route applies the **stored** payload. Status `pending|approved|rejected|cancelled`.
**Slot request.** A host's request to use a venue date for an event (pending, accepted, rejected, cancelled).
**Partnership / promoter connection / promoter assignment / referral link.** Venue↔host relationship (with a `venueShareRate`); promoter↔host-or-venue relationship; a promoter attached to one event with frozen commission terms; a per-event link carrying attribution.
**Venue share.** Percentage (0–50) of net ticket revenue a venue takes on a host's event. **Platform fee** exists in two senses: the **guest fee** (5% + 2.5% + GST on fees, charged on top) and the **partner commission** (15/12/10% of net revenue by plan). See `10-payments-and-finance/01-…`.
**Net ticket revenue.** The discounted subtotal (guest fees and GST excluded); the base of the settlement split.
**Paise.** Integer hundredths of a rupee; all money on the wire.
**Hold / reservation.** A 10-minute reservation of seats during checkout, ordered by a per-event sequence number. **Entitlement.** One ticket unit (a couple ticket is one entitlement admitting 2). **Order.** The purchase record (`paid` directly online).
**Hold sequence.** The per-event counter (`v2_hold_sequences`) that orders concurrent buyers for capacity decisions.
**Late capture.** A payment confirmed after its hold expired: revived or auto-refunded.
**Idempotency key.** Header (or body field) that makes a mutation safe to retry; scoped per actor and command; 24 h; a failed attempt is retained and returns 409.
**Optimistic locking / `If-Match`.** `version` on aggregates; five partner routes require `If-Match`; writes use compare-and-set.
**Outbox / event bus.** Domain events appended to `v2_outbox` and delivered in-process (at-least-once; five attempts then dead-letter) to audit, analytics, notifications and follower fan-out.
**BFF.** The Next.js server layer that owns cookies and CSRF and calls the gateway.
**Gateway / sidecar.** The Fastify API (`/api/v2`); the Render container that runs nginx and Fastify together.
**Origin lock / `X-Origin-Verify`.** Refusing requests that bypass Cloudflare; the secret header Cloudflare stamps.
**Release gate (`PAYOUTS_ENABLED`).** Switch that blocks partner payout requests, admin batch runs and release in production until legal, tax and provider approvals exist.
**Journal v2 / legacy ledger / journal transition.** Double-entry finance model; the older partner ledger (default read model); the controlled switch between them (no operator interface yet).
**Provider payment / provider fact.** Razorpay's side of a payment (captured amount, fee, refunds, disputes) stored by Razorpay id; facts, not balances.
**Reconciliation run / discrepancy.** A manual comparison of orders, provider facts, tickets and payouts that raises items of 14 kinds.
**Door code.** `C1R-XXXXXXXX`, minted per event (type `full|scan_only|charge`), redeemed for a shift. **Scanner session.** 12-hour token (`scn_…`) from a door code. **Device.** A registered handset (`scanner_` + 32 hex), bindable and unbindable by a manager. **Shift.** One scanner session.
**Rotating QR / magic ticket.** `entitlementId:windowStart:HMAC` with a 30 s window (about 90–120 s effective validity). **Wallet QR.** `cw:<walletId>:<window>:<hmac>`.
**`claimAdmission`.** The single Firestore transaction that spends a seat. **Scan ledger.** Append-only record of every scan attempt. **Override / staff-deny / manual check-in.** Admit a denied scan; refuse without spending the ticket; admit from the roster.
**Cover wallet / preset item.** Prepaid per-event bar tab; a venue-defined item with a fixed price (the client never names an amount).
**Offline manifest.** Signed snapshot of per-ticket keys and seats for offline door scanning (backend design on staging, not used by the app).
**FSM.** Finite state machine (explicit transition tables in `domain/fsm.ts`).
**Fixture / `dataStatus`.** Sample data bound to a UI; the partner dashboard labels it with `FixtureBadge`.
**IST.** Asia/Kolkata (UTC+05:30), used for "today" in door events, wallet termination (05:00 next day) and default organisation timezone.
**SOTA-n / D-0nn.** Numbered invariants in `docs/nginx/sota-architecture.md`, `docs/scanner-app/sota-architecture.md`; numbered decisions in `docs/architecture/decisions.md` (see `ADR-INDEX.md`). Note "A4" means two different tasks in `cloudflare_plan.md` (offline scanning) and the root `task.md` (door permission guard).
