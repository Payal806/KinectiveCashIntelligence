---
id: DATA-KIN-CI-006
title: Cash Intelligence architecture — 9. Forecasting
artifact_type: data-architecture
status: approved
content_state: approved-design
owner_role: Cash Intelligence architecture owner
applies_to:
  - PRJ-KIN-BRAU-CI
source_of_truth: golden-repo
source_refs:
  - "SRC-CONF-CIR-001"
  - "SRC-JIRA-BAP-001"
last_updated: 2026-11-22
review_due: TBC
---

# 9. Forecasting

> **Approved architecture: the R1 design baseline (DEC-006).** Approved by the Cash Intelligence architecture owner on 2026-11-05, after the 2026-11-03 design review. Generated specs follow it; a spec that needs something different is a change here first (GOV-CI-004 in `projects/branch-automation-cash-intelligence/05-delivery/delivery-model-and-gates.md`). Open questions (§16) stay open until answered. Section numbers (§n), open questions (Qn) and decisions (A–I) use the architecture's own numbering; `products/cash-intelligence/architecture/architecture-overview.md` maps each section to its file, and the open questions are in `products/cash-intelligence/architecture/open-questions.md`. Citations: `BAP-nn` = `SRC-JIRA-BAP-001`, `TDD:n` = line n of `SRC-XR-TDD-001`, `research/nn` = `SRC-CI-RES-001` (see the source index).

## 9.1 Model registry and runs

```sql
fcst.model          (model_id INT PK, model_key VARCHAR(32),          -- cm_lightgbm | ci_baseline_dow | ci_ai_v1 (R2)
                     version VARCHAR(16), producer VARCHAR(10),       -- external | ci
                     device_family VARCHAR(8) NULL, grain VARCHAR(4), unit VARCHAR(6),
                     point_statistic VARCHAR(12),                     -- mean | quantile
                     point_quantile DECIMAL(4,3) NULL,                -- one model-wide quantile only; NULL for CO (α per
                                                                      -- cluster, recorded in fcst.run.params_json)
                     has_intervals BOOLEAN, description, registered_at, retired_at NULL,
                     UNIQUE (model_key, version))
fcst.model_selection(tenant_id, device_family VARCHAR(8), model_id, effective_from DATE,
                     PRIMARY KEY (tenant_id, device_family, effective_from))
fcst.run            (run_id BIGINT PK, tenant_id, model_id,
                     source_partition VARCHAR(16) NULL,               -- CO TermType (ATM | CDM | RECYCLER)
                     data_cutoff_date DATE NULL,                      -- CI runs only; NULL for CO (not stored by CO)
                     generated_at DATETIME(3), horizon_days SMALLINT,
                     status VARCHAR(10),                              -- running | succeeded | partial | failed | missed
                     source_ref VARCHAR(128) NULL,                    -- CO UploadedOn
                     params_json JSON NULL, metrics_json JSON NULL,   -- CO: per-cluster α in params_json
                     run_month DATE GENERATED ALWAYS AS (first day of generated_at's month) STORED)   -- joins supply the partition key
fcst.device_forecast       (tenant_id, run_month DATE,                     -- first day of the run's month (partition key)
                            run_id, device_id, forecast_date,             -- PK (tenant_id, run_month, run_id, device_id, forecast_date)
                            dispense_amount, deposit_amount DECIMAL(19,4),
                            dispense_low, dispense_high, deposit_low, deposit_high DECIMAL(19,4) NULL)  -- CI models only
fcst.device_forecast_denom (tenant_id, run_month DATE, run_id, device_id, forecast_date, denomination_id,
                            dispense_count, deposit_count INT, amount_residual DECIMAL(19,4) NULL,
                            PRIMARY KEY (tenant_id, run_month, run_id, device_id, forecast_date, denomination_id))
fcst.device_forecast_weekly (tenant_id, run_week DATE,                    -- Monday of the run's week (partitioned by month)
                            device_id, forecast_date, run_id,
                            dispense_amount, deposit_amount DECIMAL(19,4),  -- totals only
                            PRIMARY KEY (tenant_id, run_week, device_id, forecast_date))
fcst.device_forecast_denom_current (tenant_id, device_id, forecast_date, denomination_id,
                            run_id, dispense_count, deposit_count INT,
                            PRIMARY KEY (tenant_id, device_id, forecast_date, denomination_id))
```

