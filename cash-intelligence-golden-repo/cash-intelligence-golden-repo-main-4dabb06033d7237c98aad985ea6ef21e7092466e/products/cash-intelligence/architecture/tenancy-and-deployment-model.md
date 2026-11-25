---
id: ARC-KIN-CI-105
title: Cash Intelligence architecture — 14.1 Tenancy and deployment model
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
last_updated: 2026-11-22
review_due: TBC
---

# 14.1 Tenancy and deployment model

> **Approved architecture: the R1 design baseline (DEC-006).** Approved by the Cash Intelligence architecture owner on 2026-11-05, after the 2026-11-03 design review. Generated specs follow it; a spec that needs something different is a change here first (GOV-CI-004 in `projects/branch-automation-cash-intelligence/05-delivery/delivery-model-and-gates.md`). Open questions (§16) stay open until answered. Section numbers (§n), open questions (Qn) and decisions (A–I) use the architecture's own numbering; `products/cash-intelligence/architecture/architecture-overview.md` maps each section to its file, and the open questions are in `products/cash-intelligence/architecture/open-questions.md`. Citations: `BAP-nn` = `SRC-JIRA-BAP-001`, `TDD:n` = line n of `SRC-XR-TDD-001`, `research/nn` = `SRC-CI-RES-001` (see the source index).
>
> **Legal review required.** This file covers tenant isolation, per-tenant restore, legal hold and offboarding. These touch compliance: Kinective's Legal team must review them before they are relied upon or selected as Astra context.

**One database and one bucket per tenant** (decision H; Crossroads D-22, ADR-026). Each tenant has its own logical
MySQL database on shared database infrastructure, its own database credential for each CI service, and its own S3
bucket with its own KMS key. Nothing is separated by `tenant_id` filter or bucket prefix alone. There is no pooled
placement. A tenant whose contract requires it can be placed on dedicated database infrastructure; the database and
DDL are the same.

| Operation | How |
|---|---|
| Provision | The Crossroads provisioning job creates the database, credentials, bucket and key; the Flyway fleet runner brings the schema to head; CI's onboarding step (maintenance job) then creates the month partitions and seeds the tenant (below) |
| Retention | Month partitions dropped per tenant database (§14); bucket lifecycle from the tenant's retention period |
| Legal hold | `ci_catalog.tenant.legal_hold` is the master (it outlives the tenant database); `ref.tenant.legal_hold` is its copy in the tenant database, read by the jobs in their tenant transaction. Either suspends every partition drop, purge and offboarding step; the bucket gets an Object Lock legal hold |
| Restore | Managed MySQL restores point-in-time per instance or cluster, not per database. One tenant is restored by restoring the cluster to a side instance, dumping that tenant's database and re-importing it; the runbook and its timing are tested before go-live |
| Export | A dump or a CSV export of one database |
| Offboard | Below |

**Rules that keep tenants self-describing.** `tenant_id` is allocated centrally and never reused, every primary key
leads with it, and every foreign key and join uses `(tenant_id, <id>)`. In a single-tenant database this is not what
isolates tenants, but it keeps every row, export and audit record attributable on its own, makes a wrong-database
routing bug return nothing instead of another tenant's rows, and lets a database be copied to other infrastructure
without key collisions. The global catalogues (`ref.denomination`, `ref.device_type`, `ref.device_model` and its
`_xref`, `ref.error_code`, `fcst.model`, `ingest.activity_map`, `acc.service_provider`) are mastered in the
control-plane catalogue and copied into every tenant database with the same ids. The catalogues that grow at run time
(device models, error codes, model bin maps, service providers) are mastered in `ci_catalog`; the seeded ones
(denominations, device types, forecast models, the activity map) are mastered by the versioned seed changeset, which
writes identical rows into every tenant database. `ci_catalog` allocates
`device_model_id`, `error_code_id`, `source_instance_id` and `provider_id` by natural key through an idempotent
allocation call, and a reference sync that meets a new model or error code takes its id from there (`research/80` §7
D3). Seeded catalogue rows carry explicit values for every column, so they compare equal across databases.

