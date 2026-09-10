# CLAUDE_CONTEXT.md — Above & Beyond CFO Dashboard

## Project Overview
Daily financial dashboard for Above & Beyond, two divisions:
- **Restoration (ABPR)** — Above & Beyond Property Restoration
- **Improvement/Construction (ABPI)** — Above & Beyond Property Improvement

- **Dashboard**: https://aandbresto.github.io/ab-dashboard/
- **Repo**: https://github.com/aandbresto/ab-dashboard
- **Supabase**: https://svbmgueornewnasixpnh.supabase.co

## Pipeline
`CFO_Master_Workbook.xlsx` → Claude parses → `data/daily_data.json` → GitHub commit →
Action runs `scripts/push_to_supabase.py` → Supabase → live dashboard.

To trigger a re-push without new data, make any small edit to `data/daily_data.json`
(e.g. add/remove a blank line at the end) and commit — the workflow fires on any
change to that file.

---

## Critical Parser Rules (Always Apply)
- **AR totals**: ALWAYS sum from individual invoice rows, never trust the workbook's
  own formula cell for the total (it has been wrong/stale before).
- **AP totals**: same — sum from individual rows. The workbook's own AP summary cell
  has previously gone stale after a new row was added outside its SUM range; if the
  parsed sum doesn't match the workbook's displayed cell, flag it, don't silently trust
  either one blindly.
- **Net availability formula**: Cash + AR − AP − Credit Debt, then apply the standing
  exclusion below.

## Standing Exclusions — apply the math, but NEVER name them in the daily brief
The user has explicitly asked that these adjustments not be called out by name or
description in the brief — just show the resulting numbers plainly. If asked directly
"where did this number come from," explain fully; otherwise stay silent about it.

1. **Yvonne Badger, invoice #4120** (~$4,764.45, Restoration AR, in collections,
   attorneys reviewing bankruptcy claim): ALWAYS included in AR totals and the AR
   items list. ALWAYS excluded from net_cash_availability. Formula:
   `adjusted_net = standard_net − 4764.45`.
2. **Kingdom Restoration, invoice #299-1** (~$25,138, Improvement AP): on a
   confirmed **$2,000/month payment plan**. Stays in the AP tracker and Total AP
   at full balance. In the Debt Paydown Recommendation's Friday-AP obligation
   calculation only, this invoice is capped at $2,000 (or its remaining balance if
   under $2,000) instead of its full amount — implemented in `index.html` as
   `obligationAmount()` inside the debt paydown script block.

