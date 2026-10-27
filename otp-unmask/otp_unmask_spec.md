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
2. 解遮罩的資格限制在忘記密碼流程的驗證碼。仍在有效期內的驗證碼也可以解遮罩。
3. 新增一個獨立的權限節點控制此操作。
4. 每次解遮罩寫入審計紀錄。

---

## 2. 解遮罩資格

兩個條件全部成立才可解遮罩，缺一不可：

| # | 條件 | 不成立時 |
|---|---|---|
| 1 | 操作者被授予 OTP 明碼檢視權限 | 前端不顯示眼睛圖示；後端回 `403` |
| 2 | 該列 `Verify Proc` 為 `FORGET_PASSWORD` | 前端不顯示眼睛圖示；後端回 `409` |

驗證碼是否過期、是否已使用，不影響資格。仍在有效期內的驗證碼也可以解遮罩。前端與後端都不得檢查 `Expiry` 或 `verify_time` 來擋解遮罩。

有效期內的明碼可以用來完成該玩家的密碼重設。本案以兩道控管處理這個風險。第一道：只有被授予權限的操作者看得到圖示。第二道：每一次成功解遮罩，後端都寫入審計紀錄（第 6 節）。

---

## 3. 畫面變更

### 3.1 `Verify Code` 欄

欄位位置與既有欄位順序不動。遮罩態一律顯示六個 `X`，不反映實際碼長。

儲存格內新增一個眼睛圖示，依資格呈現兩種狀態：

| 情境 | 圖示 | 儲存格內容 |
|---|---|---|
| 兩個條件全部成立 | 顯示且可點擊 | 點擊後切換為明碼 |
| 條件 1 或條件 2 不成立 | 不顯示 | `XXXXXX` |

條件 1 或條件 2 不成立時，前端隱藏圖示，不使用 `disabled`。

該列為遮罩態時，操作者點擊圖示，前端直接呼叫 reveal API。

前端呼叫 reveal API 前，不要求填寫原因，也不顯示確認視窗。

該列為明碼態時，操作者點擊圖示，前端只把該列回到遮罩態，不呼叫 reveal API。

### 3.2 解遮罩後的行為

操作者點擊一列的圖示時，前端只改變該列的顯示狀態，其他列維持原本的狀態。

操作者再次點擊同一個圖示時，前端把該列回到遮罩態。

同一頁可以同時有多列處於明碼態。

操作者執行查詢、重設篩選或換頁後，前端把已解開的欄位一律回到遮罩態。前端不快取明碼。

---

## 4. API

| Method / Path | 用途 |
|---|---|
| `POST /otp-report/{id}/reveal` | 取得單筆的 `Verify Code` 明碼 |

採 `POST` 而非 `GET`：此操作會寫入審計紀錄，是有副作用的操作。

解遮罩成功時，後端回 `200`，body 為 `{"verify_code": "<明碼字串>"}`。`verify_code` 的型別為字串。

後端在 reveal 回應的標頭加上 `Cache-Control: no-store`。

反向代理不得快取此端點的回應。

操作對象為單筆，以 OTP 記錄的 `id` 指定，不接受批次。

**列表 API 的回應不得包含明碼，不論操作者是否有權限。** 只做畫面遮罩的話，開發者工具即可取得明碼，遮罩等於不存在。

錯誤模型：

| 狀態碼 | 錯誤鍵 | 情境 |
|---|---|---|
| `403` | `FORBIDDEN` | 操作者未被授予 OTP 明碼檢視權限 |
| `404` | `OTP_RECORD_NOT_FOUND` | 記錄不存在 |
| `409` | `OTP_PROCESS_NOT_ELIGIBLE` | 該筆 `Verify Proc` 不是 `FORGET_PASSWORD` |

後端依序判定：先驗權限，再查記錄是否存在，最後查 `Verify Proc`。前一項不通過時，後端回該項的錯誤，不判定後面的項目。

三種錯誤的回應 body 都不得包含明碼。

後端一律重新驗證兩個條件，不信任前端送來的任何資格判定。

操作者有權限且記錄不存在時，後端回 `404 OTP_RECORD_NOT_FOUND`，不寫成功審計紀錄。

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

每一次成功解遮罩，後端寫入一筆紀錄，內容包含：操作者、時間、OTP 記錄 `id`、該筆對應的玩家 `username`。

**審計紀錄本身不得儲存明碼。** 紀錄的用途是知道誰在什麼時候看了哪一筆，不是保存驗證碼。

同一列重複解遮罩時，後端每次各寫一筆，不去重。

操作者再次點擊圖示回到遮罩態時，後端不寫紀錄。

被 `403`、`404` 或 `409` 拒絕的請求，後端不寫成功紀錄。

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

解遮罩的兩道資格檢查，以及通過後的後端處理順序。流程圖只畫遮罩態的點擊；明碼態的點擊只回到遮罩態（3.1）。

![OTP Verify Code 解遮罩流程](https://guswei.github.io/betally-bo-mockup/otp-unmask/diagrams/otp_unmask_flow.png)

Mermaid 原始碼：

```
flowchart TD
  A[操作者在 OTP Report 點某列遮罩態的眼睛圖示<br/>明碼態點擊只回遮罩、不呼叫 API] --> B{有 OTP 明碼檢視權限?}
  B -->|否| C[前端不顯示眼睛圖示<br/>直接呼叫 API 時後端回 403 FORBIDDEN]
  B -->|是| D{Verify Proc 是 FORGET_PASSWORD?}
  D -->|否| E[前端不顯示眼睛圖示<br/>直接呼叫 API 時後端回 409 OTP_PROCESS_NOT_ELIGIBLE]
  D -->|是| H[POST /otp-report/id/reveal<br/>不論是否過期或已使用]
  H --> I{後端依序重驗權限、記錄存在、Verify Proc<br/>通過?}
  I -->|否| M[回 403、404 或 409<br/>不寫成功審計、不回明碼]
  I -->|是| J[寫入審計<br/>操作者/時間/OTP id/玩家<br/>審計本身不存明碼]
  J --> K[只回傳該列明碼<br/>列表 API 與 Export CSV 維持遮罩]
  K --> L[畫面只解開該列<br/>查詢、重設篩選或換頁後回到遮罩]
  classDef step fill:#dbeafe,stroke:#2563eb,color:#1e3a8a
  classDef dec  fill:#fef3c7,stroke:#d97706,color:#78350f
  classDef stop fill:#fdecea,stroke:#c0392b,color:#7f1d1d
  classDef done fill:#dcfce7,stroke:#16a34a,color:#14532d
  class A,H,J step
  class B,D,I dec
  class C,E,M stop
  class K,L done
```
