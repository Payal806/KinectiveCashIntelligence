---
id: DATA-KIN-CI-001
title: Cash Intelligence architecture — 3. Schema map and data flow
artifact_type: data-architecture
status: approved
content_state: approved-design
owner_role: Cash Intelligence architecture owner
applies_to:
  - PRJ-KIN-BRAU-CI
source_of_truth: golden-repo
source_refs:
  - "SRC-CONF-CIR-001"
last_updated: 2026-11-21
review_due: TBC
---

# 3. Schema map and data flow

> **Approved architecture: the R1 design baseline (DEC-006).** Approved by the Cash Intelligence architecture owner on 2026-11-05, after the 2026-11-03 design review. Generated specs follow it; a spec that needs something different is a change here first (GOV-CI-004 in `projects/branch-automation-cash-intelligence/05-delivery/delivery-model-and-gates.md`). Open questions (§16) stay open until answered. Section numbers (§n), open questions (Qn) and decisions (A–I) use the architecture's own numbering; `products/cash-intelligence/architecture/architecture-overview.md` maps each section to its file, and the open questions are in `products/cash-intelligence/architecture/open-questions.md`. Citations: `BAP-nn` = `SRC-JIRA-BAP-001`, `TDD:n` = line n of `SRC-XR-TDD-001`, `research/nn` = `SRC-CI-RES-001` (see the source index).

## 3.1 Schema map

The map is split by area so each diagram stays readable. Table names are the MySQL names (area prefix + table);
only key columns are shown, and every key also carries `tenant_id` as described in §15. Full column lists are in the
sections named under each diagram.

**Areas and how they depend on each other**

```mermaid
%%{init: {"flowchart": {"useMaxWidth": false}}}%%
flowchart LR
  ING["ingest + stg<br/>feeds, staging, reconciliation"] --> REF["ref<br/>tenant, location, device"]
  ING --> CASH["cash<br/>inventory, snapshots, daily facts, orders"]
  ING --> HEALTH["health<br/>downtime, errors"]
  ING --> FCST["fcst<br/>forecast runs"]
  REF --> CASH
  REF --> HEALTH
  REF --> FCST
  PCS["Platform Config Service<br/>settings, rule parameters"] -.-> CFG
  CFG["cfg<br/>rules, assignments, recipients, schedules"] --> OPS["ops<br/>breaches, alerts, service records"]
  CASH --> OPS
  HEALTH --> OPS
  FCST --> CASH
  ACC["acc<br/>users, roles, scope"] -.-> CASH
  ACC -.-> HEALTH
  ACC -.-> OPS
  CAT["ci_catalog<br/>global ids, tenant identity"] -.-> REF
```

Dotted lines are access control (acc scopes what each request can read), configuration read from the platform, and
the control-plane catalogue. Every area lives in each tenant's own database except `ci_catalog`, which also holds the feed control tables (`ingest.source_instance`, `feed`, `feed_checkpoint`, `feed_lease`, run headers and unrouted quarantine; §14.1) and the job leases and change signal (§14.2). `ci_catalog` holds the masters of `ingest.source_instance` and `ingest.ej_txn_type_map`; each tenant database keeps a read-only copy of only the source instances that route to that tenant and their EJ type maps, written from the catalogue by ci-ingest on every change signal (only the ingest role can write the copy), for its keys and joins. A source instance with a routing key in the catalogue (an iQ client, a CO `TenantId`) is copied by that key; an EJ source instance has none, so onboarding adds it to its tenant's copy and the refresh keeps it current.

**Reference data** (§6)

```mermaid
%%{init: {"er": {"useMaxWidth": false}}}%%
erDiagram
    ref_tenant ||--o{ ref_tenant_external_key : "known as"
    ref_tenant ||--o{ ref_location : has
    ref_location ||--o{ ref_location : "region of"
    ref_location ||--o{ ref_location_source_xref : "matched to"
    ref_location ||--o{ ref_location_hours : opens
    ref_location ||--o{ ref_device : hosts
    ref_device_model ||--o{ ref_device : "model of"
    ref_device_model ||--o{ ref_error_code : "error catalog"
    ref_device ||--o{ ref_device_source_xref : "source ids"
    ref_device ||--o{ ref_device_bin_profile : bins
    ref_device ||--o{ ref_device_link : "coin to TCR"
    ref_tenant { int tenant_id PK }
    ref_location { int location_id PK
                   int parent_location_id FK
                   varchar location_type }
    ref_device { int device_id PK
                 int location_id FK
                 int device_model_id FK
                 varchar device_type_code }
    ref_device_bin_profile { int device_id PK
                             tinyint bin_no PK
                             date valid_from PK
                             smallint denomination_id }
    ref_device_link { int parent_device_id FK
                      int child_device_id FK
                      datetime effective_from }
```

**Cash facts** (§7, §9.4)

