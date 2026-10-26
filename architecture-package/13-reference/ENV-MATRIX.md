# Environment variable matrix

**Generated 2026-10-24** from `apps/api-gateway/src/config/index.ts` (the only reader of `process.env`) plus its production-only validation rules (`superRefine`). **Names only; never put values in docs.** Deployment-only names (nginx, sidecar, CI, frontends) are in the lower sections.

## 1. Backend gateway (`apps/api-gateway`)

| Name | Type / values | Default | Required? | In production | Notes |
|---|---|---|---|---|---|
| `NODE_ENV` | enum: development, test, production | `'development'` | optional |  |  |
| `PORT` | number | `8080` | optional |  |  |
| `HOST` | string | `'0.0.0.0'` | optional |  |  |
| `LOG_LEVEL` | enum: debug, info, warn, error, silent | `'info'` | optional |  |  |
| `PUBLIC_API_URL` | url | `dev URL` | optional | Must be https | Public edge URL used by callbacks and deployment-facing metadata. |
| `ALLOWED_ORIGINS` | string | `local dev origins` | optional | Every origin must be https and not localhost; no wildcards; exact origins only | Comma-separated browser origins. Wildcards are rejected below. |
| `BETTER_AUTH_TRUSTED_ORIGINS` | string |  | optional | Same rules as `ALLOWED_ORIGINS` (defaults to it) | Optional explicit Better Auth origins; defaults to ALLOWED_ORIGINS. |
| `TRUSTED_PROXY_CIDRS` | string | `loopback` | optional | Must not be empty; `0.0.0.0/0` is rejected | Comma-separated Nginx/LB/ingress addresses or CIDRs. |
| `APP_VERSION` | string | `'0.1.0'` | optional |  | Build metadata returned by the internal version endpoint. |
| `BUILD_SHA` | string | `'unknown'` | optional |  |  |
| `REDIS_URL` | string | `'redis://localhost:6379'` | optional | If `RATE_LIMIT_STORE=redis`: must be a `redis://` or `rediss://` URL that is not localhost |  |
| `RATE_LIMIT_STORE` | enum: memory, redis | `'memory'` | optional |  | Where rate-limit counters live. `memory` (default) is per instance and resets on restart; `redis` shares the window across instances using REDIS_URL, and falls back to memory if Redis is unreachable. |
| `PAYOUTS_ENABLED` | string |  | optional | Unset = off in production (release gate); set `true` only with legal, provider and CA approval | Release gate for payouts. While the funds flow (who may hold and distribute ticket money) and the tax rules are awaiting written legal, provider and CA approval, the endpoints that request, batch-run or release a payout are switched off. Unset: ON outside prod |
| `FIRESTORE_PROJECT_ID` | string | `'c1rcle-staging'` | optional |  |  |
| `STORAGE_DRIVER` | enum: memory, firestore | `'memory'` | optional | Must be `firestore` (unless `ALLOW_MEMORY_STORAGE_IN_PRODUCTION=true`, CI smoke only) | B12: which repository adapter set `lib/v2-services.ts` wires up. |
| `FIREBASE_CLIENT_EMAIL` | string |  | optional | Required whenever `STORAGE_DRIVER=firestore` | Firestore service-account credentials (only read/used when STORAGE_DRIVER=firestore). |
| `FIREBASE_PRIVATE_KEY` | string |  | optional | Required whenever `STORAGE_DRIVER=firestore` |  |
| `FIREBASE_STORAGE_BUCKET` | string |  | optional |  | Defaults to `<project>.firebasestorage.app` (KYC images, posters) Bucket for onboarding KYC images. Defaults to `<project>.firebasestorage.app`. |
| `BETTER_AUTH_SECRET` | string | `'dev-only-change-me'` | optional | Required: at least 32 characters and not the development default | B10: Better Auth. |
| `BETTER_AUTH_URL` | url | `dev URL` | optional | Must be https |  |
| `RENDER_GIT_COMMIT` | string |  | optional |  | Used as `BUILD_SHA` when no explicit value is given Render's immutable commit metadata, consumed as BUILD_SHA when no override is supplied. |
| `RAZORPAY_KEY_ID` | string |  | optional (feature off) |  | No payment provider: checkout cannot create orders (readiness fails if only some Razorpay vars are set) Phase 4: Razorpay credentials. |
| `RAZORPAY_KEY_SECRET` | string |  | optional (feature off) |  | See `RAZORPAY_KEY_ID` |
| `RAZORPAY_WEBHOOK_SECRET` | string |  | optional (feature off) |  | Webhook returns 503 (fails closed), so payments are not confirmed by webhook |
| `MAGIC_TICKET_SECRET` | string |  | optional | Required: at least 32 characters (rotating QR, wallet QR, couple tokens, offline manifest) | Dev uses a published default; in production the process refuses to boot Phase 5: HMAC key for rotating door QR codes ("magic tickets") and for signing offline admission manifests. Forging this key forges entry to a paid event, so production refuses to boot wit |
| `ENCRYPTION_KEY` | string |  | optional | Required: at least 32 characters (bank-account encryption) | At-rest encryption of bank-account numbers (AES-256-GCM, scrypt-derived). Without it the core falls back to a secret published in this repository, which protects nothing — production refuses to boot without a real one. Rotating it does not strand older records |
| `ENCRYPTION_SALT` | string |  | optional |  | Optional scrypt salt; changing it re-keys new writes Optional scrypt salt for ENCRYPTION_KEY; changing it re-keys new writes. |
| `RESEND_API_KEY` | string |  | optional (feature off) |  | Email falls back to a logging sender: no OTP, reset or invitation emails are delivered Email OTP delivery and at-rest OTP HMAC key. |
| `EMAIL_OTP_SECRET` | string |  | optional | Required: at least 32 characters |  |
| `PARTNER_DASHBOARD_URL` | url |  | optional |  | Invitation emails omit the accept link Public origin of the partner dashboard (e.g. https://partners.thec1rcle.com). Used to build absolute staff-invitation accept links in invitation emails. Optional: when absent, emails omit the link rather than linking nowh |
| `CORS_ALLOWED_ORIGINS` | string |  | optional | If set: exact https origins, no wildcard; if unset in production: no cross-origin browser access at all | Comma-separated list of exact browser origins allowed to call the gateway cross-origin (CORS) and trusted by Better Auth, e.g. `https://admin.example.com,https://partners.example.com`. Unset -> the local dev frontend origins outside production |
| `ALLOW_MEMORY_STORAGE_IN_PRODUCTION` | string | `'false'` | optional |  | Escape hatch for CI's Docker smoke-boot only — it exercises the production config guards (NODE_ENV=production) without real Firestore credentials, by design (see ci.yml's "Smoke-boot the container" step). `render.yaml` never sets this, so the real deploy still |

