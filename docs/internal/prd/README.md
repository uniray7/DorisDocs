# PRD：OLAP Platform（以 Apache Doris 為基礎的內部 OLAP 服務）

> 來源脈絡：[00-progress.md](../../00-progress.md) 現況摘要｜狀態：草稿（§1–3、§8–10 尚待補齊）

## 1. 背景與問題陳述
> 要填：Oracle license 成本與綁定問題、HBase 維運負擔、各部門重複自建分析方案的現況。為什麼現在做——sunset 舊 tech stack 的公司方向。

## 2. 目標與非目標
> 要填：
> - 目標：提供多租戶 OLAP 查詢服務 + data ingestion pipeline，以 managed / self-managed 兩種模式承接自 Oracle/HBase 轉移的分析型工作負載。
> - 非目標（已確定）：不負責既有資料遷移；不承接交易型（OLTP）工作負載。跨 workspace 資料分享待定（TBD-31）。

## 3. 目標使用者與使用情境
> 要填：後端工程師（會寫 SQL）為主，多來自 lakehouse 環境。代表情境：(a) 部門申請 managed workspace，以平台 CDC pipeline 接上游 DB 做準即時報表、(b) managed 使用者以 Doris ELT 從 raw 產出 curated 分析表、(c) 高客製/大流量團隊申請 self-managed dedicated cluster 自建 pipeline。

