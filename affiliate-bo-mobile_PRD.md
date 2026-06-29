# PRD：Affiliate BO 手機化（RWD）— 6 頁

**版本**：v2（2026-06-29）　**類型**：前端 RWD（無後端 schema 變更）　**負責**：PM / RD
**變更摘要**：v2 補上報表頁 **Columns Settings** 手機化（顯示/隱藏＋拖曳排序＋Cancel/Reset/Save），新增 FR-9、AC-11/AC-12、ASM-4，並補充凍結首欄「隨排序後第一個顯示欄位變動」規則。
**Mockup**：`affiliate-bo-mobile_mockup.html`（手機 390px 互動原型，含 RD 開發註記開關）。
**相關**：RD 簡易 Spec `affiliate-bo-mobile_spec.md`

## 1. 需求背景
現有 Affiliate BO 為 Angular + Angular Material 桌面導向 SPA，雖已設 `width=device-width` viewport，但版面（常駐側欄、寬資料表、多欄篩選、桌面 modal）皆按桌面設計，於手機瀏覽器使用時擠壓、被迫橫向捲動。需求方確認先針對使用頻率高的 6 頁做手機版（RWD）。本案為**同一份 Angular 專案內的版面層改造**：做手機版響應式；不重寫、不另建 mobile 站、不改動桌面版。

## 2. 核心功能變更
| # | 變更 (FR) | 說明 |
|---|---|---|
| FR-1 | 響應式斷點 | `<960px` 套手機版；`≥960px` 維持現有桌面版不變。建議 CDK `BreakpointObserver`（Material Handset）。 |
| FR-2 | 側欄改抽屜 | 桌面 `mat-sidenav mode="side"` → 手機 `mode="over"` 漢堡覆蓋抽屜；群組可收合；6 個對應頁項目點擊切頁並高亮，範圍外項目不跳頁。 |
| FR-3 | 篩選重排 | 多欄橫排 → 手機單欄全寬、可收合 Filters 面板；Search／Reset 置底全寬；date range 全螢幕 picker。 |
| FR-4 | 寬表手機化 | 橫向捲動 + 凍結首欄（凍結＝排序後第一個**顯示中**的欄位，各頁預設首欄見 AC-04）；分頁／Export 功能保留；Columns Settings 詳見 FR-9。 |
| FR-5 | Tab 全寬化 | Direct／Downline segmented 全寬。 |
| FR-6 | Dashboard 卡片重排 + Tab 連動 | 卡片 4 欄→1 欄；`Direct Member` 顯示 Commission Rate 卡（9 張），`Direct + Downline AFF Member` 隱藏該卡（8 張）。卡片不可點。 |
| FR-7 | 金額顯示規範 | 金額右對齊、千分位、2 位小數；負值紅字；百分比保留原精度。 |
| FR-8 | 觸控／a11y | 觸控目標 ≥44×44px；建議解除 `user-scalable=no`。 |
| FR-9 | Columns Settings 手機化 | 報表寬表頁的 Columns Settings 改為**全螢幕面板**：每欄一列＝拖曳 handle ＋ 顯示/隱藏開關，可**顯示/隱藏**與**拖曳排序**；底部 Cancel／Reset（回預設：全部顯示、原順序）／Save，**Save 才套用**至表格；至少保留 1 欄。建議用 CDK `DragDrop`（支援觸控）。Downline 頁兩張表各有獨立 Columns Settings。設定保存沿用現站機制，本案不新增。 |

範圍 IN：Dashboard、Player List、Commission Report、Downline Commission Report、Player's Report、Tracking Link（6 頁）。OUT：Notification、Sub Affiliates、Top Up for Affiliates/Players、Bets/Transactions/Games/Ads Report、Promotion Banner、Traffic Statistics、Settings(Profile/Contact Us)、桌面版（`≥960px`）版面。

## 3. 介面設計
以 Mockup 為準：`affiliate-bo-mobile_mockup.html`（手機 390px）。重點：
- 頂部 app bar：漢堡 + 頁標題 + Export（下載）+ 帳號。
- 側欄抽屜：5 群組（Notification／Management／Reports／Marketing／Settings），群組可收合。
- 寬表：首欄 sticky 凍結並補背景色，其餘欄位橫向捲動；提供「左右滑」提示。各頁預設凍結首欄詳見 §7 AC-04。
- Columns Settings：點表格下方按鈕開全螢幕面板，每欄一列＝拖曳 handle ＋ 顯示/隱藏開關；底部 Cancel／Reset／Save。詳見 FR-9 / AC-11 / AC-12。
- Dashboard：卡片 1 欄；Tab 連動顯示/隱藏 Commission Rate 卡。
- Tracking Link：卡片堆疊、長 URL 換行、COPY 全寬 ≥44px、成功 toast「Copied」、Exclusive Domain 給英文 empty-state。
- 視覺沿用現有 Material 綠色主題，不重新設計。

**BO 欄位表**：N/A —— 本案為純版面（RWD）改造，無新增表單欄位；既有報表欄位顯示精度規則見 §4 與 §8。

## 4. 資料模型
N/A —— 本案無資料表／schema 變更，無新增 API。僅約束顯示精度：金額對應後端 `DECIMAL(18,2)`（顯示 2 位小數、千分位、右對齊、負值紅字），百分比保留原精度。實際型別以後端既有 schema 為準。

