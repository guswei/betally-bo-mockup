# 訂單覆寫權限樹 — RD 開發 Spec

- 對象系統：GCP Agent BO
- 位置：`11.2 Role Setting` → `Payment Management`；`3.1 Deposit List`；`3.2 Withdraw List`
- Mockup：https://guswei.github.io/betally-bo-mockup/order-override-permission/mockup.html （三個分頁共用同一組權限，右上可切換「顯示 RD 註記」與存款鎖單開關）
- 流程圖：https://guswei.github.io/betally-bo-mockup/order-override-permission/diagrams/order_override_permission_flow.png （第 12 節有內嵌圖與 Mermaid 原始碼）

---

## 1. 背景與範圍

ticket #524 已上線的權限 `ALLOW PERFORM ORDER PROCESSED BY OTHER PERSON`，讓操作者能處理別人鎖住或認領中的單。它目前是單一開關，一勾就同時取得 Unlock、Approve、Reject 三種能力。

客戶要求拆開授權，例如只給某角色「能駁回別人鎖住的單、但不能核准」。

本次做三件事：

1. `Role Setting` 的單一權限拆成九個子節點，父子勾選連動。
2. `Deposit List` 與 `Withdraw List` 上的三顆操作按鈕，改為依各自的子節點顯示。
3. 後端依子節點逐一驗權。

不新增訂單狀態，不新增操作 API，沿用既有的 Unlock、Approve、Reject。

---

## 2. 權限節點

| 列表 | 階段 | 子節點 | 對應按鈕 | 需同時具備的基礎權限 |
|---|---|---|---|---|
| Deposit List | — | Unlock | 紅鎖 | `APPROVED` |
| Deposit List | — | Approve | 綠勾 | `APPROVED` |
| Deposit List | — | Reject | 紅叉 | `APPROVED` |
| Withdraw List | Risk Verification | Uncheck | 紅鎖 | `CHECK` |
| Withdraw List | Risk Verification | Approve | 雙勾 | `CHECK` |
| Withdraw List | Risk Verification | Reject | 紅叉 | `CHECK` |
| Withdraw List | Finance Approval | Unlock | 紅鎖 | `APPROVED` |
| Withdraw List | Finance Approval | Approve | 綠勾 | `APPROVED` |
| Withdraw List | Finance Approval | Reject | 紅叉 | `APPROVED` |

後端只儲存九個葉節點，型別為布林值。主 checkbox 與 `Risk Verification`、`Finance Approval` 兩個子群組是由葉節點推算出的顯示狀態，不另外儲存。

權限鍵名稱沿用既有權限節點的命名慣例。兩個階段的 Approve 與 Reject 是**不同的權限鍵**，不可共用：Risk 的 Approve 讓單從 `CHECKING` 變 `CHECKED`，Finance 的 Approve 讓單從 `CHECKED` 變 `PROCESSING`，兩者是不同動作。

新建角色九個節點預設全部為關。

---

## 3. 勾選連動規則（Role Setting）

### 3.1 父層狀態由子層決定

| 子層狀態 | 父層顯示 |
|---|---|
| 全部勾選 | 勾選 |
| 部分勾選 | **中間態** |
| 全部未勾 | 未勾 |

中間態必須有獨立外觀，不可與「未勾」或「勾選」相同。部分開啟若與完全沒開長得一樣，稽核角色權限的人看到主 checkbox 沒勾，會以為這個角色不能動別人的單，底下卻可能開著 `Finance Approval → Approve`。

### 3.2 點擊父層

| 點擊前父層狀態 | 點擊後 |
|---|---|
| 未勾 | 底下全部勾選 |
| 中間態 | 底下全部勾選 |
| 勾選 | 底下全部取消 |

### 3.3 Withdraw 的三層結構

主 checkbox → `Risk Verification`／`Finance Approval` → 各三個葉節點。3.1 與 3.2 的規則逐層套用。

可以出現一個子群組全勾、另一個部分勾的組合。此時主 checkbox 為中間態，全勾的子群組為勾選，部分勾的子群組為中間態。

### 3.4 與基礎權限的關係

設定頁**不**連動子節點與基礎權限。可以只勾子節點而不勾 `APPROVED` 或 `CHECK`，照樣存得進去，但執行時不生效，見第 4 節。

---

## 4. 生效條件

一個子節點要生效，必須同時滿足：

1. 角色勾了該子節點。
2. 角色具備該階段的基礎權限（第 2 節最右欄）。
3. 對象是**別人**鎖住或認領中的單。

任一條件不成立，對應按鈕不顯示，後端回 `403`。

### 4.1 存款：受鎖單開關約束

`System Config > Deposit > Deposit order lock before approval` 為 No 時，存款不鎖單，不會有別人鎖住的存款單，三個子節點沒有作用。為 Yes 時，三個子節點才有意義。

這是 #524 已有的規則，本案不改變。

### 4.2 取款：不受職責分離開關約束

