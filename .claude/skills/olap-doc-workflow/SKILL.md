---
name: olap-doc-workflow
description: 引導 0-10 階段的文件產出流程，為 Apache Doris OLAP 平台服務產出內部文件（PRD、RFC、Feature Spec、System Architecture、Data Flow）與對外文件（服務申請、審核流程、使用手冊、使用限制規範、Service Spec）。當使用者要開始/繼續/恢復撰寫 OLAP／Doris 服務相關文件，或明確輸入 /olap-doc-workflow 時使用。
---

# OLAP Service 文件產出 Workflow

這個 skill 引導一段「脈絡 → PRD → 衍生文件」的多階段對話，把使用者腦中的服務構想，逐步收斂成內部技術文件與對外使用者文件。每個階段都要**產出實體檔案**並徵求使用者確認後才進下一階段，不要一次把全部文件生完。

## 0. 啟動：辨識案子 (slug) 與進度

- 這個 skill 可能被呼叫多次，處理不同的功能／案子（例如「多租戶配額」「Stream Load 資料匯入」「服務申請入口」）。第一次執行時，向使用者確認一個英文 kebab-case slug，例如 `multi-tenant-quota`。
- 所有產出都放在 `docs/<slug>/` 底下，結構如下：
  ```
  docs/<slug>/
    00-progress.md              # 階段進度與決策紀錄（Decision Log）
    internal/
      PRD.md
      system-architecture.md
      data-flow.md
      feature-specs/<feature-slug>.md   # Phase 3 每個核心功能一份
      rfc/RFC-<NNN>-<topic-slug>.md     # Phase 8，有爭議或需記錄替代方案的技術決策
    external/
      service-application.md      # 如何申請、審核流程
      usage-guide.md               # 使用手冊
      limits-and-quotas.md         # 使用限制規範
      service-spec.md              # 服務 Spec / SLA
  ```
- 若 `docs/<slug>/00-progress.md` 不存在，用 `reference/progress-template.md` 建立一份。
- 若 `docs/<slug>/00-progress.md` 已存在，讀取它，找出「目前完成到哪個 Phase」，直接從下一個未完成的 Phase 接續，並跟使用者說明「偵測到既有進度，從 Phase X 接續」。
- 若使用者要求跳過某個 Phase，可以照做，但要明確提醒被跳過的 Phase 可能造成後續文件的缺口，並在 `00-progress.md` 記一筆。

## 階段總覽

| # | Phase | 目的 | 主要輸出 |
|---|-------|------|----------|
| 0 | 定位分流 | 確認這次驅動的是內部文件還是對外文件，決定後續側重點 | `00-progress.md` 開頭的分流紀錄 |
| 1 | 脈絡傾倒 | 挖出痛點、目標、多租戶等核心限制 | `00-progress.md` 脈絡摘要 |
| 2 | 骨架生成 | 產出符合標準產品規格的 PRD 大綱 | `internal/PRD.md`（大綱版） |
| 2.5 | 名詞與範圍對齊 | 定義術語表、劃出 Out-of-scope 邊界 | `internal/PRD.md` 附錄／術語表段落 |
| 3 | 分塊細化 | 逐一拆解 User Story、前端互動、API 邏輯 | `internal/feature-specs/*.md` |
| 4 | 邊界審查 | Tech Lead／QA 視角挑 Edge Case 與安全漏洞 | 補強 `feature-specs/*.md`、`PRD.md` |
| 5 | 非功能需求／SLA | 量化可用性、延遲、容量、配額、計費模式 | `PRD.md` NFR 段落 |
| 6 | 治理與合規審查 | 資料分類、存取控制、稽核、安全審查 | `PRD.md` 治理段落 |
| 7 | 利害關係人會簽 | Infra／Security／財務／目標用戶代表過一輪 | `00-progress.md` Decision Log |
| 8 | 衍生文件產出 | 從已核准 PRD 分岔出所有下游文件 | `system-architecture.md`、`data-flow.md`、`rfc/*.md`、`external/*.md` |
| 9 | 試點驗證 | 找真實團隊走一次流程，回饋修正 | 更新對應文件 + `00-progress.md` 回饋紀錄 |
| 10 | 發布與版本治理 | 訂版號、下次 review 週期、公告管道 | `00-progress.md` 發布紀錄 |

---

## Phase 0：定位分流

問使用者：這次要先產出「內部文件（PRD/RFC/Feature Spec）」還是「對外文件（申請/使用手冊）」，或是兩者都要、從內部先做（建議路徑，因為對外文件的內容大多衍生自 PRD）。把結論寫進 `00-progress.md`。

## Phase 1：脈絡傾倒

用開放式問題引導使用者說出：
- 這個服務／功能要解決的**痛點**是什麼？現況怎麼做（如果有）？
- **目標使用者**是誰（哪些部門、什麼角色）？預期規模？
- **多租戶隔離**要求：資源隔離到什麼粒度（cluster/database/table）？租戶間可否互相看到 metadata？
- 資料的**敏感等級**（是否含 PII）？
- 現有的**基礎建設限制**（機器資源、既有 Doris 版本、既有 ingestion 工具）？
- 這次的**時程與優先順序**壓力？

不要用是非題列表丟給使用者，一次問 3-5 題，追問模糊處。結束後把脈絡整理成條列摘要寫進 `docs/<slug>/00-progress.md`，請使用者確認無誤再進 Phase 2。

