# Feature Spec：Streaming Ingestion Pipeline（僅 Managed）

> 來源：[PRD.md](../PRD.md) § 6.5
> 狀態：草稿（Phase 3）

## 概念模型

```
使用者的 CDC 工具/producer
   │  固定 format 的 message（三種擇一）
   ▼
平台 Kafka（per-ws topic + ACL）
   │  平台 consumer：format 驗證
   ├─ 驗證失敗 → MinIO corrupted data 路徑（per-ws prefix）+ 告警
   ▼  驗證通過
tmp zone（immutable，平台管理 schema）
   │  使用者定義的 merge SQL（insert...select，平台代管排程執行）
   ▼
raw zone（審核過的 table）
   │  使用者自行 ELT（insert...select）
   ▼
curated zone（免審）
```

使用者控制**內容**（message 產生、merge SQL 邏輯）；平台控制**通道**（Kafka、format 驗證、tmp 落地、排程執行）。

## 支援的 Message Format（三種擇一，per pipeline）

| Format | 規格 | tmp zone 對應 |
|--------|------|---------------|
| `generic-cdc` | JSON，固定信封欄位 + 自由內容：`{"eventTime": "2029-10-01T12:34:56Z", "sourceName": "order_service", "operation": "INSERT", "data": "{\"col1\": \"val1\", ...}"}`。`operation` ∈ INSERT/UPDATE/DELETE；`data` 為字串編碼 JSON，內容 schema 由使用者自定 | 信封欄位展開為 columns，`data` 存為 STRING/JSON 型別 |
| `db2-json-cdc` | 依內部規格文件《DB2 JSON CDC event format》（不在本 repo，實作時引用） | 依內部規格定義 |
| `db2-xml-cdc` | 依內部規格文件《DB2 XML CDC event format》（不在本 repo，實作時引用） | 原始 XML 存 STRING，關鍵欄位（eventTime/operation 等）由平台 parser 抽出為 columns |

- Format 驗證僅檢查**信封層**（合法 JSON/XML、必要欄位存在、eventTime 可解析）；`data` 內容不驗證——內容的正確性由使用者的 merge SQL 負責（權責分界）。
- 三種 format 落地路徑完全一致（Decision Log 2026-07-13）；原「自定義 schema 模式」已取消。

## User Story

1. 作為 **managed ws 的資料負責人**，我想在 web console 自助建立一條 streaming pipeline 並取得專屬 Kafka topic 與憑證，以便把上游 DB 的 CDC 接進平台。
2. 作為**資料負責人**，我想用 SQL 自行定義 tmp → raw 的 parse/merge 邏輯，以便控制去重、型別轉換與 UPDATE/DELETE 的合併方式。
3. 作為**資料負責人**，我想看到 pipeline 的 lag、吞吐與 corrupted data 數量並在異常時收到告警，以便及早處理格式錯誤或上游中斷。
4. 作為**平台維運**，我想要 format 錯誤的訊息自動進 corrupted data 路徑 而不阻塞 pipeline，以便單筆髒資料不影響整體匯入。

## 前端互動流程（Web Console）

### 建立 pipeline
1. 選擇目標 ws（需為 managed、ACTIVE）→ 選擇 format（三種擇一）→ 命名 pipeline。
2. 系統配發：專屬 Kafka topic、producer 憑證、連線資訊、format 規格文件連結。
3. 目標 tmp table 由平台自動建立（依 format 的固定 schema，命名 `{ws}__{pipeline}__tmp`）。

### 設定 merge SQL（tmp → raw）
1. 使用者提交 merge SQL（`INSERT INTO {raw_table} SELECT ... FROM {tmp_table} WHERE ...`）；目標 raw table 必須是已審核通過的表。
2. 平台檢核：SQL 僅允許讀 tmp、寫 raw（同 ws）；禁止其他寫入目標。
3. 設定執行排程（週期，如每 5 分鐘；細節機制**待補**——平台代管排程器 vs Doris JOB，實作時定案）。
4. Merge SQL 可隨時更新（版本紀錄保留，供除錯回溯）。

### 監控頁
- Pipeline 狀態（RUNNING / PAUSED / ERROR）、consumer lag、吞吐（rows/s、MB/s）、corrupted data 計數與樣本預覽、merge job 最近執行結果。

## 行為規格

