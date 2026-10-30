# Changes in the Existing Repositories

Both live in the `jacerock-webhook` repo: the server and bot at the root and in `src/`, and the dashboard in `staff-dashboard/`. Priority: **P0** before any real customer, **P1** needed for the launch.

## 1. Server and bot (root, `src/`)

| Priority | Task |
|---|---|
| P0 | Change `src/config/supabase.js` to use `SUPABASE_SERVICE_ROLE_KEY` (with a warning if it is missing). Do this before the database lock. File already prepared |
| P0 | Verify Meta's `X-Hub-Signature-256` on `POST /webhook` (needs the raw body and `META_APP_SECRET`) |
| P0 | Require a valid staff login token on `/staff/*`, `/send-template` and `/admin/*`. Only `ADMIN` for `/admin/*`. See `docs/SECURITY.md` section 3 |
| P0 | Restrict CORS to the three allowed origins |
| P1 | New module for verification codes: start, status, register, password reset (`docs/API.md`). Uses the service role key, `crypto.randomInt`, a hash secret `VERIFICATION_PEPPER`, and `express-rate-limit` |
| P1 | In the webhook, handle `Verify <6 digits>` messages before the bot flow, with no reply |
| P1 | `POST /staff/generate-receipt` for web transactions. Reuse `renderReceiptPng` and `buildReceiptData`. Save to the private bucket, set `receipt_image_path` |
| P1 | Rework the bot into Customer Care: short prompts for a failed transaction, a payment problem, account help, talk to an agent. Remove the transaction steps (`WELCOME` to `PROCESSING`). Keep the opening hours answer |
| P1 | Keep the old `/staff/send-receipt` route only for `source = 'WHATSAPP'` transactions |
| P1 | Leave the Nomba code in place but unused until the access problem is solved |
| P1 | Add the new environment variables to Render: `META_APP_SECRET`, `VERIFICATION_PEPPER`, `BUSINESS_WHATSAPP_NUMBER`, `ALLOWED_ORIGINS` |

Customers who used the old bot keep a `customers` row. Sign-up links the new login to that row. Old transactions stay `source = 'WHATSAPP'` and are never shown in the customer app.

## 2. Staff and admin dashboard (`staff-dashboard/`)

| Priority | Task |
|---|---|
| P0 | Send `Authorization: Bearer <access token>` on every server call (replace the `x-api-key` header). Remove `VITE_INTERNAL_API_KEY` |
| P1 | Transaction queue and detail: show the source, show all receipts from `transaction_receipts` using signed links from the `customer-receipts` bucket, and show the settlement details and the settlement check |
| P1 | Buttons: "Settlement details correct" and "Settlement details incorrect" (`staff_set_settlement_check`) |
| P1 | "Fail transaction" button with a reason menu (`staff_fail_transaction`): incorrect settlement details, settlement account downtime |
| P1 | Mark as Completed for `WEB` transactions calls `/staff/generate-receipt` and sends no WhatsApp message |
| P1 | Hide WhatsApp-only controls for `WEB` transactions (send template, take over conversation) |
| P1 | New admin page: Limits. Default customer limit and company limit (`app_settings`), and a phone number search that shows a customer's limit, today's usage and lets admin change it (`admin_set_customer_limit`) |
| P1 | Status badge for `FAILED`. Failure reason shown on the detail page |
| P1 | Business hours, holidays and outage pages stay as they are, and now also control the customer app. Add a note on those pages saying so |
| P1 | In the exchange rates page, keep only `GMD_NGN` and `NGN_GMD` active |
| P1 | Agents see the limits page read only |

## 3. Order of work

1. Server: service key change, then run the database lock.
2. Server: signature check, staff token check, CORS.
3. Database migrations for the customer app.
4. Server: verification module and endpoints.
5. Customer app.
6. Dashboard changes and the receipt endpoint.
7. Bot rework.
