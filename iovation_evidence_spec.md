# Iovation Evidence — RD 簡易 Spec

> 平台：BetAlly **Agent BO**　|　落點：既有 **Iovation Report** 頁　|　Mockup：`iovation_evidence_mockup.html`
> 本文件只列**已拍板、可動工**內容（§22）。待議項在對話中與 PM 確認，不進本文件。

## 1. 背景
既有 Iovation Report 是 iovation device check（`CheckTransactionDetails`）的唯讀紀錄，每列一次查核（含 account code / device / 風險建議 Result）。目前操作者只能看、不能回報結果。本需求讓操作者**事後把真實結果（evidence）回灌 iovation**，餵養裝置信譽網路。device check 已上線，本期**只新增 Evidence 回報路徑**。

## 2. 核心功能變更
1. Iovation Report 表格**新增兩欄**：
   - `Evidence`：顯示該列是否已報過 evidence（讀我方自存紀錄，非即時查 iovation）。
   - `Actions`：每列 `Add Evidence` 按鈕。
2. `Add Evidence` 開 modal：帶出該列脈絡（唯讀）+ 選 `Evidence Type` + 填 `Message` + 二次確認 → 送出。
3. 新增後端 endpoint：前端 POST → 後端呼叫 iovation Evidence API → 落地審計紀錄。
4. 既有過濾區、既有欄位、Export CSV、device check 流程**一律不動**。

## 3. 規則與邊界
- **綁定層級**：evidence 綁「該列的 account code（usercode）+ device alias（Device ID）+ 原始 check txn」。精準掛到該次查核的該台裝置。
- **Evidence 欄資料來源**：iovation **無**「列出已送 evidence」的查詢 API → 已報狀態一律讀我方 `iovation_evidence_log`。未報顯示 `—`。
- **Message**：必填、去頭尾空白、不可全空白；`VARCHAR(500)`（上限以 iovation 欄位上限為準，取小者）；純文字落地。
- **送出不可逆**：modal 內**強制二次確認** checkbox，未勾不可送（補償控制）。
- **防重送（Idempotency）**：送出帶 `Idempotency-Key`；同一 `(iovation_report_id + evidence_type)` 重送 → 後端去重，不重複打 iovation。（Iovation Report 本就有同裝置多列重複紀錄，須避免同次查核被連點重報。）
- **權限（已拍板）**：**所有可進此 Iovation Report 的操作者皆可送出**。
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
| `iovation_report_id` | FK | 關聯被報的那列 check |
| `usercode` | string | = iovation account code |
| `device_id` | string | = iovation device alias |
| `source_txn` | string, nullable | 原始 check 的 iovation txn |
| `evidence_type` | enum | 合法值見下方相依 |
| `comment` | VARCHAR(500) | 操作者訊息 |
| `idempotency_key` | string, UNIQUE | 防重送 |
| `iovation_status` | string | iovation 回傳狀態 |
| `iovation_evidence_id` | string, nullable | iovation 回傳識別 |
| `operator_id` | FK | 送出者 |
| `operator_role` | string | 送出者角色（稽核用） |
| `created_at` | timestamp | |

**定工時前 RD 須先確認（blocker）：**
- `evidence_type` 合法 enum：iovation 按帳號在 portal 設定、**無動態 list API**。請翻既有 device check 整合文件確認實際開了哪些值（mockup 暫用 FRAUD / CHARGEBACK / ACCOUNT_TAKEOVER / BONUS_ABUSE / GOOD_CUSTOMER 當佔位，**此項暫不寫死**）。
- iovation Evidence API 的 request schema（REST/SOAP、欄位名、`comment` 長度上限）：以既有 device check 那支整合的同一份文件為準。

## 6. 範圍外（本期不做）
- 不改既有過濾、欄位、Export CSV、device check。
- 不做 evidence 的修改/撤回（iovation 端語意另議）。
- 不做帳號層級彙總視圖（本期以 report 每列為操作單位）。
- 權限不做細分（不另設獨立權限位；沿用「可進此報表即可送」）。
