# olap-platform 文件產出進度

## Phase 狀態
| Phase | 狀態 | 完成日期 |
|-------|------|----------|
| 0 定位分流 | 完成 | 2026-07-12 |
| 1 脈絡傾倒 | 進行中 | |
| 2 骨架生成 | 未開始 | |
| 2.5 名詞與範圍對齊 | 未開始 | |
| 3 分塊細化 | 未開始 | |
| 4 邊界審查 | 未開始 | |
| 5 非功能需求/SLA | 未開始 | |
| 6 治理與合規審查 | 未開始 | |
| 7 利害關係人會簽 | 未開始 | |
| 8 衍生文件產出 | 未開始 | |
| 9 試點驗證 | 未開始 | |
| 10 發布與版本治理 | 未開始 | |

## Phase 0：分流結論
- 案子範圍：整個 OLAP 平台服務首次推出（以 Apache Doris 為基礎），含 data ingestion pipeline 與服務申請/審核流程。
- 文件策略：內部與對外文件都要產出，**從內部文件（PRD）先做**，對外文件於 Phase 8 從核准後的 PRD 衍生。
- 已知環境限制：服務部署於 private network，文件撰寫過程無法連線驗證，所有服務現況以團隊口述為準。

## Phase 1：脈絡摘要

### 痛點與動機
- 現況：各部門用付費版 Oracle，部分團隊用 HBase。
- 動機：擺脫付費軟體綁定（license 成本），並 sunset 舊有 tech stack。
- 既有資料規模：Oracle + HBase 合計約數百 TB。
- **平台不負責資料遷移**：Oracle/HBase → Doris 的搬遷由各團隊自理，平台只提供 ingestion pipeline 收資料。

### 目標使用者
- 主要為後端工程師，會寫 SQL。
- 使用者多來自 lakehouse 環境，**習慣跨 workspace 存取資料**——但本平台不允許，需在對外文件中前置溝通。

### 多租戶模型
- 租戶單位：**workspace (ws)**，對應部門。
- 隔離要求：資料完全隔離 + resource group 隔離運算資源；workspace 間 metadata 完全不可見（`SHOW DATABASES` / information_schema 看不到別的 ws）。
- **不允許跨 workspace 分享/存取資料**（與 lakehouse 習慣的關鍵差異）。
- Cluster 分配 tier（草案，門檻待修正）：
  | Tier | Data Volume | Max QPS | 配置 |
  |------|------------|---------|------|
  | 1 | < 500GB | < 20 | Shared cluster + limited resource |
  | 2 | 500GB – 5TB | 20 – 100 | Small dedicated（3 BEs，16C/64GB/3.84TB SSD×2） |
  | 3 | 5TB – 30TB | 100 – 500 | Medium dedicated（5–10 BEs，32C/128GB/3.84TB SSD×4） |

### Ingestion Pipeline
- 平台提供：batch 匯入 + streaming（CDC on Kafka）。
- 允許有能力/高客製需求的使用者**自行寫入**——但自寫入的資料管理與責任歸屬**尚未釐清**。

### 基礎建設
- 非自建機房（私有雲），服務位於 private network。
- 初期硬體 pool：512 cores / 1024GB memory / 3.84TB SSD × 16。
  - ⚠️ 待確認：core:memory 比為 1:2（GB），低於 Doris BE 一般建議（1:4 以上），需確認數字或規劃取捨。

### 時程
- 未定，無已知硬 deadline；第一個目標部門未定。

### 待決事項（Open Issues，候選 RFC 題目）
1. Tier 門檻：data volume 與 QPS 綁在同一 tier 不合理，考慮拆成儲存/運算兩條獨立軸，取較高者定 tier。
2. 自寫入（bypass 平台 pipeline）使用者的資料管理與責任歸屬。
3. Row filter / column filter（使用者已提出需求）：做在平台側還是使用者自理，未決。
4. 超過 tier 3（>30TB 或 >500 QPS）的使用者如何處理。
5. 敏感資料/PII 的平台責任範圍尚未明確定義。

## Decision Log
| 日期 | 事項 | 決議 | 決策者 |
|------|------|------|--------|
| 2026-07-12 | 文件產出順序 | 先內部 PRD，對外文件由 PRD 衍生 | uniray7 |

## 試點回饋（Phase 9）


## 發布紀錄（Phase 10）
