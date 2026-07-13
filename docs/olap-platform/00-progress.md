# olap-platform 文件產出進度

## Phase 狀態
| Phase | 狀態 | 完成日期 |
|-------|------|----------|
| 0 定位分流 | 完成 | 2026-07-12 |
| 1 脈絡傾倒 | 完成 | 2026-07-13 |
| 2 骨架生成 | 完成 | 2026-07-13 |
| 2.5 名詞與範圍對齊 | 完成 | 2026-07-13 |
| 3 分塊細化 | 進行中（4/10：workspace-application, staging-validation, workspace-isolation, cluster-tiering） | |
| 4 邊界審查 | 未開始 | |
| 5 非功能需求/SLA | 未開始 | |
| 6 治理與合規審查 | 未開始 | |
| 7 利害關係人會簽 | 未開始 | |
| 8 衍生文件產出 | 未開始 | |
| 9 試點驗證 | 未開始 | |
| 10 發布與版本治理 | 未開始 | |

## Phase 0：分流結論
- 案子範圍：整個 OLAP 平台服務首次推出（以 Apache Doris 為基礎），含 data ingestion pipeline 與服務申請/審核流程。
- 文件策略：內部與對外文件都要產出，**從內部文件（PRD）先做**，對外文件於 Phase 8 從核准後的 PRD 衍生。
- 已知環境限制：服務部署於 private network，文件撰寫過程無法連線驗證，所有服務現況以團隊口述為準。

## Phase 1：脈絡摘要

### 痛點與動機
- 現況：各部門用付費版 Oracle，部分團隊用 HBase。
- 動機：擺脫付費軟體綁定（license 成本），並 sunset 舊有 tech stack。
- 既有資料規模：Oracle + HBase 合計約數百 TB。
- **平台不負責資料遷移**：Oracle/HBase → Doris 的搬遷由各團隊自理，平台只提供 ingestion pipeline 收資料。

### 目標使用者
- 主要為後端工程師，會寫 SQL。
- 使用者多來自 lakehouse 環境，**習慣跨 workspace 存取資料**——但本平台不允許，需在對外文件中前置溝通。

### 多租戶模型
- 租戶單位：**workspace (ws)**，對應部門。
- 隔離要求：資料完全隔離 + resource group 隔離運算資源；workspace 間 metadata 完全不可見（`SHOW DATABASES` / information_schema 看不到別的 ws）。
- **不允許跨 workspace 分享/存取資料**（與 lakehouse 習慣的關鍵差異）。
- Cluster 分配 tier（草案，門檻待修正）：
  | Tier | Data Volume | Max QPS | 配置 |
  |------|------------|---------|------|
  | 1 | < 500GB | < 20 | Shared cluster + limited resource |
  | 2 | 500GB – 5TB | 20 – 100 | Small dedicated（3 BEs，16C/64GB/3.84TB SSD×2） |
  | 3 | 5TB – 30TB | 100 – 500 | Medium dedicated（5–10 BEs，32C/128GB/3.84TB SSD×4） |

### 服務模式（2026-07-13 補充，取代原「自寫入」議題）
平台提供兩種服務模式，責任邊界以模式劃分：

| | Self-managed | Managed |
|---|---|---|
| DDL 自由度 | 自行 create/delete/alter table | create/delete/alter table、database 需跑 approve 流程 |
| 資料寫入 | 可自行灌資料 | 走平台 ingestion pipeline |
| 平台責任 | 僅系統異常告警；scale in/out 可代操作（反映成本） | 平台負資料管理責任 |
| 資料管理責任 | 使用者自負 | 平台承擔 |

- Self-managed 給高自由度使用者；平台責任限縮為異常告警 + 付費代操作。
- Managed 有規範約束（DDL approve 流程），換取平台承擔資料管理責任。

### 申請流程（2026-07-13 補充）
平台有**兩個獨立的申請流程**：
1. **Workspace 申請流程**（所有使用者）——申請開通 workspace，含服務模式選擇、tier 判定、佈建。
2. **Database/Table 申請流程**（僅 managed）——workspace 開通後，managed 使用者對 database/table 的 create/delete/alter 都走此流程：提交申請（schema、用途、預估量）→ 平台審核（schema 品質把關）→ 核准後由平台執行。Self-managed 使用者不適用（自行下 DDL）。

### 申請審核與 staging 驗證細節（2026-07-13，自使用者早期草稿整理）
**申請表單需提供**：
- 使用場景描述
- Table data size、預期資料成長量、資料 retention → 平台提供**公式**讓使用者估算最大資料量
- Peak QPS、預期 query performance、ingestion throughput → 平台以**公式**推算適合的規格（tier）
- Table schema 與 query pattern → 平台審核 schema 正確性

