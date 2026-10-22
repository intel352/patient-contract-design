---
id: TESTING-001
title: API testing and release evidence
version: 0.3.0
status: proposed
approval_status: pending
authors: [Codex]
approvers: []
created: 2026-10-22
updated: 2026-10-22
owner_role: API quality and security owner
supersedes: []
superseded_by: []
references:
  - ./project-standards.md
  - ./api-versioning.md
  - ./identity-and-personas.md
  - ./delivery-and-environments.md
  - ./document-governance.md
  - ../architecture.md
  - ../openapi.json
  - ../examples.json
  - ../generated/README.md
---

# API testing and release evidence

The user requires tests at every level, complete authenticated journeys, and compatibility verification for application and hardened-image updates. This document specifies the proposed evidence and gates. It does not supply application code, test implementations, CI configuration, cloud resources, or permission to run deployments or costly load tests. No application, identity integration, or performance test has run as part of this documentation work. Operational budgets, environment ownership, and implementation authorization remain separate decisions.

The service team owns backend and contract testing. Consumer teams own their frontend implementation and browser acceptance journeys. An API-owned headless authentication harness is useful integration evidence; it is not a substitute for the consuming application's browser/rendering evidence. No React architecture or browser automation product is mandated. Playwright or an equivalent consumer-approved tool may be qualified independently.

## Evidence by level

Tests must exercise distinct risks, not duplicate implementation statements at several levels. Use many focused unit checks, real dependency integrations, executable behavioral contracts, and a smaller set of complete deployed journeys. Mocks help isolate decisions and force failures; a suite composed entirely of mocks cannot establish cloud, authentication, transaction, or deployment behavior.

| Level | Trigger and gate | Environment | Accountable owner | Required evidence |
|---|---|---|---|---|
| Unit, property and focused fuzz tests | Every implementation PR and merge candidate; fast deterministic gate | Compiled Go packages using the exact reviewed project toolchain | Backend owner | Assertions, regression cases, race/concurrency results where relevant, reproducible seed for failures |
| Static contract and generated-artifact conformance | Every contract, generator, dependency or behavior change; drift checks on every implementation PR | Pinned qualified tools; compile generated Go/client artifacts | API owner | OpenAPI validation, reference/example checks, generation diff, compilation and declared compatibility impact |
| Dependency integration | Every implementation PR; affected boundary suites plus mandatory authentication, authorization, mutation and privacy regressions | Isolated real AWS resources and the built Go service; emulator runs are supplemental | Backend and platform owners | Actual requests/results, transactional invariants, IAM/isolation outcomes, safe failure/recovery evidence |
| Behavioral contract profiles | Every API-affecting PR and release candidate, including dependency/image changes | Candidate service with each supported representation and qualified consumer profile | API owner with consumer representatives | Given/When/Then outcomes and supported-version matrix; unexplained differences block release |
| Deployed end-to-end journey | Every application, contract, dependency, image or runtime PR candidate and every eligible automated image-maintenance candidate in an approved isolated environment; repeat for merge candidate/preview release | Actual candidate image on isolated EKS, regional DynamoDB/SQS and qualified real identity provider | Service quality owner; consumer owns application browser journey | Authentication through data changes and deauthentication, correlated state, safe evidence and cleanup |
| Load, browser latency and resilience | Before launch; capacity-sensitive changes; qualifying releases under the reviewed evidence-reuse policy | Approved production-representative synthetic AWS environment, never production by default | Performance, platform and consumer owners | Offered load, browser results, p50/p95/p99, success/error rates, skew, faults, cost and capacity envelope |
| Erasure and recovery | Every relevant mutation/security change; release-critical regression and scheduled recovery qualification | Isolated synthetic datasets and separately fenced restore targets | Privacy and platform owners | Deletion generations, in-flight limits, downstream completion, restore non-resurrection and retention evidence |

Purely editorial documentation changes can use document/reference checks; a behavior-changing specification is not editorial. Future implementation cannot claim conformance until its affected behavioral gates run. Reusing prior broad performance evidence requires a reviewed explanation of why it covers the exact candidate and deployment configuration. A passing small smoke suite is never substituted for required integration or end-to-end evidence. Environment or identity unavailability leaves a required gate pending/failed, not passed.

## Unit and contract checks

