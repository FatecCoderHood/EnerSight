# 07-auth-hardening-phases-0-1.md

- **Date:** 2026-06-24
- **Status:** Done
- **Component:** backend/enersight-auth, backend/enersight-api, frontend

## Context
Follows directly from the authentication audit conducted in this same conversation (not a separate
history file — pure analysis, no code). The audit's recommended plan had 4 phases; this task
implements Phases 0–1, explicitly scoped to enersight-auth, enersight-api, and the frontend per
instruction — enersight-worker, enersight-temporal-series, and Mongo are out of scope. Phase 2
(machine-to-machine auth for the Python services) is skipped entirely for the same reason. Phase 3
(evaluate adopting a real OAuth2 Authorization Server) remains a future decision point, not
something with a code deliverable today.

## Decisions
- **Shorten the access token to 15 minutes, add a 30-day rotating refresh token.** Refresh tokens
  only earn their complexity if the access token is actually short-lived — otherwise there's no
  real reason to refresh, and revocation stays meaningfully delayed regardless. This is also what
  makes logout/de-provisioning actually take effect promptly, closing the audit's "no token
  revocation" finding.
- **Refresh token rotation, not reuse.** Each successful `/refresh` call revokes the presented
  refresh token and issues a new one. Limits the replay window if a refresh token is ever
  intercepted, for a small amount of extra logic (revoke-old + issue-new in one transaction).
- **`/refresh` and `/logout` are both `permitAll()`, authenticated by the refresh token itself**
  (in the request body), not a Bearer access token. Necessary because the access token may already
  be expired by the time either is called — that's the whole point of `/refresh`, and `/logout`
  shouldn't fail just because the access token happened to expire first.
