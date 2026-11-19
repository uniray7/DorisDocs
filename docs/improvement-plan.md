# Improvement Plan：交接 PM 前的文件整理

> 日期：2026-11-19｜狀態：草稿，待確認｜範圍：`docs/olap-platform/` 全部文件 + `/olap-doc-workflow` skill
> 規則：每一批修改先給 diff 看過，確認後才 commit/push。

## 0. 已確認的方向

| 項目 | 決定 |
|------|------|
| 決策演進史 | 正文只寫**現況**；歷史（含被推翻的決議）集中到新檔 `decision-history.md` |
| 英文版 | 刪除 `product-overview.en.md`，只維護中文版 |
| 專有名詞 | 不硬翻，直接用英文（寫進 skill 規範） |
| 未研究主題 | 允許標 **TBD**，集中列清單，交給 PM／工程師後續研究 |

---

## 1. 邏輯矛盾清單

「處理」欄：**直接修**＝已有決議、只是文件沒同步；**需決定**＝要一起 review（見 §2）。

| # | 矛盾 | 出處 | 處理 |
|---|------|------|------|
| C1 | Self-managed 寫「自行下 DDL、平台不負責任」，但 07-17 已決議 **DDL 一律平台代建** | `00-progress.md` 審查分軌段、申請流程段；`database-table-application.md` 定位段 | 直接修 |
| C2 | Self-managed「資料管理責任全在使用者」，但 RFC-003 Option 3 讓平台執行 TTL 清理、巡檢、物件清冊——平台實際承擔了部分資料管理 | PRD、product-overview 模式對照表 vs RFC-003 §4/§6 | 需決定（措辭依 RFC-003 結論） |
| C3 | Decision Log 寫 managed 四大責任「含 schema 品質把關」、「raw 建表人工嚴審」，但之後改成方案 A/B 並列待決 | `00-progress.md` Decision Log 07-13 兩列 vs open issue #7 | 直接修（舊列移入 decision-history） |
| C4 | Self-managed「成本核准即通過 → 開 cluster」，但 staging spec 說「兩種模式都要過 staging」，申請狀態機也一律經 staging | `00-progress.md`、product-overview vs `staging-validation.md`、`workspace-application.md` | 需決定 |
| C5 | Staging spec 自相矛盾：「依目標 tier 佈建」vs「固定小規格不隨 tier」；User Story「量化報告界定效能責任」「與申報量級相符」vs「效能僅供參考」「不要求量級相符」 | `staging-validation.md` | 直接修 |
| C6 | Self-managed 平台支援範圍說法不一：PRD 三項、ops spec 開頭「四件事」後文又「三件事以外」；實際還有 DDL 代建、緊急介入權 | PRD §5、`self-managed-operations.md` | 直接修（統一成一張清單） |
| C7 | HA 適用範圍：product-overview 說「兩種模式皆適用」，其他文件只定義 self-managed 的 HA；managed HA 從未定義 | product-overview §五 vs `self-managed-operations.md` | 需決定 |
| C8 | 決策簡報寫 HA「秒級同步」，其他文件寫 best-effort、lag 可能暴增 | `RFC-003-decision-brief.md` | 直接修（改措辭） |
| C9 | tmp 的結構三種說法：佈建時建 `{ws}__{x}__tmp` **database**；streaming/batch 建 pipeline 時建 `{ws}__{pipeline}__tmp` **table**（用 database 命名格式命名 table，pipeline 與 batch job 同名會撞）；DDL 申請可建 database 三件組（含 tmp），但又說 tmp 不開放申請 | `workspace-application.md`、`streaming-ingestion.md`、`batch-ingestion.md`、`database-table-application.md` | 需決定（可列 TBD 給工程師） |
| C10 | 「申請流程純人工、無 API」vs 多份 spec 有完整 REST API／web console、「系統自動建議 tier」；staging 驗收寫「自動通知」「提交被擋下」 | `cluster-tiering.md`、`staging-validation.md` 等 | 需決定（初期哪些人工、哪些有系統） |
| C11 | 計費用語殘留：錯誤碼「成本中心未確認**新費用**」、self-managed 擴容「**成本反映**」，與「無付費機制」衝突 | `cluster-tiering.md` | 直接修（改「成本歸屬確認」） |
| C12 | 待決事項有**三套編號**互相引用（00-progress #1–9、PRD §11 #1–8、product-overview 待拍板 #1–9）；PRD 引用不存在的 **RFC-001** | 全部文件 | 直接修（統一成一份 TBD 清單、單一編號） |
| C13 | Self-managed「帳號/權限自理（含 GRANT）」vs「使用者帳號不授予 DDL 權限」 | `access-control.md`、`self-managed-operations.md`（RFC-003 §8 已自承） | 列 TBD（待技術驗證） |
| C14 | Self-managed「命名自由、不分 zone」vs RFC-003 Option 1/2 會引入固定庫或 zone 命名 | `workspace-application.md`、`workspace-isolation.md`、Decision Log vs RFC-003 | 依 RFC-003 結論回填，先標條件式 |
| C15 | Streaming 驗收標準寫「無法寫入 tmp/curated 以外…」，與「merge SQL 只能讀 tmp、寫 raw」不符 | `streaming-ingestion.md` 驗收標準 | 直接修 |
| C16 | Self-managed 的 console 功能清單漏了「DDL 代建提交」 | `access-control.md` Self-managed 角色段 | 直接修 |
| C17 | Journey 文件仍把「欠費」列為待修斷點，但已修完 | `managed-service-journey.md` 斷點 1 | 直接修 |
| C18 | 狀態標示過期：PRD 標「大綱版 Phase 2」但內容已細化；product-overview 日期 07-13 但內容到 07-17 | PRD、product-overview 標頭 | 直接修 |

