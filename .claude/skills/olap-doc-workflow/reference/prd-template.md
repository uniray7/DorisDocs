# PRD 範本：<服務/功能名稱>

> 來源脈絡：docs/<slug>/00-progress.md（Phase 1 脈絡摘要）

## 1. 背景與問題陳述
- 現況痛點是什麼？現在大家怎麼繞過這個問題？
- 為什麼現在要做（時機、驅動力）？

## 2. 目標與非目標
- 目標（Goals）：這次要達成什麼，可衡量的話最好。
- 非目標（Non-Goals）：明確排除的範圍，避免範圍蔓延。

## 3. 目標使用者與使用情境
- 使用者角色（哪些部門、什麼職能）
- 代表性使用情境（2-3 個）

## 4. 術語表（Phase 2.5 補上）
| 名詞 | 定義 |
|------|------|

## 5. 範圍（In-scope / Out-of-scope）
- In-scope：
- Out-of-scope：

## 6. 核心功能大綱
> Phase 2 先列清單，Phase 3 每項展開成 `internal/feature-specs/<feature-slug>.md`，此處放連結。

1. [功能 A](feature-specs/feature-a.md)
2. [功能 B](feature-specs/feature-b.md)

## 7. 多租戶與資源隔離模型
- 隔離粒度（cluster / database / table / row）
- 租戶間 metadata 可見性
- Noisy neighbor 防護機制

## 8. 非功能需求（Phase 5 補上）
| 項目 | 目標值 | 備註 |
|------|--------|------|
| 可用性 | | |
| 查詢延遲 P95/P99 | | |
| 寫入吞吐 | | |
| 容量規劃/擴容機制 | | |
| 配額/計費模式 | | |
| 備援/DR | | |

## 9. 治理與合規（Phase 6 補上）
- 資料分類（是否含 PII）
- 存取控制模型（RBAC 設計）
- 稽核紀錄需求
- 安全審查結論

## 10. 里程碑與時程
| 里程碑 | 日期 | 說明 |
|--------|------|------|

## 11. 待決事項與會簽紀錄（Phase 7）
| 事項 | 負責人 | 狀態 | 備註 |
|------|--------|------|------|

## 12. 衍生文件索引（Phase 8 完成後補上）
- System Architecture：`internal/system-architecture.md`
- Data Flow：`internal/data-flow.md`
- RFC：`internal/rfc/`
- 對外文件：`external/`
