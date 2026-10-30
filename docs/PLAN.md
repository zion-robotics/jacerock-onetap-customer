# Project Plan

Duration: 3 weeks from the first payment and the client items below being in place. Each phase is tested and reviewed with the client before the next one starts. Two rounds of revisions per phase are included.

## 1. Before day 1

| Item | Owner |
|---|---|
| First payment received | Client |
| Access to the Meta WhatsApp Business account (webhook settings and app secret) | Client |
| Access to the domain settings to add `onetap.jacerockcapital.com` | Client |
| Supabase project upgraded from the free plan before launch | Client |
| Starting limits: default per customer and company daily limit | Client |
| New repo created, docs folder committed | Zion |

## 2. Phase 1: Foundation (week 1)

**Goal:** the database is locked, and a customer can create an account and log in.

- [ ] Run `lock-database.sql` (row level security on all tables). Check the bot and the staff dashboard still work. Keep `unlock-database.sql` ready.
- [ ] Spike S1 (day 1): decide how customers log in with a phone number in Supabase Auth. Option A: Supabase phone login with password, with the phone provider enabled and no SMS sent. Option B: a hidden internal login identifier derived from the phone number, with no email collected. Record the result in `docs/DECISIONS.md`.
- [ ] Spike S2 (day 1): check the real column types and the real `country` values (queries in `docs/SCHEMA.md` section 1).
- [ ] Migrations 002 to 004: new columns and tables, `phone_verifications`, `app_settings`.
- [ ] Server (existing repo): webhook signature check, verification intake, endpoints `/customer/verify/start`, `/customer/verify/status`, `/customer/register`, `/customer/password/reset`.
- [ ] Customer app: project setup, routing, sign-up, verify screen, login, forgot password, profile, logout.
- [ ] Row level security policies for customers (own data only) and tests for them.

**Milestone 1:** a real customer signs up with a WhatsApp code and logs in. A second customer cannot read the first one's data. The bot and the dashboard still work.

## 3. Phase 2: Customer platform (week 2)

**Goal:** a customer can submit a transaction with receipts.

- [ ] Functions: `get_availability`, `get_quote`, `get_payment_options`, `get_my_limits`, `create_transaction`.
- [ ] Private receipts bucket and its storage policies.
- [ ] Customer app: home, send money flow (calculator, settlement details, payment and receipts, submit), status page, history, availability banner, limits display, Customer Care button.
- [ ] PWA setup (manifest, icons, install prompt).
- [ ] Migration for the `my_transactions` view with customer-facing statuses.

**Milestone 2:** the client submits a GMD to NGN and an NGN to GMD transaction with two receipts, and sees them in the staff dashboard.

## 4. Phase 3: Staff, Customer Care and launch (week 3)

**Goal:** staff can finish the whole process, and the system goes live.

- [ ] Staff dashboard: web transactions with all receipts, settlement check, fail with reason, limits page, complete creates the customer receipt.
- [ ] Server: `POST /staff/generate-receipt`, protected staff endpoints, restricted CORS.
- [ ] Bot: Customer Care prompts, stop the transaction flow.
- [ ] Full test pass (`docs/TESTING.md`), security checklist (`docs/SECURITY.md` section 8).
- [ ] Staff walkthrough session with the client's team.
- [ ] Production setup: domain, environment variables, database plan upgrade, backups on.
- [ ] Launch with a small group of real customers, then everyone.

**Milestone 3:** the client signs off the acceptance script and the platform is live.

## 5. Risks

| Risk | Impact | Plan |
|---|---|---|
| The phone login method in Supabase does not behave as hoped | Delays sign-up | Spike S1 on day 1, with the hidden-identifier fallback ready |
| Someone fakes webhook messages to verify a number they do not own | Account takeover | Check Meta's signature on every webhook request. This is required before launch |
| Staff endpoints on the server are open to anyone who knows the address | Anyone could create admin accounts or send messages | Require a valid staff login on every staff and admin endpoint (see `docs/SECURITY.md`) |
| Locking the database breaks the bot or the dashboard | Outage | Server uses the service key first, test after the lock, undo script ready |
| Client answers or access arrive late | Timeline moves day for day | Agreed in the proposal |
| WhatsApp charges for bot replies continue | Client cost | Bot limited to Customer Care, replies short |
| Free Supabase plan has no daily backups | Data loss risk with real customers | Upgrade before launch |

## 6. Definition of done

A feature is done when it passes its acceptance criteria in `docs/PRD.md`, has tests in `docs/TESTING.md`, is deployed to the staging address, has been checked on a real low-end Android phone, and has no open security items.

## 7. Communication

- A short demo at the end of each phase.
- The client answers questions within 48 hours.
- Changes to scope are written in `docs/DECISIONS.md` before they are built.
