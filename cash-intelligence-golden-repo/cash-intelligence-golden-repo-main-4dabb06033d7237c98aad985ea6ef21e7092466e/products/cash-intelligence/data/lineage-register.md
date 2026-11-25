---
id: DATA-KIN-CI-011
title: Cash Intelligence architecture — Appendix A — Data lineage register
artifact_type: data-lineage-register
status: approved
content_state: approved-design
owner_role: Cash Intelligence architecture owner
applies_to:
  - PRJ-KIN-BRAU-CI
source_of_truth: golden-repo
source_refs:
  - "SRC-CONF-CIR-001"
  - "SRC-XR-TDD-001"
last_updated: 2026-11-22
review_due: TBC
---

# Appendix A — Data lineage register

> **Approved architecture: the R1 design baseline (DEC-006).** Approved by the Cash Intelligence architecture owner on 2026-11-05, after the 2026-11-03 design review. Generated specs follow it; a spec that needs something different is a change here first (GOV-CI-004 in `projects/branch-automation-cash-intelligence/05-delivery/delivery-model-and-gates.md`). Open questions (§16) stay open until answered. Section numbers (§n), open questions (Qn) and decisions (A–I) use the architecture's own numbering; `products/cash-intelligence/architecture/architecture-overview.md` maps each section to its file, and the open questions are in `products/cash-intelligence/architecture/open-questions.md`. Citations: `BAP-nn` = `SRC-JIRA-BAP-001`, `TDD:n` = line n of `SRC-XR-TDD-001`, `research/nn` = `SRC-CI-RES-001` (see the source index).

Every column in the schema has one origin per source or release; where that differs, the row lists both (`D / N`)
or notes the later origin (`R3: DC Gateway, S`). **S** = source column (with transform). **D** = derived by CI from
named CI columns. **U** = user-entered. **C** = CI system value (keys, timestamps, versioning, origin flags).
**N** = not available from any source: stored NULL, flagged, never imputed. Columns not listed in a table below are
**C**: surrogate keys, `tenant_id`, `valid_from/valid_to` (except where noted), `created_*`, `updated_*`,
`recomputed_at`, `row_version`, `hist_id` and `hist_op` (update | delete) on every `_hist` table, `source_system`, `source_instance_id`, `source_pk`, `source_version`, the stored key-helper columns of §15
(`*_scope`, `*_slot`, `excl_target`), and **all
`ingest.*`, `stg.*` (key and load columns), `audit.*` and `ci_catalog.*` columns** except those listed below. The DDL
test suite asserts that every column in the database appears here or is in that list.

Abbreviations: **L** = Link/S4 (`cfm_*`), **IQ** = iQ, **CO** = Cash Optimizer, **EJ** = `SSTRMM.EJ_TRANSACTIONS`,
**KBOP** = KBOP token claims or services.

## stg, ci_catalog

| Table.column | Origin | Source / formula |
|---|---|---|
| stg.* business columns | S | Verbatim copies of the allowlisted source columns named in §5.2, at the source column's width (codes such as `CASH_POSITION_WITHDRAW` exceed 16 characters) |
| stg.ej_txn check-deposit flag | S→D | EJ `CheckId IS NOT NULL`, computed in the source query |
| ci_catalog.tenant.kinective_id | U / S | Entered at onboarding, verified against KBOP fi-service `salesforce_account_id`; equals the FI token's `entity_id` in 18-character form |
| ci_catalog.tenant.institution_id | U / S | Crossroads `institutionId`, from the platform's institution directory at onboarding |
| ci_catalog.tenant.db_cluster, db_name, bucket | C | Written by the provisioning step (§14.1) |
| ci_catalog.tenant.kbop_fi_status, kbop_checked_at | S / C | KBOP fi-service FI status; time of the check |
| ci_catalog.tenant_key.* | U | Onboarding; identical to the tenant's `ref.tenant_external_key` rows for routing keys |
| ci_catalog.service_provider_tenant | U | CI admin (CI is the master in R1) |
| ci_catalog.job_lease.* | C | Scheduler: replica holding the (job, tenant), lease expiry, claim counter, next due time (first occurrence after now), last success (§14.2) |
| ci_catalog.change_signal.seq, changed_at | C | Bumped after each commit other replicas must react to; row 0 for catalogue changes (§14.2) |

