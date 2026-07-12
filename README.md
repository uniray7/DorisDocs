# DorisDocs

公司內部 OLAP Service（以 Apache Doris 為基礎）與 Data Ingestion Pipeline 的文件庫，包含兩大類文件：

- **對外文件**（`docs/<slug>/external/`）：給其他部門的使用者，涵蓋服務申請、審核流程、使用手冊、使用限制規範、Service Spec。
- **內部技術文件**（`docs/<slug>/internal/`）：給團隊自己用，涵蓋 PRD、Feature Spec、RFC、System Architecture、Data Flow。

## 如何產出新文件

使用 `/olap-doc-workflow` 這個 skill（定義於 `.claude/skills/olap-doc-workflow/`）。它會引導一段 0-10 階段的對話——從脈絡挖掘、PRD 骨架、功能細化、邊界審查、非功能需求、治理審查、會簽，到衍生出 System Architecture / Data Flow / RFC / 對外文件——並把產出依序存進 `docs/<slug>/`。

每個功能/案子用一個英文 kebab-case slug 命名，例如 `docs/multi-tenant-quota/`。同一個 slug 底下的進度會被 `00-progress.md` 追蹤，之後可以直接呼叫 skill 接續未完成的階段。

## 目錄結構
```
docs/<slug>/
  00-progress.md
  internal/
    PRD.md
    system-architecture.md
    data-flow.md
    feature-specs/<feature-slug>.md
    rfc/RFC-<NNN>-<topic-slug>.md
  external/
    service-application.md
    usage-guide.md
    limits-and-quotas.md
    service-spec.md
```