**Staging 驗證流程**（審核通過後、正式開通前；2026-07-13 修正：staging 為**固定小規格試用環境**，機器不足以對齊目標 tier）：
1. 使用者租借平台提供的 staging 試用環境
2. 灌測試資料，兩種方式：(a) 產生假資料；(b) 平台提供機制從 production lakehouse 灌資料（或抽樣）過來測——PII 考量經評估忽略
3. 使用者提交試用報告：schema/查詢設計合理性 + 效能參考數據（**不可外推**，不構成平台效能承諾）
4. 審核通過 → 開單由平台開通正式環境（shared 或 dedicated）；正式效能以上線後監控為準

### Zone 資料管理模型（2026-07-13，自草稿整理；適用範圍待確認）
Cluster 內分三個 zone，對應 Databricks 的 Bronze/Silver/Gold：
- **tmp zone**（≈Bronze）：CDC 原始資料落地，**immutable**
- **raw zone**（≈Silver）：解析/合併後的結構化資料
- **curated zone**（≈Gold）：使用者以 Doris ELT（insert...select）產出的衍生資料表

**Database 命名**（2026-07-13 修訂）：`{ws}__{user_defined}__{tmp|raw|curated}`（雙底線分隔，zone 置尾）
- ⚠️ 已查證 Doris 限制（FeNameFormat）：database 名稱規則 `^[a-zA-Z][a-zA-Z0-9_]*$`，**不允許連字號 `-`**，長度上限 64；分隔符用底線。
- 分隔符為**雙底線 `__`**：ws 名稱禁用 `_` 與 `-`（僅小寫字母+數字）；user_defined 可含單底線、不可含 `__`、不可以底線開頭/結尾——database 名稱仍可無歧義反解析出 ws、user_defined 與 zone。

**平台 pipeline 兩種模式**（皆走平台 Kafka，資料格式 jsonline）：
1. **General CDC 模式**：固定 schema（follow IBM CDC 格式），資料直接寫入 tmp zone（immutable）；使用者再以平台提供的機制（細節待補）用 SQL（insert...select）決定如何 parse/merge，寫入 raw zone。
2. **自定義 schema 模式**：使用者透過 web console 建立 pipeline 並設定 schema；pipeline 套用 schema 驗證，**不合 schema 的資料丟到 S3/MinIO 並告警**；合規資料直接寫入 raw zone。

**Raw zone 建表審核**：raw zone 上 create table 一律經平台審核（避免錯誤設定導致效能不佳歸咎平台、事後還要幫忙 migration）；**審核通過的 raw table 才能被寫入**。

**寫入通道 × cluster 類型**（已對齊，2026-07-13 決議）：
- 寫入通道由**服務模式**決定，與 cluster 類型無關：managed（不論 shared/dedicated）只走平台 pipeline、無直接 load 權限，可用 Doris ELT 產 curated 表；self-managed 一律自建 pipeline 直接寫入。
- 草稿中「dedicated 可自建 pipeline」情境即為 self-managed 模式；資料量超過平台 Kafka 承載或高客製需求的使用者應申請 self-managed。
- Self-managed cluster 平台仍需設定保護性 config 防止 cluster 被打壞，並配告警機制提早預警（屬 self-managed 營運支援範疇）。
- **Zone 模型（含 raw 建表審核）僅適用 managed**；self-managed 不分 zone、不審表。
- Managed 的 DDL 審核精緻化：**raw zone 建表嚴審（人工），curated zone（ELT 產出）免審**但計入儲存配額；tmp zone 由平台管理（CDC 落地，使用者不建表）。

### 對外文件結構要求（2026-07-13）
對外文件需按服務模式分別闡明，深度不同：
- **Managed**：使用規範（rules of use）、使用限制（limits）、責任歸屬（responsibility split）三者皆須明文。
- **Self-managed**：重點闡明**責任歸屬**（平台只負責異常告警與付費代操作，資料管理責任全在使用者）；使用規範/限制相對薄（僅防護 cluster 穩定的保護性 config 與告警門檻）。
- Phase 8 產出對外文件時，`limits-and-quotas.md` 與 `service-spec.md` 應以「模式 × 規範/限制/責任」矩陣呈現，避免使用者搞錯自己適用哪套。

### HA 架構（2026-07-13，自草稿整理）
- Active-standby 架構，以 **CCR（cross-cluster replication）** 同步
- 搭配 failover 機制避免資料遺失

