---
name: weekly-budget-report
description: 每週預算分析報表。產出本週花費分析、月累計 vs 預算、年度進度、下週可花參考與 AI 建議，寫入指定本地資料夾。使用 `/weekly-budget-report` 觸發。建議週日晚上記帳後執行。
---

# 每週預算分析報表

分析本週花費並產出下週預算參考，寫入 config 指定的本地資料夾。

## 觸發方式

`/weekly-budget-report`

## 步驟 0：載入或建立 config

### 0a. 檢查 config.json 是否存在

路徑：`.claude/skills/weekly-budget-report/config.json`

- 存在 → 用 Read 工具讀取，跳到步驟 0.5
- 不存在 → 進入 setup wizard（0b）

### 0b. Setup Wizard

執行 0.5 的偵測（bean file、currency、budgets.bean），先一次顯示偵測結果：

```
🔍 偵測中...
  ✓ Bean file: ./main.bean
  ✓ Currency: TWD
  ✓ Found budgets.bean → N monthly categories, M yearly categories
    （或：未找到 budgets.bean → 將用所有頂層 Expenses 類別）
```

**接著逐一詢問三個問題**，每問完一題等使用者回答後再問下一題（不要一次列出 3 題）：

**Q1：** 請輸入週報輸出路徑（本地資料夾或 Obsidian vault 內子目錄）[預設 `./docs/reports/`]：

→ 等使用者回答後再問 Q2

**Q2：** 月度總預算 [預設 X，monthly 類別加總]：

→ 等使用者回答後再問 Q3

**Q3：** 年度總預算 [預設 Y，yearly 類別加總]：

→ 收齊後進入 0c

**預設值規則：**

- `output_path`：預設 = `{當前目錄}/docs/reports/`。空白 Enter 用預設。
- `monthly_budget_total`：預設 = budgets.bean 中 `"monthly"` entries 加總。沒 budgets.bean 時無預設、必填。
- `yearly_budget_total`：預設 = budgets.bean 中 `"yearly"` entries 加總。沒 budgets.bean 時無預設、必填。

### 0c. 寫入 config.json

收齊使用者輸入後：

1. 若 `output_path` 目錄不存在，用 Bash `mkdir -p` 建立
2. 用 Write 工具寫入 `.claude/skills/weekly-budget-report/config.json`：

（若使用者輸入相對路徑，先用 `realpath` 或 `cd ... && pwd` 展開為絕對路徑再寫入）

```json
{
  "output_path": "/expanded/absolute/path",
  "monthly_budget_total": 30000,
  "yearly_budget_total": 120000
}
```

3. 顯示確認訊息：`✓ 寫入 config.json`，繼續下一步

## 步驟 0.5：環境偵測（每次跑都重新做）

不快取於 config，每次都即時推導。

### 0.5a. 偵測 bean file

```bash
find . -maxdepth 2 -name '*.bean' -type f
```

優先順序：
1. `./main.bean`
2. 第一個找到的 `*.bean`
3. 找不到 → 詢問使用者並中止

```bash
if [ -f "./main.bean" ]; then
  bean_file="./main.bean"
else
  bean_file=$(find . -maxdepth 2 -name '*.bean' -type f | head -1)
fi
```

### 0.5b. 偵測 currency

讀取 bean file，搜尋：`option "operating_currency" "XXX"`

```bash
grep '^option "operating_currency"' <bean_file> | head -1
```

抽出引號內貨幣代碼：

```bash
currency=$(grep '^option "operating_currency"' <bean_file> | sed 's/.*"\([^"]*\)".*/\1/')
```

找不到時 fallback 詢問使用者。

### 0.5c. 解析 budgets.bean

檢查專案根目錄是否有 `budgets.bean` 或 `budget.bean`。

存在時，用 Read 工具讀取，正則 parse 每行：

```
^\d{4}-\d{2}-\d{2} custom "budget"\s+(\S+)\s+"(monthly|yearly)"\s+([\d,]+\.?\d*)\s+(\w+)
```

抽出 `(account, period, amount, currency)`，分成兩個列表：

- `monthly_categories`：所有 period == "monthly" 的 entries
- `yearly_categories`：所有 period == "yearly" 的 entries

每個 entry 結構：
```
{
  "name": "Food",                 # account 末段
  "pattern": "^Expenses:Food",    # 用於 bean-query regex
  "monthly_budget": 8000,         # 或 yearly_budget
  "currency": "TWD"
}
```

### 0.5d. Fallback：沒 budgets.bean

執行：

