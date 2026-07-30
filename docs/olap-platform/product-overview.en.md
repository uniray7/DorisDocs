# OLAP Platform Product Planning Summary

> Audience: Management & PM | Date: 2026-07-13 | Status: Planning (PRD refinement phase, feature spec drafts 10/10 complete)
> Detailed specs: see `internal/PRD.md` and `internal/feature-specs/`; this document is a decision-oriented summary.

---

## 1. What We're Building

Based on **Apache Doris**, we will provide a **multi-tenant OLAP analytics service** for internal company use, along with a **data ingestion pipeline** so that departments can load their data in for analysis.

**Why**: To eliminate Oracle license costs and lock-in, and to sunset the legacy HBase tech stack. Existing analytical data is roughly hundreds of TB in scale.

**Explicitly out of scope**: Migration of existing data (each team handles this on its own), cross-department data sharing (handled at the lakehouse layer), transactional (OLTP) workloads, and BI tool hosting.

---

## 2. Product Shape: Two Service Modes

This is the most central design decision in the whole product — **the boundary of responsibility is drawn by service mode**, chosen at application time and **not convertible afterward**:

| | **Managed** | **Self-managed** |
|---|---|---|
| Best for | Teams that want convenience and are willing to follow platform conventions | Teams with highly custom needs, high traffic, and the capability to self-manage |
| Data ingestion | Only via the platform pipeline (streaming CDC / batch) | Always via a self-built pipeline written by the team itself |
| Table creation/schema changes | Executed by the platform system after governance approval — **governance model still to be decided** (Option A: workspace owner approves, platform does not review; Option B: platform reviews and gatekeeps) | **Always submitted for platform-assisted creation** (validation + injection of protective settings; schema design itself is not reviewed; strict at first, may be relaxed later depending on dbt needs — RFC-003) |
| Data management responsibility | **Owned by the platform**: backup/restore, retention/purge, performance tuning; whether schema quality gatekeeping falls under platform responsibility is still to be decided (tied to the governance model) | **Owned by the user** |
| Platform support | Full support | Day-to-day infra alerting + assisted operations (scale in/out) + HA (dual-cluster CCR + failover assistance); other requests are submitted and prioritized by the PM (not an obligation) |
| Available specs | Shared or Dedicated cluster | Dedicated cluster only (Tier 2 and above) |

## 3. Resource Tiers

| Tier | Data Volume | Peak QPS | Configuration | Available Modes |
|------|--------|----------|------|----------|
| 1 | < 500GB | < 20 | Shared cluster (resource-isolated) | Managed only |
| 2 | 500GB–5TB | 20–100 | Dedicated small cluster (3 BEs) | Both |
| 3 | 5TB–30TB | 100–500 | Dedicated medium cluster (5–10 BEs) | Both |
| Above | >30TB or >500 QPS | — | **Case-by-case discussion (handling TBD)** | — |

> ⚠️ Threshold numbers are drafts. It's already recognized that "tying data volume and QPS to the same tier" is unreasonable (e.g., a team with small data but heavy queries can't be classified properly); an RFC is planned to discuss decoupling storage from compute. The hardware pool is not fixed in size (it fluctuates with procurement), and capacity planning is described using a single-tenant resource model.

## 4. User Journey

```
Fill out slide template → Weekly Wednesday application review meeting → Review (routed by mode, see below)
  → Staging trial environment validation (small spec, validates design not performance) → Platform provisions production environment → Go-live
```

- **Self-managed review**: Requires only management cost approval; once approved, a cluster is opened along with monitoring/alerting/logging.
- **Managed review**: Requires declaring the workspace's overall expected data size; the review determines shared vs. dedicated cluster.

