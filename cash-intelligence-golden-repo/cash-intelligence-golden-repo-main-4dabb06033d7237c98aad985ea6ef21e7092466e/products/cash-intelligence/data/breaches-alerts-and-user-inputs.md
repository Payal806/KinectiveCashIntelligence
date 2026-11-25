---
id: DATA-KIN-CI-008
title: Cash Intelligence architecture — 10.3 Rule state, breaches, alerts and service records
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
last_updated: 2026-11-22
review_due: TBC
---

# 10.3 Rule state, breaches, alerts and service records

> **Approved architecture: the R1 design baseline (DEC-006).** Approved by the Cash Intelligence architecture owner on 2026-11-05, after the 2026-11-03 design review. Generated specs follow it; a spec that needs something different is a change here first (GOV-CI-004 in `projects/branch-automation-cash-intelligence/05-delivery/delivery-model-and-gates.md`). Open questions (§16) stay open until answered. Section numbers (§n), open questions (Qn) and decisions (A–I) use the architecture's own numbering; `products/cash-intelligence/architecture/architecture-overview.md` maps each section to its file, and the open questions are in `products/cash-intelligence/architecture/open-questions.md`. Citations: `BAP-nn` = `SRC-JIRA-BAP-001`, `TDD:n` = line n of `SRC-XR-TDD-001`, `research/nn` = `SRC-CI-RES-001` (see the source index).
>
> **Legal review required.** This file covers staff personal data in alert recipients, technician names and service notes, and the alert email wording. These touch compliance: Kinective's Legal team must review them before they are relied upon or selected as Astra context.

```sql
ops.device_rule_state  (tenant_id, rule_id, device_id,               -- PK
                        counter_value BIGINT,                         -- notes moved since baseline | errors in 7d after baseline
                        baseline_at DATETIME(3) NULL,              -- last matching service record (or seeded history)
                        open_breach_id BIGINT NULL,                   -- the open ops.breach row, NULL when none
                        evaluated_at DATETIME(3), config_version INT, row_version INT UNSIGNED)
ops.breach             (tenant_id, breach_id BIGINT,                  -- PK (tenant_id, breach_id)
                        rule_id, device_id,
                        opened_at DATETIME(3), counter_at_open BIGINT,
                        resolved_at DATETIME(3) NULL,
                        outcome VARCHAR(12) NULL,                     -- serviced | recovered | rule_changed
                        resolved_by_service_record_id BIGINT NULL,
                        alerts_sent INT DEFAULT 0,
                        next_alert_due_at DATETIME(3) NULL,           -- UTC; computed in the alert's local zone; NULL when closed
                        config_version INT,                           -- version in force when it opened
                        open_slot VARCHAR(41) GENERATED ALWAYS AS
                          (IF(resolved_at IS NULL, CONCAT(rule_id, ':', device_id), NULL)) STORED,
                        UNIQUE (tenant_id, open_slot))                -- one open breach per rule and device (§15)
                        -- index (tenant_id, next_alert_due_at) for the reminder job
ops.alert_dispatch     (tenant_id, breach_id, alert_no INT,           -- PK: one row per alert of a breach
                        alert_kind VARCHAR(8),                        -- initial | reminder
                        due_at, attempts TINYINT DEFAULT 0, sent_at NULL,
                        status VARCHAR(24),                           -- queued | sending | sent | failed | skipped | skipped_no_recipients
                        lease_until DATETIME(3) NULL,                 -- set while `sending`; an expired lease may be re-claimed
                        recipient_count SMALLINT, provider_message_ref VARCHAR(128) NULL)
ops.work_queue         (tenant_id, queue_id BIGINT PK,
                        job_kind VARCHAR(16),                         -- projection | rule_eval | rollup
                        device_id INT NULL,                           -- NULL = whole tenant
                        location_id INT NULL,                         -- rollup: each location the device was at; NULL = all
                        from_date DATE NULL, to_date DATE NULL,       -- rollup: affected business dates
                        reason VARCHAR(48), enqueued_at DATETIME(3), done_at DATETIME(3) NULL,
                        pending_key VARCHAR(64) GENERATED ALWAYS AS (IF(done_at IS NULL,
                          CONCAT(job_kind,':',IFNULL(device_id,0),':',IFNULL(location_id,0)), NULL)) STORED,
                        UNIQUE (tenant_id, pending_key))              -- one pending item per job and target (§15)
ops.config_applied     (tenant_id, service VARCHAR(16),              -- ci_ingest | ci_service
                        config_version INT, applied_at DATETIME(3),   -- last version picked up (its re-evaluation enqueued)
                        PRIMARY KEY (tenant_id, service))
ops.service_record     (service_record_id BIGINT PK, tenant_id, device_id,
                        maintenance_type VARCHAR(12),                 -- device_down | preventative | proactive
                        service_date DATE NOT NULL,
                        serviced_at DATETIME(3) NULL,                 -- NULL only on date-only seeded rows (flagged)
                        technician_name VARCHAR(100) NULL, notes VARCHAR(2000) NULL,   -- never seeded
                        source VARCHAR(12),                           -- ci | iq | csv_import
                        source_ref VARCHAR(128) NULL,
                        created_by_user_id INT NULL, created_at,
                        request_key CHAR(36) NULL,                    -- client idempotency key; UNIQUE (tenant_id, request_key)
                        valid_from, valid_to)                          -- versioned (history table)
ops.service_record_attachment (attachment_id BIGINT PK, service_record_id, tenant_id,
                        storage_key VARCHAR(400), file_name VARCHAR(255),
                        content_type VARCHAR(100) CHECK (content_type IN ('application/pdf','application/msword',
                          'application/vnd.openxmlformats-officedocument.wordprocessingml.document','image/jpeg','image/png')),
                        size_bytes BIGINT, sha256 BINARY(32), scan_status VARCHAR(10),
                        uploaded_by_user_id, uploaded_at)
```

