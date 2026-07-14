# Feature Spec：配額與用量可視化

> 來源：[PRD.md](../PRD.md) § 6.9
> 狀態：草稿（Phase 3）

## 定位

讓使用者**自己看得到**配額、用量與查詢表現，把「超標了嗎」「還剩多少」「為什麼變慢」的高頻提問從人工支援轉為自助查詢；同時是 tier 超標偵測（cluster-tiering 的 80% 早期預警）與計費（模式待定）的資料基礎。

## 計量定義（已定案，本 spec 落實為指標）

| 計量 | 定義 | 決議來源 |
|------|------|---------|
| 儲存用量 | Doris **壓縮後實際占用**（含副本 ⚠️ 待確認：配額計單副本或全副本） | Decision Log 2026-07-13 |
| 儲存計入範圍（managed） | raw + curated + **tmp + corrupted data + batch landing 殘留** | streaming/batch spec 行為規格 |
| QPS | **5 分鐘滑動窗口平均**判超標；瞬間爆量由 workload group 併發兜底 | Decision Log 2026-07-13 |
| QPS 資料來源 | **Gateway 計量**（RFC-002 第 1 層）；與 Doris audit log 對賬。⚠️ Arrow Flight 查詢的計量路徑待 RFC-002 驗證（gateway 以 MySQL protocol 為前提） | RFC-002 |

## 使用者可見內容（Web Console 儀表板）

### 配額與用量頁
- 儲存：配額、目前占用（壓縮後）、占比、**80% 早期預警狀態**、按 zone/database 分解（managed）。
- QPS：tier 上限、目前 5 分鐘滑動平均、近 7/30 天峰值曲線。
- Ingestion：各 pipeline/job 吞吐、tmp 與 corrupted data 占用明細。
- 超標紀錄：何時超、超多少、平台採取的動作（限流/通知，依 cluster-tiering 政策）。

### 查詢表現頁
- 查詢延遲分布（P50/P95/P99）、慢查詢清單（**僅本 ws**——metadata 隔離約束，不得洩漏他租戶查詢）。
- Shared cluster（tier 1）另顯示：workload group 排隊次數、被 kill 查詢清單與原因（RFC-002 kill policy 的透明化——使用者要知道**為什麼**被殺、如何避免）。

### 告警設定
- 使用者可自訂門檻告警（儲存占比、lag、corrupted data 計數）送 ws 聯絡人；平台強制告警（80% 預警、超標、被 kill）不可關閉。

### Self-managed 的差異
- Self-managed 看 **cluster 級**視圖（節點資源、磁碟水位——即開通交付的 monitor），無 zone/pipeline 分解（無平台 pipeline）、無 DDL 治理視圖。
- QPS/儲存是否對 self-managed 也做 tier 計量展示 ⚠️ 待確認（超標升級政策是否適用 self-managed，關聯 cluster-tiering）。

## 行為規格

| # | 情境 | 行為 |
|---|------|------|
| 1 | 儲存占用達配額 80% | 儀表板標示 + 強制告警送聯絡人（cluster-tiering 早期預警） |
| 2 | 儲存達 100% | 依 cluster-tiering 超標政策（限流/擋寫入 ⚠️ 政策細節與「不願升級」處理待拍板 #4）；儀表板顯著標示 |
| 3 | QPS 滑動平均超 tier 上限 | 記錄超標事件 + 通知；gateway 限流行為依 RFC-002 |
| 4 | 查詢被 kill policy 終止 | 事件進「被 kill 查詢清單」，含觸發規則與建議（如：補分區條件） |
| 5 | 使用者查他人用量 | 不可能：所有視圖以 ws 為邊界（workspace-isolation） |
| 6 | 計量延遲 | 儲存用量非即時（refresh 週期 ⚠️ 如每小時）；QPS 近即時（gateway 端） |

## 平台側視角（內部）

- 平台管理員看全 cluster/全 ws 匯總：容量水位（浮動硬體池的採購依據）、各 ws 用量排名、超標事件、shared cluster 的 workload group 指標（RFC-002 監控項）。
- 此視角同時服務計費報表（chargeback/showback，待拍板 #1）。

## API 邏輯

| Method | Path | 說明 |
|--------|------|------|
| GET | `/api/v1/workspaces/{ws}/quota` | 配額與目前用量摘要 |
| GET | `/api/v1/workspaces/{ws}/usage/storage` | 儲存明細（zone/database/時間序列） |
| GET | `/api/v1/workspaces/{ws}/usage/qps` | QPS 時間序列與超標事件 |
| GET | `/api/v1/workspaces/{ws}/queries/slow` | 慢查詢/被 kill 查詢清單（僅本 ws） |
| PUT | `/api/v1/workspaces/{ws}/alerts` | 自訂告警門檻設定 |

## 與其他功能的依賴關係
- 80% 預警與超標處理政策：**cluster-tiering**。
- QPS 計量來源與 kill 事件：**RFC-002**（gateway、workload schedule policy）。
- tmp/corrupted/landing 占用明細：**streaming/batch-ingestion**。
- 視圖的 ws 邊界：**workspace-isolation**（metadata 隔離延伸到監控資料）。
- 配額數值、refresh 週期、計費模式：Phase 5。

## 待確認事項
1. 儲存配額計單副本還是全副本（影響使用者理解的「可用空間」）。
2. Self-managed 是否納入 tier 計量展示與超標政策。
3. 達 100% 的處置（限流 vs 擋寫入）與「超標但不願升級」——待拍板 #4。
4. 用量資料保留時長（時間序列存多久）。

## 邊界審查（Phase 4 補上）
| Edge Case | 是否適用 | 處理方式 |
|-----------|---------|----------|
| （待 Phase 4） | | |

## 驗收標準（Acceptance Criteria）
- [ ] 儲存/QPS 計量與定義一致（壓縮後、5 分鐘滑動窗口），與 gateway/audit log 對賬誤差在容許範圍
- [ ] 80% 預警與超標事件正確觸發並送達聯絡人；強制告警不可被使用者關閉
- [ ] 慢查詢/被 kill 清單僅含本 ws 查詢（隔離驗證）
- [ ] 被 kill 查詢附觸發規則說明（對齊對外使用規範的 kill 門檻條款）
- [ ] tmp/corrupted/landing 占用計入且可在明細中辨識
