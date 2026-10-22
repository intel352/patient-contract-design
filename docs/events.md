---
id: EVENTS-001
title: Event contracts and delivery
version: 0.2.0
status: proposed
approval_status: pending
authors: [Codex]
approvers: []
created: 2026-10-22
updated: 2026-10-22
owner_role: API and event contract owners
supersedes: []
superseded_by: []
references:
  - ./project-standards.md
  - ./api-versioning.md
  - ./api-testing.md
  - ./identity-and-personas.md
  - ./delivery-and-environments.md
  - ./document-governance.md
  - ../architecture.md
  - ../contracts/asyncapi.json
  - ../contracts/event-examples.json
---

# Event contracts and delivery

Use AsyncAPI for asynchronous channel/message contracts alongside OpenAPI for REST and CloudEvents for the event envelope. This is a proposed documentation-only design, with machine-readable schemas and synthetic examples. It creates no worker, broker, consumer, CI workflow or deployed event route. The [project standards](project-standards.md), [architecture](../architecture.md), current authorization/privacy rules and [API testing](api-testing.md) constrain this design.

Decision context: [ADR 0004](../decisions/0004-separate-rest-and-event-contracts.md). Equivalent JSON/YAML exports and checksums are in [generated artifacts](../generated/README.md).

## Contract roles and tooling qualification

| Contract | Responsibility | Proposed format |
|---|---|---|
| [OpenAPI](../openapi.json) | REST operations, typed JSON requests/responses, HTTP headers and errors | Existing REST contract remains authoritative and unchanged; REST responses are not wrapped in CloudEvents |
| [AsyncAPI](../contracts/asyncapi.json) | Publisher application, channels, allowed messages, schemas and delivery/security requirements | AsyncAPI 3.1.0; document `info.version: 0.1.0` is an unreleased proposal |
| CloudEvents | Common event identity, source, type, time, schema reference and data envelope | CloudEvents specification 1.0.2, JSON structured content, wire `specversion: "1.0"` |
| Event payload schema | Meaning and compatibility of one domain/audit message | Independent proposed `1.0.0` schemas and `.v1` event types; no released event versions are claimed |

