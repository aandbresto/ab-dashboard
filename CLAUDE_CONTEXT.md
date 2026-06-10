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

**To trigger the Action manually**: Edit `data/daily_data.json` → change `generated_at` timestamp by 1 second → commit

---

## Parser — Critical Rules

### AR Totals — ALWAYS sum from individual rows
⚠️ **NEVER trust the workbook formula cell for AR totals.** The workbook formula has a hardcoded range that misses new rows. Always calculate:
```python
rest_ar_total = sum(r.get('balance', 0) or 0 for r in rest_ar)
impr_ar_total = sum(r.get('balance', 0) or 0 for r in impr_ar)
total_ar = rest_ar_total + impr_ar_total
```
Then override the cash_position totals:
```python
cash_position['combined']['total_ar'] = total_ar
cash_position['restoration']['accounts_receivable'] = rest_ar_total
cash_position['improvement']['accounts_receivable'] = impr_ar_total
```

### Net Availability — Recalculate always
```python
net = total_cash + total_ar - total_ap - credit_debt
cash_position['combined']['net_cash_availability'] = net
```
Formula: **Cash + AR - AP - Credit Debt** (credit debt = CC balances + LOC balances)

### Overdue Days — Calculate from due_date vs today
```python
def calc_overdue(due_date_str):
    due = datetime.strptime(due_date_str, '%Y-%m-%d')
    return (today_dt - due).days  # positive = overdue, negative = current
```
Positive = overdue, negative = days until due.

---

## Workbook Parser — Column Mappings

### Sheet: `💳 Daily Transactions`
| Section Row | Account Name | Division |
|-------------|-------------|----------|
| 8  | Capital One – Construction (2897) | improvement |
| 32 | Construction Checking – 2657 | improvement |
| 56 | Construction MM – 2690 | improvement |
| 81 | Capital One – Restoration | restoration |
| 105 | Restoration Checking – 7363 | restoration |
| 129 | Restoration MM – 2798 | restoration |

Header rows (skipped): 9, 33, 57, 82, 106, 130
Columns (0-indexed): `col[1]=trans_date, col[2]=posted_date, col[3]=card_desc, col[4]=vendor, col[5]=amount, col[6]=explanation, col[7]=approved_by, col[8]=txn_type`

### Sheet: `📥 AR Tracker`
- **Improvement AR**: offset col 1 → `invoice_num, client_job, balance, invoice_date, due_date, days_out, expected_payment, last_update`
- **Restoration AR**: offset col 10 → same fields
- Data rows: 8 to end. Skip null/`'Invoice #'`/`'TOTAL'`

### Sheet: `📤 AP Tracker`
- **Improvement AP**: offset col 1 → `inv_date, vendor, invoice_num, amount, billed, profit_pct, approval_status, job_total, due_date, pay_friday`
- **Restoration AP**: offset col 12 → same fields

### Sheet: `💰 Cash Position`
Column index 2, rows 0-indexed:
```
row[6]=impr QB bank, row[7]=impr bank 2657, row[8]=impr MM 2690, row[9]=impr total cash
row[10]=impr AR (DO NOT USE — sum from rows instead)
row[11]=impr Cap One CC, row[12]=impr LOC 3705, row[13]=impr AP, row[15]=impr net total
row[18]=rest QB bank, row[19]=rest bank 7363, row[20]=rest MM 2798, row[21]=rest total cash
row[22]=rest AR (DO NOT USE — sum from rows instead)
row[23]=rest LOC 5064, row[24]=rest Cap One CC, row[25]=rest AP, row[27]=rest net total
row[30]=combined total cash, row[31]=combined total AR (override with summed value)
row[32]=combined total AP, row[33]=combined credit debt, row[34]=combined net availability (override)
```

---

## Supabase Tables & Exact Column Names

### `daily_snapshots`
```
report_date, impr_qb_bank, impr_bank_2657, impr_mm_2690, impr_cash,
impr_ar, impr_ap, impr_net, rest_qb_bank, rest_bank_7363, rest_mm_2798,
rest_cash, rest_ar, rest_ap, rest_net, cap_one_construction, loc_3705,
loc_5064, cap_one_restoration, total_cash, total_ar, total_ap,
credit_debt, net_availability, created_at
```

### `ar_items`
```
report_date, client, balance, invoice_date, due_date,
days_outstanding (INTEGER — cast float to int),
expected_payment_date, notes, invoice_number, division, created_at
```

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
- `account_type`: `"credit_card"` or `"bank"`
- `card_no` maps from JSON field `card_desc`

### `daily_briefs`
```
report_date, brief (JSON array of {icon, category, message})
```

