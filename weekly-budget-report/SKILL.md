---
name: weekly-budget-report
description: Weekly budget analysis report. Produces this-week spending, MTD vs. budget, YTD progress, next-week allowance, and AI recommendations; writes to a configured local folder. Trigger with `/weekly-budget-report`. Best run Sunday evening after daily bookkeeping.
---

# Weekly Budget Analysis Report

Analyzes this week's spending and produces a next-week budget reference, writing output to the local folder specified in config.

## Trigger

`/weekly-budget-report`

## Step 0: Load or Create Config

### 0a. Check Whether config.json Exists

Path: `.claude/skills/weekly-budget-report/config.json`

- Exists → read with the Read tool, skip to Step 0.5
- Does not exist → enter setup wizard (0b)

### 0b. Setup Wizard

Run the detection from Step 0.5 (bean file, currency, budgets.bean), then display detection results all at once:

```
🔍 Detecting...
  ✓ Bean file: ./main.bean
  ✓ Currency: TWD
  ✓ Found budgets.bean → N monthly categories, M yearly categories
    (or: budgets.bean not found → all top-level Expenses categories will be used)
```

**Then ask the three questions one at a time**, waiting for the user's answer before asking the next (do not list all 3 at once):

**Q1:** Enter the report output path (local folder or subdirectory inside your Obsidian vault) [default `./docs/reports/`]:

→ Wait for the user's answer, then ask Q2

**Q2:** Total monthly budget [default X, sum of monthly category budgets]:

→ Wait for the user's answer, then ask Q3

**Q3:** Total yearly budget [default Y, sum of yearly category budgets]:

→ Once all answers are collected, proceed to 0c

**Default value rules:**

- `output_path`: default = `{current directory}/docs/reports/`. Press Enter to accept default.
- `monthly_budget_total`: default = sum of `"monthly"` entries in budgets.bean. No default if budgets.bean is absent — required field.
- `yearly_budget_total`: default = sum of `"yearly"` entries in budgets.bean. No default if budgets.bean is absent — required field.

### 0c. Write config.json

After collecting all user input:

1. If the `output_path` directory does not exist, create it with Bash `mkdir -p`
2. Write to `.claude/skills/weekly-budget-report/config.json` using the Write tool:

(If the user entered a relative path, expand it to an absolute path first using `realpath` or `cd ... && pwd`)

```json
{
  "output_path": "/expanded/absolute/path",
  "monthly_budget_total": 30000,
  "yearly_budget_total": 120000
}
```

3. Show confirmation: `✓ config.json written`, then continue to the next step

## Step 0.5: Environment Detection (Run Fresh Every Time)

Not cached in config — derived live on every run.

### 0.5a. Detect Bean File

```bash
find . -maxdepth 2 -name '*.bean' -type f
```

Priority order:
1. `./main.bean`
2. First `*.bean` found
3. None found → ask the user and abort

```bash
if [ -f "./main.bean" ]; then
  bean_file="./main.bean"
else
  bean_file=$(find . -maxdepth 2 -name '*.bean' -type f | head -1)
fi
```

### 0.5b. Detect Currency

Read the bean file and search for: `option "operating_currency" "XXX"`

```bash
grep '^option "operating_currency"' <bean_file> | head -1
```

Extract the currency code from within the quotes:

```bash
currency=$(grep '^option "operating_currency"' <bean_file> | sed 's/.*"\([^"]*\)".*/\1/')
```

If not found, fall back to asking the user.

### 0.5c. Parse budgets.bean

Check whether `budgets.bean` or `budget.bean` exists in the project root.

If it exists, read it with the Read tool and regex-parse each line:

```
^\d{4}-\d{2}-\d{2} custom "budget"\s+(\S+)\s+"(monthly|yearly)"\s+([\d,]+\.?\d*)\s+(\w+)
```

Extract `(account, period, amount, currency)` into two lists:

- `monthly_categories`: all entries where period == "monthly"
- `yearly_categories`: all entries where period == "yearly"

Each entry structure:
```
{
  "name": "Food",                 # last segment of account
  "pattern": "^Expenses:Food",    # used as bean-query regex
  "monthly_budget": 8000,         # or yearly_budget
  "currency": "TWD"
}
```

### 0.5d. Fallback: No budgets.bean

Run:

```bash
bean-query <bean_file> "
  SELECT DISTINCT root(account, 2) as cat
  WHERE account ~ '^Expenses:'
"
```

