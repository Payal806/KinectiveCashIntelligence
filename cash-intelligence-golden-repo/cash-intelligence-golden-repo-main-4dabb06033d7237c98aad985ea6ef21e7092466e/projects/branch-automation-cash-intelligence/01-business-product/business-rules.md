---
id: BUS-KIN-CI-005
title: Business Rules
artifact_type: business-rules
status: draft
content_state: source-backed
owner_role: TBC
applies_to:
  - PRJ-KIN-BRAU-CI
source_of_truth: golden-repo
source_refs:
  - "SRC-CONF-CIR-001 (system stories and acceptance criteria)"
last_updated: 2026-10-28
review_due: TBC
---


# Business Rules

**Source-backed.** Each rule is restated from the acceptance criteria of the cited requirement; wording is kept as close to the source as possible. Threshold defaults are in `02-requirements/default-threshold-settings.md`; ranges are in `req-settings.md`.

## BR-01 Scheduled vs unscheduled order classification (`REQ-CI-ARH-008`)
1. An order is *scheduled* if the ATM's schedule is daily, or if the order's day of week matches the schedule's day.
2. Otherwise the order is *unscheduled*.
3. ATMs with no schedule have all orders counted as unscheduled.
4. An ATM is flagged when its unscheduled order count exceeds the configurable unscheduled order threshold (default 2).

## BR-02 Cost of unscheduled orders (`REQ-CI-DASH-005`, `REQ-CI-ARH-003`)
Total unscheduled orders × configurable per-order cost (default $350).

## BR-03 Overstock flagging (`REQ-CI-OSA-005`, `REQ-CI-SET-005`)
1. A device is flagged when it exceeded SOD fewer than the configurable dispense-exceeds count (default 10) **and** its effective average discrepancy exceeds the configurable average discrepancy threshold (default $20,000) over the lookback period (default 90 days).
2. Effective discrepancy subtracts the device's acceptable discrepancy amount (default rule $5,000) before comparison.
3. Coin Dispensers are excluded (`REQ-CI-OSA-001`).

## BR-04 Annual cost of excess cash (`REQ-CI-OSA-002`, `-003`)
Total average discrepancy (flagged) × (cost of funds % + insurance %) / 100. Both rates default to 5% on screen, minimum 0, step 0.25.

## BR-05 Cash Order Management flagging (`REQ-CI-COM-001`, `-003`, `-004`, `-005`, `REQ-CI-SET-006`)
1. ATM Cash Level Status: ATMs projected to run empty before their next scheduled order; no configuration.
2. Branch Cash Level Status: TCRs and ATMs **not** tied to a cash order schedule, projected empty within the days-until-empty threshold (default 5 days).
3. ATM Bill Acceptor Status: ATMs projected to reach bill acceptor capacity within the days-until-full threshold (default 3 days).
4. "Critical" = projected empty within 3 days.
5. Colour coding for days until empty/full: error ≤ 3 days, warning ≤ 7 days, otherwise info; 0 days shows "Empty now" / "Full now".

## BR-06 Order recommendation (`REQ-CI-COM-006`, `-007`, `REQ-CI-SET-007`)
1. Only branches with forecasts are selectable.
2. Recommended order per device = predicted usage over the period − current inventory.
3. Available vault funds = vault total − vault minimum (default rule $25,000); these are deducted from the branch order total and device orders are scaled down proportionally.
4. A device fully covered by vault shows "Covered by vault".
5. Default period: tomorrow through 14 days out; generation requires ≥ 1 branch and a period ≥ 1 day. Vault total seeded default $50,000.
6. Results show $100/$20/$10 bill breakdown; the CSV export lists $100/$50/$20/$10/$5/$1 bill counts (OQ-011).

## BR-07 TCR health categories (`REQ-CI-TDH-001`, `REQ-CI-SET-001`–`003`)
1. Needs attention now: down longer than the downtime threshold (default 240 min).
2. Needs preventative maintenance: over the notes-moved threshold of an assigned rule (default 400,000).
3. Proactive service: errors at or above the error-frequency threshold in the last 7 days (default 5), by Severity or by Error Code rule.
4. Submitting a service record removes the device from the flagged lists (`REQ-CI-TDH-005`).

## BR-08 Maintenance alerting (`REQ-CI-TDH-006`, `REQ-CI-SET-002`)
1. First breach of a preventative maintenance rule → immediate alert email to the rule's recipients.
2. Recurring reminders on the rule's cadence (daily, weekly, biweekly, monthly) at the configured time, client local time (defaults: once a week, 23:00).
3. Alert checker runs on page load and every 4 hours. Rules with no recipients are skipped.
4. When a breach is resolved, alert tracking is cleared so a future breach re-alerts.

## BR-09 Branch flagging for uneven cash distribution (`REQ-CI-BCP-004`)
1. Cash share spread is computed across the branch's devices **not** on a scheduled order, using average inventory over the interval.
2. Flag only when spread ≥ branch cash variance threshold (default 20%) **and** at least one overstocked TCR **and** the branch as a whole is not overstocked.
3. Overstock here mirrors BR-03.

## BR-10 Cash rebalancing recommendation (`REQ-CI-BCP-006`)
1. Device target = average daily usage × rebalance target days (default 14), with a minimum working balance floor (value not stated — OQ-008).
2. Surplus devices fund deficit devices first; remaining deficit is drawn from the branch vault down to its minimum; transfers rounded to the nearest $100.
3. A shortfall chip indicates when an external cash order is needed.

## BR-11 TCR Usage Health quadrants (`REQ-CI-TUH-002`, `-003`)
1. Health: uptime ≥ 97%. Utilisation: transactions ≥ median vs below.
2. "Imbalanced" when left/right usage differ by more than 30.

## BR-12 Coin Dispenser handling (`REQ-CI-CDI-005`, `-007`, `REQ-CI-HEOD-008`, `REQ-CI-OSA-001`, `REQ-CI-DASH-009`)
1. Never a standalone entry; inventory/EOD totals/denominations merged into the linked TCR; TCR label suffixed "(+ coins)".
2. Merging applies to charts, denomination tables, CSV exports and the audit report.
3. Excluded from Overstock Analysis; the Device Usage widget lists non-coin-dispenser devices.

## BR-13 Usage outliers (`REQ-CI-AUSE-006`, `REQ-CI-DASH-009`)
Bottom 5% and Top 5% of devices by total transactions in scope/window; Dashboard widget shows the 5 least-used devices over the last 90 days.

## BR-14 Role scoping (`REQ-CI-ROLE-001`–`004`)
See `personas-and-roles.md`. Service Provider model scoping is fail-closed.
