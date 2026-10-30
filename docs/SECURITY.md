# Security

This app handles money and personal data. Treat every item here as required unless it says otherwise.

## 1. Principles

1. The browser is untrusted. Anything in the app (including every `VITE_` variable) can be read and changed by anyone.
2. The database decides. Limits, opening hours, amounts and access rules are enforced in the database, never only on the screen.
3. A customer can reach only their own data, through a few database functions.
4. Secrets live only on the server and in the hosting settings. Never in the code, the repo, chat messages or logs.
5. Collect the least data possible. There is no email address in this app.

## 2. Threats and answers

| Threat | Answer |
|---|---|
| Someone reads or edits other customers' data with the public key | Row level security on every table, no customer policies, customers use functions only (`docs/SCHEMA.md`) |
| Someone signs up with another person's number | The code must arrive from that number on WhatsApp. Webhook messages are checked with Meta's signature (section 3) |
| Someone fakes webhook messages to pass the phone check | Verify `X-Hub-Signature-256` on every webhook request. Without this, the phone check is worthless |
| Someone calls staff or admin endpoints directly | Require a valid staff login token on every one (section 3) |
| A customer changes amounts in the browser | The database recalculates everything in `create_transaction` and compares the rate |
| Two requests pass the company limit together | Advisory lock inside `create_transaction` |
| A customer opens another customer's receipt | Private bucket, own-folder policy, 60 second signed links |
| Guessing passwords | Supabase Auth rate limits and a generic error message |
| Spamming the code endpoint | Rate limit per IP and per phone number on the server |
| A stolen phone session | Log out clears the session. Passwords can be reset with the phone check |
| The service key leaks | Key only in Render settings. Rotate it in Supabase if exposed, then update Render |

## 3. Required before any real customer (P0)

These exist in the current server and must be fixed. They are listed first because the customer app depends on them.

1. **Lock the database.** Run `lock-database.sql`. Test the dashboard and the bot. Keep `unlock-database.sql` ready.
2. **Verify the webhook signature.** In the server's `POST /webhook`, read the raw body, compute `sha256=` HMAC with the Meta app secret (`META_APP_SECRET`), and compare it with the `X-Hub-Signature-256` header using a timing-safe comparison. Reject with `401` when it does not match. Express needs the raw body for this (use `express.json({ verify })` to keep it).
3. **Protect staff and admin endpoints.** The version of `index.js` reviewed for this plan had no check on `/staff/*`, `/admin/*` or `/send-template`, and the dashboard sent an `x-api-key` header that the server never checked. A key inside a browser app is public anyway, so it is not protection. Replace it with: the dashboard sends `Authorization: Bearer <Supabase access token>`; the server calls `supabase.auth.getUser(token)`, loads the matching `staff_users` row, requires `is_active`, and requires `role = ADMIN` for `/admin/*`. **Check the live server first.** If the endpoints are open today, anyone who knows the Render address could create an admin account through `/admin/create-staff`.
4. **Restrict CORS.** The server used `cors({ origin: '*' })`. Allow only `https://app.jacerockcapital.com`, `https://onetap.jacerockcapital.com` and the local dev address.
5. **Keep the service role key on the server only.** `src/config/supabase.js` uses it (see the lock step). It is never in this repo or any `VITE_` variable.
6. **Private receipts.** Customer receipts go to the private `customer-receipts` bucket. The old public `receipts` bucket stays for the bot only.
7. **Check storage policies.** Run the storage policy query in `docs/SCHEMA.md` section 1. Remove any policy that lets anonymous users upload or list files.

## 4. Phone verification details

- Code: 6 digits from a cryptographically secure random source (`crypto.randomInt`), never `Math.random`.
- Store only a hash of the code and of the client token (SHA-256 with a server secret `VERIFICATION_PEPPER` added). Never log the code.
- Expiry: 10 minutes. One use. At most 5 wrong attempts per number per verification.
- The client token is a long random value returned once to the browser that started the verification. Status and register calls need it, so another person who learns the verification id cannot finish the sign-up.
- Compare the sender number from the webhook with the stored phone as digits only. Never trust a number typed in the message text.
- The webhook never replies to a verification message, so it cannot be used to message people.
- Rate limits: 5 starts per hour per phone number, and 20 per hour per IP address.

## 5. Accounts and sessions

- Password: minimum 8 characters. Do not log or store it anywhere except Supabase Auth.
- Phone login uses Supabase Auth (spike S1 decides the exact method). Whichever method is used, no real email address is collected.
- Turn on Supabase Auth rate limits and set a short access token lifetime with refresh tokens.
- Sign out removes the session on the device.

## 6. Data protection

| Data | Where | Who can read it |
|---|---|---|
| Full name, phone number | `customers` | The customer, staff and admin |
| Receipts and generated receipts | `customer-receipts` (private) | The customer, staff and admin |
| Settlement details | `transactions` | The customer, staff and admin |
| Verification codes | `phone_verifications` (hashed) | Server only |
| Passwords | Supabase Auth (hashed) | Nobody |

- No email address, no ID document images, no card numbers are collected.
- Keep phone numbers and names out of server logs where you can. Log ids and references instead.
- Staff actions are written to `audit_logs`.
- The database plan must have daily backups (the free plan has none). Upgrade before launch.

## 7. Web app hygiene

- Send security headers from Vercel: `Strict-Transport-Security`, `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`, and a content security policy that allows only the app, Supabase and the server.
- The service worker caches the app shell only. Never cache Supabase or server responses.
- Escape all user text on screen (React does this by default; never use `dangerouslySetInnerHTML`).
- Validate on both sides: phone numbers, account numbers (10 digits for Nigeria), file type and size.

## 8. Pre-launch checklist

- [ ] Lock script run, bot and dashboard tested after it
- [ ] Webhook signature check live, and a fake request is rejected
- [ ] Staff and admin endpoints reject calls without a valid staff token
- [ ] CORS limited to the three allowed origins
- [ ] Service role key only in Render
- [ ] Anon key test: reading `transactions` and `customers` returns nothing
- [ ] Two test customers cannot see each other's transactions or receipts
- [ ] A code sent from the wrong phone does not verify
- [ ] Closed hours block `create_transaction` when called directly
- [ ] Limits block `create_transaction` when called directly
- [ ] Storage: another customer's receipt link fails
- [ ] Supabase plan upgraded, backups on
- [ ] Default and company limits set by the client
- [ ] `docs/TESTING.md` acceptance script passed

## 9. If something goes wrong

1. App or dashboard stops working after a database change: run `unlock-database.sql`, then find the missing policy.
2. A key is exposed: rotate it in Supabase (service role) or Meta (app secret), update Render, redeploy.
3. Suspicious transactions: set an outage in the dashboard. It closes the customer app immediately.
4. A customer reports unknown activity: reset their password through the phone check and review their `audit_logs`.
