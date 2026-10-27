# Meta CAPI 事件加帶比對參數（BA032）

BA032 的 Meta `Purchase`、`AddToCart`、`CompleteRegistration` 事件加帶 `fbc`、`fbp`、`external_id`、IP、UA 等比對參數，以 Admin BO 的 Agent 層級開關 `Meta CAPI Customer Matching` 控制。只處理 Web／H5。

- [Mockup](https://guswei.github.io/betally-bo-mockup/meta-capi-match-params/mockup.html)
- [PRD](https://github.com/guswei/betally-bo-mockup/blob/main/meta-capi-match-params/meta_capi_match_params_PRD.md)（開單用的 Textile 版以 Redmine 為準）
- [RD Spec](https://github.com/guswei/betally-bo-mockup/blob/main/meta-capi-match-params/meta_capi_match_params_spec.md)
- [Mermaid source](https://github.com/guswei/betally-bo-mockup/blob/main/meta-capi-match-params/meta_capi_match_params_flow.mmd)
- [Rendered flow](https://github.com/guswei/betally-bo-mockup/blob/main/meta-capi-match-params/diagrams/meta_capi_match_params_flow.png)

重點：前端落地時取得 `fbc`、`fbp` 並在瀏覽器保存 7 天，註冊、登入、發起存款時帶給後端；後端保存到玩家資料與存款單。開關 `Enable` 時 sender 帶上比對參數，`Deny` 時事件與現況完全相同。報表 Details 的 `client_ip_address` 顯示 `***`。
