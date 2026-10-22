---
id: REVIEW-001
version: 0.1.0
status: proposed
approval_status: pending
authors: [Codex]
approvers: []
created: 2026-10-22
updated: 2026-10-22
references: [api-versioning.md, image-maintenance.md, api-testing.md, events.md, ../contracts/asyncapi.json, ../generated/manifest.json]
---

# Design consistency review

This records an independent agent review and the author's corrections. It is not human approval, formal compliance evidence or runtime verification. All application/infrastructure capabilities remain proposed.

## Findings and dispositions

| ID | Finding | Document correction |
|---|---|---|
| D1 | Unconditional human classification/evidence gates contradicted the user-approved image-only automation lane. | Versioning and testing now permit narrowly qualified maintenance under the standing policy, with independent checks and normal review for unexpected behavior or source changes. |
| D2 | Production-GA-only admission made required preview tests of a new version impossible. | Added isolated environment/cell candidate registries and synthetic-only admission, followed by qualified GA publication and fenced per-cell activation. A failed post-GA canary cannot erase release history or reuse its version. |
| D3 | Event audit paths used `phones.e164` while REST policy paths use `phones[].e164`. | Event paths now exactly match the canonical field-access vocabulary; parity was checked. |
| D4 | Erasure completion was specified without a distinct approved start boundary for participants. | Added transactional `patient.erasure.started.v1`, request/generation references and an authorized handler that rechecks the workflow and fences destructive work. Ordinary restriction does not authorize erasure. |
| D5 | Checking active state before writing a downstream effect allowed a delayed worker to recreate erased data. | Require revision/restriction/tombstone preconditions in the effect-and-inbox transaction. External participants fence admission, cancel/drain effects and verify purge before acknowledging completion. Require a paused-worker regression test. |

## Review coverage

| Area | Result |
|---|---|
| Project guidance and user intent | Checked standalone scope, documentation-only implementation boundary, exact version support and image-only approval exception. D1 and D3 corrected. |
| Assumptions and failure modes | Checked pre-GA testing, ambiguous publication, duplicate/out-of-order events, bootstrap and retirement races. D2, D4 and D5 corrected. |
| Security and privacy | Checked token boundaries, event references, current authorization, erasure dominance, queue isolation and logging. D3–D5 corrected. |
| Infrastructure and rollback | No provisioning or deployment performed. Design specifies same-digest promotion, immutable release history, lifecycle fencing and erasure-preserving recovery. |
| Integration and artifact consumption | Real AWS/provider/consumer tests are required before implementation acceptance. Exact OpenAPI/AsyncAPI tooling, generated code and transport behavior remain unqualified. |
| Scope and simpler alternatives | REST remains typed JSON; event contracts reuse regional outbox/SQS. A new broker, universal REST event wrapper and microservice split are unnecessary for this scope. |
| External repository precedent and product UI | Unrelated projects excluded by the user; frontend implementation and rendered UI testing belong to the consuming team. |

## Checks performed and limits

Document checks resolved local links/references, metadata and balanced fences. Contract checks resolved 217 OpenAPI and 41 AsyncAPI local references; checked 19 REST and six event examples with a schema-subset checker; verified seven bearer-protected operations with required exact-version headers; and confirmed equivalent JSON/YAML exports and source/output checksums.

These checks do not establish full specification conformance, pinned generator compatibility, generated Go/client compilation, Mermaid layout, cloud integration, policy enforcement, erasure correctness or browser-visible latency. No runtime tests or benchmarks were run. The current design can be reviewed from the repository; implementation acceptance still requires the documented qualification gates.
