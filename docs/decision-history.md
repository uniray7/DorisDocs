# 決議演進

> 記錄每一筆決議與後續變更。正文文件只寫現況；想知道「為什麼改過」來這裡查。
> 狀態：**有效**＝目前仍適用；**已取代**＝被後面的決議推翻或修改；**待決**＝已提案、尚未拍板（對應 TBD）。

## 2026-11-19：交接前整理（合併 Authorization PRD v0.12）

| 主題 | 決議 | 取代了什麼（原因） | 狀態 | 決策者 |
|------|------|-------------------|------|--------|
| Doris 版本 | 4.1.4 | 4.1.0（v0.12 為較新資料） | 有效 | uniray7 |
| 部署與組織 | Doris 跑在 HashiCorp Nomad（目前 1.11.1，因 sticky 相關 bug 評估升級 1.11.3，TBD-37）；Tool Team（隸屬資料平台團隊）負責 Web Console 與授權後端 | 「私有雲、硬體池浮動」的描述（不符現況） | 有效 | uniray7 |
| Fab 維度 | Workspace 申請時指定要在哪些 fab 使用服務（至少兩個）；每個 fab 各自提供相同數量、彼此獨立的 Doris 服務，不跨 fab 部署；與 HA 無關 | — （新增維度） | 有效 | uniray7 |
| Workspace 命名 | 允許英數字之間的單底線（例 `domain_product`），`__` 保留為分隔符 | 07-13：ws 名稱只允許小寫字母＋數字 | 有效（細節 TBD-26） | uniray7 |
| Zone | tmp zone 取消，managed 只剩 raw／curated | 07-13：tmp／raw／curated 三層 | 有效（ingestion 流程待改寫 TBD-28） | uniray7 |
| 帳號管理 | Doris user、role、密碼全由平台管理（兩種模式） | 07-14 前提：self-managed 拿管理帳號後自理（與「不授予 DDL」衝突） | 有效 | uniray7 |
| 資料存取 | Workspace 內不開放個人帳號直接存取 table，一律向平台申請 service account；raw 只能讀，curated 可 CREATE／INSERT／DROP TABLE | v0.12 以 owner、reader 人員帳號描述 | 有效（curated 建表安全性 TBD-30） | uniray7 |
| Managed curated DDL | curated 可直接 CREATE／DROP／TRUNCATE TABLE（執行身分為 service account） | 07-17：curated 開放自建擱置 | 有效（TBD-30、ALTER 見 TBD-22） | uniray7 |
| Managed raw DDL | raw table 由平台建立 | 07-13：DDL 治理方案 A（owner approve）／B（平台審核）並列 | 有效（建表前是否審核 TBD-7） | uniray7 |
| 申請流程 | 申請與審核維持人工；核准後由平台透過 Web Console 的 Admin 後台執行開通 | v0.12 的 Console 申請頁、審核頁、開通狀態（不採用） | 有效（後台範圍 TBD-34） | uniray7 |
| 跨 workspace 分享 | 方向未定 | 07-13：明確不支援；v0.12：owner 可分享給其他 workspace | 待決（TBD-31） | uniray7 |
| Self-managed dbt 直接 DDL | 需團隊討論；擔憂過多或過碎的 partition 造成 FE 記憶體壓力甚至 cluster crash，風險與防護寫在 `prd/ddl-governance.md` | 07-17：dbt 需求正式提出才啟動放寬預案 | 待決（TBD-29） | uniray7 |
| Row／column filter | 長期方向：以 OPA 外掛實作，待 PoC | — | 待決（TBD-5、TBD-27） | uniray7 |
| HA | Cluster 部署在多台機器、每個 table 3 個 replica，單一機器故障可容錯；中長期分散到 3 個機櫃或 3 個 datacenter | 07-14：self-managed 雙 cluster active-standby＋CCR＋failover | 有效（中長期做法 TBD-9） | uniray7 |
| DR | 未來需要 backup／restore 到 S3 或 MinIO | — | 有效（設計 TBD-36） | uniray7 |
| 資料搬遷 | 獨立成 `prd/data-migration.md`；CCR 從 tier 遷移流程拿掉，改列為候選做法之一 | cluster-tiering：tier 1→2「優先評估 CCR／backup-restore」 | 有效（做法 TBD-35） | uniray7 |

