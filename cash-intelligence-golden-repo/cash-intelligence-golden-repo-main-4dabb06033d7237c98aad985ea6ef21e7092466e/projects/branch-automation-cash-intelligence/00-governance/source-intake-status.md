---
id: GOV-KIN-CI-002
title: Source Intake Status
artifact_type: intake-register
status: draft
content_state: derived
owner_role: Golden Repo Administrator
applies_to:
  - PRJ-KIN-BRAU-CI
source_of_truth: golden-repo
source_refs:
  - "SRC-CONF-CIR-001"
last_updated: 2026-11-03
review_due: TBC
---


# Source Intake Status

| Source | Status in this draft | Instruction / reason |
|---|---|---|
| Confluence *Cash Intelligence Requirements* (`SRC-CONF-CIR-001`) | **Ingested** — v30, 2026-10-19 | Only product source received |
| Cash Intelligence prototype `cash-analytics` (`SRC-PROTO-001`) | **Excluded** | "Let's not consider this for Golden Repo for now" |
| Prism 2.0 design system (`SRC-PRISM-UI-001`) | **Ingested** — release 3.0.0, 2026-11-19 (DEC-017, proposed) | UI layer guidance for specs and builds; the build uses the template's vendored copy until the DEC-023 switch |
| Crossroads Foundation TDD v0.5 (`SRC-XR-TDD-001`) | **Ingested** — commit 8c55357, 2026-10-29 | Platform rules for the architecture (DEC-007) |
| Jira R1 stories (`SRC-JIRA-BAP-001`) | **Cited** | Story keys in the architecture; Confluence wins (DEC-007) |
| Cash Intelligence PRD v0.1 (`SRC-CONF-PRD-001`) | **Cited** | Section references in the architecture (DEC-007) |
| Architecture research notes (`SRC-CI-RES-001`) | **Cited, held outside this repo** | Evidence behind the architecture (DEC-007) |

When a held source is admitted, register it in `08-evidence-decisions/source-index.md`, add a decision-log entry, and re-vectorise affected files.
