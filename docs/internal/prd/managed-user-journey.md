# Managed 申請與使用旅程

> 來源：[PRD](README.md) § 核心功能大綱｜狀態：草稿
> 把使用者申請 managed service 到日常使用的流程串成一條線；每一步的細節在對應的 feature 文件。Self-managed 的流程見 [self-managed-operations.md](self-managed-operations.md)。

## 旅程總覽

```
① 申請（人工）
   填投影片 template（服務模式、使用的 fab、資料量、QPS、schema…）
   → 每週三申請會議 → 審核：申報 ws 整體 data size，決定各 fab 用 shared 或 dedicated
② Staging 驗證（managed 必經）
   租借固定小規格環境 → 灌測試資料 → 提交試用報告 → 審核設計合理性
③ 開通
   平台人員透過 Web Console Admin 後台，在每個申請的 fab 建立
   raw／curated databases、resource group（不建立 service account）；
   申請附的 schema 由平台建成 raw 初始表
④ 申請 service account 與設定權限
   Owner 透過 Console 申請 service account、設定權限
   （自助功能為後續交付，上線前由平台人員代為操作）
⑤ 寫入資料
   使用者把資料送進平台 ingestion pipeline（streaming／batch）→ 平台寫入 raw
⑥ 建表與加工
   raw 新表：向平台申請，由平台建立
   curated：service account 直接建表、用 INSERT … SELECT 從 raw 加工
⑦ 看用量
   儀表板看儲存、QPS 占比；達 80% 收到預警
⑧ Tier 升級
   用量持續逼近上限 → 申請升級 → 必要時搬遷資料到新 cluster
   完成後回到 ⑤⑥⑦，用新的配額繼續
```

## 各步驟與文件

| 步驟 | 使用者做什麼 | 平台做什麼 | 細節 | 待決 |
|------|-------------|-----------|------|------|
| ① 申請 | 填投影片、在週三會議報告 | 審核、判定 tier 與各 fab 的 cluster 類型 | [workspace-application.md](workspace-application.md) | TBD-26（命名）、TBD-7（schema 是否審核） |
| ② Staging | 灌測試資料、提交試用報告 | 佈建 staging、審核報告 | [staging-validation.md](staging-validation.md) | 租期政策 |
| ③ 開通 | — | Admin 後台開通（不建立 service account） | [web-console.md](web-console.md) | TBD-34 |
| ④ Service account | Owner 申請 service account；選 fab、cluster、database／table、service account 與權限後送出 | 建立 service account、交付憑證、執行 grant／revoke | [access-control.md](access-control.md)、[authorization.md](authorization.md) | TBD-24 |
| ⑤ 寫入 | 送 CDC message 或上傳批次檔 | Pipeline 寫入 raw | [streaming-ingestion.md](streaming-ingestion.md)、[batch-ingestion.md](batch-ingestion.md) | TBD-28（tmp 取消後的流程） |
| ⑥ 建表與加工 | 申請 raw 新表；curated 直接建表與加工 | 建立 raw table、留稽核紀錄 | [ddl-governance.md](ddl-governance.md) | TBD-7、TBD-30、TBD-32、TBD-40 |
| ⑦ 看用量 | 看儀表板、設定告警 | 計量、強制告警 | [quota-and-usage.md](quota-and-usage.md) | TBD-33 |
| ⑧ 升級 | 申請升級、配合遷移窗口 | 覆核、佈建新 cluster、搬遷資料 | [cluster-topology-and-tiering.md](cluster-topology-and-tiering.md)、[data-migration.md](data-migration.md) | TBD-1、TBD-35 |

## 銜接時要注意

- **加 quota＝升 tier**：quota 是 tier 的附帶產物，沒有獨立調整的流程；「只加儲存、不加運算」目前做不到（TBD-1）。
- **兩個數字可能不一致**：建表申請當下的配額檢查用即時數字，儀表板的儲存用量有刷新週期（TBD-15）。對外文件要說明，避免使用者困惑「儀表板還沒滿，申請卻被擋」。
- **初始 raw 表也有紀錄**：開通時建立的 raw 初始表會回寫一筆 DDL 申請紀錄，schema 版本歷史從第一天就完整。
- **80% 預警只發一次**：由哪個系統計算、哪個系統只接收事件，見 TBD-33，避免兩邊各算一次造成重複告警。
