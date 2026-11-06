# Inter-Agent Routing and Service Discovery

Chapter 02 pinned the contract. This chapter pins the mechanism: when writer-v1 says "call researcher-v2", what *actually* happens between the application's `tasks_send(...)` call and the bytes landing at a specific researcher-v2 pod — and what has to be on the wire for the trace to stay coherent and for the call to land on an instance the caller is allowed to reach.

The naive answer is "DNS, HTTP, done." That gets you a connection; it does not get you a routing decision that honours tenant isolation, version pinning, and resilience tiering, and it does not keep the trace stitched together. Every piece this chapter builds is already standard somewhere in the distributed-systems stack. The work is marrying them to A2A's semantics so an agent team is not reinventing any of it.

## The agent registry

The registry is the source of truth for "which agents exist, what do they expose, who is allowed to call them, and how does the mesh find them." It replaces the mental model of "DNS plus some config files."

```
┌──────────────────────────────────────────────────────────────┐
│  Agent Registry (per-platform, multi-tenant)                 │
├──────────────────────────────────────────────────────────────┤
│  agent_id: researcher-v2                                     │
│  tenant_id: shared                                           │
│  version: 2.4.1                                              │
│  agent_card: { ... }                                         │
│  workload_id: spiffe://platform.local/ns/shared/sa/researcher│
│  endpoint: a2a.svc.cluster.local:8080/a2a                    │
│  instances: [pod-abc, pod-def, pod-ghi]                      │
│  resilience_tier: T2                                         │
│  declared_scopes: [read:docs, search:web]                    │
│  cross_tenant_allow: [acme-financial, bigco]                 │
│  topology_modes: [manager-as-tools, handoff]                 │
│  deprecated_after: null                                      │
└──────────────────────────────────────────────────────────────┘
```

Every column is load-bearing for one of the chapters: `agent_card` for 02, `workload_id` + `declared_scopes` + `cross_tenant_allow` for 04, `resilience_tier` for 05, `topology_modes` for 06, `instances` + `endpoint` for this chapter. The registry is the one record every other plane reads from.

