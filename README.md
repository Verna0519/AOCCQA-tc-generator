# AOCCQA Skills

ASUS 內部電商平台 QA 測試案例產線的 Claude skills。將 FSD／需求解析成測試需求、產出可執行測試案例初稿、並匯出成 AOCC QA 官方 xlsx 交付檔。

## 產線位置

```
aoccqa-fsd-parser        解析需求、找缺口          （本 repo 未含）
        ↓ Requirement Matrix / 需求分析報告
aoccqa-tc-generator      規劃覆蓋 ＋ 產生測項初稿   ← 本 repo
        ↓ Test Case Draft（7 欄）
aoccqa-testcase-reviewer 審查、刪重、補漏          （本 repo 未含）
        ↓
aoccqa-case-exporter     套官方模板 → 匯出 xlsx     ← 本 repo
```

## 本 repo 收錄

| Skill | 版本 | 角色 |
|---|---|---|
| `aoccqa-tc-generator` | v1.4.0 | Phase B 案例起草員：把已確認需求展開成 7 欄 Test Case Draft（產生＋自我標記；不刪除/合併、不判 Pass/Fail、不核准） |
| `aoccqa-case-exporter` | v1.1.1 | 共用工具：把測試案例＋Jira 單套進官方 xlsx 模板匯出（只搬運套版，不判斷、不篩選、不改寫） |

版本號記錄於各 `SKILL.md` 的 frontmatter `metadata.version` 與「版本紀錄」段落。

## 目錄結構

```
aoccqa-tc-generator/
  SKILL.md
  references/coverage-and-examples.md
aoccqa-case-exporter/
  SKILL.md
  assets/Test_Case_Template_Claude.xlsx   # 官方 xlsx 模板（二進位，勿改）
  scripts/export_test_cases.py            # 匯出腳本
```

## 使用

各 skill 資料夾可直接作為 Claude skill 安裝。`aoccqa-case-exporter` 需要 Python 與 `openpyxl`：

```bash
pip install openpyxl
python aoccqa-case-exporter/scripts/export_test_cases.py \
  --template aoccqa-case-exporter/assets/Test_Case_Template_Claude.xlsx \
  --input input.json \
  --outdir ./out
```

## 慣例

- 每次產生或更新 skill 都在 frontmatter 帶 `metadata.version` 並維護「版本紀錄」。
- `SKILL.md` 內文採 caveman 壓縮以降低載入 token；安全/停止條件維持完整清楚句子，不做 fragment 化。
- `assets/` 內的 `.xlsx` 模板為二進位交付物，請勿在文字編輯器改動。
