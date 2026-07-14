# RFC-002：Shared Cluster 多租戶資源隔離機制

- 狀態：草稿
- 作者：平台團隊
- 日期：2026-07-13
- 相關 PRD：`internal/PRD.md § 7`、`feature-specs/workspace-isolation.md § 運算資源隔離`
- 環境前提：Apache Doris **4.1.0**

## 背景

Tier 1（shared cluster，僅 managed 模式）讓多個 workspace 共住同一座 Doris cluster。若不做運算資源隔離，任一租戶的重查詢/爆量會直接劣化鄰居的查詢延遲，違反「小規模團隊也能安心用 shared tier」的產品承諾，也會讓「效能不佳歸咎平台」的糾紛無從釐清。

不做的後果：shared tier 不可信 → 所有租戶被迫申請 dedicated → 硬體池撐不住、tier 1 形同虛設。

## 待決問題

Shared cluster 上，用哪一組機制、怎麼組合，來保證單一租戶的行為不明顯影響其他租戶的查詢體驗（P95/P99）？

## 選項比較

| 選項 | 內容 | 優點 | 缺點 | 成本/風險 |
|------|------|------|------|-----------|
| A：僅 Workload Group | 每 ws 一個 workload group（CPU/記憶體/併發） | 零額外元件，Doris 原生 | 無 QPS 配額能力；軟性 CPU 限制保不住 P99；IO 與 runaway query 無防護 | 低成本但隔離不完整，客訴風險高 |
| B：多層防線（Workload Group 硬限制 + Gateway + Policy + 寫入側治理） | 詳見下方建議方案 | 覆蓋准入/執行/兜底/寫入全鏈路；QPS 計量落地；gateway 順帶統一端點 | 需自建 gateway 元件；參數調校工作量 | 中等成本；gateway 成為新的單點需 HA |
| C：Resource Tag 實體隔離 | 每 ws 的資料副本釘到專屬 BE 群組 | 隔離最強（page cache/compaction 皆分離） | 每 ws 至少 3 BEs → pool 利用率大降，tier 1 小租戶不符成本 | 高硬體成本，違背 shared tier 存在目的 |

## 建議方案：選項 B（多層防線），C 保留為升級選項

### 第 1 層：查詢准入
- Workload Group（per-ws）：`max_concurrency`、`max_queue_size`、`queue_timeout`——超併發先排隊，瞬間爆量兜底（對齊 Max QPS 5 分鐘滑動窗口決議）。
- **Gateway/proxy（平台自建）**：per-ws QPS 計量與限流（Doris 無原生 QPS 配額）；統一連線端點（tier 遷移、failover 不換線）；集中連線數控管。

### 第 2 層：執行資源
- CPU：`cpu_hard_limit`（**硬限制**；不採 `cpu_share` 軟限制——忙時按比例分無法保障 P99）。
- 記憶體：`memory_limit` + `enable_memory_overcommit=false` + spill to disk（大查詢落盤，不 OOM 拖垮 BE）。
- 掃描 IO：`read_bytes_per_second` 限速 + scan 線程數上限（防單租戶吃光磁碟頻寬）。

### 第 3 層：防呆兜底
- Workload Schedule Policy：自動 kill 超時/掃描超標查詢（如：執行 > N 秒、掃描 > M rows 即殺——參數隨 tier 1 規格定）。
- SQL Block Rule：限制單查詢可掃 partition/tablet 數，強迫帶分區條件；規則同步寫入對外使用規範。
- Per-ws `max_user_connections`、`query_timeout`。

### 第 4 層：寫入側治理（本平台架構的天然優勢）
Shared cluster 僅 managed、寫入唯一通道是平台 pipeline，因此：
- Per-ws ingestion 限速做在 pipeline consumer（Kafka 消費速率），不依賴 Doris 端擋。
- Merge SQL 平台代管排程 → 可錯峰，避免多 ws merge 對撞。
- 批次大小/頻率統一治理 → 從源頭控制 compaction 壓力（高頻小批量寫入 → compaction 積壓 → 全 cluster 變慢，是 shared 最常見的隱形殺手）。

### 第 5 層（選配/升級路徑）：Resource Tag
Tier 1 預設不用。保留兩個適用場景：
1. Shared 內出現「接近 tier 2 門檻但尚未升級」的大租戶，先用 tag 隔離止血。
2. 作為 shared → dedicated 的中繼形態（tag 群組直接轉出成 dedicated cluster）。

### Tier 1 建議初始參數（草案，隨 Phase 5 配額定案調整）

| 參數 | 初始值（待調） | 備註 |
|------|---------------|------|
| cpu_hard_limit | 依「shared cluster 總核數 ÷ 預計共住 ws 數」+ 少量超賣 | 超賣比例待容量規劃 |
| memory_limit | 同上邏輯 | overcommit=false |
| max_concurrency | 依 tier 1 Max QPS（<20）與平均查詢時長推算 | |
| read_bytes_per_second | 依 SSD 總頻寬與共住數推算 | |
| kill 門檻（時長/掃描量） | query_timeout 上限的 1.5 倍；掃描 rows 依 tier 1 資料量上限推算 | |

## 已知限制（無法完全隔離的共用點）
- FE：metadata 操作、連線管理為全 cluster 共用。
- BE page cache：跨租戶共用，無法分割。
- Compaction 線程池：全域共用（第 4 層從寫入源頭緩解）。

結論：shared tier 的隔離是「工程上足夠好」而非絕對；tier 1 使用規範保守（shared 規範較緊原則）+ 明確的 dedicated 升級路徑，是機制之外的必要配套。

## 影響範圍
- 新增平台元件：gateway/proxy（需 HA 設計，納入 system-architecture）。
- 佈建原語 `provision_ws` 需擴充：建立 workload group（含全部參數）、gateway 註冊、policy 掛載。
- 對外文件：SQL Block Rule 與 kill 門檻須寫入 managed 使用規範（使用者要知道什麼查詢會被殺）。
- 監控：per-ws workload group 指標（排隊數、kill 數、限速觸發次數）進 quota-and-usage 儀表板。

## 待驗證項（實作時）
- [ ] Doris 4.1.0 各參數的實際行為與文件一致（尤其 cpu_hard_limit 與 IO 限速的組合效果）
- [ ] Workload group 數量上限（單 cluster 可承載的 ws 數）
- [ ] Spill to disk 對 shared SSD 的 IO 反噬程度
- [ ] Gateway 的 QPS 計量與 Doris 端 workload group 指標的一致性
- [ ] **Arrow Flight protocol 的涵蓋**（2026-07-14 新增）：平台提供 MySQL + Arrow Flight 雙協定，gateway 以 MySQL protocol proxy 為前提；Arrow Flight 走 gRPC 且資料面直連 BE——gateway 能否代理？不能的話 Arrow Flight 查詢的 QPS 計量與限流走哪條路（Doris audit log 事後計量？workload group 端承擔？）

## 決策紀錄
| 日期 | 決議 | 決策者 |
|------|------|--------|
| （待會簽） | | |
