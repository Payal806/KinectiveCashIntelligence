---
id: ARC-KIN-CI-104
title: Cash Intelligence architecture — 13. Cross-cutting: audit and security
artifact_type: architecture
status: approved
content_state: approved-design
owner_role: Cash Intelligence architecture owner
applies_to:
  - PRJ-KIN-BRAU-CI
source_of_truth: golden-repo
source_refs:
  - "SRC-CONF-CIR-001"
  - "SRC-JIRA-BAP-001"
  - "SRC-XR-TDD-001"
  - "SRC-CI-RES-001"
last_updated: 2026-11-22
review_due: TBC
---

# 13. Cross-cutting: audit and security

> **Approved architecture: the R1 design baseline (DEC-006).** Approved by the Cash Intelligence architecture owner on 2026-11-05, after the 2026-11-03 design review. Generated specs follow it; a spec that needs something different is a change here first (GOV-CI-004 in `projects/branch-automation-cash-intelligence/05-delivery/delivery-model-and-gates.md`). Open questions (§16) stay open until answered. Section numbers (§n), open questions (Qn) and decisions (A–I) use the architecture's own numbering; `products/cash-intelligence/architecture/architecture-overview.md` maps each section to its file, and the open questions are in `products/cash-intelligence/architecture/open-questions.md`. Citations: `BAP-nn` = `SRC-JIRA-BAP-001`, `TDD:n` = line n of `SRC-XR-TDD-001`, `research/nn` = `SRC-CI-RES-001` (see the source index).
>
> **Legal review required.** This file covers audit, security controls and staff personal data. These touch compliance: Kinective's Legal team must review them before they are relied upon or selected as Astra context.

```sql
audit.audit_log (audit_id BIGINT AUTO_INCREMENT PK, tenant_id, occurred_at DATETIME(3) NOT NULL,
                 actor_type VARCHAR(8), actor_user_id INT NULL, action VARCHAR(48),
                 entity_type VARCHAR(48), entity_id VARCHAR(64), detail_json TEXT NULL, -- JSON_VALID
                 prev_hash BINARY(32), row_hash BINARY(32))                  -- per-tenant hash chain
audit.chain_head (tenant_id PK, last_audit_id BIGINT, last_hash BINARY(32))   -- locked FOR UPDATE by the insert trigger
```

- Rules, rule assignments, recipients, schedules, CI roles, service records, devices and locations are
  **versioned** (history tables, §15); settings and rule parameters are versioned and audited by the Platform Config
  Service (`CONFIG_CHANGED`, with actor and diff). `audit_log` records who and why: every admin write records before/after JSON in its own transaction,
  plus session creation and token-validation failures (§11) and sensitive exports (`audit_log_export`; audit report
  BAP-35). Application logs carry the KBOP user id and correlation id, never email or tokens. It is append-only: the app account has INSERT only,
  `BEFORE UPDATE/DELETE` triggers reject changes (offboarding removes the rows by dropping the whole database after archiving them, §14.1), and each row carries `row_hash` = SHA-256 of `prev_hash` followed by the row's canonical form, so tampering
  is detectable (MySQL has no ledger tables). The canonical form is defined here, not by any database's JSON output,
  so a verifier in any language recomputes it (DEC-028): the fields `tenant_id`, `audit_id`, `occurred_at`,
  `actor_type`, `actor_user_id`, `action`, `entity_type`, `entity_id`, `detail_json` in that order, each written as
  its UTF-8 length in bytes (decimal), a colon and its UTF-8 bytes, or as `~` when NULL; integers in decimal;
  `occurred_at` as UTC `YYYY-MM-DDTHH:MM:SS.mmmZ`. `detail_json` is the exact text the service wrote (canonical JSON:
  object keys sorted, no insignificant whitespace), kept as text with a `JSON_VALID` check, so the stored bytes are
  the hashed bytes. The insert trigger allocates `audit_id` and `occurred_at` and computes the hash, so no caller
  chooses them; the verifier is service code shared by the maintenance job and the archive. The maintenance job also
  anchors each tenant's chain head (last `audit_id` and hash) in the tenant's locked archive every day, so the log and
  its head cannot be rewritten together unnoticed (§16 Q66).
  `prev_hash` is never NULL: a tenant's first row uses 32 zero bytes, or the archived head hash after re-onboarding. The chain is per tenant; `audit.chain_head` serializes inserts and holds
  the head hash. Offboarding and re-onboarding preserve the chain through the audit archive (§14.1).
