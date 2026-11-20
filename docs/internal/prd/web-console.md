# Web Console

> 來源：[PRD](README.md) § 核心功能大綱｜狀態：骨架｜負責：Tool Team（隸屬資料平台團隊）

## 功能範圍

| 功能 | 使用者 | 狀態 | 細節 |
|------|--------|------|------|
| Admin 後台：執行 workspace 開通（申請核准後） | 平台人員 | 範圍待定 | [workspace-application.md](workspace-application.md) |
| Admin 後台：選 fab → cluster，建立 Doris user、database／table grant 與 revoke | 平台人員 | 短期交付 | [authorization.md](authorization.md) |
| Owner 自助：申請 service account、設定權限（grant／revoke） | Owner | 後續交付 | [authorization.md](authorization.md) |
| Role 建立、指派、移除 | 平台人員 | 可擴充 | [authorization.md](authorization.md) |
| DDL 申請 | Owner | 提交管道待定（TBD-32） | [ddl-governance.md](ddl-governance.md) |
| Pipeline／batch job 管理 | Managed 使用者 | tmp 取消後流程待改寫（TBD-28） | [streaming-ingestion.md](streaming-ingestion.md)、[batch-ingestion.md](batch-ingestion.md) |
| 配額與用量儀表板 | 使用者 | 規格草稿 | [quota-and-usage.md](quota-and-usage.md) |

## Workspace 申請與開通

- **申請與審核是人工**：投影片 template＋每週三申請會議，Console 不提供 workspace 的申請頁或審核頁。
- **執行由平台透過 Admin 後台**：核准後，平台人員在 Web Console 的 Admin 後台執行開通（建立 database、帳號、權限等）。後台具體要涵蓋哪些開通動作 → TBD-34。

## 共通規則

- 所有操作先選 fab，再選 cluster；切換 fab 時清除後續選擇。
- 平台層操作全部留稽核紀錄（誰、何時、做了什麼）。
- 平台層身分是否接公司 SSO／LDAP → TBD-16。
