# Database Schema

This is the target design for the shared Supabase project. The existing tables stay and keep working for the staff dashboard and the bot. Everything new is added, not replaced.

**Rule of this design:** customers never read or write tables directly. A customer's session (an authenticated user who is not staff) is allowed to call a small set of database functions, and nothing else. Staff and admin read and write through row level security (RLS) rules that check the `staff_users` table.

## 1. Check the real schema first (spike S2)

The existing column types below were inferred from the bot and dashboard code. Run these in the Supabase SQL Editor before writing any migration, and paste the results into the "Findings" section at the end of this file.

```sql
-- all columns
select table_name, column_name, data_type, is_nullable
from information_schema.columns where table_schema = 'public'
order by table_name, ordinal_position;

-- all constraints (look for a CHECK on transactions.status)
select conrelid::regclass as table_name, conname, pg_get_constraintdef(oid) as definition
from pg_constraint where connamespace = 'public'::regnamespace order by 1, 2;

-- duplicate phone numbers in customers (must be none before adding a unique index)
select whatsapp_number, count(*) from customers group by 1 having count(*) > 1;

-- the country values used for banks and wallets
select distinct country from bank_accounts;
select distinct country from mobile_wallets;

-- existing functions
select proname from pg_proc where pronamespace = 'public'::regnamespace order by 1;

-- storage buckets and storage policies
select id, name, public from storage.buckets;
select policyname, cmd, roles, qual, with_check from pg_policies where schemaname = 'storage';
```

Where the plan below says `uuid`, use the real type of `customers.id` and `transactions.id` from the first query.

## 2. Existing tables used by the customer app

| Table | Used for |
|---|---|
| `customers` | One row per phone number. Gets a link to the login account |
| `transactions` | The transaction itself. Gets new columns |
| `exchange_rates` | `currency_pair`, `rate`, `is_active`. Only `GMD_NGN` and `NGN_GMD` stay active |
| `bank_accounts` | Where customers pay (`bank_name`, `account_name`, `account_number`, `country`, `is_active`) |
| `mobile_wallets` | Where customers pay (`wallet_name`, `account_name`, `account_number`, `country`, `is_active`) |
| `business_hours` | `day_of_week`, `is_open`, `open_time`, `close_time` (Gambia time) |
| `holidays` | `holiday_date`, `name`, `type` (`CLOSED` or `ONLINE_ONLY`), `custom_message` |
| `business_status` | One row (`id = 1`): `away_message`, `outage_active`, `outage_message` |
| `audit_logs` | `transaction_id`, `customer_id`, `action`, `old_status`, `new_status`, `performed_by`, `notes` |
| `staff_users` | `id` (same as the login user id), `full_name`, `email`, `role` (`AGENT` or `ADMIN`), `is_active` |
| `conversation_sessions`, `messages`, `support_tickets` | Used by the bot and the chat page. Not touched by the customer app |

## 3. Changes to existing tables

```sql
-- customers: link a customer to their login, and a per-customer limit
alter table public.customers
  add column if not exists auth_user_id uuid unique references auth.users(id) on delete set null,
  add column if not exists daily_limit_gmd numeric(14,2),            -- null = use the default limit
  add column if not exists phone_verified_at timestamptz,
  add column if not exists registered_via text not null default 'WHATSAPP';  -- WHATSAPP or WEB

-- one customer row per phone number (only after the duplicate check in section 1 returns nothing)
create unique index if not exists customers_whatsapp_number_key on public.customers (whatsapp_number);

-- transactions: what the web app needs
alter table public.transactions
  add column if not exists source text not null default 'WHATSAPP',      -- WHATSAPP or WEB
  add column if not exists entry_side text,                               -- SEND or RECEIVE: which amount the customer typed
  add column if not exists send_amount numeric(14,2),
  add column if not exists receive_amount numeric(14,2),
  add column if not exists gmd_amount numeric(14,2),                      -- the GMD side, used for limits
  add column if not exists settlement_method text,                        -- BANK_ACCOUNT, MOBILE_WALLET or CASH_PICKUP
  add column if not exists settlement_details jsonb,
  add column if not exists settlement_check text not null default 'PENDING',  -- PENDING, CORRECT or INCORRECT
  add column if not exists settlement_checked_by text,
  add column if not exists settlement_checked_at timestamptz,
  add column if not exists selected_wallet_id uuid,                       -- wallet the customer paid into (selected_bank_id already exists for banks)
  add column if not exists failure_reason text,
  add column if not exists failed_at timestamptz,
  add column if not exists receipt_image_path text;                       -- the generated completion receipt

alter table public.transactions
  drop constraint if exists transactions_failure_reason_check,
  drop constraint if exists transactions_failed_needs_reason,
  drop constraint if exists transactions_settlement_check_values;

alter table public.transactions
  add constraint transactions_failure_reason_check
    check (failure_reason is null or failure_reason in
      ('INCORRECT_SETTLEMENT_DETAILS', 'SETTLEMENT_ACCOUNT_DOWNTIME', 'PAYMENT_NOT_VERIFIED')),
  add constraint transactions_failed_needs_reason
    check (status <> 'FAILED' or failure_reason is not null),
  add constraint transactions_settlement_check_values
    check (settlement_check in ('PENDING', 'CORRECT', 'INCORRECT'));

create index if not exists transactions_gmd_day_idx on public.transactions (created_at) where gmd_amount is not null;
```

