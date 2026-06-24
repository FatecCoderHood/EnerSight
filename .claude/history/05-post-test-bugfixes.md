# 05-post-test-bugfixes.md

- **Date:** 2026-06-23
- **Status:** Done
- **Component:** frontend (CadastroPage.vue, UsersPage.vue)

## Context
Manual testing of [[04-registration-approval-flow]] against the real `enersight-auth` backend
surfaced 4 issues. None require backend changes — all are frontend bugs/UX decisions exposed only
once a real (non-mock) backend was actually used.

## Decisions
- **Drop phone from the signup form entirely** (input, `formData.phone`, validation, mock-fallback
  arg) — reverses the prior task's choice to keep it "for consent UX." Name stays; only phone was
  flagged as a problem.
- **No new test infrastructure for this round.** The valuable regression test (the async race in
  `handleConfirmAction`) is closed over component-internal state with no exported, testable unit,
  and this repo has zero precedent for mounting `.vue` components in tests (`@vue/test-utils` isn't
  installed; the one existing test file covers a plain `.ts` usecase). Extracting logic just to test
  it, or adding component-mount infra for 3 small fixes, was judged disproportionate. Verifying via
  dev server + live Docker backend instead, per CLAUDE.md's existing mandate for frontend changes.
- **Fix `reject` too, even though only approve/delete were reported.** It shares the exact same
  async-race pattern in `handleConfirmAction`; fixing two of three and leaving the third would just
  be deferring an identical bug report.
- **Normalize role casing at two more spots**, not just the display helpers, so the bug can't
  resurface via the edit-role flow: `openUserModal` lowercases the role when pre-filling the edit
  dropdown (an uppercase `'ADMIN'` from the real backend wouldn't match either lowercase `<option>`
  otherwise), and `saveUser` uppercases the role before sending it to the real backend (keeping
  backend-stored roles consistently uppercase regardless of which UI path wrote them). Mock-fallback
  path keeps its existing lowercase convention unchanged.

## Plan
1. **CadastroPage.vue** — remove the phone `<input>` block from the template; remove `phone` from
   `formData`; drop the `!formData.value.phone` check from `handleCadastro`'s validation; drop
   `phone: formData.value.phone` from the `MockUsers.create(...)` call in `registerWithMock`.
2. **UsersPage.vue role display** — `getRoleClass`/`getRoleName`: compare case-insensitively instead
   of `=== 'admin'`. `openUserModal`: lowercase `user.roles[0]` when populating `userForm.value.role`
   for the edit modal. `saveUser`: uppercase the role when calling `usersApi.create`/
   `usersApi.updateRoles` (real backend only — mock branch unchanged).
3. **UsersPage.vue confirm-action race** — `handleConfirmAction` becomes `async` and `await`s
   `pendingAction()` before calling `closeConfirmModal()`; `pendingAction`'s type changes from
   `(() => void) | null` to `(() => Promise<void>) | null` to match what's actually stored.
   `executeApprove`/`executeReject`/`executeDelete` each capture `pendingUser` into a local `const`
   right after their `if (!pendingUser) return` guard, so they stop depending on the shared
   closure variable surviving across their own internal `await`.

## Files touched
- `frontend/src/pages/CadastroPage.vue` — phone field removed (template + script).
- `frontend/src/pages/UsersPage.vue` — role-casing fixes (3 spots) + async-race fix (4 spots).

## Testing
No browser automation tool is available in this environment, so the UI itself could not be visually
verified directly. What was actually done instead:
- `npm run build`: same pre-existing 21 TypeScript errors as before this change, none new, none in
  either touched file beyond one already-known unused-variable line that just shifted line numbers.
- `npx vitest run`: existing 4 tests in `auth.usecase.test.ts` still pass (untouched by this change).
- Live curl against the already-running `enersight-auth` container confirmed the real admin user's
  role really does come back as `["ADMIN"]` (uppercase), grounding the role-casing fix in actual
  data rather than assumption.
- An isolated Node script (throwaway, scratchpad-only) replicated the exact before/after
  `handleConfirmAction`/`execute*` pattern: the buggy version reproduced the reported "success API
  call, false error toast" symptom; the fixed version did not.
- The user already had their own `npm run dev` + Docker backend running (from the manual testing
  session that found these bugs) with an active browser connection — Vite HMR will have already
  applied these edits live to that session. That session's actual visual behavior is the user's to
  confirm, not something observed directly here.

## Review
- Searched for other lowercase/uppercase role-comparison mismatches beyond `UsersPage.vue`: found
  one in `pages/MinhaConta.vue:24` (`user.role === 'admin' ? ... : ...`). Not fixed — it's protected
  by a separate, already-known, already-twice-deferred limitation: `MinhaConta.vue` always sources
  its own profile data via `MockUsers.getByEmail(...)` regardless of which backend authenticated the
  session (documented in [[03-frontend-auth-integration]] and [[04-registration-approval-flow]]'s
  Follow-ups), so it never actually sees the real backend's uppercase roles today. Flagged below
  rather than silently fixed, since touching it would be expanding scope beyond the 4 reported bugs.
- The `const user = pendingUser` capture in each `execute*` function is redundant with the
  `handleConfirmAction` await-ordering fix alone — either one independently prevents the reported
  crash. Kept both: the await fix is the actual root-cause correction, the local capture is a cheap
  second line of defense against the same bug class if a future edit ever reintroduces a
  fire-and-forget call site. No other shared mutable state (`confirmModalTitle`/`Message`/etc.) has
  the same exposure — those are only written before the modal opens and read only by the template
  while it's visible, never read by `execute*` after an `await`.
- Phone removal confirmed complete: no leftover `phone` reference in `CadastroPage.vue` except one
  comment, which was stale (still said "name/phone are collected" — corrected to "name is still
  collected"). No phone-specific CSS class existed (the removed block only used classes shared with
  the other fields), so nothing was orphaned.

## Follow-ups
- `MinhaConta.vue:24` has the same role-casing pattern as the bug fixed here, currently masked by
  the page always reading from mock data regardless of which backend is authenticated. Surfacing
  again as a third call-out (after [[03-frontend-auth-integration]] and this file) in case it gets
  picked up before `MinhaConta.vue` is ever wired to the real backend.
