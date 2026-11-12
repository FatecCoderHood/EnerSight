# CI pipeline guide

How the automated checks work in each EnerSight repo. Decision record: [ADR 0004](adr/0004-ci-and-quality-tooling.md).

## 1. The big picture

Every pull request to `develop` or `release` (and every push to `develop`) runs a **GitHub Actions** workflow. It builds the code, runs the checks, and sends the results to **SonarCloud** for static analysis. **Dependabot** separately opens PRs to update dependencies.

```
 you push / open PR
        |
        v
 GitHub Actions (ci.yml)
   |-- install + build
   |-- lint / tests / coverage
   '-- SonarCloud scan  --->  SonarCloud (quality gate, issues, coverage)
        |
        v
 PR shows green/red check  --->  branch protection can block merge on red

 Dependabot (weekly) --> opens "bump X" PRs --> they run the same CI
```

| Piece | Where it lives | Who runs it |
|---|---|---|
| CI workflow | `.github/workflows/ci.yml` in each repo | GitHub Actions |
| Dependency updates | `.github/dependabot.yml` in each repo | GitHub Dependabot |
| Static analysis | SonarCloud (config in workflow + `sonar-project.properties` in frontend) | Scanner in the workflow |
| Merge blocking | Repo **Settings > Branches** (not code) | GitHub |

## 2. Triggers (both repos)

```yaml
on:
  pull_request:
    types: [opened, synchronize, reopened]
    branches: [develop, release]
  push:
    branches: [develop]
```

- A PR targeting `develop` or `release` runs CI when opened, updated (new push) or reopened.
- A direct push or merge into `develop` runs CI too.
- `concurrency` cancels an older run of the same branch when a newer push arrives, so you don't wait for stale runs.
- `permissions: contents: read` gives the workflow read-only access to the repo (least privilege).

The Azure deploy workflows are separate and untouched by this pipeline.

## 3. Backend (`6_back`): job `enersight-api (test, coverage, sonar)`

Runs on `ubuntu-latest`, inside the `enersight-api` folder.

| # | Step | What it does | Fails when |
|---|---|---|---|
| 1 | Checkout (`fetch-depth: 0`) | Gets the full git history. Sonar needs it to know which code is "new". | - |
| 2 | Setup Java 17 (Temurin) + Maven cache | Same Java as `pom.xml`. Caches `~/.m2` to speed up runs. | - |
| 3 | Cache SonarCloud packages | Caches `~/.sonar/cache`. | - |
| 4 | `mvn -B verify` | Compiles, runs **all tests**, then generates the **JaCoCo coverage report** (`target/site/jacoco/jacoco.xml`). | Compile error or any failing test |
| 5 | SonarCloud analysis | `mvn ... sonar-maven-plugin:sonar` with org and project key from repo variables. Uploads code and coverage. | Bad token, wrong keys, or quality gate setup issue |
| 6 | Upload test reports (only on failure) | Saves `target/surefire-reports` as a downloadable artifact. | - |

Key facts:
- **Tests need no database set up by you.** The integration test (`AuthFlowIntegrationTest`) starts its own PostGIS with **Testcontainers**, using Docker, which `ubuntu-latest` provides. Any test that expects a database at `localhost:5432` will pass on your machine (dev DB running) but fail in CI. That's why the old `EnersightApplicationTests` was removed.
- **JaCoCo** is configured in `enersight-api/pom.xml` (`prepare-agent` before tests, `report` at `verify`). The path is given to Sonar by the property `sonar.coverage.jacoco.xmlReportPaths`.
- Step 5 runs only if `SONAR_TOKEN` exists (`if: env.SONAR_TOKEN != ''`). Pull requests from forks don't receive secrets, so Sonar is skipped there instead of failing.

Run the same thing locally (from `backend/enersight-api`):
```
mvn -B verify
```

## 4. Frontend (`6_front`): job `frontend (lint, build, sonar)`

Runs on `ubuntu-latest`, at the repo root.

| # | Step | What it does | Fails when |
|---|---|---|---|
| 1 | Checkout (`fetch-depth: 0`) | Full history for Sonar. | - |
| 2 | Setup Node 24 + npm cache | Matches the local Node version. | - |
| 3 | `npm ci` | Clean, exact install from `package-lock.json`. | Lock file out of sync |
| 4 | `npm run lint` | ESLint (flat config in `eslint.config.js`). | Any lint **error** (warnings do not fail) |
| 5 | Type-check `npx vue-tsc -b` | TypeScript check. **`continue-on-error: true`**: it reports problems but does not fail the job. | Never blocks. It currently has ~10 old errors (memory note, gap 7). |
| 6 | Build, real API mode (`npx vite build`) | Produces the production bundle. | Build error |
| 7 | Build, mock mode (`VITE_USE_MOCKS=true`) | Same, with the mock API switch on, so both modes stay buildable. | Build error |
| 8 | SonarCloud scan | `sonarqube-scan-action` using `sonar-project.properties` plus org/project key from variables. | Bad token, wrong keys |

