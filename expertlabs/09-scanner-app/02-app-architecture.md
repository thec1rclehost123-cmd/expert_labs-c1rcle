# 02. App architecture (frontend)

## Stack and versions (`apps/scanner-app/package.json`)
Expo `~55.0.0`, React Native `0.83.10`, React `19.2.0`, expo-router `~55.0.18`, react-native-web `~0.21.0`, zod `^4.2.0`, TypeScript 5.9.3. Native modules: expo-camera, expo-location, expo-secure-store, expo-haptics, expo-network, expo-crypto, expo-blur, expo-linear-gradient, expo-linking, expo-constants, expo-font, expo-status-bar, react-native-svg, react-native-safe-area-context, @react-navigation/bottom-tabs; `react-native-screens` is **pinned `~4.23.0`** (do not bump). Workspace deps: `@c1rcle/api-client`, `@c1rcle/contracts`. Fonts Anton and Archivo. Install packages only with `npx expo install` (CI runs `expo install --check`).

## Layout
| Path | Purpose |
|---|---|
| `app/_layout.tsx` | root layout, font loading, the auth redirect guard |
| `app/{index,login,pairing,redeem,profile,wallet}.tsx` | boot spinner, sign-in, pairing, shift start (door code), profile/settings, cover tab |
| `app/(tabs)/{scan,door,guests,stats,settings}.tsx` + `_layout.tsx` | the five tabs, floating tab bar, heartbeat, toast host |
| `src/api/scannerApiClient.ts` (476 lines), `schemas.ts` | every gateway call; schemas are an adapter over `@c1rcle/contracts` (three local schemas remain) |
| `src/auth/{staffAuth,scannerSession,deviceIdentity,venueBinding,authState}.ts` | staff token (memory), shift session (secure storage), device id, venue binding, the auth state machine |
| `src/storage/secureStorage.ts` | the single persistence boundary (SecureStore on native, Web Storage on web) |
| `src/config/env.ts` | zod-validated env from `Constants.expoConfig.extra` |
| `src/features/{scan,settings,toast,heartbeat}` | result sheet, couple-confirm hook, settings panel and in-memory preferences, toasts, heartbeat |
| `src/components`, `src/theme`, `src/web` | UI parts, tokens, a web-only autofill style hack |
| `ui_example/claude_design_ui/Circle Scanner.html` | the design reference every screen was transcribed from |

## Config
`app.config.ts`: name "C1RCLE Scanner", slug `c1rcle-scanner`, scheme `c1rclescanner`, version 0.1.0, portrait, dark. iOS: `supportsTablet: true`, camera and when-in-use location usage strings, **no bundleIdentifier**. Android: `CAMERA`, `ACCESS_COARSE_LOCATION`, `ACCESS_FINE_LOCATION`, **no package id**. Web: Metro, `output: 'single'`. Plugins: expo-router, expo-secure-store, expo-font, expo-camera, expo-location. `extra`: `apiBaseUrl` (`EXPO_PUBLIC_API_BASE_URL ?? 'http://localhost:8080'`), `appEnv` (`EXPO_PUBLIC_ENV ?? 'staging'`), `organizationId` (`EXPO_PUBLIC_ORGANIZATION_ID ?? null`).
**No `eas.json`, no EAS project id, no `updates`/runtimeVersion, no `owner`: no native build or store pipeline exists.** CI only runs `expo export` and `expo install --check`. `env.ts` throws on invalid config, but the `localhost:8080` fallback in `app.config.ts` means a production build with the variable missing would silently point at localhost.
Env names: `EXPO_PUBLIC_API_BASE_URL`, `EXPO_PUBLIC_ENV` (`staging|production`), `EXPO_PUBLIC_ORGANIZATION_ID` (optional; pins a dedicated handset build to one venue; login errors if the account is not a member). The app reads them directly, not via `@c1rcle/config` (exception to the single-owner env rule). `.env` is gitignored; `.env.example` carries the staging backend URL (non-secret).
Metro: pnpm-monorepo recipe (watch workspace root, `nodeModulesPaths`). Web target builds and runs with shims (localStorage/sessionStorage fallback, silent location fallback, autofill CSS) and is a preview target only.