Compile and test actual backend code with the pinned project Go baseline and each proposed toolchain update. Exercise domain functions independently of HTTP and AWS clients. Use race detection and concurrent schedules for code that shares policy snapshots, caches, leases or request state. Fuzz and property tests should target input ambiguity and invariant boundaries, not manufacture coverage counts.

Required subjects include:

- Phone parsing and normalization: explicit national parsing country, international input, invalid/ambiguous input, country metadata updates, extensions stored separately, shared numbers and exact base-number matching. Tests do not claim that a syntactically valid number belongs to a patient or is reachable.
- Tenant/patient/phone-key construction and versioned tenant HMACs: no cross-tenant collisions or caller-controlled table/key injection; sensitive values never enter diagnostic output.
- Policy evaluation: action/search/read/write independence, tenant/row restrictions, parent-group and child-field permissions, current denial precedence, and exact trusted issuer/subject mapping. Same subject under different issuers and same email without verified linking remain distinct identities.
- Response projection and PATCH: omitted versus authorized-null values, readable empty phone arrays, recursive child updates, clearing groups, whole-array replacement, unknown-field rejection, and atomic rejection when any affected child is forbidden. Hidden stored presence must not alter policy metadata.
- Version classification and concurrency: required exact MAJOR.MINOR.PATCH header, path/header matching, exact served version, missing/malformed/unknown/retired distinctions, authorization-view-bound ETags, stale revision rejection and no silent fallback. Include auth-first `401` precedence and missing version/policy headers on authentication failures.
- Idempotency, cursor and event rules: scope, expiry, payload mismatch, policy changes, replay after erasure, duplicate/out-of-order events and lease generation. Clock-controlled unit tests verify logic without claiming elapsed cloud retention.

Validate canonical OpenAPI, examples, generated JSON/YAML, schema-derived ERDs and generated code against pinned tools qualified for the chosen OpenAPI dialect. Static examples prove schema compatibility only. Use real service responses for provider conformance, including error bodies and headers. Keep sensitive fixtures synthetic, bounded and purpose-specific.

## Event contract evidence

Apply [event contracts](events.md) alongside REST provider/consumer conformance. Validate AsyncAPI references and channel/operation direction, CloudEvents attributes and JSON encoding, event payload schemas and behavior against every declared consumer version. Generated types are supporting evidence, not proof of wire compatibility. Round-trip the actual SQS transport and any optional EventBridge adapter; distinguish the AWS transport envelope from the inner CloudEvent. Test duplicate publication, crash after side effects before acknowledgement, out-of-order revisions, unknown event types, replay, dead-letter recovery and erasure-generation dominance. Prove that neither envelopes, payloads, traces, failure dumps nor AI review artifacts contain contact values or bearer claims. A downstream fetch uses current workload authorization, never the originating user token.

Required adversarial cases also include forged tenant/source/resource/workflow tuples from an otherwise permitted producer; same source/id with changed canonical content; stale retention-decision generations; hold acceptance racing destructive-work admission; callback receipts from old attempts/recovery epochs; and old processed inbox entries restored alongside resurrected data. Exercise provenance retention, poisoned-message quarantine, outbox/inbox recovery and per-tenant/work-class budgets. Merely validating CloudEvents shape or granting SQS SendMessage is not proof of these invariants.

## Security review regression matrix

The [security review](security-review.md) finding IDs are traceability keys for required implementation evidence, not claims that these tests already exist. Use separate low-privilege and attacker identities, real infrastructure where the boundary depends on AWS, and synthetic sentinels only.