Treat all results as `monthly_categories`; each entry has no budget amount (spending tracking only). `yearly_categories` is empty.

## Configuration Sources

All personal settings come from:

- **config.json** (Step 0): output_path, monthly_budget_total, yearly_budget_total
- **Environment detection** (Step 0.5): bean_file, currency, monthly_categories, yearly_categories

## Step 1: Calculate Date Range

1. Run `date +%Y-%m-%d` to get today's date
2. Calculate the dates of this week's Monday and Sunday (ISO week: Monday start)
3. Calculate the ISO week number (`date +%V`) for the filename `YYYY-WNN.md`
4. Calculate the number of days remaining in the current month

## Step 2: Query This Week's Spending

### 2a. Weekly Spending by Budget Category (for each table row)

`<monthly_categories_pattern_alternation>` = all monthly_categories[i].pattern joined with `|`. Example: `^Expenses:Food|^Expenses:Life`.

```bash
# Use alternation regex: join all monthly_categories[i].pattern with |
# (or query each category separately and merge results)
bean-query <bean_file> "
  SELECT account, sum(convert(position, '<currency>')) as total
  WHERE account ~ '<monthly_categories_pattern_alternation>'
    AND date >= YYYY-MM-DD_MONDAY AND date <= YYYY-MM-DD_SUNDAY
  GROUP BY account
"
```

Sum all matching accounts into their respective category based on category.pattern.

### 2b. All-Category Weekly Detail (for summary row + non-budget category table)
```bash
bean-query <bean_file> "
  SELECT account, sum(convert(position, '<currency>')) as total
  WHERE account ~ '^Expenses:'
    AND date >= YYYY-MM-DD_MONDAY AND date <= YYYY-MM-DD_SUNDAY
  GROUP BY account
  ORDER BY total DESC
"
```

- **Total weekly spending** (summary row / overview) = sum of all Expenses
- **Non-budget category weekly spending** (for the "Non-Budget Spending" table below) = accounts that do not match any monthly_categories or yearly_categories pattern, listed individually

### 2c. Weekly Income (for Weekly Overview / Savings Rate)
```bash
bean-query <bean_file> "
  SELECT sum(convert(position, '<currency>')) as total
  WHERE account ~ '^Income:'
    AND date >= YYYY-MM-DD_MONDAY AND date <= YYYY-MM-DD_SUNDAY
"
```

Note: bean-query returns Income as a negative value; take the absolute value.

**Savings Rate** = (Weekly Income − Weekly Spending) / Weekly Income × 100% (rounded to integer). Skip calculation if income is 0.

## Step 3: Query Month-to-Date Spending

### 3a. MTD by Budget Category (for each table row)
```bash
bean-query <bean_file> "
  SELECT account, sum(convert(position, '<currency>')) as total
  WHERE account ~ '<monthly_categories_pattern_alternation>'
    AND year(date) = YEAR AND month(date) = MONTH
  GROUP BY account
"
```

### 3b. All-Category MTD (for the summary line "MTD XX,XXX / {monthly_budget_total} {currency}")
```bash
bean-query <bean_file> "
  SELECT sum(convert(position, '<currency>')) as total
  WHERE account ~ '^Expenses:'
    AND year(date) = YEAR AND month(date) = MONTH
"
```

Note: the summary line's MTD figure should include **all** Expenses, not only the categories listed in monthly_categories.

## Step 3.5: Query Yearly Budget Progress

Run only when yearly_categories is non-empty.

### 3.5a. YTD Spending per Yearly Category

For each yearly_categories[i], query:

```bash
bean-query <bean_file> "
  SELECT sum(convert(position, '<currency>')) as total
  WHERE account ~ '<category.pattern>'
    AND year(date) = <YEAR>
"
```

### 3.5b. This Week's Spending per Yearly Category

```bash
bean-query <bean_file> "
  SELECT sum(convert(position, '<currency>')) as total
  WHERE account ~ '<category.pattern>'
    AND date >= YYYY-MM-DD_MONDAY AND date <= YYYY-MM-DD_SUNDAY
"
```

### 3.5c. Calculate Progress for Each Yearly Category

For each category:

- `ytd_used` = result from 3.5a
- `weekly_used` = result from 3.5b
- `yearly_remaining` = `yearly_budget - ytd_used`
- `ytd_pct` = `ytd_used / yearly_budget × 100`
- `timeline_pct` = `current_month / 12 × 100` (displayed as "33% (4/12)")

"Timeline %" helps judge whether a category is behind or ahead (ideal: ytd_pct ≈ timeline_pct).

