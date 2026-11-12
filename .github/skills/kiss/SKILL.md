---
name: kiss
description: Apply the KISS (Keep It Simple) principle when writing, reviewing, or simplifying code in EnerSight (Spring Boot API, Python services, Vue frontend). Use when asked to implement a feature, review code for over-engineering, or "simplify" / "keep it simple".
---

# KISS — Keep It Simple for source code

## Where the principle comes from
- KISS is attributed to Kelly Johnson (Lockheed Skunk Works): a design must be repairable by an average mechanic with basic tools. Applied to software: simple enough for an average developer to understand and fix. (Wikipedia: *KISS principle*)
- YAGNI ("You Aren't Gonna Need It", XP): don't build presumptive features. They cost build effort, delay other value, and add complexity that taxes every later change. Unused extensibility points get in your way. (Martin Fowler: *Yagni*)
- Kent Beck's four rules of simple design, in priority order (Martin Fowler: *BeckDesignRules*):
  1. Passes the tests (it works)
  2. Reveals intention (easy to read)
  3. No duplication (say things once)
  4. Fewest elements (remove anything not serving 1–3)
- Rule of least power: choose the least powerful tool/construct that does the job. (Wikipedia: *Rule of least power*)

Simple does not mean sloppy or shortest: *simple* = easy to understand and change; it must still be correct.

## Workflow
1. **State the goal in one sentence.** Implement only that (YAGNI).
2. **Prefer the idiomatic approach of the module's framework** (Spring, SQLAlchemy/pandas, Vue composition API); avoid cleverness and custom frameworks unless explicitly requested.
3. **Write the straightforward version first**, then verify it works using the module's own build/test commands (see the module's `copilot-instructions.md`; never assume commands from another module).
4. **Refine in Beck's order**: names/structure that reveal intention → remove duplication → delete anything unnecessary.
5. **Report** what was kept simple and any deliberate trade-off (e.g. recursion vs iteration, Big-O).

## Checklist for this repo
- One concept per function/method/component; explainable in one sentence. Short, single-purpose, no deep nesting (use early returns / guard clauses).
- Use what the stack already provides (Spring Data repositories, Bean Validation, Flyway, Vue composables, axios client in `src/api`) before adding a new layer, library or dependency.
- No new abstraction (interface, base class, generic helper, new module) for a single caller; wait for the second or third use.
- Descriptive names, not abbreviations; follow the naming style of the file and module you are editing.
- Comments explain *why*, not what the line does. If a comment is needed to explain *what*, simplify the code instead.
- No dead code or speculative hooks ("for later"), and no config flags nothing reads.
- Don't duplicate logic, but don't invent abstractions for only two uses either.
- Correctness and security are non-negotiable: validate input at the boundary, handle empty/missing data, never hard-code secrets.

## Smells to flag when reviewing
- Parameters/flags/options nothing uses; generic code with one caller.
- Wrapper functions that only forward a call.
- Clever one-liners (nested ternaries, chained stream/comprehension pipelines that need a comment to decode).
- Deep nesting, long functions, multiple responsibilities in one function.
- Copy-pasted blocks that differ in one value.

## Example (illustrative, not taken from the code base)
Simple: a service method that loads by id, throws a not-found exception if absent, and returns the entity.
Over-engineered (avoid): a generic `Finder<T, ID, Strategy>` hierarchy introduced for a single entity.

## Output expectations
Keep answers short: show the simple solution, one or two lines on why it is simple, and the verification result. Suggest simplifications as concrete diffs, not general advice.
