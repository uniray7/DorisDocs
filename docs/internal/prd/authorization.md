# Apache Doris Workspace 權限管理

> 來源：Authorization PRD v0.12（2026-11-16，資料平台團隊內部評審）｜狀態：評審中
> Tool Team 隸屬資料平台團隊，負責建置 Web Console。短期交付給平台人員：先選 fab、cluster，再於該 cluster 建立 Doris user，或對指定 user 的 database／table privilege 執行 grant 或 revoke。

> **與 repo 現有決議的差異（合併時確認，2026-11-19）**
> - **跨 workspace 分享**：v0.12 允許 owner 分享給其他 workspace；目前不支援。
> - **帳號模型**：v0.12 的 owner、reader 是人員的 Doris 帳號。改為：owner 是人，只在 Web Console 操作，不持有 Doris 帳號；資料存取（含 owner 本人）一律透過 service account，授權對象是 service account，權限由 owner 在平台 policy 上限內決定（見 [access-control.md](access-control.md)）。本文已依此改寫。
> - **Console 申請頁、審核頁、開通狀態**：不採用。Workspace 申請與審核維持人工，核准後由平台透過 Web Console 的 Admin 後台執行開通（見 [web-console.md](web-console.md)）。v0.12 的「開通 workspace 與查看進度」一節未併入。
> - **Managed curated 的 DDL**：執行身分為 service account；建表的 FE 風險 → TBD-40；刪表會連帶取得刪庫權限 → TBD-30。
> - **TRUNCATE 所需權限**：v0.12 依 4.x 官方文件寫 `DROP_PRIV`，但 4.1.4 source 檢查的是 `LOAD_PRIV`，待實機驗證 → TBD-38。
> - **Arrow Flight SQL 支援範圍**：目前只使用過 SELECT，其他語句待驗證 → TBD-39。

## 服務與授權範圍

- 資料平台在 HashiCorp Nomad 上管理 Apache Doris 4.1.4。十幾座 fab 各有多座 cluster，整體預計 20–40 座。
- 同一位使用者通常可在多個 fab 使用服務。每個 workspace 都必須橫跨至少兩個不同的 fab；每個 fab 的 Doris 服務彼此獨立，不跨 fab 部署。
- 平台與 owner 均先選 fab。平台人員可管理指定 cluster 內的 database 或 table privilege；owner 只能對自己 workspace 的 service account 授權，範圍限於自己 workspace 的 database 與 table。
- MySQL 與 Arrow Flight SQL 執行相同的權限規則。

## 角色與責任

| 角色 | 責任 |
|------|------|
| 資料平台團隊 | 審核申請、決定服務模式與 cluster、建立 database，並執行由平台管理的 table 操作。所屬 Tool Team 建置 Web Console 與授權後端，提供平台帳號與權限管理，以及後續的 owner 自助功能。 |
| Workspace owner | 人。申請通過後由申請人擔任，也是該 workspace 的 data owner。在 Web Console 為自己的 workspace 申請 service account、設定其權限；本身沒有 Doris 帳號，自己要讀資料也透過 service account。 |
| Service account | 實際存取資料的 Doris 帳號，權限由 owner 設定，不超過平台 policy 上限。目前不支援授權給其他 workspace 的 service account。 |

## 交付範圍

| 範圍 | 內容 |
|------|------|
| 短期 | 平台管理 Console，以 Doris 4.1.4 原生授權為驗收基準 |
| 後續 | Owner 自助：申請 service account、在平台 policy 上限內設定權限。上線前由平台人員以 Admin 後台代為操作 |
| 長期 | 評估與 Trino 共用 OPA（row filter、column mask） |

## 短期交付：平台管理 Console

平台人員先選 fab，再從該 fab 選擇目標 cluster。畫面顯示目前的 fab 與 cluster，提供三種操作：

| 操作 | 平台人員的選擇與預期結果 |
|------|--------------------------|
| 建立 user | 輸入目標 Doris user identity，確認後在該 cluster 建立 user。顯示成功、已存在或失敗原因；建帳與資料存取授權各自有明確的操作結果。 |
| Grant privilege | 選現有 user、database、授權層級及適用的 privilege。Database 層級顯示涵蓋該庫所有 table；table 層級再選具體 table。確認完整目標後執行 grant。 |
| Revoke privilege | 選 user 並查看現有 grants，選取要移除的 database 或 table grant，確認層級與 privilege 後執行 revoke。若仍有 database grant 或 role 提供同一權限，顯示剩餘來源。 |

