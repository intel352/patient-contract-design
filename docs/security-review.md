---
id: SECURITY-REVIEW-001
version: 0.1.0
status: proposed
approval_status: pending
authors: [Codex]
approvers: []
created: 2026-10-22
updated: 2026-10-22
owner_role: Security architecture owner
supersedes: []
superseded_by: []
references: [../architecture.md, identity-and-personas.md, events.md, delivery-and-environments.md, image-maintenance.md, api-testing.md, ../openapi.json, ../contracts/asyncapi.json, FOLLOWUPS.md]
---

# Security design review and revised acceptance requirements

This review examines the proposed design, including canonical contracts, against hostile inputs, stolen credentials, tenant confusion, compromised workloads and suppliers, and concurrent recovery/deletion. The findings below are **design gaps**, not demonstrated vulnerabilities in an implemented service. All dispositions mean documented proposed controls; implementation, independent penetration testing, AWS policy verification and human acceptance remain pending. No patient data, production environment or runtime was inspected.

## Threat model and trust boundaries

Protected assets include contact values and comments; membership, identity mappings and field policy; phone HMACs and opaque references; credentials and keys; erasure/hold evidence; audit history; and approved build artifacts. Opaque identifiers and keyed phone digests remain linkable restricted data.

| Boundary / adversary | Security expectation | Residual limit |
|---|---|---|
| Browser or external client; malicious or compromised authorized user | Validate tokens and input, bind current tenant/operation/row/field policy, prevent enumeration and unrestricted work | A user can retain data legitimately disclosed earlier; erasure cannot recall delivered bytes |
| Identity provider / callback / key endpoint | Qualified issuer-specific token profile, callback binding, bounded trusted key retrieval, current application denial state | Offline JWT validation cannot instantly observe an unseen provider logout |
| Ingress → service → AWS | Authenticate every application operation, constrain workload identity and egress, verify cell placement | Network location and ALB forwarding are not end-user authorization |
| Pooled service → DynamoDB | Application checks enforce tenant scope; separate AWS roles/resources bound damage by cell and purpose | Compromise of a cell-wide data role can affect its authorized tenants; prefixes and SSE do not prevent this |
| Outbox / SQS / worker / downstream participant | Bind publication authority, tenant and resource; fence duplicates, erasure, holds and restore epochs | Queue sender permission is not proof that an arbitrary tenant claim is authorized |
| Application → logs / audit / browser telemetry | Explicit allowlists, protected references, frozen disclosure audit and restricted evidence | Infrastructure access records may contain linkable metadata and require their own controls |
| Source / AI reviewer / builder / signer / deployer | Untrusted changes have no production authority; independent input/provenance policy precedes promotion | Valid signatures alone do not prove a supplier or approved source is uncompromised |
| Operator / privacy approver / recovery actor | Bounded delegation, independent approvals, recent assurance, auditable emergency actions | Human abuse and account compromise require detection, incident response and accountable oversight |

