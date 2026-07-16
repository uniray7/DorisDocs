# PRD：OLAP Platform（以 Apache Doris 為基礎的內部 OLAP 服務）

> 來源脈絡：[00-progress.md](../00-progress.md) Phase 1 脈絡摘要
> 狀態：大綱版（Phase 2），各章節細節將於後續 Phase 補齊

## 1. 背景與問題陳述
> 要填：Oracle license 成本與綁定問題、HBase 維運負擔、各部門重複自建分析方案的現況。為什麼現在做——sunset 舊 tech stack 的公司方向。

## 2. 目標與非目標
> 要填：
> - 目標：提供多租戶 OLAP 查詢服務 + data ingestion pipeline，以 managed / self-managed 兩種模式承接自 Oracle/HBase 轉移的分析型工作負載。
> - 非目標（已確定）：不負責既有資料遷移；不支援跨 workspace 資料分享；不承接交易型（OLTP）工作負載。

## 3. 目標使用者與使用情境
> 要填：後端工程師（會寫 SQL）為主，多來自 lakehouse 環境。代表情境：(a) 部門申請 managed workspace，以平台 CDC pipeline 接上游 DB 做準即時報表、(b) managed 使用者以 Doris ELT 從 raw 產出 curated 分析表、(c) 高客製/大流量團隊申請 self-managed dedicated cluster 自建 pipeline。

## 4. 術語表
| 名詞 | 定義 | 易混淆處 |
|------|------|----------|
| Workspace (ws) | 租戶單位，資料、metadata、運算資源的隔離邊界。申請粒度由團隊自定（團隊或部門皆可），但每個 ws 必須綁定一個成本中心作為成本歸屬單位。 | 一個成本中心可擁有多個 ws；成本報表以成本中心彙總 |
| Shared Cluster | 多個 tier 1 workspace 共用的 Doris cluster，以 resource group 隔離運算。 | 資料仍完全隔離，「共用」僅指硬體 |
| Dedicated Cluster | 單一 workspace 專屬的 Doris cluster（tier 2 以上）。 | |
| Tier | Workspace 的資源等級（1/2/3），由 data volume 與 QPS 決定（門檻待 RFC-001 修正）。 | 儲存與運算是否解耦待決 |
| Resource Group | Doris 的運算資源隔離機制，用於 shared cluster 內限制單一 ws 的 CPU/記憶體用量。 | 不是資料隔離機制 |
| Quota | 分配給 workspace 的資源上限（儲存量、QPS、併發數等）。 | |
| Data Volume | Workspace 占用的儲存量。**申請/審核階段**以來源原始大小估算 tier；**上線後計量**以 Doris 壓縮後實際占用計配額。 | 對外文件需提供壓縮率參考換算例 |
| Max QPS | Workspace 的查詢速率上限，以**滑動窗口平均（5 分鐘）**計量判定超標。 | 瞬間爆量由 resource group 併發限制兜底，不直接判超標 |
| 查詢協定 | 平台提供兩種查詢介面：**MySQL protocol**（完整 SQL，含 metadata SQL）與 **Arrow Flight protocol**（僅 SELECT 相關 SQL，不支援 metadata SQL 如 `SHOW DATABASES`）。 | 共用同一權限體系；細節見 access-control spec |
| Managed 模式 | 平台承擔資料管理責任（備份還原、保留清除、效能調校；schema 品質把關與否依 DDL 治理方案 A/B，待決）的服務模式。寫入一律走平台 pipeline；DDL 需經治理流程核准後由平台系統執行（核准者依方案 A/B：ws owner 或平台）。 | 申請時選定，**不可事後轉換** |
| Self-managed 模式 | 使用者自行管理資料的服務模式：可自行灌資料與執行 DDL。平台責任限縮為日常 infra 告警與代操作（scale in/out，目前無付費機制）；其他需求（使用問題/新版本/新功能/建表諮詢/效能優化）提 request 由 PM 排優先權，非平台義務。**限定 dedicated cluster（tier 2+）**。 | 平台不負資料管理責任；不可用平台 pipeline |
| Ingestion Job | 使用者在平台 pipeline 上設定的一條匯入任務（batch 或 streaming）。**僅 managed 模式可用**。 | |
| Batch Ingestion | 平台提供的批次匯入通道（僅 managed）。 | |
| Streaming Ingestion | 平台提供的準即時匯入通道（僅 managed）：使用者以固定 format 的 message 灌進平台 Kafka，統一落 tmp zone，再以 SQL parse/merge 進 raw。支援三種 format：generic CDC event、DB2 JSON CDC、DB2 XML CDC（後兩者規格見內部文件）。 | Format 驗證失敗落 corrupted data（MinIO 指定路徑）並告警 |
| Database/Table 申請 | Managed 模式下的 DDL 治理流程，**兩案並列待 PM/管理層決定**——方案 A：ws owner approve、平台不審核、選配建表建議；方案 B：平台審核（raw 嚴審/curated 免審）。核准後皆由平台系統執行。 | Self-managed 不適用（自行下 DDL、平台不負責任）；與 workspace 申請是兩條獨立流程 |
| Zone | Managed 模式的資料分層模型：**tmp**（CDC 原始落地，immutable，平台管理）→ **raw**（解析/合併後結構化資料，建表經 DDL 治理流程核准）→ **curated**（使用者 ELT 產出，治理較輕）。對應 Databricks Bronze/Silver/Gold。核准者依 DDL 治理方案 A/B（待決）。 | 僅 managed 適用；self-managed 不分 zone |
| Database 命名 | `{ws}__{user_defined}__{tmp\|raw\|curated}`（雙底線分隔，zone 置尾）；Doris database 名稱不允許 `-`（規則 `^[a-zA-Z][a-zA-Z0-9_]*$`，上限 64 字元）。ws 名稱僅小寫字母+數字；user_defined 可含單底線、不可含 `__`、不可以底線開頭/結尾。 | 僅 managed（self-managed 命名自由） |
| Staging 環境 | 正式開通前的**固定小規格試用環境**（不隨目標 tier 調整）：使用者租借後灌測試資料（自產假資料或自 production lakehouse 匯入）試用，提交設計合理性報告，通過才開通正式環境。 | 效能數據僅供參考、不可外推，不構成平台效能承諾 |
| CCR | Cross-cluster replication，active-standby HA 架構的同步機制，搭配 failover 避免資料遺失。 | |
| FE / BE | Doris 的 Frontend（查詢規劃/metadata）與 Backend（儲存/運算）節點。 | 對外文件不使用此術語 |

