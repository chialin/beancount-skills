# beancount-skills

A collection of [Claude Code](https://claude.ai/code) skills for [Beancount](https://beancount.github.io/) double-entry bookkeeping.

Each skill is self-contained markdown instructions for Claude Code to follow. Drop a skill folder into your project's `.claude/skills/` (or install via `npx skills`) and trigger it with the slash command.

## Available Skills

| Skill | Language | Trigger | What it does |
|-------|----------|---------|--------------|
| [`weekly-budget-report`](./weekly-budget-report) | English | `/weekly-budget-report` | Weekly budget analysis: monthly + yearly tracking, top-5 expenses, next-week allowance, AI suggestions. |
| [`weekly-budget-report-zh-TW`](./weekly-budget-report-zh-TW) | 繁體中文 | `/weekly-budget-report` | Same skill, output in Traditional Chinese. |

More skills coming. PRs welcome.

## Installation

### Option A — `npx skills` CLI (recommended)

[`npx skills`](https://github.com/vercel-labs/skills) is the Vercel Labs installer for Claude Code skills. It puts the skill files in the right place automatically.

```bash
# List available skills in this repo
npx skills add chialin/beancount-skills --list

# Install one skill into the current project's .claude/skills/
npx skills add chialin/beancount-skills --skill weekly-budget-report

# Or install everything
npx skills add chialin/beancount-skills --all

# Install globally (~/.claude/skills/, shared across projects)
npx skills add chialin/beancount-skills --skill weekly-budget-report -g
```

After install, trigger inside Claude Code with `/weekly-budget-report` — the setup wizard runs on first invocation.

### Option B — Manual clone & copy

```bash
git clone https://github.com/chialin/beancount-skills.git /tmp/beancount-skills
mkdir -p .claude/skills/
cp -r /tmp/beancount-skills/weekly-budget-report .claude/skills/
```

### Always: gitignore your personal config

Each skill writes a `config.json` (output path, budget totals, etc.) on first run. **Never commit this file.** Add it to your `.gitignore`:

```
.claude/skills/<skill-name>/config.json
```

Each skill folder ships with its own `README.md` covering prerequisites, configuration, and per-skill details.

## Prerequisites

- [Claude Code](https://claude.ai/code) CLI
- [Beancount](https://beancount.github.io/) with `bean-query` (`pip install beanquery`)
- A beancount project with `main.bean` (or any `*.bean` — first found wins)

## Conventions

- Skills assume the beancount project root is your current working directory.
- Skills read `option "operating_currency"` from your bean file to determine display currency — no hardcoded TWD/USD/etc.
- Skills that need budget categories parse `budgets.bean` for `custom "budget"` directives. Without `budgets.bean` they fall back to listing all top-level `Expenses:*` accounts.

See each skill's own README for specifics.

## License

MIT — see [LICENSE](./LICENSE).

## Contributing

Bug reports, skill ideas, and PRs welcome. The canonical language is English; translations go into their own `<skill>-<lang>` folder (e.g. `weekly-budget-report-zh-TW`).

### Paired-language PR policy

PRs that touch one language folder of a skill **must** also touch the paired variant folder. The `i18n Sync Check` GitHub Action enforces this on every PR. To bypass for a legitimate single-language change (typo fix in one translation, e.g.), add the label `i18n-sync-skip` and justify in the PR description.

---

## 繁體中文簡介

這是給 [Claude Code](https://claude.ai/code) 用的 [Beancount](https://beancount.github.io/) 複式記帳 skill 集合。每個 skill 是一份 markdown 指令，丟進你專案的 `.claude/skills/`（或用 `npx skills` 安裝）就能用 slash command 觸發。

中文使用者建議裝 `weekly-budget-report-zh-TW` 變體：

```bash
npx skills add chialin/beancount-skills --skill weekly-budget-report-zh-TW
```

詳細的中文設定說明見 [`weekly-budget-report-zh-TW/README.md`](./weekly-budget-report-zh-TW/README.md)。
