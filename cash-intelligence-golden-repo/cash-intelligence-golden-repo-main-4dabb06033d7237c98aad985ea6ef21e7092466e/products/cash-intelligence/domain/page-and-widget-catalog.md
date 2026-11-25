---
id: PROD-KIN-CI-002
title: Page and Widget Catalog
artifact_type: domain-catalog
status: draft
content_state: derived
owner_role: TBC
applies_to:
  - PRJ-KIN-BRAU-CI
source_of_truth: golden-repo
source_refs:
  - "SRC-CONF-CIR-001 §Dashboard"
  - "SRC-CONF-CIR-001 (all page sections)"
last_updated: 2026-10-28
review_due: TBC
---


# Page and Widget Catalog

**Derived** from the section headings and the Dashboard requirements in `SRC-CONF-CIR-001`. This is a navigation aid; the requirements themselves are in `projects/.../02-requirements/`.

## Dashboard widgets (seven, per `REQ-CI-DASH-001`/`-002`)

| Widget (as named in source) | Route | Detail page section in source | Device scope stated |
|---|---|---|---|
| Current device inventory | `/inventory` | Current Device Inventory | ATMs and TCRs; Coin Dispensers rolled into linked TCR |
| Historical EOD inventory | `/historical` | Historic EOD Inventory | ATMs and TCRs; Coin Dispensers merged into linked TCR |
| ATM cash replenish history | `/replenish-history` | ATM Cash Replenish History | ATM devices only |
| Cash order management | `/forecasting` | Cash Order Management | ATMs and TCRs |
| Overstock analysis | `/overstock` | Overstock Analysis | ATMs and TCRs; Coin Dispensers excluded |
| TCR device health | `/health` | TCR Device Health (page title "TCR/TCD Maintenance Status") | TCR devices only |
| Device usage | `/usage` | ATM Usage | Widget: non-coin-dispenser devices; page: ATM devices only (see OQ-003) |

## Pages without a Dashboard widget or stated route

| Page section in source | Device scope stated | Notes |
|---|---|---|
| Branch Cash Positioning | Devices not on a scheduled order; overstocked TCRs | No route stated (OQ-004) |
| TCR Usage Health (page "TCR/TCD Usage and Health") | TCR devices only | No route stated (OQ-004); Service Provider accessible |
| Settings | — | Admin section; no route stated |
| Profile | — | Read-only; no route stated |
| Roles & Access Control | — | Cross-cutting, not a page |