- **Append-only rests on privileges too.** `TRUNCATE` bypasses the rejecting triggers (`research/80` §6 test 16), so no
  application, ETL or job account has DROP or TRUNCATE on any table (TRUNCATE needs the DROP privilege). Partition
  maintenance (create and drop month partitions, §14) runs under a separate maintenance account with no API or feed
  use. That account holds the privileges `ALTER TABLE` needs for it (ALTER, CREATE, INSERT, and DROP for a partition
  drop) on the partitioned fact and history tables only, never on `audit_log` or `audit.chain_head`, and retention
  never drops `audit_log` partitions. The maintenance account also inserts its own `system` rows into `audit_log`
  (offboarding, a re-onboarded chain's genesis), which the triggers stamp and hash like any other. Legal review: the
  audit guarantee depends on these grants.
- Scope per §11; InnoDB data-at-rest encryption (or the managed service's); TLS in transit; no secrets in this schema.
- **Operational access.** In production nobody queries the databases directly, read access included: people use the
  product, and services use their own credentials (§14.1). An emergency uses the hosting provider's break-glass role (an AWS IAM
  role on AWS, the provider's equivalent if §16 Q21 chooses another), whose use is time-limited, logged and reviewed. DevOps owns and operates the databases until a DBA role exists. With AWS hosting, data stays in the US (§14.1 Disaster recovery);
  another provider must restate residency before go-live (§16 Q21, Q58). Legal review: client due-diligence answers rely on these statements.
- **Attachment rule.** Service-record attachments accept only the types the requirement lists (`REQ-CI-TDH-005`) (PDF, Word
  DOC and DOCX, JPEG, PNG) up to a size limit, are malware-scanned before anyone can read them, and the upload page tells
  the user not to attach check images or card data. Whether legacy DOC (binary, macro-capable) stays accepted is for
  the security review (§16 Q59). A file found to hold such data is deleted with an audit row. How "before
  anyone can read them" is built: the upload is written to the tenant bucket under a `pending/` prefix that no
  download path reads, with `scanStatus = pending` (as the contract returns); the scanner (the hosting provider's
  object-storage malware scanning, or a scanner job in CI's namespace, decided with §16 Q21) moves a clean file to
  its final key and sets `clean`, and deletes an infected one, setting `rejected` (the contract's terminal value) with
  an audit row. Downloads answer
  409 `scan_pending` until the file is `clean`. The size limit is part of §16 Q59. CI cannot inspect every image, so the
  rule limits rather than guarantees; the scope statement (§1) is about what CI ingests.
- **Archive.** The audit archive (§14.1) and service-record files are in the tenant's own S3 bucket, with its own KMS
  key; the archive uses Object Lock, and a legal hold on the tenant is an Object Lock legal hold on its bucket.
- **Never ingested:** Link `cfm_da_*`, credential tables (`cfm_credstore`, `cfm_credkey`, `cfm_key_control`,
  `cfm_email_setting`, `cfm_user.password`), free-text `cfm_journal.comments`/`tag`, `cfm_device_service_memo`; CO
  cardholder data; iQ service-record technician names. **EJ**: only Id, TenantId, TerminalId, TransactionTimeLocal,
  TransactionTime, TxnType, TxnName, TxnStatus, Result, TxnAmount, TxnCurrency and Ej_File_Date are read, plus a
  check-deposit flag computed in the source query (`CheckId IS NOT NULL`). Never read: Pan, every `Emv_*` column,
  AccountType, FromAccount, ToAccount, FromAccountType, ToAccountType, HostData, Ej_Data_Chunk (raw journal text, may
  hold PANs), CashDetails, CodeLine, CheckMICR, CheckNumber, CheckDepositDetails, TraceNumber, LinkingKey,
  ConsumerMessageId, the preference columns, and PanBinId/PanAccountId where present; also the `EJ_PAN_ACCOUNT` and
  `EJ_PAN_BIN` tables. **iQ** (Mongo `CFM`): only the `s4_sync_cfm_*` collections named in §5.2 are read, and within
  them the per-table column allowlist of §5.2; never `s4_sync_cfm_user`, `_role`, `_user_activity`,
  `_email_setting`, `_teller*`, `s4_raw_data` or `s4_sync_staging`. Enforced by
  `field_allowlist_json` and the onboarding schema check; reader accounts are not granted these tables or columns.
- **Defect to raise with the Link/S4 and iQ owners:** `cfm_email_setting`, a credential table, is change-tracked and
  sent to iQ by the existing S4 sync (raise it with the reference-table sync request, §4); the iQ test database holds copies of it (`s4_sync_cfm_email_setting`,
  `research/91`). Stopping the sync does not remove the copies; rotation and purge are open (§16 Q46).

## 13.1 Staff personal data

CI holds no end-customer personal data (P10). It does hold **staff** personal data, in the tenant's own database and
bucket only:

| Data | Where | Why | Not sent to |
|---|---|---|---|
| User email, display name, optional phone | `acc.app_user` | Profile page; login mapping (cache of KBOP claims) | Logs (KBOP user id and correlation id only), exports |
| Alert recipient email addresses | `cfg.rule_alert_recipient` | Maintenance alerts (requirement) | The Platform Config Service, logs |
| Technician name, service notes | `ops.service_record` | Service records (requirement) | Exports and AI prompts by default; iQ seeding (P10) |
| Uploaded service files | Tenant S3 bucket; metadata in `ops.service_record_attachment` | Service records | — |
| Actor ids and before/after JSON | `audit.audit_log` | Audit | — |

None of it goes onto Crossroads: CI publishes no events and runs no journeys in R1, and the platform configuration
holds only thresholds, cadences and amounts. The alert email is where this data leaves Kinective. Retention and purge
are per tenant: dropping the tenant's database and emptying its bucket removes it, except the audit archive, which is
kept for its retention under Object Lock (§14.1). CI's ABS brief for Crossroads says
"holds staff personal data in its own store; none on the platform", not "PII-free". Kinective's Legal team should
review this section before it is relied on.
