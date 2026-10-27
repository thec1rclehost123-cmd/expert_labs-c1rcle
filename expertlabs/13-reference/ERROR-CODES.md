# Error codes and the error envelope

Authority in the repo: `docs/api-contracts/error-contract.md`, decisions D-004 and D-009, `packages/contracts/src/contracts/shared.ts`, `apps/api-gateway/src/plugins/error-handler.ts`, `packages/core/src/domain/errors.ts`.

## Wire envelope (flat, identical everywhere)
`{ status, code, message, requestId, fieldErrors?, details? }`. Success bodies are bare DTOs (lists `{items, pageInfo}`). **5xx messages are always redacted** to "Internal server error". The response also carries `x-request-id` (the gateway echoes it; it accepts an incoming value only from a trusted proxy peer and only when it matches `^[A-Za-z0-9][A-Za-z0-9._:-]{0,127}$`, otherwise it mints a UUID).

## `ApiErrorCode` (12 values)
`validation | unauthorized | forbidden | not_found | conflict | rate_limited | network | timeout | aborted | parse | unknown | server`
| Status | Code | Emitted by |
|---|---|---|
| 400, 422 | `validation` | zod failures (`validate-v2`, 422 with `fieldErrors`), `InvalidOperationError` (400) |
| 401 | `unauthorized` | no session, deactivated admin, invalid scanner token, `UnauthorizedError` |
| 403 | `forbidden` | missing permission, role too low for a tier, path-org mismatch, geofence, unbound device, `passwordChangeRequired` (with `details`) |
| 404 | `not_found` | the NotFound family; cross-tenant reads (deliberately 404, not 403); unknown routes |
| 409 | `conflict` | `VersionConflict` (details: expected and current version), `StateTransition`, `Conflict`, idempotency conflict, in-flight and **failed** attempts, `payment_refunded` (with `details.outcome = "refunded"`) |
| 429 | `rate_limited` | `plugins/rate-limit.ts` (with `Retry-After`), stream budgets in `phase5-routes.ts`, nginx edge (`edge_rate_limited`) |
| 5xx | `server` | unhandled errors, `ServiceUnavailableError` (503, e.g. payments not configured, payouts gate off, webhook secret unset) |
`network`, `timeout`, `aborted`, `parse` and `unknown` are produced **client-side only** (`FE/packages/api-client/src/errors.ts`); the gateway never sends them. A `parse` error means a response failed the frontend's zod schema (frontend and backend out of sync).
The edge adds its own codes (`edge_request_too_large`, `edge_rate_limited`, `edge_upstream_unavailable`, `edge_upstream_timeout`, `bff_upstream_*` in the unused full-edge variant), all with `requestId`.

## Internal `DomainError.code` strings (logs only, not on the wire)
`organization_not_found`, `venue_not_found`, `slot_request_not_found`, `event_not_found`, `ticket_tier_not_found`, `promoter_assignment_not_found`, `partnership_not_found`, `onboarding_request_not_found`, `proposal_not_found`, `notification_not_found`, `not_found`, `forbidden`, `device_not_authorized` (answered 403), `unauthorized`, version conflict, state transition, `payment_refunded`, `invalid_operation`, `conflict`, `service_unavailable`, `provider_rejected`, idempotency conflict / in-flight / failed. They appear in logs as `domain_error` and `unhandled_domain_error`. Mapping is in `mapDomainError` (`routes/v2/partner/events.ts`); `door-sale-routes.ts`, `cover-wallet-routes.ts` and `finance-routes.ts` carry their own copies that do not handle every code (door and wallet copies miss `device_not_authorized`, `conflict`, `payment_refunded`; the finance copy misses `state_transition` and `conflict`).

## Contract-level error codes that are never emitted
`phase5ErrorCodeSchema` values such as `SCANNER_SESSION_EXPIRED` and `VELOCITY_LIMIT_EXCEEDED` exist in the contract but the gateway uses the generic codes above.

## Consequences for integrators
The machine-readable taxonomy is coarse: nuance lives in `message` and `details`. Specific cases to key off: `409` with `details.outcome = "refunded"` (payment auto-refunded), `403` with `details.passwordChangeRequired`, `409` idempotency conflicts versus version conflicts (distinguish by `details`), `429` with `Retry-After`. The frontend `describeApiError`/`describeAdminError` helpers show `fieldErrors` for status below 500 only; many console pages ignore the gateway message entirely.
