---
id: INDEX-001
version: 0.2.0
status: proposed
approval_status: pending
authors: [Codex]
approvers: []
created: 2026-10-22
updated: 2026-10-22
references: [docs/project-standards.md, docs/document-governance.md]
---

# Patient contact service — standalone design

This is a theoretical project specification, not an implemented service. No application code, CI workflows, Kubernetes manifests or AWS runtime resources are supplied. The design is independent of the author's existing organizations, repositories and frameworks.

Repository: [intel352/patient-contract-design](https://github.com/intel352/patient-contract-design), private. The user authorized direct commits and pushes for this documentation repository; the proposed application PR/release controls remain future standards.

The project owns the Go service, REST/entity contract, generated consumer-contract artifacts, security, data lifecycle, delivery requirements and API operational behavior. Frontend implementation belongs to its consuming team. Browser-visible latency is a joint acceptance requirement measured with that consumer.

| Start here | Purpose |
|---|---|
| [Project standards](docs/project-standards.md) | Confirmed expectations, technology baselines, scope and proposed maintenance rules |
| [Agent instructions](AGENTS.md) | Required reading and working rules for developers and AI assistants |
| [Architecture](architecture.md) | Regional Kubernetes/Go/DynamoDB design, identity, privacy, erasure and latency |
| [API versioning](docs/api-versioning.md) | Schema/behavior SemVer, exact selectors, configurable support counts, overrides and automatic decommissioning |
| [Delivery and environments](docs/delivery-and-environments.md) | Git, human/AI review, preview, production and isolated engineering deployments |
| [Hardened-image maintenance](docs/image-maintenance.md) | Explicit pins, release detection, approved-source rebuilds and lifecycle-aware rollout |
| [Event contracts](docs/events.md) | OpenAPI REST, AsyncAPI channels/messages and CloudEvents envelopes |
| [Backend testing](docs/api-testing.md) | Unit, integration, behavior and complete authenticated API client journeys |
| [Document governance](docs/document-governance.md) | Metadata, approval evidence, immutable decisions and supersession |
| [Identity and personas](docs/identity-and-personas.md) | Token contracts, service authorization and customizable default personas |
| [System diagrams and ERDs](docs/diagrams.md) | AWS architecture, data relationships, request flow and sequence charts |
| [Generated contract artifacts](generated/README.md) | JSON/YAML exports, schema-derived ERD and source checksums |
| [AsyncAPI contract](contracts/asyncapi.json) | Proposed asynchronous channels, operations and CloudEvents messages |
| [Event examples](contracts/event-examples.json) | Synthetic event-contract examples |
| [OpenAPI contract](openapi.json) | Current proposed transport/entity definition; not a published API |
| [Contract examples](examples.json) | Synthetic examples; no real patient information |
| [Capacity model](capacity-model.md) | Record-growth scenarios and throughput amplification |
| [Design review](docs/design-review.md) | Findings, corrections and the limits of current verification |
| [Open follow-ups](docs/FOLLOWUPS.md) | Proposed values, qualification gates and unresolved ownership |

The document status is deliberately explicit: user-confirmed requirements are recorded, while suggested numerical policies and implementation choices remain proposed until human review. `openapi: 3.2.1` is the specification format; `info.version: 0.1.0` is this unreleased API draft, not a production release or a claim that the service exists.

Decision records live in [decisions/](decisions/). Reusable document templates live in [docs/templates/](docs/templates/).
