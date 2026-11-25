---
id: ARC-KIN-CI-102
title: Cash Intelligence architecture — 11. Identity and access (Roles stories, BAP-110, PRD §8)
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
  - "SRC-CONF-PRD-001"
  - "SRC-XR-TDD-001"
  - "SRC-CI-RES-001"
last_updated: 2026-11-22
review_due: TBC
---

# 11. Identity and access (Roles stories, BAP-110, PRD §8)

> **Approved architecture: the R1 design baseline (DEC-006).** Approved by the Cash Intelligence architecture owner on 2026-11-05, after the 2026-11-03 design review. Generated specs follow it; a spec that needs something different is a change here first (GOV-CI-004 in `projects/branch-automation-cash-intelligence/05-delivery/delivery-model-and-gates.md`). Open questions (§16) stay open until answered. Section numbers (§n), open questions (Qn) and decisions (A–I) use the architecture's own numbering; `products/cash-intelligence/architecture/architecture-overview.md` maps each section to its file, and the open questions are in `products/cash-intelligence/architecture/open-questions.md`. Citations: `BAP-nn` = `SRC-JIRA-BAP-001`, `TDD:n` = line n of `SRC-XR-TDD-001`, `research/nn` = `SRC-CI-RES-001` (see the source index).
>
> **Legal review required.** This file covers tenant isolation, fail-closed access scoping and staff identity data. These touch compliance: Kinective's Legal team must review them before they are relied upon or selected as Astra context.

```sql
acc.app_user              (user_id INT PK, tenant_id,
                           host_system VARCHAR(16),                    -- 'kbop'
                           host_subject VARCHAR(128),                  -- KBOP user_id (https://kinective.app/user_id)
                           auth0_sub VARCHAR(128) NULL,                -- secondary lookup, not unique
                           user_type VARCHAR(32),                      -- FINANCIAL_INSTITUTION_USER | PARTNER_USER
                                                                       -- | KINECTIVE_EMPLOYEE | SYSTEM_ADMINISTRATOR
                           kbop_entity_type VARCHAR(16) NULL, kbop_entity_id VARCHAR(128) NULL,
                           kbop_roles JSON NULL,                       -- snapshot at last login; audit only
                           email VARCHAR(320),                         -- lower-cased attribute, not a key; never logged
                           display_name VARCHAR(100) NULL,
                           phone VARCHAR(32) NULL,                     -- CI-owned, optional (§16 Q25)
                           has_enterprise_access BOOLEAN, last_seen_at,
                           UNIQUE (tenant_id, host_system, host_subject))   -- index (tenant_id, email), not unique;
                                                                       -- index (host_system, host_subject), not unique
acc.user_role             (tenant_id, user_id, ci_role VARCHAR(16),    -- admin | user | service_provider
                           granted_by INT NULL, granted_at DATETIME(3),
                           PRIMARY KEY (tenant_id, user_id),
                           valid_from, valid_to)                       -- versioned (history table)
acc.role_mapping          (host_system, host_role VARCHAR(64),         -- R1: one bootstrap row (below)
                           tenant_id INT NULL, grants VARCHAR(24))      -- manage_ci_users | ci_role:<role> (future)
acc.user_location_access  (tenant_id, user_id, location_id,
                           source VARCHAR(16),                         -- ci_admin (R1) | kbop_claim (future)
                           granted_by INT NULL, granted_at,
                           PRIMARY KEY (tenant_id, user_id, location_id))
acc.service_provider      (provider_id INT PK, host_entity_ref VARCHAR(128) NULL,   -- KBOP partner_id
                           name VARCHAR(100), status)
acc.service_provider_user (provider_id, user_id, PRIMARY KEY (provider_id, user_id))
acc.service_provider_model(provider_id, tenant_id, device_model_id, source VARCHAR(16),
                           PRIMARY KEY (provider_id, tenant_id, device_model_id))
```

`acc.service_provider` is a global catalogue (ids allocated centrally, copied into every tenant database), and the
partner → tenant grants live in the control-plane catalogue (`ci_catalog.service_provider_tenant`, §14.1), because
they are read before a tenant database is chosen.

**Interim ownership of roles and scope.** Crossroads' target is that authorization comes from token claims (Auth0 FGA
or organization roles carrying roles, licences and scope attributes such as branch ids and serviced device models;
Contract 16), not from product tables. KBOP's tokens carry none of these today, and Crossroads' shell and identity
increment is planned for 2026-12-16 to 2027-02-28. CI therefore holds roles and scope in these tables for R1, behind
one interface (`acc.v_user_*_scope`, below), so that switching the source to claims later changes the views' inputs,
not the page queries (§12, §16 Q1).

