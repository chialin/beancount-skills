# Weekly Budget Report Skill（繁體中文）

Claude Code skill：從你的 beancount 檔案產出每週預算分析報表，寫成 markdown 到你指定的本地資料夾（Obsidian vault、純資料夾皆可）。

> Looking for the English version? See [`../weekly-budget-report/`](../weekly-budget-report).

## 功能

- **本週概覽** — 總花費、儲蓄率、月度進度
- **本週前五大支出**
- **月度預算追蹤** — 每類別花費 vs 預算
- **年度預算追蹤** — YTD 進度搭配時程比較
- **下週可花參考** — 依剩餘預算算出每日可花
- **AI 建議** — 2-3 條可執行的建議

## 前置需求

- Beancount 安裝，含 `bean-query`
- 一個本地資料夾用來放 markdown 報表（任何目錄皆可）
- 可選：`budgets.bean` 含 `custom "budget"` directives 以自動偵測類別

## 安裝

```bash
npx skills add chialin/beancount-skills --skill weekly-budget-report-zh-TW
```

接著把產生的 config 加進 `.gitignore`：

```
.claude/skills/weekly-budget-report-zh-TW/config.json
```

在 Claude Code 內執行 `/weekly-budget-report` — 第一次跑會進 setup wizard。

## 設定

第一次跑時 wizard 會逐一問三個問題：

| 欄位 | 預設值 | 說明 |
|------|--------|------|
| `output_path` | `./docs/reports/`（自動建立） | 週報 markdown 的存放路徑 |
| `monthly_budget_total` | `budgets.bean` 中 `"monthly"` 加總 | 月度總預算上限 |
| `yearly_budget_total` | `budgets.bean` 中 `"yearly"` 加總 | 年度總預算上限 |

自動偵測（不問你）：

- Bean file（當前目錄的 `*.bean`；`main.bean` 優先）
- Currency（讀 bean file 的 `option "operating_currency"`）
- 預算類別（從 `budgets.bean` 解析）

## budgets.bean 格式

```
2024-01-01 custom "budget" Expenses:Food           "monthly"    13000 TWD
2024-01-01 custom "budget" Expenses:Health:Medical "yearly"     42000 TWD
```

沒有 `budgets.bean` 時，所有頂層 `Expenses:*` 都當 monthly 類別追蹤，無預設預算金額。

## 自訂

初次設定後直接編輯 `config.json` 即可調整 output path 或預算上限。

## 語言版本

- **English**：[`../weekly-budget-report/`](../weekly-budget-report) — canonical
- **繁體中文**：本資料夾（`weekly-budget-report-zh-TW/`）

歡迎其他語言的 PR。
