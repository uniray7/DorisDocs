# DorisDocs

公司內部 OLAP 平台（以 Apache Doris 為基礎）的文件庫，涵蓋服務設計、授權、資料寫入（ingestion）、營運支援與對外使用文件。

> ⚠️ 目錄整理中：文件正依 [`improvement-plan.md`](docs/improvement-plan.md) 搬到下方新結構，搬移完成前部分連結會暫時失效。

## 從哪裡開始讀

| 你是 | 先讀 | 接著讀 |
|------|------|--------|
| 管理層 | [產品摘要](docs/product-overview.md) | [RFC](docs/internal/rfc/) 的決策簡報 |
| PM | [現況與 TBD 清單](docs/00-progress.md) | [PRD 總覽](docs/internal/prd/README.md) → [決議演進](docs/decision-history.md) |
| 工程師 | [PRD 總覽](docs/internal/prd/README.md) | 各 feature 文件 → [RFC](docs/internal/rfc/) → [架構](docs/internal/architecture/) |
| 平台使用者 | [對外文件](docs/external/)（階段 5 產出） | — |

## 目錄

```
docs/
  00-progress.md        現況摘要 + TBD 清單 + 文件狀態
  decision-history.md   決議演進
  product-overview.md   管理層一頁版
  internal/
    prd/                PRD 總覽（README.md）與各 feature 細節
    rfc/                有取捨的技術／產品決策
    architecture/       系統架構、data flow
  external/             對外文件：申請、使用手冊、使用限制、Service Spec
```

## 文件規則

- **只寫現況**：被推翻的決議記在 `decision-history.md`，正文不保留。
- **TBD**：未定案或未研究的主題標 `TBD-<編號>`，統一登記在 `00-progress.md`，由 PM／工程師後續處理。
- **專有名詞用英文**（workspace、fab、cluster、tier、managed／self-managed 等），不硬翻。
- **同一件事只寫一處**，其他文件用連結。

## 如何更新文件

在 Claude Code 使用 `/olap-doc-workflow`（定義於 `.claude/skills/olap-doc-workflow/`）。它會讀取 `00-progress.md`、回報各文件狀態與 TBD，再依你指定的文件、階段或 TBD 繼續。任何改動都會先給 diff 確認，再 commit。