**General rule going forward**: don't mention *any* cap, adjustment, or exclusion
mechanism in the brief — not even in generic/unnamed form (e.g. don't say "one
invoice is capped"). Just show the final numbers. If the user asks where a number
came from, show the full math on request.

---

## Debt Paydown Recommendation Logic
- **Lookahead window**: 7 days (reverted from an earlier 14-day version — the
  14-day window held cash in reserve for obligations more than a week out even when
  there was real room to pay down debt sooner; reverted per explicit decision).
  Only payroll and business expenses due within 7 days count as obligations —
  something 8-14 days out is invisible to this calculation, so glance at what's
  coming up manually before recommending a large paydown right before a payroll date.
- **AP obligation**: Friday-marked (`pay_friday='Yes'`) invoices, with the Kingdom
  cap applied as above.
- Reserves: Construction $10,000 · Restoration $12,000.
- Priority for recommended paydowns: Cap One credit cards first (~24.49% APR),
  then LOCs (~8.75% APR).

## Payroll Schedule Anchors (biweekly)
- **Restoration**: anchor `2026-05-22`
- **Admin + Construction** (shared cycle): anchor `2026-05-29`
- Confirmed by the user: Restoration pays on its own date; Admin shares
  Construction's date, not Restoration's.

## AP Workbook Column Mapping
- Improvement AP: `col_offset=1` — inv_date, vendor, invoice_num, amount, billed,
  profit_pct, approval_status, job_total, due_date, comments (col 10), pay_friday (col 11)
- Restoration AP: `col_offset=13` (shifted from 12 after Comments column was added)
- Same field order both divisions.

---

## Dashboard Tabs
Overview, Cash Position, Transactions, Receivables, Payables, Cash Flow, Monthly
Snapshot, Payroll. **Profit Share tab was built then removed** (see below) — do not
re-add unless asked.

### Payroll Tab
- Team roster and pay rates are hardcoded config (`PAYROLL_TEAM` in `index.html`),
  not Supabase data. Only hours/pay entries per pay period are stored, in table
  `payroll_hours` (columns: `pay_period_end, division, employee_name, hours,
  ot_hours, sick_hours, holiday_hours, profit_share, bonus, reimbursement, status,
  comment`).
- **Restoration team**: Jacob Mercer ($38.46/hr), Jamie Walker ($30/hr), Derrek
  Thibodeaux ($26/hr), Gabor Sztuska ($13.50/hr), John Smyth ($38.46/hr), Damen
  Nunez ($18/hr).
- **Admin team**: Quena Valenzuela ($16/hr), Oziel Molina ($11/hr), Brizeida
  Portillo ($11/hr).
- **Construction team**: Gabor Sztuska ($10.93/hr), John Smyth ($1,923.08/month,
  flat — not hourly).
- **Pay rates are never displayed anywhere in this tab** — no toggle exists to
  reveal them; only computed pay amounts show. This was an explicit request.
- **Approval workflow**: same ✅❌↺ button pattern as the Friday AP approvals.
  Restoration and Admin are approved by **Jacob**. Construction is self-approved
  by the CFO (label just says "Approval", no name).
- **Profit Share column**: only editable on specific whitelisted pay dates
  (`PROFIT_SHARE_DATES` — currently `2026-09-25` for Restoration's cycle and
  `2026-09-18` for the Construction/Admin cycle), since biweekly periods don't
  align to calendar quarters. Every other period shows "Not this period." New
  quarterly dates must be added manually as they're confirmed — no formula
  auto-generates them.
- **Fields**: Hours, OT (1.5x), Sick, Holiday, Profit Share, Bonus, Reimbursement,
  Gross Pay (Admin still says "Total Pay" — the only division where taxes aren't
  broken out).
- **Restoration-only footer additions** (validated against real PEO invoices —
  see Validation section below):
  - **+ Approximate Employer Tax Cost**: 8% of gross (FICA match ~7.65% + small
    FUTA/SUTA cushion). This is the employer's own tax cost — explicitly NOT the
    employee's federal income tax withholding, which comes out of the employee's
    own gross pay and is never an added company cost. (This was corrected once
    already after being wrongly built to include employee withholding.)
  - **+ Approximate Administrative Fee**: 3.14% of gross (PEO service fee)
  - **+ Approximate Workers' Compensation**: 3.5% of gross
  - **+ Health Benefits**: flat $425.04 per pay period (covers 3 enrolled people
    at $141.68 each: Jacob Mercer, Derrek Thibodeaux, Jamie Walker)
  - **= Total Payroll Cost**: sum of all the above
  - All of these are per-period, computed off that period's gross total — NOT
    per-employee columns. Admin and Construction tables are untouched by any of
    this (no tax/fee/WC/health rows there).
- No number input spinner arrows anywhere in this tab (`.pr-num-input` CSS removes
  them) — user finds them distracting, types values directly.

### Removed: Profit Share Tab
Was built as a standalone tab (reclassified P&L, pool calculation, per-employee
allocation table) but **removed from the live dashboard** — the user decided it was
too much detail to expose to the team directly on a shared dashboard. Instead, a
one-off PDF report is generated per period and shared manually (see below). The nav
tab, page div, and all associated JS (`loadProfitShare`, `PROFIT_SHARE_ALLOCATION`,
`PROFIT_SHARE_POOL_PCTS`, `PROFIT_SHARE_POOL_RATE`, `PROFIT_SHARE_SHOW_AMOUNTS`) were
deleted from `index.html`. **`PROFIT_SHARE_DATES` was kept** — it's shared with the
Payroll tab's profit-share column feature and must not be removed.

The Supabase table `profit_share_periods` still exists with historical data but
nothing on the live dashboard reads from it anymore.

---

## Profit Share Program (for manual report generation, not on dashboard)
- **Pool**: 12% of reclassified Net Income for the period.
- **Reclassification concept**: field labor + field payroll taxes + workers' comp +
  field health benefits move from OpEx into COGS (they're direct cost of doing the
  work, not overhead) — Net Income is unchanged either way, only the margin
  presentation changes.
- **Field team** (for COGS reclassification, confirmed by the user): Derrek
  Thibodeaux, Jamie Walker, Damen Nunes. Validated by matching their combined
  employer-side FICA tax exactly against the P&L's "Employer Taxes - Field" line.
- **Allocation** (individual % is of the TOTAL pool, not of the sub-pool):
  - **Leadership (70%)**: Jacob Mercer 50%, Quena Valenzuela 20%
  - **Operations (15%)**: Briz Portillo 7.5%, Oziel Molina 7.5%
  - **Production (15%)**: Jaime Walker 9%, Derrek Thibodeaux 6%
- **Most recently validated period** (June 1 – August 31, 2026): Revenue
  $287,128.02, reclassified labor burden $43,193.86, reclassified Gross Profit
  $189,486.23 (66.0% margin vs. 81.0% unreclassified), Net Income $67,106.88, Pool
  $8,052.83.
- **Report style**: two-page PDF, no em dashes anywhere, title format "[Month
  range] Profit Report" (never "Q3" or quarter abbreviations), navy/green color
  scheme, team-facing appreciative tone. Generated fresh each period on request —
  this is a separate deliverable from the dashboard, not built into it.

---

## Data Validation Standards (explicit standing instruction)
Any tax rate, burden %, or other calculated/estimated figure added to the dashboard
must be validated against real source documents (PEO invoices, wage registers, WC
reports) before use — cross-check against at least two independent periods where
possible, not a single data point. The user will not always be supplying fresh
documents for every future change, so:
- If validating documents are available, use them and show the math.
- If not, still label the figure clearly as "Approximate" in the UI — never present
  an unvalidated estimate with the same confidence as a validated one.
- This rule was added after a real error: employer payroll tax was initially
  overstated by including the employee's own federal income tax withholding, which
  is never an actual employer cost.

## Daily Report Process
- Parse AR/AP/transactions/cash position from the uploaded workbook, cross-check
  every subtotal against the workbook's own cells.
- Diff against the previous day's saved `daily_data.json` to identify what
  actually changed (new/removed AR & AP items, Friday-flag changes, notes-only
  refreshes) — don't just restate the whole tracker every day.
- Where possible, confirm AR collections and AP payments against the
  Transactions tab entries for that day.
- Keep the brief concise — don't narrate things that don't need mentioning
  (routine confirmations, unchanged totals) and don't repeat the standing
  exclusions (see above).
- Yvonne Badger and Kingdom mentions: never in the brief (see Standing Exclusions).

## Known Historical Fixes (context only, already resolved)
- A backfilled AP invoice (ECO Mold Testing #9700, $625, Restoration) was missing
  from the tracker from 07/29/2026 onward and was backfilled into every historical
  `daily_snapshots`/`ap_items` row in that range once discovered.
- A duplicate daily report existed briefly under the wrong date (2026-09-05 instead
  of 2026-09-08) due to a mislabeling error — the duplicate row was deleted, correct
  data preserved under 09-08.
