---
name: aoccqa-case-exporter
metadata:
  version: 1.1.1
description: >
  共用工具（人工指令觸發、Agent 執行）。把前一個 agent（測試案例產生器）產出的
  測試案例，連同一張 Jira 單，套進 AOCC QA 官方 xlsx 模板並匯出。只整理輸出、
  不做內容判斷或篩選。凡是使用者要「匯出測試案例成 xlsx」「套進 AOCC 模板」
  「接上一個 agent 的案例輸出做成 Excel」「產出 Test Case 檔」「把案例清單存成
  Test_Case 檔」，或提供 Jira 單＋測試案例要打包成交付檔時，都應觸發此 skill，
  即使沒說出 "case-exporter" 這個字。不屬於任何角色，Phase B/C/D 任何階段都可
  直接呼叫，與 AOCCQA-decision-archiver 無關聯。
---

# AOCCQA-case-exporter

「前一個 agent 案例輸出」+「Jira 單」→ 套官方模板 → 匯出 xlsx。
**確定性格式化工具:只搬運套版,不判斷、不篩選、不改寫案例內容。**

## 契約（不可違反）

1. **只整理輸出**:agent 產出什麼案例,就原樣填進 xlsx,不增刪、不改寫、不過濾、不重排序。
2. **欄位固定**:Test case 分頁沿用模板既有 12 欄,順序與命名不變。
3. **公式不動**:Report 的 Pass/Fail/N-A 統計與比率、Bug list 嚴重度統計,全部保留原公式。
4. **Bug list / Screenshot 分頁**:原樣保留,不更動任何內容或格式。
5. 呼叫時機不限;與 decision-archiver 無關聯。

## 輸入

**① Jira 單**(供 Report 分頁＋檔名):Assignee、Summary、link(feature / release note)、MCC#、Description 內文(抓「測試時程」「測試環境」段落)。

**② agent 案例輸出**(7 欄):
`Test Case ID, Category, Feature, Pre-condition, Test Case, Steps, Expected Result`
外加選填 `Test Data`(延伸欄)。有給 → 寫入 L 欄;沒給 → 留白。**不自行產生測試資料。**

**③ 模組代號**(選填):檔名第二段(如 `CRM`)。Summary 標籤已含模組則免給。

## 對應規則（已鎖定）

### Report 分頁 ← Jira

| Report 欄位（標籤格 → 寫入格） | 來源 |
|---|---|
| Project（A2 → **C2**） | Jira Summary 完整標題（含 `[UAT-QA][XX]` 標籤,原樣填） |
| Test date（A3 → **C3**） | Description 內時程,正規化為 **`YYYY/MM/DD-YYYY/MM/DD`**（最早起日～最晚迄日,不帶 Internal Testing/UAT 字樣） |
| Test Version（A4 → **C4**） | 暫留白（未指定） |
| Tester（A5 → **C5**） | `AOCC_{Assignee}`（取 Jira Assignee 姓名;覆蓋模板預填值） |
| New feature & Release Note（A6 → **C6**） | Jira link（如 `.../jira/browse/DV2IN1-44637`） |
| Test Country（A13 → **C13**） | MCC#（如 `MCC1 (BE-NL/IT/PL/CZ)`） |
| Test Environment（A14 → **C14**） | Description 內「測試環境」段落（如 `MCC1 Stage`） |

- Pass / Fail / N-A 統計（L2:L4）與比率（P2:P4）:**保留公式,不填值**。
- `Device`（C11 預填 PC / Mobile）、`Browser`（C12 預填 Chrome / Edge）:**保留預填,不動**。
- Description 屬半結構化內文,時程/環境採關鍵字定位。**抓不到 → 該格留白 + 回報是哪一格,絕不臆測填值。**

### Test case 分頁 ← agent 輸出

資料列自第 2 列起（模板備至第 201 列,上限 200 筆）。逐筆對應:

| agent 欄位 | 寫入模板欄 |
|---|---|
| Test Case ID | A（ID） |
| Category | E（Category） |
| Pre-condition | F（Pre-condition） |
| Test Case | G（Test case） |
| Steps | H（Steps,保留換行） |
| Expected Result | I（Expected result） |
| **Test Data**（選填） | **L — 有給就寫、沒給留白** |
| **Feature** | **捨棄,不寫入任何欄** |

- **不填**:B（PC）、C（Mobile）、D（Tablet）、J（Test result）、K（Note）。
- **L 為條件填入**:case 有非空 `test_data` 才寫,否則留白。**絕不臆造測試資料。** 多行原樣保留換行。事後回報 `test_data_filled` / `test_data_blank` 筆數。
- **案例數 > 200 → 停止並回報使用者,不覆蓋公式範圍。**

## 檔名規則（已鎖定）

```
{市場}_{模組}_{標題}_TestCase_{YYYYMMDD}.xlsx
```

