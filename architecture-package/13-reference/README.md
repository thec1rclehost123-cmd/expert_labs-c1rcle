# 13. Reference

Cross-cutting material that does not belong to one service. Three of these files are **generated from source** (so they stay exact); the rest are written from code reads and verified where noted.

| File | What it is |
|---|---|
| `GETTING-STARTED.md` | Prerequisites, run modes (memory, Firestore, emulator), env files, running each app, every seed and ops script, third-party setup, seeding order, pitfalls |
| `ENV-MATRIX.md` | **Generated.** All 32 backend variables from the config schema with production rules, plus frontend, nginx, CI and staging-script names, and name traps |
| `API-ROUTE-INDEX.md` | **Generated.** All 275 route registrations with method, full path, rate class, permission, cache, idempotency, source line; plus the 155 routes with no `requirePermission` |
| `COLLECTIONS-INDEX.md` | **Generated.** All 69 `v2_*` Firestore collections with the files that reference them |
| `SERVICES-INDEX.md` | Services, models, ports, contracts and route families, with where each is documented and the documentation gaps |
| `ERROR-CODES.md` | The wire envelope, the 12 codes, internal domain codes, who emits what |
| `SECURITY-AND-PRIVACY.md` | PII inventory, controls, secrets and rotation, findings, missing policies |
| `OPERATIONS-AND-DELIVERY.md` | Branching, CI/CD for both repos, conventions, release gaps, observability, load testing, backups |
| `ADR-INDEX.md` | D-001 to D-030 with the duplicate D-030, and decisions that have no ADR |
| `GLOSSARY.md` | Terms used across the code and docs |
| `DEEP-DIVE-AUTH-INTERNALS.md` | How authentication really works: gateway pipeline, Better Auth settings, the 7-day token, frontend session store and API client behaviour |
| `DEEP-DIVE-SOCIAL-SAFETY-LEADERBOARD-PROFILE.md` | Follows and notifications, guest profile, promoter leaderboard, and the unwired safety-report feature |
| `DEEP-DIVE-FRONTEND-SHARED-PACKAGES.md` | The 14 shared frontend packages and the rules that bind them |
| `DEEP-DIVE-OTP-RSVP-CONTRACTS.md` | Email OTP (and why verification is not enforced), free-ticket RSVP, the analytics backfill script, the contracts package and the parity check |
| `DATA-MODEL-FIELDS.md` | **Generated.** Every domain aggregate with all fields, types and notes (155 interfaces, 1,330 fields) |
| `CONTRACT-SCHEMAS.md` | **Generated.** All 363 exported zod schemas with kind and top-level fields (optional, nullable and default markers) |
| `COVERAGE-AND-GAPS.md` | What the package covers, what it does not, and accuracy risks |

## Regenerating the generated files
They were produced by scripts that scan the backend checkout (`apps/api-gateway/src/routes/v2/**`, `apps/api-gateway/src/config/index.ts`, `packages/core/src/**`). They describe the **checked-out backend branch** (`fix/partner-connections-and-events`); staging is 9 commits ahead (none touch payments or the door route list, but PR #138 changes the offline manifest and sync handlers). Re-generate against the branch you care about before relying on counts. The route index lists registrations by regular expression: routes registered in loops (`${action}`) appear with the placeholder.
