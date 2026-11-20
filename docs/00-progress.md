# 文件進度與現況

> 更新：2026-11-19｜決議演進見 [decision-history.md](decision-history.md)｜PRD 總覽見 [internal/prd/README.md](internal/prd/README.md)

## 現況摘要

| 主題 | 目前結論 |
|------|----------|
| 定位 | 公司內部多租戶 OLAP 平台，以 Apache Doris 4.1.4 為基礎、部署在 HashiCorp Nomad 1.11.1（評估升級 1.11.3，TBD-37）；取代付費 Oracle 與 HBase。既有資料約數百 TB，**Oracle／HBase → Doris 的遷移由各團隊自理** |
| 使用者 | 後端工程師為主（會寫 SQL），多來自 lakehouse 環境 |
| 租戶與拓撲 | 租戶單位為 workspace，綁定成本中心。Workspace 至少使用兩個 fab，每個 fab 各自提供相同數量、彼此獨立的 Doris 服務，不跨 fab 部署。Tier 1 為 shared cluster，tier 2 起為 dedicated（[cluster-topology-and-tiering.md](internal/prd/cluster-topology-and-tiering.md)） |
| 服務模式 | Managed（平台承擔資料管理責任，寫入只走平台 pipeline）與 self-managed（使用者自建 pipeline、自負資料責任，限 dedicated）；申請時選定、不可轉換。對照表見 [PRD](internal/prd/README.md) |
| 申請與開通 | 人工：投影片 template＋每週三申請會議＋追蹤表。核准後 managed 先過 staging 小規格試用（self-managed 選配），再由平台透過 Web Console 的 Admin 後台開通 |
| 資料寫入 | Managed：平台 streaming（三種固定 CDC format）與 batch pipeline 寫入 raw；tmp zone 已取消，流程待改寫（TBD-28）。Self-managed：自建 pipeline |
| DDL | Database 由平台建立。Managed：raw 由平台建立，curated 由 service account 直接建立。Self-managed：table 一律平台代建（[ddl-governance.md](internal/prd/ddl-governance.md)） |
| 帳號與授權 | Doris user、role、密碼全由平台管理；資料存取一律透過 service account，開通時不建立 service account，由 owner（人）在 Web Console 申請，並在平台 policy 上限內設定權限；查詢協定為 MySQL 與 Arrow Flight SQL（[access-control.md](internal/prd/access-control.md)、[authorization.md](internal/prd/authorization.md)） |
| HA／DR | 每個 table 3 個 replica，單一機器故障可容錯；中長期分散到 3 機櫃或 3 datacenter。DR 未來以 backup／restore 到 S3 或 MinIO（[ha-dr.md](internal/prd/ha-dr.md)） |
| 支援與費用 | Self-managed 平台只負責 infra 告警、保護性 config、scale 代操作、DDL 代建；其他需求提 request 給 PM。目前無任何付費機制 |
| 時程 | 未定；首個目標部門未定（TBD-20） |

## 文件狀態

| 文件 | 階段 | 狀態 | 相關 TBD |
|------|------|------|----------|
| [product-overview.md](product-overview.md) | 管理層摘要 | 草稿 | — |
| [prd/README.md](internal/prd/README.md) | 1 定義 | 草稿 | 1、2、3、6、20 |
| [workspace-application.md](internal/prd/workspace-application.md) | 2 Feature 細化 | 草稿 | 7、26、34 |
| [staging-validation.md](internal/prd/staging-validation.md) | 2 Feature 細化 | 草稿 | 15 |
| [cluster-topology-and-tiering.md](internal/prd/cluster-topology-and-tiering.md) | 2 Feature 細化 | 草稿 | 1、2、4、21、37 |
| [workspace-isolation.md](internal/prd/workspace-isolation.md) | 2 Feature 細化 | 草稿 | 18 |
| [authorization.md](internal/prd/authorization.md) | 2 Feature 細化 | 評審中 | 21–24、26、27、38、39 |
| [access-control.md](internal/prd/access-control.md) | 2 Feature 細化 | 草稿（待再討論） | 10、11、16、22、24、30、38、39 |
| [ddl-governance.md](internal/prd/ddl-governance.md) | 2 Feature 細化 | 草稿（待再討論） | 7、8、22、23、29、30、32、38、40、41 |
| [streaming-ingestion.md](internal/prd/streaming-ingestion.md) | 2 Feature 細化 | 草稿（tmp 流程待改寫） | 28 |
| [batch-ingestion.md](internal/prd/batch-ingestion.md) | 2 Feature 細化 | 草稿（假設待確認） | 12、28 |
| [quota-and-usage.md](internal/prd/quota-and-usage.md) | 2 Feature 細化 | 草稿 | 3、4、15、19、33 |
| [self-managed-operations.md](internal/prd/self-managed-operations.md) | 2 Feature 細化 | 草稿 | 5、8、17 |
| [ha-dr.md](internal/prd/ha-dr.md) | 2 Feature 細化 | 草稿（骨架） | 9、36 |
| [data-migration.md](internal/prd/data-migration.md) | 2 Feature 細化 | 草稿 | 35、36 |
| [monitoring-alerting.md](internal/prd/monitoring-alerting.md) | 2 Feature 細化 | 草稿（骨架） | 15、33 |
| [web-console.md](internal/prd/web-console.md) | 2 Feature 細化 | 草稿（骨架） | 16、28、32、34 |
| [RFC-002](internal/rfc/RFC-002-shared-cluster-resource-isolation.md) | 4 定案 | 評審中（待會簽） | 10 |
| [RFC-003](internal/rfc/RFC-003-self-managed-ddl-governance-and-rr.md)＋[決策簡報](internal/rfc/RFC-003-decision-brief.md) | 4 定案 | 評審中（待 manager） | 8、11、29 |
| [managed-user-journey.md](internal/prd/managed-user-journey.md) | 2 Feature 細化 | 草稿 | 1、7、28、30–35 |