## 5. 範圍（In-scope / Out-of-scope）

### In-scope
- 兩種服務模式（managed / self-managed）及其責任邊界
- Workspace 生命週期：申請、審核、staging 驗證、佈建、（升降級）、停用
- 多租戶隔離：資料、metadata、運算三層
- Managed 資料管理：zone 模型（tmp/raw/curated）、database/table 申請流程（治理方案 A/B 待決，平台代執行）
- Ingestion pipeline（僅 managed）：batch + streaming（三種固定 CDC format：generic / DB2 JSON / DB2 XML）
- Self-managed 營運支援：異常告警、保護性 config、scale 代操作
- 使用者側的配額/用量可視化
- Workspace 內帳號與存取控制
- HA（active-standby + CCR + failover）

### Out-of-scope
- **既有資料遷移**（Oracle/HBase → Doris 由各團隊自理）：平台聚焦服務本體，遷移工具成本高且一次性
- **跨 workspace 資料分享/存取**：與 lakehouse 習慣不同，明確不支援；有共享需求應在 lakehouse 層解決
- **OLTP 工作負載**：Doris 定位為分析型查詢，不承接交易型場景
- **BI 工具託管**：平台提供查詢介面，BI 工具由使用者自備（如需連線支援僅提供文件）

### Future（非本期範圍，設計時預留）
- **Lakehouse → Doris 灌資料通道**：作為正式 ingestion 路徑（非僅 staging 測試用）。batch pipeline 的來源介面應預留擴充空間；staging 的 lakehouse 匯入機制可視為前身。

