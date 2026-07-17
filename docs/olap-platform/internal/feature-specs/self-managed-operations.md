# Feature Spec：Self-managed 營運支援

> 來源：[PRD.md](../PRD.md) § 6.8
> 狀態：草稿（Phase 3）

## 定位

Self-managed 模式的平台支援**刻意限縮**為四件事：**日常 infra 相關告警**、**保護性 config**、**代操作（scale in/out）**、**HA（雙 cluster CCR 維運 + failover 介入協助，2026-07-14 釐清）**。除此之外資料管理責任全在使用者——這條責任邊界是 self-managed 對外文件的核心內容（對外文件深度要求：self-managed 重點講責任歸屬）。

四件事以外的任何支援需求（使用問題、新版本、新功能、建表諮詢、查詢效能優化）**不是平台義務**，一律走「**提 request 給 PM**」管道（見下方第 5 節）——由 PM 排優先權決定要不要做，平台不做即時支援承諾。

使用者另提出「**指定 table 的 row filtering / column masking**」需求——是否由平台承接**未決**（open issue #3，候選 RFC；與「cluster 內權限使用者自理」的邊界衝突見責任歸屬矩陣註記）。

> ⚠️ **2026-07-17 重大修訂進行中**：self-managed 的 DDL 治理與資料管理 R&R 已立 [RFC-003](../rfc/RFC-003-self-managed-ddl-governance-and-rr.md) 待 manager 決策。已定前提（07-17 改訂）：**DDL（database＋table，含 ALTER/DROP）一律走平台代建流程**——初期緊縮政策（先緊縮後放寬），事前攔截＋保護性 property 注入，使用者帳號不授予 DDL 權限；**儲存健康巡檢與封頂防線涵蓋全部表**（cluster 自保）。曾定案的 table 雙軌制（07-16）降為**放寬預案**（RFC-003 §9），dbt/高頻建表需求正式提出時再啟動。RFC-003 定案後，本 spec 的責任歸屬矩陣等條目將依決議改寫。

## 平台提供的內容

### 1. 開通時隨 cluster 交付（provision 階段，見 workspace-application）
- Dedicated cluster（tier 2+）+ 初始管理帳號（使用者自行管理後續帳號/權限）。
- **Monitor / Alert / Log** 三件套：cluster 級指標儀表板、告警通道（送 ws 聯絡人）、FE/BE log 查詢入口。⚠️ 具體堆疊（Prometheus/Grafana 或內部系統）實作時定。
- **保護性 config**：防止 cluster 被打壞的底線設定，使用者**不可修改**。範圍草案 ⚠️：
  | 類別 | 例 | 目的 |
  |------|----|------|
  | 記憶體保護 | BE 記憶體上限比例、query 記憶體上限預設 | 防 OOM 整機倒 |
  | 連線保護 | max_connections 上限 | 防連線耗盡 |
  | 磁碟保護 | 磁碟水位告警/寫入保護門檻 | 防寫爆 SSD |
  | 元資料保護 | FE 關鍵參數鎖定 | 防誤改導致 cluster 不可用 |

  保護性 config 清單須寫入對外文件（使用者要知道哪些設定碰不了、為什麼）。

### 2. 日常 infra 告警（常態營運，2026-07-14 決議：平台唯一的常態支援）
- **平台告警範圍＝infra/系統層**：節點宕機、磁碟水位、FE/BE 進程異常、cluster 不可用。
- **不含資料層**：查詢變慢、資料品質、schema 問題、匯入失敗——皆屬使用者自理範圍（責任歸屬明文）。
- 告警送 ws 聯絡人（主要/備援）；平台僅在**系統層異常**時介入處理（如換機、重啟節點）。
- **緊急介入權**（2026-07-16，依 RFC-003 穩定性框架）：偵測到危及 cluster 存活的行為（compaction 積壓將全面拒寫、磁碟逼近水位等）時，平台可**先處置後通知**（限寫、kill 查詢、極端時停用問題表）——條款寫入對外使用規範。

### 3. 代操作：scale in/out
- 使用者無機器操作權限，擴縮容由平台代操作。**目前平台無任何付費機制**（2026-07-14 決議），代操作不收費；若未來引入計費（待拍板 #1）再定成本歸屬。
- 申請方式 ⚠️：比照 workspace 申請走**週三會議**提出（規模變更本質是資源池調度，需人工排程）；緊急擴容另開綠色通道（定義與 SLA 待 Phase 5）。
- 執行流程：申請（目標 BE 數/規格）→ 管理層成本核准 → 排程執行（硬體不足進 WAITING_FOR_CAPACITY）→ 完成通知。
- Scale in 前使用者須自行確認資料可容納於縮容後空間；平台執行前做水位檢查，不足即拒絕。