### 短期驗收案例

在 Fab A 的目標 cluster 建立一個測試用 service account，確認其他 cluster 的帳號清單不受影響。以下用這個沒有其他資料權限的測試帳號，透過一般查詢連線檢查：

1. 同一 database 的兩張測試表，只對第一張 grant `SELECT_PRIV`。測試帳號只能讀第一張；revoke 該筆 table grant 後，兩張都無法讀取。
2. 對整個 database grant `SELECT_PRIV`。測試帳號可讀兩張表；revoke 該 database grant 後，兩張都無法讀取。
3. 同時存在 database 與 table 的直接 grant 時，只 revoke table grant，測試帳號仍可透過 database grant 讀取該表；Console 應指出仍有效的來源。

## Managed 模式

### 開通後的授權

- 申請通過後，申請人成為 data owner。Owner 不持有 Doris 帳號；開通時也不建立 service account，資料存取透過 owner 之後申請的 service account（見 [access-control.md](access-control.md)）。
- Owner 透過平台申請建立 database。目標 table 存在後，owner 可透過 Console 對 service account 授權。
- Raw table 由平台建立；curated table 可直接建立（DDL 規則見 [ddl-governance.md](ddl-governance.md)）。

### Database 命名

- Raw：`{workspace}__{user_defined}__raw`
- Curated：`{workspace}__{user_defined}__curated`

兩欄位的英文字母限 a–z，數字限 0–9；禁止 hyphen（`-`）與欄位內的雙底線（`__`）。

建議首版白名單（TBD-26）：單底線僅可置於英數字之間；workspace 以小寫字母開頭，user_defined 可用數字開頭；其餘符號逐項評估。組合後再檢查完整 database 名稱長度。例：`domain_product`、`user_defined` 合法；`domain__product`、`user-defined`、`User_defined` 回報欄位錯誤。

| Database 範例 | 誰能讀寫 | 誰能管理資料表 |
|---------------|----------|----------------|
| `domain_product__user_defined__raw` | Service account 只能讀；僅平台 ingestion pipeline 可寫入指定的 raw 表 | 平台負責建表、改表、刪表及清空資料 |
| `domain_product__user_defined__curated` | Service account 可讀寫（依 owner 設定），例如用 `INSERT … SELECT` 將 raw 資料整理後寫入 | Service account 可 CREATE／DROP／TRUNCATE TABLE（依 owner 設定；建表風險 TBD-40、刪表方式 TBD-30）；ALTER 待確認（TBD-22） |

### 範例：同一 owner 在兩個 fab 授權

`domain_product` 橫跨 Fab A 和 Fab B，兩地各有一張 `domain_product__user_defined__curated.yield_daily`。報表系統要讀這張表，owner 為它申請了一個「報表用 service account」，然後選 Fab A → 目標 cluster → curated database → `yield_daily`。

- 報表用 service account 已在 Fab A 的 cluster 建立：owner 可將 Fab A 這張表的 `SELECT_PRIV` grant 給它。
- 報表用 service account 在 Fab B 的 cluster 還沒有帳號：須先建立才能另行 grant（多 fab 的帳號如何建立 → TBD-24）。
- 日後在 Fab A 執行 revoke，只移除 Fab A 的那筆 grant。

### Managed 驗收

- 以平台 pipeline 寫入指定 raw 表成功；直接寫 raw 被拒絕；從 raw 讀取並寫入 curated 成功。
- 以測試表驗證 curated 可 CREATE、DROP、TRUNCATE TABLE；對 raw 執行 DROP 或 TRUNCATE 被拒絕；確認表級操作沒有擴大為刪除 database 的權限。

## Self-managed 模式