| Findings | Required cases and observable outcome |
|---|---|
| AUTH-01–02 | Algorithm/key confusion; token-provided key URLs; unknown-kid floods, stale JWKS and issuer outage; callback issuer mix-up; refresh reuse and logout distinctions. No arbitrary fetch, authentication bypass or unbounded key work. |
| AUTH-03–04 | Missing policy sequence, falsely fresh heartbeat, clock uncertainty, restart, delayed reads/audit, dispatch-versus-commit race, cross-resource/session/assurance cache reuse and denied compound PATCH. Measure the complete proposed five-second bound from durable acceptance. Unproven freshness produces safe 503; no unauthorized disclosure/mutation. |
| AUTH-05 | Privileged actions without recent qualified assurance, independent approver using the same human identity, indirect self-grant, approval state changed before activation, account recovery and emergency role abuse. All prohibited paths fail and record protected evidence. |
| DATA-01–06 | Execute the event adversarial cases above plus audit projection freezing: change rights/record/erasure state while SQS acknowledgment is delayed. Never release changed bytes under an old disclosure intent or accept stale completion as current erasure proof. |
| DATA-07 | Reuse idempotency key across exact API versions with differing semantics: 409, one logical mutation and one deduplication namespace. Cross-version/codec/ordering/normalization cursor reuse gives generic 400 without data or work amplification. |
| BOUNDARY-01 | Duplicate decoded names including escaped equivalents, case-mismatched fields, invalid UTF-8/unpaired surrogates, trailing JSON, excessive depth/bytes, unsupported/compressed media, duplicate sensitive headers and HTTP framing/path ambiguity across ingress and Go. Same accepted object reaches policy and mutation; 413/415/application-safe errors or pre-admission transport rejection as specified. Test boundary values and largest valid encoded requests too. |
| BOUNDARY-02 / DATA-05 | One principal/tenant scans many phones or floods expensive pages/retries; queue redrive and poison work compete with audits/erasure. Bound work and cost, preserve critical recovery and other tenants, and report genuine latency/error impact. |
| BOUNDARY-03 / PLATFORM-02 | Use real workload credentials to attempt foreign cell/environment tables, policy/hold administration, audit archive reads, export/restore, wrong service account, pod exec and node metadata credentials. Verify denied boundaries and explicitly record any accepted shared-cell access. Test key rotation/overlap/recovery without raw key output. |
| BOUNDARY-04 / PLATFORM-03 | Inject synthetic sensitive values into body, path, query, raw errors, request IDs, baggage/tracestate and sampling flags; inspect application, ALB, WAF logging and sampled requests, SDK, OTel and review artifacts. Test control-audit outage before privilege/hold mutation and backlog expiry protection. |
| PLATFORM-01,04,05 | Untrusted build overrides/PassRole, poisoned caches/artifacts, substituted attestation, compromised publisher key, revoked digest, stale release registry, AI prompt injection and network/tool exfiltration. No unapproved input obtains signer/deployer authority; image-only policy stops on out-of-policy deltas. |

Do not fulfill this matrix with always-denied mocks. Assert successful authorized cases beside denials; verify tenant/field/error semantics on every supported representation. Security findings remain open until accountable owners review evidence for the actual release/configuration. Independent penetration testing and incident/key/restore exercises are separate launch gates, not replacements for deterministic regression tests.

## Real dependency integration

Run the candidate Go implementation against isolated real DynamoDB, SQS, identity and network/IAM boundaries in the intended AWS region. EKS deployment, ingress, workload credentials and telemetry configuration are part of the deployed evidence. DynamoDB Local, local queues, stubbed JWTs or simulated networking can accelerate development and failure exploration, but cannot certify AWS consistency, IAM, quotas, latency, provider revocation or cloud recovery.

Verify transactional create/update/restrict operations across authoritative patient items, phone mappings, inventory, idempotency and durable outbox. Inject revision conflicts and conditional failures; prove no partial state commits and that retries do not create duplicate logical records. Test same-item transaction deduplication, action/byte limits and encoded item size. Exercise strongly consistent reads, shared-phone fan-out, partial `BatchGetItem` results including `UnprocessedKeys`, and bounded retries. Query followed by BatchGet is not a snapshot: change/restrict a patient between stages and assert revalidation or a safe failure, not stale disclosure or a fabricated no-match.

Use actual SQS acceptance and an idempotent audit consumer. Test durable acceptance before disclosure, ambiguous sends, duplicate delivery, unavailable transport, sink outage and backlog protection. Synthetic fault injection can force otherwise rare failures, but success-path proof still crosses the real service boundary. Verify mutation outbox recovery after missed stream notifications through retained pending records and reconciliation. Separate deterministic clock/fault tests from elapsed-time evidence for Streams, queue retention and leases; do not label a simulated long outage as a real multi-day observation.

Prove tenant/environment isolation through both application and AWS credentials. A Kubernetes namespace or name prefix is not the isolation test. Destructive suites use dedicated disposable tables/queues or an equivalently proven boundary, scoped keys and identity clients, registered cleanup, and orphan detection. Never borrow production patient data, secrets, queues or tables to make a test representative.