階段 1 已完成；階段 2 的 feature 文件都有草稿，邊界審查尚未進行；階段 3（NFR 與治理）、5（架構與對外文件）尚未開始。

## TBD 清單

> 全 repo 唯一的待決清單，文件內以 `TBD-<編號>` 引用。「被引用」列出提到該 TBD 的文件（不含本頁與 improvement-plan），新增或移除引用時要同步更新。需要管理層拍板的項目，在「主題」欄註明**影響**。

| # | 主題 | 建議負責 | 阻擋 | 出處 | 被引用 | 狀態 |
|---|------|----------|------|------|--------|------|
| TBD-1 | Tier 門檻、儲存與運算解耦（RFC 尚未撰寫）。**影響**：定價與硬體採購規劃 | PM＋工程師 | 無 | cluster-topology-and-tiering | cluster-topology-and-tiering、managed-user-journey、prd/README、product-overview | 未開始 |
| TBD-2 | 超過 tier 3 的需求如何承接。**影響**：決定要為此擴充硬體，或明確拒收 | PM | 無 | cluster-topology-and-tiering | cluster-topology-and-tiering、prd/README、product-overview、workspace-application | 未開始 |
| TBD-3 | 計費模式（chargeback／showback／免費）。**影響**：成本中心綁定的意義、超額處理、代操作成本歸屬；目前無任何付費機制 | PM／管理層 | prd/README NFR 章節 | product-overview | prd/README、product-overview、quota-and-usage、self-managed-operations | 未開始 |
| TBD-4 | 超標但不願升級的處置（限流或擋寫）。**影響**：與計費模式連動，宜與 TBD-3 一併決定 | PM | 無 | quota-and-usage | cluster-topology-and-tiering、product-overview、quota-and-usage | 未開始 |
| TBD-5 | Row／column filter 是否由平台提供（長期方向：OPA）。**影響**：self-managed 使用者已提出需求 | PM＋工程師 | 無 | authorization | decision-history、prd/README、product-overview、self-managed-operations、RFC-003 | 未開始 |
| TBD-6 | PII／機敏資料的平台責任範圍。**影響**：審核流程與合規成本 | PM＋Security | prd/README 治理章節 | prd/README | prd/README、product-overview、staging-validation、workspace-application、RFC-003 | 未開始 |
| TBD-7 | Managed raw table 建表前是否需平台審核 schema。**影響**：審核可控品質但增加平台人力；不審則品質責任在使用者 | PM／管理層 | 無 | ddl-governance | ddl-governance、decision-history、managed-user-journey、prd/README、product-overview、workspace-application | 未開始 |
| TBD-8 | Self-managed database 組織方式（固定三庫／命名規則／表單宣告）。**影響**：團隊建議 Option 3（表單宣告制）；RFC-003 為決策文件 | Manager | 無 | RFC-003 | authorization、ddl-governance、decision-history、prd/README、product-overview、self-managed-operations、workspace-application、workspace-isolation、RFC-003 | 研究中 |
| TBD-9 | HA 中長期做法：分散到 3 機櫃或 3 datacenter；可用性目標、RPO／RTO | PM＋工程師 | 無 | ha-dr | decision-history、ha-dr、prd/README、product-overview | 未開始 |
| TBD-10 | Arrow Flight SQL 能否經 gateway、QPS 怎麼計量 | 工程師 | 無 | RFC-002 | access-control、quota-and-usage、RFC-002 | 未開始 |
| TBD-11 | 使用者帳號不含 `GRANT_PRIV` 的技術驗證 | 工程師 | 無 | access-control | access-control、RFC-003 | 未開始 |
| TBD-12 | Batch ingestion 的 6 項假設（來源、格式、觸發、錯誤門檻等） | PM（訪談使用者）＋工程師 | 無 | batch-ingestion | batch-ingestion | 未開始 |
| TBD-13 | tmp 結構與命名 | — | — | — | — | 已解決：tmp 取消，後續併入 TBD-28 |
| TBD-14 | Merge SQL 排程機制 | — | — | — | — | 併入 TBD-28 |
| TBD-15 | NFR 數字：SLA、各類 retention、配額值、refresh 週期、審核時效、告警門檻 | PM＋工程師 | external 對外文件 | prd/README | access-control、batch-ingestion、cluster-topology-and-tiering、ddl-governance、managed-user-journey、monitoring-alerting、prd/README、quota-and-usage、self-managed-operations、streaming-ingestion、workspace-application、RFC-002、RFC-003 | 未開始 |
| TBD-16 | 平台層身分是否接公司 SSO／LDAP | 工程師 | 無 | access-control | access-control、web-console | 未開始 |
| TBD-17 | 保護性 config 清單、FE 封頂參數 | 工程師 | 無 | self-managed-operations | self-managed-operations、RFC-003 | 未開始 |
| TBD-18 | Metadata 隔離 3 項驗證（information_schema、PROCESSLIST、錯誤訊息） | 工程師 | 無 | workspace-isolation | workspace-isolation | 未開始 |
| TBD-19 | 儲存配額計單副本或全副本 | PM＋工程師 | 無 | quota-and-usage | quota-and-usage | 未開始 |
| TBD-20 | 時程與首個目標部門。**影響**：目前皆未定，越早決定越好 | PM | 無 | prd/README | prd/README、product-overview | 未開始 |
| TBD-21 | 同一 workspace 在同一 fab 能否使用多座 cluster（目前傾向一座） | 主管 | 無 | authorization | authorization、cluster-topology-and-tiering | 未開始 |
| TBD-22 | Managed curated 是否開放 ALTER TABLE | PM＋工程師 | 無 | authorization | access-control、authorization、ddl-governance、decision-history | 未開始 |
| TBD-23 | Self-managed TRUNCATE／DROP 的執行身分與範圍；dbt 範圍 A／B | 主管＋工程師 | 無 | ddl-governance | authorization、ddl-governance | 未開始 |
| TBD-24 | Service account 的申請是否需平台審核、憑證交付方式、workspace 橫跨多個 fab 時帳號如何建立（帳密是否一致）、命名規則、owner 離職或轉調時的交接 | 工程師＋PM | 無 | authorization | access-control、authorization、decision-history、managed-user-journey | 未開始 |
| TBD-26 | 命名白名單細節（user_defined 可否數字開頭等） | 工程師 | 無 | authorization | authorization、decision-history、managed-user-journey、prd/README、workspace-application | 未開始 |
| TBD-27 | OPA 外掛 PoC（4.1.4 相容性、兩種協定一致性、revoke 生效來源） | 工程師 | 無 | authorization | authorization、decision-history | 未開始 |
| TBD-28 | tmp 取消後的 ingestion 流程：CDC／batch 資料如何進 raw、parse／merge 由誰做、排程機制 | 工程師＋PM | streaming／batch 文件改寫 | streaming-ingestion、batch-ingestion | batch-ingestion、decision-history、managed-user-journey、prd/README、product-overview、streaming-ingestion、web-console | 未開始 |
| TBD-29 | Self-managed 開放 dbt 直接 DDL 的風險與防護。**影響**：過多或過碎的 partition 可能造成 FE 記憶體壓力甚至 cluster crash | 工程師＋主管 | 無 | ddl-governance | access-control、authorization、ddl-governance、decision-history、monitoring-alerting、prd/README、product-overview、self-managed-operations、RFC-003 | 研究中 |
| TBD-30 | Managed curated 刪表的執行方式：service account 自行 DROP 需要 database 級 `DROP_PRIV`，會連帶能刪整個 curated database（含無法復原的 `FORCE`）。選項 (a)–(d) 與建議見 ddl-governance。**影響**：選 (a) 可能整庫誤刪且無法復原；選 (b) 增加使用者等待與平台工作量 | PM＋工程師 | 無 | ddl-governance | access-control、authorization、ddl-governance、decision-history、managed-user-journey、prd/README、product-overview | 未開始 |
| TBD-32 | DDL 申請的提交管道（人工或 Console）與執行時效 | PM＋工程師 | 無 | ddl-governance | ddl-governance、managed-user-journey、self-managed-operations、web-console、RFC-003 | 未開始 |
| TBD-33 | 監控堆疊與告警歸屬（含 80% 預警由哪個系統計算） | 工程師 | 無 | monitoring-alerting | managed-user-journey、monitoring-alerting、prd/README | 未開始 |
| TBD-34 | Web Console Admin 後台涵蓋哪些開通動作 | 工程師（Tool Team）＋PM | 無 | web-console | decision-history、managed-user-journey、web-console、workspace-application | 未開始 |
| TBD-35 | 資料搬遷做法、流程、回退；換服務模式時由誰搬 | 工程師＋PM | 無 | data-migration | cluster-topology-and-tiering、data-migration、decision-history、managed-user-journey、prd/README | 未開始 |
| TBD-36 | DR：backup／restore 到 S3 或 MinIO 的範圍、頻率、保留期、還原流程與演練 | 工程師＋PM | 無 | ha-dr | data-migration、decision-history、ha-dr、prd/README、product-overview、self-managed-operations、RFC-003 | 未開始 |
| TBD-37 | Nomad 由 1.11.1 升級到 1.11.3（sticky 相關 bug）：以 L2 release notes 確認是否已修正、評估升級影響 | 工程師 | 無 | cluster-topology-and-tiering | cluster-topology-and-tiering、decision-history | 研究中 |
| TBD-38 | TRUNCATE TABLE 所需權限實機驗證：4.x 官方文件寫 `DROP_PRIV`，4.1.4 source 檢查 `LOAD_PRIV`；若為 LOAD，有寫入權即可清空表 | 工程師 | 無 | ddl-governance | access-control、authorization、ddl-governance | 未開始 |
| TBD-39 | Arrow Flight SQL 在 4.1.4 的支援範圍實機驗證（目前只用過 SELECT；官方 3.x 文件示範可執行 DDL、SHOW、INSERT），以及平台是否要限制只能 SELECT | 工程師 | 無 | access-control | access-control、authorization、decision-history、prd/README | 未開始 |
| TBD-40 | Managed curated 開放 service account 建表（與 self-managed dbt 直接 DDL）的 FE 風險：防護參數的調整值與理由（partition、bucket、replica quota、回收站保留時間），shared 與 dedicated 是否不同。**影響**：shared cluster 上一個 workspace 建表失控會影響同 cluster 所有租戶 | 工程師＋PM | 無 | ddl-governance | access-control、authorization、decision-history、ddl-governance、managed-user-journey、prd/README、product-overview | 未開始 |
| TBD-41 | 回收站相關權限實機驗證：清空回收站（`DROP CATALOG RECYCLE BIN`）文件寫需 `ADMIN_PRIV`，4.1.4 source 找不到權限檢查，若屬實任何帳號都能永久清除回收站中其他 workspace 的資料；RECOVER 文件寫 `ADMIN_PRIV`，source 為 ADMIN、ALTER、CREATE 任一即可 | 工程師 | 無 | ddl-governance | ddl-governance | 未開始 |

## 各文件的細項待確認

> 只影響單一文件的細項（數值、流程細節）留在該文件的「待確認事項」，這裡只放連結；跨文件或需要決策的才登記為 TBD。

| 文件 | 細項 |
|------|------|
| [cluster-topology-and-tiering.md](internal/prd/cluster-topology-and-tiering.md#待確認事項) | 升級由誰觸發、持續超標天數 |
| [staging-validation.md](internal/prd/staging-validation.md#待確認事項) | 租借期限與延長政策 |
| [quota-and-usage.md](internal/prd/quota-and-usage.md#待確認事項) | self-managed 是否納入 tier 計量、用量資料保留時長 |
| [self-managed-operations.md](internal/prd/self-managed-operations.md#待確認事項) | scale 申請流程與緊急擴容、PM request 的提交形式 |
| [batch-ingestion.md](internal/prd/batch-ingestion.md#待確認假設清單脈絡傾倒待補) | 6 項預設假設（整體見 TBD-12） |

## 試點回饋（階段 5）

## 發布紀錄（階段 5）
