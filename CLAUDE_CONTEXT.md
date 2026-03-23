# REV / A&B Dashboard — Claude Context File
> Paste the raw URL of this file at the start of every new chat to restore full project context instantly.

---

## Project Overview
Daily financial dashboard for **REV Construction & Restoration** (two divisions: Improvement and Restoration).
Every morning a CFO master workbook is uploaded, parsed into `daily_data.json`, pushed to Supabase, and displayed on a GitHub Pages dashboard.

---

## URLs
- **Dashboard**: https://aandbresto.github.io/ab-dashboard/
- **GitHub Repo**: https://github.com/aandbresto/ab-dashboard
- **Supabase Project**: https://svbmgueornewnasixpnh.supabase.co
- **Supabase Publishable Key**: `sb_publishable_kdp8f09n4MJKzoQS6amd0A_rVJPnXxf`

---

## Repo Structure
```
ab-dashboard/
├── index.html                  # Full dashboard frontend (single file)
├── data/
│   └── daily_data.json         # Generated daily from workbook, triggers Action
├── scripts/
│   └── push_to_supabase.py     # Parses daily_data.json → inserts into Supabase
└── .github/workflows/
    └── upload.yml              # Triggers on data/daily_data.json push
```

---

## Daily Workflow
1. User uploads `CFO_Master_Workbook.xlsx` to Claude
2. Claude runs the parser (Python) → generates `data/daily_data.json`
3. User commits `daily_data.json` to GitHub repo under `data/`
4. GitHub Action triggers automatically → runs `scripts/push_to_supabase.py`
5. Dashboard at GitHub Pages reads from Supabase and displays data

