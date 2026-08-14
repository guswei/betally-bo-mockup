# VIP 等級升級改為每小時檢查

GCP Agent BO：VIP 升級從每日（隔日）處理改為依可設定的間隔檢查，預設每 60 分鐘。

- [PRD (Textile)](https://github.com/guswei/betally-bo-mockup/blob/main/vip-hourly-upgrade/vip_hourly_upgrade_PRD.textile)
- [PRD (Markdown)](https://github.com/guswei/betally-bo-mockup/blob/main/vip-hourly-upgrade/vip_hourly_upgrade_PRD.md)
- [RD Spec](https://github.com/guswei/betally-bo-mockup/blob/main/vip-hourly-upgrade/vip_hourly_upgrade_spec.md)
- [Mermaid source](https://github.com/guswei/betally-bo-mockup/blob/main/vip-hourly-upgrade/vip_hourly_upgrade_flow.mmd)
- [Rendered flow](https://github.com/guswei/betally-bo-mockup/blob/main/vip-hourly-upgrade/diagrams/vip_hourly_upgrade_flow.png)

本需求無新畫面，UI 面積僅 `System Config` → `VIP` tab 的一個設定欄位，因此沒有 mockup。

重點：VIP 只升不降，`新等級 = max(現有等級, 達標等級)`；執行間隔預設 `60` 分鐘、下限 `60`；跨級時每經過一級發一筆該級獎金，總額與執行頻率無關；冪等鍵為「玩家 + 目標 VIP 等級」；CleverTap `VIP Upgrade` 事件只送最後達成的等級。
