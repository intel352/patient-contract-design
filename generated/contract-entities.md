---
id: GENERATED-ERD-001
version: 0.1.0
status: proposed
approval_status: pending
authors: [Codex]
approvers: []
created: 2026-10-22
updated: 2026-10-22
references: [../openapi.json, ../docs/diagrams.md, README.md]
---

# Contract-derived entity relationships

Generated mechanically from the named `components.schemas` in [canonical OpenAPI](../openapi.json), source SHA256 `976a7553b6a75f7200e4d093b365bf4f9edc1c76f9579962cc50654757462bd6`. This describes API representations and containment, not DynamoDB tables, global identity or enforced foreign keys. Presence/array cardinality comes from schema structure; conditional field authorization still comes from backend policy. Primitive helper types are resolved, and nine selected domain schemas are included. This is a design export, not introspection of a running API.

```mermaid
%% Generated from ../openapi.json; design-time schema export, not database DDL.
%% Source SHA256: 5c11e599971ce556f1e9e29e436e6b7582602f079537042e573d138e2567155e
erDiagram
    PatientRead {
        string patientId "required"
        string version "required"
        object access "required"
        object name "optional, nullable"
        array phones "optional"
        object address "optional, nullable"
        string comment "optional, nullable"
    }
    NameRead {
        string display "optional, nullable"
        string given "optional, nullable"
        string family "optional, nullable"
    }
    PhoneRead {
        string e164 "optional, nullable"
        string extension "optional, nullable"
    }
    AddressRead {
        string countryCode "optional, nullable"
        array lines "optional, nullable"
        string locality "optional, nullable"
        string administrativeArea "optional, nullable"
        string postalCode "optional, nullable"
    }
    FieldAccess {
        array readableFields "required"
        array writableFields "required"
        string policyVersion "required"
    }
    LookupResponse {
        array items "required"
        string nextCursor "required, nullable"
    }
    Session {
        string subjectId "required"
        array memberships "required"
        string nextCursor "required, nullable"
    }
    TenantMembership {
        string tenantId "required"
        string residencyRegion "required"
        string apiBaseUrl "required"
        array capabilities "required"
    }
    Erasure {
        string requestId "required"
        string status "required"
        string retentionReasonCode "optional"
        string failureCode "optional"
        string backupExpiryAt "optional"
        string updatedAt "required"
    }
    PatientRead ||--|| FieldAccess : access
    PatientRead ||--o| NameRead : name
    PatientRead ||--o{ PhoneRead : phones
    PatientRead ||--o| AddressRead : address
    LookupResponse ||--o{ PatientRead : items
    Session ||--o{ TenantMembership : memberships
```

The [logical storage and persona ERD](../docs/diagrams.md) covers internal entities not exposed by this patient API. Regeneration must preserve the declared selection and reference rules, and future CI must render/validate the Mermaid source as well as compare it for drift.
