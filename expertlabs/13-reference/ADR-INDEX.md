# Decision records index

`C1RCLE-BACKEND/docs/architecture/decisions.md` (828 lines, "append-only"). **Numbering problems:** **D-030 appears twice** (lines 497, before D-025, and 815, after D-029; verified; the copies differ only in point 4, `maxPerUser`, and the first notes it ships with `feat/partner-create-event-flow-v2` only), so the file is out of order. Reversals: D-009 reverses D-004's envelope description; D-008 supersedes D-006's "never register BLOCKED slices"; D-011 supersedes the parallel SQLite implementation; D-024 supersedes the frozen `session.*` manifest; D-028 supersedes the roadmap's WebSocket note.
| ID | Decision (one line) |
|---|---|
| D-001 | Auth is the Better Auth library with cookie sessions, not hand-rolled JWT or Firebase |
| D-002 | Repository-first storage: in-memory, then Firestore, with Postgres as the destination |
| D-003 | Contracts are backend-owned; frontend copies are parity-checked |
| D-004 | Error envelope single-sourced in `packages/contracts` (maps `DomainError` to HTTP) |
| D-005 | Route is thin, service decides, model holds rules; enforced by `check-boundaries` |
| D-006 | BLOCKED slices are absent (404), never stubbed (501); partly superseded by D-008 |
| D-007 | Page-based pagination on the wire (`PageInfo`); cursors stay internal |
| D-008 | Scope expanded to a phased full-platform roadmap |
| D-009 | Error envelope is flat everywhere (bug fix) |
| D-010 | `publish()` walks the FSM instead of widening the transition table |
| D-011 | Reconciled with a parallel session's independent B08–B12 work (SQLite vs Firestore) |
| D-012 | Policy order: rate-limit, validate, authorize, cache |
| D-013 | Invitations are a first-class aggregate addressed by email |
| D-014 | Availability is derived, never stored |
| D-015 | Compare-and-set closes the lost-update race (completes D-002) |
| D-016 | Durable idempotency (Firestore store); the menu as public-profile data |
| D-017 | Platform authority is separate from organisation authority (`PlatformAdmin`) |
| D-018 | The original KYC "verification" was a format check, now labelled as one; pluggable provider seam |
| D-019 | Platform fee lives on the organisation, outside `OrganizationProps` (plan→percent: basic 15, silver 12, diamond 10; set at approval) |
| D-020 | Phase 4 execution: guest checkout and tickets (ports, memory and Firestore adapters, services) |
| D-021 | Transactional outbox completion for Phase 4 fulfilment |
| D-022 | Razorpay webhook security: HMAC verification is not optional |
| D-023 | Frontend contract enforcement: no stubs in production |
| D-024 | Frontend→gateway auth: live `/api/v2/auth/*` supersedes the frozen manifest; bare-DTO success envelope; CSRF is a Next BFF concern |
| D-025 | The door is authenticated twice and admission is claimed, not checked |
| D-026 | Scanner backend: a shift is a session plus a bound handset |
| D-027 | Money at the door: a tab names an item, a sale names a tier |
| D-028 | Live door stats use Server-Sent Events, not WebSocket |
| D-029 | Two holes found reviewing the scanner surface (unbind could be undone; roster over-fetch), with `scanner-threat-model.md` |
| D-030 (×2) | Partner-chain audit closes: N+1 name resolution, block TOCTOU, poster ACL, the `maxPerUser` knob |
Frontend ADRs (`C1RCLE-FRONTEND/docs/architecture/adr/`): `0001` TypeScript 5, not 7; `0002` ESLint 9 with `import-x`; `0003` compiled packages.
**Decisions that are not recorded as ADRs** (the lived decision is in another document or only in practice): the nginx sidecar topology and the Cloudflare origin-lock model (`docs/nginx/sidecar-deployment.md`, `cloudflare-edge.md`); the offline-scanning model (SOTA forbids an offline queue without a new ADR; none exists); the payments design (branch `docs/payments-design`, PR #116) and refund and chargeback decisions (PR #131, open); journal v2 and the read-model transition; the payout release gate (`payments-release-gate.md`); the keep-Vercel-for-compute choice; free-plan-only infra (owner decision in `cloudflare_plan.md`). Suggest: fix the D-030 duplicate, then add ADRs for these.
