---
name: clean-code
description: Apply Clean Code practices (readable names, small single-purpose functions, useful comments, code smells and safe refactoring) to EnerSight code (Java/Spring API, Python services, Vue/TypeScript frontend). Use when writing, reviewing, or refactoring code, or when asked about "clean code", readability, naming, or code smells.
---

# Clean Code for this repository

## Sources and how far to trust them
- *Clean Code* is Robert C. Martin's book and paradigm, tied to SOLID, TDD, KISS and structured programming (Wikipedia: *Robert C. Martin*). Core teaching: "Functions should do one thing. They should do it well. They should only do it."
- **Code smell** (Kent Beck, popularised by Martin Fowler, *CodeSmell*): a quick-to-spot surface hint of a deeper problem. A smell is an indicator, not always a defect — investigate before changing.
- Refactoring = behaviour-preserving restructuring (Fowler, *Refactoring* catalog at refactoring.com).
- SOLID (Wikipedia: *SOLID*) targets object-oriented design. **Single responsibility** transfers to every module here; the others (open/closed, Liskov, interface segregation, dependency inversion) fit the Java API's layers but are overkill for a small Python script or a Vue component — apply them only where they remove a real problem.
- **Known disagreement:** John Ousterhout argues that shrinking functions beyond a few dozen lines doesn't help readability, that too-small functions add interfaces to learn, and that useful comments are valuable. So: apply guidance with judgement, favour clarity, and don't split code into tiny fragments for their own sake.
- Balance with the `kiss` skill: clean code means *readable and changeable*, not more abstraction.

## Practices to apply
**Names**
- Reveal intent and be pronounceable/searchable; avoid `tmp2`, `data`, `x` (except short loop indexes).
- Name functions as verbs, data as nouns; keep one word per concept (don't mix `add`/`insert`/`create` for the same idea).
- Follow the naming convention of the module you are editing (Java `camelCase`/`PascalCase`, Python `snake_case`, Vue/TS conventions already present in `src/`). Detect it from existing files; never import conventions from another module or project.

**Functions**
- Do one thing at one level of abstraction; prefer few parameters; no hidden side effects (a `find…`/`get…` must not modify state).
- Use early returns/guard clauses to reduce nesting.
- Keep controllers/routes thin: validation and mapping at the edge, business rules in the service layer, persistence in repositories (API); keep components presentational and move logic to composables/services (frontend).

**Comments**
- Code should explain itself; comments explain *why* or a business rule. Delete comments that restate code, stale comments, and commented-out code *you* introduced (the owner's commented blocks stay unless asked).

**Formatting & structure**
- Follow the formatter/linter configuration present in the module; if none exists, match the surrounding file instead of reformatting it.
- Avoid magic numbers and strings (constants, enums, i18n keys in the frontend).

**Error handling & resources**
- Fail explicitly with meaningful exceptions/messages; don't swallow errors or ignore return values; close resources (connections, files) via the framework's managed mechanisms.
- Never log or commit secrets, tokens or personal data.
- Every new exception type needs an explicit handler/mapping; an unmapped exception becomes an opaque 500.
- Compare enums with enums, not with strings (a known bug here: an enum `.equals("MANDATORY")` is always false).
- Existing code is evidence of convention, not of correctness: don't replicate a flaw (`any`, permissive security rules, hard-coded values) just because it is already there; mention it.

**Tests**
- See the `testing` skill. A change that alters behaviour needs a test that would fail without it.

## Code smells to look for (Fowler/Beck vocabulary)
Long function; duplicated code; unclear/misleading name; long parameter list; deep nesting; magic numbers; dead code; commented-out code; mixed responsibilities; global mutable state; inconsistent naming; swallowed exceptions; business logic in controllers or components.

## Refactoring workflow
1. Make sure current behaviour is captured by tests (add characterization tests first if none exist).
2. Apply **one** small refactoring at a time (rename, extract method, replace magic value, remove duplication, introduce guard clause).
3. Re-run the module's tests/build after each step (commands in the module's `copilot-instructions.md`).
4. Stop when the code reads clearly; don't over-polish.

## Review output
List findings by priority (correctness → readability → style), each with file/line, the smell, and a concrete suggested change. Mention opinionated advice as a trade-off, not a rule. Don't touch unrelated modules.
