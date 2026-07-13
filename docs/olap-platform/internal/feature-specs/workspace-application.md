# Feature Spec：Workspace 申請與審核

> 來源：[PRD.md](../PRD.md) § 6.1
> 狀態：草稿（Phase 3）

## User Story

1. 作為**團隊 Tech Lead**，我想要為我的團隊申請一個 workspace，以便把分析型工作負載從 Oracle/HBase 搬到 OLAP 平台。
2. 作為**平台管理員**，我想要依申請者提供的預估規模審核申請並分配 tier，以便控管硬體資源池的使用。
3. 作為**申請者**，我想要追蹤申請的處理狀態與被拒絕的原因，以便補件或調整需求後重新申請。
4. 作為**成本中心負責人**，我想要在申請時被通知並確認成本歸屬，以便掌握部門的平台支出。

## 前置條件與假設

- **申請流程為純人工，無任何自動化 API**（2026-07-13 決議）：使用者依平台提供的**投影片 template** 填入必要資訊，於**每週三申請會議**向平台團隊報告，審核決議當場或會後給出。
- 每個 workspace 必須綁定一個有效的成本中心；申請前由申請者取得成本中心負責人同意（會議中確認）。

## 申請流程（人工）

### 1. 取得並填寫投影片 template
平台提供投影片 template，申請者依序填入（欄位即審核所需資訊）：
1. Workspace 名稱（公司內唯一，命名規則：`^[a-z][a-z0-9]{2,20}$`——僅小寫字母+數字，**禁用 `-` 與 `_`**：ws 名稱會組成 Doris database 名 `{ws}__{user_defined}__{zone}`，Doris 不允許 `-` 且雙底線 `__` 保留為分隔符）
2. **服務模式**（managed / self-managed）——template 需並列兩種模式的責任分工說明頁，並明示**申請後不可轉換**；附註「self-managed 限定 dedicated cluster，最低以 tier 2 起配（成本較高）」
3. 成本中心代碼
4. 團隊聯絡人（至少兩位：主要/備援）
5. 容量估算三要素：目前 table data size、預期資料成長量、資料 retention → 依 template 附的**公式**試算最大資料量（用於 tier 判定；上線後配額改以壓縮後占用計）
6. 效能需求三要素：peak QPS（5 分鐘滑動窗口平均）、預期 query performance（目標延遲）、ingestion throughput → 依平台**公式**試算建議規格
7. Table schema 與 query pattern（managed 附上；**平台是否審核待 PM 決定**——選項 A：平台審核把關；選項 B：不審，僅用於容量評估與選配的建表建議。無論何者，開通後的建表以 ws owner approve 為準）
8. 使用情境描述（審核參考）
9. 資料敏感度聲明（是否含 PII/機敏資料——影響 Phase 6 治理流程）
10. 預計匯入方式（managed：streaming（三種 CDC format）/ batch，可複選；self-managed 略過此頁）

### 2. 每週三申請會議（審查按模式分軌）
- 申請者攜投影片向平台團隊報告。
- **Self-managed 審查**：僅需**管理層對成本支出核准**即通過（會議確認成本中心與規模即可，不審技術內容）。
- **Managed 審查**：必須申報**整個 workspace 的預期 data size**；審查結論決定開 **shared 或 dedicated cluster**（tier 判定）。平台當場提問規模估算依據與模式選擇的理解。
- 會議結論三種：**核准**（可調整 tier，記錄理由）、**補件**（下次會議再報，附具體缺項）、**拒絕**（附原因，可再申請）。
- 申請聲明含 PII 時，需完成安全/合規會簽才能核准（銜接 PRD § 9）。
- 決議記錄於會議紀錄（申請追蹤表），並以 email/IM 通知申請者與成本中心負責人。

### 3. 核准後
- 平台團隊開內部工單執行 staging 租借與後續佈建（見 [staging-validation.md](staging-validation.md)）。
- 狀態追蹤：以人工維護的申請追蹤表為準（欄位對應下方狀態機）。

### 佈建與交付
- Staging 測試通過後由平台團隊執行佈建（內部可用腳本輔助，對使用者無自動化介面）：
  - Tier 1（必為 managed）：在 shared cluster 建立 zone databases（`{ws}__{user_defined}__tmp`／`__raw`／`__curated`）+ resource group + 初始帳號；申請投影片附的 schema 視為 ws owner 已核准的初始建表，由平台系統執行
  - Tier 2/3 managed：開立 dedicated cluster，其餘同上
  - Tier 2/3 self-managed：開立 dedicated cluster + **monitor、alert、log** + 保護性 config + 初始帳號（不建 zone databases，命名自由）
  - 硬體不足時，狀態停在 WAITING_FOR_CAPACITY 並通知申請者預計時間
- 完成後交付：連線資訊、初始管理帳號、配額明細、快速開始文件連結（依模式給對應文件：managed 給 pipeline/DDL 申請指南，self-managed 給責任歸屬與告警說明）。

## 投影片 Template 規格

Template 章節對應上方欄位，每個申請必附（審核會議即照此順序報告）：
1. 團隊與使用情境介紹（含成本中心、聯絡人）
2. 服務模式選擇與理由（managed / self-managed；含「不可轉換」的認知確認頁）
3. 容量估算（三要素 + 平台公式試算結果）
4. 效能需求（peak QPS、目標延遲、ingestion throughput）
5. Table schema 與 query pattern（managed 必附；self-managed 可略）
6. 資料敏感度聲明（是否含 PII）
7. 預計匯入方式（managed：三種 CDC format / batch；self-managed 略）
8. 預計時程