A pragmatic implementation: a small HTTP service backed by a relational database (Postgres is the default on the track — see mod-402). Mesh configuration is derived (chapter 02's mesh config generator). Changes go through a review gate; the registry's audit log is a first-class artefact.

## The discovery flow

A2A specifies AgentCard discovery via a well-known HTTP path on the agent itself. That path is the right primitive for cross-organisation calls (a different platform's agent is reachable, its AgentCard is a GET away). Inside the platform, the paved-road route is cached-in-the-registry — the registry is the aggregator, and the per-agent well-known endpoint is the fallback for fresh discovery and for external callers.

The flow:

1. **Caller invokes `A2AClient.for_agent(target="researcher-v2")`.** The client scaffold holds a short-lived in-process cache keyed by `(tenant_id, agent_id)`.
2. **On cache miss, the client queries the registry.** `GET /registry/agents/researcher-v2?tenant=acme-financial`. The registry returns the full record (AgentCard + platform metadata) or a 404.
3. **The registry resolves the scope.** If `tenant=acme-financial` and the target is in the `shared` tenancy, the registry checks `cross_tenant_allow`. If `acme-financial` is listed, the call is allowed; otherwise the registry returns a 403 and the client refuses to dial.
4. **The client caches the record.** TTL is short (seconds to tens of seconds) — long enough to amortise the lookup across a conversation's turns; short enough that an AgentCard revision propagates within a conversation.
5. **The client builds the destination.** The `endpoint` field is a logical mesh hostname (`researcher-v2.a2a.svc.cluster.local`), not a pod IP. Resolving to a specific pod is the mesh's job (see "load balancing" below).
6. **External callers bypass the registry.** A caller in a different platform or a different organisation fetches the AgentCard from the well-known path directly — `GET https://agents.example.com/.well-known/agent-card.json`. The platform's edge gateway serves this; the tenant isolation check happens at authentication, not at registry lookup.

<!-- needs-research: confirm the exact well-known path and media type the current A2A spec mandates for AgentCard discovery. The repository and the hosted spec have diverged across revisions; pin the one your scaffold emits. -->

## Logical hostnames and the mesh's job

Inside the mesh, every agent is addressable by a logical hostname that does not depend on the current pod set. The naming convention this module recommends:

```
<agent_id>.a2a.svc.cluster.local
```

(Adjust the domain to the cluster's actual DNS; the key point is the `a2a` label that separates agent endpoints from arbitrary services.)

The mesh's `VirtualService` (Istio) or `ServiceProfile` (Linkerd) binds the logical hostname to the Kubernetes `Service` that fronts the agent's Deployment. The `DestinationRule` (or equivalent) adds:

- **Subset selection by version.** `version: 2.4.1` subsets can be selected by header for canaries (chapter 05).
- **Load balancing.** The default (least-request or round-robin — pick one, pin it platform-wide) is usually right. [Envoy's load-balancing docs](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/load_balancers) walk the choices; the cross-cutting concern is that *the same* policy applies to every agent, so operator judgement about a slow instance is comparable.
- **Connection pool size.** Sets the mesh-level bulkhead (chapter 05).
- **Outlier detection.** Threshold for ejecting a bad instance (chapter 05).
- **mTLS mode.** `STRICT` on every inter-agent edge; `DISABLE` only where an explicit cross-tenant or external integration demands it (chapter 04).

The team's application code does not see any of this. The client scaffold hits `https://researcher-v2.a2a.svc.cluster.local/a2a`; the local sidecar does the rest.

## Routing policies the plane supports

The plane exposes a small, deliberate set of routing policies. Teams can apply them by annotating the AgentCard; the mesh config generator (chapter 02) turns the annotations into mesh CRDs.

### Pin to a version

A caller that wants a specific version of the callee (because it is mid-conversation and the state shape matters) writes it on the call:

```python
client = A2AClient.for_agent("researcher-v2", version="2.4.1")
```

The client scaffold adds `x-platform-agent-version: 2.4.1` to the outbound request; the mesh's `VirtualService` has a match rule that routes to the `v2.4.1` subset when that header is present.

### Canary / shadow routing

The registry supports a per-agent canary record: "5% of traffic for researcher-v2 goes to version 2.4.2; the rest to 2.4.1." The mesh config generator turns the percentages into weighted route destinations. [Istio's traffic management documentation](https://istio.io/latest/docs/concepts/traffic-management/) is the reference.

- **Canary.** Live traffic to the new version; errors count.
- **Shadow.** Duplicate traffic (fire-and-forget mirroring) to the new version; errors go to a separate metric. Istio's `mirror` field. Useful for validating a new agent's behaviour against real conversations without the user seeing its output.

Canary + shadow decisions are a routing concern, not an application concern. The team's `A2AClient.for_agent("researcher-v2")` does not change.

### Fallback to a sibling agent

An AgentCard can declare a fallback list: "if researcher-v2 is unavailable, try researcher-v1." The mesh config generator turns this into retry policy with a destination override; the client scaffold emits a single `A2ACallRecord` with the fallback destination in `resolved_instance`.

Fallbacks are a separate concept from retries (chapter 05). A retry targets the same `agent_id`. A fallback targets a different one.

### Geographic / zonal preference

Multi-cluster, multi-region platforms route preferentially to in-zone agents to keep latency down. [Istio's locality load balancing](https://istio.io/latest/docs/tasks/traffic-management/locality-load-balancing/) is the reference. The plane sets this as a platform-wide default; teams do not need to configure it per agent.

## Correlation-ID propagation across the hop

Chapter 01 and mod-408 chapter 03 cover the vocabulary. The inter-agent plane's job is making the propagation inevitable — the paved-road scaffolds do it; the conformance test (cross-reference mod-408 chapter 05) fails any agent that does not.

### What has to be on every outbound A2A call

Headers the client scaffold injects on every call:

| Header | Source | Required |
| --- | --- | --- |
| `traceparent` | OTel propagator from current context | Yes |
| `tracestate` | OTel propagator from current context | When present |
| `baggage` | OTel propagator; carries `tenant_id`, `session_id`, `conversation.id`, `platform.turn_id` | Yes |
| `authorization` | Token-exchange result (chapter 04) | Yes |
| `x-platform-deadline-ms` | Caller's remaining deadline | Yes |
| `x-platform-agent-version` | If version-pinned call | Conditional |
| `x-platform-conversation-id` | Convenience mirror of baggage; some servers prefer a dedicated header | Yes (either here or on baggage) |

Reading them on the server:

- **`traceparent` / `tracestate`.** The server scaffold's middleware calls the OTel propagator's `extract()`; the server span becomes a child of the client span. One `trace_id` end to end.
- **`baggage`.** The scaffold reads baggage and sets the resulting entries as span attributes (`platform.tenant_id`, `platform.session_id`, `platform.conversation_id`, `platform.turn_id`). Every emitted span carries them without the handler doing anything.
- **`authorization`.** Verified against the identity provider's JWKS; the subject, the delegation chain, and the scopes are surfaced on the request context (chapter 04).
- **`x-platform-deadline-ms`.** Converted into an absolute deadline against the server's clock (adjust for clock skew — [Google's "Time, Clocks, and the Ordering of Events"](https://dl.acm.org/doi/10.1145/359545.359563) and [AWS's "Keep your servers in sync"](https://aws.amazon.com/blogs/aws/keeping-time-with-amazon-time-sync-service/) are the primers if the team has not thought about it). Refuse work that cannot finish in time.

### The `platform.turn_id` baggage entry

A platform convention specific to this module: `platform.turn_id` is a monotonic counter inside a session that lets the observability plane group spans by "which turn the user saw." Agent B's call to agent C in the context of agent A's turn 5 should carry `turn_id=turn_00005` on baggage so that mod-408's `TraceRecord` ties everything under one `turn_id` bucket regardless of which process emitted the span. The scaffold writes it on baggage automatically from the current turn's context.

### The AgentCard discovery call

A nuance: the `GET /.well-known/agent-card.json` call that retrieves an AgentCard is itself an outbound call. The platform's convention is:

- For intra-platform calls, the registry serves the AgentCard, so this is a plain HTTP GET to the registry — correlated like any other internal call.
- For cross-organisation discovery, the external GET goes through the egress gateway. The egress gateway strips `authorization` (the external endpoint does not authenticate the discovery call), preserves `traceparent` / `baggage` (the trace is still ours), and emits the outbound span.

## The deadline convention

Deadlines, not timeouts, propagate with inter-agent requests. The distinction matters:

- A **timeout** is the caller's local clock budget: "I will wait 15 seconds for the response." If the caller times out, the server still happily computes the answer and wastes the work.
- A **deadline** is a propagated promise: "The whole request must finish by time T." Every hop refuses to start work it cannot finish before T, and cancels in-flight work when T passes.

The plane's rule: every A2A call carries `x-platform-deadline-ms` with the *remaining* deadline, not the original. When writer-v1 has 15 s remaining and calls researcher-v2, researcher-v2 receives `x-platform-deadline-ms: 15000`. If researcher-v2 then calls fact-checker-v1 after 3 s of work, fact-checker-v1 receives `x-platform-deadline-ms: 12000`. If fact-checker-v1's handler estimates it will take 20 s, it refuses the call immediately and returns `failed` with reason `deadline_exceeded`.

This is standard distributed-systems practice; [gRPC's deadlines](https://grpc.io/blog/deadlines/) is the clearest reference. The novelty for the inter-agent plane is that agent calls are long enough and the chain deep enough (three, four, five hops) that a timeout-only policy wastes real work.

<!-- needs-research: confirm whether the current A2A spec has standardised a deadline / timeout header. If so, use that header name; if not, document the `x-platform-deadline-ms` extension explicitly in the AgentCard. -->

## Idempotency keys and retries at the right layer

A2A tasks are idempotent by `taskId`: calling `tasks/send` twice with the same `taskId` is defined to produce one task. That is the key the mesh's retry policy relies on.

The plane's rule:

- **Client scaffold generates `taskId` as a UUID v7** (time-ordered, globally unique). One task per logical intent, not one per transport-level request.
- **Mesh retries are safe only on transport failures** (connection reset, 5xx without a response body, idempotent-marked errors). The `DestinationRule`'s retry policy says `retryOn: connect-failure,refused-stream,5xx` and `attempts: 3`. The server receives the same `taskId` three times; it de-duplicates.
- **Application-level retries are the client scaffold's job.** A `failed` task with `reason: deadline_exceeded` is not retried silently — the caller gets the failure and decides. A `failed` task with `reason: transient_backend` can be retried by the client with a new `taskId` and a retry budget (chapter 05).
- **Streaming tasks cannot be mesh-retried.** If the SSE stream breaks mid-task, the mesh re-establishes the connection via `tasks/sendSubscribe` with the same `taskId`; the server resumes streaming from where it left off (if it supports `stateTransitionHistory`) or re-emits (if it does not). Spec-version-sensitive; test against the server scaffold.

## Topology-aware routing for handoffs

Chapter 06 is the full treatment of the two topology modes. For routing, the key fact is that a *handoff* changes the routing identity for the rest of the conversation.

- In **manager-as-tools**, every call is a one-shot A2A request from the manager to a subagent. The routing identity stays the manager's. Each call is routed independently.
- In **handoff**, the conversation's "top-level agent" changes. Future user turns on this `session_id` route to the new agent, not the old one. The session-affinity layer (on top of the registry) records which agent owns the session now; the entry gateway (the thing that receives user requests from the client) consults this record on every turn.

The session-affinity record:

```json
{
  "session_id": "sess_01HV9...",
  "tenant_id": "acme-financial",
  "current_agent_id": "researcher-v2",
  "handoff_history": [
    {"from": "router-v1", "to": "writer-v1", "at": "2026-11-06T14:22:03Z", "turn_id": "turn_00001"},
    {"from": "writer-v1", "to": "researcher-v2", "at": "2026-11-06T14:23:12Z", "turn_id": "turn_00005"}
  ],
  "last_updated_turn_id": "turn_00005"
}
```

Durable storage, linearised per `session_id` to avoid two concurrent handoff writes. The entry gateway reads this on every turn; the scaffold's `handoff(to="researcher-v2")` helper writes it as the handoff completes. Chapter 06 details the handshake.

## Cross-cluster, cross-region, cross-organisation

Not every A2A call is within one cluster. The plane supports three dial-out shapes:

- **Intra-cluster.** The common case. Mesh handles everything.
- **Cross-cluster, same platform.** A multi-cluster mesh (Istio multi-cluster, Linkerd multi-cluster) spans clusters; the logical hostname resolves across clusters; mTLS spans too. [Istio multi-cluster installation](https://istio.io/latest/docs/setup/install/multicluster/) is the reference.
- **Cross-organisation.** A2A to an agent on another platform. The egress gateway originates the call over mTLS with a federated identity (SPIFFE federation or an exchanged token). The registry records the remote agent's AgentCard URL; the client scaffold routes through the egress gateway automatically. The mesh does *not* promise mTLS to the far side; the authorisation posture shifts to the token layer (chapter 04).

Cross-organisation calls are a different threat model than intra-platform ones. The plane's rule: a cross-organisation call is explicitly declared on the AgentCard's `platform.external` block; the calling team must opt in; the observability plane's trace shows a visible boundary span (`gen_ai.operation.name=handoff_external`) so audits can distinguish internal and external delegation.

## The conformance checklist for a new A2A server

Before a new agent lands on the registry, the platform's onboarding runs an executable conformance test. The checklist:

- **AgentCard served at the spec-pinned path.** `GET /.well-known/agent-card.json` returns a well-formed card.
- **AgentCard declares at least one skill.** An agent with no skills is a misconfiguration.
- **AgentCard declares `platform.tenancy`, `platform.declared_scopes`, `platform.workload_id`, `platform.resilience_tier`.** Platform metadata is complete.
- **`tasks/send` with a well-formed message returns a task with `submitted` or `working` state.** Basic contract works.
- **`tasks/send` with the same `taskId` twice returns the same task.** Idempotency honoured.
- **Streaming (`tasks/sendSubscribe` or equivalent) emits at least `working` and a terminal state.** Streaming contract honoured.
- **The server rejects a call without a bearer token with 401.** Authentication is enforced.
- **The server rejects a call whose bearer is valid but whose scope does not cover the invoked skill with 403.** Scope narrowing is honoured (chapter 04).
- **The server emits a span parented to the inbound `traceparent`.** Correlation is unbroken.
- **The server honours `x-platform-deadline-ms` — a call with `deadline_ms=1` returns `failed: deadline_exceeded` before doing meaningful work.** Deadline propagation is honoured.
- **A cross-tenant call is denied by the mesh's `AuthorizationPolicy`** unless the registry allows it.

This is the chapter-01 invariant list, executable. A team that passes it is on the registry; a team that fails it fixes the failure before shipping. The conformance harness is the one from mod-408 chapter 05 with an A2A-specific rule pack.

## Summary

- The agent registry is the source of truth for inter-agent routing: it holds the AgentCard, the workload identity, the version, the instances, the tenant-isolation allowlist, and the resilience tier. The mesh configuration is derived from it; teams update the registry, not the mesh YAML.
- Discovery is registry-first inside the platform, well-known-path at the edge. Both produce an AgentCard; both honour tenant isolation at discovery time, before any call is made.
- Routing inside the mesh is by logical hostname (`<agent_id>.a2a.svc.cluster.local`). Version pinning, canary, shadow, fallback, and locality preference are all mesh-level concerns configured from registry metadata, not application code.
- Correlation and context propagation are mandatory on every A2A hop: `traceparent`, `baggage` with tenant/session/conversation/turn identifiers, the bearer token, and the propagated deadline. The paved-road scaffolds make the propagation inevitable; the conformance test catches the lapses.
- Deadlines (not timeouts) propagate with the request. Each hop refuses work it cannot finish in time. gRPC's deadline pattern applied to A2A.
- Task `taskId` is the idempotency key; mesh retries are safe only because of it. Application-level retries are the scaffold's job and are policy-bounded (chapter 05).
- Topology matters at the routing layer only for handoffs — a handoff rewrites the session's top-level agent and future user turns route to the new agent. Manager-as-tools calls are routed independently per call.
- Cross-cluster calls stay inside the mesh; cross-organisation calls go through the egress gateway with a declared opt-in on the AgentCard and a visible external-handoff span.

The next chapter picks up the identity layer that the routing layer promised to carry: user-identity delegation, permission-scope propagation, scope narrowing, and tenant isolation from the audit record's perspective.
