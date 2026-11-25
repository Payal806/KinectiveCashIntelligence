---
id: DOC-KIN-CI-TEST-PACK
title: Test Harness Context Pack
artifact_type: selection-guide
status: draft
content_state: derived
owner_role: Cash Intelligence architecture owner
applies_to:
  - PRJ-KIN-BRAU-CI
source_of_truth: golden-repo
source_refs:
  - "projects/branch-automation-cash-intelligence/05-delivery/harness/testing-with-the-harness.md"
last_updated: 2026-11-20
review_due: TBC
---


# Test Harness Context Pack

Select this pack, with the Architecture Context Pack (`docs/architecture-context-pack.md`), when a flow produces
technical specifications, test cases or work items with acceptance criteria. It tells the flow how each criterion is
verified in the R1 harness: the test levels, the seeded users and data, the data a test makes and the simulator behaviours to
name, and the sections a specification and a test case carry for them. The harness files are approved (DEC-014), and so
are the guide and the templates in this pack (DEC-024).

`astra.yaml` lists the pack (`test_harness_context_pack`) as a convention only: attach these files to the flow's stage
explicitly, and record what was attached in the stage's trace.

| # | File | Why required |
|---|---|---|
| 1 | `projects/branch-automation-cash-intelligence/05-delivery/harness/testing-with-the-harness.md` | The test levels, seeded users, seed sets, test data and simulator manifests a spec or test case names (approved, DEC-024) |
| 2 | `projects/branch-automation-cash-intelligence/05-delivery/harness/harness-intent.md` | What the harness is and its acceptance (approved, DEC-014) |
| 3 | `projects/branch-automation-cash-intelligence/05-delivery/harness/seed-data.md` | The seed sets a test runs on (a rule or flag's test makes its own cases on them) (approved) |
| 4 | `projects/branch-automation-cash-intelligence/05-delivery/harness/source-shapes.md` | The source shapes and simulator behaviours an ingestion criterion is tested with (approved) |
| 5 | `products/cash-intelligence/architecture/repositories-and-local-development.md` | Where code and tests live in the code repository, and the local stand-ins (§14.3) |
| 6 | `shared/templates/spec-verification-template.md` | The `## Verification` section every specification carries (approved, DEC-024) |
| 7 | `shared/templates/test-case-template.md` | The test case fields: level, data, signs in as, automated in (approved, DEC-024) |

Before generation, verify every selected file is vectorised and not stale (`SRC-ASTRA-IMPL-001` §10, §12).
