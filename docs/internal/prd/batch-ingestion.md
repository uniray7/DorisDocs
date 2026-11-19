# Batch Ingestion Pipeline（僅 Managed）

> 來源：[PRD](README.md) § 核心功能大綱｜狀態：草稿
> ⚠️ **tmp zone 已取消（2026-11-19）**：本文件「落 tmp → 使用者 merge SQL → raw」的流程待改寫（TBD-28）。改寫前，涉及 tmp 與 merge SQL 的段落僅供參考。
> ⚠️ **本篇多處為平台預設草案**：batch 的脈絡傾倒尚未進行，以下設計按常見做法起草，標 ⚠️ 處皆待團隊確認（見文末「待確認假設清單」）。

## 概念模型

與 streaming 共用同一套權責分界與 zone 路徑——使用者控制**內容**，平台控制**通道**；平台通道只寫 tmp，進 raw 一律經使用者的 merge SQL：

```
使用者側批次產出（檔案）
   │  上傳/放置到 per-ws landing 路徑（平台 MinIO）⚠️
   ▼
平台 batch loader（排程或手動觸發）
   │  檔案格式驗證 + 逐行解析
   ├─ 整檔不可解析 → 標記 FAILED + 告警（檔案留在 landing 供修正）
   ├─ 行級錯誤 → MinIO corrupted data 路徑（per-ws prefix，與 streaming 共用）
   ▼  解析成功的行
tmp zone（immutable，平台管理 schema）⚠️
   │  使用者定義的 merge SQL（insert...select，平台代管排程執行）
   ▼
raw zone（經 DDL 治理流程核准的 table）
   │  使用者自行 ELT（insert...select）
   ▼
curated zone（免審）
```

- Batch 與 streaming 落地路徑一致（皆經 tmp → merge SQL → raw）的理由：
  1. 寫入 raw 的唯一路徑保持是使用者的 merge SQL，權責分界與 streaming 相同、對外文件講一次就好。
  2. 平台端可用 Doris load label 做檔案級冪等（重跑不重複匯入）。
  3. 未來 lakehouse → Doris 通道可作為新的「來源類型」接入同一條 loader → tmp 路徑，不需要另一套治理。
- ⚠️ 替代方案（若團隊想省 merge 一步）：全量快照型批次允許「直接 load 進 raw + 整分區替換（replace partition）」——需另訂 raw 直寫的權限例外，暫不採用，列入待確認。

## 來源與檔案格式

| 項目 | 本期支援 | 說明 |
|------|---------|------|
| 來源類型 | **Object storage（平台 MinIO）**：per-ws landing prefix，平台配發上傳憑證 ⚠️ | 來源介面抽象為 source type，預留擴充 |
| 檔案格式 | CSV（含自訂分隔符）、Parquet、JSON Lines ⚠️ | Doris load 原生支援的格式為準 |
| Future | Lakehouse → Doris 通道（PRD § Future） | 作為新 source type 接入，staging 的 lakehouse 匯入機制為前身 |

- 檔案命名慣例與 landing 路徑結構（如 `{landing}/{job}/{date}/...`）實作時定；平台以「路徑 + 檔名 pattern」界定一個 job 的輸入範圍。
- 單檔大小上限、單批檔案數上限：TBD-15 配額定案。

## User Story

1. 作為 **managed ws 的資料負責人**，我想在 web console 建立一個 batch job、取得 landing 路徑與上傳憑證，以便把每日批次產出的檔案匯入平台。
2. 作為**資料負責人**，我想設定 job 的觸發方式（排程或手動），以便配合上游批次作業的產出時間。
3. 作為**資料負責人**，我想知道每一批的匯入結果（成功行數、錯誤行數、失敗原因），以便對帳與修正上游資料。
4. 作為**平台維運**，我想要行級錯誤進 corrupted data、整檔錯誤標記失敗且不阻塞後續批次，以便單批髒資料不擴散影響。

## 前端互動流程（Web Console）

### 建立 batch job
1. 選擇目標 ws（需為 managed、ACTIVE）→ 選擇來源類型（本期僅 object storage）→ 選擇檔案格式 → 命名 job。
2. 系統配發：per-ws landing 路徑（`{job}` 子 prefix）、上傳憑證、檔案格式規格說明連結。
3. 目標 tmp table 由平台自動建立（依檔案 schema 宣告，命名 `{ws}__{job}__tmp` ⚠️——與 streaming pipeline 的 tmp 命名同一慣例）。
4. 使用者宣告檔案 schema（欄名/型別/分隔符等），作為 tmp table schema 與解析依據；schema 變更需更新 job 設定（版本紀錄保留）。

### 設定觸發與 merge SQL
1. 觸發方式擇一 ⚠️：
   - **排程**：cron 表達式（如每日 02:00 掃 landing 路徑，有新檔即匯入）；
   - **手動**：console 上點擊觸發（適合不定期批次）；
   - （Future）檔案到達自動觸發——需 MinIO event 整合，本期不做。
