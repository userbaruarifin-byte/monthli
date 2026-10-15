# How Monthli Works — a plain-language tutorial

Written 2026-10-15 from the current code (`lib/`). If code and this file
disagree, the code wins. File paths are given so you can check each part.

---

## 0. The whole idea in one picture

Monthli answers one question every month:

> "How much can I safely take from my business for myself — without
> hurting tax, bills, or the business's safety cushion?"

```text
 ┌───────────┐   ┌────────────┐   ┌──────────────┐   ┌──────────────┐   ┌─────────────┐
 │ 1. RECORD │ → │ 2. CLOSE   │ → │ 3. PROTECT   │ → │ 4. OWNER PAY │ → │ 5. POCKETS  │
 │ money in  │   │ the month  │   │ tax, bills,  │   │ how much you │   │ split what's│
 │ & out     │   │ (lock it)  │   │ cushion      │   │ can take     │   │ left to goals│
 └───────────┘   └────────────┘   └──────────────┘   └──────────────┘   └─────────────┘
```

Three rules hold everything together:

1. **Cash, not profit.** Only money that actually arrived counts. An unpaid
   invoice is income on paper but adds Rp 0 to cash.
2. **Protected money is counted once and never paid out.** Tax, bills,
   the safety cushion and business pockets are set aside first.
3. **The app calculates; AI only helps.** Every number comes from fixed
   formulas in `lib/logic/`. AI can suggest entries or explain — it never
   decides a number.

---

## 1. Where the data lives

| What | Where | File |
|---|---|---|
| Transactions, receipts, categories, goals, closes, audit log | **On the phone only** (sembast database) | `lib/data/` |
| Login + subscription/trial | Supabase (Google sign-in) | `lib/providers/auth_provider.dart`, `supabase/` |
| AI features (record with AI, AI review) | Small server on Vercel → AI model | `server/src/routes/` |
| Backup | A JSON file you export/restore | `lib/logic/data_backup_service.dart` |

So: your books never leave the phone unless you use an AI feature (which asks
for consent first) or export a backup yourself.

**Code layers** (read top → bottom):

```text
screens/   what you see and tap
   ↓
providers/ keeps the app's current state (ledger, closes, goals, settings)
   ↓
logic/     pure formulas — no UI, fully unit-tested
   ↓
data/      models + repositories that read/write the phone database
```

---

## 2. Step 1 — Recording money (the ledger)

Every record is a `TransactionEntry` (`lib/data/models/transaction_entry.dart`).
All amounts are **whole rupiah integers** (no decimals, ever).

There are 5 types:

| Type | Meaning | Effect on cash |
|---|---|---|
| `income` | Client pays you / you invoice a client | + only the part actually **paid** |
| `invoicePayment` | Later payment of an earlier invoice | + full amount |
| `expense` | Business spending | − amount |
| `ownerDraw` | You take money for yourself (prive) | − amount (but **not** an expense) |
| `taxPayment` | You paid tax | − amount |

Income has a payment status:

```text
paid / not applicable → cash = full amount
partially paid        → cash = the paid part
unpaid                → cash = 0        (still shows as revenue in reports)
```

That is the "cash, not profit" rule in code (`cashAmountRupiah`, `cashDelta`).

Ways to add a record: manual form, receipt scan (on-device OCR,
`receipt_ocr.dart`), or **Record with AI** chat (`ai_chat_screen.dart`). Lines
the AI adds carry a **NEW** badge until you tap "Looks right" or edit them.

---

## 3. Step 2 — Closing the month

`lib/logic/close_checklist_service.dart` builds a checklist from your real data:

1. Transactions reviewed (no unchecked AI lines left)
2. Income categorized
3. Expenses categorized
4. Receipts complete

When all are ticked you can **close**. Closing:

- **Locks** the month — its records can't be edited or deleted
  (`ledger_provider.dart` throws "Reopen it first").
- **Freezes a snapshot** of the numbers and formula version, so old
  explanations don't change later.
- Counts as one more "closed month" for the owner-pay average (next step).

You can **reopen** a month (with a reason). Reopen + reclose makes a new
revision; the old one stays in the audit history.

---

## 4. Steps 3 & 4 — Safe Owner Pay (the heart of the app)

Code: `lib/logic/owner_pay_calculator.dart` (formulas) and
`lib/logic/owner_pay_inputs.dart` (where each input comes from).
Policy label: `owner-pay-policy v1`.

### 4.1 The inputs (letters used in the code)

| Letter | Name | Where it comes from |
|---|---|---|
| **K** | Business cash now | Your latest cash check (counted bank/cash balances) + every record after it |
| **P** | Tax reserve | Tax set aside for last month + this month − tax already paid this month |
| **H** | Bills due this month | Fixed costs due this month not yet paid + unpaid loan installments this month |
| **B** | Fixed costs per month | Recurring costs (rent, salaries, subscriptions) + loan installments |
| **R** | Runway months | Default **3**, you can set 1–12. Starts **next** month |
| **S** | Business pockets | Balance of goals funded by the business |
| **Lᵢ** | Cash profit of a closed month | Cash in − expenses − tax paid (owner draw is **not** subtracted) |
| **n** | Averaging window | Default **3** months (options 3 / 6 / 12) |

Tax: Monthli no longer has a tax engine. **You** set tax as a percent of
collected income or a fixed monthly amount (Settings → Tax). Percent is applied
to cash actually collected, rounded down.

### 4.2 The formulas

```text
1. Protected money      D   = P + H + (B × R) + S
2. Hard limit           M   = max(0, K − D)
3. Average profit       Ā   = average of the last n CLOSED months' Lᵢ (rounded down)
4. Recommendation       Rek = max(0, min(M, Ā − owner draws already taken this month))
```

