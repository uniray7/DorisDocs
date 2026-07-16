# RFC-003：Self-managed 服務的 DDL 治理與資料管理 R&R

- 狀態：草稿（**待 manager 決策**）
- 作者：平台團隊
- 日期：2026-07-16
- 相關：`internal/PRD.md § 6.8`、`feature-specs/self-managed-operations.md`、open issue #3（row/column filter）
- 環境前提：Apache Doris 4.1.0；self-managed HA 為雙 cluster + CCR cluster 級全同步（已決議）

## 背景與決策動機

Self-managed 使用者的既有需求是「自己灌資料 + 任意開 database/table」，平台原承諾 DDL 完全自由。但在釐清資料管理責任時發現矛盾：

1. **平台對看不懂的東西負不了責**。資料管理責任（備份、保留清除、健康巡檢）需要平台至少知道「有哪些物件、哪些重要」。DDL 完全自由下平台既無物件清冊、也無語義，責任無處掛載。
2. **CCR 不是備份**。HA 方案是 cluster 級全同步——使用者誤 `DROP TABLE`、灌錯資料，standby 幾秒內同樣消失/污染。HA 擋機器故障，擋不住邏輯錯誤——這個 gap 需要被有意識地決定怎麼處理（見「子決策：備份/還原」，**暫定不提供、明文免責 + audit log 舉證**）。
3. **保護性 config 需要下沉到 table 粒度**——但注入點受限：使用者以 **dbt** 處理資料，dbt 高頻 create/drop 暫時性 table，平台經手每次建表不可行（見已定前提 #1 的修訂），表級保護改以「cluster/database 級預設值 + 巡檢偵測違規」實現。

## 已定前提（團隊已收斂，2026-07-16）

以下適用**全部選項**，不在 manager 決策範圍內：

1. **DDL 治理分兩級**（2026-07-16 修訂：原「table 也代建」因 dbt 相容性撤回）：
   - **Database DDL 走代建流程**：不直接打 Doris——「提交 → 驗證 → 平台代建」（初期人工執行，之後推出 API/GUI 工具）。database 是低頻操作，摩擦可接受。
   - **Table DDL 開放直接打 Doris**：使用者以 **dbt** 處理資料，dbt 的運作方式即高頻 create/drop 暫時性 table（staging model、`__dbt_tmp` 中繼表、create-then-swap），代建流程會直接讓 dbt 不可用。
2. **驗證深度（database 代建流程）**：語法合法性 + 危險操作攔截 + 撞名檢查；平台在建 database 時**注入/設定保護性預設**。**不驗 schema 設計**——品質責任維持在使用者（既有決議不變）。
3. **表級保護改為「預設值 + 巡檢」**：replication 等保護性設定以 cluster/database 級預設值生效（self-managed 為 dedicated cluster，cluster 級預設可行）；建表時的危險樣態（無分區大表等）無法事前攔截，改由**巡檢事後偵測 + 告警**。
4. 灌資料維持完全自由（寫入路徑在使用者手上，這是 self-managed 的定義性特徵）。

> ⚠️ 對產品定位的影響（manager 需知悉）：相對原承諾「任意開 database/table」，本前提只收回 **database 層的直接執行**；table 層維持完全自由（dbt 工作流不受影響）。Managed 與 self-managed 的差異重心維持在「**寫入路徑 + 資料管理責任**」。
> ⚠️ dbt 相容性注意：dbt 慣以 schema（在 MySQL 協定下即 Doris database）組織 target/custom schema，且會嘗試自動 `CREATE SCHEMA`——使用規範需指引「database 先透過代建流程建好、關閉 dbt 的自動建 schema」，此行為列入待驗證。

## 待決問題

DDL 代建流程確定後，剩下的決策是**治理形態**：要不要把 managed 的 zone/命名模型套到 self-managed？三個選項的差異本質是「**資料語義用什麼方式承載**」。

## 核心分析：zone 語義在 self-managed 是空心的

這是評估選項 1/2 的關鍵，先講清楚：