Other config facts: a missing or invalid variable makes the process **refuse to start** (`GatewayConfigError`, issues listed per variable). `BUILD_SHA` falls back to `RENDER_GIT_COMMIT`. The development defaults are `BETTER_AUTH_SECRET=dev-only-change-me` (rejected in production), the local frontend origins `http://localhost:3000`, `3001`, `3002`, and the default Firestore project id `c1rcle-staging`. Memory storage in production is refused unless the CI-only escape hatch is set.


## 2. Backend: other names read from the environment
- **Tests and tooling only:** `REDIS_TEST_URL`, `FIRESTORE_EMULATOR_HOST`, `C1RCLE_FRONTEND_PATH` (contract export and parity), `C1RCLE_DEBUG`, `C1RCLE_REPO_ROOT`.
- **Not in `apps/api-gateway/.env.example` although the config reads them:** `ALLOWED_ORIGINS`, `BETTER_AUTH_TRUSTED_ORIGINS`, `TRUSTED_PROXY_CIDRS`, `PUBLIC_API_URL`, `ENCRYPTION_SALT`, `ALLOW_MEMORY_STORAGE_IN_PRODUCTION`, `APP_VERSION`, `BUILD_SHA`, `RENDER_GIT_COMMIT`, `HOST`. The example documents `CORS_ALLOWED_ORIGINS` but `render.yaml` sets `ALLOWED_ORIGINS`. **Both are unioned** into the CORS list and Better Auth's trusted origins.
- **`render.yaml` secrets (`sync: false`, values live only in the Render dashboard):** `FIRESTORE_PROJECT_ID`, `FIREBASE_CLIENT_EMAIL`, `FIREBASE_PRIVATE_KEY`, `FIREBASE_STORAGE_BUCKET`, `BETTER_AUTH_SECRET`, `ALLOWED_ORIGINS`, `BETTER_AUTH_TRUSTED_ORIGINS`, `NGINX_READINESS_TOKEN`, `MAGIC_TICKET_SECRET`, `ENCRYPTION_KEY`, `REDIS_URL`, `RAZORPAY_KEY_ID`, `RAZORPAY_KEY_SECRET`, `RAZORPAY_WEBHOOK_SECRET`, `RESEND_API_KEY`, `EMAIL_OTP_SECRET`. **Production needs these but they are not declared in `render.yaml`:** `CORS_ALLOWED_ORIGINS` (optional), `PARTNER_DASHBOARD_URL`, `ENCRYPTION_SALT`, and the origin-lock variables below.
- **Sidecar and nginx names** (read by `deploy/docker/*.sh` and `deploy/staging/*`): set in `render.yaml`: `SIDECAR_FASTIFY_PORT` (8081), `NGINX_PROFILE`, `NGINX_SERVER_NAME`, `NGINX_FORWARDED_PROTO`, `NGINX_READINESS_TOKEN`, and the dead `NGINX_READINESS_ALLOWLIST_LINES`. Others: `FASTIFY_UPSTREAM`, `NGINX_HTTP_PORT`, `NGINX_HTTPS_PORT`, `NGINX_TLS_CERTIFICATE`, `NGINX_TLS_CERTIFICATE_KEY`, `NGINX_VALIDATE_ONLY`, `NGINX_TOPOLOGY`, `NGINX_API_SERVER_NAME`, `NGINX_GUEST_SERVER_NAME`, `NGINX_PARTNER_SERVER_NAME`, `NGINX_ADMIN_SERVER_NAME`, `SIDECAR_HEALTH_CHECK_BUDGET_S` (default 90), `NGINX_ADDITIONAL_SERVER_NAMES`, `NGINX_ORIGIN_LOCKED_SERVER_NAMES`, `NGINX_ORIGIN_VERIFY_SECRET` (the last three exist only on staging and in the dashboard, not in `render.yaml`).

