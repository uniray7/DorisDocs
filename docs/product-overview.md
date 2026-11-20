# OLAP Platform 產品規劃摘要

> 對象：管理層與 PM ｜ 更新：2026-11-19 ｜ 狀態：規劃中（PRD 細化階段，各 feature 文件為草稿）
> 詳細規格見 `internal/prd/README.md` 與 `internal/prd/` 各 feature 文件；本文為決策導向摘要。

---

## 一、我們要做什麼

以 **Apache Doris** 為基礎，提供公司內部的**多租戶 OLAP 分析服務**，並附帶 **data ingestion pipeline** 讓各部門把資料灌進來分析。

**為什麼做**：擺脫 Oracle license 成本與綁定、sunset HBase 舊 tech stack。既有分析型資料規模約數百 TB。

**明確不做**：既有資料遷移（各團隊自理）、交易型（OLTP）工作負載、BI 工具託管、跨 workspace 資料分享。

---

## 二、產品形狀：兩種服務模式

這是整個產品最核心的設計——**責任邊界以服務模式劃分**，申請時選定、**不可事後轉換**：

- **Managed**：適合想省事、接受平台規範的團隊；平台承擔資料管理責任，寫入只走平台 pipeline，可用 shared 或 dedicated cluster。
- **Self-managed**：適合高客製、大流量、有能力自管的團隊；使用者自建 pipeline、自負資料責任，限 dedicated cluster。

兩種模式在審查、開通、寫入、DDL、責任、支援上的完整對照見 [PRD §7](internal/prd/README.md#7-多租戶與資源隔離模型)。

## 三、資源分級（Tier）

Tier 1 為 shared cluster（僅 managed），tier 2 起為 dedicated；超過 tier 3 個案洽談（TBD-2）。各 tier 的資料量、QPS 門檻與配置見 [cluster-topology-and-tiering.md](internal/prd/cluster-topology-and-tiering.md#tier-定義草案)。

> ⚠️ 門檻數字為草案。「資料量與 QPS 綁死在同一 tier」已知不合理（例：資料小但查詢重的團隊無法歸類），已規劃 RFC 討論儲存/運算解耦。

**Fab**：workspace 申請時指定要在哪些 fab 使用服務（至少兩個）；平台在每個 fab 各自提供相同數量、彼此獨立的 Doris 服務，不跨 fab 部署。

## 四、使用者旅程

```
填投影片 template → 每週三申請會議報告 → 審核（按模式分軌，見下）
  → Staging 試用環境驗證（managed 必經、self-managed 選配；小規格，驗設計不驗效能）→ 平台佈建正式環境 → 上線
```

- **Self-managed 審查**：僅需管理層成本核准；通過即開 cluster + monitor/alert/log。
- **Managed 審查**：須申報 workspace 整體預期 data size，審查決定 shared 或 dedicated cluster。

- **申請流程純人工**（投影片 + 會議 + 追蹤表），不建自動化系統；申請量成長後再依已定義的狀態機做自助入口。
- **Staging 是試用性質**：固定小規格（機器不足以對齊正式規格），驗證 schema/查詢設計合理性；效能數據僅供參考，**不構成平台效能承諾**。

## 五、Managed 模式的資料管理

- **兩層 zone**：`raw`（平台建表、平台 pipeline 寫入）→ `curated`（service account 以 `INSERT … SELECT` 產出、直接建表）。資料一律由平台 pipeline 寫入 raw，寫入細節待定（TBD-28）。
- **Streaming pipeline**：使用者以固定 format 打 message 進平台 Kafka，支援三種 format（generic CDC / DB2 JSON CDC / DB2 XML CDC）；格式錯誤的資料進 corrupted data 區（MinIO）並告警，不中斷匯入。
- **Batch pipeline**：使用者上傳檔案至平台儲存的 landing 路徑，排程/手動觸發匯入，與 streaming 走同一套落地與治理路徑；來源介面預留未來 lakehouse → Doris 通道的擴充（規格草稿多處假設待確認）。
- **高可用**（兩種模式）：cluster 部署在多台機器、每個 table 3 個 replica，單一機器故障可容錯；中長期分散到 3 個機櫃或 3 個 datacenter（做法待研究，TBD-9）。
- **災難復原**：未來需要 backup／restore 到 S3 或 MinIO（TBD-36）。

## 六、已定案的關鍵決策（節錄）

| 決策 | 內容 |
|------|------|
| 租戶單位 | Workspace（團隊自定粒度），**強制綁定成本中心** |
| 隔離要求 | 資料、metadata、運算三層完全隔離；不支援跨 ws 分享 |
| 模式轉換 | 不可轉換；要換模式需開新 ws 搬資料 |
| 計量方式 | 儲存：審核用原始大小、上線後按壓縮後占用；QPS：5 分鐘滑動窗口平均 |
| 命名慣例 | Managed 一律 `{ws}__{自訂}__{raw\|curated}`；self-managed 待定（TBD-8） |
| 對外文件深度 | Managed 須講明使用規範+限制+責任歸屬；self-managed 重點講責任歸屬 |

## 七、需要管理層/PM 拍板或注意的事項

### 待拍板（會影響產品走向）

需要管理層拍板的事項：TBD-1、TBD-2、TBD-3、TBD-4、TBD-5、TBD-6、TBD-7、TBD-8、TBD-20、TBD-29、TBD-30、TBD-40。各項的影響、負責人與狀態見 [00-progress.md 的 TBD 清單](00-progress.md#tbd-清單)。

### 需注意的風險
- **人工申請流程的規模瓶頸**：每週三會議承載量有限，申請量成長時會排隊；已預留自動化路徑，但需要有觸發升級的判斷點。
- **Staging 與正式環境落差**：staging 驗不了真實效能，上線後才見真章——對外文件會明確免責，但第一批使用者的預期管理要做好。
- **Lakehouse 使用習慣落差**：使用者習慣跨 workspace 存取，平台目前不支援，對外文件需事先說明。

## 八、文件進度與下一步

各文件狀態與完整 TBD 清單見 [00-progress.md](00-progress.md)；決議演進見 [decision-history.md](decision-history.md)。
