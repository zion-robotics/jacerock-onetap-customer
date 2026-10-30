# Test Plan

## 1. Layers

| Layer | How | When |
|---|---|---|
| Unit tests | `vitest` for money helpers and formatters | Every change |
| Database tests | SQL scripts that act as each kind of user (section 2) | After every migration |
| Flow tests | Manual script on a real phone (section 4) | End of each phase |
| Acceptance | The client runs the script in section 7 | Before launch |

## 2. Database tests: who can do what

Pretend to be a user in the SQL Editor:

```sql
set role authenticated;
set request.jwt.claim.sub = '<the user id>';
-- run the query here
reset role;
```

For the signed-out case use `set role anon;`. Create at least: one admin, one agent, one deactivated staff user, two customers (A and B), one signed-in user who is not a customer.

| # | Who | Action | Expected |
|---|---|---|---|
| 1 | anon | `select` from `transactions`, `customers`, `exchange_rates`, `staff_users` | 0 rows or permission denied |
| 2 | anon | call `get_quote` or `create_transaction` | permission denied |
| 3 | customer A | `select * from transactions` | 0 rows or permission denied |
| 4 | customer A | `insert` or `update` on `transactions` | denied |
| 5 | customer A | `list_my_transactions()` | only A's web transactions |
| 6 | customer A | `get_my_transaction(<B's transaction id>)` | `NOT_FOUND` |
| 7 | customer A | `update customers set daily_limit_gmd = 1000000000` | denied |
| 8 | signed-in non-customer | `create_transaction(...)` | `NOT_A_CUSTOMER` |
| 9 | agent | read and update `transactions` | allowed |
| 10 | agent | change `customers.daily_limit_gmd` | error from the column guard |
| 11 | agent | insert or update `staff_users` | denied |
| 12 | admin | change `customers.daily_limit_gmd`, `app_settings`, `staff_users` | allowed |
| 13 | deactivated staff | any table | 0 rows, but can read their own `staff_users` row |
| 14 | service role | any table | allowed (the server) |
| 15 | any customer | storage: read a file under another customer's folder | denied |
| 16 | any customer | storage: upload outside their own folder | denied |
| 17 | any customer | direct `select` on `phone_verifications` | denied |

## 3. Calculator test values

Rounding is half up to 2 decimals. The typed amount is kept exactly.

| Pair | Rate | Typed | Result |
|---|---|---|---|
| `GMD_NGN` | 17.7 | send 1,000 GMD | receive 17,700.00 NGN |
| `GMD_NGN` | 17.7 | send 1 GMD | receive 17.70 NGN |
| `GMD_NGN` | 17.7 | send 0.5 GMD | receive 8.85 NGN |
| `GMD_NGN` | 17.7 | receive 500,000 NGN | send 28,248.59 GMD |
| `GMD_NGN` | 17.65 | send 0.1 GMD | receive 1.77 NGN (half up, not 1.76) |
| `NGN_GMD` | 18.1 | send 20,000 NGN | receive 1,104.97 GMD |
| `NGN_GMD` | 18.1 | receive 1,104.97 GMD | send 19,999.96 NGN |
| `NGN_GMD` | 18.1 | receive 1,000 GMD | send 18,100.00 NGN |
| `NGN_GMD` | 18.2 | send 1,350,000 NGN | receive 74,175.82 GMD |

The client app and `get_quote` must give identical results for every row. Also test: amount 0, a negative amount, empty input, more than 2 decimals typed, and a very large amount.

## 4. Flow tests (manual, on a low-end Android phone)

