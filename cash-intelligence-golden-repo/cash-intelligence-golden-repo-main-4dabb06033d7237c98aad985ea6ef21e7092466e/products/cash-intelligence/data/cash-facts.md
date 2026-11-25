---
id: DATA-KIN-CI-004
title: Cash Intelligence architecture — 7. Cash facts
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

# 7. Cash facts

> **Approved architecture: the R1 design baseline (DEC-006).** Approved by the Cash Intelligence architecture owner on 2026-11-05, after the 2026-11-03 design review. Generated specs follow it; a spec that needs something different is a change here first (GOV-CI-004 in `projects/branch-automation-cash-intelligence/05-delivery/delivery-model-and-gates.md`). Open questions (§16) stay open until answered. Section numbers (§n), open questions (Qn) and decisions (A–I) use the architecture's own numbering; `products/cash-intelligence/architecture/architecture-overview.md` maps each section to its file, and the open questions are in `products/cash-intelligence/architecture/open-questions.md`. Citations: `BAP-nn` = `SRC-JIRA-BAP-001`, `TDD:n` = line n of `SRC-XR-TDD-001`, `research/nn` = `SRC-CI-RES-001` (see the source index).

## 7.1 Current inventory (BAP-16, BAP-58)

```sql
cash.device_inventory_current      (tenant_id, device_id PK,
                                    as_of DATETIME(3), source_system, source_version VARCHAR(64),
                                    basis VARCHAR(12),                -- live | last_eod
                                    total_amount DECIMAL(19,4),
                                    valid_from, valid_to)             -- versioned (history table)
cash.device_inventory_current_line (tenant_id, device_id, bin_role, denomination_id,
                                    note_count INT, face_value DECIMAL(9,2),
                                    amount GENERATED ALWAYS AS (note_count * face_value) STORED,
                                    capacity_count INT NULL,
                                    amount_residual DECIMAL(19,4) NULL, -- CO Amount tenants only (P4)
                                    derivation VARCHAR(16),           -- source | nearest_order | current_profile
                                    PRIMARY KEY (tenant_id, device_id, bin_role, denomination_id),
                                    valid_from, valid_to)
```

- **TCR**: which Link table is "current" follows iQ's code, in this order: (1) if only one table has inventory XML for
  the device, it wins; (2) model aliases `LgLta350`, `HyosungMS400A` and `HyosungMS500` use `cfm_live_inventory`; (3) if exactly one table is
  non-empty, it wins; (4) `cfm_live_inventory` when the `cfm_lookup_value` entry `NormAsPrimaryWS` is set (synced to
  iQ; the device column is not) or the device is a dispenser, else
  `cfm_live_device_inventory`. A change to either table re-evaluates the rule. `cfm_live_device_inventory` refreshes
  only on device checks and can lag.
- **ATM**: CO has no intraday inventory. ATM current = the last approved EOD (`basis = 'last_eod'`), and the UI shows
  its date. Counts follow the tenant's `Cash_Mode` (§5.2).
- **ATM acceptor notes** (BAP-69) have **no source**: CO records no deposit-bin removal (`HnD_Amount` is a positional
  "recover" amount on the dispense hoppers). No deposit line is written for ATMs, and ATM days-until-full is not
  computed unless a real source is found (§16 Q16).
- **Dispensable inventory** (Cash Order Management "Current Inventory" and days-until-empty) = lines with
  `bin_role IN ('dispense', 'recycle')`; deposit and reject bins are excluded.

## 7.2 Snapshots — EOD and SOD (BAP-17, BAP-29..36, BAP-83)