- **Rule evaluation is server-side** (BAP-78). A breach is one device breaking one
  rule; it opens when the device first breaks the rule and closes once. A later breach of the same rule is a new
  `ops.breach` row, so it re-alerts (requirement), and the history gives time-to-resolve reporting. Breaches come
  from the maintenance rules only; device-down episodes are shown but do not alert in R1 (§16 Q38).
- **One evaluator, one writer.** The rule evaluator is a module shared by ci-ingest and ci-service and the only code
  that writes `device_rule_state`, `breach` and initial `alert_dispatch` rows. It runs for a device when anything that
  can change the device's state commits: a feed run's facts, a service record, an assignment or rule change, or a
  configuration change. Feed runs, service records and assignment changes enqueue `rule_eval` in `ops.work_queue` in
  their own transaction. A configuration activation happens in the Platform Config Service, so it cannot share a
  transaction with CI: each service records the last configuration version it has picked up per tenant
  (`ops.config_applied`; recorded in the same transaction that enqueues the tenant's re-evaluation, so a version is
  never marked picked up without its work queued), and on the change notification, on start-up and on every sweep it compares that with the
  active version and enqueues the tenant if they differ, so a missed notification is caught.
  **Lock order.** Every writer of a device's breach state (the evaluator and the reminder job) first locks the
  device's `ref.device` row (`SELECT … FOR UPDATE`; it always exists, unlike lazily created rule-state rows), then
  the rule-state and breach rows in `rule_id` order. Workers run at READ COMMITTED, so no gap locks are taken. So a
  service record and a feed run cannot race and cannot deadlock. The fact recompute for a (device, day) batch takes
  the same `ref.device` lock first, so two feeds or two replicas touching one device (a CO and an EJ run, or two
  Kafka partitions) cannot overwrite each other's recompute from a stale read. A sweep every 4 hours re-evaluates every assigned
  device, one device per transaction (requirement); it also catches a missed queue item. Evaluation also runs when
  the data behind a page changes, which is a superset of the requirement's "on page load" trigger: a page load never
  needs to evaluate, because the state it would compute is already current.
- **Opening.** In one transaction: insert `ops.breach`, set `open_breach_id`, insert `alert_dispatch` alert 1
  (`initial`; `queued`, or `skipped` when the rule's alert is disabled, so the history shows why no email went, or
  `skipped_no_recipients` when the rule has none), compute `next_alert_due_at` from the rule's cadence and local time
  in the alert's zone.
- **Closing.** The evaluator closes the open breach with the outcome taken from what changed: a newer matching service
  record → `serviced` (with `resolved_by_service_record_id`); the rule deactivated or deleted, its threshold raised past
  the counter (the counter has not fallen below its value at opening), or the device unassigned or retired →
  `rule_changed`; otherwise the condition cleared on its own (a proactive rule's errors aging out of the window) →
  `recovered`. When several hold, a rule that no longer applies to the device wins over a service record, and a
  service record over a raised threshold. A preventative breach cannot recover by itself,
  because notes moved since the last service only grows. Closing clears `open_breach_id` and `next_alert_due_at`; any
  unsent dispatch row for it is marked `skipped`. When a rule stops applying to a device (deactivated, deleted,
  unassigned, or the device retired), the evaluator also deletes the device's `device_rule_state` row for it; the
  breach rows keep the history, and the state is computed afresh if the rule applies again. Deleting a rule is
  logical when it has breach history (it is deactivated, so its breaches keep their rule) and physical only when it
  has none; the physical delete removes the rule's state rows, assignments, recipients and schedules in the same
  transaction, before the rule row, so no foreign key blocks it.
- **Reminders** run hourly in ci-service's scheduler, per tenant, and continue until the breach is resolved
  (requirement; there is no expiry). For each open breach with `next_alert_due_at <= now`, claimed with
  `SELECT … FOR UPDATE SKIP LOCKED` after the device lock above: insert the next `alert_dispatch` row with
  `alert_no` = MAX(`alert_no`) + 1 for the breach (so a rerun or a second instance hits the primary key and sends
  nothing twice, and an unsent earlier alert does not block it), set `next_alert_due_at` to the **first** occurrence
  of the rule's cadence and local time strictly after now, and commit. One reminder is sent after an outage, not one
  per missed period. Local times are computed in the service with java.time in the alert's IANA zone: a local time
  that falls in a daylight-saving gap moves to the first valid instant after it, and in an overlap the earlier
  instant is used.
- **Alert outbox.** `ops.alert_dispatch` is written in the same transaction as the state change and sent after
  commit by ci-service's sender, which a commit that wrote dispatch rows signals directly (from ci-ingest, an internal
  call), so the initial alert goes out at once (requirement); a poll every minute is the backstop. The sender claims
  a `queued` row (or a `sending` row whose lease has expired) by setting `sending` and `lease_until`, reads the rule's
  recipients, sends one email with `breach_id:alert_no` as the provider's idempotency key, and records `sent_at`,
  `provider_message_ref` and `sent`. Delivery is at least once; the idempotency key keeps a crash between sending and
  recording from producing a second email where the provider supports it. A failure is queued again after a
  backoff of 2^`attempts` minutes (at most 2^10) and marked `failed` after 5 attempts; the completion names the
  `attempts` its claim set, so a sender whose lease expired and was claimed again changes nothing. Only `queued` and
  lease-expired rows are claimed, so a `skipped` row is never sent; a rule with no
  recipients writes `skipped_no_recipients`. The email's content is Kinective-branded and names the device, branch and rule; reminders
  are marked as reminders (requirement). Sending email from ci-service directly is pending a Crossroads ruling
  (§16 Q35).