| # | Test | Expected |
|---|---|---|
| F1 | Sign up with a real phone, send the code from WhatsApp | Account created, logged in, no WhatsApp reply received |
| F2 | Sign up, send the code from another phone | Nothing happens |
| F3 | Wait 11 minutes, then send the code | Fails, "code expired", new code works |
| F4 | Sign up with a number that already has an account | Told to log in |
| F5 | Log in with a wrong password | Generic error |
| F6 | Forgot password with the phone check | New password works, old one does not |
| F7 | Calculator both directions | Matches section 3 |
| F8 | GMD to NGN with a Nigerian account | Submits, shows Processing with a reference |
| F9 | NGN to GMD with bank, then wallet (APS and WAVE), then cash pick-up | Each submits, staff see correct details |
| F10 | Two receipts (one image, one PDF) | Both appear in the staff dashboard |
| F11 | Sixth receipt | Blocked with the message |
| F12 | Receipt over 10 MB or a wrong file type | Blocked |
| F13 | Double tap on Submit | Only one transaction |
| F14 | Close the app mid-flow, open again | No half-made transaction |
| F15 | Install as an app | Installs, opens full screen, works after a restart |
| F16 | Slow connection (browser throttling "Slow 3G") | Screens stay usable, uploads show progress |

## 5. Rules tests

**Opening hours**
- Set today's hours to closed. `create_transaction` called directly returns `BUSINESS_CLOSED`.
- Set the outage switch. Same result, and the banner shows the outage message.
- Set a holiday of type `CLOSED`. Same result. Set `ONLINE_ONLY`: open, but cash pick-up is hidden.
- Before opening time: the message says "We have not started business for today yet" with the opening time.
- Admin edits the hours to open: a transaction goes through at once.

**Limits** (default customer limit 100,000 GMD for these examples)
- Submit 60,000 GMD, then 50,000 GMD: second fails with `LIMIT_CUSTOMER` and `remaining` 40,000.
- Fail the first transaction in the dashboard: the customer can submit 50,000 again.
- An `NGN_GMD` transaction counts its GMD receive amount, not the NGN amount.
- Give one customer a 500,000 limit by phone number: only that customer changes.
- Set the company limit low and submit from two customers at the same time with scripts: only the transactions that fit are accepted.
- Limits not set (null): `LIMIT_NOT_SET`.
- The day changes at 00:00 Gambia time: usage starts again.

**Verification** (server tests)
- Wrong code, right number: no effect, attempts go up.
- Right code, wrong number: no effect.
- Code reuse after `USED`: no effect.
- Webhook request without a valid signature: `401`.
- Message "verify 123456" text variations: `Verify 123456`, `verify123456`(should not match), extra spaces (should match).

## 6. Staff and admin tests

| # | Test | Expected |
|---|---|---|
| S1 | New web transaction appears in the queue | Source WEB, all receipts, settlement details |
| S2 | Hold | Customer still sees Processing |
| S3 | Approve, then Mark as Completed | Customer sees Completed and the receipt |
| S4 | Receipt content | Name, reference, amounts, details, AfrikBerry One-Tap only, no Jacerock |
| S5 | Fail with "incorrect settlement details" | Customer sees that text and the Customer Care button |
| S6 | Fail with "settlement account downtime" | Same, with that text |
| S7 | Mark settlement details correct or incorrect | Saved with the staff name and time |
| S8 | Admin limits page | Defaults, company limit and per-customer limit save and take effect |
| S9 | Agent opens the limits page | Cannot change anything |
| S10 | WEB transaction: WhatsApp template buttons | Hidden or disabled |
| S11 | Old WhatsApp transactions | Still open and work as before |

## 7. Acceptance script for the client

1. Create two customer accounts on two phones.
2. On each, make a GMD to NGN and an NGN to GMD transaction with at least two receipts on one of them.
3. In the staff dashboard, hold one, approve and complete one, and fail one with each reason.
4. Check what each customer sees after every step.
5. Close the business from the dashboard and try to start a transaction. Open it again.
6. Set a low customer limit and try to pass it. Raise it for one customer by phone number.
7. Use the Customer Care button on a failed transaction and check the bot answers.
8. Confirm the receipt looks right and carries only the AfrikBerry One-Tap name.

The client signs below when every step passes: ____________________  Date: ____________

## 8. Device and browser list

Test on: a low-end Android phone with Chrome, a mid-range Android phone with Samsung Internet, an iPhone with Safari, and a desktop browser. Use a screen width of 360 px as the smallest size.
