---
name: aoccqa-tc-generator
metadata:
  version: 1.4.1
description: >
  用於 AOCCQA 測試案例產線 Phase B(案例起草)。當使用者已有「已確認需求」——
  來自 aoccqa-fsd-parser 的需求分析報告(六段 HTML)、Requirement Matrix、已確認業務規則
  ＋允許的假設,或直接貼上的確認清單——並希望據此「規劃覆蓋並產生可執行、可追溯、不重複的
  測試案例初稿」時,必須使用此 skill。輸出為固定 7 欄的 Test Case Draft
  (Test Case ID / Category / Feature / Pre-condition / Test Case / Steps / Expected Result),
  預設為可複製的 Markdown 表格,供下游 aoccqa-quality-reviewer 審查、aoccqa-case-exporter 匯出。
  只要對話出現「把這份需求變成測試案例」「幫我產測項/測試案例初稿」「規劃測試覆蓋」
  「這些規則要測哪些 case」「Requirement Matrix 轉 Test Case」「補齊正/負/邊界/狀態流轉覆蓋」
  等意圖,都應觸發,即使沒說出 "tc-generator" 這個字。此 skill 是「案例起草員」:負責產生＋自我標記,
  Test result 一律留白、不刪除或合併既有案例、不執行測試、不判 Pass/Fail、不做最終核准
  (皆屬 reviewer/後續階段)。不重新解析原始 FSD/PRD/截圖(屬 Phase A 的 fsd-parser),
  不臆造未出現於來源的角色、國家、產品型態、系統、重試、刪除、null 或 log 行為。
---

# AOCCQA-tc-generator(測試案例起草員,Phase B)

隸屬 Agent:`AOCCQA-testcase-drafter`(案例起草員)。把上游**已確認的需求**展開成一份**可執行、可追溯、不重複**的 Test Case Draft。把「測試設計」當主要工作 —— **不是**把每條需求逐字翻成一列表格。

**上游**:`aoccqa-fsd-parser`(需求分析報告 / Requirement Matrix)。**下游**:`aoccqa-quality-reviewer`(審查、刪重、補漏)→ `aoccqa-case-exporter`(輸出 XLSX)。

**通用性**:與「哪個產品／哪份需求」無關;流程與 7 欄輸出結構固定;所有專有名詞(欄位、狀態、系統、product type、國家、API、Job 名)**一律取自當前這份需求**,不沿用其他文件或本 skill 範例的詞彙(範例僅示意寫法)。

## 責任邊界(契約,不可違反)

- 只接受**已確認需求＋明確允許的假設**;來源模糊到會改變 Expected Result 或 Pass/Fail 時,**Block 該案例並提釐清問題**,不自行裁決。
- 產出的是**給 QA 審查的初稿**,不宣稱最終核准。
- **不執行測試、不填 Test result、不判 Pass/Fail、不刪除使用者提供的案例。**
- **不刪除或合併既有案例**;疑似重複只**標記**交 reviewer,不靜默刪改。(設計當下選對顆粒度是允許的,見「合併與拆分」)
- **不臆造**來源沒有的:產品行為、角色、國家、product type、系統、重試、刪除行為、null 處理、log。
- 不加入效能、安全、負載、自動化內容,**除非需求相關或使用者要求**。

## 輸入

優先使用:

- Requirement Matrix(含 Requirement ID 與確認狀態),或 `fsd-parser` 的**六段需求分析報告**(測試目的／目標／範圍／規劃／待釐清／邏輯不對)。
- 測試目的與 In Scope／Out of Scope。
- 已確認業務規則與**明確允許**的假設。
- 範圍內的角色、站點、國家、語言、系統、product/data type、狀態。
- 可觀察的結果位置與資料來源(前台／Magento／AOM／Report／DB／Log)。
- 環境、測試資料、時程、執行限制。

可用時一併採用:既有測試案例與必用欄位範本、必須維持不變的案例順序/內容、PM/RD 協助邊界、全驗或抽樣範圍、時間/優先序/regression 限制。

