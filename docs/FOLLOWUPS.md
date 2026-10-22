---
id: FOLLOWUPS-001
version: 0.2.0
status: proposed
approval_status: pending
authors: [Codex]
approvers: []
created: 2026-10-22
updated: 2026-10-22
references: [project-standards.md, api-versioning.md, delivery-and-environments.md, ../architecture.md]
---

# Open project decisions and qualification work

This queue belongs only to the standalone health-organization design. Owner roles are placeholders until assigned to named humans. No item authorizes implementation, provisioning or publication.

| ID | Owner role | Open decision/evidence | Completion criterion |
|---|---|---|---|
| F-001 | API owner and consumer representatives | Approve concrete major/minor/patch counts, declared notice/age holds, exact-version overrides and their ownership. Required exact version selection and automatic decommission outside the declared set are confirmed directions. | Accepted version-policy revision with ownership, cost and consumer commitment review. |
| F-002 | Release owner and operations | Approve automatic-production readiness criteria and emergency/manual overrides. | Explicit transition decision, measured rollout/rollback evidence, named incident ownership and kill switch. |
| F-003 | Platform owner | Select supported Kubernetes distribution/version, CI/Git host, registry, GitOps/deployment mechanism and engineer self-service implementation. | Qualified exact versions, accounts, cluster/namespace isolation and reproducible lifecycle design. |
| F-004 | API/tooling owner | Qualify OpenAPI 3.2.1 validator, semantic diff, documentation and Go/consumer generators. | Full spec validation, generated-artifact compilation and semantic conformance; no silent specification downgrade. |
| F-005 | API owner | Complete lifecycle-header definitions and released-version registry when retirement policy is accepted. | OpenAPI describes negotiated headers, RFC Deprecation/Sunset/Link behavior and errors; registry and compatibility gate are designed consistently. |
| F-006 | Operations and consumer owner | Define US latency cohort, endpoint mix, measurement window, availability, RPO/RTO and failure objectives. | Concurrent browser and 20k-RPS acceptance plan with representative skew/payloads and explicit error accounting. |
| F-007 | Security/privacy owner | Finalize field policy, revocation bounds, audit durability/retention, erasure/hold rules and regional identity selection criteria. | Approved policy and recovery/deletion proof requirements; no assumption that the draft constitutes compliance approval. |
| F-008 | Platform and service owners | Measure DynamoDB quotas/warm throughput, read/write amplification, hot keys, audit volumes, namespace cleanup and large-scale restore/erasure cost. | Cost/capacity and recovery evidence for launch plus separate growth scenarios. |
| F-009 | Technical owner | Approve runtime-update cadence, exact tool pins and responsibility matrix. | Named owners and release evidence; Node Current for contract tooling distinguished from any future production LTS requirement. |
| F-010 | API and operations owners | Design authoritative low-cardinality usage counters and restricted consumer registry for retirement. | Complete bounded usage accounting and migration alerts inform explicit retention decisions; authoritative registry generations, holds/pins and fail-closed lifecycle reconciliation govern decommissioning. |
| F-011 | Identity and API owners | Approve persona templates, delegation ceilings, claim mappings and persona-management transport contract. | Authenticated administration specification and conformance cases for custom personas, assignment, revocation and self-escalation prevention. |
| F-012 | Security and platform owners | Select authorization evaluator and qualify AWS Verified Permissions if considered. | Measured regional availability/latency, policy correctness and revocation behavior; no assumption a remote decision service meets the existing budget. |
| F-013 | Documentation/tooling owner | Implement reproducible contract/ERD/diagram generation and rendering checks in future CI. | Pinned tool support, full OpenAPI validation, schema-to-ERD drift checks and visual review; current generated artifacts are design-time exports only. |
| F-014 | Security and release owners | Qualify the user-approved automatic image-only maintenance lane. | Standing policy limits, allowed delta proof, independent checks and rollback evidence; application source/behavior changes retain ordinary review gates. |
| F-015 | API and operations owners | Qualify automatic decommissioning outside the user-declared supported set. | Count/hold/pin resolution, shared-deployment references, source/policy epochs, route drain, retired-version response and no-resurrection race tests; inactivity is advisory rather than a hidden veto. |
| F-016 | Security and platform owners | Select approved hardened-image publishers, release feeds, signature trust, explicit version/digest pins and update-response targets. | Qualified image catalog and release-triggered rebuild/rollout matrix for last approved service sources; no reactivation of pruned targets. |
| F-017 | QA, API and identity owners | Qualify full API session journeys and deauthentication semantics. | Real provider flow, chained returned-state usage, supported-version behavior, refresh/revocation, tenant/field denials and audit/erasure evidence in isolated AWS tests. |

| F-018 | API, integration and security owners | Qualify AsyncAPI/CloudEvents tooling and producer/consumer contracts. | Pinned compatible parser, validator, generator/documentation tools, real transport round trips, schema/behavior compatibility, event privacy and replay/erasure evidence. |

Implementation work, CI definitions, real environment creation and production releases remain outside the currently authorized documentation scope.