## 6. 核心功能大綱
> Phase 3 逐項展開成 `feature-specs/<slug>.md`，此處放連結。

1. **Workspace 申請與審核**（feature-specs/workspace-application.md）——**純人工流程**：使用者依投影片 template 填入需求（服務模式、資料量/成長量/retention、QPS 與效能預期、schema 與 query pattern、成本歸屬），於每週三申請會議報告，平台以公式推算 tier 並審核；無自動化 API（留作 roadmap）。
2. **Staging 驗證**（feature-specs/staging-validation.md）——申請核准後租借固定小規格的 staging 試用環境、灌測試資料（假資料或自 lakehouse 匯入）、提交設計合理性報告（效能數據僅供參考），通過才開通正式環境。
3. **多租戶隔離**（feature-specs/workspace-isolation.md）——資料、metadata、運算資源（resource group）三層隔離的行為定義。
4. **Cluster tier 分配與升級**（feature-specs/cluster-tiering.md）——shared/dedicated 判定、超標偵測、升降級流程；self-managed 限定 tier 2+。
5. **Streaming ingestion pipeline（CDC on Kafka，僅 managed）**（feature-specs/streaming-ingestion.md）——固定三種 message format（generic CDC / DB2 JSON CDC / DB2 XML CDC）統一落 tmp zone（immutable），使用者以 SQL parse/merge 進 raw；format 驗證失敗落 MinIO corrupted data 路徑並告警。
6. **Batch ingestion pipeline（僅 managed）**（feature-specs/batch-ingestion.md）——批次匯入的設定、排程、錯誤處理；來源介面預留擴充（future：lakehouse → Doris 通道）。
7. **Database/Table 申請流程（僅 managed）**（feature-specs/database-table-application.md）——ws 成員提交 DDL 申請 → 依治理方案核准（**方案 A**：ws owner approve；**方案 B**：平台審核，raw 嚴審/curated 免審——**待 PM/管理層決定**）→ 平台系統自動執行（zone 命名由系統保證）。與 workspace 申請是兩條獨立流程。
8. **Self-managed 營運支援**（feature-specs/self-managed-operations.md）——日常 infra 告警、保護性 config、scale in/out 代操作；其他需求（使用問題/新版本/新功能/建表諮詢/效能優化）走 PM request 排優先權（非義務，目前無付費機制）。
9. **配額與用量可視化**（feature-specs/quota-and-usage.md）——使用者查看自己的配額、用量、查詢效能。
10. **帳號與存取控制**（feature-specs/access-control.md）——workspace 內的帳號/權限模型（兩種模式的權限差異：managed 無直接 load/DDL 權限）；查詢協定（MySQL protocol 完整 SQL + Arrow Flight 僅 SELECT）；row/column filter 是否納入依 RFC 決議。

## 7. 多租戶與資源隔離模型
> 要填：workspace 為租戶單位；資料完全隔離、metadata 互不可見、resource group 隔離運算。Tier 草案（門檻待 RFC 修正）：
> | Tier | Data Volume | Max QPS | 配置 | 可用服務模式 |
> |------|------------|---------|------|--------------|
> | 1 | < 500GB | < 20 | Shared cluster + limited resource | 僅 managed |
> | 2 | 500GB – 5TB | 20 – 100 | Small dedicated（3 BEs） | managed / self-managed |
> | 3 | 5TB – 30TB | 100 – 500 | Medium dedicated（5–10 BEs） | managed / self-managed |
>
> 服務模式 × 審查與責任分工：
> | | Managed | Self-managed |
> |---|---------|--------------|
> | 申請審查 | 決定 shared/dedicated；須申報 ws 整體預期 data size | 僅需管理層成本核准 |
> | 開通內容 | Cluster/zone databases + pipeline + 監控 | Cluster + monitor/alert/log |
> | 寫入通道 | 平台 pipeline（唯一） | 一律自寫 |
> | DDL | 經治理流程核准 → 平台系統代執行（**方案 A**：owner approve、平台不審；**方案 B**：平台審核——待決） | 自由（不經平台，平台不負責任） |
> | 備份還原/保留清除/效能調校 | 平台承擔 | 使用者自負 |
> | Schema 品質 | 依 DDL 治理方案：A＝使用者自負（平台選配建議）；B＝平台把關（待決） | 使用者自負 |
> | 平台支援 | 完整 | 日常 infra 告警 + 代操作（scale in/out）；其他需求走 PM request 排優先權 |
> | 模式轉換 | 不可（需開新 ws 搬資料） | 不可 |

