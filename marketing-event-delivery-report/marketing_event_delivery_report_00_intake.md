# Marketing Event Delivery Report — Intake Decision Record

- 日期：2026-07-14
- 平台：Agent BO 與 Admin BO（入口均已確認）
- 目標：新增一份唯讀報表，供 BO 使用者查詢 Meta Conversion API、Adjust API 與 CleverTap API 的事件發送紀錄，預設只顯示成功紀錄，並可切換查詢失敗紀錄。
- Quick Spec 判定：適用。單一目標、單一報表頁面，預估不超過三項相關行為變更；不含事件重送、第三方設定修改或既有事件觸發邏輯改造。

## 證據與已知事實

### 已查證

1. 使用者提供的 2026-07-14 BO 截圖顯示既有 `9.21 Iovation Report` 的查詢區、列表、`Export CSV` 與 `Report Management` 選單樣式。本需求的新報表需沿用該視覺語言。
2. 使用者要求報表同時涵蓋 Meta Conversion API、Adjust API 與 CleverTap API，並以「確認對應事件確實由 BO 後端送出」為主要用途。
3. 查詢結果預設只顯示成功發送的紀錄；失敗紀錄不預設顯示，但必須可以查詢。
4. Meta 已提供兩組 mapping：

   | User Behavior | Meta Default Event | 已提供的成功資料 |
   |---|---|---|
   | `Deposit` | `AddToCart` | `Status`、`Username`、`Amount`、`Currency`、`Date` |
   | `First Deposit` | `Purchase` | `Status`、`Username`、`Amount`、`Currency`、`Date` |

   兩者備註皆為成功入款後回傳資料；範例值為 `Status: Success`、`Username: test123`、`Amount: 100`、`Currency: IDR`、`Date: DD/MM/YYYY HH/MM/SS`。
5. Adjust 只處理 `Deposit` 事件；報表以 RD 實際寫入的 log 欄位呈現，不另要求本需求定義 Adjust event token 或 provider payload。
6. CleverTap 由使用者提供 12 個事件與各端責任；後端觸發事件及欄位如下：

   | Event Name | Backend properties |
   |---|---|
   | `Sign Up` | `Username (String)`、`Phone Number (String)`、`Affiliate (String)`、`Whatsapp (String)`、`Telegram (String)`、`Platform (String)` |
   | `Deposit Initiated` | `Method (String)`、`Channel (String)`、`Amount (Integer)`、`Currency (String)` |
   | `Deposit Completed` | `Amount (Integer)`、`Currency (String)`、`Transaction ID (String)`、`Method (String)`、`Channel (String)`、`Time Taken (Float)`、`First Deposit (Boolean)` |
   | `Withdrawal Initiated` | `Method (String)`、`Channel (String)`、`Amount (Integer)`、`Currency (String)` |
   | `Withdrawal Completed` | `Amount (Integer)`、`Currency (String)`、`Transaction ID (String)`、`Method (String)`、`Channel (String)`、`Time Taken (Float)`、`First Withdrawal (Boolean)` |
   | `VIP Upgrade` | `Previous Level (String)`、`New Level (String)` |
   | `Bet Placed` | `Provider (String)`、`Game Type (String)`、`Game Name (String)`、`Amount (Integer)` |

   CleverTap 的 `Login`、`Visit Deposit Page`、`Visit Withdraw Page`、`Visit VIP page`、`Visit Promo page` 由 Web／iOS／Android 觸發，不由後端觸發。
7. Workspace 現有 Agent BO mockup 已存在 `Report Management` 選單群組，但尚無子項目（`src/components/layout/AppSidebar.vue:112`）。
8. Workspace 現有 Agent BO System Config 已有 Adjust、Meta 與 CleverTap 連線設定欄位（`src/views/SystemConfig.vue:41`、`src/views/SystemConfig.vue:43`、`src/views/SystemConfig.vue:46`）。其中 Adjust 設定標示為 `OPEN USE ADJUST TRACK REGIST AND FIRST DEPOSIT`（`src/views/SystemConfig.vue:462`），與本次只列 `Deposit` 的描述不完全一致。
9. Workspace 只有 Vue mockup 前端，未找到事件送出、事件 log、DB schema 或報表查詢 API 的實作證據；正式技術契約不能從本 workspace 反推。

10. 使用者於 2026-07-14 確認：

    - Agent BO 頁名與入口採 `Report Management` → `Marketing Event Delivery Report`。
    - Adjust 只納入 `Deposit`。
    - 報表狀態以 RD 實際 event delivery log 為準；每次 retry 保留獨立紀錄，不覆蓋舊紀錄。
    - 列表保留 `Amount` 與 `Transaction ID`，移除 `Currency`。
    - 日期可自選；每頁預設 10 筆，其他可選筆數沿用 BO 現行規則。
    - Admin 可查所有 Agent，列表新增 `Prefix`；Agent 只能查自己的紀錄。
    - 日期顯示格式採 `DD/MM/YYYY HH:mm:ss`。

### 推論

