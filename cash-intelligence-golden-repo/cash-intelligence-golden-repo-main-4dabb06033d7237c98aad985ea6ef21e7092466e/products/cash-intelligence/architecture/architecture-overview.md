---
id: ARC-KIN-CI-100
title: Cash Intelligence architecture — overview and decisions
artifact_type: architecture-overview
status: approved
content_state: approved-design
owner_role: Cash Intelligence architecture owner
applies_to:
  - PRJ-KIN-BRAU-CI
source_of_truth: golden-repo
source_refs:
  - "SRC-CONF-CIR-001"
  - "SRC-JIRA-BAP-001"
  - "SRC-CONF-PRD-001"
  - "SRC-XR-TDD-001"
  - "SRC-CI-RES-001"
last_updated: 2026-11-05
review_due: TBC
---

# Cash Intelligence architecture — overview and decisions

> **Approved architecture: the R1 design baseline (DEC-006).** Approved by the Cash Intelligence architecture owner on 2026-11-05, after the 2026-11-03 design review. Generated specs follow it; a spec that needs something different is a change here first (GOV-CI-004 in `projects/branch-automation-cash-intelligence/05-delivery/delivery-model-and-gates.md`). Open questions (§16) stay open until answered. Section numbers (§n), open questions (Qn) and decisions (A–I) use the architecture's own numbering; `products/cash-intelligence/architecture/architecture-overview.md` maps each section to its file, and the open questions are in `products/cash-intelligence/architecture/open-questions.md`. Citations: `BAP-nn` = `SRC-JIRA-BAP-001`, `TDD:n` = line n of `SRC-XR-TDD-001`, `research/nn` = `SRC-CI-RES-001` (see the source index).
>
> **Legal review required.** This file covers retention, staff-data handling, tenant isolation and fail-closed access scoping (detail in the linked files). These touch compliance: Kinective's Legal team must review them before they are relied upon or selected as Astra context.

Target: **MySQL 8** (InnoDB; 8.0.36+ and 8.4 LTS feature set only, so it runs unchanged on AWS RDS or Aurora MySQL 3
and on Oracle Cloud MySQL HeatWave; the hosting choice is open, §16 Q21). Naming: `snake_case`. CI is an ABS (Agentic
Banking Solution) built on the **Crossroads Foundation** platform (§2a). The logical areas (`ref`, `cash`, `cfg`, …) are
written `area.table` in this document for readability and implemented as **table-name prefixes in one database per
tenant** (`ref_device`, `cash_inventory_snapshot`). Only CI's own store is MySQL; the source systems are unchanged.
MySQL-specific implementation patterns are in §15. Scope: R1 design-complete; R2 (AI forecasting, external signals) and R3
(vault/drawer via DC Gateway, Check21) have extension points but no detailed tables.

## Provenance

The architecture was written and checked by the Cash Intelligence architecture owner against the registered sources: the Confluence requirements (`SRC-CONF-CIR-001`), the PRD (`SRC-CONF-PRD-001`), the Jira R1 stories (`SRC-JIRA-BAP-001`) and the Crossroads Foundation TDD v0.5 (`SRC-XR-TDD-001`). Source-system behaviour (Link/S4, S4DataSync, iQ, Cash Optimizer, KBOP) was verified in code, and the schema was built and tested on MySQL 8; those records are in `SRC-CI-RES-001`, cited as `research/nn`. Every acceptance criterion was traced to the design, every source-fed column has a registered origin (`lineage-register.md`), and a consistency review after the move onto Crossroads lists what must be re-run (`research/95`). The physical DDL and its tests are not in this repository; they will live with the code.

## Decisions

| # | Decision | Consequence |
|---|---|---|
| A | **Standalone data, platform services from Crossroads** — CI has its own ingestion, read model and breach alerting; hosting, identity, product configuration and tenancy follow Crossroads (§2a) | CI reads the source systems itself and shares no tables with other products. No cash data and no breach state goes onto Crossroads topics or Orkes |
| B | **MySQL 8**, decided; hosting open (AWS or Oracle Cloud) | Portable MySQL 8.0/8.4 features only; implementation patterns (versioning, filtered uniqueness, analytics) in §15 |
| C | **Freshness: design for options** | Same schema serves daily batch, micro-batch and push (§4) |
| D | **CI owns the configuration shape; the Crossroads Platform Config Service stores it** | Settings and rule parameters live in the platform, validated against CI's schema (§10). Rule identity, assignments, recipients and replenish schedules stay in CI's database. Values may be seeded from Cash Optimizer at onboarding |
| E | **Forecasting is pluggable; CI stores all forecasts with history** | Model registry + run vintages; R1 loads CO forecasts for ATMs and computes a baseline for TCRs (§9) |
| F | **Leave room for R2/R3** | Extension points in §12 |
| G | **No made-up data** | Every column has a registered origin (Appendix A). A value with no source is stored as NULL with a reason, never imputed silently. Every feed run is reconciled against its source (§5.3) |
| H | **One database and one object-storage bucket per tenant** (Crossroads D-22) | No pooled placement. Isolation between tenants is structural; `tenant_id` stays on every row for audit and portability (§14.1) |
| I | **Breach alerting stays in CI** | A three-state lifecycle over CI's own data needs no workflow engine. A Crossroads journey is the extension point if breaches join the shared work surface (§10.3, §12) |