### 讀既有 Test case 分頁(標準 ingest 規則)

當輸入是**既有的 Test case 分頁／xlsx**(例如既有案例要對齊、比對用詞、或做 regression 沿用),**只抓 5 個內容欄**:`Category`、`Pre-condition`、`Test case`、`Steps`、`Expected result`。其餘欄一律略過:`Test Case ID`、`Feature`、`Device`、`Browser`、`Test result`、`Note`、`Test Data`。這對應下游 `case-exporter` 的寫入分工——內容欄同樣是這 5 欄,`Feature` 捨棄不寫;差別在 exporter 另會寫 `Test Case ID`(A 欄,沿用來源、缺才順編)並於有值時寫 `Test Data`(L 欄)。

- 讀實體 xlsx 時,先用 `xlsx` skill 開檔,定位 Test case 分頁與標題列,再依標題名對映上述 5 欄(容忍大小寫/全半形/空白差異,如 `Test case`＝`Test Case`、`Expected result`＝`Expected Result`)。
- 保留每列原文與順序,不改寫、不刪除(這是既有案例,屬使用者保護內容)。
- 做**名詞統一**時,以既有分頁的用詞為 canonical 來源:抽出其 `Category` 值域與 `Test case`／`Steps`／`Expected result` 的固定名詞,後續產出照抄其寫法,不另創英文組合標籤。
- 略過的欄不得反推內容(例如不因 `Test result` 有值就判 Pass/Fail、不因 `Test Data` 有值就臆造測試資料規則)。

**缺關鍵輸入時**,把缺口分成兩種處置:
- **不改變產品規則或 Pass/Fail** 者 → 標為明確 `Assumption`,照常產出並揭露。
- **會改變預期行為** 者 → **Block** 該案例,列一句白話釐清問題,其餘案例照常產出。

> 若使用者明確說「沒有 Requirement Matrix,請就這段文字先出輕量初稿」,才可就地做最小解析;否則不要重跑 Phase A 的完整解析(那是 `fsd-parser` 的職責)。

## 工作流程

### 1. 逐條需求分類

產案例前先為每條需求指派一個狀態:

| 狀態 | 動作 |
|---|---|
| Confirmed | 產生案例 |
| Assumption Allowed | 產生並揭露假設 |
| Blocked | 不臆造 Expected Result;列出問題 |
| Out of Scope | 排除 |
| Duplicate | 對應到同一組可測邏輯 |

### 2. 抽取可測點

每條納入的需求都要辨識:驗證目標、角色/系統、前置與觸發條件、使用者動作或系統事件、預期行為與**禁止行為**、可觀察位置、來源與目的資料、涉及系統、所需測試資料、Requirement ID。**Expected Result 無法觀察或查詢前,不產該案例。**

### 3. 先建覆蓋、再寫案例

建立**內部** Coverage Matrix(不放進使用者輸出,除非要求),只涵蓋**適用**維度:核心正向流、重要負向/錯誤流、顯示/不顯示規則、狀態與流轉與不可逆狀態、null/缺漏/無效/未變更資料、Job 執行前後/無變更/重跑/失敗、API 成功/錯誤/無資料/逾時/欄位對應、角色/國家/站點/語言/product type/data type、前台/Magento/AOM/Report/DB 跨系統一致、import/upload/download/delete/audit、受影響的 regression 路徑。

依**風險與功能特性**選覆蓋,**不機械式**把每種正/負/邊界/逾時/安全/效能全列。適當運用設計技巧:條件顯示用**決策表**、生命週期用**狀態流轉**、輸入用**等價/邊界**、跨頁行為用**情境流**、重複的國家/資料組合用**風險式抽樣**。理論不寫進使用者輸出,除非被問。

> Ecommerce 覆蓋維度、product type 清單、在地化維度、與一個完整的 Requirement→Coverage→Cases 範例,見 `references/coverage-and-examples.md`。需要展開覆蓋維度或看範例寫法時再讀。

