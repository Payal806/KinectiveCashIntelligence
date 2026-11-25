---
id: DOC-KIN-CI-GAPS
title: Draft Status and Known Gaps
artifact_type: status-note
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


# Draft Status and Known Gaps

This is v0.1. The table shows which scaffold areas are populated from the source and which are empty because the source has nothing for them.

| Area | Status | Reason |
|---|---|---|
| `global/` operating guidance | Populated (derived from Astra guideline) | Not customer-specific |
| `products/cash-intelligence/strategy`, `domain` | Populated | Objective, background, pages, domain model are in the source |
| `products/.../architecture`, `data`, `integrations` | Populated, *approved-design* (DEC-006) | Architecture owner's design, approved 2026-11-05; Product Owner agreement and Legal review pending |
| `products/.../operations`, `evidence` | Empty | Runbooks and verification evidence not yet written; evidence is held outside the repo (`SRC-CI-RES-001`) |
| `global/03-engineering-architecture` | Crossroads ABS obligations | Source-backed from `SRC-XR-TDD-001` |
| `projects/.../00-governance` | Charter + intake status only | No roles, risks, regulatory or retention statements in source |
| `projects/.../01-business-product` | Populated | Source-backed; no success measures in source |
| `projects/.../02-requirements` | Fully populated | 84 rows + 16 thresholds |
| `projects/.../03-design-architecture` | Routes, exports, touchpoints | Touchpoints now point to the approved architecture; NFR targets still absent |
| `projects/.../04-data-ai` | Conceptual dictionary only | Derived from attribute names in source |
| `projects/.../05-delivery` | Delivery model; R1 harness specification and testing guide | Delivery model proposed (DEC-012); harness approved (DEC-014) and built in the code repository; testing guide approved (DEC-024) |
| `projects/.../07-release-operations`, `09-archive` | Empty | Not in source |
| `projects/.../06-quality-security` | Access-control extract only | Source has RBAC requirements; nothing else security-related |
| `projects/.../08-evidence-decisions` | Populated | Source record, source index, decision log |

Next inputs expected: prototype (if later admitted), owner assignments, a delivery plan, release and operations content, resolution of open questions, Product Owner agreement to DEC-006, DEC-007 and DEC-012, Legal review of the architecture's compliance topics, coding standards, NFR targets.
