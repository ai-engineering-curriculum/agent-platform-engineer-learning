# The A2A Protocol and the Service-Mesh Mapping

Chapter 01 named the slice. This chapter picks the first piece: the on-the-wire protocol every agent on the platform speaks (A2A) and how the plane maps it onto the existing service-mesh primitives (Istio/Envoy or Linkerd) so an agent team gets A2A without having to learn xDS.

The temptation is to pick one and ignore the other. A team that only reads [A2A](https://a2a-protocol.org/) ends up implementing their own retry logic, timeout logic, mTLS, identity attribution, and outlier detection in application code — and getting one of them wrong. A team that only configures the mesh ends up inventing their own JSON shape over HTTP — and losing framework interop in the first cross-team integration. The paved road is both: A2A is the semantic contract between agents; the mesh is the operational contract between workloads. The two meet at the sidecar, and the platform is what makes the meeting deterministic.

## What A2A is

[Agent2Agent (A2A)](https://a2a-protocol.org/) is an open protocol for agent-to-agent interoperation — originally announced by Google and donated to the Linux Foundation in 2025. The design goal is the one that matters here: a framework-neutral way for one agent to discover another, send it a task, stream progress, and receive structured output, over standard web transports an operations team already knows how to run. The pieces this module cares about:

- **Transport.** HTTP(S). Request/response and streaming.
- **Messaging format.** [JSON-RPC 2.0](https://www.jsonrpc.org/specification) over HTTP for synchronous methods, plus Server-Sent Events for streaming.
- **Core surface.** `tasks/send`, `tasks/get`, `tasks/cancel`, `tasks/pushNotification/set`, and streaming variants. The names are stable enough to design against; the exact method list and parameter shapes should be read from the current spec before shipping.
- **Discovery.** An `AgentCard` — a JSON document describing an agent's identity, skills, input / output modalities, authentication requirements, and endpoint — served at a well-known path.
- **Task lifecycle.** `submitted → working → input-required ⇄ working → completed | failed | canceled`. The states let a client reason about progress; the semantics match a durable-execution step more than a request/response.
- **Content.** `Message`s with typed `Part`s (text, file, structured data) and `Artifact`s for outputs. Multi-modal by construction.
- **Push notifications.** A webhook pattern so an agent can tell a client "the task has progressed" without the client holding a stream open.

<!-- needs-research: A2A's method inventory, discovery path, and task-state alphabet have been revised since the initial announcement. Before shipping against this chapter's names, read the authoritative spec at https://a2a-protocol.org/ and pin the revision in your AgentCard. -->

### The AgentCard

AgentCard is the public interface an agent exposes to the platform's registry and to callers. The exact field set is spec-version-sensitive, but the shape is stable:

```json
{
  "name": "researcher-v2",
  "description": "Research agent that takes a topic and returns cited findings.",
  "url": "https://researcher-v2.svc.cluster.local/a2a",
  "version": "2.4.1",
  "capabilities": {
    "streaming": true,
    "pushNotifications": true,
    "stateTransitionHistory": true
  },
  "authentication": {
    "schemes": ["bearer"]
  },
  "defaultInputModes": ["text"],
  "defaultOutputModes": ["text", "application/json"],
  "skills": [
    {
      "id": "research.web",
      "name": "web-research",
      "description": "Research a topic using web sources.",
      "tags": ["research", "web"],
      "inputModes": ["text"],
      "outputModes": ["application/json"]
    }
  ],
  "platform": {
    "tenancy": "shared",
    "declared_scopes": ["read:docs", "search:web"],
    "workload_id": "spiffe://platform.local/ns/shared/sa/researcher",
    "resilience_tier": "T2",
    "topology_modes": ["manager-as-tools", "handoff"]
  }
}
```

The `platform` block is a platform-specific extension — this module's convention, not A2A's. Chapters 03 and 04 use it for registry lookup, scope-narrowing policy, and circuit-breaker tiering.

### Tasks and the lifecycle

An A2A task is a long-lived unit of work with a deterministic state machine. A caller sends `tasks/send` with a `taskId` (client-generated for idempotency), a `sessionId`, a `message`, and optional `pushNotification` config. The server moves the task through states:

- `submitted` — accepted, not yet started.
- `working` — the agent is making progress.
- `input-required` — the agent needs more input from the caller before continuing. The caller responds with another `tasks/send` referencing the same `taskId`.
- `completed` — final artifact available.
- `failed` — terminal; error attached.
- `canceled` — the caller invoked `tasks/cancel`.

Streaming (`tasks/sendSubscribe` or the equivalent in the current spec) is a convenience: the caller opens an SSE stream and the server emits task-state updates and incremental messages until the task reaches a terminal state. Without streaming, the same information is reachable by polling `tasks/get` or by registering a webhook via `tasks/pushNotification/set`.

The lifecycle matters because it changes what the mesh can and cannot do. A2A tasks are longer-lived than a typical HTTP request; chapter 05's circuit-breaker policy has to reason about stream-level failures rather than per-request ones, and chapter 03's retries are idempotent only because `taskId` is client-generated.

## What a service mesh is

A service mesh is the infrastructure layer between services that handles routing, encryption, identity, retries, timeouts, circuit breaking, outlier detection, and observability — without the services themselves knowing. Two shapes dominate as of this writing:

- **[Istio](https://istio.io/) with [Envoy](https://www.envoyproxy.io/) sidecars.** The dominant general-purpose mesh. The control plane (`istiod`) pushes configuration over [xDS](https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds_protocol) to Envoy sidecars on every pod. Envoy terminates and originates all service traffic. Istio adds workload identity ([SPIFFE](https://spiffe.io/)-based SPIFFE IDs), mTLS by default, traffic policy (`VirtualService`, `DestinationRule`), and telemetry.
- **[Linkerd](https://linkerd.io/).** The minimalist mesh. A Rust-based micro-proxy (`linkerd2-proxy`) on every pod. mTLS by default, retries and timeouts, telemetry, workload identity. Fewer features than Istio; simpler to operate. [Buoyant, Linkerd's steward](https://buoyant.io/), publishes the operator model.
- **Ambient / sidecarless modes.** Istio ambient mesh, Cilium service mesh, and others ship a node-level data plane without per-pod sidecars. For this module, the data-plane primitive is still the same (an Envoy / proxy at the mesh edge of the workload); the deployment topology differs.

What the mesh does for the inter-agent plane:

- **mTLS between every pair of agents.** No application code needed; the sidecars terminate and originate. Every connection is encrypted and identity-asserted.
- **Workload identity.** Every workload has a cryptographic identity (a SPIFFE SVID or equivalent). The mesh binds it to the TLS handshake. Chapter 04 uses this for audit attribution.
- **Routing policy.** Which upstream a request goes to, with what retry / timeout / hedging policy, under what circuit-breaker budget. Expressed in mesh CRDs, not in application code.
- **Outlier detection.** The data plane ejects hosts that fail a threshold of requests, re-admits after a backoff. Chapter 05 wires this into the inter-agent dependency graph.
- **Telemetry.** The mesh emits spans and metrics on every hop. Cross-reference mod-408 chapter 03 — the mesh's span is the one that carries the W3C Trace Context when the application is lax about it.
- **Egress and ingress gateways.** Dedicated proxies for traffic entering or leaving the mesh. The inter-agent plane uses an egress gateway when an agent calls a *remote* A2A endpoint (a different cluster, a different organisation).

What the mesh does **not** do:

- **Semantic validation.** The mesh moves bytes; it does not verify that a `tasks/send` request has the required fields or that a task-state transition is legal.
- **Scope narrowing.** The mesh carries the token; the plane's scaffold is what narrows the scope before the call goes out (chapter 04).
- **Agent-level resilience.** A circuit breaker at the pod level does not know that agent B is the specialist for topic X and agent C is a fallback. The plane's resilience policy sits on top of the mesh's primitives (chapter 05).

## The mapping: A2A surface to mesh primitive

The table below is the module's north star. Every A2A concept has a mesh-level realisation; every mesh primitive has an A2A-level consumer.

| A2A concept | Mesh primitive |
| --- | --- |
| HTTPS transport | mTLS terminated at the sidecar; the agent process talks plaintext HTTP to its local proxy. |
| JSON-RPC request/response | HTTP routing rule on the destination `agent_id` + method; mesh retries + timeouts applied per rule. |
| SSE streaming (`tasks/sendSubscribe`) | HTTP/1.1 or HTTP/2 upgrade; mesh stream-level timeout + circuit-breaker policy. |
| Push notification webhook | Outbound HTTP from the server-side agent; mesh egress policy + retries. |
| AgentCard (`/.well-known/...`) | Public, cacheable HTTP GET; mesh allows-list on unauthenticated endpoints. |
| Bearer authentication | Application-layer token (chapter 04 details the OAuth 2.0 + token-exchange flow); mesh also enforces mTLS on top. |
| Task `taskId` | Idempotency key; mesh retries are safe only because `taskId` is client-generated. |
| Task state `input-required` | A paused HTTP request; mesh keeps-alive policy must tolerate the pause, or the plane uses push notification instead. |
| Task `cancel` | HTTP `DELETE` or JSON-RPC `tasks/cancel`; mesh deadline is informative but not canonical (cancel must reach the server even if the deadline passed). |
| AgentCard `authentication.schemes` | The identity provider the mesh integrates with ([OIDC](https://openid.net/specs/openid-connect-core-1_0.html), SPIFFE-federated OIDC, or similar). |
| AgentCard `capabilities.streaming` | Whether the mesh's `DestinationRule` (or equivalent) must enable HTTP/2 upstream. |

The pattern is "A2A is semantic; mesh is operational". When in doubt, put the knob on the layer that owns the semantics it depends on. Scope narrowing is semantic; keep it in the plane's scaffold. Outlier ejection is operational; keep it in the mesh.

## What the inter-agent plane actually provides as paved road

The plane ships three concrete artefacts that bind A2A and the mesh together. Teams consume these; they do not re-invent them.

### 1. The A2A client scaffold

A platform-blessed HTTP client that:

- Reads the target agent's AgentCard from the registry.
- Resolves the destination URL by `agent_id`, not by hostname — the mesh's `VirtualService` turns the logical name into the pod set.
- Injects `traceparent` and baggage (`tenant_id`, `session_id`, `platform.turn_id` per chapter 03).
- Attaches the identity-delegated bearer token (chapter 04).
- Attaches the propagated deadline as a header (`x-platform-deadline-ms` or equivalent — read by the server scaffold on the far side, chapter 05).
- Rejects calls to agents outside the caller's tenant unless the registry marks the destination as cross-tenant allowed.
- Emits the `A2ACallRecord` from chapter 01.

The team's code is:

```python
from platform_a2a import A2AClient

client = A2AClient.for_agent(target_agent_id="researcher-v2")
task = client.tasks_send(
    session_id=ctx.session_id,
    message=user_message,
    metadata={"topic": "quarterly-outlook"},
)
async for update in client.tasks_stream(task.id):
    ...
```

Everything else — mTLS, retries, circuit-breaker state, scope narrowing, trace propagation — is in the client.

### 2. The A2A server scaffold

A platform-blessed server framework that:

- Serves the AgentCard from `/.well-known/agent-card.json` (or the spec-pinned equivalent).
- Validates inbound JSON-RPC requests against the A2A spec version the server declares.
- Extracts the identity-delegated bearer, verifies it, and surfaces the end-user subject and delegation chain to the handler.
- Extracts `traceparent` + baggage; starts the server span with the right attributes.
- Extracts the propagated deadline; refuses to start work the server cannot finish by it.
- Enforces that the invoked skill is declared on the server's AgentCard.
- Emits task state updates on the configured stream / push-notification channel.

The team's code is handler-only:

```python
from platform_a2a import A2AServer, skill

server = A2AServer.from_agent_card("./agent-card.json")

@skill("research.web")
async def research_web(request, ctx):
    user = ctx.end_user_subject
    tenant = ctx.tenant_id
    ...
    async for partial in research_iterate(request):
        yield partial
```

### 3. The mesh configuration generator

A thin tool that reads the agent registry and generates:

- `VirtualService` / `DestinationRule` CRDs for Istio (or the Linkerd equivalent) per registered agent.
- Circuit-breaker and outlier-detection policy keyed by the `resilience_tier` on the AgentCard (chapter 05).
- `AuthorizationPolicy` that denies cross-tenant calls unless the registry records an allow.
- `PeerAuthentication` requiring mTLS (`STRICT`) on every pod the plane manages.

Teams do not edit the mesh YAML. They update the registry; the generator updates the mesh. [Argo CD](https://argo-cd.readthedocs.io/) or [Flux](https://fluxcd.io/) can own the apply loop.

## Where A2A ends and MCP begins

A reasonable question — "is calling another agent just calling a tool?" — is a trap that most platforms fall into once. The two protocols overlap in shape but differ in contract.

- **MCP (mod-403) is for *tools*.** A tool is a short, stateless, function-call-shaped operation. "Search the web for X." "Read file Y." "Send an email with body Z." The contract is one request, one response, no mid-call state. The policy plane (mod-405) checks the call against a short list of allowed actions.
- **A2A is for *agents*.** An agent is a stateful, long-lived party with its own loop, its own identity, its own tools. "Research this topic and produce a report with citations." The contract is a task with a lifecycle, streaming, and the possibility that the callee needs to ask the caller for more input mid-task.

In practice:

- **Short, stateless, single-skill, no loop of its own → MCP tool.** Wrap it in an MCP server; call it over the MCP gateway. The caller does not need to know about AgentCards, task states, or streaming.
- **Long, stateful, multi-skill, has its own agentic loop, may delegate further → A2A agent.** Expose an AgentCard; wrap it in the A2A server scaffold. The caller gets streaming, task lifecycle, and the identity-delegation chain.

The platform team is the arbiter. A product team that insists "we only need to call this tool, but we want streaming and task cancellation" is a hint that the tool has outgrown MCP and belongs behind A2A. A product team that wraps an A2A agent around a one-shot API call is a hint that it should have been MCP. Chapter 06's handoff section discusses the architect-level review process for the question.

A single team's agent can of course be *both* an A2A agent and a consumer of MCP tools. The agent's own skills surface in its AgentCard; the tools it calls internally go through MCP. The plane's conformance test checks that nothing crosses the line — an A2A server cannot also expose itself as an MCP server under the same workload identity, because the audit record shapes are different.

## A concrete trace: one A2A call end to end

Follow a single `tasks/send` from writer-v1 (caller) to researcher-v2 (callee) under the mesh, assuming both agents have paved-road scaffolds, Istio with Envoy, and SPIFFE-based workload identity.

1. **Writer-v1's handler decides to delegate.** It calls `A2AClient.for_agent("researcher-v2").tasks_send(...)`.
2. **Client scaffold looks up researcher-v2 in the registry.** Gets the AgentCard, caches locally, resolves the logical hostname `researcher-v2.a2a.svc.cluster.local`.
3. **Client scaffold mints a narrowed token.** Via RFC 8693 token exchange with the identity provider: writer-v1's user-delegated token in, a new token out with `act=researcher-v2`, scope `read:docs search:web` (writer-v1's scope intersected with researcher-v2's declared scopes). Chapter 04 is the full flow.
4. **Client scaffold builds the HTTP request.** POST to `/a2a`, JSON-RPC body `{"method":"tasks/send", ...}`, headers `authorization: Bearer <narrowed-token>`, `traceparent: ...`, `baggage: tenant_id=acme-financial,session_id=sess_01HV9`, `x-platform-deadline-ms: 15000`.
5. **The call leaves writer-v1's pod.** The local Envoy sidecar intercepts. It initiates mTLS to researcher-v2's sidecar using writer-v1's SPIFFE SVID.
6. **Mesh routing kicks in.** `VirtualService` routes to the `researcher-v2` service. `DestinationRule` picks a healthy instance via the load-balancing policy and consults the circuit breaker: current state is closed, bulkhead has capacity, call proceeds. (Chapter 05.)
7. **mTLS completes at researcher-v2's sidecar.** `AuthorizationPolicy` checks: writer-v1 is in the same tenant namespace as researcher-v2's "shared" tenancy — allowed. The sidecar forwards plaintext HTTP to the local researcher-v2 process.
8. **Researcher-v2's server scaffold receives the request.** Validates JSON-RPC shape and the declared skill (`research.web`). Verifies the bearer token against the identity provider's JWKS — accepts. Extracts the `act` chain: user → writer-v1 → researcher-v2. Extracts `traceparent` and starts a server span parented to writer-v1's client span. Extracts the deadline (15000 ms); records it.
9. **Handler runs.** It emits `working` updates on the stream, may itself make further MCP tool calls (chapter 02's boundary) or further A2A calls (recursive case), and reaches `completed`.
10. **Response flows back.** Streaming `completed` state over SSE, through the sidecar, back to writer-v1's client.
11. **Client scaffold finalises the `A2ACallRecord`.** Outcome `completed`, latency measured, retries 0, circuit-breaker state `closed`. Emits the record to mod-408.

Every piece in that flow has a chapter later in this module that pins its details. The point of the walkthrough is that from the application's perspective — writer-v1's handler — the whole thing is `tasks_send(...)` and an async iterator. The plane is the reason that works.

## Summary

- A2A is the semantic contract: AgentCard for discovery, task lifecycle for the call shape, JSON-RPC + SSE for the transport. The service mesh is the operational contract: mTLS, workload identity, routing, retries, circuit breaking, telemetry. The plane binds them.
- The platform provides three paved-road artefacts: the A2A client scaffold (injects trace, identity, deadline; emits the call record), the A2A server scaffold (serves AgentCard, validates request, extracts identity and trace), and the mesh config generator (turns the registry into `VirtualService` / `DestinationRule` / `AuthorizationPolicy` CRDs).
- A2A is for *agents* — stateful, long-lived, multi-skill parties with their own loops. MCP is for *tools* — short, stateless, function-shaped operations. The plane enforces the boundary; a product team's insistence on blurring it is a signal to re-scope.
- Every A2A concept has a mesh-level realisation. When the semantics and the operation disagree, put the knob on the layer that owns the semantics; the mesh is for moving bytes correctly, the plane is for making the call make sense.
- A concrete A2A call under the plane is `tasks_send(...)` in application code. The scaffold, the sidecar, the registry, the identity provider, and the mesh conspire so the team does not see the rest.

The next chapter pins the first of those pieces in detail: service discovery, correlation-ID propagation, and how the plane routes an inter-agent request so that one external user interaction produces one coherent trace from the first turn to the last handoff.
