---
id: ADR-0004
version: 0.1.0
status: proposed
approval_status: pending
authors: [Codex]
approvers: []
created: 2026-10-22
updated: 2026-10-22
owner_role: API and integration owners
supersedes: []
superseded_by: []
references: [../docs/events.md, ../openapi.json, ../contracts/asyncapi.json, ../docs/api-testing.md]
---

# 0004. Separate REST and event contracts

## Context

The user requested CloudEvents for generic envelopes or at least downstream events, then asked to consider AsyncAPI where tooling makes it a useful complement to OpenAPI. Request/response operations and asynchronous occurrence notifications need different contracts, while both require consistent versioning, security and test evidence.

## Proposed decision

Use OpenAPI for REST resources and errors, AsyncAPI for event channels/operations/messages, and CloudEvents for the wire event envelope. Keep ordinary REST bodies typed with common version/request metadata in headers. CloudEvents can be transported in HTTP requests or responses, but does not require treating every resource read as an event. A universal wrapper would add event identity/source semantics to calls that do not need them. [CloudEvents HTTP binding](https://github.com/cloudevents/spec/blob/v1.0.2/cloudevents/bindings/http-protocol-binding.md).

Tool compatibility is a qualification gate, not inferred from a specification's latest version. This recommendation adds contracts to the existing API/worker architecture; it does not require a microservice split or a new broker.

## Consequences

- Event envelope version, event payload version, REST version and image digest remain distinct.
- Producer and consumer behavior, transport encoding, duplicate processing and erasure replay need tests beyond schema checks.
- Event references remain restricted information; downstream consumers use their own current permissions.
- AsyncAPI artifacts and examples are design inputs, not generated application code or proof that a chosen generator supports the entire contract.
