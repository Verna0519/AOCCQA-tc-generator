# 覆蓋維度目錄與範例

本檔是 `aoccqa-tc-generator` 的展開參考。**只在需要展開覆蓋維度、確認要考慮哪些 product type / 在地化面向、或需要範例寫法時才讀。** 所有維度**僅在需求相關時**採用 —— 不機械式地把每個維度都生一條案例。

---

## 1. 測試設計類別(適用才用)

規劃覆蓋時,對照下列類別檢查有無遺漏,只挑與需求風險相關者:

Positive Flow、Negative Flow、Boundary Value、Error Handling、Exception Flow、Permission Validation、State Transition、Data Validation、Integration Testing、Regression Testing、UI Validation、Backend Validation、API Validation、Scheduled Job Validation、Localization、Compatibility、Performance Risks、Security Risks。

- Performance(Response Time / Loading / Concurrency / Large Data / Timeout / Retry)與 Security(Permission / Authentication / Authorization / Sensitive Data / Input Validation / Injection)**只在需求相關或使用者要求時**才納入。

## 2. Ecommerce 觀察位置(適用才驗)

案例的 Expected Result 應落在可觀察的系統位置。常見:

Frontend UI、Magento、AOM／ECMW、My Account、Checkout、Cart、Order Detail、Order Email、Reports、Cron Jobs、Background Jobs、API Response、Database Update(適用時)、System Logs(適用時)。

> 跨系統一致性(例:前台顯示 vs Magento 值 vs AOM 值 vs Report 值)是高價值案例,但**只在需求牽涉多系統時**才設。

## 3. Product Type(需求涉及商品時考慮)

Simple、Configurable、Bundle、Add-on、Freebie、Gift、WEP、Virtual、Downloadable。

- 只在**該 product type 會改變流程、規則、觀察點或必要覆蓋**時才分開設案例;否則用代表性抽樣,不逐型複製。

## 4. Country / Localization(適用才考慮)

Different Countries、Different Languages、Currency、Regional Configuration、Localized Text、Tax Rules、Shipping Rules、Payment Methods。

- 重複市場共用同一規則時,授權下用**風險式抽樣**挑代表市場,不逐國複製整組案例。

## 5. Regression / 自動化 / 風險標註(被要求時展開)

需要時可另外標:Smoke 候選、Regression 候選、Automation 候選(優先:核心流程、高頻、穩定功能、regression suite)、高風險情境、關鍵業務流。**預設不塞進 7 欄主表**,除非使用者要這些欄位或另開清單。

---

## 6. 完整範例:Requirement → Coverage → Cases

> 以下為**示意寫法**,詞彙是假想需求,不是固定清單。實作時所有名詞改抓當前需求。

### 假想需求(已確認)

- R1(Confirmed):Bundle 商品在 Cart 顯示組合折扣;折扣 = 各 component 原價總和 − Bundle 售價。
- R2(Confirmed):Bundle 內任一 component 缺貨時,整組 Bundle 在 Cart 標記 `Out of Stock` 且不可結帳。
- R3(Assumption Allowed):折扣顯示幣別依站點設定(使用者授權「先假設沿用站點預設幣別」)。
- R4(Blocked):component 價格於下單後、出貨前變動時,已成立訂單的折扣是否重算 —— 來源未定義。

### 內部 Coverage(不進使用者輸出)

| 維度 | 是否覆蓋 | 說明 |
|---|---|---|
| 正向:折扣計算與顯示 | ✓ | R1 |
| 顯示位置一致 | ✓ | Cart 顯示 vs 計算值 |
| 負向/狀態:缺貨阻擋結帳 | ✓ | R2 狀態流轉 |
| 在地化:幣別 | ✓(抽樣) | R3,標假設 |
| 訂單後價變重算 | ✗ Block | R4 → 提問,不臆造 |

### 產出(7 欄,節錄)

| Test Case ID | Category | Feature | Pre-condition | Test Case | Steps | Expected Result |
|---|---|---|---|---|---|---|
| 001 | Cart | Bundle Product + Positive Flow | (1) 已建立含 ≥2 component 的 Bundle 商品,component 原價與 Bundle 售價已設定 | 驗證 Bundle 加入 Cart 後顯示的組合折扣等於 component 原價總和減 Bundle 售價 | 1. 將該 Bundle 加入 Cart<br>2. 開啟 Cart 頁 | 1. Bundle 成功加入 Cart<br>2. Cart 顯示折扣金額 = 各 component 原價總和 − Bundle 售價,數值相符 |
| 002 | Cart | Bundle Product + Component Out of Stock | (1) 承上 Test Case 001 測試,將其中一個 component 設為缺貨 | 驗證 Bundle 內任一 component 缺貨時整組標記 Out of Stock 且無法結帳 | 1. 開啟含缺貨 component 的 Bundle 所在 Cart 頁<br>2. 嘗試進入結帳 | 1. 該 Bundle 顯示 `Out of Stock` 標記<br>2. 無法進入結帳,系統阻擋且顯示對應提示 |
| 003 | Cart | Bundle Product + Localization(Currency) | (1) 站點幣別設定為該市場預設幣別 | 驗證 Bundle 折扣金額以站點預設幣別顯示(假設:沿用站點預設幣別) | 1. 於該站點將 Bundle 加入 Cart<br>2. 開啟 Cart 頁 | 1. Bundle 成功加入 Cart<br>2. 折扣金額以站點預設幣別符號與格式顯示 |

- **Assumption 揭露**:003 依「沿用站點預設幣別」假設(R3),PM/RD 確認後才轉 Confirmed。
- **Blocked＋問題**:R4 未產案例。需 PM 釐清:「Bundle 內 component 價格於**下單後、出貨前**變動時,已成立訂單的組合折扣是否**重算**?若重算,以哪個時點的價格為準、是否回寫 Order 與 Report?」
- **Traceability**:R1→001、R2→002、R3→003、R4→(Blocked,無案例)。