- **Databricks 的 bronze/silver/gold（我們 tmp/raw/curated 的原型）本身不附帶任何引擎強制的管理行為**。Medallion 是組織慣例；引擎不擋你把髒資料寫進 gold。各層的管理行為（bronze append-only + retention、silver 品質規則、gold 寬存取 + 優化）全是採用團隊**自己用 pipeline 工具與權限綁上去的**。
- 我們 managed 模式的 zone 之所以「真的執行得了」（tmp 只有 pipeline 能寫、raw 只有 merge SQL 能寫、tmp 平台代清），唯一原因是**平台握著寫入路徑和 DDL**。
- Self-managed 的寫入路徑在使用者手上：平台**無法要求** pipeline 只寫 raw、curated 只從 raw 衍生。使用者可以把黃金資料放 tmp、把落地資料放 curated，平台無從驗證。
- **結論**：self-managed 下 zone 是「宣告」不是「保證」。zone 能換到的管理行為，只剩「按宣告的 zone 套一包預設值」（tmp＝短 retention 不備份、curated＝要備份長保留）——而這件事用**建立時的表單宣告**（勾選：要備份嗎、TTL 多久、含敏感資料嗎）承載，資訊量更大、事後可改、不用使用者背命名文法。**宣告制是實體，zone 命名只是它的一種介面，而且是最脆的一種。**

## 選項

> 註：「database 代建、table 直接執行」為共同前提，三個選項的日常摩擦幾乎相同（table 層全都自由）；差異集中在 database 的組織方式與語義承載。

### Option 0（基線，已排除）：DDL 完全自由（直接打 Doris）
維持原承諾。**排除理由**：無法注入保護性 property、無物件清冊、無宣告入口——資料管理責任完全無處掛載，平台只能做 cluster 級告警。列出僅供對照「我們為什麼要做這個決策」。

### Option 1：只開三個固定 database（`{ws}_tmp` / `{ws}_raw` / `{ws}_curated`）
使用者不能開 database，所有表建在三個預建庫內（table 走代建流程）。

| Pros | Cons |
|------|------|
| 與 managed 心智模型一致，對外文件說一次 | **Zone 語義空心**（見核心分析）：平台給了結構卻保證不了語義，「名實不符」時的責任糾紛反而更難釐清 |
| 物件盤點最整齊 | **組織粒度被鎖死**：多資料域的團隊全部擠三個庫；database 級的權限/管理操作失去彈性 |
| 可按 zone 套預設值包（宣告縮寫） | 名字承載的宣告資訊量最小、且不可改 |
| | 命名形式（單底線）與 managed 慣例（雙底線）不一致，工具鏈要處理兩套 |

**團隊評估：不建議。** 付出最大的自由度代價，換到的語義是空心的。

### Option 2：自由開 database，強制 `{ws}__{user_defined}__{tmp|raw|curated}` 命名
Database 數量自由，但名字必須帶 zone 段；database/table 皆走代建流程。

| Pros | Cons |
|------|------|
| 組織粒度自由 | Zone 語義同樣空心（寫入路徑不在平台手上，同 Option 1） |
| 與 managed 命名完全一致，平台自動化工具可共用解析邏輯 | 使用者要背命名文法；名字一旦建立不可改 |
| Zone 段可當宣告縮寫套預設值包 | **與 Option 3 的機制完全相同**（代建流程一樣），若平台不對 zone 附加差異化行為，命名規則就是純摩擦 |

**團隊評估：不建議。** 它與 Option 3 只差「宣告用名字載還是用表單載」——表單載得更多、可改、免文法。唯一保留價值是工具鏈一致性，但不足以抵掉語義空心的根本問題。

### Option 3：自由開 database/table，無命名規則，代建流程 +「建立時宣告」（**團隊建議**）
Database/table 自由規劃、不套 zone；代建流程的提交表單同時是**宣告入口**：

- **Database 建立時宣告**（代建流程內建，皆可日後修改）：
  - **TTL/retention**：要不要平台代清、保留多久——使用者指定保留期，平台檢查後執行清理，清理失敗或未生效時告警
  - **敏感性**：是否含敏感資料 → 銜接 open issue #3（row/column masking）與 PII 治理（open issue #5）
  - （備份宣告屬下方「子決策」的乙案，**暫不提供**）
- **Table 級宣告為選用登記**（table DDL 不經手平台，無法內建）：dbt 產生的暫時表無需理會；使用者只為重要的長存表登記 TTL/敏感性。未登記的表僅受 database 級宣告與巡檢涵蓋。
- 平台的**物件清冊**兩個來源：database 由代建流程產生（完整）；table 由 metadata 週期掃描（近即時），巡檢與告警以此掛載。

