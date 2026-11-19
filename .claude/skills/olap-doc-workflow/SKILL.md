---
name: olap-doc-workflow
description: 引導 Apache Doris OLAP 平台文件的產出與維護：內部文件（PRD 與各 feature 細節、RFC、architecture）與對外文件（服務申請、使用手冊、使用限制、Service Spec），以 5 個階段推進、按文件追蹤狀態，允許保留 TBD 交給 PM／工程師後續研究。當使用者要開始、繼續或更新 OLAP／Doris 平台文件、合併新的 PRD 草稿，或明確輸入 /olap-doc-workflow 時使用。
---

# OLAP Platform 文件 Workflow

讀者是 PM、工程師與管理層。目標是讓文件**精簡、只寫現況、找得到**。還沒研究或還沒定案的主題標成 TBD，留給 PM／工程師後續處理，不必卡住整體進度。

## 目錄結構

```
README.md                    交接入口：依讀者列閱讀路徑（不放內容）
docs/
  00-progress.md             現況摘要 + TBD 清單 + 文件狀態
  decision-history.md        決議演進（含被推翻的決議與原因）
  product-overview.md        管理層一頁版
  internal/
    prd/
      README.md              PRD 本體：背景、範圍、術語、模式對照表、feature 索引、NFR、治理
      <feature>.md           各 feature 的細節
    rfc/
      RFC-<NNN>-<topic>.md   有取捨的決策；可附 RFC-<NNN>-decision-brief.md 給決策者
    architecture/
      system-architecture.md
      data-flow.md
  external/
    service-application.md   如何申請、審核流程
    usage-guide.md           使用手冊
    limits-and-quotas.md     使用限制
    service-spec.md          Service Spec／SLA
```

## 啟動

1. 讀 `docs/00-progress.md`，簡短回報：各文件狀態、TBD 數量、正在阻擋其他文件的 TBD。
2. 問使用者這次要做什麼：推進某份文件或某個階段、處理某個 TBD，或合併新資料（例如新版 PRD 草稿）。**不要自行決定下一步。**
3. `docs/00-progress.md`、`docs/decision-history.md` 不存在時，分別用 `reference/progress-template.md`、`reference/decision-history-template.md` 建立。

## 寫作規範

1. **專有名詞不硬翻，直接用英文**。例：workspace、fab、cluster、tier、managed／self-managed、pipeline、staging、quota、zone（tmp／raw／curated）、DDL、CCR、workload group、owner、reader、grant／revoke、privilege。中文用在說明句。
2. **只寫現況**：正文不寫「某日補充」、不留刪除線、不保留被推翻的方案。決議變更與原因記到 `docs/decision-history.md`；被否決的選項只留在 RFC。
3. **單一來源**：同一張表、同一個定義只放一處，其他文件用連結。例：模式對照表在 `prd/README.md`，tier 表在 `prd/cluster-topology-and-tiering.md`。
4. **精簡優先**：能用表格就不寫段落；一份 feature 文件只談一個 topic。
5. **對外文件**：面向不熟 Doris 內部實作的使用者，只講行為、限制與責任歸屬，不放內部架構與術語。
6. **語言**：繁體中文為主，不維護英文版。

## TBD 機制

- 未研究或未定案的內容，在原處標記：
  `> TBD-<編號>｜待研究：<問題>｜建議負責：PM／工程師／主管`
- 每個 TBD 同步登記在 `docs/00-progress.md` 的 TBD 清單，全 repo 只用這一套編號。
- TBD 清單的「被引用」欄列出提到該 TBD 的所有文件；新增或移除引用時同步更新（可用 `grep -rn "TBD-<n>" docs` 核對）。
- **只影響單一文件的細項**（數值、流程細節）不另編 TBD：在原處標 ⚠️，並列在該文件的「待確認事項」；`00-progress.md` 的「各文件的細項待確認」只放連結。
- **有 TBD 也可以往下一個階段走**。只有其他文件的前提才需要先定案（例：計費模式影響 NFR 的配額章節），這種 TBD 在清單標「阻擋：<文件>」。
- TBD 定案時：把原處改寫成現況、移除標記、清單標「已解決」，並在 `decision-history.md` 記一筆。

## 階段

