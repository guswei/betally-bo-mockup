# OTP Verify Code 解遮罩

GCP Agent BO：`OTP Report` 的 `Verify Code` 欄從全遮罩改為「符合資格時可逐列解遮罩」。

- [Mockup](https://guswei.github.io/betally-bo-mockup/otp-unmask/mockup.html)
- [PRD](https://github.com/guswei/betally-bo-mockup/blob/main/otp-unmask/otp_unmask_PRD.md)（開單用的 Textile 版以 Redmine 為準）
- [RD Spec](https://github.com/guswei/betally-bo-mockup/blob/main/otp-unmask/otp_unmask_spec.md)
- [Mermaid source](https://github.com/guswei/betally-bo-mockup/blob/main/otp-unmask/otp_unmask_flow.mmd)
- [Rendered flow](https://github.com/guswei/betally-bo-mockup/blob/main/otp-unmask/diagrams/otp_unmask_flow.png)

重點：三個條件全部成立才可解遮罩——有 OTP 明碼檢視權限、`Verify Proc` 為 `FORGET_PASSWORD`、該碼已失效（`now > expiry` 或 `verify_time` 有值）。權限不足時圖示隱藏，碼仍有效時圖示 `disabled` 並說明原因。列表 API 與 `Export CSV` 一律不含明碼。每次解遮罩寫一筆審計，不去重，審計本身不存明碼。
