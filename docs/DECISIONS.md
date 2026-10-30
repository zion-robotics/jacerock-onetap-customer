# Decisions and Assumptions

## 1. Decisions made by the client (October 2026)

| ID | Decision |
|---|---|
| D-01 | Customers transact on a website. WhatsApp is no longer used for transactions, because of Meta's message charges |
| D-02 | Customers only ever see the name AfrikBerry One-Tap. The JR logo stays. The reference prefix `JAC` stays. The bot's daily rates header is unchanged |
| D-03 | Addresses: `onetap.jacerockcapital.com` for customers, `app.jacerockcapital.com` for staff and admin. Both accepted |
| D-04 | Sign-up with full name and phone number only, then a password. Login with phone number and password. No email, because of the data protection policy. Email may come later as a customer's own choice |
| D-05 | Phone check: the customer sends a code to the business WhatsApp number (the same number the bot uses). No reply is sent |
| D-06 | No exchange rate board. The rate shows under the amount when the customer starts a payment |
| D-07 | Only `GMD_NGN` and `NGN_GMD` at first. Other pairs are switched on manually later |
| D-08 | The calculator shows both amounts, in both directions (examples: 1,000 GMD = 17,700 NGN at 17.7, and 20,000 / 18.1 = 1,104.97 GMD) |
| D-09 | Daily limits apply to each customer and to the company. Admin can change them for all customers or one customer by phone number |
| D-10 | When the business is closed, customers cannot start a transaction. Admin changes the times in the dashboard to open it |
| D-11 | Customers can upload more than one receipt (for example Wave and a bank transfer), and all reach the staff dashboard |
| D-12 | The customer sees: Processing after submit, Completed with the receipt when approved, Failed when there is a problem. A transaction on hold stays Processing |
| D-13 | Failure reasons shown to the customer: incorrect settlement details, settlement account downtime |
| D-14 | Failed transactions are handled through the Customer Care bot |
| D-15 | The bot stays on and becomes Customer Care with short prompts. The same number also receives the verification codes |
| D-16 | All customers sign up fresh. Old WhatsApp history is not moved. Old bot customers can still use the bot for complaints and codes |
| D-17 | Dalasi payouts: bank account, APS, Wave or cash pick-up |
| D-18 | Staff check settlement details by hand and mark them correct or incorrect. The Nomba name check is on hold |
| D-19 | Payment display (bank transfer or mobile wallet) stays as approved |

## 2. Assumptions (the client may change these)

| ID | Assumption |
|---|---|
| A-01 | The daily limit counts the GMD side of every transaction, resets at 00:00 Gambia time, and failed transactions do not count |
| A-02 | A customer can upload up to 5 receipts per transaction (JPG, PNG, WEBP or PDF, 10 MB each) |
| A-03 | The customer can type either amount in the calculator |
| A-04 | A null (not yet set) limit blocks new transactions, so the platform cannot open without limits |
| A-05 | Countries at sign-up: Gambia (+220) and Nigeria (+234) |
| A-06 | Password: at least 8 characters |
| A-07 | Customers cannot cancel a submitted transaction in version 1 |
| A-08 | A third failure reason "payment not verified" may be added (suggested, no answer yet) |

## 3. Technical decisions

| ID | Decision | Date |
|---|---|---|
| T-01 | Customer logic (quotes, limits, hours, create) lives in Postgres functions. Customers have no table access | 24 Oct 2026 |
| T-02 | Verification endpoints live on the existing Render server, not as public database functions, so they can be rate limited | 24 Oct 2026 |
| T-03 | Receipts for web customers go to a private bucket with signed links | 24 Oct 2026 |
| T-04 | Spike S1 result (phone login method in Supabase Auth): | to fill in |
| T-05 | Spike S2 result (real column types and country values): | to fill in |

## 3. Change log

Add a line here for every change to scope after 24 October 2026: date, what changed, who agreed.
