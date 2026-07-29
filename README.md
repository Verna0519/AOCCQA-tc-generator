# AOCCQA Skills

ASUS 內部電商平台（EC / Magento）QA 測試案例產線的 Claude skills。

這條產線把一份需求，一路變成可交付的測試案例檔：**解析需求 → 規劃覆蓋並起草測項 → 審查刪重補漏 → 套官方模板匯出 xlsx**。本 repo 收錄產線中段負責「起草」與末段負責「匯出」的兩個 skill，其餘階段各自獨立維護、不含在本 repo。

## 產線位置

```
aoccqa-fsd-parser        解析 FSD／需求、找缺口            （本 repo 未含）
        │  需求分析報告（六段 HTML）／ Requirement Matrix
        ▼
aoccqa-tc-generator      規劃覆蓋 ＋ 起草 7 欄測項初稿      ◀── 本 repo
        │  Test Case Draft（7 欄 Markdown）
        ▼
aoccqa-quality-reviewer  審查、刪重、補漏、判讀            （本 repo 未含）
        │
        ▼
aoccqa-case-exporter     套官方模板 → 匯出 xlsx 交付檔      ◀── 本 repo
```

上下游透過**固定的 7 欄結構**銜接，因此中段換人、換工具都不影響交付格式。

## 本 repo 收錄

| Skill | 版本 | 角色 | 做什麼 | 不做什麼 |
|---|---|---|---|---|
| [`aoccqa-tc-generator`](aoccqa-tc-generator/SKILL.md) | v1.4.1 | Phase B・案例起草員 | 把已確認需求展開成 7 欄 Test Case Draft，並自我標記假設／Blocked／疑似重複 | 不刪除或合併既有案例、不判 Pass/Fail、不填 Test result、不做最終核准、不重跑 Phase A 解析 |
| [`aoccqa-case-exporter`](aoccqa-case-exporter/SKILL.md) | v1.2.0 | 共用工具 | 把測試案例＋Jira 單搬進官方 xlsx 模板匯出 | 不判斷、不篩選、不改寫案例內容；公式與 Bug list／Screenshot 分頁原樣保留 |

版本號同時記於各 `SKILL.md` 的 frontmatter `metadata.version` 與內文「版本紀錄」段落。

### 7 欄 Test Case Draft 契約

兩個 skill 之間的介面。欄序固定、不增減、不改名：

```
Test Case ID | Category | Feature | Pre-condition | Test Case | Steps | Expected Result
```

外加一個選填的延伸欄 `Test Data`（不屬鎖定 7 欄）。

**匯出時的欄位分工**（`case-exporter` 實際行為）：

- 內容欄寫入 5 欄 → `Category`、`Pre-condition`、`Test Case`、`Steps`、`Expected Result`。
- `Test Case ID` → 寫入模板 A 欄，沿用草稿來源 ID、缺才自 1 順編。
- `Test Data`（選填）→ 有非空值才寫入 L 欄，否則留白，**絕不代為產生**。
- `Feature` → 供 reviewer 判讀決定性條件組合，匯出時捨棄不寫。
- `Report` 分頁的 **Test date 由腳本正規化**為 `YYYY/MM/DD-YYYY/MM/DD`（自動去除 Internal Testing／UAT 等字樣）；抓不到的動態欄一律留白並回報，絕不臆測。

## 目錄結構

```
aoccqa-tc-generator/
  SKILL.md
  references/coverage-and-examples.md          # 覆蓋維度目錄與完整範例（需要時才載入）
aoccqa-case-exporter/
  SKILL.md
  assets/Test_Case_Template_Claude.xlsx        # 官方 xlsx 模板（二進位交付物，勿改）
  scripts/export_test_cases.py                 # 匯出腳本
```

## 使用

各 skill 資料夾可直接作為 Claude skill 安裝。`aoccqa-case-exporter` 的腳本需要 Python 與 `openpyxl`：

```bash
pip install openpyxl
python aoccqa-case-exporter/scripts/export_test_cases.py \
  --template aoccqa-case-exporter/assets/Test_Case_Template_Claude.xlsx \
  --input input.json \
  --outdir ./out
```

`input.json` 的結構（Jira 欄位 ＋ 測試案例陣列）見 [`aoccqa-case-exporter/SKILL.md`](aoccqa-case-exporter/SKILL.md)。腳本會印出交付檔路徑、案例筆數、以及 Report 分頁「哪些格有填／留白」的回報清單。

## 慣例

- 每次產生或更新 skill 都在 frontmatter 帶 `metadata.version`，並於內文「版本紀錄」補一筆。
- `SKILL.md` 內文採 caveman 壓縮以降低載入 token；安全／停止條件維持完整清楚句子，不做 fragment 化。
- `assets/` 內的 `.xlsx` 模板為二進位交付物，請勿在文字編輯器改動。