## Behavioral compatibility profiles

Maintain executable Given/When/Then profiles linked to requirements, OpenAPI operations and the supported release registry. Profiles specify externally observable semantics rather than a copy of handler internals. For example: given a current tenant membership with phone-search permission but no comment-read permission, when an exact lookup matches two permitted patients, then only those patients and readable fields are returned, comment is omitted, and no comment value enters telemetry.

Run each exact supported major/minor/patch representation on its approved implementation mapping, including explicit retention overrides and declared holds. Usage alone does not silently extend the supported set. Include supported generated-client/consumer generations. The matrix comes from the release registry, not a hard-coded assumption that only the latest API matters. Exercise missing-version rejection, exact selection, configured major/minor/patch counts, override races, shared-deployment decommissioning, mixed implementation-digest rollout and rollback-eligible bundles.

Before GA, exercise each proposed representation through the environment/cell-scoped candidate registry and isolated candidate route defined by [API versioning](api-versioning.md). Bind the evidence to candidate/build ID, contract and image digests, policy generation and expiry; only explicitly allowed synthetic principals/tenants may use it. Prove that an ordinary production caller, another environment, an expired candidate or a forged candidate identifier cannot gain admission. Request/response headers remain exact numeric `MAJOR.MINOR.PATCH`; reject prerelease/build suffixes and partial selectors. Candidate qualification must not add production support, consume a retention slot, start a GA clock or expose customer data. The design's `info.version: 0.1.0` and proposed first GA `1.0.0` do not establish an available candidate.

Test production-cell synthetic qualification before GA, the atomic GA/support metadata transition, and conditional activation of the approved generation in each ready cell. Force an unready cell, stale routing generation, conflicting support update and failure partway through activation. Assert no claim of atomic cross-Region rollout: unhealthy cells preserve prior qualified service only for still-authorized views, a supported but unavailable view returns its availability error, and excluded views cannot remain admitted. Public traffic canary starts only after GA. On failed public canary, verify eligible rollback, immutable GA/version history, no version-number reuse or GA-clock reset, and no resurrection of a retired route. Exercise explicit transition holds rather than silently retaining a version because rollout is slow.

Structural schema diffing cannot infer whether changed flags, defaults, normalization, ordering, validation, null/absence handling, error precedence, pagination, phone matching, idempotency or concurrency behavior are compatible. Profiles must cover these behaviors. Compare the latest merge candidate with immutable released profiles and current main; unexplained divergence or an unjustified lower SemVer classification blocks release. Changing a test expectation is not proof that a breaking change is compatible.

Repeat the same functional journey through an older exact supported representation served by its current approved maintained implementation. Assert that its shape/behavior remains supported while current field restrictions, tenant policy, revocation and erasure still apply. Old API versions cannot preserve withdrawn access. Preserve exact-selector behavior and truthful version headers during deployment; never silently reroute an unsupported or retired version.

## Complete authenticated end-to-end journey

Use a qualified test identity and real configured provider flow, obtaining an access token through Authorization Code with PKCE and registered callbacks. Do not replace the provider exchange with a locally fabricated success JWT. A service-owned headless browser can verify the protocol without constructing the product frontend; the consuming team supplies its own real UI journey. Workload-client flows have separate adapter conformance cases and cannot stand in for the human login flow.

Carry a per-run context through the entire journey: environment and regional endpoint, external test identity and resolved principal, approved tenant scopes/personas, requested and served API versions, synthetic phone inputs, created patient IDs, returned ETags/revisions, cursors, idempotency keys and erasure-request IDs. Capture outputs from each real operation and feed them into later operations and asynchronous callbacks. Fixtures may seed known test memberships, but must not hard-code newly created resource IDs, precomputed successful responses, or a different hidden fixture to make a later step pass. Keep tokens and restricted references in protected in-memory/test state, not public reports.

The release-critical journey must demonstrate:

1. **Unauthenticated boundaries:** each existing operation rejects missing/invalid credentials; even invalid/retired-version requests return the auth-first `401`, with Bearer challenge and no version/policy metadata. Protected schema/docs/lifecycle delivery is tested when that separately defined hosting surface exists. CORS preflight and private probes remain transport/operational boundaries, not alternate patient APIs.
2. **Authenticate and discover:** complete real sign-in, obtain the access token, call `/v1/session`, resolve the immutable application principal, choose a permitted membership/region and record the negotiated version. Test wrong issuer/audience/client/token purpose and untrusted claim mappings separately.
3. **Create and carry state:** create synthetic patients with fresh idempotency keys; retain returned IDs, authorized projections and ETags. Repeat a request and prove one logical creation, then test mismatched payload reuse without partial changes.
4. **Read, search and update:** get the created patient, perform conditional PATCH with the returned ETag, carry the updated revision forward, and reject the stale prior ETag. Look up the same normalized base phone; create a second permitted patient sharing that phone and verify both authorized matches. Verify a legitimate no-match separately. Bound paging and reuse only returned opaque cursors; cross-principal/tenant/filter/policy/expiry reuse fails safely.
5. **Enforce scope and fields:** attempt another tenant's IDs, cursors and idempotency keys without access. Exercise permitted child updates, denied comment access, absent/null/empty semantics and whole-group clearing. Verify no withheld value appears in responses, errors, ordinary logs/traces, queue payloads or browser diagnostics; approved audit references remain restricted.
6. **Change authorization:** apply a verified administrative revocation/persona change, retaining the previously issued access token. Test current policy enforcement within the documented application-accepted freshness bound, stale-node failure and a final validity check after slow storage/audit work. Default and customized personas, multiple assignments, delegation constraints and direct/indirect self-escalation need negative cases as well as allowed actions.
7. **Deauthenticate:** execute the provider adapter's specified logout/revocation flow and discard consumer-held credentials. Test refresh rejection when that qualified flow revokes the refresh credential; a browser logout that only clears local state must not be misreported as server revocation. Ordinary offline validation may still accept an old signed access token until its documented validity window unless application denial state blocks it. Independently test the proposed five-second bound from application acceptance of revocation; do not assert that Cognito logout instantly invalidates every ordinary JWT verifier.
8. **Cleanup and verify:** remove synthetic resources through the authorized lifecycle, record completion/retention evidence, revoke test credentials and reconcile environment leftovers. Cleanup failure is visible and cannot be hidden by a successful main assertion.

The existing seven-operation contract does not define application logout, persona administration, erasure approval administration or a global identity broker. Tests use a separately qualified provider adapter or approved management contract for those boundaries. Until those contracts and implementations exist, their journey coverage is an explicit traceability gap; inventing a successful mock endpoint is not end-to-end proof. Test setup privileges are separate from the persona whose behavior is under test, and setup must not bypass the invariant being asserted.

## Erasure, privacy and recovery tests

Exercise accepted, retained, failed and completed cases, with authorized decisions and synthetic records. Race lookups, detail reads, updates, imports, retries and delayed audit acknowledgments against the restrict transaction. New strong reads must honor restriction; earlier response work follows the bounded in-flight policy and must drain before active-system completion. Already transmitted bytes are not retroactively retractable. Worker failures never reactivate a restricted patient.

Prove removal of registered mappings/copies and deletion-aware suppression of delayed writes or reimports. Restore an actual synthetic backup into isolated non-serving tables; replay the protected erasure ledger, reconcile mappings/outbox/control state and verify absence before enabling any test traffic. A unit-test clock can check expiry calculations; it does not prove real DynamoDB TTL, SQS retention, backup expiry or elapsed recovery behavior. Test real expiry/retention over approved observation periods or record the unverified interval explicitly. Distinguish active-data-erased, backup-expiry-pending and legally retained evidence rather than declaring all copies deleted.

Inject recognizable synthetic sentinels in every sensitive field and through success/failure paths. Check application, ingress/WAF, AWS SDK, worker, OTel, crash and approved browser diagnostic outputs. Assert no ordinary telemetry disclosure even for privileged callers or failed validation. Test reports summarize violations without copying the offending sensitive payload; protected diagnostic access and retention follow the same policy.

## Browser latency, load and resilience

