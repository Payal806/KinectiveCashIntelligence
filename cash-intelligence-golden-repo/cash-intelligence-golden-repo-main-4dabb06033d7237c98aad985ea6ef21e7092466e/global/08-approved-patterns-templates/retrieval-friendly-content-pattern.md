---
id: GR-PAT-001
title: Retrieval-Friendly Content Pattern
artifact_type: content-pattern
status: draft
content_state: derived
owner_role: Golden Repo Administrator
source_of_truth: golden-repo
source_refs:
  - "SRC-ASTRA-IMPL-001 §6-7"
last_updated: 2026-10-28
review_due: TBC
---


# Retrieval-Friendly Content Pattern

- One topic per file; prefer focused documents over large mixed-purpose ones.
- Open with the source reference and content state so a retrieved chunk is self-describing.
- Write explicit requirements, rules, examples and exceptions; do not rely on folder names to carry meaning.
- Use stable IDs for requirements, rules, open questions and decisions.
- Supported formats: `.md`, `.txt`, `.yaml`/`.yml`, `.pdf`, `.docx`; files ≤ 5 MB; no images or binaries as retrievable guidance.
- Keep filenames and paths descriptive; Astra uses paths for selection, warnings and attribution.
