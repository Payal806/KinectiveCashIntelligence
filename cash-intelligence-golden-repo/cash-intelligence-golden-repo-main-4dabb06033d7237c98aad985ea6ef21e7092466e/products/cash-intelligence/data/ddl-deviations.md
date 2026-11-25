---
id: DAT-KIN-CI-120
title: Cash Intelligence architecture — DDL deviations register
artifact_type: architecture
status: approved
content_state: approved-design
owner_role: Cash Intelligence architecture owner
applies_to:
  - PRJ-KIN-BRAU-CI
source_of_truth: golden-repo
source_refs:
  - "cash-intelligence-code-repo db/ (deviation notes D-A*, D-B*, D-F* in the changesets)"
last_updated: 2026-11-22
review_due: TBC
---

# DDL deviations register

> **Approved (DEC-027, 2026-11-21).** Where the built DDL in the code repository departs from, or fills a gap in, the design
> sections it implements. Each row names the note in the code (search the code repository's `db/` for it), and is part
> of the design; a deviation that should no longer stand is changed in the code and removed here.

| Note | Design section | What the DDL does | Why |
|---|---|---|---|
| D-A3 | §14.1, §6 | Run-time catalogues (device models, error codes, model bin maps, service providers) are mastered in `ci_catalog`; seeded ones (denominations, device types, forecast models, activity map) by the versioned seed changeset written into every tenant | The design named "the global catalogue masters" without their shape (now stated in §14.1) |
| D-A4 | §13 | The audit archive is a database, `ci_audit_archive`, locally and in CI | Stand-in for the tenant's S3 bucket under Object Lock (§14.3) |
| D-A5 | §5.1 | `ingest_quarantine` in a tenant database has `tenant_id` NOT NULL, leading the key | A row reaches a tenant database only after routing; unrouted rows (`unknown_client`) stay in `ci_catalog.ingest_quarantine` |
| D-A6 | §11 | `acc_role_mapping` keys on a generated `tenant_scope` (`IFNULL(tenant_id, 0)`) | The design gives no key and `tenant_id` is nullable (§15 key helper) |
| D-A9 | §6 | `acceptor_capacity_source` is `VARCHAR(20)` | `cm_threshold_proxy` needs 18 (now in §6) |
| D-A10 | §6 | `ref_device_link` keys on (tenant, child device, effective from), with one active child | The design gives no key |
| D-A11 | §5.2 | `stg_link_start_inventory_line` holds the start inventory's lines | The design lists no line table; the source XML is shredded into it |
| D-A12 | §11 | `acc_user_capability` is versioned (`valid_from` and history) | So a revoked capability is never granted again (stated in §11) |
| D-B1 | §11, §14.1 | Scope views lead with `tenant_id` | So page joins can use `(tenant_id, id)` |
| D-B2 | §11 | An assigned region grants its branches and sites | §11 says "assigned branches/sites"; regions are assigned in practice |
| D-B9 | §13 | No audit purge procedure | A tenant's audit log goes with its database at offboarding, after archiving |
| D-B11 | §13 | The audit trigger allocates `audit_id` and stamps `occurred_at` | No caller can choose either |
| D-B21 | §7.3a | The coin roll-up (`cash.v_device_rollup`) is three views: current inventory, the daily map, and the design's shape. Current inventory takes each coin device's open link (`effective_to` NULL, one per child), not the link effective at the database clock | Each stays small or mergeable by the optimiser; no view reads the clock (DEC-028) |
| D-B22 | §10.2 | A branch's or site's vault minimum is its own assignment, else its region's, else the tenant default | §10.2 (`cfg.v_effective_rule`) resolves the target's own active assignment, else the tenant default; the DDL adds the region's assignment between the two, so a region's rule applies to its branches and sites (as D-B2 does for scope) |
| D-F1 | §14.2 | Every unit that commits a change other replicas must see bumps `change_signal`; api and jobs read run headers and checkpoints | The §14.2 grant list omitted these |

Designed but not yet built (DEC-026; built with staging, code repository `TEMPORARY-MEASURES.md`): the staging
version guard (`source_seq`, material hash), staging retention, the source-count and content-digest reconciliation
checks (`check_kind = 'source'`, `source_object`), and backfill requests and runs (§5.2, §5.3, §5.5).
