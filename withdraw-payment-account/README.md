# Payment Account 管理（出款指定帳號、帳號餘額與手動單）

GCP Agent BO：財務核准出款時可以指定由哪個 `BANK-OFFLINE` 帳號付款；系統記錄每個帳號的餘額與每一筆變動；新增 `3.11 Payment Account Transactions` 頁查交易紀錄、看每帳號合計，並建立內部轉帳、轉到外部、調整餘額三種手動單。Admin BO 可以決定每個 Agent 能不能使用 `3.11`。

- [Mockup：Agent BO](https://guswei.github.io/betally-bo-mockup/withdraw-payment-account/withdraw_payment_account_mockup.html)（左側選單切換 3.2／3.7／3.11，上方可切換權限節點與 RD 註記）
- [Mockup：Admin BO](https://guswei.github.io/betally-bo-mockup/withdraw-payment-account/withdraw_payment_account_mockup_admin.html)（Agent List › Edit › Role Setting）
- [操作示範影片](https://guswei.github.io/betally-bo-mockup/withdraw-payment-account/withdraw_payment_account_demo.mp4)（約 2 分鐘，字幕＋旁白，走過主要操作）
- [PRD](https://github.com/guswei/betally-bo-mockup/blob/main/withdraw-payment-account/withdraw_payment_account_PRD.md)（開單用的 Textile 版以 Redmine 為準）
- [RD Spec](https://github.com/guswei/betally-bo-mockup/blob/main/withdraw-payment-account/withdraw_payment_account_spec.md)
- Mermaid source：[核准出款](https://github.com/guswei/betally-bo-mockup/blob/main/withdraw-payment-account/withdraw_payment_account_flow_approve.mmd)、[手動單](https://github.com/guswei/betally-bo-mockup/blob/main/withdraw-payment-account/withdraw_payment_account_flow_manual.mmd)
- Rendered flow：[核准出款](https://github.com/guswei/betally-bo-mockup/blob/main/withdraw-payment-account/diagrams/withdraw_payment_account_flow_approve.png)、[手動單](https://github.com/guswei/betally-bo-mockup/blob/main/withdraw-payment-account/diagrams/withdraw_payment_account_flow_manual.png)

重點：全部功能只適用於 `BANK-OFFLINE` 通道的帳號。餘額不可為負，交易紀錄不可修改或刪除。指定帳號出款、內部轉出、轉到外部都要通過 `Allow Withdraw`、餘額與每日出款門檻三個條件。上線時餘額為 0，不回補歷史。
