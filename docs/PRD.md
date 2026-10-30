# Product Requirements Document (PRD)

Version 1.0. Date: 24 October 2026. Status: approved by the client.

## 1. Background

The first platform used a WhatsApp bot for every step of a transaction. WhatsApp message charges made that too expensive. The client wants customers to transact on a website, like Opay, and to keep WhatsApp only for Customer Care and for confirming phone numbers.

## 2. Goals and non-goals

**Goals**
- Customers complete a transaction on a website with no WhatsApp message charges.
- Simple enough for customers with low digital literacy: few steps, large numbers, plain words.
- Staff and admin keep control: approve, hold, fail with a reason, daily limits, opening hours.
- A customer can only ever see their own data.

**Non-goals for version 1**
- The Nomba name check on settlement accounts (on hold until the access problem is solved).
- Text message (SMS) codes, email addresses, push notifications.
- Currency pairs other than GMD to NGN and NGN to GMD.
- Native apps for the Apple or Google app stores.
- Moving old WhatsApp history into the new accounts.
- Customers cancelling or editing a submitted transaction.

## 3. Users

| User | Description |
|---|---|
| Customer | Signs up with full name and phone number. Uses the customer app only. |
| Staff (role `AGENT`) | Processes transactions in the staff dashboard. |
| Admin (role `ADMIN`) | Everything staff can do, plus limits, business hours, staff accounts and rates. |

## 4. Functional requirements

Each requirement has acceptance criteria (AC). A requirement is done when all its AC pass.

### Customer app

**FR-1 Sign-up**
- Fields: full name (as on ID), country, phone number, password (minimum 8 characters), confirm password. There is no email field anywhere.
- After the form, the app shows a 6-digit code and an "Open WhatsApp" button that opens WhatsApp with a message pre-filled with the code, addressed to the business WhatsApp number.
- The account is created only when the server receives the same code from the same phone number on WhatsApp. The app waits and moves on by itself. No WhatsApp reply is sent.
- The code is single-use, expires after 10 minutes, and is tied to that sign-up attempt. A new code can be requested (at most 5 per hour per number).
- AC1: a code sent from a different phone number never verifies the account.
- AC2: an expired or already used code fails.
- AC3: a number that already has a web account is told to log in.
- AC4: a number that used the old bot can sign up, and no old history is shown.

**FR-2 Login and logout**
- Login with phone number and password. Error messages never say which of the two was wrong.
- The session stays on the device until the customer logs out.

**FR-3 Forgot password**
- Same verification as FR-1 (purpose `RESET`), then the customer sets a new password.

**FR-4 Opening hours gate**
- While the business is closed, the customer cannot start a new transaction. They can still log in, see their history and receipts, and see a banner with the reason.
- The rule is enforced in the database function that creates a transaction, not only on the screen.
- Closed means: an outage is active, or today is a holiday of type `CLOSED`, or the time is outside today's opening hours (Gambia time). Admin opens the platform by editing business hours in the staff dashboard.
- Before opening time, the message says the business has not started yet and gives the opening time. After closing time, the message is the admin's away message.
- AC: with business hours set to closed, `create_transaction` fails with `BUSINESS_CLOSED` even if called directly.

**FR-5 Calculator and rate**
- Only the pairs `GMD_NGN` and `NGN_GMD` (active in `exchange_rates`) are offered. The customer picks the direction.
- Two boxes side by side: "Send amount" and "Receive amount". The customer can type in either one and the other updates immediately.
- The rate is shown under the boxes in the form "1 GMD = 17.70 NGN". There is no page that lists all rates.
- If the rate changes between the quote and the submit, the submit fails with `RATE_CHANGED` and the screen refreshes with the new quote.
- AC: see the test values in `docs/TESTING.md` section 3.

**FR-6 Daily limits**
- A daily limit per customer and a daily limit for the whole company, both counted in GMD.
- The GMD amount of a transaction is the send amount for `GMD_NGN` and the receive amount for `NGN_GMD`.
- Failed transactions do not count. The day runs from 00:00 to 24:00 Gambia time.
- Admin sets the default customer limit and the company limit, and can give one customer a different limit by searching their phone number. A customer's own limit wins over the default.
- The customer sees how much they can still send today. They are never shown the company limit; they only see a general message if it blocks them.
- AC: a transaction that would pass the customer limit fails with `LIMIT_CUSTOMER`. One that would pass the company limit fails with `LIMIT_COMPANY`. Two requests sent at the same moment cannot both pass the company limit.