1. Agent BO 新報表位於 `Report Management`；Admin BO 位於 `System Report`。
2. 報表應查我方於每次第三方呼叫後寫入的不可變事件發送紀錄，而不是開頁時即時呼叫三家 provider 查詢。依據是需求要證明「我方有送出」且各 provider 未確認有一致的歷史查詢 API。
3. 頁面應為唯讀；事件重送會改變外部狀態，不應在未明確授權下併入報表需求。
4. CleverTap 報表範圍應只收 Backend 欄有定義的 7 個事件。前端觸發的 5 個事件無法證明「由 BO 後端送出」，若納入會改變資料蒐集範圍。

## 十項 intake 狀態

| 項目 | 狀態 | 目前結論／缺口 |
|---|---|---|
| 1. 狀態機 | 已查證 | 報表狀態讀取 RD event delivery log；每次 delivery attempt 固定保留自己的結果，retry 新增 attempt，不覆寫舊紀錄。成功／失敗的底層判定以 RD sender 寫入 log 的結果為準。 |
| 2. 多重性 | 推論 | 一個業務事件可對多個 provider 各產生一筆紀錄；同 provider 每次重試各一筆，使用 `event_id` 與 `attempt_no` 串聯。 |
| 3. API | 推論 | Workspace 無後端契約。新增唯讀 list/detail API，查詢 RD event delivery log；本需求不重定義三家 sender 的 provider payload。 |
| 4. 唯一性與重送 | 已查證 | 報表不提供重送；log 以 delivery attempt ID 唯一，retry 新增紀錄且不覆蓋。 |
| 5. 欄位 | 已查證 | 列表保留 `Amount`、`Transaction ID`，不顯示 `Currency`，並新增 `Prefix`。Details 顯示範圍依下方建議契約。 |
| 6. 權限 | 已查證 | Admin 可查所有 Agent；Agent 只能查自己的資料。暫不拆更細權限層級。Admin BO 入口放在 `System Report`。 |
| 7. 外部依賴 | 已查證 | Adjust 只處理 `Deposit`；報表以 RD log 為資料權威，不在本需求重定義 provider token/payload。 |
| 8. 邊界驗證 | 推論 | 前端限制篩選格式；後端強制 agent ownership、權限、日期範圍與 enum，不能只靠 UI。 |
| 9. 範圍外 | 已查證 | 排除事件重送、provider 設定、事件 mapping 維護、前端 SDK event、既有發送規則改造與細部角色權限拆分。 |
| 10. 設定落點 | 推論 | Provider／event／status 值直接來自 RD log；事件 token 與憑證沿用既有設定，不在報表頁維護。 |

## 建議契約（待使用者一次確認）

1. 頁名使用 `Marketing Event Delivery Report`；Agent BO 放在 `Report Management`，Admin BO 放在 `System Report`。
2. 篩選條件：`Username`、`Provider`（All／Meta／Adjust／CleverTap）、`Event`、`Status`（預設 Success，可選 Failed／All）、`Start Time – End Time`；預設日期為 BO 當日 00:00:00–23:59:59。
3. 列表欄位：`Sent Date`、`Prefix`、`Username`、`Provider`、`User Behavior`、`Provider Event`、`Status`、`Amount`、`Transaction ID`、`Details`。未適用欄位顯示 `—`；不顯示 `Currency`。
4. `Details` drawer 顯示 `event_id`、`attempt_no`、request properties（敏感值遮罩）、provider HTTP status、provider response／error code、error message、request time、response time；不顯示 access token、phone、WhatsApp、Telegram 或其他明文 PII。
5. 報表不自行重算 `SUCCESS`／`FAILED`；直接顯示 RD event delivery log 的狀態。RD sender 寫 log 時應以 provider 實際 accepted/success 為成功，HTTP 2xx 但 provider response 表示錯誤仍記為 `FAILED`；timeout、network error、HTTP non-2xx、provider rejection 均記為 `FAILED`。
6. CleverTap 只納入 7 個 Backend 事件。Meta 納入 `Deposit → AddToCart` 與 `First Deposit → Purchase`。Adjust 只納入 `Deposit`，欄位以 RD log 實際內容為準。
7. 每次發送嘗試新增 append-only log；重試不覆寫前次結果。報表只讀，不提供 retry。
8. 預設分頁每頁 10 筆，其他可選筆數沿用 BO 現行規則；依 `sent_at` 新到舊排序；日期區間可自選，單次查詢上限 31 天；資料保留 180 天。
9. 提供 `Export CSV`，輸出目前篩選結果，但不輸出 request／response body 與 PII。
10. 前後端共同執行 enum、日期格式與最大區間驗證。後端以登入身分強制資料範圍：Admin 不套 Agent 限制，Agent 強制限定自己的 `agent_id`，不得信任前端傳入的 Prefix。日期顯示格式為 `DD/MM/YYYY HH:mm:ss`。

## 阻擋決策

無。使用者於 2026-07-14 確認 Admin BO 入口放在 `System Report`。
