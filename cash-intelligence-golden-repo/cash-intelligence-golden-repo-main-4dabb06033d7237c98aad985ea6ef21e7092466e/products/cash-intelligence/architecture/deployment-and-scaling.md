---
id: ARC-KIN-CI-106
title: Cash Intelligence architecture — 14.2 Deployment and scaling
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
last_updated: 2026-11-21
review_due: TBC
---

# 14.2 Deployment and scaling

> **Approved architecture: the R1 design baseline (DEC-006).** Approved by the Cash Intelligence architecture owner on 2026-11-05, after the 2026-11-03 design review. Generated specs follow it; a spec that needs something different is a change here first (GOV-CI-004 in `projects/branch-automation-cash-intelligence/05-delivery/delivery-model-and-gates.md`). Open questions (§16) stay open until answered. Section numbers (§n), open questions (Qn) and decisions (A–I) use the architecture's own numbering; `products/cash-intelligence/architecture/architecture-overview.md` maps each section to its file, and the open questions are in `products/cash-intelligence/architecture/open-questions.md`. Citations: `BAP-nn` = `SRC-JIRA-BAP-001`, `TDD:n` = line n of `SRC-XR-TDD-001`, `research/nn` = `SRC-CI-RES-001` (see the source index).

Every CI unit in EKS is stateless: it keeps no data between requests, any replica can serve any tenant, and a replica
that dies loses nothing. Shared state lives in the tenant databases and `ci_catalog`, and coordination uses only
MySQL (row locks, `SKIP LOCKED` queues and leases), so CI adds no infrastructure to the platform. Each unit has its
own credential per tenant (TDD:399), with grants on that tenant's database only: granted to the credential or to a
role per tenant and unit, never through a role shared by several tenant databases, which would let one tenant's
credential read another's. The DDL creates those roles per tenant database (`<database>__api`, `__jobs`, `__ingest`,
`__maint`; a role name has at most 32 characters, so a tenant database name has at most 24). No tenant credential has any grant on `ci_catalog`. Each unit has one more credential, for
`ci_catalog` only, shared by all tenants because the catalogue holds no tenant data: `api` reads the tenant
directory, routing keys and provider grants, reads and bumps `change_signal`, and inserts `security_event`; `jobs`
reads `tenant`, reads and bumps `change_signal`, and claims and renews `job_lease`; ci-ingest uses the ingest control
store and the id allocators and bumps `change_signal`; the freshness alarm reads run headers and checkpoints (an
`ingest` grant); the maintenance unit runs the offboarding steps on the catalogue (status, legal hold, removal). Offboarding revokes the tenant's credentials, not these
shared ones, which can no longer reach the dropped database.