- 平台只提供專屬 cluster，並負責建立 database；database 命名待確認（TBD-8）。
- 團隊維護自己的 ingestion pipeline，在 workspace 可使用的 database 內寫入資料。
- Service account 權限上限為 SELECT、LOAD、INSERT、DELETE、UPDATE；owner 在 Console 選定 fab，於上限內為 service account grant 或 revoke（見 [access-control.md](access-control.md)）。
- TRUNCATE／DROP 由誰執行、dbt 能在哪些 database 建表與刪表 → TBD-23、TBD-29，選項與風險見 [ddl-governance.md](ddl-governance.md)。

## Doris 原生 privilege 對照

Doris 4.x 官方 GRANT 文件所列的十項原生 privilege，供 Tool Team 設計 4.1.4 的授權後端。Owner 在 Console 為 service account 設定權限時，後端只會授予平台 policy 上限內的 privilege（上限見 [access-control.md](access-control.md)）；授權管理與資源使用類 privilege 由平台保留。

### 資料與物件操作

以下六項可在 global、catalog、database 或 table 層級授權；`SELECT_PRIV` 另支援指定 column。Global 指目前連線的 Doris cluster；平台的 fab 與 workspace 決定目標 cluster 及其內的物件。

| Privilege | 在指定範圍內允許的行為 |
|-----------|------------------------|
| `SELECT_PRIV` | 讀取 table 或 view。Table 層級可讀整張表；column 層級只允許指定欄位。 |
| `LOAD_PRIV` | 資料寫入，涵蓋匯入、INSERT 與 DELETE 等。Raw pipeline 僅取得指定目標表的寫權。`INSERT … SELECT` 還需要來源表的讀權。4.1.4 source 中 TRUNCATE TABLE 檢查的也是 `LOAD_PRIV`（待驗證，TBD-38）。 |
| `CREATE_PRIV` | 建立物件，例如在指定 database 建立 table 或 view。平台建立 database；managed 的建表範圍限於 curated database。 |
| `ALTER_PRIV` | 變更既有物件，例如重新命名 table、增刪或修改 column、調整 partition。Managed curated 是否開放 ALTER 待決（TBD-22）。 |
| `DROP_PRIV` | 刪除授權範圍內的物件。表級 `DROP_PRIV` 用於 DROP TABLE；database 級用於 DROP DATABASE。TRUNCATE TABLE：4.x 官方文件寫屬於 `DROP_PRIV`，與 4.1.4 source（`LOAD_PRIV`）不一致，待驗證（TBD-38）。Doris 沒有建立者擁有權，要讓帳號刪自己建的表只能授予 database 級 `DROP_PRIV`，會連帶允許 DROP DATABASE（見 [ddl-governance.md](ddl-governance.md)）。 |
| `SHOW_VIEW_PRIV` | 用 SHOW CREATE VIEW 查看 view 定義；要查詢 view 的資料仍需 `SELECT_PRIV`。 |

### 授權管理與資源使用

| Privilege | 作用範圍與行為 |
|-----------|----------------|
| `GRANT_PRIV` | 依授權層級管理 grant／revoke。一般授權者須有對應範圍的 `GRANT_PRIV` 與欲授予的 privilege；將 role 指派給 user 需要 global `GRANT_PRIV`。原生 user、role 與授權操作由平台後端負責。 |
| `ADMIN_PRIV` | Global。廣泛的叢集與資料管理權限；節點變更另需 `NODE_PRIV`。平台保留。 |
| `NODE_PRIV` | Global。管理叢集節點（新增、移除 FE／BE，BE 下線退役），供平台維運身分使用。 |
| `USAGE_PRIV` | 使用指定資源，例如 resource、workload group；部分部署也用於 compute group、storage vault。讀 table 仍需 `SELECT_PRIV`。 |

### Tool Team 串接與驗收

- 以 `domain_product__user_defined__curated.yield_daily` 為例：owner 為報表用 service account 授權後，後端在所選 fab 的目標 cluster 對這張表授予 `SELECT_PRIV`。若改授整個 database，下轄其他 table 也會落入範圍。驗收應確認該帳號能讀目標表、無法憑此 grant 讀同庫其他表，且 revoke 後能辨識其他 role 是否仍提供讀權。
- `ALL` 與 `READ_WRITE` 是舊版簡寫，會展開為 SELECT、LOAD、ALTER、CREATE、DROP 五項；`READ_ONLY` 對應 `SELECT_PRIV`。後端應使用明確的 privilege 名稱，才能比對實際 grant 與 owner 核准的範圍。
- 版本核對：在目標 4.1.4 cluster 執行 `SHOW PRIVILEGES` 查看可用項目與範圍，以 `SHOW GRANTS` 檢視帳號持有的權限。本節為文件對照，尚未在公司 cluster 實測。

