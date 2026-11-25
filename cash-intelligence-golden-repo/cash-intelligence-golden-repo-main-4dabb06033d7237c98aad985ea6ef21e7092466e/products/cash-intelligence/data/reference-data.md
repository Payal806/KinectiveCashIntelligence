---
id: DATA-KIN-CI-003
title: Cash Intelligence architecture — 6. Reference data
artifact_type: data-architecture
status: approved
content_state: approved-design
owner_role: Cash Intelligence architecture owner
applies_to:
  - PRJ-KIN-BRAU-CI
source_of_truth: golden-repo
source_refs:
  - "SRC-CONF-CIR-001"
  - "SRC-JIRA-BAP-001"
  - "SRC-CONF-PRD-001"
  - "SRC-XR-TDD-001"
  - "SRC-CI-RES-001"
last_updated: 2026-11-21
review_due: TBC
---

# 6. Reference data

> **Approved architecture: the R1 design baseline (DEC-006).** Approved by the Cash Intelligence architecture owner on 2026-11-05, after the 2026-11-03 design review. Generated specs follow it; a spec that needs something different is a change here first (GOV-CI-004 in `projects/branch-automation-cash-intelligence/05-delivery/delivery-model-and-gates.md`). Open questions (§16) stay open until answered. Section numbers (§n), open questions (Qn) and decisions (A–I) use the architecture's own numbering; `products/cash-intelligence/architecture/architecture-overview.md` maps each section to its file, and the open questions are in `products/cash-intelligence/architecture/open-questions.md`. Citations: `BAP-nn` = `SRC-JIRA-BAP-001`, `TDD:n` = line n of `SRC-XR-TDD-001`, `research/nn` = `SRC-CI-RES-001` (see the source index).

```sql
ref.tenant              (tenant_id INT PK, display_name VARCHAR(100),   -- used in export file names (PRD §5.3)
                         default_timezone VARCHAR(64), default_currency CHAR(3) DEFAULT 'USD',
                         cm_cash_mode VARCHAR(10) NULL,               -- amount | bill_count; CO tenants only (§5.2)
                         legal_hold BOOLEAN DEFAULT 0,                -- suspends purge, partition drops, offboarding (§14.1)
                         status, created_at)
ref.tenant_external_key (tenant_id, key_type,                         -- institution_id (Crossroads) | kinective_id (= KBOP FI entity_id)
                                                                       -- | link_client | link_client_uuid | cm_tenant | esq_rbac_tenant | iq_client
                         source_instance_id INT NULL,                  -- required for instance-scoped keys (link_client, cm_tenant)
                         external_key VARCHAR(128) COLLATE utf8mb4_bin, -- exact match, as in ci_catalog
                         instance_scope GENERATED ALWAYS AS (IFNULL(source_instance_id, 0)) STORED,   -- 0 = global key (§15)
                         UNIQUE (key_type, instance_scope, external_key))
```

`cfm_client.id` restarts in every Link database and CO's `TenantId` defaults to 100 everywhere, so those keys are
unique only within a source instance. Link's `cfmuuid` is globally unique but is **not synced to iQ** (S4DataSync's
client and device models carry no such column), so it is unavailable on the iQ path. iQ has **no immutable client
id**: S4 sends `cfm_client.name` and iQ keys every document by (client name, id). The name is stored only as an
`iq_client` external key, never displayed, exported or written to logs. A client rename in Link re-keys every iQ
document, so the freshness monitor alarms on rows from an unknown client name and an admin re-maps it (§16 Q3).

This table is the tenant's own copy of its keys. The keys that route rows or requests before a tenant database is
chosen (`iq_client`, `cm_tenant`, `kinective_id`, `institution_id`) are mastered in the control-plane catalogue
(`ci_catalog.tenant_key`, §14.1), which enforces uniqueness across tenants; the two are kept identical by the
onboarding step.

## Location hierarchy