---

## 2. 需要一起 review 的事項

| # | 議題 | 關聯 | 我的建議 |
|---|------|------|----------|
| R0 | **7/18 之後有沒有新決議？**（文件停在 7/18，現在 11/19）例如 RFC-003、方案 A/B 是否已拍板。已知新資料：Authorization PRD v0.12（11/16），衝突見 §7 | 全部 | 先處理 §7，再做其他批次 |
| R1 | Self-managed 要不要過 staging | C4 | 不強制（self-managed 不審技術內容，staging 對它意義小），改為選配 |
| R2 | HA 是否適用 managed；預設或選配 | C7 | 先寫「managed HA：TBD」，不要在 overview 先承諾 |
| R3 | tmp 是 database 還是 table、命名規則 | C9 | 列 TBD 給工程師；文件先統一成一種說法 |
| R4 | 初期哪些作業人工、哪些有 console/API | C10 | spec 裡的 API 段落改標「目標設計（實作時程 TBD）」，不刪 |
| R5 | 80% 用量預警由誰計算 | journey 斷點 2 | quota-and-usage 計算、cluster-tiering 只收事件 |
| R6 | 佈建時建的初始 raw 表要不要補稽核紀錄 | journey 提醒 1 | 要，回寫一筆 DDL 申請紀錄 |
| R7 | Self-managed DDL 代建「初期人工」走什麼管道、時效多久 | RFC-003 §8 | 列 TBD（PM 定管道，工程師估時效） |
| R8 | `managed-service-journey.md` 的去留 | §3 | 斷點處理完後併入 PRD，刪檔 |

---

## 3. 刪減與精簡

| 對象 | 動作 | 預估效果 |
|------|------|----------|
| `product-overview.en.md` | 刪除 | −100 行 |
| `00-progress.md` Phase 1 脈絡摘要（約 150 行） | 改成「現況摘要」；歷史移到 `decision-history.md` | 219 → 約 80 行 |
| 全部文件的「（2026-07-1x 補充／修訂）」註記、刪除線、「已銷案」段 | 刪除（歷史在 decision-history 與 git log） | 可讀性大幅提升 |
| 重複的表格：tier 表有 4 份、模式對照表有 3 份 | 只留一份（tier → `cluster-tiering.md`；模式對照 → PRD），其他改連結 | 消除不一致來源 |
| 各 spec 空白的「邊界審查（Phase 4 補上）」表格 | 改成一行 `TBD（Phase 4）` | 每份 −5 行 |
| `managed-service-journey.md` | 對話式語氣（「你」「我會直接修」）不適合交接；見 R8 | −96 行 |
| `RFC-003`（234 行）＋決策簡報（86 行） | 內容重複度高：簡報保留給決策者；RFC-003 §8–9 精簡 | RFC-003 約 −60 行 |

---

## 4. TBD 清單（交 PM／工程師研究）