```bash
bean-query <bean_file> "
  SELECT DISTINCT root(account, 2) as cat
  WHERE account ~ '^Expenses:'
"
```

把結果全當 monthly_categories，每個 entry 沒有 budget 數字（純追蹤花費）。yearly_categories 為空。

## 設定來源

所有個人設定來自：

- **config.json**（步驟 0）：output_path、monthly_budget_total、yearly_budget_total
- **環境偵測**（步驟 0.5）：bean_file、currency、monthly_categories、yearly_categories

## 步驟一：計算日期範圍

1. 執行 `date +%Y-%m-%d` 取得今天日期
2. 計算本週一和本週日的日期（ISO 週：週一起始）
3. 計算 ISO 週數（`date +%V`），用於檔名 `YYYY-WNN.md`
4. 計算本月剩餘天數

## 步驟二：查詢本週花費

### 2a. 預算類別本週花費（用於表格各列）

`<monthly_categories_pattern_alternation>` = 把所有 monthly_categories[i].pattern 用 `|` 串起來。例：`^Expenses:Food|^Expenses:Life`。

```bash
# 使用 alternation regex：把所有 monthly_categories[i].pattern 用 | 串起來
# （或逐一查詢每個 category 後彙總結果）
bean-query <bean_file> "
  SELECT account, sum(convert(position, '<currency>')) as total
  WHERE account ~ '<monthly_categories_pattern_alternation>'
    AND date >= YYYY-MM-DD_MONDAY AND date <= YYYY-MM-DD_SUNDAY
  GROUP BY account
"
```

依 category.pattern 把符合的所有 account 加總到該 category。

### 2b. 本週全類別明細（用於摘要行 + 非預算類別表格）
```bash
bean-query <bean_file> "
  SELECT account, sum(convert(position, '<currency>')) as total
  WHERE account ~ '^Expenses:'
    AND date >= YYYY-MM-DD_MONDAY AND date <= YYYY-MM-DD_SUNDAY
  GROUP BY account
  ORDER BY total DESC
"
```

- **本週總花費**（摘要行/概覽）= 全部 Expenses 加總
- **非預算類別本週花費**（用於下方「非預算中的消費」表格）= 將不屬於 monthly_categories 或 yearly_categories 任一 pattern 的科目單獨列出

### 2c. 本週收入（用於本週概覽）
```bash
bean-query <bean_file> "
  SELECT sum(convert(position, '<currency>')) as total
  WHERE account ~ '^Income:'
    AND date >= YYYY-MM-DD_MONDAY AND date <= YYYY-MM-DD_SUNDAY
"
```

注意：bean-query 會把 Income 顯示為負值，取絕對值使用。


## 步驟三：查詢月累計花費

### 3a. 各預算類別月累計（用於表格各列）
```bash
bean-query <bean_file> "
  SELECT account, sum(convert(position, '<currency>')) as total
  WHERE account ~ '<monthly_categories_pattern_alternation>'
    AND year(date) = YEAR AND month(date) = MONTH
  GROUP BY account
"
```

### 3b. 全類別月累計（用於摘要行「月累計 XX,XXX / {monthly_budget_total} {currency}」）
```bash
bean-query <bean_file> "
  SELECT sum(convert(position, '<currency>')) as total
  WHERE account ~ '^Expenses:'
    AND year(date) = YEAR AND month(date) = MONTH
"
```

注意：摘要行的月累計應包含**所有** Expenses，不只 monthly_categories 列出的類別。

## 步驟 3.5：查詢年度預算進度

僅當 yearly_categories 非空時執行。

### 3.5a. 各 yearly 類別 YTD 花費

對每個 yearly_categories[i]，查詢：

```bash
bean-query <bean_file> "
  SELECT sum(convert(position, '<currency>')) as total
  WHERE account ~ '<category.pattern>'
    AND year(date) = <YEAR>
"
```

### 3.5b. 各 yearly 類別本週花費

```bash
bean-query <bean_file> "
  SELECT sum(convert(position, '<currency>')) as total
  WHERE account ~ '<category.pattern>'
    AND date >= YYYY-MM-DD_MONDAY AND date <= YYYY-MM-DD_SUNDAY
"
```

### 3.5c. 計算每個 yearly 類別的進度

對每個 category：

- `ytd_used` = 3.5a 結果
- `weekly_used` = 3.5b 結果
- `yearly_remaining` = `yearly_budget - ytd_used`
- `ytd_pct` = `ytd_used / yearly_budget × 100`
- `timeline_pct` = `current_month / 12 × 100`（顯示「33% (4/12)」）

