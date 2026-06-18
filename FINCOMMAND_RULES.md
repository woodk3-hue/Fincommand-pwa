# FinCommand PWA — App Rules & Architecture Document

> This document defines the architecture, data model, design decisions and conventions for the FinCommand personal finance PWA. Reference this before making any code changes.

---

## App Overview

Single-file HTML/CSS/JS offline Progressive Web App for Android.
No external frameworks. No build step. No server required.
All data stored locally in IndexedDB.

**Current file:** `financial-command-centre-v5.html`
**DB Name:** `FinCmdDB8` · **DB Version:** `8`

---

## File Structure

Everything lives in one `.html` file:
- `<style>` — all CSS (dark theme, CSS variables)
- `<body>` — shell HTML (topbar, nav, views, FAB, sheet, toast)
- `<script>` — all JavaScript (DB, state, rendering, sheets, logic)

**No external dependencies.** No CDN. No npm. No build.

---

## Database Stores (IndexedDB)

| Constant | Store Name | Purpose |
|---|---|---|
| `ST.ACC` | `accounts` | Bank accounts, credit cards |
| `ST.TX` | `transactions` | All transactions |
| `ST.BILLS` | `bills` | Bills & subscriptions |
| `ST.SF` | `sinkingFunds` | Sinking fund records |
| `ST.PAY` | `paydayTemplates` | Auto transfer templates |
| `ST.PROP` | `properties` | Investment properties |
| `ST.WISE` | `wise` | Wise currency balances |
| `ST.LOANS` | `loans` | Loan records (Caravan) |
| `ST.INCOME` | `income` | Income records |
| `ST.PAYS` | `payRecords` | Payday history records |
| `ST.RENTS` | `rentRecords` | Rent payment records |
| `ST.PERIODS` | `payPeriods` | Pay period records |
| `ST.SNAPS` | `snapshots` | Account balance snapshots |
| `ST.BILLPAY` | `billPayments` | Bill payment history |
| `ST.TRIPS` | `trips` | Travel trips |
| `ST.WISEEX` | `wiseExchanges` | Currency exchange records |
| `ST.WISESP` | `wiseSpend` | Travel spending records |
| `ST.RATES` | `interestRates` | Mortgage interest rate history |
| `ST.INSTAL` | `instalments` | PayPal/Afterpay instalment plans |
| `ST.DASHCFG` | `dashConfig` | Dashboard card configuration |
| `ST.CASH` | `cashInHand` | Physical cash per currency |
| `ST.SETTLEMENTS` | `settlements` | Trip debt settlements |
| `ST.ATMDRAW` | `atmWithdrawals` | ATM withdrawal records |
| `ST.TRIPCOSTS` | `tripCosts` | Trip cost records (flights, hotels etc) |

**DB upgrade rule:** Bumping `DB_VER` triggers `onupgradeneeded`. New stores are auto-created. Version-specific migrations (e.g. Wise balance reset at v7) are gated with `if(e.oldVersion < N)`.

---

## Account Structure

### Macquarie
- Big Save *(master reserve — all pays deposited here)*
- Transaction Account *(everyday + overseas card)*
- Travel Save
- Body Corporate Fund

### HSBC
- Groceries
- Credit Card *(limit $6,000 · type: credit)*

### UP
- Bills
- Balloon Fund *(savings · target $20,000 · due 2030-02-13)*

### Bank Australia
- Griffin Transaction *(rent credited here)*
- Griffin Mortgage *(type: mortgage)*
- Bowen Hills Transaction *(rent credited here)*
- Bowen Hills Mortgage *(type: mortgage)*

### Wise *(ST.WISE — NOT in ST.ACC)*
- AUD, JPY, VND, IDR, CNY *(start at 0 — user enters manually)*

### Other
- Spaceship *(investment)*
- PayPal *(credit · limit $1,000 · pay-in-4)*
- Afterpay *(credit · limit $1,000 · pay-in-4)*

**Important:** Wise currencies live in `ST.WISE`, not `ST.ACC`. Never add Wise to the accounts seed data.

---

## Credit Accounts

Credit Card, PayPal, Afterpay all use the same pattern:

```js
{ type: 'credit', creditLimit: 6000, owing: 1351.29, balance: -1351.29 }
```

- `owing` = amount owed (positive)
- `balance` = negative of owing
- Spend → `owing` increases, `balance` decreases
- Repayment → `owing` decreases, `balance` increases
- Display: Limit / Owing / Available

---

## Properties

| Property | Value | Mortgage | Rent | Mortgage paid from |
|---|---|---|---|---|
| Griffin | $900K | $344,039.93 | $1,100/fn | Griffin Transaction |
| Bowen Hills | $800K | $410,388.35 | $750/fn | Bowen Hills Transaction |

**Rent rule:** Griffin rent → Griffin Transaction account. Bowen Hills rent → Bowen Hills Transaction account. Both create income TX records and update account balance immediately.

---

## Auto Transfer Templates (Payday)

All deducted from Big Save in this order:

| # | Name | Amount | Destination |
|---|---|---|---|
| 1 | Partner Rent | $400 | External |
| 2 | UP Bills | $225 | Bills |
| 3 | HSBC Groceries | $200 | Groceries |
| 4 | Body Corporate Fund | $180 | Body Corporate Fund |
| 5 | Griffin Buffer | $116 | Griffin Transaction |
| 6 | Bowen Buffer | $100 | Bowen Hills Transaction |
| 7 | Travel Save | $50 | Travel Save |
| 8 | Kids Savings | $30 | External |
| 9 | Transaction Account | $200 | Transaction Account |
| **Total** | | **$1,501/fn** | |

