# [Agent BO] 存取款訂單覆寫權限樹更新 #732

**版本**：v1.0（2026-09-23）　**類型**：功能變更　**負責**：PM
**Mockup**：https://guswei.github.io/betally-bo-mockup/order-override-permission/mockup.html
**需求來源**：https://hub.tri-7.com/crm/type/162/details/732/
**相關**：延續 ticket #524：https://redmine.sitclouds.com/issues/24450；[RD Spec](https://github.com/guswei/betally-bo-mockup/blob/main/order-override-permission/order_override_permission_spec.md)

## 1. 需求背景

ticket #524 上線了權限 `ALLOW PERFORM ORDER PROCESSED BY OTHER PERSON`，讓操作者能處理別人鎖住或認領中的存提款單。它是單一開關，一勾就同時得到 Unlock、Approve、Reject 三種能力，營運無法只給其中一部分

客戶要求拆開授權，例如讓某個角色能駁回別人鎖住的單，但不能核准

本案把這個權限拆成九個子節點，並讓兩個列表頁上的按鈕依子節點分別顯示。#524 的既有規則全部保留，包含存款受鎖單開關約束、取款不受職責分離開關約束

## 2. 核心功能變更

| # | 變更 (FR) | 說明 |
|---|---|---|
| FR-1 | 權限拆為九個子節點 | Deposit List 下拆 Unlock、Approve、Reject；Withdraw List 下分 Risk Verification（Uncheck、Approve、Reject）與 Finance Approval（Unlock、Approve、Reject） |
| FR-2 | 勾選連動與中間態 | 父層依子層推算為勾選、中間態或未勾；點擊父層可一次開關底下全部子節點 |
| FR-3 | 列表頁按鈕依子節點個別顯示 | 別人鎖住或認領中的單，三顆按鈕各自依對應的子節點顯示或隱藏 |
| FR-4 | 生效條件 | 子節點須搭配該階段的基礎權限；存款受鎖單開關約束；取款不受職責分離開關約束；自己的單不需子節點 |
| FR-5 | 後端逐節點驗權 | 既有 Unlock、Approve、Reject API 依子節點與基礎權限逐一驗權 |
| FR-6 | 既有角色 migration | 上線時依 #524 原開關值轉換，各角色行為不變 |
| FR-7 | 操作後的既有欄位寫入與清除 | 不新增欄位。Approve 與 Reject 寫入實際執行者與執行時間，覆寫原鎖單人或認領人；Unlock 與 Uncheck 清除鎖單人或認領人及其時間 |

範圍：`11.2 Role Setting` 的 Payment Management 權限樹、`3.1 Deposit List` 與 `3.2 Withdraw List` 的操作按鈕、既有操作 API 的驗權。OUT：`Force Approve` 的行為、兩個 System Config 開關、既有基礎權限節點、訂單狀態與操作 API 的新增、獨立的 Reject 基礎權限、設定頁上子節點與基礎權限的連動、新的紀錄欄位

## 3. 介面設計

以 Mockup 為準：https://guswei.github.io/betally-bo-mockup/order-override-permission/mockup.html

- Role Setting：`ALLOW PERFORM ORDER PROCESSED BY OTHER PERSON` 底下展開子節點。部分勾選時父層顯示中間態，外觀與未勾、勾選皆不同
- Deposit List：別人鎖住的 `LOCK` 單，`Actions` 欄的紅鎖、綠勾、紅叉依 Deposit 三個子節點分別顯示
- Withdraw List：別人認領中的 `CHECKING` 單，`Risk Verification` 欄的紅鎖、雙勾、紅叉依 Risk 三個子節點顯示；別人鎖住的 `CHECKED` 單，`Finance Approval` 欄的紅鎖、綠勾、紅叉依 Finance 三個子節點顯示
- 權限不足時按鈕隱藏，不使用 disabled

**BO 欄位表**：

| 欄位 | 型別/精度 | 必填 | 說明 / enum / 邊界 |
|---|---|---|---|
| 九個子節點 | `BOOLEAN` | 是 | 各自獨立儲存。新建角色預設 `false`。前端隱藏按鈕＋後端逐節點驗權 |
| 父層顯示狀態 | 推算值 | — | 由子節點推算為勾選、中間態、未勾，不儲存 |