```sql
cash.inventory_snapshot      (tenant_id, device_id, business_date DATE,
                              snapshot_kind VARCHAR(8),               -- eod | sod   (one row per kind per day)
                              captured_at DATETIME(3),
                              source_system, source_instance_id, source_pk VARCHAR(64),
                              derivation VARCHAR(16),                 -- source | source_backfill | prior_eod
                                                                      -- | journal_replay | current_profile
                              total_amount DECIMAL(19,4),
                              PRIMARY KEY (tenant_id, device_id, business_date, snapshot_kind))
cash.inventory_snapshot_line (tenant_id, device_id, business_date, snapshot_kind,
                              bin_role, denomination_id, note_count INT,
                              face_value DECIMAL(9,2), amount GENERATED ALWAYS AS (note_count * face_value) STORED,
                              amount_residual DECIMAL(19,4) NULL,     -- CO Amount tenants only (P4)
                              PRIMARY KEY (tenant_id, device_id, business_date, snapshot_kind, bin_role, denomination_id))
```

Both tables use the natural key (no surrogate id) so they can be partitioned by `business_date` (§15). Intraday states
live in the current-inventory history tables, not here.

- **TCR EOD** for business date *d* = the **latest** `cfm_start_inventory` row whose branch-local time is at or after
  the cutoff on *d* and before the cutoff on *d+1* (setting `eod_cutoff_local_time`, default **09:00, exclusive**: a
  row stamped before 09:00 local belongs to the previous business day). This reproduces iQ's rule (local hour ≤ 8 →
  previous day). iQ falls back to US/Mountain for a branch with no zone; CI quarantines instead (`no_branch_tz`).
  `cfm_start_inventory` keeps many rows per device (auto balance, manual balance, backfill), so the EOD is chosen,
  not overwritten.
- **Re-dated source rows.** Link's backfill re-dates the device's latest start-inventory row in place, once per missing
  day, so the genuine EOD of the old date disappears from the source. CI **never deletes or re-derives a captured
  snapshot because its source row was re-dated**: it keeps the old snapshot, adds the new date, and marks the new
  snapshot `source_backfill`. Link writes no marker; the observable signal is an in-place change of `rec_timestamp` on a
  start-inventory `id` CI has already staged (iQ resends the same id with the new time). The snapshot captured under the old time is kept, and the date the row moves to is `source_backfill`. Link's
  backfill also replaces the row's XML with the last journal's cassettes, so a backfilled EOD can carry a later day's
  state (§5.3 explains the resulting breaks). Rows from historical repair scripts are treated the same way. Backfill rows may collapse all denominations into one cassette; counts per denomination
  survive.
- **ATM EOD** = approved `DailyCashBalanceDetails.Hn_PrvDayEnd` (amounts or counts per `Cash_Mode`, §5.2), unpivoted
  through the bin profile valid on that date, dated by CO's offset rule. Whether `PrvDayEnd` on a row dated *LoadDate* is the EOD of *LoadDate* or of
  *LoadDate − 1* is unconfirmed (§16 Q17); the mapping is one setting so it can be corrected without reprocessing logic.
- **SOD** for *d* = EOD of *d−1* (`prior_eod`) unless a source supplies one.
- **Backfill at onboarding**: retained `cfm_start_inventory` rows, read from iQ's Mongo copy
  `s4_sync_cfm_start_inventory` (same rows, same selection rule); CO balance history (with the hopper caveat above).
  About 1% of device-days have several start-inventory rows in the iQ test data, so the selection rule always applies.

## 7.3 Daily activity (BAP-19, BAP-85, BAP-79, BAP-114)

```sql
cash.device_daily_activity       (tenant_id, device_id, business_date,       -- PK; clustered
                                  dispensed_amount, deposited_amount,         -- customer category only
                                  loaded_amount, removed_amount,              -- load / remove categories
                                  correction_amount NULL,                     -- excluded from usage, kept for audit
                                  dispense_txn_count, deposit_txn_count, txn_count INT NULL,
                                  txn_count_source VARCHAR(12),               -- link_journal | ej | none
                                  txn_count_left, txn_count_right, txn_count_access INT NULL,   -- TCR only
                                  notes_moved BIGINT NULL,                    -- Σ cfm_journal.notesio (TCR)
                                  sod_amount, eod_amount DECIMAL(19,4) NULL,  -- copied from snapshots
                                  open_minutes INT NULL, downtime_minutes INT NULL,   -- NULL when branch hours unknown
                                  error_count_total, error_count_sev1, error_count_sev2, error_count_sev3 INT,
                                  is_complete BOOLEAN,                        -- CI rule, below
                                  derivation VARCHAR(16) NULL,                -- NULL | nearest_order | current_profile (CO backfill)
                                  flow_missing_reason VARCHAR(12) NULL,       -- ATM: pending | rejected | quarantined | no_row
                                  deposited_unvalued_count INT NULL,          -- CO Bill_Count: notes on hoppers with no denomination
                                  recomputed_at DATETIME(3))
cash.device_daily_activity_denom (tenant_id, device_id, business_date, denomination_id,   -- PK
                                  dispensed_count, deposited_count, loaded_count, removed_count INT NULL,
                                  face_value DECIMAL(9,2) NULL,
                                  deposited_unsorted_amount DECIMAL(19,4) NULL,   -- 'Unsorted' row only
                                  amount_residual DECIMAL(19,4) NULL,         -- CO Amount tenants: Σ |residual| over measures
                                  dispensed_amount GENERATED ALWAYS AS (dispensed_count * face_value) STORED,
                                  deposited_amount GENERATED ALWAYS AS
                                    (COALESCE(deposited_count * face_value, deposited_unsorted_amount)) STORED)
```

- **TCR** amounts come from the cassette XML per journal row, summed by `activity_map` category (§5.1). DpsitAccept
  is excluded (it duplicates the deposit). UndoDpsit nets against deposits, not a dispense (iQ files it as a
  dispense). Load/unload is replenishment, not usage.
- **TCR transaction count** = distinct `transaction_id` over **customer** rows only. **Side usage**:
  `remoteTransactionId` present ⇒ access, else `side` 0/1 ⇒ left/right (assumption, §16 Q8).
- **ATM amounts** come from approved `DailyCashPositionsDetails` (dispensed and deposited per hopper per day; amounts
  or counts per `Cash_Mode`, §5.2). Deposits on a hopper with no denomination in the bin profile (a deposit-only
  acceptor) cannot be attributed to a denomination: they are stored on the **Unsorted** row
  (`deposited_unsorted_amount`, counts NULL), exported as an "Unsorted" denomination row, and the export metadata says
  so. In a `Bill_Count` tenant such a deposit is a note count with no face value, so it has no amount: it is kept in
  `device_daily_activity.deposited_unvalued_count`, deposits are shown as "$x + n notes of unknown value", and
  nothing is dropped (§16 Q31).
- **ATM transaction counts** come from the EJ parser's `EJ_TRANSACTIONS` (live, `_HISTORY` and `_ARCHIVE`), staged
  one allowlisted row per transaction (§5.2) and counted per device-day through the instance's
  `ingest.ej_txn_type_map`; successful dispense and deposit transactions only, check deposits excluded, dated by
  `TransactionTimeLocal`. Card, PAN, account, check and EMV columns are never selected (§13). Where no EJ feed exists,
  `txn_count` is NULL and `txn_count_source = 'none'`; usage pages **exclude** such devices instead of ranking them as
  least used.
- `error_count_total` includes severity 0 (unknown), so error totals are not lost.
- **`is_complete`** is a CI rule, since no source signals "all rows landed": 1 when no reconciliation break is open for
  the device-day, **no quarantined row of the device-day is unresolved** (§5.3), **and** every feed serving
  the device has read its source up to the end of the business day, taken in
  the branch's zone, plus that feed's expected lag: the lowest `feed_checkpoint.source_watermark_at` (UTC) over the
  feed's partitions, and over the tenant's parked row if any, is past that point. `last_version` (an offset) is not
  compared.
- **ATM days with no approved position.** When a day's only `DailyCashPositionsDetails` row is pending, rejected or
  quarantined, or there is none, the dispensed and deposited columns are **NULL**, not 0, and `flow_missing_reason`
  says why (decision G). A pending row that is approved later fills the day. Such days are `is_complete = 0`.
- **Incomplete days.** Device-days with `is_complete = 0` are excluded from overstock statistics and branch
  positioning and counted separately ("n days incomplete"). Trend charts mark days where `reporting_device_count < device_count` (§7.3a).

## 7.3a Location rollups (fleet-wide pages)

```sql
cash.location_daily_activity (tenant_id, location_id, business_date,
                              device_family VARCHAR(8),               -- atm | tcr | coin
                              linked BOOLEAN NOT NULL DEFAULT 1,      -- coin rows: 0 = unlinked coin
                              device_count, reporting_device_count, incomplete_device_count INT,
                              dispensed_amount, deposited_amount, loaded_amount, removed_amount,
                              sod_amount, eod_amount DECIMAL(19,4),
                              txn_count, dispense_txn_count, deposit_txn_count INT NULL, txn_reporting_device_count INT,
                              downtime_minutes, open_minutes, error_count_total INT,
                              recomputed_at DATETIME(3),
                              PRIMARY KEY (tenant_id, location_id, business_date, device_family, linked))
cash.location_daily_denom    (tenant_id, location_id, business_date, device_family, linked, denomination_id,   -- PK
                              eod_count, dispensed_count, deposited_count BIGINT)
```

Branch-level (and site-level) sums of the device facts. Each device-day is attributed to the device's location **as
of that `business_date`** (versioned `ref.device.location_id`); a device move or a coin-link change re-queues the
rollup recompute for the affected history. Rollup rows are rewritten once per feed run for the touched (location, day)
pairs, not once per device-day, to avoid write contention on branch rows. A rollup writer first locks the touched
`ref.location` rows in ascending `location_id` order, so two writers of one branch-day (a feed run and a queued rollup)
serialise instead of losing an update. Region and enterprise totals sum branch rows
(≤ a few hundred per day). Coin is kept as its **own `device_family = 'coin'` rows** (`linked = 0` for unlinked coin):
"(+ coins)" pages add `coin` to `tcr`; coin-excluding pages never add it.

**Which pages may read the rollups:** Dashboard EOD trend, Historic EOD chart at region and branch level (`tcr` +
`coin`, `atm`), ATM Usage map and line chart (`atm`, including the date dialog's deposit/dispense split). **Never:**
Overstock Analysis and Branch Cash Positioning (coin and scheduled-device exclusions need device facts), device-level
tables and exports, and any Service Provider session (rollups mix families and models; §11). Every rollup row is
reconciled against the sum of its device rows nightly (§5.3).

**Device roll-up view.** `cash.v_device_rollup` is the single place where coin merges into a TCR, for current
inventory, snapshots and daily facts:

- one `row_kind = 'device'` row per listed device (TCR, TCD, ATM) and date: the device's own facts plus those of every
  coin device linked to it **on that date** (`ref.device_link` with `effective_from <= day < effective_to`, open-ended
  when `effective_to` is NULL; for current inventory, the link active now);
- one `row_kind = 'unlinked_coin'` row per branch and date: the sum of coin devices with no link on that date, carrying
  the data-quality flag.

The nightly reconciliation checks parity: per branch and day, Σ `v_device_rollup` (both kinds) = Σ location rollup
(`tcr` + `coin` + `atm`).

## 7.4 Cash orders and replenishments (BAP-18, BAP-20)

```sql
cash.cash_order        (cash_order_id BIGINT PK, tenant_id, device_id,
                        source_system, source_instance_id,
                        source_order_ref VARCHAR(64), source_detail_ref VARCHAR(64),  -- CashOrderId, CashOrderDetailsId
                        planned_load_date DATE,                       -- DATE(CashRefillDate + INTERVAL TermOffset MINUTE)
                        created_at_source DATETIME(3) NULL,
                        position_type VARCHAR(8),                     -- refill | withdraw
                        status VARCHAR(24),                           -- order-level Status/SubStatus, refined per device by TermStatus:
                                                                      -- unapproved | approved | dispatched | pending_replenishment
                                                                      -- | in_transit | partially_replenished | replenished
                                                                      -- | vault_report_uploaded | rejected | revised
                        source_status VARCHAR(48),                    -- order-level Status/SubStatus
                        source_run_type VARCHAR(12) NULL,             -- scheduled | unscheduled (CO RunType)
                        is_emergency BOOLEAN NULL,                    -- captured on first sight; NULL if the request was gone
                        ordered_amount DECIMAL(19,4),
                        cost_amount DECIMAL(19,4) NULL,               -- CO contract tariff ExecutionRunCost (D)
                        parent_cash_order_id BIGINT NULL,             -- CO revisions (0 → NULL)
                        is_full_replenishment BOOLEAN NULL,               -- CO ≥ 1.6 only
                        UNIQUE (source_instance_id, source_detail_ref))
cash.cash_order_line   (cash_order_id, denomination_id, ordered_count INT, forecast_amount DECIMAL(19,4) NULL,
                        amount_residual DECIMAL(19,4) NULL,           -- CO Amount tenants only (P4)
                        PRIMARY KEY (cash_order_id, denomination_id))
cash.replenishment     (replenishment_id BIGINT PK, tenant_id, device_id, business_date,
                        occurred_at DATETIME(3) NULL,              -- TCR only; CO loads are daily
                        source_system, source_instance_id, source_pk VARCHAR(128),
                        replenishment_type VARCHAR(8) NULL,           -- add_on | swap | empty | unload
                        cash_order_id BIGINT NULL,
                        match_method VARCHAR(12) NULL,                -- source_fk | none
                        loaded_amount, removed_amount DECIMAL(19,4),
                        UNIQUE (source_instance_id, source_pk))
cash.replenishment_line(replenishment_id, denomination_id, loaded_count, removed_count,
                        amount_residual DECIMAL(19,4) NULL,           -- CO Amount tenants: Σ |residual| over measures
                        PRIMARY KEY (replenishment_id, denomination_id))
```

Views (all replenish-history queries go through them):

- `cash.v_cash_order_effective` — refill orders only, excluding `rejected` and every order that has been superseded
  by a revision (keeps the latest in each `parent_cash_order_id` chain).
- `cash.v_cash_order_classified` — adds `ci_schedule_class`, **computed at query time** from the device's **current**
  exclusive schedule rule, as the requirement states (BAP-44): scheduled if the schedule is daily or the order's
  weekday is one of the schedule's days; else unscheduled; no schedule ⇒ unscheduled. Editing a schedule
  reclassifies history on the next page load. (An anchor-date rule is an alternative for Product to consider, §16 Q6.) This
  **deliberately differs** from `cfg.schedule_calendar` (§10.2), which also applies frequency and anchor: a biweekly
  off-week order counts as scheduled here although the calendar has no occurrence that week.
- The **planned series** on the replenished-vs-ordered chart (BAP-40) is the configured schedule order value from
  `cfg.rule_replenish_schedule_line` on the dates in `cfg.schedule_calendar`, per the requirement — not CO order
  amounts.
- **Planned load date**: CO stores `CashRefillDate` in UTC, shifted by the first terminal's `TermOffset`, so
  `planned_load_date = DATE(CashRefillDate + INTERVAL TermOffset MINUTE)`.
- **Unscheduled cost** (BAP-39) uses the setting `unscheduled_order_cost`. `cost_amount` is derived from CO's contract
  tariff (`ContractRulesDetails.ExecutionRunCost` for the order's contract rule and run type, as CO's own cost report
  does) and kept for reconciliation; `ContractCashOrderMap.RunCost` is always 0 and is not used.
- **Loads**: CO manual loads link to an order by `CashOrderId` (`source_fk`); EJ auto-loads link by the stored
  `DailyCashLoadDetails_Auto.CashOrderId` (`source_fk`; 0 → NULL and `none`, which is common because dispatched orders
  lose the SubStatus CO filters on). TCR loads and empties come from Link load/remove activities, with no order behind
  them.
- ATM orders exist only for tenants that run CO; elsewhere these tables are empty.
