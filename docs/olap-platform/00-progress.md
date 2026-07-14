# olap-platform 文件產出進度

## Phase 狀態
| Phase | 狀態 | 完成日期 |
|-------|------|----------|
| 0 定位分流 | 完成 | 2026-07-12 |
| 1 脈絡傾倒 | 完成 | 2026-07-13 |
| 2 骨架生成 | 完成 | 2026-07-13 |
| 2.5 名詞與範圍對齊 | 完成 | 2026-07-13 |
| 3 分塊細化 | 完成（10/10；batch-ingestion 與 access-control 含待確認假設清單，待脈絡傾倒銷案） | 2026-07-13 |
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

- Self-managed 給高自由度使用者；平台責任限縮為日常 infra 告警 + 代操作（scale in/out）。
- **Self-managed 支援邊界（2026-07-14 補充）**：平台常態只負責日常 infra 相關告警。使用上的問題、新版本、新功能、建表諮詢（consultant）、query performance 優化——一律**提 request 給 PM，由 PM 排優先權決定要不要做**（非平台義務，不做即時支援承諾）。
- **無付費機制（2026-07-14 補充）**：平台目前**沒有任何付費機制**，代操作與 PM request 承接皆不收費；文件中「付費」措辭全數撤除，若未來引入計費（待拍板 #1）再議。
- Managed 有規範約束（DDL approve 流程），換取平台承擔資料管理責任。

### 申請流程（2026-07-13 補充）
平台有**兩個獨立的申請流程**：
1. **Workspace 申請流程**（所有使用者）——申請開通 workspace，含服務模式選擇、tier 判定、佈建。**純人工、無自動化 API**（2026-07-13 決議）：平台提供投影片 template，使用者填入必要資訊，於每週三申請會議報告，平台審核後以內部工單執行佈建；狀態以人工追蹤表維護。

**審查流程按服務模式分軌（2026-07-13 補充，取代先前「平台審 schema」的設計）**：
- **Self-managed 審查**：僅需**管理層對成本支出核准**即通過。審查後平台執行：開 cluster + 對應的 monitor、alert、log。開通後更動 database/table **不需經過平台、平台不負任何責任**。
- **Managed 審查**：審查決定開 **shared 或 dedicated cluster**；申請時**必須表明預期整個 workspace 的 data size**。開通後 database/table 的治理**兩案並列，待 PM/管理層決定**（見下方「DDL 治理兩案」）。
2. **Database/Table 申請流程**（僅 managed）——workspace 開通後，managed 使用者對 database/table 的 create/delete/alter 都走此流程。**治理模式兩案並列，待 PM/管理層決定**：
   - **方案 A（owner 治理）**：ws 成員提交申請 → **workspace owner approve**（平台不審核）→ 平台系統自動執行；使用者可主動尋求平台建表建議。Schema 品質責任在使用者。
   - **方案 B（平台治理）**：提交申請（schema、用途、預估量）→ **平台審核**（raw zone 建表嚴審、curated 免審）→ 核准後平台執行。Schema 品質把關屬平台 managed 四大責任之一。
   - Self-managed 皆不適用（自行下 DDL，平台不負責任）。

### 申請審核與 staging 驗證細節（2026-07-13，自使用者早期草稿整理）
**申請表單需提供**：
- 使用場景描述
- Table data size、預期資料成長量、資料 retention → 平台提供**公式**讓使用者估算最大資料量
- Peak QPS、預期 query performance、ingestion throughput → 平台以**公式**推算適合的規格（tier）
- Table schema 與 query pattern → ~~平台審核 schema 正確性~~（2026-07-13 改版：申請時是否平台審核**待 PM 決定**；開通後 DDL 由 ws owner approve、平台不審）

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

