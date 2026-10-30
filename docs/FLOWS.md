# Flows, Statuses and On-Screen Messages

All times are Gambia time (GMT). All messages follow `docs/UI.md`: short sentences, plain words, no em dashes.

## 1. Sign-up and phone check

1. Customer fills in full name, country (Gambia +220 or Nigeria +234), phone number, password and confirm password.
2. App calls `POST /customer/verify/start` with `purpose = SIGNUP`. The server:
   - removes spaces and a leading 0, and builds the number as digits with country code (for example `2207012345`),
   - refuses if a web account already exists for it (`account_exists`),
   - creates a 6-digit random code, stores only its hash, sets a 10-minute expiry, and returns the code, a `clientToken` and a WhatsApp link.
3. App shows the **Verify** screen: the code in large digits, a green "Open WhatsApp" button, and the text "Send this code to confirm your phone number." The link is `https://wa.me/<business number>?text=Verify%20<code>`.
4. Customer taps the button, WhatsApp opens with "Verify 482913", and the customer presses send.
5. The server webhook receives the message. If the sender number and code match a pending verification that has not expired, it marks it `VERIFIED`. It sends no reply.
6. The app polls `GET /customer/verify/status` every 2 seconds. When it sees `VERIFIED`, it calls `POST /customer/register`, which creates the login and the customer row, and marks the verification `USED`.
7. App signs the customer in and opens the home screen.

Edge cases:

| Situation | What happens |
|---|---|
| Code expires | The screen shows "This code has expired." with a "Get a new code" button |
| Customer sends the code from a different number | Nothing happens. The screen keeps waiting |
| Customer has no WhatsApp on that number | The screen shows "You need WhatsApp on this number to sign up. Contact us for help." |
| More than 5 codes in one hour for one number | "Too many tries. Please wait an hour." |
| Customer used the old bot before | Sign-up works as normal. The customer row is reused, and old history stays hidden because it is not from the web |
| Customer closes the app and comes back | Starting again creates a new code. The old one expires |

## 2. Login

Phone number and password. On failure the message is always "The phone number or password is not right." After 5 failures in a row the screen says "Too many tries. Please wait a few minutes." (Supabase Auth limits apply as well.)

## 3. Forgot password

Same as sign-up, with `purpose = RESET` and no full name. A number with no account gets "We could not find an account with this number." After verification the customer types a new password (twice) and the server updates it with `POST /customer/password/reset`. The customer is then asked to log in.

## 4. Send money

```
Home -> Send money
   Step 1  Amount        direction, calculator, rate under the boxes
   Step 2  Receive       settlement details (depends on the direction)
   Step 3  Pay           choose bank transfer or wallet, see account details, upload 1 to 5 receipts
   Step 4  Review        everything on one screen, one big "Submit" button
   Done    Status page   "Processing"
```

Rules for every step:
- On entering the Send money flow, call `get_availability()`. If closed, show the closed screen instead (section 6).
- Show the remaining limit on step 1 ("You can send up to 85,000 GMD today").
- Going back never loses what was typed.
- Receipts are uploaded to the private bucket as soon as the customer picks them, with a progress bar. If an upload fails, show "This file did not upload. Try again." next to that file only.
- Submit calls `create_transaction` once. The button is disabled while waiting so a double tap cannot send twice.

Step 1 behaviour:
- Direction switch: GMD to NGN or NGN to GMD. Default: GMD to NGN.
- Two boxes, "Send amount" (with the currency) and "Receive amount". Typing in one updates the other after each key press (no waiting).
- Under the boxes: "Exchange rate: 1 GMD = 17.70 NGN".
- Numbers are shown with thousands separators (17,700).
- The Continue button stays disabled until the amount is greater than 0 and within the limit.

Step 2 behaviour:
- GMD to NGN: three fields, "Nigerian account number" (10 digits), "Account name" ("Beneficiary's full name"), "Bank name" (pick from a list, or type).
- NGN to GMD: three choices as large cards: Bank account, Mobile wallet, Cash pick-up. Bank account asks for account name, account number, bank name. Mobile wallet asks for wallet type (APS or WAVE), wallet name, wallet number. Cash pick-up shows the office address and no fields. Hide Cash pick-up when `cash_pickup_available` is false.

