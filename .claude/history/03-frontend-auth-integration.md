# 03-frontend-auth-integration.md

- **Date:** 2026-06-23
- **Status:** Done
- **Component:** frontend

## Context
[[02-enersight-auth-extraction]] moved real login to a dedicated `enersight-auth` service, but the
frontend's actual login flow (`LoginPage.vue` → `auth.usecase.ts`) has never called a real backend —
it's a pure client-side mock (`MockUsers.authenticate`). A second file, `api/auth.api.ts`, was shaped
to call a real `/auth/login` endpoint but is dead code (nothing imports it). This work wires the real
call to enersight-auth into the actual login flow, with the existing mock kept as a fallback for when
the auth service is unreachable.

## Decisions
- **Real bug found and fixed**: `api/client.ts`'s axios interceptor reads `localStorage.getItem('auth')`
  (a single JSON blob with `.token`), but `auth.usecase.ts.persist()` writes to two separate keys
  (`auth_token`, `user_data`). The keys never matched, so no outgoing request has ever actually
  carried a Bearer token — `/api/geo` (JWT-protected since [[01-oauth2-enersight-api]]) has been
  silently 401-able this whole time. Fixed by reading the actual persisted key.
- **Separate Vite proxy + base URL for enersight-auth**: `/auth-api` → `http://localhost:8082`
  (rewritten to `/api`), distinct from the existing `/api` → `:8080` (enersight-api). Avoids any
  proxy-matching ambiguity between two real backend targets.
- **`ApiClient` parameterized** (accepts a `baseURL`) so a second instance (`authClient`) can point
  at enersight-auth while `apiClient` keeps pointing at enersight-api — reuse, not duplication.
- **Response normalization in `auth.usecase.ts`**: enersight-auth returns
  `{ token, user: { id (UUID), email, roles[] } }` — no `name`, no singular `role`. Normalized to the
  shape every existing consumer already expects: `uuid = id`, `name` derived from the email's
  local-part, `role = roles[0].toLowerCase()`. No changes needed to `MinhaConta.vue`, `UsersPage.vue`,
  or the route guards.
- **Fallback only on unreachability, not on a real rejection**: network error (no `error.response`)
  → fall back to `MockUsers.authenticate`. A real 401 from a reachable enersight-auth is a genuine
  credential rejection and must surface as a login failure, not silently retry against mock data —
  otherwise a wrong real password could be bypassed by a coincidentally-matching mock account.
- **"CONTA_PENDENTE" stays mock-exclusive** — enersight-auth has no pending-approval concept (an
  explicit follow-up deferred earlier); real-backend rejections always show the generic
  "Credenciais inválidas".
- **Accepted limitation**: mock-fallback tokens (`fake-jwt-token-N`) are not real JWTs, so `/api/geo`
  will correctly 401 against the real enersight-api when running on the fallback path. Mock fallback
  restores login/navigation continuity, not access to real protected data.
- **Out of scope, unchanged**: `users.api.ts`, `UsersPage.vue`, `CadastroPage.vue`,
  `MinhaConta.vue`'s profile-edit-via-mock flow, `AgentePrevisao.vue`/`PrevisaoMock.ts` — none of
  these make any real backend call today; registration/admin-approval stays deferred.
- **Vitest added** as the frontend's first test runner (none existed) — chosen over skipping
  automated tests, per explicit user confirmation, to actually exercise `auth.usecase.ts`'s branching
  logic rather than relying on live-only verification.

## Plan
1. `vite.config.ts` — add `/auth-api` proxy entry (→ `:8082`, rewritten to `/api`).
2. `src/config/api.config.ts` — add `AUTH_API_CONFIG.BASE_URL` (`VITE_AUTH_API_BASE_URL` override,
   default `/auth-api`).
3. `src/api/client.ts` — fix the interceptor's storage key; parameterize `ApiClient` with a
   `baseURL` constructor arg; export `authClient` alongside the existing `apiClient`.
4. `src/api/auth.api.ts` — use `authClient`; correct request/response typing to the real
   enersight-auth contract.