```sql
ref.location            (location_id INT PK, tenant_id,
                         location_type VARCHAR(12),                    -- region | branch | site  (R3: vault | drawer)
                         parent_location_id NULL,                      -- branch/site → region
                         name VARCHAR(255), code VARCHAR(32) NULL,     -- code: admin-entered (no source)
                         timezone_id VARCHAR(64) NULL,                 -- IANA; required for branch/site unless utc_offset set
                         utc_offset_minutes SMALLINT NULL,             -- CO-only sites: TermOffset fallback, no DST
                         timezone_source VARCHAR(12),                  -- link | admin | cm_offset
                         city, state_code, postal_code NULL,
                         latitude DECIMAL(9,6) NULL, longitude DECIMAL(9,6) NULL,
                         geo_source VARCHAR(12) NULL,                  -- cm_terminal | geocoded | manual
                         status, valid_from, valid_to)                 -- versioned (history table)
                         -- CHECK (location_type = 'region' OR timezone_id IS NOT NULL OR utc_offset_minutes IS NOT NULL)
ref.location_source_xref(tenant_id, location_id, source_instance_id,
                         source_entity VARCHAR(32),                    -- cfm_region | cfm_branch | cm_region_text | rbac_org
                         source_key VARCHAR(128),
                         match_method VARCHAR(8),                      -- exact | rule | manual
                         matched_by NULL, matched_at,
                         PRIMARY KEY (source_instance_id, source_entity, source_key))
ref.location_hours      (location_id, day_of_week TINYINT, open_time TIME NULL, close_time TIME NULL,
                         is_closed BOOLEAN NULL,                       -- all three NULL = unknown (source unparsable)
                         effective_from DATE,
                         PRIMARY KEY (location_id, day_of_week, effective_from))
```

- **Region** exists as an entity only in Link (`cfm_region`, one level). CO stores region as free text; RBAC has
  Region orgs. CI owns the hierarchy; sources map in via `location_source_xref`. A TCR branch from Link and an ATM
  branch from CO/RBAC may be the same branch — onboarding matches them (rule, then manual).
- **Site** covers off-site ATMs ("Branch/Site" in the requirements); it is a peer of branch under a region.
- **Time zone**: Link branches carry an IANA `timezone_id`. CO has none; a CO-only site gets an admin-entered zone,
  or falls back to the terminal's fixed `TermOffset` (`timezone_source = 'cm_offset'`), which has no DST — alert and
  hours logic then uses the fixed offset.
- **Geo** (BAP-47): Link has none; CO has per-terminal `TermCustomData` coordinates, where (0,0) means missing. Branch
  coordinates are the terminals' centroid, else geocoded from city/state/zip, else admin-entered.
- **Hours** come from `cfm_branch_hours` text, parsed to minutes with explicit AM/PM (S4's own parser assumes whole
  hours and a PM close, so CI downtime minutes will differ slightly from Link's `downlength`). Unparsable hours are
  stored as unknown (NULL) and flagged (`unparsable_hours` quarantine row); no default is applied (decision G). That
  branch's `open_minutes` and downtime minutes are NULL and uptime shows "n/a". CO-only sites have no hours in R1 (ATM
  health is R2).

## Devices

