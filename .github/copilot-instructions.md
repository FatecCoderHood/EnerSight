# EnerSight — workspace instructions

Shared rules for the whole workspace. Each submodule has its own `.github/copilot-instructions.md` with module-specific rules; read it before editing inside that module. If a module file and this file disagree, the module file wins for code inside that module.

## What this is
EnerSight is an analytical web platform for TECSYS that collects public ANEEL data and lets users analyse network continuity indicators (DEC, FEC, their regulatory limits) by distributor, state, electrical group and substation, with ranking and heatmap views, user management and logging. It is a FATEC São José dos Campos student project (team "CoderHood"). The root `README.md` is the source of truth for requirements (RF01–RF10, RNF01–RNF16), sprints, Definition of Done/Ready and the branch/commit standard — read it instead of guessing business rules.

## Workspace layout
| Path | What | Own git repo |
|---|---|---|
| `backend/` (submodule `6_back`) | `enersight-api` (Java/Spring), `enersight-worker` (Python ingestion), `enersight-temporal-series` (Python/Prophet), `docker/` (compose) | yes |
| `frontend/` (submodule `6_front`) | Vue 3 + TypeScript + Vite app | yes |
| `docs/` | Technical docs, roadmaps | parent |
| `.github/` | These instructions and shared skills | parent |

`backend/` and `frontend/` are separate repositories: commit and push inside the submodule first, then commit the updated submodule pointer in the parent. Never run a module's tooling against another module.

## Business rules to respect (from the root README)
- New users are created as `PENDING`; only `ACTIVE`/approved users may use the platform; rejection requires a justification (RF01, RF02). The API's role enum is `USER`, `ADMIN`, `PENDING`.
- Terms of use, privacy notice and optional marketing consent are versioned and stored separately (RF03). Personal data requests/anonymization must be supported (RF04, RNF03, RNF07).
- Logs must never contain passwords, tokens, secrets, cookies or unnecessary personal payloads (RNF05).
- No secret (including the initial admin credential) may live in source code, versioned scripts or docs (RNF01).
- Analytical views use only public, aggregated ANEEL data, never personal data (RNF13, RNF16).
- Imports must be traceable to source/batch and idempotent: no duplication on reprocessing (RF07).

## Branching, commits and PRs (root README standard)
- Branches: `feature/*` flow into `develop`; `release` is the stable production line; hotfixes branch from `release` and are merged into both `release` and `develop`.
- Feature branch name: `<TASK-ID>/<lowercase-hyphenated-description>`, e.g. `ES-01/login-screen`. Create from an up-to-date `develop`.
- Commit message: `<TASK-ID>: <type>(<scope>) <short lowercase description>`, e.g. `ES-01 feat: add login endpoint`. Types: feat, fix, chore, docs, style, refactor, perf, test, build, ci, revert, hotfix. Small, frequent commits; one task per commit, e.g. `ES-67: fix(SonarCloud) exclude lombok generated code from coverage`.
- PRs target `develop` and need two approvals. Releases happen at the end of each sprint; unstable work stays out of a release.
- Do not commit or push unless the user asks.

## Language
Code, identifiers, commit messages and workspace docs in English. Keep existing Portuguese where it already exists (ANEEL domain terms, the temporal-series docs, frontend page names and `pt` locale) and don't translate files unprompted.

## Safety rules for every module
- Never add real secrets, tokens, passwords or personal data to code, config, docs, tests or logs. Use environment variables or placeholders.
- Don't commit `.venv`, `node_modules`, `target/`, build outputs or large data files.
- Don't edit an already-applied Flyway migration; add a new one.

## Project state and known issues (do not "fix" silently; mention and ask)
Existing code shows conventions in use, not correctness. Details are in each module's file; the cross-cutting ones are:
- **Secrets:** `jwt.secret`, DB credentials, compose passwords and Python default passwords are literals in tracked files (violates RNF01); the seeded admin has a documented known password.
- **Access control:** the API permits all GET and POST requests and has role checks only in one service method (conflicts with RF01/RF02/RNF02); the frontend gates roles client-side only.
- **Frontend is partly mocked:** login calls the real API but registration and the users page do not match its contract; logs and forecast use mocks (details in `docs/project-memory/authentication.md`).
- **Ingestion is incomplete:** the worker downloads and extracts but its load step is disabled; there is no validation, batch traceability or scheduler (RF06/RF07).
- **Missing requirements:** terms history, consent revocation, anonymization, audit logs (RF03–RF05, RF08).
- **Docs drift:** the root README claims Java 21+, Spring Boot 3+, Swagger and Vuetify; the code uses Java 17, Spring Boot 4.0.6, no OpenAPI and no Vuetify.
- **Tooling:** no tests beyond `contextLoads`, and no linter or formatter in any module.
- **Local setup:** the Maven wrapper is broken (no `.mvn/`, `mvnw` not executable; use system `mvn`), WSL has no `node` on PATH, and the frontend submodule remote is HTTPS while `.gitmodules` says SSH.
- **Run check (2026-10-30):** the stack starts cleanly (PostGIS, API on 8080, Vite on 5173 with working `/api` proxy, 0 API errors).

## Skills (workspace `.github/skills/`)
`dev-workflow` (Plan → Implement → Test → Doc → Deliver), `kiss`, `clean-code`, `testing`, `code-documentation`, `project-memory`. Use `dev-workflow` for any feature or fix.

**Project memory:** verified findings live in `docs/project-memory/` (index in its `README.md`; decisions in `docs/adr/`). Read the note for your area before working, and update it in the same change when you fix or touch something it records. Current notes: `authentication.md` (backend and frontend). Skills are stack-neutral; commands and conventions come from the module's own instructions.

## How to work here
1. Identify the module, then read its `.github/copilot-instructions.md` and the existing code around the change.
2. Prefer the simplest change that satisfies the requirement; follow the conventions of the file you edit, not conventions from other projects.
3. Verify with the module's own commands and report exactly what you ran. If something can't be verified, say so.
