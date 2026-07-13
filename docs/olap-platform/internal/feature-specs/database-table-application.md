# Feature Spec：Database/Table 申請流程（僅 Managed）

> 來源：[PRD.md](../PRD.md) § 6.7
> 狀態：草稿（Phase 3）
> ⚠️ **治理模式兩案並列，待 PM/管理層決定**（PRD open issue #7）。本 spec 將兩案的差異集中在「核准段」，提交段與執行段兩案完全相同——定案後只需刪除落選案。

## 定位

- Managed workspace 開通後，database/table 的 **create / alter / delete** 一律走本流程：使用者**沒有直接 DDL 權限**，核准後由**平台系統自動執行**（zone 命名與 Doris 命名規則由系統保證）。
- 與 workspace 申請是**兩條獨立流程**：workspace 申請純人工（投影片+週三會議）；本流程為**線上自助**（web console），因為它是日常高頻操作，人工會議承載不了。
- Self-managed 不適用：自行下 DDL、平台不負責任。

## 各 zone 的治理強度

| Zone | create/alter/delete | 說明 |
|------|--------------------|------|
| tmp | **不開放申請**——由平台隨 pipeline/job 建立與管理 | 使用者無 tmp DDL 入口 |
| raw | 走本流程（核准段依方案 A/B） | pipeline merge SQL 只能寫入經本流程核准的 raw table |
| curated | **建表免審**（兩案皆同）：使用者 ELT 產出，提交即執行；僅做命名/配額檢核 | 計入儲存配額；alter/delete 同樣提交即執行 ⚠️（是否也要 owner approve 待確認） |
| database 層 | 新增 database（`{ws}__{user_defined}__{zone}` 三件組）走本流程，核准段同 raw | 刪 database 屬高風險操作，兩案皆需 ws owner approve + 平台冷靜期 ⚠️ |

## 流程

### 1. 提交（兩案相同）
ws 成員在 web console 提交申請，內容：
- 操作類型（create / alter / delete）與目標物件（database 或 table、zone）
- Table schema（create/alter：完整 DDL 或表單式欄位定義；系統產生最終 DDL）
- 用途說明、預估資料量與成長（raw 建表必填，供容量與選配評估）
- 系統即時檢核（提交前擋下）：命名符合 `{ws}__{user_defined}__{zone}` 且總長 ≤64、user_defined 不含 `__`；目標 zone 合法（tmp 不可申請）；ws 儲存配額餘裕。

### 2. 核准（兩案分歧點）

| | 方案 A：Owner 治理 | 方案 B：平台治理 |
|---|---|---|
| raw 建表/alter | **ws owner approve**（平台不審核） | **平台審核（嚴審）**：schema 設計、分區/分桶、Key 模型、預估量合理性 |
| curated | 免審（提交即執行） | 免審（提交即執行） |
| delete（table/database） | ws owner approve | ws owner approve + 平台確認（資料管理責任方） |
| 平台角色 | 被動：使用者可主動尋求建表建議（選配服務） | 主動把關：schema 品質屬 managed 四大責任之一 |
| 品質責任 | 使用者（責任歸屬條款處理「效能不佳歸咎平台」風險） | 平台 |
| 核准時效 | owner 即時操作，分鐘級 | 平台工作日內回覆（SLA 待 Phase 5）⚠️ |

### 3. 執行（兩案相同）
- 核准後平台系統**自動執行** DDL：使用者全程無直接 DDL 權限。
- 執行結果（成功/失敗+原因）通知申請者與 owner；所有申請與執行留存稽核紀錄（誰申請、誰核准、何時執行、DDL 全文）。
- alter/delete 執行前系統自動記錄變更前 schema（供回溯；資料本身的備份依 managed 備份政策）。

## 行為規格