- **Host = KBOP.** CI owns no credentials or sessions. `app_user` is a cache refreshed from validated token claims at
  session start. KBOP tokens carry `user_id`, `user_type`, `entity_id`, `entity_type`, `roles`, `role`, `connection` and
  `connection_strategy` on both tokens, and `email` and `full_name` on the access token only (namespace
  `https://kinective.app/`). They carry no phone, branch, serviced-model or product-role claims. `full_name` is the
  KBOP record's first and last name; confirm it is live in each tenant (§16.1). The user is
  keyed by KBOP `user_id`, not `sub`, which changes when an FI moves the user to another connection. `display_name`
  comes from the access token's `full_name` claim and is refreshed at each session start; CI cannot call KBOP's user
  API (it is cluster-internal or KBOP-audience only). Email is an attribute, never a key, and never written
  to application logs (log the KBOP user id and the correlation id). A partner or Kinective user has one row in each
  tenant they use.
- **CI is the master for roles and scope in R1** (KBOP has no product roles before its R2, no branch concept and no
  partner → FI grants): `acc.user_role` (**no row = no access**), `user_location_access`, `ci_catalog.service_provider_tenant`
  and `service_provider_model`. Never authorise on KBOP's `role` claim (the alphabetically first of `roles`);
  `kbop_roles` is an audit snapshot. `acc.role_mapping` holds only the **bootstrap rule** — a KBOP `FIUser_UserAdmin`
  may administer CI users (roles and location access) of their own FI, matched exactly on the role name with
  `user_type = FINANCIAL_INSTITUTION_USER` and `entity_type = FI` (never a prefix, never the `role` claim) — and is the hook for a future KBOP product-role
  claim. The bootstrap rule grants exactly that and no more: a **user-administration capability**
  (`acc.user_role.ci_role` stays NULL, and the session reports `canAdministerUsers`), not the `admin` role, so a KBOP
  user administrator sees no cash data unless a CI admin also gives them a role. The capability is stored as a row of
  `acc.user_capability (tenant_id, user_id, capability = 'administer_users', granted_by NULL, granted_at)`, separate
  from `acc.user_role`, so a bootstrap user has no `user_role` row. The capability is versioned like a role: a
  revoked capability is not granted again at the next sign-in. It covers roles and location access, not enterprise
  access or service providers, which stay with the `admin` role. The session reports `ciRole` null and
  `canAdministerUsers` true; 403 `no_ci_role` is for a user with neither a role nor the capability. The contract
  already names it (`bootstrap_user_admin`). ci-service's session start applies this rule.
- **The last admin.** A tenant always keeps at least one user who can administer CI users: removing or
  changing the role of the last `admin`, when no bootstrap user administrator remains either, is refused (409
  `last_admin`). Recovery when a tenant has lost every administrator is a support action under §11's staff rules,
  audited.
- **Service providers.** `host_entity_ref` is the KBOP `partner_id` when the provider is a KBOP Partner (§16 Q2). For
  KBOP partner users `service_provider_user` is derived from the token (`entity_id` = partner) and checked against an
  active `service_provider` row; it is maintained by hand only for CI-local providers.
- **Kinective staff** (`SYSTEM_ADMINISTRATOR`, `KINECTIVE_EMPLOYEE`) get **no FI data by default**. Optional support
  access to one chosen tenant, with a reason, a time limit and an audit row, is an open question (§16 Q26). KBOP's own
  precedent is standing, role-based cross-FI access (Kinective Employee FI View Only and User Admin) without a reason or
  time limit. Staff tokens carry `entity_type` `SYSTEM` or `KINECTIVE` and no `entity_id`. This needs Kinective Legal
  review.

**Token validation** (in ci-service, before any tenant query):

1. Validate the token the micro-frontend sends with the call (`Authorization: Bearer`; there is no server-side session): RS256 against the Auth0 tenant's JWKS (cached, refreshed
   on an unknown `kid`, fail closed if unavailable), `iss` ∈ the configured issuers of the environment's Auth0 tenant (the canonical
   `https://<tenant>.us.auth0.com/` and, while KBOP migrates to its custom login domain, that domain too; both share
   the tenant's JWKS), `aud` = the audience configured explicitly for the environment (the candidates are CI's API identifier and KBOP's `api.kbop`, which the micro-frontend template defaults to; which one CI accepts is §16 Q48), `exp`/`iat`
   with at most 60 s clock skew.