## 3. Frontend (`packages/config/src/schema.ts` declares them all; `@c1rcle/config` is the only reader of `process.env`, enforced by lint)
Client (inlined into the browser bundle, never put secrets in them):
| Name | Required | Default | Used by |
|---|---|---|---|
| `NEXT_PUBLIC_API_BASE_URL` | yes (URL) | none | guest, partner, admin; the build fails if missing or invalid |
| `NEXT_PUBLIC_APP_NAME` | yes | none | all three |
| `NEXT_PUBLIC_APP_ID` | yes (`guest`/`partner`/`admin`) | none | namespaces the CSRF cookie |
| `NEXT_PUBLIC_ENVIRONMENT` | yes (`development`/`preview`/`production`) | none | switches Secure cookies, reCAPTCHA bypass (any non-production) |
| `NEXT_PUBLIC_GUEST_PORTAL_URL` | no | `https://thec1rcle.com` | not in any `.env.example`; unused by partner/admin |
| `NEXT_PUBLIC_SENTRY_DSN` | no | none | declared; **no Sentry SDK exists anywhere**, so it does nothing |
| `NEXT_PUBLIC_FIREBASE_API_KEY`, `_AUTH_DOMAIN`, `_PROJECT_ID` | no | none | partner onboarding phone verification |
| `NEXT_PUBLIC_FIREBASE_TEST_PHONE` | no | none | non-production prefill |
Server-only: `NODE_ENV`, `SITE_URL` (guest canonical origin; production default `https://thec1rcle.com`, previews use `VERCEL_URL`), `VERCEL_ENV`, `VERCEL_URL`, `ANALYZE`, `GATEWAY_READINESS_TOKEN` (admin console health BFF; must equal the backend `NGINX_READINESS_TOKEN`). Also `SMOKE_URL` (admin e2e), Turbo `CI`, `TURBO_TOKEN`, `TURBO_TEAM`.
Scanner app: `EXPO_PUBLIC_API_BASE_URL`, `EXPO_PUBLIC_ENV`, `EXPO_PUBLIC_ORGANIZATION_ID` (optional; pins a handset to one venue and takes precedence over discovery). The scanner reads these directly, not through `@c1rcle/config`.

