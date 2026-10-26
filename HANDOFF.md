# Handoff notes

This repository is a clean, single-repository snapshot assembled on **2026-10-26** from the two original `C1RCLE-BACKEND` and `C1RCLE-FRONTEND` repositories. The original Git histories are **not** included: this tree was created from each repository's tracked files at its then-current branch, then sanitized.

## Read this before sharing

1. **Do not add the original `.env.local` files.** They contain live credentials (Firebase service-account key, Razorpay secret, Resend key, Better Auth secret, encryption key, magic-ticket secret and OTP secret). They were deliberately left out.
2. **Rotate every credential that ever lived on a developer machine.** Removing a file from this snapshot does not invalidate a key. Rotate the original Firebase service account, Better Auth secret, Razorpay keys/webhook secret, Resend API key, `ENCRYPTION_KEY`, `MAGIC_TICKET_SECRET`, `EMAIL_OTP_SECRET`, Redis credentials and any CI/deploy tokens.
3. **Decide on the NDA before delivering `architecture-package/`.** It is written for the receiving team, describes known weaknesses and security findings, and is marked confidential. If the third party has not signed an NDA, remove `architecture-package/` and regenerate a sanitized architecture overview.
4. **Do not give the third party access to the original cloud accounts.** Create new Firebase/GCP, Render/Vercel, Razorpay, Resend and Redis projects for them (or grant access only after a written agreement).
5. **Review the license/ownership terms.** No `LICENSE` file was added. Add one that matches the commercial agreement before handing over source.

## What was removed

### Secrets and credentials

- Every real `.env`, `.env.local`, `.env.*.local` file (backend and all four frontend apps).
- `.firebaserc` (original Firebase project alias) and `.mcp.json` (developer-local tool paths).
- `.claude/` and `.code-review-graphignore` (local agent/worktree state and local command hooks).
- `apps/api-gateway/src/lib/v1-read-only-client.ts` and `apps/api-gateway/src/scripts/migrate-and-seed-v1-sample.ts` (read credentials from a hard-coded developer path and targeted the old v1 project).
- `apps/api-gateway/src/scripts/create-scanner-event.ts` and `verify-scanner-event.ts` (contained a real test-owner user id, organization id, door code and email, and wrote to the original Firestore project).
- Hard-coded seed password in the scanner E2E scripts was replaced with the `SEED_E2E_OWNER_PASSWORD` environment variable.
- Fixed local Windows path in the roadmap docs and the personal email in the architecture decision log.

### Internal / personal material

- Root and per-repo `task.md` execution plans, `cloudflare_plan.md`, audit output and notebook/junk files.
- Work logs: `docs/Sagar_code_Changes/`, `docs/WORK_DONE_BY_SAGAR_*.md`, `docs/Intern/`, frontend `docs/Interns/`, frontend `docs/superpowers/`.
- Internal gap/status audits: `docs/architecture/IMPLEMENTATION-STATUS-2026-08-31.md`, `docs/frontend-backend-integration-audit.md`, `docs/admin-console/ADMIN-CONSOLE-GAP-CLOSURE-2026-09-11.md`, `docs/operations/payment-audit-2026-10.md`, `docs/operations/payments-release-gate.md`, `docs/operations/financial-journal-transition.md`, `docs/scanner-app/task.md`.
- Point-in-time planning material: most of `docs/reference/` (kept only `V1_TO_V2_PARITY.md`, `route-manifest.ts` and `frontend-api-map.md`), `docs/frontend-services/`.
- Personal/local references in roadmap and decision docs.
- `CLAUDE.md` files, `.github/` workflows and templates (they referenced the original two-repository org, cross-repo tokens, Render service ids and deploy pipeline; recreate CI for your environment).
- QA screenshots under `qa-artifacts/`.
- Build/runtime output: `node_modules/`, `dist/`, `.next/`, `.turbo/`, `coverage/`, `tmp/`, logs.

### Infrastructure identifiers

- Original live API hostname replaced with `https://api.example.com` in docs, scripts, tests, the scanner `.env.example` and `render.yaml`.
- `render.yaml` service name changed to `c1rcle-api` and re-labelled as a handoff example. Replace it with your own service name and domain before adopting it.
- Original GitHub organization/repository defaults replaced in `docs/ci-cd.md`.
- The scanner `.env.example` now points at `http://localhost:8080` instead of the original staging API.

## What was kept

- All application source, tests, migrations/seed scripts, package manifests, lockfiles, TypeScript/ESLint/Prettier configs and workspace files.
- Docker, nginx, Firestore rules/indexes and Render blueprint examples (sanitized identifiers only).
- `.env.example` templates, including the public Firebase web-app placeholders.
- Product/engineering documentation needed to work on the apps: architecture, API contracts, roadmap, scanner-app backend docs, nginx/operations docs, admin-console docs, integration flows and frontend ADRs.
- `architecture-package/` (under NDA) with system overview, app-by-app deep dives, data model, API reference, payments/finance, infrastructure and known issues.

## Recommended next steps

```sh
git init
git add .
git commit -m "Initial C1RCLE platform handoff snapshot"
git remote add origin <your-new-empty-repository-url>
git push -u origin main
```

Then, in the receiving repository:

1. Copy each app's `.env.example` to `.env.local` and fill in **new** credentials.
2. Run `pnpm run install:all` and `pnpm run check` to confirm the baseline.
3. Recreate CI (the old workflows are intentionally absent). A minimal starting point is one job per workspace running `pnpm install --frozen-lockfile` and `pnpm check`.
4. Replace `api.example.com` in `render.yaml` and the scanner `.env.example` with your own domains.
5. Keep `architecture-package/` access-controlled until the NDA is signed.

## Known limitation of this snapshot

This is a working-tree snapshot, not an import of the original commit history. If you need provenance (blame, PR history, old branches), keep the original private repositories archived and grant read-only access separately rather than merging their histories into the shared repository.