- **`ops.work_queue`** is CI's durable recompute queue: feed runs, configuration changes, assignment writes and
  service records enqueue projection refresh and rule evaluation for the affected devices in the same transaction as
  their change, so a restart never loses one. A worker lists pending items without marking them and runs each in one
  transaction: its device or location locks first, then the item's queue row (`FOR UPDATE`; an item another worker
  completed meanwhile is skipped, so two replicas never both run it), the work, and `done_at` set only if
  `enqueued_at` is not newer than the version it locked. A crash leaves the item pending, and an enqueue during the
  work waits for that transaction and then leaves a pending item behind, so it is never lost. A whole-tenant
  item (`device_id` NULL, after a configuration change) is worked only by the `jobs` unit, under the tenant's `sweep`
  job lease (§14.2), which it shares with the 4-hourly sweep: a replica that cannot claim the lease leaves the item
  for the holder, so one replica sweeps a tenant at a time, and the lease's epoch fences the done stamp. It is worked
  like the sweep, one device per transaction under that device's lock, and stamped done, under the same condition,
  after its last device, or at once when the tenant has no device; a crash restarts it, and re-evaluation is
  idempotent. A device move or a coin-link change enqueues a
  `rollup` item for the old and new locations over the affected dates; the rollup job merges overlapping items per
  location and rewrites those (location, day) rows (§7.3a). One pending item per job kind and target: an enqueue
  of a target that is already pending upserts it (`ON DUPLICATE KEY UPDATE`, widening a rollup's date range and setting `enqueued_at` to the later of
  now and the old value + 1 ms, so it strictly increases even within one millisecond), so a
  Tier B feed adds no rows for a device still waiting. Done rows are purged daily. Rollup items are serialized per
  location (the job locks the location's rollup rows first), so a concurrent Cash Optimizer and EJ run cannot
  overwrite each other's rollup from a stale read. It is internal to CI's data pipeline, not a platform task
  queue.
- **Writes by Users and Service Providers** go through ci-service's scoped writes: log a service record and add an
  attachment (§11).
- **A service record clears every flag of its type** (BAP-77): *preventative* resets the notes-moved baseline;
  *proactive* resets the error baseline (errors after `baseline_at` only); *device down* acknowledges the open
  downtime episode (§8). The device leaves the flagged list for everyone. Whether one record should clear every flag
  type is open (§16 Q24).
- **Pre-CI history**: a one-time onboarding import of iQ's `cfmservicerecord` (Cassandra, read through iQ's data connector; neither the Kafka stream nor the Mongo copies, so it needs the iQ team's agreement, §16 Q3) or a CSV import seeds
  `service_record` with `source` set, so "Last Service" and notes-moved baselines exist from day one. iQ holds a date
  only and two types: only preventative and proactive records are seeded (there is no device-down type), with
  `serviced_at` NULL (flagged as date-only; a baseline then starts at the end of `service_date`). Technician names are
  not seeded (iQ keys on them; P10).
- Technician name and notes are staff free text: excluded from exports and AI prompts by default (§13.1). Files live
  in the tenant's S3 bucket with malware scanning; SQL holds only metadata.

# 10.4 User-entered inputs

```sql
ops.vault_balance_entry (entry_id BIGINT PK, tenant_id, location_id, balance_amount DECIMAL(19,4),
                         entered_by_user_id, entered_at,
                         request_key CHAR(36) NULL,                    -- client idempotency key; UNIQUE (tenant_id, request_key)
                         source VARCHAR(12) DEFAULT 'user')            -- user | dc_gateway (R3)
```

The order dialog seeds the vault total from `default_vault_total` ($50,000), as the requirement states. Entries are
written through `ops_add_vault_entry` (§11) and stored for audit and as R3's landing place for DC Gateway balances; R1 does not pre-fill one user's entry for others.