## ingest, audit

| Table.column | Origin | Source / formula |
|---|---|---|
| ingest.source_instance.instance_ref, tenant_hint | U | Registered at onboarding: the iQ client, or the CO server/database identifier; the tenant when one instance serves one tenant |
| ingest.feed.extract_mode, freshness_tier, schedule_expr, window_days, connector_config_json, field_allowlist_json, active | U | Feed configuration set at onboarding (§4, §5.1); the allowlist follows §13 |
| ingest.feed_checkpoint.source_table, source_partition, last_version, source_watermark_at | S | Source table or collection, Kafka partition, its offset or watermark, and the source write time of the last row read in order |
| ingest.feed_checkpoint.last_success_at | C | Time of the last successful run |
| ingest.feed_lease.owner, lease_until, epoch | C | Replica holding the feed, lease expiry, claim counter (§14.2) |
| ingest.run.started_at, ended_at, status, rows_read, rows_upserted, rows_quarantined, days_recomputed, days_skipped_partial, error_summary | C | Written by the feed run |
| ingest.quarantine.run_id, source_table, reason_code, quarantined_at | C | Run, source table and reason that quarantined the row |
| ingest.quarantine.payload_excerpt | S | Allowlisted source columns of the row only |
| ingest.quarantine.resolved_at, resolution | U / C | Operator's resolution; time stamped |
| ingest.activity_map.source_code | S | The source's activity alias or code (§5.1) |
| ingest.activity_map.category, effective_from, effective_to | U | CI classification with product sign-off (Q10) |
| ingest.ej_txn_type_map.txn_type, txn_status | S | EJ `TxnType`, `TxnStatus` values seen per source instance |
| ingest.ej_txn_type_map.category, is_success | U | Mapped per source instance (§5.2) |
| ingest.reconciliation.business_date | D | Touched business date |
| ingest.reconciliation.measure, check_kind | C | Control measure and check (§5.3) |
| ingest.reconciliation.source_value, excluded_value | S→D | Computed from staged source rows only (§5.3) |
| ingest.reconciliation.ci_value | D | Computed from CI core tables only |
| ingest.reconciliation.difference, status | D | `source_value − excluded_value − ci_value`; match or break (`explained`: U) |
| audit.audit_log.occurred_at, actor_type, actor_user_id, action, entity_type, entity_id | C | `audit_id` and `occurred_at` stamped by the insert trigger; the rest written by ci-service's write layer; actor from the validated session (§11) |
| audit.audit_log.detail_json | D | Before/after JSON of the change, or the export's scope and period |
| audit.audit_log.prev_hash, row_hash | D | Hash chain (§13); a genesis row's `prev_hash` is the archived head hash (§14.1) |
| audit.chain_head.last_audit_id, last_hash | D | Head of the tenant's chain |

## ref

