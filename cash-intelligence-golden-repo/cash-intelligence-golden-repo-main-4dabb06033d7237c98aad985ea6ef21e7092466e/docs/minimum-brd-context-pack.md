---
id: DOC-KIN-CI-BRD-PACK
title: Minimum BRD Context Pack
artifact_type: selection-guide
status: draft
content_state: derived
owner_role: Golden Repo Administrator
applies_to:
  - PRJ-KIN-BRAU-CI
source_of_truth: golden-repo
source_refs:
  - "SRC-ASTRA-IMPL-001 §9-10"
  - "SRC-CONF-CIR-001"
last_updated: 2026-10-28
review_due: TBC
---


# Minimum BRD Context Pack

Use explicit Astra file selection for this pack when the goal is a source-grounded BRD. The pack is deliberately small; add the per-area `req-*.md` files when the flow needs full acceptance-criteria detail for a specific page.

| # | File | Why required |
|---|---|---|
| 1 | `projects/.../01-business-product/business-context.md` | Objective and background as stated in the source |
| 2 | `projects/.../01-business-product/scope-and-boundaries.md` | The pages in scope and what the source does not cover |
| 3 | `projects/.../01-business-product/personas-and-roles.md` | Admin / User / Service Provider and the actor wording used in stories |
| 4 | `projects/.../01-business-product/business-rules.md` | Classification, flagging, recommendation and rebalancing rules |
| 5 | `projects/.../01-business-product/glossary.md` | Terms as used in the source |
| 6 | `projects/.../02-requirements/requirements-index.md` | All 84 requirement IDs with their area and file |
| 7 | `projects/.../02-requirements/default-threshold-settings.md` | The 16 configurable defaults |
| 8 | `projects/.../02-requirements/assumptions-dependencies-open-questions.md` | Prevents silent invention where the source is incomplete |
| 9 | `projects/.../03-design-architecture/navigation-and-routes.md` | Routes, page titles and navigation stated in the source |
| 10 | `projects/.../03-design-architecture/external-touchpoints.md` | Host system, email alerting, exports, file upload as stated |
| 11 | `projects/.../04-data-ai/conceptual-data-dictionary.md` | Entities and attributes named in the source |
| 12 | `projects/.../08-evidence-decisions/source-index.md` | Provenance and authority rules |
| 13 | `projects/.../05-delivery/delivery-model-and-gates.md` | How a BRD relates to this repository: cite `REQ-CI-*`, add nothing (DEC-012, proposed) |

Before generation, verify every selected file is vectorised and not stale (`SRC-ASTRA-IMPL-001` §10, §12).