## Phase 2：骨架生成

依 `reference/prd-template.md` 產出 PRD **大綱版**（章節標題 + 每節 1-2 句話說明要填什麼，先不寫細節），寫入 `internal/PRD.md`。跟使用者核對大綱的章節順序、有沒有缺漏的核心功能，再進下一階段。

## Phase 2.5：名詞與範圍對齊

用 `reference/glossary-template.md`，跟使用者一起定義關鍵名詞（cluster、tenant、workspace、quota、ingestion job 等），並明確列出 **In-scope / Out-of-scope**。這段內容附加到 `PRD.md` 的術語表章節。這一步的目的是避免 Phase 3 分塊細化時，不同功能段落對同一個詞有不同解讀。

## Phase 3：分塊細化

針對 Phase 2 大綱中列出的每個核心功能，逐一（不要一次全部展開，一次處理一個功能）用 `reference/feature-spec-template.md` 拆解：
- User Story（作為 [角色]，我想要 [動作]，以便 [價值]）
- 前端互動流程（如果有 UI：申請表單、審核 console、監控儀表板等）
- API 邏輯（endpoint、request/response、狀態機、錯誤碼）

每個功能存成 `internal/feature-specs/<feature-slug>.md`，並在 `PRD.md` 大綱對應章節加上連結。

## Phase 4：邊界審查

切換視角為資深 Tech Lead + QA，對 Phase 3 產出的每份 feature spec 過一輪 `reference/edge-case-checklist.md`（涵蓋多租戶隔離失效、配額超限行為、SQL 注入、ingestion pipeline 逾時/背壓、schema 演進衝突、冪等性等 OLAP/Doris 常見情境）。把挑出的 Edge Case 與對應處理方式直接補寫回 feature spec，不要只列問題不給結論。

## Phase 5：非功能需求／SLA

明確量化：可用性目標（如 99.9%）、查詢延遲 P95/P99、寫入吞吐、容量規劃與擴容機制、配額與計費模式（chargeback/showback 或免費內部服務）、備援與 DR 策略。寫入 `PRD.md` 的 NFR 章節。這段內容會直接餵給 Phase 8 的 System Architecture 與對外 `limits-and-quotas.md`。

## Phase 6：治理與合規審查

確認資料分類（是否含 PII/機敏資料）、存取控制模型（RBAC 設計）、稽核紀錄需求、與既有安全審查流程的銜接（可套用 repo 內既有的 `security-review` skill 概念做一輪檢查）。寫入 `PRD.md` 治理章節。

## Phase 7：利害關係人會簽

列出需要會簽的角色（Infra owner、Security、財務/成本owner、代表性目標用戶），整理待決事項成一份簡短的 Decision Log，附上「已解決」與「待解決」清單，寫入 `00-progress.md`。若使用者说某些角色已经口头同意，仍要記錄「誰、何時、同意什麼」。

## Phase 8：衍生文件產出

PRD 核准後，從 PRD 各章節分岔出下游文件，並在每份文件開頭標註「來源：PRD.md § X」以利日後追溯：

- `internal/system-architecture.md`（用 `reference/system-architecture-template.md`）
- `internal/data-flow.md`（用 `reference/data-flow-template.md`）
- `internal/rfc/RFC-<NNN>-<topic-slug>.md`（用 `reference/rfc-template.md`；只針對 Phase 7 中有爭議、有替代方案取捨的技術決策寫 RFC，不是每個功能都要）
- `external/service-application.md`（用 `reference/external-service-application.md`）
- `external/usage-guide.md`（用 `reference/external-usage-guide.md`）
- `external/limits-and-quotas.md`（用 `reference/external-limits-spec.md`，內容衍生自 Phase 5 的 NFR）
- `external/service-spec.md`（用 `reference/external-service-spec.md`）

對外文件語氣要面向「不熟悉 Doris 內部實作的其他部門使用者」，避免直接貼內部術語或架構圖，只講使用者需要知道的行為與限制。

## Phase 9：試點驗證

建議找 1-2 個真實內部團隊，依照剛產出的 `external/service-application.md` 與 `usage-guide.md` 實際走一次申請/使用流程。把回饋（卡關點、遺漏的邊界情況）記錄到 `00-progress.md`，並回頭修正對應文件（可能需要回到 Phase 4 或 Phase 8）。

## Phase 10：發布與版本治理

跟使用者確認：文件版號規則、下次 review 週期（建議跟 Doris 版本升級或季度對齊）、對外文件的公告/教育訓練管道。把結論寫入 `00-progress.md` 並標記整個 workflow 完成。

---

## 操作原則

- **服務位於 private network，這個環境無法連線**：不要嘗試連線 Doris cluster、呼叫服務 API、或驗證 endpoint 是否可達。所有關於服務現況的事實（版本、規格、行為）都以使用者口述為準，不確定就問使用者，不要自行測試或猜測。
- 每個 Phase 結束前，簡短總結產出並明確問「確認進入下一階段？」，不要自動連續跑完多個 Phase。
- 每次寫檔案前用 Read 工具讀取既有內容（若存在），用 Edit 增補而非整份覆蓋，保留使用者手動修改過的內容。
- 語言：跟隨使用者輸入的語言（預設繁體中文），技術詞彙可中英夾雜。
- 模板放在 `reference/` 底下，讀取後依當次脈絡填寫，不要照抄佔位符文字進最終文件。
