name: plan

description:
High-level planning skill that orchestrates other skills before implementation.

Workflow:

1. Understand the request
   - Clarify intent and expected outcome
   - Identify constraints and scope

2. Inspect existing codebase
   - Locate relevant modules and patterns
   - Detect existing architecture and reuse opportunities

3. Define approach (keep-simple + clean-code)
   - Prefer minimal and direct solutions (keep-simple skill)
   - Avoid over-engineering or unnecessary abstractions
   - Align with existing code style and readability standards (clean-code skill)

4. Impact analysis
   - List all files likely to be modified
   - Identify dependencies and side effects
   - Highlight risks or breaking changes

5. Implementation plan
   - Break work into small, sequential steps
   - Ensure changes are incremental and scoped
   - Prefer smallest possible diff

6. Testing strategy (tdd skill)
   - Define required tests before implementation
   - Identify existing tests to extend
   - Specify edge cases to validate

7. Review strategy (review skill)
   - Define validation checklist before completion
   - Ensure no unrelated changes are introduced
   - Verify consistency with architecture and naming

8. Approval gate
   - Present final plan clearly
   - Wait for explicit approval before any implementation