2. Require `user_id` and `user_type`, and an `entity_type` that matches the type: `FI` for
   `FINANCIAL_INSTITUTION_USER` and `PARTNER` for `PARTNER_USER`, each with a non-empty `entity_id`; `SYSTEM` or
   `KINECTIVE` with no `entity_id` for staff. Any other combination, a missing claim, or a claim longer than its `acc.app_user` column → deny. (KBOP's login action lets a login succeed without claims when its user-service is down.)
3. Resolve the tenant: FI user → the `ci_catalog.tenant` row whose `kinective_id = entity_id` and
   `status` is `active` or `frozen` (a frozen tenant is served read-only, step 4; any other status is refused); a 15-character `entity_id` is first converted to its 18-character form (Salesforce checksum
   suffix), any other length → deny; partner user → the tenant chosen in CI's tenant switcher, which must have a
   `ci_catalog.service_provider_tenant` grant for the user's partner (one tenant per session, §16 Q22); staff → none
   unless support access is granted. A launch parameter, URL path or header may only **select** among allowed
   tenants, never widen them.
4. Connect to that tenant's database with that tenant's credential (§14.1) — the only way any query reaches the
   tenant's rows — and start the session in one transaction (ci-service's session start; DEC-028): upsert `app_user`
   from the claims (KBOP user id, user type, entity id, KBOP roles, display name, email), apply `acc.role_mapping`
   (the bootstrap rule) and return `user_id`, `ci_role` and the mode applied. A tenant that is neither `active` nor
   `frozen` is refused; a `frozen` tenant is read-only.
   `kbop_roles` is stored only as the audit snapshot.
5. Audit session creation and validation failures in `audit.audit_log` once the tenant is resolved. There is no
   server-side session and every call runs step 4 (DEC-021), so the session is not recorded on every call: `session.create` is
   written at most once per user per 15 minutes while the claims are unchanged. A call by a user with a CI role, with
   unchanged claims, less than 15 minutes after that user's last recorded `session.create` writes nothing (no
   `app_user` update, no audit row); a continuously active user therefore gets one row about every 15 minutes, and
   `last_seen_at` is the time of the last recorded row, not of the last call. A first call and a call with changed
   claims always write. *(Proposed: not part of the approved baseline until the architecture owner approves it.)* Failures before
   that (missing claims, invalid token, JWKS unavailable, unknown or inactive tenant) have no tenant database and go
   to `ci_catalog.security_event`, append-only like the audit log; a rising rate alarms.

These steps validate **KBOP's claims**, the interim. Crossroads Contract 16 is the target: a shell token carrying
`institutionId`, `partnerId`, `roles[]`, `licences[]` and scope attributes (TDD:529). When it arrives, step 3
resolves the tenant from `institutionId` (`ci_catalog.tenant.institution_id`), a missing CI licence denies, and roles
and scope come from claims (§11, interim ownership).

KBOP's App Launcher opens CI's production URL with no parameters: there is no signed redirect, no launch-time
entitlement check and no launch audit, so CI treats every arrival as a direct visit and enforces tenant status and
entitlement itself. There is no server-side session (the Crossroads micro-frontend template, SRC-XR-MFE-001): the micro-frontend sends the shell's Auth0 access token with every `/api/v1` call, and ci-service validates it every time. The template's server library (`crcommons-mfe-server`, java-jwt) does step 1's signature, issuer, audience, expiry and issued-at checks, with 30 s leeway (inside step 1's 60 s); it does not require `exp` or check the claim combinations, so CI's own layer refuses a token without `exp` and runs steps 2–5. One exception: the KBOP App
Launcher widget in CI's top bar needs a browser-side access token for audience `https://kbop-api` (FI users only), so
hosting it needs a user grant for CI's client on that API and a decision on how the page obtains the token (§16 Q33).
KBOP access tokens last 30 minutes and its sign-in session 30 minutes idle / 8 hours at most, which bounds CI's
silent sign-on.

**Fail-closed scope, applied by ci-service.** Tenants are separated by database and credential (§14.1); within a
tenant, branch and device-model scope is a join every page query makes:

```sql
acc.v_user_device_scope   (user_id, device_id)      -- devices the user may see
acc.v_user_location_scope (user_id, location_id)    -- pickers and labels
acc.v_user_rollup_scope   (user_id, location_id)    -- location rollups (admin and user only)
```

