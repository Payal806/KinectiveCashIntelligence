---
id: ARC-KIN-CI-101
title: Cash Intelligence architecture — 2a. Platform and services
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
last_updated: 2026-11-05
review_due: TBC
---

# 2a. Platform and services

> **Approved architecture: the R1 design baseline (DEC-006).** Approved by the Cash Intelligence architecture owner on 2026-11-05, after the 2026-11-03 design review. Generated specs follow it; a spec that needs something different is a change here first (GOV-CI-004 in `projects/branch-automation-cash-intelligence/05-delivery/delivery-model-and-gates.md`). Open questions (§16) stay open until answered. Section numbers (§n), open questions (Qn) and decisions (A–I) use the architecture's own numbering; `products/cash-intelligence/architecture/architecture-overview.md` maps each section to its file, and the open questions are in `products/cash-intelligence/architecture/open-questions.md`. Citations: `BAP-nn` = `SRC-JIRA-BAP-001`, `TDD:n` = line n of `SRC-XR-TDD-001`, `research/nn` = `SRC-CI-RES-001` (see the source index).

CI is built on **Crossroads Foundation**, Kinective's shared platform for ABSs
(`gitlab.com/kinective.io/internal/platform/crossroads/crossroads-foundation`, technical design v0.5; `research/39`,
`research/39a`). CI takes from it what a regular service would otherwise rebuild, and keeps its data
work in its own services.

| From Crossroads | How CI uses it |
|---|---|
| Runtime | AWS (or Oracle Cloud for the database, §16 Q21); containers on EKS in namespace `ci`; Kotlin on JDK 25 with Ktor |
| Host and identity | A React child micro-frontend of the KBOP shell on Prism React (Crossroads D-23), built from the Crossroads ABS micro-frontend template (SRC-XR-MFE-001): a web component the shell mounts, whose bundle, `config.json` and `nav.json` ci-service's `api` unit serves under `/mfe/ci/` through the template's server library (`crcommons-mfe-server`, §2b); Auth0 tokens issued through KBOP and sent with every `/api/v1` call (§11). |
| Product configuration | The Platform Config Service holds every tenant's settings and rule parameters against CI's schema (§10) |
| Tenancy and storage | One MySQL database and one S3 bucket per tenant, one credential per tenant per service; provisioning and the Flyway fleet runner are platform-owned (§14.1) |
| Isolation test | Run against CI's BFF before go-live (§11) |

**Not used in R1:** Orkes journeys, the Data Triage service, platform event topics and the Platform Gateway (D-24). CI's
breach lifecycle (open, alert, remind, close) runs over CI's own data, and durable state, timers, idempotent
reminders and retries are a few tables and a scheduled job (§10.3); no cash data goes onto Crossroads. A journey is
the extension point if breaches move into the shared work surface (§12). Whether an ABS with no journeys meets
Crossroads' go-live gates is open (§16 Q35).

**Gateway-target obligations.** No run calls ci-service in R1, so it is not yet a Platform Gateway target. It becomes
one if the Platform Config Service runs CI's preview evaluator by calling CI (TDD:476; where preview runs is open,
§16 Q35), or when a journey is added. ci-service is built to the target rules now (TDD:466–470):
- the institution in the header credential (user token or journey assertion, verified against the platform JWKS)
  must equal `institutionId`; a target still in warn mode is a go-live blocker;
- no public URL; reached only over in-cluster networking (D-24). How the micro-frontend's calls from the KBOP shell
  reach it is open (§16 Q48);
- write operations take an idempotency key: `ops_log_service_record` and `ops_add_vault_entry` take a client
  `request_key`, unique per tenant, so a retried call writes once;
- a conformance probe (R-25).

The TDD's fit table expects CI's only egress to be email notifications through the gateway (TDD:1027), which is the
open ruling on direct email (§16 Q35).

**Services**

| Service | Runs | What it does | Database access |
|---|---|---|---|
| **ci-ingest** | EKS, continuous | Reads iQ and receives CO/EJ extracts; stages, normalizes, writes current state and history into each tenant's database; reconciles; runs the rule evaluator on every change | Direct, per tenant |
| **ci-service** | EKS, two deployment units (`api`, `jobs`; §11) | The BFF for the micro-frontend (API contract: §2b): pages, exports, service records, vault entries, rule assignments, forecasts and order recommendations, live-update notifications. Runs CI's **scheduler**: nightly closing snapshot, forecast load and the TCR baseline, reconciliation sweep, breach alert and reminder email (provider and egress path pending, §16 Q13, Q35b), housekeeping | Direct, per tenant |
| **CO connector** | At the bank, next to Cash Optimizer; owned and built by CI | Windowed read-only extract of `CashManagementDB` and `EJ_TRANSACTIONS` with the allowlist; sends batches to ci-ingest through the Crossroads **Ingress Gateway** (the only public ingress under D-19, which is still proposed in the Crossroads TDD; routes in `config/ingress/`), authenticated per bank. ci-ingest verifies the forwarded token as ci-service does (platform JWKS signature, issuer, audience, expiry), takes the tenant only from its verified institution claim, never from the batch or a client-supplied header, and resolves CO `TenantId`s only within that tenant's keys. The route is direct to ci-ingest (decided, DEC-025) and the batch interface is §5.5 (proposed); what Crossroads must still confirm is in §16 Q43 | Source only (read-only) |
| CI micro-frontend | KBOP shell | Pages | Through ci-service only |

The rule evaluator is one shared module used by both services and serialized per device, so a service record, a
rule change and a feed run cannot race on the same breach (§10.3). Both services read configuration from the Platform
Config Service (Contract 9) through CI's own configuration interface, whose real adapter is the Crossroads services SDK,
and refresh on its change notification (DEC-022). Neither depends on connection state, so they
work with whichever connection layer Crossroads chooses for hundreds of tenant databases (§16 Q36). Every unit
runs as several stateless replicas; feeds and scheduled jobs are shared out by leases in `ci_catalog`, and changes
reach other replicas through a change signal (§14.2).