## 4. 資料模型

九個子節點存於既有角色權限資料，型別為布林值，與既有權限節點的儲存方式相同。父層與兩個子群組的狀態由子節點推算，不另外儲存

| Table | 欄位 | 型別 | 約束 |
|---|---|---|---|
| 既有角色權限資料 | 九個子節點 | `BOOLEAN` | `NOT NULL`，預設 `false` |

Migration：依各角色 #524 原開關值轉換。原本開的，該列表底下子節點全部開；原本關的全部關。存款與取款各自處理

## 5. 流程圖

操作者按下 override 按鈕時的判斷順序

![訂單覆寫權限判斷流程](https://guswei.github.io/betally-bo-mockup/order-override-permission/diagrams/order_override_permission_flow.png)

## 6. 選單位置

沿用現有，無新增選單項。權限不綁定 4-tier 層級，由角色權限設定控制

| 選單路徑 | 角色 / 權限 | 說明 |
|---|---|---|
| `11.2 Role Setting` → `Payment Management` | 沿用既有 Role Setting 授權 | 九個子節點在此設定 |
| `3.1 Deposit List` | `APPROVED` ＋ Deposit 子節點 | 別人鎖住的單上的三顆按鈕 |
| `3.2 Withdraw List` | Risk 欄：`CHECK` ＋ Risk 子節點；Finance 欄：`APPROVED` ＋ Finance 子節點 | 別人認領或鎖住的單上的按鈕 |

## 7. 驗收標準（AC）

| AC-ID | 對應 FR | 驗收條件 |
|---|---|---|
| AC-01 | FR-1 | Role Setting 的 Deposit List 下，`ALLOW PERFORM ORDER PROCESSED BY OTHER PERSON` 底下顯示 Unlock、Approve、Reject 三個子節點。 |
| AC-02 | FR-1 | Withdraw List 下顯示主項、`Risk Verification`（Uncheck、Approve、Reject）與 `Finance Approval`（Unlock、Approve、Reject）。 |
| AC-03 | FR-2 | 點擊未勾或中間態的父層，底下子節點全部勾選；點擊已勾的父層，底下子節點全部取消。 |
| AC-04 | FR-2 | 子節點全勾時父層自動顯示勾選；部分勾時父層顯示中間態；全不勾時父層未勾。 |
| AC-05 | FR-2 | Withdraw 的 Risk 三項全勾、Finance 只勾一項時，主項為中間態、Risk 為勾選、Finance 為中間態。 |
| AC-06 | FR-2 | 中間態的外觀與未勾、勾選三者皆可區分。 |
| AC-07 | FR-3 | 角色有 `Deposit › Approve` 與 `APPROVED` 時，別人鎖住的存款單出現綠勾；點擊後直接變 `APPROVED`，不需先 Unlock。 |
| AC-08（負向） | FR-3 | 角色沒有 `Deposit › Approve` 時，別人鎖住的存款單**不得**出現綠勾；紅鎖與紅叉仍依各自子節點顯示。 |
| AC-09 | FR-3、FR-7 | 對別人認領中的 `CHECKING` 取款單執行 Uncheck，單退回 `PENDING` 並清除 `Checked By` 與 `Checked Date`。 |
| AC-10（負向） | FR-3 | `CHECKED` 的取款單**不得**出現 Uncheck；直接呼叫 Uncheck API 回 `409` 且**不得**變更狀態。 |
| AC-11 | FR-3 | 對別人鎖住的 `CHECKED` 取款單執行 Finance Approve，出現選出款通道對話框，送出後變 `PROCESSING`。 |
| AC-12 | FR-3、FR-7 | 對別人鎖住的 `CHECKED` 取款單執行 Finance Unlock，狀態維持 `CHECKED`，清除 `Processed By` 與 `Processed Time`。 |
| AC-13（負向） | FR-4 | 角色勾了子節點但缺該階段的基礎權限時，對應按鈕**不得**出現；直接呼叫 API 回 `403` 且**不得**變更資料。 |
| AC-14 | FR-4 | 存款鎖單開關為 No 時，沒有 `LOCK` 狀態的存款單，Deposit 三個子節點勾與不勾的畫面與行為相同。 |
| AC-15 | FR-4 | 取款六個子節點在職責分離開關為 Yes 或 No 時皆照常生效。 |
| AC-16 | FR-4 | 自己鎖住或認領的單不需要子節點，依基礎權限即可操作。 |
| AC-17（負向） | FR-5 | 對**別人鎖住**的 Finance 單，有 `Risk Verification › Approve` 而沒有 `Finance Approval › Approve` 的操作者，呼叫 Finance Approve API 回 `403` 且**不得**變更資料。 |
| AC-18（負向） | FR-5 | 操作者建立的單，即使持有任何子節點，也**不得**由該操作者審核。 |
| AC-19 | FR-6 | 上線時，#524 開關原本為開的角色，該列表底下子節點全部為開；原本為關的全部為關；各角色實際行為與上線前相同。 |
| AC-20（負向） | FR-7 | 以 override 對別人鎖住或認領的單執行 Approve 或 Reject 後，該階段的操作者與時間欄位寫入實際執行者與執行時間，**不得**保留原鎖單人或認領人：Deposit 為 `Updated By`／`Updated Time`，Risk 為 `Checked By`／`Checked Date`，Finance 為 `Processed By`／`Processed Time`。 |
| AC-21 | FR-4 | 只有某階段的 Unlock 或 Uncheck 與該階段基礎權限、沒有同階段 Approve 的操作者，釋放別人鎖住或認領的單後，可自行認領並依一般流程核准。Deposit、Risk Verification、Finance Approval 三個階段皆同。 |
| AC-22（負向） | FR-7 | 對別人鎖住的存款單執行 Unlock 後，`Locked By And Time` 的鎖單人與鎖單時間皆須清空，**不得**保留原值，也不得改寫為其他值。 |

## 8. 非功能需求（NFR）

| NFR-ID | 類別 | 需求 |
|---|---|---|
| NFR-REL | 可靠性 | `Force Approve`、兩個 System Config 開關與既有基礎權限的行為不得改變。Migration 後每個角色的實際行為須與上線前完全相同。 |
| NFR-DATA | 金額/資料精度 | 本案不新增金額欄位，既有金額欄位的 `DECIMAL(18,2)` 精度不受影響。九個子節點為布林值，不得以字串儲存。 |
| NFR-SEC | 安全 | 後端逐節點驗權，前端隱藏按鈕不是防線。兩個階段的 Approve 與 Reject 為不同權限鍵，不可共用。任何子節點都不得使自審防護失效。 |

## 9. 假設與限制（ASM / CST）

| ID | 內容 |
|---|---|
| ASM-1 | 既有 Unlock、Approve、Reject API 的授權檢查是集中在同一處或散在各 endpoint，需 RD 確認。影響逐節點驗權的改動範圍。 |
| ASM-2 | 自審防護目前是否已對 Manual Deposit 與 Manual Withdraw 生效，需 RD 確認。AC-18 以其已生效為前提；若現行未實作，屬另案。 |
| CST-1 | `Force Approve` 維持現行行為，與 override Approve 並存。 |
| CST-2 | 兩個 System Config 開關本身不改動。 |
| CST-3 | 不新增獨立的 Reject 基礎權限，Reject 跟隨所屬階段的基礎權限。 |
| CST-4 | 取款子節點不受職責分離開關約束，持有者可對自己做過 Risk Verification 的單執行 Finance Approve。此為 #524 已上線的行為，本案刻意維持。 |
| CST-5 | 不新增紀錄欄位。既有的 By 與 Time 欄位記錄最終做出決定的人，不保存完整操作歷程：Approve 與 Reject 寫入實際執行者；Unlock 與 Uncheck 清除欄位，不在訂單上留下紀錄。 |
| CST-6 | 子節點只限制對別人仍鎖住或認領中的單直接操作。釋放後的單依一般流程處理，釋放者本人也可接手，所以沒有勾 Approve 不等於不能核准這筆單。 |
