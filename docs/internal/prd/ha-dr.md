# HA／DR

> 來源：[PRD](README.md) § 核心功能大綱｜狀態：骨架｜中長期做法與 DR 由接手的 PM 與工程師研究（TBD-9、TBD-36）

## 目前的 HA 做法（2026-11-19 決議）

- 每座 cluster 部署在多台機器上，**每個 table 開 3 個 replica**。單一機器 crash 時，cluster 仍可容錯、持續服務。
- **不採用** active-standby 雙 cluster＋CCR 的架構。
- **中長期目標**：讓機器與 table replica 分散到 **3 個機櫃，甚至 3 個 datacenter**，以承受機櫃或機房層級的故障；怎麼做待研究（TBD-9）。
- 3 replica 擋的是機器故障，**擋不住**誤刪、灌錯資料等邏輯錯誤（所有 replica 會一起套用），這類情況要靠 DR。

## 目前的 DR 方向

- 未來需要 **backup／restore 機制，備份存放到 S3 或 MinIO**。
- 範圍、頻率、保留期、還原流程與演練、managed 與 self-managed 是否都提供 → TBD-36。
- 同一套 backup／restore 也是資料搬遷的候選做法（[data-migration.md](data-migration.md)）。

## 為什麼需要 HA（High Availability）

HA 的目標是**服務不中斷**：單一元件或節點故障時，使用者仍能查詢與寫入。

- **平台是共用基礎設施**：一座 cluster 可能承載多個 workspace（shared cluster）或一個部門的核心報表。cluster 停擺時，受影響的是所有依賴它的團隊與下游應用。
- **故障是常態，不是例外**：機器、磁碟、網路、程序都會出問題；服務規模越大（全平台預計 20–40 座 cluster），某處出狀況的機率越高。
- **使用者從 Oracle／HBase 轉移過來**：原有系統已有一定的可用性水準，新平台不能明顯退步，否則會阻礙轉移。
- **對外承諾的基礎**：要寫出 SLA（可用性目標、維護窗口、故障時的影響），必須先有 HA 設計支撐。

## 為什麼需要 DR（Disaster Recovery）

DR 的目標是**資料救得回來**：發生重大事故或人為錯誤後，能把資料與服務恢復到可接受的狀態。

- **HA 救不了所有事**：HA 通常靠同步或副本，誤刪 table、灌錯資料、錯誤的 DDL 會一起同步過去。這類邏輯錯誤只能靠備份或時間點還原救回。
- **重大事故**：整座 cluster 或機房層級的故障、資料損毀，需要有在別處重建的能力。
- **責任邊界需要明文**：managed 模式由平台承擔資料管理責任；self-managed 的資料責任在使用者。平台救什麼、不救什麼，都要依 DR 能力寫清楚。
- **事故後要能證明與追溯**：還原到哪個時間點、遺失多少資料（RPO）、多久恢復（RTO），是事後溝通與究責的依據。

## 待研究（交接手的 PM 與工程師）

| 問題 | TBD |
|------|-----|
| 中長期：機器與 replica 分散到 3 個機櫃或 3 個 datacenter 的做法（replica 擺放、機器與網路規劃） | TBD-9 |
| 可用性目標、RPO、RTO 的數字 | TBD-9 |
| Backup／restore 機制（S3 或 MinIO）的範圍、頻率、保留期、還原流程與演練 | TBD-36 |
