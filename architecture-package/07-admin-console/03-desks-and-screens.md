# 03. Desks and screens

All pages are client components that fetch live through `lib/admin/admin-api.ts`. No fixtures in app code. List calls use `limit` (default 100). "Confirm" below means an inline Confirm/Cancel panel (no modal, no focus management).

| Route | Purpose | Endpoints | Filters | Mutations and UX |
|---|---|---|---|---|
| `/login` | Email + password | `/api/auth/login` then hard redirect to `/` | none | any 4xx → generic "Invalid email or password" |
| `/` | `OverviewDesk`: 5 counters | `listProposals('pending',1)`, `listRefunds('pending',100)`, `listOnboardingApplications('submitted',1)`, `listAdmins(100)` | none | read-only; failure "Unavailable". Pending refund **value** sums only up to 100 items (wrong above 100); "Admins" capped at 100 |
| `/analytics` | Net revenue, tickets sold, active events, top hosts, truncated-scan warning | `GET /admin/analytics` | none | read-only |
| `/lookup` | Exact-id (or email) search across venue/event/organisation/user, ≥3 chars | `GET /admin/lookup?q=` | none | none. Results show `shortId` (12 chars + ellipsis), so full ids cannot be copied |
| `/proposals` | Dual-control inbox | `GET /proposals?status&limit`; `POST approve/reject/cancel`; executors | status (default `all`) | See `04-…`. Approve/Cancel bare Confirm; Reject needs a reason; approved rows show one executor button by action. The error banner omits `provisionMutation` (failed admin provisioning is silent). Nothing in the UI blocks self-approval (the gateway refuses) |
| `/onboarding` | Application decision desk | list, detail, approve, reject, request-changes, read-url | status (default `submitted`) | Approve disabled until all required documents verified (client mirrors the backend rule). Approve/Reject/Request-changes share an inline note box; note optional. Document viewing via `useDocumentWindow` (pre-opens `about:blank` with `opener=null`, then sets the signed URL; same-tab fallback; URL not scheme-validated) |
| `/kyc-review` | Per-document verify/reject | list (`submitted` + `changes_requested`), detail, read-url, verify, reject | none | Verify disabled once verified; Reject needs a reason; "View all details" panel; no application-level actions |
| `/orders` | Read-only list + CSV | `GET /admin/orders`, `export.csv` | status (client-side over 100 rows) | CSV only |
| `/tickets` | Same shape | `/admin/tickets`, `export.csv` | status (client-side) | CSV (includes holder name, PII) |
| `/promotions` | Promo code list | `/admin/promotions`, `export.csv` | none | CSV |
| `/promoters` | Promoter assignments | `/admin/promoters`, `POST suspend/reinstate`, `export.csv` | none | Suspend has "Confirm suspend"; reinstate has no confirm; bulk across all assignments (`{affected, at}`) |
| `/commissions` | Platform fee per organisation | `listHosts(100)`, `raiseProposal(COMMISSION_ADJUST)` | none | Inline form: integer percent 0–100 + reason → raises a TIER3 proposal; "(free tier)" label for 0%. Units differ from Settings (percent vs fraction) |
| `/support` | Ticket desk (1,118-line page) | see `09-…` | status, priority, category, search, include-deleted | Many mutations; delete, merge, close, escalate have no confirm; all mutation errors roll into one generic banner; search not debounced; draft resets when switching ticket |
| `/refunds` | Refund requests | list, approve, reject, execute, export | status (default `pending`) | Approve single-click (no confirm); Reject needs a reason; Execute / "Retry provider refund" for `approved`/`failed`; approvals column `approvals.length/approversRequired`; `requestRefund` exists in the API layer but **no UI creates a refund request** |
| `/disputes` | Partner-raised disputes | `/admin/disputes`, `POST resolve` | status (default `open`) | Pick upheld/denied + required note, then "Confirm upheld/denied" |
| `/chargebacks` | Razorpay provider cases (`respondBy`, evidence state) | `GET /admin/chargebacks` | none | read-only; no evidence submission or actions |
| `/payouts` | Payout desk | list, `POST batch-run`, `POST /{id}/complete`, `raiseProposal(PAYOUT_FREEZE\|RELEASE)` | status | "Run" = single-click batch of one payout (for `requested`/`processing`/`failed`); Freeze/Release raise proposals then link to `/proposals`; "Record transfer" (for `processing`) collects reference, timestamp, method, **hand-typed SHA-256, content type and size** (no file upload or hashing); completion is direct (not via proposals) |
| `/reconciliation` | Discrepancy list | list, `POST runs`, `POST discrepancies/{id}/retry`, `.../resolve` | state (open, finance_review, support_required, resolved, all) | "Run reconciliation" has no confirm; Resolve needs a reason; Recheck shown only when `repair === 'recheck_evidence'`; state badge always `warning` tone; evidence shows first four key/values |
| `/adjustments` | Two-step manual adjustment | `raiseProposal(FINANCIAL_ADJUSTMENT)`, `POST proposals/{id}/post-financial-adjustment` | none | Propose: organisation id (no picker), amount in paise, direction, reason. Post: **paste an approved proposal id** (no link back to the proposals list; the Proposals page has no executor for this action) |
| `/venues` | Venue list | `/admin/venues`, suspend, reinstate, export | none | Suspend has Confirm; no reason field |
| `/events` | Event list | `/admin/events`, `POST pause/resume/force-complete`, export | none | Pause, Resume, **Force complete (irreversible)** are single-click, no confirm, no reason. Force-complete offered for `published`, `sales_paused`, `started` |
| `/users` | User list | `/admin/users`, `POST ban/unban`, export | none | Ban has optional reason + "Confirm ban"; Unban immediate; footer `n/total shown` |
| `/hosts` | Organisations | `/admin/hosts`, `POST /organizations/{id}/suspend\|reinstate`, export | none | Suspend Confirm; no reason field |
| `/admins` | Admin roster | `GET /admin/admins`, `POST /admins/{id}/revoke`, `raiseProposal(ADMIN_ROLE_UPDATE)` | none | Revoke: "Keep / Confirm revoke" (direct, not dual control). Change role raises a proposal (payload `{targetUserId, role}`, reason required, disabled when unchanged). **No screen raises `ADMIN_PROVISION`** though the page says provisioning goes through Proposals (dead flow) |
| `/audit` | Audit trail | `GET /admin/audit?limit=200&targetId=`, `export.csv` | target id | Table: action, target (name/type/short id), admin id/role, reason, time. Before/after payloads are in the schema but **not displayed** |
| `/health` | Dependency checks, version, build | `/api/health/readiness` (30 s refresh), `/api/health/version` | none | read-only |
| `/settings` | Own profile and platform settings | `GET/PUT /admin/settings/platform`, `listAdmins(100)` | none | Fee rate (0–1), single and dual refund thresholds (paise), maintenance toggle; sends only changed fields but also sends `maintenanceMode: settings?.maintenanceMode` (can overwrite a concurrent change); maintenance toggle has no confirm; not dual control |

CSV export is offered on 13 desks via blob download (some fail silently): audit, users, venues, events, hosts, promoters, orders, tickets, promotions, refunds (plus reconciliation exists on the backend only). Payouts, disputes, chargebacks and support have no export.

## Not-yet-confirmed UX details
Inline confirm panels swap in place with no focus management; mutation success messages have no `aria-live`; status is often communicated by colour alone. See `02-…` and `14-…`.
