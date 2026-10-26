# 05. Onboarding and KYC

Code: FE `src/app/onboard/PageClient.tsx` (3,303 lines), `src/lib/onboarding/*`, `src/app/api/bff/onboarding/*`; BE `apps/api-gateway/src/routes/v2/onboarding.ts`, `admin/onboarding-review.ts`, `packages/core/src/application/onboarding/onboarding-service.ts`, `domain/models/onboarding.ts`. Status: **Live** (BE `44983ba` email OTP, `39d94e7` phone verification; FE BFF migration in PR #77). Backend PR #109 (business/signatory onboarding requirements) is **open**; verify before assuming business KYC is deployed.

## Wizard (as implemented)
Step order: `role → signup → email_verify → phone_verify → entity_type → details → [kyc_identity (individual) | kyc_business → kyc_signatory (business)] → success`. This supersedes the 2026-09-01 six-step spec (which dropped OTP and treated phone as typed-only).
- **Role:** venue/host/promoter from `?type=`; sent as `requestedType`. Plan fixed to `basic` (`DEFAULT_PLAN`), not shown to the user.
- **Signup:** name, email shape, password ≥ 8; `signUp` then `sendOtp`. A 409 shows a "Welcome Back" login panel that resumes from `getMine()`.
- **Email OTP:** `/api/auth/otp/send|verify` via BFF with CSRF; 6 digits; 15 s resend cooldown (backend: 60 s cooldown, 5-attempt lockout, `OTP_SEND` 5/min, `OTP_VERIFY` 10/min).
- **Phone OTP:** Firebase JS `signInWithPhoneNumber` with an invisible `RecaptchaVerifier` (container `phone-verify-recaptcha`); the Firebase user is signed out right after the ID token is obtained; the token is POSTed to the gateway `verify-document` (`documentType:'phone'`, E.164, `proofToken`) **directly with Bearer**, not via the unused BFF routes. A bare 10-digit number is assumed `+91`. Backend verifies through `FirebasePhoneVerificationProvider` (GCP Identity Platform).
- **Test bypass:** `lib/firebase/phone-auth.ts` sets `appVerificationDisabledForTesting = (NEXT_PUBLIC_ENVIRONMENT !== 'production')`, which is true for `preview` as well as `development`: reCAPTCHA is disabled on every non-production deploy. `NEXT_PUBLIC_FIREBASE_TEST_PHONE` prefills a test number.
- **Entity type:** individual or business.
- **Details:** hand-rolled validators (`lib/onboarding/validation.ts`: name, phone, city typeahead from `cities.ts`, area, website, capacity, instagram, bio, registration number, PAN, GSTIN, CIN, Aadhaar format). `profilePayload()` sanitises; schema is `.strict()`; plan, host category and event text stay local.
- **Start:** `POST /api/bff/onboarding/applications` with an `Idempotency-Key` held in a ref per intent; 409 → "You already have an application in progress."
- **Autosave:** `PATCH /api/bff/onboarding/applications/:id` with the whole profile when advancing KYC steps; failures are silent. No stored step: the resume point derives from `status`, `missingDocuments` and profile fields. `withVersionRecovery` sends `If-Match` from a module-level version map and retries once on 409 after `getMine()`.
- **KYC upload:** client checks ≤ 5 MB and JPG/PNG/WEBP; the BFF `…/documents/upload?label=…` does signed-URL → server PUT → confirm. Labels: `id_front`, `id_back`, `selfie`, `registration_certificate`, `sig_id_front`, `sig_id_back`, `sig_selfie`. Individuals need 3; business needs `registration_certificate` + 3 signatory slots. Progress bar jumps 10→100 (not real). ID numbers are format-checked only and never persisted by the wizard; the format check result must never be rendered as "verified" (D-018).
- **Submit:** last KYC step → `POST …/submit` (idempotent, versioned); a 400 shows field errors or "upload all required documents".
- **Success/polling:** `getMine()` every **10 s**; statuses draft, submitted (shown as pending), changes_requested, approved, rejected. `reviewNote` is hard-coded `''`, so reviewer notes are **not shown**. On approval, `routeAfterAuth` runs when `isApproved` flips.

## Backend contract
| Call | Notes |
|---|---|
| `GET /onboarding/me` → `{request \| null}` | session only; no org header |
| `POST /onboarding/applications` `{requestedType, plan, profile}` | `STANDARD_COMMAND` + `Idempotency-Key`; `onboarding.start`; 201 |
| `PATCH /onboarding/applications/:id` | strict partial profile; no idempotency key (autosave) |
| `POST …/documents/upload-url` | signed PUT; `kyc/<userId>/<requestId>/<label>`, private, 5 MB, 10-min TTL |
| `POST …/documents` | records the label; re-upload resets that doc to `pending` |
| `POST …/submit` | idempotent; refuses while required documents are missing |
| `POST /onboarding/verify-document` | `SENSITIVE_COMMAND`; format-check (aadhaar/pan/gstin) or Firebase phone; 5 attempts per 24 h per applicant; result is advisory, **not persisted** onto the request |

**FSM:** `draft → submitted`; `submitted → approved | rejected | changes_requested`; `changes_requested → submitted`; approved/rejected terminal. One open application per user (draft/submitted/changes_requested). Edits only in draft or changes_requested. Required documents: individual `id_front, id_back, selfie`; business (`entityType === 'business'`) `registration_certificate, sig_id_front, sig_id_back, sig_selfie`.

## Admin approval (platform authority, not partner RBAC)
`admin/onboarding-review.ts`: list/get applications; document `read-url` (any admin, 10-min TTL), `verify`, `reject`; `approve`, `reject`, `request-changes`. All gated by `authority.authorize(adminId, 'ONBOARDING_APPROVE')` = **TIER2** (support cannot approve). **Approve** requires status `submitted` and every required document `verified` (checked before anything is written), then: creates the org (name = `legalName`, slug = name + last-6 of request id, owner = applicant, **only the requested capability**, `platformFeePercent` from plan basic 15 / silver 12 / diamond 10, timezone `Asia/Kolkata`); for `venue` also creates a starter venue (bio, capacity, city); marks the request approved; writes an audit record. The org is written first so a later failure leaves the request retryable. Request-changes needs a note and emails the applicant best-effort; no approval email exists.

## Verification providers
Default `FormatCheckVerificationProvider` regex-checks aadhaar/pan/gstin and reports `provider: 'format-check'`, `reason: 'format_ok'` so it cannot be read as an identity check. On the Firestore driver `CompositeVerificationProvider` routes `phone` to Firebase and everything else to the format check.

## Open points
- Whether the plan step should be user-visible (spec open decision #1); the business-field toggle; document count is a fixed backend enum (more documents = backend contract change first).
- `POST /applications` 409-on-draft behaviour and whether `requestedType`/`plan` are editable on a draft: unconfirmed (owner noted in the spec).
- Whether the e2e journey has ever run against a real Firestore: unverified. Playwright `onboarding.spec.ts` mocks all network (including Identity Toolkit).
- `/api/auth/phone-verification` and `/api/bff/onboarding/verify-document` are dead routes; the wizard's direct gateway call is the live path (and is what e2e asserts).
- Flaky CI test: `onboard/PageClient.test.tsx` "requires a full 6-digit code" under load.