## 4. 術語表
| 名詞 | 定義 | 易混淆處 |
|------|------|----------|
| Workspace (ws) | 租戶單位，資料、metadata、運算資源的隔離邊界。申請粒度由團隊自定（團隊或部門皆可），但每個 ws 必須綁定一個成本中心作為成本歸屬單位。 | 一個成本中心可擁有多個 ws；成本報表以成本中心彙總 |
| Fab | 工廠據點。Workspace 申請時指定要在哪些 fab 使用服務（至少兩個）；平台在每個 fab 各自提供相同數量、彼此獨立的 Doris 服務，不跨 fab 部署，各 fab 各自指定 shared 或 dedicated。 | 與 HA 無關 |
| Shared Cluster | 多個 tier 1 workspace 共用的 Doris cluster，以 resource group 隔離運算。 | 資料仍完全隔離，「共用」僅指硬體 |
| Dedicated Cluster | 單一 workspace 專屬的 Doris cluster（tier 2 以上）。 | |
| Tier | Workspace 的資源等級（1/2/3），由 data volume 與 QPS 決定（門檻待修正，TBD-1）。 | 儲存與運算是否解耦待決 |
| Resource Group | Doris 的運算資源隔離機制，用於 shared cluster 內限制單一 ws 的 CPU/記憶體用量。 | 不是資料隔離機制 |
| Quota | 分配給 workspace 的資源上限（儲存量、QPS、併發數等）。 | |
| Data Volume | Workspace 占用的儲存量。**申請/審核階段**以來源原始大小估算 tier；**上線後計量**以 Doris 壓縮後實際占用計配額。 | 對外文件需提供壓縮率參考換算例 |
| Max QPS | Workspace 的查詢速率上限，以**滑動窗口平均（5 分鐘）**計量判定超標。 | 瞬間爆量由 resource group 併發限制兜底，不直接判超標 |
| 查詢協定 | 平台提供兩種查詢介面：**MySQL protocol**（完整 SQL，含 metadata SQL）與 **Arrow Flight protocol**（僅 SELECT 相關 SQL，不支援 metadata SQL 如 `SHOW DATABASES`）。 | 共用同一權限體系；細節見 access-control spec |
| Managed 模式 | 平台承擔資料管理責任（備份還原、保留清除、效能調校）的服務模式。寫入 raw 一律走平台 pipeline；raw table 由平台建立（建表前是否審核 schema：TBD-7），curated table 由 service account 直接建立（TBD-30）。 | 申請時選定，**不可事後轉換** |
| Self-managed 模式 | 使用者自行管理資料的服務模式：可自行灌資料（自建 pipeline）；database 由平台建立，table DDL 目前一律提交平台代建，是否因 dbt 開放直接 DDL 待定（TBD-29）。平台責任限縮為日常 infra 告警與代操作（scale in/out，目前無付費機制）；其他需求（使用問題/新版本/新功能/建表諮詢/效能優化）提 request 由 PM 排優先權，非平台義務。**限定 dedicated cluster（tier 2+）**。 | 平台不負資料管理責任；不可用平台 pipeline |
| Ingestion Job | 使用者在平台 pipeline 上設定的一條匯入任務（batch 或 streaming）。**僅 managed 模式可用**。 | |
| Batch Ingestion | 平台提供的批次匯入通道（僅 managed）。 | |
| Streaming Ingestion | 平台提供的準即時匯入通道（僅 managed）：使用者以固定 format 的 message 灌進平台 Kafka，由平台 pipeline 寫入 raw（tmp zone 已取消，流程待改寫：TBD-28）。支援三種 format：generic CDC event、DB2 JSON CDC、DB2 XML CDC（後兩者規格見內部文件）。 | Format 驗證失敗落 corrupted data（MinIO 指定路徑）並告警 |
| DDL 治理 | Database 由平台建立。Managed：raw table 由平台建立（是否審核：TBD-7），curated table 由 service account 直接建立（TBD-30）。Self-managed：一律平台代建（TBD-29）。細節見 [ddl-governance.md](ddl-governance.md)。 | 與 workspace 申請是兩條獨立流程 |
| Zone | Managed 的資料分層：**raw**（平台 pipeline 寫入、平台建表）→ **curated**（service account 以 `INSERT … SELECT` 產出）。tmp zone 已取消。 | 僅 managed 適用 |
| Database 命名 | `{ws}__{user_defined}__{raw\|curated}`（雙底線分隔，zone 置尾）；規則與白名單見 [authorization.md](authorization.md)（TBD-26）。 | 僅 managed；self-managed 命名待定（TBD-8） |
| Staging 環境 | 正式開通前的**固定小規格試用環境**（不隨目標 tier 調整）：使用者租借後灌測試資料（自產假資料或自 production lakehouse 匯入）試用，提交設計合理性報告，通過才開通正式環境。 | 效能數據僅供參考、不可外推，不構成平台效能承諾 |
| HA | Cluster 部署在多台機器、每個 table 3 個 replica，單一機器故障可容錯；細節見 [ha-dr.md](ha-dr.md)。 | 不採用 active-standby＋CCR |
| FE / BE | Doris 的 Frontend（查詢規劃/metadata）與 Backend（儲存/運算）節點。 | 對外文件不使用此術語 |

## 5. 範圍（In-scope / Out-of-scope）

### In-scope
- 兩種服務模式（managed / self-managed）及其責任邊界
- Workspace 生命週期：申請、審核、staging 驗證、佈建、（升降級）、停用
- 多租戶隔離：資料、metadata、運算三層
- Managed 資料管理：zone 模型（raw／curated）、DDL 治理（[ddl-governance.md](ddl-governance.md)）
- Ingestion pipeline（僅 managed）：batch + streaming（三種固定 CDC format：generic / DB2 JSON / DB2 XML）
- Self-managed 營運支援：異常告警、保護性 config、scale 代操作
- 使用者側的配額/用量可視化
- 帳號模型與授權（[access-control.md](access-control.md)、[authorization.md](authorization.md)）
- HA／DR（[ha-dr.md](ha-dr.md)）、資料搬遷（[data-migration.md](data-migration.md)）

### Out-of-scope
- **既有資料遷移**（Oracle/HBase → Doris 由各團隊自理）：平台聚焦服務本體，遷移工具成本高且一次性
- **OLTP 工作負載**：Doris 定位為分析型查詢，不承接交易型場景
- **BI 工具託管**：平台提供查詢介面，BI 工具由使用者自備（如需連線支援僅提供文件）

