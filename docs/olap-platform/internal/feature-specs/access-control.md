# Feature Spec：帳號與存取控制

> 來源：[PRD.md](../PRD.md) § 6.10
> 狀態：草稿（Phase 3）
> ⚠️ 帳號體系的公司內整合（SSO/LDAP）脈絡未傾倒，標 ⚠️ 處待確認。

## 定位

定義 workspace 內「誰、用什麼身分、能做什麼」。隔離邊界（ws 之間）由 workspace-isolation 負責；本 spec 負責 **ws 之內**的帳號模型與權限矩陣，並落實「managed 無直接 load/DDL 權限」這條產品鐵律。

## 兩層身分模型 ⚠️

| 層 | 身分 | 用途 |
|----|------|------|
| 平台層（web console/API） | 公司員工身分（SSO 整合 ⚠️ 待確認） | 申請 DDL、管理 pipeline/job、看儀表板、核准（owner） |
| 資料層（Doris SQL） | 平台簽發的 Doris 帳號 | 實際連線查詢（經 gateway 統一端點，RFC-002） |

- 平台層操作全部留稽核紀錄（誰、何時、做了什麼）。
- 資料層帳號由平台簽發與回收，使用者不能自建 Doris 帳號（managed；self-managed 拿到管理帳號後自理）。

## 角色模型（Managed）

| 角色 | 平台層權限 | 資料層權限 |
|------|-----------|-----------|
| **Workspace Owner**（至少 1，建議 2 ⚠️） | 成員管理、DDL 核准（方案 A 的核准者）、pipeline/job 管理、告警設定、delete 類申請核准 | 同 Member |
| **Member** | 提交 DDL 申請、管理 pipeline/job、看儀表板 | SELECT：本 ws 全 zone；INSERT/UPDATE/DELETE：**僅 curated**（ELT 產出）；**無** load / DDL / tmp・raw 寫入 |
| **Service Account**（應用程式接平台查詢用） | 無 console 權限 | **僅 SELECT** ⚠️（範圍可否限縮到指定 database 待確認）；不可互動式操作 |
| **平台系統帳號**（使用者不可見） | — | pipeline 寫 tmp、代執行 merge SQL 寫 raw、代執行 DDL、清理 tmp |

- 寫入路徑總覽（與 zone 模型對齊）：tmp ← 平台 pipeline；raw ← 平台代執行的 merge SQL；curated ← 使用者 ELT。使用者帳號能寫的只有 curated。
- Streaming producer 憑證（Kafka）與 batch 上傳憑證（MinIO）獨立於上表，隨 pipeline/job 簽發、可獨立輪換（見各 ingestion spec）。

## 角色模型（Self-managed）

- 平台交付**初始管理帳號**後，cluster 內帳號/權限全由使用者自理（責任歸屬矩陣）。
- 平台保留一個系統帳號做監控與保護性 config 維護（對外文件揭露其存在與用途）。
- 平台層（console）僅剩：儀表板檢視、scale 申請、聯絡人維護——角色簡化為 Owner/Member 兩級。

## 帳號生命週期

| 事件 | 行為 |
|------|------|
| ws 開通 | 建 Owner（申請單上的聯絡人）+ 初始資料層帳號 |
| 成員加入/移除 | Owner 在 console 操作；移除即回收資料層帳號與進行中的核准權 ⚠️（離職自動同步依 SSO 整合而定） |
| 憑證輪換 | 資料層帳號密碼/Service Account token 支援輪換；週期政策 Phase 5/6 ⚠️ |
| ws SUSPENDED | 全部資料層帳號停用（isolation spec 行為 #6）；console 唯讀 |
| ws DECOMMISSIONED | 全部帳號回收，稽核紀錄保留 |

## Row / Column Filter（未決，open issue #3）

使用者已提出需求；**做在平台側 vs 使用者自理**未決，候選 RFC 題目：
- 若做：Doris 原生 row policy / column masking 為基礎，平台包裝設定介面；權限矩陣加一層「同 ws 內分級」。
- 若不做：ws 內全員同權限，敏感欄位由使用者以 curated 視圖自行控制。
- 本 spec 先按「不做」寫權限矩陣，RFC 決議後增修。

## 行為規格

| # | 情境 | 行為 |
|---|------|------|
| 1 | Managed member 嘗試 stream load / insert into raw / CREATE TABLE | 權限錯誤（isolation spec 行為 #8 的帳號級落實） |
| 2 | Member 對 curated 表 insert...select | 成功（唯一開放的寫入） |
| 3 | Service account 嘗試寫入或 console 登入 | 拒絕（僅 SELECT、僅資料層） |
| 4 | Owner 移除成員 | 該成員資料層帳號立即失效、進行中申請轉派 ⚠️ |
| 5 | 最後一位 Owner 欲退出 | 擋下：須先指派新 Owner |
| 6 | 憑證外洩通報 | 平台立即輪換該憑證；稽核追查使用紀錄 |

## API 邏輯

| Method | Path | 說明 |
|--------|------|------|
| GET/POST/DELETE | `/api/v1/workspaces/{ws}/members` | 成員列表/加入/移除（Owner 權限） |
| PATCH | `/api/v1/workspaces/{ws}/members/{id}` | 角色變更（Owner ⇄ Member） |
| POST | `/api/v1/workspaces/{ws}/service-accounts` | 建立 service account → 回連線憑證 |
| POST | `/api/v1/workspaces/{ws}/credentials/{id}/rotate` | 憑證輪換 |

## 與其他功能的依賴關係
- ws 間隔離邊界與 GRANT 實作：**workspace-isolation**（本 spec 是其「帳號模型細節」的展開）。
- DDL 核准權限：**database-table-application**（方案 A 的 owner approve 即本 spec 的 Owner 角色）。
- Producer/上傳憑證：**streaming/batch-ingestion**。
- 連線一律經 gateway：**RFC-002**（連線數控管、QPS 計量以帳號歸屬 ws）。
- Row/column filter：RFC 決議（待拍板 #5）。

## 待確認事項
1. 平台層身分是否接公司 SSO/LDAP；離職是否自動回收。
2. Service account 的權限粒度（全 ws SELECT vs 指定 database）。
3. Doris 帳號的併發/超時等 per-user 屬性是否由平台統一設定（關聯 RFC-002 per-ws 參數）。
4. 憑證輪換週期政策（Phase 5/6）。

## 邊界審查（Phase 4 補上）
| Edge Case | 是否適用 | 處理方式 |
|-----------|---------|----------|
| （待 Phase 4） | | |

## 驗收標準（Acceptance Criteria）
- [ ] 權限矩陣逐格實測：managed member 僅能寫 curated，無 load/DDL 路徑
- [ ] Service account 無法執行 SELECT 以外操作
- [ ] 成員移除後其帳號立即無法連線（含既有連線終止）
- [ ] 最後一位 Owner 無法自行退出
- [ ] 平台層操作稽核紀錄完整（成員異動、憑證簽發/輪換）