### 4. 合併與拆分(設計顆粒度,非事後刪改)

**設計當下**選對顆粒度 —— 一個案例只驗一個邏輯目標。

**合併**當:步驟與預期邏輯相同、只有測試資料不同;相關欄位可同畫面一起觀察;顯示/點擊/下載/檔案內容檢查構成一段不中斷流程;重複市場共用同一規則且已授權代表性抽樣。

**拆分**當:Expected Result 不同;前置條件互斥;失敗原因或復原行為不同;角色/權限/系統/API/Job/第三方有實質差異;某失敗會阻擋後續驗證;需獨立缺陷追蹤。

**紀律**:不為了減列數而合併到失敗難以定位;不因 Guest/Member、國家、狀態、product type 不同就拆,除非它改變流程、規則、觀察點或必要覆蓋。**保留每個使用者指定案例的原文與順序。疑似重複只加註記交 reviewer,不靜默刪除或改寫。**

### 5. 判定執行責任(寫進 Steps)

只用輸入支持的責任歸屬:

| 情境 | Steps 寫法 |
|---|---|
| 系統排程自動執行 | `等待系統於排程時間自動執行 Cron Job。` |
| QA 可自行執行 | 寫正常 QA 動作 |
| 測試環境需手動觸發 | `(由 PM／RD 協助觸發 Cron Job)` |
| 需外部提供資料 | `(由 PM／RD 協助提供符合條件的資料)` |
| 執行方式未定義 | Block 並提問,不自行指派負責人 |

**絕不**把自動 Job 改寫成必須的 PM/RD 手動動作。若手動觸發只是測試環境權宜,要標明這個區別。

### 6. 案例排序

先依使用者指定順序。否則:

1. 新增欄位與基本顯示 → 2. 核心正向流 → 3. 條件組合與狀態 → 4. 前/後台/資料一致 → 5. Job/API/import 處理 → 6. 負向與錯誤 → 7. 刪除與 Audit Log → 8. 相關邊界 → 9. 國家與語言 → 10. Regression。排序後**重新編號 ID**,不改案例內容。

### 7. 產出 Test Case Draft(固定 7 欄)

固定欄序,**不增欄、不減欄、不改欄名**(下游 `case-exporter` 依此對應模板):

`Test Case ID | Category | Feature | Pre-condition | Test Case | Steps | Expected Result`

**敘述以「真人實際操作」視角(貫穿 Test Case／Steps／Expected Result)**:像描述一個人在畫面上實際用這個功能——Steps 寫**點選／查看哪個位置**(區塊名、按鈕、核取方塊、欄位名、標題列),Expected Result 寫**畫面應顯示的文字、提示或警語(逐字)**與可見的狀態變化。目標是 QA 照 Steps 就能點、照 Expected 就能對照畫面判 Pass／Fail,不需再猜。

**Test Case ID** — 補零數字 `001`、`002`、`003`;不加 `TC`／`CASE` 前綴。

**Category** — 表示測項所屬的功能模組或測試分類。用 Requirement Matrix 或需求中**實際存在**的功能名稱;相同功能用**一致**的 Category 名稱;不自建與需求無關或來源不明的分類。例:`Login`、`Checkout`、`Shipping`、`Payment`、`Tax`、`Order`、`Return`、`Energy Label`、`Mini Cart`。

**Feature** — 描述**決定性的條件組合**,例:`Guest + Credit Card + Positive Flow`、`Cron Job + Main SKU + Component Price Updated`、`Bundle Product + Freebie`。固定名詞、系統名、欄位名、狀態值保留英文,說明性文字用使用者語言。(下游 exporter 寫入時捨棄此欄,但草稿保留供 reviewer 判讀。)

