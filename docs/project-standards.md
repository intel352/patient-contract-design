---
id: STANDARDS-001
version: 0.2.0
status: proposed
approval_status: pending
authors: [Codex]
approvers: []
created: 2026-10-22
updated: 2026-10-22
owner_role: Project technical owner
supersedes: []
superseded_by: []
references: [../architecture.md, api-versioning.md, delivery-and-environments.md, document-governance.md, FOLLOWUPS.md]
---

# Project standards and assumptions

This is the standalone project's guidance. User-confirmed requirements below are binding design inputs. Numerical maintenance, support and promotion defaults are proposals awaiting named human approval; they must not be mistaken for configured controls or benchmark results. No implementation is authorized by this document.

## Responsibility boundary

The service team owns the Go REST API, entity and error definitions, version negotiation, generated consumer-contract artifacts, provider/consumer conformance, auth adapters, authorization, regional storage, privacy, erasure, telemetry and delivery requirements. It publishes stable contracts and compatibility evidence.

The consuming team owns frontend architecture, bundler, state management, components, rendering strategy and Node/React runtime choices inside its application. Current stable React is a consumer assumption, not a mandated frontend implementation. This project does not select SSR, a Node BFF, hooks or a UI build framework. Typed SDKs/examples prove contract consumption without prescribing how the UI is built.

The browser-visible p95 requirement remains joint work: the service team supplies load, server and trace evidence; the consumer team supplies representative browser journeys/render markers. API-only benchmarks cannot approve that objective. Boundaries do not excuse dropping network, token refresh or rendering time from measurement.

## Confirmed design inputs

| ID | Requirement |
|---|---|
| REQ-001 | 20,000 isolated customer organizations; initially 10 million patient-organization records and 20,000 aggregate requests/second. These are scenarios, not storage ceilings. |
| REQ-002 | Exact phone lookup and patient-detail reads target browser-visible p95 below 100 ms for an agreed US population/network cohort. |
| REQ-003 | US first; explicit regional placement and qualified regional identity providers for expansion. |
| REQ-004 | Go REST API with a canonical current OpenAPI/entity contract; frontend implementation is outside this team's scope. |
| REQ-005 | DynamoDB primary storage; pluggable identity, Cognito first; backend tenant/action/row/field authorization; safe observability and tracked erasure. |
| REQ-006 | Git, human PR approval and independent AI review for application and policy changes. AI-authored code meets the same gates; eligible image-only maintenance has an explicitly approved automated exception. |
| REQ-007 | Kubernetes; shared `preview` and `production`; on-demand isolated engineer/PR deployments, including clusters when necessary. No permanent historical `dev` tier is required. |
| REQ-008 | Successful changes on the integration branch continuously deploy to preview. Ordinary application production promotion initially needs human approval; later automation requires an explicit policy transition. Eligible image-only maintenance follows its standing automatic policy. |
| REQ-009 | Repository-local design/specification and decision records with version, authorship, approval and cross-reference metadata; decisions support supersession. |
| REQ-010 | CI enforces schema and behavioral API SemVer. Configurable major/minor/patch counts plus declared retention overrides determine supported versions; deployments outside that set automatically decommission. Request and response identify the API version; usage informs explicit overrides. |
| REQ-011 | AWS is the cloud platform; use suitable AWS services, with regional availability and latency qualification. EKS is the Kubernetes reference deployment. |
| REQ-012 | Bearer access-token authentication and service-side authorization for every application API operation; no anonymous patient, session, schema or lifecycle API. Stateless request handling is preferred, with explicit policy/revocation freshness guarantees. |
| REQ-013 | Configurable default personas and tenant-defined additional personas; auditable grants bounded by the administrator's delegation authority. |
| REQ-014 | Generated OpenAPI artifacts and ERDs, plus maintained overall architecture, request-flow and sequence diagrams. |
| REQ-015 | Hardened container images, explicit version/digest pins and prompt release-triggered refresh of the last approved service source for every serving supported API representation. Eligible image-only updates automatically test and deploy under an approved policy while application changes retain human review. Pruned deployments must not be accidentally reactivated. |
| REQ-016 | Backend unit, integration, behavioral and end-to-end tests, including stateful client journeys from authentication through dependent requests to deauthentication. Behavioral compatibility participates in SemVer classification. |

| REQ-017 | Downstream application events use CloudEvents; evaluate and qualify AsyncAPI as the machine-readable event/channel contract alongside OpenAPI REST. |

