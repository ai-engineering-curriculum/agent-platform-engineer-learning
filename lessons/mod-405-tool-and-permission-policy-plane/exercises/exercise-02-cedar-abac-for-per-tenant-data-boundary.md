# exercise-02: Cedar ABAC for a Per-Tenant Data Boundary

**Estimated effort:** 3 hours · **Output:** one Cedar schema, three to five Cedar policies (plus one template), a Cedar test file, a short memo (~1 page) · **Pairs with:** chapter [03](../03-cedar-and-abac-for-per-tenant-data-boundary.md); feeds exercise 04 and projects 402 and 403.

## Objective

Author the per-tenant ABAC layer for a data-touching operation — the policies that answer *"does this session, with its declared purpose and region, have the right to touch this specific record under this specific tenant's rules?"* The deliverable is Cedar code: a schema that the tenant data-access layer validates against, the policy set that enforces tenant ownership, region residency, classification ceiling, and purpose limitation, and a template that lets a specific tenant override one dimension within pre-agreed bounds.

A well-authored ABAC layer makes cross-tenant leaks structurally impossible under the schema — not merely unlikely under good behaviour. If your policy set admits any input where `principal.parent != resource.owner_tenant` and allows, it has failed the exercise. Cedar's schema-validation and analyser are the tools that keep this honest.

## Prerequisites

