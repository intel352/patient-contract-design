---
id: IDENTITY-001
version: 0.1.0
status: proposed
approval_status: pending
authors: [Codex]
approvers: []
created: 2026-10-22
updated: 2026-10-22
owner_role: Identity and security owner
supersedes: []
superseded_by: []
references:
  - ./project-standards.md
  - ./delivery-and-environments.md
  - ./api-versioning.md
  - ./document-governance.md
  - ../architecture.md
  - ../decisions/0001-pluggable-regional-identity.md
  - ../openapi.json
---

# Identity, authorization, and customizable personas

This document defines the standalone project's proposed identity and access standard. It does not implement an identity service, persona-management API, policy engine, or infrastructure. Approval remains pending; default personas, assurance requirements, delegation controls, and numerical operating targets require named human approval under [document governance](document-governance.md).

## Confirmed requirements and proposed choices

Confirmed design inputs are an AWS-based platform, authentication and backend authorization for every business API operation, multiple default user personas that customers can customize or extend, pluggable regional identity with Cognito first, and preferably stateless token-authenticated requests. Go services must validate the presented credentials and make the authorization decision. Client-side controls never grant access.

The proposal uses signed JWT access tokens, stable application principals, tenant-owned persona definitions, backend capability and field policy, and bounded-fresh local authorization state. This provides interchangeable request-serving replicas without sticky sessions. It does not claim that immediate permission changes can be enforced with no shared policy or revocation state.

The [architecture](../architecture.md) governs patient isolation, field projection, logging, audit, and erasure. [API versioning](api-versioning.md) applies to contract evolution; no historical API representation freezes a user's old access rights.

## Every business request has two independent checks

Authentication establishes which approved user or workload is calling. Authorization establishes what that application principal may do now, for the selected tenant, patient, field, and operation. Possession of a valid token is insufficient on its own.

All seven operations currently described by the [OpenAPI contract](../openapi.json) inherit bearer-token authentication. The service must also enforce the following application decisions:

| Existing operation | Required backend authorization |
|---|---|
| `GET /v1/session` | Active application principal; disclose only that principal's permitted memberships, regional routes, and capability context |
| `POST /v1/tenants/{tenantId}/patient-lookups` | Current tenant membership, `patients.lookup.phone`, permitted search use, and per-result row/field visibility |
| `GET /v1/tenants/{tenantId}/patients/{patientId}` | Current tenant membership, `patients.read`, permitted patient row, and field-read grants |
| `POST /v1/tenants/{tenantId}/patients` | Current tenant membership, `patients.create`, permitted resource scope, and field-write grants |
| `PATCH /v1/tenants/{tenantId}/patients/{patientId}` | Current tenant membership, `patients.update`, permitted patient row/fields, and concurrency preconditions |
| `POST /v1/tenants/{tenantId}/erasure-requests` | Current tenant membership, `patients.erase`, authority to initiate this patient's workflow, and applicable retention/approval rules |
| `GET /v1/tenants/{tenantId}/erasure-requests/{requestId}` | Current tenant membership, `patients.erase`, and visibility into the particular request/case; progress access never grants patient-data access |

The erasure capability is further constrained by workflow actions and case scope. Initiating a request does not authorize its approval or physical execution. Current endpoint coverage does not imply that approval administration has already been specified.

Missing, invalid, or expired authentication returns the safe `401` contract before version negotiation or resource semantics, with `WWW-Authenticate: Bearer` and no selected-version or policy-version metadata. Authenticated requests lacking permission return `403`, or the established indistinguishable `404` when object existence must remain concealed. Error bodies contain no tokens, raw claims, patient values, or hidden membership details. Documentation and generated SDKs must express these distinctions; operational error telemetry uses the same privacy restrictions. API schemas, lifecycle registries, and developer documentation are delivered only to authenticated users or approved developer identities with applicable grants; HTTPS alone is not caller authentication.