Step 3 behaviour:
- Two cards, "Bank transfer" and "Mobile wallet". After choosing, show the active accounts or wallets for the currency being sent. Each shows name, account name and number with a "Copy number" button.
- The upload area says "Add your payment receipt". After the first file it also shows "Add another receipt". Maximum 5. Each file shows a thumbnail or a PDF icon and a remove button.

## 5. Transaction status

Customer status comes from the internal status (see `docs/SCHEMA.md` section 5). The status page refreshes every 15 seconds while it is open.

| Customer status | Screen |
|---|---|
| Processing | Spinner or clock icon. "Your transaction is processing. We will update this page when it is done." Reference, amounts, receipts you uploaded |
| Completed | Green tick. "Transaction completed." The receipt image, with "Download receipt" |
| Failed | Red icon. The reason text. "Please contact Customer Care." with the Customer Care button |

Staff actions and their effect on the customer:

| Staff action | Internal status | Customer sees |
|---|---|---|
| (submit) | `AWAITING_STAFF_APPROVAL` | Processing |
| Approve | `PAYMENT_VERIFIED` | Processing |
| Hold | `UNDER_REVIEW` | Processing |
| Mark as Completed | `SETTLEMENT_COMPLETED` and the receipt is created | Completed and the receipt |
| Fail with a reason | `FAILED` and `failure_reason` | Failed and the reason |

## 6. Closed screens

| Reason | Message |
|---|---|
| `BEFORE_OPENING` | We have not started business for today yet. We open at {time} Gambia time. |
| `AFTER_CLOSING`, `SCHEDULED_CLOSED`, `HOLIDAY_CLOSED` | The admin's away message from `business_status`. If it is empty: "We are closed right now. Please come back during our business hours." |
| `OUTAGE` | The outage message. If empty: "We cannot take new transactions right now. Please try again later." |

A banner with the same text shows at the top of the home screen. The "Send money" button is disabled while closed. History and receipts stay available.

## 7. Error messages

| Code | What the customer sees |
|---|---|
| `BUSINESS_CLOSED` | The closed message from section 6 |
| `LIMIT_CUSTOMER` | The most you can send today is {remaining} GMD. |
| `LIMIT_COMPANY` | We cannot take this amount right now. Please try a smaller amount or try again later. |
| `LIMIT_NOT_SET` | We cannot take new transactions right now. Please try again later. |
| `RATE_CHANGED` | The rate has changed. Please check the new amount and continue. (the quote on screen is refreshed) |
| `PAIR_DISABLED` | This option is not available right now. |
| `SETTLEMENT_INVALID` | Please check your details and try again. (with the field marked) |
| `RECEIPT_REQUIRED` | Please add your payment receipt. |
| `TOO_MANY_RECEIPTS` | You can add up to 5 receipts. |
| `CUSTOMER_BLOCKED` | We cannot process your transaction. Please contact Customer Care. |
| any other error | Something went wrong. Please try again. If it keeps happening, contact Customer Care. |

## 8. Customer Care

The button opens `https://wa.me/<business number>?text=<message>`. With a transaction: "Hello, I need help with transaction {reference}." Without: "Hello, I need help." The bot answers from there.

## 9. Staff completes a web transaction

1. Staff open the transaction, check the receipts and the settlement details.
2. Staff mark the settlement details as correct or incorrect.
3. Approve, then Mark as Completed. For `source = 'WEB'`, the dashboard calls `POST /staff/generate-receipt`, not the WhatsApp receipt endpoint.
4. The server draws the receipt, saves it in `customer-receipts` under the customer's folder, and stores the path in `receipt_image_path`.
5. The customer's status page shows Completed and the receipt on its next refresh.
6. If the settlement details are wrong or the bank is down, staff use Fail with a reason instead.
