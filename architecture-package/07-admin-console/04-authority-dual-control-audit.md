# 04. Authority, dual control and audit

Code: `packages/core/src/domain/models/admin-authority.ts`, `application/admin/admin-authority-service.ts`, `routes/v2/admin/onboarding-review.ts`, `domain/ports/audit.ts`.

## Identity
`PlatformAdmin {id (= Better Auth user id), email (lower-cased), role, isActive, version…}` in `v2_admins`. `requireAdmin(userId)` loads the record and throws `UnauthorizedError('Platform admin authority required')` if missing or inactive (a revoked admin looks like a non-admin). The record is never inferred from org membership or the Better Auth `user.role`. Admin routes use **no** `requirePermission`, no `X-Organization-Id` and no org RBAC; identity is `services.actor(request).userId` from the session. (Memory driver outside production fabricates an actor from `x-user-id` or defaults to `user_1`: test-only, never on Firestore.)
`authorize(action)`: TIER1 any admin; TIER2 `TIER2_ROLES` = `super|admin|ops|finance`; TIER3 `super`. Failure: 403 `Role "x" cannot perform ACTION (tier n)`.

## Rate limits and idempotency on admin routes
Reads `AUTH_READ` (240/min); every mutation `SENSITIVE_COMMAND` (**10/min** per IP+user+org, so a bulk operator will be throttled). Every POST/PUT/DELETE requires an `Idempotency-Key` header (`idempotencyKeySchema`) and runs inside `runIdempotent`: key scoped `${actorId}:${commandName}:${key}`, 24 h TTL, request hash over method + URL + path params + validated body; same hash replays the stored status/body; different hash → 409 with conflict id; in-flight duplicate → 409; a previously **FAILED** key retains its claim and returns `IdempotencyFailedError` (409), so after a failure the admin must use a new key (repository docs predate this, task B1). Repeating a no-op transition returns success without rewriting the row or writing a new audit row. There is **no `If-Match` on any admin route** (admin has no PATCH; `PUT /admin/settings/platform` is last-write-wins).

## Dual control (TIER3 proposals)
`ProposedAction {id, action, proposedBy, reason, payload (frozen free-form), status, resolvedBy, resolvedAt, resolutionReason, version…}` in `v2_proposed_actions`. **Status: `pending | approved | rejected | cancelled`**. (Docs say `proposed`; code says `pending`.) There is **no `executed` status, no `executedAt/By` and no expiry**.
1. **Raise** `POST /admin/proposals` (strict body: action, reason 1–2000, optional payload). `propose` requires the proposer's role to be able to initiate the action, requires a dual-control (TIER3) action (TIER1/TIER2 refused with 400) and a non-empty reason. Audit `proposal.raise:<ACTION>`.
2. **Approve/Reject** `POST /admin/proposals/:id/{approve,reject}` (optional reason ≤ 2000): requires `pending`; the resolver must differ from the proposer (domain check) and must be able to initiate the action, i.e. be `super`. Exactly **one** second approver (no N-of-M). Audit `proposal.approved|rejected:<ACTION>`.
3. **Cancel** `POST …/cancel`: proposer only, only while `pending`.
4. **Execute** (separate routes, each idempotent) reads parameters from the **stored payload**, never from the request body; `isExecutable` is simply `status === 'approved'`.
| Action | Execute route | Notes |
|---|---|---|
| `ADMIN_PROVISION` | `POST /admin/proposals/:id/provision-admin` (201) | re-creates a deactivated admin at `version+1`; does not verify the target user id exists in auth users (a typo makes a useless record); "already active" guard |
| `ADMIN_ROLE_UPDATE` | `…/update-admin-role` | last-super guard scans `admins.list({limit:1000})` which Firestore caps at 100 |
| `COMMISSION_ADJUST` | `…/adjust-commission` (in `organization-actions.ts`) | `adjustPlatformFeePercent` integer 0–100; no-op if unchanged |
| `FINANCIAL_ADJUSTMENT` | `…/post-financial-adjustment` (in `reconciliation.ts`, route-inline) | requires `journalTransition.readModel === 'financial_journal_v2'`; payload validated here (`{organizationId, amountPaise int>0, direction credit\|debit}`), not at propose time |
| `PAYOUT_FREEZE` | `…/freeze-payout` | not behind the payouts gate |
| `PAYOUT_RELEASE` | `…/release-payout` | behind `requirePayoutsEnabled()` |
Payload readers (`readProvisionPayload`, `readRoleUpdatePayload`, `readCommissionPayload`, `readPayoutIdPayload`) re-validate at execute time and fail loudly (400); a malformed approved proposal is stuck and must be re-raised.

