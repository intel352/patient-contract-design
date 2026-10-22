---
id: ADR-0001
version: 0.1.0
status: proposed
approval_status: requirement_confirmed
authors: [Codex]
approvers: []
created: 2026-10-22
updated: 2026-10-22
owner_role: Service architecture owner
supersedes: []
superseded_by: []
references: [../architecture.md, ../docs/identity-and-personas.md, ../docs/document-governance.md]
---

# 0001. Support pluggable regional identity

**Status:** User-confirmed requirement; detailed design approval and implementation pending
**Date:** 2026-10-22
**Decision-makers:** User and architecture collaborators
**Related:** [Architecture](../architecture.md)

## Context

The user requires Cognito initially and the ability to use other identity providers in regional markets. Patient APIs, tenant isolation, field restrictions, audit and erasure must retain their behavior when a provider changes. Regional availability and identity capabilities must be validated before market entry.

## Decision

Use configured regional identity adapters: OIDC Code with PKCE for sign-in, provider-specific validation of signed JWT API access tokens, and exact trusted issuer/subject mapping to immutable application principals. Application policy owns all tenant and data permissions. The REST contract uses provider-neutral bearer authentication and internal principal IDs.

Cognito-only coupling would obstruct regional substitution. A mandatory global broker adds a shared dependency and residency concerns without a current need. The design specifies Cognito as the initial adapter; another adapter is qualified when a target market requires it. Library and integration choices are evaluated independently against this standalone project's requirements.

## Consequences

- Provider changes preserve application principals, policy bindings and audit continuity through explicit verified linking; no automatic email-based merging.
- Each adapter must meet shared claim, MFA, revocation, error-redaction and availability requirements. OIDC support alone is insufficient qualification.
- Lookup requests use local token verification and bounded application identity/policy resolution; no per-request provider RPC.
- Provider migrations require an explicit cutover and old-session revocation. Providers and regions never fail over implicitly.
- A future provider requiring opaque tokens or SAML needs a reviewed regional integration and a renewed latency assessment.