- **The application process is fully manual** (slides + meeting + tracking sheet); no automated system is being built. Once application volume grows, a self-service portal will be built based on an already-defined state machine.
- **Staging is for trial purposes only**: a fixed small spec (the hardware isn't sufficient to match production spec), used to validate the reasonableness of schema/query design; performance figures are for reference only and **do not constitute a platform performance commitment**.

## 5. Data Management in Managed Mode

- **Three-zone layering** (analogous to Databricks Bronze/Silver/Gold): `tmp` (raw CDC landing, platform-managed) → `raw`/`curated` (tables created by the platform system after governance approval; the approver depends on the governance model still to be decided).
- **Streaming pipeline**: Users send messages into the platform's Kafka in a fixed format, supporting three formats (generic CDC / DB2 JSON CDC / DB2 XML CDC); data with format errors goes into a corrupted-data area (MinIO) with alerting, without interrupting ingestion.
- **Batch pipeline**: Users upload files to a platform-managed landing path in storage, triggered on a schedule or manually, sharing the same landing and governance path as streaming; the source interface reserves room for a future lakehouse → Doris channel extension (many assumptions in the spec draft remain to be confirmed).
- **High availability** (applies to both modes): active-standby dual-cluster + cross-cluster replication kept **best-effort in sync** + platform intervention during failover; RPO/RTO, default vs. opt-in, and how **standby resources (~2x the hardware)** are calculated remain to be quantified — this has implications for capacity planning and cost that management needs to be aware of.

## 6. Key Decisions Already Finalized (Excerpt)

| Decision | Content |
|------|------|
| Tenant unit | Workspace (granularity self-defined by the team), **mandatory cost-center binding** |
| Isolation requirements | Full isolation across data, metadata, and compute layers; **cross-workspace sharing is not supported** (different from lakehouse habits, requires upfront communication with users) |
| Mode conversion | Not convertible; switching modes requires opening a new workspace and migrating data |
| Metering | Storage: raw size for approval purposes, post-compression usage after go-live; QPS: 5-minute sliding window average |
| Naming convention | Managed always uses `{ws}__{custom}__{zone}`; self-managed naming is free-form |
| Depth of external-facing documentation | Managed must clearly state usage rules + limitations + accountability; self-managed focuses on accountability |

## 7. Items Requiring Management/PM Decision or Attention

### Pending Decisions (Will Affect Product Direction)
| # | Item | Impact | Suggested Timing |
|---|------|------|----------|
| 1 | **Billing model**: chargeback (actual charges) vs. showback (reporting only) vs. free internal service (**currently no payment mechanism exists at all**) | Affects the meaning of cost-center binding, handling of overages, and attribution of assisted-operation costs | Before NFRs are finalized (soon) |
| 2 | **Tier thresholds and storage/compute decoupling** (RFC) | Affects pricing and hardware procurement planning | RFC discussion |
| 3 | How to accommodate **large-scale needs beyond Tier 3** | Whether to expand the pool for this or explicitly decline | Before the first large account appears |
| 4 | Handling of **exceeding the tier but unwilling to upgrade**: keep throttling vs. force upgrade | Tied to the billing model | Together with #1 |
| 5 | Whether to build **row/column filtering** (fine-grained permissions) on the platform side | The requirement is now concrete: self-managed users are requesting row filtering / column masking on **specific tables** — if accepted, this would be the only exception to the "self-managed users handle their own permissions" principle, and responsibility for policy drift after users alter tables needs to be defined first | RFC discussion |
| 6 | **Scope of platform responsibility for PII/sensitive data** | Affects the review process and compliance costs | Governance review phase |
| 7 | **DDL/schema governance model for Managed** | Option A (owner approves, platform doesn't review): low review cost, quality responsibility on the user; Option B (platform reviews, strict review for raw): quality is controllable, high platform review cost. Affects platform staffing and scope of responsibility | **PM/management decision**, both options are laid out side by side in the PRD and specs |
| 8 | **Timeline and first target department** | Both currently undetermined | The sooner the better |
| 9 | **Self-managed DDL governance model and data management R&R** (RFC-003 already drafted) | Premise already set: DDL is always platform-assisted (strict at first, relaxed later if dbt needs arise). Pending: governance model Option 1/2/3 (team recommends Option 3: declaration-by-form) + backup Option A/B (tentatively A: not provided, explicit disclaimer + audit log as evidence) | **Manager decision**, RFC-003 is the decision document |

### Risks to Watch
- **Scaling bottleneck of the manual application process**: The weekly Wednesday meeting has limited capacity; as application volume grows, a queue will form. An automation path has been reserved, but a trigger point for when to escalate needs to be defined.
- **Gap between staging and production environments**: Staging cannot validate real performance — the truth only shows after go-live. External-facing documentation will explicitly disclaim this, but expectation management for the first cohort of users needs to be handled well.
- **Hardware pool fluctuation**: Tier 2/3 provisioning may hit a "waiting for resources" state depending on procurement progress, affecting commitments on provisioning lead time.
- **Gap in lakehouse usage habits**: Users are accustomed to cross-workspace access, which this platform explicitly prohibits — insufficient communication about this will become an early source of user complaints.

## 8. Documentation Progress and Next Steps

| Item | Status |
|------|------|
| PRD outline + terminology + scope | ✅ Complete |
| Feature specs (10 items) | ✅ 10/10 drafts complete (batch pipeline and account permissions include unconfirmed assumptions, pending team confirmation to close out) |
| Subsequent phases | Boundary review (QA perspective) → NFR/SLA quantification → governance compliance → stakeholder sign-off → produce architecture documents and external-facing documents |
