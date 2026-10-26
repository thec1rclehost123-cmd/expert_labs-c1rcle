# C1RCLE Platform

Single-repository handoff snapshot of the C1RCLE ticketing and nightlife-operations platform. It combines the former `C1RCLE-FRONTEND` and `C1RCLE-BACKEND` repositories into one tree while keeping each side's pnpm workspace, lockfile and tooling intact.

Prepared for third-party application development on **2026-10-26**.

## Layout

| Path | What it is | Stack | Node / pnpm |
| --- | --- | --- | --- |
| `C1RCLE-FRONTEND/` | Web apps: `admin-console`, `guest-portal`, `partner-dashboard`, `scanner-app`, plus shared packages | Next.js, React Native/Expo, Turborepo | 22.x / 10 |
| `C1RCLE-BACKEND/` | `api-gateway` (Fastify) plus `core` and `contracts` packages | Fastify, Firestore, Redis, Docker, Turborepo | 24 / 11 |
| `architecture-package/` | Verified architecture, API, data-model and known-issue documentation produced for the receiving team | Markdown / Mermaid | — |

`architecture-package/` is confidential and intended to be shared under NDA. It deliberately includes known weaknesses and audit findings.

Each of `C1RCLE-FRONTEND/` and `C1RCLE-BACKEND/` is a self-contained pnpm workspace with its own lockfile, CI configuration and documentation. Do not run `pnpm install` at this root; use the delegating scripts below.

## Prerequisites

- Node.js 22.13+ (frontend is pinned to 22.22.2; backend to 24)
- pnpm 10 for the frontend, pnpm 11 for the backend
- Docker (optional, for the backend container)
- A Firebase project (Firestore) for anything beyond memory-driver tests
- Razorpay, Resend and Redis credentials only when exercising those integrations

Use `nvm use` inside each workspace to switch to its pinned Node version.

## Quick start

```sh
# from this directory
pnpm run install:all

# backend (Fastify on :8080)
cp C1RCLE-BACKEND/apps/api-gateway/.env.example C1RCLE-BACKEND/apps/api-gateway/.env.local
pnpm run dev:backend

# frontend apps (guest :3000, partner :3001, admin :3002)
cp C1RCLE-FRONTEND/apps/guest-portal/.env.example C1RCLE-FRONTEND/apps/guest-portal/.env.local
# repeat for partner-dashboard / admin-console / scanner-app as needed
pnpm run dev:frontend
```

The backend defaults to `STORAGE_DRIVER=memory`, so tests and the API boot without cloud credentials. Set `STORAGE_DRIVER=firestore` and fill the Firebase variables to use a real database.

## Commands

Run from this directory:

```sh
pnpm run install:all   # install both workspaces
pnpm run dev:backend   # start the API gateway
pnpm run dev:frontend  # start the web apps
pnpm run build         # build both workspaces
pnpm run lint          # lint both workspaces
pnpm run typecheck     # typecheck both workspaces
pnpm run test          # run both test suites
pnpm run check         # full format/lint/typecheck/test/build in both
```

You can also run commands inside either workspace directly, for example:

```sh
cd C1RCLE-BACKEND && pnpm test
cd C1RCLE-BACKEND && pnpm check
```

## Contracts

The shared wire contract has two owners:

- Frontend: `@c1rcle/types` and `@c1rcle/api-client`
- Backend: `C1RCLE-BACKEND/packages/contracts`

The backend has a parity check:

```sh
cd C1RCLE-BACKEND
node scripts/contract-parity.mjs --frontend ../C1RCLE-FRONTEND
```

## Environment and secrets

Only `.env.example` files are included. No real `.env`, `.env.local`, service-account key or private key is present in this repository or its history.

Before running against a real backend you must provision your own:

- Firebase / Firestore service account (`FIRESTORE_PROJECT_ID`, `FIREBASE_CLIENT_EMAIL`, `FIREBASE_PRIVATE_KEY`)
- `BETTER_AUTH_SECRET`
- `MAGIC_TICKET_SECRET`
- `ENCRYPTION_KEY`
- `EMAIL_OTP_SECRET`
- Razorpay keys and webhook secret
- Resend API key
- Redis URL (optional for a single instance)

Generate strong random values; do not reuse the original deployment's secrets. The original production credentials are not part of this handoff and should be rotated by the previous owner.

## Documentation

- `architecture-package/README.md` — start here; full system, domain, app-by-app and infrastructure documentation, including known issues.
- `C1RCLE-BACKEND/docs/README.md` — backend documentation map (architecture, API contracts, roadmap, operations).
- `C1RCLE-FRONTEND/docs/architecture/` — frontend architecture decisions and ADRs.
- `C1RCLE-BACKEND/docs/api-contracts/openapi.yaml` — partial OpenAPI description.

## What was intentionally excluded

See `HANDOFF.md` for the full list of removed/sanitized material: production credentials, personal work logs, internal planning documents, local agent/tool state, deployment account identifiers and one-off scripts that pointed at the original live environment.
