---
id: BUS-KIN-CI-002
title: Scope and Boundaries
artifact_type: scope-statement
status: draft
content_state: derived
owner_role: TBC
applies_to:
  - PRJ-KIN-BRAU-CI
source_of_truth: golden-repo
source_refs:
  - "SRC-CONF-CIR-001"
last_updated: 2026-10-28
review_due: TBC
---


# Scope and Boundaries

## In scope (source-backed — the functional areas documented in `SRC-CONF-CIR-001`)

1. Roles & Access Control (Admin, User, Service Provider; device-model scoping)
2. Dashboard (seven widgets, navigation)
3. Current Device Inventory
4. Historic EOD Inventory (incl. CSV export and branded audit report)
5. ATM Cash Replenish History
6. ATM Usage (incl. geographic heat map and transactions CSV)
7. Cash Order Management (incl. order recommendation and CSV export)
8. TCR Device Health / "TCR/TCD Maintenance Status" (incl. service records and email alerting)
9. Overstock Analysis
10. Branch Cash Positioning (incl. rebalancing recommendation)
11. TCR Usage Health / "TCR/TCD Usage and Health" (incl. two CSV exports)
12. Settings (nine threshold / rule / schedule groups)
13. Profile (read-only)

## Non-goals and exclusions

The source states **no explicit non-goals**. The only exclusions are repository-level and by instruction: the prototype is not represented in this draft; Prism 2.0 is, as the UI layer (DEC-017, proposed; see `00-governance/source-intake-status.md`).

## Boundaries not defined by the source

Non-functional requirements, technology stack, hosting, data sources and ingestion, authentication mechanism, audit logging, localisation, accessibility, and rollout are not addressed in the source. Do not treat their absence as a decision.
