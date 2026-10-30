# API Reference

Two kinds of calls: database functions (RPC) used by the customer app through Supabase, and server endpoints on Render. The app never writes to tables directly.

## 1. Database functions (Supabase RPC)

Called with `supabase.rpc('name', args)`. All need a signed-in customer unless noted. Money values are numbers with up to 2 decimals.

### `get_availability()`
Returns:
```json
{ "open": false, "reason": "BEFORE_OPENING", "message": "We have not started business for today yet.", "opens_at": "09:30", "cash_pickup_available": true }
```
`reason` is one of `OPEN`, `OUTAGE`, `HOLIDAY_CLOSED`, `SCHEDULED_CLOSED`, `BEFORE_OPENING`, `AFTER_CLOSING`. `opens_at` is Gambia time or null.

### `get_quote(p_pair text, p_amount numeric, p_side text)`
`p_side` is `SEND` or `RECEIVE`. Returns:
```json
{ "pair": "GMD_NGN", "rate": 17.7, "send_currency": "GMD", "receive_currency": "NGN",
  "send_amount": 1000.00, "receive_amount": 17700.00, "entry_side": "SEND" }
```
Errors: `PAIR_DISABLED`, `INVALID_AMOUNT`.

### `get_payment_options(p_pair text)`
```json
{ "currency": "GMD",
  "banks":   [ { "id": "...", "bank_name": "...", "account_name": "...", "account_number": "..." } ],
  "wallets": [ { "id": "...", "wallet_name": "...", "account_name": "...", "account_number": "..." } ] }
```

### `get_my_limits()`
```json
{ "limit_gmd": 100000.00, "used_today_gmd": 15000.00, "remaining_gmd": 85000.00 }
```
Never returns the company limit.

### `create_transaction(p jsonb)`
Input:
```json
{
  "pair": "NGN_GMD",
  "entry_side": "SEND",
  "amount": 20000,
  "expected_rate": 18.1,
  "settlement_method": "MOBILE_WALLET",
  "settlement_details": { "wallet_type": "WAVE", "wallet_name": "Fatou Ceesay", "wallet_number": "7012345" },
  "payment_method": "BANK_TRANSFER",
  "pay_to_id": "uuid of the bank account or wallet the customer paid into",
  "receipts": [
    { "path": "<auth uid>/uploads/<uuid>.jpg", "file_name": "receipt1.jpg", "mime_type": "image/jpeg", "size_bytes": 182034 }
  ]
}
```
Returns `{ "id": "...", "reference": "JAC-20261024-000012", "status": "PROCESSING" }`.

Errors: `NOT_A_CUSTOMER`, `CUSTOMER_BLOCKED`, `BUSINESS_CLOSED`, `PAIR_DISABLED`, `INVALID_AMOUNT`, `RATE_CHANGED`, `LIMIT_CUSTOMER`, `LIMIT_COMPANY`, `LIMIT_NOT_SET`, `SETTLEMENT_INVALID`, `RECEIPT_REQUIRED`, `TOO_MANY_RECEIPTS`, `RECEIPT_INVALID`.

### `list_my_transactions(p_limit int default 50, p_offset int default 0)`
Returns rows: `id, reference, currency_pair, send_amount, receive_amount, customer_status, failure_reason, created_at`. `customer_status` is `PROCESSING`, `COMPLETED` or `FAILED`.

### `get_my_transaction(p_id uuid)`
Returns the list fields plus: `exchange_rate`, `settlement_method`, `settlement_details`, `completed_at`, `failed_at`, `receipts` (array of `{ file_name, storage_path }`) and `receipt_image_path`. Fails with `NOT_FOUND` if the transaction is not the caller's.

### `get_my_profile()`
Returns `{ "full_name": "...", "phone": "2207012345" }`.

### Staff and admin functions
| Function | Who | Input | Notes |
|---|---|---|---|
| `staff_fail_transaction` | staff | `p_transaction_id uuid, p_reason text` | Reason must be one of the allowed values |
| `staff_set_settlement_check` | staff | `p_transaction_id uuid, p_result text` | `CORRECT` or `INCORRECT` |
| `admin_set_customer_limit` | admin | `p_phone text, p_limit numeric` | `null` clears the custom limit. Raises `CUSTOMER_NOT_FOUND` |

Default and company limits are edited in `app_settings` by admins.

## 2. Server endpoints (Render)

Base URL: `VITE_API_URL`. All bodies and responses are JSON. The customer endpoints are public but rate limited by IP and by phone number.

### `POST /customer/verify/start`
Body: `{ "phone": "2207012345", "purpose": "SIGNUP", "fullName": "Fatou Ceesay" }` (`fullName` only for sign-up).
Success `200`:
```json
{ "verificationId": "uuid", "clientToken": "random secret", "code": "482913",
  "expiresAt": "2026-10-24T10:40:00Z", "whatsappUrl": "https://wa.me/2207000000?text=Verify%20482913" }
```
Errors: `400 invalid_phone`, `409 account_exists` (sign-up), `404 no_account` (reset), `429 too_many_requests`.

### `GET /customer/verify/status?id=<verificationId>`
Header: `x-client-token: <clientToken>`. Returns `{ "status": "PENDING" | "VERIFIED" | "EXPIRED" | "USED" }`. Wrong token gives `404`.

### `POST /customer/register`
Body: `{ "verificationId", "clientToken", "password" }`. Needs the verification to be `VERIFIED` for purpose `SIGNUP`. Creates the login (see spike S1), creates or links the `customers` row (`registered_via = 'WEB'`, `phone_verified_at = now()`, `full_name` and `kyc_name` from the verification), marks the verification `USED`. Returns `201 { "ok": true }`. Errors: `400 weak_password`, `409 account_exists`, `410 expired`, `403 not_verified`.

### `POST /customer/password/reset`
Body: `{ "verificationId", "clientToken", "newPassword" }`. Needs `VERIFIED` and purpose `RESET`. Returns `200 { "ok": true }`.

### `POST /staff/generate-receipt`
Header: `Authorization: Bearer <staff access token>`. Body: `{ "transactionId": "uuid" }`. The server checks the token belongs to an active staff user, loads the transaction, draws the receipt (existing generator), saves it to `customer-receipts/<auth uid>/generated/<reference>.png`, sets `receipt_image_path`, writes an audit log. Returns `{ "path": "..." }`. Only for `source = 'WEB'` transactions.

### `POST /webhook` (existing, extended)
1. Verify the `X-Hub-Signature-256` header against the raw request body with the Meta app secret. Reject with `401` if it is missing or wrong.
2. If an inbound text message matches `^\s*verify\s+(\d{6})\s*$` (case-insensitive), call the verification handler with the sender number and the code and stop. Send no reply.
3. Otherwise continue to the Customer Care bot.

Verification handler rules: hash the code, find a `PENDING` row with the same hash and the same phone, not expired, fewer than 5 attempts; mark `VERIFIED`. On no match, add 1 to `attempts` of that phone's latest pending row, and do nothing else.

## 3. Error convention

Database functions raise an exception whose message is exactly the code, and whose detail is JSON with extra data:

```sql
raise exception 'LIMIT_CUSTOMER' using detail = '{"remaining": 85000.00}';
```

In the app, `error.message` is the code and `JSON.parse(error.details)` is the extra data. Map codes to the texts in `docs/FLOWS.md` section 7. Server endpoints return `{ "error": "<code>" }` with a matching HTTP status.