> 整理後會放進 `00-progress.md`，作為**唯一**的待決清單，取代現有三套編號。

| # | 主題 | 建議負責 | 原出處 |
|---|------|----------|--------|
| T1 | Tier 門檻、儲存/運算解耦（RFC 尚未撰寫） | PM＋工程師 | open #1 |
| T2 | 超過 tier 3 的需求如何承接 | PM | open #4 |
| T3 | 計費模式（chargeback/showback/免費） | PM／管理層 | 待拍板 #1 |
| T4 | 超標但不願升級的處置（限流 vs 擋寫） | PM | 待拍板 #4 |
| T5 | Row/column filter 是否由平台提供 | PM＋工程師 | open #3 |
| T6 | PII／機敏資料的平台責任範圍 | PM＋Security | open #5 |
| T7 | Managed DDL 治理：方案 A／B | PM／管理層 | open #7 |
| T8 | Self-managed DDL 治理形態 Option 1/2/3、備份甲／乙 | Manager | RFC-003 |
| T9 | HA：預設或選配、standby 資源計算、RPO/RTO、failover 觸發與判定 | PM＋工程師 | self-managed-operations |
| T10 | Arrow Flight 能否經 gateway、QPS 怎麼計量 | 工程師 | RFC-002 |
| T11 | 初始帳號不含 GRANT_PRIV 的可行性 | 工程師 | RFC-003 §11 |
| T12 | Batch ingestion 的 6 項假設（來源、格式、觸發、錯誤門檻等） | PM（訪談使用者）＋工程師 | batch-ingestion |
| T13 | tmp 結構與命名（C9） | 工程師 | 多份 spec |
| T14 | Merge SQL 排程機制（平台排程器 vs Doris JOB） | 工程師 | streaming-ingestion |
| T15 | NFR 數字：SLA、各類 retention、配額值、refresh 週期、審核時效 | PM＋工程師 | Phase 5 |
| T16 | 平台身分是否接 SSO/LDAP | 工程師 | access-control |
| T17 | 保護性 config 清單、FE 封頂參數 | 工程師 | self-managed-operations、RFC-003 |
| T18 | Metadata 隔離 3 項驗證（information_schema、PROCESSLIST、錯誤訊息） | 工程師 | workspace-isolation |
| T19 | 儲存配額計單副本或全副本 | PM＋工程師 | quota-and-usage |
| T20 | 時程與首個目標部門 | PM | PRD §10 |

---

## 5. Workflow（`/olap-doc-workflow` skill）修改

1. **TBD 機制**：統一格式 `TBD｜待研究：…｜建議負責：PM／工程師`；所有 TBD 同步列到 `00-progress.md` 的 TBD 清單。
2. **允許帶著 TBD 前進**：不必全部定案才能進下一階段。
2a. **12 個 Phase 精簡為 5 個階段**（定義／Feature 細化／NFR 與治理／定案／產出與發布），進度改為按文件追蹤狀態（草稿／評審中／定案）；階段 3–5 可不照順序、分頭進行。
3. **專有名詞不硬翻**：workspace、tier、managed／self-managed、pipeline、staging、quota、DDL、CCR、workload group、zone（tmp/raw/curated）等直接用英文。
4. **只寫現況**：正文不標「某日補充」、不留刪除線；決議變更記在 `decision-history.md`。
5. **單一事實來源**：同一張表或同一個定義只放一處，其他文件用連結。
6. **精簡優先**：讀者是 PM 與工程師，能用表格就不寫段落。
7. 模板同步調整：`progress-template.md` 加 TBD 清單；`feature-spec-template.md` 的邊界審查改為選填。
8. 修正 SKILL.md 中混入的簡體字。

---

## 6. 執行順序（每批先 diff、確認後才 commit／push）