Notes:
- `amount` (existing) is kept equal to `send_amount`, and `settlement_amount_ngn` (existing, badly named) is kept equal to `receive_amount`, because the staff dashboard and the receipt generator read them. New code reads `send_amount` and `receive_amount`.
- If `transactions.status` has a CHECK constraint (section 1), add `FAILED` to it.
- `PAYMENT_NOT_VERIFIED` is allowed in the constraint but only used if the client approves that reason.
- Existing transactions keep `source = 'WHATSAPP'`, so they never show in the customer app.

## 4. New tables

```sql
-- Verification codes for sign-up and password reset. Server only (service role).
create table if not exists public.phone_verifications (
  id uuid primary key default gen_random_uuid(),
  phone text not null,                                  -- digits with country code, no plus
  purpose text not null check (purpose in ('SIGNUP', 'RESET')),
  code_hash text not null,                              -- hash of the 6-digit code, never the code itself
  client_token_hash text not null,                      -- hash of a secret only the browser that started this knows
  full_name text,                                       -- signup only
  status text not null default 'PENDING' check (status in ('PENDING', 'VERIFIED', 'USED', 'EXPIRED')),
  attempts integer not null default 0,
  expires_at timestamptz not null,
  verified_at timestamptz,
  created_at timestamptz not null default now()
);
create index if not exists phone_verifications_phone_idx on public.phone_verifications (phone, status, created_at desc);

-- Several receipts per transaction
create table if not exists public.transaction_receipts (
  id uuid primary key default gen_random_uuid(),
  transaction_id uuid not null references public.transactions(id) on delete cascade,
  storage_path text not null,                           -- path in the customer-receipts bucket
  file_name text,
  mime_type text,
  size_bytes integer,
  uploaded_at timestamptz not null default now()
);
create index if not exists transaction_receipts_tx_idx on public.transaction_receipts (transaction_id);

-- Limits and other settings the admin can change
create table if not exists public.app_settings (
  key text primary key,
  value_numeric numeric(14,2),
  value_text text,
  updated_by text,
  updated_at timestamptz not null default now()
);
insert into public.app_settings (key, value_numeric) values
  ('default_customer_daily_limit_gmd', null),            -- client provides the starting value before launch
  ('company_daily_limit_gmd', null)                      -- null means no limit set yet: block new transactions until admin sets it
on conflict (key) do nothing;
```

A null limit means "not set". For safety, `create_transaction` treats a null default customer limit or a null company limit as a block (`LIMIT_NOT_SET`), so the platform cannot open with no limits by accident.

## 5. Statuses

Internal `transactions.status` values and who sets them:

| Status | Set by | Customer sees |
|---|---|---|
| `AWAITING_STAFF_APPROVAL` | `create_transaction` (web) | Processing |
| `PAYMENT_VERIFIED` | Staff: Approve | Processing |
| `UNDER_REVIEW` | Staff: Hold | Processing |
| `SETTLEMENT_COMPLETED` (and legacy `CLOSED`) | Staff: Mark as Completed | Completed |
| `FAILED` | Staff: Fail with a reason (`staff_fail_transaction`) | Failed, with the reason |
| `REJECTED` (legacy) | Old Reject button | Failed |

