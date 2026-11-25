---
id: DATA-KIN-CI-007
title: Cash Intelligence architecture — 10. Configuration, rules and CI-owned operational data (BAP-92, BAP-77, BAP-78)
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
  - "SRC-XR-TDD-001"
last_updated: 2026-11-21
review_due: TBC
---

# 10. Configuration, rules and CI-owned operational data (BAP-92, BAP-77, BAP-78)

> **Approved architecture: the R1 design baseline (DEC-006).** Approved by the Cash Intelligence architecture owner on 2026-11-05, after the 2026-11-03 design review. Generated specs follow it; a spec that needs something different is a change here first (GOV-CI-004 in `projects/branch-automation-cash-intelligence/05-delivery/delivery-model-and-gates.md`). Open questions (§16) stay open until answered. Section numbers (§n), open questions (Qn) and decisions (A–I) use the architecture's own numbering; `products/cash-intelligence/architecture/architecture-overview.md` maps each section to its file, and the open questions are in `products/cash-intelligence/architecture/open-questions.md`. Citations: `BAP-nn` = `SRC-JIRA-BAP-001`, `TDD:n` = line n of `SRC-XR-TDD-001`, `research/nn` = `SRC-CI-RES-001` (see the source index).

Configuration is split by what it is (decision D):

| Kind | Where | Why |
|---|---|---|
| Scalar settings (§10.1) and rule **parameters** (thresholds, cadences, alert time, amounts) | **Crossroads Platform Config Service**, per tenant, against CI's registered schema `schemas/config/product.v1.json` | The platform versions, validates against published ranges, previews, activates and audits (`CONFIG_CHANGED`) every change, so CI builds none of that |
| Rule identity, device and location **assignments**, alert **recipients**, replenish **schedules** and their calendar (§10.2) | CI's database (`cfg.*`) | Device-level data that changes when devices move, thousands of rows per tenant, and recipient email addresses, which the platform's configuration store must not hold (it is free of personal data) |

## 10.1 Scalar settings

CI's configuration schema holds every scalar setting below as a typed field with its range, step and default; the
Platform Config Service holds each tenant's values and versions. CI's database has **no settings table**. The schema
also carries what Crossroads requires of every schema (Contract 9): a `version`; the `inferenceGateField` (declared;
CI R1 has no inference) and `autonomyLevel` `off`; a published range for every numeric field; and a **preview**
evaluator CI supplies, which runs a candidate configuration against the tenant's current data and reports what would
change (for example how many devices a new threshold would flag) without activating it. Activation needs no approval
work item in R1; an Admin's activation takes effect at once (§16 Q35).

Defaults come from the requirements' defaults table. Ranges and steps from the Settings stories are shown in brackets. Contract 9 needs a published range for every
numeric field; the stories give none for the other settings or for the rule parameters (`notes_moved_threshold`,
the error count, `amount`, `minimum_amount`), so those ranges are open (§16 Q47).

| `setting_key` [range] | Default |
|---|---|
| `downtime_threshold_minutes` [30–4320, step 30] | 240 |
| `overstock_dispense_exceeds_count` [1–30] | 10 |
| `overstock_avg_discrepancy_amount` | 20,000 |
| `overstock_lookback_days` [14–180] | 90 |
| `days_until_empty_threshold` [1–10] | 5 |
| `acceptor_days_until_full_threshold` [1–10] | 3 |
| `unscheduled_order_threshold` [0–15] | 2 |
| `unscheduled_order_cost` [0–1000, step 25] | 350 |
| `branch_cash_variance_pct` | 20 |
| `branch_rebalance_target_days` | 14 |
| `default_notes_moved_threshold` (new-rule default) | 400,000 |
| `default_error_frequency_7d` (new-rule default) | 5 |
| `default_alert_cadence` / `default_alert_local_time` | weekly / 23:00 |
| `default_vault_total` (order dialog seed) | 50,000 |
| `rebalance_min_working_balance` | 2,000 (provisional: the configuration schema needs a default, and the source gives no value or rule, OQ-008; replace it when that is answered) |
| `uptime_healthy_pct` | 97 |
| `side_imbalance_pct` | 30 |
| `eod_cutoff_local_time` (exclusive: local time before it belongs to the previous business day) | 09:00 |
| `cm_prvdayend_offset_days` | 0 (see §16 Q17) |
| `cost_of_funds_pct` / `insurance_pct` (page defaults) [min 0, step 0.25] | 5 / 5 |
| `forecast_stale_days` (stale-forecast alarm) | 4 |