| 步驟 | 內容 | 為什麼放這裡 |
|------|------|-------------|
| 0 | **討論文件架構與要涵蓋的 topic**：讀者分層、目錄組織、topic 清單、每個 topic 對應的主文件（✅ 已定案，見 §8） | 後續每一步都依賴「哪個 topic 放哪份文件」 |
| 1 | Workflow：SKILL.md、模板、README（加 PM 交接入口） | 依步驟 0 的架構寫規範 |
| 2a | 依 §8 搬移目錄與改名（`git mv`）、更新內部連結，內容不動；順帶刪除英文版 `product-overview.en.md` | 合併內容要寫進新路徑，先搬再改，diff 只有路徑變化 |
| 2b | 合併 Authorization PRD v0.12：依 §7.4 確認結果併入對應文件；新增 authorization、ha-dr、monitoring-alerting、web-console 骨架 | 會推翻部分舊決議，要在精簡前做，避免同一份文件改兩次 |
| 3 | 新增 `decision-history.md`；精簡 `00-progress.md`；建立統一 TBD 清單（英文版已於 2a 刪除） | 合併後被推翻的舊決議一起移入 decision-history |
| 4 | 修正 §1 中標「直接修」的矛盾；各文件瘦身（重複表格、空白段落、日期註記） | 此時內容已是最新 |
| 5 | 回填 §2 review 的結論 | |

---

## 7. 合併 Authorization PRD v0.12（2026-11-16）

### 7.1 與現有文件的衝突

| # | 主題 | 現有文件（≤ 7/18） | Authorization PRD v0.12 | 影響文件 |
|---|------|------------------|------------------------|----------|
| A1 | **跨 workspace 分享** | 明確**不支援**（已定案、列 Out-of-scope） | Owner 可把自己的 table 分享 SELECT 給**其他 workspace** 的 reader | PRD、workspace-isolation、product-overview、對外溝通 |
| A2 | **Fab 維度** | 沒有 fab 概念；一個 workspace 對應一座 cluster | 十幾座 fab、20–40 座 cluster；每個 workspace **至少橫跨兩個 fab**；各 fab 各自指定 shared／dedicated | cluster-tiering、workspace-application、quota-and-usage、HA |
| A3 | **申請流程** | 純人工：投影片＋週三會議，無系統 | Console 申請頁、平台審核頁、開通狀態（含「部分完成」與重試） | workspace-application、staging-validation |
| A4 | **Managed curated 的 DDL** | 使用者沒有任何直接 DDL（含 curated）；開放 curated 自建已擱置 | Owner 可直接 CREATE／DROP／TRUNCATE TABLE（ALTER 待確認） | database-table-application、access-control、RFC-003 §9 |
| A5 | **Managed DDL 方案 A／B** | 兩案並列待決 | Raw table 由平台建、curated 由 owner 建——實質上取代 A／B？raw 建表要不要審核沒寫 | database-table-application、TBD T7 |
| A6 | **Zone** | tmp／raw／curated 三層 | 只出現 raw／curated；tmp 沒提 | 各 ingestion spec、TBD T13 |
| A7 | **Workspace 命名** | 只允許小寫字母＋數字，禁止 `_` | 允許英數字之間的單底線（例：`domain_product`） | workspace-application、PRD 術語表 |
| A8 | **Self-managed DDL／dbt** | RFC-003：table DDL 一律平台代建，dbt 等需求出現再談放寬 | 已在討論 dbt 的 CREATE／DROP 範圍（A：所有庫／B：指定輸出庫）——等於放寬預案已啟動 | RFC-003、self-managed-operations |
| A9 | **帳號管理** | Self-managed 拿管理帳號後自理（與「不授予 DDL」衝突） | Doris user、role、密碼**全由平台管理** | access-control、self-managed-operations（可解 C13） |
| A10 | **角色** | Owner（建議 2 位）、Member、Service Account | Owner（＝申請人＝data owner）、Data reader；沒有 Member、Service Account | access-control |
| A11 | **Doris 版本** | 4.1.0 | 4.1.4 | 全部 |
| A12 | **Row／column filter** | 待決（open #3） | 長期方案：以 OPA 外掛做 row filter／column mask，待 PoC | access-control、TBD T5 |
| A13 | **部署與組織** | 私有雲、硬體池浮動 | HashiCorp Nomad；Tool Team（隸屬資料平台團隊）負責 Web Console 與授權後端 | 00-progress 基礎建設段、PRD |

### 7.2 新增的 TBD（來自 v0.12 自己列的待決）

