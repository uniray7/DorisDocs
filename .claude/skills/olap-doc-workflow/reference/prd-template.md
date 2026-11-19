# PRD 範本：<服務/功能名稱>

> 來源脈絡：`docs/00-progress.md` 現況摘要｜本檔位置：`docs/internal/prd/README.md`

## 1. 背景與問題陳述
- 現況痛點是什麼？現在大家怎麼繞過這個問題？
- 為什麼現在要做（時機、驅動力）？

## 2. 目標與非目標
- 目標（Goals）：這次要達成什麼，可衡量的話最好。
- 非目標（Non-Goals）：明確排除的範圍，避免範圍蔓延。

## 3. 目標使用者與使用情境
- 使用者角色（哪些部門、什麼職能）
- 代表性使用情境（2-3 個）

## 4. 術語表（階段 1）
| 名詞 | 定義 |
|------|------|

## 5. 範圍（In-scope / Out-of-scope）
- In-scope：
- Out-of-scope：

## 6. 核心功能大綱
> 階段 1 先列清單，階段 2 每項展開成同資料夾的 `<feature>.md`，此處放連結。

1. [功能 A](feature-a.md)
2. [功能 B](feature-b.md)

## 7. 多租戶與資源隔離模型
- 隔離粒度（cluster / database / table / row）
- 租戶間 metadata 可見性
- Noisy neighbor 防護機制

## 8. 非功能需求（階段 3）
| 項目 | 目標值 | 備註 |
|------|--------|------|
| 可用性 | | |
| 查詢延遲 P95/P99 | | |
| 寫入吞吐 | | |
| 容量規劃/擴容機制 | | |
| 配額/計費模式 | | |
| 備援/DR | | |

## 9. 治理與合規（階段 3）
- 資料分類（是否含 PII）
- 存取控制模型（RBAC 設計）
- 稽核紀錄需求
- 安全審查結論

## 10. 里程碑與時程
| 里程碑 | 日期 | 說明 |
|--------|------|------|

## 11. 待決事項與會簽（階段 4）
> 待決事項統一登記在 `docs/00-progress.md` 的 TBD 清單，這裡只放連結；會簽結果記到 `docs/decision-history.md`。

- 本 PRD 相關的 TBD：TBD-<編號>、…

## 12. 衍生文件索引（階段 5）
- System Architecture：`../architecture/system-architecture.md`
- Data Flow：`../architecture/data-flow.md`
- RFC：`../rfc/`
- 對外文件：`../../external/`