```sql
ref.device_type         (device_type_code VARCHAR(16) PK,             -- ATM | CDM | RECYCLER | TCR | TCD | COIN_DISPENSER
                                                                       -- | VAULT | COIN_VAULT
                         device_family VARCHAR(8),                     -- atm | tcr | coin | vault
                         display_name, holds_coin BOOLEAN, listed_standalone BOOLEAN)
ref.device_model        (device_model_id INT PK, manufacturer VARCHAR(64) NULL,   -- admin-entered; no source
                         model_code VARCHAR(64), display_name, device_type_code,
                         end_of_life BOOLEAN NULL,                     -- admin-entered; no source
                         UNIQUE (model_code, device_type_code))
ref.device_model_xref   (device_model_id, source_system, source_key,  -- cfm_device_model.alias / CO Make+Model text
                         PRIMARY KEY (source_system, source_key))
ref.device              (device_id INT PK, tenant_id, location_id,
                         device_type_code, device_model_id NULL,
                         name VARCHAR(256), serial_no VARCHAR(64) NULL, -- Link only; CO has no serial
                         connected_to_hub BOOLEAN NULL, communication_mode VARCHAR(32) NULL,
                         hub_mode VARCHAR(8) NULL,                     -- sap | aap | unknown (below)
                         is_kiosk BOOLEAN NULL,                        -- Genix hub cluster; NULL = unknown (below)
                         acceptor_capacity_notes INT NULL,             -- ATM bill acceptor (BAP-69)
                         acceptor_capacity_source VARCHAR(20) NULL,    -- manual | cm_threshold_proxy
                         status VARCHAR(12),                           -- incl. excluded (CO ApprovalStatus_1)
                         administratively_offline BOOLEAN,
                         commissioned_at NULL, decommissioned_at NULL, -- no reliable source; admin-entered or NULL
                         valid_from, valid_to)                         -- versioned (history table)
ref.device_source_xref  (tenant_id, device_id, source_instance_id,
                         id_kind VARCHAR(16),                          -- s4_id | physical_id | serial | cfmuuid | cm_termid | cm_termkey | ej_terminal
                         external_id VARCHAR(128),
                         valid_from DATE, valid_to DATE NULL,          -- CO TermId renames
                         PRIMARY KEY (source_instance_id, id_kind, external_id, valid_from))
ref.device_bin_profile  (tenant_id, device_id, bin_no TINYINT,         -- Link cassette position / CO hopper 1..6
                         bin_role VARCHAR(8),                          -- dispense | recycle | deposit | reject | coin
                         denomination_id SMALLINT NULL, capacity_count INT NULL,
                         origin VARCHAR(20),                           -- cm_hopper_profile | link_model_map | manual
                         valid_from DATE, valid_to DATE NULL,          -- SCD2 maintained by daily diff
                         PRIMARY KEY (tenant_id, device_id, bin_no, valid_from))
ref.device_link         (tenant_id, parent_device_id, child_device_id,
                         link_type VARCHAR(12) DEFAULT 'coin_rollup',
                         origin VARCHAR(20),                           -- hub_cluster | workstation | comm_mode | physical_id | manual
                         effective_from DATETIME(3), effective_to NULL, set_by NULL)
                         -- UNIQUE (tenant_id, child_device_id) WHERE effective_to IS NULL
```

- **Device types.** Every source type maps to one code; unmapped types are quarantined (`unknown_device`).

  | Source type | `device_type_code` | Family | Listed in R1 |
  |---|---|---|---|
  | Link `rcyr` (recycler) | TCR | tcr | yes |
  | Link `disp` (dispenser) | TCD | tcr | yes |
  | Link `cdisp`, `crcyr` (coin) | COIN_DISPENSER | coin | no: rolled into the linked TCR |
  | Link `vault` | VAULT | vault | no (R3) |
  | Link `vault_coin` | COIN_VAULT | vault | no (R3) |
  | CO `ATM` | ATM | atm | yes |
  | CO `CDM` | CDM | atm | yes |
  | CO `Recycler` | RECYCLER | atm | yes |

- **Coin → TCR link** (BAP-28, BAP-36). Link has no FK. Ingestion derives it in priority order — same hub cluster
  (`cfm_conf_peripheral_device`, once synced to iQ, else the cluster CSV, §4), same workstation (`cfm_workstation_device`), coin device
  "Operated by CDR" (communication mode 4/6) in the same branch, or `device_physical_id = <cluster>-COIN_DISPENSER`
  — and an admin can override. All "(+ coins)" roll-ups go through `cash.v_device_rollup` (§7.3a). **An unlinked coin
  device is not listed as a device**, as the requirement says. Its cash still counts in branch and region totals and
  appears as a branch-level "Unlinked coin" line with a data-quality flag; its `unlinked_coin` quarantine row alarms by
  age until an admin links it. In the EOD CSV and audit report that line is one row per branch and day with Device
  "Unlinked coin", so the export still sums to the branch total (confirm, §16 Q40).
- **Bin profiles** replace CO's missing hopper history: a daily diff of `TermHopperProfile` writes SCD2 rows. For Link,
  bin role comes from a model-level map, `ref.model_bin_map (device_model_id, bin_no, bin_role, denomination_id)`, a global catalogue mastered in `ci_catalog` (origin `link_model_map`), because cassette XML has no bin type.
