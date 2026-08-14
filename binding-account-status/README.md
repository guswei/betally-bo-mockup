# Binding Account — Activate / Deactivate

GCP Agent BO：`2.1 Player List` → `Player's Info` → `Binding Account` 分頁新增啟用與停用。

- [Interactive mockup](https://guswei.github.io/betally-bo-mockup/binding-account-status/mockup.html)
- [PRD](https://github.com/guswei/betally-bo-mockup/blob/main/binding-account-status/binding_account_status_PRD.md)（開單用的 Textile 版以 Redmine 為準）
- [RD Spec](https://github.com/guswei/betally-bo-mockup/blob/main/binding-account-status/binding_account_status_spec.md)
- [Mermaid source](https://github.com/guswei/betally-bo-mockup/blob/main/binding-account-status/binding_account_status_flow.mmd)
- [Rendered flow](https://github.com/guswei/betally-bo-mockup/blob/main/binding-account-status/diagrams/binding_account_status_flow.png)

Mockup 右上可切換「顯示 RD 註記」與權限開關。黃底文字為繁中開發註記，UI 文案維持英文。

重點：綁定帳號新增 `status`（`ACTIVE` / `INACTIVE`）；唯一性約束涵蓋停用中的資料，自然鍵比對前先正規化；停用帳號在會員端與提款建立 API 兩層都要擋；刪除維持現狀但新增二次確認。