The official latest specification resolves to **AsyncAPI 3.1.0**, verified on 2026-10-22. Its operation `action` is from the described application's perspective. This contract models the patient service's publishing boundary: the transactional-outbox relay and API audit producer both **send**. Receiving worker applications need their own reviewed `receive` operations and real queue mappings; do not automatically invert this document or pretend one SQS queue broadcasts to every consumer. [AsyncAPI specification](https://www.asyncapi.com/docs/reference/specification/v3.1.0).

| Tooling area | Primary-source evidence | Qualification required here |
|---|---|---|
| Authoring/validation | The AsyncAPI CLI supports creation, validation and bundling. Parser-JS describes 3.x support. | Pin exact releases and prove 3.1.0 validation, references, message examples, SQS 0.3.0 bindings and our closed schemas. A broad 3.x claim is not this proof. |
| Documentation/generation | AsyncAPI Generator uses templates. | Qualify the selected renderer/template and preserve semantic distinctions, version metadata and protected document access. |
| Go models | Modelina documents Go structs/enums, but its inspected input-support table stops at AsyncAPI 3.0.0. | Verify exact 3.1.0 handling or a reviewed deterministic schema extraction path; do not silently downgrade the source contract. Models do not supply secure SQS consumer behavior. |
| Go CloudEvents | The CloudEvents project publishes an official Go SDK. | Qualify the pinned SDK, structured JSON mapping, extensions and chosen AWS adapter; no working generated Go transport is assumed. |

Sources: [CLI](https://www.asyncapi.com/docs/tools/cli), [Parser-JS compatibility](https://github.com/asyncapi/parser-js#api-documentation), [Generator](https://www.asyncapi.com/docs/tools/generator), [Modelina inputs and Go output](https://github.com/asyncapi/modelina#features), [CloudEvents Go SDK](https://github.com/cloudevents/sdk-go). Tool versions remain unselected. This work does not install or run those tools, generate Go code or claim parser/runtime conformance. Adoption is conditional on the exact-version qualification gate; the draft is still useful as the design contract while that evidence is outstanding.

The canonical AsyncAPI document's proposed revision is recorded separately as `x-document-metadata.version: 0.2.0`; public draft `info.version` stays `0.1.0`. Its `x-security-invariants` profile is normative for publisher/consumer behavior, and each operation/message points to it. These extensions document required enforcement; a validator or generated model does not implement provenance, content-digest checks, hold fences, frozen disclosure or recovery safety. The payload `1.0.0` identities remain unreleased proposals, so these draft changes do not rewrite any released schema.

## Regional topology and publication

The baseline reuses regional DynamoDB outbox/SQS workers. Domain notifications go to a cell/environment domain queue; disclosure intents go to a separate restricted audit queue with different IAM permissions, consumers and retention. Each queue has a regional DLQ and encrypted, access-controlled recovery path. Queue URLs, accounts, keys and source-to-cell ownership come from trusted deployment inventory, never message data. The AsyncAPI SQS binding is pinned to 0.3.0; queue names are logical examples, not created resources. [SQS binding](https://www.asyncapi.com/docs/reference/bindings/sqs).

Separate capabilities within those existing workloads: the outbox relay reads/claims approved outbox records and sends domain events; the API sends audit intents; the audit writer receives those intents and appends its archive. None gains queue administration, `PurgeQueue`, unrestricted redrive, backup/export/restore, legal-hold administration or patient-table access merely because it handles events. Grant an erasure worker only its registered participant's required deletion/control operations. Queue/KMS policies name approved roles and destinations, require TLS, and constrain AWS-service publishers with applicable source-account/source-ARN conditions. Encryption protects storage, but does not authorize a tenant or authenticate the event's claimed source. [AWS SQS security](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-security-best-practices.html), [encrypted-queue least privilege and confused-deputy controls](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-least-privilege-policy.html).

A pooled-table role may still reach multiple tenants in its cell. Make that residual compromise boundary explicit: an `ORG#` prefix and a CloudEvents `tenantid` are not IAM isolation. Use trusted tenant-bound key construction and validate every batch/transaction member; test any `dynamodb:LeadingKeys` restriction against the actual composite keys, indexes and actions instead of assuming a shared role inherits the caller's tenant. Do not introduce per-request STS calls without a separate performance/security design. Restrict ordinary roles from inventory/export/scan/restore and key-management operations they do not need. [DynamoDB fine-grained conditions](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/specifying-conditions.html).

A domain queue is a work queue for one logical worker application with competing replicas. Registered handlers execute through that application's durable dispatch/completion model. Independent subscribers that each require every event need distinct delivery queues and a qualified fan-out path. The baseline does not require Kafka or a new broker. A future regional EventBridge fan-out adapter is optional and cannot fan patient references into an unapproved region/account.

```mermaid
sequenceDiagram
    participant API as Regional Go API
    participant DB as DynamoDB patient and outbox
    participant Relay as Outbox relay
    participant Queue as Regional SQS domain queue
    participant Consumer as Domain worker
    participant State as Consumer state and inbox
    API->>DB: Transaction patient or erasure-workflow revision plus immutable outbox event
    DB-->>API: Transaction success
    Note over API,DB: Event time is recorded application time, not a database commit timestamp
    Relay->>DB: Pending discovery then strong read and conditional lease
    DB-->>Relay: Committed event and ownership
    Relay->>Queue: Send structured CloudEvent with stable source and id
    alt Send response is ambiguous
        Relay->>Queue: Retry identical event identity and content
    end
    Queue-->>Relay: Durable transport acceptance
    Relay->>DB: Record handoff evidence, retain required completion tracking
    Queue->>Consumer: Deliver event, possibly duplicated or out of order
    Consumer->>State: Atomically check inbox and revision, restriction, tombstone fences
    Consumer->>State: Persist effect or durable work intent plus inbox in that transaction
    State-->>Consumer: Durable effect or work acceptance
    Consumer->>DB: Record required processing completion idempotently
    Consumer->>Queue: Delete using receipt handle after durable acceptance
    Note over DB,State: Transport acknowledgment alone is not erasure or business completion
    Note over Consumer,State: Erasure handler fences and drains external effects before participant completion
```

Create/update/restrict transactions record the event with the patient mutation, revision, idempotency and required mapping changes. An approved erasure's transition into restricted processing atomically records its workflow, patient restriction/deletion generation, `retentionDecisionGeneration` and `patient.erasure.started.v1` outbox event. That event dispatches to the existing domain worker's authorized erasure handler, which strongly reads the authoritative workflow/generation, verifies holds and acquires the required conditional participant fence before destructive work. It is a committed start fact, not an unconditional purge command. A hold or retention-decision change supersedes the recorded decision generation; an old start cannot authorize deletion. Resumption requires an explicit authorized transition and a new immutable `patient.erasure.started.v1` event carrying the current decision generation, not modification of a retry. An ordinary `patient.restricted` event does not imply erasure. Registered participants report durable idempotent completion evidence to the authoritative workflow; only its verified `completed` transition atomically records `patient.erasure.completed.v1` with the decision generation under which completion was verified. Failed transactions publish nothing; ambiguous transaction results reconcile the persisted mutation/idempotency record before publication. Events are allocated once per distinct accepted transition, not recreated on each outbox lease or retry.

Streams are wake-ups, not the durable outbox. The architecture's sharded pending index, strong read/conditional claim, lease recovery and periodic reconciliation remain required beyond the stream retention window. Retain durable pending state until the required handoff **and processing completion evidence** exist. A successful SQS send can mark delivery accepted; it must not mark an erasure workflow completed. Preserve or reconstruct uncompleted work if queue retention, poison messages or consumer outages threaten delivery. Downstream completion belongs to the registered participant/workflow inventory, not an assumption that the queue is empty.

Read operations do not emit `patient.updated` or a general domain event. The existing required application audit intent remains separate: API authorization/projection → durable audit queue acceptance → protected response. This is not a DynamoDB mutation-outbox write per read. Failure/ambiguity handling, same-event retry, admission stopping before backlog expiry and emergency preservation of accepted audit records follow the architecture. OTel traces, metrics and ordinary operational logs do not become CloudEvents or mandatory audit substitutes.

Freeze the projected response and its audit description together: exact patient references, internal revisions, field categories and policy version. After durable acceptance, revalidate authorization freshness, request deadline and applicable erasure/disclosure fences before writing response bytes. Never enlarge or substitute the response after its audit intent was accepted. If re-evaluation changes the disclosure, either abort or create a fresh projected response and a new audit identity within the remaining deadline; do not mutate/reuse an accepted event ID. An accepted intent for an aborted response remains an intent, not evidence of actual delivery. Archive writes must be idempotent and tamper-evident, with append rights separated from archive administration/deletion; record privileged archive access without copying patient/event payloads to ordinary telemetry. These are proposed integrity controls, not a claim of compliance. [OWASP logging protection](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html), [NIST SP 800-53 AC-6 and AU-9](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-53r5.pdf).

## CloudEvents profile and payloads

Use structured JSON: the SQS message body is one complete CloudEvent, with logical content type `application/cloudevents+json`; `datacontenttype` is `application/json` and `data` is a JSON object. SQS has no application HTTP response `Content-Type` to infer; the queue contract identifies the encoding. Do not split authoritative attributes across message headers, use `data_base64`, publish batches inside one event or nest the envelope in an undocumented wrapper. [CloudEvents JSON format](https://github.com/cloudevents/spec/blob/v1.0.2/cloudevents/formats/json-format.md).

This project requires `specversion`, `id`, `source`, `type`, `time`, `datacontenttype`, `dataschema`, `tenantid` and `data`. The example source is `urn:patient-contact:preview:us-east-1:cell-example:domain`: environment/region/cell identify the producer context; neither tenant nor patient identifiers belong in `source`, subject, channel names or transport routing labels. The lowercase `tenantid` extension is a restricted opaque reference, not a grant or region selector. Patient references exist only inside the restricted payload. The profile deliberately allows only reviewed extensions and fields; CloudEvents itself permits broader extensions. [CloudEvents 1.0.2](https://github.com/cloudevents/spec/blob/v1.0.2/cloudevents/spec.md).

Allocate an event UUID once and preserve both `source` and `id` across retries, relay, DLQ redrive and replay. Uniqueness/deduplication uses the pair. Preserve the original application-recorded time and schema reference. For domain mutations, the timestamp is prepared with and persisted by the successful transaction; it is not an invented DynamoDB commit timestamp. For audit, it is the recorded authorized disclosure-intent time, not browser receipt. Neither timestamps nor queue order establish aggregate order.

| Type suffix, under `patientcontact.` | Emission point | Allowed payload |
|---|---|---|
| `patient.created.v1` | Committed creation | Patient reference, internal record revision, access generation, deletion generation |
| `patient.updated.v1` | Committed contact mutation | Same references/revisions; no before/after snapshot or changed values |
| `patient.restricted.v1` | Committed patient restriction | Same references/revisions plus restriction-policy version; increments restriction generation |
| `patient.erasure.started.v1` | Approved erasure starts or resumes restricted processing atomically with the deletion/retention-decision generations and outbox | Patient/erasure references and generations; authorized handler checks current workflow and fences before purge |
| `patient.erasure.completed.v1` | Verified workflow `completed` transition | Patient/erasure/evidence references, terminal deletion generation and verified retention-decision generation; includes backup/restore and registered-copy obligations, not merely `active_data_erased` |
| `audit.disclosure.intent.v1` | Authorized projection accepted by durable audit transport before response | Internal actor/request/patient references, exact disclosed field categories per patient, policy version and decision; no proof of delivery |

Internal record revisions/access generations are event ordering metadata, not the REST principal-bound ETag, identity claims or permission grants. The opaque references remain linkable restricted data. Audit field paths match OpenAPI `FieldAccess` exactly, including `phones[].e164` and `phones[].extension`. The audit schema's current operation list covers patient lookup/detail/create/update disclosure; additional audit purposes require an explicit schema, not arbitrary strings or reuse as generic telemetry. No phone numbers, phone hashes, names, address values, comments, search input, credentials, raw claims or free-text error content are allowed. [Synthetic fixtures](../contracts/event-examples.json) illustrate all six messages.

Both erasure payloads require a positive `retentionDecisionGeneration`, distinct from `deletionGeneration`: the former versions approved retention/hold decisions, while the latter identifies the erasure fence for this patient. Neither authorizes an action without current authoritative workflow checks. Domain schemas accept only `:domain` sources and the audit schema only `:audit` sources; this syntactic separation supplements, and never replaces, trusted producer/provenance validation.

The optional EventBridge adapter passes the **serialized full CloudEvent** as `PutEvents.Detail`, with an approved native `Source` and matching `DetailType`. Delivery has the native EventBridge outer envelope; its parsed `detail` contains the CloudEvent. The receiving adapter must unwrap `detail` before CloudEvents validation and use the inner `source`/`id` for deduplication. Native EventBridge `id`, `time`, `version: "0"` and region metadata do not replace CloudEvents identity or `specversion`. Check each PutEvents result entry; partial failures retry the same inner event. Qualify the final target's input transformation explicitly. [PutEvents entry](https://docs.aws.amazon.com/eventbridge/latest/APIReference/API_PutEventsRequestEntry.html), [EventBridge envelope](https://docs.aws.amazon.com/eventbridge/latest/ref/events-structure.html).

## Consumer correctness, restrictions and erasure

Expect at-least-once, duplicate and out-of-order deliveries. Consumers validate allowed source/type/schema, cell/residency, bounded size and reviewed fields before processing. Queue delivery is authenticated with scoped IAM workload identity/SigV4 and queue/KMS policies; event claims never grant patient access. Any subsequent patient API call still requires its own valid bearer credential and current backend authorization. Contract/schema access remains authenticated and authorized. [SQS at-least-once behavior](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/standard-queues-at-least-once-delivery.html).

For domain events, bind the complete tuple `(source, id, tenantid, patientId, type, schema, generations)` to its committed outbox record or protected immutable publication evidence before dispatch. Retain that verification evidence through the permitted replay horizon; a self-asserted source string is not provenance. Independently resolve the tenant's current cell and migration generation and validate any authorized historical publisher during migration. For erasure, load the workflow by its trusted tenant scope and verify that its patient, request ID, approved action and generation all agree. Reject cross-tenant identifier mixtures, unmatched evidence and unsupported destinations before an effect. Audit intents use the separate authorized API producer boundary and are not forced through a new per-read DynamoDB outbox. This binding addresses object-level authorization across asynchronous boundaries; broker access alone is insufficient. [OWASP object-level authorization](https://owasp.org/API-Security/editions/2023/en/0xa1-broken-object-level-authorization/).

Persist an inbox keyed by consumer identity plus event `source`/`id`, transactionally with the local effect or durable work intent. In that same transaction, condition the effect on the current record revision and restriction/deletion fences; a prior strong read alone does not close the race. A crash between a bare “seen” write and the side effect must not lose work. External effects need their own idempotency key, destination-enforced fencing and durable completion/reconciliation; no cross-service exactly-once transaction is claimed. Each erasure participant must close admission, advance its durable fence, cancel or drain in-flight effects, purge and verify before acknowledging completion. An effect started before erasure cannot resume and rematerialize data afterward. A destination that cannot enforce that protocol is not an approved materialization participant. Delete the received SQS message only after durable acceptance, using its receipt handle. Define inbox retention against the full supported replay horizon, not a short cache TTL. If that horizon cannot be bounded, retain durable deduplication or prove the effect inherently idempotent with version/tombstone checks.

Store a content digest beside each inbox identity using a qualified, versioned canonical serialization of the inner CloudEvent, including tenant/type/schema/data and excluding native broker delivery metadata. An identical identity and identical content is a duplicate; the same identity with different content is an integrity failure. Quarantine and alert without overwriting prior evidence or treating the changed event as already processed. The digest is a comparison against protected evidence, not a substitute for producer authentication. Apply the same rule to the audit sink and redrive tooling.

Destructive work also conditions on the authoritative retention/legal-hold decision revision, workflow state, registered participant and current work fence. Hold/review transitions and deletion admission must use the same serialized authority; checking for a hold once at workflow start is insufficient. A newly accepted hold prevents subsequent destructive units and invalidates outstanding work leases. Define the linearization point and bounded in-flight unit: an irreversible deletion already committed before the hold cannot be undone or reported as having been preserved. Completion callbacks are authenticated and bound to tenant, patient, request, deletion/hold generations, participant, work attempt and recovery epoch; stale callbacks cannot advance completion. Same-region DynamoDB conditional transactions are one available mechanism, with destination fencing/drain required for external effects. [DynamoDB transactions and conditional actions](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/transaction-apis.html).

Maintain monotonic per-patient revision and access/deletion generations. A restriction cannot be undone by a late ordinary update; discard stale generations and resolve current authoritative restriction state before an effect that could disclose or materialize data. Restriction notifications are not the sole revocation freshness mechanism: dropped/late notifications cannot extend an authorization grant. Terminal erasure tombstones dominate earlier updates and replays, survive recovery, and prohibit reuse of that patient ID. A new consumer or rebuilt projection must reconcile current restriction/erasure state before serving; replaying history is not sufficient authority to reconstruct a patient.

A restore starts a fenced recovery epoch. Reconcile restored data, inbox/deduplication records, participant receipts and the protected current erasure ledger before readiness. A historical “processed” inbox entry or completion callback is not proof that a restored copy is erased now; re-establish the required local effects and generation fences, and reject callbacks from the former recovery epoch. Retain minimized tombstones and provenance only for approved purposes/horizons, with their own deletion inventory; “reference-only” does not mean anonymous or exempt from erasure accounting.

Reference-only messages still require residency, access, retention and erasure accounting. Queue/DLQ/archive retention and replay windows are approved regional policy; they are not yet numerically selected here. Set DLQ recovery headroom with original creation/enqueue age tracked, because Standard-queue expiry is not reset merely by moving to a DLQ. Alert before expiry and preserve outstanding required work durably. SQS cannot bulk-delete an individual's messages by patient ID. Register these protected residual references and retention obligations in erasure accounting; never claim all copies are gone merely because active patient records were deleted. Replays and restores reapply tombstones before traffic or downstream effects. [SQS DLQ retention](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html).

## Abuse controls and security acceptance cases

Apply contract-derived message-byte, nesting, string, array, batch and processing-deadline bounds before expensive validation or database access; reject duplicate JSON member names and unsupported encodings. Resolve schemas only from the pinned allowlist, with no network fetch based on `dataschema`. Bound retries and work amplification per event, tenant, producer and cell; reserve capacity for audit preservation and erasure so ordinary updates or replay cannot starve them. Rate admission before generating expensive work; mandatory accepted audit records cannot be sampled or silently discarded. Numerical budgets require capacity evidence and policy approval, not invented limits that reject valid worst-case messages. [OWASP resource-consumption guidance](https://owasp.org/API-Security/editions/2023/en/0xa4-unrestricted-resource-consumption/).

Poison events go to the regional restricted quarantine/DLQ with bounded retry and an alert. Do not log rejected bodies, patient/tenant references, arbitrary schema strings or raw parser errors. Ordinary metrics use bounded type/error/queue classes. Redrive requires a narrowly authorized workload/operator, an explicit source/target inventory, the same validation/fences as first delivery and a bounded rate; it cannot route across residency boundaries or bypass tombstones. Stop before expiry threatens accepted obligations and preserve them through the existing protected durable path. Default worker credentials cannot purge a queue, bulk replay an archive or change retention to hide failures.

| Security case | Required result |
|---|---|
| A valid queue producer swaps tenant/patient/request references or claims another cell's source | Evidence/ownership mismatch rejects the event before side effects; no cross-tenant data is fetched or disclosed |
| Same `source`/`id` is redelivered with changed tenant, type, schema or payload | Integrity failure is quarantined; prior inbox/audit evidence remains intact |
| Hold is accepted between erasure-start delivery and the next delete; an older callback arrives later | New destructive units stop and the stale callback cannot complete the workflow; already committed work is reported accurately |
| Permissions or erasure state changes while audit acknowledgment is delayed | No stale protected response after its freshness/deadline fence; a changed projection obtains its own intent |
| Restored data is paired with an old “processed” inbox or participant receipt | Recovery stays fenced until current erasure effects are re-established |
| One tenant floods updates, malformed messages or redrive while audit/erasure work is pending | Bounds and fair scheduling protect other tenants and critical work; no payload logging or lost accepted obligations |

These cases supplement the general matrix below and are proposed requirements, not evidence that an implementation has passed them.

## Event version lifecycle and validation

Event payload SemVer, AsyncAPI document revision, AsyncAPI format version, CloudEvents wire version, REST API SemVer and container digest are separate dimensions. `dataschema` identifies an immutable reviewed schema; its URN resolves through a protected contract registry/artifact mapping, not arbitrary network fetching. Current `.v1`/`1.0.0` examples are proposed, unreleased identities.

Because this profile has strict schemas, even an additive field can break an existing strict consumer. Classify compatibility using actual supported consumers, not an automatic “new optional field is safe” rule. An incompatible payload or semantic change needs a new event major/type, supported side-by-side publication/delivery and a recorded migration. Dual-published representations of one business occurrence have distinct immutable event identities plus a documented consumer migration/deduplication strategy; never reuse `source`/`id` for different payloads. Retain old event schemas/consumers for their authorized replay and recovery horizons. An event compatibility registry records producer/consumer releases, schema digests, queues, retention/replay obligations and migration completion. REST count-based API pruning must not implicitly garbage-collect event consumers, schemas, outbox entries, archives or tombstones.

Before implementation acceptance, qualify the exact AsyncAPI validator/renderer/Go model path; validate positive and negative synthetic examples and round-trips; and run the [API testing](api-testing.md) matrix across real transaction, SQS and consumer boundaries. Cover duplicate/ambiguous sends, per-entry fan-out failures, outbox recovery beyond stream retention, crashes around inbox/effects/ack, stale revisions, restriction-before-update, erasure-before-replay, consumer bootstrap, DLQ expiry, wrong-region/tenant/source messages, unknown schemas/fields, malformed envelopes and injected synthetic PII. Explicitly pause a worker after its active-state read, complete participant erasure, then resume it: the stale effect must be rejected, never recreate data. Verify duplicate erasure-start delivery, stale/held workflow rejection, durable participant callbacks and completion only after every required obligation. Test optional EventBridge wrapping only if adopted. Preserve audit-ack latency and browser p95/load evidence; domain publication is asynchronous and must not add a broker round trip to the normal lookup path.

Open adoption evidence: exact tool pins and 3.1.0 support, per-event consumers/owners, regional retention/replay limits, fan-out need, throughput and capacity, and operational recovery proof. No parser, generator, broker or consumer has been qualified by this document. JSON parsing and locally checked references/examples are limited document-consistency evidence, not runtime conformance.
