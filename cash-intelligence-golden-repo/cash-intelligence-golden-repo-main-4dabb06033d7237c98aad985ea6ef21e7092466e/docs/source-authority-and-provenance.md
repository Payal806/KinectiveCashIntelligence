---
id: DOC-KIN-CI-PROV
title: Source Authority and Provenance Notes
artifact_type: generation-notes
status: draft
content_state: derived
owner_role: Golden Repo Administrator
applies_to:
  - PRJ-KIN-BRAU-CI
source_of_truth: golden-repo
source_refs:
  - "SRC-CONF-CIR-001"
  - "SRC-ASTRA-IMPL-001"
  - "SRC-REF-GR-001"
last_updated: 2026-11-03
review_due: TBC
---


# Source Authority and Provenance Notes

- All Cash Intelligence facts in this repository were taken from `SRC-CONF-CIR-001` (Confluence page ID 2473689093, space BrAu, version 30, last modified 2026-10-19, read on 2026-10-28). The page body is retained verbatim in `projects/.../08-evidence-decisions/source-records/cash-intelligence-requirements-confluence-v30.md`.
- Requirement files under `02-requirements/` were generated mechanically from that source record: requirement name, user story and acceptance criteria are copied unchanged; only repository IDs (`REQ-CI-…`) and list formatting were added. The source's `Work Items` column is empty for every row and was omitted.
- The Astra guideline (`SRC-ASTRA-IMPL-001`) was used for repository structure, content rules and the global operating guidance. It is not a source of Cash Intelligence facts.
- The Northline Grocers reference repo (`SRC-REF-GR-001`) was used only for structural patterns (front matter, source index, open-questions register, context pack). No Northline content was transferred.
- The Cash Intelligence prototype was not consulted (DEC-002). Prism 2.0 (`SRC-PRISM-UI-001`) was ingested on 2026-11-19 as the UI layer (DEC-017, proposed); the build's vendored Prism copy reaches CI only through the Crossroads template (DEC-023). See `repo-manifest.yaml` and the decision log.
- No page or paragraph numbers were fabricated.
- Added 2026-11-03 (DEC-006, DEC-007): the architecture files were split from the architecture owner's design document, one topic per file, keeping its section numbers and open-question IDs. Platform rules come from `SRC-XR-TDD-001` at commit 8c55357, verified line by line. Source-system facts were verified in code and are recorded in `SRC-CI-RES-001`, which is not in this repository. Statements that relied on the excluded prototype were restated (DEC-008). Source citations use the Confluence section headings (e.g. `§Cash Order Management`).