| # | 階段 | 要做什麼 | 產出位置 |
|---|------|----------|----------|
| 1 | 定義 | 確認先做內部或對外文件（建議先內部，對外文件由 PRD 衍生）；一次問 3–5 個開放式問題挖脈絡（痛點、目標使用者、多租戶隔離、資料敏感度、基礎建設限制、時程）；依 `reference/prd-template.md`、`reference/glossary-template.md` 產出 PRD 大綱、術語、In／Out-of-scope | `00-progress.md` 現況摘要、`prd/README.md` |
| 2 | Feature 細化 | 一次一個 feature，依 `reference/feature-spec-template.md` 展開；再以 Tech Lead＋QA 視角跑 `reference/edge-case-checklist.md`，結論直接補進文件 | `prd/<feature>.md` |
| 3 | NFR 與治理 | 可用性、延遲、吞吐、容量、配額與計費、DR；資料分類／PII、授權模型、稽核。數字未定就標 TBD | `prd/README.md` NFR、治理章節 |
| 4 | 定案 | 有取捨的決策寫 RFC（依 `reference/rfc-template.md`）；列出會簽角色（Infra、Security、成本 owner、使用者代表），記「誰、何時、同意什麼」 | `rfc/`、`decision-history.md` |
| 5 | 產出與發布 | 架構文件與對外文件（開頭標「來源：prd/README.md § X」）；找 1–2 個內部團隊照對外文件試點；訂版號、review 週期、公告管道 | `architecture/`、`external/`、`00-progress.md` |

- 階段 1、2 依序進行；**階段 3–5 可不照順序、可由不同人分頭進行**。
- 進度**按文件追蹤**，不追蹤全專案的階段編號。

## 文件狀態

每份文件開頭標狀態，並與 `00-progress.md` 的文件狀態表保持一致：

| 狀態 | 意思 |
|------|------|
| 草稿 | 未經評審，可能含 TBD |
| 評審中 | 已送相關人評審，等確認 |
| 定案 | 已會簽；之後的變更要記到 `decision-history.md` |

## 外部文件查核

**時機**：合併新資料、階段 2 邊界審查、階段 4 定案之前，先查核文件中的技術說法，再改寫內容。

### 兩類事實

| 類型 | 例子 | 依據 |
|------|------|------|
| 決策與環境事實 | 版本、部署方式、服務模式、HA 決議 | 團隊決議或口述（記在 `decision-history.md`），不用外部文件查證 |
| 技術行為 | 某個 privilege 涵蓋哪些操作、某參數的效果、某版本是否有 bug | 依下方信任度分級查證 |

### 信任度分級（技術行為）

| 等級 | 來源 | 用法 |
|------|------|------|
| L1 | Doris、Nomad 官方文件與 source code | 最高依據；文件與 source code 不一致時以 source code 為準 |
| L2 | 官方 release notes／changelog、官方 GitHub issue 與 PR（maintainer 回覆） | 確認行為在哪個版本加入、修正或改變 |
| L3 | 商業發行版文件（如 VeloDB／SelectDB）、雲廠商文件、技術部落格、社群問答 | 只當線索，版本與設定可能不同 |
| L4 | 沒有來源的推論（含 AI 推論） | 必須標「推論」，不能當結論 |

- **實機驗證**：在公司環境、目標版本上的實測結果，是技術行為的最終裁定；關鍵行為即使有 L1 依據也標「需實機驗證」。
- **內部規格**（如 DB2 CDC format）：對內部格式而言即為權威，不套用分級。

### 規則

1. **版本對齊**：引用要對應目前版本（版本記在 `docs/00-progress.md` 現況摘要）；dev 版文件可能描述尚未發布的功能，引用時註明。
2. **引用格式**：`[L1] 標題｜URL｜適用版本｜查閱日期`。
3. **衝突處理**：不同等級說法不一時，高等級優先；同等級說法不一時列為 TBD，交實機驗證。
4. **產出**：每次查核列出「與外部文件矛盾」「有風險」「找不到依據（推論）」三類，各附來源等級；結論先給使用者確認，再改文件。

## 合併新資料

收到新的 PRD 草稿或其他外部文件時：

1. 先列出它與現有文件的**衝突**（舊說法 → 新說法、影響哪些文件），以及它自己列的待決事項；技術說法依「外部文件查核」標出矛盾與風險。
2. 衝突**逐項請使用者確認**採哪一邊，不要自行判定新版就是定案。
3. 確認後改寫對應文件；被推翻的舊決議記到 `decision-history.md`；未定的部分登記 TBD。

## 操作原則

- **服務位於 private network，這個環境無法連線**：不要嘗試連 Doris cluster 或呼叫服務 API。服務現況以使用者口述為準，不確定就問，或標 TBD。
- 修改前先讀既有內容，用增補而非整份覆蓋，保留使用者手動修改過的內容。
- **任何改動先給使用者看 diff，確認後才 commit／push。**
- 模板在 `reference/`，依當次脈絡填寫，不要照抄佔位文字。
