# AfrikBerry One-Tap Customer App

The customer web app for **AfrikBerry One-Tap**, a cross-border payment product between GMD (Gambian dalasi) and NGN (Nigerian naira). Customers sign up, check the rate in a calculator, pay, upload receipts and follow their transaction. Staff and admin process the transactions in the existing staff dashboard.

**Brand rule:** customers only ever see the name **AfrikBerry One-Tap**. Do not show "Jacerock" or the parent company name anywhere in the app, in receipts, in page titles, in error messages or in the PWA (progressive web app) manifest.

## How the pieces fit together

| Piece | What it is | Where it lives | Hosted at |
|---|---|---|---|
| Customer app (this repo) | React web app, installable as a PWA | this repo | onetap.jacerockcapital.com (Vercel) |
| Staff and admin dashboard | React dashboard | `staff-dashboard/` folder in the `jacerock-webhook` repo | app.jacerockcapital.com |
| Server and WhatsApp bot | Node.js and Express | root `index.js` and `src/` in the `jacerock-webhook` repo | Render |
| Database, auth and file storage | One shared Supabase project (`jacerockcapital`) | Supabase | Supabase |

All three apps share the same Supabase project. This repo owns the new database objects for the customer app (see `supabase/migrations/`), and the other two repos get small changes (see `docs/EXISTING_REPOS.md`).

## Documents

| File | Read it for |
|---|---|
| `docs/PRD.md` | What we are building: requirements, business rules, acceptance criteria |
| `docs/PLAN.md` | The 3-week plan, tasks, risks and milestones |
| `docs/BUILD.md` | How to set up and build it, step by step, with checks |
| `docs/SCHEMA.md` | Database changes: tables, columns, functions, access rules, storage |
| `docs/FLOWS.md` | Every user flow, the status model and the exact on-screen messages |
| `docs/API.md` | The database functions and server endpoints the app calls |
| `docs/SECURITY.md` | Threats, required protections and the pre-launch checklist |
| `docs/TESTING.md` | Test plan, test values and the acceptance script |
| `docs/UI.md` | Brand, layout rules, screens and copy rules |
| `docs/EXISTING_REPOS.md` | Changes needed in the staff dashboard and the server and bot |
| `docs/DECISIONS.md` | Decisions the client has made, and assumptions he can change |
| `.github/copilot-instructions.md` | Rules for AI coding assistants working in this repo |

## Stack

React 18, TypeScript, Vite, Tailwind CSS, React Router, `@supabase/supabase-js`, `decimal.js` for money display, `vite-plugin-pwa`. Backend logic lives in Postgres functions (Supabase) and in the existing Render server.

## Glossary

| Term | Meaning |
|---|---|
| PRD | Product requirements document |
| GMD / NGN | Gambian dalasi / Nigerian naira |
| GMT | Greenwich Mean Time. The Gambia uses GMT all year, so "Gambia time" and GMT are the same here |
| RLS | Row level security: database rules that decide which rows each user may read or change |
| RPC | Remote procedure call: calling a database function from the app |
| PWA | Progressive web app: a website that can be installed on a phone like an app |
| JWT | JSON web token: the signed login token Supabase gives a signed-in user |
| CORS | Cross-origin resource sharing: which websites may call the server |
| UAT | User acceptance testing: the client tests the finished work |
| KYC | Know your customer: the customer's identity details |
"# jacerock-onetap-customer" 