Summary → 檔名,依序:
1. 移除 `[UAT-QA]` 標籤(不分大小寫)。
2. 取出**開頭所有** `[標籤]`,去中括號作前綴段,`_` 相連(house style `[市場][模組]` 兩段;一段或三段亦可)。
3. Summary 只帶市場標籤 + 另給模組代號(`jira.module`) → 插為**第二段**;已存在標籤中則不重複(不分大小寫)。
4. 接 `_TestCase_{YYYYMMDD}`(**`TestCase` 無空格**),`YYYYMMDD` = 匯出當天。

範例:
- `[UAT-QA][TW][CRM] ASUS Membership reward points redemption minimum threshold`
  → `TW_CRM_ASUS Membership reward points redemption minimum threshold_TestCase_20260722.xlsx`
- `[UAT-QA][TW] ASUS Membership ...` ＋ `module="CRM"` → 同上(模組由欄位補入)
- `[UAT-QA][EU] Customized Bundle maintenance mechanism enhancement`(無模組)
  → `EU_Customized Bundle maintenance mechanism enhancement_TestCase_20260722.xlsx`

- 檔名不含狀態字樣。日後要加「定稿版／草稿版／進度快照」再調整。
- 非法字元（`/ \\ : * ? " < > |`）以底線取代,避免存檔失敗。

## 執行步驟

1. 讀 Jira 單,抽出上表 Report 欄位與 Summary。
2. 讀 agent 案例輸出,解析成逐筆 7 欄結構。
3. 整理成 `input.json`(結構見下),交給腳本。
4. 執行:
   ```bash
   python scripts/export_test_cases.py \
     --template assets/Test_Case_Template_Claude.xlsx \
     --input input.json \
     --outdir /mnt/user-data/outputs
   ```
5. 腳本套模板、填 Report、填 Test case、產檔名、存 outputs,印出最終路徑。
6. 用 `present_files` 交付,並回報:檔名、案例筆數、Report 哪些格有填/留白。

### input.json 結構

```json
{
  "jira": {
    "assignee": "VernaChen",
    "summary": "[UAT-QA][EU] Customized Bundle maintenance mechanism enhancement",
    "module": "CRM",
    "link": "https://jira.example.com/browse/XXXX-1234",
    "mcc": "123",
    "test_date": "Internal Testing: 2026/07/20 ~ 2026/07/24",
    "test_environment": "Staging / EU PROD-like",
    "test_version": ""
  },
  "test_cases": [
    {
      "id": "1",
      "category": "Cart",
      "pre_condition": "(1) No special pre-condition",
      "test_case": "Verify bundle discount applies correctly",
      "steps": "1. Add bundle to cart\n2. Open cart",
      "expected_result": "1. Bundle added\n2. Discount shown correctly",
      "test_data": "網站=EU\n角色=已登入會員*1"
    }
  ]
}
```

- `id` 沿用來源流水號;未給則腳本自 1 順編。
- `test_date` / `test_environment` 抓不到給空字串 → 腳本留白並回報。
- `feature` 一律不進 input(放了也忽略)。
- `module`(選填):檔名第二段;Summary 已含模組可省略。
- `test_data`(選填):有值才寫 L 欄;無值整欄留白,**不代為產生**。
- Report C2 取自 `summary`,**無 `project` key**。

## 版本紀錄

- **v1.1.1**:修正兩處內部矛盾 —— 邊界段 Tester 前綴 `AOCCQA_` → `AOCC_`(與對應規則一致);輸入段移除已不使用的 `Project name`。內文 caveman 壓縮,行為不變。
- **v1.1.0**:① `Test Data` 條件寫入 L 欄(有給就寫、沒給留白,不臆造),回報 `test_data_filled`/`test_data_blank`;② 檔名支援 `[市場][模組]` 多段前綴(可由 `jira.module` 補入、不重複),後綴改 **`TestCase` 無空格**;③ Tester 前綴定案 **`AOCC_{Assignee}`**;④ 補版本號、移除 input.json 未用的 `project` key。
- **v1.0.0**:初版。Report 七欄對應、Test case A/E/F/G/H/I 對應、Feature 捨棄、公式與 Bug list／Screenshot 保留、檔名 `_Test Case_{YYYYMMDD}`。

## 邊界與回報

**動態欄位一律「抓得到就填、抓不到留空並回報,絕不臆測」。** 腳本輸出兩份清單:
- `report_captured`:實際填入的欄位、儲存格、值。
- `report_blank`:留空的欄位與儲存格(需人工補或校正抓取關鍵字)。

動態欄位:Project(Summary)、Test date、Test Country(MCC#)、Test Environment、New feature & Release Note(link)、Tester(`AOCC_{Assignee}`)、Test Version。回覆時如實轉述兩份清單,明講哪些抓到、哪些沒抓到。

停止/邊界條件:
- Description 抓不到時程 / 環境 → 該格留空 + 回報,**不**臆測填值。
- Assignee 缺失 → Tester 留模板預設值 + 回報。
- **案例 0 筆 → 停止並回報,不產空檔。**
- **案例 > 200 筆 → 停止並回報(模板公式範圍上限)。**
- **模板檔缺失 → 停止並回報,不自建替代模板。**