```mermaid
%%{init: {"er": {"useMaxWidth": false}}}%%
erDiagram
    ref_device ||--|| cash_device_inventory_current : "current state"
    cash_device_inventory_current ||--o{ cash_device_inventory_current_line : lines
    ref_device ||--o{ cash_inventory_snapshot : "EOD and SOD"
    cash_inventory_snapshot ||--o{ cash_inventory_snapshot_line : lines
    ref_device ||--o{ cash_device_daily_activity : "per day"
    cash_device_daily_activity ||--o{ cash_device_daily_activity_denom : "per denomination"
    ref_location ||--o{ cash_location_daily_activity : "branch rollup"
    cash_location_daily_activity ||--o{ cash_location_daily_denom : "per denomination"
    ref_device ||--o{ cash_cash_order : ordered
    cash_cash_order ||--o{ cash_cash_order_line : lines
    cash_cash_order ||--o{ cash_replenishment : "fulfilled by"
    cash_replenishment ||--o{ cash_replenishment_line : lines
    ref_device ||--|| cash_device_projection : projection
    cash_inventory_snapshot { int device_id PK
                              date business_date PK
                              varchar snapshot_kind PK }
    cash_device_daily_activity { int device_id PK
                                 date business_date PK }
    cash_location_daily_activity { int location_id PK
                                   date business_date PK
                                   varchar device_family PK }
    cash_cash_order { bigint cash_order_id PK
                      int device_id FK }
    cash_replenishment { bigint replenishment_id PK
                         int device_id FK
                         bigint cash_order_id FK }
```

**Device health and forecasting** (§8, §9)

```mermaid
%%{init: {"er": {"useMaxWidth": false}}}%%
erDiagram
    ref_device ||--o{ health_downtime_episode : down
    ref_device ||--o{ health_device_error_event : errors
    ref_device ||--|| health_device_status_current : status
    ref_error_code ||--o{ health_device_error_event : "coded as"
    fcst_model ||--o{ fcst_model_selection : "selected for"
    fcst_model ||--o{ fcst_run : produces
    fcst_run ||--o{ fcst_device_forecast : "device days"
    fcst_run ||--o{ fcst_device_forecast_denom : "denomination split"
    fcst_run ||--o{ fcst_device_forecast_weekly : "weekly archive"
    ref_device ||--o{ fcst_device_forecast_denom_current : "current split"
    health_downtime_episode { bigint episode_id PK
                              int device_id FK
                              datetime ended_at "NULL = down now" }
    fcst_run { bigint run_id PK
               int model_id FK }
    fcst_device_forecast { date run_month PK
                           bigint run_id PK
                           int device_id PK
                           date forecast_date PK }
```

**Configuration and operations** (§10)

```mermaid
%%{init: {"er": {"useMaxWidth": false}}}%%
erDiagram
    cfg_rule ||--o{ cfg_rule_assignment : "assigned to"
    cfg_rule ||--o| cfg_rule_replenish_schedule : details
    cfg_rule ||--o{ cfg_rule_alert_recipient : recipients
    cfg_rule ||--o{ cfg_schedule_calendar : "occurs on"
    cfg_rule ||--o{ ops_device_rule_state : "evaluated as"
    ops_device_rule_state ||--o| ops_breach : "open breach"
    ops_breach ||--o{ ops_alert_dispatch : alerts
    ref_device ||--o{ ops_service_record : serviced
    ops_service_record ||--o{ ops_service_record_attachment : files
    cfg_rule { int rule_id PK
               varchar rule_type
               boolean is_active }
    ops_breach { bigint breach_id PK
                 int rule_id FK
                 int device_id FK
                 varchar outcome }
    cfg_rule_assignment { int rule_id FK
                          varchar target_kind
                          int target_id }
```

**Access** (§11)

```mermaid
%%{init: {"er": {"useMaxWidth": false}}}%%
erDiagram
    acc_app_user ||--o| acc_user_role : "CI role"
    acc_app_user ||--o{ acc_user_location_access : "branch scope"
    acc_service_provider ||--o{ acc_service_provider_model : "serves model"
    acc_service_provider ||--o{ acc_service_provider_user : staff
    acc_app_user { int user_id PK
                   varchar host_subject }
    acc_user_location_access { int user_id PK
                               int location_id PK }
```

Page queries join the views `acc.v_user_device_scope`, `acc.v_user_location_scope` and `acc.v_user_rollup_scope`,
derived from these tables, on the caller's `user_id` (§11).

**Ingestion and reconciliation** (§5)

```mermaid
%%{init: {"er": {"useMaxWidth": false}}}%%
erDiagram
    ingest_source_instance ||--o{ ingest_feed : "read by"
    ingest_feed ||--o{ ingest_feed_checkpoint : checkpoints
    ingest_feed ||--o{ ingest_run : executes
    ingest_run ||--o{ ingest_quarantine : quarantines
    ingest_run ||--o{ ingest_reconciliation : "checked by"
    ingest_source_instance ||--o{ ingest_ej_txn_type_map : "EJ code map"
    ingest_feed { int feed_id PK
                  int source_instance_id FK
                  varchar extract_mode }
    ingest_run { bigint run_id PK
                 int feed_id FK }
    ingest_reconciliation { bigint run_id PK
                            date business_date PK
                            varchar measure PK
                            int device_id PK }
```

| Schema | Area | Section |
|---|---|---|
| `ref` | Tenant, location hierarchy, device registry, bin profiles, denominations, error catalog | §6 |
| `ingest`, `stg` | Source instances, feeds, checkpoints, runs, reconciliation, quarantine; transient per-source staging | §5 |
| `cash` | Current inventory, snapshots, daily activity, orders, replenishments, projections | §7, §9.4 |
| `health` | Downtime, errors, device status | §8 |
| `fcst` | Forecast models, runs, device forecasts, weekly archive, current denomination split | §9 |
| `cfg` | Rule identity, assignments, alert recipients, replenish schedules, schedule calendar (settings and rule parameters are in the Platform Config Service) | §10 |
| `ops` | Service records, rule state, breaches, alert outbox, user-entered balances, recompute queue | §10 |
| `acc` | User cache, CI roles, bootstrap role rule, location access, service providers, scope views | §11 |
| `ext` | External series (R2) | §12 |
| `audit` | Audit log | §13 |