- **What the views return.** Devices: admin → all in the tenant; user → devices at assigned branches/sites, or all
  with enterprise access; service provider → TCR/TCD whose model has a `service_provider_model` row (none means
  nothing). Locations: the locations of visible devices and the user's assigned branches/sites, each with its
  **ancestor regions** (so a page can label a device's region for a service provider too). Rollups: admin → all locations; user → assigned branches/sites, or all with enterprise access;
  **service provider → none**. A user with no `user_role` row gets no rows from any of them.
- **Every page query joins a scope view** on `user_id = ?`, bound from the validated session (never from the
  client): device-keyed reads join `v_user_device_scope`, location labels `v_user_location_scope`, the location
  rollups (`cash.location_daily_*`) `v_user_rollup_scope`. The queries live in one repository module, and a test
  (ci-service `PageScopeTest`) rejects a registered page query against a device- or location-keyed table that names no
  scope view or binds no session `:user_id` (a check of names, not a parse of the join; the plan test checks the join
  of the pages the §15 rules were written for). No scope rows
  means no data.
- **No connection state.** Scope is a bind parameter, not a session variable or a per-connection table, so any
  connection pool or proxy works and read replicas can serve page reads (§16 Q36).
- **Grants.** Business logic lives in the services; the database holds no stored procedures or functions (DEC-028).
  Each unit's account has SELECT on the base tables and scope views it reads, and INSERT, UPDATE or DELETE only on the
  tables its own writes change, granted table by table; no account has DDL, DROP or TRUNCATE (§13). Every write goes
  through ci-service's one write layer, which commits the change and its `audit_log` row (before/after JSON) in one
  transaction. **Admin writes** (assignments, recipients, schedules, roles, location and enterprise access, service
  providers) check the caller's `ci_role = 'admin'`; the user-administration writes also accept the bootstrap
  capability. **Scoped writes** check the target against the caller's scope view and stamp the session's `user_id`:
  - log a service record (device ∈ device scope)
  - add an attachment (the record's device ∈ device scope)
  - add a vault entry (location ∈ rollup scope, so not Service Providers)
  - record an audit export (its scope and period)

  The `api` account can therefore write those tables directly. What stands between a defect in ci-service and an
  unchecked or unaudited change is the write layer, the per-table grants, the history triggers (which record every
  writer, §15) and the anchored audit head (§13); §16 Q65 asks Legal and Security to accept this.
  Ingestion and jobs use a separate account per service with no API exposure. ci-service therefore runs as **two
  deployment units** from one codebase, each with its own credential per tenant (Crossroads issues one credential per
  institution per deployment unit, TDD:399): `api` (pages, exports, user writes; the grants above) and `jobs`
  (scheduler, shared rule evaluator, reminder job, alert sender; DML on `ops`, `cash` derived tables and queues, no
  inbound network route).
- **Settings writes.** Writes to the Platform Config Service go through ci-service, which checks `ci_role = 'admin'`
  before every Config Client write or activation and passes the acting user (KBOP user id) as the change's actor, so
  the platform's `CONFIG_CHANGED` audit names the CI user; ci-service also writes an `audit_log` row with the
  before/after values. A User or Service Provider token can never reach a Config Client write (requirement: Users
  cannot access Settings).
- **Scope audit.** Because the API account can read base tables, scope rests on the repository check above. A nightly
  job reads `performance_schema.events_statements_summary_by_digest` for the API account and alarms on any statement
  that reads a device- or location-keyed table without its scope view. Page reads stay scoped SQL in the service
  (§16 Q39, closed by DEC-028): every page query is a method of a read repository that takes the session user, so its
  scope join cannot be left out, and a test checks every registered query for it.
- **Global catalogues are readable by every role**, Service Providers included: `ref.device_model`, `ref.device_type`,
  `ref.denomination`, the generic rows of `ref.error_code` and `fcst.model` hold no tenant data. This is a deliberate
  exception to fail-closed: a Service Provider with no serviced models sees no device or location rows and only these
  catalogues (`research/80` §6 test 11). Legal review with the rest of this section.
- **Isolation test.** Crossroads' isolation harness runs against CI's BFF before go-live: every endpoint replayed
  with a second tenant's claims returns zero rows and zero bytes. CI adds its own scope tests (a User outside a branch,
  a Service Provider with and without serviced models) at the same level.
- **Performance** was measured with the view join (`research/86`: one 5,000-device tenant, 180 days, 306 users):
  every page is under 2 s warm by median without materialized scope tables. The views cost 2–25 ms when `user_id` is
  pushed into them; joined inside an outer join or beside another non-mergeable view, MySQL materializes one for
  every user (0.6–0.9 s), so page queries follow the §15 scope-join rule. The cost grows with users × access rows, so
  it is re-measured before go-live at the largest expected scope cardinality: the most users, the most access rows
  per user, and the most devices any tenant is expected to have. Views that union user and provider
  scope or walk region ancestors are not mergeable in MySQL; if the join is slow, the views read a materialized
  `acc.user_device_scope` / `acc.user_location_scope` table (user × device, user × location), refreshed in the same
  transaction as a role, branch/site access, enterprise-access flag, service-provider membership or status,
  provider-model, device or location-hierarchy change (a location's parent or region). The views check provider
  status on every request, so a deactivated provider loses access at its users' next request; no session holds scope. That is still keyed by `user_id`, not connection
  state.

Because this is access control with compliance implications, the scope design, the staff-access rule and the KBOP
identity mapping need Kinective Legal review before go-live.

- **Scope mismatch to settle**: providers are scoped by **model** (requirement) while Link links them per **device**
  (`cfm_device.service_provider_id`) (§16 Q2).
