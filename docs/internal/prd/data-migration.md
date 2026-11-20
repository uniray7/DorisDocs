# 資料搬遷

> 來源：[PRD](README.md) § 核心功能大綱｜狀態：草稿｜做法由接手的 PM 與工程師研究（TBD-35）
> 範圍：平台內 workspace 的資料在 cluster 之間搬移。**不含** Oracle／HBase → Doris 的遷移（由各團隊自理，屬 Out-of-scope）。

## 為什麼需要資料搬遷

| 情境 | 說明 |
|------|------|
| Tier 升級：shared → dedicated | Tier 1 workspace 成長超過 shared cluster 的門檻，需要搬到自己的 dedicated cluster（[cluster-topology-and-tiering.md](cluster-topology-and-tiering.md)） |
| Dedicated 換規格 | Tier 2 → 3 原則上在原 cluster 加 BE 節點、不需搬遷；機型不符時才要整座搬到新 cluster |
| 降級 | 用量長期遠低於 tier，使用者申請縮容或搬回 shared |
| 換服務模式 | Managed 與 self-managed 不可事後轉換；要換模式需開新 workspace，再把資料搬過去 |
| Cluster 退役、機器汰換 | 機器汰換或 cluster 整併時，上面的 workspace 要搬到其他 cluster |

## 不論採用哪種做法都要滿足

- **資料一致**：搬遷後做一致性核對，確認無資料遺失。
- **停機最短**：目標是只在切換窗口短暫唯讀。
- **寫入處理**：managed 暫停 ingestion pipeline，積壓的資料在搬遷後追上；self-managed 由使用者配合窗口暫停寫入（責任歸屬寫入對外文件）。
- **名稱不變**：database 命名跟服務模式走、與 tier 無關，搬遷不改名。
- **帳號與權限**：Doris user、role、grant 屬於各別 cluster，搬遷時要在新 cluster 重建（由平台後端執行，見 [authorization.md](authorization.md)）。
- **連線端點**：搬遷後交付新連線資訊；或評估由 gateway 讓端點不變（[RFC-002](../rfc/RFC-002-shared-cluster-resource-isolation.md)）。

## 候選做法（待評估）

| 做法 | 概念 | 需評估 |
|------|------|--------|
| CCR（cross-cluster replication） | 在新舊 cluster 之間建立同步，追上後切換 | 4.1.4 支援的同步粒度；對來源 cluster 的負擔；在 shared cluster 上只搬單一 workspace 是否可行 |
| Backup／Restore（經 S3 或 MinIO） | 舊 cluster BACKUP 到物件儲存，新 cluster RESTORE | 可與 DR 的 backup 機制共用（[ha-dr.md](ha-dr.md)）；備份期間新寫入的資料如何補上；大資料量的耗時 |
| 匯出後重新匯入 | 匯出到物件儲存，再匯入新 cluster | Schema 與型別是否保真；耗時 |
| 原 cluster 加節點（不搬遷） | 在原 cluster 加 BE | 只適用同一 cluster 擴容；shared → dedicated 不適用 |

## 待研究

| 問題 | TBD |
|------|-----|
| 選定搬遷做法（可依情境不同）、搬遷流程與切換步驟、失敗時如何回退 | TBD-35 |
| 換服務模式時的搬遷由平台協助還是使用者自理 | TBD-35 |
| 備份還原機制的設計（與 DR 共用） | TBD-36 |
