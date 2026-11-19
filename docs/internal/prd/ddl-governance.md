# DDL 治理

> 來源：[PRD](README.md) § 核心功能大綱｜狀態：草稿｜涵蓋 managed 與 self-managed

## 總覽

| 物件 | Managed | Self-managed |
|------|---------|--------------|
| Database | 平台建立（owner 透過平台申請） | 平台建立；命名方式 → TBD-8 |
| Table | **raw**：平台建表、改表、刪表、清空；建表前是否審核 schema → TBD-7<br>**curated**：service account 直接 CREATE／DROP／TRUNCATE（是否安全 → TBD-30）；ALTER → TBD-22 | 一律提交平台代建（[RFC-003](../rfc/RFC-003-self-managed-ddl-governance-and-rr.md) 前提）；是否因 dbt 開放直接 DDL → TBD-29 |

- DDL 申請的提交管道（人工或 Console）與執行時效 → TBD-32。
- 所有經平台執行的 DDL 都留稽核紀錄：誰申請、誰核准、何時執行、DDL 全文；ALTER／DROP 執行前記錄變更前 schema。

## Managed

### Database 命名

`{workspace}__{user_defined}__{raw|curated}`，規則與範例見 [authorization.md](authorization.md#database-命名)。提交時即檢核：命名合法、總長不超過 Doris 上限、目標 zone 合法、ws 儲存配額足夠。

### Raw table（平台執行）

1. Owner 提交申請：操作類型（create／alter／drop）、目標表、schema、用途、預估資料量。
2. 是否需要平台審核 schema → TBD-7。
3. 平台系統帳號執行，結果通知申請者。

開通時依申請投影片建立的 raw 初始表，同樣回寫一筆 DDL 申請紀錄（來源標記為 workspace 申請），不會「憑空出現」。

| # | 情境 | 行為 |
|---|------|------|
| 1 | 命名違規（含 `__`、超長、zone 不符） | 提交即擋，附具體規則 |
| 2 | 預估量超過 ws 儲存配額餘裕 | 擋下，引導走 tier 升級（[cluster-topology-and-tiering.md](cluster-topology-and-tiering.md)） |
| 3 | 刪除仍被 ingestion pipeline 寫入的 raw 表 | 先解除引用才可執行 |
| 4 | Doris 端執行失敗 | 標 FAILED 並通知；平台處理，不留半完成狀態 |
| 5 | 刪除後的資料處置 | 立即清除或保留冷靜期 → TBD-15 |

### Curated table（service account 直接執行）

- Service account 在 curated database 可直接 CREATE／DROP／TRUNCATE TABLE，計入儲存配額。
- **刪表會連帶刪庫的權限**：Doris 沒有建立者擁有權，建表不會自動授權給建立者。要讓 service account 刪自己建的表，只能授予 database 級 `DROP_PRIV`，而 DROP DATABASE 檢查的正是這個權限，等於也能刪掉整個 curated database。可行方向（待 TBD-30 討論）：接受並靠回收站與稽核兜底；DROP 改由平台執行；或建表後由平台補授表級 `DROP_PRIV`。
- **待 PM 重新評估（TBD-30）**：開放直接建表會帶來下方「開放直接 DDL 的風險」。tier 1 是 shared cluster，FE 由多個 workspace 共用，一個 workspace 出錯會影響同 cluster 的所有租戶，風險高於 self-managed。

## Self-managed

- Database 與 table 的 DDL 目前一律走「提交 → 驗證（語法、危險操作、撞名）→ 平台代建並注入保護性設定」，不審 schema 設計；細節與責任分工見 RFC-003 §2、§5。
- Database 組織方式（固定三庫／命名規則／表單宣告）→ TBD-8（待 manager 決策）。

### dbt 的範圍（若開放直接 DDL）

| 範圍 | 適用情境與取捨 |
|------|----------------|
| A：所有可用 database | 團隊負責所有表的生命週期時較直接，平台工單較少；團隊須承擔誤刪來源表或正式表的影響 |
| B：指定的輸出 database | dbt 讀取來源、在指定 database 重建模型；可保護其他團隊依賴的表，平台須維護 database 用途及跨庫讀權 |

- ALTER、RENAME 及 view 權限依實際模型確認。
- DROP TABLE 需要表級 `DROP_PRIV`。TRUNCATE TABLE 所需權限：4.x 官方文件寫 `DROP_PRIV`，4.1.4 source 檢查的是 `LOAD_PRIV`，待實機驗證（TBD-38）。若以 source 為準，**有寫入權就能 TRUNCATE**，代建流程擋不住。
- 執行身分、允許範圍與 dbt 操作方式 → TBD-23。

## 開放直接 DDL 的風險與可研究的防護

適用於 self-managed 開放 dbt 直接 DDL（TBD-29）與 managed curated 開放建表（TBD-30）。下列「依據」標「推論」者需在 4.1.4 實機驗證。

### 風險

| 風險 | 說明 | 依據 |
|------|------|------|
| FE metadata 膨脹 | FE 把 partition、tablet、replica 的 metadata 放在記憶體；tablet 數約為 partition 數 × bucket 數 × 副本數。partition 過碎或 bucket 過多會讓 tablet 數成倍放大，造成 FE heap 壓力、長時間 Full GC，嚴重時 FE OOM，整座 cluster 的查詢與 DDL 停擺；重啟重播 metadata 也會變慢 | Doris 架構；官方 memory FAQ 指出 tablet 過多多半來自 partition 與 bucket 過多 |
| 回收站累積 | DROP TABLE 預設進回收站保留 1 天（`catalog_trash_expire_second`），期間仍占 metadata 與磁碟。dbt 反覆重建 model（建暫存表 → 寫入 → 替換 → 刪舊表），每小時跑一次就累積 24 份 | 官方 Recycle Bin 文件 |
| DDL 頻率 | 每次 create／drop 都寫入 FE metadata journal，高頻 DDL 加重 journal 與 checkpoint 負擔 | 推論 |
| BE 負擔 | tablet 過多拖慢 compaction、增加 BE 記憶體；小批量高頻寫入使版本數暴增，觸發拒寫（-235） | 官方 memory FAQ；RFC-003 §7 |

### 可研究的防護

| 防護 | 擋什麼 | 依據 |
|------|--------|------|
| 調低 partition、bucket、replica quota 等上限參數 | 限制單表與單庫可建出的 tablet 數 | 見下方「防護參數」；4.1.4 預設值都很寬 |
| `DROP … FORCE` 或縮短回收站保留時間 | 避免舊表在回收站累積 | 官方 Recycle Bin 文件；代價是誤刪無法救回 |
| 限定 dbt 只寫指定的輸出 database（上表方案 B） | 把風險圈在單一 database，搭配 replica quota | — |
| 平台提供 dbt 設定範本、macro（預設 partition、bucket）＋ CI 檢查 | 從源頭擋設定錯誤 | 組織面配套 |
| 監控 FE heap、tablet 數、回收站大小 | 在崩潰前發現 | 接 [monitoring-alerting.md](monitoring-alerting.md) |
| Gateway 限制 DDL 頻率 | 擋失控的 DDL 迴圈 | 推論；需確認 gateway 能否辨識 DDL |

建議先在 4.1.4 實測：replica quota 是否有效擋下超量建表、FE heap 與 tablet 數的關係，再決定是否開放。

### 防護參數（調整值待討論，TBD-40）

下列 FE 參數的 4.1.4 預設值都很寬，平台必須主動調低；**調多少待討論**，下表說明每個參數擋什麼、為什麼要調。

| 參數 | 4.1.4 預設 | 限制什麼 | 為什麼要調 |
|------|-----------|----------|------------|
| `max_auto_partition_num` | 20000 | Auto partition 表最多可有的 partition 數 | 錯誤的寫入或 dbt model 會自動長出大量 partition；每個 partition 乘上 bucket 與 replica 數就是 tablet，直接放大 FE metadata |
| `max_dynamic_partition_num` | 20000 | Dynamic partition 一次可建的 partition 數 | 設錯 start／end 會一次建出大量 partition |
| `max_multi_partition_num` | 4096 | 建表時批次建立 partition 的上限 | 一句 CREATE TABLE 就能建出數千個 partition |
| `max_bucket_num_per_partition` | 768 | 使用者指定的單一 partition bucket 數 | bucket 過多是 tablet 爆炸的另一個來源；一般資料量不需要上百個 bucket（此參數改動需重啟 FE） |
| `autobucket_max_buckets` | 128 | Auto bucket 自動決定的 bucket 上限 | 同上 |
| `default_db_replica_quota_size` | 1073741824 | 新 database 預設的 replica 數上限；也可用 `ALTER DATABASE … SET REPLICA QUOTA` 逐庫設定 | 預設等於沒有限制。這是唯一能限制「單一 database 總 tablet 數」的手段，可擋住每張表都合規、加總卻失控的情況 |
| `catalog_trash_expire_second` | 86400（1 天） | DROP 後在回收站保留多久 | 保留越久越能救回誤刪；但反覆重建的表會讓回收站累積大量 metadata，兩者要取捨 |

**為什麼不能用預設值**：在預設值下，一張 auto partition 表理論上最多 20000 個 partition × 768 個 bucket × 3 個 replica，約 4,600 萬個 tablet replica，遠超過一座 cluster 的 FE 能承受的量。

**決定數值時要考慮**：
- 一座 cluster 的 FE heap 能承受多少 tablet（需在 4.1.4 實測）。
- 正常業務需要的 partition 與 bucket 數（例如按天分區、保留 N 年的表需要幾個 partition）。
- Shared 與 dedicated cluster 可能要不同的值：shared 的 FE 由多個 workspace 共用。
- 這些是 cluster 全域設定，調低會影響該 cluster 所有使用者；超過上限時操作會直接失敗，對外文件要說明上限與失敗時的行為。

## 驗收標準

- [ ] Managed service account 對 raw 無任何 DDL 權限
- [ ] 命名檢核擋下全部違規樣態
- [ ] 稽核紀錄可回溯：申請 → 核准 → 執行，含 DDL 全文與變更前 schema
- [ ] 仍被 pipeline 寫入的 raw 表無法被誤刪

## 參考來源

- [L1] TruncateTableCommand.java（檢查 `LOAD_PRIV`）｜https://raw.githubusercontent.com/apache/doris/4.1.4/fe/fe-core/src/main/java/org/apache/doris/nereids/trees/plans/commands/TruncateTableCommand.java｜4.1.4｜2026-11-19
- [L1] TRUNCATE TABLE 文件（寫 `Drop_priv`）｜https://raw.githubusercontent.com/apache/doris-website/master/versioned_docs/version-4.x/sql-manual/sql-statements/table-and-view/table/TRUNCATE-TABLE.md｜4.x｜2026-11-19
- [L1] DropDatabaseCommand.java（database 級 `DROP_PRIV`）｜https://raw.githubusercontent.com/apache/doris/4.1.4/fe/fe-core/src/main/java/org/apache/doris/nereids/trees/plans/commands/DropDatabaseCommand.java｜4.1.4｜2026-11-19
- [L1] Config.java（partition、bucket、replica quota、回收站參數預設值）｜https://raw.githubusercontent.com/apache/doris/4.1.4/fe/fe-common/src/main/java/org/apache/doris/common/Config.java｜4.1.4｜2026-11-19
- [L1] status.h（`TOO_MANY_VERSION` = -235）｜https://raw.githubusercontent.com/apache/doris/4.1.4/be/src/common/status.h｜4.1.4｜2026-11-19
- [L1] Built-in Authorization 文件（權限層級、無擁有權概念）｜https://raw.githubusercontent.com/apache/doris-website/master/versioned_docs/version-4.x/admin-manual/auth/authorization/internal.md｜4.x｜2026-11-19