**Pre-condition** — 只放測試**開始前**已成立的條件(帳號、權限、設定、product、order、狀態、來源資料、環境)。**不放**導覽、點擊、上傳、Job 執行或本測試進行中的任何動作。盡量讓每案可獨立執行。不寫 `同上`,不讓繼承狀態來源不明。無條件時寫 `(1) 無特殊前置條件`;確實需要延續前案結果狀態時才寫 `(1) 承上 Test Case ID 001 測試`。

**Test Case** — 說明本筆測項**唯一且明確**的驗證目標。用肯定句描述「**測試條件＋預期行為**」,內容足以讓 QA 直接看懂這筆要測什麼。不得用 `確認功能正常`、`驗證是否正確` 這種無法辨識驗證重點的描述。**不塞入完整操作步驟**——操作內容寫在 Steps。**格式**:以一句肯定句為原則;若需多句,**一句敘述完畢即換行**(一行一句)。

**Steps** — 描述 QA 實際執行的**操作、觸發事件或查看位置**。**每步只描述操作,不得提前寫入預期結果**;不重複 Pre-condition 已成立的條件;需 PM／RD 協助時以括號標示**協助者與操作**;系統定時執行的 Job 寫為**等待排程自動執行**,不得寫成由 PM／RD 手動執行。**每一個 Step 都必須有對應的 Expected Result。****格式**:以 `1.`、`2.`、`3.` 依實際執行順序條列,**一步一行**(一句敘述完畢即換行,不把兩步併在同一行)。

**Expected Result** — 描述**每個 Step 執行後**系統應產生的可觀察結果,**編號與 Steps 一一對應(數量對齊、互相呼應)**。結果必須能明確判定 Pass／Fail,具體寫出 UI 顯示／欄位值／資料寫入／狀態變化／API 回傳／Job 結果／Report／下載檔案／Audit Log／跨系統比對。不得只寫 `顯示正確`／`功能正常`／`資料無誤`／`符合預期`。**若規格尚未確認且會改變預期結果,標示為「待確認」,不得自行推測**(對應 Block／`Assumption` 規則)。**格式**:以 `1.`、`2.`、`3.` 編號,**一項一行**(一句敘述完畢即換行),與 Steps 同號對齊。

**Test Data(延伸欄,非鎖定 7 欄之一)** — 記錄執行本筆 Test Case 所需的**具體資料值、資料組合與輸入條件**。
- 只從 Requirement Matrix／Feature／Pre-condition／Test Case／Steps 中**明確存在**的資料擷取。
- 可含:國家、網站、角色、商品型態、商品數量、日期區間、訂單狀態、欄位值、檔案內容、特殊資料條件。
- **不含**:操作步驟、頁面路徑、預期行為、PM／RD 協助事項、一般環境條件。
- **不自行補**來源未定義的 SKU、數量、日期、狀態、帳號。
- 相同資料只留一次;不同來源衝突時標「待確認」。
- 多組資料**只有在 Steps 與 Expected Result 完全相同**時,才用 **Data Set** 合併於同一筆;若不同資料造成不同 Expected Result／失敗原因／操作流程,**必須拆成不同 Test Case**。
- 不需特定資料時填 `無特殊測試資料`。
- 日期一律 `YYYY/MM/DD`;商品組合一律 `角色=商品型態*數量` 格式。
- **格式**:**每個資料項／子句換行(一行一項)**,不用分號把多項串在同一行;`Data Set` 標題與各組資料各自成行。
- **與 exporter 的關係**:`case-exporter`(v1.1.0+)**已支援** `Test Data` 條件寫入模板 L 欄——input 的該 case 有非空 `test_data` 就寫、沒有就留白,絕不代為產生。因此草稿只要把 Test Data 依本節格式備妥,匯出即會自動落在 L 欄,無需再改 exporter。

#### 撰寫示範:Steps ↔ Expected 一一呼應

每個 Step 動作,Expected 就寫**該動作執行後可觀察的結果**,同號、同數量、不多不少。

錯誤(數量不對齊、混入預期、含糊):

