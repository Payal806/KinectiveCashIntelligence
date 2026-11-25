---
id: GR-GOV-002
title: Artifact Status Model
artifact_type: governance-model
status: draft
content_state: derived
owner_role: Golden Repo Administrator
source_of_truth: golden-repo
source_refs:
  - "SRC-ASTRA-IMPL-001 §6, §11"
last_updated: 2026-10-28
review_due: TBC
---


# Artifact Status Model

| Status | Meaning |
|---|---|
| `draft` | Written from source; not yet reviewed by an accountable owner. Default for all v0.1 content. |
| `proposed` | Reviewed by author; awaiting owner approval. |
| `approved` | Approved by the named owner role; may be selected as active Astra context. |
| `deprecated` | Superseded; must be removed from active Astra selections and moved to an archive folder. |

Status is recorded in front matter and is a convention only; Astra does not validate it (`SRC-ASTRA-IMPL-001` §14). Label drafts, deprecated material and examples in the document body as well, because Astra retrieves content, not metadata.
