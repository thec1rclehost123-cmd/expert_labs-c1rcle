# 11. Backend API reference and FE↔BE mapping

All routes under `/api/v2/admin` unless noted; registered in `apps/api-gateway/src/routes/v2/route-manifest.ts` (17 route modules). Reads: `AUTH_READ` (240/min). Mutations: `SENSITIVE_COMMAND` (10/min) and require `Idempotency-Key`. Authority is resolved inside services (`requireAdmin`/`authorize`), not by `requirePermission`. Errors: non-admin or deactivated → 401; role too low → 403; not found → 404; `version_conflict`, `conflict`, idempotency conflict/in-flight/failed, `state_transition` → 409; `invalid_operation` → 400; unmapped → 500 (`mapDomainError` in `routes/v2/partner/events.ts`; finance routes use a local copy that does not map `state_transition`/`conflict`). Zod failures 422 with `fieldErrors`. Lists `{items, pageInfo:{page:1, pageSize, total, hasNextPage}}`. Responses validated with `validateV2Response`.

## Endpoint table (BE file → FE function in `admin-api.ts`)
| Area | Endpoints | Authority | FE |
|---|---|---|---|
| **Proposals** (`onboarding-review.ts`) | `GET /proposals?status`; `POST /proposals` (201); `POST /proposals/:id/{approve,reject,cancel}` | any admin to read; raise by role-eligible; resolve = `super` ≠ proposer | `listProposals`, `raiseProposal`, `approveProposal`, `rejectProposal`, `cancelProposal` |
| **Proposal executors** | `…/provision-admin`, `…/update-admin-role`, `…/adjust-commission`, `…/freeze-payout`, `…/release-payout`, `…/post-financial-adjustment` | TIER3 `super` | `provisionAdminFromProposal`, `updateAdminRoleFromProposal`, `adjustCommissionFromProposal`, `freezePayoutFromProposal`, `releasePayoutFromProposal`, `postApprovedFinancialAdjustment` |
| **Admins/audit** | `GET /admins`, `POST /admins/:id/revoke` (super, no self-revoke), `GET /audit?limit≤200&targetId`, `GET /audit/export.csv` | any admin (revoke: `super`) | `listAdmins`, `revokeAdmin`, `listAudit`, audit CSV |
| **Onboarding** | `GET /onboarding/applications[/:id]`, `GET …/documents/:label/read-url`, `POST …/documents/:label/{verify,reject}`, `POST …/:id/{approve,reject,request-changes}` | read any admin; mutations TIER2 | `listOnboardingApplications`, `getOnboardingApplication`, `getOnboardingDocumentReadUrl`, `verifyOnboardingDocument`, `rejectOnboardingDocument`, `approve/reject/requestOnboardingChanges` |
| **Refunds** (`refunds.ts`) | `POST /refunds` (201), `POST /refunds/:id/{approve,reject,execute}`, `GET /refunds`, `GET /refunds/:id`, `GET /refunds/export.csv` | TIER2 `FINANCIAL_REFUND` | `requestRefund` (unused in UI), `approveRefund`, `rejectRefund`, `executeRefund`, `listRefunds`, CSV; `GET /refunds/:id` BE-only |
| **Disputes** | `GET /disputes[/:id]`, `POST /disputes/:id/resolve` | resolve TIER2 | `listDisputes`, `resolveDispute`; `/:id` BE-only |
| **Payouts** | `GET /payouts[/:id]`, `POST /payouts/batch-run`, `POST /payouts/:id/complete` | TIER2; batch-run gated, **complete not gated** | `listPayouts`, `runPayoutBatch`, `completeManualPayout`; `/:id` BE-only |
| **Reconciliation/chargebacks** (`reconciliation.ts`) | `POST /reconciliation/runs`, `GET /reconciliation/runs[/:id]`, `GET /reconciliation/discrepancies[/:id]`, `POST …/discrepancies/:id/{retry,resolve}`, `GET /reconciliation/export.csv`, `GET /chargebacks` | TIER2 for run/resolve | `runReconciliation`, `listReconciliationDiscrepancies`, `retry…`, `resolve…`, `listProviderChargebacks`; run detail/discrepancy detail/export BE-only |
| **Directory** (`directory.ts`) | `GET /venues|events|hosts|users` and each `/export.csv`, `GET /lookup` | any admin | `listVenues/Events/Hosts/Users`, `globalLookup`, CSV functions |
| **Commerce reads** | `GET /orders`, `/tickets`, `/promotions`, `/promoters` + `export.csv`; `GET /analytics` | any admin | `listOrders`, `listTickets`, `listPromotions`, `listPromoterAssignments`, `getAnalyticsSummary` |
| **Direct commands** | venues `suspend|reinstate`; organizations `suspend|reinstate`; promoters `suspend|reinstate`; users `ban|unban`; events `pause|resume|force-complete` | TIER2 (events TIER1) | matching `suspend…/reinstate…/ban…/pause…` functions |
| **Settings** | `GET|PUT /settings/platform` | **no tier** | `getPlatformSettings`, `updatePlatformSettings` |
| **Support** (`support.ts`) | list, detail, `assign, priority, reply, notes, link, resolve, merge, escalate, close, reopen, restore`, `DELETE /:id` | any admin (TIER1) | `listSupportTickets`, `getSupportTicket`, … |
| **Health** (non-admin) | `GET /api/v2/internal/{readiness,version}` | token-gated at nginx | via admin BFF `/api/health/*` |

### FE↔BE mapping summary
- No FE-only (missing-backend) calls found; **no fixtures** in the admin console.
- **BE-only (no FE caller):** `GET /admin/refunds/:id`, `/disputes/:id`, `/payouts/:id`, `/reconciliation/runs/:id`, `/reconciliation/discrepancies/:id`, `/reconciliation/export.csv`.
- **Docs-only/ambiguous:** the plan docs mention `GET/POST /admin/lookup`; only GET exists.
- **Unreachable on the backend:** no `/safety` and no alerts routes.

## Idempotency commands (selection)
`admin.proposals.raise|approve|reject|cancel`, `admin.provision`, `admin.role_update`, `admin.revoke`, `admin.onboarding.approve|reject|request-changes|document.verify|document.reject`, `admin.refund.request|approve|reject|execute`, `admin.payout.freeze|release|batch_run|complete`, `admin.dispute.resolve`, `admin.organization.adjust_commission`, `admin.platform-settings.update`, `admin.support.*`, `admin.event.pause|resume|force_complete`, `admin.venue.suspend|reinstate`, `admin.promoter.suspend|reinstate`, `admin.user.ban|unban`. Scoped `${actorId}:${commandName}:${key}`.

## Wire conventions specific to admin
Money integer paise; ISO timestamps except audit `occurredAt` (epoch ms); `platformFeePercent` and `venueShareRate` whole-number percents (0 is valid; use `??`); a body never carries `role`; `AdminAuditRecord` DTO omits IP/user agent.
