# 帳號模型與查詢協定

> 來源：[PRD](README.md) § 核心功能大綱｜狀態：草稿
> 授權操作（grant／revoke 流程、Doris privilege 對照）見 [authorization.md](authorization.md)；本文件只寫帳號模型與查詢協定。

## 帳號模型

- **Doris user、role、密碼全由平台管理**，managed 與 self-managed 皆同；使用者不能自建 Doris 帳號。
- **資料存取一律透過 service account**：workspace 內不開放個人帳號直接存取 table，需要存取的程式或團隊都向平台申請 service account。
- 平台層（Console）的身分是否接公司 SSO／LDAP → TBD-16。

### Managed 的權限

| 身分 | raw database | curated database |
|------|--------------|------------------|
| Service account | 只能讀（SELECT） | SELECT、INSERT、CREATE TABLE、DROP TABLE（TRUNCATE 同屬 `DROP_PRIV`）→ 開放建表是否安全見 TBD-30；ALTER 見 TBD-22 |
| 平台 ingestion pipeline 帳號 | 寫入指定的 raw 表 | — |
| 平台系統帳號 | 建表、改表、刪表、清空 | — |

### Self-managed 的權限

| 身分 | 權限 |
|------|------|
| Service account | SELECT、LOAD、INSERT、DELETE、UPDATE；可對自己管理的表 grant／revoke 讀取權（見 authorization.md） |
| 平台系統帳號 | 建 database、DDL 代建、監控、保護性 config 維護（對外文件揭露其存在與用途） |

- Table DDL 目前一律由平台代建（[RFC-003](../rfc/RFC-003-self-managed-ddl-governance-and-rr.md)）；是否因 dbt 開放 service account 直接 DDL → TBD-29，見 [ddl-governance.md](ddl-governance.md)。
- 因帳號全由平台管理，使用者沒有 `GRANT_PRIV` 可自行開回 DDL 權限；初始帳號不含 `GRANT_PRIV` 的技術可行性 → TBD-11。

### 帳號生命週期

| 事件 | 行為 |
|------|------|
| Service account 申請 | 向平台申請，平台建立並交付憑證（交付方式 → TBD-24） |
| 憑證輪換 | 支援輪換；週期政策 → TBD-15 |
| 憑證外洩通報 | 平台立即輪換，並以稽核紀錄追查使用情形 |
| Workspace SUSPENDED | 全部帳號停用 |
| Workspace DECOMMISSIONED | 全部帳號回收，稽核紀錄保留 |

## 查詢協定

兩種協定**共用同一套 Doris 帳號與權限**，權限不因協定而異。

| 協定 | 定位 | 支援範圍 |
|------|------|----------|
| MySQL protocol | 互動式查詢、BI 工具連線、metadata 操作 | 完整 SQL（受權限約束），含 metadata SQL |
| Arrow Flight SQL | 高吞吐欄式讀取（應用程式、資料處理框架大量拉資料） | 僅 SELECT 類 SQL；不支援 metadata SQL（如 `SHOW DATABASES`） |

- 兩者互補：探索 schema、寫入、管理操作用 MySQL protocol；大量讀取用 Arrow Flight SQL。
- Curated 的寫入（`INSERT … SELECT`）只能走 MySQL protocol。
- Gateway（統一端點、QPS 計量，[RFC-002](../rfc/RFC-002-shared-cluster-resource-isolation.md)）以 MySQL protocol 為前提；Arrow Flight SQL 走 gRPC、資料面直連 BE，能否經 gateway、QPS 如何計量 → TBD-10。

## 行為規格

| # | 情境 | 預期行為 |
|---|------|----------|
| 1 | Managed service account 寫入或 DROP raw 表 | 權限錯誤 |
| 2 | Managed service account 對 curated 表 `INSERT … SELECT` | 成功 |
| 3 | Arrow Flight SQL 執行 metadata SQL | 不支援錯誤，需改走 MySQL protocol |
| 4 | Arrow Flight SQL 執行非 SELECT 語句 | 拒絕 |
| 5 | 同一帳號經兩種協定連線 | 權限行為一致；metadata 隔離在兩種協定下皆成立 |

## 驗收標準

- [ ] 權限表逐格實測（兩種模式）
- [ ] 使用者端無任何帳號持有 `GRANT_PRIV`
- [ ] Workspace 停用後所有帳號無法連線（含既有連線）
- [ ] 兩種協定的權限結果一致
