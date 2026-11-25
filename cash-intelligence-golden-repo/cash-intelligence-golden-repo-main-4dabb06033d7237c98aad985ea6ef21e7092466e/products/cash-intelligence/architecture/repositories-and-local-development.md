---
id: ARC-KIN-CI-108
title: Cash Intelligence architecture — 14.3 Repositories, deployment and local development
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

# 14.3 Repositories, deployment and local development

> **Approved architecture: the R1 design baseline (DEC-006).** Approved by the Cash Intelligence architecture owner on 2026-11-05, after the 2026-11-03 design review. Generated specs follow it; a spec that needs something different is a change here first (GOV-CI-004 in `projects/branch-automation-cash-intelligence/05-delivery/delivery-model-and-gates.md`). Open questions (§16) stay open until answered. Section numbers (§n), open questions (Qn) and decisions (A–I) use the architecture's own numbering; `products/cash-intelligence/architecture/architecture-overview.md` maps each section to its file, and the open questions are in `products/cash-intelligence/architecture/open-questions.md`. Citations: `BAP-nn` = `SRC-JIRA-BAP-001`, `TDD:n` = line n of `SRC-XR-TDD-001`, `research/nn` = `SRC-CI-RES-001` (see the source index).

**Repositories.** CI has two: the golden repo (requirements, architecture, decisions and open questions, read by Astra
and reviewers) and one code repository, `cash-intelligence-code-repo` (https://gitlab.com/kinective.io/internal/abs/cash-intelligence/cash-intelligence-code-repo), laid out as the Crossroads ABS repository convention
expects. One code repository, because ci-ingest and ci-service share one schema, so a schema change and the code that
uses it ship in one merge request, and the registration tooling expects one repository per ABS. The CO connector,
which runs at the bank and is released on its own schedule, has its own pipeline in the same repository.

```
cash-intelligence-code-repo/
  services/ci-ingest/  services/ci-service/   Kotlin and Ktor; ci-service builds the api and jobs units
  connectors/co-connector/
  frontend/                       React web component on Prism React (Crossroads micro-frontend template), served by ci-service api
  schemas/config/product.v1.json  configuration schema (Contract 9)
  config/ingress/                 ingress routes (D-19)
  db/tenant/                      Flyway changesets for every tenant database (platform fleet runner)
  db/catalog/  db/audit-archive/  Flyway changesets for ci_catalog and the audit archive (CI's own job, §14.1)
  db/tools/                       history-trigger generator (§15 Versioning)
  tests/db/                       data generator, seed checks, reconciliation oracle, lineage check (the database
                                  tests are Kotlin, next to the code they test)
  dev/                            local environment (below)
  deploy/                         Helm chart, values per environment, Harness pipelines, OpenTofu modules
  docs/brief.md
```

- **Database.** Migrations live in `db/`, not in a service, because both services use the schema and migrations run
  before either rolls out. The baseline is the tested DDL (`research/86`), with the seed data in a changeset of its own.
  Every later change is a new changeset (§14.1 Schema changes); one that alters a versioned table regenerates its
  history triggers. Who runs the tenant changesets when several units share one database is open (§16 Q50).
- **Deployment.** `deploy/` holds a Helm chart for CI's deployables (ci-ingest, ci-service `api` and `jobs`, the
  maintenance and migration jobs) with values per environment, the Harness pipeline definitions, and OpenTofu modules
  (ADR-027) for what CI owns outside the chart: namespace resources, IAM roles and caller identities per unit, alarms.
  It holds non-secret values only. Secrets stay in AWS Secrets Manager and `deploy/` only names them; product and
  tenant settings live in the Platform Config Service (§10.1); tenant databases, buckets and credentials are
  provisioned by the platform (§14.1). Crossroads has published no ABS repository template yet; its micro-frontend template (SRC-XR-MFE-001) shapes ci-service `api` and `frontend/`, and CI adopts a repository template when one is published.

**Local development environment.** CI is developed and tested end to end without any outside system. Each outside
system sits behind an interface in the services, with a real adapter and a local one chosen by configuration, so no
code branches on a development mode. `dev/` holds one Docker Compose file with profiles (database only; with the
sources; everything):

