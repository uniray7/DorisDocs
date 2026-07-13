# Feature Spec：Workspace 申請與審核

> 來源：[PRD.md](../PRD.md) § 6.1
> 狀態：草稿（Phase 3）

## User Story

1. 作為**團隊 Tech Lead**，我想要為我的團隊申請一個 workspace，以便把分析型工作負載從 Oracle/HBase 搬到 OLAP 平台。
2. 作為**平台管理員**，我想要依申請者提供的預估規模審核申請並分配 tier，以便控管硬體資源池的使用。
3. 作為**申請者**，我想要追蹤申請的處理狀態與被拒絕的原因，以便補件或調整需求後重新申請。
4. 作為**成本中心負責人**，我想要在申請時被通知並確認成本歸屬，以便掌握部門的平台支出。

## 前置條件與假設

- 申請者具備公司內部帳號（SSO），能存取申請入口。
- 每個 workspace 必須綁定一個有效的成本中心；申請時由申請者填寫，成本中心負責人確認。
- ⚠️ 假設待確認：申請入口為平台自建 portal + API；若公司已有統一工單系統（如 Jira Service Desk），申請入口應改走既有系統，只有審核後的佈建走平台 API。

## 前端互動流程

### 申請表單
申請者填寫：
1. Workspace 名稱（公司內唯一，命名規則：`^[a-z][a-z0-9-]{2,30}$`）
2. **服務模式**（managed / self-managed）——表單需並列兩種模式的責任分工說明，並明示**申請後不可轉換**；選 self-managed 且預估規模落在 tier 1 時，前端即時提示「self-managed 限定 dedicated cluster，將以 tier 2 起配（成本較高）」
3. 成本中心代碼
4. 團隊聯絡人（至少兩位：主要/備援）
5. 預估**來源原始資料量**（用於 tier 判定，上線後配額改以壓縮後占用計）
6. 預估 Max QPS（5 分鐘滑動窗口平均）
7. 使用情境描述（自由文字，審核參考）
8. 資料敏感度聲明（是否含 PII/機敏資料——影響 Phase 6 治理流程）
9. 預計匯入方式（managed：平台 batch / 平台 streaming；self-managed：一律自寫，此欄位隱藏）

### 送出後
- 顯示申請編號與目前狀態；狀態變更時通知申請者（管道待定：email / 公司 IM）。
- 成本中心負責人收到確認請求，確認後申請才進入平台審核。

### 審核（平台管理員視角）
- 審核清單：待審申請列表，顯示預估規模與系統建議 tier。
- 管理員可：核准（可調整 tier）、退回補件（附原因）、拒絕（附原因）。
- 申請聲明含 PII 時，需完成安全/合規會簽才能核准（銜接 PRD § 9）。

### 佈建與交付
- 核准後系統自動佈建：
  - Tier 1：在 shared cluster 建立 database + resource group + 初始帳號
  - Tier 2/3：於資源池開立 dedicated cluster（若當下硬體不足，狀態停在「等待資源」並通知申請者預計時間）
- 完成後交付：連線資訊、初始管理帳號、配額明細、快速開始文件連結。

## API 邏輯

### Endpoints
| Method | Path | 說明 | 呼叫者 |
|--------|------|------|--------|
| POST | `/api/v1/workspace-applications` | 送出申請 | 申請者 |
| GET | `/api/v1/workspace-applications/{id}` | 查詢申請狀態 | 申請者/管理員 |
| POST | `/api/v1/workspace-applications/{id}/cost-center-confirmation` | 成本中心負責人確認 | 成本中心負責人 |
| POST | `/api/v1/workspace-applications/{id}/review` | 審核（approve / request-changes / reject） | 平台管理員 |
| GET | `/api/v1/workspaces/{ws}` | 查詢 workspace 狀態與配額 | ws 成員 |

### Request（POST /workspace-applications）
```json
{
  "name": "ads-analytics",
  "service_mode": "managed",
  "cost_center": "CC-1042",
  "contacts": ["alice@corp", "bob@corp"],
  "estimated_raw_volume_gb": 800,
  "estimated_max_qps": 50,
  "use_case": "廣告成效報表，取代現有 Oracle 分析庫",
  "contains_sensitive_data": false,
  "ingestion_methods": ["batch", "streaming"]
}
```
規則：`service_mode = "self-managed"` 時 `ingestion_methods` 必須為空（自寫入不需申報）；tier 判定結果最低為 2。

### Response（201）
```json
{
  "application_id": "wsapp-20260713-0001",
  "status": "PENDING_COST_CENTER_CONFIRMATION",
  "suggested_tier": 2
}
```

### 申請狀態機
```
SUBMITTED
  → PENDING_COST_CENTER_CONFIRMATION   （成本中心負責人確認中）
  → UNDER_REVIEW                        （平台審核中；含 PII 者需完成合規會簽）
  → CHANGES_REQUESTED ──(申請者補件)──→ UNDER_REVIEW
  → REJECTED                            （終態，附原因；可重新申請）
  → APPROVED
  → PROVISIONING / WAITING_FOR_CAPACITY （tier 2/3 硬體不足時）
  → ACTIVE                              （交付完成，workspace 生效）
```

### Workspace 生命週期狀態機（申請完成後）
```
ACTIVE ⇄ SUSPENDED（違規/欠費/閒置，見 quota 與治理規範）
ACTIVE → DECOMMISSIONING → DECOMMISSIONED（申請者主動或平台終止，含資料清除）
```

### 錯誤碼
| HTTP | 錯誤碼 | 情境 |
|------|--------|------|
| 400 | `WS_NAME_INVALID` | 名稱不符命名規則 |
| 409 | `WS_NAME_TAKEN` | 名稱已被使用 |
| 400 | `COST_CENTER_INVALID` | 成本中心代碼不存在或已停用 |
| 403 | `NOT_COST_CENTER_OWNER` | 非成本中心負責人嘗試確認 |
| 409 | `APPLICATION_NOT_REVIEWABLE` | 狀態機不允許此操作（如已 REJECTED 再 approve） |
| 422 | `ESTIMATE_EXCEEDS_MAX_TIER` | 預估規模超過 tier 3 上限（>30TB 或 >500 QPS），需人工洽談（open issue #4） |
| 400 | `INGESTION_NOT_ALLOWED_FOR_SELF_MANAGED` | self-managed 申請填了平台 ingestion 方式 |

## 與其他功能的依賴關係
- 佈建動作依賴 **多租戶隔離**（database/resource group 建立）與 **Cluster tier 分配**（tier 判定規則）。
- 交付的初始帳號依賴 **帳號與存取控制** 的帳號模型。
- `ESTIMATE_EXCEEDS_MAX_TIER` 的處理依 open issue #4 決議。

## 邊界審查（Phase 4 補上）
| Edge Case | 是否適用 | 處理方式 |
|-----------|---------|----------|
| （待 Phase 4） | | |

## 驗收標準（Acceptance Criteria）
- [ ] 申請 → 成本中心確認 → 審核 → 佈建 → 交付全流程可在無人工介入（tier 1）下完成
- [ ] 每個狀態變更都通知申請者，且拒絕/退回必附原因
- [ ] 含 PII 聲明的申請無法在未完成合規會簽前被核准
- [ ] 同名 workspace 申請被正確拒絕
- [ ] tier 2/3 在資源不足時正確進入 WAITING_FOR_CAPACITY 並回報預估等待時間