In words:

- **M (maximum)** = cash left after protecting everything. Taking more than
  this eats into tax/bills/cushion money.
- **Rek (recommended)** = what the business actually *earns* on average, but
  never more than M. This stops you treating one lucky month as salary.
- No closed month yet → no recommendation (only M is shown).
- Fewer closed months than n → result labelled **"provisional"**.

### 4.3 Zones when you record a draw of x

```text
🟢 Safe                x ≤ Rek
🟡 Above recommended   Rek < x ≤ M   → asks you to confirm
🔴 Over the limit      x > M         → confirm + optional reason + written to audit log
                                        shortfall = x − M  ("Refill Rp X" shows on Home)
```

Monthli **never blocks** a draw (it doesn't hold your money; the books must
match the bank). It only warns and records.

If you go over M, the shortfall is taken from protected money in this order:
**fixed-cost cushion → business pockets → bills → tax** (tax last).

Extra warning: if this year's total draws > this year's total profit, Monthli
says your business capital is shrinking.

### 4.4 Worked example (also a unit test)

```text
K = 50 jt   P = 2 jt   H = 3 jt   B = 10 jt   R = 3   S = 5 jt
Last 3 closed months profit L = 12, 18, 15 jt

D   = 2 + 3 + (10 × 3) + 5   = 40 jt
M   = 50 − 40                = 10 jt
Ā   = (12 + 18 + 15) ÷ 3     = 15 jt
Rek = min(10, 15)            = 10 jt

Draw 8 jt  → 🟢
Draw 10 jt → 🟢
Draw 13 jt → 🔴  shortfall 3 jt, taken from the fixed-cost cushion
```

If cash grows to 70 jt → M = 30 jt, Rek = 15 jt (now limited by average profit).

### 4.5 Why each piece is there (no double counting)

- This month's fixed costs are in **H**; the cushion **B × R** starts **next**
  month — so this month isn't counted twice.
- A bill you already paid is already out of K, so H subtracts it — not counted
  twice.
- Tax already paid is out of K, so P subtracts it.
- Recording a draw lowers K (so M drops) **and** is subtracted from Rek.

---

## 5. Step 5 — Dream Vault (goal pockets) and allocation

Code: `lib/logic/dream_calculator.dart`, `lib/providers/dream_goal_provider.dart`,
`lib/screens/dream_allocation_review_screen.dart`.

### 5.1 A pocket is a label on money, not new money

Each goal has a funding source (**business** or **personal**), a priority and
a target. Its balance is rebuilt only from its movements:

```text
balance = opening balance + deposits − withdrawals − spending   (never below 0)
```

Moving money into a pocket does **not** change income, expenses or profit.

### 5.2 Four goal types → how much each needs this month

| Type | Monthly need |
|---|---|
| Emergency fund | remaining ÷ months to build |
| Dated target | remaining ÷ months until the target date |
| Business asset | your planned monthly deposit (capped at remaining) |
| Flexible wishlist | whatever is left after higher priorities |

Optional inflation raises the target over time:
`effective target = base × (1 + inflation%)^(months since created ÷ 12)`.

Business-asset goals slow down when profit drops:
profit down 15–30% → deposit scaled down; down ≥ 30% or profit ≤ 0 → paused.

### 5.3 The budgets (the two sides never mix)

```text
Business budget limit = M   (the hard limit from owner pay)
Personal budget limit = owner draws you actually recorded this month
                        − personal deposits already confirmed this month
```

Why business uses M: a business-pocket deposit raises S → raises D → lowers M.
So pocket money and owner pay can never spend the same rupiah.

Why personal uses *recorded* draws: you can only save personal money you
actually took out — a recommendation isn't money yet.

### 5.4 The waterfall

For each source separately, goals are filled from highest priority down:

```text
deposit = min(goal's monthly need, remaining target, budget left)
```

You review, edit (invalid edits are rejected), then **confirm**. Only **one
confirmed batch per month** is allowed; totals can't exceed the budgets.

---

## 6. AI — what it can and can't do

| AI does | AI never does |
|---|---|
| Proposes transactions from your text/photo (you tap Add) | Saves anything without your tap |
| Reviews a month for duplicates/odd items | Calculates cash, tax, owner pay or allocations |
| Explains results in plain words | Records owner draws (those go through the Owner Pay screen) |

The phone validates every AI suggestion (`lib/logic/ai_entry.dart`): amount > 0,
date not in the future, category must exist. AI lines block the close until
checked. The app works fully with AI off.

---

## 7. One full month, start to finish

```text
1. During the month   add income/expenses (manual, scan, or AI chat)
2. Month end          Close screen → fix checklist → Close (month locks)
3. Owner Pay screen   see Rek and M → record a draw (green/yellow/red)
4. Dream Vault        Allocate → review business + personal waterfall → Confirm
5. Home               shows next safe number, and "Refill Rp X" if you went over
```

---

## 8. Where to look in the code

| Question | File |
|---|---|
| Owner pay formulas | `lib/logic/owner_pay_calculator.dart` |
| Where K, P, H, B, L come from | `lib/logic/owner_pay_inputs.dart` |
| Puts it all together for a month | `lib/providers/owner_pay_snapshot.dart` |
| Close checklist | `lib/logic/close_checklist_service.dart` |
| Goals and waterfall | `lib/logic/dream_calculator.dart` |
| One-batch-per-month rule | `lib/providers/dream_goal_provider.dart` |
| Cash charts | `lib/logic/cash_flow_view.dart` |
| Tests proving the formulas | `test/owner_pay_calculator_test.dart` and others in `test/` |
| Decisions and history | `next_plan.md` (formula agreed 2026-10-09) |