## 8. 非功能需求（Phase 5 補上）
> 要填：可用性、查詢延遲 P95/P99、寫入吞吐、擴容機制、計費模式、備援/DR。
> 注意：硬體 pool 浮動（視採購狀況），容量以「單位租戶資源模型」描述，不綁定總量。
> 已知方向（自草稿；2026-07-14 對 self-managed 釐清）：HA 採 active-standby 雙 cluster 架構，以 CCR **盡量維持同步（best-effort，不承諾零丟失）** + failover 時平台介入協助。RPO/RTO、預設 vs 選配、standby 資源計算待 Phase 5 量化。
> 使用規範鬆緊：shared cluster 規範較緊（保護鄰居租戶）；dedicated 較寬鬆（僅防 cluster 整體不穩／no response）。

## 9. 治理與合規（Phase 6 補上）
> 要填：資料分類與 PII 責任範圍（open issue #5）、RBAC 設計、稽核紀錄、安全審查結論。

## 10. 里程碑與時程
> 要填：目前時程未定、首個目標部門未定（Phase 7 會簽時確認）。

## 11. 待決事項與會簽紀錄（Phase 7）
| # | 事項 | 狀態 | 備註 |
|---|------|------|------|
| 1 | Tier 門檻：儲存/運算解耦 | 待討論 | 候選 RFC |
| 2 | ~~自寫入的資料管理與責任歸屬~~ | 已解決 | 由服務模式劃分（Decision Log 2026-07-13） |
| 3 | Row/column filter 做在平台側與否——需求已具體化（2026-07-14）：self-managed 使用者要求對指定 table 做 row filtering / column masking；含政策漂移責任、managed 是否一併提供 | 待討論 | 候選 RFC |
| 4 | 超過 tier 3 規模的處理方式 | 待討論 | |
| 5 | 敏感資料/PII 的平台責任範圍 | 待討論 | |
| 6 | ~~Self-managed 代操作計費方式~~ → 2026-07-14 決議：目前無任何付費機制、代操作不收費 | 已決議 | 未來計費與否併入計費模式決策（open issue #1 類別，Phase 5） |
| 7 | **Managed 的 DDL/schema 治理模式** | **待 PM/管理層決定** | 方案 A：ws owner approve、平台不審、選配建議（審核成本低，schema 品質責任在使用者）；方案 B：平台審核（raw 嚴審/curated 免審）、schema 把關屬平台四大責任（品質可控，審核成本高）。申請時投影片 schema 是否平台審核，同屬此決策 |
| 8 | **Self-managed 的 DDL 治理形態與資料管理 R&R** | **待 manager 決策（RFC-003）** | 前提已定：**database DDL 代建；table DDL 直接執行**（dbt 相容，2026-07-16 修訂）。選項：1 固定三庫 / 2 強制 zone 命名 / 3 表單宣告制（團隊建議）；子決策：備份/還原暫定不提供（audit log 舉證）。R&R 三案幾乎相同，差異在語義承載方式 |

## 12. 衍生文件索引（Phase 8 完成後補上）
- System Architecture：`system-architecture.md`
- Data Flow：`data-flow.md`
- RFC：`rfc/`
- 對外文件：`../external/`

> 對外文件結構要求（2026-07-13）：按服務模式分別闡明——managed 須完整涵蓋**使用規範、使用限制、責任歸屬**三者；self-managed 重點為**責任歸屬**（規範/限制僅保護性 config 與告警門檻）。`limits-and-quotas.md` 與 `service-spec.md` 以「模式 × 規範/限制/責任」矩陣呈現。
