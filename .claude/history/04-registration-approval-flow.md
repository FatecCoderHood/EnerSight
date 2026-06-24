# 04-registration-approval-flow.md

- **Date:** 2026-06-23
- **Status:** Done
- **Component:** backend/enersight-auth, frontend

## Context
[[03-frontend-auth-integration]] wired real login but left registration/admin-approval
(`CadastroPage.vue`, `UsersPage.vue`) on mock data, deferred twice already. This work wires both to
`enersight-auth`. Tracing the actual code revealed `api/users.api.ts` + `service/mockData.ts` are
entirely dead (zero importers) — the real, live mock system is `UsersMock.ts`'s `MockUsers`, used
directly by `CadastroPage.vue`/`UsersPage.vue`/`MinhaConta.vue`.

This also surfaced a genuine schema/feature conflict: the LGPD-minimal `users` table
(`id, email, password, roles` — no `name`/`phone`, no soft-delete) collides with the existing UI's
full feature set (name/phone columns and edit fields, a "soft delete, data retained for audit"
action). Resolved via user decisions below rather than silently picked.

## Decisions
- **Drop name/phone entirely** from the real-backend-facing UI (table columns, search, modal
  fields) — not even a derived display name. Email becomes the row identifier/search key
  everywhere `name` was used. The registration form itself is unchanged (still collects name/phone
  for the mock-fallback path and consent UX) — those fields just aren't sent to or stored by the
  real backend.
- **Soft-delete-with-audit-retention becomes mock-fallback-only.** Against the real backend, every
  delete (reject-pending or exclude-approved) is a physical `DELETE`, matching the LGPD erasure rule
  already established. The "Excluir" confirmation copy is corrected — it currently claims data is
  retained, which would become false.
- **Admin direct-create stays**, as a separate admin-only endpoint (`POST /api/auth/users`,
  `approved=true` immediately) — distinct from self-registration (`POST /api/auth/register`, always
  `roles=['USER']`, `approved=false`).
- **Reinstating the third login-rejection path** (`approved` gate) — discussed as a hypothetical
  earlier in this work, now required since "properly use the auth API" for approval demands it.
  `403` for "exists, correct password, not yet approved," distinguishable from `401` for
  unknown/wrong credentials — same precedent as the very first OAuth2 implementation in
  `enersight-api`, before the LGPD pass removed it.
- **`UserController`/`UserService` kept separate from `AuthController`/`AuthService`** — the
  security boundary between public self-service and admin-only management is worth keeping visible
  in the class structure, not just route config.
- **`spring-boot-starter-validation` added** to enersight-auth — `register()` is a system boundary
  (untrusted input); validating password length there via Jakarta Bean Validation rather than
  trusting the frontend's client-side check.
- **`JwtAuthenticationConverter` gap found and fixed**: enersight-auth never had one (no protected,
  role-gated endpoint existed before this). Without it, `hasRole("ADMIN")` would be meaningless.
- **Accepted edge case, not solved further**: an admin authenticated via mock fallback whose fake
  token later hits a now-reachable real backend gets a real `401`, not a silent fallback to mock
  data — same "no fallback on a real rejection" rule as login, not a new gap.

## Plan
1. `V2__add_approval_to_users.sql` — `ALTER TABLE users ADD COLUMN approved BOOLEAN NOT NULL DEFAULT false`
   + explicit `UPDATE ... SET approved = true WHERE email = 'admin@tecsys.com'`.
2. `User` entity — add `approved` field.
3. `AuthService` — `register()` (duplicate-email check, bcrypt, forced `roles=['USER']`,
   `approved=false`); `login()` gets the `403`-vs-`401` branch.
4. `RegisterRequestDto` (`@Email`, `@NotBlank`, `@Size(min=6)` on password).
5. `AuthController` — `POST /api/auth/register`.
6. New `UserController`/`UserService`/DTOs — list, direct-create, update-roles, approve, delete.
7. `SecurityConfig` — add `JwtAuthenticationConverter`; permit register alongside login; gate
   `/api/auth/users/**` with `hasRole("ADMIN")`.