### 待定
- **跨 workspace 資料分享**：v0.12 提出 owner 可分享 table 讀取權給其他 workspace，方向未定（TBD-31）。

### Future（非本期範圍，設計時預留）
- **Lakehouse → Doris 灌資料通道**：作為正式 ingestion 路徑（非僅 staging 測試用）。batch pipeline 的來源介面應預留擴充空間；staging 的 lakehouse 匯入機制可視為前身。

## 6. 核心功能大綱
> 各 feature 的細節見同資料夾的文件。

1. **Workspace 申請與審核**（[workspace-application.md](workspace-application.md)）——**純人工流程**：使用者依投影片 template 填入需求（服務模式、資料量/成長量/retention、QPS 與效能預期、schema 與 query pattern、成本歸屬），於每週三申請會議報告，平台以公式推算 tier 並審核；無自動化 API（留作 roadmap）。
2. **Staging 驗證**（[staging-validation.md](staging-validation.md)）——申請核准後租借固定小規格的 staging 試用環境、灌測試資料（假資料或自 lakehouse 匯入）、提交設計合理性報告（效能數據僅供參考），通過才開通正式環境。
3. **多租戶隔離**（[workspace-isolation.md](workspace-isolation.md)）——資料、metadata、運算資源（resource group）三層隔離的行為定義。
4. **Cluster 拓撲與 tier**（[cluster-topology-and-tiering.md](cluster-topology-and-tiering.md)）——fab 與 cluster 拓撲、shared/dedicated 判定、超標偵測、升降級流程；self-managed 限定 tier 2+。
5. **Streaming ingestion pipeline（CDC on Kafka，僅 managed）**（[streaming-ingestion.md](streaming-ingestion.md)）——固定三種 message format（generic CDC / DB2 JSON CDC / DB2 XML CDC）；tmp zone 取消後寫入 raw 的流程待改寫（TBD-28）；format 驗證失敗落 MinIO corrupted data 路徑並告警。
6. **Batch ingestion pipeline（僅 managed）**（[batch-ingestion.md](batch-ingestion.md)）——批次匯入的設定、排程、錯誤處理；來源介面預留擴充（future：lakehouse → Doris 通道）。
7. **DDL 治理**（[ddl-governance.md](ddl-governance.md)）——database 由平台建立；managed raw 由平台建立（是否審核：TBD-7）、curated 由 service account 直接建立（TBD-30）；self-managed 一律平台代建（TBD-29）。含開放直接 DDL 的風險與可研究的防護。
8. **Self-managed 營運支援**（[self-managed-operations.md](self-managed-operations.md)）——日常 infra 告警、保護性 config、scale in/out 代操作；其他需求（使用問題/新版本/新功能/建表諮詢/效能優化）走 PM request 排優先權（非義務，目前無付費機制）。
9. **配額與用量可視化**（[quota-and-usage.md](quota-and-usage.md)）——使用者查看自己的配額、用量、查詢效能。
10. **帳號模型與查詢協定**（[access-control.md](access-control.md)）——帳號全由平台管理、資料存取一律透過 service account；MySQL protocol 與 Arrow Flight SQL。
11. **授權**（[authorization.md](authorization.md)）——Authorization PRD v0.12：平台管理 Console 的 user 建立與 grant／revoke、owner 讀權分享、Doris privilege 對照、OPA 長期方向。
12. **HA／DR**（[ha-dr.md](ha-dr.md)）——3 replica 容錯；中長期分散到 3 機櫃或 3 datacenter（TBD-9）；DR 以 backup／restore 到 S3 或 MinIO（TBD-36）。
13. **資料搬遷**（[data-migration.md](data-migration.md)）——為何需要在 cluster 之間搬遷、共通要求、候選做法（TBD-35）。
14. **監控與告警**（[monitoring-alerting.md](monitoring-alerting.md)）——各類告警的現況整理（TBD-33）。
15. **Web Console**（[web-console.md](web-console.md)）——Tool Team 負責的 Admin 後台與使用者功能範圍。
16. **Managed 申請與使用旅程**（[managed-user-journey.md](managed-user-journey.md)）——把申請、staging、開通、寫入、建表、分享、用量、升級串成一條流程。

