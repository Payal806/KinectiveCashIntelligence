---
id: DATA-KIN-CI-009
title: Cash Intelligence architecture — 14. Physical design, volumes and retention
artifact_type: data-architecture
status: approved
content_state: approved-design
owner_role: Cash Intelligence architecture owner
applies_to:
  - PRJ-KIN-BRAU-CI
source_of_truth: golden-repo
source_refs:
  - "SRC-CONF-CIR-001"
  - "SRC-CI-RES-001"
last_updated: 2026-11-22
review_due: TBC
---

# 14. Physical design, volumes and retention

> **Approved architecture: the R1 design baseline (DEC-006).** Approved by the Cash Intelligence architecture owner on 2026-11-05, after the 2026-11-03 design review. Generated specs follow it; a spec that needs something different is a change here first (GOV-CI-004 in `projects/branch-automation-cash-intelligence/05-delivery/delivery-model-and-gates.md`). Open questions (§16) stay open until answered. Section numbers (§n), open questions (Qn) and decisions (A–I) use the architecture's own numbering; `products/cash-intelligence/architecture/architecture-overview.md` maps each section to its file, and the open questions are in `products/cash-intelligence/architecture/open-questions.md`. Citations: `BAP-nn` = `SRC-JIRA-BAP-001`, `TDD:n` = line n of `SRC-XR-TDD-001`, `research/nn` = `SRC-CI-RES-001` (see the source index).
>
> **Legal review required.** This file covers data retention and purge. These touch compliance: Kinective's Legal team must review them before they are relied upon or selected as Astra context.

**Keys and indexes.** `INT` surrogates for tenant, location, device and rule; `BIGINT` for events and runs. Every
primary key — InnoDB's clustered index — leads with `tenant_id`, then the main access path: daily facts
`(tenant_id, device_id, business_date)`, so a device drill-down is one range read. Fleet-wide date ranges read the
location rollups (§7.3a); a secondary index `(tenant_id, business_date, device_id)` serves device-level date slices.
Large analytical tables may use `ROW_FORMAT=COMPRESSED` where the chosen host supports it (verify on Aurora and HeatWave). A compressed table cannot take `ALGORITHM=INSTANT` column additions, so every expand migration on it rebuilds the table in every tenant; decide per table after the load test, and keep the widest-changing tables uncompressed.

**Partitioning.** `RANGE COLUMNS` by month on `business_date` (device and location facts, snapshots, error events), on
`run_month` (forecast vintages) and on `run_week` (weekly forecast archive). MySQL requires the partition column in every unique key, which is why those tables use
natural keys, and partitioned tables cannot have foreign keys, so referential integrity for facts is enforced by the
ingestion step and checked by reconciliation (§5.3). Retention drops whole partitions (`ALTER TABLE … DROP PARTITION`),
never row deletes, under the maintenance account (§13); `audit_log` partitions are never dropped by retention jobs.

**Indicative volume** (largest single FI, ~5,000 devices): ~1.8M `device_daily_activity` rows/year; ~10M
`_denom` and ~10.5M snapshot lines (measured, below); ~0.3M `location_daily_activity`; ~2M forecast-denom rows per
40-day run. CO runs Mon–Fri, so about 22 runs a month, or ~130M rows in three months of vintages for the largest
tenant; if that proves heavy, keep only the current run plus the weekly totals archive. Current-inventory line history grows with every
inventory change (~500M rows/year at Tier B for 5,000 devices), hence its short retention.

**Measured** (`research/80`): one FI of 5,000 devices (400 branches and sites) with 25 months of daily facts, plus a
70-device second tenant, on 4 vCPU, 15 GB RAM and a 6 GB buffer pool.

| Measure | Result |
|---|---|
| Size | 13.5 GB data + index for both tenants; largest: `device_daily_activity` 3.8 M rows, `_denom` 20.0 M, snapshot lines 21.9 M, staged journals 6.6 M (45 days) |
| Yearly growth (5,000 devices) | 1.83 M `device_daily_activity`, 9.6 M `_denom` (~5.3 active denominations per device-day), 10.5 M snapshot lines, 0.29 M location rows |
| Pages, warm | p95 < 2 s for Admin, a 20-branch User and a Service Provider with the query rules of §15, except the Admin dashboard at 2.1 s with its widgets run one after another |
| Pages, re-measured (2026-11-22) | On a shared lab host (4 vCPU, 15 GB, another product's stack running; 180 days of facts): scoped user and service provider ≤ 0.5 s, Admin ≤ 1.9 s except the dashboard 4.2 s and ATM replenish history 3.0 s. The pre-refactor and the Kotlin queries time the same there and a 6 GB buffer pool changed nothing, so the port and the buffer pool are ruled out; the host load is the likely cause, not isolated (no measurement on an idle host yet). Each page's merge request measures its time on a host with no other workload (DEC-030) |
| Jobs | 100,000-row micro-batch 24 s; nightly EOD capture for 5,000 devices 20.5 s; month partition drop 0.2 s; history triggers +24 % per single-device update |
| Access | Fail-closed scoping held in every view (no scope, other user, other tenant: 0 rows), measured with scope materialized per request; to be repeated with the user-keyed scope views (§11) |

**Storage sizing.** Cold-cache pages took 1–16 s at enterprise scope on a ~200 IOPS virtual disk; the disk, not the
queries, was the constraint for cold reads and bulk loads (`research/80` §9 P7). Production needs SSD-class storage
(target ≥ 3,000 IOPS sustained) and a buffer pool sized for the hot working set — the last ~90 days of facts and
snapshots, current state and the current day's staging — warmed at restart from a buffer-pool dump (§16 Q29).

**Retention — open question (requires Legal review).**

| Data | Proposed hot retention |
|---|---|
| Daily facts and snapshots | 25 months (UI needs 180 days; R2 seasonality ≥ 13 months) |
| Staging | 45 days (iQ); CO window + margin |
| Reconciliation | 13 months |
| Forecast vintages (`device_forecast`, `_denom`) | ~3 months of monthly `run_month` partitions; before a month is dropped, its first run per device per week is copied to `device_forecast_weekly` |
| Weekly forecast archive (totals) | 25 months (proposed; for R2 model evaluation) |
| Current denomination split (`device_forecast_denom_current`) | Until superseded by a newer run for the device |
| Current-inventory history (`_hist`) | Header 25 months; lines 60 days (EOD snapshots keep the long view) |
| Service records and audit | Per Legal |