| Table.column | Origin | Source / formula |
|---|---|---|
| tenant.display_name, default_timezone, default_currency | U | Onboarding |
| tenant.cm_cash_mode | S | CO RBAC tenant attribute `Cash_Mode`, by reference sync; NULL for tenants without CO |
| tenant.status | C | Lifecycle status, as in `ci_catalog.tenant.status` |
| tenant.legal_hold | U | Set by an authorized Kinective role on Legal's instruction |
| tenant_external_key.key_type, instance_scope | C | Kind of key; `IFNULL(source_instance_id, 0)` for the unique key (§15) |
| tenant_external_key.external_key | S | IQ client name (`iq_client`); L `cfm_client.id`; CO `TenantId`; KBOP `entity_id` (Kinective ID). `cfmuuid` is N on the iQ path (not synced) |
| location.name (region) | S | L `cfm_region.description`; CO `TermProfile.Region` text; else U |
| location.name (branch/site) | S | L `cfm_branch.name`; CO `SiteProfile.SiteName` via `TermProfile.SiteKey`; else U |
| location.code | U | No source (Link has no branch code) |
| location.parent_location_id | S / U | L `cfm_branch.region_id`; CO region text, matched; else U |
| location.location_type, timezone_source, geo_source, status | D | Lineage and type flags set by ingestion or onboarding |
| location.timezone_id | S / U | L `cfm_branch.timezone_id` (Java zone ids); U for CO-only sites |
| location.utc_offset_minutes | S | CO `TermProfile.TermOffset` (per terminal) |
| location.city, state_code | S | L `cfm_branch.city/state`; CO `TermProfile.City/State` |
| location.postal_code | S / N | L `cfm_branch.zip`; N for CO-only sites |
| location.latitude, longitude | S / D / U | CO `TermCustomData` per terminal → centroid (D); geocoded (D); else U |
| location_hours.* | S→D / N | L `cfm_branch_hours` text, parsed; unparsable → NULL (N) and flagged, no default |
| location_source_xref.source_key, match_method, matched_by | S / D / U | Source keys; rule or manual match |
| location_source_xref.source_entity, matched_at | C | Kind of source entity; time of the match |
| device_type.* | C | Seeded catalog and source-type mapping (§6) |
| device_model.model_code, display_name, device_type_code | S | L `cfm_device_model` (alias, name, type); CO `TermProfile.Make/Model` |
| device_model.manufacturer, end_of_life | U / N | No source in either system; admin-entered, else NULL |
| device_model_xref.source_key | S | L `cfm_device_model.alias`; CO Make+Model text |
| device.location_id | S | L `cfm_device.branch_id`; CO `TermProfile.SiteKey`/`ParentBranch`, resolved via `location_source_xref` |
| device.device_model_id | S | L `cfm_device.device_model_id`/`device_model_alias`; CO Make/Model via `device_model_xref` |
| device.name | S | L `device_physical_id`; CO `TermId` |
| device.serial_no | S / N | L `serial_id`; N for CO |
| device.device_type_code | S | L model → `cfm_device_type`; CO `TermType`; mapping in §6 |
| device.connected_to_hub, communication_mode | S | L `communication_mode` (N.O.R.M* or Aspyn ⇒ connected) |
| device.hub_mode | D | Proxy from `communication_mode` (§6); `unknown` for N.O.R.M* |
| device.is_kiosk | D | Hub cluster type first (`cfm_conf_cluster.cluster_type = 2`, Genix ⇒ true; other types ⇒ false); else the iQ-path proxy (physical-id pattern with a kiosk model alias; the journal `tag` is never ingested, §13), flagged inferred (§6); NULL = neither source available (§16 Q37) |
| device.acceptor_capacity_notes | U / N | Admin entry; CO `HigherThreshold` offered as proxy only |
| device.acceptor_capacity_source | D | `manual` or `cm_threshold_proxy` |
| device.status, administratively_offline | S | L `offline_flag` + open downtime; CO `TermStatus`, `MarkedForDelete`, `BlackListed`, `ApprovalStatus_1` → `excluded` |
| device.commissioned_at, decommissioned_at | N / U | No reliable source (`cfm_device.rec_timestamp` is unreliable per S4); admin-entered, else NULL |
| device_source_xref.external_id | S | L `id`, `device_physical_id`, `serial_id` (via IQ); `cfmuuid` N on the iQ path; CO `TermKey`, `TermId`; EJ `TerminalId` with the CO tenant |
| device_source_xref.id_kind | C | Kind of id |
| device_source_xref.valid_from, valid_to | S | CO TermId renames (`TermProfileHistory`); not system versioning |
| model_bin_map.bin_no, bin_role | U / S | Link model bin map (origin `link_model_map`), a global catalogue maintained at onboarding |
| device_bin_profile.* | S / U | CO `TermHopperProfile` Hn_Denomination, Hn_MaxCapacity (daily diff; tenant via `TermProfile`); L model map (U, maintained) |
| device_link.parent/child | D / U | L hub cluster, workstation, comm mode, physical-id pattern; admin override |
| device_link.link_type, origin, effective_from/to, set_by | D / U | Set by derivation or admin |
| denomination.* | C | Seeded catalog, incl. one Unsorted row per currency |
| error_code.* | S | L `cfm_error_code`, `cfm_generic_error_code`; unseen codes from journal `<errorCode>` with NULL description |