`System Config > Withdraw > Allows the "Risk Control Check" and "Finance Approval" to be performed by the same person` 不論設為 Yes 或 No，六個取款子節點照常生效。

這表示職責分離開關為 No 時，持有 `Finance Approval → Approve` 的操作者，仍可對自己做過 Risk Verification 的單執行 Finance Approve，即同一人做兩關。這是 #524 已上線的行為，本案維持不變。

### 4.3 自己鎖住的單

自己鎖住或認領中的單走原本流程，不需要任何子節點，只看基礎權限。

**釋放後可由釋放者本人接手。** 持有 Unlock 或 Uncheck 的操作者釋放別人的單之後，單回到未鎖定狀態。此時任何具備基礎權限的操作者都能依一般流程認領，再核准或駁回，釋放者本人也包含在內。

所以子節點限制的是「在別人仍鎖住或認領時直接動手」，不是「這筆單最終能做什麼」。沒有勾 `Approve` 的角色，仍可以先 Unlock、再自行鎖單，完成核准。

---

## 5. 列表頁按鈕

### 5.1 Deposit List

別人鎖住的單（狀態 `LOCK`、`Locked By` 不是自己），`Actions` 欄的三顆按鈕：

| 按鈕 | 子節點 | 動作 |
|---|---|---|
| 紅鎖 | `Deposit › Unlock` | 釋放對方的鎖 |
| 綠勾 | `Deposit › Approve` | 直接核准，不需先 Unlock |
| 紅叉 | `Deposit › Reject` | 直接駁回，不需先 Unlock |

### 5.2 Withdraw List — Risk Verification 欄

別人認領中的單（狀態 `CHECKING`、`Checked By` 不是自己）：

| 按鈕 | 子節點 | 動作 |
|---|---|---|
| 紅鎖 | `Withdraw › Risk Verification › Uncheck` | 釋放對方的認領 |
| 雙勾 | `Withdraw › Risk Verification › Approve` | 直接通過風控，不需先 Uncheck |
| 紅叉 | `Withdraw › Risk Verification › Reject` | 直接駁回，不需先 Uncheck |

**Uncheck 只作用於 `CHECKING`，不作用於 `CHECKED`。** 兩個狀態名很像：`CHECKING` 是有人正在審，`CHECKED` 是已審完通過。Uncheck 是搶回處理權，不是推翻別人已完成的風控結論。`CHECKED` 的單不顯示 Uncheck，後端對 `CHECKED` 的單收到 Uncheck 請求回 `409`。

### 5.3 Withdraw List — Finance Approval 欄

別人鎖住的單（狀態 `CHECKED`、`Processed By` 不是自己）：

| 按鈕 | 子節點 | 動作 |
|---|---|---|
| 紅鎖 | `Withdraw › Finance Approval › Unlock` | 釋放對方的鎖 |
| 綠勾 | `Withdraw › Finance Approval › Approve` | 直接核准，不需先 Unlock |
| 紅叉 | `Withdraw › Finance Approval › Reject` | 直接駁回，不需先 Unlock |

Finance Approve 沿用既有的 `Select Payment Channel` 對話框，必須選出款通道才能送出。這一步把錢送出去，是九個節點中風險最高的。

### 5.4 顯示規則

三顆按鈕**各自**依自己的子節點顯示或隱藏，不再綁在一起。例如只勾了 `Deposit › Reject`，別人鎖住的存款單上只出現紅叉。

權限不足時按鈕隱藏，不使用 disabled，這是本 BO 的既有慣例。

---

## 6. 狀態轉換

本案不新增狀態。三個動作作用在別人鎖住或認領的單上時：

| 列表 | 動作 | 轉換前 | 轉換後 |
|---|---|---|---|
| Deposit | Unlock | `LOCK` | `PENDING`，清除 `Locked By And Time` |
| Deposit | Approve | `LOCK` | `APPROVED` |
| Deposit | Reject | `LOCK` | `REJECTED` |
| Withdraw | Risk Uncheck | `CHECKING` | `PENDING`，清除 `Checked By` 與 `Checked Date` |
| Withdraw | Risk Approve | `CHECKING` | `CHECKED` |
| Withdraw | Risk Reject | `CHECKING` | `REJECTED` |
| Withdraw | Finance Unlock | `CHECKED`（有鎖單人） | `CHECKED`，清除 `Processed By` 與 `Processed Time` |
| Withdraw | Finance Approve | `CHECKED`（有鎖單人） | `PROCESSING` |
| Withdraw | Finance Reject | `CHECKED`（有鎖單人） | `REJECTED` |

Finance Unlock 後狀態維持 `CHECKED`，只清除鎖單人。Finance 階段的鎖不是狀態，是另外記錄的鎖單人。

各動作對操作者欄位的影響見第 8 節。

---

## 7. 後端驗權

既有的 Unlock、Approve、Reject API，在對象為別人鎖住或認領中的單時，依第 2 節逐一驗證對應的子節點與基礎權限。

