---
id: ARC-KIN-CI-107
title: Cash Intelligence on Crossroads — platform fit (R1)
artifact_type: architecture
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
  - "SRC-CI-RES-001"
last_updated: 2026-11-05
review_due: TBC
---

# Cash Intelligence on Crossroads — platform fit (R1)

> **Approved architecture: the R1 design baseline (DEC-006).** Approved by the Cash Intelligence architecture owner on 2026-11-05, after the 2026-11-03 design review. Generated specs follow it; a spec that needs something different is a change here first (GOV-CI-004 in `projects/branch-automation-cash-intelligence/05-delivery/delivery-model-and-gates.md`). Open questions (§16) stay open until answered. Section numbers (§n), open questions (Qn) and decisions (A–I) use the architecture's own numbering; `products/cash-intelligence/architecture/architecture-overview.md` maps each section to its file, and the open questions are in `products/cash-intelligence/architecture/open-questions.md`. Citations: `BAP-nn` = `SRC-JIRA-BAP-001`, `TDD:n` = line n of `SRC-XR-TDD-001`, `research/nn` = `SRC-CI-RES-001` (see the source index).
>
> **Legal review required.** This file covers staff personal data and Legal approval of anonymized benchmarks and email wording (*Personal data*, *What CI must not do*). These touch compliance: Kinective's Legal team must review them before they are relied upon or selected as Astra context.

Outcome of the 2026-11-02 design review that tested the Crossroads Foundation design with Cash Intelligence (CI) as a real ABS. Pinned to `SRC-XR-TDD-001` (TDD v0.5, commit 8c55357). The platform rules any ABS must follow are in `global/03-engineering-architecture/crossroads-abs-obligations.md`; this file records how CI applies them. Where this file and the topic files differ, the topic files win. `Fn` marks a finding raised with the Crossroads team (recorded in `SRC-CI-RES-001`, `research/39` §11); each maps to an open question in §16.

## Fixed decisions

| Decision | Value |
|---|---|
| Database | **MySQL 8, decided.** Portable 8.0.36+/8.4 features only, nothing specific to Aurora or HeatWave |
| Hosting | Services on AWS EKS, namespace `ci` (TDD:387–388). Database hosting is **open**: AWS (RDS or Aurora MySQL 3) or Oracle Cloud (MySQL HeatWave). Crossroads lists it as open too (TDD:405, R-36 at TDD:1180), and notes that Oracle Cloud puts a cross-cloud hop between services and databases (finding F8) |
| Tenancy | One database and one S3 bucket per tenant (D-22, TDD:394–405), treated as decided. No pooled placement |
| Naming | Logical areas (`ref`, `cash`, `cfg`, `ops`, …) are written `area.table` in documents and implemented as **table-name prefixes** in the one database per tenant (`ops_breach`). In MySQL a schema is a database, so real schemas would mean several databases per tenant |
| Language | Kotlin on JDK 25 with Ktor (D-21, TDD:371–392). CI is greenfield. The micro-frontend template's JDK 17 is incidental (its author, 2026-11-19); CI stays on JDK 25 |
| Frontend | A **React** child micro-frontend of the KBOP shell on the React library of the Prism 2.0 design system, `@kinective/prism-react` (DEC-017): the template's vendored Prism copy until a published release supports React 19, has the rail-panel Sidebar and Crossroads' web half is written against it, then release-pinned packages from JFrog (DEC-023) (D-23, TDD:378), built from the Crossroads ABS micro-frontend template (SRC-XR-MFE-001): a web component served by ci-service's `api` unit through `crcommons-mfe-server`; the published 3.0.0 declares React 18 and Crossroads' web half is written against the vendored copy's API (DEC-023, §16 Q60). |
| Breach lifecycle | **Stays in CI.** No Orkes journey in R1 (see *Breach lifecycle in CI*) |

## What CI uses from Crossroads in R1

| Used | How |
|---|---|
| Runtime and stack | EKS namespace `ci`, Kotlin/Ktor, OpenAPI 3.2, React MFE |
| Identity and host | Auth0 tokens issued through the KBOP shell. Contract 16 (`institutionId`, `roles[]`, `licences[]`, scope attributes, TDD:529) is the target; until it arrives CI validates KBOP's own claims and resolves the tenant from KBOP `entity_id` (§11) |
| Product configuration | Platform Config Service (Contract 9; see *Configuration*), through CI's configuration interface whose real adapter is the Crossroads services SDK (DEC-022; whether that is the TDD's Foundations SDK is §16 Q62) |
| Tenancy and storage | Per-tenant database, bucket and credential; platform-owned provisioning and Flyway fleet runner (TDD:394–405) |
| Isolation test | Run against CI's BFF before go-live (Contract 13; Step 11, TDD:970) |

**Not used in R1:** Orkes journeys and schedules, the Data Triage service, platform event topics and the Platform
Gateway. CI has no workers. Whether such an ABS passes the go-live gates is finding F2.

## Deployables

