# Notification Enhancement — Spec

> 平台:BetAlly。介面:BO(Player Management → Notification Management)+ 會員端(WebFE / App)。
> 互動 mockup(視覺參考):<https://guswei.github.io/betally-bo-mockup/notification_enhancement_mockup.html>
> 最後更新:2026-06-23

---

## 1. 需求背景
1. **BO 通知管理**清單需新增以 **Username / User ID** 搜尋,並能依 **已讀 / 未讀** 過濾。
2. **會員端(WebFE / App)** 通知內文目前連結是純文字、不可點,需變成可點擊,提升體驗。

---

## 2. 功能變更

### 2.1 需求1 — BO Notification Management 過濾增強
本頁僅動「過濾區 + 清單欄位」,現有 `時間區間 / Type / Title / Content` 不變。

| 新增項 | 位置 | 型別 / 值 | 行為 |
|---|---|---|---|
| **Username or ID** | 過濾區 | `text`,前後去空白 | **完全比對(exact match)**;空值=不過濾;查無→空清單 + empty state,**不報錯** |
| **Read Status** | 過濾區 | enum `All / Read / Unread`,預設 `All` | 依已讀狀態過濾 |
| **Read Status 欄** | 清單欄位 | badge `Read`(綠)/ `Unread`(紅) | 走現有 `Columns Settings` 可開關,預設開 |

- 所有過濾條件 **AND 疊加**(可與 Type / Title / Content / 時間並用)。
- 操作者 = `Admin`,全部會員通知皆可查。**本期不做 Brand 隔離。**

### 2.2 需求2 — 會員端通知連結可點(WebFE / App)
做法:**自動 linkify**——前端在渲染(展開)通知內文時,偵測內文中的 `http(s)://` 網址,轉成可點連結。**不需改 BO 撰寫端**,管理者只要在 Content 貼網址即可,舊通知同步生效。

- 偵測:`https?://` 起,到空白 / 換行 / 中文字前止。
- 一則內多個網址 → 各自獨立可點。
- 結尾標點(`. , ) 」` 等)不納入連結。
- 純文字 `www.`(無協定)**不轉**。
- 開啟方式:**WebFE → 新分頁**(`target="_blank"` + `rel="noopener noreferrer"`);**App → 系統瀏覽器**。

---

## 3. 安全基線(硬需求,RD 必做)
> 與「連結網域全放行」**無關**——這是 XSS 防線,不可省。會員端為登入態金流頁。

- **禁止**把 Content 當 HTML 塞 `innerHTML` / `v-html` / `dangerouslySetInnerHTML`。
- 必須 **escape-then-linkify**:整段內文先 HTML-escape,再「只」把偵測到的 URL 片段包成 `<a>`,其餘一律純文字。
- 連結只允許 `http` / `https` 協定,**拒絕** `javascript:` / `data:` / `vbscript:`(即使網域全放行,協定仍要擋,否則等於可點的腳本注入)。
- 所有外連 `<a>` 一律 `target="_blank"` + `rel="noopener noreferrer"`。

參考實作見 mockup 內 `safeLinkify()`(escape → 只換合法 http(s) URL → 加 rel=noopener)。

---

## 4. 後端 / 資料影響(🚩 RD 先確認)
**前置確認**:後端是否已記錄每筆通知的會員已讀狀態(`is_read` / `read_at`)?

- **若已有** → 需求1 的 Read 過濾與欄位 = **純前端**;會員端展開通知時若尚未回寫,補上回寫即可。
- **若沒有** → 需 **新增 `is_read` / `read_at` 欄位** + 會員端開啟通知時**回寫已讀** + BO 顯示。屬 full-stack,工時較大。

> 需求2(linkify)為純前端,**不卡此確認**,可獨立先做。

---

## 5. 範圍外(本期不做)
- BO 撰寫端新增正式超連結 / 富文本欄位(本期用純 linkify)。
- 連結網域白名單 / 過濾(本期任何 http(s) 網址皆可點)。
- App「自家網域 → 站內 webview 保留登入態」的深連跳轉(Agent BO 發的訊息目前不需深連 App 內部頁;未來要做再補白名單路由)。
- 會員端已讀 / 未讀自助篩選(本期僅 BO 過濾)。
- Brand 隔離。
- Username 模糊比對(本期完全比對)。

---

## 6. 介面參考
互動 mockup(三介面:BO / 會員 Web / 會員 App,可開關 RD 註記):
<https://guswei.github.io/betally-bo-mockup/notification_enhancement_mockup.html>
