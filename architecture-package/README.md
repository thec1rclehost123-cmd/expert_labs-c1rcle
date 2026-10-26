# C1RCLE: Architecture Package for Expert Labs

Prepared 2026-10-23, deepened and re-verified 2026-10-24 from a read-through of both repositories (`C1RCLE-BACKEND`, `C1RCLE-FRONTEND`) and their existing docs.
**Confidential. Share only under an NDA.** This package includes known weaknesses and audit findings on purpose (see `12-status-and-known-issues/`).

> **Snapshot note (2026-10-26):** this package describes the original repositories as they stood on 2026-10-23/24. The code snapshot delivered alongside it has since been sanitized: real `.env` files, `.mcp.json`, agent state and the one-off scripts that embedded credentials or real ids have been removed. Where this package says those files are present or hard-code values, read it as a finding about the original repositories, not about the current tree.

## What C1RCLE is
A ticketing and nightlife-operations platform for India: venues, hosts (organisers) and promoters run events; guests discover events, buy tickets (Razorpay) and enter through a QR scanner at the door; platform admins run finance, support and oversight.

## Reading order
| # | Folder | Read it for |
|---|---|---|
| 00 | `00-system-overview/` | Whole-system picture, repos, topology, request flows. **Start here.** |
| 01 | `01-api-gateway/` | The single backend deployable (Fastify): plugins, lifecycle, endpoint map |
| 02 | `02-core-domain/` | Business logic layer shared by the gateway |
| 03 | `03-contracts/` | The wire contract and how both repos stay in sync |
| 04 | `04-auth-and-rbac/` | Sessions, cookies/CSRF, OTP, roles, admin dual control, scanner credentials |
| 05 | `05-data-and-storage/` | Firestore collections, storage, Redis, encryption |
| 06 | `06-partner-dashboard/` | Venue / host / promoter web app: **in-depth set of 15 files** (frontend + backend), start at its `README.md` |
| 07 | `07-admin-console/` | Internal operations console: **in-depth set of 16 files** (frontend + backend + money/support/authority flows), start at its `README.md` |
| 08 | `08-guest-portal/` | Consumer site and checkout: **in-depth set of 10 files** (routes, BFF, discovery and SEO, checkout and recovery, tickets and QR, defects), start at its `README.md` |
| 09 | `09-scanner-app/` | Door staff mobile/web app: **in-depth set of 15 files** (app + door backend, QR and admission, wallets, offline), start at its `README.md` |
| 10 | `10-payments-and-finance/` | Money flow: **in-depth set of 12 files** (pricing, checkout, hold, ledger and journal, Razorpay, payouts, refunds, reconciliation, failure cases), start at its `README.md` |
| 11 | `11-infrastructure-and-cicd/` | Render, Vercel, nginx, Cloudflare, CI/CD |
| 12 | `12-status-and-known-issues/` | What is live vs partial vs unmerged; audits; open risks |
| 13 | `13-reference/` | Generated route, env and collection indexes; services, errors, security and privacy, operations, decisions, glossary, getting started, **coverage and gaps** |
| - | `diagrams/` | Mermaid sources (also embedded in the pages) |

## Edge architecture
- nginx and the target edge design (current state, constraints, three horizons, defect register): `00-system-overview/NGINX-DREAM-ARCHITECTURE.md`, rendered in `00-system-overview/nginx-dream-architecture.html`

## Visual guides (diagrams first)
- Partner dashboard: `06-partner-dashboard/00-visual-guide.md`, rendered in `06-partner-dashboard/visual-guide.html`
- Admin console: `07-admin-console/00-visual-guide.md`, rendered in `07-admin-console/visual-guide.html`
- Scanner app: `09-scanner-app/00-visual-guide.md`, rendered in `09-scanner-app/visual-guide.html`
- Payments and finance: `10-payments-and-finance/00-visual-guide.md`, rendered in `10-payments-and-finance/visual-guide.html`
- Guest portal: `08-guest-portal/00-visual-guide.md`, rendered in `08-guest-portal/visual-guide.html`

The `.html` files draw the diagrams with the Mermaid library loaded from a CDN, so open them in a browser with internet access. The `.md` files also render on GitHub and in VS Code.

## Status legend (used everywhere)
- **Live**: on `staging` or `main` and in use.
- **Partial**: exists but incomplete or fixture-backed.
- **Unmerged**: implemented in an open PR/branch, **not** on `staging`/`main`.
- **Planned**: designed, not built.

## Important caveats about this snapshot
- Both repos were on feature branches when this was written (backend `fix/partner-connections-and-events`, frontend `fix/partner-connections-contracts`). Where a page says "Unmerged" it means open PR or branch only; the payments stack and frontend PR #72 work are on staging (see `12-status-and-known-issues/`).
- Memory and several docs in the repos are point-in-time; where code and docs disagree this package follows the code and notes the drift.
- No secret values appear here, only environment variable **names**. Account/project identifiers have been left out deliberately.

## Existing docs in the repos (linked, not duplicated)
Backend: `docs/architecture/README.md`, `docs/architecture/decisions.md` (D-001..D-030), `docs/ci-cd.md`, `docs/nginx/sidecar-deployment.md`, `docs/api-contracts/*`, `docs/roadmap/ROADMAP.md`.
Frontend: `docs/architecture/README.md` and `docs/architecture/adr/`.