---

## Navigation

Bottom nav order (fixed):
1. **Dashboard** — Action centre
2. **Bills** (tab id: `payday`) — Auto transfers, bills, pay history
3. **Accounts** — All accounts including Wise
4. **Travel** — Active trip, Wise currencies, trip management
5. **More** — Net worth, properties, timeline, FY report, export/import

---

## Dashboard Layout

Top to bottom:
1. Pay Period compact strip (always visible)
2. Active Trip card (if trip is active — shows assigned currencies with Wise + Cash tiles)
3. New Payday / + Transaction buttons
4. Last Payday progress (tappable)
5. Transaction History (collapsible, collapsed by default)
6. Optional dashboard cards (toggleable via ⚙)

**Removed from dashboard:** Available Cash hero card, Balloon Fund card, Upcoming Bills card.

---

## Travel Module

### Active Trip Detection
`_detectActiveTrip()` runs on every load and tab switch. Sets `activeTripId` to earliest-start active trip. Multiple active trips supported — earliest wins.

### Trip Currency Assignment
Currencies are assigned when creating/editing a trip via checkboxes. Stored as `trip.currencies: ['VND']`. All spend/exchange/ATM sheets default to trip currencies.

### Cash in Hand
Stored in `ST.CASH` per currency. Separate from Wise card balance.
- ATM withdrawal: deducts from Wise or Macquarie Transaction → adds to cash
- Cash spending: deducts from cash only, never Wise
- End of trip: carry-forward prompt saves remaining cash

### ATM Withdrawals
Card options: **Wise Card** or **Macquarie Transaction Account** only.
- Wise → deducts Wise balance
- Macquarie → enter AUD charged, deducts Transaction Account AUD balance

### Currency Exchange (from account to Wise)
Source accounts: **Big Save**, **Transaction Account**, **Travel Save** only.

### Travel Spending — Payment Methods
1. Wise Card → deducts Wise balance
2. Cash on Hand → deducts cash balance
3. HSBC Credit Card → enter VND amount + AUD charged; adds to HSBC owing
4. Macquarie Transaction → enter VND amount + AUD charged; deducts Transaction Account

### Trip Costs (ST.TRIPCOSTS)
Three payment statuses:
- **Already Paid** — no balance change, included in Actual costs
- **Pay Now** — deducts from selected account, creates TX record, included in Actual costs
- **Planned** — no balance change, included in Planned costs only

Reporting: Budget / Actual / Planned / Est. Total / Remaining Budget

### Trip Completion
Manual "Mark Complete" or auto-prompt when end date passes.
Carry-forward: remaining cash per currency is confirmed and saved to `ST.CASH`.

### Leftover Travel Cash
Physical cash from all trips shown in collapsible section on Travel tab.

### Settlements
Per-trip debt tracking. Supports "I owe" and "they owe me". Offsets reduce balance. Mark settled zeroes it.

---

## Key Architectural Rules

### Single Source of Truth
- **Balloon Fund balance** → read from `ST.ACC` (UP account), not `ST.SF`
- **Wise balances** → read from `ST.WISE`, never from `ST.ACC`
- **Trip currencies** → read from `trip.currencies[]`, fall back to exchange history

### Mortgage Deduplication
Net Worth uses `props[]` for mortgage debt. Never include `type==='mortgage'` accounts in credit debt loop — avoids double counting.

### No window.confirm()
All delete/confirm actions use `showInlineConfirm()` — a custom bottom-sheet panel appended to `document.body` at `z-index:700`. Android PWA blocks `window.confirm()`.

### Toast Notifications
Use CSS keyframe animation (`toastIn`). Appended via `showToast(msg, col, dur)`. Auto-dismisses after `dur` seconds (default 3.5s). Tappable to dismiss early.

### Inline Confirm Z-index Stack
```
z-index 700 — inline confirm (above everything)
z-index 600 — toast
z-index 301 — bottom sheet
z-index 300 — overlay
z-index 200 — bottom nav
z-index 190 — FAB
z-index 100 — topbar
```

### FAB Context Awareness
FAB re-renders on every tab switch. On Travel tab shows: Record Spending, Record Exchange, ATM Withdrawal, Add Trip. On all other tabs shows: Add Transaction, Transfer Funds, Update Balance, Add Bill, New Payday.

---

## Delivery Method

When generating the file, Python file I/O is used to avoid bash heredoc size limits. Content is written in appended string parts. Always validate with `node --check` before delivering.

**Validation checklist before delivery:**
- `node --check` passes (no syntax errors)
- No duplicate function names
- All required functions present
- File closes with `</script></body></html>`

---

## User Preferences & Conventions

- Always ask clarifying questions one at a time before coding
- Never guess on account relationships, bill sources, or transfer destinations
- Do not rebuild — make targeted changes only
- Confirm interpretation before implementing complex revisions
- Account names must match seed data exactly (e.g. `'Transaction Account'` not `'Macquarie Transaction Account'`)
- Never use `window.confirm()` — always use `showInlineConfirm()`
- Never add Wise currencies to `ST.ACC`
- Template literals inside string concatenation inside template literals will break — use helper functions instead