| # | 主題 | 建議負責 |
|---|------|----------|
| T21 | 同一 workspace 在同一 fab 能否使用多座 cluster | 主管 |
| T22 | Managed curated 是否開放 ALTER TABLE | PM＋工程師 |
| T23 | Self-managed TRUNCATE／DROP 的執行身分與範圍；dbt 範圍 A／B | 主管＋工程師 |
| T24 | 公司識別碼對應 Doris 帳號、首次登入憑證交付、owner 離職／轉調接手 | 工程師＋PM |
| T25 | 是否支援整庫分享（含未來新表）、限時分享 | PM |
| T26 | 命名白名單細節（user_defined 可否數字開頭等） | 工程師 |
| T27 | OPA 外掛 PoC（4.1.4 相容性、兩種協定一致性、revoke 生效來源） | 工程師 |
| T28 | tmp 取消後的 ingestion 流程：CDC／batch 資料如何進 raw、parse／merge 由誰做 | 工程師＋PM |
| T29 | Self-managed 開放 dbt 直接 DDL 的風險與防護（FE metadata、partition／tablet 上限、回收站、DDL 頻率） | 工程師＋主管 |
| T30 | Managed curated 開放 service account CREATE／DROP TABLE 是否安全（同 T29 的 FE metadata 風險；shared cluster 上 FE 為多租戶共用，影響範圍更大） | PM＋工程師 |

### 7.3 合併方式（建議）

- v0.12 轉成 markdown，新增為 `internal/prd/authorization.md`，保留原版號與評審狀態。
- `access-control.md` 與 v0.12 重疊最多：授權相關內容以 v0.12 為準，`access-control.md` 精簡後改連結過去。
- A1–A13 依確認結果同步修正其他文件；被推翻的舊決議記到 `decision-history.md`。

### 7.4 逐項確認紀錄（步驟 2）

| # | 決定 | 確認日期 |
|---|------|----------|
| A11 | 全部文件改為 Doris **4.1.4** | 2026-11-19 |
| A13 | 以 v0.12 為準：HashiCorp Nomad、Tool Team 負責 Web Console 與授權後端；**刪除「私有雲」「硬體池浮動」的描述** | 2026-11-19 |
| A9 | 採用 v0.12：Doris user、role、密碼全由平台管理（兩種模式皆同），C13 隨之解決 | 2026-11-19 |
| A7 | 採用 v0.12：workspace 名稱允許英數字之間的單底線，`__` 保留為分隔符；細節見 T26 | 2026-11-19 |
| A1 | **列 TBD**：跨 workspace 分享（v0.12 的 owner 分享 SELECT 給其他 workspace reader）方向未定，現有「不支援」與 v0.12 說法都不寫死 | 2026-11-19 |
| A2 | 採用 fab 維度，**與 HA 無關**：workspace 申請時指定要在哪些 fab 使用服務，平台在每個 fab 各自提供**相同數量、彼此獨立**的 Doris 服務，**不跨 fab 部署** | 2026-11-19 |
| A3 | **維持純人工申請**（投影片＋週三申請會議＋人工追蹤表）；v0.12 的 Console 申請／審核頁不作為現況 | 2026-11-19 |
| A4 | 採用 v0.12 方向：curated database 可直接 CREATE／DROP／TRUNCATE TABLE（執行身分為 service account，見 A10；是否安全見 T30），ALTER 見 T22；7/17「curated 自建擱置」決議移入 decision-history | 2026-11-19 |
| A3 補充 | v0.12 的 Console 申請頁／審核頁／開通狀態**不寫入**文件 | 2026-11-19 |
| A5 | Managed DDL 方案 A／B 縮小為一個 TBD：**raw 由平台建，建表前要不要平台審核 schema**（取代 T7） | 2026-11-19 |
| A6 | **tmp zone 取消**，managed 只剩 raw／curated。連帶影響：streaming／batch 原本「CDC → tmp → 使用者 merge SQL → raw」的流程要改寫（新 TBD：T28） | 2026-11-19 |
| A8 | **未定，需團隊討論**：是否讓 self-managed 使用者因 dbt 直接 CREATE／DROP table（不經平台）。主要擔憂：過多／過碎的 partition 造成 FE memory 壓力甚至 cluster crash；需研究防護（新 TBD：T29）。**不另開 RFC**：風險與可研究的防護寫在 `docs/internal/prd/ddl-governance.md` | 2026-11-19 |
| A10 | **資料存取一律透過 service account**：workspace 內不開放個人帳號直接存取 table，所有存取都向平台申請 service account。權限：raw database **只能讀**；curated database 可 **CREATE／INSERT／DROP TABLE**（Doris 的 TRUNCATE 與 DROP 同屬 DROP_PRIV）。curated 開放建表列 TBD（T30），請接手 PM 依 A8 風險重新評估 | 2026-11-19 |
| A12 | Row／column filter 維持待決（T5），補上 v0.12 的長期做法：以 OPA 外掛做 row filter／column mask，待 PoC（T27） | 2026-11-19 |

