# OPA and Rego — Authoring Agent-Tool-Invocation Policy

[Open Policy Agent](https://www.openpolicyagent.org/) is the general-purpose policy engine the industry has converged on for RBAC and capability-style decisions across platforms. It is a CNCF graduated project, authored in Go, run as a sidecar or library, and driven by [Rego](https://www.openpolicyagent.org/docs/latest/policy-language/) — a declarative policy language whose closest intellectual ancestor is Datalog.

This chapter writes two things against OPA: the RBAC rules that gate the *operator* actions on the platform, and the capability rules that gate the *agent* actions on tools. The next chapter does per-tenant ABAC in Cedar. The split is deliberate — OPA fits the capability shape cleanly; Cedar fits the ABAC shape cleanly; the plane can use both.

## Why OPA for this slice

OPA is a good fit for the capability and RBAC parts of this plane because:

- **It runs as a sidecar.** Decisions are local, sub-millisecond, and the PDP has no egress dependencies for the hot path. Policy bundles (compiled, signed) are pulled asynchronously from a bundle service.
- **The request shape is explicit.** Every decision is a function from a JSON `input` document to a JSON `data` document plus a set of Rego rules. If the input shape is missing a field, the policy fails closed; there is no implicit state.
- **Rego expresses "allow if all of these hold" and "deny if any of these holds" naturally.** Capability rules look like set-containment and attribute-compare, which Rego does well.
- **Decision logging is a first-class feature.** OPA emits structured decision logs that carry the input hash, the policy bundle hash, the result, and the evaluator version. The compliance envelope from chapter 01 falls out of this almost for free.
- **The ecosystem is deep.** Conftest for CI-time policy testing, Styra DAS and Minder for policy management, OPA bundles signed with Cosign or Sigstore, [Rego Playground](https://play.openpolicyagent.org/) for prototyping.

OPA is not the best fit for every slice. Chapter 03 argues Cedar for ABAC because Cedar's schema-first design is better for tenant-scale attribute policies. Many platforms run both.

## The decision request shape

Fix the input before writing policy. The gateway (or the runtime, or the broker — whichever component is the PEP for this seam) constructs the input; OPA evaluates against it; the decision envelope from chapter 01 falls out. A typical shape for an agent tool-call decision:

```json
{
  "action": {
    "kind": "tool.invoke",
    "tool_id": "calendar-mcp/find_free_slot",
    "tool_version": "2.4.1",
    "arguments": {
      "attendee": "alice@acme.com",
      "range": {"start": "2026-11-10T09:00:00Z", "end": "2026-11-10T17:00:00Z"}
    },
    "arguments_hash": "sha256:a3f5...",
    "requested_scope": "calendar.read"
  },
  "session": {
    "id": "s_01HV8X3M9F5A6R2B4C0D7E8G9H",
    "tenant_id": "acme",
    "principal": {
      "kind": "user",
      "subject": "alice@acme.com",
      "idp": "okta",
      "groups": ["eng", "all-staff"]
    },
    "actor": {
      "kind": "workload",
      "spiffe_id": "spiffe://platform.local/gateway/prod"
    },
    "opened_at": "2026-11-06T18:10:00Z",
    "capability_bundle_hash": "sha256:9ce4...",
    "capabilities": [
      { "tool_id": "calendar-mcp/find_free_slot",
        "scope": ["calendar.read"],
        "max_calls": 50,
        "expires_at": "2026-11-06T19:10:00Z" },
      { "tool_id": "notes-mcp/*",
        "scope": ["notes.read"],
        "max_calls": 200,
        "expires_at": "2026-11-06T19:10:00Z" }
    ]
  },
  "environment": {
    "region": "eu-west-1",
    "risk_tier_declared": "T2",
    "timestamp": "2026-11-06T18:22:04.173Z"
  }
}
```

Three design points worth internalising:

1. **The argument values and the argument hash are both carried.** The policy sees the values so it can enforce argument-level constraints; the hash is what lands on the decision envelope so an auditor can verify arguments against a hash later without the policy itself depending on the full payload size.
2. **`principal` is who consented; `actor` is who is executing on their behalf.** Both land in the decision log. Confused-deputy audits require both.
3. **The capability bundle is on the input, not fetched by the policy.** Fetching capabilities on every decision is how you discover under load that your policy depends on a database. Pre-stage.

## Rego, in just enough depth

Rego is small. Four concepts cover ninety percent of the policies this plane writes.

### Packages and rules

A `.rego` file declares a package (`package platform.tool.invoke`) and defines rules. A rule has a head, an optional value, and a body of expressions that must all be true. Rules with the same head whose bodies all pass combine as a logical OR; rules with different heads can be composed.

```rego
package platform.tool.invoke

import future.keywords.if
import future.keywords.in

default allow := false

allow if {
    some cap in input.session.capabilities
    cap.tool_id == input.action.tool_id
    input.action.requested_scope in cap.scope
    time.parse_rfc3339_ns(cap.expires_at) > time.parse_rfc3339_ns(input.environment.timestamp)
}
```

Default-deny is a convention worth hard-coding: a `default allow := false` at the top of every policy that produces an `allow` decision means that an input the policy does not match returns deny, not undefined.

### Partial rules — sets, objects, arrays

Rego builds collections from partial rules. A `deny` rule with multiple reasons accumulates every reason that fires:

```rego
deny contains reason if {
    not matching_capability
    reason := {
        "code": "capability.no_match",
        "text": sprintf("no capability covers tool_id=%s with scope=%s",
                         [input.action.tool_id, input.action.requested_scope])
    }
}

deny contains reason if {
    some cap in input.session.capabilities
    cap.tool_id == input.action.tool_id
    cap.max_calls <= cap.calls_so_far
    reason := {"code": "capability.exhausted",
               "text": sprintf("capability for %s exhausted (%d/%d)",
                                 [cap.tool_id, cap.calls_so_far, cap.max_calls])}
}
```

A decision envelope's `reason_code` and `reason_text` come from exactly this pattern. All firing reasons land in the envelope; the caller may surface the first for UI purposes.

### Composition and `import`

Policies are files; files group into packages; packages import each other. The RBAC rules for operators and the capability rules for agents live in different packages so they can be tested, reviewed, and shipped independently. The decision entry point is a single `main` package that imports both and dispatches on `input.action.kind`.

### Helpers — functions, comprehensions, built-ins

Rego has functions (`f(x) := y if ...`), set/array comprehensions (`{x | some cap in input.session.capabilities; x := cap.tool_id}`), and a substantial built-in library ([OPA built-ins reference](https://www.openpolicyagent.org/docs/latest/policy-reference/)) for strings, JWT, regex, time, graph, and crypto. Lean on built-ins — a Rego policy that reimplements JWT verification is a policy that will drift from spec.

A practical heuristic: if your policy has more than three nested comprehensions, you are describing an ABAC decision and chapter 03 is probably the right home.

## RBAC for operators

The operator layer is the first thing to write and the easiest to get right. The vocabulary is small. The decision entry point is `platform.operator.action`.

### Role and permission data

Roles and permissions live in the policy bundle as data, not code. A role definition is a flat JSON document:

```json
{
  "roles": {
    "platform_admin": {
      "permissions": [
        "tool.admit", "tool.deprecate", "tool.revoke",
        "policy.publish", "policy.rollback",
        "broker.secret.rotate", "broker.secret.view_metadata"
      ]
    },
    "tool_owner": {
      "permissions": ["tool.admit", "tool.update_schema"],
      "constraints": {"tool_owner_match_required": true}
    },
    "on_call": {
      "permissions": ["tool.revoke", "policy.rollback"],
      "constraints": {"approval_required": "platform_admin", "max_delay_seconds": 300}
    },
    "tenant_admin": {
      "permissions": ["tenant.user.add", "tenant.capability.grant"],
      "constraints": {"tenant_match_required": true}
    }
  },
  "role_bindings": {
    "acme/eng": ["tool_owner"],
    "acme/platform-admins": ["platform_admin"],
    "acme/oncall-rotation": ["on_call"]
  }
}
```

The Rego side is:

```rego
package platform.operator.action

import future.keywords.if
import future.keywords.in
import data.platform.rbac.roles
import data.platform.rbac.role_bindings

default allow := false

user_roles contains role if {
    some group in input.session.principal.groups
    some binding, role_list in role_bindings
    binding == group
    role := role_list[_]
}

has_permission(action) if {
    some role in user_roles
    action in roles[role].permissions
}

tenant_scope_satisfied(role) if {
    not roles[role].constraints.tenant_match_required
}

tenant_scope_satisfied(role) if {
    roles[role].constraints.tenant_match_required
    input.session.tenant_id == input.action.target_tenant
}

allow if {
    has_permission(input.action.name)
    some role in user_roles
    input.action.name in roles[role].permissions
    tenant_scope_satisfied(role)
}
```

Three design points:

1. **Role bindings are keyed by IdP group.** The IdP is the source of truth for group membership; you do not duplicate user lists in the policy bundle. The nightly sync job that writes role bindings is itself a thing to audit, but it is not policy-plane code.
2. **Constraints are expressed as Rego predicates, not free text.** `tenant_match_required: true` is a field the policy knows how to interpret; a new constraint is a new predicate, reviewed in code.
3. **Approval-required actions produce an obligation, not a deny.** When `approval_required` is set on a role, the policy returns `allow_with_obligations`; the caller enforces the approval UI. Chapter 05 goes deep on approval workflows.

### Testing an RBAC rule

Rego has a built-in testing framework ([Policy testing docs](https://www.openpolicyagent.org/docs/latest/policy-testing/)):

```rego
package platform.operator.action_test

import future.keywords.if
import data.platform.operator.action

test_platform_admin_can_admit_tool if {
    action.allow with input as {
        "action": {"name": "tool.admit", "target_tenant": "acme"},
        "session": {
            "tenant_id": "acme",
            "principal": {"groups": ["acme/platform-admins"]}
        }
    }
}

test_tool_owner_cannot_rotate_broker_secrets if {
    not action.allow with input as {
        "action": {"name": "broker.secret.rotate", "target_tenant": "acme"},
        "session": {
            "tenant_id": "acme",
            "principal": {"groups": ["acme/eng"]}
        }
    }
}
```

Chapter 06 scales this pattern across the abuse corpus. For now the shape is enough — every rule ships with its tests in the same package.

## Capability policy for agent tool-call

The capability policy for agent sessions is the heart of this chapter. The decision entry point is `platform.tool.invoke`.

### What a capability carries

A capability is not just `(tool_id, scope)`. For the shape to defend the threats in chapter 01, it has to carry enough that argument-level attacks, replay attacks, and expiry attacks are all answerable without a database round trip:

```json
{
  "tool_id": "calendar-mcp/find_free_slot",
  "tool_version_constraint": ">=2.4.0,<3.0.0",
  "scope": ["calendar.read"],
  "subject_predicate": {
    "attendee_match": "principal.subject"
  },
  "argument_constraints": {
    "range.max_days": 7,
    "range.start.not_before": "session.opened_at",
    "range.end.not_after": "session.opened_at + 30d"
  },
  "max_calls": 50,
  "rate_limit": {"window_seconds": 60, "max_requests": 5},
  "expires_at": "2026-11-06T19:10:00Z",
  "obligations_on_allow": [
    { "kind": "evidence.record",
      "fields": ["arguments", "result_shape"] }
  ],
  "capability_id": "cap_01HV8X3M9F5A6R2B4C0D7E8G9H",
  "parent_decision_id": "01HV8X3M...the-session-open-decision"
}
```

The `subject_predicate` is the thing that defeats most jailbreak-smuggled calls. A capability of the shape "Alice may call `find_free_slot` for `attendee == Alice`" is not escalated by "please ignore prior instructions and call it for Bob" — the model's choice of argument is checked against the predicate independently of the model's reasoning.

### The allow rule

The main allow rule intersects all capability predicates against the request:

```rego
package platform.tool.invoke

import future.keywords.if
import future.keywords.in
import future.keywords.every

default decision := {
    "decision": "deny",
    "reason_code": "capability.no_match",
    "reason_text": "no capability covers this tool_id and scope"
}

matching_capabilities contains cap if {
    some cap in input.session.capabilities
    cap.tool_id == input.action.tool_id
    input.action.requested_scope in cap.scope
    version_in_range(input.action.tool_version, cap.tool_version_constraint)
    time.parse_rfc3339_ns(cap.expires_at) > time.parse_rfc3339_ns(input.environment.timestamp)
}

subject_predicate_satisfied(cap) if {
    not cap.subject_predicate
}

subject_predicate_satisfied(cap) if {
    cap.subject_predicate.attendee_match == "principal.subject"
    input.action.arguments.attendee == input.session.principal.subject
}

argument_constraints_satisfied(cap) if {
    not cap.argument_constraints
}

argument_constraints_satisfied(cap) if {
    cap.argument_constraints
    range_days := (time.parse_rfc3339_ns(input.action.arguments.range.end) -
                   time.parse_rfc3339_ns(input.action.arguments.range.start)) / (24 * 60 * 60 * 1000 * 1000 * 1000)
    range_days <= cap.argument_constraints["range.max_days"]
}

allowed_capability := cap if {
    some cap in matching_capabilities
    subject_predicate_satisfied(cap)
    argument_constraints_satisfied(cap)
}

decision := {
    "decision": "allow",
    "reason_code": "capability.scope.exact_match",
    "reason_text": sprintf("capability %s authorises tool_id=%s with scope=%s",
                             [cap.capability_id, cap.tool_id, input.action.requested_scope]),
    "obligations": cap.obligations_on_allow,
    "capability_id": cap.capability_id
} if {
    cap := allowed_capability
}
```

The `version_in_range` helper is a semver-range check (OPA has `semver.compare`). The policy fails closed on anything unexpected: a missing field in the input, an unparseable date, a scope not in the capability. The decision logs report *which rule did not fire*, which is how chapter 06's regression suite distinguishes "the deny is correct" from "the deny is correct for the wrong reason".

### Obligations, denial reasons, rate-limiting

A capability can carry an *obligation on allow* — a directive the PEP must honour before the call completes. Three common obligations:

- **`evidence.record`.** Instructs the gateway to attach the named fields to the compliance record. Chapter 06 of mod-404 picks up the evidence envelope.
- **`approval.step_up`.** Instructs the PEP to pause the call and prompt the user for consent before continuing. The approval returns a signed artefact the gateway attaches to the decision log.
- **`rate_limit`.** The PDP cannot enforce rate-limiting directly (the PDP is stateless on the hot path). The obligation names the key, the window, and the max — the PEP (or a dedicated rate-limit service) enforces.

Denials cascade reason codes:

```rego
deny_reasons contains reason if {
    count(matching_capabilities) == 0
    reason := {"code": "capability.no_match", "severity": "info"}
}

deny_reasons contains reason if {
    some cap in matching_capabilities
    not subject_predicate_satisfied(cap)
    reason := {"code": "capability.subject_predicate_violation", "severity": "warn",
               "capability_id": cap.capability_id}
}

deny_reasons contains reason if {
    some cap in matching_capabilities
    not argument_constraints_satisfied(cap)
    reason := {"code": "capability.argument_constraint_violation", "severity": "warn",
               "capability_id": cap.capability_id}
}
```

Every `deny_reasons` entry lands on the decision envelope. The on-call dashboard groups denies by reason code; a spike in `capability.subject_predicate_violation` is a red-team signal (see chapter 06 on abuse corpus).

## Integrating OPA into the gateway

Three integration modes are plausible, and the choice affects latency, blast radius, and deployment complexity.

### Sidecar HTTP API

OPA runs as a sidecar container on the gateway pod; the gateway POSTs the input to `/v1/data/platform/tool/invoke/decision`; OPA returns JSON. Pros: language-agnostic, OPA's version is independent from the gateway's. Cons: an HTTP round trip, even on localhost, adds ~0.5–2 ms at p99, which is non-trivial on a hot path.

### In-process via `go-opa`

The gateway imports OPA as a Go library and evaluates in-process. Pros: lower latency (sub-ms for a compiled policy), fewer moving parts. Cons: the gateway must be in Go (or a language with a compatible binding); upgrading OPA means rebuilding the gateway.

### WASM-compiled policies

OPA policies compile to WASM; the gateway embeds a WASM runtime and evaluates the compiled policy in-process from any language. Pros: sub-ms, language-agnostic, policy version and gateway version are independent. Cons: not every OPA built-in is available in WASM-compiled form (the [OPA WASM support](https://www.openpolicyagent.org/docs/latest/wasm/) matrix moves; verify at reading).

Most production gateways run one of the first two; WASM is the right shape when you have a heterogeneous polyglot stack. All three are legitimate; the one you cannot do is "run OPA across the network from the gateway" — the latency and failure-mode consequences are severe.

## Decision logs

OPA's [decision logging](https://www.openpolicyagent.org/docs/latest/decision-logs/) emits a structured record for every evaluated decision — input, result, metrics, bundle revision, evaluator version. The default shape needs light customisation to match the chapter-01 envelope:

- **Hash the input server-side before emission.** The decision log can carry the full input, a redacted projection, or just the hash. For most call-path decisions you want the hash plus a redacted projection of policy-relevant fields; the full input lives in the compliance evidence stream (mod-404 chapter 06) if at all.
- **Attach the `decision_id` the gateway allocated.** OPA does not know about ULIDs; you pass it on the input and echo it back through a `decision_id` field.
- **Sign the log on emission.** Each entry's hash chains with the previous; a log that cannot prove it has not been truncated is not an audit log.
- **Route to the evidence stream with the gateway's identity (SPIFFE SVID).** Chapter 04 covers SVID issuance; mod-408 picks up the stream.

OPA can also emit logs to a Kafka or S3 sink via its built-in [log plugins](https://www.openpolicyagent.org/docs/latest/configuration/#decision-logs). Build toward "logs land on the compliance stream, indexed by `decision_id`, retained per the regulatory horizon in mod-404 chapter 06".

## Bundle lifecycle

Policies are code; the code lives in git; the bundle is the shipped artefact. Operational shape:

1. **Policy author opens a PR.** The PR is reviewed by someone with `policy.publish` permission; chapter 06's regression suite runs in CI; `conftest` checks the policy against the abuse corpus.
2. **CI builds the bundle.** `opa build` produces a signed `.tar.gz` with the Rego, the data, a manifest, and a Cosign signature (or an OPA-native signing key).
3. **The bundle is pushed to the bundle service.** [OPA bundles](https://www.openpolicyagent.org/docs/latest/management-bundles/) can be served from S3, GCS, or a dedicated bundle service.
4. **PDPs pull the bundle on a schedule.** Each PDP verifies the signature, atomically swaps, and reports its new bundle revision to the control plane.
5. **The PEPs record which bundle revision was in effect for every decision.** When a decision is replayed six months later, the auditor fetches the exact bundle hash.

Emergency rollback is a critical path: chapter 05 goes into the deprecation / revocation ladder. For now, pin the invariant: **the bundle revision is on the decision envelope**, and the bundle service retains every revision for at least the regulatory horizon.

## Latency budget

A plane-wide latency budget for policy-plane decisions on the hot path:

- OPA sidecar HTTP: p50 ~0.3 ms, p99 ~1.5 ms for a well-written policy.
- OPA in-process: p50 ~0.05 ms, p99 ~0.5 ms.
- OPA + attribute fetch from a database: do not do this on the hot path.

If your policy breaches the budget, profile with [OPA's `--metrics` and `--profile` flags](https://www.openpolicyagent.org/docs/latest/profiling/). Common culprits: `walk` over a large data document, `regex.match` on a non-anchored pattern, repeated `time.parse_rfc3339_ns` calls (parse once at the top of the rule). Pre-staging data into indexed Rego structures (e.g. a map keyed by tool_id instead of an array) frequently removes a p99 spike.

## Summary

- OPA is the right PDP for the RBAC and capability slices of this plane. It runs locally, decides in sub-ms, and emits decision logs natively.
- Rego is small — packages, rules, partial rules, imports, built-ins — and expresses capability predicates and RBAC role / permission rules naturally.
- The RBAC layer binds IdP groups to roles to permissions, enforces tenant-match constraints, and emits approval obligations for sensitive actions.
- The capability layer intersects session capabilities against the request, enforces subject predicates and argument constraints, and emits rate-limit and evidence obligations on allow.
- Integration is sidecar HTTP, in-process Go, or WASM; all three are legitimate. Decision logs are signed, routed to the evidence stream, and tagged with the bundle revision for replay.
- Bundles ship through a signed pipeline — git → CI → bundle service → PDP. The bundle revision is on the decision envelope, so every decision is replayable against the exact rules that produced it.

The next chapter opens Cedar and walks the per-tenant ABAC layer: the one where "which tenant's data can this request touch" is the question.