| Deployable | Runs | What it does | Database access |
|---|---|---|---|
| **ci-ingest** | EKS, continuous | Reads iQ and receives Cash Optimizer extracts; writes current state and history into each tenant's database; reconciles; runs the rule evaluator on each change | Direct, per tenant |
| **ci-service** | EKS, two deployment units: `api` and `jobs`, each with its own credential | The BFF: pages, exports, service records, assignments, forecasts and order recommendations. The `jobs` unit runs CI's **scheduler**: nightly closing snapshot, forecast load, reconciliation sweep, reminders, housekeeping; and sends alert email | Direct, per tenant |
| **CO connector** | At the bank, next to Cash Optimizer; owned and built by CI | Windowed read-only extract of CO and EJ; sends batches to ci-ingest through the Crossroads Ingress Gateway, the only public ingress (D-19, TDD:863); ci-ingest binds the tenant from the forwarded token (F10) | Source only |
| CI micro-frontend | KBOP shell | Pages | Through ci-service; the browser route from the KBOP shell is open (Q48) |

Every EKS unit runs as stateless replicas; feeds and scheduled jobs are shared out by leases in `ci_catalog`, and a
polled change signal carries live updates between replicas, so CI adds no infrastructure (§14.2,
Q44). Partition maintenance runs as its own CronJob under the maintenance account.

## Data sources and freshness

| Source | Path into CI | Freshness |
|---|---|---|
| **TCR data (Link/S4)** | Link → S4DataSync (at the bank) → **iQ**, shared multi-tenant on Kinective AWS (research/30:161). ci-ingest reads iQ's Kafka `s4-sync-table` (needs the iQ team's agreement) or its Mongo copies | Near real time with Kafka; 5–15 min with Mongo |
| **Reference tables iQ lacks** (`cfm_conf_cluster`, `cfm_conf_peripheral_device`, `cfm_error_code`) | Ask the Link/S4 team for change tracking and S4DataSync senders, and the iQ team to store them. Until a bank has upgraded, a **CSV** | Slow-changing |
| **ATM data (Cash Optimizer)** | The CO connector | Last approved EOD; EJ activity hourly at best |

- Link/S4 is not read directly.
- Reading iQ's Kafka is a **source connector**, not a use of Crossroads topics. The "no raw Kafka client" rule
  (TDD:415) governs Crossroads platform topics (ruling in research/39a, V3).
- The `cfm_conf_cluster` sync must send safe columns only. It has credential-like columns: `shared_token`,
  `cluster_public_key`, `os_upgrade_hub_password`, `repair_token` (research/10:58). Raise the `cfm_email_setting`
  defect at the same time (research/10:258).
- Product must give a freshness number for TCRs. ATM pages show an "as of" label with the EOD time.
- No cash data goes onto Crossroads. The TDD itself says to publish state changes, not aggregates (TDD:1161), and
  that Crossroads is not a data pipeline (TDD:912).

## Breach lifecycle in CI

**A breach** is a device breaking one of the bank's TCR maintenance rules: notes moved since the last service over a
threshold (*preventative*), or N errors at or above a severity in a window (*proactive*). It opens when the device
first breaks the rule, alerts, reminds on the rule's cadence, and closes when a service record is logged or the
condition clears. A later breach is a new breach (research/00:374–387). Device-down episodes do not alert in R1.

**Why not an Orkes journey** (decision of 2026-11-02, on merit):
- The requirement is three states over CI's own data. Durable state, timers, idempotent reminders, retries and
  multi-instance safety are a few tables and a scheduled job.
- On Crossroads the same reminder would pass through an Orkes schedule, a child run per tenant, a WAIT, a SWITCH, a
  worker task and the gateway into ci-service. Workers have no database access, so all of that is relay.
- Orkes is priced per task execution, and the TDD warns against polling loops in journeys (TDD:894). A
  remind-until-resolved journey sits close to that.
- What Orkes adds that CI lacks (a shared work surface, approvals, cross-system compensation) is not in the R1
  requirements. Acknowledge and assign appear only in the TDD fit table (TDD:1020–1028).

**Shape** (detail in §10.3):
- The rule evaluator is the only writer of breach state. It is shared by ci-ingest and ci-service, triggered by feed
  runs, service records, and rule, assignment or configuration changes, and serialized per device by locking the
  device's state rows.
- `ops.device_rule_state` holds the counter, baseline and open breach. `ops.breach` is the history; its outcome is
  `serviced`, `recovered` or `rule_changed`. There is no expiry: reminders continue until the breach is resolved
  (requirement).
- `ops.alert_dispatch` is an outbox with a unique key on (breach, alert number), written in the state-change
  transaction and sent after commit with retries.
- Reminders run hourly in ci-service's scheduler: "due and not yet sent" in the alert's local time zone, claimed
  with `SELECT … FOR UPDATE SKIP LOCKED` after locking the device. A sweep every 4 hours re-evaluates every assigned
  device (requirement).

