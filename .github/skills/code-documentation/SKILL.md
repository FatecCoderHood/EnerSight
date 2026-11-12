---
name: code-documentation
description: Write or update documentation for EnerSight (module READMEs, root README, docs/ folder, API contracts, in-code comments). Use when asked to document, explain, or create a README or technical doc for a feature, module, endpoint, migration or data pipeline.
---

# Documentation in EnerSight

## Sources and how they are used
- **Diátaxis** (diataxis.fr): documentation serves four distinct needs — tutorials, how-to guides, reference, explanation. Don't mix them in one page.
- **Write the Docs, Documentation principles**: docs should be precursory (written early), skimmable, exemplary (show examples), consistent, current; accept some repetition (ARID) because docs can't be strictly DRY with code.
- **Google developer documentation style guide**: plain, active voice, descriptive link text, consistent formatting for code.
- Clean Code view (see `clean-code` skill): comments explain *why*, not what a line does. Ousterhout's counterpoint: comments that capture intent or interface are valuable. Both agree: no stale or redundant comments.

## Where documentation lives (observed)
| Place | Purpose | Observed style |
|---|---|---|
| `README.md` (workspace root) | Team, project challenge, requirements, backlog, sprints, stack, branching and commit standard | English, HTML anchors, badges, tables |
| `docs/` (workspace root) | Technical documentation and roadmaps (`Injestor-Roadmap.md`) | Markdown (+PDF export) |
| `backend/enersight-api/README.md` | How to run, reset DB, tests, build, common issues | English, short sections with emoji headings and fenced `bash` blocks |
| `backend/enersight-temporal-series/README.md` + `docs/` | Pipeline flow, collections, import details | **Portuguese**, flow diagram in a code block |
| `frontend/README.md` | Currently the unchanged Vue/Vite template | n/a |

## Language rule
Follow the language of the file you are editing; do not translate existing documents unless asked. New workspace-level or API documents default to English; keep Portuguese for the data-pipeline docs that already use it. Business terms from ANEEL (DEC, FEC, indicadores de continuidade, etc.) keep their original Portuguese names.

## What to document, by change type
- **New/changed endpoint**: method and path, auth/role required, request and response shape, error cases. The `README` stack badges mention Swagger, but the API `pom.xml` had no OpenAPI dependency when this was written — verify before claiming generated API docs exist.
- **New Flyway migration**: purpose in the PR description and, if it changes domain meaning, a line in the relevant doc. Never rewrite an applied migration.
- **Data pipeline change**: update the flow, source/destination collections or tables, and any assumption about input format (delimiters, optional files) in that module's README.
- **Setup/run changes**: update the module README commands; test them as written.
- **Business rule**: put it in the root documentation (single place), not scattered in code comments.

## In-code comments
- Sparse, explain *why* or the business rule. Put them immediately above the code they describe. Match the language already used in that file.
- Don't comment obvious lines, don't leave commented-out code you wrote, keep comments in sync with code in the same change.

## Workflow
1. Read the code and run it (using the module's own commands) so the documentation reflects real behaviour; never invent steps or values.
2. Find the existing document that owns the topic and edit it instead of creating a parallel one.
3. Keep examples runnable and minimal; never include secrets, tokens, passwords or real personal data (use placeholders).
4. Check links, code fences and tables render on GitHub.
5. If behaviour changed, update the related docs in the same change.

## Output expectations
Say which files were created or changed and show the final diff. Keep chat answers short.