Use [OWASP ASVS 5.0](https://asvs.owasp.org/) as the proposed verification catalog: map applicable Level 2 requirements and explicitly selected higher-assurance controls to implementation evidence. Record exclusions and owners. This is a verification plan, not an ASVS certification or a substitute for jurisdiction-specific review.

## Findings and dispositions

Severity describes potential confidentiality, integrity or availability impact if an implementation inherited the gap. High does not mean an exploit was demonstrated. The linked documents contain normative controls and negative cases.

| Finding | Severity | Gap and revised solution | Evidence required before closure |
|---|---|---|---|
| AUTH-01 | High | JWT algorithm/key-source confusion, attacker-controlled discovery and unknown-key refresh amplification. [Identity](identity-and-personas.md) binds algorithms/keys to configured issuers and limits refresh, stale trust and negative caching. | Forged key sources, algorithm confusion, key rotation/outage and unknown-key flood tests. |
| AUTH-02 | High | Provider callback mix-up and ambiguous refresh/logout guarantees. Bind callbacks to issuer/client/transaction and qualify rotation/replay plus separate logout outcomes. | Real multi-provider callback, refresh reuse and stolen-token deauthentication journeys. |
| AUTH-03 | High | Revocation acceptance and watermark completeness were underspecified. Define durable acceptance and a total proposed five-second budget, including clock uncertainty and decision lifetime. | Missing/out-of-order updates, restarts, delayed audit and in-flight remote-write tests. |
| AUTH-04 | High | Authorization cache dimensions and compound writes could omit decision inputs. Bind all relevant resource/session/assurance context and enforce recursive group/child writes and current replay authorization. | Cross-context cache reuse and atomic denied-child write cases. |
| AUTH-05 | High | Privileged approval/recovery could bypass assurance or separation of duties. Require recent phishing-resistant step-up, independent human identities and conditional activation. | Indirect self-escalation, stale approval, recovery and emergency-access exercises. |
| DATA-01 | High | A trusted queue sender could assert a false tenant/resource/source. [Events](events.md) bind domain messages to committed publication evidence and current ownership. | Forged source, mixed-tenant references and unauthorized producer cases. |
| DATA-02 | High | Same event identity with changed content could poison deduplication. Store a canonical content digest and quarantine conflicting identity reuse. | Alter tenant/type/data while preserving source/id. |
| DATA-03 | High | A hold accepted after a check could race destructive work or stale completion. Serialize hold decisions with deletion admission, fence participants and bind receipts to current attempts/epochs. | Hold-versus-delete races and delayed callbacks, including already-dispatched work. |
| DATA-04 | High | A response could change after its audit intent was accepted. Freeze projection and audit description together; revalidate before disclosure and issue a new intent when changed. | Delay acknowledgment while policy, record or erasure state changes. |
| DATA-05 | Medium | Poison messages/replay could starve audit and erasure or amplify cost. Separate work classes, bound parsing/retries/fan-out, protect critical capacity and govern redrive. | Flood one tenant or work class and prove bounded impact and backlog preservation. |
| DATA-06 | High | Restored data could inherit old processed receipts and skip deletion. Reconcile data/inbox/receipts with current ledger and recovery epochs before readiness. | Restore a synthetic erased record alongside an old completed inbox entry. |
| DATA-07 | Medium | Cursors/idempotency lacked exact-contract interpretation binding. Bind cursor semantics; include exact contract and normalization semantics in mutation fingerprints while preserving one deduplication namespace. | Cross-version key/cursor reuse fails safely without a second mutation. |
| PLATFORM-01 | High | Untrusted CI inputs could override build roles, buildspecs or trusted artifacts. [Delivery](delivery-and-environments.md) separates untrusted execution from privileged release orchestration. | Build override, cache/artifact substitution and credential-exfiltration attempts. |
| PLATFORM-02 | High | EKS workload/operator constraints were too general. Specify admission, service-account selection, RBAC/exec, metadata-service isolation, network and secret boundaries. | Compromised-pod and ephemeral-environment escape tests against real AWS permissions. |
| PLATFORM-03 | High | Target HTTPS and WAF redaction could be mistaken for stronger guarantees. State ALB certificate-validation limits and separately govern WAF sampled requests/access logs. | Backend trust-boundary and synthetic PII probes through every logging surface. |
| PLATFORM-04 | High | An upstream signature could authorize a compromised image. [Image maintenance](image-maintenance.md) requires independent release-policy checks, digest revocation and recovery controls. | Trusted-publisher compromise, revoked artifact, kill-switch and eligible rollback exercises. |
| PLATFORM-05 | High | A read-only AI reviewer could exfiltrate material through network/tools. Confine evidence, credentials, egress and tool authority; treat source instructions as untrusted. | Prompt-injection/exfiltration probes and proof the reviewer cannot mutate approval or release state. |
| BOUNDARY-01 | High | Decoder/header ambiguity could give authorization and mutation different inputs. Apply the strict request profile below and document transport rejection separately from auth-first application handling. | Duplicate escaped property names, trailing documents, invalid Unicode, case-folded fields and proxy framing tests. |
| BOUNDARY-02 | Medium | Exact lookup and audit amplification still permit systematic enumeration or cost denial. Apply hierarchical request/work budgets without recording raw search values. | Hot-tenant, stolen-token scan, expensive page and audit-outage tests at representative load. |
| BOUNDARY-03 | High | Pooled IAM isolation and encryption guarantees were implicit. Declare the cell compromise boundary and scoped key/role lifecycle below. | Real role-policy negative tests, rotation/recovery evidence and explicit acceptance of pooled risk. |
| BOUNDARY-04 | High | Disclosure audit alone misses security-control changes; caller telemetry context can carry PII. Audit security mutations durably and allowlist generated correlation context. | Audit outage on privilege changes, context injection and immutable evidence access tests. |

## Strict application request profile

The canonical requirements live in `openapi.json` under `x-request-security`; they must be implemented, not merely annotated. These limits are proposed for the unreleased draft and require acceptance/compatibility review before publication:

- Maximum body **128 KiB**, JSON nesting **16 levels**, bearer credential **8 KiB**, and application header section **32 KiB**, with deployment-specific HTTP/1.1 and HTTP/2 accounting verified end to end. Lower edge limits must be made explicit before client qualification. Bound header/body read time, concurrent requests and slow clients too.
- Body operations accept `application/json` with no charset or UTF-8 only, and no compressed request content. Reject unsupported media/content encoding with `415`; oversized content with `413`. No body is expected for the declared GET operations. No form, XML, multipart or JSONP fallback.
- Decode exactly one UTF-8 JSON document. Reject invalid UTF-8/unpaired surrogate escapes, duplicate decoded member names at every depth, trailing documents, nonfinite numbers and unknown/case-mismatched property names. Ordinary case-insensitive struct matching or last-key-wins decoding is not sufficient. Authorization, validation and mutation use the same validated object.
- Require one unambiguous bearer credential. Reject duplicate security-sensitive headers instead of coalescing them. Invalid credential multiplicity yields `401`; after authentication duplicate version/idempotency/concurrency selectors yield safe `400`. Auth-first ordering does not require an edge to process unsafe HTTP framing, oversized traffic or transport abuse.
- Keep proxy and Go parser behavior consistent for Content-Length/Transfer-Encoding, path escaping, encoded separators and header normalization. Reject ambiguous framing before application processing. Trust forwarded routing/address headers only from configured ingress; clients cannot select tenant placement, table, issuer endpoint or internal destination.
- Transport rejection may close the connection or return an edge error without the application Problem/header shape. It must expose no business/version semantics. Once normal application handling is admitted, the documented authentication-first and safe Problem behavior applies.

The chosen strict JSON profile is stronger than simply accepting whatever a parser tolerates. [RFC 8259](https://www.rfc-editor.org/rfc/rfc8259.html) explains interoperability hazards around duplicate members and Unicode; [OWASP REST guidance](https://cheatsheetseries.owasp.org/cheatsheets/REST_Security_Cheat_Sheet.html) supports explicit input, size and media-type limits.

## Bounded abuse controls

Budget traffic by principal/session/client, tenant, operation class and cell; use coarse network controls before authentication. Do not let one tenant consume all cell concurrency. Account for candidate reads, retries, mutation transactions, audit writes, queue publication and erasure fan-out, not only HTTP counts. Reserve audit/revocation/erasure recovery capacity separately from optional integrations. Quotas and retry/backoff policies are explicit deployment configuration with owners; `429` communicates safe retry guidance without revealing another tenant's activity.

Detect anomalous enumeration by bounded outcome/work counters and restricted security analysis. No raw phone searches, arbitrary identifiers or JWT fields become metric labels. Any keyed query correlation needs separate privacy approval and short retention; a phone HMAC remains personal/linkable data. Rate limits do not authorize searching hidden fields or bulk export. Test peak legitimate traffic and attack skew together before accepting browser-visible p95; relaxing authorization or dropping required audit is not a performance remedy. [OWASP API resource-consumption guidance](https://api-security.owasp.org/editions/2023/en/0xa4-unrestricted-resource-consumption/).

## Storage isolation and key lifecycle

Retain the proposed pooled regional cell model, with an explicit compromise boundary. Separate API data, policy administration, outbox publisher, consumer, audit writer and restore identities; each receives only needed table/index/actions and destinations. Patient API roles cannot change IAM/KMS policy, restore/export tables, administrate holds/personas or read audit archives. Deny unused Scan/PartiQL/bulk/export surfaces. IAM `LeadingKeys` may strengthen isolation only where a trusted scoped credential policy actually constrains every allowed action/key. A shared role permitted all cell tenant keys is not tenant-isolated by a string prefix. [DynamoDB fine-grained policies](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/specifying-conditions.html).

Use customer-managed KMS keys scoped by environment, regional cell and data purpose, with separate key administration/use permissions and controlled recovery. Audit evidence has separate write/read/delete authority from the patient API. Encryption at rest protects stored media and unauthorized key use; it does not prevent a compromised authorized data role from reading plaintext. Tenants requiring a stronger workload/credential boundary need a separately qualified dedicated cell/account arrangement before onboarding. Application-level field encryption is a separate ADR if the threat model requires protection from storage readers; a key available to the same compromised application does not eliminate that compromise.

Phone-index HMAC keys, cursor-authentication/encryption keys and idempotency-fingerprint keys have separate purposes and versions; never reuse JWT signing keys. Use reviewed cryptographic libraries and managed wrapping/storage, bounded in-memory key caches and narrowly scoped workload access. Rotation and emergency revocation include reindex/reconciliation, bounded old/new key overlap, key-cache expiry, backup recovery and removal evidence. They must not add a KMS/Secrets Manager call to every lookup without a revised latency test. KMS rotation alone is not proof of erasure while usable historical key/ciphertext copies remain.

## Security auditing and telemetry trust

Disclosure intent follows [events](events.md): freeze the exact safe response projection and its field/reference description, acknowledge durable intent, then check validity before releasing it. A changed projection requires corresponding new intent; retries retain event identity only for unchanged content.

Also durably record privileged control changes: membership/persona grants and revocations, identity linking/recovery, erasure approval/hold transitions, key and provider trust changes, retained-version overrides, release-policy changes and emergency access. Include actor, independent approver where required, action, bounded reason code, target reference, policy/decision version, result and server correlation ID; exclude contact values and raw credentials. Couple security-state changes to durable audit/outbox evidence so audit failure cannot silently apply an unaudited privilege grant or hold removal. These management event schemas remain a required separate contract before those surfaces are implemented; do not overload a patient-disclosure intent with unrelated event semantics.

Unauthenticated floods use bounded/aggregated security detection; they do not synchronously enqueue unlimited audit events per rejected packet. Required disclosures and committed privileged changes cannot be sampled away. Protect audit read access, append authority, retention and deletion separately, and audit access to evidence itself. [OWASP logging guidance](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html).

Treat incoming request IDs, `traceparent`, `tracestate` and baggage as untrusted. Generate server request IDs, validate bounded trace context, discard arbitrary baggage/tracestate, and never let a caller sampling bit force unlimited tracing or costs. A consumer correlation link is accepted only under a qualified bounded profile; do not reflect arbitrary caller text into logs/headers. Source allowlists and private collectors remain mandatory. [OpenTelemetry sensitive-data guidance](https://opentelemetry.io/docs/security/handling-sensitive-data/).

## Acceptance and remaining risk

Each finding requires a named owner, executable negative cases and deployment-specific evidence in [API testing](api-testing.md). The review is not closed by prose edits. Security-critical unknowns block launch: concrete management/auth lifecycle contracts, bounded-fresh authorization implementation, legal-hold participant fences, trusted publication evidence, pooled-role acceptance, audit preservation/retention, restore proof, platform isolation and production release authority.

Before launch, run an independent tenant/field/authentication penetration test and a tabletop plus technical recovery exercise for stolen privileged tokens, compromised signing/base-image suppliers, exposed phone-index keys, policy-distribution outage, audit backlog expiry and lost regional capacity. Define incident owners, containment/revocation authority, evidence preservation and recovery approval. Test these in isolated synthetic environments; this specification grants no authority to exercise production.

Performance evidence must include these controls under the representative 20,000 requests/second workload and browser cohort. No claim of sub-100 ms performance, regulatory compliance, hardened deployment or full specification-tool compatibility follows from this design review.

## Design-time verification record

On 2026-10-22, independent consistency review found no remaining design blockers in these revisions. Canonical checks resolved 235 OpenAPI and 41 AsyncAPI local references; checked all seven bearer-authenticated, explicitly versioned operations and 413/415 declarations on all four body operations; validated 22 REST and six event fixtures with the schema-keyword subset used here; checked paired embedded event examples, eight negative event mutations and all 14 shared field-policy paths. JSON/YAML round trips, generated-source hashes, local links and document metadata also passed.

A maximally populated schema-valid compact create request using escaped non-BMP text measured 78,315 bytes against the proposed 131,072-byte bound. Some deliberately maximal phone strings fail numbering-domain validation, so this is a conservative schema bound rather than a valid clinical fixture. Extra whitespace remains subject to the transport limit.

These checks used temporary design-time tooling. They are not official OpenAPI/AsyncAPI conformance, generated-code compilation or implementation security tests. The concrete enforcement and launch evidence listed above remain required. Diagram rendering evidence is recorded separately in [diagrams](diagrams.md).
