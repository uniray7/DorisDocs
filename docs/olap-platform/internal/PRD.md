# PRD：OLAP Platform（以 Apache Doris 為基礎的內部 OLAP 服務）

> 來源脈絡：[00-progress.md](../00-progress.md) Phase 1 脈絡摘要
> 狀態：大綱版（Phase 2），各章節細節將於後續 Phase 補齊

## 1. 背景與問題陳述
> 要填：Oracle license 成本與綁定問題、HBase 維運負擔、各部門重複自建分析方案的現況。為什麼現在做——sunset 舊 tech stack 的公司方向。

## 2. 目標與非目標
> 要填：
> - 目標：提供多租戶 OLAP 查詢服務 + data ingestion pipeline，承接自 Oracle/HBase 轉移的分析型工作負載。
> - 非目標（已確定）：不負責既有資料遷移；不支援跨 workspace 資料分享；不承接交易型（OLTP）工作負載。

## 3. 目標使用者與使用情境
> 要填：後端工程師（會寫 SQL）為主，多來自 lakehouse 環境。代表情境：(a) 部門申請 workspace 並匯入資料做分析、(b) 以 CDC 接上游 DB 做準即時報表、(c) 大量批次資料的定期分析。

## 4. 術語表
| 名詞 | 定義 | 易混淆處 |
|------|------|----------|
| Workspace (ws) | 租戶單位，對應一個申請部門/團隊。資料、metadata、運算資源的隔離邊界。 | ⚠️ 對應粒度待定：部門 vs 團隊 vs 成本中心 |
| Shared Cluster | 多個 tier 1 workspace 共用的 Doris cluster，以 resource group 隔離運算。 | 資料仍完全隔離，「共用」僅指硬體 |
| Dedicated Cluster | 單一 workspace 專屬的 Doris cluster（tier 2 以上）。 | |
| Tier | Workspace 的資源等級（1/2/3），由 data volume 與 QPS 決定（門檻待 RFC-001 修正）。 | 儲存與運算是否解耦待決 |
| Resource Group | Doris 的運算資源隔離機制，用於 shared cluster 內限制單一 ws 的 CPU/記憶體用量。 | 不是資料隔離機制 |
| Quota | 分配給 workspace 的資源上限（儲存量、QPS、併發數等）。 | ⚠️ 各項計量定義待定（見下方待確認） |
| Data Volume | Workspace 占用的儲存量。 | ⚠️ 計量基準待定：Doris 壓縮後 vs 來源原始大小 |
| Max QPS | Workspace 的查詢速率上限。 | ⚠️ 計量方式待定：瞬時峰值 vs 滑動窗口平均 |
| Ingestion Job | 使用者在平台 pipeline 上設定的一條匯入任務（batch 或 streaming）。 | |
| Batch Ingestion | 平台提供的批次匯入通道。 | |
| Streaming Ingestion | 平台提供的準即時匯入通道，CDC 資料經 Kafka 接入。 | |
| Self-write | 使用者繞過平台 pipeline、自行寫入 Doris 的通道。 | 責任邊界待 RFC-002 |
| FE / BE | Doris 的 Frontend（查詢規劃/metadata）與 Backend（儲存/運算）節點。 | 對外文件不使用此術語 |

## 5. 範圍（In-scope / Out-of-scope）

### In-scope
- Workspace 生命週期：申請、審核、佈建、（升降級）、停用
- 多租戶隔離：資料、metadata、運算三層
- Ingestion pipeline：batch + streaming（CDC on Kafka）
- 自寫入通道的開放與管理（依 RFC-002 決議）
- 使用者側的配額/用量可視化
- Workspace 內帳號與存取控制

### Out-of-scope
- **既有資料遷移**（Oracle/HBase → Doris 由各團隊自理）：平台聚焦服務本體，遷移工具成本高且一次性
- **跨 workspace 資料分享/存取**：與 lakehouse 習慣不同，明確不支援；有共享需求應在 lakehouse 層解決
- **OLTP 工作負載**：Doris 定位為分析型查詢，不承接交易型場景
- **BI 工具託管**：平台提供查詢介面，BI 工具由使用者自備（如需連線支援僅提供文件）

## 6. 核心功能大綱
> Phase 3 逐項展開成 `feature-specs/<slug>.md`，此處放連結。

1. **Workspace 申請與審核**（feature-specs/workspace-application.md）——使用者提交需求（資料量、QPS 預估、成本歸屬），平台審核並佈建。
2. **多租戶隔離**（feature-specs/workspace-isolation.md）——資料、metadata、運算資源（resource group）三層隔離的行為定義。
3. **Cluster tier 分配與升級**（feature-specs/cluster-tiering.md）——shared/dedicated 判定、超標偵測、升降級流程。
4. **Batch ingestion pipeline**（feature-specs/batch-ingestion.md）——批次匯入的設定、排程、錯誤處理。
5. **Streaming ingestion pipeline（CDC on Kafka）**（feature-specs/streaming-ingestion.md）——CDC 接入、schema 對應、延遲與失敗行為。
6. **自寫入（Self-write）通道**（feature-specs/self-write-access.md）——開放條件、責任邊界（依 RFC 決議補齊）。
7. **配額與用量可視化**（feature-specs/quota-and-usage.md）——使用者查看自己的配額、用量、查詢效能。
8. **帳號與存取控制**（feature-specs/access-control.md）——workspace 內的帳號/權限模型；row/column filter 是否納入依 RFC 決議。

## 7. 多租戶與資源隔離模型
> 要填：workspace 為租戶單位；資料完全隔離、metadata 互不可見、resource group 隔離運算。Tier 草案（門檻待 RFC 修正）：
> | Tier | Data Volume | Max QPS | 配置 |
> |------|------------|---------|------|
> | 1 | < 500GB | < 20 | Shared cluster + limited resource |
> | 2 | 500GB – 5TB | 20 – 100 | Small dedicated（3 BEs） |
> | 3 | 5TB – 30TB | 100 – 500 | Medium dedicated（5–10 BEs） |

## 8. 非功能需求（Phase 5 補上）
> 要填：可用性、查詢延遲 P95/P99、寫入吞吐、擴容機制、計費模式、備援/DR。
> 注意：硬體 pool 浮動（視採購狀況），容量以「單位租戶資源模型」描述，不綁定總量。

## 9. 治理與合規（Phase 6 補上）
> 要填：資料分類與 PII 責任範圍（open issue #5）、RBAC 設計、稽核紀錄、安全審查結論。

## 10. 里程碑與時程
> 要填：目前時程未定、首個目標部門未定（Phase 7 會簽時確認）。

## 11. 待決事項與會簽紀錄（Phase 7）
| # | 事項 | 狀態 | 備註 |
|---|------|------|------|
| 1 | Tier 門檻：儲存/運算解耦 | 待討論 | 候選 RFC |
| 2 | 自寫入的資料管理與責任歸屬 | 待討論 | 候選 RFC |
| 3 | Row/column filter 做在平台側與否 | 待討論 | 候選 RFC |
| 4 | 超過 tier 3 規模的處理方式 | 待討論 | |
| 5 | 敏感資料/PII 的平台責任範圍 | 待討論 | |

## 12. 衍生文件索引（Phase 8 完成後補上）
- System Architecture：`system-architecture.md`
- Data Flow：`data-flow.md`
- RFC：`rfc/`
- 對外文件：`../external/`
