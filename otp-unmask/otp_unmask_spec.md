# OTP Verify Code 解遮罩 — RD 開發 Spec

- 對象系統：GCP Agent BO
- 位置：`OTP Report`
- Mockup：https://guswei.github.io/betally-bo-mockup/otp-unmask/mockup.html （右上可切換「顯示 RD 註記」與權限開關）
- 流程圖：https://guswei.github.io/betally-bo-mockup/otp-unmask/diagrams/otp_unmask_flow.png （第 9 節有內嵌圖與 Mermaid 原始碼）

---

## 1. 背景與範圍

`OTP Report` 的 `Verify Code` 欄目前一律顯示 `XXXXXX`。客服處理忘記密碼的申訴時看不到實際發出的驗證碼，無法判斷碼是否正確送出。客戶要求開放檢視，並希望與權限設定連動。

本次只改 `Verify Code` 一欄的顯示行為。`Mobile or Email` 欄現況即為明文，不在本次範圍。

本次做四件事：

1. `Verify Code` 欄新增逐列解遮罩。
2. 解遮罩的資格限制在忘記密碼流程且已失效的驗證碼。
3. 新增一個獨立的權限節點控制此操作。
4. 每次解遮罩寫入審計紀錄。

---

## 2. 解遮罩資格

三個條件全部成立才可解遮罩，缺一不可：

| # | 條件 | 不成立時 |
|---|---|---|
| 1 | 操作者被授予 OTP 明碼檢視權限 | 前端不顯示眼睛圖示；後端回 `403` |
| 2 | 該列 `Verify Proc` 為 `FORGET_PASSWORD` | 前端不顯示眼睛圖示；後端回 `409` |
| 3 | 該列的驗證碼已失效 | 前端圖示 `disabled` 並說明原因；後端回 `409` |

### 2.1 「已失效」的判定

符合下列任一項即為已失效：

| 判定 | 條件 |
|---|---|
| 已過期 | 目前時間晚於該列 `Expiry` |
| 已使用 | 該列 `verify_time` 有值 |

`verify_time` 有值代表玩家已完成驗證，該組驗證碼不可能再被使用。以此欄判定，與 `verify_status` 的 enum 值域無關，不需要等待該欄位值域確認。

判定是否過期時，`Expiry` 與當下時間必須先換算到同一個時區基準再比較，不得直接比較字串，也不得把兩者當成同一時區而省略換算。基準錯一個時區，仍在有效期內的驗證碼就會被判定為已過期而開放解遮罩。

系統狀態列顯示的時區為 `GMT+07:00`。

### 2.2 為什麼限制在已失效

忘記密碼的驗證碼在有效期內等同於改密碼的授權憑證。取得有效期內的明碼即可完成該玩家的密碼重設。限制在已失效的碼，移除這條路徑。

---

## 3. 畫面變更

### 3.1 `Verify Code` 欄

欄位位置與既有欄位順序不動。遮罩態一律顯示六個 `X`，不反映實際碼長。

儲存格內新增一個眼睛圖示，依資格呈現三種狀態：

| 情境 | 圖示 | 儲存格內容 |
|---|---|---|
| 三個條件全部成立 | 顯示且可點擊 | 點擊後切換為明碼 |
| 條件 3 不成立（仍有效） | 顯示但 `disabled` | `XXXXXX`，圖示下方顯示英文說明 |
| 條件 1 或條件 2 不成立 | 不顯示 | `XXXXXX` |

權限不足時圖示直接隱藏，不使用 `disabled`。這是本 BO 的既有慣例。因業務規則而不可操作的情況（驗證碼仍有效）採 `disabled` 並附說明文字，讓操作員知道原因與可以做什麼。

### 3.2 解遮罩後的行為

一次點擊只解開該列，其他列維持遮罩。

再次點擊同一個圖示回到遮罩態。

執行查詢、重設篩選或換頁後，已解開的欄位一律回到遮罩態。明碼不快取於前端。

---

## 4. API

| Method / Path | 用途 |
|---|---|
| `POST /otp-report/{id}/reveal` | 取得單筆的 `Verify Code` 明碼 |

採 `POST` 而非 `GET`：此操作要寫入審計紀錄，且不應被瀏覽器或反向代理快取，記錄 `id` 也不應出現在存取紀錄的 URL 中。

操作對象為單筆，以 OTP 記錄的 `id` 指定，不接受批次。

**列表 API 的回應不得包含明碼，不論操作者是否有權限。** 只做畫面遮罩的話，開發者工具即可取得明碼，遮罩等於不存在。

錯誤模型：

