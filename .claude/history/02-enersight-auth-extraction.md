# 02-enersight-auth-extraction.md

- **Date:** 2026-06-23
- **Status:** Done
- **Component:** backend/enersight-auth (new), backend/enersight-api, backend/docker

## Context
[[01-oauth2-enersight-api]] put the full OAuth2 stack (login, JWT issuance, user storage) inside
`enersight-api`. This work extracts that into a dedicated `enersight-auth` microservice (Identity
Provider) and turns `enersight-api` into a strict Resource Server that only validates tokens
against a shared HMAC secret — no DB user lookups, no password checks, no `User` entity.

`enersight-api` is not in `docker-compose.yaml` today (it runs via `mvn spring-boot:run`), so
compose changes are additive only: a new `enersight-auth` service + its own dedicated Postgres.

## Decisions
- **Dedicated Postgres container** (`enersight-auth-db`) for enersight-auth, not a shared schema in
  `enersight-db` — true DB-per-service boundary, no host port published, own credentials. Chosen by
  the user over the shared-schema alternative for stronger LGPD isolation.
- **LGPD data minimization**: `users` table is `id (UUID), email, password (bcrypt), roles (text[])`
  — no `name`, no separate `uuid` column (`id` *is* the UUID), and **no `active`/status flag**.
  Deactivation = a real SQL `DELETE` of the row (right to erasure), not a soft-delete flag. This
  was an explicit correction during plan review: an `active` column was initially proposed and
  rejected by the user for exactly this reason.
- **Single DB role** (`auth_user`) for both Flyway migration and runtime access in
  `enersight-auth-db` — simpler than enersight-api's `flyway_user`/`app_user` split, since the real
  isolation boundary here is the dedicated container, not intra-DB role separation.
- **No `mvnw` wrapper** for enersight-auth — the wrapper in enersight-api was broken (missing
  `.mvn/wrapper/maven-wrapper.properties`) and unused; system `mvn` and the official `maven` Docker
  image cover both local dev and the container build.
- **Breaking response contract** (accepted, not fixed now): login response drops `uuid`/`name`/
  singular `role`, becomes `{ token, user: { id, email, roles } }`. Frontend integration was
  already out of scope per [[01-oauth2-enersight-api]]'s follow-ups.