**Crossroads rules this must respect:** "Keep run state anywhere but Conductor and the registry; ABS tables hold
domain data, not orchestration state" (TDD:991). CI's position is that with no run there is no run state, so breach
state is domain data. That needs a ruling (F2).

**Extension path:** if breaches join the shared work surface (acknowledge, assign, escalate), the evaluator publishes
`ci.breach.opened` / `ci.breach.resolved` through CI's own interface whose real adapter is the Crossroads services SDK (DEC-022; whether that is the TDD's Foundations SDK is §16 Q62), and a journey keyed `device + rule` raises the
work item and takes over reminders. `ops.breach` keeps the domain history. That path needs a work-item withdraw
transition (F4). To keep it cheap: one writer, a stable breach id, and outcome codes that `abs_state.outcome` can
carry.

## Configuration

| Where | What |
|---|---|
| **Platform Config Service** (CI's schema `product.v1.json`, Contract 9, TDD:472–478) | Rule thresholds and cadences, reminder time, EOD time, overstock defaults, freshness tier, and the required `version`, `inferenceGateField`, `autonomyLevel` (`off`) and numeric ranges |
| **CI's database** | Rule identity, device assignments, **alert recipients** and replenish schedules |

- The platform store is PII-free (TDD:478), so recipient email addresses cannot go there. Assignments are device-level
  data that changes when devices move. This contradicts "configuration is no longer held in any ABS domain store"
  (TDD:478); finding F5.
- Every schema also needs a `preview` mode with a CI-supplied sample evaluator (TDD:476). Where the evaluator runs is
  open (F11). Activation approval is the ABS's choice (TDD:922): **none in R1**; an Admin's activation takes effect
  at once. ci-service checks the Admin role before every Config Client write and passes the user as the actor.
- ci-ingest and ci-service read configuration through the Config Client and re-evaluate on the change notification
  (TDD:417, 477).

## Tenancy and storage

- Provisioning and the Flyway fleet runner are platform-owned. Every change is expand-then-contract; code runs against
  schema N and N−1.
- **Connections:** ci-ingest and ci-service connect to every tenant's database. Use whichever layer Crossroads picks for
  R-34 (TDD:1178). Never depend on connection or session state.
- **User scope** (branches, device models) is a filter applied in CI's queries from the user's identity. The database
  separates tenants.
- **Routing iQ data:** iQ is one shared feed keyed by client name. ci-ingest maps each row to the tenant and its
  database:
  - map on an immutable client id if iQ has one;
  - hold rows from unknown clients; never use a default database;
  - test that rows from two clients land only in their own databases;
  - keep the mapping between `institutionId`, KBOP `entity_id` and the iQ client in one place.
- A cash-managing processor that serves several banks must be modelled as an institution, not a partner. Confirm with
  Crossroads (§16 Q34).

## Personal data

- CI holds **staff** personal data in its own database and S3: alert recipient addresses, technician names and
  service notes, uploaded service files, user records.
- CI holds no bank-customer data.
- CI publishes nothing to Crossroads in R1. If breach events are published later, they carry device, rule and codes
  only, with every string flagged `x-pii: false` (Contract 17, TDD:534–543).
- The alert email is where staff data leaves Kinective. Sending it from ci-service is pending F3. The email is
  Kinective-branded; CMP-3 (TDD:101) requires Legal review of customer-facing wording produced by a workflow. The alert
  goes to bank staff, not customers, but have Legal confirm the template anyway.
- CI's brief says "holds staff personal data in its own store; none on the spine", not "PII-free".
- Retention and purge are per tenant: drop the tenant's database and bucket, after the audit archive and unless a legal
  hold is set (the hold's master copy is in `ci_catalog`, so it outlives the database).

## What CI must not do

From the TDD (TDD:988–995), the rules that apply in R1:
- publish to a Crossroads topic with a raw Kafka client, or without `institutionId`;
- run its own work queue or approval surface **for platform work items**. CI's internal recompute queue is part of
  its data pipeline, not a work surface;
- hold, cache or show another tenant's data anywhere, including benchmarks unless they are anonymized by design and
  approved by Legal (TDD:994);
- put CI-specific values into platform tables, enums or schemas.

If a journey is added later, the rest of the list applies too: no direct Conductor calls, no run state in CI tables,
workers with no database connection.

## Artifacts CI produces (R1)

```
schemas/config/product.v1.json     CI configuration schema, with preview evaluator
services/ci-ingest, services/ci-service, connectors/co-connector
frontend/                          React MFE on Prism React
db/                                Flyway changesets: tenant databases, ci_catalog, audit archive
deploy/                            Helm chart, values per environment, Harness pipelines, OpenTofu modules
dev/                               local development environment and simulators
docs/brief.md                      entity key (device + rule, for the extension path), ingest modes,
                                   staff-PII statement, why no journey in R1
```

The full repository layout and the local environment are in §14.3
(`products/cash-intelligence/architecture/repositories-and-local-development.md`).

No `conductor/`, `config/work/` or `config/gateway/` entries in R1. Go-live: the isolation test against CI's BFF and
whatever the gate ruling (F2) requires.
