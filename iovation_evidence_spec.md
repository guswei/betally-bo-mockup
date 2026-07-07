# Iovation Evidence — RD 簡易 Spec

> 平台：BetAlly **Agent BO**　|　落點：既有 **Iovation Report** 頁　|　Mockup：`iovation_evidence_mockup.html`
> 本文件只列**已拍板、可動工**內容（§22）。待議項在對話中與 PM 確認，不進本文件。

## 1. 背景
既有 Iovation Report 是 iovation device check（`CheckTransactionDetails`）的唯讀紀錄，每列一次查核（含 account code / device / 風險建議 Result）。目前操作者只能看、不能回報結果。本需求讓操作者**事後把真實結果（evidence）回灌 iovation**，餵養裝置信譽網路。device check 已上線，本期**只新增 Evidence 回報路徑**。

## 2. 核心功能變更
1. Iovation Report 表格**新增兩欄**：
   - `Evidence`：顯示該列已報 evidence 的狀態（讀我方自存紀錄，非即時查 iovation）。未報 `—`；已報顯示代碼 badge；已撤回顯示 `Retracted / Ignored`（刪除線）。
   - `Actions`：每列 `Add Evidence` 按鈕；**已報過的列另出現 `Retract / Ignore` 按鈕**。
2. `Add Evidence` 開 modal（欄位帶自該列、**可手動編輯**）：`Applied to type` + `Device ID` + `Player Username` + `User type` + `Evidence Type` + `Comment` + 二次確認 → 送出。
3. `Retract / Ignore` 開 modal：對該列既有 evidence 選 `Retract`（撤回）或 `Ignore`（請 iovation 忽略）+ 填理由 + 二次確認 → 送出。
4. 新增後端 endpoint：前端 POST → 後端呼叫 iovation Evidence API → 落地審計紀錄。
5. 既有過濾區、既有欄位、Export CSV、device check 流程**一律不動**。

## 3. 規則與邊界
- **互動模型**：每列 `Add Evidence` **帶入該列脈絡但欄位可編輯**（客戶要求 risk 可手動輸入／覆寫 Device ID、Username）。以 `iovation_report_id` 記錄來源列供稽核回溯；送出時**以 modal 內實際填的值**綁定 iovation。
- **Add Evidence 欄位**：
  - `Applied to type`（下拉，必填）：`ACCOUNT`（綁 usercode）/ `DEVICE`（綁 device_id）。決定掛載對象與對應必填欄位（選 Account 時 Username 必填；選 Device 時 Device ID 必填）。
  - `Device ID` / `Player Username`：帶自該列、可編輯；純文字。
  - `User type`（下拉，必填）：`PLAYER` / `DEVICE`。**🚩 與 `Applied to type` 語意疑似重疊，見 §5 blocker①。**
  - `Evidence Type`（下拉，必填）：iovation 正式代碼清單（`1-1`~`100-19`，見 §5）。
  - `Comment`：必填、去頭尾空白、不可全空白；`VARCHAR(500)`（上限以 iovation 欄位上限為準取小者）；純文字落地。
- **Retract / Ignore**：對既有 evidence 送出 `RETRACT` 或 `IGNORE`+理由；**不刪原 SUBMIT 紀錄**，另寫一筆 `action_type` 稽核。iovation 端 Retract/Ignore 對應動作 **🚩 RD 對照整合文件確認**。
- **Evidence 欄資料來源**：iovation **無**「列出已送 evidence」的查詢 API → 已報／已撤回狀態一律讀我方 `iovation_evidence_log`。
- **送出不可逆**：Add 與 Retract/Ignore 兩個 modal 皆**強制二次確認** checkbox，未勾不可送（補償控制）。
- **防重送（Idempotency）**：送出帶 `Idempotency-Key`；同一 `(iovation_report_id + applied_to_type + evidence_type)` 重送 → 後端去重，不重複打 iovation。
- **權限（已拍板）**：**所有可進此 Iovation Report 的操作者皆可送出／撤回**。
- **送出點**：一律後端呼叫 iovation；前端**不可**直連 iovation。

## 4. 安全基線（硬需求）
- 前端絕不直連 iovation（憑證僅存後端）。
- iovation 認證/endpoint **沿用既有 device check 整合**（同一組 SubscriberId / passcode、同一 host）。
- `Message` / `Evidence` 欄顯示時須 HTML-escape，防 XSS（後台列表/紀錄頁皆是）。
- 審計欄（operator / 時間 / iovation 回傳）**寫入後不可竄改**。

## 5. 後端相依與資料模型
新增表 `iovation_evidence_log`（審計鏈，無金額欄位故無 `DECIMAL` 精度需求；惟**比照金流等級**做審計與權限控管）：

| 欄位 | 型別 | 說明 |
|---|---|---|
| `id` | PK | |
| `iovation_report_id` | FK | 關聯來源列 check（稽核回溯） |
| `action_type` | enum | `SUBMIT` / `RETRACT` / `IGNORE`；撤回另寫一筆，不覆蓋原 SUBMIT |
| `applied_to_type` | enum | `ACCOUNT` / `DEVICE`（evidence 掛載對象） |
| `user_type` | enum | `PLAYER` / `DEVICE`（🚩 與 applied_to_type 疑重疊，見 blocker①） |
| `usercode` | string | = iovation account code（可由操作者編輯後送出） |
| `device_id` | string | = iovation device alias（可由操作者編輯後送出） |
| `source_txn` | string, nullable | 原始 check 的 iovation txn |
| `evidence_type` | string(iovation code) | iovation 代碼，如 `1-1`/`3-4`/`100-1`（見下方相依） |
| `comment` | VARCHAR(500) | 操作者訊息／撤回理由 |
| `idempotency_key` | string, UNIQUE | 防重送 |
| `iovation_status` | string | iovation 回傳狀態 |
| `iovation_evidence_id` | string, nullable | iovation 回傳識別 |
| `operator_id` | FK | 送出者 |
| `operator_role` | string | 送出者角色（稽核用） |
| `created_at` | timestamp | |

**定工時前 RD 須先確認（blocker）：**
- **① `Applied to type`（Account/Device）與 `User type`（Player/Device）語意疑似重疊**：客戶回饋把兩者列為獨立欄位，但 `Device` 值同時出現在兩下拉。RD 對照 device check 整合文件確認是**兩個不同 iovation 參數**（掛載對象 vs 帳號屬性）還是同一參數重複，再決定是否合併。mockup 先忠實照客戶兩欄呈現。
- **② `evidence_type` 代碼清單**：已依客戶截圖建入 iovation 正式代碼（`1-1`~`100-19`）。iovation **無動態 list API** → 清單靜態維護；`100-x「Subscriber Specific: Good Account」`為**各訂閱帳號在 portal 自訂**，RD 以我方帳號 portal 設定核對實際開了哪些、對應標籤。
- **③ iovation Evidence API request schema**（REST/SOAP、欄位名、`comment` 長度上限、`Retract`/`Ignore` 對應動作）：以既有 device check 同一份整合文件為準。

## 6. 範圍外（本期不做）
- 不改既有過濾、欄位、Export CSV、device check。
- 不做帳號層級彙總視圖（本期以 report 每列為操作單位）。
- 權限不做細分（不另設獨立權限位；沿用「可進此報表即可送」）。
- 已報 evidence 的**編輯**（改內容）不做；僅提供 `Add more`（再送一筆）與 `Retract / Ignore`（撤回）。
