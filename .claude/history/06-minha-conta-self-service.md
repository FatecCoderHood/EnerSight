# 06-minha-conta-self-service.md

- **Date:** 2026-06-23
- **Status:** Done
- **Component:** backend/enersight-auth, frontend

## Context
`MinhaConta.vue` ("minha-conta") always sourced its own profile data via `MockUsers.getByEmail(...)`
regardless of which backend authenticated the session — a gap flagged but explicitly deferred in
[[03-frontend-auth-integration]], [[04-registration-approval-flow]], and [[05-post-test-bugfixes]].
This task closes it: enersight-auth has zero self-service endpoints today (only public
login/register and admin-only `/users/**`), so "Informações Pessoais," "Segurança" (password
change), and "Revogar Consentimento" (self account deletion) all need a new authenticated
self-service surface, plus optional name/phone fields that don't exist on the real backend yet.

## Decisions
- **Email becomes read-only in Informações Pessoais.** It's the login identifier; self-editing it
  would need uniqueness re-checks and risks lockout with no verification flow. Name/phone become
  genuinely editable/persisted; email just displays.
- **Consent (communications checkbox, terms/privacy) stays frontend/mock-only.** Not promoted to a
  real backend column — low-stakes preference, not worth the schema/endpoint surface right now.
- **Admin User Management screen is untouched.** Even though `name`/`phone` will exist as real
  columns after this task, task 4's removal of those columns from the admin screen stays as-is;
  revisiting that is a separate, explicit decision if ever wanted.
- **New `MeController`/`MeService`, not folded into `AuthController`/`UserController`.** Mirrors the
  task-5 precedent of keeping security boundaries visible in class structure: public self-service
  (login/register), admin-only management (`/users/**`), and now authenticated self-service (`/me`)
  are three distinct categories, three distinct controllers.
- **`/api/auth/me/**` needs no new `SecurityConfig` rule** — it doesn't match the admin-only
  `/api/auth/users/**` pattern, so it already falls under the existing `anyRequest().authenticated()`
  catch-all (any valid role).
- **Password change actually verifies the current password now**, on both the new real endpoint and
  the corrected mock path — the existing mock implementation never checked it at all, a latent
  correctness gap independent of the real-backend wiring.
- **`isMockSession()` added to `auth.usecase.ts`** rather than reusing the existing
  "unreachable → fallback" heuristic. A session that logged in via mock fallback carries a
  `fake-jwt-token-*` token; if the real backend later comes back up, sending that fake token to it
  would get a genuine 401 (invalid token) — which the existing heuristic would treat as "real
  backend rejected this," not "this session needs mock data." Checking the token shape up front
  avoids that whole class of confusion.
- **TDD scope matches precedent**: full backend unit/controller tests (new logic, same pattern as
  every prior backend task); frontend stays manual-verification-only, same constraint noted in
  [[05-post-test-bugfixes]] (component logic isn't unit-testable without introducing new test infra,
  which would be disproportionate for this change).

## Plan
1. **Migration `V3__add_profile_fields_to_users.sql`** — nullable `name`/`phone` columns on `users`.
2. **`User` entity** — add nullable `name`/`phone` fields.
3. **`RegisterRequestDto`** — add optional `name`/`phone` (no `@NotBlank`). **`AuthService.register()`**
   stores them when present.
4. **New DTOs**: `MeResponseDto` (id, email, name, phone, roles, approved), `UpdateProfileRequestDto`
   (name, phone — both optional, `null` means "leave unchanged"), `ChangePasswordRequestDto`
   (currentPassword, newPassword — `@Size(min=6)` on the new one).
