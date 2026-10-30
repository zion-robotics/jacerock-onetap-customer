# Instructions for AI coding assistants

You are working on the AfrikBerry One-Tap customer app. Read `README.md` and the files in `docs/` before changing anything. `docs/PRD.md` is the source of truth for what to build, `docs/SCHEMA.md` for the database, `docs/API.md` for calls, `docs/FLOWS.md` for screens and messages.

## Hard rules

1. **Never show "Jacerock" or the parent company name** in the UI, titles, errors, manifest or assets. The only name is AfrikBerry One-Tap.
2. **Secrets.** Never put the Supabase service role key, the Meta app secret, the verification secret or any password in this repo, in `VITE_` variables, in logs or in chat. Only the Supabase URL and anon key reach the browser.
3. **No direct table access from the app.** The app reads and writes only through the database functions in `docs/API.md` and the server endpoints. Never `supabase.from('transactions')` and the like in customer code.
4. **Money.** Use `decimal.js` through `src/lib/money.ts`. Never use plain floating point arithmetic on amounts. Rounding is half up to 2 decimals. The database function is the authority; the screen only previews.
5. **Business rules live in the database.** Opening hours, limits, rate checks and amounts are enforced in `create_transaction`. Do not rely on the screen for any of them.
6. **Migrations.** Each file in `supabase/migrations/` must be safe to run twice. Never edit a migration that has been run; add a new numbered file.
7. **Phone numbers** are digits only with the country code and no plus (for example `2207012345`).
8. **Copy.** Short sentences, plain words, no em dashes. Use the exact texts in `docs/FLOWS.md`. Status words are Processing, Completed and Failed.
9. **Accessibility.** Text 16 px or more, inputs 18 px or more, buttons 48 px or more, visible labels, number keypad for numbers.
10. **Do not change what you were not asked to change.** Keep edits small and explain them.

## How to work

- TypeScript strict mode. No `any`.
- Run `npm run build` and `npm run test` before saying a task is done. Fix every error.
- Write or update tests for money logic and for every function you add.
- When a requirement is unclear, check `docs/PRD.md` and `docs/DECISIONS.md`. If it is still unclear, ask instead of guessing.
- Record any decision that changes scope in `docs/DECISIONS.md`.
- Spell out acronyms the first time they appear in docs you write.

## Commands

```bash
npm run dev      # local app
npm run build    # type check and production build
npm run test     # unit tests
npm run lint
```
