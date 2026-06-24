# 01-oauth2-enersight-api.md

- **Date:** 2026-06-22
- **Status:** Done
- **Component:** backend/enersight-api

## Context
enersight-api has no security at all today — a single open endpoint (`GET /api/geo`). The frontend
already assumes a real backend login: `auth.api.ts` posts `{ email, password }` to `/auth/login`
and expects `{ token, user }` back, then sends `Authorization: Bearer <token>` on every request
(`client.ts`). Today this is faked client-side via `auth.usecase.ts` (`fake-jwt-token-${id}` against
`UsersMock.ts`). This work replaces that mock with a real OAuth2-style bearer-token flow on the
backend.

No external identity provider (Keycloak/Auth0/Google) is referenced anywhere in the repo, so a
full Authorization Server is unnecessary.

## Decisions
- **enersight-api is its own OAuth2 Resource Server and token issuer** — no external IdP. Matches
  what the frontend already expects and avoids standing up infrastructure not needed for one
  internal API.
- **HMAC (symmetric secret) JWT signing**, not RSA — single-instance internal API, no need for
  asymmetric key distribution/rotation.
- **Seed one bootstrap admin user** via the new Flyway migration (mirrors the frontend's existing
  mock admin, `admin@tecsys.com`) so login is testable end-to-end without building full user
  registration/approval (out of scope — see Follow-ups).
- **Default-deny security**: every endpoint requires a valid bearer token except
  `POST /api/auth/login`. `GET /api/geo` becomes protected with no code changes to that
  controller.
- **First JPA entity in this codebase** for `User`/`core.users` (existing `SsdmtRepository` uses
  raw native SQL via `EntityManager`); a keyed user lookup fits Spring Data JPA better than
  hand-written SQL, so this introduces the pattern without changing the existing repository.

## Plan
1. `pom.xml` — add `spring-boot-starter-security`, `spring-boot-starter-oauth2-resource-server`,
   test-scope `spring-security-test`.
2. `V5__create_users_table.sql` — `core.users` (id, uuid, name, email unique, password_hash, role,
   status, created_at) + one bcrypt-hashed seed admin row.
3. `model/User.java` (JPA entity) + `repository/UserRepository.java` (`findByEmail`).
4. `security/JwtService.java` (HMAC encode/decode: `sub`=uuid, claims email/role/exp) +
   `security/SecurityConfig.java` (stateless, CSRF disabled, `permitAll` on login,
   `authenticated()` elsewhere, `JwtAuthenticationConverter` mapping `role` claim →
   `ROLE_ADMIN`/`ROLE_USER`, `PasswordEncoder` bean).
5. `dto/LoginRequestDto.java`, `dto/LoginResponseDto.java`, `service/AuthService.java`
   (validate credentials + `status == approved`, issue token), `controller/AuthController.java`
   (`POST /api/auth/login`).
6. No changes needed to `SsdmtController`/`SsdmtService`/`SsdmtRepository` — they inherit
   protection from the new default-deny `SecurityConfig`.

## Files touched
- `pom.xml`
- `src/main/resources/application.properties`
- `src/main/resources/db/migration/V5__create_users_table.sql`
- `src/main/java/com/enersight/model/User.java`
- `src/main/java/com/enersight/repository/UserRepository.java`
- `src/main/java/com/enersight/security/JwtService.java`
- `src/main/java/com/enersight/security/SecurityConfig.java`
- `src/main/java/com/enersight/dto/LoginRequestDto.java`
- `src/main/java/com/enersight/dto/LoginResponseDto.java`
- `src/main/java/com/enersight/service/AuthService.java`
- `src/main/java/com/enersight/controller/AuthController.java`

## Testing
- `JwtServiceTest` — encode/decode round-trip, expired token rejected, tampered signature
  rejected.
- `AuthServiceTest` — correct credentials issue a token; wrong password, unknown email, and
  non-`approved` status are all rejected.
- `AuthControllerTest` (`@WebMvcTest`, mocked `AuthService`) — 200 on valid body, 401 on bad
  credentials, 400 on malformed request.
- `SecurityConfigIntegrationTest` (`@SpringBootTest` + `MockMvc`) — `/api/geo` returns 401 without
  a token and 200 with a valid one; `/api/auth/login` is reachable without a token.

## Follow-ups
- User registration / admin-approval workflow (`CadastroPage`, `UsersPage` admin actions) — stays
  frontend-mocked; not part of this OAuth2 change.
- Token refresh and logout/blacklisting — not implemented; tokens simply expire.
- Frontend integration (pointing `auth.usecase.ts` at the real endpoint instead of `UsersMock.ts`)
  — separate task.

## Review notes
- Caught during `/review`: the seed admin's `uuid` literal (`ad-550e8400-...`, copied from the
  frontend mock's 2-char-prefixed format) was 39 characters against a `VARCHAR(36)` column — would
  have failed the migration on a fresh database. Fixed by using a standard 36-char UUID for the
  seed row instead of replicating the frontend mock's prefix convention.
- Local Postgres (`enersight-db` container) has no port published to the host and hadn't run any
  migrations yet, so true `@SpringBootTest` + real-DB integration tests weren't possible in this
  environment. `SecurityConfigIntegrationTest` was implemented as `@WebMvcTest({SsdmtController,
  AuthController}) + @Import(SecurityConfig, JwtService)` instead — exercises the real security
  filter chain and JWT encode/decode beans with `SsdmtService`/`AuthService` mocked, so no
  datasource is ever started.
- Spring Boot 4 renamed/moved some test infra used here: `@WebMvcTest` now lives in
  `org.springframework.boot.webmvc.test.autoconfigure`, and `@MockBean` is replaced by
  `org.springframework.test.context.bean.override.mockito.MockitoBean`.