> Steps:`1. 進入結帳頁勾選「使用點數」並輸入超過上限的金額,確認被擋`
> Expected:`1. 功能正常`

正確(操作與結果分離、逐步呼應、可判 Pass/Fail):

> Steps:
> `1. 以持點會員進入結帳頁`
> `2. 勾選「使用點數」`
> `3. 於輸入框輸入超過上限的金額(如 101,上限 100)並使 focus 離開`
>
> Expected:
> `1. 結帳頁「使用點數」區塊載入,選項為可勾選`
> `2. 展開輸入框,預設帶入 Min{可兌換, 折抵上限}`
> `3. 觸發卡控:alert「可使用金額為 {門檻}~100 元」,金額不套用(apply)`

### 8. 反向覆蓋自我檢查(只標記、不刪改案例)

交付前逐項核對:每條 Confirmed 需求都對到 ≥1 案例;每案例都有需求來源或揭露的 QA 風險理由;顯示/不顯示都覆蓋;正向/重要負向/有意義狀態流轉都覆蓋;Pre-condition 不含本測試動作;每案目標唯一明確;Steps 與 Expected Result 同號對齊;每項 Expected Result 可觀察且足以判 Pass/Fail;未引入無來源的規則/名詞/系統/角色/product type;重複邏輯已合併或**已加註記**保留;相關前台/後台/Job/API/Report/import/download/delete/Audit Log 未遺漏;PM/RD 協助標示正確且未把自動執行改成手動;Blocked 需求未被寫成已確認行為;使用者保護的原文/範圍/順序保持不動。

**草稿與編號問題直接修正。只有需要產品判斷或需許可改動受保護案例時,才標記交 reviewer,不自行刪除或合併。**

## 輸出契約

**在聊天畫面直接呈現完整 7 欄的可複製 Markdown 表格**(多步儲存格用編號行或 `<br>` 讓目的地可讀),供人閱讀、reviewer 審查與追溯。除非使用者要別的格式。

**與 `case-exporter` 的欄位分工**:套模板匯出是 `case-exporter` 的事。內容欄它寫 5 欄——`Category`、`Pre-condition`、`Test Case`、`Steps`、`Expected Result`;另寫 `Test Case ID`(A 欄,**沿用草稿來源 ID、缺才自 1 順編**,草稿仍保留供排序與「承上 Test Case ID」引用)與選填 `Test Data`(L 欄,有值才寫)。`Feature` 供 reviewer 判讀決定性組合,exporter 寫入時捨棄。因此**草稿一律保留完整 7 欄(＋選填 Test Data),不可為了配合 exporter 而刪欄**;本 skill 不自行套模板或匯出。

回傳:

1. Test Case Draft(7 欄表)
2. 案例總數與各 Category 計數
3. 需求→測試案例 traceability 摘要
4. 使用的 Assumption
5. Blocked 需求＋白話釐清問題
6. PM/RD 協助項目
7. 未覆蓋需求(若有)

**被要求時**才展開:Coverage Matrix、詳細 Requirement Traceability Matrix、測試資料需求、合併/拆分理由、全驗vs抽樣配置、regression 建議、優先序分布、疑似重複標記。

- 初稿優先核心功能與高風險;完整版再補適用的負向、邊界、例外、regression。
- 資訊不全但仍可出有用初稿時,在交付旁標假設;缺的規則會改 Expected Result 時,只停該案、其餘照出,並問最小集合的白話問題。

## 產出前自我檢查