**平台 pipeline 兩種模式**（皆走平台 Kafka）：
1. **CDC 模式**：固定 message format，資料直接寫入 tmp zone（immutable）；使用者再以平台提供的機制（細節待補）用 SQL（insert...select）決定如何 parse/merge，寫入 raw zone。
   - **支援三種 message format**（2026-07-13 補充）：
     | Format | 說明 |
     |--------|------|
     | Generic CDC event | JSON，例：`{"eventTime": "2029-10-01T12:34:56Z", "sourceName": "order_service", "operation": "INSERT", "data": "{\"col1\": \"val1\", ...}"}`（data 為字串編碼的 JSON，內容 schema 由使用者自定） |
     | DB2 JSON CDC event | 依內部規格文件（文件不在本 repo，實作時引用） |
     | DB2 XML CDC event | 依內部規格文件（文件不在本 repo，實作時引用） |
2. ~~**自定義 schema 模式**~~（2026-07-13 決議**取消**，被三種固定 format 取代）：generic CDC 的 `data` 欄位可承載自定義內容，所有 streaming 資料統一落 tmp 再由使用者 SQL parse 進 raw。原「驗證失敗落 MinIO 並告警」機制保留，改為 **format 驗證**（非法 JSON/XML、缺必要欄位）失敗的資料稱 **corrupted data**，存放於 MinIO 指定路徑。

**Raw zone 建表治理**（2026-07-13 更新：**兩案並列待 PM/管理層決定**）：
- 方案 A：ws owner approve → 平台系統執行；平台不審、提供選配建議；schema 品質責任在使用者（「效能不佳歸咎平台」風險由責任歸屬條款處理）。
- 方案 B：平台審核（raw 嚴審/curated 免審）→ 平台執行；「避免錯誤設定導致效能不佳歸咎平台、事後幫忙 migration」由審核把關。
- 兩案共同點：**經核准建立的 raw table 才能被 pipeline 寫入**；tmp 由平台管理；建表由平台系統執行（使用者無直接 DDL 權限）。

**寫入通道 × cluster 類型**（已對齊，2026-07-13 決議）：
- 寫入通道由**服務模式**決定，與 cluster 類型無關：managed（不論 shared/dedicated）只走平台 pipeline、無直接 load 權限，可用 Doris ELT 產 curated 表；self-managed 一律自建 pipeline 直接寫入。
- 草稿中「dedicated 可自建 pipeline」情境即為 self-managed 模式；資料量超過平台 Kafka 承載或高客製需求的使用者應申請 self-managed。
- Self-managed cluster 平台仍需設定保護性 config 防止 cluster 被打壞，並配告警機制提早預警（屬 self-managed 營運支援範疇）。
- **Zone 模型（含 raw 建表審核）僅適用 managed**；self-managed 不分 zone、不審表。
- Managed 的 DDL 治理：方案 A（owner approve）/方案 B（平台審核，raw 嚴審/curated 免審）兩案並列待決；無論何案，建表計入儲存配額、tmp zone 由平台管理、平台系統代執行。

### 查詢介面與 Security Control（2026-07-14 補充）
平台提供**兩種查詢協定**：
| 協定 | 支援範圍 |
|------|---------|
| MySQL protocol | 完整 SQL（受權限體系約束），含 metadata SQL |
| Arrow Flight protocol | **僅 SELECT 相關 SQL**；metadata SQL（如 `SHOW DATABASES`）**不支援** |

- 兩協定的權限/安全控管落在 access-control spec；QPS 計量路徑是否涵蓋 Arrow Flight（gateway 為 MySQL protocol proxy）待 RFC-002 驗證。

### 對外文件結構要求（2026-07-13）
對外文件需按服務模式分別闡明，深度不同：
- **Managed**：使用規範（rules of use）、使用限制（limits）、責任歸屬（responsibility split）三者皆須明文。
- **Self-managed**：重點闡明**責任歸屬**（平台只負責日常 infra 告警與代操作，資料管理責任全在使用者；其他需求走 PM request、非義務）；使用規範/限制相對薄（僅防護 cluster 穩定的保護性 config 與告警門檻）。
- Phase 8 產出對外文件時，`limits-and-quotas.md` 與 `service-spec.md` 應以「模式 × 規範/限制/責任」矩陣呈現，避免使用者搞錯自己適用哪套。

### HA 架構（2026-07-13，自草稿整理）
- Active-standby 架構，以 **CCR（cross-cluster replication）** 同步
- 搭配 failover 機制避免資料遺失

