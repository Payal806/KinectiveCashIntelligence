---
id: GR-SEC-001
title: Context Handling Baseline
artifact_type: security-baseline
status: draft
content_state: derived
owner_role: Security Lead (TBC)
source_of_truth: golden-repo
source_refs:
  - "SRC-ASTRA-IMPL-001 §4, §6, §8"
last_updated: 2026-10-28
review_due: TBC
---


# Context Handling Baseline

- Never store passwords, PATs, API keys, private keys, certificates, connection strings or session tokens in this repository. Provider credentials are entered only through Astra source configuration.
- Never store personal data of financial-institution end customers or members (names, account numbers, SSNs or any other PII), uncontrolled production data, or customer data outside its approved boundary. Field *names* such as "Technician Name" or "Recipient emails" may be documented; field *values* may not.
- Use a separate repository where customer, tenant, legal-entity, data-residency or jurisdictional separation is required. A top-level folder is not a security boundary.
- Use provider-side repository permissions, branch protection, required pull requests and CODEOWNERS to control change. Astra does not enforce these.
- Any legal, compliance or regulatory guidance added to this repository must be reviewed by Kinective's Legal team before it is relied upon or selected as Astra context.
