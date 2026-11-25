---
id: INT-KIN-CI-001
title: Cash Intelligence architecture — 4. Freshness options
artifact_type: integration-design
status: approved
content_state: approved-design
owner_role: Cash Intelligence architecture owner
applies_to:
  - PRJ-KIN-BRAU-CI
source_of_truth: golden-repo
source_refs:
  - "SRC-CONF-CIR-001"
  - "SRC-CONF-PRD-001"
  - "SRC-CI-RES-001"
last_updated: 2026-11-05
review_due: TBC
---

# 4. Freshness options

> **Approved architecture: the R1 design baseline (DEC-006).** Approved by the Cash Intelligence architecture owner on 2026-11-05, after the 2026-11-03 design review. Generated specs follow it; a spec that needs something different is a change here first (GOV-CI-004 in `projects/branch-automation-cash-intelligence/05-delivery/delivery-model-and-gates.md`). Open questions (§16) stay open until answered. Section numbers (§n), open questions (Qn) and decisions (A–I) use the architecture's own numbering; `products/cash-intelligence/architecture/architecture-overview.md` maps each section to its file, and the open questions are in `products/cash-intelligence/architecture/open-questions.md`. Citations: `BAP-nn` = `SRC-JIRA-BAP-001`, `TDD:n` = line n of `SRC-XR-TDD-001`, `research/nn` = `SRC-CI-RES-001` (see the source index).

Freshness is undefined in the PRD (§9) while the R1 brief says "real-time". The schema is **identical across all
three tiers**; only the feed configuration and refresh jobs change.

| Tier | Latency | TCR path (iQ) | ATM path (CO connector) | Notes |
|---|---|---|---|---|
| **A — Daily** | T+1 morning | Nightly read of iQ's Mongo copies (`s4_sync_cfm_*`) | Nightly 45-day window re-read | Satisfies every R1 page except intraday "current" inventory |
| **B — Micro-batch** | 5–15 min | Poll iQ's Mongo copies on a timer | Hourly 3-day window re-read; CO data itself is daily (imports) or hourly (EJ loads) | ATM latency is bounded by CO, not by CI |
| **C — Push** | seconds–1 min | Consume iQ's Kafka stream (`s4-sync-table` per iQ code comments; topic, key and retention to be confirmed) with CI's own consumer group | Not available without source changes | Highest TCR fidelity; needs the iQ team's agreement (§16 Q3) |

How the schema supports all three:

- **Current-state tables are versioned** (`cash.device_inventory_current*`, `health.device_status_current`): a trigger
  copies the prior row to a `_hist` table on every update (§15), so any tier gets intraday "as of".
- Every current row carries `as_of` (source event time) and `source_version`. The UI shows `as_of`.
- `ingest.feed.freshness_tier` and `schedule_expr` drive the jobs. Daily facts are recomputed for every touched
  `(device, business_date)`, so late, re-sent and mutated rows correct the day in place.
- A Kafka consumer that falls behind iQ's topic retention re-reads the gap from iQ's Mongo copies, keyed by
  `(client, id)`; the oldest `feed_checkpoint.source_watermark_at` of the feed alarms before the retention is reached,
  since a consumer can keep completing runs while one partition falls behind.
- **CO has no change tracking, CDC or rowversion** on its cash tables, and approvals change rows without touching
  `UploadedOn`. CO feeds therefore re-read a sliding `LoadDate` window every run and diff against staging; only order
  tables have a usable `UpdatedOn` watermark.

**TCR source: iQ only.** Link is not read directly, so no CI connector runs at the bank for TCRs. iQ has full raw
history (`research/91`) but lacks three things:

| Gap | What CI does |
|---|---|
| Reference tables never synced: `cfm_conf_cluster` and `cfm_conf_peripheral_device` (hubs, coin links, kiosk clusters); `cfm_error_code` (iQ's `s4_sync_cfm_error_code` is empty) | Ask the Link/S4 team to change-track them and add S4DataSync senders, and the iQ team to store them. `cfm_conf_cluster` must sync **safe columns only**: it holds `shared_token`, `cluster_public_key`, `os_upgrade_hub_password` and `repair_token` (`research/10`). Until a bank has upgraded, an admin uploads a **CSV** (`source_system = 'csv'`) |
| Deletes: S4 sends a delete for every change-tracked table, but iQ drops them except for devices (`sync_deleted`) and open downtime rows, which it saves as closed at its own processing time | Downtime deletes are detected and flagged approximate, with the stale rule as backstop (§8); ask iQ to keep deletes on more tables (they may be visible on Kafka, unverified) |
| Live status (`cfm_device_error_state`, `cfm_device_live_downtime`) not synced | Status comes from downtime rows and journals; ask for both |

The same request should fix the existing defect that `cfm_email_setting`, a credential table, is synced to iQ (§13).

**Routing iQ rows to tenants.** iQ is one shared, multi-tenant feed keyed by client name. ci-ingest maps each row to
its tenant through the routing copy of its external keys (`ci_catalog.tenant_key`, §14.1) and writes only to
that tenant's database. iQ has no immutable client id (§6), so routing is on the client name and a rename needs a re-map (§16 Q3). Rows from an unknown client are
held in `ingest.quarantine` (`unknown_client`), never written to a default database. A routing test runs at every
onboarding: rows from two clients land only in their own databases.

## Real-time behaviour

"Current" on the pages is not a daily figure. Each current value is a row that every feed run keeps up to date:

| Page shows | Read from | Updated |
|---|---|---|
| Current Inventory | `cash.device_inventory_current(_line)` | Replaced by every run that changes the device; the prior version moves to `_hist` |
| Device down now | Open `health.downtime_episode` (`ended_at` NULL), `health.device_status_current` | By every run that reads a downtime row |
| Days until empty | `cash.device_projection` | Refresh queued for the touched devices after each run |
| Today so far | Today's `cash.device_daily_activity` row (`is_complete = 0` until the day closes) | Recomputed for every touched device on each run |

- **Latency is the feed tier's:** A nightly; B every 5–15 min (a 100,000-row micro-batch — stage, resend, recompute,
  rollups, reconciliation — took 24 s, `research/80` §5); C seconds, through iQ's Kafka stream.
- **Source limits.** ATM current = the last approved EOD (`basis = 'last_eod'`): CO has no intraday data. The EJ column
  `RemainingNotesDetails` could supply ATM remaining notes if populated, but it is empty in the dev data
  (`research/90`) and is not on the EJ allowlist (§13). There are no per-transaction views (§1); if Product needs
  intraday trends, an optional device × hour activity table at the same pattern would serve them.
- **Change notifications.** After each feed run ci-service pushes a notification (server-sent events; the change
  signal in §14.2 tells whichever `api` replica holds the session) to sessions whose scope contains a touched device; the page then re-reads through its scoped queries, so a
  notification carries no data. Each run already records the devices it touched (its `ingest.reconciliation` rows
  and recompute-queue entries), so no schema change is needed. Filtering notifications by scope is access control:
  Legal review with §11.
- **R1 proposal (Q5):** Tier B or C for TCRs (Product must give a freshness target; the PRD has none); for ATMs the
  last approved EOD, shown with an "as of" label carrying the EOD time.
