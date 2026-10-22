---
id: DOCS-001
version: 0.1.0
status: proposed
approval_status: pending
authors: [Codex]
approvers: []
created: 2026-10-22
updated: 2026-10-22
supersedes: []
superseded_by: []
references: [project-standards.md, api-versioning.md, delivery-and-environments.md]
---

# Repository documents and decision governance

## Locations and authority

| Location | Contents |
|---|---|
| `AGENTS.md` | Short entry point for humans and AI assistants; links normative sources without duplicating policy. |
| `docs/project-standards.md` | Project scope, confirmed constraints, baselines and assumptions. |
| `docs/` | Versioned technical standards and specifications; future feature specs may use `docs/specs/`. |
| `decisions/NNNN-short-title.md` | Architecture Decision Records (ADRs), including alternatives, rationale and consequences. |
| `openapi.json` | Current working API contract. Future immutable released contracts and release index live under `api/releases/`. |
| `docs/FOLLOWUPS.md` | Project-only unresolved decisions and evidence gaps, with owner role and completion criterion. |
| `docs/templates/` | Specification and ADR starting points. Templates are not accepted decisions. |

This is the layout to establish in the future Git repository. Currently the design folder contains documents only; it has not been initialized or published as a new repository. Do not create empty release artifacts that imply an API has shipped.

User-confirmed requirements constrain proposed design. Accepted standards/ADRs constrain implementation. A draft cannot silently override an accepted decision. OpenAPI is canonical for transport definitions; discrepancies between prose, schemas and accepted policy block release and must be resolved in the same reviewed change.

## Required header

Specifications and ADRs use YAML front matter with these fields:

| Field | Rule |
|---|---|
| `id` | Immutable identifier, unique in the repository. Links survive title and file moves. |
| `version` | Document revision SemVer; independent of API SemVer and OpenAPI format. |
| `status` | `draft`, `proposed`, `accepted`, `superseded` or `retired`. |
| `approval_status` | `pending`, `approved`, `rejected` or `requirement_confirmed`. Last value confirms an input only, not the full design. |
| `authors` | Named human contributors or named AI assistant; do not invent human authorship. |
| `approvers` | Named accountable humans with review evidence when approved; empty while pending. AI is never a human approver. |
| `created`, `updated` | ISO dates. Approved docs also record `approved_on` and `approval_evidence`. |
| `owner_role` | Role accountable for maintenance; resolve to a named owner before operational adoption. |
| `references` | Relative repository paths or exact external sources, including requirement/spec/ADR relationships. |
| `supersedes`, `superseded_by` | Reciprocal document references; empty arrays when unused. |

The templates demonstrate the full header. Root index and small queue documents may omit fields irrelevant to their role. Machine-readable sources use equivalent `x-document` metadata where their format permits; auxiliary examples can reference the owning specification rather than embed invalid metadata. `AGENTS.md` is an operational entry point, not a substitute approval record.

## Approval lifecycle

1. Author a proposal with explicit requirements, assumptions, alternatives, security/privacy impact, compatibility effects, acceptance evidence and unresolved questions.
2. Obtain AI review and the accountable human review required by [delivery](delivery-and-environments.md). Domain owners review API, security or operations decisions they own. Record findings and dispositions in the PR.
3. Accept only after an authorized human explicitly approves the exact revision. Record approver, date and immutable PR/review or decision evidence; linking a conversation without explicit approval is insufficient.
4. Merge through the protected branch gates. A document's `accepted` label alone is not approval evidence. CI should check status/evidence consistency; review-system authorization is the source of truth.
5. Amend through another reviewed PR. A material normative change increments the document major version and requires renewed approval; an additive clarification increments minor; an editorial correction increments patch. These rules describe document revisions, not API compatibility classification.

An AI may draft, critique and recommend acceptance. It must leave approval pending when the human has only requested documentation or agreed to a requirement. The current pack contains confirmed requirements and unapproved proposed policies; no implementation authorization or staffed operational commitment follows from drafting it.

## Decisions and supersession

ADRs record why a consequential choice was made, the alternatives rejected, affected requirements and operational consequences. Assign the next unused four-digit ID in merge order; IDs never get reused. References use both IDs and stable paths.

Preserve an accepted ADR's substantive reasoning. A changed decision gets a new ADR with `supersedes` pointing to the old one. Once the new ADR is approved, update only the old ADR's lifecycle metadata and a short supersession notice; set `status: superseded` and `superseded_by` to the new record. Keep both accessible. A proposed successor does not invalidate an accepted predecessor.

Do not overwrite an older decision to make it appear that the team always held the new position. Normal editorial corrections retain their history. Specifications may evolve in place with revision history and Git diffs; replacing a whole specification uses the same explicit supersession links.

## Future automated checks

- Required metadata, valid dates/statuses and unique IDs; no claimed approval without evidence.
- Resolvable references and reciprocal supersession links; no cycles or reuse of identifiers.
- OpenAPI format and examples, published API version history, compatible generated consumer contracts and declared PR change severity.
- Mermaid syntax and rendering, safe examples and secrets/PII checks; no real patient fixtures.
- Changed normative sources trigger their human ownership reviews. PR approval refers to the exact reviewed content, not a moving branch name.

These checks are specified, not implemented in this documentation task. Their failures must block a future release rather than be described as passed based on a Markdown/JSON syntax check alone.
