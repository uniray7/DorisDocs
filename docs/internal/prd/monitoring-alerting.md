# 監控與告警

> 來源：[PRD](README.md) § 核心功能大綱｜狀態：骨架（目前內容散在各文件，之後收斂到這裡）

## 現況：各類告警與出處

| 類別 | 內容 | 收件人 | 目前出處 |
|------|------|--------|----------|
| Infra 告警（self-managed） | 節點宕機、磁碟水位、FE／BE 進程異常、cluster 不可用 | ws 聯絡人 | [self-managed-operations.md](self-managed-operations.md) §2 |
| 用量告警 | 儲存或 QPS 達 80% 預警、超標、查詢被 kill | ws 聯絡人（強制，不可關閉） | [quota-and-usage.md](quota-and-usage.md) |
| Pipeline 告警（managed） | consumer lag、corrupted data 數量、寫入失敗 | ws 聯絡人 | [streaming-ingestion.md](streaming-ingestion.md)、[batch-ingestion.md](batch-ingestion.md) |
| Cluster 穩定性巡檢 | 儲存健康巡檢（占用、tablet 健康、compaction）、緊急介入 | 平台＋ws 聯絡人 | [RFC-003](../rfc/RFC-003-self-managed-ddl-governance-and-rr.md) §7 |
| FE metadata 風險 | FE heap、tablet 數、回收站大小 | 平台 | [ddl-governance.md](ddl-governance.md)（TBD-29） |

## 待確認

| 主題 | TBD |
|------|-----|
| 監控堆疊（例：Prometheus／Grafana 或內部系統）；80% 預警由哪個系統計算、哪個系統只接收事件 | TBD-33 |
| 各告警門檻數值 | TBD-15 |