審核規則（會議中人工檢核，對應原欄位驗證）：
- ws 名稱符合 `^[a-z][a-z0-9]{2,20}$` 且未被使用
- 最大資料量由容量三要素以公式推得，tier 由容量與效能需求共同推算（公式待 Phase 5 定案）
- self-managed：不申報匯入方式（一律自寫）、schema 可免附（不審表）、tier 最低為 2
- 超過 tier 3 上限（>30TB 或 >500 QPS）：會議中直接轉人工洽談（open issue #4）

### 申請狀態機（人工追蹤表欄位）
> 無自動化系統，狀態由平台團隊在申請追蹤表上人工維護；狀態語義如下，供追蹤表與未來自動化沿用。
```
SUBMITTED                               （投影片提交，排入最近的週三申請會議）
  → UNDER_REVIEW                        （會議報告與審核，按模式分軌：self-managed 僅管理層成本核准；managed 審 data size 定 shared/dedicated；含 PII 者需完成合規會簽）
  → CHANGES_REQUESTED ──(補件，下次會議再報)──→ UNDER_REVIEW
  → REJECTED                            （終態，附原因；可再申請）
  → APPROVED
  → STAGING_PROVISIONED                 （staging 試用環境租借給申請者：固定小規格，不隨 tier 調整）
  → STAGING_TESTING                     （申請者灌測試資料試用：自產假資料或自 production lakehouse 匯入）
  → TEST_REPORT_REVIEW                  （平台審核試用報告：schema/查詢設計合理性；效能數據僅供參考）
      ├─ 未達標 → STAGING_TESTING       （調整 schema/規模預估後重測；必要時退回 UNDER_REVIEW 改 tier）
      └─ 通過 ↓
  → PROVISIONING / WAITING_FOR_CAPACITY （tier 2/3 硬體不足時）
  → ACTIVE                              （交付完成，workspace 生效；staging 環境回收）
```
> Staging 驗證的細節（租借期限、假資料產生工具、lakehouse 匯入機制、報告格式）見 [staging-validation.md](staging-validation.md)。

### Workspace 生命週期狀態機（申請完成後）
```
ACTIVE ⇄ SUSPENDED（違規/欠費/閒置，見 quota 與治理規範）
ACTIVE → DECOMMISSIONING → DECOMMISSIONED（申請者主動或平台終止，含資料清除）
```

### 常見退件/補件原因（會議檢核清單）
| 原因 | 說明 |
|------|------|
| ws 名稱不符規則或已被使用 | `^[a-z][a-z0-9]{2,20}$`，公司內唯一 |
| 成本中心無效或負責人未同意 | 代碼不存在/已停用，或會議中無法確認成本歸屬 |
| 規模估算依據不足 | 容量/效能三要素缺項，公式無法推算 tier |
| 超過 tier 3 上限 | >30TB 或 >500 QPS，轉人工洽談（open issue #4） |
| self-managed 填了平台匯入方式 | self-managed 一律自寫，不申報 ingestion |
| Managed 未申報 ws 整體預期 data size | shared/dedicated 判定無依據 |
| PII 聲明未完成合規會簽 | 銜接 PRD § 9 |

### 未來自動化（Roadmap，非本期範圍）
> 申請量成長到人工會議吃不消時，再依上方狀態機實作自助申請入口與 API；屆時投影片欄位即表單欄位、檢核清單即驗證規則。本期不做。

## 與其他功能的依賴關係
- 正式佈建前依賴 **Staging 驗證**（staging-validation.md）的測試報告通過。
- 佈建動作依賴 **多租戶隔離**（database/resource group 建立）與 **Cluster tier 分配**（tier 判定規則）。
- Managed 申請時審核的 schema 即 raw zone 初始建表，後續增修表走 **Database/Table 申請流程**（database-table-application.md）。
- 交付的初始帳號依賴 **帳號與存取控制** 的帳號模型。
- `ESTIMATE_EXCEEDS_MAX_TIER` 的處理依 open issue #4 決議。

## 邊界審查（Phase 4 補上）
| Edge Case | 是否適用 | 處理方式 |
|-----------|---------|----------|
| （待 Phase 4） | | |

## 驗收標準（Acceptance Criteria）
- [ ] 投影片 template 完成並涵蓋全部審核所需欄位；週三申請會議節奏建立且有會議紀錄
- [ ] 申請追蹤表狀態欄與本 spec 狀態機一致；每次狀態變更以 email/IM 通知申請者，拒絕/補件必附原因
- [ ] 含 PII 聲明的申請無法在未完成合規會簽前被核准
- [ ] 同名 workspace 申請被正確拒絕
- [ ] self-managed 申請不會被配到 shared cluster（最低 tier 2），且不含 zone databases
- [ ] managed 佈建後 zone databases 命名符合 `{ws}__{user_defined}__{zone}` 且通過 Doris 命名檢核（總長 ≤64、user_defined 不含 `__`）
- [ ] tier 2/3 在資源不足時正確進入 WAITING_FOR_CAPACITY 並回報預估等待時間