## 4. CI and deployment names
- **Backend GitHub secrets:** `RENDER_API_KEY`, `RENDER_STAGING_SERVICE_ID`, `RENDER_SERVICE_ID` (future, `main`), `NGINX_READINESS_TOKEN`, `FRONTEND_REPO_TOKEN`. **Variables:** `CONTRACT_PARITY_ENABLED`, `FRONTEND_REPO`, `FRONTEND_REF`, `COVERAGE_MIN_LINES`, `COVERAGE_MIN_STATEMENTS`, `COVERAGE_MIN_FUNCTIONS`, `COVERAGE_MIN_BRANCHES`.
- **Frontend:** variables `BACKEND_REPO`, `BACKEND_REF`; secret `BACKEND_REPO_TOKEN`.
- **Staging scripts** (`deploy/staging/*`): `STAGING_BASE_URL`, `STAGING_READINESS_TOKEN`, `STAGING_ACCESS_TOKEN`, `STAGING_COOKIE`, `STAGING_ORGANIZATION_ID`, `STAGING_IF_MATCH`, `STAGING_IDEMPOTENCY_KEY`, `STAGING_AUTH_EMAIL`, `STAGING_AUTH_PASSWORD`, `STAGING_FRONTEND_ORIGIN`, `STAGING_DIRECT_FASTIFY_URL`, `STAGING_NGINX_IMAGE`, `STAGING_PREFLIGHT_DRY_RUN`, `STAGING_PREFLIGHT_OFFLINE`, `STAGING_REQUIRE_*`, `STAGING_FAILURE_CONFIRM`, `STAGING_MUTATION_CONFIRM`, `STAGING_TLS_MODE`, `STAGING_SAFE_*`, and about 25 `STAGING_LOAD_*` names (see `load-baseline.sh`).

## 5. Name mismatches and traps
- `ALLOWED_ORIGINS` vs `CORS_ALLOWED_ORIGINS`: both are valid and unioned; `render.yaml` uses the first; the example file documents the second.
- Partner `.env.example` says `FIREBASE_PROJECT_ID`; the backend variable is `FIRESTORE_PROJECT_ID`.
- `GATEWAY_READINESS_TOKEN` (frontend) must equal `NGINX_READINESS_TOKEN` (backend) and the GitHub secret of the same name.
- `FIRESTORE_PROJECT_ID` default `c1rcle-staging` vs `.firebaserc` default `c1rcle-v2`.
- `NODE_ENV` must be `production`, not `staging` (the schema accepts only development, test, production).
- `MAGIC_TICKET_SECRET` and `ENCRYPTION_KEY` rotation effects are undocumented in the repo (rotating the first invalidates every QR, manifest and ticket key at once; losing the second makes newly written bank accounts unreadable, while legacy-key records stay readable).