## 7. 多租戶與資源隔離模型
> Workspace 為租戶單位；資料完全隔離、metadata 互不可見、resource group 隔離運算。Fab 拓撲與 tier 表見 [cluster-topology-and-tiering.md](cluster-topology-and-tiering.md)（門檻草案，TBD-1）；隔離機制見 [workspace-isolation.md](workspace-isolation.md)。
>
> 服務模式 × 審查與責任分工：
> | | Managed | Self-managed |
> |---|---------|--------------|
> | 申請審查 | 決定 shared/dedicated；須申報 ws 整體預期 data size | 僅需管理層成本核准 |
> | 開通內容 | Cluster、raw／curated databases、pipeline、監控 | Cluster + monitor/alert/log |
> | 寫入通道 | 平台 pipeline（唯一） | 一律自寫 |
> | DDL | Database 由平台建立；raw 由平台建立（是否審核：TBD-7）；curated 由 service account 直接建立（TBD-30） | 一律平台代建（驗證＋保護性 property 注入，不驗 schema 設計）；是否因 dbt 開放直接 DDL：TBD-29 |
> | 備份還原/保留清除/效能調校 | 平台承擔 | 使用者自負 |
> | Schema 品質 | Raw：依 TBD-7；curated：使用者自負 | 使用者自負 |
> | 平台支援 | 完整 | 日常 infra 告警 + 代操作（scale in/out）；其他需求走 PM request 排優先權 |
> | 模式轉換 | 不可（需開新 ws 搬資料） | 不可 |

## 8. 非功能需求（階段 3，數字見 TBD-15）
> 要填：可用性、查詢延遲 P95/P99、寫入吞吐、擴容機制、計費模式、備援/DR。
> 已知方向：HA 為 cluster 部署在多台機器、每個 table 3 個 replica；中長期分散到 3 機櫃或 3 datacenter（[ha-dr.md](ha-dr.md)、TBD-9）。DR 未來以 backup／restore 到 S3 或 MinIO（TBD-36）。可用性目標與 RPO/RTO 待量化。
> 使用規範鬆緊：shared cluster 規範較緊（保護鄰居租戶）；dedicated 較寬鬆（僅防 cluster 整體不穩／no response）。

## 9. 治理與合規（階段 3）
> 要填：資料分類與 PII 責任範圍（TBD-6）、RBAC 設計、稽核紀錄、安全審查結論。

## 10. 里程碑與時程
> 要填：目前時程未定、首個目標部門未定（TBD-20）。

## 11. 待決事項與會簽（階段 4）
> 待決事項統一登記在 [00-progress.md 的 TBD 清單](../../00-progress.md#tbd-清單)；會簽結果記到 [decision-history.md](../../decision-history.md)。

- 產品層級：TBD-1（tier 門檻）、TBD-2（超過 tier 3）、TBD-3（計費模式）、TBD-6（PII）、TBD-20（時程）
- 治理與權限：TBD-5（row／column filter）、TBD-7（raw 建表審核）、TBD-8（self-managed DDL 治理）、TBD-29／30（開放直接建表）、TBD-31（跨 workspace 分享）

## 12. 衍生文件索引（階段 5）
- System Architecture：`../architecture/system-architecture.md`
- Data Flow：`../architecture/data-flow.md`
- RFC：`../rfc/`
- 對外文件：`../../external/`

> 對外文件結構要求：按服務模式分別闡明——managed 須完整涵蓋**使用規範、使用限制、責任歸屬**三者；self-managed 重點為**責任歸屬**（規範/限制僅保護性 config 與告警門檻）。`limits-and-quotas.md` 與 `service-spec.md` 以「模式 × 規範/限制/責任」矩陣呈現。