- **Connected to hub**: N.O.R.M* **and** Aspyn (iQ's rule misses Aspyn). **`hub_mode`**: from the device's hub
  cluster when cluster data is available: template type `aspyn` → `aap`. The template type is not a database column:
  S4 reads it from its template configuration by the cluster's `template_id` (`TemplateUtil`), so the sync or CSV
  (§4) must carry it, not only `cfm_conf_cluster`. (`cluster_type` is 2 Kiosk, 3 Nomadix, 4 Aspyn, 5 Norm; that
  template `aspyn` coincides with types 2–4 is inferred.) Without them, a proxy from communication mode: TCP/IP or Serial Converter → `sap` (S4 drives the device);
  N.O.R.M* → `unknown` (SaP, NaP or AaP; current S4 code gives AaP cluster recyclers `N.O.R.M. IP/Serial`, and the
  `Aspyn` mode belongs to a retired registration path, `research/37`). To be confirmed with the Link team.
- **Kiosk recyclers** (Kiosk+Serve, Genix) are ordinary Link devices: the kiosk's recycler (alias `SourceTech`, type
  `rcyr`) and coin dispenser (`CFMAspynCoin`) journal through S4 `/journal` into `cfm_journal`, `cfm_live_inventory`
  and `cfm_device_downtime`, so they already arrive as TCRs and coin (`research/37`; the kiosk-side posting is
  unconfirmed). R1 **tags** them with `ref.device.is_kiosk`, set from the hub cluster (`cfm_conf_cluster.cluster_type
  = 2`, Genix) when cluster rows are available. Until then the iQ-path proxy of `research/37` sets it, flagged as
  inferred: `device_physical_id LIKE '%-CDR'` with a kiosk model alias (`SourceTech`, `CFMAspynCoin`). The journal
  `tag` is never ingested (§13), so it is not used as a proxy. The proxy misfires for Nomadix-only towers, so with
  neither source the flag stays NULL (unknown). Pages can then label or exclude them once Product
  decides (§16 Q37). A kiosk device type is an R2 extension (§12). CI never reads the kiosk session stream (iQ
  `ssk_sync_*`) or the kiosk transaction database: they hold member data.
- **Status**: Link `offline_flag` plus open downtime; CO `TermStatus`, `MarkedForDelete`, `BlackListed`. CO terminals
  with `TermProfile.ApprovalStatus = ApprovalStatus_1` are `excluded`, as CO excludes them, and feed no page.
- **Bill-acceptor capacity** has no source: CO has no deposit hoppers. Admin-entered; CO's `HigherThreshold` (an
  amount) is offered as a proxy and flagged. Acceptor **fill** has no source either (§7.1).

## Denominations and error catalog

```sql
ref.denomination        (denomination_id SMALLINT PK, currency_code CHAR(3),
                         face_value DECIMAL(9,2) NULL,                 -- NULL only on the 'Unsorted' row
                         media_type VARCHAR(8),                        -- note | coin | unsorted
                         display_label VARCHAR(8), sort_order,
                         face_scope DECIMAL(9,2) GENERATED ALWAYS AS (IFNULL(face_value, 0)) STORED,  -- 0 = Unsorted (§15)
                         UNIQUE (currency_code, face_scope, media_type))
ref.error_code          (error_code_id INT PK, device_model_id NULL,  -- NULL = generic
                         source_code VARCHAR(255), description VARCHAR(1500) NULL,
                         severity TINYINT,                             -- 0 unknown | 1 warning | 2 recoverable | 3 fatal
                         severity_band GENERATED ALWAYS AS (CASE severity WHEN 3 THEN 'high' WHEN 2 THEN 'medium'
                                                         WHEN 1 THEN 'low' ELSE 'unknown' END) STORED,
                         source_system,
                         model_scope INT GENERATED ALWAYS AS (IFNULL(device_model_id, 0)) STORED,  -- 0 = generic (§15)
                         UNIQUE (model_scope, source_code))
```

"Severity 3 / Severity 2" in the requirements are Link's Fatal / Recoverable; Low/Medium/High map to 1/2/3 (assumed,
§16 Q9). `cfm_error_code` is not change-tracked, so the catalog comes from iQ once it is synced (§4), and until then
from `cfm_error_code.csv`. Codes seen in journals but missing from the catalog are inserted with `description = NULL`.

`ref.denomination` has one special **Unsorted** row per currency (`media_type = 'unsorted'`, `face_value` NULL) for ATM
deposits that cannot be attributed to a denomination (§7.3).
