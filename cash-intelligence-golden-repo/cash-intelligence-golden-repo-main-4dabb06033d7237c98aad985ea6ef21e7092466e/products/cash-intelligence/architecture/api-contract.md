---
id: ARC-KIN-CI-110
title: Cash Intelligence architecture — 2b. ci-service API contract
artifact_type: architecture
status: approved
content_state: approved-design
owner_role: Cash Intelligence architecture owner
applies_to:
  - PRJ-KIN-BRAU-CI
source_of_truth: golden-repo
source_refs:
  - "SRC-CONF-CIR-001"
  - "SRC-XR-TDD-001"
  - "SRC-CI-RES-001"
last_updated: 2026-11-17
review_due: TBC
---

# 2b. ci-service API contract

> **Approved (DEC-016, 2026-11-20).** Part of the approved design baseline.

The CI micro-frontend talks to one backend: ci-service's `api` unit, the BFF (§2a, §11). The contract between them is
written first, in `products/cash-intelligence/architecture/ci-service-api.openapi.yaml` (OpenAPI 3.2) in the golden
repository, so that the frontend can build against a mock generated from it and ci-service can implement against the
same file. It follows this design (GOV-CI-004): a change to the contract is made in that file first, then in code.
Every operation names the CI roles that may call it (`x-ci-roles`), the `REQ-CI-*` rows it serves
(`x-ci-requirements`) and the sections and tables it reads or writes (`x-ci-reads`).

**What does not depend on the KBOP shell.** The endpoints, request and response shapes, authorization and scope, paging,
exports and live updates follow from this design and the requirements, and are fixed now. What depends on the shell
or on Crossroads is kept to four seams (below), each tied to an open question, so their answers change the base URL
and the session hand-off, not the endpoints. How the micro-frontend is loaded into the shell (module federation,
shared dependencies, manifest, routes; ADR-028) is the frontend build's concern, not this contract's.

**Conventions**

