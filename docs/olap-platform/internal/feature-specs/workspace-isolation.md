# Feature Spec：多租戶隔離（Workspace Isolation）

> 來源：[PRD.md](../PRD.md) § 6.3
> 狀態：草稿（Phase 3）

## User Story

1. 作為 **workspace 成員**，我只能看到並查詢自己 ws 的資料，以便安心存放部門資料。
2. 作為 **workspace 成員**，我執行 `SHOW DATABASES` 或查 `information_schema` 時完全看不到其他 ws 的存在，以便租戶間互不感知。
3. 作為 **shared cluster 上的租戶**，我的查詢不因鄰居租戶的重載而明顯劣化，以便小規模團隊也能安心用 shared tier。
4. 作為 **平台管理員**，我需要一鍵佈建/回收一個 ws 的完整隔離組（databases + 帳號 + 資源限制 + 授權），以便佈建流程可自動化。

## 隔離模型：三層

### 1. 資料隔離（Data）
- 每個 ws 擁有專屬的 Doris databases。命名自由度跟著**服務模式**走、與 cluster 類型無關：managed（不論 shared 或 dedicated）一律遵循 `{ws}__{user_defined}__{tmp|raw|curated}`（平台自動化與 zone 治理依賴此慣例，且 tier 升級時 database 名不變）；self-managed 命名自由（平台不解析其結構，僅做 cluster 層級告警）。
- 授權以 **Doris GRANT 到 database 層級**實作：ws 帳號僅被授予自己 ws databases 的權限，無任何跨 ws 授權路徑。
- **跨 ws 分享明確不支援**：平台不提供任何跨 ws GRANT 的申請管道；有共享需求應在 lakehouse 層解決（對外文件需前置溝通，使用者多來自習慣跨 ws 的 lakehouse）。
- Managed 附屬資源同樣按 ws 隔離：Kafka topic（per-ws + ACL）、format 驗證失敗的 corrupted data MinIO 路徑（per-ws prefix）。

### 2. Metadata 隔離
- `SHOW DATABASES`、`information_schema`、`SHOW PROCESSLIST` 等不得洩漏其他 ws 的 database 名、table 名、查詢內容。
- Doris 行為基礎：`SHOW DATABASES` 僅列出使用者有權限的 database（MySQL 相容行為）——以「不授權即不可見」達成 metadata 隔離。
- ⚠️ 需驗證項（實作時逐一確認，文件先列為驗收條件）：
  - `information_schema.tables` / `columns` 是否確實只回傳有權限的物件
  - `SHOW PROCESSLIST` / query audit 是否會露出他人 SQL（若會，需限制一般帳號使用）
  - 錯誤訊息是否可能洩漏他 ws 物件存在性（例如查詢不存在 vs 無權限的錯誤是否可區分）

### 3. 運算資源隔離（Compute）

> 完整選型分析與參數配置見 [RFC-002：Shared Cluster 資源隔離機制](../rfc/RFC-002-shared-cluster-resource-isolation.md)。環境為 Doris 4.1.0，下列能力皆可用（實作時以 4.x 官方文件為準逐項驗證）。

- **Dedicated cluster**（tier 2/3）：cluster 即隔離邊界；cluster 內不再細分（self-managed 使用者可自行運用 workload group）。
- **Shared cluster**（tier 1，僅 managed）：採**多層防線**，單一機制皆不足以保證 P99：