**To trigger the Action manually** (when script changes but JSON hasn't changed):
- Edit `data/daily_data.json` → change `generated_at` timestamp by 1 second → commit

---

## Workbook Parser — Column Mappings

### Sheet: `💳 Daily Transactions`
Sections identified by row index. Header rows skipped. Only rows with actual `datetime` objects in col 1 are parsed.

| Section Row | Account Name |
|-------------|-------------|
| 8  | Capital One – Construction (2897) / division: improvement |
| 32 | Construction Checking – 2657 / division: improvement |
| 56 | Construction MM – 2690 / division: improvement |
| 81 | Capital One – Restoration / division: restoration |
| 105 | Restoration Checking – 7363 / division: restoration |
| 129 | Restoration MM – 2798 / division: restoration |

Header rows (skipped): 9, 33, 57, 82, 106, 130

Columns (0-indexed): `col[1]=trans_date, col[2]=posted_date, col[3]=card_desc, col[4]=vendor, col[5]=amount, col[6]=explanation, col[7]=approved_by, col[8]=txn_type`

**txn_type values**: `Credit`, `Debit`, `Transfer` (capitalized)

### Sheet: `📥 AR Tracker`
- **Improvement AR**: offset starting col 1 → `invoice_num, client_job, balance, invoice_date, due_date, days_out, expected_payment, last_update`
- **Restoration AR**: offset starting col 10 → same fields
- Data rows: 8 to end of sheet
- Skip rows where invoice field is null/nan/'Invoice #'/'TOTAL'

### Sheet: `📤 AP Tracker`
- **Improvement AP**: offset starting col 1 → `inv_date, vendor, invoice_num, amount, billed, profit_pct, approval_status, job_total, due_date, pay_friday`
- **Restoration AP**: offset starting col 12 → same fields
- Date parsing handles both `datetime` objects AND string dates in `MM/DD/YYYY` format
- Data rows: 8 to end of sheet

### Sheet: `💰 Cash Position`
Single column of values (col index 2), rows 0-indexed:
```
row[6]  = impr QB bank balance
row[7]  = impr bank 2657
row[8]  = impr MM 2690
row[9]  = impr total cash
row[10] = impr AR
row[11] = impr Cap One CC
row[12] = impr LOC 3705
row[13] = impr AP
row[15] = impr net total
row[18] = rest QB bank balance
row[19] = rest bank 7363
row[20] = rest MM 2798
row[21] = rest total cash
row[22] = rest AR
row[23] = rest LOC 5064
row[24] = rest Cap One CC
row[25] = rest AP
row[27] = rest net total
row[30] = combined total cash
row[31] = combined total AR
row[32] = combined total AP
row[33] = combined credit debt
row[34] = combined net availability
```

---

## daily_data.json Schema
```json
{
  "date": "2026-03-23",
  "generated_at": "2026-03-23T21:39:17Z",
  "cash_position": {
    "improvement": {
      "qb_bank_balance": 46428.56,
      "bank_balance_2657": 34306.48,
      "money_market_2690": 20099.23,
      "total_cash_in_bank": 54405.71,
      "accounts_receivable": 17523.17,
      "cap_one_cc": 15994.44,
      "loc_3705": 4440.05,
      "accounts_payable": 61221.45,
      "net_total": 2395.02
    },
    "restoration": {
      "qb_bank_balance": 15913.75,
      "bank_balance_7363": 16394.75,
      "money_market_2798": 25108.17,
      "total_cash_in_bank": 41502.92,
      "accounts_receivable": 60725.96,
      "loc_5064": 12055.70,
      "cap_one_cc": 36693.49,
      "accounts_payable": 15135.00,
      "net_total": 37863.69
    },
    "combined": {
      "total_cash_in_bank": 95908.63,
      "total_ar": 78249.13,
      "total_ap": 76356.45,
      "credit_debt": 69183.68,
      "net_cash_availability": 28617.63
    }
  },
  "ar": {
    "totals": { "total_ar": 78249.13, "impr_ar": 17523.17, "rest_ar": 60725.96 },
    "improvement": [
      { "invoice_num": "33080", "client_job": "Custom Craft Homes:680979", "balance": 1631.20,
        "invoice_date": "2026-01-13", "due_date": "2026-03-27", "days_out": 69,
        "expected_payment": "2026-03-27", "last_update": "3/23 Check run", "division": "improvement" }
    ],
    "restoration": [ "...same fields..." ]
  },
  "ap": {
    "totals": { "total_ap": 76356.45, "impr_ap": 61221.45, "rest_ap": 15135.00 },
    "improvement": [
      { "inv_date": "2025-05-07", "vendor": "Kingdom Restoration", "invoice_num": "299-1",
        "amount": 25138.00, "billed": null, "profit_pct": null,
        "approval_status": "680934 -", "job_total": null,
        "due_date": "2025-06-06", "pay_friday": "No", "division": "improvement" }
    ],
    "restoration": [ "...same fields..." ]
  },
  "transactions": [
    { "trans_date": "2026-03-20", "posted_date": "2026-03-20",
      "card_desc": "INV PMT Baumgardner Hous 032026", "vendor": null,
      "amount": 1800.00, "explanation": "Client Payment - BHL",
      "approved_by": null, "txn_type": "Credit",
      "account": "Restoration Checking – 7363", "division": "restoration" }
  ]
}
```

---

## Supabase Tables & Exact Column Names

### `daily_snapshots`
```
id, report_date, impr_qb_bank, impr_bank_2657, impr_mm_2690, impr_cash,
impr_ar, impr_ap, impr_net, rest_qb_bank, rest_bank_7363, rest_mm_2798,
rest_cash, rest_ar, rest_ap, rest_net, cap_one_construction, loc_3705,
loc_5064, cap_one_restoration, total_cash, total_ar, total_ap,
credit_debt, net_availability, created_at
```

### `ar_items`
```
id, report_date, client, balance, invoice_date, due_date,
days_outstanding (INTEGER — must cast float to int),
expected_payment_date, notes, invoice_number, division, created_at
```
⚠️ `days_outstanding` is INTEGER — always use `int(float(value))` before inserting

### `ap_items`
```
report_date, invoice_date, vendor, invoice_number, amount,
billed_to_client, profit_pct, notes, job_total, due_date,
pay_friday, division, created_at
```

### `transactions`
```
report_date, account_type, division, card_no, vendor, amount,
explanation, approved_by, txn_type, account, created_at
```
- `account_type`: `"credit_card"` or `"bank"` (derived from account name)
- `account`: full account name e.g. `"Restoration Checking – 7363"`
- `card_no` maps from JSON field `card_desc`
- ⚠️ NO `trans_date`, `posted_date` columns in this table

### `daily_briefs`
```
report_date, brief (JSON array of {icon, category, message})
```

### `payment_approvals`
```
report_date, vendor, invoice_number, division, status, comment, updated_at
```

---

## push_to_supabase.py Field Mappings

### JSON → Supabase

| JSON field | Supabase column | Table | Notes |
|-----------|----------------|-------|-------|
| `date` | `report_date` | all | was `report_date` in old script — THIS WAS THE BUG |
| `cash_position.improvement.qb_bank_balance` | `impr_qb_bank` | daily_snapshots | |
| `cash_position.improvement.total_cash_in_bank` | `impr_cash` | daily_snapshots | |
| `cash_position.combined.total_cash_in_bank` | `total_cash` | daily_snapshots | |
| `cash_position.combined.net_cash_availability` | `net_availability` | daily_snapshots | |
| `ar[].invoice_num` | `invoice_number` | ar_items | |
| `ar[].client_job` | `client` | ar_items | |
| `ar[].days_out` | `days_outstanding` | ar_items | cast to int |
| `ar[].expected_payment` | `expected_payment_date` | ar_items | |
| `ar[].last_update` | `notes` | ar_items | |
| `ap[].inv_date` | `invoice_date` | ap_items | |
| `ap[].invoice_num` | `invoice_number` | ap_items | |
| `ap[].billed` | `billed_to_client` | ap_items | |
| `ap[].approval_status` | `notes` | ap_items | |
| `txn.card_desc` | `card_no` | transactions | column is card_no not card_desc |
| `txn.account` | `account` | transactions | full name |
| derived | `account_type` | transactions | "credit_card" if "capital one" in name else "bank" |

---

## Known Bugs Fixed
1. **`payload["report_date"]` → `payload["date"]`** — original push script used wrong key, caused KeyError on line 16
2. **BHL transaction `txn_type`** — workbook had `"Credit"` correctly; old JSON had `"Debit"` by mistake
3. **`days_outstanding` float** — Supabase column is INTEGER, pandas reads as `69.0`; must cast with `int(float(v))`
4. **`card_desc` vs `card_no`** — Supabase transactions table uses `card_no`, not `card_desc`
5. **Snapshot column names** — old script used `impr_total_cash`, `total_cash_in_bank`, `net_cash_availability` etc. — actual columns are `impr_cash`, `total_cash`, `net_availability`

---

## Dashboard index.html — Key JS Functions
- `loadTransactions()` — fetches from `transactions` table, splits by `account_type` for CC vs bank summary
- `filterTxn(filter, el)` — filters by `t.account === filter` where filter is full account name
- `accountLabels{}` — maps full account name to display label
- `loadAR()` — reads `r.days_outstanding`, `r.expected_payment_date`, `r.notes`, `r.invoice_number`, `r.client`
- `loadAP()` — reads `r.notes` for approval status, `r.billed_to_client`, `r.invoice_number`
- `loadCash()` — reads `snap.impr_cash`, `snap.rest_cash`, `snap.total_cash`, `snap.net_availability` etc.

---

## Business Expenses (hardcoded in index.html)
Fixed monthly expenses by division used in Cash Flow projections.
Update `BUSINESS_EXPENSES` array in `index.html` when amounts change.

## Payroll Schedule (hardcoded in index.html)
Bi-weekly payroll calculated from `lastPaid` date + 14 days.
Update `PAYROLL_SCHEDULES` array when payroll dates or amounts change:
- Improvement: last paid 2026-03-06, est. $1,658.70
- Admin: last paid 2026-03-06, est. $3,119.00
- Restoration: last paid 2026-03-13, est. $15,500.00

---

## How to Start a New Chat
1. Open a new Claude chat
2. Type: "Here is the context file for the REV/A&B dashboard project:"
3. Paste this URL: `https://raw.githubusercontent.com/aandbresto/ab-dashboard/refs/heads/main/CLAUDE_CONTEXT.md`
4. Then describe what you need help with
5. Upload the workbook only when generating new JSON