## cash

| Table.column | Origin | Source / formula |
|---|---|---|
| device_inventory_current_line.bin_role, face_value, inventory_snapshot_line.bin_role, face_value, device_daily_activity_denom.face_value (every fact line; denomination_id likewise) | D | Bin profile; L XML `value` + `currency`; CO `Hn_Value`/`Hn_Denomination` |
| device_inventory_current_line.amount, inventory_snapshot_line.amount, device_daily_activity_denom.dispensed_amount, deposited_amount | D | Generated: count × `face_value` (deposited: else the Unsorted amount) |
| device_daily_activity.business_date, device_daily_activity_denom.business_date, inventory_snapshot_line.business_date, snapshot_kind | D | Business date by the source's own rule (P3); detail rows copy their header's date and kind |
| Every CO-derived line's amount_residual | D | CO `Amount` tenants: source amount − count × face value (Σ absolute values where a line holds several measures); NULL otherwise |
| device_inventory_current.as_of | S | L `rec_timestamp` (UTC); ATM: approved EOD date |
| device_inventory_current.basis | D | `live` (TCR) / `last_eod` (ATM) |
| device_inventory_current.total_amount | D | Σ lines.amount |
| device_inventory_current_line.note_count (TCR) | S | L `cfm_live_inventory`/`cfm_live_device_inventory` XML `notesAvailable` per denomination |
| device_inventory_current_line.note_count (ATM dispense) | S | CO approved `Hn_PrvDayEnd` via bin profile: ÷ face value (`Amount`) or as counts (`Bill_Count`) |
| device_inventory_current_line (ATM acceptor notes) | N | No source: CO records no deposit-bin removal. No line is written |
| device_inventory_current_line.capacity_count | S / N | L XML `<capacity>` summed over the cassettes of that role and denomination (N for multi-denomination cassettes); CO bin-profile max; else N |
| device_inventory_current_line.derivation | D | `source`, `nearest_order` or `current_profile` |
| inventory_snapshot.captured_at | S | L `cfm_start_inventory.rec_timestamp`; CO `LoadDate` |
| inventory_snapshot.business_date | D | TCR: local time before `eod_cutoff_local_time` (09:00, exclusive) → previous day; CO: offset rule |
| inventory_snapshot.derivation | D | Per §7.2; `source_backfill` when a staged start-inventory id's `rec_timestamp` changes in place |
| inventory_snapshot.snapshot_kind, total_amount | D | Kind; Σ lines |
| inventory_snapshot_line.note_count | S / D | L start-inventory XML; CO `Hn_PrvDayEnd` per `Cash_Mode`; SOD = prior EOD (D) |
| device_daily_activity.dispensed/deposited_amount | S→D | L Σ XML counts × value over customer category; CO approved `Hn_DispensedAmount`/`Hn_DepositedAmount` per `Cash_Mode` |
| device_daily_activity.loaded/removed_amount | S→D | L load/remove categories; CO manual `Hn_Amount`/`HnD_Amount`; auto staged `Hn_Amount`/`HnD_Amount` (= Σ `Cassette_n_Added/Subtracted`; units unverified) |
| device_daily_activity.correction_amount | S→D | L correction category |
| device_daily_activity.txn_count, dispense/deposit_txn_count | S→D / N | L distinct `transaction_id` over customer rows; EJ rows per instance type map, successful, no check deposits, dated by `TransactionTimeLocal`; else N |
| device_daily_activity.txn_count_source, derivation | D | Which source fed the counts; CO hopper fallback |
| device_daily_activity.txn_count_left/right/access | S→D / N | L `side`, `remoteTransactionId`; N for ATMs |
| device_daily_activity.notes_moved | S→D / N | L Σ `notesio`; N for ATMs |
| device_daily_activity.sod/eod_amount | D | `inventory_snapshot` totals |
| device_daily_activity.open_minutes | D / N | `location_hours`; N when hours are unknown |
| device_daily_activity.downtime_minutes | D / N | `downtime_episode` clipped to hours; N when hours are unknown |
| device_daily_activity.error_count_* | S→D | L journal `<errorSeverity>` counts |
| device_daily_activity.is_complete | D | CI rule: no open reconciliation break, no unresolved quarantined row of the device-day, and every serving feed's lowest `source_watermark_at` past day end + lag |
| device_daily_activity_denom.*_count | S→D | As the amounts above, per denomination; NULL on the Unsorted row |
| device_daily_activity_denom.deposited_unsorted_amount | S | CO `Hn_DepositedAmount` of a hopper with no denomination in the bin profile |
| cash_order.device_id, source_order_ref, source_detail_ref | S | CO `CashOrderDetails.TermId`/`TermKey` (resolved), `CashOrderId`, `CashOrderDetailsId` |
| cash_order.planned_load_date | S→D | `DATE(CashRefillDate + INTERVAL TermOffset MINUTE)` (CO stores UTC; first terminal's offset) |
| cash_order.created_at_source | S | CO `GeneratedOn` (UTC) |
| cash_order.position_type | S | CO `CashOrderDetails.CashPositionType` |
| cash_order.status | S→D | CO `CashOrders.Status`/`SubStatus` (rejected, revised, unapproved, approved, in transit, vault report uploaded); `CashOrderDetails.TermStatus` refines dispatched / replenished per device |
| device_daily_activity.flow_missing_reason | D | ATM: why no approved position fed the day (§7.3) |
| device_daily_activity.deposited_unvalued_count | S→D | CO `Bill_Count` `Hn_DepositedAmount` of hoppers with no denomination in the bin profile |
| cash_order.source_status | S | CO `Status`/`SubStatus` (order level) |
| cash_order.source_run_type | S | CO `RunType` |
| cash_order.is_emergency | S / N | CO `CashOrderRequests.IsEmergency` via `RequestId`, captured on first sight; N for orders first seen after the request was purged (not in CO history) |
| cash_order.ordered_amount | S | CO `CashOrderDetails.TotalAmount` per `Cash_Mode` |
| cash_order.cost_amount | D | CO `ContractRulesDetails.ExecutionRunCost` for the order's contract rule and run type (`ContractCashOrderMap.RunCost` is always 0, not used) |
| cash_order.parent_cash_order_id | S | CO `ParentCashOrderId` (0 → NULL) |
| cash_order.is_full_replenishment | S / N | CO `AdditionalAttributes.$.fullReplenishment` (≥ 1.6), else N |
| cash_order_line.ordered_count | S→D | CO `Hn_ActualAmount` per `Cash_Mode` (+ `CashOrderBufferAmount` joined on detail and denomination) |
| cash_order_line.forecast_amount | S | CO `Hn_ForecastAmount` |
| v_cash_order_classified.ci_schedule_class | D | Current schedule rule's weekday mask vs order weekday |
| v_cash_order_classified.schedule_rule_id, schedule_name, schedule_frequency, schedule_days_of_week_mask | D | The device's current exclusive schedule rule (`cfg.v_effective_rule`) |
| v_device_rollup.* | D | Device facts + coin linked on each `business_date`; unlinked coin per branch (§7.3a) |
| location_daily_activity.*, location_daily_denom.* | D | Σ device facts per location as of each `business_date`; coin as its own family with `linked` |
| replenishment.device_id | S | Resolved source device |
| replenishment.business_date | S→D | L journal timestamp; CO `LoadDate` (manual: offset rule; auto: stored EJ file date, zone unverified) |
| replenishment.occurred_at | S / N | L journal timestamp; N for CO |
| replenishment.replenishment_type | S / N | L activity; CO `TermLoadProfile.LoadType` (current value only) |
| replenishment.cash_order_id, match_method | S | CO `CashOrderId` (manual); stored `DailyCashLoadDetails_Auto.CashOrderId` (auto; 0 → NULL, `none`) |
| replenishment.loaded/removed_amount, lines | S→D | As daily loaded/removed |
| device_projection.* | D | Current dispensable inventory, current forecast, schedule calendar, thresholds; `config_version` = the active configuration version used |
| device_projection.projected_full_date, days_until_full (ATM) | N | No ATM acceptor-fill source |

## health

| Table.column | Origin | Source / formula |
|---|---|---|
| downtime_episode.started_at, ended_at | S | L `downstart`, `downend` (UTC) |
| downtime_episode.ended_by | D | `source`, `stale` (§8), or `source_delete` (detection time) once iQ carries deletes |
| downtime_episode.initial_error_code_id | S | L `initialdownerrorcode` |
| downtime_episode.source_down_minutes | S | L `downlength` (reconciliation only) |
| downtime_episode.acknowledged_service_record_id | D | Set by a `device_down` service record |
| device_error_event.occurred_at, error_code_id, severity | S | L journal `rec_timestamp`, `<errorCode>`, `<errorSeverity>` |
| device_error_event.business_date | D | Business date of `occurred_at` by the journal rule (branch zone, 09:00 cutoff), so Σ events per day = `error_count_total` |
| device_error_event.downtime_minutes | D / N | Matched episode; else N |
| device_status_current.* | D | Open episode, last error, `offline_flag` |

## fcst

| Table.column | Origin | Source / formula |
|---|---|---|
| model.*, model_selection.* | C / U | Registered models; per-tenant selection |
| model.point_quantile | C / N | One model-wide quantile only; N for CO (α per cluster) |
| run.generated_at, source_ref, source_partition | S | CO `UploadedOn`, `TermType`; CI run time |
| run.model_id, horizon_days, status | C / D | 40 days (CO config), 60 days (CI baseline); `missed` for an unstaged CO write |
| run.params_json, metrics_json | C / S | CO per-cluster α (CO config); CI parameters and metrics |
| run.data_cutoff_date | D / N | CI runs; N for CO |
| run.run_month | D | First day of the month of `generated_at` |
| device_forecast.run_month, device_forecast_denom.run_month | D | Copied from `fcst.run.run_month` (partition key) |
| current forecast (model_id, run_month, forecast_date, per device and denomination); device_forecast_denom_current.forecast_date, run_id | D | From the current run on the bound day (§9.1); `device_forecast_denom_current` stores no model or vintage, which come through `run_id` → `fcst.run` |
| device_forecast.forecast_date, device_forecast_denom.forecast_date | S / D | CO `PreForecast.ForecastDate` (zone unverified, Q17); CI horizon dates |
| device_forecast.dispense/deposit_amount | S / D | CO `Total_Dispensed`/`Total_Deposit` (quantile; counts × face value in `Bill_Count` tenants); CI baseline mean |
| device_forecast.*_low/_high | D / N | CI baseline ± k·sd; N for CO |
| device_forecast_denom.*_count | S→D / D | CO `Hn_Dispensed`/`Hn_Deposit` via bin profile per `Cash_Mode`; CI per denomination |
| device_forecast_weekly.* | D | Copy of the first run per device per week from `device_forecast` (totals) |
| device_forecast_denom_current.* | D | Copy of the current run's `device_forecast_denom` rows per device |

## cfg, ops, acc

| Table.column | Origin | Source / formula |
|---|---|---|
| Settings and rule parameters (Platform Config Service) | U / S | Admin via Settings (Config Client preview and activate); defaults from CI's configuration schema; `seeded_cm` values from CO at onboarding |
| cfg.rule, rule_assignment, rule_alert_recipient, rule_replenish_schedule* | U / S | Admin via Settings; `seeded_cm` rows from CO `EventsProfile`, `TermHopperProfile`, `VaultEntityMapping` |
| cfg.rule_assignment.rule_active | D | Copy of `cfg.rule.is_active` |
| cfg.schedule_calendar.occurrence_date | D | Expansion of the schedule rule (frequency, mask, anchor) |
| ops.device_rule_state.* | D | Rule evaluator over daily facts, errors, service records, assignments and rule parameters; `config_version` = the active configuration version used |
| ops.breach.opened_at, counter_at_open, resolved_at, outcome, resolved_by_service_record_id | D | Rule evaluator (§10.3) |
| ops.breach.config_version | C | Active configuration version when the breach opened (§10.3) |
| ops.config_applied.service, config_version, applied_at | C | Last configuration version each service picked up for the tenant, recorded with the enqueue of its re-evaluation (§10.3) |
| ops.breach.alerts_sent, next_alert_due_at | D | Evaluator and reminder job: cadence and local time from the rule parameters |
| ops.alert_dispatch.* | D / S | Evaluator (initial), reminder job (reminders); `sent_at`, `provider_message_ref`, status from the email provider |
| ops.work_queue.job_kind, device_id, location_id, from_date, to_date, reason, enqueued_at, done_at | C | Written by feed runs, configuration writes and service records; closed by the job (§10.3) |
| ops.service_record.* | U / S | Technician entry through ci-service's scoped write (§11); `source = iq` from IQ `cfmservicerecord` (preventative/proactive, date only); CSV import |
| ops.service_record.serviced_at (seeded) | N | iQ holds a date only |
| ops.service_record.technician_name (seeded) | N | Not seeded (P10) |
| ops.service_record_attachment.* | U | Upload metadata via `ops_add_attachment` |
| ops.vault_balance_entry.balance_amount, location_id | U | User entry via `ops_add_vault_entry` (R3: DC Gateway, S) |
| ops.vault_balance_entry.entered_by_user_id, entered_at, source | C | Stamped from the validated session |
| ops.vault_balance_entry.request_key | C | Client idempotency key (§11) |
| acc.app_user.host_subject, auth0_sub, user_type, kbop_entity_type, kbop_entity_id, kbop_roles, email | S | KBOP token claims (`user_id`, `sub`, `user_type`, `entity_type`, `entity_id`, `roles`, `email`); `user_id` is a KBOP UUID, `entity_id` a Salesforce Account ID (FI) or a partner UUID |
| acc.app_user.display_name | S / N | KBOP access-token `full_name` claim; else N |
| acc.app_user.phone | U / N | Not in KBOP; optional CI entry (Q25) |
| acc.app_user.has_enterprise_access | U | CI admin |
| acc.app_user.host_system, last_seen_at | C | Constant `kbop`; session time |
| acc.user_role.* | U | CI admin |
| acc.user_capability.* | U | Bootstrap rule (`role_mapping`, KBOP `FIUser_UserAdmin`); versioned, so a revoked capability is not granted again |
| acc.service_provider_tenant.* | C | Read-only copy of `ci_catalog.service_provider_tenant` for this tenant, refreshed by ci-ingest |
| acc.role_mapping.* | C / U | Bootstrap rule; future product-role mappings |
| acc.user_location_access, service_provider_model | U | CI admin (CI is the master in R1) |
| acc.service_provider.host_entity_ref | S / U | KBOP `partner_id` when the provider is a KBOP Partner; else NULL |
| acc.service_provider.name, status | U | CI admin |
| acc.service_provider_user | D / U | From the token for KBOP partner users; CI admin for CI-local providers |
| acc.v_user_device_scope, v_user_location_scope, v_user_rollup_scope | D | Views over `user_role`, `user_location_access`, `service_provider_model`, `ref.device`, `ref.location` (§11) |

## Not available — what the pages do instead

| Value | Page behaviour |
|---|---|
| ATM transaction counts without an EJ feed | Device excluded from count-based rankings; shown as "—" |
| ATM intraday inventory | Shows last EOD with its date |
| ATM bill-acceptor capacity | Admin-entered; until then days-until-full is not computed for that ATM |
| ATM bill-acceptor fill (notes in the acceptor) | No source: days-until-full is not computed for ATMs; the acceptor card and table show "n/a" (Q16) |
| ATM deposits with no denomination | Exported as an "Unsorted" row; noted in the export metadata |
| Unparsable branch hours | Open and downtime minutes NULL; uptime shown as "n/a" |
| CO forecast intervals and data cutoff | Not shown for ATMs |
| CO forecast vintages CI did not stage in time | Recorded as `missed` runs; no history for those dates |
| CO replenishment time of day | Date only |
| CO emergency flag for orders first seen after the request was purged | Shown as unknown |
| Device serial for CO terminals | Not shown |
| Device commissioned/decommissioned dates, model manufacturer and end of life | Shown only if admin-entered |
| Time of seeded (pre-CI) service records | Date only |
| User phone | Shown only if entered in CI |