- **`JWT_SECRET`-default check warns, does not hard-fail.** Both services currently boot with the
  literal default (docker-compose.yaml doesn't override it) and this app has no Spring profile
  configuration anywhere — hard-failing would break this repo's own current default setup. A loud
  startup `WARN` is the safe middle ground.
- **`logout()` on the frontend always clears local storage, even if the server revoke call fails.**
  Matches this codebase's existing philosophy (seen in the mock-fallback patterns throughout) of
  never letting a network hiccup strand the user in a stuck state — best-effort revoke, but the
  client-side "you're logged out" outcome is unconditional.
- **Mock sessions short-circuit out of the refresh flow entirely.** A `fake-jwt-token-*` session has
  no real refresh token; `refreshAccessToken()` fails fast for them rather than attempting a
  network call that was never going to succeed.
- **TDD scope**: full backend coverage (this is genuinely new security-sensitive logic — token
  rotation, revocation, the two startup checks). Frontend stays manual-verification-only, the same
  constraint noted in every prior frontend-only task in this codebase — and the riskiest new
  frontend logic (the interceptor's retry-once behavior) isn't a pure function that's testable
  without mounting real HTTP behavior anyway.

## Plan
1. **enersight-api** — `SecurityConfig`: add `.requestMatchers("/error").permitAll()`.
2. **Both Java services** — new small `@Component` (`JwtSecretWarningCheck` or similar) with a
   `@PostConstruct`/startup hook that logs `WARN` if the resolved `jwt.secret` equals the known
   default literal.
3. **enersight-auth** — migration `V4__add_refresh_tokens_table.sql`; new `RefreshToken` entity +
   repository; `jwt.expiration-minutes` default drops to `15`, new
   `jwt.refresh-expiration-days=30`.
4. **enersight-auth** — `AuthService.login()` also issues + persists a refresh token (random
   opaque value, stored hashed); `LoginResponseDto` gains a `refreshToken` field. New
   `RefreshTokenService`: `issue(user)`, `rotate(rawToken)` (validate not expired/revoked → revoke
   old → issue + persist new pair), `revoke(rawToken)`.
5. **enersight-auth** — `AuthController`: `POST /api/auth/refresh` (body `{refreshToken}` →
   new `{token, refreshToken, user}`), `POST /api/auth/logout` (body `{refreshToken}` → 204).
   `SecurityConfig`: both added to the existing `permitAll()` matcher list alongside login/register.
6. **Frontend** — `auth.api.ts`: `login()`'s response type gains `refreshToken`; new `refresh()`/
   `logout()` methods. `auth.usecase.ts`: persist the refresh token alongside the access token;
   new `refreshAccessToken()` (fails fast for mock sessions); `logout()` becomes `async`,
   best-effort server revoke then unconditional local clear. `client.ts`: shared response
   interceptor catches a 401, attempts exactly one refresh + retry (flagged on the request config
   to prevent loops), forces logout + redirects to `/login` if the refresh itself fails too.

## Files touched
**enersight-api:** `security/SecurityConfig.java`, new `security/JwtSecretWarningCheck.java` (or
equivalent).

**enersight-auth:** `V4__add_refresh_tokens_table.sql` (new), `model/RefreshToken.java` (new),
`repository/RefreshTokenRepository.java` (new), `service/RefreshTokenService.java` (new),
`service/AuthService.java`, `dto/LoginResponseDto.java`, `dto/RefreshRequestDto.java` (new),
`controller/AuthController.java`, `security/SecurityConfig.java`, `application.properties`, new
startup-check component (same as enersight-api's).

**Backend tests:** `RefreshTokenServiceTest.java` (new), `AuthServiceTest.java` (extended),
`AuthControllerTest.java` (extended for `/refresh` and `/logout`).

**Frontend:** `api/auth.api.ts`, `service/auth.usecase.ts`, `api/client.ts`.

## Testing
Backend: new unit tests for `RefreshTokenService` (issue/rotate-success/rotate-expired/
rotate-revoked/revoke), extended `AuthServiceTest` (login issues a refresh token),
`AuthControllerTest` extended for the two new endpoints. Frontend: manual verification against the
live Docker stack — login, wait/force-expire an access token, confirm a protected call transparently
refreshes and retries, confirm logout revokes the refresh token server-side (a second refresh
attempt with the same token must fail).

## Review findings (fixed before completion)
- **TOCTOU race in `AuthService.refresh()`**: the original implementation read the refresh token's
  state via `RefreshTokenService.validate()`, then separately wrote `revoked=true` via `revoke()` —
  two concurrent requests replaying the same not-yet-rotated token could both pass validation before
  either write landed, both minting a new session from one token. Fixed by adding
  `RefreshTokenRepository.revokeIfActive()` (a conditional `UPDATE ... WHERE revoked = false`,
  `@Modifying`/`@Transactional`) and having `AuthService.refresh()` reject with 401 if that
  conditional update affects zero rows — the database, not application code, now serializes who wins
  a concurrent replay. Caught the missing `@Transactional` (Spring Data `@Modifying` queries require
  an active transaction) only via live Docker verification, not the mocked unit tests — confirmed
  live with 5 truly concurrent `/refresh` calls on the same token: exactly 1 succeeded, 4 got 401,
  both before (failed loudly, 500s) and after (passed cleanly) the `@Transactional` fix.
- Reviewed and found sound, no changes needed: the `/auth/*` exemption-list matching in `client.ts`'s
  interceptor (checked against every real endpoint path called via `apiClient`/`authClient`, no
  false positives/negatives); the dynamic-import approach for breaking the real
  `client.ts ↔ auth.usecase.ts` and `client.ts ↔ routes/index.ts` circular dependencies (confirmed
  both cycles are real, not hypothetical); SHA-256-not-BCrypt for refresh token hashing (correct
  call — BCrypt's per-call random salt makes exact-match hash lookup impossible, and its slow/salted
  design solves a different problem than this one: a 256-bit CSPRNG token has no brute-force/rainbow
  table exposure the way a human-chosen password does); the `permitAll()` choice for `/refresh` and
  `/logout` (the raw token in the body is correctly the credential, no Bearer requirement needed).

## Follow-ups
- Phase 2 (machine-to-machine auth for enersight-worker/enersight-temporal-series) and Phase 3
  (evaluate a real OAuth2 Authorization Server) remain open, out of scope here by instruction.