| Topic | Rule |
|---|---|
| Base path | `{baseUrl}/api/v1`. The base URL is a deployment setting (seam 1). A breaking change gets `/v2`; additions stay in `v1`. The same unit serves the micro-frontend's bundle under `/mfe/ci/` (§2a); with the template's server library the API sits under that path, so `{baseUrl}` ends in `/mfe/ci` (`/mfe/ci/api/v1`, DEC-021) |
| Scope | Never a client parameter. ci-service binds the session's `user_id` to the scope views on every read (§11). A device or location outside the caller's scope answers 404, exactly like one that does not exist; lists simply omit it |
| Roles | Per operation (`x-ci-roles`); any other role gets 403. Service Providers reach only TCR Device Health and TCR Usage Health (REQ-CI-ROLE-003), plus the session, locations, catalogues, export jobs and events. Settings, rules and serviced models are Admin only. User administration is Admin, or a KBOP `FIUser_UserAdmin` of the same FI through the bootstrap rule for roles and location access only (§11) |
| Tenant | FI users: from the token; an FI user who sends `CI-Tenant` is refused with 403 `tenant_not_granted`, never redirected. Partner users: the `CI-Tenant` header carries the tenant chosen from `GET /session/tenants` (one per session, §16 Q22); it selects among the partner's granted tenants and never widens them. Staff: none (§16 Q26). A frozen tenant is read-only: writes answer 403 `tenant_read_only` |
| Drill-down and filters | `level` (`region`, `branch` — branches and sites — or `device`) with `locationId`, the parent of the current level (absent = all regions in scope); `deviceType` (`both`, `atm`, `tcr`; coin dispensers always roll into their linked TCR); `q`, a case-insensitive name filter |
| Periods | `from` and `to`, inclusive business dates (`YYYY-MM-DD`, the branch's local business day, §15). Presets and default periods are the page's |
| Values | Money is a decimal string in USD with up to 4 decimals (`DECIMAL(19,4)`); the page formats it. Counts are integers. Instants are UTC RFC 3339 |
| Freshness and configuration | Current values carry `asOf` with its `basis` (`live`, or `last_eod` for ATMs, §4). Figures computed from settings carry the `configVersion` they used (§10.1) |
| Lists | Keyset paging: `cursor` (opaque, from the previous `nextCursor`) and `limit` (default 50, at most 500). `sort` names a field, `-field` for descending; each operation lists its fields and default |
| Writes | Every write takes an `Idempotency-Key` header (a UUID), stored as `request_key`; a retry with the same key writes once and returns the first result. The replay lookup runs after the session, role and scope checks (a caller who has lost access never gets a stored result) and before the configuration-version check, so a keyed retry of a write that succeeded is never answered 409. Settings and rule edits also carry `baseVersion` and answer 409 `config_version_conflict` when another edit activated first |
| Crossroads Contract 17 | Every string field carries `x-pii` (`true` for personal data: names, emails, phone, technician name, notes, file names). `institutionId` is not an operation parameter: the institution comes from the token, or for a partner user from `CI-Tenant` (§16 Q64) |
| Errors | RFC 9457 problem details (`application/problem+json`) with a stable `code` and the request's `correlationId`. 400 a malformed parameter (`invalid_<parameter>`, for example `invalid_device_type`); 401 invalid session or token; 403 role not permitted for the operation (`role_not_permitted`), no CI role (`no_ci_role`), a tenant not granted (`tenant_not_granted`) or read-only tenant; 404 not found or out of scope; 409 exclusive assignment held (`exclusive_assignment`, naming the holder), stale configuration, tenant selection required; 422 a value outside its published range, a period past the forecast horizon, a branch without forecasts. Exception: a 401 for a token the template's server library refuses (signature, issuer, audience, expiry) and the library's own 403 and 404 come with no body, before CI's code runs (DEC-021; code repository temporary measure TM-03, until the library returns problem details or this row allows bare codes) |
| Exports | `text/csv`, streamed in primary-key order with denominations pivoted and coin merged in the API (§15 rule 4); every export is logged by `audit_log_export` with its scope and period; staff free text is never exported (§13.1). The server names the file (Content-Disposition); the names are open (OQ-001). An export too large to stream answers 202 and runs as a background job under the user's scope (`GET /exports/{id}`, then a short-lived link); file retention needs Legal review |
| Dashboard | One request per widget, run in parallel (§15 rule 3) |
| Live updates | `GET /events`, server-sent events. After a feed run, service record or configuration change, sessions whose scope contains a touched device get an event with no data, and the page re-reads through its endpoints (§4, §14.2). Filtering events by scope is access control: Legal review with §11 |

**Endpoints**

| Page | Endpoints | Roles | Requirements |
|---|---|---|---|
| Session and reference | `GET /session`, `GET /session/tenants`, `GET /locations`, `GET /catalog/{catalog}` | all | ROLE-001…003 |
| Dashboard | `GET /dashboard/widgets/{widget}` (seven widgets) | admin, user | DASH-001…009 |
| Current Device Inventory | `GET /inventory/summary`, `/inventory/items`, `/inventory/denominations` | admin, user | CDI-001…007 |
| Historic EOD Inventory | `GET /eod/trend`, `/eod/denominations`, `/eod/export?format=csv\|audit-report` | admin, user | HEOD-001…008 |
| ATM Cash Replenish History | `GET /replenish-history/summary`, `/chart`, `/flagged` | admin, user | ARH-001…008 |
| ATM Usage | `GET /atm-usage/heatmap`, `/trend` (with `date`: one day's breakdown), `/rankings`, `/export` | admin, user | AUSE-001…007 |
| Cash Order Management | `GET /cash-orders/summary`, `/cash-orders/levels/{panel}`; `POST /cash-orders/recommendations`, `/recommendations/export` | admin, user | COM-001…008 |
| Overstock Analysis | `GET /overstock/summary` (with the two rates), `/chart`, `/flagged` | admin, user | OSA-001…005 |
| Branch Cash Positioning | `GET /branch-positioning/summary`, `/branches`, `/branches/{locationId}` | admin, user | BCP-001…006 |
| TCR Device Health | `GET /tcr-health/summary`, `/tcr-health/flagged?tab=`; `POST /devices/{deviceId}/service-records`, `/service-records/{id}/attachments` | admin, user, service provider | TDH-001…005 |
| TCR Usage Health | `GET /tcr-usage/quadrants`, `/tcr-usage/devices?quadrant=`, `/devices/{deviceId}/errors`, `/tcr-usage/export?report=transactions\|usage` | admin, user, service provider | TUH-001…006 |
| Settings | `GET`/`PUT /settings`, `POST /settings/preview`; `GET`/`POST /rules`, `GET`/`PUT`/`DELETE /rules/{ruleId}`, `PUT /rules/{ruleId}/assignments`, `PUT /rules/{ruleId}/default` | admin | SET-001…009 |
| User administration | `GET /users`, `PUT /users/{userId}/access`; `GET`/`PUT /service-providers/{providerId}/models` | admin (users: also the bootstrap user admin) | ROLE-001, -002, -004 |
| Profile | `GET /profile` (by the session's KBOP user id, not by email) | admin, user | PROF-001 |
| Exports and events | `GET /exports/{exportId}`, `GET /events` | all | — |

The behaviour behind each endpoint is the design's: the order recommendation is computed on request and not stored
(§9.3, §16 Q12) while the entered vault totals are stored through `ops_add_vault_entry` (§10.4); ATM Usage ranks by
transaction count where the tenant's EJ counts exist and by amount otherwise, and says which (`rankedBy`, §16 Q4); a
service record clears the open flags of its type (§10.3, §16 Q24); settings are previewed and activated through the
Platform Config Service with the acting user as actor (§10.1, §11). Where the requirements leave a point open, the
contract carries the question and does not answer it: whether the inventory summary cards follow the Device Type filter
(OQ-016; the response echoes the filters it applied), ATM acceptor fill (§16 Q16; `daysUntilFull` stays null for
ATMs), the overstock look-back (§16 Q40), most settings ranges (§16 Q47; `maximum` is null where open), attachment DOC
and size limit (§16 Q59), and where the settings preview runs (§16 Q35).

**Seams**

| # | Seam | Depends on | In the contract |
|---|---|---|---|
| 1 | How the browser reaches ci-service, and its base URL | §16 Q48 | `baseUrl` server variable; no host in any path |
| 2 | How the session is carried | **Decided** by the Crossroads micro-frontend template (SRC-XR-MFE-001; DEC-021): no server-side session; every call sends the shell's Auth0 access token as `Authorization: Bearer`, validated by §11 steps 1–5. Its audience is §16 Q48 | `kbopBearer` only. EventSource cannot send request headers, so the page reads `/events` with fetch (the template's `useEventStream`) |
| 3 | Cross-origin requests | Same as seam 1 | Left to the route decision; same-origin through the shell needs no CORS. Bearer tokens involve no cookies |
| 4 | Claims-based roles and scope (Crossroads Contract 16) | §16 Q1, §12 | None: scope is server-side, so moving its source from CI's tables to claims changes no endpoint |

**Not in this contract:** the KBOP App Launcher widget's browser token for `https://kbop-api` (§16 Q33), the
micro-frontend's loading contract with the shell, the ingestion path from the CO connector through the Ingress Gateway
(§5), the Platform Config Service and Crossroads isolation-test interfaces (§10.1, §11), and health and readiness
probes, which are deployment concerns (§14.2).