| Pros | Cons |
|------|------|
| 責任邊界最誠實：平台只承諾「執行宣告 + 掉了會叫」，不假裝懂語義 | 與 managed 的模型不一致（但本來就是兩種模式） |
| Database 級宣告內建在必經流程、涵蓋率天然高；table 級選用登記不干擾 dbt 工作流 | Table 級宣告涵蓋率取決於使用者自律（未登記的長存表僅受 db 級宣告與巡檢涵蓋） |
| 宣告可事後修改（名字不行） | 平台要維護宣告與物件的對應（清冊系統） |
| 為 row/column masking（open issue #3）鋪好同一套宣告機制 | |

## R&R 矩陣（以 Option 3 為基準；選 1/2 時差異見註）

| 責任 | 平台 | 使用者 |
|------|------|--------|
| Database DDL 代建執行、語法/危險操作/撞名驗證 | ✅ | 提交申請 |
| Table DDL（含 dbt 高頻建/刪暫時表） | ❌ 不經手 | ✅ 直接打 Doris、自由執行 |
| 保護性預設（cluster/database 級 replication 等）＋巡檢偵測違規表並告警 | ✅ | 收到告警後處置；自行覆寫表級 property 的後果自負 |
| Schema 設計品質（分區/分桶/型別/效能） | ❌ 不審不揹 | ✅ |
| 物件清冊維護（database＝代建紀錄；table＝metadata 週期掃描） | ✅ | |
| 儲存健康巡檢（占用/成長/tablet 健康/compaction/反模式偵測），**告警附建議、不強制處置** | ✅（infra 告警的延伸，預設提供） | 處置 |
| 備份/還原 | ❌ **暫不提供**（子決策暫定甲案，見下節） | ✅ 自行負責 |
| 誤刪/誤操作救援 | ❌ **一律不救援**（CCR 會同步誤操作、無備份即無還原）——**明文寫入對外文件** | ✅ 全責 |
| 稽核舉證 | ✅ database DDL 有代建申請紀錄；**table DDL 與 DML 舉證依 Doris audit log**（規劃集中至 ELK），供「操作出自使用者」的責任釐清 | |
| TTL 清理（使用者指定保留期，平台檢查後執行；table 級靠選用登記） | ✅ 執行清理；清理失敗/未生效時告警 | 保留期數字的正確性；重要長存表的登記 |
| 資料流向/血緣、資料正確性 | ❌ | ✅（寫入路徑在使用者手上） |
| Row filtering / column masking | 待 open issue #3 決議（若承接，走同一套宣告機制） | 政策內容、schema 變更後重新宣告 |

> 選 Option 1/2 的差異：上表全部成立，另加「zone 段＝預設值包的宣告縮寫」（tmp 預設短 TTL、curated 預設長保留）——但預設值包在 Option 3 用表單預設值同樣做得到。**R&R 實質差異：無。**（同仁「option 2/3 R&R 無差別」的判斷正確，且在 table 代建為前提下，三個選項的 R&R 都幾乎相同。）

## 子決策：備份/還原提供與否（兩案並列，**暫定甲**）

背景：HA 的 CCR 是 cluster 級**全同步**——使用者誤 `DROP`、灌壞資料，standby 幾秒內同樣消失/污染。**HA 救得了機器故障，救不了邏輯錯誤**；不提供備份，「誤操作資料丟失」即永久不可救援。此 gap 需被有意識地接受，故明列兩案：