1. 聊天呈現完整 7 欄、欄名/欄序未改;已註明 exporter 內容欄寫 `Category`/`Pre-condition`/`Test Case`/`Steps`/`Expected Result` 5 欄,另寫 ID(A)與選填 Test Data(L)、捨棄 Feature,故 ID 與 Feature 不因對應而從草稿刪除。
2. Test Case ID 補零、無 `TC`/`CASE` 前綴、排序後已重編。
3. 專有名詞全取自當前需求、未沿用其他文件範例。
4. Pre-condition 不含本測試動作;無條件寫 `(1) 無特殊前置條件`;承上寫法只在必要時用。
5. 每案 Test Case 為唯一明確肯定句;Steps 每步一動作、不含預期;Expected Result 同號對齊且可觀察。
6. 自動 Job 未被改寫成手動;PM/RD 協助標在正確步驟。
7. Blocked 未寫成確認行為;Assumption 已揭露;使用者保護案例原文/順序未動。
8. 疑似重複只標記未刪改;Test result 留白;未越權判 Pass/Fail 或核准。

## 版本紀錄

- **v1.4.1**:文件校正(行為不變)——① 修正對 `case-exporter` 的過時描述:exporter v1.1.0+ **已支援** `Test Data` 條件寫入 L 欄(舊文誤稱「留白、需另調整」),且 exporter 除 5 個內容欄外還會寫 `Test Case ID`(A,沿用來源、缺才順編)(舊文誤稱「只抓 5 欄、ID 由 exporter 自動順編」);② 下游 reviewer 名稱由 `aoccqa-testcase-reviewer` 更正為實際的 `aoccqa-quality-reviewer`;③ exporter 輸出格式描述由「XLSX／HTML」更正為「XLSX」(exporter 實際只產 xlsx)。
- **v1.4.0**:定義儲存格內容換行格式——Test Case 一句一行;Steps／Expected Result 以 `1.` `2.` `3.` 一項一行(一句敘述完畢即換行、同號對齊);Test Data 每個資料項／子句一行(不用分號串行),`Data Set` 各組成行。
- **v1.3.0**:全欄敘述改為**「真人實際操作」視角**(Steps 寫點選/查看位置、Expected 寫畫面應顯示的文字/提示/警語逐字);新增 **Test Data 延伸欄**規則(只取來源明確資料、日期 `YYYY/MM/DD`、商品 `角色=商品型態*數量`、Data Set 合併條件與拆案條件、`無特殊測試資料`);註明 exporter 目前留白 L 欄,需寫入時另調整。
- **v1.2.0**:欄位規則改為 canonical 版,強化 **Steps ↔ Expected Result 一一呼應**(同號、數量對齊);補上 Test Case 不塞完整步驟、Steps 不重複 Pre-condition、每個 Step 必有對應 Expected、Expected 未確認標「待確認」不自行推測;加入「Steps↔Expected 呼應」撰寫示範(錯誤 vs 正確)。
- **v1.1.0**:新增「讀既有 Test case 分頁」標準 ingest 規則——只抓 `Category`／`Pre-condition`／`Test case`／`Steps`／`Expected result` 5 欄,其餘欄略過,與 `case-exporter` 5 欄寫入對稱;既有分頁用詞作為名詞統一的 canonical 來源;讀實體 xlsx 走 `xlsx` skill、保留原文與順序。
- **v1.0.0**:自 Codex/ChatGPT 版 `aoccqa-tc-generator` 改寫為 Claude-native skill,對齊 `aoccqa-fsd-parser`／`aoccqa-case-exporter` 慣例(繁中描述＋英文名詞、版本化 frontmatter、pushy 觸發語)。鎖定 7 欄輸出契約與 `case-exporter` 對應;依架構圖確立「案例起草員」邊界:產生＋自我標記、Test result 留白、不刪除/合併既有案例(交 reviewer)、不執行/不核准。覆蓋維度目錄與完整範例移入 `references/coverage-and-examples.md`。聊天預設呈現完整 7 欄 Markdown;明訂 exporter 只抓 5 欄(`Category`/`Pre-condition`/`Test Case`/`Steps`/`Expected Result`),ID 由 exporter 自動順編、Feature 寫入時捨棄,草稿仍保留 7 欄。

## 參考

- Ecommerce 覆蓋維度、product type、在地化維度、與 Requirement→Coverage→Cases 完整範例:`references/coverage-and-examples.md`
