---
id: GR-KIN-CI-README
title: Kinective Cash Intelligence Golden Repository v0.1 (initial draft)
artifact_type: golden-repo-readme
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


# Kinective Cash Intelligence — Golden Repository v0.1 (initial draft)

This repository is the **initial draft** context package for the Kinective **Cash Intelligence** initiative (Branch Automation). It follows the Astra Golden Repository guideline (`SRC-ASTRA-IMPL-001`) scaffold and is intended for Astra retrieval across BRD, Workflow, Specs, QE and related SDLC generation flows.

## What this draft contains, and what it does not

**Contains:** everything stated in the Confluence page *Cash Intelligence Requirements* (`SRC-CONF-CIR-001`, page version 30, last modified 2026-10-19). That page provides 84 requirement rows (user story + acceptance criteria) across 13 functional areas, plus a table of 16 default threshold settings.

**Does not contain:**

- The Cash Intelligence prototype (`cash-analytics`) — explicitly excluded from this draft.
- The published Prism packages as the build's UI source: until the DEC-023 switch the frontend builds on the Prism copy vendored in the Crossroads micro-frontend template. (The Prism 2.0 design-system guidance itself is included, see below.)
- Any non-functional requirement target, delivery plan, release or operations content, risk register, success metric or regulatory statement, nor a test strategy beyond how specifications and test cases are verified in the R1 harness (DEC-024). None of these are in the source, so none have been written. The corresponding folders are present but empty (`.gitkeep`). The one exception is `projects/branch-automation-cash-intelligence/05-delivery/`: the delivery model (DEC-012), which sets how requirements, BRDs, backlog and specs relate, and the R1 harness specification (DEC-013, DEC-014), from which the harness is built directly. Neither is a delivery plan.

**Added 2026-11-19 (DEC-017, DEC-018, proposed):** the Prism 2.0 design-system guidance under `global/03-engineering-architecture/prism-*.md`, the build-stage pattern and the screen map (`03-design-architecture/prism-ui-layer-map.md`). **Added 2026-11-20 (DEC-024):** the harness testing guide, the spec Verification and test case templates, and the Test Harness Context Pack (`docs/test-harness-context-pack.md`).

**Added 2026-11-03 (DEC-006), approved 2026-11-05 as the R1 design baseline:** the Cash Intelligence architecture — technology and platform, data model, integrations, identity and access, tenancy, deployment and scaling, and the data lineage register — under `products/cash-intelligence/architecture/`, `data/` and `integrations/`, plus the Crossroads Foundation rules every ABS follows under `global/03-engineering-architecture/`. Start at `products/cash-intelligence/architecture/architecture-overview.md`; the selection guide is `docs/architecture-context-pack.md`.

Nothing in this repository has been invented. Architecture content is *approved-design*: authored by the Cash Intelligence architecture owner, traced to the registered sources, and approved by that owner as the R1 design baseline. How requirements, BRDs, backlog and specs relate is in `projects/branch-automation-cash-intelligence/05-delivery/delivery-model-and-gates.md` (DEC-012, proposed). Where the source is silent, ambiguous or internally inconsistent, the gap is recorded as an open question in `projects/branch-automation-cash-intelligence/02-requirements/assumptions-dependencies-open-questions.md` rather than resolved.

## Authority model

1. `SRC-CONF-CIR-001` — Confluence *Cash Intelligence Requirements*: the only authoritative Kinective product/business source in this draft.
2. `SRC-ASTRA-IMPL-001` — Astra Golden Repository Guideline: governs repository organisation and Astra usage, not Cash Intelligence facts.
3. `SRC-REF-GR-001` — Northline Grocers reference Golden Repo: structural pattern reference only. No Northline facts, requirements, systems, risks or controls have been transferred.
4. `SRC-XR-TDD-001` — Crossroads Foundation TDD v0.5: authoritative for Crossroads platform rules only.
5. `SRC-JIRA-BAP-001`, `SRC-CONF-PRD-001` — Jira R1 stories and the PRD: secondary product sources; `SRC-CONF-CIR-001` wins where they differ.
6. `SRC-CI-RES-001` — the architecture research and verification notes, held outside this repository by the architecture owner.

## Related repositories

- Code: `cash-intelligence-code-repo` (https://gitlab.com/kinective.io/internal/abs/cash-intelligence/cash-intelligence-code-repo) — services, CO connector, frontend, Flyway migrations, `deploy/` and the local development environment (DEC-011; layout in `products/cash-intelligence/architecture/repositories-and-local-development.md`). Canonical requirements, architecture and decisions stay in this repository.

## Content-state vocabulary

- **Source-backed** — explicitly stated in a registered authoritative source, within that source's scope (`SRC-CONF-CIR-001` for the initiative; see `docs/source-authority-and-provenance.md`).
- **Derived** — a conservative reorganisation of source-backed content (e.g. a glossary, a route list) that adds no new requirement.
- **Proposed** — a recommendation that needs owner approval; used for the delivery model (DEC-012).
- **Approved design** — authored and approved by an owner role; the design baseline generated output follows. Used for the architecture content.
- **Open Question** — information missing or inconsistent in the source; must be resolved by an accountable Kinective stakeholder.

No *Assumption* content has been written beyond the minimal planning assumptions listed in the open-questions file.

## Repository organisation

- `global/` — reusable Golden Repo operating guidance drawn from the Astra guideline, and the Crossroads Foundation obligations for any ABS. No customer facts.
- `products/cash-intelligence/` — reusable product context: what the product is, its pages, its domain model, and the approved architecture (`architecture/`, `data/`, `integrations/`).
- `projects/branch-automation-cash-intelligence/` — the requirement baseline, business rules, personas, glossary, data dictionary, open questions, source records and decision log.
- `shared/` — templates, controlled vocabularies and the metadata convention.
- `docs/` — provenance, selection guidance and the file inventory. Do not select these as business context.

## Current Astra implementation boundary

Folder names, manifests, YAML front matter and `astra.yaml` are organisational conventions. Astra consumes selected and vectorised file content and paths; it does not execute this governance schema (`SRC-ASTRA-IMPL-001` §14). Project administrators must explicitly select and vectorise the intended files.

## Review and approval

Each file's front matter gives its status. The requirement and business files are `draft` with owner role `TBC`, because the source names none and the page owner has not confirmed version 30 as the baseline (DEP-004); the architecture, the harness files and the testing guide are approved by the architecture owner (DEC-006, DEC-014, DEC-024); the delivery model and several decisions are still proposed (see the decision log). Approval by the architecture owner is a design approval, not a legal one: the architecture files marked "Legal review required" have not yet been reviewed by Kinective's Legal team (DEP-010), and neither they nor any legal, compliance or regulatory content added later may be relied upon until that review is done.
