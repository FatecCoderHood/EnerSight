---
name: dev-workflow
description: Standard end-to-end workflow for any change in EnerSight - Plan, Implement (kiss + clean-code), Test (testing), Document (code-documentation), delivered with the team's branch and commit standard. Use when asked to add, implement, change or fix a feature, or when the user mentions "the workflow".
---

# Development workflow

```
Plan  →  Implement (kiss + clean-code)  →  Test (testing)  →  Doc (code-documentation)  →  Deliver
```
Run the steps in order. If one fails, go back to the step that caused it. Keep each step proportionate to the change.

## 1. Plan
- Restate the goal in one sentence and list the **behaviours** to deliver (inputs → outputs, edge cases, errors, who is allowed).
- Identify the **module(s)** involved (`backend/enersight-api`, `backend/enersight-worker`, `backend/enersight-temporal-series`, `frontend`) and read that module's `copilot-instructions.md` before touching it. Work that spans modules must be planned per module (they are separate git repositories).
- Find the existing code and conventions to follow; don't assume they match another module. Check the module's "Known gaps" first: existing code shows what is conventional, not what is correct, so don't copy a flawed pattern (e.g. `permitAll`, mocks, hard-coded credentials).
- Read the area's note in `docs/project-memory/` (skill `project-memory`) and re-check its claims against the code.
- If the work touches a requirement (RF/RNF) in the root README, note its current status (implemented, mocked, missing) in the plan.
- Apply YAGNI: list what you are deliberately **not** doing.
- Define the tests up front (sanity / unit / integration) from the behaviours, as a test-case list for TDD (see `testing`).
- Tie the work to a task ID (`ES-NN`). If none was given, ask.
- If requirements or business rules are ambiguous, ask before coding. Share a brief plan for non-trivial changes.

## 2. Implement — skills `kiss` and `clean-code`
- Simplest correct version first; readable names, single-purpose units, guard clauses, the module's existing conventions.
- Change only what the plan covers; no unrelated refactors, reformatting or dependency upgrades.
- Validate input at the boundary, handle errors explicitly, keep secrets out of code and logs.

## 3. Test — skill `testing` (TDD: red → green → refactor)
- Steps 2 and 3 run together as a TDD loop where a test runner exists: one failing test, the simplest code to pass, refactor, next case.
- Sanity, unit and integration as planned, using the module's own commands.
- Fix failures at the cause; never weaken tests to pass. Report any level that could not be run.

## 4. Doc — skill `code-documentation`
- Update the module README / `docs/` / comments that the change makes stale. Update `.github` instruction files only if conventions changed.
- Update the matching `docs/project-memory/` note (and add an ADR for a lasting decision) when the change fixes or touches something it records.

## 5. Deliver (branch and commit standard from the root README)
- Branch from `develop`, named `<TASK-ID>/<lowercase-hyphenated-description>` (e.g. `ES-01/login-screen`). Hotfixes branch from `release` and are merged back into both `release` and `develop`.
- Commit message: `<TASK-ID> <type>: <short lowercase description>` (e.g. `ES-01 feat: add login endpoint`). Types: feat, fix, chore, docs, style, refactor, perf, test, build, ci, revert, hotfix. Small, frequent commits; one task per commit.
- Pull request targets `develop`; the team's rule is two approvals before merge.
- A change inside `backend/` or `frontend/` is committed and pushed in **that submodule's** repository first; the parent repository then records the new submodule pointer in its own commit.
- **Commit or push only when the user asks.** Never commit build outputs, virtual environments (`.venv`), `node_modules`, large CSV data or secrets.

## Finish
Show a short summary: what changed, tests run (commands + results), docs updated, and anything not verified.

## Checklist
- [ ] Task ID, goal, edge cases and non-goals written down
- [ ] Correct module identified and its instructions read
- [ ] Simple, readable implementation; no unused code; no secrets
- [ ] Tests run with the module's commands; results reported honestly
- [ ] Docs updated and consistent with the code (including `docs/project-memory/` notes)
- [ ] Only intended files changed; branch and commit names follow the standard