- **Package `com.enersight.auth`** for the new module (vs. enersight-api's `com.enersight`) to keep
  the two deployables visually distinct despite sharing no classpath.

## Plan
1. Scaffold `backend/enersight-auth` Maven module (pom.xml mirrors enersight-api: Boot 4.0.6
   parent, Java 17, Lombok, Flyway, postgresql, security, oauth2-resource-server).
2. `V1__create_users_table.sql` — `id UUID PK DEFAULT gen_random_uuid()`, `email UNIQUE`,
   `password`, `roles TEXT[]` + one seeded bcrypt admin row.
3. `model/User.java`, `repository/UserRepository.java` (`findByEmail`).
4. `security/JwtService.java` (encode: `sub`=id, claims email/roles) + `security/SecurityConfig.java`
   (encoder, decoder, passwordEncoder beans; `permitAll` on `POST /api/auth/login`, else
   `authenticated()`).
5. `dto/LoginRequestDto.java`/`LoginResponseDto.java`, `service/AuthService.java` (two rejection
   paths only: unknown/deleted email, wrong password — no inactive-account branch),
   `controller/AuthController.java`.
6. Multi-stage `Dockerfile` (maven builder → jre-alpine runtime) + `.dockerignore`.
7. `enersight-api` deletions: `model/User.java`, `repository/UserRepository.java`,
   `service/AuthService.java`, `controller/AuthController.java`,
   `db/migration/V5__create_users_table.sql`, `security/JwtService.java` (dead once issuance moves
   out), plus their tests.
8. `enersight-api` `security/SecurityConfig.java` simplification: drop `JwtEncoder`/`PasswordEncoder`
   beans, drop the login `permitAll` matcher → bare `anyRequest().authenticated()`, update
   `JwtAuthenticationConverter` claim name `"role"` → `"roles"`.
9. `enersight-api` `application.properties`: drop `jwt.expiration-minutes`, keep `jwt.secret`.
10. `enersight-api` `SecurityConfigIntegrationTest.java`: stop referencing `AuthController`; build
    its test JWT with a locally-constructed Nimbus encoder (no `JwtEncoder` bean left in this
    module) to prove decode-only validation works against a token from anyone holding the secret.
11. `docker-compose.yaml`: add `enersight-auth-db` (no published port) and `enersight-auth`
    (port 8082, `depends_on: enersight-auth-db: condition: service_healthy`, `JWT_SECRET` env var
    shared with wherever enersight-api runs).

## Files touched
**New (enersight-auth):** `pom.xml`, `EnersightAuthApplication.java`, `model/User.java`,
`repository/UserRepository.java`, `security/JwtService.java`, `security/SecurityConfig.java`,
`dto/LoginRequestDto.java`, `dto/LoginResponseDto.java`, `service/AuthService.java`,
`controller/AuthController.java`, `application.properties`,
`db/migration/V1__create_users_table.sql`, `Dockerfile`, `.dockerignore`, plus test classes.

**Deleted (enersight-api):** `model/User.java`, `repository/UserRepository.java`,
`service/AuthService.java`, `controller/AuthController.java`,
`db/migration/V5__create_users_table.sql`, `security/JwtService.java`, `JwtServiceTest.java`,
`AuthServiceTest.java`, `AuthControllerTest.java`.

**Modified (enersight-api):** `security/SecurityConfig.java`, `application.properties`,
`security/SecurityConfigIntegrationTest.java`.

**Modified:** `backend/docker/docker-compose.yaml`.

## Testing
- enersight-auth: `JwtServiceTest` (round-trip with UUID `sub` + `roles` array claim, expiry,
  tamper), `AuthServiceTest` (correct creds issue token; wrong password rejected; unknown/deleted
  email rejected — same code path, no information leak), `AuthControllerTest` (`@WebMvcTest` +
  `MockitoBean`: 200/401/400).
- enersight-api: `SecurityConfigIntegrationTest` adapted as above.

## Follow-ups
- Physical user-deletion endpoint (`DELETE /api/users/{id}` or similar) is out of scope for this
  migration — the schema only enforces that erasure must be a real SQL `DELETE`, not a flag flip,
  whenever that admin capability gets built.
- Frontend integration with the new `{ token, user: { id, email, roles } }` response shape —
  separate task, same as noted in [[01-oauth2-enersight-api]].
- User registration/admin-approval workflow — still out of scope, still frontend-mocked.

## Review notes

**Verified live, not just unit-tested**: built the actual Docker image, ran `enersight-auth-db` +
`enersight-auth` via `docker compose up`, confirmed Flyway applied `V1` and Hibernate validated the
`TEXT[]` ↔ `String[]` array mapping against real Postgres (this can't be caught by `ddl-auto=validate`
without a live DB), then hit `POST /api/auth/login` for real: correct credentials return a properly
shaped JWT (`sub`=UUID, `roles`=["ADMIN"], `email` claim); wrong password and unknown email both
return 401 with no distinguishing signal. Containers/volume/image torn down afterward; pre-existing
containers (`enersight-db`, `enersight-mongo`, etc.) confirmed untouched.

**Fixed during review**: two test fixtures (`JwtServiceTest`, `AuthServiceTest`) used a lowercase
`"admin"` role, inconsistent with the seed migration's uppercase `ARRAY['ADMIN']` convention. Not a
live bug today (no `hasRole()` checks exist yet anywhere), but `JwtAuthenticationConverter` in
enersight-api applies the `ROLE_` prefix with no case transform, so role casing at rest will matter
the moment any authorization rule is added. Fixed fixtures to use `"ADMIN"` to match the real
convention.

**Security analysis — key management:**
- The HS256 symmetric secret is, by construction, both the signing key and the verification key.
  Anyone holding `enersight-api`'s copy of `JWT_SECRET` can *forge* tokens, not just validate them —
  this is the structural tradeoff of the symmetric-HMAC constraint specified for this task. An
  asymmetric scheme (RS256: enersight-auth holds the private key, enersight-api only gets the public
  key) would mean leaking enersight-api's key material — the more externally-exposed service —
  couldn't be used to mint new tokens. Worth a future iteration; not a regression introduced here,
  since HS256 was an explicit requirement, not a default I chose.
- No rotation story: a single static secret means rotating it invalidates every outstanding token on
  both services simultaneously (no `kid` claim, no decoder that accepts an old+new key during
  overlap). Acceptable given short-lived tokens (60 min default) but worth flagging for production.
- **Distribution asymmetry**: `enersight-auth` gets `JWT_SECRET` from `docker-compose.yaml`, but
  `enersight-api` isn't containerized at all — it runs via `mvn spring-boot:run` outside Docker. So
  there's no single source of truth enforcing the same value on both sides; today they happen to
  match only because both `application.properties` files hardcode the *identical* dev-only literal
  as their fallback default. The moment either side's `JWT_SECRET` is set without the other, every
  token validation silently fails (generic invalid-signature 401, no diagnostic pointing at "secret
  mismatch"). A real deployment needs one secret store (Vault, Docker secrets, etc.) feeding both.

**Security analysis — container initialization order:**
- `enersight-auth-db`'s healthcheck (`pg_isready`) is safe to depend on: the official `postgres`
  image runs its `POSTGRES_DB`/`POSTGRES_USER` provisioning synchronously during initdb, before the
  server process that `pg_isready` checks even starts — confirmed by the live run above (Flyway
  connected and migrated on the very first attempt, no retry needed).
- `enersight-auth` declares `depends_on: enersight-auth-db: condition: service_healthy`, but Spring
  Boot itself has no connection-retry/backoff configured beyond Hikari's defaults. Under cold-start
  resource contention (e.g., `docker compose up` starting many services at once) a first-connection
  failure would crash the JVM, relying solely on `restart: unless-stopped` to retry the whole
  container. This is the same risk profile already accepted by `enersight-worker`/
  `enersight-job-producer` against `enersight-db` elsewhere in this compose file — not a new defect,
  but worth naming since "initialization order" was asked about directly.
- **Genuine resilience strength**: because validation is purely local (shared secret, no network
  call), `enersight-api` has *no runtime dependency on enersight-auth being up at all*. It keeps
  validating already-issued tokens even if enersight-auth is down, crashed, or being redeployed —
  the opposite of a typical "auth service is a single point of failure" coupling.
