---
name: project-memory
description: Keep EnerSight's verified knowledge (feature state, findings, gaps, decisions) in versioned files under docs/project-memory and docs/adr, and keep it current. Use before working on a feature (to recall what is already known), after an investigation or analysis (to record findings), and whenever a change fixes or touches something already recorded (to update the live documentation).
---

# Project memory

Durable team knowledge lives **in the repository**, reviewed like code. The agent's own memory is not a substitute.

## Why files in the repo (validated sources)
- **GitHub Copilot Memory** is created automatically, is checked against citations, **expires after 90 days without reinforcement by default**, and GitHub states that a stored fact "should not be relied on as a rule that will always be followed"; to give instructions consistently, use custom instructions. It also does not store secrets or short-lived/task-specific instructions. (docs.github.com, *About GitHub Copilot Memory*)
- **Custom instructions and skills** are the supported, versioned way to give Copilot lasting context: `.github/copilot-instructions.md`, `.github/instructions/*.instructions.md`, nearest `AGENTS.md`, and `.github/skills/<name>/SKILL.md`. (docs.github.com, *Adding repository custom instructions*, *Adding agent skills*)
- **Docs as code** (Write the Docs): write documentation with the same tools as code — issue tracker, Git, plain-text markup, code review — and merges can be blocked when a feature lacks documentation.
- **Architecture Decision Records** (Michael Nygard, 2011; adr.github.io): short, modular, numbered files kept in the repo; a reversed decision is kept and marked superseded; the point is to preserve the *motivation* so people neither blindly accept nor blindly change past decisions. "Large documents are never kept up to date. Small, modular documents have at least a chance."
- **Agent Skills format** (agentskills.io): `name` must match the directory; `description` says what and when; keep `SKILL.md` under 500 lines and move detail into files one level deep (progressive disclosure).

Design choices in this skill that are **ours**, not from those sources: the `docs/project-memory/` folder name, the "Verified against" header, and the status labels below.

## Where things go
| What | Where | Shape |
|---|---|---|
| State of a feature or area, findings, gaps, backlog | `docs/project-memory/<area>.md` (one per area, e.g. `authentication.md`) | `assets/note-template.md` |
| A decision with lasting consequences (why we chose X over Y) | `docs/adr/NNNN-short-title.md` | `assets/adr-template.md` (Context, Decision, Status, Consequences) |
| A rule the agent must follow on every task | the module's `.github/copilot-instructions.md` | short; link to the note instead of copying it |
| Index of all notes | `docs/project-memory/README.md` | one line per note |

ADR rules (Nygard): numbered sequentially, numbers never reused, one decision per file, never delete — mark `Superseded by NNNN`.
Notes about backend or frontend code live in the **workspace** `docs/`; the module instruction files only point to them.

## When to use it
1. **Recall** — at the start of any feature, fix or analysis: read `docs/project-memory/README.md`, then the notes for the area. Treat them as *claims to re-check against the code*, not as truth (see "Verified against").
2. **Record** — after an investigation, review or incident: write or update the note for that area. Facts only, each with the file (and line or symbol) where it was observed. Mark anything not observed as `Unverified`.
3. **Update** — when a change fixes, removes or touches something a note describes: update the note **in the same change**. Mark the gap `Fixed in <TASK-ID>` or remove it, refresh "Verified against", and add an ADR if a decision was made.
4. **Cross-module** — if a finding involves both backend and frontend, put it in one note, label each item `[back]` or `[front]`, and add a pointer line to **both** modules' instruction files so either side finds it.

## Rules for notes
- Only what was **observed** (code read, command run, response received). Never infer or guess; label uncertainty `Unverified` and say how to verify it.
- Every note starts with **Verified against**: date, and the branch and commit of each module read. If the code on your branch differs, re-verify before relying on it.
- Status labels: `Works`, `Partial`, `Missing`, `Broken`, `Unverified`. Say *how* it was checked: `read code` or `ran it`.
- Small and modular: one area per note, about two pages at most. Split rather than grow.
- **Never store secrets**, tokens, real passwords, personal data or local-only paths. Reference where a secret lives, not its value.
- No task-specific chatter or plans; backlog items state the problem and the target behaviour.
- Same language as neighbouring docs (English by default here).

## Process
1. Find the existing note for the area (README index). Edit it; create a new one from the template only if none fits, and add it to the index.
2. Re-check each claim you rely on against the current code before reusing it.
3. Write or update the note and its "Verified against" block.
4. If a decision was made, add an ADR (copy the template, next free number).
5. If a module needs to know the note exists, add or keep a one-line pointer in its `.github/copilot-instructions.md`.
6. Mention in the change summary which memory files changed. Commit only when the user asks.

## Checklist
- [ ] Note read before the work; claims re-verified against code
- [ ] New facts recorded with where they were observed; unknowns marked `Unverified`
- [ ] Touched or fixed items updated in the same change; "Verified against" refreshed
- [ ] ADR added or superseded when a decision was made
- [ ] Index and module pointers up to date (back and front when both are involved)
- [ ] No secrets or personal data