### 4. HA：Active-Standby 雙 Cluster（2026-07-14 需求釐清）

使用者需求「希望平台提供 HA」，平台方案：

- **架構**：兩座 Doris cluster（active-standby）；平台以 **CCR（cross-cluster replication）盡量維持兩座同步**——**best-effort**，不承諾零資料丟失（RPO 量化與對外措辭 Phase 5）。
- **平台責任**：standby cluster 佈建、CCR 鏈路建立與維運、**sync lag 監控納入日常 infra 告警範圍**、**failover 時介入幫忙**（切換操作由平台執行）。
- **使用者責任**：failover 後的資料完整性驗證與應用側重連（gateway 統一端點是否延伸到 self-managed 以免換線 ⚠️ 待確認）。
- **CCR 為 cluster 層級全同步**（2026-07-14 確認）：使用者任意新建的 database/table **自動納入同步**，平台無需逐 db 掛載例行操作。
- **CCR lag 與使用者寫入行為連動**（2026-07-16 補充）：暴力灌資料會使 sync lag 暴增——lag 期間若發生 failover，資料落差即擴大。這是 **RPO 只能承諾 best-effort 的直接原因**；lag 告警需標明成因（鏈路問題屬平台責任、寫入量問題屬使用者責任），寫入量造成的 lag 由使用者降載處置。
- 待決：HA 是**預設**（所有 self-managed 皆雙 cluster）還是**選配**；standby 資源怎麼計（等同兩倍硬體，影響 tier 佈建與容量規劃）；failover 觸發條件與判定者、RTO 目標。

### 5. 其他需求：PM Request 管道（2026-07-14 決議）

三件事以外的需求**一律提 request 給 PM，由 PM 排優先權決定要不要做**，適用範圍包含（但不限於）：

| 需求類型 | 例 |
|----------|----|
| 使用上的問題 | 操作疑難、行為不符預期的排查協助 |
| 新版本 | 想升級 Doris 版本 |
| 新功能 | 想啟用/新增平台未提供的能力 |
| 建表諮詢（consultant） | schema 設計、分區/分桶建議 |
| 查詢效能優化 | 慢查詢分析與調校建議 |

- 流程：使用者提 request → PM 收斂進 backlog 排優先權 → 決定承接與否（**可以不做**）→ 承接者才由平台排程執行。
- 對外文件措辭重點：這是**需求提案管道，不是支援承諾**——沒有回應時效 SLA，與日常 infra 告警（平台義務）明確區隔。
- Request 的提交形式（工單/表單/會議）⚠️ 待確認。

| 事項 | 平台 | 使用者 |
|------|------|--------|
| 硬體/OS/Doris 進程存活 | ✅ | |
| 系統層監控與告警 | ✅ | |
| 保護性 config 維護 | ✅ | |
| Scale in/out 執行 | ✅（代操作） | 提出申請 |
| HA：standby 佈建、CCR 同步維運、failover 介入 | ✅（sync lag 屬 infra 告警） | failover 後資料完整性驗證、應用重連 |
| 指定 table 的 row filtering / column masking | ⚠️ 未決（open issue #3）——若承接，將是「cluster 內權限使用者自理」的唯一例外，且需解決使用者改表/刪表後的政策漂移責任 | 指定 table 與政策內容 |
| 使用問題/新版本/新功能/建表諮詢/效能優化 | 依 PM request 排優先權，**非義務** | 提 request 給 PM |
| Doris 版本升級 | 提 request → PM 排優先權決定；核准後平台執行 | 提出需求 |
| 資料寫入/pipeline | | ✅ |
| DDL 執行（database＋table，含 ALTER/DROP） | ✅ 代建：驗證（語法/危險操作/撞名）＋保護性 property 注入（RFC-003 前提） | 提交申請 |
| Schema 設計品質 | ❌ 不審不揹 | ✅（建表諮詢可提 PM request，不保證承接） |
| 備份/還原 | ❌ 不提供（RFC-003 子決策暫定甲：誤刪/誤操作一律不救援、明文免責；平台保存 DDL 申請紀錄 + Doris audit log〔規劃 ELK〕作舉證） | ✅ 自行負責 |
| 查詢效能調校 | | ✅（優化協助可提 PM request，不保證承接） |
| 帳號/權限管理（cluster 內） | | ✅ ⚠️（與「使用者帳號不授予 DDL 權限」的技術強制有衝突——擁有 GRANT 權即可自行開回 DDL；解法見 RFC-003 §8 隱患 3，傾向初始帳號不含 GRANT_PRIV、帳號管理收回平台） |
| 配額與用量申報 | | ✅（超標處理見 cluster-tiering） |

