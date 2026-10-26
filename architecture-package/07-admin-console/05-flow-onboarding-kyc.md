# 05. Flow: onboarding review and KYC

Code: FE `app/onboarding/page.tsx`, `app/kyc-review/page.tsx`, `components/admin/onboarding-application-detail`, `lib/admin/document-window.ts`; BE `routes/v2/admin/onboarding-review.ts`, `packages/core/src/application/onboarding/onboarding-service.ts` (≈L296-580), `domain/models/onboarding.ts`. Applicant side: `../06-partner-dashboard/05-feature-onboarding-kyc.md`.

## Queue and detail
`GET /admin/onboarding/applications?status&limit` (`listQueue`, any admin) and `GET …/:requestId` (`getForReview`). The console's Onboarding desk defaults to `submitted`; KYC review lists `submitted` + `changes_requested`.

## Document review
1. `GET …/documents/:label/read-url` mints a signed GET URL (TTL `READ_URL_TTL_MS`, 10 minutes). **Any admin including `support`; unaudited.** The console opens it via `useDocumentWindow` (blank window opened synchronously with `opener=null`, then `location.href = url`; falls back to same tab if blocked; URL scheme/host not validated, trusted from the gateway).
2. `POST …/documents/:label/verify` and `…/reject` (body `rejectKycDocumentSchema`, reason required): `authorize('ONBOARDING_APPROVE')` (TIER2), idempotent (`admin.onboarding.document.verify|reject`), audit `onboarding.document.verify|reject`. They change the **document** state (`pending|verified|rejected`), never the request status.

## Decision
- `POST …/:requestId/approve` (optional body `reviewOnboardingSchema {note}`), `…/reject`, `…/request-changes`: all TIER2 `ONBOARDING_APPROVE`, idempotent (`admin.onboarding.approve|reject|request-changes`), `SENSITIVE_COMMAND`.
- **FSM:** `draft → submitted`; `submitted → approved | rejected | changes_requested`; `changes_requested → submitted`; approved/rejected terminal.
- **KYC gate:** `allRequiredDocumentsVerified` needs `id_front, id_back, selfie` (individual) or, for `entityType === 'business'`, `registration_certificate, sig_id_front, sig_id_back, sig_selfie`. The service pre-checks it **before** provisioning anything (400 otherwise). The console mirrors the rule to disable the Approve button.
- **Approve provisions:** creates the `Organization` (name = `legalName`, slug = name + last 6 of request id, owner = applicant, only the requested capability, `platformFeePercent` from plan: basic 15, silver 12, diamond 10, timezone `Asia/Kolkata`); for `venue` also a starter `Venue`; then saves the approved request and audits. **The organisation is saved first and the writes are not atomic**; a retry with a different idempotency key after a mid-failure could create a second organisation (random id). Using the same key replays safely.
- **Request-changes:** requires a note; sends a best-effort email (`sendOnboardingChangesRequestedEmail`, failure logged only). **No approval email exists.**
- Reject/Request-changes/Approve in the console share one inline note box (note optional in the UI for approve and request-changes).

## Defects and gaps
- Document read-URL minting is unaudited and open to every admin role (headline gap; see `14-…`).
- Approve is not atomic (above); proposal-style dual control is not applied to approvals (single TIER2 admin decides).
- `onboarding-review.test.ts` (237 lines) covers lists, revoke replay and audit reads but **not** the main approve/reject/request-changes/verify flows directly (reported: covered elsewhere, e.g. `routes/v2/onboarding.test.ts`, `organization-actions.test.ts`, `payouts.test.ts`).
- Backend PR #109 (business/signatory onboarding requirements) is open; FE PR #45 and BE PR #86 (KYC document review gate) are merged. Whether the business document set is deployed on staging is **unverified**.
- The verification providers (format-check; Firebase phone) are advisory and not persisted to the request; the admin console shows document state, not provider outcomes.
