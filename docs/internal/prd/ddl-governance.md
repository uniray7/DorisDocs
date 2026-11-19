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
- Doris 的 TRUNCATE TABLE 與 DROP TABLE 都需要表級 `DROP_PRIV`，無法分開授權。若要分開管理，可評估由平台依操作類型代為執行，避免直接授予 `DROP_PRIV`。
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
| Database 級 replica quota（`ALTER DATABASE … SET REPLICA QUOTA`） | 限制單一 database 的 replica 總數，替爆炸半徑設上限 | 官方 ALTER DATABASE 文件；預設約 10 億，需自行調低 |
| Partition 數量上限 | 限制單次建立的 partition 數 | `max_dynamic_partition_num` 預設 500；auto partition 的上限待查 |
| `DROP … FORCE` 或縮短回收站保留時間 | 避免舊表在回收站累積 | 官方 Recycle Bin 文件；代價是誤刪無法救回 |
| 限定 dbt 只寫指定的輸出 database（上表方案 B） | 把風險圈在單一 database，搭配 replica quota | — |
| 平台提供 dbt 設定範本、macro（預設 partition、bucket）＋ CI 檢查 | 從源頭擋設定錯誤 | 組織面配套 |
| 監控 FE heap、tablet 數、回收站大小 | 在崩潰前發現 | 接 [monitoring-alerting.md](monitoring-alerting.md) |
| Gateway 限制 DDL 頻率 | 擋失控的 DDL 迴圈 | 推論；需確認 gateway 能否辨識 DDL |

建議先在 4.1.4 實測：replica quota 是否有效擋下超量建表、FE heap 與 tablet 數的關係，再決定是否開放。

## 驗收標準

- [ ] Managed service account 對 raw 無任何 DDL 權限
- [ ] 命名檢核擋下全部違規樣態
- [ ] 稽核紀錄可回溯：申請 → 核准 → 執行，含 DDL 全文與變更前 schema
- [ ] 仍被 pipeline 寫入的 raw 表無法被誤刪