- You have read chapters [01](../01-the-policy-plane-and-the-authorization-triad.md) and [03](../03-cedar-and-abac-for-per-tenant-data-boundary.md).
- You have [Cedar](https://www.cedarpolicy.com/) installed, either the [Rust CLI](https://github.com/cedar-policy/cedar) (`cedar --version`) or one of the language bindings. The exercise is language-agnostic; a CLI is simplest.
- You have the [Cedar language reference](https://docs.cedarpolicy.com/policies/syntax-policy.html), the [schema reference](https://docs.cedarpolicy.com/schema/schema.html), and the [template docs](https://docs.cedarpolicy.com/policies/templates.html) open.
- Optional but recommended: Amazon Verified Permissions documentation, if your production target is a managed Cedar runtime.

## Problem statement

You are writing ABAC for a specific data operation. Pick one — the acceptance criteria are the same.

- **Support-agent reads a customer record.** `ReadRecord` action on `Record` entity. The agent is a `Session` with declared `purpose`, `region`, `max_classification`. The record carries `owner_tenant`, `classification`, `region`, `consented_purposes`.
- **Data-export tool exports a record set.** `ExportRecords` action on a set of `Record` entities. The destination carries `allowed_regions`; the session carries `declared_region` and `purpose`.
- **Analytics agent reads an aggregated-metrics view.** `ReadAggregate` action on an `Aggregate` entity with `owner_tenant`, `min_row_count`, `classification` (the classification of the underlying data), and `aggregation_tier` (raw, k-anonymised, differential-privacy).

Three tenants must coexist in the test fixtures:

- **`tenant_eu_strict`** — a European financial-services tenant. Default region `eu-central-1`; classification ceiling for all sessions `pii.medium`; no cross-region access.
- **`tenant_us_broad`** — a US commercial tenant. Default region `us-east-1`; classification ceiling `pii.critical` for `platform_admin` sessions; cross-region allowed between US regions.
- **`tenant_apac_custom`** — an APAC tenant with a region override: default region `ap-northeast-1` but reads from `ap-southeast-2` are allowed under a template-instantiated exception. Tests must prove the exception works and that the exception's shape is auditable.

## Requirements

### Part 1 — The schema (main deliverable #1)

Author `schema.cedarschema`:

- Declare entity types `Tenant`, `User`, `Session`, `Record` (or `Aggregate` / an export-destination type, depending on the operation you picked).
- Model hierarchy: `User in Tenant`, `Session in Tenant`, `Record in Tenant`.
- Attribute types carry semantic meaning — `region: String` (with an implicit vocabulary documented in the schema comment), `classification: String`, `classification_rank: Long` (the rank is Long to let policies compare ordinally; chapter 03 argues for this), `consented_purposes: Set<String>`.
- Actions declare `appliesTo` with principal and resource types. Context carries `declared_region`, `declared_purpose`, and anything your chosen action requires.

Run `cedar validate --schema schema.cedarschema --policies policies/` and ensure it passes.

### Part 2 — The policy set (main deliverable #2)

Author `policies/*.cedar`. At minimum:

- **`t1_tenant_ownership.cedar`** — a `permit` policy that allows the action only when `resource.owner_tenant == principal.parent`.
- **`t2_region_residency.cedar`** — a `forbid` policy that blocks cross-region. Note: `forbid` overrides `permit`.
- **`t3_classification_ceiling.cedar`** — a `forbid` policy that blocks reads above the session's `classification_rank`.
- **`t4_purpose_limitation.cedar`** — a `forbid` policy that blocks reads whose declared purpose is not in the resource's `consented_purposes` set.
- **`t5_legal_hold.cedar`** — a `forbid` policy on `ExportRecords` (if applicable) that blocks any record with `legal_hold: true`.

Each policy carries a `@id("...")` annotation naming it. Each is testable independently; a test asserts the `@id` is the policy that forbids on the matching input.

### Part 3 — The template

Author `templates/region_exception.cedar`:

```cedar
@id("tenant_region_override")
permit (
    principal == ?principal,
    action in [Action::"ReadRecord", Action::"ExportRecords"],
    resource
)
when {
    resource.region == ?resource_region
};
```

Author `templates/region_exception.links.json` (or the equivalent your Cedar runtime uses) that instantiates the template for `tenant_apac_custom` with `resource_region = ap-southeast-2`.

Note: the exception does not override the `forbid` in T-2 — Cedar's semantics mean a `forbid` wins. Explain in your memo (Part 5) how you handle this: typically by giving the template a different action (`ReadRecordCrossRegion`) or by narrowing T-2's condition so it does not apply when the override is active. Both are legitimate; the memo argues your choice.

### Part 4 — The tests

Author `tests/*.cedartest` (or a test harness in the binding you chose):

- **At least 10 permit scenarios** — same-region reads, in-ceiling classifications, in-purpose reads, template-exception reads.
- **At least 10 forbid scenarios** — cross-region (by default), classification-ceiling violation, purpose mismatch, legal-hold export, cross-tenant principal.parent mismatch.
- **Reason assertions** — each forbid scenario names the `@id` of the policy that should fire.

Run the test suite; all green.

### Part 5 — The memo

One page covering:

- **The operation you chose and why.** One paragraph on why the operation's data-touching shape is representative.
- **The schema trade-offs.** Where you chose `Long` over `String` (and why), where you nested vs. flattened attributes, any place the schema cannot express a rule you wanted (and the workaround).
- **The template-vs-overriding-forbid choice.** How you reconciled T-2 with the template exception; why your choice is defensible.
- **Latency considerations.** A short estimate of how many entities the data-access layer will stage per decision and whether that scales. Note that Cedar's entity graph has a cost.
- **What you would route to `ai-risk-engineer`.** One open question — for example, is `pii.medium` the right classification ceiling default for `tenant_eu_strict`, or is this a harm-model decision?

## Starter guidance

- Resist the urge to write a single "god policy" with many `when` branches. Each dimension (tenant, region, classification, purpose) is its own policy; this is how Cedar expresses correctness clearly and how the analyser gives you the most leverage.
- `principal.parent` is the hierarchy accessor — for a `Session in Tenant`, it returns the parent `Tenant` entity. Use it; avoid duplicating `tenant_id` as an attribute on both `Session` and `Record` unless you have a specific reason.
- Pre-compute `classification_rank` in the data-access layer before calling Cedar. Chapter 03 explains why; the policy becomes a clean `Long` comparison.
- When asserting on test forbids, assert the specific `@id` that fired. A test that only asserts `Deny` masks a regression where the wrong `forbid` wins.
- Cedar's error messages on schema validation are informative; paste them into your memo if you had to iterate. Reviewers care about the schema evolution.
- `<!-- needs-research: ... -->` anything you cannot verify against the current Cedar release — notably the exact syntax for templates, extension functions, and the analyser's CLI flags, all of which move.

## Acceptance criteria

Mark done when:

- [ ] `cedar validate --schema schema.cedarschema --policies policies/` passes.
- [ ] The test suite has at least 20 scenarios, split evenly between permits and forbids; all green.
- [ ] Every forbid test asserts the `@id` of the policy that fires.
- [ ] The template instantiation for `tenant_apac_custom` is tested with at least one permit that depends on it and one forbid proving the template does not grant reads to the wrong region.
- [ ] The memo names the operation, the schema trade-offs, the template-vs-forbid choice, the latency note, and one `ai-risk-engineer` question.
- [ ] No test uses literal tenant IDs in a way that would make adding `tenant_four` require rewriting policies; the policies are tenant-agnostic.

## Stretch goals

- **Run the differential analyser.** Clone the [cedar-analysis](https://github.com/cedar-policy/cedar-analysis) tool. Add a widening to one of your policies (e.g. drop the `purpose_limitation` forbid) and run the analyser to detect it. Report the output; add a note on how a CI integration would use it.
- **Three-tier purpose vocabulary.** Extend T-4 to distinguish `consent.explicit`, `consent.contract`, `consent.legitimate_interest` (as per GDPR Article 6). Add tests proving a record consented under `contract` is not readable when the session's declared purpose only qualifies for `legitimate_interest`.
- **Bind to a stub data-access layer.** Write a 100-line service (any language with a Cedar binding) that reads a record ID, loads the record entity, builds the request, calls Cedar, returns allow/deny with the fired policy IDs. Package as a Docker image.
- **Verified Permissions deployment.** If you have AWS credentials, deploy the schema and policies to [Amazon Verified Permissions](https://aws.amazon.com/verified-permissions/) and run the test suite against the hosted runtime. Report any semantic differences you observe between the open-source Cedar and the hosted version.
- **Region-change sprint.** `tenant_eu_strict` acquires a UK subsidiary. Write the policy change required to allow a session in `eu-central-1` to read `uk-west-1` records when both regions are in the same `region_group`. Run the analyser on the before and after to prove you widened only the intended scope.