### 3.5d. Yearly Section Subtotal

- `yearly_subtotal_ytd` = sum of 3.5a results
- `yearly_subtotal_pct` = `yearly_subtotal_ytd / yearly_budget_total × 100`

## Step 3.6: Query Total YTD Expenses

```bash
bean-query <bean_file> "
  SELECT sum(convert(position, '<currency>')) as total
  WHERE account ~ '^Expenses:'
    AND year(date) = <YEAR>
"
```

- `yearly_total_ytd` = query result
- `monthly_avg` = `yearly_total_ytd / current_month` (current_month is the current month number, 1–12)

## Step 4: Calculate Next Week's Budget Reference

- **Monthly Remaining** = Monthly Budget − MTD
- **Days Remaining** = last day of month − today (counting from tomorrow)
- **Next Week Allowance** = Monthly Remaining / Days Remaining × 7 (capped at days until month end if it crosses a month boundary)
- **Per Day Allowance** = Monthly Remaining / Days Remaining
- Over-budget categories do not get a next-week calculation; mark with `⚠️`

## Step 5: Generate AI Recommendations

Based on the data below, produce 2–3 concrete recommendations:
- Which categories have high utilization and need attention
- Which categories are on track
- Any unusual spikes compared to last week (if queryable)

## Step 5.5: Top 5 Expenses This Week

From the all-category query results in Step 2b, take the 5 accounts with the highest amounts, regardless of whether they belong to a budget category.
Display the label as the last segment of the account path (e.g. `Expenses:Food:Coffee` → `Coffee`).

## Step 6: Write Output File

1. Confirm `{config.output_path}` directory exists (setup wizard already created it, but re-run `mkdir -p` each time as a safeguard):
```bash
mkdir -p "{config.output_path}"
```

2. Write file: `{config.output_path}/YYYY-WNN.md`

3. Rendering rules:
   - If `yearly_categories` is empty, **omit the entire "### Within Budget — Yearly" section** (do not output an empty table)
   - If `monthly_categories` is empty (rare), handle the monthly block the same way
   - `{yearly_total_ytd}` / `{monthly_avg}` come from Step 3.6
   - `{top1_label}` etc. = last segment of the top-five account names (same as Step 5.5)

Output format:

```markdown
# YYYY-WNN Weekly Budget Analysis (MM/DD - MM/DD)

## 💰 Weekly Overview

| Weekly Spending | Savings Rate |
|-----------------|--------------|
| **XX,XXX {currency}** | **XX%** |

| Weekly Income | MTD Spending |
|---------------|--------------|
| X,XXX,XXX {currency} | XX,XXX {currency} |

> N days left in month

## 🏆 Top 5 Expenses This Week

| Category | Amount |
|----------|--------|
| {top1_label} | X,XXX {currency} |
| {top2_label} | X,XXX {currency} |
| ... | ... |

## This Week's Spending

### Within Budget — Monthly

| Category | Week Spend | Monthly Budget | MTD | Remaining | Usage % |
|----------|------------|----------------|-----|-----------|---------|
(generated dynamically per monthly_categories)

> Monthly subtotal X,XXX {currency}｜MTD XX,XXX / {monthly_budget_total} {currency} (XX%)

### Within Budget — Yearly

(only shown when yearly_categories is non-empty)

| Category | This Week | Annual Budget | YTD | Annual Remaining | YTD % | Timeline % |
|----------|-----------|---------------|-----|------------------|-------|------------|
(generated dynamically per yearly_categories)

> Yearly subtotal｜YTD XXX,XXX / {yearly_budget_total} {currency} (XX%)｜Timeline XX% (M/12)

### Non-Budget Spending

| Category | Week Spend |
|----------|------------|
(Expenses matching neither monthly nor yearly category patterns)

> Non-budget subtotal X,XXX {currency}

> **Total weekly spending XX,XXX {currency}** (budget + non-budget combined)

## Total Yearly Spending

> YTD total XXX,XXX {currency}｜Monthly average XX,XXX {currency}

## Next Week's Budget Reference

| Category | Next Week | Per Day |
|----------|-----------|---------|
(generated dynamically per monthly_categories; mark over-budget categories with ⚠️)

## Recommendations
1. ...
2. ...
```

## Step 7: Done

Once the file is written, the skill exits.

Note: if `{config.output_path}` is inside a git-managed directory, it is up to the user to decide whether to commit the file; the skill does not commit automatically.