### Ingestion Pipeline
- 平台提供：**batch pipeline** + **streaming pipeline**（CDC on Kafka），**僅限 managed 模式**使用，為 managed 的唯一寫入通道。
- Self-managed 使用者一律自行寫入，不開放使用平台 pipeline（責任邊界最乾淨）。
- **未來 roadmap（2026-07-13 補充）**：可能提供 lakehouse → Doris 的灌資料通道（正式 ingestion 路徑，非僅 staging 測試用）。設計 batch pipeline 時應預留來源擴充空間；staging 的 lakehouse 匯入機制可視為此通道的前身。

### 基礎建設
- 非自建機房（私有雲），服務位於 private network。
- 硬體 pool 為**浮動**，視當下採購狀況而定；初期參考值：512 cores / 1024GB memory / 3.84TB SSD × 16。
- 因硬體浮動，容量規劃（Phase 5）與架構文件（Phase 8）應以「單位租戶資源模型」描述，而非綁定總量數字。

### 時程
- 未定，無已知硬 deadline；第一個目標部門未定。

### 待決事項（Open Issues，候選 RFC 題目）
1. Tier 門檻：data volume 與 QPS 綁在同一 tier 不合理，考慮拆成儲存/運算兩條獨立軸，取較高者定 tier。
2. ~~自寫入使用者的資料管理與責任歸屬~~ → **已由服務模式（self-managed / managed）解決**（2026-07-13），細節待 Phase 3 展開。
3. Row filter / column filter（使用者已提出需求）：做在平台側還是使用者自理，未決。
4. 超過 tier 3（>30TB 或 >500 QPS）的使用者如何處理。
5. 敏感資料/PII 的平台責任範圍尚未明確定義。
6. ~~服務模式衍生問題 (a)(b)(c)~~ → 已決議（見 Decision Log 2026-07-13）；**(d) self-managed 代操作的計費方式仍未決**（Phase 5 處理）。

## Decision Log
| 日期 | 事項 | 決議 | 決策者 |
|------|------|------|--------|
| 2026-07-12 | 文件產出順序 | 先內部 PRD，對外文件由 PRD 衍生 | uniray7 |
| 2026-07-13 | Workspace 粒度 | 團隊自定（團隊/部門皆可），每個 ws 綁定一個成本中心 | uniray7 |
| 2026-07-13 | Data volume 計量 | 審核用來源原始大小估 tier，上線後以 Doris 壓縮後實際占用計配額 | uniray7 |
| 2026-07-13 | Max QPS 計量 | 5 分鐘滑動窗口平均判超標；瞬間爆量由 resource group 併發限制兜底 | uniray7 |
| 2026-07-13 | 服務模式 | 提供 self-managed 與 managed 兩種模式，責任邊界以模式劃分 | uniray7 |
| 2026-07-13 | Self-managed × tier | Self-managed 限定 dedicated cluster（tier 2+），不可用 shared | uniray7 |
| 2026-07-13 | 模式轉換 | 不可事後轉換；要換模式需開新 ws 搬資料 | uniray7 |
| 2026-07-13 | Managed 資料管理責任 | 含備份還原、保留與清除政策、效能調校、schema 品質把關（DDL 審核）四項 | uniray7 |
| 2026-07-13 | Pipeline 開放範圍 | 平台 ingestion pipeline 僅限 managed；self-managed 一律自寫 | uniray7 |
| 2026-07-13 | 寫入通道對齊 | 通道由服務模式決定、與 cluster 類型無關；不採 hybrid（managed+dedicated 不可另自建 pipeline） | uniray7 |
| 2026-07-13 | Zone 模型範圍 | tmp/raw/curated 與 raw 建表審核僅適用 managed；self-managed 不分 zone、不審表 | uniray7 |
| 2026-07-13 | DDL 審核精緻化 | Raw zone 建表人工嚴審；curated（ELT 產出）免審但計儲存配額；tmp 由平台管理 | uniray7 |
| 2026-07-13 | Database 命名 | ~~`{ws}_{tmp\|raw\|curated}_{userdefined}`~~ → 修訂為 `{ws}__{user_defined}__{tmp\|raw\|curated}`（雙底線分隔、zone 置尾）；ws 名稱僅小寫字母+數字；user_defined 可含單底線、不可含 `__` | uniray7 |
| 2026-07-13 | Staging 定位 | 固定小規格試用環境（機器不足以對齊 tier）；效能數據僅供參考不可外推，審核重點為設計合理性 | uniray7 |
| 2026-07-13 | Staging PII | Lakehouse 真實資料進 staging 的 PII 風險經評估忽略 | uniray7 |
| 2026-07-13 | 命名自由度歸屬 | 命名自由僅限 self-managed（跟服務模式走）；managed 不論 shared/dedicated 一律遵循 zone 命名慣例 | uniray7 |

## 試點回饋（Phase 9）


## 發布紀錄（Phase 10）