---

## 8. 目標文件架構（步驟 0 結論，2026-11-19）

### 8.1 目錄結構

拿掉 `olap-platform` 這一層，repo 只放這個平台的文件。

```
README.md                         交接入口：讀者導覽（不放內容）
docs/
  00-progress.md                  現況摘要 + 統一 TBD 清單 + Phase 狀態
  decision-history.md             決議演進（新增）
  product-overview.md             管理層一頁版
  improvement-plan.md             本次整理計畫（完成後可刪）
  internal/
    prd/                          各 feature 的細節
      README.md                   ← 原 PRD.md：範圍、術語、模式對照表、feature 索引
      workspace-application.md
      staging-validation.md
      cluster-topology-and-tiering.md   ← 原 cluster-tiering，加入 fab／cluster 拓撲
      workspace-isolation.md
      authorization.md            ← Authorization PRD v0.12
      access-control.md           ← 精簡：只留查詢協定，授權改連結 authorization.md
      ddl-governance.md           ← 原 database-table-application，涵蓋兩種模式的 DDL
      streaming-ingestion.md
      batch-ingestion.md
      quota-and-usage.md
      self-managed-operations.md
      ha-dr.md                    ← 新增骨架
      monitoring-alerting.md      ← 新增骨架
      web-console.md              ← 新增骨架（Tool Team 的 Console 總覽）
    rfc/
      RFC-002-shared-cluster-resource-isolation.md
      RFC-003-self-managed-ddl-governance-and-rr.md
      RFC-003-decision-brief.md
    architecture/                 Phase 8
      system-architecture.md
      data-flow.md                ← managed-service-journey.md 併入
  external/                       Phase 8（對外文件）
```

### 8.2 讀者路徑

| 讀者 | 起點 | 接著讀 |
|------|------|--------|
| 管理層 | `docs/product-overview.md` | `internal/rfc/` 的決策簡報 |
| PM | `README.md` | `docs/00-progress.md`（現況＋TBD）→ `internal/prd/README.md` |
| 工程師 | `internal/prd/README.md` | 各 feature 文件 → `internal/rfc/` → `internal/architecture/` |
| 對外使用者 | `docs/external/` | — |

### 8.3 Topic → 主文件

| Topic | 主文件 | 狀態 |
|-------|--------|------|
| 產品定位、範圍、術語、模式對照 | `prd/README.md` | 現有（收斂重複表格） |
| Fab／cluster 拓撲、tier | `prd/cluster-topology-and-tiering.md` | 改名＋擴充 |
| 申請與開通 | `prd/workspace-application.md` | 現有 |
| Staging | `prd/staging-validation.md` | 現有 |
| 隔離 | `prd/workspace-isolation.md` | 現有 |
| 授權與帳號 | `prd/authorization.md` | 新增（v0.12） |
| 查詢協定 | `prd/access-control.md` | 精簡 |
| DDL 治理（兩種模式） | `prd/ddl-governance.md` | 改名＋整合 |
| 資料寫入 | `prd/streaming-ingestion.md`、`prd/batch-ingestion.md` | 現有 |
| 配額與用量 | `prd/quota-and-usage.md` | 現有 |
| HA／DR | `prd/ha-dr.md` | 新增骨架 |
| 監控告警 | `prd/monitoring-alerting.md` | 新增骨架 |
| Web Console | `prd/web-console.md` | 新增骨架 |
| Self-managed 營運支援 | `prd/self-managed-operations.md` | 現有 |
| 治理合規／PII | `prd/README.md` 治理章節 | TBD（Phase 6） |
| 系統架構、data flow | `architecture/` | TBD（Phase 8） |
| 對外文件 | `external/` | TBD（Phase 8） |

### 8.4 連帶影響

- 步驟 1 的 workflow skill 路徑規範由 `docs/<slug>/…` 改為本結構。
- 搬移與改名在步驟 3 執行（`git mv`，保留檔案歷史），同時更新所有內部連結。