The failure reason texts shown to the customer:

| `failure_reason` | Text |
|---|---|
| `INCORRECT_SETTLEMENT_DETAILS` | Transaction failed due to incorrect settlement details. |
| `SETTLEMENT_ACCOUNT_DOWNTIME` | Transaction failed due to settlement account downtime. |
| `PAYMENT_NOT_VERIFIED` | Transaction failed because we could not verify your payment. (only if approved) |

## 6. Functions

All functions are `security definer`, set `search_path = public`, and grant `execute` to `authenticated` only (not `anon`). Customer functions begin by finding the caller's customer row with `auth_user_id = auth.uid()` and raise `NOT_A_CUSTOMER` if there is none. Errors are raised as exceptions whose message is the error code (see `docs/API.md` section 3).

| Function | Who | What it does |
|---|---|---|
| `is_active_staff()`, `is_admin()` | all policies | From `lock-database.sql` |
| `get_availability()` | customer | Applies the rules in section 6.1 and returns the result |
| `get_quote(pair, amount, side)` | customer | Returns the rate and both amounts (section 6.2) |
| `get_payment_options(pair)` | customer | Active banks and wallets for the currency the customer sends |
| `get_my_limits()` | customer | The customer's limit, used today and remaining |
| `create_transaction(payload)` | customer | The only way to create a transaction (section 6.3) |
| `list_my_transactions(limit, offset)` | customer | Own web transactions with the customer-facing status |
| `get_my_transaction(id)` | customer | One own transaction with receipts and the completion receipt path |
| `get_my_profile()` | customer | Name and phone |
| `staff_fail_transaction(id, reason)` | staff | Sets `FAILED`, reason, `failed_at`, writes the audit log |
| `staff_set_settlement_check(id, result)` | staff | Sets `CORRECT` or `INCORRECT` and who and when |
| `admin_set_customer_limit(phone, limit)` | admin | Sets or clears (`null`) one customer's limit |

### 6.1 Availability rules (same order as the bot)

1. `business_status.outage_active` is true: closed, reason `OUTAGE`, message `outage_message` or `away_message`.
2. A holiday exists for today (Gambia date) with type `CLOSED`: closed, reason `HOLIDAY_CLOSED`.
3. A holiday with type `ONLINE_ONLY`: open, and `cash_pickup_available = false`.
4. Today's `business_hours` row is missing or `is_open` is false: closed, reason `SCHEDULED_CLOSED`.
5. Current Gambia time is before `open_time`: closed, reason `BEFORE_OPENING`, with `opens_at`.
6. Current Gambia time is at or after `close_time`: closed, reason `AFTER_CLOSING`.
7. Otherwise open.

Use `now() at time zone 'UTC'` for "today" and "now". `create_transaction` calls the same logic.

### 6.2 Calculation

Inputs: pair, amount greater than 0, side (`SEND` or `RECEIVE`). The rate comes from `exchange_rates` where `currency_pair = pair` and `is_active`. Rules are in `docs/PRD.md` section 5. The typed amount is stored exactly, the other is rounded half up to 2 decimals (`round(x, 2)` in Postgres rounds half away from zero, which is the same for positive numbers).

### 6.3 `create_transaction(payload jsonb)`

Steps, in one database transaction:
1. Find the customer. Raise `CUSTOMER_BLOCKED` if `is_blacklisted`.
2. Take an advisory lock for the company limit: `pg_advisory_xact_lock(hashtext('onetap_company_limit'))`.
3. Raise `BUSINESS_CLOSED` if availability says closed.
4. Check the pair is active and one of the two allowed pairs, else `PAIR_DISABLED`.
5. Recalculate the amounts. If `expected_rate` differs from the current rate, raise `RATE_CHANGED` with the new rate in the detail.
6. Compute `gmd_amount`. Compare with the customer limit (own limit, else default) minus today's used amount: `LIMIT_CUSTOMER`. Compare with the company limit minus today's total: `LIMIT_COMPANY`. Null limits raise `LIMIT_NOT_SET`.
7. Validate the settlement details for the pair and method (`SETTLEMENT_INVALID`). Cash pick-up needs `cash_pickup_available`.
8. Validate the receipts: 1 to 5, each path starts with the caller's own folder `<auth uid>/uploads/`, and each object exists in the `customer-receipts` bucket. Otherwise `RECEIPT_REQUIRED` or `TOO_MANY_RECEIPTS` or `RECEIPT_INVALID`.
9. Insert the transaction: `reference` from `generate_transaction_reference()`, `source = 'WEB'`, `status = 'AWAITING_STAFF_APPROVAL'`, the amounts (and the old `amount` and `settlement_amount_ngn` columns), `settlement_account_name` in the labelled text format of section 6.4, `payment_method`, `selected_bank_id` or `selected_wallet_id`, `kyc_name` and `whatsapp_number` from the customer row, `receipt_uploaded_at = now()`.
10. Insert the `transaction_receipts` rows. Write an `audit_logs` row (`TRANSACTION_SUBMITTED`, performed by the phone number).
11. Return `{ id, reference, status: 'PROCESSING' }`.