## 2026-07-12 ～ 07-17：PRD 細化期

| 日期 | 主題 | 決議 | 取代了什麼（原因） | 狀態 | 決策者 |
|------|------|------|-------------------|------|--------|
| 07-12 | 文件產出順序 | 先內部 PRD，對外文件由 PRD 衍生 | — | 有效 | uniray7 |
| 07-13 | Workspace 粒度 | 團隊自定（團隊或部門皆可），每個 ws 綁定一個成本中心 | — | 有效 | uniray7 |
| 07-13 | Data volume 計量 | 審核用來源原始大小估 tier；上線後以 Doris 壓縮後實際占用計配額 | — | 有效 | uniray7 |
| 07-13 | Max QPS 計量 | 5 分鐘滑動窗口平均判超標；瞬間爆量由併發限制兜底 | — | 有效 | uniray7 |
| 07-13 | 服務模式 | 提供 managed 與 self-managed，責任邊界以模式劃分 | — | 有效 | uniray7 |
| 07-13 | Self-managed × tier | Self-managed 限定 dedicated cluster（tier 2 起） | — | 有效 | uniray7 |
| 07-13 | 模式轉換 | 不可事後轉換；要換模式需開新 ws 搬資料 | — | 有效 | uniray7 |
| 07-13 | Managed 資料管理責任 | 備份還原、保留與清除、效能調校、schema 品質把關四項 | — | 已取代：schema 把關改為待決（07-13 方案 A／B → 11-19 TBD-7） | uniray7 |
| 07-13 | Pipeline 開放範圍 | 平台 ingestion pipeline 僅限 managed；self-managed 一律自寫 | — | 有效 | uniray7 |
| 07-13 | 寫入通道 | 由服務模式決定、與 cluster 類型無關；不採 hybrid | — | 有效 | uniray7 |
| 07-13 | Zone 模型範圍 | tmp／raw／curated 與 raw 建表審核僅適用 managed | — | 已取代：11-19 tmp 取消、raw 審核改為 TBD-7 | uniray7 |
| 07-13 | DDL 審核精緻化 | Raw 建表人工嚴審；curated 免審；tmp 由平台管理 | — | 已取代：同日改為方案 A／B 並列，11-19 再改 | uniray7 |
| 07-13 | Database 命名 | `{ws}__{user_defined}__{tmp\|raw\|curated}`，雙底線分隔、zone 置尾 | 先前的 `{ws}_{zone}_{userdefined}`（Doris 不允許 `-`，單底線會歧義） | 已取代：11-19 zone 只剩 raw／curated、ws 允許單底線 | uniray7 |
| 07-13 | Staging 定位 | 固定小規格試用環境；效能數據僅供參考，審核重點為設計合理性 | — | 有效 | uniray7 |
| 07-13 | Staging PII | Lakehouse 真實資料進 staging 的 PII 風險經評估忽略 | — | 有效 | uniray7 |
| 07-13 | 命名自由度 | 命名自由僅限 self-managed；managed 一律遵循 zone 命名 | — | 已取代：self-managed 命名改為待決（RFC-003，TBD-8） | uniray7 |
| 07-13 | Streaming format | 固定三種：generic CDC、DB2 JSON CDC、DB2 XML CDC | — | 有效 | uniray7 |
| 07-13 | 自定義 schema 模式 | 取消，被三種固定 format 取代；統一落 tmp 再由使用者 SQL 進 raw | — | 部分取代：「落 tmp」因 11-19 tmp 取消待改寫（TBD-28） | uniray7 |
| 07-13 | Corrupted data | Format 驗證失敗的資料稱 corrupted data，存 MinIO per-ws 路徑 | — | 有效 | uniray7 |
| 07-13 | 申請流程純人工 | 投影片 template＋每週三申請會議＋人工追蹤表，無自動化 API | — | 有效（11-19 補充：開通由 Admin 後台執行） | uniray7 |
| 07-13 | 審查分軌 | Self-managed：管理層成本核准即通過；managed：審查定 shared／dedicated、須申報 ws 整體 data size | — | 有效（其中「self-managed 開通後改表不經平台」已被 07-16 取代） | uniray7 |
| 07-13 | Managed DDL 治理 | 方案 A（ws owner approve）／方案 B（平台審核）並列待決 | 同日「DDL 審核精緻化」 | 已取代：11-19 raw 由平台建、curated 由 service account 建 | uniray7 |
| 07-14 | Self-managed 支援邊界 | 平台常態只負責 infra 告警；其他需求提 request 給 PM 排優先權，非義務 | — | 有效 | uniray7 |
| 07-14 | 無付費機制 | 平台目前沒有任何付費機制，代操作與諮詢不收費 | 先前的付費選配說法 | 有效 | uniray7 |
| 07-14 | 查詢協定 | MySQL protocol（完整 SQL）＋ Arrow Flight（僅 SELECT 類） | — | 有效 | uniray7 |
| 07-14 | Self-managed HA | 兩座 cluster active-standby＋CCR best-effort 同步＋failover 時平台介入 | — | 已取代：11-19 改為 3 replica | uniray7 |
| 07-14 | Row／column filter | Self-managed 使用者要求對指定 table 做 row filtering／column masking；承接與否未決 | — | 待決（TBD-5） | uniray7 |
| 07-14 | CCR 同步層級 | CCR 為 cluster 層級全同步 | — | 已取代：11-19 不採用 CCR 做 HA | uniray7 |
| 07-16 | Self-managed DDL 代建 | Database 與 table DDL 一律「提交 → 驗證 → 平台代建」並注入保護性設定，不驗 schema 設計 | 07-13：self-managed 開通後改表不經平台 | 有效 | uniray7 |
| 07-16 | Self-managed 治理形態 | 三選項（固定三庫／命名規則／表單宣告制）交 manager 決策，團隊建議表單宣告制 | — | 待決（TBD-8） | uniray7（待 manager） |
| 07-16 | Self-managed 備份還原 | 暫定不提供；誤刪不救援、明文免責，以 DDL 申請紀錄＋audit log 舉證 | — | 待決（TBD-8；DR 方向見 11-19） | uniray7（待 manager） |
| 07-16 | Table DDL 直接執行 | 因 dbt 需求，table DDL 改為直接打 Doris，只有 database 代建 | 同日「DDL 代建」的 table 部分 | 已取代：同日改雙軌制 | uniray7 |
| 07-16 | Table DDL 雙軌制 | 平台代建（有保護、有責任）與自建（零責任）並存 | 同日「Table DDL 直接執行」 | 已取代：07-17 改回全代建，雙軌制降為放寬預案 | uniray7 |
| 07-16 | Cluster 穩定性責任框架 | 四層防線：代建攔截、上限封頂、巡檢偵測、恢復＋緊急介入權；承諾「弄不沉、劣化會叫、沉了會救、誰弄的有證據」，不保證不變慢 | — | 有效（CCR lag 相關描述已隨 11-19 移除） | uniray7 |
| 07-17 | Table DDL 全代建 | 一律平台代建（先緊縮後放寬）；放寬是送禮、收回是毀約 | 07-16 雙軌制 | 有效（dbt 是否開放 TBD-29） | uniray7 |
| 07-17 | Managed curated 開放自建 | 擱置 | — | 已取代：11-19 curated 可由 service account 直接建表 | uniray7 |
