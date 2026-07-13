# Feature Spec：Cluster Tier 分配與升級

> 來源：[PRD.md](../PRD.md) § 6.4
> 狀態：草稿（Phase 3）
> 職責邊界：本 spec 管「一個 ws 該落在哪種 cluster/規格、如何變更」；tier 內限制的**執行**（超額拒絕/降級/通知）屬 quota-and-usage。

## User Story

1. 作為**申請者**，我想要依申報的容量與效能需求被分配到合適的 tier，以便不為用不到的資源付費。
2. 作為**平台管理員**，我想要系統依公式自動建議 tier、我只做覆核，以便審核有一致標準。
3. 作為**成長中的 ws 負責人**，我的用量接近 tier 上限時我想提前收到通知並有明確的升級路徑，以便服務不中斷。
4. 作為**平台管理員**，我想掌握每個 tier 的資源池水位，以便硬體採購有依據（pool 浮動）。

## Tier 定義（草案）

| Tier | Data Volume | Max QPS | 配置 | 可用模式 |
|------|------------|---------|------|----------|
| 1 | < 500GB | < 20 | Shared cluster + workload group | 僅 managed |
| 2 | 500GB – 5TB | 20 – 100 | Small dedicated（3 BEs，16C/64GB/3.84TB SSD×2） | managed / self-managed |
| 3 | 5TB – 30TB | 100 – 500 | Medium dedicated（5–10 BEs，32C/128GB/3.84TB SSD×4） | managed / self-managed |
| >3 | >30TB 或 >500 QPS | — | 人工洽談（open issue #4） | — |

> ⚠️ **門檻待 RFC 定案（open issue #1）**：現行 data volume 與 QPS 綁在同一 tier 不合理（300GB/200QPS 或 20TB 低頻查詢的 ws 無法歸類）。候選方案：儲存與運算**解耦為兩條軸**——儲存量決定磁碟/BE 數下限，QPS 決定 CPU/併發配置，**取兩者較高者**決定 tier；BE 機型浮動（採購狀況）時以「等效資源單位」換算。本 spec 的流程設計不依賴門檻數字，門檻改版不影響流程。

## Tier 判定

### 申請時（初次判定）
- 輸入：申請表單的容量估算三要素（公式推得最大資料量，來源原始大小）+ 效能三要素（peak QPS、目標延遲、ingestion throughput）。
- 公式輸出建議 tier；平台管理員審核時可調整（附理由，記錄於申請單）。
- Self-managed 判定結果最低為 tier 2。

### 上線後（持續評估）
- 計量基準：儲存以 Doris 壓縮後實際占用；QPS 以 5 分鐘滑動窗口平均。
- 系統對每個 ws 持續比對用量 vs tier 上限：
  - **≥80% 上限**：通知 ws 負責人（預警，附升級指引）
  - **持續超標**（連續 N 天 ≥100%，N 待定）：觸發升級評估
- ⚠️ 待決：升級由使用者申請、平台主動觸發、或兩者皆可？超標但不願升級（成本考量）的 ws 如何處理（限流維持現 tier？寬限期後強制？）——涉及計費模式，與 Phase 5 一併定案。

## 升級流程（tier N → N+1）

```
TIER_CHANGE_REQUESTED（使用者申請或平台觸發）
  → TIER_REVIEW            （平台覆核：新 tier 判定 + 成本確認，成本中心負責人再確認）
  → SCHEDULED              （排定遷移窗口，通知 ws）
  → MIGRATING              （見下：遷移方式依模式與方向而異）
  → VERIFYING              （資料一致性核對 + 使用者確認查詢正常）
  → COMPLETED              （舊資源回收）
  ↘ WAITING_FOR_CAPACITY   （目標 tier 硬體不足：排隊 + 回報預估時間，pool 浮動）
```

### 遷移方式
| 情境 | 方式 | 停機 |
|------|------|------|
| Tier 1 → 2（shared → dedicated，必為 managed） | 平台負責：新 dedicated cluster 佈建 → 資料搬遷（優先評估 CCR/backup-restore）→ 切換連線端點 | 目標：僅切換窗口短暫唯讀 |
| Tier 2 → 3（dedicated 擴容，managed） | 平台負責：優先原 cluster 加 BE 節點（無資料搬遷）；機型不符時才整座搬遷 | 加節點無停機 |
| Tier 2 → 3（self-managed） | 屬「scale 代操作」：平台執行加節點，成本反映（見 self-managed-operations） | 加節點無停機 |
| 降級（用量長期遠低於 tier） | 使用者申請；平台評估後執行（縮容或搬回 shared）；不強制 | 同上 |

- Database 命名跟模式走、與 tier 無關（Decision Log 2026-07-13），遷移**不涉及改名**；連線端點變更透過交付新連線資訊（未來可評估統一 proxy/DNS 讓端點不變）。
- 遷移期間的寫入處理：managed 暫停 pipeline（Kafka 積壓緩衝，遷移後追上）；self-managed 由使用者配合窗口暫停寫入（責任歸屬寫入對外文件）。

## API 邏輯

| Method | Path | 說明 | 呼叫者 |
|--------|------|------|--------|
| GET | `/api/v1/workspaces/{ws}/tier` | 查詢目前 tier、用量比、升級建議 | ws 成員 |
| POST | `/api/v1/workspaces/{ws}/tier-changes` | 申請升/降級 | ws 負責人 |
| GET | `/api/v1/workspaces/{ws}/tier-changes/{id}` | 查詢變更進度（狀態機） | ws 成員 |
| POST | `/api/v1/workspaces/{ws}/tier-changes/{id}/review` | 覆核（approve/reject/defer） | 平台管理員 |
| GET | `/api/v1/capacity/pools` | 資源池水位（內部） | 平台管理員 |

### 錯誤碼
| HTTP | 錯誤碼 | 情境 |
|------|--------|------|
| 409 | `TIER_CHANGE_IN_PROGRESS` | 已有進行中的變更 |
| 422 | `TIER_NOT_AVAILABLE_FOR_MODE` | self-managed 申請降到 tier 1 |
| 422 | `EXCEEDS_MAX_TIER` | 目標超過 tier 3，導人工洽談 |
| 409 | `COST_CENTER_CONFIRMATION_REQUIRED` | 成本中心未確認新費用 |

## 與其他功能的依賴關係
- 初次判定輸入來自 **workspace-application**；公式定義於 Phase 5（NFR）。
- 用量數據來自 **quota-and-usage** 的計量；預警通知共用其通知管道。
- 遷移中的隔離組重建呼叫 **workspace-isolation** 佈建原語。
- Self-managed 擴容的成本反映與 **self-managed-operations** 銜接。
- 資料搬遷機制與 HA 的 CCR 能力共用（system-architecture 展開）。

## 邊界審查（Phase 4 補上）
| Edge Case | 是否適用 | 處理方式 |
|-----------|---------|----------|
| （待 Phase 4） | | |

## 驗收標準（Acceptance Criteria）
- [ ] 申請時公式判定 + 人工覆核路徑可運作，self-managed 最低 tier 2 被強制
- [ ] 用量達 80% 門檻時通知正確觸發，內容含升級指引
- [ ] 升級全程狀態機正確，WAITING_FOR_CAPACITY 回報預估時間
- [ ] Tier 1→2 遷移後資料一致性核對通過、命名與權限不變
- [ ] 遷移期間 managed pipeline 積壓資料在完成後追上，無資料遺失
- [ ] 降級路徑可運作且不強制