| | **甲：不提供備份/還原（暫定採用）** | **乙：宣告制備份（保留為未來選項）** |
|---|---|---|
| 內容 | 平台不做任何備份；誤刪/誤操作/資料丟失**一律不救援**，使用者自行負責（含自建備份） | 使用者建立物件時宣告「要備份、頻率、保留份數」→ 平台按宣告執行 Doris BACKUP 至 MinIO，並監控備份 job 成功率（靜默失敗即告警） |
| 配套 | **舉證機制**：database 刪除有代建申請紀錄（誰提交、何時執行）；**table 級操作（含 DROP TABLE）與 DML 的舉證完全依賴 Doris audit log**（規劃集中至 ELK）——證據鏈仍完整，但成立前提是 audit log 的保存不可中斷（保留期 Phase 5/6 定）；對外文件**明文免責** | 責任切分：平台對「忠實執行宣告」負責；宣告內容正確性、未宣告物件的丟失歸使用者 |
| 成本 | 零開發、零維運 | 備份排程系統、還原流程、儲存空間、監控——需投入開發與常態維運 |
| 風險 | 第一個誤刪重要資料的使用者無法救援（免責條款可擋責任，擋不了觀感） | 涵蓋率與宣告正確性的邊際糾紛；維運負擔 |

**啟動乙案的但書**：目前無備份相關計畫；待**平台有充足人力**且**使用者實際開出需求**時，再評估投入開發（屆時以獨立提案增補，不影響本 RFC 其餘決議）。

## 三選項比較（決策摘要）

| | Option 1：固定三庫 | Option 2：自由開＋命名規則 | Option 3：自由開＋表單宣告 |
|---|---|---|---|
| Database 組織自由 | ❌ 鎖死 3 個 | ✅ | ✅ |
| 使用者額外負擔 | 無命名負擔，但受組織限制 | 背命名文法 | 表單多幾個欄位 |
| 語義承載方式 | 固定結構（不可改） | 名字（不可改、資訊量小） | 表單宣告（可改、資訊量大） |
| 語義可信度 | 宣告（空心，無法保證） | 宣告（空心，無法保證） | 宣告（**誠實承認**是宣告） |
| R&R 實質差異 | 無 | 無 | 基準 |
| 與 managed 工具鏈一致性 | 低（命名形式不同） | 高 | 低（但本來就是不同模式） |
| 團隊建議 | 不建議 | 不建議 | **建議** |

**一句話總結給 manager**：三個選項的管理能力和責任分工幾乎一樣（都靠代建流程＋宣告），差別只在把「宣告」刻在結構裡、刻在名字裡、還是寫在表單裡——結構和名字改不了又裝不多，所以我們建議表單（Option 3）。

## 對既有文件/決議的衝擊（定案後回填）

- `feature-specs/self-managed-operations.md`：責任矩陣併入本 RFC 的 R&R；「任意開 database/table」需求的回應方式更新。
- `PRD.md`：self-managed 術語定義（「可自行執行 DDL」→「DDL 走代建流程」）；§6.8。
- `feature-specs/workspace-application.md`：「self-managed 命名自由」的表述（Option 3 下仍成立，但需補「DDL 走代建」）；佈建交付項增加 DDL 提交管道說明。
- `feature-specs/access-control.md`：self-managed 帳號**保留 table 級 DDL 權限、收回 database 級 DDL**（database 僅平台系統帳號可建/刪）——Doris 權限體系能否精確做到「禁 database DDL、允 table DDL」需驗證（見待驗證項）。
- `00-progress.md`：使用者需求清單第 2 條的決議狀態。
- 新增平台元件：DDL 提交/驗證/代建管道（初期人工 + 內部工具，之後 API/GUI）——納入 system-architecture（Phase 8）。

## 待驗證項（實作時）
- [ ] **Doris 權限粒度**：能否對使用者帳號「禁 database DDL、允 table DDL」（catalog 級 vs database 級 CREATE 權限的行為）
- [ ] **Cluster/database 級保護性預設**：replication 等預設值在 Doris 4.1.0 的設定點與生效行為（FE config / db property）
- [ ] **dbt on Doris 行為**：schema（=database）自動建立的處理方式（關閉 dbt 自動建 schema 的設定指引）、materialization 使用的暫時表樣態
- [ ] 危險操作攔截清單（database 代建流程適用）；表級危險樣態（無分區大表等）的**巡檢偵測**規則
- [ ] Doris audit log 集中保存管道（規劃 ELK）與保留期（Phase 5/6 定案）
- [ ] （僅子決策乙案啟動時）Doris BACKUP/RESTORE 在 4.1.0 的物件粒度、與 CCR 的並存行為
- [ ] 宣告清冊的存放與稽核（平台側系統）

## 決策紀錄
| 日期 | 決議 | 決策者 |
|------|------|--------|
| （待 manager 決策） | | |
