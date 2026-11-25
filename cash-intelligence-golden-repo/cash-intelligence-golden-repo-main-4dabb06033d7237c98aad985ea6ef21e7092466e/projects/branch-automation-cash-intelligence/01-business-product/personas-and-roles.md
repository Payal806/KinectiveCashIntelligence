---
id: BUS-KIN-CI-003
title: Personas and Roles
artifact_type: personas
status: draft
content_state: source-backed
owner_role: TBC
applies_to:
  - PRJ-KIN-BRAU-CI
source_of_truth: golden-repo
source_refs:
  - "SRC-CONF-CIR-001 §Roles & Access Control"
  - "SRC-CONF-CIR-001 (user-story actors)"
last_updated: 2026-10-28
review_due: TBC
---


# Personas and Roles

## Access-control roles (source-backed, `REQ-CI-ROLE-001`–`004`)

| Role | Access | Scoping |
|---|---|---|
| **Admin** | All analytics pages ("My institution" navigation) and the Admin section including Settings. | Not restricted by branch/site; not model-scoped. |
| **User** | All analytics pages. Cannot access the Admin section or Settings. | Analytics scoped to assigned branches/sites; not model-scoped. |
| **Service Provider** | Only the TCR/TCD Maintenance Status and TCR/TCD Usage and Health pages; all other pages and the Admin section are hidden; any other route redirects to TCR/TCD Maintenance Status. | Sees only devices whose model is in their serviced models (provided by the host system). No serviced models → no devices (fail-closed). |

## Actor wording used in user stories (source-backed)

The stories use these actor labels; the source does not map them to the three roles above except where noted:

- "cash operations user" (Dashboard layout)
- "user" / "signed-in user" (most pages; Profile)
- "service coordinator" (TCR Device Health summary, maintenance alerting)
- "analyst" (Overstock Analysis, TCR Usage Health)
- "admin" (all Settings stories)
- "system" (classification and flagging logic stories)

Whether "analyst", "service coordinator" and "cash operations user" are distinct roles or descriptive labels for Admin/User is not stated (OQ-012).

## Profile attributes (source-backed, `REQ-CI-PROF-001`)

A user has: email (used for sign-in lookup, case-insensitive), name, avatar initials, phone, role, and Branches/Sites — either "All branches / sites" when the user has *enterprise access*, or a list of branches each with its region. "Enterprise access" is not defined (OQ-007).