| 狀態碼 | 錯誤鍵 | 情境 |
|---|---|---|
| `403` | `FORBIDDEN` | 操作者未被授予 OTP 明碼檢視權限 |
| `404` | `OTP_RECORD_NOT_FOUND` | 記錄不存在 |
| `409` | `OTP_PROCESS_NOT_ELIGIBLE` | 該筆 `Verify Proc` 不是 `FORGET_PASSWORD` |
| `409` | `OTP_STILL_VALID` | 該筆驗證碼仍在有效期內且未被使用 |

四種錯誤的回應 body 都不得包含明碼。

後端一律重新驗證三個條件，不信任前端送來的任何資格判定。

---

## 5. 權限

新增一個獨立的權限節點，掛在 `Role Setting`，供後台勾選。

此權限**不綁定 4-tier 層級**。被授予的操作者即可執行，與其在 Admin / Brand Admin / Agent Admin / Sub Agent Admin 的哪一層無關。

| 操作 | 授權方式 |
|---|---|
| 解遮罩 `Verify Code` | 新增的權限節點 |
| 檢視 `OTP Report`、篩選、分頁、`Export CSV` | 維持現狀，沿用既有權限 |

既有的 `OTP Report` 檢視權限不變更。能進入此頁面不代表能解遮罩，兩者是分開的授權。

---

## 6. 審計

每一次成功解遮罩寫入一筆紀錄，內容包含：操作者、時間、OTP 記錄 `id`、該筆對應的玩家 `username`。

**審計紀錄本身不得儲存明碼。** 紀錄的用途是知道誰在什麼時候看了哪一筆，不是保存驗證碼。

同一列重複解遮罩，每次各寫一筆，不去重。反覆查閱同一筆是需要被看見的訊號，去重會把它抹掉。

再次點擊圖示回到遮罩態時不寫紀錄，該動作沒有取得新的資料。

驗權失敗或資格不符而被拒絕的請求不產生成功紀錄。

保存期限沿用既有操作紀錄的規則。

---

## 7. Export CSV

`Export CSV` 的 `Verify Code` 欄一律輸出 `XXXXXX`，不因操作者持有解遮罩權限而改變。

畫面一次呈現一筆，CSV 一次帶走整個查詢結果且脫離系統管控，兩者的外洩規模不同級。

---

## 8. 範圍外

- 不改動 `Mobile or Email` 欄的顯示，該欄現況為明文。
- 不改動 OTP 的產生、發送、驗證流程與有效期長度。
- 不開放 `FORGET_PASSWORD` 以外的 `Verify Proc`，其驗證碼維持全遮罩。
- 不改動既有篩選、分頁、`Columns Settings` 與 `Export CSV` 的其他欄位。
- 不新增批次解遮罩。
- 不新增解遮罩紀錄的查詢頁面。

---

## 9. 流程圖

解遮罩的三道資格檢查，以及通過後的後端處理順序。

![OTP Verify Code 解遮罩流程](https://guswei.github.io/betally-bo-mockup/otp-unmask/diagrams/otp_unmask_flow.png)

Mermaid 原始碼：

```
flowchart TD
  A[操作者在 OTP Report 點某列的眼睛圖示] --> B{有 OTP 明碼檢視權限?}
  B -->|否| C[前端不顯示眼睛圖示<br/>直接呼叫 API 回 403 FORBIDDEN]
  B -->|是| D{Verify Proc 是 FORGET_PASSWORD?}
  D -->|否| E[不顯示眼睛圖示<br/>後端回 409 OTP_PROCESS_NOT_ELIGIBLE]
  D -->|是| F{已過期或已使用?}
  F -->|否 仍有效| G[眼睛圖示 disabled 並說明原因<br/>後端回 409 OTP_STILL_VALID]
  F -->|是| H[POST /otp-report/id/reveal]
  H --> I[後端重驗權限與解遮罩資格]
  I --> J[寫入審計<br/>操作者/時間/OTP id/玩家<br/>審計本身不存明碼]
  J --> K[只回傳該列明碼<br/>列表 API 與 Export CSV 維持遮罩]
  K --> L[畫面只解開該列<br/>翻頁或重新查詢後回到遮罩]
  classDef step fill:#dbeafe,stroke:#2563eb,color:#1e3a8a
  classDef dec  fill:#fef3c7,stroke:#d97706,color:#78350f
  classDef stop fill:#fdecea,stroke:#c0392b,color:#7f1d1d
  classDef done fill:#dcfce7,stroke:#16a34a,color:#14532d
  class A,H,I,J step
  class B,D,F dec
  class C,E,G stop
  class K,L done
```