8. Frontend: rewrite `api/users.api.ts` (real client, replacing its dead mock-based body);
   `CadastroPage.vue` submit logic (real-first, mock-fallback unchanged); `UsersPage.vue` (every
   action real-first/mock-fallback, name/phone dropped, delete copy corrected); `auth.usecase.ts`
   (403 → CONTA_PENDENTE branch); `UsersMock.ts` (`create()`'s name/phone become optional).

## Files touched
**Backend (enersight-auth):** `pom.xml` (added `spring-boot-starter-validation`),
`V2__add_approval_to_users.sql` (new), `model/User.java` (`approved` field),
`service/AuthService.java` (`register()`, `login()`'s 403 branch), `service/UserService.java` (new),
`controller/AuthController.java` (`POST /register`), `controller/UserController.java` (new),
`security/SecurityConfig.java` (`JwtAuthenticationConverter` bean, new authorization rules — see
Review), `dto/RegisterRequestDto.java`, `dto/RegisterResponseDto.java`, `dto/UserDto.java`,
`dto/CreateUserRequestDto.java`, `dto/UpdateRolesRequestDto.java` (all new). Tests:
`AuthServiceTest.java` extended (also fixed a latent fixture bug — see Review),
`UserServiceTest.java` (new), `UserControllerTest.java` (new).

**Frontend:** `api/client.ts` (added `patch()`), `api/users.api.ts` (rewritten from dead
mock-only file to a real client), `pages/CadastroPage.vue`, `pages/UsersPage.vue`,
`service/auth.usecase.ts` (403 branch), `service/UsersMock.ts` (`create()`'s name/phone optional),
`service/auth.usecase.test.ts` (extended).

## Testing
Backend (28 tests, all passing): `AuthServiceTest` extended (register success forces
`roles=['USER']`/`approved=false`; register rejects duplicate email with 409; login 403-vs-401
distinction), `UserServiceTest` (list/create-forces-approved-true/approve/delete/updateRoles,
duplicate-email rejection), `UserControllerTest` (real signed JWTs via the actual `JwtService` bean,
proving every `/api/auth/users/**` endpoint — list/create/updateRoles/approve/delete — 403s a
non-admin token and 401s no token at all). Frontend: `auth.usecase.test.ts` extended for the
403-to-`CONTA_PENDENTE` branch (4 tests total, all passing). `users.api.ts` itself got no dedicated
test, matching the existing precedent that `auth.api.ts` (an equally thin `authClient` pass-through)
has none either — the branching logic worth testing lives in the page components, not the client.

## Review
Full review (complexity/duplication/dead-code/naming/tests/backwards-compatibility) plus a live
Docker smoke test against a freshly built `enersight-auth` image (fresh DB volume, both migrations
applied cleanly). The smoke test caught a real, severe bug that no unit/MockMvc test surfaced:

- **`/error` was not in the `permitAll()` list.** Spring Boot's default error handling renders a
  thrown exception's response by internally forwarding to `GET /error`. That forward re-enters the
  *same* `SecurityFilterChain` as a brand-new request. Since `/error` matched none of this config's
  rules, it fell through to `anyRequest().authenticated()` — which an anonymous (unauthenticated)
  security context never satisfies — so the real status/body (409 duplicate-email, 403 unapproved,
  400 validation) was clobbered with a bare 401 + `WWW-Authenticate: Bearer` from the OAuth2
  resource server's entry point. This silently affected the *pre-existing* login-rejection paths too
  (wrong-password, unknown-email were always supposed to be 401, so the bug was invisible there by
  coincidence — same wrong mechanism, same-looking result). Fixed by adding
  `.requestMatchers("/error").permitAll()` as the first authorization rule in `SecurityConfig.java`.
  MockMvc-based tests (the entire existing suite) cannot catch this class of bug — `MockMvc` doesn't
  replicate the real servlet container's error-dispatch forward — so this is now documented here as
  a known blind spot of the unit-test layer; live container verification is the only thing that
  caught it, and remains necessary for any future `SecurityConfig` change.
- Re-verified after the fix: register → 201 (`approved:false`); duplicate email → 409; short
  password → 400; login while pending → 403; admin login → 200; admin list/approve/create/delete →
  all correct; non-admin token on any `/api/auth/users/**` route → 403; no token → 401. Full 28-test
  backend suite re-run clean after the `SecurityConfig` fix (confirming MockMvc's blind spot cuts
  both ways — it also didn't regress anything when the fix went in).
- `UserService.create()` hardcodes `.approved(true)` and `CreateUserRequestDto` has no `approved`
  field at all — confirmed no client input can set a user's approval state through either endpoint
  except the dedicated `/approve` route.
- `SecurityConfig`'s rule order (`/error` permitAll → login/register permitAll → users/** hasRole →
  anyRequest authenticated) is correct: Spring evaluates in declaration order, first match wins, and
  none of the matchers shadow a more-specific one declared after it.
- `UsersPage.vue`'s page-level `usingMock` flag was checked for staleness: it's only ever set inside
  `loadUsers()` (called on mount and after every mutation), so the rendered list and the flag are
  always from the same refresh cycle — the delete-confirmation copy can't show the wrong backend's
  wording for a row that's actually on-screen.
- No leftover references to the dropped `name`/`phone`/`uuid`/`createdAt` fields in either rewritten
  page. Two i18n keys (`users.name`, `users.date`) are now unused dead strings in the locale files —
  left in place (harmless, out of scope; `users.fullName`/`users.phone` are still used by
  `MinhaConta.vue` so those locale keys stay).
- Docker cleanup: removed only the `enersight-auth`/`enersight-auth-db` containers and the
  `docker_enersight_auth_postgres_data` volume created for this verification. Pre-existing
  `enersight-db` (left running, untouched) and other already-exited containers from before this
  session were left exactly as found.

## Follow-ups
- None of `MinhaConta.vue`'s profile view/edit is touched here — it still sources via
  `MockUsers.getByEmail`, same limitation noted in [[03-frontend-auth-integration]].
- A "demo/fallback mode" UI indicator (so an admin knows they're viewing mock data) was considered
  and not built — out of scope, matches login's existing silent-fallback precedent.
- The `/error` permitAll gap is worth keeping in mind for `enersight-api`'s `SecurityConfig` too, if
  one exists — not checked as part of this task since it was out of scope, but the same class of bug
  could be lurking there.
