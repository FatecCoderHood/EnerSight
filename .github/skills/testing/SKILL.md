---
name: testing
description: Write and run unit, sanity (smoke) and integration tests for EnerSight (Spring Boot API, Python services, Vue frontend). Use when asked to test, verify, validate or add tests for a change, or during the "Test" step of the dev-workflow skill.
---

# Testing in EnerSight

## Sources and how they are used
- **Unit test** (Martin Fowler, *UnitTest*): low-level, focused on a small part of the system, written by programmers with ordinary tools, and significantly faster than other tests. What counts as a "unit" is situational: a class/method (API), a function (Python), a component or composable (frontend).
- **Integration test** (Fowler, *IntegrationTest*): checks that separately developed units work correctly when connected. It can be narrow in scope; it doesn't have to be broad.
- **Sanity test** (Wikipedia, *Sanity check*): quick, broad, shallow check that a change works and it is reasonable to continue testing. Often used interchangeably with smoke tests. This repo uses: sanity = the changed thing builds/starts and its main path works.
- **Test pyramid** (Fowler, *TestPyramid*): many fast low-level tests, fewer high-level ones. When a higher-level test finds a bug, reproduce it with a unit test first so it stays fixed.
- **Test-Driven Development** (Fowler, *TestDrivenDevelopment*, after Kent Beck): repeat three steps: write a test for the next bit of functionality; write the functional code until it passes; refactor new and old code. Start by writing a list of test cases, pick one, run red-green-refactor, then pick the next, adding cases as they occur. Benefits named by the source: self-testing code, and designing the interface before the implementation. The source calls neglecting the refactor step the most common way to get TDD wrong.

## TDD in EnerSight
Default to TDD for new or changed behaviour in modules that have a test runner (today only `backend/enersight-api`).
1. **List** the test cases from the plan's behaviours: typical, edge, error, allowed and denied.
2. **Red:** write one test, run it, and confirm it fails **for the expected reason** (not a compile error or a wrong setup).
3. **Green:** write the simplest code that passes (see `kiss`). Don't add behaviour no test asks for.
4. **Refactor:** clean the new and the touched old code (see `clean-code`) with all tests green. Don't skip this step.
5. Pick the next case; repeat. Run the module's whole suite before finishing.
- **Bug fix:** write a test that reproduces the bug (red) first, then fix it.
- **No runner** (frontend, Python modules): do not add one without asking (see tooling below). Until approved, write the case list and verify behaviour with the sanity run, and report that TDD was not possible.
- Where a test needs infrastructure that isn't available (e.g. a database), say so; don't fake a red or green result.

## Test levels
| Level | What | Example |
|---|---|---|
| Sanity | Builds, starts, main path works | API starts and Flyway migrates; frontend `build` passes; a service script runs on a small sample |
| Unit | One class/function/component, one behaviour, edge cases included | Service rejects a duplicate user; a pandas treatment function drops years before the minimum; a composable maps an API error |
| Integration | Several units connected, narrow scope | Controller + service + repository against a test database; worker job writes rows to the DB |

Prefer many unit tests, a handful of integration tests, and one sanity run per change.

## Tooling per module (verified at time of writing; re-check before relying on it)
- **`backend/enersight-api`** (Java, Maven wrapper): the dependency list has Spring Boot test starters for webmvc, data-jpa and flyway; the README documents `./mvnw test` and `./mvnw clean package`. Only `EnersightApplicationTests` existed when this was written, so there is little to imitate; follow standard JUnit 5 / Spring Boot test conventions.
- **`backend/enersight-worker`** and **`backend/enersight-temporal-series`** (Python): no tests, runner, linter or formatter were found. Do not assume `pytest`; if you add tests, ask which runner to adopt and keep it out of production `requirements.txt`. Worker integration needs Postgres and `ogr2ogr`; temporal-series needs MongoDB and Prophet/cmdstan, so prefer unit tests on pure functions (treatment, MAE logic) with small inline data.
- **`frontend`** (Vue 3, Vite, TypeScript): `package.json` defines only `dev`, `build` (`vue-tsc -b && vite build`) and `preview`; no test framework is installed. Sanity = `npm run build` (type-check + build). Adding Vitest is a dependency decision to confirm with the owner first. Most screens run on mocks, so a passing screen does not prove backend integration.
- **Candidates for first tests** (known defects, see the module files): `TermsOfUseService.create` enum/string comparison, unmapped exceptions returning 500, `approve-access` without a role check, `permitAll` on write endpoints.

## Writing good tests
- **Name** tests after the behaviour; one behaviour per test (Arrange → Act → Assert, separated by blank lines).
- **Deterministic and fast**: no real network, no wall-clock or randomness without control, no dependence on the developer's local data. Use test containers or an in-memory/test database for integration tests only if the module already uses them; otherwise ask.
- **Always cover**: empty/missing input, single element, typical case, boundary values, duplicates, and error paths (validation failures, not found, unauthorized).
- **Security-relevant code** (JWT, roles, user approval flow): test both the allowed and the denied case.
- **Database changes**: a new Flyway migration is verified by starting the application against a clean database (`validate` mode will fail on mismatch); never edit an already-applied migration.
- **Data pipelines**: assert on shapes and values of the output (columns, row counts, no nulls where forbidden), using small inline fixtures instead of the large CSV files.
- A failing test is information: fix the code or the expectation deliberately; never delete or weaken a test just to go green.
- A bug found by an integration/sanity run gets a unit test first, then the fix.

## Workflow
1. Decide the levels needed (always sanity; unit for each new/changed behaviour; integration when units interact).
2. Work test-first (see "TDD in EnerSight"): list cases, red, green, refactor. Where TDD isn't possible, write tests right after implementing and say so.
3. Run the module's own commands only (see its `copilot-instructions.md`); never run another module's tooling.
4. Report: files, which levels were run, the exact command, and the pass/fail result. If a level could not be run (no framework, no database), say so instead of implying it passed.

## Output expectations
Short summary: tests added (names), command run, result. For TDD work, say that each new test was seen failing first and why. Include failing output verbatim if anything fails.