## 行為規格

| # | 情境 | 行為 |
|---|------|------|
| 1 | BE 節點宕機 | 平台告警 + 平台介入處理（系統層責任）；資料副本修復期間的查詢劣化不屬平台賠責範圍 ⚠️（SLA 措辭 Phase 5） |
| 2 | 使用者查詢效能劣化（系統層正常） | 平台不介入；使用者可提 request 給 PM 排優先權（不保證承接） |
| 3 | 使用者嘗試修改保護性 config | 無權限；如有正當需求走人工申請由平台評估 |
| 4 | 磁碟水位超過告警門檻 | 告警送使用者；持續逼近寫入保護門檻時平台可主動限制寫入（保護 cluster）並通知 |
| 5 | 使用者申請 scale out、硬體不足 | WAITING_FOR_CAPACITY + 預估時間（與 workspace 申請同機制） |
| 6 | 使用者把 cluster 打掛（誤設定/超載） | 平台恢復「系統可用」狀態（進程/節點層）；資料修復與根因排除由使用者自理，如需平台協助提 request 給 PM（不保證承接） |
| 7 | CCR sync lag 超門檻 | infra 告警送 ws 聯絡人 + 平台排查鏈路；**告警標明成因**——鏈路問題屬平台責任、寫入量超出 CCR 承載屬使用者責任（由使用者降載處置） |
| 8 | Active cluster 故障 | 平台介入執行 failover 至 standby（觸發條件/判定者待決）；failover 後資料完整性由使用者驗證，落差處理依 RPO 免責措辭（Phase 5） |
| 9 | 使用者需要新建/變更/刪除 database 或 table | 提交代建申請 → 平台驗證後執行（RFC-003 前提，初期人工；時效預期 Phase 5 定）；CCR cluster 層級同步自動涵蓋新物件（2026-07-14 確認），初次同步期間的 lag 屬正常行為 |
| 10 | 使用者要求 dbt 直連建表（高頻 create/drop） | 現階段不支援（全代建政策）；引導透過 PM 管道正式提出需求 → 觸發 RFC-003 §9 放寬預案討論 |

## 與其他功能的依賴關係
- 開通交付內容與 WAITING_FOR_CAPACITY 狀態依 **workspace-application**。
- Tier 變更（scale 到跨 tier 規格）依 **cluster-tiering** 的門檻與流程。
- 系統層指標同時進平台側容量管理（**quota-and-usage** 的平台視角）。
- 目前無付費機制；若未來引入計費（product-overview 待拍板 #1），代操作與 PM request 承接的成本歸屬屆時再定。

## 待確認事項
1. Scale 申請是否一律走週三會議，或另設輕量流程；緊急擴容綠色通道定義。
2. 保護性 config 具體清單（實作時隨 Doris 4.1.0 參數定案）。
3. PM request 的提交形式（工單/表單/會議）與 backlog 可視化方式。
4. HA 是預設還是選配；standby 資源計算方式（兩倍硬體對 tier 佈建與容量規劃的影響）。
5. Failover 觸發條件、判定者（平台單方 vs 與使用者確認）、RTO 目標；RPO 的 best-effort 對外措辭（Phase 5）。
6. Gateway 統一端點是否延伸到 self-managed（failover 不換線）。（~~CCR 同步層級~~ → 2026-07-14 確認：cluster 層級全同步，新建物件自動涵蓋）
7. 指定 table 的 row filtering / column masking 承接與否（open issue #3，候選 RFC）。

### 已銷案（2026-07-14 脈絡傾倒）
- ~~Doris 版本升級責任~~ → 使用者提 request 給 PM 排優先權，核准後平台執行。
- ~~付費選配範圍~~ → 目前無任何付費機制；諮詢/優化/協助類需求一律走 PM request 管道，非平台義務。

## 邊界審查（Phase 4 補上）
| Edge Case | 是否適用 | 處理方式 |
|-----------|---------|----------|
| （待 Phase 4） | | |

## 驗收標準（Acceptance Criteria）
- [ ] 責任歸屬矩陣經會簽後原文進對外文件（self-managed 文件的核心章節）
- [ ] 保護性 config 清單完成、實測使用者帳號不可修改
- [ ] 系統層告警可送達 ws 聯絡人，且資料層事件不誤發平台告警
- [ ] Scale in 的水位檢查有效（空間不足時拒絕執行）
- [ ] PM request 管道的「需求提案、非支援承諾」定位寫入對外文件，與 infra 告警（平台義務）明確區隔
