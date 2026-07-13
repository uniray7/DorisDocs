# Feature Spec：Staging 驗證

> 來源：[PRD.md](../PRD.md) § 6.2
> 狀態：草稿（Phase 3）
> 對應申請狀態機區段：`STAGING_PROVISIONED → STAGING_TESTING → TEST_REPORT_REVIEW`（見 [workspace-application.md](workspace-application.md)）

## 目的

正式開通前，用與申報量級相符的資料驗證「peak QPS 下能否達到預期 query performance」，避免：
- 使用者上線後才發現效能不如預期，歸咎平台；
- 錯誤的 schema/規模預估進入正式環境後需要平台協助 migration。

## User Story

1. 作為**申請者**，我想要在正式開通前於 staging 環境驗證我的 schema 與查詢效能，以便在錯誤成本最低的時候調整設計。
2. 作為**平台管理員**，我想要以量化的測試報告作為開通依據，以便把「效能達標」的責任在開通前就界定清楚。
3. 作為**申請者**，我想要方便地產生或匯入與申報量級相符的測試資料，以便不用自己造一套資料生成工具。

## 前置條件與假設

- 申請已達 `APPROVED` 狀態（schema 與規模預估已通過平台審核）。
- Staging 環境規格與**目標 tier 一致**（tier 2 申請就給 small cluster 規格）——否則測試結果無法外推，報告無效。
  - ⚠️ 待確認：staging 硬體是否從同一個資源 pool 撥出？若 pool 緊張，staging 租借可能要排隊（影響開通 lead time 的承諾）。
- 兩種服務模式都要通過 staging 驗證；差異在灌資料方式（見下）。

## 流程

### 1. Staging 租借（STAGING_PROVISIONED）
- 系統依目標 tier 佈建 staging 環境（managed 含 zone databases 與審核過的 raw 表；self-managed 為空 cluster）。
- 租借有**期限**（建議預設 14 天，可申請延長一次）——避免 staging 資源被長期占用。
  - ⚠️ 期限與延長政策待定案。
- 交付 staging 連線資訊與測試指引。

### 2. 灌測試資料（STAGING_TESTING）
量級**必須與申報資料量相符**（平台以壓縮後占用抽查比對，偏差過大退回）。兩種方式：

| 方式 | 說明 | 適用 |
|------|------|------|
| (a) 假資料產生 | 使用者自產大量假資料；平台提供資料生成工具/範本（依審核過的 schema 產生指定筆數） | 兩種模式 |
| (b) Lakehouse 匯入 | 平台提供機制從 production lakehouse 灌真實資料到 staging 測試 | 兩種模式 |

- ⚠️ 方式 (b) 的治理疑慮（Phase 6 需處理）：production 資料進 staging——若含 PII/機敏資料，staging 的存取控制與資料清除必須比照 production 等級；申請時聲明含 PII 者是否禁用方式 (b) 或需額外簽核，待 Phase 6 決議。
- 灌入通道：managed 走平台 pipeline（staging 版）；self-managed 自行寫入。

### 3. 效能測試與報告（STAGING_TESTING → TEST_REPORT_REVIEW）
- 使用者以申報的 query pattern 執行負載測試，需涵蓋 **peak QPS 場景**。
- 測試報告（平台提供固定格式範本）必含：
  1. 實際灌入資料量（壓縮後）vs 申報量
  2. 測試期間達到的 QPS（5 分鐘滑動窗口）
  3. Peak QPS 下的查詢延遲 P95/P99 vs 申報的預期 query performance
  4. Ingestion throughput 實測值（managed：pipeline 吞吐；self-managed：自寫吞吐）
  5. 未達標項目與調整計畫（若有）
- 平台審核報告：
  - **通過** → 進入正式佈建（PROVISIONING），staging 環境回收（含資料清除）
  - **未達標** → 退回 STAGING_TESTING 調整重測；若判定是規模預估錯誤 → 退回 UNDER_REVIEW 重新定 tier

## API 邏輯

| Method | Path | 說明 | 呼叫者 |
|--------|------|------|--------|
| POST | `/api/v1/workspace-applications/{id}/staging` | 申請佈建 staging（APPROVED 後） | 申請者 |
| GET | `/api/v1/workspace-applications/{id}/staging` | 查詢 staging 狀態/租期 | 申請者 |
| POST | `/api/v1/workspace-applications/{id}/staging/extension` | 申請延長租期（限一次） | 申請者 |
| POST | `/api/v1/workspace-applications/{id}/test-report` | 提交測試報告 | 申請者 |
| POST | `/api/v1/workspace-applications/{id}/test-report/review` | 審核報告（pass / fail-retest / fail-rereview） | 平台管理員 |

### 錯誤碼
| HTTP | 錯誤碼 | 情境 |
|------|--------|------|
| 409 | `APPLICATION_NOT_APPROVED` | 申請未達 APPROVED 就申請 staging |
| 409 | `STAGING_EXPIRED` | 租期已過，需重新申請或已被回收 |
| 409 | `EXTENSION_ALREADY_USED` | 延長次數已用完 |
| 422 | `DATA_VOLUME_MISMATCH` | 灌入量與申報量偏差超過容許值（門檻待定） |
| 422 | `REPORT_INCOMPLETE` | 測試報告缺必填項目 |

## 與其他功能的依賴關係
- 上游：workspace-application（APPROVED 狀態）；schema 來自申請時審核結果。
- 方式 (b) 依賴 lakehouse 匯入機制（實作細節待補，與 batch ingestion 可能共用）。
- 資料量抽查依賴配額計量（quota-and-usage 的壓縮後占用計算）。
- Staging 回收的資料清除需符合治理規範（Phase 6）。

## 邊界審查（Phase 4 補上）
| Edge Case | 是否適用 | 處理方式 |
|-----------|---------|----------|
| （待 Phase 4） | | |

## 驗收標準（Acceptance Criteria）
- [ ] Staging 規格與目標 tier 一致，測試結果可外推
- [ ] 租期到期自動通知並回收（資料清除有紀錄）
- [ ] 灌入量與申報量偏差超過門檻時，報告無法提交
- [ ] 報告缺項無法提交；審核結果三種出路（pass / 重測 / 重審 tier）狀態轉換正確
- [ ] 含 PII 的 lakehouse 匯入依 Phase 6 決議受控
