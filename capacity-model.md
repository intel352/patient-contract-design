---
id: CAPACITY-001
version: 0.1.0
status: proposed
approval_status: pending
authors: [Codex]
approvers: []
created: 2026-10-22
updated: 2026-10-22
owner_role: Service architecture owner
supersedes: []
superseded_by: []
references: [architecture.md, docs/project-standards.md]
---

# Patient contact capacity model

Status: planning scenarios, not a market forecast or benchmark.
Date: 2026-10-22.

## Storage decision

Recommend DynamoDB as the primary patient-contact store for the defined tenant-scoped exact-phone and patient-ID access patterns. This choice puts horizontal data distribution under the managed service rather than making PostgreSQL cluster growth and tenant relocation a primary operating responsibility. It does not establish this application's browser-visible latency, costs, or hot-key behavior without validation.

Population-based upper bounds are useful stress cases. Launch capacity should follow measured customer datasets and request patterns. Grow regional capacity as needed instead of provisioning for an entire hypothetical global market on day one.

## Count records, people, and organizations separately

The reliable count is:

`patient records = sum(retained distinct patient records in each subscribed organization)`

A person's record in two customer organizations counts twice. Repeated visits to the same organization generally update the same record. The meaning of organization must match the tenancy contract: one hospital group, one practice, and one individual practitioner cannot be interchanged when estimating market share.

Before customer data is available, use:

`R ≈ P × H × S`

| Input | Meaning |
|---|---|
| P | Population planning base; not automatically a current demographic estimate |
| H | Average retained distinct patient-organization relationships per person over the relevant retention horizon |
| S | Adoption weighted by those patient-organization relationships |
| R | Stored patient-organization records |

Organization-count share is not S unless adopters have approximately representative patient populations. If q is the fraction of organizations adopting and adopters average a times the patient count of all organizations, the implied relationship-weighted share is `S = q × a` (a consistent model must keep S at or below 1). At q=30%, an adopter-size factor of 0.5 yields S=15%; a factor of 2 yields S=60%. These are sensitivity examples, not predictions about hospitals or practices.

Historical, inactive, and deceased patients may remain retained. Consequently, current living population is not a strict upper bound on the retained person base. Calibrate H or use explicit cohorts; do not treat annual visits as distinct organization relationships. Unique people reached is a separate measure and must not be added to record count a second time.

## Illustrative scenarios

Population values below deliberately reuse round planning inputs from the discussion. They are neither current population claims nor forecasts of achievable adoption. The 30% scenario is a high-adoption design case.

| Scenario | P | H | S, record-weighted | R |
|---|---:|---:|---:|---:|
| US, smaller adoption | 400 million | 5 | 5% | 100 million |
| US, middle adoption | 400 million | 5 | 10% | 200 million |
| US, high adoption | 400 million | 5 | 30% | 600 million |
| US, more retained relationships | 400 million | 10 | 30% | 1.2 billion |
| Large new market, small adoption | 1.4 billion | 5 | 1% | 70 million |
| Large new market, middle adoption | 1.4 billion | 5 | 5% | 350 million |
| Large new market, higher adoption | 1.4 billion | 5 | 10% | 700 million |

The US high-adoption case plus one large new market at 5% gives 950 million patient-organization records across their separate regional deployments. This is an example of cumulative volume; it does not require a single worldwide patient table.

The initial requirements imply 10 million records / 20,000 customers = 500 records/customer on average. Confirm whether that is a full retained patient population, an initial import, or an active subset. For sizing, collect median, p95, p99, and maximum tenant records and traffic; averages hide the largest organizations.

## Convert records into physical items and bytes

A patient record and its phone mappings are separate DynamoDB items in the proposed model:

`base items ≈ R × (1 + mean phone mappings per patient) + auxiliary items`

For example, 600 million patients with two mappings each means approximately 1.8 billion base items before outbox, erasure, inventory, and other operational items. Secondary-index storage, backups, and any approved regional replicas require separate accounting.

Estimate bytes using measured serialized item sizes, including attribute names and key sizes, and the chosen AWS billing/storage rules. A 4 KB comment allowance is not a 4 KB average patient item. Capture size percentiles and how often reads cross the next capacity-unit boundary. Field projections reduce returned data but do not generally turn a large underlying item into a smaller billed read.

## Model throughput independently

Storage grows with retained records. Throughput grows with users, workflow frequency, automation, concurrent imports, and bursts. A billion mostly inactive records can generate less request traffic than a much smaller active tenant.

Use endpoint mix and measured request distributions:

`read units/second ≈ Q_lookup × E[lookup read units] + Q_detail × E[detail read units] + other reads`

A small strongly consistent phone-mapping query plus one patient item up to 4 KB is approximately two read units. If all 20,000 API requests/second were such lookups, that would be approximately 40,000 read units/second before additional matches, retries, and headroom. Real mapping queries may consume more units; patient items may exceed 4 KB; partial batch results require bounded retries. Detail-only traffic has a different cost. Strong reads and transactional writes are deliberate correctness choices, not free scaling assumptions. [DynamoDB capacity model](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Constraints.html).

Launch with known table/index quotas, explicit request-rate headroom, and sufficient warm throughput. On-demand mode does not make an unprepared table immune to sudden traffic steps or concentrated keys. Proactive pre-warming is available and has a cost. [On-demand behavior](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/on-demand-capacity-mode.html), [warm throughput](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/warm-throughput.html).

Validate the original 20k aggregate API RPS requirement independently from the larger storage scenarios. Test the full Go authentication/authorization → DynamoDB → durable audit → browser-render path, with realistic largest-tenant and shared-phone traffic. Keep the agreed browser-visible p95 below 100 ms as an acceptance gate, not an implication of database selection.

## Regional expansion

Operate regional datasets and deploy the same versioned REST/entity contract. Region-specific configuration owns identity issuers, encryption keys, endpoints, capacity, telemetry destinations, and allowed recovery destinations. Market adoption in one country does not force patient replication into another country.

China needs a specific deployment review: its AWS regions are in the separate `aws-cn` partition, with independent IAM and account credentials. AWS currently documents Cognito User Pools as unavailable in its China offering. Identity-provider pluggability is now a confirmed requirement: Cognito is the first adapter, with configured regional alternatives supported through the common authentication contract. Provider selection, claim validation and revocation behavior must be qualified for each destination; alternate providers are not implemented by this design draft. [AWS partitions](https://docs.aws.amazon.com/whitepapers/latest/aws-fault-isolation-boundaries/partitions.html), [Cognito China differences](https://docs.amazonaws.cn/en_us/aws/latest/userguide/cognito.html).

## Next evidence to collect

- Tenant definition and distribution of retained patient counts, including historic data.
- Mean and upper-percentile phone mappings, patient item sizes, and matches per lookup.
- Endpoint read/write mix, active hours, sustained versus peak RPS, and largest-tenant/key share.
- Market-share scenarios weighted by record volume, with a time horizon and retention assumptions.
- Regional cost estimates for strong reads, transactional writes, indexes, backups, audit retention, and pre-warming.

The fixed API contract does not change with these capacity scenarios. No new schema-extension mechanism or patient cache is required by this model.
