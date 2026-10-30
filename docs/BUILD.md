# Build Guide

Follow the steps in order. Each step ends with a check. Do not move on until the check passes.

## 0. What you need

- Node.js 20 or newer, npm, Git
- Access to the Supabase project `jacerockcapital` (project URL and the **anon** key from Settings, API). The service role key is **never** used in this repo.
- Access to Vercel, to the domain settings for `jacerockcapital.com`, and to the Render service
- The business WhatsApp number in international format, digits only

## 1. Repo layout

```
.
├── README.md
├── .github/copilot-instructions.md
├── docs/                     all the documents
├── supabase/
│   └── migrations/           numbered SQL files, run in order
├── public/                   icons, manifest assets
├── src/
│   ├── main.tsx
│   ├── App.tsx               routes
│   ├── lib/                  supabase client, money helpers, formatters
│   ├── features/
│   │   ├── auth/             sign-up, verify, login, reset
│   │   ├── send/             calculator, settlement, payment, receipts
│   │   ├── transactions/     history and status
│   │   └── profile/
│   ├── components/           shared UI (Button, Input, AmountBox, Banner)
│   └── styles/
└── vite.config.ts
```

## 2. Create the project

```bash
npm create vite@latest . -- --template react-ts
npm install
npm install react-router-dom @supabase/supabase-js decimal.js
npm install -D tailwindcss postcss autoprefixer vite-plugin-pwa
npx tailwindcss init -p
```

Check: `npm run dev` opens a blank page on `http://localhost:5173` with no errors.

## 3. Environment variables

Create `.env.local` (never commit it) and `.env.example` (commit it, with empty values):

```
VITE_SUPABASE_URL=
VITE_SUPABASE_ANON_KEY=
VITE_API_URL=                      # the Render server, for /customer/* endpoints
VITE_BUSINESS_WHATSAPP_NUMBER=     # digits only, used for wa.me links
VITE_APP_NAME=AfrikBerry One-Tap
```

Only variables that start with `VITE_` reach the browser, so everything in this file must be safe to make public. The service role key, the verification secret and the Meta app secret live only on the server.

Check: `import.meta.env.VITE_SUPABASE_URL` prints in the browser console during `npm run dev`.

## 4. Database first (week 1)

1. Copy `lock-database.sql` and `unlock-database.sql` into `supabase/migrations/001_lock_database.sql` and `001_unlock_database.sql` (the unlock file is not run, it is the undo).
2. Run the discovery queries in `docs/SCHEMA.md` section 1 and write the answers into that file.
3. Write migrations `002` to `007` from `docs/SCHEMA.md` section 7, run them one at a time in the Supabase SQL Editor, and test each with the cases in `docs/TESTING.md` section 2.

Check: after migration 005, a customer account can read only its own rows and the anon key reads nothing (test queries in `docs/TESTING.md`).

## 5. Supabase client

`src/lib/supabase.ts` creates one client with the URL and anon key. Persist the session, auto-refresh the token.

Check: calling `supabase.auth.getSession()` returns without error.

## 6. Money helpers

`src/lib/money.ts` uses `decimal.js` with rounding mode "round half up" and exposes `quote({ pair, rate, side, amount })` returning `{ send, receive }`. The rules are in `docs/PRD.md` section 5. This code is for display only; the database function is the authority.

Check: the test values in `docs/TESTING.md` section 3 pass as unit tests (`npm run test`).

## 7. Auth screens

Build, in this order: sign-up form, verify screen (shows the code, the "Open WhatsApp" button, waits by polling `/customer/verify/status` every 2 seconds, expires after 10 minutes), login, forgot password, profile. Flow details are in `docs/FLOWS.md`.

Check: a full sign-up with a real phone works, and the same code sent from another phone does nothing.

## 8. Send money flow

Screens: amount (calculator and rate), settlement details, payment and receipts, review, submit, status. Receipts upload to the private bucket first, then `create_transaction` receives the file paths.

Check: a transaction appears in the staff dashboard queue with all receipts, the settlement details and the source WEB.

## 9. PWA

`vite-plugin-pwa` with: name and short name "AfrikBerry One-Tap", theme colour `#12306B`, background `#EDF1F6`, icons 192 and 512 (plus maskable). Cache only the app shell. Never cache requests to Supabase or the server.

Check: Chrome offers "Install app", and the installed app opens full screen.

## 10. Deploy

1. Push to GitHub, import the repo in Vercel, add the `VITE_` variables for Production and Preview.
2. In Vercel, Settings, Domains, add `onetap.jacerockcapital.com`. Add the DNS record Vercel shows (normally a CNAME) in the domain settings.
3. In Supabase, Authentication, URL configuration: set the Site URL to `https://onetap.jacerockcapital.com` and add the same address to the redirect list.
4. On Render, add the new address to the CORS allow-list.

Check: `https://onetap.jacerockcapital.com` loads over HTTPS and a sign-in works.

## 11. Scripts

```json
{
  "dev": "vite",
  "build": "tsc -b && vite build",
  "preview": "vite preview",
  "test": "vitest run",
  "lint": "eslint ."
}
```

`npm run build` must pass with no TypeScript errors before any merge.

## 12. Conventions

- TypeScript strict mode. No `any`.
- All money in the UI goes through `src/lib/money.ts`. Never use plain floating point arithmetic on amounts.
- All writes go through database functions or server endpoints. The app never inserts directly into tables.
- UI copy follows `docs/UI.md`: plain words, short sentences, no em dashes.
- One feature folder per area. Small components.
- Commit messages say what changed and why.