## Where each section lives

One topic per file. Section numbers are kept so cross-references (§n) resolve through this table; a subsection (§7.3a, §10.1) lives in its parent's file unless listed.

| Section | Topic | File |
|---|---|---|
| §1–§2 | What CI is; design principles; decisions A–I | `products/cash-intelligence/architecture/architecture-overview.md` |
| — | How CI uses Crossroads in R1 (from the 2026-11-02 review) | `products/cash-intelligence/architecture/crossroads-platform-fit.md` |
| §2a | Platform and services | `products/cash-intelligence/architecture/platform-and-services.md` |
| §2b | ci-service API contract (DEC-016); the machine-readable contract is `ci-service-api.openapi.yaml` beside it | `products/cash-intelligence/architecture/api-contract.md` |
| §3 | Schema map and data flow | `products/cash-intelligence/data/schema-map.md` |
| §3.2 | Data processing flow | `products/cash-intelligence/data/data-processing-flow.md` |
| §4 | Freshness options | `products/cash-intelligence/integrations/freshness-options.md` |
| §5 | Ingestion, staging and reconciliation | `products/cash-intelligence/integrations/ingestion-staging-and-reconciliation.md` |
| §6 | Reference data | `products/cash-intelligence/data/reference-data.md` |
| §7 | Cash facts | `products/cash-intelligence/data/cash-facts.md` |
| §8 | Device health (BAP-21, BAP-85) | `products/cash-intelligence/data/device-health.md` |
| §9 | Forecasting | `products/cash-intelligence/data/forecasting.md` |
| §10 | Configuration, rules and CI-owned operational data (BAP-92, BAP-77, BAP-78) | `products/cash-intelligence/data/configuration-and-rules.md` |
| §10.3 | Rule state, breaches, alerts and service records | `products/cash-intelligence/data/breaches-alerts-and-user-inputs.md` |
| §11 | Identity and access (Roles stories, BAP-110, PRD §8) | `products/cash-intelligence/architecture/identity-and-access.md` |
| §12 | Extension points (R2, R3, adjacent initiatives) | `products/cash-intelligence/architecture/extension-points.md` |
| §13 | Cross-cutting: audit and security | `products/cash-intelligence/architecture/audit-and-security.md` |
| §14 | Physical design, volumes and retention | `products/cash-intelligence/data/physical-design-volumes-and-retention.md` |
| §14.1 | Tenancy and deployment model | `products/cash-intelligence/architecture/tenancy-and-deployment-model.md` |
| §14.2 | Deployment and scaling | `products/cash-intelligence/architecture/deployment-and-scaling.md` |
| §14.3 | Repositories, deployment and local development | `products/cash-intelligence/architecture/repositories-and-local-development.md` |
| §15 | MySQL implementation patterns (decision B) | `products/cash-intelligence/data/mysql-implementation-patterns.md` |
| — | DDL deviations register: where the built DDL departs from or fills a gap in these sections (DEC-027) | `products/cash-intelligence/data/ddl-deviations.md` |
| §16 | Open questions (Q1–Q64) and questions for the KBOP team | `products/cash-intelligence/architecture/open-questions.md` |
| Appendix A | Data lineage register | `products/cash-intelligence/data/lineage-register.md` |

## 1. What CI is, in data terms

CI is a **read-mostly analytics product over cash records** (positions, flows, orders, device health), unifying two
legacy estates that never shared data:

| Estate | Devices | Source of record | How CI gets it |
|---|---|---|---|
| Link/S4, through iQ | TCR, TCD, coin dispenser (kiosk recyclers arrive as TCRs, §6) | `cfm_*` database per bank (one DB may hold several clients), synced by S4DataSync to **iQ**, which is shared and multi-tenant on Kinective AWS | Read iQ's Kafka `s4-sync-table` or its Mongo copies (`s4_sync_cfm_*`). Link itself is not read (§4) |
| ESQ Cash Optimizer (CO) | ATM, CDM, recycler | `CashManagementDB` database per bank or cohort, on-prem | A CI-owned CO connector at the bank: windowed read-only extract (CO has no change tracking) |
| ESQ EJ parser (`SSTRMM`) | ATM | `EJ_TRANSACTIONS` | Through the same connector; ATM transaction counts only, strict allowlist (§7.3) |

Engineering's constraint (kickoff, 2026-10-22): **"No changes in the source applications… There'll be a data
pipeline."** CI is about cash records, not transactions, so it stores **daily device-level facts and
current state**, not a transaction ledger. Source journal rows exist only transiently in staging (§5). Line-level
journals and check images are out of scope: CI ingests no check images or card data from any source, and where
they live (a separate journal service is proposed) is open (§16 Q57); user uploads are limited by the attachment rule in §13. One request
goes beyond that constraint and needs the Link/S4 and iQ teams' agreement: syncing the reference tables iQ lacks (§4).