| 情境 | 回應 |
|---|---|
| 缺子節點 | `403` |
| 有子節點但缺基礎權限 | `403` |
| Risk Uncheck 的對象不是 `CHECKING` | `409` |

前端隱藏按鈕只是顯示層，不是防線。每個子節點都要在後端獨立驗證。例如：對**別人鎖住**的 Finance 單，有 `Risk Verification › Approve` 而沒有 `Finance Approval › Approve` 的操作者，呼叫 Finance Approve API 必須回 `403`。對象是自己鎖住的單時不適用本節，依第 4.3 節只看基礎權限。

---

## 8. 紀錄

本案不新增紀錄欄位。訂單上既有的操作者與時間欄位，記錄的是**最終做出決定的人**，不是完整的操作歷程。

| 動作 | 欄位結果 |
|---|---|
| Deposit Approve／Reject | `Updated By`、`Updated Time` 寫入執行者與執行時間 |
| Risk Approve／Reject | `Checked By`、`Checked Date` 寫入執行者與執行時間，覆寫原認領人 |
| Finance Approve／Reject | `Processed By`、`Processed Time` 寫入執行者與執行時間，覆寫原鎖單人 |
| Deposit Unlock | 清除 `Locked By And Time` |
| Risk Uncheck | 清除 `Checked By`、`Checked Date` |
| Finance Unlock | 清除 `Processed By`、`Processed Time` |

**Approve 與 Reject 必須寫入實際執行者，不可保留原本的鎖單人或認領人。** 否則 override 之後，欄位顯示的是當初鎖單的人，而不是實際做決定的人。

**Unlock 與 Uncheck 不在訂單上留下紀錄。** 兩者清除鎖單人或認領人之後，無法從訂單得知是誰釋放的、原本是誰在處理。本案不為這兩個動作新增紀錄。

`Force Approve` 維持現行行為，與 override Approve 並存。

比對同一筆取款的 `Checked By` 與 `Processed By`，可以撈出第 4.2 節所述「同一人做兩關」的出款。這依賴上表「寫入實際執行者」的規則；若保留原鎖單人，同一人做兩關會顯示成兩個不同的人。

---

## 9. 既有角色 migration

#524 已上線，每個角色身上已有 `ALLOW PERFORM ORDER PROCESSED BY OTHER PERSON` 的開關值。上線時依該值轉換，存款與取款各自處理：

| 原本的值 | 轉換後 |
|---|---|
| 開 | 該列表底下所有子節點全部開 |
| 關 | 該列表底下所有子節點全部關 |

轉換後每個角色的實際行為與上線前完全相同。轉換完成才開放新的設定畫面。

---

## 10. 職責分離與自審防護

兩者是不同的規則：

- **職責分離**：同一筆取款的 Risk Verification 與 Finance Approval 不能由同一人完成，由 System Config 開關控制。第 4.2 節所述，持有子節點者可繞過。
- **自審防護**：操作者不能審核自己建立的單，例如自己開的 Manual Deposit 或 Manual Withdraw。**本案任何子節點都不得使自審防護失效。**

---

## 11. 範圍外

- 不改動 `Force Approve` 的行為。
- 不改動兩個 System Config 開關本身。
- 不改動 `APPROVED`、`CHECK` 等既有權限節點。
- 不新增獨立的 Reject 基礎權限。Reject 跟隨所屬階段的基礎權限。
- 不新增訂單狀態與操作 API。
- 設定頁不做子節點與基礎權限的連動或鎖定。
- 不新增紀錄欄位，沿用第 8 節所列的既有欄位。

---

## 12. 流程圖

操作者按下 override 按鈕時的判斷順序。

![訂單覆寫權限判斷流程](https://guswei.github.io/betally-bo-mockup/order-override-permission/diagrams/order_override_permission_flow.png)

Mermaid 原始碼：

```
flowchart TD
  A[操作者對某筆單按下<br/>Unlock / Uncheck / Approve / Reject] --> B{這筆單被別人鎖住<br/>或認領中?}
  B -->|否| N[走原本流程<br/>只看基礎權限]
  B -->|是| E{角色有這顆按鈕<br/>對應的子節點?}
  E -->|否| F[按鈕隱藏<br/>直接呼叫 API 回 403]
  E -->|是| G{角色有該階段的基礎權限?<br/>Deposit 與 Finance：APPROVED<br/>Risk：CHECK}
  G -->|否| F
  G -->|是| H[直接執行<br/>不需先 Unlock 或 Uncheck]
  classDef step fill:#dbeafe,stroke:#2563eb,color:#1e3a8a
  classDef dec  fill:#fef3c7,stroke:#d97706,color:#78350f
  classDef stop fill:#fdecea,stroke:#c0392b,color:#7f1d1d
  classDef done fill:#dcfce7,stroke:#16a34a,color:#14532d
  class A,N step
  class B,E,G dec
  class F stop
  class H done
```