Run the actual Go/EKS → strong DynamoDB reads → durable SQS audit path at the approved 20,000 aggregate requests/second scenario, with representative synthetic retained data, endpoint mix, skew, record sizes and shared-phone fan-out. Use open-loop load so slowdown does not lower offered traffic unnoticed. Run representative consumer browser probes concurrently against the same candidate/capacity envelope. Include required token refresh, cold/warm connections, actual network geography, UI/render marker and the declared measurement population; an API timer or API-owned harness alone cannot certify browser-visible p95 below 100 ms.

Record offered/completed traffic, successful-and-fast ratio, p50/p95/p99, errors/timeouts, retries, throttling, unprocessed keys, authorization freshness, audit acknowledgment/backlog and worker recovery. Fast errors, excluded slow requests or a throttled load generator must not improve the result. OTel traces explain selected paths; complete request histograms and browser measurements establish the population. Validate telemetry privacy and bounded labels under load.

Exercise approved bursts, AZ/pod restart, network/dependency failure, policy-distribution loss, audit outage and recovery with explicit stop limits. These are planned synthetic-environment exercises with resource budgets and accountable owners, not authority to run production chaos or a costly benchmark now. Missing geography, availability thresholds, workload distribution or cost approval prevents claiming acceptance.

## Hardened image and toolchain updates

A base-image-only rebuild still creates a different executable runtime environment. Rebuild from the same approved application source and lockfiles with the exact reviewed base digest/toolchain, record provenance/SBOM and scan/signature evidence, and test the resulting immutable image. Do not treat unchanged Go source or a passing vulnerability scan as behavioral compatibility proof.

For every such PR, release or eligible automated maintenance candidate, rerun relevant unit/contract/integration checks, every release-critical deployed authentication-to-deauthentication journey, and the supported API/consumer compatibility matrix against that image. Qualify TLS negotiation and CA trust to configured IdP/AWS services, JWKS retrieval, DNS, clock/expiry handling, non-root UID/GID, read-only filesystem, explicitly writable temporary paths, signals/shutdown, probes, network/workload credentials and telemetry. Confirm restricted containers still enforce privacy and deadlines. Toolchain/compiler changes also require rebuilding and testing actual Go packages; a label or version-string change proves nothing.

Performance-sensitive runtime/image changes require new load/browser evidence. Otherwise a reviewed reuse decision must identify applicable current capacity evidence; release-critical compatibility and runtime journeys still run. Compare behavior across mixed-version rollout and eligible rollback, preserving tombstones and denial state. If a dependency/image change alters observable contract behavior, classify it under API SemVer rather than assuming every infrastructure update is automatically a compatible patch.

## Gates, ownership and evidence quality

Bind results to the exact source/merge candidate, image digest, base/toolchain versions, configuration and policy versions, contract patches, identity-adapter profile, test seed, environment and timestamps. Retain a compact supported-version/persona/operation matrix, failures and disposition, cleanup result, and trace/metric references in protected evidence storage. Never attach credentials, raw claims, real patient data or production dumps to CI artifacts or AI review.

Required failing tests block merge/promotion. Report flaky or quarantined coverage with an owner, risk and expiry; quarantine is not a passing result, and release-critical paths cannot silently disappear. Retries retain the original failure and reason instead of replacing it with a clean-looking result. Ordinary application, dependency, contract and policy changes require human and independent AI review of test relevance and compatibility evidence. Eligible image-only maintenance uses the expressly approved standing policy to assess unchanged-contract classification and evidence without per-update human approval; independent AI review and every required test/provenance/lifecycle gate remain mandatory. An out-of-policy input, unexpected behavior change, missing evidence or blocking review finding stops automatic promotion and returns to the prescribed review/remediation path. Neither a coverage percentage nor generated assertions establish correctness.

Use branch/condition coverage to locate untested decisions, then require targeted assertions for tenant isolation, field permissions, auth/version precedence, concurrency, erasure and failure recovery. There is no arbitrary 100% coverage target or permission to inflate line coverage with tests that mirror implementation. A regression test should fail when the violated contract is deliberately broken. Real-boundary evidence, negative cases and supported-consumer behavior are the release criteria.

Remaining implementation decisions include the approved field/persona matrix and management interfaces, adapter logout semantics, full OpenAPI tool qualification, exact CI ownership/environment budgets, browser consumer cohorts, workload distributions, fault/retention observation periods and evidence-reuse rules. Record accountable decisions in the project's follow-up queue before operational adoption; this document does not resolve them by inventing test infrastructure.
