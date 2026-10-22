---
id: ADR-0005
version: 0.1.0
status: proposed
approval_status: pending
authors: [Codex]
approvers: []
created: 2026-10-22
updated: 2026-10-22
owner_role: Security and service architecture owners
supersedes: []
superseded_by: []
references: [../architecture.md, ../docs/security-review.md, ../docs/delivery-and-environments.md, ../docs/events.md, ../docs/FOLLOWUPS.md]
---

# Bound and explicitly accept pooled-cell compromise risk

## Context

The design distributes tenant patient/phone keys in pooled regional DynamoDB tables. Current backend policy provides tenant/row/field isolation for legitimate application execution. A key prefix, encrypted table or private subnet does not stop a compromised cell-wide application role from accessing the multiple tenants that role is permitted to serve. This was implicit in the original design and needs an explicit onboarding and security-acceptance decision.

## Proposed decision

Retain pooled regional cells for the baseline, with separately scoped API data, policy-administration, publisher, consumer, audit and restore roles. Limit a role's resources/actions and minimize tenants per operational failure domain according to a qualified capacity/risk model; no numerical cell-size limit is invented here. Separate environments and approved residency boundaries. Apply the [security review](../docs/security-review.md) key lifecycle, producer provenance, request and recovery controls.

Record the residual multi-tenant blast radius before production approval. If a customer's requirements prohibit that boundary, qualify a dedicated workload/resource/credential cell or account before onboarding; do not claim a shared role is independently tenant-isolated. A tenant-scoped IAM credential design is an alternative requiring trustworthy credential issuance, supported action constraints, cache/rotation and latency evidence. No per-request STS call is silently added to the lookup budget.

## Alternatives and consequences

- A table or IAM role per tenant can strengthen some boundaries but adds lifecycle/quotas/credential complexity and does not isolate tenants if the same compromised workload can assume every role. It is not the baseline for 20,000 organizations.
- Application-level field encryption can address particular storage-reader threats, but a compromised application with decryption access still has authority to read. Select it through a separate threat-model decision rather than implying SSE or tenant HMACs already provide it.
- Separate cells/accounts reduce compromise scope and simplify some contractual boundaries at higher operational cost. They require tested routing, deployment, recovery and identity isolation rather than name prefixes alone.

Approval requires named security/architecture owners, real AWS negative permission tests and customer-isolation criteria. This proposed ADR neither certifies the design nor changes the documentation-only scope. Track evidence in F-019 through F-021.