## 5. 流程圖
N/A —— 本案無狀態流／條件分支，純版面響應式切換。唯一動態行為：
1. 視窗寬度 `<960px` → 套手機版版面；`≥960px` → 桌面版。
2. Dashboard Tab：`Direct Member` → 顯示含 Commission Rate 的 9 張卡；`Direct + Downline AFF Member` → 隱藏 Commission Rate（8 張）。

## 6. 選單位置
沿用現有選單結構，本案**不新增選單項、不變更選單權限**；權限沿用 Affiliate BO 現有角色機制（現有角色模型細節以系統現況為準）。下列 6 個既有選單項做手機化：

| 選單路徑 | 角色 / 權限 | 說明 |
|---|---|---|
| Management > Player List | 沿用現有角色 | 本案手機化 |
| Reports > Commission Report | 沿用現有角色 | 本案手機化 |
| Reports > Downline Commission Report | 沿用現有角色 | 本案手機化 |
| Reports > Player's Report | 沿用現有角色 | 本案手機化 |
| Marketing > Tracking Link | 沿用現有角色 | 本案手機化 |
| （登入首頁）Dashboard | 沿用現有角色 | 本案手機化；選單無此項，由左上 logo 進入 |

## 7. 驗收標準（AC）
| AC-ID | 對應 FR | 驗收條件 |
|---|---|---|
| AC-01 | FR-1 | 視窗 `<960px` 時 6 頁套手機版版面；`≥960px` 時版面/欄位/功能與現行桌面版一致。 |
| AC-02 | FR-6 | Dashboard Tab=Direct Member 顯示 9 張卡（含 Commission Rate）；Tab=Direct+Downline 顯示 8 張（不含）。 |
| AC-03（負向） | FR-6 | 切到 Direct+Downline 後，Commission Rate 卡**不得**出現；卡片**不得**可點。 |
| AC-04 | FR-4 | 各寬表手機版預設凍結首欄（Player List=Date of Registration、Commission=Year、Downline 兩表=Agent Prefix、Player's Report=Username），其餘欄位可橫向捲動；經 Columns Settings 排序/隱藏後，凍結欄＝第一個顯示中的欄位。 |
| AC-05（負向） | FR-1 | 套用手機版後，桌面版（`≥960px`）版面、欄位、查詢/匯出/分頁功能**不得**改變（回歸不破）。 |
| AC-06 | FR-2 | 側欄 6 個對應頁項目點擊切到正確頁並高亮；範圍外項目**不得**跳頁。 |
| AC-07 | FR-7 | 金額欄右對齊、千分位、2 位小數；負值（如 Net Revenue／Total Commission）以紅字顯示。 |
| AC-08（負向） | FR-7 | 金額**不得**因前端以 float 處理造成顯示精度漂移（顯示層固定 2 位小數）。 |
| AC-09 | FR-4 | Tracking Link COPY 複製正確 URL 並顯示「Copied」；COPY 觸控目標 ≥44px。 |
| AC-10（負向） | FR-3, FR-4 | 篩選/搜尋/分頁/Columns Settings/Export 於手機版功能與桌面一致，**不得**因 RWD 失效。 |
| AC-11 | FR-9 | 點 Columns Settings 開全螢幕面板，列出該表全部欄位；關閉某欄並 Save → 表格不顯示該欄；拖曳調序並 Save → 表格欄序隨之改變。Downline 兩表各自獨立設定。 |
| AC-12（負向） | FR-9 | 面板按 Cancel／關閉 → 變更**不得**套用（表格維持原樣）；未按 Save 前表格**不得**變動；**不得**隱藏全部欄位（至少保留 1 欄）。 |

## 8. 非功能需求（NFR）
| NFR-ID | 類別 | 需求 |
|---|---|---|
| NFR-REL | 可靠性 | RWD 為純前端版面層，**不得**影響既有報表查詢、匯出、分頁邏輯；桌面版需通過回歸測試。 |
| NFR-DATA | 金額/資料精度 | 金額顯示精度固定 2 位小數（對應 `DECIMAL(18,2)`），百分比保留原精度；**不得**用 float 造成顯示漂移。 |

## 9. 假設與限制（ASM / CST）
| ID | 內容 |
|---|---|
| ASM-1 | 多品牌白標：同一份前端套多品牌皮（OLE777／Lucky Game／Betally）。RD 須確認 6 頁為共用元件或各品牌各一份（決定改 1 處或 N 處、影響工時）。 |
| ASM-2 | 斷點 960px 為建議值，依現有 theme 可調整。 |
| ASM-3 | 無新後端 API，沿用現有報表 API 與回傳欄位。 |
| ASM-4 | Columns Settings 已確認存在於 Commission Report；其餘報表頁與 Player List 是否具備，及設定保存範圍（per-user / per-device / 後端 preference），均沿用現站既有機制，待 RD 核對；本案不新增保存機制。 |
| CST-1 | 不得改動桌面版（`≥960px`）既有版面與功能。 |
| CST-2 | 不重新設計視覺，沿用現有 Angular Material 綠色主題。 |
| CST-3 | 範圍限定 6 頁，其餘頁面不在本案。 |
