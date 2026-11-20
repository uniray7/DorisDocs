# Streaming Ingestion Pipeline（僅 Managed）

> 來源：[PRD](README.md) § 核心功能大綱｜狀態：草稿
> Managed 的資料一律由平台 pipeline 寫入 raw database；self-managed 不使用平台 pipeline，由使用者自行寫入。Message 如何對應到 raw table、UPDATE／DELETE 與去重如何套用 → TBD-28。

## 概念模型

```
使用者的 CDC 工具/producer
   │  固定 format 的 message（三種擇一）
   ▼
平台 Kafka（per-ws topic + ACL）
   │  平台 consumer：format 驗證
   ├─ 驗證失敗 → MinIO corrupted data 路徑（per-ws prefix）+ 告警
   ▼  驗證通過
平台 pipeline 寫入 raw（table 由平台建立，見 ddl-governance）
   │  service account 自行加工（INSERT … SELECT）
   ▼
curated
```

使用者控制**內容**（message 產生）；平台控制**通道**（Kafka、format 驗證、寫入 raw）。

## 支援的 Message Format（三種擇一，per pipeline）

| Format | 規格 |
|--------|------|
| `generic-cdc` | JSON，固定信封欄位 + 自由內容：`{"eventTime": "2029-10-01T12:34:56Z", "sourceName": "order_service", "operation": "INSERT", "data": "{\"col1\": \"val1\", ...}"}`。`operation` ∈ INSERT/UPDATE/DELETE；`data` 為字串編碼 JSON，內容 schema 由使用者自定 |
| `db2-json-cdc` | 依內部規格文件《DB2 JSON CDC event format》（不在本 repo，實作時引用） |
| `db2-xml-cdc` | 依內部規格文件《DB2 XML CDC event format》（不在本 repo，實作時引用） |

- Format 驗證僅檢查**信封層**（合法 JSON/XML、必要欄位存在、eventTime 可解析）；`data` 內容不驗證。內容如何對應 raw table 欄位、對應失敗如何處理 → TBD-28。
- 三種 format 的寫入路徑相同。

## User Story

1. 作為 **managed ws 的資料負責人**，我想在 web console 自助建立一條 streaming pipeline 並取得專屬 Kafka topic 與憑證，以便把上游 DB 的 CDC 接進平台。
2. 作為**資料負責人**，我想讓上游的 INSERT／UPDATE／DELETE 正確反映到 raw table，以便 raw 與來源一致（套用規則由誰定義 → TBD-28）。
3. 作為**資料負責人**，我想看到 pipeline 的 lag、吞吐與 corrupted data 數量並在異常時收到告警，以便及早處理格式錯誤或上游中斷。
4. 作為**平台維運**，我想要 format 錯誤的訊息自動進 corrupted data 路徑 而不阻塞 pipeline，以便單筆髒資料不影響整體匯入。

## 前端互動流程（Web Console）

### 建立 pipeline
1. 選擇目標 ws（需為 managed、ACTIVE）→ 選擇 format（三種擇一）→ 命名 pipeline。
2. 系統配發：專屬 Kafka topic、producer 憑證、連線資訊、format 規格文件連結。
3. 寫入哪些 raw table、message 如何對應 → TBD-28。

### 監控頁
- Pipeline 狀態（RUNNING / PAUSED / ERROR）、consumer lag、吞吐（rows/s、MB/s）、corrupted data 計數與樣本預覽。

## 行為規格

| # | 情境 | 行為 |
|---|------|------|
| 1 | Message 通過 format 驗證 | 由平台 pipeline 寫入 raw；UPDATE／DELETE 的套用方式 → TBD-28 |
| 2 | Message format 驗證失敗 | 整筆落 corrupted data 路徑（含原始 payload + 失敗原因 + offset），計數告警（門檻可設）；pipeline 不中斷 |
| 3 | 傳遞語義 | at-least-once，可能重複消費；去重與冪等如何做 → TBD-28（定案後寫入對外文件） |
| 4 | 順序 | 同 Kafka partition 內有序；跨 partition 不保證——寫入 raw 時依 `eventTime`（或內部序號）判定新舊，不依讀取順序 |
| 5 | 寫入 raw 失敗 | 告警 + 重試（次數/間隔待定）；連續失敗 pipeline 標記 ERROR，不丟資料（保留方式 → TBD-28） |
| 6 | 上游停止/backpressure | Kafka 積壓緩衝；lag 超門檻告警；topic retention 內恢復即不丟資料（retention 時長待 TBD-15 定） |
| 7 | 配額 | corrupted data 占用計入 ws 儲存配額（防止殭屍 pipeline 無限累積） |

## 與其他功能的依賴關係
- 目標 raw table 由平台建立（**ddl-governance**；建議建表前審核 schema，TBD-7）。
- Topic/憑證/corrupted data 路徑的 per-ws 隔離依 **workspace-isolation**。
- 指標與配額計入 **quota-and-usage**。
- Staging 版 pipeline 行為一致（**staging-validation** 的 ingestion 接通驗證即測此流程）。

## 邊界審查
未進行（階段 2 邊界審查，依 `edge-case-checklist.md`）。

## 驗收標準（Acceptance Criteria）
- [ ] 三種 format 各自可建 pipeline、寫入 raw，corrupted data 行為一致
- [ ] format 驗證失敗不中斷 pipeline，corrupted data 含原始 payload 與失敗原因
- [ ] pipeline 只能寫入同一 ws 的 raw table
- [ ] at-least-once 語義與去重方式寫入對外文件
- [ ] lag/corrupted data 告警可設門檻並送達 ws 聯絡人
- [ ] corrupted data 占用正確計入儲存配額