**Control-plane catalogue** (`ci_catalog`, one small database outside the tenant databases, read by ci-ingest and
ci-service; free of personal data):

```sql
ci_catalog.tenant          (tenant_id INT PK,
                            institution_id VARCHAR(64) NOT NULL UNIQUE,  -- Crossroads institutionId
                            kinective_id CHAR(18) NOT NULL UNIQUE,       -- Salesforce Account ID, 18-char form (ASCII, binary collation)
                            db_cluster VARCHAR(128), db_name VARCHAR(64), bucket VARCHAR(63),
                            status VARCHAR(12),                          -- onboarding | active | frozen | offboarding | removed
                            legal_hold BOOLEAN DEFAULT 0,                -- master copy (§14.1 operations)
                            kbop_fi_status VARCHAR(16) NULL,             -- from KBOP fi-service
                            kbop_checked_at DATETIME(3) NULL,
                            updated_at DATETIME(3))
ci_catalog.tenant_key      (tenant_id, key_type, source_instance_id INT NULL,
                            external_key VARCHAR(128) COLLATE utf8mb4_bin,
                            instance_scope GENERATED ALWAYS AS (IFNULL(source_instance_id, 0)) STORED,
                            UNIQUE (key_type, instance_scope, external_key))   -- routing copy of ref.tenant_external_key
ci_catalog.service_provider_tenant (provider_id, tenant_id, PRIMARY KEY (provider_id, tenant_id))
ci_catalog.job_lease       (job_name VARCHAR(48), tenant_id INT,         -- one row per scheduled job per tenant (§14.2)
                            owner VARCHAR(64) NULL, lease_until DATETIME(3) NULL,
                            epoch BIGINT NOT NULL DEFAULT 0,
                            next_due_at DATETIME(3), last_succeeded_at DATETIME(3) NULL,
                            PRIMARY KEY (job_name, tenant_id))
ci_catalog.change_signal   (tenant_id INT PK,                          -- 0 = catalogue change, read by every replica
                            seq BIGINT NOT NULL,                       -- bumped by every commit other replicas must see
                            changed_at DATETIME(3))
ci_catalog.security_event (event_id BIGINT AUTO_INCREMENT PK,       -- validation failures before a tenant is resolved (§11)
                            occurred_at DATETIME(3), service VARCHAR(16), reason VARCHAR(32),
                            kbop_user_id VARCHAR(128) NULL, entity_id VARCHAR(128) NULL,   -- as wide as acc.app_user;
                            correlation_id VARCHAR(64))                                     -- a longer claim is denied (§11 step 2) and logged
                                                                       -- cut to 128 with reason claim_too_long
                                                                       -- INSERT only; retained like audit.audit_log
-- plus: the global catalogue masters and id allocators; ingest.source_instance, ingest.feed, ingest.feed_checkpoint, ingest.feed_lease,
-- ingest.ej_txn_type_map, ingest.run headers, and quarantine rows whose tenant is unknown (`unknown_client`)
```

Crossroads also keeps a control-plane database (institution directory, licences, schema versions). `ci_catalog`
holds only what is CI-specific; whether some of it can live in the platform's directory is open (§16 Q36). The three
identifiers of a tenant — Crossroads `institutionId`, KBOP `entity_id` (Kinective ID) and its source keys (iQ client,
CO tenants) — are held together in one place, `ci_catalog.tenant` and `tenant_key`. `kbop_fi_status` is refreshed
from KBOP so that a tenant KBOP disables (no active subscription) can be moved to `frozen` or `offboarding`
deliberately; the status mapping is a Product decision. One Kinective ID can be a holding company, so it is not
assumed to be one Link client or one CO tenant.