### `monthly_reports`
```
report_month, division, revenue, gross_profit, net_profit,
payroll, total_expenses, monthly_actual, target,
services (jsonb array), metrics (jsonb array), prior_year (jsonb), summary
```
- `services`: `[{"name":"Water Damage","val":107509.22}, ...]`
- `metrics`: `[{"label":"Gross Profit %","v26":"85.14%","dir":"up_good"}, ...]`
- `prior_year`: `{"revenue":340908,"gross_profit":286908,...,"services":[...],"metrics":[...]}`

---

## Debt Paydown Recommendation Card
Built into `index.html` Overview tab. Calculates per division:

**Inputs used:**
- Cash in bank (per division)
- Payroll due within 7 days (from `PAYROLL_SCHEDULES`)
- Friday AP only — invoices marked `pay_friday = 'Yes'` in AP tracker (NOT weekly AP due dates)
- Business expenses due within 7 days (from `BUSINESS_EXPENSES` in index.html)
- Reserve: Construction $10,000 / Restoration $12,000

**Priority:** Credit cards first (24.49% APR), then LOCs (8.75% APR)

**Important:** Only Friday-marked AP is included. Unmarked AP invoices are excluded even if due date falls within the week — the user controls what gets paid each week by marking Friday.

---

## Payroll Schedule (in index.html `PAYROLL_SCHEDULES`)
Current as of June 2026:
- **Improvement Payroll**: lastPaid `2026-05-29`, estAmount `$3,500`, division `improvement`
- **Admin Payroll**: lastPaid `2026-05-29`, estAmount `$3,500`, division `admin`
- **Restoration Payroll**: lastPaid `2026-06-05`, estAmount `$15,500`, division `restoration`

Admin payroll goes entirely to Construction (not split). Payroll is biweekly (every 14 days).

---

## Interest Rates
- **Capital One CC (both divisions)**: 24.49% APR
- **LOC 3705 (Construction)**: 8.75% APR
- **LOC 5064 (Restoration)**: 8.75% APR

---

## Monthly Financial Report Workflow
1. Download QB P&L for both divisions (Jan 1 through last day of month, YTD with prior year comparison)
2. Upload PDFs to Claude — ask to build the monthly snapshot
3. Provide monthly actuals (single-month revenue for Construction and Restoration)
4. Claude generates SQL — paste directly into Supabase SQL Editor
5. Dashboard Monthly Snapshot tab updates automatically

**Monthly actuals loaded (as of June 2026):**
| Month | Division | Monthly Actual |
|-------|----------|----------------|
| Jan 2026 | construction | $8,525.20 |
| Jan 2026 | restoration | $100,278.08 |
| Feb 2026 | construction | $35,085.75 |
| Feb 2026 | restoration | $50,105.28 |
| Mar 2026 | construction | $33,044.48 |
| Mar 2026 | restoration | $54,257.37 |
| Apr 2026 | construction | $72,502.84 |
| Apr 2026 | restoration | $69,983.30 |
| May 2026 | construction | $28,608.93 |
| May 2026 | restoration | $92,248.15 |

---

## Known Bugs Fixed (do not reintroduce)
1. `payload["report_date"]` → must be `payload["date"]`
2. `card_desc` → maps to `card_no` in Supabase
3. `days_outstanding` float → cast to int
4. Em dash in account names → use substring number matching
5. `.execute()` calls → not needed in Supabase JS v2
6. `window._saveAppr` alias needed
7. Comment `if (status)` → must be `if (status || comment)`
8. Monthly report date matching → use `_month = (r.report_month||'').slice(0,10)`
9. services/metrics stored as JSONB arrays not objects
10. Sales Discounts excluded from service charts
11. txn_type null handling: `txn_type_raw.capitalize() if txn_type_raw else None`
12. AR totals → ALWAYS sum from individual rows, never trust workbook formula cell
13. Net availability → always recalculate: Cash + AR - AP - Credit Debt
14. Debt rec card must be inside `page-overview` div or it shows on all tabs

---

## index.html — Current State
The dashboard has these tabs: Overview, Cash Position, Transactions, Receivables, Payables, Cash Flow, Monthly Snapshot.

**Overview tab extras:**
- CFO Daily Brief card
- 💳 Debt Paydown Recommendation card (inside page-overview div)

**Do NOT modify index.html unless fixing bugs.** Always start from the current GitHub version when making changes.

---

## How to Start a New Chat
1. Open a new Claude chat
2. Paste: `https://raw.githubusercontent.com/aandbresto/ab-dashboard/refs/heads/main/CLAUDE_CONTEXT.md`
3. Upload workbook for daily JSON, or P&L PDFs for monthly report
4. Claude always uses today's actual date as report_date
5. Claude always sums AR from individual rows — never trusts workbook formula cell