「時程進度」幫助判斷該類別是否落後/超前（理想 ytd_pct ≈ timeline_pct）。

### 3.5d. yearly 區塊小計

- `yearly_subtotal_ytd` = sum of 3.5a results
- `yearly_subtotal_pct` = `yearly_subtotal_ytd / yearly_budget_total × 100`

## 步驟 3.6：查詢年度全 Expenses 累計

```bash
bean-query <bean_file> "
  SELECT sum(convert(position, '<currency>')) as total
  WHERE account ~ '^Expenses:'
    AND year(date) = <YEAR>
"
```

- `yearly_total_ytd` = 查詢結果
- `monthly_avg` = `yearly_total_ytd / current_month`（current_month 為 1-12 的當月月份數）

## 步驟四：計算下週預算參考

- **月剩餘** = 月預算 - 月累計
- **剩餘天數** = 本月最後一天 - 今天（從明天算起）
- **下週可花** = 月剩餘 / 剩餘天數 × 7（若跨月則取到月底天數）
- **每日可花** = 月剩餘 / 剩餘天數
- 超支類別不計算下週可花，標記 `⚠️`

## 步驟五：產生 AI 建議

根據以下數據產生 2-3 條具體建議：
- 哪個類別使用率偏高，需要注意
- 哪個類別進度正常
- 與上週相比有無異常波動（若可查詢）

## 步驟 5.5：本週前五大支出

從步驟 2b 的全類別查詢結果中，取金額最大的前 5 筆，無論是否屬於預算類別。
顯示時 label 用 account 的最後一段（例如 `Expenses:Food:Coffee` 顯示為 `Coffee`）。

## 步驟六：寫入輸出檔

1. 確認 `{config.output_path}` 目錄存在（setup wizard 已建立過，但每次跑保險再 mkdir -p）：
```bash
mkdir -p "{config.output_path}"
```

2. 寫入檔案：`{config.output_path}/YYYY-WNN.md`

3. 渲染規則：
   - 若 `yearly_categories` 為空，**省略整個「### 預算內 — 年度」section**（不要輸出空表格）
   - 若 `monthly_categories` 為空（罕見），月度區塊一樣處理
   - `{yearly_total_ytd}` / `{monthly_avg}` 來自步驟 3.6
   - `{top1_label}` 等 = 前五大 account 的末段名稱（同步驟 5.5）

輸出格式：

```markdown
# YYYY-WNN 週預算分析（MM/DD - MM/DD）

## 💰 本週概覽

| 本週支出 | 本週收入 | 本月累計支出 |
|---------|---------|-------------|
| **XX,XXX {currency}** | X,XXX,XXX {currency} | XX,XXX {currency} |

> 本月剩餘 N 天

## 🏆 本週前五大支出

| 類別 | 金額 |
|------|------|
| {top1_label} | X,XXX {currency} |
| {top2_label} | X,XXX {currency} |
| ... | ... |

## 本週花費

### 預算內 — 月度

| 類別 | 本週花費 | 月預算 | 月累計 | 月剩餘 | 使用率 |
|------|---------|--------|--------|--------|-------|
（依 monthly_categories 動態產生每一列）

> 月度小計 X,XXX {currency}｜月累計 XX,XXX / {monthly_budget_total} {currency} (XX%)

### 預算內 — 年度

（僅當 yearly_categories 非空時呈現此 section）

| 類別 | 本週 | 年預算 | YTD | 年剩餘 | YTD 使用率 | 時程進度 |
|------|------|--------|-----|--------|-----------|---------|
（依 yearly_categories 動態產生每一列）

> 年度小計｜YTD XXX,XXX / {yearly_budget_total} {currency} (XX%)｜時程 XX% (M/12)

### 非預算中的消費

| 類別 | 本週花費 |
|------|---------|
（不屬於任何 monthly/yearly category pattern 的 Expenses）

> 非預算小計 X,XXX {currency}

> **本週總花費 XX,XXX {currency}**（含預算內 + 非預算）

## 年度總支出

> 年度總支出 XXX,XXX {currency}｜月平均 XX,XXX {currency}

## 下週預算參考

| 類別 | 下週可花 | 每日可花 |
|------|---------|---------|
（依 monthly_categories 動態產生；超支類別標 ⚠️）

## 建議
1. ...
2. ...
```

## 步驟七：完成

寫入檔案後直接結束 skill 即可。

注意：若 `{config.output_path}` 是受 git 管理的目錄，由使用者自行決定是否 commit；skill 不主動 commit。
