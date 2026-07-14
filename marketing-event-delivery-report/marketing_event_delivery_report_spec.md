# Marketing Event Delivery Report — RD Spec

## 1. 背景與目標

Agent BO 與 Admin BO 新增同名唯讀報表，查詢既有 RD event delivery log。報表合併 Meta Conversion API、Adjust API 與 CleverTap API 的 delivery attempts，預設只顯示 `SUCCESS`。本需求不修改 sender payload、provider 設定或事件觸發條件。

對應 PRD：[GitHub](https://github.com/guswei/betally-bo-mockup/blob/main/marketing-event-delivery-report/marketing_event_delivery_report_PRD.md)。

## 2. Scope

### 2.1 IN scope

- Agent BO route：`/marketing-event-delivery-report`。
- Admin BO route：`/admin/marketing-event-delivery-report`。
- List、detail、CSV 三個唯讀 API。
- Admin 跨 Agent 查詢與 Prefix 篩選。
- Agent ownership 強制限制。
- `SUCCESS`／`FAILED` 查詢、遮罩後 details、日期上限、分頁與 180 天保留期。
- Meta 2 個 mapping、Adjust `Deposit`、CleverTap 7 個 Backend events。

### 2.2 OUT scope

- Provider retry、補送、刪除或修改 log。
- Meta、Adjust、CleverTap 憑證與事件 mapping 維護。
- Sender payload、第三方成功判定或事件觸發條件改造。
- CleverTap Web／iOS／Android 觸發的 `Login` 與 Visit Page events。
- Currency 顯示或匯出。
- 四層級權限拆分與新權限位。

## 3. 角色、入口與 ownership

| 身分 | 選單 | Route | Prefix filter | 後端資料範圍 |
|---|---|---|---|---|
| Agent | `Report Management` | `/marketing-event-delivery-report` | 不顯示 | JWT/session 的 `agent_id`。 |
| Admin | `System Report` | `/admin/marketing-event-delivery-report` | 顯示 | 所有 Agent；指定 Prefix 時轉成對應 `agent_id`。 |

後端不得使用 Agent request 傳入的 Prefix 決定 scope。Agent 呼叫 list 或 export 時帶 `prefix`，回傳 403 與 `REPORT_SCOPE_FORBIDDEN`。Agent 讀取其他 Agent 的 `delivery_id` 時同樣回傳 403。前端隱藏 Prefix 不是授權控制。

本期不新增 View／Export 權限位。Admin 與 Agent 能否進入 BO 沿用現有身分驗證，API 再執行上表的 ownership。

## 4. 狀態模型與事件範圍

### 4.1 Delivery attempt 狀態

| Status | 報表行為 |
|---|---|
| `SUCCESS` | RD event delivery log 已記錄本次 attempt 成功；預設查詢會顯示。 |
| `FAILED` | RD event delivery log 已記錄本次 attempt 失敗；使用者選 `Failed` 或 `All` 才顯示。 |

報表不重算狀態，也不把 `FAILED` 改成 `SUCCESS`。Sender retry 會新增 attempt；每筆以 `delivery_id` 唯一識別，同一業務事件的 `attempt_no` 依序遞增。

### 4.2 Event catalog

| Provider | `user_behavior` | `provider_event` |
|---|---|---|
| `META` | `Deposit` | `AddToCart` |
| `META` | `First Deposit` | `Purchase` |
| `ADJUST` | `Deposit` | 讀 RD log；無獨立名稱時輸出 `Deposit` |
| `CLEVERTAP` | `Sign Up` | `Sign Up` |
| `CLEVERTAP` | `Deposit Initiated` | `Deposit Initiated` |
| `CLEVERTAP` | `Deposit Completed` | `Deposit Completed` |
| `CLEVERTAP` | `Withdrawal Initiated` | `Withdrawal Initiated` |
| `CLEVERTAP` | `Withdrawal Completed` | `Withdrawal Completed` |
| `CLEVERTAP` | `VIP Upgrade` | `VIP Upgrade` |
| `CLEVERTAP` | `Bet Placed` | `Bet Placed` |

Provider enum 固定為 `META`、`ADJUST`、`CLEVERTAP`。Status enum 固定為 `SUCCESS`、`FAILED`。不在 event catalog 的紀錄不由本報表 API 回傳。

## 5. UI 契約

### 5.1 Filters

| 欄位 | 型別 | 預設 | 驗證 |
|---|---|---|---|
| `Prefix` | Select，Admin only | `All` | 值必須存在於 Admin 可見 Agent 清單。 |
| `Username` | Text | 空白 | Trim；最大 100 字元；包含查詢。 |
| `Provider` | Select | `All` | All 或 provider enum。 |
| `Event` | Select | `All` | 選項依 Provider 取自 event catalog。 |
| `Status` | Select | `Success` | `Success`、`Failed`、`All`。 |
| `Start Time – End Time` | Datetime range | BO 當日 00:00:00–23:59:59 | 必填；start ≤ end；區間 ≤ 31 天。 |

日期輸入依 BO timezone 解讀，送 API 前轉 ISO 8601。畫面顯示格式固定為 `DD/MM/YYYY HH:mm:ss`。

### 5.2 List columns

依序顯示：`Sent Date`、`Prefix`、`Username`、`Provider`、`User Behavior`、`Provider Event`、`Status`、`Amount`、`Transaction ID`、`Details`。不顯示 `Currency`。Nullable 欄位顯示 `—`。

`Amount` 在 DB/read model 使用 `DECIMAL(18,2)`；API 使用 decimal string，前端不得轉成 float。資料依 `sent_at DESC, delivery_id DESC` 排序，預設每頁 10 筆。其他 page size 選項沿用 BO 既有分頁元件，API 接受 1–100。

### 5.3 UI states

| State | 顯示與互動 |
|---|---|
| Default | 套用 `Status = SUCCESS` 與 BO 當日。 |
| Loading | 列表顯示 `Loading...`；Search disabled，避免同一查詢重複送出。 |
| Empty | 保留表頭並顯示 `No Data`。 |
| Error | 清除本次列表並顯示 `Failed to load event delivery records. Please try again.`。 |
| Invalid date | 在日期欄下顯示錯誤，不呼叫 API。 |
| Details loading | Modal 先顯示 loading；成功後替換為內容。 |
| Details forbidden | 關閉 loading，顯示無權存取訊息，不顯示 response body。 |

### 5.4 Details masking

Details 顯示 `delivery_id`、`attempt_no`、Prefix、Username、Provider、Status、HTTP status、request／response time、error code、error message、request properties 與 provider response。Currency 不回傳到前端。

Report adapter 對 request properties 做 case-insensitive recursive key filter。Key 命中 `token`、`authorization`、`secret`、`password`、`phone`、`whatsapp`、`telegram` 時，value 固定替換成 `***`。Provider response 只保留 `accepted`、`status`、`event_id`、`code`、`message`；其他 raw response 欄位不回傳。Error message 必須移除 request body、query token 與 header value。

## 6. API 契約

所有時間參數與 response timestamp 使用 ISO 8601 UTC。HTTP response header 包含 `Cache-Control: no-store`。

### 6.1 List

`GET /api/v1/reports/marketing-event-deliveries`

#### Query parameters

| Parameter | Required | 型別／規則 |
|---|---|---|
| `prefix` | No | Admin only；省略代表所有 Agent。 |
| `username` | No | String，trim 後 1–100 字元。 |
| `provider` | No | `META`、`ADJUST`、`CLEVERTAP`；省略代表 All。 |
| `event` | No | Event catalog 的 `user_behavior`；省略代表 All。 |
| `status` | No | `SUCCESS`、`FAILED`；省略時預設 `SUCCESS`。UI 的 All 以省略參數表示。 |
| `start_at` | Yes | ISO 8601。 |
| `end_at` | Yes | ISO 8601；不得早於 `start_at`，區間不得超過 31 天。 |
| `page` | No | Integer ≥ 1；預設 1。 |
| `page_size` | No | Integer 1–100；預設 10。 |
| `snapshot_at` | No | 第一頁 response 回傳的 UTC timestamp；後續分頁沿用。 |

首次查詢未帶 `snapshot_at` 時，後端在開始查詢時產生 `snapshot_at`。List 條件固定加上 `sent_at <= min(end_at, snapshot_at)`，後續分頁與 CSV 沿用同值，避免查詢期間的新紀錄讓頁面重複或跳號。

#### 200 response

```json
{
  "items": [
    {
      "delivery_id": "evt_20260714_000108",
      "attempt_no": 1,
      "prefix": "AG001",
      "username": "test123",
      "provider": "META",
      "user_behavior": "Deposit",
      "provider_event": "AddToCart",
      "status": "SUCCESS",
      "amount": "100.00",
      "transaction_id": "DEP260714000108",
      "sent_at": "2026-07-14T08:42:18Z"
    }
  ],
  "pagination": {
    "page": 1,
    "page_size": 10,
    "total": 1
  },
  "snapshot_at": "2026-07-14T08:45:00Z"
}
```

`username`、`provider_event`、`amount`、`transaction_id` 可為 null。Response 不含 `currency`。

### 6.2 Detail

`GET /api/v1/reports/marketing-event-deliveries/{delivery_id}`

#### 200 response

```json
{
  "delivery_id": "evt_20260714_000108",
  "attempt_no": 1,
  "prefix": "AG001",
  "username": "test123",
  "provider": "META",
  "user_behavior": "Deposit",
  "provider_event": "AddToCart",
  "status": "SUCCESS",
  "amount": "100.00",
  "transaction_id": "DEP260714000108",
  "sent_at": "2026-07-14T08:42:18Z",
  "request_at": "2026-07-14T08:42:17Z",
  "response_at": "2026-07-14T08:42:18Z",
  "http_status": 200,
  "error_code": null,
  "error_message": null,
  "request_properties": {
    "Username": "test123",
    "Amount": "100.00",
    "access_token": "***"
  },
  "provider_response": {
    "accepted": true,
    "event_id": "fb_901882"
  }
}
```

Response 不含 `currency`。`request_at`、`response_at`、`http_status`、`error_code`、`error_message`、`request_properties`、`provider_response` 可為 null。

### 6.3 Export

`GET /api/v1/reports/marketing-event-deliveries/export`

Query parameters 與 list 相同，但不使用 `page`、`page_size`。前端傳入目前 list 的 `snapshot_at`。Response：

- 200：`Content-Type: text/csv; charset=utf-8`，`Content-Disposition: attachment; filename="marketing-event-delivery-report.csv"`。
- CSV 欄位依序為 `Sent Date`、`Prefix`、`Username`、`Provider`、`User Behavior`、`Provider Event`、`Status`、`Amount`、`Transaction ID`。
- 不含 Currency、PII、request body、provider response body。
- 日期格式為 `DD/MM/YYYY HH:mm:ss`，timezone 與畫面一致。

### 6.4 Error model

```json
{
  "error": {
    "code": "REPORT_DATE_RANGE_INVALID",
    "message": "The date range must not exceed 31 days."
  }
}
```

| HTTP | Code | 條件 |
|---|---|---|
| 400 | `REPORT_FILTER_INVALID` | Provider、event、status、page 或 page_size 不合法。 |
| 400 | `REPORT_DATE_RANGE_INVALID` | 日期缺漏、start > end 或區間超過 31 天。 |
| 401 | `AUTH_REQUIRED` | 未登入或 session 失效。 |
| 403 | `REPORT_SCOPE_FORBIDDEN` | Agent 帶 Prefix 或讀取其他 Agent 資料。 |
| 404 | `REPORT_DELIVERY_NOT_FOUND` | `delivery_id` 不存在或紀錄已超過 180 天。 |
| 504 | `REPORT_QUERY_TIMEOUT` | List／detail 查詢超過 10 秒，或 export 超過 60 秒。 |
| 500 | `REPORT_INTERNAL_ERROR` | 未分類後端錯誤。 |

前端可對 list/detail 的 network error 或 5xx 自動重試 1 次，延遲 1 秒。400、401、403、404 不自動重試。Export 不自動重試。

## 7. Read model、保留與 rollback

### 7.1 Read model

本需求不新增 table、不修改 sender DB schema，也不建立平行事件來源。Report repository 從 RD event delivery log projection 下列欄位：

| 欄位 | 型別 | Nullable | 規則 |
|---|---|---|---|
| `delivery_id` | String/UUID | No | 唯一 delivery attempt。 |
| `attempt_no` | Integer | No | 同一業務事件從 1 遞增。 |
| `agent_id` | String/UUID | No | ownership 查詢鍵。 |
| `prefix` | String | No | 由現行 Agent mapping 取得。 |
| `username` | String | Yes | 玩家帳號。 |
| `provider` | Enum | No | `META`、`ADJUST`、`CLEVERTAP`。 |
| `user_behavior` | String | No | Event catalog 值。 |
| `provider_event` | String | Yes | Provider event name。 |
| `status` | Enum | No | `SUCCESS`、`FAILED`。 |
| `amount` | `DECIMAL(18,2)` | Yes | 禁止 float。 |
| `transaction_id` | String | Yes | 業務交易編號。 |
| `sent_at` | UTC timestamp | No | 排序與保留期依據。 |
| `request_at` | UTC timestamp | Yes | Detail only。 |
| `response_at` | UTC timestamp | Yes | Detail only。 |
| `http_status` | Integer | Yes | Detail only。 |
| `error_code` | String | Yes | Detail only。 |
| `error_message` | String | Yes | 已移除敏感內容。 |
| `request_properties` | JSON/Object | Yes | 回傳前套用遮罩與 Currency 排除。 |
| `provider_response` | JSON/Object | Yes | 只輸出 allowlist keys。 |

Report query 必須使用 RD log 現有的等效索引能力支援 `(agent_id, sent_at)`、`(status, sent_at)`、`(provider, user_behavior, sent_at)` 查詢。此項屬既有 log 的查詢效能契約，不改變事件唯一性。

### 7.2 Retention

資料保留 180 天。List 與 export 固定加上 `sent_at >= now - 180 days`；detail 對超過 180 天的紀錄回傳 404。RD log retention job 每日刪除超過 180 天的紀錄；刪除以 batch 執行，不鎖住 sender 寫入路徑。

### 7.3 Migration 與 rollback

- Schema migration：N/A，本需求不新增或修改 table/column/constraint。
- 發布順序：先發布 report API，再發布兩個 BO route 與選單。
- Rollback：先移除 BO 選單與 route，再停用 report API。Rollback 不刪 RD log，也不改 sender。

## 8. 權限、安全與 audit

### 8.1 Enforcement matrix

| 操作 | Agent | Admin | 後端檢查 |
|---|---|---|---|
| List | 自己的 `agent_id` | 所有 Agent／指定 Prefix | Session role + ownership。 |
| Detail | 自己的 `agent_id` | 所有 Agent | 先找紀錄，再比對 ownership。 |
| Export | 自己的 `agent_id` | 所有 Agent／指定 Prefix | 與 list 完全相同。 |
| Retry／modify／delete | 禁止 | 禁止 | 無對應 endpoint。 |

### 8.2 Audit and security events

- CSV 成功匯出寫入 `MARKETING_EVENT_REPORT_EXPORTED`，包含 operator ID、role、filters、snapshot_at、row count、requested_at；不寫 request／response body。
- Scope 違規寫入 `MARKETING_EVENT_REPORT_ACCESS_DENIED`，包含 operator ID、role、requested prefix 或 delivery ID、requested_at。
- List/detail 走既有 BO access log，記錄 endpoint、operator、status code、latency，不記錄 PII 或 payload。

## 9. Idempotency、重試、timeout 與 concurrency

- 三個 API 均為 GET，不接受 `Idempotency-Key`，不產生 provider side effect。
- Sender 的 retry 與冪等策略不在本需求內；報表只保留並顯示每次 attempt。
- List/detail query timeout 為 10 秒；export timeout 為 60 秒。
- `snapshot_at` 固定分頁與 export 的資料上界。排序 key 為 `sent_at DESC, delivery_id DESC`；相同時間仍可穩定排序。
- Report query 與 sender insert 並行，不鎖 sender table；查詢採資料庫預設 read-consistent isolation，不執行 `SELECT FOR UPDATE`。

## 10. NFR 與觀測性

- `NFR-SEC`：ownership 由後端 session 套用；敏感 key 在 serialization 前遮罩；HTTP response 使用 `Cache-Control: no-store`。
- `NFR-REL`：報表查詢不得呼叫 provider API。錯誤 response 不包含 stack trace、SQL、token 或 raw payload。
- `NFR-DATA`：Amount 使用 `DECIMAL(18,2)` 與 decimal string。UTC 儲存，BO timezone 顯示。保留 180 天。
- List API 在 page size 10、31 天內的查詢 p95 ≤ 2 秒；detail p95 ≤ 1 秒。Export 以 streaming response 輸出，記憶體不得載入完整結果集。
- Metrics：`marketing_event_report_requests_total{endpoint,status}`、`marketing_event_report_latency_seconds{endpoint}`、`marketing_event_report_export_rows_total`、`marketing_event_report_scope_denied_total`。
- Log：request ID、operator ID、role、endpoint、filters hash、status code、latency；不得寫 Username、PII、request properties 或 provider response。
- Alert：5 分鐘內 list/detail 5xx 比例 > 5% 或 p95 > 2 秒持續 10 分鐘時告警。

## 11. FR／AC 實作追蹤

| FR | 實作區域 | AC |
|---|---|---|
| `FR-1` | List repository、排序、attempt projection、retention | `AC-1`、`AC-10`、`AC-14`、`AC-15` |
| `FR-2` | Filter form、validation、pagination、snapshot | `AC-1`、`AC-2`、`AC-11`、`AC-12` |
| `FR-3` | Session scope、Prefix mapping、403 | `AC-6`、`AC-7` |
| `FR-4` | List columns、detail API、masking、UI states | `AC-8`、`AC-9`、`AC-14` |
| `FR-5` | Event catalog filter | `AC-3`、`AC-4`、`AC-5` |
| `FR-6` | CSV endpoint、export audit | `AC-13`、`AC-15` |

## 12. 測試與驗收

- Unit：date range、event catalog、status default、amount serialization、recursive masking、provider response allowlist。
- Repository：Admin all-prefix、Admin single-prefix、Agent ownership、stable sort、snapshot、180-day boundary。
- API：200、400、401、403、404、500、504 與 error code mapping。
- UI：Default、Failed、All、loading、empty、error、invalid date、details、CSV、Agent／Admin routes。
- Security：Agent 注入 Prefix、猜測其他 `delivery_id`、CSV 跨 Agent、error message payload leakage。
- Regression：既有 Report Management、System Report、System Config 的 Adjust／Meta／CleverTap 設定與其他報表不變。
