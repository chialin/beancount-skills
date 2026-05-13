# Weekly Budget Report Skill

A Claude Code skill that generates weekly budget analysis reports from your beancount file and writes them as markdown to a local folder of your choice (Obsidian vault, plain directory, etc.).

## Features

- **Weekly overview** — total spending, savings rate, monthly progress
- **Top 5 expenses** of the week
- **Monthly budget tracking** — per-category spend vs. budget
- **Yearly budget tracking** — YTD progress with timeline comparison
- **Next week allowance** — how much you can spend per day given remaining budget
- **AI suggestions** — 2-3 actionable recommendations based on the data

## Prerequisites

- Beancount installation with `bean-query`
- A local folder to receive the markdown report (any directory works)
- Optional: `budgets.bean` with `custom "budget"` directives for auto-category detection

## Installation

1. Copy `.claude/skills/weekly-budget-report/` into your project's `.claude/skills/`
2. Add `.claude/skills/weekly-budget-report/config.json` to your `.gitignore`
3. Run `/weekly-budget-report` — setup wizard kicks in on first run

## Configuration

On first run, a setup wizard asks:

| Field | Default | Notes |
|-------|---------|-------|
| `output_path` | `./docs/reports/` (auto-created) | Where weekly markdown reports go |
| `monthly_budget_total` | sum of `"monthly"` budgets in `budgets.bean` | Global monthly cap |
| `yearly_budget_total` | sum of `"yearly"` budgets in `budgets.bean` | Global yearly cap |

Auto-detected (no questions asked):

- Bean file (`*.bean` in current directory; `main.bean` priority)
- Currency (read from `option "operating_currency"` in bean file)
- Budget categories (parsed from `budgets.bean` if present)

## budgets.bean Format

```
2024-01-01 custom "budget" Expenses:Food          "monthly"     13000 TWD
2024-01-01 custom "budget" Expenses:Health:Medical "yearly"     42000 TWD
```

Without `budgets.bean`, all top-level `Expenses:*` accounts are tracked as monthly with no preset budget.

## Customization

Edit `config.json` directly to change output path or budget totals after initial setup.

## Translation

Skill output is currently zh-TW. PRs welcome for English / other languages.