5. `src/service/auth.usecase.ts` — `login()` tries `authApi.login()` first, normalizes the response;
   catches network-unreachable errors and falls back to the existing mock logic (extracted into a
   private helper, behavior unchanged); a real 401 throws `'Credenciais inválidas'` directly.
6. Add Vitest + minimal config; unit tests for the three `login()` branches.

## Files touched
`vite.config.ts`, `src/config/api.config.ts`, `src/api/client.ts`, `src/api/auth.api.ts`,
`src/service/auth.usecase.ts`, new `vitest.config.ts` (or reuse `vite.config.ts`'s test block),
new `src/service/auth.usecase.test.ts`, `package.json` (new devDependency + test script).

## Testing
Vitest unit tests for `auth.usecase.ts.login()`: real success → normalized shape; real 401 → throws
`'Credenciais inválidas'`, no fallback attempted; network-unreachable → falls back to mock path
unchanged. `authApi`/`authClient` mocked via `vi.mock`, no real network calls in tests. Plus a live
check: real login against the running `enersight-auth` container, then stop it and confirm the
fallback to mock login still works.

## Follow-ups
- User registration/admin-approval workflow, and `UsersPage.vue`/`CadastroPage.vue`'s mock-CRUD —
  still out of scope, still mock-backed.
- No real audit-log backend exists; `LogsMock.registerAction` keeps being called for both real and
  mock logins for UI consistency, but isn't a real audit trail.
- `MinhaConta.vue` (untouched) loads the full profile via `MockUsers.getByEmail(currentUser.email)`
  — see Review notes below.

## Review notes

**Verified live**: brought `enersight-auth` + its DB back up, started the Vite dev server, and hit
the new `/auth-api` proxy directly: real admin credentials returned a real JWT, a wrong password
returned 401 — confirming the new proxy/base-URL plumbing reaches the right service end-to-end.
Containers, volume, and image torn down afterward. (Noticed `mongo-express`/`enersight-mongo`/
`enersight-temporal-series-api` were already `Exited` with mixed signals from before this session's
verification — not caused by any command here, since none of them were ever targeted; flagged to
the user rather than restarted unprompted.)

**Fallback-bypass check (the one decision with real security weight)**: confirmed axios omits
`error.response` entirely on network-level failures and populates it on any HTTP error response.
`if (!err.response)` is therefore a correct, idiomatic discriminator — a real 401 from a reachable
enersight-auth cannot fall through to the mock branch. Verified by a unit test that spies on
`MockUsers.authenticate` and asserts it's never called on a 401.

**Consequence found, not a bug introduced**: `MinhaConta.vue` (out of scope, untouched) has always
sourced full profile data via `MockUsers.getByEmail(currentUser.email)`, regardless of how the
current user authenticated. For the seeded admin this coincidentally works (same email used in both
the real seed and the mock seed). For any other real-backend user whose email isn't in the static
mock dataset, the page's `if (fullUser)` guard prevents a crash, but profile fields stay at their
default/blank values. This limitation predates this change and is now reachable via a new path
(real login) — not fixed here, since full profile-management against a real backend is the same
out-of-scope registration/admin-management work already deferred twice.

**Build hygiene**: confirmed precisely — 27 pre-existing TypeScript errors existed before this task
across files never touched here (`App.vue`, `MapView.vue`, `AgentePrevisao.vue`, `UsersPage.vue`,
`mockData.ts`, etc.). One of those 27 was in `auth.usecase.ts` (`MockUser` unused import) — fixed,
since it sat in the exact file being rewritten, not a drive-by fix of unrelated code. 26 remain,
confirmed identical before/after in every file outside this task's scope.

**Known simplification**: any HTTP error response from a reachable enersight-auth (not just 401)
throws the same `'Credenciais inválidas'` message — e.g. a 500 would misleadingly read as "invalid
credentials" rather than "server error." Matches the existing mock path's message vocabulary (which
never had a separate "server error" case either), so no UX regression — just not a new improvement
beyond what was asked.