## Routing and the auth state machine
`useScannerAuthState()` (`src/auth/authState.ts`) returns one of five states:
| State | Meaning | Redirect target |
|---|---|---|
| `checking` | initial | none |
| `logged_out` | no active staff token | `/login` |
| `needs_pairing` | staff active, no device id + name stored | `/pairing` |
| `needs_redeem` | paired, shift session missing or expired | `/redeem` |
| `active_session` | shift open | `/(tabs)/scan` |
`app/_layout.tsx` redirects only when the current path is not already satisfied (`SATISFIED_PREFIXES`): `/profile` is allowed in the three authenticated states; `/scan`, `/door`, `/stats`, `/guests`, `/settings`, `/wallet` in `active_session`; `/redeem` deliberately **not** in `active_session`. Screens never call `router.replace` after an auth change: they call `notifyAuthStateChanged()` (module-level pub-sub) and the root layout re-checks. The state re-checks every **60 s**, which is how clock expiry of the staff token or the 12 h session is noticed. `FONT_MAP` is module-level to avoid a blink/remount bug (do not move it inline).
**Cold start:** the staff token is in memory only, so every app restart returns to `/login`, even when the device pairing and a still-valid shift token survive.

## Session, device and storage
- `staffAuth.ts`: in-memory `{accessToken, organizationId, user{id,email,displayName,role}, expiresAt}`; never persisted. The `ApiClient` is created with `getToken` only (no `reauth`/`onUnauthorized`), so a 401 is surfaced by screens, not refreshed. A 401 means `isAuthFailure`; only `scan.tsx` reacts (toast + notify); other screens show an error, and the 60 s watchdog covers the rest.
- `deviceIdentity.ts`: `scanner_<32 hex>` from `Crypto.getRandomBytes(16)`; keys `c1rcle_scanner_device_id`, `c1rcle_scanner_device_name` (device scope). `clearDevicePairing` exists (for a 403 "device unbound") but is **never called**; no screen handles that 403.
- `scannerSession.ts`: stores `c1rcle_scanner_session_token` and `c1rcle_scanner_session_meta` (session scope). `isSessionExpired()` compares `sessionExpiresAt` with the **device clock**. `getSessionMeta` does an unvalidated `JSON.parse(...) as SessionMeta`. No refresh and no server revoke.
- `venueBinding.ts`: resolution order: build pin → on-device binding (`c1rcle_scanner_organization_id`) → discovery via `GET /organizations` (one membership auto-binds, several prompt, none errors). A stale binding is replaced silently when a different account signs in; the build pin is never overridden.
- `secureStorage.ts`: native = Keychain/Keystore for everything; web = `device` scope in localStorage (device id, name, org binding) and `session` scope in sessionStorage (scanner token and meta). The web target cannot meet the secure-storage requirement and is a disclosed deviation.

## API client (`src/api/scannerApiClient.ts`, wraps `@c1rcle/api-client`)
Base URL from env; lazily created singleton. Headers per call: `x-organization-id` and, on door calls, `x-scanner-session-token`. `fetchTodaysEvents`, `registerDevice`, `redeemDoorCode` send the org header only; `openStatsStream` sends the org header only (correct, see `07-…`).
- Default timeout **60 s** (`DEFAULT_TIMEOUT_MS`), so a scan on a bad network can wait up to a minute before the offline card shows. GETs retry once with backoff and `Retry-After` (≤30 s); writes never retry. No scan-specific timeout.
- **Idempotency keys are sent in the JSON body** (`idempotencyKey`) for walk-in, dine-in, ticket-sale and wallet-charge, **not** as an `Idempotency-Key` header. `check-in`, `confirm`, `override`, `staff-deny` and manual check-in carry no key. The offline sync design uses a header, so the conventions are inconsistent.
- Request ids and door/wallet idempotency keys use `crypto.randomUUID()`. Whether Hermes provides the global on real devices is **unverified** (the web target has it); `expo-crypto` is already a dependency.
- Function map and schemas: `09-backend-api-reference.md`.

## Settings and preferences (mostly decorative today)
`SettingsPanel` (shared by `/profile` and the Settings tab): staff card; "assigned gate" segmented control (local state only, never persisted or sent); toggles `soundOn`, `hapticOn`, `autoAdmit`, `continuousScanning` in `scannerPreferences.ts` (**in-memory, reset on restart**); only `autoAdmit` is read (`scan.tsx`); haptics always fire; there is no audio library. "Offline mode" is a disabled row. "Switch event" clears the session (→ `needs_redeem`); "Log out" clears session and staff session; **neither calls the server**. Footer hard-codes "THE C1RCLE SCANNER - v2.4" (package version is 0.1.0).