- **ATM → `cm_lightgbm`**: CO's per-terminal LightGBM quantile model (40-day horizon, nightly Mon–Fri). Each CO write
  (tenant × term type × `UploadedOn`) becomes one `fcst.run`; a write interrupted mid-batch is `partial` and not
  selected. CO keeps **no vintages**: `PreForecast` is overwritten in place, so CI stages each write before the next
  CO run (§5.2); a write CI did not stage in time is lost and recorded as a `missed` run with no rows. CO's values are
  **quantile predictions** (α 0.80–0.85, set per cluster and recorded in `params_json`), deliberately high, not means;
  `fcst.model` records that so no page compares them with mean forecasts unawares. In `Bill_Count` tenants the values
  are counts (§5.2). CO produces no intervals and no data cutoff; those columns stay NULL. Hourly-forecast tenants are
  aggregated to day.
- **TCR → `ci_baseline_dow`**: computed by CI from `device_daily_activity_denom` — windowed day-of-week mean and
  standard deviation per denomination, intervals = mean ± k·sd, **60-day horizon**. The requirement lets the order dialog take any future range; a range past the forecast horizon (ATM: CO's 40 days; TCR: 60) returns no recommendation for the uncovered days, with the reason shown (§16 Q32).
  A 30-day average is the fallback for devices with too little history.
- **R2** registers `ci_ai_v1` and switches `model_selection`; no schema change.
- **Current forecast**: for a device on a business day, the latest succeeded run of the model selected for the
  device's family on that day (and, for ATMs, of the device type's partition). One definition in ci-service with the
  day bound (the projection job's business date, the page's day), used by the projection and the page queries. It is
  not a view: a view could read the day only from the database clock (DEC-028).
- **Retention by partition** (§14): `device_forecast` and `_denom` keep about three months of vintages in monthly
  `run_month` partitions. Before a month is dropped, the first run per device per week is copied (totals only) into
  `device_forecast_weekly`.
- **Current denomination split**: `device_forecast_denom_current` holds the denomination rows of each device's current
  run (the current forecast's run: the selected model's) and is overwritten when a newer run of that model succeeds for
  the device (a small, unpartitioned table). R1 has one model per family, so the newest run is the selected model's; a
  model-selection change (R2) must rebuild the split from the newly selected model's run. The split therefore
  never disappears for a stale run. A current run older than `forecast_stale_days` raises the stale-forecast alarm.

## 9.2 What reads forecasts

| Page / story | Use |
|---|---|
| Cash Order Management — ATM cash level (BAP-67) | days until empty before next scheduled order |
| Branch cash level (BAP-68) | days until empty for unscheduled ATMs and TCRs |
| Bill acceptor status (BAP-69) | days until full = (capacity − acceptor notes) ÷ forecast deposit notes/day. **Not computed for ATMs in R1**: acceptor fill has no source (§7.1, §16 Q16) |
| Order recommendation (BAP-70/71) | predicted usage over the period; denomination split from `device_forecast_denom_current` per §9.3 |

Branch positioning rebalancing (BAP-120) uses **historical average daily usage** over the selected interval, as the
requirement states, not the forecast.

## 9.3 Order recommendation

Computed on demand, not persisted (§16 Q12). Per device: max(0, predicted usage over the period − current dispensable
inventory) (§7.1), before vault allocation and denomination rounding, so no order or note count is negative. For ATMs the predicted usage sums CO's daily high-quantile forecasts, a conservative bias (§16 Q23).
Branch vault funds available = max(0, entered vault total − effective vault-minimum rule); deducted from the branch total, with
device orders scaled down proportionally; a device fully covered shows "Covered by vault". **Denomination split**: the
device's forecast denomination mix over the period (`fcst.device_forecast_denom_current`), else its schedule mix
(`rule_replenish_schedule_line`), else the tenant's default mix; rounded to whole notes. Each result row exposes
**`split_basis`** (`forecast` | `schedule` | `default`) and the current run's date.

## 9.4 Device projection (materialized)

```sql
cash.device_projection (tenant_id, device_id PK,
                        computed_at DATETIME(3), inventory_as_of DATETIME(3), forecast_run_id BIGINT,
                        config_version INT,                           -- configuration version used (§10.1)
                        projected_empty_date DATE NULL, days_until_empty SMALLINT NULL,
                        projected_full_date DATE NULL,  days_until_full SMALLINT NULL,   -- NULL for ATMs (§7.1)
                        next_scheduled_order_date DATE NULL,          -- next date in cfg.schedule_calendar
                        empty_before_next_order BOOLEAN)
```

Days until empty use dispensable inventory (§7.1). Refreshed when a device's inventory, current forecast, **schedule
rule or calendar, or a projection-relevant setting** changes (a schedule write or a configuration activation enqueues the affected devices), plus
a nightly sweep.
