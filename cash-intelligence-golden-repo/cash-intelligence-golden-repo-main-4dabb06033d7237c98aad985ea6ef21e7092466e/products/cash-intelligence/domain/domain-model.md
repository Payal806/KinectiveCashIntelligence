---
id: PROD-KIN-CI-003
title: Domain Model
artifact_type: domain-model
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


# Domain Model

**Derived** by collecting the entities, hierarchy and relationships named in `SRC-CONF-CIR-001`. No entity or relationship has been added beyond what the source names. Attribute-level detail is in `projects/.../04-data-ai/conceptual-data-dictionary.md`.

## Organisational hierarchy

`Region` → `Branch` (also written "Branch/Site" or "branches/sites") → `Device`. Drill-down, breadcrumbs and count chips follow this hierarchy throughout the source. The device level does not drill down further.

## Device types named in the source

| Type | How the source uses it |
|---|---|
| ATM | Has cash inventory, bill acceptor, replenish orders and an optional ATM replenish order schedule; counted in transactions (deposits, dispenses). |
| TCR | Has cash inventory, notes moved, uptime/downtime, errors (with severity and error code), left/right/access usage, model, "Connected to Hub", communication mode; may have a linked Coin Dispenser. |
| TCD | Appears only in page titles ("TCR/TCD Maintenance Status", "TCR/TCD Usage and Health") and one export filename. Not defined in the source (OQ-003). |
| Coin Dispenser | Never shown as a standalone entry; its inventory, EOD total and denominations are rolled into its linked TCR, whose label is suffixed "(+ coins)". Excluded from Overstock Analysis and from the Device Usage widget. |

## Cash and denominations

- Denominations supported: notes $100, $50, $20, $10, $5, $1; coins $0.25, $0.10, $0.05, $0.01 (`REQ-CI-CDI-003`).
- Measures named: cash on hand / current inventory, end-of-day (EOD) total, start-of-day (SOD), dispensed, deposited, net, amount replenished, cash order amount, predicted usage, average inventory, average discrepancy, cash share of branch cash.
- Currency is formatted as USD (`REQ-CI-DASH-003`, `REQ-CI-CDI-001`, others).

## Configuration entities (Settings)

Downtime threshold; Preventative Maintenance Threshold Rules (with alert configuration); Proactive Service Threshold Rules; Overstock Thresholds; Acceptable Discrepancy Rules; Cash Order Thresholds; Vault Minimum Rules; ATM Replenish Order Schedules; ATM Replenish History Thresholds. Each "rule" or "schedule" has a name and assigned devices or branches. See `req-settings.md`.

## Operational records named

Cash order (scheduled / unscheduled); service record (maintenance type, service date, technician name, notes, uploaded file); maintenance alert (initial and recurring reminder); error occurrence (message, date/time, downtime, severity, code); branch vault (total, minimum); user (email, name, phone, role, assigned branches/sites, serviced device models for Service Providers).
