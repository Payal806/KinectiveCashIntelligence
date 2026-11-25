---
id: GR-GOV-001
title: Golden Repo Charter
artifact_type: governance-charter
status: draft
content_state: derived
owner_role: Golden Repo Administrator
applies_to:
  - PRJ-KIN-BRAU-CI
source_of_truth: golden-repo
source_refs:
  - "SRC-ASTRA-IMPL-001 §1-6, §14"
last_updated: 2026-10-28
review_due: TBC
---


# Golden Repo Charter

## Purpose

Maintain reviewed, retrievable context so that Astra and delivery teams generate consistent SDLC artifacts without reconstructing project knowledge from chat, memory or unsupported assumptions.

## Rules

1. Customer- and product-specific facts must trace to a registered source in the project's `source-index.md`.
2. Unknowns remain *Open Question*; they are never silently promoted to requirements.
3. All content is `draft` until an accountable owner approves it.
4. Secrets, credentials, uncontrolled production data and personal data are prohibited (see `global/04-security-privacy-regulatory/context-handling-baseline.md`).
5. Active guidance is focused, retrieval-friendly, non-duplicative and non-contradictory.
6. Paths stay stable once selected in Astra.
7. Structural patterns may be learned from reference repositories; reference-customer facts never transfer.

## Escalation

Conflicting sources, missing approvals or ambiguous requirements are surfaced through the open-questions register and decision log before downstream artifacts are treated as controlled baselines.