CI itself writes only: service records, breach and alert state, user-entered vault balances, rule assignments and
recipients, CI roles and access scope (until host claims carry them, §11), and its own forecasts. Settings and rule
parameters are written to the Platform Config Service (§10). Identity belongs to the host (KBOP); CI references it.

**Tenant.** A CI tenant is the **organisation that manages the cash and subscribes to CI**. That is usually a bank or
credit union, but it can be a non-bank, such as a processor or an ESQ client that runs ATMs for others. Being a bank is
not what makes a tenant; owning the cash operation and the subscription is. In Crossroads terms a tenant is one
**institution** (`institutionId`); in KBOP terms it is one FI entity (`entity_id`, the Kinective ID). One tenant can
span several Link clients, iQ clients or CO tenants (`ref.tenant_external_key`); one device belongs to exactly one
tenant. Where this document says "FI" or "bank" it means a tenant, except where it names KBOP's FI entity type. A
tenant that serves several banks' devices may need an optional grouping level above region (open, §16 Q34).

---

## 2. Design principles

| # | Principle | Driven by |
|---|---|---|
| P1 | **Daily device facts are the core.** Grain: device × business date (× denomination). Every page in R1 is a slice or roll-up of these plus current state | All analytics pages; "cash records, not transactions" |
| P2 | **CI keeps its own history.** iQ's Mongo copies (`s4_sync_cfm_*`) keep raw journal, start-inventory and downtime rows indefinitely (back to 2023 in the test environment), so onboarding backfills from them; Link itself also retains journals and start inventory (purge is opt-in, disabled devices only). CO overwrites hopper profiles and `PreForecast` in place. CI snapshots what it ingests and never depends on re-reading history later | `research/60` C1, C2, C5; `research/82` S-N16, S-N20; `research/91` |
| P3 | **Explicit business date, assigned once per source with that source's own rule.** Link: UTC timestamp → branch IANA zone, then the EOD cutoff (before 09:00 local → previous day). CO: UTC − fixed `TermOffset` minutes (CO's own inverse, no DST); EJ auto-loads: the stored local date. Never re-derive one source's date with another's rule | `research/60` headline 2–3, C4 |
| P4 | **Counts are truth, amounts are derived.** Denomination detail stores counts and a `denomination_id`; amounts are persisted computed columns. Where a source gives amounts (CO tenants with `Cash_Mode = Amount`), counts are amount ÷ face value, rounded, with the residual kept in `amount_residual`; `Bill_Count` tenants already give counts. Money is `DECIMAL(19,4)` | NFR "exact money"; C18; `research/82` S-N4, S-N22 |
| P5 | **One database per tenant; `tenant_id` on every row anyway.** Tenants are separated by database and credential (decision H). `tenant_id` still leads every primary key (pre-tenant lookups and staging keys excepted, §15), so rows stay self-describing in audit, exports and moves, and a routing bug that reaches the wrong database still finds no matching tenant | Crossroads D-22, Contract 13 |
| P6 | **Idempotent, source-keyed ingestion.** Every ingested row has a natural key `(source_instance_id, source_table, source_pk)`. Source identities are per database, not global (Link `cfm_journal.id`, CO `TenantId = 100` everywhere). Replays upsert. Never key on payload hash | `research/60` C3, C13 |
| P7 | **Resolve identities through alias tables, with validity dates.** One device has four ids in Link; a CO terminal can change `TermId` over time. `*_source_xref` is the single resolution point | `research/60` C7 |
| P8 | **Config versioned, with defaults and explicit precedence.** Settings and rule parameters are versioned by the Platform Config Service against CI's schema; rule assignments and recipients are versioned rows in CI. Exclusive rule types resolve to exactly one rule per target, falling back to a tenant default rule | Settings epic; `research/50` G3; Crossroads Contract 9 |
| P9 | **Identity referenced, scope applied by the service, fail-closed.** CI stores host ids, not credentials. Branch and device-model scope is a join every page query makes on the caller's `user_id` (`acc.v_user_device_scope`); no scope rows means no data. Nothing depends on database session or connection state | Roles stories; G13; `research/81` N1; `research/35` D1–D3; Crossroads Contract 16 |
| P10 | **No end-customer PII, minimal staff data.** No member, account or card data in any ingested column. Staff data limited to what a story needs. Source free text excluded by per-feed allowlist. The staff data CI holds is listed in §13.1 | Org policy |
| P11 | **Derived values are computed at request time**, except rule/alert state, device projections and the optional dashboard 90-day aggregate (§15), which are materialized and **refreshed on any input change, including configuration changes** | Keeps Settings instantly effective; G4 |
| P12 | **Every value has a registered origin.** Source column, CI derivation, user entry, or not available (Appendix A). "Not available" columns are NULL plus a `*_source` or quality flag that tells the page to exclude, not to rank as zero | Decision G |
