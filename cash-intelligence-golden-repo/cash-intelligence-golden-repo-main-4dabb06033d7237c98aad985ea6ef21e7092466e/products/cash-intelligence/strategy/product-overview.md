---
id: PROD-KIN-CI-001
title: Cash Intelligence — Product Overview
artifact_type: product-overview
status: draft
content_state: source-backed
owner_role: TBC
applies_to:
  - PRJ-KIN-BRAU-CI
source_of_truth: golden-repo
source_refs:
  - "SRC-CONF-CIR-001 §Objective"
  - "SRC-CONF-CIR-001 §Background"
last_updated: 2026-10-28
review_due: TBC
---


# Cash Intelligence — Product Overview

**Source-backed.** The following is what `SRC-CONF-CIR-001` states about the product; nothing further is known from the sources received.

- The **Cash Intelligence Dashboard** is an at-a-glance landing page that summarises cash operations across all views and links to each detailed page.
- The Cash Intelligence prototype (named `cash-analytics`) presents a Dashboard composed of **seven clickable summary widgets**. Each widget summarises a deeper page and is driven by **configurable thresholds defined in Settings**.
- The source page documents the Dashboard requirements and the default threshold settings that drive widget calculations.

The device fleet addressed by the product consists of ATMs, TCRs (with linked Coin Dispensers) and, in page titles, TCR/TCD devices, organised as Region → Branch/Site → Device. See `domain/domain-model.md`.