**FR-7 Settlement details (where the customer receives money)**
- `GMD_NGN` (customer receives NGN): Nigerian account number (10 digits), account name, bank name.
- `NGN_GMD` (customer receives GMD): the customer chooses one of: bank account (account name, account number, bank name), mobile wallet (wallet type APS or WAVE, wallet name, wallet number), or cash pick-up at the office (the address is shown, no extra fields).
- Cash pick-up is hidden when today is a holiday of type `ONLINE_ONLY`.

**FR-8 Payment and receipts**
- The customer chooses bank transfer or mobile wallet. The app shows the active account or wallet details for the currency they are sending (GMD accounts in The Gambia, NGN accounts in Nigeria).
- The customer uploads between 1 and 5 receipts, for example one from Wave and one from a bank transfer. Allowed types: JPG, PNG, WEBP, PDF. Each file is at most 10 MB.
- All receipts reach the staff dashboard with the transaction.

**FR-9 Submit**
- One submit creates the transaction with its receipts and settlement details, and shows "Processing" with the reference (format `JAC-YYYYMMDD-NNNNNN`).

**FR-10 Status and history**
- Customer-facing statuses: **Processing**, **Completed**, **Failed** (with the reason). Internal statuses are mapped in `docs/FLOWS.md`. A transaction on hold shows as Processing.
- History lists the customer's own web transactions, newest first.

**FR-11 Receipt**
- When staff mark a transaction completed, the AfrikBerry One-Tap receipt image is created and appears on the transaction page, where the customer can view and download it.

**FR-12 Customer Care**
- A "Contact Customer Care" button opens WhatsApp to the business number with a message that includes the transaction reference when there is one. Failed transactions show this button.

**FR-13 Profile**
- Shows name and phone number (read only), a link to change the password through FR-3, and logout.

### Staff and admin dashboard

- **S-1** Web transactions appear in the existing queue, with every receipt, the settlement details and the source (WEB).
- **S-2** Approve, Hold and Mark as Completed keep working and update the customer's page. Hold shows as Processing to the customer.
- **S-3** Staff mark the settlement details as correct or incorrect.
- **S-4** Staff can fail a transaction with a reason: incorrect settlement details, or settlement account downtime. The reason list can be extended.
- **S-5** Marking a web transaction completed creates the receipt image for the customer app (not a WhatsApp message).
- **S-6** Admin page for limits: default customer limit, company limit, per-customer limit by phone number.
- **S-7** Business hours, holidays and the outage switch now also control the customer app.

### Server and bot

- **B-1** The webhook recognises a verification message and handles it before the bot flow, and sends no reply.
- **B-2** The bot becomes Customer Care with short prompts. Starting set: help with a failed transaction, a payment problem, account help, talk to an agent.
- **B-3** The bot no longer runs the transaction flow.

## 5. Business rules

| Rule | Value |
|---|---|
| Time zone | Gambia time (GMT, UTC+0) for opening hours, holidays and the daily limit day |
| Rate meaning | Always "NGN per 1 GMD" |
| `GMD_NGN` | receive (NGN) = send (GMD) x rate. send = receive / rate |
| `NGN_GMD` | receive (GMD) = send (NGN) / rate. send = receive x rate |
| Rounding | Half up to 2 decimals. The amount the customer typed is kept exactly; the other one is calculated |
| Amount | Greater than 0, and no more than the remaining limits |
| Receipts | 1 to 5 per transaction |
| Reference | `JAC-YYYYMMDD-NNNNNN` from `generate_transaction_reference()` (kept as it is) |
| Phone format | Digits only, with country code, no plus (for example `2207012345`). This matches the sender number WhatsApp reports |
| Countries at sign-up | Gambia (+220) and Nigeria (+234) |
| Blacklisted customers | Cannot create a transaction |

## 6. Non-functional requirements

- Mobile first. Works on small Android phones and slow connections. The first screen should be usable in under 3 seconds on a typical mobile connection.
- Text at least 16 px, inputs at least 18 px, buttons at least 48 px high.
- Installable as a PWA. The service worker caches the app shell only, never API responses or private files.
- Customers can only read their own data (see `docs/SECURITY.md`).
- Works in the current versions of Chrome and Safari, and Samsung Internet.

## 7. Open items

| Item | Status |
|---|---|
| Third failure reason "payment not verified" for staff Reject | Suggested to the client, not yet answered |
| Receipt limit of 5 and the daily limit counting rule | Assumptions, client may change them |
| Starting values for the default customer limit and the company limit | Client to provide before launch |
| Exact `country` values in `bank_accounts` and `mobile_wallets` | Check in the database (see `docs/SCHEMA.md`) |
| Phone authentication method in Supabase | Decided by spike S1 in week 1 (see `docs/PLAN.md`) |