2. Merge SQL（tmp → raw）設定同 streaming：僅允許讀 tmp、寫 raw（同 ws）；可設定「load 完成後自動接續執行 merge」或獨立排程。
3. Merge SQL 可隨時更新（版本紀錄保留）。

### 監控頁
- Job 狀態（ENABLED / PAUSED / ERROR）、每批執行紀錄（batch id、檔案清單、成功/錯誤行數、耗時）、corrupted data 計數與樣本預覽、merge job 最近執行結果。

## 行為規格

| # | 情境 | 行為 |
|---|------|------|
| 1 | 檔案解析成功 | 逐行寫入 tmp（append-only）；每檔案一個 Doris load label（`{ws}__{job}__{檔案指紋}`）確保**重跑冪等**——同檔重複觸發不重複匯入 |
| 2 | 行級錯誤（型別不符、欄位數不符） | 錯誤行落 corrupted data 路徑（含原始行 + 失敗原因 + 來源檔名/行號），計數告警；錯誤率超過門檻（預設 ⚠️ 如 10%）整批標 FAILED 不落地，防止半批髒資料進 tmp |
| 3 | 整檔不可解析（損毀、格式不符宣告） | 該檔標 FAILED + 告警，留在 landing 供使用者修正；同批其他檔案照常處理 |
| 4 | 傳遞語義 | 檔案級 exactly-once（load label 冪等）；但上游重複產檔（同內容不同檔名）平台不去重——**merge SQL 仍須冪等**（與 streaming 同一要求，對外文件明示） |
| 5 | 已匯入檔案的處置 | 匯入成功後移至 processed prefix（保留 N 天後清除，N 待 TBD-15）⚠️；landing 殘留檔案計入儲存配額 |
| 6 | Merge SQL 執行失敗 | 告警 + 重試（次數/間隔待定，與 streaming 對齊）；連續失敗 job 標 ERROR，tmp 資料保留不丟 |
| 7 | tmp zone 清理 | 平台依 retention 政策清理已 merge 的 tmp 資料（與 streaming 同一政策，TBD-15） |
| 8 | 配額 | landing + tmp + corrupted data 占用計入 ws 儲存配額；批次匯入吞吐受 shared cluster 寫入側治理（RFC-002 第 4 層：錯峰排程、批次大小治理） |

## 與其他功能的依賴關係
- 目標 raw table 需先經 **ddl-governance** 流程（治理方案 A/B 待決 → 平台系統執行）建立。
- Landing/corrupted data 路徑、tmp table 的 per-ws 隔離依 **workspace-isolation**；寫入側治理參數依 **RFC-002**。
- 指標與配額計入 **quota-and-usage**；tmp/processed 清理政策 TBD-15 定案。
- Staging 版 batch job 行為一致（**staging-validation** 可測此流程）。
- Future lakehouse 通道：以新增 source type 方式擴充本 spec，不另立寫入路徑。

## 邊界審查
未進行（階段 2 邊界審查，依 `edge-case-checklist.md`）。

## 待確認假設清單（脈絡傾倒待補）

> 整體列為 TBD-12；以下為各項細節。

| # | 假設（現行草案） | 待確認問題 |
|---|----------------|-----------|
| 1 | 來源＝檔案放置到平台 MinIO landing 路徑 | 是否也要平台主動去使用者的 DB/儲存拉資料（pull 模式）？還是一律使用者 push 檔案？ |
| 2 | 檔案格式支援 CSV / Parquet / JSON Lines | 實際上游批次產出是什麼格式？是否有內部標準格式（類似 streaming 的三種 CDC format）？ |
| 3 | batch 也走 tmp → merge SQL → raw（與 streaming 一致） | 全量快照型批次是否需要「直寫 raw + replace partition」捷徑？ |
| 4 | 觸發＝使用者自訂 cron + 手動 | 排程是否需要平台統一錯峰控管（shared cluster 寫入側治理）？使用者可自選任意時間嗎？ |
| 5 | 行級錯誤率門檻預設 10%、超過整批 fail | 門檻值與「整批 fail vs 部分落地」的取捨 |
| 6 | 匯入成功檔案移 processed prefix 保留 N 天 | 保留天數、由誰清（平台自動 vs 使用者自理） |

## 驗收標準（Acceptance Criteria）
- [ ] 三種檔案格式各自可建 job、落 tmp；行級錯誤落 corrupted data 且含原始行/檔名/行號
- [ ] 同一檔案重複觸發不重複匯入（load label 冪等驗證）
- [ ] 錯誤率超門檻時整批不落地且告警
- [ ] merge SQL 範圍檢核有效（與 streaming 共用檢核邏輯）
- [ ] landing/tmp/corrupted data 占用正確計入儲存配額
- [ ] 待確認假設清單全數銷案（升級為決議或修改本 spec）