"Used today" counts rows in `transactions` with `gmd_amount` not null, `created_at` on today's Gambia date, and `status` not in (`FAILED`, `REJECTED`, `CANCELLED`).

### 6.4 The labelled settlement text

The receipt generator on the server reads `settlement_account_name` as text, so `create_transaction` writes it in this exact format:

```
Account Name: <name>
Bank Name: <bank>
Account Number: <number>
```

```
Method: Mobile Wallet
Wallet Type: <APS or WAVE>
Wallet Name: <name>
Wallet Number: <number>
```

```
Method: Cash pick-up
Please visit our office to collect your settlement:
Mosque Complex, behind Marimoo Building, Africell Head Office, Kairaba Avenue, The Gambia
```

`settlement_details` holds the same information as JSON: `{ "account_name", "bank_name", "account_number" }`, `{ "wallet_type", "wallet_name", "wallet_number" }` or `{}` for cash.

## 7. Row level security

Turn RLS on for every table (done by `lock-database.sql`) and add these policies. Customers get no table policies at all.

| Table | Staff and admin | Customer | Anyone not signed in |
|---|---|---|---|
| All existing tables | Full access if `is_active_staff()` | None | None |
| `staff_users` | Read all if staff. Change only if `is_admin()`. Everyone can read their own row | None | None |
| `app_settings` | Read if staff. Change only if `is_admin()` | None | None |
| `transaction_receipts` | Full access if staff | None | None |
| `phone_verifications` | None | None | None (service role only) |

Column guard: a trigger on `customers` raises an error when `daily_limit_gmd` changes and `is_admin()` is false, because RLS cannot limit columns.

The server uses the service role key, which skips RLS, and so it can read `phone_verifications` and create users.

## 8. Storage

| Bucket | Public | Contents | Limits |
|---|---|---|---|
| `receipts` (existing) | Yes | Bot receipts and old generated receipts | Leave as it is |
| `customer-receipts` (new) | **No** | `<auth uid>/uploads/<uuid>.<ext>` customer receipts. `<auth uid>/generated/<reference>.png` completion receipts | 10 MB, JPG, PNG, WEBP, PDF |

Storage policies on `customer-receipts`:
- Customers: insert and select only when the first folder of the path equals their own `auth.uid()`. No update or delete.
- Staff: select on everything when `is_active_staff()`.
- The server (service role) writes the generated receipts.

The app shows files with short-lived signed links (60 seconds), never public links.

## 9. Migration order

| File | Content |
|---|---|
| `001_lock_database.sql` | The lock script (already prepared) |
| `002_customer_columns.sql` | Section 3 |
| `003_new_tables.sql` | Section 4 |
| `004_functions.sql` | Section 6 |
| `005_policies.sql` | Section 7 and the column guard trigger |
| `006_storage.sql` | Section 8 |
| `007_seed.sql` | Turn off all pairs except `GMD_NGN` and `NGN_GMD`. Set the starting limits when the client gives them |

Run each in the SQL Editor, check the matching tests in `docs/TESTING.md` section 2, then move on. Every migration file must be safe to run twice (`if not exists`, `drop policy if exists`).

## Findings (fill in after spike S2)

- Type of `customers.id` and `transactions.id`:
- Existing CHECK constraints on `transactions.status`:
- Duplicate phone numbers in `customers`:
- Country values in `bank_accounts` and `mobile_wallets`:
- Storage policies found:
