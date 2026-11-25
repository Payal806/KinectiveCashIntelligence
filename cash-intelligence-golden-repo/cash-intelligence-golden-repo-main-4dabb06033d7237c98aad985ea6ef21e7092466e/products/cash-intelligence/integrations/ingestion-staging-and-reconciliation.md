---
id: INT-KIN-CI-002
title: Cash Intelligence architecture — 5. Ingestion, staging and reconciliation
artifact_type: integration-design
status: approved
content_state: approved-design
owner_role: Cash Intelligence architecture owner
applies_to:
  - PRJ-KIN-BRAU-CI
source_of_truth: golden-repo
source_refs:
  - "SRC-CONF-CIR-001"
  - "SRC-CI-RES-001"
last_updated: 2026-11-05
review_due: TBC
---

# 5. Ingestion, staging and reconciliation

> **Approved architecture: the R1 design baseline (DEC-006).** Approved by the Cash Intelligence architecture owner on 2026-11-05, after the 2026-11-03 design review. Generated specs follow it; a spec that needs something different is a change here first (GOV-CI-004 in `projects/branch-automation-cash-intelligence/05-delivery/delivery-model-and-gates.md`). Open questions (§16) stay open until answered. Section numbers (§n), open questions (Qn) and decisions (A–I) use the architecture's own numbering; `products/cash-intelligence/architecture/architecture-overview.md` maps each section to its file, and the open questions are in `products/cash-intelligence/architecture/open-questions.md`. Citations: `BAP-nn` = `SRC-JIRA-BAP-001`, `TDD:n` = line n of `SRC-XR-TDD-001`, `research/nn` = `SRC-CI-RES-001` (see the source index).

## 5.1 Feeds

```sql
ingest.source_instance (source_instance_id INT PK,
                        source_system VARCHAR(16),                    -- iq_sync | cash_optimizer | ej_parser | csv | fred
                        instance_ref VARCHAR(128),                    -- server/DB identifier as registered at onboarding
                        tenant_hint INT NULL,                         -- when one instance serves one tenant
                        source_version VARCHAR(32) NULL,              -- current detected version; each run's is on ingest.run (§5.4)
                        UNIQUE (source_system, instance_ref))
ingest.feed            (feed_id INT PK, source_instance_id,
                        extract_mode VARCHAR(16),                     -- mongo_poll | kafka | window_reread | full_snapshot | file | api
                        freshness_tier CHAR(1), schedule_expr NULL,
                        window_days SMALLINT NULL,                    -- window_reread only
                        connector_config_json,                        -- non-secret only; secrets in vault
                        field_allowlist_json NOT NULL,                -- P10: a column is read only if listed
                        active BOOLEAN, created_at, updated_at)
ingest.feed_checkpoint (feed_id, source_table VARCHAR(128),
                        source_partition INT NOT NULL DEFAULT 0,      -- Kafka partition (0 for Mongo and CO); a partition's
                                                                      -- offset advances only after every tenant write of its
                                                                      -- batch has committed (§5.2 "At least once"), and the
                                                                      -- consumer seeks from it on assignment
                        tenant_scope INT NOT NULL DEFAULT 0,          -- 0 = the feed; a tenant id = that tenant is parked (§14.1)
                        last_version VARCHAR(64),                     -- Kafka offset / Mongo or identity watermark
                        source_watermark_at DATETIME(3) NULL,         -- UTC source write time of the last row read in order
                        last_success_at DATETIME(3),
                        PRIMARY KEY (feed_id, source_table, source_partition, tenant_scope))
ingest.feed_lease      (feed_id INT PK,                               -- not used by Kafka feeds (the consumer group assigns partitions)
                        owner VARCHAR(64) NULL,                       -- replica id
                        lease_until DATETIME(3) NULL,
                        epoch BIGINT NOT NULL DEFAULT 0)              -- incremented on every claim; fences a stale owner (§14.2)
ingest.run             (run_id BIGINT PK, feed_id, started_at, ended_at NULL,
                        source_version VARCHAR(32) NULL,              -- source version this run detected and read with (§5.4)
                        status,                                       -- running | succeeded | failed | partial
                        rows_read, rows_upserted, rows_quarantined, days_recomputed, days_skipped_partial,
                        error_summary NULL)
ingest.quarantine      (quarantine_id BIGINT PK, run_id, tenant_id NULL, source_table, source_pk,
                        reason_code,                                  -- unknown_device | unknown_alias | unexpected_movement
                                                                      -- | unparsable_hours | bad_xml | amount_not_divisible
                                                                      -- | negative_amount | unlinked_coin | no_branch_tz
                                                                      -- | unknown_client | schema_mismatch | children_missing
                                                                      -- | unknown_message
                        payload_excerpt VARCHAR(4000) NULL,          -- allowlisted columns only
                        quarantined_at, resolved_at NULL, resolution NULL)
ingest.activity_map    (source_system, source_code VARCHAR(48),
                        category VARCHAR(20),                         -- customer | customer_reversal | load | remove
                                                                      -- | vault_transfer | correction | ignore_duplicate
                                                                      -- | ignore_noncash | ambiguous
                        effective_from DATE, effective_to DATE NULL,
                        PRIMARY KEY (source_system, source_code, effective_from))
ingest.ej_txn_type_map (source_instance_id, txn_type VARCHAR(100), txn_status VARCHAR(150),   -- free text, per vendor/language
                        category VARCHAR(12),                         -- dispense | deposit | other
                        is_success BOOLEAN,
                        PRIMARY KEY (source_instance_id, txn_type, txn_status))
```

A tenant is resolved per row (iQ client; CO `TenantId` scoped by source instance), not per feed, because iQ serves
every bank and one Link or CO database can hold several clients. A feed and its checkpoints live in ci-ingest's
control store (§14.1); each resolved row is written to its tenant's database, and that database keeps its own copy of
`ingest.run`, `quarantine` and `reconciliation` rows for its tenant. A row that cannot be resolved stays in the
control store's quarantine (`unknown_client`) and is never written to a tenant database.

**Activity classification** (proposal in `research/60` §7, re-checked in `research/82` §3.9; product sign-off needed on
*vault_transfer* and the *ambiguous* set, Q10). An alias maps to a **category**; the **direction** (into or out of the
device) always comes from the XML `notesDeposited`/`notesDispensed` fields, because one row can carry both and both S4
and iQ sum the XML.

| Category | Aliases |
|---|---|
| customer | TDisp, TDpsit, MDDMDisp, MDDMDpsit, SummitDisp; TCDpsit (role **unverified**: the driver overwrites its alias with Close, so it may never carry notes) |
| customer_reversal (nets against the same side) | UndoDpsit, MDDMUndoDpsit |
| load | Load, LdCassette, LdCoins, DispLoad, Replenish |
| remove | Unload, EmptyCassette, Empty, EmptyAll, UnLdCoins, DispUnLoad, DispRmInve, DispRmPartInve, Deplete |
| vault_transfer (TCR ↔ branch vault; treated like load/remove pending sign-off) | VSell, VBuy, UndoVtSell |
| correction (excluded from usage) | CDisp, CDpsit, AudCDisp, AudCDpsit, LastCorrect, Self-audit |
| ignore_duplicate | DpsitAccept, StartDeposit |
| ignore_noncash | TSDpsit (the open-bracket command of a two-phase deposit; the counted deposit is journaled as TDpsit), and the device-command aliases listed in `research/60` §7 (AlarmOff … WithdrawBook, plus the Java-only spellings) |
| ambiguous (excluded from facts until sampled and mapped) | Deposit, Deliver, DispAccept, DispByNeom, DispByCassette, HoldupDisp (a real duress dispense on kiosk/Aspyn devices, `AspynUtils`; likely *customer*, Q10), Inventory, Reject, Unknown |

Unknown aliases are quarantined (`unknown_alias`); an `ignore_noncash` or `ambiguous` row that carries XML movement is
quarantined (`unexpected_movement`).

## 5.2 Staging

Staging holds narrow, allowlisted copies of source rows **keyed by the source's own primary key within its source
instance**, so that rows which mutate after insert (Link `side`, `close_flag`, re-dated start inventory; CO approvals
and order status) and rows that arrive twice (S4DataSync resends) upsert cleanly and the affected days are
recomputed. Each iQ client is its own source instance (`instance_ref` = the client), because Link ids restart in every
Link database. Retention: 45 days for iQ rows, the re-read window plus margin for CO. Staging is not queried by the
product.

| Staging table | Source | Key | Notes |
|---|---|---|---|
| `stg.link_journal` (+ `_denom`, XML shredded on load) | `cfm_journal` | `(source_instance_id, id)` | Tenant from the iQ client (Link `client_id` is NOT NULL since the S4 upgrade). Purge deletes (disabled devices) are ignored |
| `stg.link_start_inventory` | `cfm_start_inventory` | `(source_instance_id, id)` | **Many rows per device per day** (auto balance, manual balance, backfill); a row can be re-dated in place (§7.2). Purge deletes are ignored |
| `stg.link_live_inventory` | `cfm_live_inventory`, `cfm_live_device_inventory` | `(source_instance_id, table, id)` | `cfm_live_device_inventory` has no UNIQUE on device; take the latest per device |
| `stg.link_downtime` | `cfm_device_downtime` | `(source_instance_id, id)` | iQ carries no deletes for downtime; open episodes close by the stale rule (§8) |
| `stg.cm_order`, `stg.cm_order_detail` | `CashOrders`, `CashOrderDetails` ∪ their `*History`; `CashOrderRequests.IsEmergency` | `(source_instance_id, CashOrderId)`; `(source_instance_id, CashOrderDetailsId)` | Dedupe on id preferring live; orders are archived daily. `IsEmergency` is captured on first sight (requests are purged daily and their history omits it). Per-terminal state is `CashOrderDetails.TermStatus` |
| `stg.cm_contract_tariff` | `ContractRulesDetails` (+ the order → contract-rule mapping in `ContractCashOrderMap`) | `(source_instance_id, source PK)` | `ExecutionRunCost` per contract rule and run type, for `cash_order.cost_amount`. `ContractCashOrderMap.RunCost` is always 0 and is not read |
| `stg.cm_load_manual` | `DailyCashLoadDetails` ∪ history | `(source_instance_id, TenantId, TermId, CashOrderId)` | Read base tables, not `view_DailyCashLoadDetailsComposite` (no `TenantId`, no history) |
| `stg.cm_load_auto` | `DailyCashLoadDetails_Auto` | `(source_instance_id, CashLoadId)` | One row per terminal per day, accumulated in place. Staged `Hn_Amount`/`HnD_Amount` and the stored `CashOrderId` (0 → NULL) |
| `stg.cm_balance`, `stg.cm_position` | `DailyCashBalanceDetails`, `DailyCashPositionsDetails` ∪ history | Live `(source_instance_id, TenantId, TermId, LoadDate)`; history `(source_instance_id, HistoryId)` | History rows exist only after a terminal rename and keep the **old** `TermId`. `Total_DispensedAmount`, `Total_DepositedAmount` and `Total_PrvDayEnd` are staged for the independent check (§5.3) |
| `stg.cm_hopper_profile` | `TermHopperProfile` | `(source_instance_id, TermKey)` | No `TenantId` in the source: tenant via `TermProfile`. Daily diff → `ref.device_bin_profile` SCD2 (CO keeps no as-of history) |
| `stg.cm_preforecast` | `PreForecast` | `(source_instance_id, TenantId, TermId, TermType, ForecastDate, UploadedOn)` | CO MERGEs each run over the last on `(ForecastDate, TenantId, TermId)`, so every write must be staged before the next CO run (Mon–Fri nightly). A write not staged in time is lost and recorded as a `missed` run (§9.1) |
| `stg.ej_txn` | `EJ_TRANSACTIONS` ∪ `_HISTORY` ∪ `_ARCHIVE` | `(source_instance_id, TenantId, Id)` | One allowlisted row per transaction (§13): no card, account, check or EMV content. A check-deposit flag is computed in the source query |

Source transforms that must happen in the staging→core step:

- **Link cassette XML** → (denomination value, currency, counts). Coin `value` is in cents; divide by 100 for
  `cdisp`/`crcyr` devices (S4's own rule). `vault_coin` units are unverified: check a sample before applying ÷100.
  Multi-denomination cassettes are kept (iQ drops them and undercounts). Bin role comes from `ref.device_bin_profile`
  (model-level), because the XML has no bin-type element.
- **Link timestamps** are UTC (S4 forces its JVM zone to UTC). Store them as `DATETIME(3)` UTC, then assign
  `business_date` in the branch's `timezone_id` (journal: `cfm_journal.branch_id`'s branch, else the device's) in
  ci-ingest with java.time (§15). Journal rows use the **same cutoff as the TCR EOD** (`eod_cutoff_local_time`, §7.2): a row stamped
  before the cutoff local time belongs to the previous business day, so each day's flows fall between its SOD and EOD
  and the balance identity holds. Downtime and open minutes stay on the branch-hours calendar day (§8). Link stores Java zone ids, including legacy `US/*` links, which java.time resolves; an unknown id is quarantined (`no_branch_tz`).
- **CO dates**: `business_date = DATE(LoadDate + INTERVAL TermOffset MINUTE)`, which is CO's own conversion back to
  terminal-local time; EJ auto-loads use the date part of `LoadDate` unchanged (CO stores the EJ file date there; its
  zone is unverified).
- **CO hoppers** H1..H6 are positional. Unpivot through `ref.device_bin_profile` valid on the business date. For
  history before onboarding, use the `CashOrderDetails.Hn_Value` of the latest order loaded **on or before** the day
  (an order describes the cassettes it loads) and mark the snapshot or day `derivation = 'nearest_order'`; with no
  such order use the current profile and mark it `current_profile`. Both are inferred: a hopper change CO did not keep
  can put a day's counts on the wrong denomination while its totals stay right.
- **CO amounts → counts** branch on the RBAC tenant attribute `Cash_Mode`, copied to `ref.tenant.cm_cash_mode` by the
  reference sync (a CO tenant with no value is quarantined, never assumed):
  - `Amount` tenants: count = amount ÷ face value, rounded; the remainder is kept in `amount_residual`. A residual
    above tolerance or a negative value is quarantined.
  - `Bill_Count` tenants: balances (`Hn_PrvDayEnd`), positions, order amounts and `PreForecast` values are already
    counts and are used as counts directly (amount = count × face value). CO recomputes `Total_PrvDayEnd` after its own
    conversion, so it is not trusted there. Positions are inferred to be counts; load units are unverified (§16 Q18).
- **CO approval**: only `ApprovalStatus` 2 (Approved) and 4 (Approved/Updated) feed facts; pending rows are staged and
  promoted when approved; rejected rows never feed facts. Treating EJ auto-loads (written as `'0'`) as approved is a
  **CI convention**, not a CO rule. CO's own screens use unapproved balances for end cash and unfiltered positions for
  forecasting, so CI totals can differ from CO screens.
- **EJ transactions**: TxnType and TxnStatus are free text that varies by vendor and language, so each source instance
  has its own `ingest.ej_txn_type_map` (category and success). Check deposits are excluded. Business date =
  `DATE(TransactionTimeLocal)`, not `Ej_File_Date`. Rows move to `_HISTORY` after ~90 days and `_ARCHIVE` after ~180
  (configurable at the source), so the feed reads all three. The source schema is checked on every run (§5.4).
- **At least once, made idempotent.** A feed's checkpoint lives in ci-ingest's control store and its staging rows
  in each tenant's database, so the two cannot share a transaction. A batch is therefore processed at least once:
  ci-ingest writes each tenant's rows in that tenant's own transaction, and only when every tenant write of the batch
  has committed does it advance the checkpoint (fenced, §14.2). A crash in between replays the batch from the old
  checkpoint, and the version guard below makes the replay harmless. A tenant whose write keeps failing is parked
  (`tenant_scope`, §14.1) so the others advance.
- **Version guard.** Every staging row carries `source_seq`, the position of the source write it came from:
  the Kafka offset (iQ's topic is keyed by client, table and id, so one row's versions arrive on one partition in
  order), the CO connector run's `asOf` followed by its run id, or, for a Mongo poll, the time the poll read it. An
  upsert ignores a row whose `source_seq` is older than the stored one. A newer row with the same material hash (the
  allowlisted columns) only moves the stored `source_seq` forward, recomputing nothing, so a delayed older version
  that arrives later is still recognised as older; a newer row with a different hash replaces the staged row and its
  day is recomputed. A replay or a resend therefore changes nothing.
- **Mongo poll change detection.** iQ replaces a document in place when its Link row changes, and its
  documents carry no reliable update stamp (`iq_sync` is a candidate, unconfirmed: §16 Q3). The Mongo poll (the
  interim path until the Kafka feed is agreed) therefore re-reads a window by `rec_timestamp` (the last
  `window_days`, as the CO connector does) and stages a document only when its material hash differs from the staged
  row's. A change to a document older than the window is caught by the content check of §5.3 ("Counted at the
  source") and recovered as a late correction (below).
- **Staging retention.** The maintenance job deletes staged rows whose `business_date` is older than the
  feed's horizon (45 days for iQ; `window_days` plus 14 for CO; the stored CO batches of §5.5 with their run), in
  batches by primary key, never by a whole-table statement, and records what it removed on the run. A staged row is
  never deleted while its day is incomplete.
- **Late corrections.** A source change for a day older than the staging horizon cannot be recomputed from
  staging. ci-ingest then marks the day incomplete for that device and asks for a **backfill** of that device-day:
  for iQ it re-reads the day's documents from iQ's Mongo copy, which keeps full history; for CO it posts a backfill
  request that the connector collects from `GET /connector/v1/health` and answers with a run for that table and
  date range (§5.5). When the day's rows are staged again it is recomputed and reconciled.
- **Recompute only complete days.** A touched `(device, business_date)` is recomputed only if all of that day's source
  rows are still in staging; otherwise the change is logged, the day counted in `days_skipped_partial` and a backfill
  requested (above).
- **Rollups on every feed path.** iQ (TCR), Cash Optimizer and EJ runs each end with the location-rollup step for the
  (branch or site, business_date) pairs they touched (§7.3a). Without it, branch-days fed only by ATMs have no rollup
  (`research/80` §6 test 14); the nightly reconciliation sweep checks rollup parity and rewrites any pair that breaks.
- **Tenant on staging rows.** Rows are routed to their tenant before staging (the source key that names the tenant
  — iQ client, CO `TenantId` within its instance — is resolved first), so staging lives in the tenant's database and
  every staging row, parent and child (`_denom`, `_line`), carries a non-NULL `tenant_id`; their keys are the source
  keys above.
- **Source formats are adapters.** Staging is in CI's row shape, not the source's: column names are source-neutral
  (e.g. `source_value`, not an XML element name). A reader adapter in ci-ingest owns the source's **shape** only: it
  parses the source (shreds the cassette XML, reads child tables) into staging rows that keep the source's values in
  each staging column's declared unit; an adapter for a new format converts its values to those same units. The
  **value rules** of this section (direction, coin cents, CO amount → count, approval) are applied once, by the
  staging → core transform, for every format. The adapter is selected by source instance, entity and effective date
  (the format register below). A new source format — for example S4
  replacing the cassette XML with child tables, possibly for new clients only — adds an adapter that fills the same
  staging rows; core tables, jobs and pages do not change. When a second format first appears, a dated format
  register per source instance and entity is added (an additive migration; existing instances default to the
  current format). The Kafka consumer does not process message types it does not know, so new source tables on
  iQ's topic never stall the feed, but it does not drop them either: each is quarantined as `unknown_message` with its
  identity only (topic, partition, offset, source table and key; no payload), before the checkpoint advances. When an
  adapter for that type arrives, those rows are re-read from iQ's Mongo copy, which keeps full history, so Kafka
  retention never loses them. Detection and migration are in §5.4.

## 5.4 Source changes and format migrations

Only the reader adapters and staging see a source's structure (§5.2); core tables, jobs, pages and the UI see CI's
model. An upstream change is therefore contained to one adapter and at most one staging table, shipped as an
expand-only migration. What the design adds is detecting a change and stopping safely.

- **Checked on every run, not only at onboarding.** Each run compares what the source holds with what the adapter
  expects for that source instance: the columns and types for CO and EJ (from the source's catalogue), and the fields
  of every document or message for iQ (Mongo and Kafka have no fixed schema).
  - An allowlisted field missing or retyped stops the feed for that source instance only. Its checkpoint holds
    (parking, §14.1), other instances and tenants keep flowing, and an alarm is raised. Once the adapter is fixed the
    feed re-reads from the held position: iQ's copies keep their history, and CO and EJ are re-read over a window.
  - A new field is logged and reported but never read (P10); nothing stops.
  - A document with a required field absent or of the wrong type is quarantined as `schema_mismatch`. A rate above
    the per-run threshold stops the feed as above, so a rename upstream cannot fill CI with NULLs.
- **Versions per source instance.** CO runs at each bank, and banks upgrade on their own schedule, so the connector
  supports several CO versions at once: each run records the version it detected in `ingest.run.source_version`
  (`ingest.source_instance.source_version` holds the current one), so a replay of earlier rows uses the adapter their
  run used; the connector picks the adapter for that version range, and an unknown version fails closed for that instance. iQ
  and Link/S4 change once for every tenant; the per-run check catches them.
- **Format migrations.** A new source format (for example S4 replacing the cassette XML with child tables, possibly
  for new clients only) adds an adapter that fills the same staging rows (§5.2), and the format register records each
  source instance's format with the date it took effect. A run uses one adapter per (source instance, entity), so
  `ingest.run.source_version` identifies the adapter of every row it staged; while a client switches, the old and new
  forms are read by separate runs. Rows can arrive in both forms: CI
  reads both, uses the new one, and compares them per row as an extra reconciliation measure, matched on the logical
  identity both formats share: journal id, device, bin and denomination. A row present in only one form is a
  reconciliation break, and so is a matched pair whose values differ; two matches for one identity are quarantined.
  A break holds the day incomplete (§5.3); the new-format row stays the one used while it is open. The two adapters
  never write the same staging row: from the switch date only the new format's adapter fills staging, the old
  format's rows go to a comparison store added with the format register, and an old-format row for that instance
  dated after the switch (a retry or replay) is never staged. The old path is retired
  for that client once the comparison stays clean for an agreed period. The new risks are in how rows arrive:
  - Parent and child rows arrive separately (different messages or collections). A parent waits for its children,
    only complete days are recomputed (§5.2), and a parent still incomplete after the wait is quarantined as
    `children_missing`. A child that arrives later looks up a quarantined parent by its key, releases it and requeues
    the day's recompute; the nightly sweep retries the rest within the 45-day window, after which the quarantine is
    final and the day stays a reconciliation break. Source totals (§5.3) are the backstop.
  - Deleted child rows. iQ drops deletes except for devices (§4); with child tables, a delete that never reaches CI
    is over-counted. iQ must sync deletes for any new table CI reads (§16 Q51).
  - Lineage. Appendix A rows that cite an XML element get a second origin for the table format.
- **Early warning.** Each source version CI supports has a recorded schema fixture in CI's repository; the same
  fixtures drive the iQ and CO simulators (§14.3). Each fixture set has the four cases the per-run check handles (an
  allowlisted field present, missing, retyped, and a new field), and every simulator's contract test runs all four. CI asks the iQ, Link/S4 and CO teams for notice before any change
  to the tables and fields CI reads, and to run CI's contract tests in their own pipelines (§16 Q51–Q53).

## 5.3 Reconciliation controls (decision G)

```sql
ingest.reconciliation (run_id, tenant_id, business_date,
                       device_id INT NOT NULL DEFAULT 0,              -- 0 = aggregate row (a PK column cannot be NULL)
                       measure VARCHAR(24),                           -- row_count | dispensed_amount | deposited_amount
                                                                      -- | loaded_amount | removed_amount | correction_amount
                                                                      -- | eod_amount | ordered_amount | txn_count
                                                                      -- | error_count | downtime_minutes
                       check_kind VARCHAR(8),                         -- equal | bound (source is an upper bound)
                                                                      -- | source (read at the source itself, below)
                       source_value DECIMAL(19,4),                    -- computed from staged source rows
                       ci_value DECIMAL(19,4),                        -- computed from CI core tables
                       excluded_value DECIMAL(19,4),                  -- see below
                       difference GENERATED ALWAYS AS (source_value - excluded_value - ci_value) STORED,
                       status VARCHAR(8),                             -- match | break | explained
                       source_object VARCHAR(128) NOT NULL DEFAULT '',  -- the source table or collection of a 'source' check
                       PRIMARY KEY (tenant_id, run_id, business_date, measure, device_id, check_kind, source_object))
```

Every feed run writes control totals for each touched day. `source_value` is computed **only from staged source
rows** with the documented filters; `ci_value` only from CI's core tables. `excluded_value` is the staged movement CI
deliberately leaves out of the measure: quarantined and unapproved rows, **and movement in other categories** (for a
usage measure: load, remove, correction, vault transfer, ignored and ambiguous rows). So the source side sums every
alias except the source's own duplicate (DpsitAccept, as S4 does) and never reuses `activity_map`. An `equal` check
breaks on any non-zero `difference`; a `bound` check breaks when `ci_value` exceeds the bound. A break alarms, and the
day is marked `is_complete = 0` until explained. The nightly sweep re-checks 45 days, so a late source change that CI
missed also surfaces as a break.

- **Counted at the source.** The checks above compare staging with core, so a row the reader never staged is
  missing from both sides and matches. Each feed is therefore also checked against the source itself, with
  `check_kind = 'source'` on the `row_count` measure, one row per source table or collection (`source_object`). The
  count compared is every row of that table inside the run's window that is in staging after the run, whether the run
  wrote it or found it unchanged: for CO, it must equal the connector's `runComplete` count for the table, read at the
  bank (§5.5); for iQ, the nightly sweep counts the documents per client, collection and business day in iQ's Mongo
  copy over the staging horizon and compares them with the staged rows for those days. Counts alone miss a changed
  amount, so iQ's sweep also compares a **content digest** per client, collection and business day (an order-free
  hash over the allowlisted fields, computed by an aggregation in Mongo and by ci-ingest over the staged rows); and
  because staging keeps only its horizon, ci-ingest keeps each device-day's digest after the rows are purged, and a
  weekly sweep compares the digests over iQ's full history. A difference in a count or a digest is a break: it
  alarms, marks the days incomplete and requests a backfill (§5.2).
- **Quarantined rows keep a day incomplete.** A quarantined row whose tenant and device are resolved belongs
  to a device-day, and that day is `is_complete = 0` whatever its measures say, until the row is processed again after
  the fix (an alias mapped, a zone corrected, a coin dispenser linked) or a person accepts the day with a reason,
  which is audited and writes `explained`. So a day can no longer look complete because the rows that would have
  changed it were set aside. A quarantined row with no device (`unknown_client`, `unknown_device`) belongs to no
  device-day; it alarms per source instance instead.

- **Customer reversals.** UndoDpsit returns notes through the dispensed column but nets against deposits (§7.3). For
  `deposited_amount`, `excluded_value` therefore adds the reversal: Σ deposited movement in other categories
  + Σ dispensed movement of `customer_reversal` rows (a $100 deposit and a $20 reversal: 100 − 20 − 80 = 0). For `dispensed_amount` the reversal rows are part of the excluded
  movement.
- **Explained automatically.** A balance-identity break on *d* or *d+1* where the EOD of *d* is `source_backfill`
  is written as `explained` with the reason `source_backfill`, and the page shows those days as backfilled. Every
  other `explained` status is set by a person, with an audit row.

Not every measure can be independent of CI's own transform. Those are marked, and each has a cross-check instead:

| Measure | Independent source query | Cross-check where not independent |
|---|---|---|
| row_count (all feeds) | Yes | — |
| Link dispensed / deposited | Yes: Σ value × `notesDispensed`/`notesDeposited` over all aliases except DpsitAccept, other categories in `excluded_value` | — |
| Link loaded / removed / correction | **No** (needs the category map) | S4 balance identity per device: start inventory + movement = live inventory at the next auto balance |
| Link eod_amount | **No** (EOD selection is CI's rule) | Latest day vs S4's latest-row rule; iQ day summary where available |
| Link txn_count | **No** (customer category) | `bound`: distinct `transaction_id` over all aliases, or iQ `CFMJournalDaySummary.numberOfTransactions` for calendar-day totals only (iQ dates journals by branch-local calendar day with no 09:00 cutoff, so it bounds a two-day sum, not one CI day) |
| Link error_count | Yes: journal rows carrying `<errorCode>` | — |
| Link downtime_minutes | **No** (CI clips to branch hours) | `bound`: Σ `downlength` of the day's episodes |
| CO dispensed / deposited, eod_amount | Yes for `Amount` tenants (`Total_*` columns, approved); **no** for `Bill_Count` (needs the bin profile; `Total_PrvDayEnd` not trusted) | `Bill_Count`: row_count plus Σ hopper counts per row |
| CO loaded / removed | Yes: Σ `H1..H6_Amount` / `HnD_Amount` over manual ∪ auto loads | — |
| CO ordered_amount | Partly (repeats the effective-order rule) | Σ `Hn_ActualAmount` (+ buffer), independent of `TotalAmount` |
| EJ txn_count | Yes: staged per-transaction rows, counted per mapped type | `bound`: all staged rows for the terminal-day, no type filter |

This is the production guarantee that no figure on a page is made up: every daily total is tied back to source rows by
an independent calculation where one exists, and by a stated cross-check where it cannot.

## 5.5 CO connector batch interface

How the CO connector at a bank delivers what it reads (§5.1) to ci-ingest. The route and the pattern are decided
(DEC-025): the Crossroads Ingress Gateway, the same way the Fraud Review Portal's bank agent reaches its service. The
details below are the interface CI builds against; what Crossroads must still confirm is in §16 Q43.
*(Proposed: not part of the approved baseline until the architecture owner approves it.)*

- **Route.** The connector calls out over HTTPS (443) to the Ingress Gateway, which forwards to a route of ci-ingest
  in CI's namespace, declared in `config/ingress/`. Every call is started by the connector; nothing calls in to the
  bank. No ingress topic and no Data Triage step is used in R1.
- **Endpoint.** `POST /connector/v1/batches`: one batch per call. `GET /connector/v1/health` needs a token and
  answers the connector's registration status, for the installer and for monitoring.
- **Authentication.** OAuth 2.0 client credentials, one client per installed connector (per bank), issued by the
  platform's identity provider with the ingestion audience. The secret reaches the connector from the bank host's
  protected store or its environment, never from a file in the installation and never in a log. The connector caches
  the token until shortly before it expires.
- **What ci-ingest checks itself**, whatever the gateway has already checked: the token's signature against the
  platform JWKS, issuer, audience and expiry; that the token's client is a registered connector that is `active`
  (two new columns on `ingest.source_instance`: `connector_client_id` and `connector_status` `active` | `revoked`;
  else 403 `connector_not_registered` or `connector_revoked`); and the tenant, taken only from the token's
  verified institution claim (§11), never from the batch, a header or the path. The tenant must be `active` (a frozen
  tenant is read-only, §11, and ci-ingest writes); a connector whose source instance names another tenant
  (`tenant_hint`) is refused as not registered. CO `TenantId` values in a batch are
  resolved only within that tenant's routing keys (§6); a row for another institution is quarantined, never routed.
- **Batch.** JSON, gzip-compressed (`Content-Encoding: gzip`): `connectorVersion`, `sourceVersion` (§5.4), `runId`
  (one per connector run), `asOf`, `database`, `table`, `mode` (`window_reread` or `full_snapshot`), the `window` for
  a windowed table, `columns`, `part` and `parts` (a table's rows are split so that a batch stays under 4 MiB
  uncompressed), and `rows` as arrays in column order. Only allowlisted columns and computed values appear (P10).
  A windowed batch's window ends where the table's read ends, the day after `asOf` plus the table's look-ahead
  (`feeds.json`); a catch-up run widens only the start. After a run's last table the connector sends a batch with
  `"runComplete": true` (a JSON boolean) and `rowCounts`, the row count per table, naming every table of the source
  version (an empty table is sent as one empty part), so ci-ingest knows the run is whole before it applies a
  `full_snapshot` or closes a window. Dates must lie in MySQL's `DATE` range; the body must be valid UTF-8.
- **Idempotency.** The key is (connector, `runId`, `database`, `table`, `part`). ci-ingest stores the batch and its
  answer's body in the same transaction; a repeat of a stored batch stages nothing twice (see Answers). The run id is
  compared exactly (run ids differing in case are different runs). A repeat whose content differs from the stored
  batch gets the stored answer and is logged.
- **Answers.** 202 accepted (stored; staging follows) for a batch seen for the first time; 200 for a repeat of a stored
  batch, with the same body as the first answer, so the connector treats both as delivered; 400 malformed; 401 token refused; 403
  connector revoked or not registered; 409 `source_version_unknown` (the feed stops for that source instance, §5.4);
  413 too large; 422 a batch that can never be accepted (unknown table, columns outside the allowlist, a window that
  does not end where the table's read ends, a batch after its run is complete, a `runComplete` that does not match
  what is stored); 426 a `Connector-Version` missing or no longer supported; 429 and 5xx try again later.
- **Delivery from the bank.** The connector's outbox is the durable store: a run is written whole (§5.1), then sent
  oldest run first, tables in order, parts in order. 2xx marks a part sent; 400, 413 and 422 park the run and raise an
  alarm (they need a new connector or configuration, not a retry); 401, 403, 409, 426, 429, 5xx, any other answer and
  timeouts stop the cycle and retry with backoff from 30 seconds to 15 minutes with jitter, the run staying queued
  (409 and 426 also raise an alarm). Every call has a deadline for the whole exchange, body included. Progress is kept
  per part, so a restarted connector resumes where it stopped. A run is removed from the outbox only after its
  `runComplete` batch is accepted.
- **Gaps.** While the bank is cut off from the gateway the connector keeps reading on schedule and its runs wait in
  the outbox, each covering its own window, so nothing is lost as long as the connector runs. When the connector
  itself did not run for longer than the window (host down, credential to CO revoked), rows changed in that span would
  fall outside the next window. So the connector keeps the `asOf` of its last completed read (recovered from the runs
  still in its outbox if it stopped before recording it), and a run whose normal window would start after that date
  (with 7-day windows, a read 7 or more days later) widens its window back to it (a catch-up run, marked `catchUp` with the span
  in each batch). ci-ingest checks the same from its side: when a run's window does not reach back to the last
  accepted run's `asOf` for that connector, it accepts the run, marks the uncovered business days incomplete for that
  source instance (§5.3) and raises an alarm; those days stay incomplete until a catch-up run or a backfill covers
  them. How far back CO keeps the rows a catch-up needs is part of §16 Q52.
- **Backfill requests** (approved with DEC-026; the transport details stay as proposed as the rest of this section).
  ci-ingest cannot call the bank, so a backfill it needs (§5.2 "Late corrections", §5.3 "Counted at the source") is
  queued for the connector and returned in the answer to `GET /connector/v1/health`, which the connector calls before
  each send cycle: table, date range and a request id. The connector answers with a **backfill run**: its own `runId`,
  `mode` `backfill` and the `backfillRequestId` in each batch, the requested range as the batch `window` (a backfill's
  window is the request's range, not the read window ending after `asOf`), and a `runComplete` whose `rowCounts` names
  only the requested table. ci-ingest validates a backfill run against its open request (table and range must match,
  else 422 `backfill_mismatch`), closes the request when the run completes, and skips the gap check for it. The range
  is limited to what CO still keeps (§16 Q52).
- **Operation at the bank.** The connector sends `Connector-Version` on every call; ci-ingest answers 426 to a version
  it no longer supports. Packaging, installation, updates and who operates the connector at a bank are not designed
  yet (§16 Q43).
- **Local development.** The harness's ingress stub (§14.3) stands in for the gateway, and the mock token issuer for
  the identity provider, issuing client-credentials tokens for a registered connector.
- **As built in R1** (code repository, `connectors/co-connector`, `services/ci-ingest`): received batches are stored
  whole in the tenant database (`ingest.connector_run`, `ingest.connector_batch`) and not yet turned into staging
  rows, so the uncovered days of a gap are recorded on the run and logged but not yet marked incomplete (§5.3), and
  CO `TenantId` values are not yet resolved; both come with staging (code repository temporary measure TM-08).
