# beancount-skills

A collection of [Claude Code](https://claude.ai/code) skills for [Beancount](https://beancount.github.io/) double-entry bookkeeping.

Each skill is self-contained markdown instructions for Claude Code to follow. Drop a skill folder into your project's `.claude/skills/` and trigger it with the slash command.

## Available Skills

| Skill | Trigger | What it does |
|-------|---------|--------------|
| [weekly-budget-report](./weekly-budget-report) | `/weekly-budget-report` | Generates a weekly budget analysis report from your beancount file (monthly + yearly budget tracking, top-5 expenses, next-week allowance, AI suggestions). Writes markdown to a local folder of your choice. |

More skills coming. PRs welcome.

## Installation

For each skill you want to use:

1. Copy the skill folder into your beancount project:
   ```bash
   # Example for weekly-budget-report
   mkdir -p .claude/skills/
   cp -r path/to/beancount-skills/weekly-budget-report .claude/skills/
   ```

2. Add the per-skill config file to your `.gitignore` (skills generate a local `config.json` on first run that contains personal data):
   ```
   .claude/skills/<skill-name>/config.json
   ```

3. Run the slash command (e.g. `/weekly-budget-report`) inside Claude Code — first run kicks off a setup wizard.

Each skill folder has its own `README.md` with prerequisites and configuration details.

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