## Technology baselines

Verified against primary release sources on **2026-10-22**. "Latest" means the latest eligible release evaluated and pinned through a reviewed change; it never means a floating `latest` image tag or downloading whatever happens to exist during CI.

| Technology | Baseline and update expectation | Boundary |
|---|---|---|
| Go | **1.27.3**, latest stable verified. Pin the exact toolchain in future developer/CI/build configuration; move to newer stable releases through qualification. | Backend compiler/runtime; no prerelease production compiler. |
| Node.js | **26.12.1 Current**, latest stable verified, for this project's contract generation/validation tooling. **24.22.1 LTS** is the verified latest LTS alternative. | Current is stable but not LTS. No Node production service is selected here. If one is later added, choose a supported LTS line through its design review, consistent with Node's production guidance. |
| OpenAPI | **3.2.1**, latest published specification verified. Pin the exact dialect/version in source and CI. | Distinct from the API's own SemVer. Generator, validator, diff and documentation tooling must actually support the selected features. |
| AsyncAPI | **3.1.0**, current published specification verified; separate event API draft version. | Event channels/operations/messages; exact parser/generator support requires qualification. |
| CloudEvents | **1.0.2** specification release, with **`specversion: "1.0"`** on the wire. | Standard event envelope; payload schemas and REST API have independent versions. |
| React | Current stable consumer expectation; official documentation currently identifies **19.3**, with release **19.3.0**. | Informational consumer assumption. No frontend build or dependency policy is imposed by this API project. |
| Kubernetes | Organization-supported, upstream-supported stable EKS release; exact version and add-ons require platform qualification. | EKS is the AWS reference deployment, aligned with regional DynamoDB. A precise supported version must be recorded before implementation. |
| Images, dependencies and tools | Exact reviewed versions/digests, committed dependency locks/checksums, reproducible inputs and verified artifacts. | SDK/validator/generator choices remain candidates until the contract compatibility gate passes. |

