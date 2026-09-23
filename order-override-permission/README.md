# 訂單覆寫權限樹

GCP Agent BO：把 ticket #524 的 `ALLOW PERFORM ORDER PROCESSED BY OTHER PERSON` 拆成九個可個別授權的子節點，存提款列表頁的按鈕依子節點分別顯示。

- [Mockup](https://guswei.github.io/betally-bo-mockup/order-override-permission/mockup.html)（三個分頁共用同一組權限）
- [PRD](https://github.com/guswei/betally-bo-mockup/blob/main/order-override-permission/order_override_permission_PRD.md)（開單用的 Textile 版以 Redmine 為準）
- [RD Spec](https://github.com/guswei/betally-bo-mockup/blob/main/order-override-permission/order_override_permission_spec.md)
- [Mermaid source](https://github.com/guswei/betally-bo-mockup/blob/main/order-override-permission/order_override_permission_flow.mmd)
- [Rendered flow](https://github.com/guswei/betally-bo-mockup/blob/main/order-override-permission/diagrams/order_override_permission_flow.png)

重點：每個子節點須搭配該階段的基礎權限才生效（Deposit 與 Finance 為 `APPROVED`、Risk 為 `CHECK`）。部分勾選時父層顯示中間態。Risk 的 Uncheck 只作用於 `CHECKING`，不作用於 `CHECKED`。存款子節點受鎖單開關約束；取款子節點不受職責分離開關約束，此為 #524 既有行為。不新增紀錄欄位，沿用既有的 By 與 Time 欄位。