CORS preflight is a narrow transport exception: an allowed `OPTIONS` response can describe approved origins/methods/headers without a business bearer token, but cannot execute business logic or disclose customer data. Private health probes and platform administration use their own protected operational boundary. Public sign-in/token endpoints belong to the configured identity provider. Trusted regional deployment configuration can supply public login settings before sign-in; `/v1/session` remains authenticated. These are not anonymous patient-API paths.

## Human identity and regional provider contract

Use Cognito first for browser OIDC Authorization Code with PKCE, using S256 and a public client without an embedded secret. A reviewed OIDC client must validate state, nonce where applicable, and exact approved redirects. The frontend obtains an access token and sends `Authorization: Bearer ...` to the regional API. Consumer teams own their frontend implementation; these protocol controls do not prescribe a React framework or Node production service. [Cognito authorization endpoint](https://docs.aws.amazon.com/cognito/latest/developerguide/authorization-endpoint.html).

Each region has a trusted provider registry and qualified token-validation profile. A configured alternative OIDC provider must meet the same application boundaries before use; provider pluggability does not mean accepting arbitrary issuers, SAML assertions, opaque tokens, or ID tokens as this API's bearer credential. Regional service availability and assurance/revocation behavior require qualification. The [identity decision](../decisions/0001-pluggable-regional-identity.md) records provider pluggability; the [architecture](../architecture.md) records the China deployment caveat and controlled regional migration requirements.

The Go service must verify signature and allowed signing algorithm/key, exact issuer, expiry and applicable not-before, token purpose, configured audience/resource and client binding, subject, required coarse scopes, and any required authentication assurance. Cognito access-token checks include `token_use=access` and the expected `client_id`; validate resource audience where the qualified profile provides it. Do not substitute an ID-token audience check for the access-token profile. [Cognito JWT verification](https://docs.aws.amazon.com/cognito/latest/developerguide/amazon-cognito-user-pools-using-tokens-verifying-a-jwt.html).

Treat all headers and claims as untrusted before verification. An unverified issuer may select an already configured verifier only; it cannot supply a discovery URL, arbitrary JWKS endpoint, region, or policy. Scope JWKS caches to the trusted issuer and key identity; bound key-miss refreshes and protect them from request amplification. Reject unknown algorithms, wrong token types, and mismatched resource audiences. MFA/assurance requirements use explicitly verified provider-specific mappings; absent or unmapped assurance fails the action's policy. [JWT Best Current Practices](https://www.rfc-editor.org/rfc/rfc8725.html).

Resolve the verified `(exact issuer, exact subject)` to an immutable application principal in the regional identity domain. A backend-controlled signed claim can carry that application principal only when its issuer, claim provenance, mapping, and current enablement are trusted; a client-editable profile attribute is not sufficient. Membership, audit actor references, idempotency ownership, and policy bindings use this stable principal.

Never auto-link identities by matching email, phone, display name, or provider group. Link/migrate credentials through authenticated ownership proof, authorized approval, and audit. A provider replacement cannot change tenant membership, grant patient access, reset a revocation, or create a second principal silently. An issuer's subject is not globally unique.

## Tokens, request statelessness, and changing permissions

The API does not require a sticky cookie session or a per-request round trip to Cognito, a user-info endpoint, a central session service, or a remote authorization provider. Any healthy replica can verify the token and evaluate a bounded-fresh local policy view. Refresh and login are separate identity operations; necessary token refresh still counts toward the user-visible interaction timing defined by the architecture.

Amazon Verified Permissions is an AWS candidate for managed policy administration and remote authorization decisions, not a selected per-request dependency. Its authorization API evaluates policies in a service-side policy store. Adopting it requires region/feature qualification and measurement of latency, quotas, availability, and revocation behavior at the required load. The local-decision baseline cannot be implemented by assuming a cached `Allow` response or an unverified policy export reproduces current service decisions. Any local policy engine/distribution design requires semantic conformance and freshness proof. [Amazon Verified Permissions](https://docs.aws.amazon.com/verifiedpermissions/latest/userguide/what-is-avp.html), [IsAuthorized API](https://docs.aws.amazon.com/verifiedpermissions/latest/apireference/API_IsAuthorized.html).

Tokens carry bounded authentication context and coarse access claims, such as trusted principal/client reference, intended API, limited scopes, selected tenant context where the issuance profile supports it, and relevant policy/session version references. They must not embed patient PII, comments, complete per-patient access lists, or an unbounded collection of organization memberships. A tenant claim narrows or identifies context; it does not override current backend membership. A token's policy version is a reference to validate, not evidence that stale grants remain valid.

The proposed request replicas are stateless with respect to local durable sessions. The system still has durable identity mappings, persona assignments, policy versions, and revocation state. These are distributed and locally evaluated with authoritative versions/watermarks, bounded freshness, missing-update detection, and readiness gates. Token-only authorization without current denial state can continue accepting an old permission until token expiry; it cannot simultaneously promise rapid permission revocation.

The proposed initial bound is **five seconds after backend acceptance** for application membership, persona/policy, principal, or session revocation to reach enforcement. This is a proposal, not a measured or approved guarantee. A node that cannot prove state fresh within the accepted bound rejects protected operations. Carry the authorization validity deadline through storage/audit waiting and check it before disclosure. A recent transport heartbeat alone does not prove current policy.

This bound does not cover a provider-side logout or account disable that the application has not observed. Every adapter must describe event/feed guarantees, token lifetimes, key rotation, and outage behavior. Without a proven bounded provider-revocation signal, document the remaining token-validity window instead of promising immediate invalidation. Cognito notes that ordinarily verified JWT signatures and expiry can still validate a revoked token. [Cognito token revocation](https://docs.aws.amazon.com/cognito/latest/developerguide/token-revocation.html).

A signed JWT is not inherently encrypted; its claims are generally readable by whoever holds it. TLS protects transport, not token copies in logs. Do not log bearer tokens, raw claims, authorization headers, or sensitive policy context. Service-to-service mTLS and private networks strengthen transport but do not replace application token verification and authorization at each protected service boundary. Forwarded principal headers are untrusted unless an explicitly authenticated, constrained internal protocol supplies them.

## Proposed default personas

A persona is a named, versioned template of backend permissions and constraints. It is not a numeric access rank. The following defaults are starting proposals, not approved clinical or organizational policy. All operate within assigned tenants and permitted patient/case rows; no default has cross-tenant patient lookup.

| Default persona | Proposed permissions | Explicit limits |
|---|---|---|
| Organization Administrator | Manage tenant membership and persona definitions/assignments inside a separately granted delegation boundary | No implicit patient lookup/read/write, comment access, or erasure authority; administration is a future contract surface |
| Registrar | Phone lookup, read contact fields, create patients, and update permitted contact fields | No comment read/write by default; no erasure approval or persona administration |
| Care Team | Phone lookup, read contact fields and comments, and update the comment field where the clinical/work assignment permits | No patient creation or contact edits by default; no erasure or persona administration; comment write is field- and row-scoped |
| Read-only Contact | Phone lookup and read permitted contact fields | No comments, writes, erasure, or persona administration |
| Privacy Officer | Initiate/manage assigned erasure cases, evaluate documented workflow decisions, and read the minimum contact fields needed for an assigned case | No general browse/search permission by default; approval is separately delegated and cannot approve one's own request; approval-management API remains to be designed |
| Auditor | Read authorized audit evidence within the assigned audit scope | No patient contact/comment access or writes; audit references remain protected data; audit-reader API/access workflow remains to be designed |

Platform Operator is a separate infrastructure role, not a tenant patient persona. AWS/EKS operation does not implicitly grant patient-API access. Any exceptional patient access requires explicit application authorization, approved purpose, scoped duration, and audit; elevated-access procedures remain a design follow-up.

Contact fields comprise name, phone, and structured postal address, including city/locality, state/administrative area, and ZIP/postal code. Comment is a distinct sensitive field. Read/write/search grants are independent and can be applied to nested fields; a masked display is not a permission. Required fields for creation must remain satisfiable by a persona's permitted write set, or creation is disabled for that persona.

The current schema has one comment field, not a clinical-note subsystem. Care Team's proposed limited write means permission to update that bounded plain-text field only for authorized patient rows; it does not claim note authorship, append-only entries, clinical records, or note-specific workflows already exist.

## Tenant customization and delegation

Tenants can clone/adjust default templates and create additional personas using a canonical backend permission vocabulary. Initial operation capabilities are `patients.lookup.phone`, `patients.read`, `patients.create`, `patients.update`, and `patients.erase`, combined with field permissions, row/case conditions, assurance, and workflow restrictions. Membership/persona administration and audit-reader capabilities need their own contract review before implementation; this document does not silently add operations to the current seven-operation API.

Store persona identity, tenant, definition version, capability/field grants, constraints, assignments, author, and approval/audit references. Validate definitions against an allowlist; unknown capabilities or unclassified new fields grant nothing. Template revisions must not widen customized or assigned personas automatically. Version each approved change and distribute it through the same revocation/freshness mechanism as membership.

When a principal has multiple personas, combine only grants valid for the same selected tenant and request context. Mandatory denials, row/field restrictions, separation of duties, assurance requirements, retention rules, and erasure restrictions remain effective; a union of persona labels cannot cancel them. Provider roles/groups are inputs to an approved mapping, never direct authority to create application permissions.

Administration requires explicit **delegation authority**, separate from the administrator's own patient-data permissions. An administrator can grant only capabilities, row scopes, and assurance constraints inside that approved delegation boundary. This allows a membership administrator to assign an approved Registrar role without gaining patient access personally, while preventing the administrator from inventing unrestricted privacy/audit grants.

Prevent direct and indirect self-escalation: an administrator cannot widen their own assignment, edit a persona they hold to increase effective permissions, use a second account to bypass approval, or expand the delegation boundary they depend on. Evaluate the resulting effective permissions rather than checking only the edited object. Assignment and persona-definition updates must be atomic/versioned to prevent stale approval or concurrent-update bypasses.

Proposed default: high-privilege grants, delegation-boundary changes, and erasure approval authority require two distinct authorized human participants, with the requester unable to approve the same change. Record the exact proposed definition/assignment version, participants, decision, purpose, and expiry where relevant. Bootstrapping and emergency recovery require a separately approved process; there is no permanent unrestricted tenant-admin bypass. These controls are proposals pending accountable ownership and approval.

### Proposed persona-management contract boundary

Persona administration is required functionality with the following proposed operation semantics. A separate management OpenAPI contract, exact routes, administrative capability names, and provider/consumer tests must be qualified before implementation; these operations are **not** implemented by, or silently added to, the existing seven patient/session operations.

| Proposed management operation | Request and response contract | Required guards |
|---|---|---|
| Create tenant persona | Tenant scope, display name, canonical grant/constraint set, optional immutable template ID/version; returns persona ID, materialized definition, revision, and policy version | Tenant delegation authority, allowed capability/field vocabulary, idempotency protection, and approval before any privileged activation |
| List tenant personas | Tenant scope plus bounded pagination; returns only authorized summaries and current revisions | No cross-tenant discovery; administrative read grant; opaque scoped cursor |
| Read tenant persona | Tenant scope and persona ID; returns exact definition, template provenance, active revision, and strong ETag | Administrative read grant and tenant binding; redact restricted approval/identity details |
| Update tenant persona | Persona ID, permitted changes, and `If-Match` for the current revision; returns new immutable revision and approval/activation status | Atomic optimistic concurrency, effective-grant/delegation checks, no indirect self-escalation; widenings remain inactive until approved |
| Assign persona to membership | Target tenant membership, persona ID/approved revision, bounded row/case scope and optional expiry; returns assignment ID/revision and effective policy version | Target membership belongs to tenant; delegable grant intersection; exact approved version; proposed two-person rule where privileged |
| Revoke membership assignment | Tenant membership and assignment IDs, `If-Match` revision, and approved reason code; returns revocation state and policy version | Authorized revoker, atomic/idempotent transition, audit, and bounded propagation to every serving replica |

Default template versions are immutable. Creating a custom persona copies the selected template's grants and constraints into a materialized tenant definition, retaining its source template ID/version. Later template publication does not change that definition. Updates create immutable definition revisions plus an approved active-revision transition; assignment history records the revision actually granted. An old revision is historical evidence, not a client-selectable way to escape a current denial or revocation.

Every mutation records actor, tenant, target persona/membership references, before/after revision references, approved grant delta, delegation/approval evidence, and timestamp in restricted audit. Do not put patient content in persona descriptions or reason text. A failed precondition or approval returns a safe defined error and applies no partial grant. Activation and its policy-distribution event must commit atomically so replicas cannot observe an unaudited or half-applied permission change. Rollback cannot reactivate revoked access merely by selecting an older persona, API, or deployment version.

## Workload identities and AWS permissions

Automated callers use distinct confidential workload identities with short-lived, API-audience-bound access tokens and the smallest assigned operation/tenant scope. Do not reuse a human account/token, place a client secret in a browser, or give every service a shared administrative bearer credential. Cognito client-credentials authorization supports M2M custom scopes; exact resource/audience binding and workload identity mapping must be qualified for the selected token profile. Obtain/cache tokens outside the patient lookup execution path, with safe refresh before expiry. [Cognito resource servers and M2M authorization](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-define-resource-servers.html).

Use AWS IAM roles and EKS Pod Identity, where supported and qualified, for pod access to AWS APIs such as DynamoDB, SQS, and KMS. These supply AWS credentials and IAM permissions; they do not automatically mint or authorize a patient-API JWT. An integration from an AWS workload identity to this API's access-token model would require an explicit trusted issuer/exchange design, not acceptance of arbitrary AWS credentials as patient access. [EKS Pod Identity](https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html).

Use separate workload principals and permissions for request handling, policy distribution, audit writing, and erasure execution. An erasure worker's authority is constrained to accepted jobs and their tenant/patient/generation; it cannot accept arbitrary caller payloads as deletion authorization. Preserve initiator and executing workload references in the restricted audit record. Private network location or Kubernetes service-account membership alone is insufficient application authorization.

## Privacy, erasure, and acceptance checks

Field visibility is enforced before serialization on every version and every request. Logging is stricter than UI permission: even a persona allowed to read comments must not cause their values to enter ordinary logs/traces. Audit access has its own policy and retention; permission to administer personas does not confer audit contents. Persona/identity references are linkable data with regional access and retention controls.

Patient erasure does not automatically delete a staff principal, persona definition, or staff Cognito account. Staff offboarding disables identity bindings, assignments, and sessions through a separate process. A Privacy Officer's authority to request erasure does not override a retention decision; an Auditor's read permission does not make audit records immune to their own justified erasure/retention rules. All supported API views respect the same patient tombstones and current access decisions.

Before implementation is approved, define the persona-management and approval contract, canonical administrative capabilities, delegation ownership, workload-token profile, and field/row matrix. Demonstrate negative tests for every existing endpoint: unauthenticated access; wrong issuer/client/audience/token type; disabled principals; stale policies; cross-tenant use; denied fields/search inference; indirect self-escalation; provider migration; workload impersonation; expired assurance; and erasure races. Verify protected data never appears in ordinary logs, including failed-token and failed-policy paths. Load and failure tests must include verification, local policy checks, durable audit, token refresh, and policy-distribution outage behavior; no results are claimed by this document.