## 後續交付：Owner 申請 service account 與設定權限

### 申請 service account

- Owner 在 Console 選 fab、填寫用途後送出；平台建立帳號並交付憑證。
- 是否需平台審核、憑證交付方式、workspace 橫跨多個 fab 時帳號如何建立 → TBD-24。

### Grant（六步）

1. 選 fab：指定要授權哪個 fab 的資料。
2. 選 cluster：從該 fab 中目前 workspace 可使用的 cluster 選取。
3. 選 database。
4. 選範圍：整個 database，或其中一張或多張 table。
5. 選 service account：須屬於本 workspace，且已在所選 cluster 建立。
6. 選 privilege 並確認：只列出平台 policy 上限內的項目（例：raw 只能選 `SELECT_PRIV`）；確認 fab、cluster、範圍、service account 與 privilege 後送出 grant。

- 切換 fab 時，清除原有的 cluster、database、table、service account 選擇。
- 後端確認 owner 對該 workspace 的管理權、所選 privilege 未超出平台 policy 上限、service account 已在目標 cluster 建立；任一不符即拒絕並說明原因，通過後套用 grant 並回報結果。

### Revoke

- 選定 fab，以 service account 或 table 搜尋並勾選 grant。
- 該表的最後一項 grant revoke 後，後續查詢應被拒絕；其他 fab 的 grant 繼續有效。
- 若仍透過其他 role 可讀，顯示該 role。
- 操作失敗時顯示「revoke 未完成」、受影響的 cluster、原因及重試入口。

## Doris user 與 role 的管理

- Doris user、role、帳號認證與密碼由平台管理。
- Owner 介面只提供 service account 申請與權限的 grant／revoke；service account 由平台後端建立，owner 不能直接建立 Doris user、設定或查看密碼（憑證交付與重設 → TBD-24）。
- 平台管理頁短期提供建立 user 與 database／table grant、revoke；role 建立、指派與移除屬可擴充功能。

## 長期：與 Trino 共用 OPA

- 保留身分、資源、動作與範圍，以供日後與 Trino 共用 OPA。整合前驗證 row filter、column mask、兩種查詢協定，以及切換後的 revoke 行為。
- 技術路徑：Doris 4.x 提供 Java 自訂授權介面。平台可針對 4.1.4 編譯實作 `AccessControllerFactory` 與 `CatalogAccessController` 的外掛，將權限檢查轉交 OPA，並把政策結果轉為 row filter 或 column mask。外掛須由整合方實作與驗證。
- PoC（TBD-27）：比較 MySQL 與 Arrow Flight SQL 對同一身分的結果，並驗證切換至外部政策後 grant／revoke 的生效來源。
- 版本背景：2026 年 9 月 apache/doris PR #66770 引入開發版的新 authorization SPI；4.x 官方指南使用上述兩個介面，並區分開發版的 API 與封裝方式。短期仍採原生 RBAC。

## 待確認

| 主題 | TBD |
|------|-----|
| 同一 workspace 在同一 fab 能否使用多座 cluster（目前傾向一座） | TBD-21 |
| Managed curated 是否開放 ALTER TABLE | TBD-22 |
| Self-managed TRUNCATE／DROP 的執行身分與範圍、dbt 範圍 | TBD-23 |
| Service account 申請是否需平台審核、憑證交付、多 fab 的帳號建立、命名、owner 離職或轉調交接 | TBD-24 |
| 命名白名單細節 | TBD-26 |
| OPA 外掛 PoC | TBD-27 |

## 參考文件

v0.12 原稿附有以下文件連結，合併時未保留網址，需補上：Doris 4.x 原生 RBAC 與授權層級、GRANT 語法與 privilege 清單、SHOW PRIVILEGES、Custom Authorization Plugin 指南（4.x）、authorization SPI 變更（PR #66770）、開發版 SPI 文件、物件命名、TRUNCATE TABLE、DROP TABLE、DROP DATABASE、UPDATE、資料存取控管。