**Connections.** ci-ingest and ci-service reach every tenant database. They hold a bounded, evicting pool per tenant
(or use the proxy Crossroads chooses, §16 Q36) behind one connection interface, and nothing depends on connection
state (§11), so either works.

**Scale limits.** Each tenant database has about 115 tables, 60 triggers and monthly partitions. At hundreds of
tenants on one cluster that is hundreds of thousands of tablespace files, which strains the table and definition
caches, open-file limits and crash recovery. So: a cap on tenants per cluster, set by load test; partitions created
about three months ahead by the maintenance job rather than years ahead; small tables unpartitioned; pools with no
minimum size that evict idle tenants. ci-ingest parks a tenant whose database is unreachable so one tenant cannot stall the others: the feed's checkpoint
advances, and a `feed_checkpoint` row with `tenant_scope` = that tenant keeps the position where its rows stopped.
On recovery ci-ingest re-reads from that position for that tenant only (rows of other tenants are already applied,
P6), then deletes the row. The freshness alarm watches the oldest position of each feed, parked rows included, so
a tenant parked close to iQ's retention alarms before its rows are lost (§16 Q3).

**`ci_catalog` availability.** Every login and every routed row reads it, so both services cache it with a short TTL
and keep serving known tenants from the cache if it is briefly unavailable; a change (new key, frozen tenant) bumps
`change_signal` row 0, which every replica polls (§14.2), and the TTL is the backstop. How long the cache may serve
while the catalogue is down, and whether status and routing fail closed instead, is open (§16 Q45). It is migrated by CI's own Flyway job, because the Crossroads fleet runner migrates
institution databases only (TDD:402).

**Engine.** Crossroads lists the database engine as open (TDD:392, 405; R-36) and its fleet runner assumes one
engine (ADR-026). CI's MySQL decision therefore needs the platform's provisioning job, credential broker, Flyway
runner and backups to support MySQL for ABS databases (§16 Q21).

**Schema changes** ship as Flyway SQL migrations applied to every tenant database by the Crossroads fleet runner:
expand-then-contract, code runs against schema N and N−1, DDL and data changes are never in one migration, and each
migration is validated against a fresh database and a copy of the oldest live version. Seed data for the global
catalogues is a separate, repeatable migration.

**Onboarding:** allocate `tenant_id` → verify the Kinective ID exists and is active in KBOP and record the
Crossroads `institutionId` → provisioning job and Flyway (above) → the maintenance job creates the month partitions
(tables are created with their catch-all partition only, §14) → copy the global catalogues → seed `ref.tenant`, the
default rules and the tenant's configuration (defaults, via the Config Client) → map source instances and external
keys, in `ci_catalog.tenant_key` and `ref.tenant_external_key` → run the routing test (§4) → backfill (§7.2) → set
`active`.

**Offboarding:**
1. Set `offboarding`. Feeds stop, and ci-service refuses the tenant, so no user can read its data. A tenant under
   legal hold (`ci_catalog.tenant.legal_hold`) cannot proceed past this step. A hold can also be set during
   offboarding, so every destructive step (4, 5 and 6) re-reads `ci_catalog.tenant.legal_hold`, the master, never a
   copy, immediately before it starts, and stops with the tenant left in `offboarding` if it is set.
2. Export the tenant's data if the contract requires it, then verify the export against row counts and
   reconciliation totals.
3. Export the tenant's `audit.audit_log` rows and its chain head (last hash) to the audit archive in the tenant's
   bucket (Object Lock), verify the chain there, and retain it per Legal's retention rule.
4. Revoke the tenant's credentials and terminate every open session that uses them (dropping a MySQL user does not
   close its sessions), and confirm none remains; then drop the database and every grant that names it (MySQL keeps
   object-level grants after `DROP DATABASE`).