**Consequences (verified):** an approved proposal can be executed repeatedly by any `super` (including the proposer or the approver) with a fresh `Idempotency-Key`: an old approved freeze can re-freeze a payout after it was released; an old role update can re-apply a stale role; `FINANCIAL_ADJUSTMENT` journals are idempotent by key `m2:manual-adjustment:<proposalId>` but each re-execution writes another audit row. Dual control needs only **two `super` accounts**.
**Deliberate asymmetries:** `revokeAdmin` is a single-actor `super` act (`POST /admin/admins/:id/revoke`: no self-revoke, no reason field, no last-super guard, deactivates rather than deletes) so removing power stays easy; venue suspend is TIER2 direct, not dual control.
No `GET /admin/proposals/:id`, no listing filter by proposer or action (the internal `getProposal` exists).

## Audit trail (`v2_admin_audit_logs`)
`AdminAuditRecord`: `id, adminId, actorId?, organizationId?, adminRole?, action?, targetType?, targetId?, before?, after?, reason?, ipAddress?, userAgent?, occurredAt (epoch ms)`. Written by `AdminAuthorityService.record` after the state change (**not atomic** with it). The Firestore adapter uses `.create()` and **silently ignores an already-exists error**, so a duplicate id is dropped. Reads: `listRecent` (order by `occurredAt desc`) and `listForTarget` (composite index `targetId` + `occurredAt`); `GET /admin/audit` has `limit` 1–200 (default 50), optional `targetId`, **no cursor**. Storage-level immutability: only Firestore rules deny-all for clients; no hash chain.
Wire DTO `adminAuditRecordDtoSchema` omits `ipAddress`, `userAgent`, `organizationId` and adds live-resolved `targetName` for `venue`, `event`, `organization` and (only for `super`/`finance` viewers) `platform_user`.
Action strings seen: the `AdminAction` values plus `proposal.raise|approved|rejected|cancel:<ACTION>`, `ADMIN_REVOKE`, `ADMIN_SEED`, `ADMIN_EXPORT`, `PLATFORM_SETTINGS_UPDATE`, `FINANCIAL_REFUND.request|approve|reject|execute`, `onboarding.*` (incl. `onboarding.document.verify|reject`), `SUPPORT_*`, `SAFETY_REPORT_*`.

### Audit completeness defects (verified unless noted)
- `onboarding-review.ts` (≈L410-418): the `approve`, `reject` and `cancel` proposal closures take `(userId, proposalId, reason)` and **drop the `meta` argument** that `registerProposalAction` passes, so IP and user agent are not recorded for those three actions (raise and execute routes do forward them).
- Suspend/reinstate for venues, organisations and users, and pause/resume/force-complete for events, record `reason: null` (there is no reason input at all). Idempotent no-ops write no audit row.
- Settings update audits only fee rate, thresholds, `maintenanceMode` and the accounting-policy **version**; `featureFlags` and most policy fields are not recorded.
- `executeRefund` writes no audit row on the provider-rejection and ambiguous-outcome paths (reported).
- KYC document read-URL minting (`issueDocumentReadUrl`) is unaudited though it exposes identity images; venues/events/hosts/orders/tickets/promotions/refunds/reconciliation CSV exports are unaudited (only users and audit exports write `ADMIN_EXPORT`).
- The seed script writes `ADMIN_SEED` with id `seed_${Date.now()}` and no IP/user agent.
- Replies and notes write the first 80 characters of content as the audit `reason`, leaking message text into the audit trail.

## Seed script
`pnpm --filter api-gateway seed:admin` (`apps/api-gateway/src/scripts/seed-platform-admin.ts`): needs `--user-id` (Better Auth user id) and `--email`; `--role` defaults to `super`; refuses the memory driver; refuses to run if any active admin exists unless `--force`. The roster check uses `list({limit:1})` then filters, so it only sees the first document (if that one is inactive and later ones are active it could wrongly proceed).

## Separation of duties (summary)
| Control | Enforced? |
|---|---|
| Proposer ≠ approver (proposals) | Yes (domain) |
| Executor ≠ proposer/approver | **No** |
| Refund requester ≠ approver; no repeat approver | Yes |
| Refund requester ≠ executor; zero-approver refunds | **No**: under the single-approver threshold with no redeemed ticket, one admin can request and execute alone |
| Payout `approvedBy ≠ requestedBy` | Yes, but the requester is a partner user, so it never constrains an admin |
| Reconciliation resolve, dispute resolve, onboarding approval, suspension, ban | Single admin, no second signature |
| Platform settings (incl. refund thresholds) | No tier, no dual control |
