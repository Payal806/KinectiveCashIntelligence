---
id: ARC-KIN-CI-103
title: Cash Intelligence architecture — 12. Extension points (R2, R3, adjacent initiatives)
artifact_type: architecture
status: approved
content_state: approved-design
owner_role: Cash Intelligence architecture owner
applies_to:
  - PRJ-KIN-BRAU-CI
source_of_truth: golden-repo
source_refs:
  - "SRC-CONF-CIR-001"
  - "SRC-XR-TDD-001"
  - "SRC-CI-RES-001"
last_updated: 2026-11-05
review_due: TBC
---

# 12. Extension points (R2, R3, adjacent initiatives)

> **Approved architecture: the R1 design baseline (DEC-006).** Approved by the Cash Intelligence architecture owner on 2026-11-05, after the 2026-11-03 design review. Generated specs follow it; a spec that needs something different is a change here first (GOV-CI-004 in `projects/branch-automation-cash-intelligence/05-delivery/delivery-model-and-gates.md`). Open questions (§16) stay open until answered. Section numbers (§n), open questions (Qn) and decisions (A–I) use the architecture's own numbering; `products/cash-intelligence/architecture/architecture-overview.md` maps each section to its file, and the open questions are in `products/cash-intelligence/architecture/open-questions.md`. Citations: `BAP-nn` = `SRC-JIRA-BAP-001`, `TDD:n` = line n of `SRC-XR-TDD-001`, `research/nn` = `SRC-CI-RES-001` (see the source index).

| Future need | Where it lands | Change required |
|---|---|---|
| **R2 AI forecasting** | `fcst.model` + `model_selection` | New model row |
| **R2 external signals** (Fed rate, weather, events, pay days) | `ext.series (series_id, series_key, provider, unit, frequency)`, `ext.observation (series_id, obs_date, location_id NULL, value)` | Add tables in R2 |
| **R2 ATM downtime / health** | `health.*` with `source_system = 'cash_optimizer'`; CO operating hours (`OPERHR` schedules) → `location_hours` | Feed + mapping only |
| **R3 vault and drawer positions** (DC Gateway) | `ref.location` types `vault`/`drawer`, `cash.inventory_snapshot`, `ops.vault_balance_entry.source = 'dc_gateway'` | New `location_type` values and a feed |
| **R3 Check21 deposit volumes** | New `cash.location_daily_check_activity` at the same grain | One table |
| **Maintenance Intelligence** | `health.*`, `ops.service_record` (`source = 'csv_import'`) | Work-order tables later |
| **Enterprise Cash Digital Twin** | Reads `cash.*`, `fcst.*` through views/APIs | None |
| **Cross-client peer comparison** | Not in this database; separate anonymized store | Legal review first |
| **KBOP product roles** (KBOP R2) | `acc.role_mapping` rows mapping a product-role claim to a CI role; `user_location_access.source = 'kbop_claim'` if KBOP adds locations | Mapping rows; no DDL |
| **Scope from token claims** (Crossroads Contract 16; shell and identity increment) | `acc.v_user_*_scope` read claim-supplied branch ids and serviced models instead of CI's tables | View inputs only; page queries unchanged (§11) |
| **Breaches in the shared work surface** (acknowledge, assign, escalate) | The evaluator publishes `ci.breach.opened` / `ci.breach.resolved` through CI's own interface whose real adapter is the Crossroads services SDK (DEC-022; the SDK's identity is §16 Q62); a Crossroads journey per breach, keyed by `breach_id` (device and rule only correlate), raises a work item and owns the alert lifecycle; `ops.breach` keeps the domain history and the service-record link | Journey, work-item descriptor, gateway tool for reminders; needs a work-item withdraw transition when a breach resolves on its own (§16 Q35) |
| **Kiosk device type** (Kiosk+Serve, Genix) | `ref.device_type` `KIOSK_RECYCLER` (family `tcr`) or a `kiosk` family, set from `is_kiosk`; coin stays `COIN_DISPENSER` and rolls into its kiosk recycler; per-device bin roles where an FI repurposes a cassette | Mapping rows and one classification rule; no new feed (`research/37`) |
| **Datava outcome write-back** (Crossroads Step 6) | Breach outcomes from `ops.breach` | Only if required (§16 Q35) |
