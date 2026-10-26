# Coverage and gaps: what this package covers and what it does not

Result of the "check what is missing" audit, 2026-10-24. Method: a mechanical scan of the source against the docs, then manual review of each area by reading source and verifying key claims with `gh`/`git`.

## 1. Mechanical coverage (scripted)
| Item | In source | Mentioned in docs |
|---|---|---|
| Gateway route registrations | 275 | 275 (generated index) |
| Firestore collections | 69 | 69 |
| Backend env vars (config schema) | 32 | 32 (the 4 extra names flagged are dev-only constants in other files) |
| Core application services | all | all, by name, in `SERVICES-INDEX.md` |
| Operational scripts | all | all (GETTING-STARTED) |
| Guest-portal pages | 22 | 22 |
| Partner-dashboard pages | 32 | 32 |
| Admin-console pages | 27 | 27 |
The scan is a name match: it proves a thing is mentioned, not that it is explained correctly.

## 2. Package size
About 92 Markdown files, 92k words, 146 Mermaid diagrams (all parse), 7 browser viewers (`visual-guide.html` for partner, admin, scanner, payments, guest portal; nginx dream architecture).

## 3. Depth by area
| Area | Depth | Notes |
|---|---|---|
| Partner dashboard, admin console, scanner app, payments and finance, guest portal | **In-depth sets** (index, visual guide, flows, defects, testing) | |
| Edge / nginx | One deep page (current state, dream architecture, defect register) | |
| Auth, shared frontend packages, social/safety/leaderboard/profile | **Deep-dive pages** in `13-reference` | |
| API gateway, core domain, contracts, data, infrastructure | **First-pass single pages**, enriched by the generated indexes in `13-reference` | candidates for a deeper pass |
| Reference | route, env, collection, services, error, security, operations, ADR, glossary | |

## 4. Known gaps (not yet documented in depth)
1. ~~Social, leaderboard, safety, guest profile~~ now covered in `DEEP-DIVE-SOCIAL-SAFETY-LEADERBOARD-PROFILE.md`; ~~auth internals and shared frontend packages~~ covered by the two other deep dives (UI kit and ESLint config only at outline level).
2. ~~Contracts internals, parity tooling, OTP flow~~ in `DEEP-DIVE-OTP-RSVP-CONTRACTS.md`; ~~field-by-field schemas~~ in `CONTRACT-SCHEMAS.md` (names and optionality; constraints such as min, max and regex are in the source files).
3. ~~Better Auth tables and cookies~~ in `DEEP-DIVE-AUTH-INTERNALS.md` section 7.
4. **Analytics read model** formulas are described in the partner set, not as a standalone spec.
5. **Mobile builds and releases** (EAS profiles, signing, store listings) do not exist in the repo, so there is nothing to document; flagged as a gap.
6. **Data**: field-level reference now exists (`DATA-MODEL-FIELDS.md`, generated from the domain interfaces); it describes domain objects, not any stored fields that differ, and does not map fields to indexes.
7. **Legal and policy**: privacy notice, terms, retention, erasure, DR and incident response do not exist (see `SECURITY-AND-PRIVACY.md`).
8. **Frontend shared packages** (14): `DEEP-DIVE-FRONTEND-SHARED-PACKAGES.md`, including the UI kit and every lint rule group (the `react-native` preset was only skimmed).
9. **Runtime behaviour on staging** was not exercised for most areas; one earlier read-only check of staging endpoints was used for edge claims only.

## 5. Accuracy risks
- Every claim labelled "reported" came from a code read and was not independently re-run. Where the docs and code disagree, the code was followed.
- The checked-out backend branch is behind staging by about 9 commits (including PRs #140–#143 and the offline-manifest change #138); counts in the generated indexes describe the checked-out branch. Re-generate before quoting numbers.
- PR and branch states change daily: confirm with `gh pr list` before acting on any "open" or "merged" label (verified 2026-10-24).
- Earlier drafts were wrong on: payments stack and frontend #72 shown as unmerged, "refunds never settle", #138/#81 merge state, the partner legacy trees, `canDo` usage and TanStack Query use. All were corrected; if you find another inconsistency, trust the source file cited next to it.
- The mechanical audit cannot detect a page that mentions a feature but gets it wrong.

## 6. Handling
No secret values, account IDs, personal emails or per-person logs are included; environment variables appear by name only. Recommend an NDA before sharing: the package includes unfixed security findings. Nothing has been sent or uploaded.