5. **New `MeService`** — resolves the current user from the JWT's `sub` claim
   (`@AuthenticationPrincipal Jwt`), then: `getProfile()`, `updateProfile(...)`,
   `changePassword(...)` (verifies `currentPassword` via `passwordEncoder.matches`, throws 401 if
   wrong — same semantic as login's wrong-password case), `deleteSelf()`.
6. **New `MeController`** at `/api/auth/me`: `GET`, `PATCH`, `PATCH /password`, `DELETE`.
7. **`LoginResponseDto.UserSummary`** — add `name` field; `AuthService.login()` populates it from the
   stored value (may be `null`).
8. **Frontend — `CadastroPage.vue`**: re-add the phone input (no `required`), drop `required` from
   the name input; validation only blocks on missing email/password. `registerWithFallback`/
   `registerWithMock` pass name/phone through.
9. **Frontend — `users.api.ts`**: `register()` takes optional `name`/`phone`; add `getMe()`,
   `updateMe(name, phone)`, `changePassword(current, next)`, `deleteMe()`.
10. **Frontend — `auth.usecase.ts`**: add `isMockSession()`; `login()`'s normalization prefers the
    real `name` from the response, falling back to the derived-from-email helper when absent.
11. **Frontend — `MinhaConta.vue`**: branch every action (`loadUserData`, `saveUserInfo`,
    `changePassword`, `confirmRevokeConsent`) on `isMockSession()` — real path calls the new
    `usersApi.*` methods (falling back to mock only on genuine unreachability, consistent with
    every other real/mock call site); email input becomes `disabled`; `changePassword` verifies the
    current password on both paths before proceeding; role display reuses the same
    case-insensitive fix pattern as [[05-post-test-bugfixes]]; consent tab/`saveConsent()` unchanged.

## Files touched
**Backend:** `V3__add_profile_fields_to_users.sql` (new), `model/User.java`,
`dto/RegisterRequestDto.java`, `service/AuthService.java`, `dto/LoginResponseDto.java`,
`dto/MeResponseDto.java` (new), `dto/UpdateProfileRequestDto.java` (new),
`dto/ChangePasswordRequestDto.java` (new), `service/MeService.java` (new),
`controller/MeController.java` (new).

**Backend tests:** `AuthServiceTest.java` (extended for name/phone on register),
`MeServiceTest.java` (new), `MeControllerTest.java` (new).

**Frontend:** `pages/CadastroPage.vue`, `api/users.api.ts`, `service/auth.usecase.ts`,
`pages/MinhaConta.vue`.

## Testing
Backend (43 tests, up from 28): `MeServiceTest` (6, Mockito + a real `Jwt` built via
`Jwt.withTokenValue(...).build()`), `MeControllerTest` (7, `WebMvcTest` + real signed JWTs via
`JwtService` — proving a plain `USER` token, not just `ADMIN`, gets through `/me` since it's for
everyone), `AuthServiceTest` (+3: null name/phone doesn't break register, name/phone stored when
given, login response includes the stored name). Adding fields to `RegisterRequestDto` broke two
existing tests' positional-arg `@AllArgsConstructor` calls (arity changed from 2 to 4) — fixed by
adding `@Builder` and switching those call sites to it, which is also more robust against the next
field addition. Frontend: manual only, against a live Docker `enersight-auth` — register with
name/phone → admin-approve → login (response now includes the real name) → `GET`/`PATCH /me` →
wrong-password change rejected 401 → correct change 204 → old password fails, new one works →
`DELETE /me` → subsequent login 401 (account genuinely gone) → confirmed the seed admin's own
`/me` shows `name: null, phone: null` correctly (pre-existing row, migration didn't backfill) →
confirmed the admin `/users` list endpoint is completely unaffected (still only
id/email/roles/approved). This ran against the user's own already-running dev server + Docker
container (from their manual testing session) — rebuilt the `enersight-auth` image in place and
recreated the container, preserving the database volume; V3 applied cleanly on top of existing
rows. Could not visually verify the UI itself (no browser automation tool available); Vite HMR
should have applied the `MinhaConta.vue`/`CadastroPage.vue` changes live to that running session.

## Review
- **Found and fixed a real bug**: `saveConsent()` was left "untouched" per the consent-stays-mock
  decision, but it unconditionally called `MockUsers.update(Number(user.id), {...})`. For a
  real-backend session, `user.id` is a UUID string — `Number(uuidString)` is `NaN`, and
  `MockUsers.update`'s `findIndex(u => u.id === NaN)` never matches (`NaN === NaN` is always
  `false` in JS), so the call silently no-ops. The success toast fired anyway, falsely claiming the
  preference was saved. Fixed by gating the `MockUsers.update` call on `isMockSession()`, matching
  every other branch in this file — a real session's communications preference is now honestly
  session-local (resets on reload, since there's genuinely nowhere real to persist it) instead of
  silently failing while claiming success.
- Confirmed the "don't fall back to mock on real-backend failure" deviation (intentional, unlike
  every other real/mock call site in this codebase) is now applied consistently across all of
  `loadUserData`, `saveUserInfo`, `changePassword`, and `confirmRevokeConsent` — the reasoning holds
  throughout: mock data has no record of a real-backend-only user, so falling back would either
  silently no-op (as the consent bug above did) or, worse, operate on the wrong account.
  `confirmRevokeConsent` especially must not fall back here — silently "succeeding" by deleting an
  unrelated mock account while the user believes their real one is gone would be the worst version
  of this class of bug.
- `MeService.updateProfile`'s partial-update semantics (`null` = leave unchanged) can't strand a
  field as uncleerable: the frontend always sends both fields as strings (possibly `""`), never
  `null` — JSON `""` deserializes to Java `""`, which is `!= null`, so clearing a field via the UI
  does take effect. `null` only arises if a future caller omits the field entirely, which the
  current frontend never does.
- `userInitials` and `isAdminRole(user.role)` both already degrade sensibly when `name`/`role` are
  empty (existing `if (!user.name) return '?'` guard; `isAdminRole`'s optional chaining), so a
  real-backend user who never set a name doesn't break the header.
- Checked whether `isMockSession()` could answer wrong with no token at all (it returns `false`,
  i.e. "treat as real," which would hit `/me` with no `Authorization` header) — not reachable in
  practice: `/minha-conta` has `requiresAuth: true` in the router, and the global `beforeEach` guard
  redirects to `/login` before this component ever mounts without a token.
- Backend `/api/auth/me` correctly requires no new `SecurityConfig` rule and is reachable by any
  authenticated role (verified: a plain `USER` token succeeds on every method, not just `ADMIN`).
  Admin's `/users` list endpoint is unaffected by the new `name`/`phone` columns — confirmed via
  live curl that its response shape didn't change.

## Follow-ups
- None identified beyond what's already fixed above.