Key facts:
- **No tests and no coverage yet**: there is no test runner. Sonar still analyses the sources in `src/`.
- **Lint rules** are tuned for the existing code: legacy patterns (`any`, `v-html`, unused variables) are **warnings**, not errors. Tighten them to `error` in `eslint.config.js` as code is cleaned.
- `vite build` is used instead of `npm run build` because the latter runs `vue-tsc -b` first, which currently fails on old errors. When gap 7 is fixed, remove `continue-on-error` and use `npm run build`.
- Sonar scope is defined in `sonar-project.properties` (`sonar.sources=src`, plus exclusions).

Run the same thing locally (from `frontend/`):
```
npm ci
npm run lint
npx vite build
VITE_USE_MOCKS=true npx vite build
```

## 5. SonarCloud

- Analysis runs **in GitHub Actions** (not SonarCloud's "Automatic Analysis"), because that is the only mode that includes coverage.
- Each repo is one SonarCloud project:

| Repo | Variable for project key |
|---|---|
| `6_back` | `SONAR_PROJECT_KEY_API` |
| `6_front` | `SONAR_PROJECT_KEY_WEB` |

- Shared: variable `SONAR_ORGANIZATION` (org key, a short slug).
- The **quality gate** (default "Sonar way") judges **new code**: coverage, duplication, bugs, vulnerabilities, security hotspots. Results appear in the SonarCloud project page and as PR decoration if the GitHub app is installed.
- Note: the workflow step does not wait for the quality gate result. To make a failed gate fail the pipeline, add `-Dsonar.qualitygate.wait=true` to the Sonar arguments.

## 6. Secrets and variables (set per repo in GitHub **Settings > Secrets and variables > Actions**)

| Name | Type | Repos | Notes |
|---|---|---|---|
| `SONAR_TOKEN` | Secret | both | SonarCloud token. Never put it in a file or a variable (variables are visible in logs). |
| `SONAR_ORGANIZATION` | Variable | both | Org key, for example `fateccoderhood` |
| `SONAR_PROJECT_KEY_API` | Variable | back | for example `FatecCoderHood_6_back` |
| `SONAR_PROJECT_KEY_WEB` | Variable | front | for example `FatecCoderHood_6_front` |

The variable names must match the workflow exactly. A wrong name shows up as an **empty** value and Sonar fails with "You must define the following mandatory properties: sonar.projectKey".

## 7. Dependabot

Config: `.github/dependabot.yml`. It opens PRs weekly, which then run this CI.

| Repo | Ecosystems | Grouping |
|---|---|---|
| `6_back` | Maven (`/enersight-api`), pip (`/enersight-worker`, `/enersight-temporal-series`), GitHub Actions | Spring Boot packages together, testing libraries together |
| `6_front` | npm (`/`), GitHub Actions | Vue/Vite packages together, ESLint packages together |

How to treat them: read the PR, wait for green CI, merge. A red CI on a Dependabot PR means the update breaks something. This is the point of having tests.

## 8. Reading a failed run

1. Open the PR > **Checks** tab (or the repo **Actions** tab) and click the red job.
2. Open the first step with a red X and read the **last lines**.
3. Common causes:

| Symptom | Likely cause |
|---|---|
| `Connection to localhost:5432 refused` | A test needs a DB but doesn't start one. Use Testcontainers. |
| `You must define the following mandatory properties: sonar.projectKey` | Missing or misnamed repo variable |
| `Not authorized` / 401 from Sonar | Wrong or revoked `SONAR_TOKEN`, or the token has no access to the project |
| `npm ci` fails | `package.json` and `package-lock.json` are out of sync |
| Lint error | Run `npm run lint` locally and fix the reported file |
| Job sits on "Waiting for a runner" | GitHub runner queue or Actions quota. Not a code problem. |

Backend test reports are downloadable from the run page under **Artifacts** (`surefire-reports`) when tests fail.

## 9. Not covered yet

- **Branch protection** requiring these checks is a GitHub setting (Settings > Branches). Without it, a red CI doesn't stop a merge.
- **Frontend tests and coverage**: no test runner yet.
- **Type-check is non-blocking** until gap 7 is fixed.
- Secret scanning, push protection and CodeQL are not configured.
- Python projects (`enersight-worker`, `enersight-temporal-series`) have Dependabot only, with no CI job.