5. Delete the bucket's other objects (service-record files, exports) once the archive's retention allows.
6. Delete the tenant's values in the Platform Config Service, and its `ci_catalog` keys, source instances, feeds,
   checkpoints and unrouted quarantine rows; keep only the `ci_catalog.tenant` row (status, hold, ids) so `tenant_id`
   is never reused.
7. Set `removed`.

Cluster snapshots and binary logs still hold the dropped database until they age out of the backup retention period;
that period is stated in the contract as the time to full erasure.

A re-onboarded tenant starts a new chain whose genesis row records the archived head hash as its `prev_hash`, so the
archived and new chains verify as one.

Offboarding, the audit archive and its retention need Kinective Legal review before go-live.

**Disaster recovery and data residency.** The primary region is AWS us-west-2 (Oregon); backups and the standby
are in us-east-2 (Ohio), so tenant data stays in the US. The managed MySQL service provides point-in-time recovery
and cross-region backup copies; `ci_catalog` and the per-tenant buckets (the Object Lock audit archive included)
replicate to the same secondary region. CI's architecture defines the standby and DevOps deploys and runs it; the
RPO and RTO targets are set with DevOps (§16 Q58). If the hosting choice (§16 Q21) puts the databases outside AWS,
every control above (point-in-time recovery, cross-region backup copies, the standby, `ci_catalog` and bucket
replication, residency and the secondary region) is restated for that provider before go-live.

**Purge and reload (break-glass).** When a source sent wrong data that passed every check (for example device ids
swapped at the source, or a test server licensed under a production client name), CI re-reads; it never edits a
fact by hand. Under an approved change: the operator obtains the maintenance role through the break-glass process
(§13), time-limited and tagged with the change reference, which is written to every audit row of the purge; the
access is revoked when the change closes.
1. Park the affected source instance for the tenant (parking, above), so nothing new lands.
2. Record the purge window (tenant, source instance, devices or all, first business date) and check that every
   source can replay all of it: iQ from its history; CO and EJ only within their re-read windows and history
   tables. If any source cannot, stop before deleting anything.
3. Delete that instance's staged rows and the derived facts, snapshots and rollups in the window, one (device, day)
   batch per transaction. Breach and alert history, service records and audit rows are kept.
4. Fix the cause at the source or in CI's mapping (device xref, client key). A mapping change bumps
   `ci_catalog.change_signal` after it commits, and the replay waits until ci-ingest and ci-service have refreshed
   their routing caches (§14.2). Reset every affected checkpoint (each
   feed, source table and partition of the instance) to the window start; a Kafka feed re-reads the window from
   iQ's Mongo copy, since the topic may no longer hold it; CO and EJ run a one-off window re-read covering the purge
   window. Then unpark; the normal recompute and reconciliation rebuild the facts.
5. Write an audit row for each step with the operator, the change reference and row counts.

It is not a routine path: corrections the source makes reach CI through the normal recompute (§5.2), and a figure
the source will not correct is a product question (§16 Q54).

**Client identity changes.** iQ names a client by its Link client name, and the platform by Kinective ID; both can
change under CI.
- A renamed client: map the new name to the same source instance in `ci_catalog.tenant_key` and the tenant's
  `ref.tenant_external_key` (the onboarding step writes both) before the next run.
  Rows that arrive first are quarantined as `unknown_client` and replayed after the mapping.
- A test or second server sending under a production client name is a new source instance once found; data it has
  already sent is removed by purge and reload.
- A merger of two FIs: the surviving tenant keeps its database, and the absorbed tenant's source instances are
  remapped to it from an agreed date. The absorbed history is either reloaded into the survivor by purge and
  reload, or stays in its own database; the second option lets the survivor's users read another tenant's data, an
  exception to tenant isolation, so it needs its own design and approval first (routing, scope, audit and export
  records, legal hold and retention). Which one is a product decision (§16 Q55).
- When Link adopts the Kinective ID as its client key, `ci_catalog.tenant_key` and each tenant's
  `ref.tenant_external_key` gain that key type together, through the onboarding step; no schema change.