| Unit | Runs | Scales on | How replicas share the work |
|---|---|---|---|
| **ci-ingest** | EKS Deployment, 2+ replicas | Kafka consumer lag; overdue feed leases | Kafka feeds: the consumer group assigns partitions. Mongo-poll and CO window feeds: one `feed_lease` per feed. CO batches arrive through the Ingress Gateway, are load-balanced across replicas, and are idempotent by source key |
| **ci-service `api`** | EKS Deployment, 2+ replicas | Request rate, CPU | Any replica serves any request; scope is a bind parameter (§11). Page reads may go to a read replica |
| **ci-service `jobs`** | EKS Deployment, 2+ replicas, no inbound route | Queue depth (`work_queue`, `alert_dispatch`); overdue job leases | Queue rows are claimed with `SKIP LOCKED`; each scheduled job runs per tenant under a `job_lease`, so a nightly job spreads across replicas and never runs twice |
| **maintenance** | Kubernetes CronJob | — (one run at a time) | Partition create and drop under the maintenance account (§13); CI's own Flyway job for `ci_catalog` (§14.1) |
| **CO connector** | At the bank, one per CO installation | Does not need to | One extract at a time per installation; a missed hour is covered by the next window (§4) |
| **CI micro-frontend** | Web-component bundle served by ci-service `api` under `/mfe/ci/` (the template's server library sets the cache headers), mounted by the KBOP shell. | With `api` | — |
| **Tenant databases** | Managed MySQL clusters | Add clusters | A cap on tenants per cluster, set by load test (§14.1 scale limits); one tenant is one primary |

**Leases.** `ingest.feed_lease` and `ci_catalog.job_lease` are claimed with
`UPDATE … SET owner = ?, lease_until = now + 60 s, epoch = epoch + 1 WHERE … AND (lease_until IS NULL OR lease_until < now)`,
and renewed every 20 s while the work runs with `UPDATE … SET lease_until = now + 60 s WHERE … AND owner = ? AND epoch = ?`;
a renewal that changes no row means the lease was lost, and the replica stops that work at once. A replica that stops
renewing loses the lease within a minute and another replica takes over. The **epoch fences a stale owner**: a feed run advances its checkpoint, and a job records success,
only in a transaction that confirms `epoch` still equals the one it claimed, so a replica that paused (GC, network)
and resumes after losing its lease cannot move a checkpoint backwards. The fenced advance is one transaction: lock
the lease row (`SELECT … FOR SHARE`), confirm `owner`, `epoch` and `lease_until > now`, then write the checkpoint (or
the job's success). A takeover's claim waits for that lock, so an expired but not yet re-claimed lease cannot
interleave with it. Kafka feeds have no lease: a consumer writes a partition's checkpoint only while the partition
is assigned to it, after the batch's tenant writes have committed (§5.2 "At least once"), and stops in the revoke
callback before the rebalance completes;
the advance is monotonic, so a late write from a revoked owner cannot move a checkpoint backwards. For Kafka,
`last_version` holds the offset as a decimal string, compared numerically: `UPDATE … SET last_version = ? WHERE … AND
(last_version IS NULL OR CAST(last_version AS UNSIGNED) < ?)`; a new partition's row starts with `last_version` NULL
(nothing read yet), so the first write always succeeds. The checkpoint tests cover 9 → 10, 10 → 9 and the first write. The fence is enforced in the services' code, not by grants (ci-ingest may update
`ingest.feed_checkpoint` directly), so the fenced advance lives in one repository method with its own test. Everything a feed or job writes is idempotent
(P6), so the worst case of a lost lease is repeated work, never wrong data. A replica releases its leases on
shutdown with the same owner-and-epoch condition (`SET owner = NULL, lease_until = NULL`), so a rolling deploy hands work over at once. `job_lease.next_due_at` is computed like reminders: the first
occurrence after now, so a job missed during an outage runs once on recovery, not once per missed slot. Leases live
in `ci_catalog`, so scheduled jobs and lease-based feeds pause while it is unreachable and resume on recovery; Kafka
feeds and queue workers carry on.

**Per-device serialization.** Fact recompute, the rule evaluator and the reminder job all lock the device's
`ref.device` row first (§10.3), and rollups lock the location's rollup rows first (§10.3). These are database locks,
so they hold across replicas and across units.

**Change signal.** A commit that other replicas must react to bumps the tenant's `ci_catalog.change_signal.seq` after
it commits: a feed run, a service record, a configuration change, a new `alert_dispatch` row. Every `api` and `jobs`
replica polls the table every 2 seconds (one indexed read of a few hundred rows). When a tenant's `seq` has moved:
- `api` replicas with open live-update sessions for that tenant read the devices touched since the last `seq` (from
  `ingest.run` and `ops.work_queue`) and notify the sessions whose scope contains one (§4, real-time behaviour);
- `jobs` replicas wake the alert sender for that tenant;
- row 0 invalidates the `ci_catalog` cache.

The signal carries no data and is only a hint: the page re-reads through its scoped queries, the alert sender's
one-minute poll and the cache TTL remain the backstops, so a missed bump delays a change by seconds and never loses
it. The added latency is at most the poll interval. Live updates use server-sent events; a page reconnecting to
another replica re-reads its data. If Crossroads offers a platform pub/sub, the signal moves there without a schema
change (§16 Q44).

**Connections.** Connections grow with replicas × tenants × pool size. Pools have no minimum and evict idle tenants
(§14.1). Each iQ client is its own feed, so feed leases also spread tenants across ci-ingest replicas; the Kafka feed
does not, because one topic carries every client. The hard limit is the connection layer Crossroads chooses (§16
Q36).

**Limits horizontal scaling does not remove.** One tenant is bounded by one MySQL primary (comfortable at the
measured volumes, §14); objects per cluster cap tenants per cluster (§14.1); the CO feed is bounded by CO, not CI
(§4). The load test before go-live sets the tenants-per-cluster cap and the replica counts.