| # | 情境 | 行為 |
|---|------|------|
| 1 | 建 raw table 且命名合法 | 依方案 A/B 進核准 → 核准後系統執行 → pipeline merge SQL 即可指向此表 |
| 2 | 命名違規（含 `__`、超長、zone 不符） | 提交即擋，附具體規則說明 |
| 3 | 對 tmp zone 提出 DDL 申請 | 拒絕：tmp 由平台管理 |
| 4 | 建 curated table | 免審即執行；計入儲存配額 |
| 5 | delete table（raw/curated） | 需 owner approve（方案 B 另加平台確認）；執行前快照 schema、依保留政策處理資料 ⚠️（立即清除 vs 軟刪除冷靜期，Phase 5/6 定） |
| 6 | 申請時 ws 儲存配額不足以容納預估量 | 提交擋下，引導走 tier 升級流程（cluster-tiering） |
| 7 | 被 pipeline merge SQL 引用中的 raw table 遭 delete 申請 | 系統警示引用中的 pipeline/job，需先解除引用才可核准 |
| 8 | DDL 執行失敗（Doris 端錯誤） | 申請標 FAILED + 通知；平台介入處理（managed 責任），不留半完成狀態 |

## API 邏輯

| Method | Path | 說明 |
|--------|------|------|
| POST | `/api/v1/workspaces/{ws}/ddl-requests` | 提交申請（操作類型、目標、schema、用途、預估量） |
| GET | `/api/v1/workspaces/{ws}/ddl-requests` | 列出申請與狀態（PENDING_APPROVAL / APPROVED / EXECUTED / REJECTED / FAILED） |
| POST | `/api/v1/workspaces/{ws}/ddl-requests/{id}/approve` | 核准（方案 A：owner 權限；方案 B：平台審核介面另計） |
| POST | `/api/v1/workspaces/{ws}/ddl-requests/{id}/reject` | 退回（附原因） |
| GET | `/api/v1/workspaces/{ws}/schemas` | 檢視 ws 現有 database/table 與 schema 版本歷史 |

### 錯誤碼
| HTTP | 錯誤碼 | 情境 |
|------|--------|------|
| 422 | `WS_NOT_MANAGED` | self-managed ws 呼叫本流程 |
| 422 | `NAMING_RULE_VIOLATION` | 命名不符慣例或 Doris 限制 |
| 422 | `TMP_ZONE_NOT_ALLOWED` | 對 tmp 提 DDL |
| 422 | `QUOTA_INSUFFICIENT` | 預估量超出配額餘裕 |
| 409 | `TABLE_IN_USE_BY_PIPELINE` | delete 目標仍被 pipeline 引用 |

## 與其他功能的依賴關係
- **workspace-application**：申請投影片附的 schema 即 raw 初始建表（核准來源依方案 A/B）；開通後增修表走本流程。
- **streaming/batch-ingestion**：merge SQL 目標必須是本流程核准的 raw table（`RAW_TABLE_NOT_APPROVED` 檢核的資料來源）。
- **workspace-isolation**：DDL 執行帳號為平台系統帳號；使用者帳號無 DDL 權限（access-control 落實）。
- **quota-and-usage**：建表預估量與實際占用的配額檢核/計量。

## 待確認事項
1. **方案 A vs B**（PM/管理層，open issue #7）——本 spec 定案後刪落選案。
2. curated 的 alter/delete 是否完全免審，或 delete 至少要 owner approve。
3. delete 的資料處置：立即清除 vs 軟刪除冷靜期（天數）。
4. 方案 B 的平台審核 SLA（Phase 5）。

## 邊界審查（Phase 4 補上）
| Edge Case | 是否適用 | 處理方式 |
|-----------|---------|----------|
| （待 Phase 4） | | |

## 驗收標準（Acceptance Criteria）
- [ ] 使用者帳號實測無任何直接 DDL 權限（含 curated）
- [ ] 命名檢核擋下全部違規樣態（`__`、超長、tmp、zone 不符）
- [ ] 兩案的核准路徑均可運作（定案前 console 以 feature flag 切換展示 ⚠️ 或僅實作定案方案）
- [ ] 稽核紀錄完整：申請→核准→執行全鏈路可回溯，含 DDL 全文與變更前 schema
- [ ] pipeline 引用中的表無法被誤刪