| # | 情境 | 行為 |
|---|------|------|
| 1 | Message 通過 format 驗證 | 寫入 tmp（append-only；tmp 為 immutable，不因 UPDATE/DELETE operation 而改寫既有列） |
| 2 | Message format 驗證失敗 | 整筆落 corrupted data 路徑（含原始 payload + 失敗原因 + offset），計數告警（門檻可設）；pipeline 不中斷 |
| 3 | 傳遞語義 | at-least-once：tmp 可能出現重複事件；**merge SQL 必須自行去重/冪等**（對外文件明示，並提供 Unique Key 表 + 依 eventTime 取最新的範例） |
| 4 | 順序 | 同 Kafka partition 內有序；跨 partition 不保證——merge 依 `eventTime`（或內部序號）判定新舊，不依讀取順序 |
| 5 | Merge SQL 執行失敗 | 告警 + 重試（次數/間隔待定）；連續失敗 pipeline 標記 ERROR，tmp 持續累積不丟資料 |
| 6 | 上游停止/backpressure | Kafka 積壓緩衝；lag 超門檻告警；topic retention 內恢復即不丟資料（retention 時長待 Phase 5 定） |
| 7 | tmp zone 清理 | 平台依 retention 政策清理已 merge 的 tmp 資料（屬 managed 資料管理責任；政策細節 Phase 5/6） |
| 8 | 配額 | tmp + corrupted data 占用計入 ws 儲存配額（防止殭屍 pipeline 無限累積） |

## API 邏輯

| Method | Path | 說明 |
|--------|------|------|
| POST | `/api/v1/workspaces/{ws}/pipelines` | 建立 pipeline（format、名稱）→ 回 topic/憑證 |
| GET | `/api/v1/workspaces/{ws}/pipelines` | 列出 pipelines 與狀態 |
| PATCH | `/api/v1/workspaces/{ws}/pipelines/{id}` | 暫停/恢復/更名 |
| DELETE | `/api/v1/workspaces/{ws}/pipelines/{id}` | 停用（topic 回收、tmp 保留至 retention 到期） |
| PUT | `/api/v1/workspaces/{ws}/pipelines/{id}/merge-sql` | 提交/更新 merge SQL（含排程） |
| GET | `/api/v1/workspaces/{ws}/pipelines/{id}/metrics` | lag/吞吐/corrupted data 指標 |
| GET | `/api/v1/workspaces/{ws}/pipelines/{id}/corrupted-data` | 瀏覽 corrupted data 樣本 |

### 錯誤碼
| HTTP | 錯誤碼 | 情境 |
|------|--------|------|
| 422 | `WS_NOT_MANAGED` | self-managed ws 嘗試建 pipeline |
| 422 | `RAW_TABLE_NOT_APPROVED` | merge SQL 目標不是審核通過的 raw table |
| 422 | `MERGE_SQL_SCOPE_VIOLATION` | merge SQL 讀寫範圍超出「讀 tmp 寫 raw（同 ws）」 |
| 409 | `PIPELINE_LIMIT_REACHED` | 超過 ws 的 pipeline 數量上限（上限值 Phase 5） |

## 與其他功能的依賴關係
- 目標 raw table 需先經 **database-table-application** 審核。
- Topic/憑證/corrupted data 路徑的 per-ws 隔離依 **workspace-isolation**。
- 指標與配額計入 **quota-and-usage**；tmp 清理政策屬 managed 資料管理責任（Phase 5/6 定案）。
- Staging 版 pipeline 行為一致（**staging-validation** 的 ingestion 接通驗證即測此流程）。

## 邊界審查（Phase 4 補上）
| Edge Case | 是否適用 | 處理方式 |
|-----------|---------|----------|
| （待 Phase 4） | | |

## 驗收標準（Acceptance Criteria）
- [ ] 三種 format 各自可建 pipeline、落 tmp、corrupted data 行為一致
- [ ] format 驗證失敗不中斷 pipeline，corrupted data 含原始 payload 與失敗原因
- [ ] merge SQL 範圍檢核有效（無法寫入 tmp/curated 以外 ws 或未審核表）
- [ ] at-least-once 語義與冪等要求寫入對外文件，附 Unique Key 去重範例
- [ ] lag/corrupted data 告警可設門檻並送達 ws 聯絡人
- [ ] tmp/corrupted data 占用正確計入儲存配額
