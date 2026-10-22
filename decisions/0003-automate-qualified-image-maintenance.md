---
id: ADR-0003
version: 0.1.0
status: proposed
approval_status: requirement_confirmed
authors: [Codex]
approvers: []
created: 2026-10-22
updated: 2026-10-22
owner_role: Security and release owners
supersedes: []
superseded_by: []
references: [../docs/image-maintenance.md, ../docs/delivery-and-environments.md, ../docs/api-testing.md]
---

# 0003. Automate qualified hardened-image maintenance

## Context

The user explicitly approved automatic image-only updates under a standing policy on 2026-10-22, while retaining human review for application changes. New hardened bases should refresh the last approved source across supported API versions without reviving pruned deployments.

## Decision

Use a narrowly authorized maintenance lane that verifies unchanged application source, contracts, dependencies and policy; changes only approved image inputs and explicitly permitted toolchain patches; runs independent review/checks and the full applicable test matrix; and automatically canaries eligible signed artifacts. This exception applies even while ordinary application production releases require human approval. The complete operational policy still needs review and qualification before activation.

Per-update human approval was rejected for this eligible change class. Unrestricted automated dependency/source changes were rejected because they exceed the approval and can change behavior. An unexpected behavior change exits the maintenance lane and receives normal contract classification and review.

## Consequences

- A new digest does not automatically create an API version or consume a retention slot.
- Rebuild and promotion attestations identify exact source, allowed input changes, test evidence and policy authority.
- Lifecycle checks prevent stale maintenance jobs from reactivating excluded versions.
- Publisher trust, image families, response targets and rollback readiness must be qualified before automation is enabled.
