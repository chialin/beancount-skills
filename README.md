# beancount-skills

A collection of [Claude Code](https://claude.ai/code) skills for [Beancount](https://beancount.github.io/) double-entry bookkeeping.

Each skill is self-contained markdown instructions for Claude Code to follow. Drop a skill folder into your project's `.claude/skills/` and trigger it with the slash command.

## Available Skills

| Skill | Trigger | What it does |
|-------|---------|--------------|
| [weekly-budget-report](./weekly-budget-report) | `/weekly-budget-report` | Generates a weekly budget analysis report from your beancount file (monthly + yearly budget tracking, top-5 expenses, next-week allowance, AI suggestions). Writes markdown to a local folder of your choice. |

More skills coming. PRs welcome.

## Installation

### Option A：用 `npx skills` CLI（推薦）

[`npx skills`](https://github.com/vercel-labs/skills) 是 Vercel Labs 的 skill 安裝工具，會自動把 skill 放到對應位置。

```bash
# 列出本 repo 內可用的 skills
npx skills add chialin/beancount-skills --list

# 安裝特定 skill 到當前專案的 .claude/skills/
npx skills add chialin/beancount-skills --skill weekly-budget-report

# 或安裝全部 skills
npx skills add chialin/beancount-skills --all

# 安裝到全域（~/.claude/skills/，所有專案共用）
npx skills add chialin/beancount-skills --skill weekly-budget-report -g
```

裝完後在 Claude Code 直接打 `/weekly-budget-report` 觸發（首次跑會進 setup wizard）。

### Option B：手動複製

```bash
# 在你的 beancount 專案根目錄
git clone https://github.com/chialin/beancount-skills.git /tmp/beancount-skills
mkdir -p .claude/skills/
cp -r /tmp/beancount-skills/weekly-budget-report .claude/skills/
```

### 安裝後務必 .gitignore 個人 config

每個 skill 第一次跑會在自己資料夾下產生 `config.json`，內含路徑與預算等個人設定。記得加進你的 `.gitignore`：

```
.claude/skills/<skill-name>/config.json
```

每個 skill 資料夾內都有自己的 `README.md`，含詳細 prerequisite 與設定說明。

## Prerequisites

- [Claude Code](https://claude.ai/code) CLI
- [Beancount](https://beancount.github.io/) with `bean-query` (`pip install beanquery`)
- A beancount project with `main.bean` (or any `*.bean` — first found wins)

## Conventions

- Skills assume your beancount project root is your current working directory.
- Skills read `option "operating_currency"` from your bean file to determine the display currency (no hardcoded TWD/USD/etc.).
- If a skill needs budget categories, it parses `budgets.bean` for `custom "budget"` directives. Without `budgets.bean` it falls back to listing all top-level `Expenses:*` accounts.

See each skill's own README for specifics.

## License

MIT — see [LICENSE](./LICENSE).

## Contributing

Bug reports, skill ideas, and PRs welcome. Skill output is currently zh-TW; English translation PRs especially welcome.
