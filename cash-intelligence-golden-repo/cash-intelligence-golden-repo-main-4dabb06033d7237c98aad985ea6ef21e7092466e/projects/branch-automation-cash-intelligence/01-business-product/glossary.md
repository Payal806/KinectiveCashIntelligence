---
id: BUS-KIN-CI-004
title: Glossary
artifact_type: glossary
status: draft
content_state: derived
owner_role: TBC
applies_to:
  - PRJ-KIN-BRAU-CI
source_of_truth: golden-repo
source_refs:
  - "SRC-CONF-CIR-001"
last_updated: 2026-10-28
review_due: TBC
---


# Glossary

**Derived.** Each entry records how the term is used in `SRC-CONF-CIR-001`. Where the source does not define a term, the entry says so rather than supplying a definition.

| Term | Usage in source |
|---|---|
| ATM | A device type with cash inventory, a bill acceptor, replenish orders and optional replenish schedule. Not expanded in source. |
| TCR | A device type with notes moved, uptime/downtime, errors, left/right/access usage. Not expanded in source. |
| TCD | Appears in page titles "TCR/TCD Maintenance Status", "TCR/TCD Usage and Health" and filename "TCR-TCD Usage Report". Not defined (OQ-003). |
| Coin Dispenser | A device linked to a TCR; never shown standalone; merged into linked TCR totals; TCR label suffixed "(+ coins)". |
| Region / Branch / Branch/Site / Device | The three-level drill-down hierarchy. "Branch" and "Branch/Site" are used interchangeably. |
| EOD | End-of-day (cash total / balance). |
| SOD | Start of Day (cash amount), compared with Dispensed in Overstock Analysis. |
| Notes moved | A TCR counter used for the preventative-maintenance threshold (default 400,000). |
| Downtime | Time a TCR has been down; threshold default 240 minutes. |
| Scheduled / Unscheduled order | An ATM replenish order is scheduled if the schedule is daily or the order's weekday matches the schedule's day; otherwise unscheduled. No schedule → all orders unscheduled. |
| Flagged | A device/branch/ATM that meets a page's threshold rule (see `business-rules.md`). |
| Overstocked | A device whose effective average discrepancy exceeds the configurable threshold while having exceeded SOD fewer than the dispense-exceeds count. |
| Acceptable discrepancy | A per-device dollar amount subtracted from average discrepancy before overstock comparison. |
| Vault / Vault minimum / Vault total | Branch cash vault; minimum set by Vault Minimum Rules; total entered by the user in recommendations. |
| Predicted usage | Usage forecast over a period used in order recommendations. Method not stated (OQ-009). |
| Days until empty / Days until full | Projections used for ATM/TCR cash level and bill acceptor status. Method not stated (OQ-009). |
| Low-share device | A device below the branch average share of branch cash. |
| Cash share spread / discrepancy | Spread of average inventory share across a branch's non-scheduled devices. |
| Quadrant (TCR Usage Health) | Health (uptime ≥ 97%) × utilisation (transactions ≥ median) grouping. |
| Imbalanced | A TCR whose left/right usage differ by more than 30. |
| Proactive service | Health category for TCRs meeting an error-frequency rule in the last 7 days. |
| Needs attention now / Needs preventative maintenance | Health categories for downtime and notes-moved thresholds respectively. |
| Service record | A logged maintenance entry (type, date, technician, notes, file) that clears a flag. |
| Host system | The system that provides a Service Provider's serviced device models. Not identified (OQ-002). |
| My institution navigation | The navigation group containing the analytics pages. |
| Admin section | Navigation area containing Settings; Admin role only. |
| Enterprise access | Profile condition showing "All branches / sites". Not defined (OQ-007). |
| Client | Placeholder in the audit-report title "{Client} End of Day Totals Report". |
| Meridian Bank | Name appearing in four CSV export filenames. Whether it is a placeholder is not stated (OQ-001). |