Event sources: [AsyncAPI 3.1.0](https://www.asyncapi.com/docs/reference/specification/v3.1.0), [CloudEvents releases](https://cloudevents.io/). See [event contracts](events.md) for tooling evidence and limitations.

Sources: [Go downloads](https://go.dev/dl/), [Node release policy](https://nodejs.org/en/about/previous-releases), [Node 26.12.1](https://nodejs.org/en/download/archive/v26.12.1), [Node 24.22.1](https://nodejs.org/en/download/archive/v24.22.1), [OpenAPI 3.2.1](https://spec.openapis.org/oas/v3.2.1.html), [React versions](https://react.dev/versions).

Adopting OpenAPI 3.2.1 does not establish tool support. For example, the currently inspected `oapi-codegen` release advertises initial 3.1 support, not a qualification of this 3.2.1 contract. Keep generator selection open; select compatible tooling or prove a semantics-preserving generation path. An unverified downgrade or merely relabeling an input to satisfy a tool is not acceptable. [Generator release evidence](https://github.com/oapi-codegen/oapi-codegen/releases)

Proposed maintenance cadence: review release/support status weekly; qualify routine patch updates within 14 days and new stable toolchain lines within 30 days when ecosystem support permits. A documented exception has an owner, reason and expiry. Security triage overrides routine cadence: assess reachable critical issues within 24 hours, then apply a severity/exposure-based remediation or containment deadline approved by security. These are proposed operating targets, not an existing staffed response promise. Unsupported tools/runtimes require migration before production support ends.

## Contract and implementation expectations

- Keep transport DTOs, internal domain models and persistence items separate. The published API is the client contract; DynamoDB layout is not exposed as that contract.
- Use the canonical OpenAPI source and synthetic examples. Generate consumer/server artifacts reproducibly; validate real request/response behavior as well as static types.
- Apply [API versioning](api-versioning.md) to implementation behavior, errors, authorization semantics, pagination, concurrency, idempotency and documented limits, not just JSON property changes.
- API versions never preserve obsolete access rights. Current authorized policy, erasure state and residency rules apply to every representation. Security-related compatibility exceptions require explicit evidence and governance, not a misleading patch label.
- All service operations require verified bearer tokens and current authorization. Browser CORS preflight is narrowly restricted transport negotiation, with no business handling; private probes are infrastructure checks. Neither creates an anonymous application API. AWS infrastructure APIs use scoped AWS workload identity rather than reusing end-user bearer tokens.
- [Identity and personas](identity-and-personas.md) defines default/custom persona policy, claim validation and the difference between stateless service instances and the state needed for prompt revocation. Signed claims alone cannot guarantee immediate policy changes.
- Generate JSON/YAML OpenAPI exports and schema-derived ERDs from canonical inputs; maintain system and sequence diagrams alongside them. Record provenance and check drift. Generated documentation is not evidence of a deployed implementation.
- [Image maintenance](image-maintenance.md) defines hardened builder/runtime/test images and refresh provenance. [API testing](api-testing.md) defines required evidence at each layer. A base-image refresh is a new implementation artifact even when API SemVer and service source are unchanged.
- [Event contracts](events.md) separate OpenAPI REST, AsyncAPI channels/messages and CloudEvents payload envelopes. Event schemas and behavior have their own compatibility lifecycle; an API retirement cannot implicitly retire an event consumer contract.
- Fixed patient schema initially. New fields require classification, permissions, example/schema updates and compatibility analysis before introduction.
- Keep lookup work, retries, queues, batch sizes and deadlines bounded. Service availability and successful-and-fast measurements accompany latency. Fast errors do not satisfy performance requirements.

## Collaboration and release rules

Git is the source of truth for code, specifications, migrations, deployment intent and release metadata. Protect the integration branch and immutable releases. AWS CodePipeline and CodeBuild are the CI/CD reference. Git hosting, exact reviewer integration and detailed deployment qualification remain open; [delivery](delivery-and-environments.md) defines the controls those choices must satisfy.

Every future change record or PR must include a human-readable change rationale, linked requirement/spec/decision, API impact classification, validation evidence, privacy/security impact, migration/rollback considerations and AI-assistance disclosure. Small coherent PRs are preferred; no quantity-based exception bypasses review.

Human reviewers own application/policy approval and must be able to explain the change. The approved image-only lane uses policy-authorized automated pin/provenance changes and independent review/checks without per-update human approval; its exact allowed inputs must be verified. A source, dependency, policy or unexpected behavior change cannot use that exception. AI review is an additional independent check, not a replacement approval or proof. An unavailable AI review is a visible failed/pending gate and blocks a new change. Incident recovery may use an already approved eligible artifact under recorded human authority; it does not authorize a new unreviewed application change or silently waive either PR-review gate. Restrict AI/review services to approved source and synthetic evidence; never submit patient data, secrets or production dumps.

Build, test, sign and promote immutable artifacts. Preview updates automatically after successful merge validation. For ordinary application changes, human production approval is initially separate from human merge approval. Qualified image-only maintenance has a separate standing automated lane even during this initial mode. Later automatic production promotion retains all PR, compatibility, security and rollout gates; enabling it is an approved policy change with an accountable owner and immediate disable mechanism.

## Assumptions requiring validation

| ID | Assumption | Evidence before launch |
|---|---|---|
| ASSUME-001 | Regional EKS and the selected AWS platform services are available. | Versions, networking, identity, quota and ownership decisions; market-specific availability review. |
| ASSUME-002 | Two static environment names suffice; ephemeral environments cover unmerged work. | Isolation tests, self-service lifecycle, cleanup evidence and a representative preview setup. |
| ASSUME-003 | Current stable OpenAPI can be used end-to-end. | Strict format validation plus Go/consumer generation/compilation/conformance with pinned tools. |
| ASSUME-004 | Bounded local identity/policy state, DynamoDB reads and durable audit can fit the latency budget. | Concurrent browser and 20k-RPS workload evidence including skew/failure cases. |
| ASSUME-005 | Version support can grow beyond the proposed rolling target when real clients require it. | Funded owners, compatibility automation, usage evidence and capacity for retained lines. |

The definitive unresolved queue is [FOLLOWUPS.md](FOLLOWUPS.md). Proposed support durations and automation-readiness thresholds are not approved by appearing in this document.

## Evolution and exceptions

A material change to tenancy, identity trust, residency, API compatibility, erasure, production approval or scope requires a proposed decision record and human review. Approved exceptions identify affected controls, risks, compensating measures, an owner and expiry. Do not silently weaken a requirement to fit a tool or benchmark.

| Date | Revision | Change |
|---|---|---|
| 2026-10-22 | 0.1.0 | Initial standalone standards from user requirements; runtime facts verified, operational defaults proposed. |
| 2026-10-22 | 0.2.0 | Confirmed automated image-only lane and configurable declared API support; added behavior/session test standards and complementary event contracts. |
