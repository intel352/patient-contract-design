---
id: ADR-0002
version: 0.1.0
status: proposed
approval_status: requirement_confirmed
authors: [Codex]
approvers: []
created: 2026-10-22
updated: 2026-10-22
owner_role: API owner
supersedes: []
superseded_by: []
references: [../docs/api-versioning.md, ../docs/image-maintenance.md]
---

# 0002. Declare the supported API set explicitly

## Context

The user clarified on 2026-10-22 that configurable major/minor/patch counts and explicit retention overrides determine supported versions. This replaces the earlier proposed rule that activity or a seven-day idle window controls deployment retention. Long-lived consumers still need visible commitments and migration planning.

## Decision

Compute an explicit supported set from configured release counts, exact retained-version overrides and any declared notice/age holds. Automatically retire excluded versions and decommission infrastructure no supported view or required worker still needs. Usage informs retention decisions but does not silently change the set. Requests explicitly select an exact contract version and responses identify the served version.

Counts, dates and override procedures remain proposed until reviewed. The policy direction is user-confirmed; this is not a record of operational activation. A hard count without override capability would mishandle commitments; indefinite automatic retention from traffic would contradict the clarified policy.

## Consequences

- Continued requests do not automatically prevent retirement; owners must record needed overrides before cutover.
- API contracts and implementation images have independent identities. Hardened images can preserve multiple exact contract views.
- Retirement, image promotion and rollback share generation checks so stale jobs cannot recreate excluded routes.
- Exact patch selection increases the compatibility-test matrix and requires consumer upgrade discipline.
