# ADR 0004: CI and quality tooling (GitHub Actions, SonarCloud, Dependabot)

- Status: Proposed (workflows not yet run on GitHub)
- Context: Only a frontend deploy workflow existed (branch `mistery`). No PR validation, no lint, no coverage.
- Decision:
  - One `ci.yml` per repo (`6_back`, `6_front`) on pull requests to `develop`/`release` and pushes to `develop`.
  - Backend: `mvn verify` (Testcontainers integration tests) with JaCoCo; the report feeds SonarCloud.
  - Frontend: ESLint (flat config), `vite build` in real and mock modes, type-check (non-blocking, see gap 7), SonarCloud scan. No coverage: no test runner yet.
  - Dependabot weekly for Maven, npm, pip (worker, temporal-series) and GitHub Actions.
  - Secrets/config: `SONAR_TOKEN` (secret), `SONAR_ORGANIZATION`, `SONAR_PROJECT_KEY_API`, `SONAR_PROJECT_KEY_WEB` (repository variables). Sonar steps are skipped when the token is absent.
- Consequences:
  - Legacy lint findings (`any`, `v-html`, unused vars) are warnings; tighten to errors as cleaned up.
  - Making checks required needs branch protection (repo settings, not code).
  - The Azure deploy workflow is untouched.
