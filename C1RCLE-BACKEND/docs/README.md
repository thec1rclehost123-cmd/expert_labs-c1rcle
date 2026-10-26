# C1RCLE-BACKEND documentation map

Several kinds of document live here, each with a different job. Don't mix them —
that's what caused this folder to need reorganizing once already.

```
docs/
├── README.md            you are here — index only, no content of its own
├── roadmap/              WHAT'S IMPLEMENTED VS REMAINING (start here)
├── architecture/         HOW the system is built, and WHY (design + decisions)
├── reference/            Point-in-time material copied in from elsewhere —
│                         not maintained, read for context, not truth
├── api-contracts/        wire contracts the frontends code against (auth, errors, scanner app)
├── admin-dashboard/      per-desk admin console docs (tiers, proposals, support desk)
├── scanner-app/          door scanner backend/app docs (SOTA blueprint, topic docs, task/impl logs)
├── integration-flows/    checkout + ticket-booking frontend↔gateway flows
├── nginx/ operations/    edge/deploy docs (nginx, Render staging, env contract, rollback)
└── ci-cd.md              CI/CD + deploy pipeline (adapt to your own infrastructure)
```

## `docs/roadmap/` — implemented vs remaining (the status tracker)

**Start here.** [`ROADMAP.md`](roadmap/ROADMAP.md) is the master index: one row
per phase, current status, link to detail. Each phase file names its exact
endpoints, the v1 business logic it ports, and a dated Session Log so a new
session can resume without re-deriving context. This is the *only* place
status is tracked — don't duplicate a "what's done" list anywhere else.

- Phase 0 (Foundation: auth, persistence, the frozen partner slice) — **done**.
- Phase 1 (Partner dashboards) — **done**; finance/analytics shipped with Phase 6, `venueShareRate` shipped 2026-09-17.
- Phase 2 (KYC/Onboarding) — **done**; pre-signed storage upload URLs shipped 2026-08-30.
- Phase 3 (Event-catalog & scheduling) — **done**.
- Phase 4 (Guest checkout & tickets) — **done** (verified 2026-09-07).
- Phase 5 (Door/Scanner/Cover-wallet) — **done** (verified 2026-09-28); both former 501s are now implemented.
- Phase 6 (Finance/Ledger/Payouts) — **done** (verified 2026-09-08); `venueShareRate` closed 2026-09-17, analytics fallback 2026-09-19.
- Phase 7 (Admin console backend) — **done** (A/B/D 2026-09-16; support desk + guest intake 2026-09-20); safety/content moderation and invite-by-email deferred, see `roadmap/ADMIN-DASHBOARD-GAPS.md`.
- Phase 8 (Social/notifications) — **partial**: follow graph + guest notifications + partner notification inbox live; chat/DM not started.

## `docs/architecture/` — how the system is built, and why

Living design documentation. Update it when the system's shape changes, not
just when a task finishes.

- [`README.md`](architecture/README.md) — the one request end-to-end, every
  file's purpose, the security rule list, a built/pending status ledger.
- [`decisions.md`](architecture/decisions.md) — the decision log (D-001…).
  Append-only: every architectural choice that must survive a session, with
  its problem/options/choice/why. Never rewrite past entries.

## `docs/reference/` — point-in-time material, not maintained

Everything here was copied from elsewhere (the frozen `thec1rcle` repo, or a
frontend-audit snapshot) at a specific date. Treat it as **a snapshot to read
for context, not as current truth** — if it disagrees with live code or with
`docs/architecture/`, live code and `docs/architecture/` win.

- `frontend-api-map.md` — `C1RCLE-FRONTEND` route/contract mapping as of
  2026-08-12. Useful for "what does the frontend expect," not for "what's
  live in this backend today" (that's `docs/roadmap/`).
- `route-manifest.ts` — the original full-platform planning manifest. Live
  registration is `apps/api-gateway/src/routes/v2/route-manifest.ts`.
- `V1_TO_V2_PARITY.md` — V1 → V2 behavioural parity notes.

## Rule of thumb for adding new docs

Ask which of the core folders (roadmap, architecture, reference) it belongs to before creating a file:
- Tracking what's built vs not, phase by phase → `docs/roadmap/`
- Explaining a lasting design choice or how a layer works → `docs/architecture/`
- Copying in outside material for one-time reference → `docs/reference/`

If it doesn't fit any of those, it probably shouldn't be a top-level `docs/`
file — consider whether it belongs inside an existing phase file instead.
