---
id: GR-ARC-001
title: Crossroads Foundation — Obligations for an ABS (TDD v0.5)
artifact_type: platform-obligations
status: draft
content_state: source-backed
owner_role: Crossroads Foundation architecture owner (TBC)
applies_to:
  - PRJ-KIN-BRAU-CI
source_of_truth: golden-repo
source_refs:
  - "SRC-XR-TDD-001"
last_updated: 2026-11-03
review_due: TBC
---


# Crossroads Foundation — Obligations for an ABS (TDD v0.5)

**Source-backed, draft.** Every rule below is stated in `SRC-XR-TDD-001`: the Crossroads Foundation Technical Design
Document v0.5 (Draft, 2026-10-29), `kinective.io/internal/platform/crossroads/crossroads-foundation`, commit
`8c55357`, file `docs/architecture/crossroads-technical-design.md`. `TDD:n` is line n of that file at that commit. The
TDD is itself a draft: re-check these lines when a newer version is published, and treat the TDD, not this summary, as
authoritative. This file holds no product facts; how Cash Intelligence applies each rule is in
`products/cash-intelligence/architecture/crossroads-platform-fit.md`.

## Stack and runtime

| Rule | TDD |
|---|---|
| One stack for the platform and every ABS (D-21): Kotlin on JDK 25 LTS with Ktor, Gradle with Kotlin DSL; no second server framework | TDD:371–377 |
| One OpenAPI 3.2 document per API, with `institutionId` in every operation and `x-pii` on every string field (Contract 17); one AsyncAPI document per ABS. *CI: `x-pii` is met; `institutionId` is bound from the token instead, asked as §16 Q64* | TDD:382–383 |
| Shared cache: every key carries the institution; nothing PII-bearing in a shared cache | TDD:380 |
| Frontend: React with the Prism 2.0 React library (`@kinective/prism-react`), shipped as child micro-frontends of the KBOP shell (D-23) | TDD:378 |
| Every service containerized; AWS only (D-10); Amazon EKS, with separate node pools for platform services and ABS workloads. The TDD still lists the database engine and hosting as open (TDD:405, below); Cash Intelligence tracks the gap as Q21 and finding F8 | TDD:387 |
| Service packaging is the ABS's choice: one deployable each or split by solution, scaling profile or blast radius | TDD:511 |
| No platform or ABS service gets a public endpoint; the Platform Gateway reaches targets over internal networking | TDD:863 |

## Storage per institution (D-22)

| Rule | TDD |
|---|---|
| One logical database per institution per ABS, and one object-storage bucket per institution per ABS | TDD:394–396, 386 |
| One database credential per institution per deployment unit, brokered from the platform secret store, so a mis-routed connection fails at authentication | TDD:399 |
| Migrations are Flyway SQL changesets, one history table per institution database, applied across the fleet by a platform-owned runner | TDD:402 |
| Open: connection fan-out at hundreds of institutions (R-34), and the database engine and hosting | TDD:405 |

## Events, ingress and egress

| Rule | TDD |
|---|---|
| Publish only through the Foundations SDK, which validates the envelope against the registered schema and refuses to publish without `institutionId` | TDD:415 |
| The Platform Gateway is the only way a worker or agent reaches any system, including the ABS's own (D-24) | TDD:464 |
| The Data Triage service is the only consumer of every `crossroads.<abs>.ingress.v1` topic; batch and file facts arrive there through the SDK's batch processing | TDD:517 |
| Publish state changes, not aggregates (R-17) | TDD:1161 |
| Crossroads is a shared execution spine, not a replacement for an ABS's own systems | TDD:912 |

## Identity and configuration

| Rule | TDD |
|---|---|
| Contract 16: the OIDC token from the KBOP shell carries `institutionId`, `partnerId` where relevant, roles and scope | TDD:529 |
| Contract 9: product configuration is a JSON Schema per ABS; every schema needs a `version`, an `inferenceGateField`, an `autonomyLevel` and a published range for every numeric field | TDD:472, 476 |
| The configuration store is the platform `config` schema, PII-free, one per institution | TDD:478 |
| The ABS defines the parameters FI staff may set; whether activation needs approval is the ABS's choice | TDD:922 |
| An ABS declares which facts and outcomes it writes back to Datava, through `datava.recordOutcome` | TDD:946 |

## Go-live and prohibitions

| Rule | TDD |
|---|---|
| Step 11: verification and the cross-institution isolation test | TDD:970 |
| Definition of done: seven go-live gates | TDD:975 |
| Do not publish with a raw Kafka client, publish to the audit or errors topics, or publish without `institutionId` | TDD:990 |
| Do not keep run state anywhere but Conductor and the registry; ABS tables hold domain data, not orchestration state | TDD:991 |
| Do not hold, cache or display another institution's data in any store, prompt or view, including aggregate benchmarks unless they are anonymized by design and approved by Legal | TDD:994 |

> **Legal review required.** The isolation, benchmark and personal-data rules above touch compliance. Kinective's Legal
> team must review any reliance on them before it is treated as approved guidance.
