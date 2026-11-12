# Authentication, registration and login — current state

**Verified against:** 2026-10-30 — branch `ES-67/update-back-with-new-front` in both modules, working tree on top of backend `a49cc51` and frontend `9c763fc` (changes uncommitted at the time of writing; update the commits after the user commits).
**How verified:** backend `mvn test` (31 unit/web-layer tests plus the Testcontainers integration test `AuthFlowIntegrationTest`), real curl run against the restarted API on the dev DB, frontend `vite build` in mock and real modes. Frontend UI flows were not exercised in a browser.
**Owners to notify when this changes:** backend and frontend

## Summary
Users have a `role` (`USER`, `ADMIN`) and a lifecycle `status` (`PENDING`, `ACTIVE`, `REJECTED`, `ANONYMIZED`). Registration creates a `PENDING` user; an admin approves or rejects (reason required). A user may anonymize their own account; admins may anonymize any account. Anonymization keeps the row, frees the email, and the person can register again as a new account. Authentication events and anonymizations go to `audit_log`; self-requests also create a `data_subject_requests` row.

## Role / permission matrix (enforced by the backend)
| Endpoint | Public | USER | ADMIN |
|---|---|---|---|
| `POST /api/auth/login`, `POST /api/users/register`, `GET /api/terms`, `GET /api/terms/{id}` | yes | yes | yes |
| `GET /api/users/{id}`, `/terms-history` | no | own only | any |
| `DELETE /api/users/me` | no | yes | yes |
| `GET /api/users?status=`, `PATCH /{id}/approve-access`, `PATCH /{id}/reject`, `DELETE /{id}` (anonymize) | no | 403 | yes |
| Terms writes (`/api/terms/**` non-GET) | no | 403 | yes |
| Everything else (e.g. `/api/geo`) | no (401) | yes | yes |

Frontend: USER sees Home and Map; ADMIN also sees Dashboard, Users, Logs and Reserve (sidebar and route guard use `user.role`).
Login: `PENDING`, `REJECTED`, `ANONYMIZED` or inactive users get 403 and no token. No or invalid token on a protected route gets 401.

## State by piece
| Piece | Side | Status | Evidence |
|---|---|---|---|
| Login, JWT | back | Fixed in ES-67 | `AuthService` audits success/failure; pending 403 (was 500); `JwtAuthenticationFilter` ignores invalid/expired/stale tokens |
| Register | back | Fixed in ES-67 | `UserService.register` creates USER/PENDING; audited |
| List, approve, reject, anonymize | back | Fixed in ES-67 | `UserController`, `UserService` role checks; `AuthWebLayerTest`, `UserServiceTest` |
| Self anonymize | back | Fixed in ES-67 | `DELETE /api/users/me`; `data_subject_requests` row |
| Route security | back | Fixed in ES-67 | `SecurityConfig` permits only the public routes; 401 entry point; terms writes ADMIN |
| Schema | back | Fixed in ES-67 | V13 (status, rejection_reason), V14 (audit_log, data_subject_requests), V15 (restores `fk_uta_user`) |
| Mock/real switch | front | Fixed in ES-67 | `VITE_USE_MOCKS`, `src/api/*` (see ADR 0002) |
| Registration page | front | Fixed in ES-67 | fetches terms, sends `acceptedTermIds`, no phone |
| Role guard, sidebar | front | Fixed in ES-67 | ES-66 guard kept (`authUsecase.getUserRole()`, lowercase `admin`/`user`); `authUsecase.login` lowercases the API `role`, keeps `PENDING` accounts out (403) |
| 401 handling | front | Fixed in ES-67 | `client.ts` response interceptor clears `auth_token`/`user_data` and redirects (login 401 excluded) |
| Users page | front | Fixed in ES-67 | ES-66 UI kept; real calls via `usersApi`, reject asks a reason (`ConfirmModal` `reasonLabel`), delete = anonymize; no silent mock fallback; phone column removed |
| Delete account | front | Fixed in ES-67 | `MinhaConta.vue` calls `DELETE /users/me`, then clears auth. Page still shows hard-coded profile data (`João Silva`) |
| Audit log read | both | Fixed in ES-67 | back: admin-only `GET /api/audit-logs` (AuditService.list, names resolved at read time, so anonymized actors show Anonymized); front: logs.api.ts (real) or LogsMock (mock mode only). logout() no longer writes real names into LogsMock |
| Confirm modal | front | Fixed in ES-67 | approve/delete buttons were disabled (:disabled=" is true in Vue); now !!reasonLabel && ... |
| My account page | front | Fixed in ES-67 | profile loaded from usersApi.getById (session fallback); MockUsers only adds phone/consent |
| Login page | front | Fixed in ES-67 | 403 shows the backend message (pending/rejected/anonymized); 401 shows an error |

## Open gaps (backlog)
Next up (HIGH): gap 6, then the rest in order.
1. **[back] Seeded admin `admin@admin.com` with password `1234`** (V9) violates RNF01. Decision: leave as known gap. Target: env-provisioned admin and forced password change on first login. The integration test uses this account.
2. **[back] Hard-coded secrets in `application.properties`** (`jwt.secret`, DB passwords). Target: environment variables.
3. **[both] Admin create/edit user**, **email change (RF01)**, **role management**: no endpoints; the Users page no longer offers create/edit (ES-66 mock-only feature removed in the ES-67 merge until endpoints exist).
4. **[both] Re-registration after REJECTED** is not defined (a rejected email stays taken).
5. **[back] Deactivated (`active=false`) user's existing token** is still accepted until it expires.
6. **[both] `MinhaConta.vue` self-service profile (user CRUD). Priority: HIGH, next task after the auth cycle closes.** Personal-info edit, password change and consent changes are not wired to any endpoint (the backend has none) and the save buttons show a fake success toast. Needs: backend endpoints (update own name, change password with current-password check, consent update recorded in `user_terms_acceptance`, plus admin create/edit from gap 3), TDD, front wiring behind the mock switch, no fake success. Phone/consent currently come from the local `MockUsers`.
7. **[front]** Pre-existing `vue-tsc` errors outside auth files; no test runner.
8. **[back]** Admin-initiated anonymization writes only `audit_log` (no data-subject request).

## Unverified
- Why V12 dropped `fk_uta_user` (probably hard-delete failures). V15 restores it after deleting orphan acceptances (rows whose user no longer exists).
- Whether Copilot auto-loads the submodule `.github` instruction files.
- Frontend flows in a browser (only `vue-tsc`, builds in both modes and API curl flows were run). Mock accounts: `admin@tecsys.com`, `user@tecsys.com`, `pending@tecsys.com`, password `1234`.
- ES-66 was merged into this branch by hand (hybrid): ES-66 UI + `src/api` switch. Not exercised in a browser.`n- Logs page against the real API was checked by curl only (not in a browser). Stale browser-side entries remain in `localStorage` key `enersigh_logs` from earlier sessions; they are not shown in real mode.
- Dev DB still holds test users created by the curl run (`flow*@example.test`, and anonymized rows); `fk_dsr_user` blocks deleting them, which is expected.

## Related
- ADRs: `docs/adr/0001-anonymize-instead-of-delete.md`, `0002-mock-real-api-switch.md`, `0003-user-status-separate-from-role.md`.
- Instruction files that point here: `backend/.github/copilot-instructions.md`, `frontend/.github/copilot-instructions.md`, workspace `.github/copilot-instructions.md`.
