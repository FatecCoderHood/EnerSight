# 0002 — Mock/real API switch

- Status: accepted (ES-67)
- Context: the frontend team must not be blocked by backend work.
- Decision: `VITE_USE_MOCKS=true|false` is read once in `src/config/api.config.ts` (`USE_MOCKS`). Each API module (`auth.api.ts`, `users.api.ts`, `terms.api.ts`) picks the mock or real implementation once, behind the interfaces in `src/api/types.ts`. Mocks live in `src/api/mocks/` and return backend DTO shapes and `ErrorResponse` errors (400/401/403/409/422). Pages never branch on the flag. `.env.example` is committed; real `.env*` files are git-ignored.
- Consequences: mocks must be updated when a DTO or rule changes (e.g. status model). Silent fallbacks to mocks on API failure were removed.
