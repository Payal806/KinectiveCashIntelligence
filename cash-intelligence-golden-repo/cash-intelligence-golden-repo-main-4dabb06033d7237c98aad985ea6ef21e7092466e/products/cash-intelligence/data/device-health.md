---
id: DATA-KIN-CI-005
title: Cash Intelligence architecture — 8. Device health (BAP-21, BAP-85)
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

# 8. Device health (BAP-21, BAP-85)

> **Approved architecture: the R1 design baseline (DEC-006).** Approved by the Cash Intelligence architecture owner on 2026-11-05, after the 2026-11-03 design review. Generated specs follow it; a spec that needs something different is a change here first (GOV-CI-004 in `projects/branch-automation-cash-intelligence/05-delivery/delivery-model-and-gates.md`). Open questions (§16) stay open until answered. Section numbers (§n), open questions (Qn) and decisions (A–I) use the architecture's own numbering; `products/cash-intelligence/architecture/architecture-overview.md` maps each section to its file, and the open questions are in `products/cash-intelligence/architecture/open-questions.md`. Citations: `BAP-nn` = `SRC-JIRA-BAP-001`, `TDD:n` = line n of `SRC-XR-TDD-001`, `research/nn` = `SRC-CI-RES-001` (see the source index).

```sql
health.downtime_episode    (episode_id BIGINT PK, tenant_id, device_id,
                            source_system, source_instance_id, source_pk VARCHAR(64),
                            started_at DATETIME(3), ended_at DATETIME(3) NULL,
                            ended_by VARCHAR(16) NULL,                 -- source | source_delete | stale
                            initial_error_code_id INT NULL,
                            source_down_minutes INT NULL,              -- Link downlength (reconciliation only)
                            acknowledged_service_record_id BIGINT NULL,-- set by a 'device_down' service record
                            UNIQUE (source_instance_id, source_pk))
                            -- index (tenant_id, ended_at, device_id): ended_at IS NULL = currently down
health.device_error_event  (tenant_id, device_id, occurred_at DATETIME(3), business_date,
                            error_code_id INT, severity TINYINT,
                            downtime_minutes INT NULL,                 -- derived from the matching episode, else NULL
                            source_system, source_pk VARCHAR(64),
                            PRIMARY KEY (tenant_id, device_id, business_date, occurred_at, source_pk))
health.device_status_current (tenant_id, device_id PK, status, status_since,
                            open_downtime_episode_id NULL, last_error_at NULL, as_of,
                            valid_from, valid_to)
```

- **Downtime minutes and uptime %** clip episodes to `ref.location_hours` in branch time (iQ's rule). This makes
  uptime stricter than a wall-clock formula; whether the 97% cutoff assumes wall-clock uptime is open (§16 Q19). Where
  the branch hours are unknown (unparsable source), downtime and open minutes are NULL and uptime shows "n/a".
  The service computes each branch-day's hours window in the branch's zone with java.time and binds it to SQL in
  UTC (§15 Timestamps).
- A Link downtime row **deleted while open**: S4 sends the delete, and iQ saves it as a close with `downend` set to
  iQ's own processing time and `downlength` computed from it (`S4TableConverterDowntimeHandler`). The row then looks
  like a real close but its end time is approximate. ci-ingest detects it (a row that was open in staging closes with
  the delete-only payload pattern, or with `downend` within seconds of iQ's write time) and records
  `ended_by = 'source_delete'` with the end time flagged approximate; the iQ team should confirm a reliable marker
  (§16 Q3). As a backstop, an episode still open is closed as `ended_by = 'stale'` when the device has recorded
  journal activity after the episode started and iQ shows no open episode for it at the next reference sync; until
  then it is shown as open with a data-quality flag. Deletes of already-closed rows come from Link's purge and are
  ignored.
- **Needs attention now** = open episodes with no `acknowledged_service_record_id`, longer than
  `downtime_threshold_minutes`. Logging a *device down* service record acknowledges the open episode, removing the
  device from the list (BAP-77).
- Per-occurrence downtime (Error Details, BAP-89) is not on the journal; it is derived by matching the episode whose
  `initial_error_code` and start time match the error, else NULL.
- **ATM health** is R2. CO's `TermFaultDetails` (the only CO table with a rowversion) maps onto the same tables.
