# Cedar and ABAC — The Per-Tenant Data Boundary

Chapter 02 picked OPA / Rego for the RBAC layer (humans operating the platform) and the capability layer (agent sessions invoking tools). This chapter picks [Cedar](https://www.cedarpolicy.com/) for the ABAC layer — the one that decides *whether a specific request touches data that belongs to a specific tenant with the right classification, region, and consent*.

You can do ABAC in Rego. Many platforms do. The argument for Cedar at this seam is not that Rego cannot express the rules — it is that Cedar was designed from the schema outwards for exactly this shape of problem, has a formal small-step semantics with [a soundness proof](https://www.amazon.science/publications/cedar-a-new-language-for-expressive-fast-safe-and-analyzable-authorization), and ships [analysis tooling](https://github.com/cedar-policy/cedar-analysis) that answers *"does this policy change allow any request the old policy denied?"* at build time. For the tenant-boundary layer, where a bug is a cross-tenant data leak, the formal-analysis story is the differentiator.

## Why a second policy language at all

A platform engineer's first instinct is to pick one policy language and use it everywhere. Resist. The three layers — RBAC, ABAC, capability — have different shapes:

- **RBAC** is a graph walk over users → groups → roles → permissions. Small vocabulary, stable over months.
- **Capability** is a set-containment check with argument predicates. The policy is almost always "does this request fit any capability the session holds, under its predicates". Rego's partial rules and comprehensions handle this cleanly.
- **ABAC** is a predicate language over attributes on principal, resource, action, and context. The vocabulary is large (every attribute on every resource type a tenant can store), the policies are numerous (per-tenant, per-classification, per-regulatory-regime), and the correctness bar is high (a wrong predicate leaks data). Cedar's schema plus analysis plus small-step evaluation is pitched at exactly this.

Running both is not operational overhead if the plane already treats policy as a first-class artefact with CI, bundles, signing, and PDPs. The gateway calls OPA for capability / RBAC on the hot path; a tenant-data operation additionally calls Cedar before executing. The decision envelopes from both engines combine into a single plane-level envelope.

## What Cedar is, at a sentence

Cedar is an open-source policy language ([cedar-policy/cedar](https://github.com/cedar-policy/cedar)) originally developed at AWS and released in 2023. It evaluates `(principal, action, resource, context)` quads against a set of policies, each of which is `permit` or `forbid`, with a `when` and optional `unless` block, under a schema that declares entity types and their attributes. A decision is `Allow` if at least one `permit` policy applies and no `forbid` policy applies; `Deny` otherwise.

That is the whole mental model. The complexity is in the schema and the policy library you build on top.

## Entities, schemas, and the four-tuple

Cedar's four-tuple is the shape your decision request takes:

- **`principal`** — who is making the request. For the tenant-boundary layer, the principal is typically a session with attributes `{tenant: ..., user: ..., actor_workload: ..., max_classification: ..., session_region: ...}`.
- **`action`** — what is being done. `Action::"ReadRecord"`, `Action::"UpdateRecord"`, `Action::"ExportRecords"`. Actions are first-class entities.
- **`resource`** — what is being acted on. `Record::"r_01HV..."`, with attributes `{owner_tenant: ..., classification: ..., region: ..., legal_hold: ...}`.
- **`context`** — environment attributes (time, risk tier, feature flags, declared purpose).

The *schema* declares every type (`Session`, `Record`, `Tool`, `Action::"ReadRecord"`) with its attributes and the valid principal/resource types each action accepts. A policy against an undeclared attribute is a schema error at build time, not a runtime failure. This is the invariant Rego cannot give you.

A minimal schema fragment:

```cedar-schema
entity Tenant;

entity User in Tenant {
  email: String,
  groups: Set<String>
};

entity Session in Tenant {
  user: User,
  actor_workload: String,
  max_classification: String,
  region: String,
  purpose: String,
  opened_at: Long
};

entity Record in Tenant {
  owner_tenant: Tenant,
  classification: String,
  region: String,
  legal_hold: Bool,
  consented_purposes: Set<String>
};

action ReadRecord appliesTo {
  principal: [Session],
  resource: [Record],
  context: { declared_region: String, declared_purpose: String }
};

action ExportRecords appliesTo {
  principal: [Session],
  resource: [Record],
  context: { destination: String, declared_region: String }
};
```

Three design points:

1. **Hierarchies are first-class.** `User in Tenant` means every `User` entity has a parent `Tenant`. Policies can reason over the hierarchy (`principal in Tenant::"acme"`) without manually threading attributes.
2. **Attributes are typed.** `classification: String` not `classification: any`. Cedar's [analyser](https://github.com/cedar-policy/cedar/tree/main/cedar-policy-validator) catches mistyped comparisons at build time.
3. **Actions name which principals and resources they apply to.** An action that accepts a principal that is not a `Session` fails validation. The schema is a contract the gateway and the data-access layer both honour.

## The tenant-boundary policy library

Build the ABAC layer from a small number of policies, each with a single clear intent. A sketch of the library, with the four policies most platforms end up writing early:

### Policy T-1 — Owner tenant constraint

A session may only read records whose `owner_tenant` matches the session's tenant.

```cedar
permit (
    principal,
    action == Action::"ReadRecord",
    resource
)
when {
    resource.owner_tenant == principal.parent
};
```

Reads: "permit any principal to perform `ReadRecord` on any resource *when* the resource's `owner_tenant` equals the principal's parent tenant". `principal.parent` returns the parent entity in the hierarchy — for a `Session in Tenant`, that is the tenant. One policy covers the entire tenant-isolation invariant for reads.

### Policy T-2 — Region residency

Data classified by region may only be read by sessions configured for the same region. This is the policy that satisfies data-residency clauses in European contracts and in regulated-industry deals.

```cedar
forbid (
    principal,
    action in [Action::"ReadRecord", Action::"ExportRecords"],
    resource
)
when {
    resource.region != principal.region
};
```

Note it is a `forbid`, not a narrowing of the `permit`. A forbid cannot be overridden by any permit — this is the plane's hard rule for cross-region access. When a legitimate cross-region operation is needed (a tenant migrates regions, a disaster-recovery procedure runs), the operation is driven by a session configured in the new region, not by relaxing this policy.

### Policy T-3 — Classification ceiling

A session's `max_classification` caps the classification of records it can touch. This is the layer that lets a reduced-privilege session (set up for a less-trusted agent flow) read `pii.low` records while being blocked from `pii.high` records.

```cedar
forbid (
    principal,
    action == Action::"ReadRecord",
    resource
)
when {
    classification_rank(resource.classification) > classification_rank(principal.max_classification)
};
```

Cedar does not (at the time of writing) support user-defined functions in the policy language itself (<!-- needs-research: confirm the current state of extension functions / UDFs in the Cedar language; the project has an extensions proposal that has been moving. -->). The `classification_rank` mapping lives in Cedar [context](https://docs.cedarpolicy.com/policies/syntax-policy.html) as a pre-computed attribute, or is unrolled into a chain of `if/else` expressions in the policy itself:

```cedar
forbid (
    principal,
    action == Action::"ReadRecord",
    resource
)
when {
  (resource.classification == "pii.critical" && principal.max_classification != "pii.critical") ||
  (resource.classification == "pii.high" &&
    !(principal.max_classification == "pii.critical" ||
      principal.max_classification == "pii.high")) ||
  (resource.classification == "pii.medium" &&
    principal.max_classification == "pii.low")
};
```

Readability suffers. The alternative — pre-computing `classification_rank(x)` as `principal.max_classification_rank` (a `Long`) in the data-access layer before calling Cedar — is what most platforms do. The schema then declares `max_classification_rank: Long`, the policy compares `Long`s, and you get back a readable rule:

```cedar
forbid (
    principal,
    action == Action::"ReadRecord",
    resource
)
when {
    resource.classification_rank > principal.max_classification_rank
};
```

### Policy T-4 — Purpose limitation and consent

GDPR Article 5(1)(b) and the regulatory successors that cite it require data to be processed for declared, specific purposes. A session carries a declared `purpose`; a record carries the set of purposes its subject has consented to.

```cedar
forbid (
    principal,
    action == Action::"ReadRecord",
    resource
)
when {
    !(principal.purpose in resource.consented_purposes)
};
```

Purpose is the attribute that most often shows up in agent-platform audits. The declared purpose of the session is whatever the application told the plane at session-open time (`"support_triage"`, `"marketing_campaign"`, `"internal_analytics"`), carried through the whole call chain, and checked against the data subject's consent record every time. A missing `declared_purpose` fails closed.

## Templates for per-tenant variations

Not every tenant wants the same policy. A regulated-industry tenant may need an additional forbid for `legal_hold` records; a tenant with its own identity provider may need a different `region` vocabulary.

Cedar's [policy templates](https://docs.cedarpolicy.com/policies/templates.html) parameterise policies so each tenant's variation ships as a template instantiation, not a bespoke policy. A template:

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

A template-linked policy fills in `?principal` and `?resource_region` at policy-store time. The policy-store API records which template was instantiated with which parameters, so an auditor sees "tenant Acme has an instantiation of `tenant_region_override` for `Tenant::"acme"` with `resource_region = eu-central-1`" rather than reverse-engineering why Acme's policy differs.

Templates are the right shape when tenants need *parameters* on a policy. They are not the right shape when a tenant needs a *different* policy — that is a data decision (which policies are in the tenant's policy set), not a template decision.

## The decision shape in Cedar

A single Cedar evaluation returns an `Authorization Decision` with the result, the set of policies that `permit`ted, the set that `forbid` (which overrides), and the set of errors. The gateway composes this into the chapter-01 envelope:

```python
# pseudocode — exact binding depends on which Cedar binding you use
decision = cedar.authorize(
    policies=current_tenant_policy_set,
    schema=platform_schema,
    entities=request_entities,
    request=Request(
        principal=Session("s_01HV..."),
        action=Action("ReadRecord"),
        resource=Record("r_01HV..."),
        context={
            "declared_region": "eu-west-1",
            "declared_purpose": "support_triage",
        },
    ),
)

envelope = {
    "decision_id": allocate_ulid(),
    "decision": "allow" if decision.decision == "Allow" else "deny",
    "reason_code": first_reason_code(decision),
    "obligations": derive_obligations(decision),
    "evidence": {
        "input_hash": hash(canonical_json(request)),
        "policy_set_hash": hash(current_tenant_policy_set),
        "evaluator": "cedar@4.0",
    },
    "trace_id": otel_trace_id(),
}
```

Four things worth noting:

1. **The `policy_set_hash` is tenant-scoped.** Different tenants run different policy sets; the hash on the envelope identifies the exact set used.
2. **A `forbid` that fires carries the policy ID.** The reason code maps `forbid`-policy IDs to a vocabulary (`abac.tenant_region_mismatch`, `abac.classification_ceiling_exceeded`) the dashboards and the regression suite share.
3. **The entities are passed to the evaluator, not fetched by the evaluator.** Cedar does not call your database; the data-access layer stages the resource entity (with its attributes) and the session entity (with its attributes) into the request. The ABAC decision runs against *claims* about the entities, which is why those claims must be authentic.
4. **Attribute authenticity is a plane problem, not a Cedar problem.** If the caller can forge the resource's `region` attribute, the policy is defeated. The attribute source must be the authoritative store (the record's canonical row in your database), and the plane must prove that in the request pipeline. In most deployments this means the ABAC call runs in a data-access gateway that reads the resource row before calling Cedar.

## Where to put the Cedar PDP

Three deployment shapes:

### In the data-access layer, in-process

Cedar has [Rust](https://crates.io/crates/cedar-policy), [Java](https://github.com/cedar-policy/cedar-java), [TypeScript](https://www.npmjs.com/package/@cedar-policy/cedar-wasm) (via WASM), and [Go](https://github.com/cedar-policy/cedar-go) bindings. Evaluating in-process in the data-access layer (the service that already reads the row) is the lowest-latency, most defensible shape: the resource entity is already in memory, there is no second trip, and the evaluation is a function call.

Use this when: the data-access layer is a single service (or a bounded set of services) that can embed Cedar and share a bundle-pull loop with the policy service.

### As a sidecar PDP alongside the data-access service

Cedar runs as a standalone service; the data-access layer POSTs the request; it returns a decision. [Amazon Verified Permissions](https://aws.amazon.com/verified-permissions/) is the hosted equivalent of this shape.

Use this when: multiple services need ABAC decisions and you prefer centralising Cedar's bundle lifecycle over embedding it in every one.

### As a step in a central authorisation gateway

The gateway composes OPA (capability) and Cedar (ABAC) into a single authorisation call, taking the intersection and returning a unified envelope. The data-access layer calls the gateway, not the two PDPs directly.

Use this when: the plane is early, the number of services is small, and you want to hide the "two engines" shape from everyone downstream. The trade-off is one more network hop between the data-access layer and the authorisation call.

Most platforms end up with a mix: in-process Cedar in the data-access layer, in-process OPA in the gateway, and a thin control plane that owns both bundles. Latency budgets drive the detail.

## Validation and analysis

Cedar's [validator](https://docs.cedarpolicy.com/policies/validation.html) checks policies against the schema at build time — unknown attributes, mistyped comparisons, actions applied to the wrong principal / resource types all surface as errors instead of runtime deny-fires.

The [differential analysis tooling](https://github.com/cedar-policy/cedar-analysis) (currently in the `cedar-analysis` SMT backend) answers the question that most policy repositories cannot answer about Rego: *"does this proposed policy change permit a request the current policy denies, or deny a request the current policy permits?"* For a plane whose correctness bar is "no tenant leak", this is the difference between "I think the change is safe" and "I proved the change is safe within the schema".

Build a CI job around the analyser:

```bash
cedar validate --schema schema.cedarschema --policies policies/
cedar analyse --schema schema.cedarschema \
              --policies policies/ \
              --baseline main \
              --fail-on-policy-change-widening
```

The `--fail-on-policy-change-widening` check (pseudo-flag; the exact invocation depends on your analysis tool version <!-- needs-research: confirm the current cedar-analysis CLI flags and whether widening detection is bundled in the main CLI or exposed as a separate tool -->) blocks merges that would allow new requests without an explicit review. Chapter 06 picks this up and ties it to the abuse-corpus regression.

## When Cedar is not the right tool

Cedar is designed for a specific shape. It is not the right tool when:

- **Decisions depend on state the policy cannot see.** Cedar is pure over the `(principal, action, resource, context)` tuple. If a decision depends on the last 24 hours of behaviour (rate limits, cool-downs, anomaly detection), that is enforcement-plane logic, not a policy. Pre-compute into context or move the decision to a different layer.
- **The vocabulary is tiny.** For an RBAC decision over a dozen permissions, Rego or a hand-rolled check is less overhead. Cedar's value scales with schema complexity.
- **You need obligation-rich decisions.** Cedar is permit / forbid plus policy IDs. Obligations (redact, step-up, rate-limit) have to be derived by the caller from which policies fired, not from Cedar itself. OPA's `allow_with_obligations` shape is more direct.

A plane that mixes OPA and Cedar typically does so with Cedar for the data-touching policies (tenant, region, classification, purpose, consent) and OPA for everything else. Chapter 02's capability policy handles the tool-invocation question; Cedar handles the tenant-data question; the two are composed by the gateway.

## Summary

- Cedar is the right PDP for the per-tenant ABAC layer. Its schema-first design, formal small-step semantics, analysable policies, and typed attributes fit the shape where correctness bars are "no cross-tenant leak".
- Policies are expressed over the `(principal, action, resource, context)` four-tuple, with a schema that declares entities, actions, and attributes. Templates parameterise per-tenant variations.
- Four policies cover most planes early: tenant ownership, region residency, classification ceiling, purpose limitation + consent. The library grows with the harm model (chapter 07).
- The data-access layer or a dedicated sidecar runs Cedar; the authoritative attributes are staged into the request so the evaluator never phones home. Attribute authenticity is a plane problem, not Cedar's.
- CI validates policies against the schema and runs differential analysis against the previous version. The analyser is what answers "does this change widen access?"
- Cedar is not a universal replacement for Rego. OPA for capability / RBAC, Cedar for ABAC, both composed into the chapter-01 envelope.

The next chapter switches to the identity layer — how the plane knows *who* is calling, and how non-human agents, workers, and sidecars get authentic, short-lived credentials.