ci-ingest and ci-service hold the tenant's active configuration version in memory through CI's own configuration
interface, whose real adapter is the Crossroads services SDK's Config Client (Contract 9; DEC-022), and
refresh it on the platform's change notification. A query that needs a setting receives the value
as a bind parameter; nothing in SQL reads a settings table. Rows derived from configuration (`cash.device_projection`,
`ops.device_rule_state`) record the `config_version` they were computed with, so a page can show and an auditor can
trace which version produced a figure. A configuration change enqueues projection refresh and rule evaluation for the
tenant (§10.3). Business constants are settings, not code. The cost-of-funds and insurance rates are only the
**initial values** of the Overstock page inputs; an analyst's changes on that page are a what-if, not saved
(BAP-82), because Users cannot write Settings.

## 10.2 Rules

```sql
cfg.rule               (rule_id INT PK, tenant_id,
                        rule_type VARCHAR(24),                        -- preventative_maintenance | proactive_service
                                                                      -- | acceptable_discrepancy | vault_minimum | replenish_schedule
                        name VARCHAR(100), is_active BOOLEAN,
                        is_tenant_default BOOLEAN DEFAULT 0,              -- exclusive types only
                        origin VARCHAR(12), origin_ref VARCHAR(128) NULL,   -- ci | seeded_cm | seeded_iq
                        created_by, created_at, updated_by, updated_at,
                        UNIQUE (tenant_id, rule_type, name), UNIQUE (tenant_id, rule_id, rule_type),
                        default_slot VARCHAR(24) GENERATED ALWAYS AS
                          (IF(is_tenant_default AND is_active, rule_type, NULL)) STORED,
                        UNIQUE (tenant_id, default_slot),             -- one active default per exclusive type
                        CHECK (rule_type IN ('preventative_maintenance', 'proactive_service',
                                             'acceptable_discrepancy', 'vault_minimum', 'replenish_schedule')),
                        valid_from, valid_to)                          -- versioned (history table)
cfg.rule_assignment    (tenant_id, rule_id, rule_type,
                        rule_active BOOLEAN,                           -- copy of cfg.rule.is_active, set by the same write
                        target_kind VARCHAR(8), target_id INT,         -- device | location
                        is_exclusive GENERATED ALWAYS AS (CASE WHEN rule_type IN ('acceptable_discrepancy','vault_minimum',
                                                                     'replenish_schedule') THEN 1 END) STORED,
                        PRIMARY KEY (tenant_id, rule_id, target_kind, target_id),
                        FOREIGN KEY (tenant_id, rule_id, rule_type) REFERENCES cfg.rule (tenant_id, rule_id, rule_type),
                        CHECK (target_kind = CASE WHEN rule_type = 'vault_minimum' THEN 'location' ELSE 'device' END),
                        excl_target VARCHAR(48) GENERATED ALWAYS AS
                          (IF(is_exclusive AND rule_active, CONCAT(rule_type, ':', target_kind, ':', target_id), NULL)) STORED,
                        UNIQUE (tenant_id, excl_target),              -- one active exclusive rule per type and target
                        valid_from, valid_to)                          -- versioned (history table)
cfg.rule_alert_recipient (tenant_id, rule_id, email VARCHAR(320) CHECK (email COLLATE utf8mb4_bin = LOWER(email)),
                        PRIMARY KEY (tenant_id, rule_id, email),
                        valid_from, valid_to)                          -- versioned (history table)
cfg.rule_replenish_schedule (rule_id PK,
                        frequency VARCHAR(8),                          -- daily | weekly | biweekly | monthly
                        days_of_week_mask TINYINT NULL,                -- bit 0 = Sunday … bit 6 = Saturday; NULL when daily
                                                                       -- (several days allowed: CO schedules are often e.g. Mon+Thu)
                        anchor_date DATE NULL,                         -- defaults to first matching weekday on/after creation
                        recurrence_rule VARCHAR(200) NULL)             -- RFC 5545 RRULE, for CO seeding/round-trip
cfg.rule_replenish_schedule_line (rule_id, denomination_id, note_count INT,
                        PRIMARY KEY (rule_id, denomination_id))
cfg.schedule_calendar  (tenant_id, rule_id, occurrence_date DATE,      -- materialized expansion of a schedule
                        PRIMARY KEY (tenant_id, rule_id, occurrence_date))
```

