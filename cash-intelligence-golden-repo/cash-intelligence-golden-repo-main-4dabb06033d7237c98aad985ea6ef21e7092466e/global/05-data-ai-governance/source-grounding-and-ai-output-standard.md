---
id: GR-AI-001
title: Source Grounding and AI Output Standard
artifact_type: ai-governance-standard
status: draft
content_state: derived
owner_role: Golden Repo Administrator
source_of_truth: golden-repo
source_refs:
  - "SRC-ASTRA-IMPL-001 §6, §10, §14"
last_updated: 2026-10-28
review_due: TBC
---


# Source Grounding and AI Output Standard

1. Generation flows must be grounded in selected, vectorised Golden Repo files; verify selection and vectorisation before running a flow.
2. Generated artifacts must cite the Golden Repo file path and, where available, the requirement ID (`REQ-…`) they rely on.
3. Where the Golden Repo records an *Open Question*, generated output must carry the question forward, not answer it.
4. Astra provides no conflict resolution across contradictory files; content owners must keep active guidance non-contradictory.
5. Review skipped-file and missing-file warnings and re-run chunking when source content changes.
