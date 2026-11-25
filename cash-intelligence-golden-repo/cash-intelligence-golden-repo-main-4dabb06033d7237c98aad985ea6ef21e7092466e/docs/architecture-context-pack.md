---
id: DOC-KIN-CI-ARCH-PACK
title: Architecture Context Pack
artifact_type: selection-guide
status: draft
content_state: derived
owner_role: Cash Intelligence architecture owner
applies_to:
  - PRJ-KIN-BRAU-CI
source_of_truth: golden-repo
source_refs:
  - "SRC-ASTRA-IMPL-001 §9-10"
  - "SRC-XR-TDD-001"
last_updated: 2026-11-20
review_due: TBC
---


# Architecture Context Pack

Use explicit Astra file selection for this pack when the flow produces technical specifications, data models or work
items that touch storage, ingestion, access or deployment. A flow that writes acceptance criteria, specifications or
test cases also selects the Test Harness Context Pack (`docs/test-harness-context-pack.md`), so each criterion names
how it is verified in the R1 harness. Add the Minimum BRD Context Pack
(`docs/minimum-brd-context-pack.md`) for requirement detail, and the remaining topic files under
`products/cash-intelligence/data/` and `integrations/` when the flow needs a specific area (cash facts, device health,
forecasting, configuration and rules, breaches and alerts, reference data, lineage).

**Selection gates.** The Cash Intelligence architecture files under `products/cash-intelligence/` are approved by
the architecture owner as the R1 design baseline (DEC-006, content state *approved-design*). Do not select this pack
for Astra generation until (1) the architecture and design review has accepted it (done, 2026-11-03), (2) the Golden
Repo Administrator and the product owner have agreed DEC-006 and DEC-007 (Golden Repo Administrator: the merge
request that approved the files; Product Owner: pending), and (3) Kinective's Legal team has reviewed every file
marked "Legal review required". Until then the pack is for review only. Once selected, generated specs follow these
files (GOV-CI-004 in `projects/branch-automation-cash-intelligence/05-delivery/delivery-model-and-gates.md`) and
carry their open questions forward. The other files in the pack
keep their own status: the Crossroads obligations are source-backed rules every ABS must follow, and the decision log
and source index are the governance and provenance records.

| # | File | Why required |
|---|---|---|
| 1 | `products/cash-intelligence/architecture/architecture-overview.md` | Decisions A–I, design principles, and the map from each section number to its file (Legal review required) |
| 2 | `global/03-engineering-architecture/crossroads-abs-obligations.md` | Crossroads rules every ABS must follow, cited to TDD lines (Legal review required) |
| 3 | `products/cash-intelligence/architecture/crossroads-platform-fit.md` | How CI applies those rules in R1: what it uses, its deployables, breach lifecycle, configuration split (Legal review required) |
| 4 | `products/cash-intelligence/architecture/platform-and-services.md` | Services, platform obligations and the bank-side connector |
| 5 | `products/cash-intelligence/data/schema-map.md` | Every table by logical area |
| 6 | `products/cash-intelligence/data/data-processing-flow.md` | How a feed run, recompute and rule evaluation move data |
| 7 | `products/cash-intelligence/integrations/freshness-options.md` | Freshness tiers and real-time behaviour |
| 8 | `products/cash-intelligence/integrations/ingestion-staging-and-reconciliation.md` | Feeds, staging, quarantine and reconciliation controls |
| 9 | `products/cash-intelligence/architecture/identity-and-access.md` | Tokens, roles, scope and how every query is scoped (Legal review required) |
| 10 | `products/cash-intelligence/architecture/tenancy-and-deployment-model.md` | One database and bucket per tenant, catalogue, restore, offboarding (Legal review required) |
| 11 | `products/cash-intelligence/architecture/deployment-and-scaling.md` | Replicas, leases, change signal, connection limits |
| 12 | `products/cash-intelligence/data/mysql-implementation-patterns.md` | MySQL 8 patterns every implementation must follow |
| 13 | `products/cash-intelligence/architecture/open-questions.md` | The architecture questions Q1–Q64 and how the architecture answers OQ-001–OQ-015, which flows must carry forward |
| 14 | `projects/branch-automation-cash-intelligence/08-evidence-decisions/decision-log.md` | DEC-006 onward: what was added, approved and still pending |
| 15 | `projects/branch-automation-cash-intelligence/08-evidence-decisions/source-index.md` | Source authority, including `SRC-XR-TDD-001` and `SRC-CI-RES-001` |
| 16 | `products/cash-intelligence/architecture/repositories-and-local-development.md` | Code repository layout, database migrations, `deploy/` (Helm, Harness), local development environment |
| 17 | `products/cash-intelligence/architecture/api-contract.md` | ci-service API contract: conventions, endpoint map, the four shell seams (DEC-016) |
| 18 | `products/cash-intelligence/architecture/ci-service-api.openapi.yaml` | The machine-readable contract (OpenAPI 3.2) the micro-frontend and ci-service build against (DEC-016) |

Before generation, verify every selected file is vectorised and not stale (`SRC-ASTRA-IMPL-001` §10, §12).
