# UI and Copy Guide

## 1. Brand

- Name: **AfrikBerry One-Tap**. Written as the wordmark "Afrik" in green and "Berry" in navy, followed by "One-Tap" in a smaller weight.
- Never show "Jacerock" or the parent company name anywhere (screens, titles, errors, manifest, share text).
- The JR logo (transparent PNG) stays. Use the original file from `staff-dashboard/public/logo.png` in the existing repo. Keep it as `public/logo.png` here, and make 192 and 512 pixel square icons from it for the PWA.
- Tagline for the sign-in screen: "Cross-border payments, made simple".

### Colours

| Use | Colour |
|---|---|
| Primary green (buttons, success) | `#1E9E52`, dark `#14804A` |
| Navy (headings, header) | `#12306B` |
| Ink (main text) | `#0B1B33` |
| Soft text | `#5B6B80` |
| Muted labels | `#7A8797` |
| Page background | `#EDF1F6` |
| Card background | `#FFFFFF` |
| Green tint (receive box) | `#F1F7F3` |
| Lines | `#E3E9F0` |
| Warning (amber) | `#B45309` |
| Error (red) | `#B91C1C` |

### Type

Inter (weights 400, 500, 600, 700, 800). Body text 16 px or more. Inputs 18 px or more. Amounts 28 to 36 px, weight 800.

## 2. Principles for low digital literacy

1. One job per screen. Few fields. A big button at the bottom.
2. Numbers are large and have thousands separators (17,700).
3. Plain words. No banking jargon. Say "Send amount", not "Debit value".
4. Show the result before asking for a decision. The rate and both amounts are always visible together.
5. Buttons are at least 48 px high and at least 8 px apart.
6. Say what happens next ("We will update this page when it is done").
7. Errors say what to do, not what failed technically.
8. Never rely on colour alone. Pair colours with words or icons.
9. Keyboard type matches the field: number pad for amounts and phone numbers.
10. Keep the user's data when they go back.

## 3. Screens

| Screen | Content |
|---|---|
| Welcome / login | Logo, tagline, phone number, password, "Log in", "Create account", "Forgot password" |
| Sign up | Full name, country, phone number, password, confirm password, "Continue" |
| Verify phone | The 6-digit code in large digits, "Open WhatsApp" button, countdown, "Get a new code" |
| Home | Greeting with first name, availability banner, remaining limit, "Send money" button, last 3 transactions, "Contact Customer Care" |
| Send: amount | Direction switch, Send amount box, Receive amount box (green tint), rate line, remaining limit, "Continue" |
| Send: receive details | Fields for the direction (see `docs/FLOWS.md` section 4) |
| Send: pay | Bank transfer or Mobile wallet, account details with copy buttons, receipts upload (1 to 5) |
| Send: review | Everything on one screen, "Submit" |
| Transaction | Status, reference, amounts, details, receipts you uploaded, completion receipt and download, Customer Care button when failed |
| History | List with status chips (Processing, Completed, Failed) |
| Profile | Name, phone, "Change password", "Log out" |
| Closed | The closed message with the opening time, history still reachable |

### The amount screen

Follow the client's mock-up: a navy header block with the logo and the tagline, a "Send Money" title with a green pill showing the direction ("GMD to NGN"), a "Transfer amount" section with two boxes side by side. The left box is white with a rounded input ("Send amount", currency, the number). The right box is grey-green and shows the result in large green digits ("Receive amount", currency, the number). Under them, a row with a refresh icon, "Exchange rate" and "1 GMD = 17.70 NGN". Both boxes accept typing.

## 4. Copy rules

- Short sentences. Active voice. Everyday words.
- No em dashes. Use a full stop, a comma or a colon.
- Write currencies as the code with the name on first mention: "Dalasi (GMD)", "Naira (NGN)". After that, the code is enough.
- Buttons are verbs: "Continue", "Submit", "Add another receipt", "Copy number".
- Status words are exactly: Processing, Completed, Failed.
- Use the message texts in `docs/FLOWS.md` sections 6 and 7 word for word.
- Dates: "24 Oct 2026". Times with "Gambia time" where the time matters.

## 5. States every screen needs

| State | Treatment |
|---|---|
| Loading | Skeleton blocks, not a blank page. Buttons disabled |
| Empty | A friendly sentence and the next action ("You have no transactions yet. Send money to get started.") |
| Error | The message from `docs/FLOWS.md` plus a "Try again" button |
| Offline | A banner: "You are offline. Check your connection." Disable submit |
| Success | A clear confirmation and the next step |

## 6. Accessibility

- Contrast at least 4.5 to 1 for text.
- Every input has a visible label (not only a placeholder).
- Focus is visible. Tab order follows the screen.
- Images and icons that carry meaning have text alternatives.
- Support text size changes up to 200 percent without breaking the layout.

## 7. PWA

- Name "AfrikBerry One-Tap", short name "One-Tap", `display: standalone`, theme colour `#12306B`, background `#EDF1F6`.
- Show the install prompt after the customer's first completed transaction, not on first visit.
- The service worker caches the app shell only.
