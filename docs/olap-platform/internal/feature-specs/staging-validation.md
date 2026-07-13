# Feature Spec：Staging 驗證

> 來源：[PRD.md](../PRD.md) § 6.2
> 狀態：草稿（Phase 3）
> 對應申請狀態機區段：`STAGING_PROVISIONED → STAGING_TESTING → TEST_REPORT_REVIEW`（見 [workspace-application.md](workspace-application.md)）

## 目的

正式開通前，讓使用者在 staging 環境**試用與驗證設計**（schema 正確性、query pattern 可行性、pipeline 接通），並取得效能參考數據，避免：
- 錯誤的 schema/規模預估進入正式環境後需要平台協助 migration；
- 使用者完全沒摸過系統就上線，上線後才發現用法錯誤。

> **規格限制（2026-07-13 決議）**：staging 硬體規格**不會**與目標 tier 一致（機器不足），屬固定的小規格試用環境。因此效能測試結果**僅供參考、不可直接外推**至正式環境；報告審核重點是 schema/查詢設計的合理性與單位效能異常（例如小規模就明顯超時的查詢，上了正式環境也不會好）。

## User Story

1. 作為**申請者**，我想要在正式開通前於 staging 環境驗證我的 schema 與查詢效能，以便在錯誤成本最低的時候調整設計。
2. 作為**平台管理員**，我想要以量化的測試報告作為開通依據，以便把「效能達標」的責任在開通前就界定清楚。
3. 作為**申請者**，我想要方便地產生或匯入與申報量級相符的測試資料，以便不用自己造一套資料生成工具。

## 前置條件與假設

- 申請已達 `APPROVED` 狀態（按模式完成審查：self-managed 成本核准；managed 規模審查定 shared/dedicated）。
- Staging 為**固定小規格的試用環境**，不隨目標 tier 調整；多個申請者可能共用 staging 資源（隔離方式比照 shared cluster）。
- 兩種服務模式都要通過 staging 驗證；差異在灌資料方式（見下）。

## 流程

### 1. Staging 租借（STAGING_PROVISIONED）
- 系統依目標 tier 佈建 staging 環境（managed 含 zone databases 與投影片 schema 建立的 raw 表；self-managed 為空 cluster）。
- 租借有**期限**（建議預設 14 天，可申請延長一次）——避免 staging 資源被長期占用。
  - ⚠️ 期限與延長政策待定案。
- 交付 staging 連線資訊與測試指引。

### 2. 灌測試資料（STAGING_TESTING）
資料量以 staging 環境容量為上限（**不要求**與申報量級相符——staging 是小規格試用環境）。兩種方式：

| 方式 | 說明 | 適用 |
|------|------|------|
| (a) 假資料產生 | 使用者自產假資料；平台提供資料生成工具/範本（依申請時附的 schema 產生指定筆數） | 兩種模式 |
| (b) Lakehouse 匯入 | 平台提供機制從 production lakehouse 灌真實資料（或抽樣子集）到 staging 測試 | 兩種模式 |

- 灌入通道：managed 走平台 pipeline（staging 版）；self-managed 自行寫入。
- 方式 (b) 的 PII 考量經評估後**忽略**（2026-07-13 決議）。

### 3. 試用驗證與報告（STAGING_TESTING → TEST_REPORT_REVIEW）
- 使用者以申報的 query pattern 在 staging 上實測（規模受限，數據僅供參考）。
- 測試報告（平台提供固定格式範本）必含：
  1. 實際灌入資料量與測試規模說明
  2. 各 query pattern 的實測延遲（P95/P99）與資料量的關係（供平台判斷 scaling 合理性）
  3. Ingestion 接通驗證（managed：pipeline 跑通、schema 驗證行為確認；self-managed：自寫通道跑通）
  4. 發現的問題與調整紀錄（schema/查詢改了什麼）
- 平台審核報告，重點是**設計合理性**而非絕對效能數字：
  - **通過** → 進入正式佈建（PROVISIONING），staging 環境回收（含資料清除）
  - **設計有疑慮**（例如小規模就明顯超時、query pattern 與 schema 不匹配）→ 退回 STAGING_TESTING 調整重測；若判定是規模預估錯誤 → 退回 UNDER_REVIEW 重新定 tier
- 正式環境的實際效能以上線後的監控為準（quota-and-usage）；staging 報告不構成平台的效能承諾。

## 作業方式（人工）

> 申請流程無自動化 API（2026-07-13 決議），staging 相關作業一律人工：

| 作業 | 方式 |
|------|------|
| Staging 租借 | 申請核准後，平台團隊開內部工單佈建，交付連線資訊給申請者 |
| 租期延長（限一次） | 申請者向平台聯絡窗口提出，平台更新追蹤表 |
| 測試報告提交 | 申請者依報告範本（文件/投影片）提交，可於週三會議報告或 email 送審 |
| 報告審核 | 平台團隊審核，結論（pass / 重測 / 重審 tier）記入申請追蹤表並通知申請者 |

人工檢核清單（對應原自動驗證規則）：
- 申請未達 APPROVED 不受理 staging 租借
- 租期已過即回收（回收前通知）；延長以一次為限
- 灌入量超過 staging 容量上限時要求清理
- 報告缺必填項目退回補件

> 未來自動化（roadmap）：申請量成長後再實作 staging 租借/報告提交 API，本期不做。

## 與其他功能的依賴關係
- 上游：workspace-application（APPROVED 狀態）；schema 來自申請時審核結果。
- 方式 (b) 依賴 lakehouse 匯入機制（實作細節待補，與 batch ingestion 可能共用）。
- Staging 回收的資料清除需符合治理規範（Phase 6）。

## 邊界審查（Phase 4 補上）
| Edge Case | 是否適用 | 處理方式 |
|-----------|---------|----------|
| （待 Phase 4） | | |

## 驗收標準（Acceptance Criteria）
- [ ] 對外文件明確告知：staging 為小規格試用環境，效能數據僅供參考、不構成平台效能承諾
- [ ] 租期到期自動通知並回收（資料清除有紀錄）
- [ ] 灌入量超過 staging 容量上限時被正確擋下
- [ ] 報告缺項無法提交；審核結果三種出路（pass / 重測 / 重審 tier）狀態轉換正確