| Outside system | Local stand-in |
|---|---|
| Tenant databases, `ci_catalog`, audit archive, Flyway fleet runner | MySQL 8.4 with two tenants, the catalogue and the archive, built by the same changesets and a script that does what the platform's provisioning job does |
| Audit archive in the tenant's S3 bucket under Object Lock (§13) | A database, `ci_audit_archive`, holding the archived audit rows and chain segments of every tenant, written only by the maintenance job's archive login; append-only by its own triggers. |
| Per-tenant S3 bucket with Object Lock | MinIO, which supports Object Lock. MinIO no longer publishes container images, so the stand-in is built from its last source release *(Proposed: not part of the approved baseline until the architecture owner approves it.)* |
| iQ's Mongo copies (`s4_sync_cfm_*`) | Mongo with an iQ simulator that writes documents for two clients, including deletes, renames and an unknown client. The stand-in runs MongoDB 7.0: 8.x exits at start on Linux 6.19 to 7.0.13 (MongoDB SERVER-121912), which shared lab hosts run; iQ's own version is part of §16 Q3 *(Proposed: not part of the approved baseline until the architecture owner approves it.)* |
| iQ Kafka topic (tier C) | One Kafka node with several partitions, so per-partition checkpoints and parking are exercised |
| Cash Optimizer and `EJ_TRANSACTIONS` at the bank | A CO database with the allowlisted tables and a CO simulator (daily imports, approvals that change rows, EJ loads), read by the real CO connector. Until the batch interface through the Ingress Gateway is designed (§5, §16 Q43), the connector has its read side only and writes its batches to a local outbox *(Proposed: not part of the approved baseline until the architecture owner approves it.)* |
| Crossroads Ingress Gateway | A stub that checks the token, strips client-supplied institution headers and forwards to ci-ingest |
| Auth0 through KBOP | The template server library's development mode (`AUTH_MODE=dev`: unsigned tokens, as the shell's DevLogin mints them) to run the app, and a mock token issuer with its own JWKS for the security tests: tokens per role and tenant, and bad ones (wrong signature, expired, wrong institution). |
| KBOP shell | The template's development harness page (`@kinective/crcommons-mfe-web`), which mounts the micro-frontend and passes it a token. |
| Platform Config Service | CI's configuration interface with a local adapter that reads `dev/config`, validates each document against CI's schema and notifies on change; no stand-in of the Config Service API. The real adapter is the Crossroads services SDK once delivered, or its test double if Crossroads ships one (§16 Q49, Q62). |
| Crossroads libraries | The template's libraries (`crcommons-mfe-server` and the modules it needs, and later `@kinective/crcommons-mfe-web`) as a committed copy built from a pinned `crossroads-commons` commit, a temporary measure in the code repository until they are published to JFrog (§16 Q61); Prism as the template's vendored copy until a published release supports React 19, has the rail-panel Sidebar and Crossroads' web half is written against it, then from JFrog Artifactory (`kux-npm-dev-local`) (DEC-023). |
| Secret store | Local credentials per tenant and unit, never committed |
| Email | Mailpit, which captures outgoing email so tests can check alerts and reminders |
| Time | A clock the tests control, so cut-offs, snapshots, reminders and lease expiry run without waiting |

- Two replicas of each service run, so leases, epoch fencing and the change signal (§14.2) are exercised.
- Data is synthetic only, from scenario files; each scenario states its expected result and the end-to-end tests check
  it. No production or personal data enters the environment.
- Contract tests run the same checks against each simulator and, where access is granted, against the real system's
  development instance, so the simulators do not drift (§16 Q53). The schema fixtures of §5.4 feed both.
- Where Crossroads ships test doubles (the services SDK, the token issuer), CI uses them instead (§16 Q49): today the template's development token mode and its test helpers (`crcommons-mfe-testing`).
- Every Crossroads platform service CI's services call sits behind an interface of CI's own, with a local adapter for
  development and tests and the Crossroads services SDK as the real adapter (DEC-022). In R1 that is the
  Platform Config Service, unless the open Crossroads rulings add one (§16 Q35: email egress through the gateway, b;
  outcomes recorded in Datava, c): CI publishes no events, and the Ingress Gateway is a route into ci-ingest, not a
  service CI calls. The micro-frontend template's libraries are not part of the SDK and are used directly (DEC-021).
