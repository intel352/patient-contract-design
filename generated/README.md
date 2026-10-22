---
id: GENERATED-001
version: 0.2.0
status: proposed
approval_status: pending
authors: [Codex]
approvers: []
created: 2026-10-22
updated: 2026-10-22
references: [../contracts/asyncapi.json, ../docs/events.md, ../openapi.json, ../docs/diagrams.md, ../docs/document-governance.md]
---

# Generated design artifacts

These files were mechanically exported from the canonical design specification. They are not generated from Go implementation, a database or a running service; none exists in this task. No released API or deployed endpoint is implied.

| Artifact | Source and derivation |
|---|---|
| [openapi.json](openapi.json) | JSON export of `../openapi.json`; logically identical document. |
| [openapi.yaml](openapi.yaml) | YAML serialization of the same parsed JSON; round-trip equality checked. |
| [asyncapi.json](asyncapi.json) | JSON export of `../contracts/asyncapi.json`; logically identical document. |
| [asyncapi.yaml](asyncapi.yaml) | YAML serialization of the same event contract; round-trip equality checked. |
| [contract-entities.mmd](contract-entities.mmd) | Mermaid ERD derived from nine named OpenAPI object schemas, properties, required/nullability and references. |
| [contract-entities.md](contract-entities.md) | Reviewable Markdown wrapper around the generated ERD. |
| [manifest.json](manifest.json) | Source/output checksums, selected schemas, API/spec versions and generation scope. |

The event contract uses AsyncAPI **3.1.0**, with independent draft `info.version: 0.1.0` and CloudEvents wire `specversion: "1.0"`. Event payload versions are separate. These exports do not prove a CLI, parser, model generator or broker accepts the contract; exact tooling qualification remains open.

The REST canonical source is OpenAPI **3.2.1**; its independent API `info.version` remains **0.1.0**, an unreleased proposal. Future production `/v1` starts at an explicitly approved GA release, not by exporting these files.

Regeneration procedure: parse canonical JSON; serialize equivalent JSON/YAML without changing schema semantics; derive ERD object properties and referenced-object containment for the selected schema set; update source/output SHA-256 provenance; verify equality, references, security inheritance and examples. Production CI must implement this procedure with pinned, reviewed generators and full format/conformance tests. This documentation task uses temporary export commands and does not introduce generation application code or a CI workflow.

Checks currently performed do not establish full OpenAPI 3.2.1 tool compatibility, generated Go/TypeScript compilation, rendered diagram correctness, runtime authorization or performance. Those qualification gates are explicit in [project follow-ups](../docs/FOLLOWUPS.md).

When these artifacts are hosted, access requires approved caller authentication and authorization. Their presence in a local documentation folder does not create an anonymous schema endpoint.