**Rule parameters** are in the configuration schema, one entry per `rule_id` (Platform Config Service, §10):

| Rule type | Parameters (in the platform) |
|---|---|
| preventative_maintenance | `notes_moved_threshold`, optional `device_model_id` |
| proactive_service | `match_kind` (severity \| error_code), `severity` (Low/Medium/High as entered, §16 Q9), `occurrences_7d` (≥ 1), `error_code_ids[]` |
| acceptable_discrepancy | `amount` |
| vault_minimum | `minimum_amount` |
| maintenance rules' alerts | `cadence` (daily \| weekly \| biweekly \| monthly), `alert_local_time`, `timezone_id` (NULL = tenant default), `is_enabled` |

`rule_replenish_schedule` and its lines stay in CI: a schedule is per-denomination note counts plus dates, read by
SQL with the calendar, and seeded from CO.

- **One rule, two stores, one write.** ci-service creates a rule by allocating `rule_id` in `cfg.rule`
  (inactive), writing its parameters into a new configuration version, and activating that version through the
  Config Client; only then does it set `is_active`. Deleting or deactivating reverses the order. A rule whose
  parameters are missing from the active version is treated as inactive by the evaluator and raises an alarm, so a
  failure between the two steps can never evaluate a rule with no thresholds.

- **Explicit precedence.** Exclusive rule types (acceptable discrepancy, vault minimum, replenish schedule) allow one
  assignment per target; a second fails, and the Settings UI must explain why (no silent "first rule wins"). Maintenance rules are non-exclusive and evaluated independently.
- **Defaults.** Each exclusive type except schedules has one tenant default rule, seeded at onboarding ($25,000 vault
  minimum, $5,000 acceptable discrepancy). `cfg.v_effective_rule(rule_type, target)` returns the assigned **active**
  rule, else the active tenant default; inactive rules are ignored. A device with no schedule has no schedule.
- **Inactive rules release their slots.** The exclusive-target and default-slot keys (§15) include the active flag, so
  deactivating a rule frees its targets and its default slot. `rule_assignment.rule_active` is a copy because a
  generated column cannot read `cfg.rule`, and MySQL does not allow a cascading foreign key to update the base column
  of a stored generated column; the write that changes `is_active` updates both in one transaction.
- **Target kinds** are fixed per rule type by the CHECK: vault minimum → location; schedule, acceptable discrepancy and
  maintenance rules → device (§16 Q30). The composite foreign key keeps `rule_assignment.rule_type` equal to the rule's type.
- **Schedule calendar.** `cfg.schedule_calendar` is the one expansion of a schedule (frequency, weekday mask, anchor)
  into dates, used by the projection's next-order date and by the planned series (§7.4). Every schedule write rewrites
  the rule's rows in the same transaction; a nightly job keeps every rule at least **60 days ahead**. Rows also reach back
  over the retention window, so the planned series can be drawn for any page range under the current schedule, as
  classification does. Classification itself stays weekday-in-mask (§7.4) as the requirement says. Monthly expansion
  waits for §16 Q20; until then a monthly schedule writes no calendar rows and its next order date shows as unknown.
- **Change history.** `cfg.rule`, `rule_assignment`, recipients and schedules are versioned in CI, and every admin
  write also writes an `audit.audit_log` row with before/after JSON in the same transaction, so a deletion records who deleted. Parameter
  history is the platform's configuration versions and its `CONFIG_CHANGED` audit records.
- **Seeding from CO** (decision D): CO RRULE event groups attached through `TermLoadProfile.LoadEventGroupId`, with
  denomination mixes from `TermHopperProfile`, become `replenish_schedule` rules with `origin = 'seeded_cm'`; CI owns
  them from then on.
- **Thresholds compare strictly**: "over" in the requirements is `>` (notes moved > threshold).
- **Error-code proactive rules** (BAP-95) are new; iQ has the column but never evaluates it.