| 層 | 機制 | 說明 |
|----|------|------|
| 1. 查詢准入 | Workload Group `max_concurrency` + `max_queue_size` + `queue_timeout`（per-ws） | 超過併發先排隊再拒絕；瞬間爆量的兜底（對齊 Max QPS 決議） |
| 1. 查詢准入 | **Gateway/proxy 層**（平台自建） | per-ws QPS 計量（5 分鐘滑動窗口）與限流——Doris 無原生 QPS 配額；順帶統一連線端點（tier 遷移不換線） |
| 2. 執行資源 | Workload Group `cpu_hard_limit`（硬限制，不用 cpu_share 軟限制） | 軟限制在鄰居爆量時保不住 P99 |
| 2. 執行資源 | Workload Group `memory_limit` + `enable_memory_overcommit=false` + spill to disk | 大查詢落盤而非 OOM 拖垮 BE |
| 2. 執行資源 | 掃描 IO 限速（`read_bytes_per_second`）+ scan 線程數限制 | 防止單租戶吃光磁碟頻寬——最易被忽略的一層 |
| 3. 防呆兜底 | Workload Schedule Policy：自動 kill 超時/掃描量超標查詢 | Runaway query 自動處決 |
| 3. 防呆兜底 | SQL Block Rule：限制單查詢可掃 partition/tablet 數 | 強迫帶分區條件；規則寫入對外使用規範 |
| 4. 寫入側 | 平台 pipeline 統一控制：per-ws 消費限速、merge SQL 錯峰排程、批次大小治理 | Shared 只有 managed、寫入全走 pipeline——壓力源頭在平台手上；同時治理 compaction 壓力（高頻小批量寫入的隱形殺手） |
| 5. 實體隔離（選配） | Resource tag 將 ws 資料副本釘到指定 BE 群組 | 隔離最強（page cache/compaction 都分開）但每群組至少 3 BEs，pool 利用率大降；保留給 shared 內大租戶或 shared→dedicated 中繼形態，tier 1 預設不用 |

- **已知無法完全隔離的共用點**（誠實揭露）：FE（metadata/連線管理）、page cache、compaction 線程池。故 tier 1 規範保守（shared 規範較緊原則），並保留升級 dedicated 的明確路徑。

## 佈建原語（供 workspace-application 佈建流程呼叫）

| 動作 | 內容 |
|------|------|
| `provision_ws(ws, mode, tier)` | 建立 databases（managed）、ws 管理帳號、workload group（shared）、GRANT 授權、Kafka topic + ACL（managed）、corrupted data prefix（managed） |
| `suspend_ws(ws)` | 撤銷登入/查詢權限，保留資料（違規/閒置；**目前無付費機制，不含欠費**，2026-07-14 決議） |
| `decommission_ws(ws)` | 回收帳號 → 資料清除（含 Kafka topic、corrupted data）→ 釋放資源；清除有紀錄可稽核 |

## 行為規格（驗收時逐條測試）

| # | 情境 | 預期行為 |
|---|------|----------|
| 1 | ws A 帳號 `SHOW DATABASES` | 只見 A 的 databases（+ information_schema） |
| 2 | ws A 帳號查詢 ws B 的 table（全名指定） | 權限錯誤，且錯誤訊息不確認該 table 是否存在 |
| 3 | ws A 帳號查 `information_schema.tables` | 不含任何 ws B 物件 |
| 4 | shared cluster 上 ws B 跑重查詢 | ws A 的查詢延遲劣化在 workload group 保障範圍內 |
| 5 | ws A 達查詢併發上限 | A 的新查詢被拒/排隊，不影響 B |
| 6 | ws 被 suspend | 既有連線終止、新連線拒絕、資料保留 |
| 7 | ws 被 decommission | 資料/topic/帳號全清除且有稽核紀錄 |
| 8 | managed ws A 嘗試寫入（stream load / insert into raw） | 除平台 pipeline 帳號與 ELT（curated）外無寫入權限 |

## 與其他功能的依賴關係
- 佈建原語被 **workspace-application**（正式與 staging 佈建）呼叫。
- Workload group 參數由 **cluster-tiering** 的 tier 規格決定。
- 帳號模型細節（人員帳號 vs 服務帳號、權限矩陣）在 **access-control** 展開；本 spec 只定隔離邊界。
- Managed 寫入路徑限制與 **streaming/batch-ingestion**、**database-table-application** 銜接（僅平台 pipeline 帳號可寫 raw；使用者 ELT 只可寫 curated）。

## 邊界審查（Phase 4 補上）
| Edge Case | 是否適用 | 處理方式 |
|-----------|---------|----------|
| （待 Phase 4） | | |

## 驗收標準（Acceptance Criteria）
- [ ] 行為規格表 8 條全數通過（含三項 ⚠️ metadata 洩漏驗證）
- [ ] 佈建/回收原語冪等（重複執行不產生殘留或報錯）
- [ ] decommission 後以平台管理員權限確認無資料殘留，且清除紀錄可稽核
- [ ] 對外文件明確傳達「不支援跨 ws 分享」與 lakehouse 的差異