### Ingestion Pipeline
- 平台提供：**batch pipeline** + **streaming pipeline**（CDC on Kafka），**僅限 managed 模式**使用，為 managed 的唯一寫入通道。
- Self-managed 使用者一律自行寫入，不開放使用平台 pipeline（責任邊界最乾淨）。
- **未來 roadmap（2026-07-13 補充）**：可能提供 lakehouse → Doris 的灌資料通道（正式 ingestion 路徑，非僅 staging 測試用）。設計 batch pipeline 時應預留來源擴充空間；staging 的 lakehouse 匯入機制可視為此通道的前身。

### 基礎建設
- **Doris 版本：4.1.0**（2026-07-13 確認）——2.1+ 的隔離能力（cpu_hard_limit、掃描 IO 限速、workload schedule policy）皆可用。
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
6. ~~服務模式衍生問題 (a)(b)(c)~~ → 已決議（見 Decision Log 2026-07-13）；~~(d) self-managed 代操作的計費方式~~ → 2026-07-14 決議：目前無任何付費機制、代操作不收費；成本歸屬併入計費模式決策（open issue：計費模式，Phase 5）。
7. **Managed 的 DDL 治理模式**：方案 A（ws owner approve + 平台選配建議）vs 方案 B（平台審核，raw 嚴審/curated 免審）——**待 PM/管理層決定**；申請時投影片 schema 是否平台審核同屬此決策。
8. Shared cluster 資源隔離機制 → **RFC-002 草稿已完成**（建議：多層防線＝workload group 硬限制 + 自建 gateway + kill policy + 寫入側治理），待會簽核准。

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
| 2026-07-13 | Streaming format | 固定三種 message format：generic CDC event、DB2 JSON CDC、DB2 XML CDC（後兩者規格見內部文件） | uniray7 |
| 2026-07-13 | 自定義 schema 模式取消 | 被三種固定 format 取代；所有 streaming 統一落 tmp → 使用者 SQL parse/merge 進 raw；驗證失敗資料稱 corrupted data（MinIO 路徑），觸發條件為 format 驗證失敗 | uniray7 |
| 2026-07-13 | Corrupted data 命名 | Format 驗證失敗的資料統一稱 **corrupted data**（不用 dead-letter），存放於 MinIO 指定路徑（per-ws prefix） | uniray7 |
| 2026-07-13 | 申請流程純人工 | Workspace 申請（含 staging 租借/報告審核）無自動化 API：投影片 template + 每週三申請會議 + 人工追蹤表；API 留作未來 roadmap | uniray7 |
| 2026-07-13 | 審查分軌 | Self-managed：管理層成本核准即通過，平台開 cluster+monitor/alert/log；Managed：審查定 shared/dedicated，須申報 ws 整體預期 data size | uniray7 |
| 2026-07-13 | DDL 治理兩案並列 | Managed 的 database/table 治理**未定案**，兩案並列待 PM/管理層決定——方案 A：ws owner approve、平台不審、選配建議；方案 B：平台審核（raw 嚴審/curated 免審）、schema 把關屬平台責任。Self-managed 不經平台、平台不負責任（此點已定） | uniray7 |
| 2026-07-14 | Self-managed 支援邊界 | 平台常態只負責日常 infra 告警；使用問題/新版本/新功能/建表諮詢/查詢效能優化一律提 request 給 PM，由 PM 排優先權決定是否承接（非平台義務） | uniray7 |
| 2026-07-14 | 無付費機制 | 平台目前沒有任何付費機制，「付費」措辭全數撤除（代操作、諮詢皆不收費）；未來是否引入計費併入計費模式決策（待拍板 #1） | uniray7 |
| 2026-07-14 | 查詢協定 | 提供 MySQL protocol（完整 SQL）與 Arrow Flight protocol（僅 SELECT 類 SQL，不支援 metadata SQL 如 SHOW DATABASES）兩種查詢介面 | uniray7 |

## 試點回饋（Phase 9）


## 發布紀錄（Phase 10）
