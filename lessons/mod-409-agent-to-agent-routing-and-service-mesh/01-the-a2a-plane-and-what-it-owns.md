# The Inter-Agent Routing Plane and What It Owns

The runtime (mod-402) owns one agent's turn. The MCP gateway (mod-403) owns the tool catalogue. The policy plane (mod-405) owns the permission check. The model gateway (mod-406) owns the call to a provider. The observability plane (mod-408) owns the trace. Up to this module, every chapter has reasoned about one agent — one loop, one process, one identity, one trace. The moment agents start calling other agents — a writer agent handing off to a researcher agent, a supervisor agent routing to a coder agent, a sales agent delegating to a finance agent across tenants — a new plane shows up. The things that used to be in-process (the identity of the caller, the permissions that follow them, the correlation id on the trace, the fallback when a dependency hangs) are now on the wire. And the wire is the platform's problem.

The naive answer is "each agent team picks a framework, framework picks a transport, done." Within three months the platform has three copies of inter-agent routing (one per framework), two incompatible identity models, zero circuit breakers between two agents owned by different teams, and a cascading failure the first time a downstream agent is slow. The pattern is the same one that killed service-oriented architectures in the pre-mesh era, and the fix is the same: the platform owns the inter-agent wire, not the individual agent teams. The specific standard that gives the platform a chance of owning it coherently is [Agent2Agent (A2A)](https://a2a-protocol.org/) — an open protocol for agents to discover each other, exchange tasks, stream updates, and delegate work, which the platform maps onto its existing service-mesh primitives rather than re-inventing them.

This chapter fixes the slice. The chapters that follow pick the pieces: the A2A protocol surface and how it maps to a service mesh (02), inter-agent routing and service discovery (03), identity delegation and permission-scope propagation (04), circuit breakers and bulkheads against cascading failure (05), and the two agent-topology modes — handoffs vs. manager-as-tools — plus the handoffs to `agentic-systems-architect` and `senior-agentic-ai-engineer` (06).

## What the inter-agent routing plane owns

The plane sits between the runtime (mod-402) that hosts each agent process, the MCP gateway (mod-403) that each agent calls for tools, the policy plane (mod-405) that each agent consults for authorisation, the observability plane (mod-408) that captures the trace, and — on the far side — every pair of agents that need to call each other, whether they live in the same tenant, different tenants, or different organisations entirely. The plane owns:

- **The inter-agent wire protocol.** [Agent2Agent](https://a2a-protocol.org/) as the platform's north-star spec. Every agent on the platform exposes an A2A surface — [AgentCard](https://a2a-protocol.org/latest/specification/) for discovery, `tasks/send` for invocation, `tasks/get` and streaming for progress, task states (`submitted`, `working`, `input-required`, `completed`, `failed`, `canceled`) for the lifecycle. Chapter 02.
- **The service-mesh mapping.** A2A is a protocol, not an implementation. The thing that routes, retries, breaks circuits, encrypts, and attributes identity between two agents is a service mesh — [Istio](https://istio.io/) with [Envoy](https://www.envoyproxy.io/) as the data plane, [Linkerd](https://linkerd.io/), or a hand-rolled equivalent. The plane picks one, pins the data-plane proxy version, and configures the mesh so that an agent team writing against A2A does not see sidecar YAML. Chapter 02.
- **Service discovery and inter-agent routing.** An agent registry keyed by `agent_id` and tenant, backed by AgentCards. A routing layer that resolves a call to an upstream agent instance, honours tenant isolation at the resolver, and emits the right span attributes for mod-408 to reconstruct the chain. Chapter 03.
- **Correlation-ID propagation.** Every inter-agent call carries the `traceparent` (mod-408 chapter 03) and the platform's cross-cutting identifiers — `tenant_id`, `session_id`, `conversation.id`, and a `platform.turn_id` — on baggage. Chapter 03.
- **User-identity delegation.** The end user authenticated once at the edge. When agent A calls agent B on that user's behalf, agent B must know who the user is, what tenant they belong to, what scopes they actually have, and that the chain of delegation is cryptographically evidenced. The pattern is [OAuth 2.0 Token Exchange (RFC 8693)](https://datatracker.ietf.org/doc/html/rfc8693) with [JWT act/may_act](https://datatracker.ietf.org/doc/html/rfc8693#section-4.1) claims, bound to workload identity from [SPIFFE/SPIRE](https://spiffe.io/) at the mesh layer. Chapter 04.
- **Permission-scope propagation and narrowing.** Agent A has scopes `read:docs write:drafts send:email`. Agent A hands off to agent B to summarise. Agent B must never receive `send:email` — it does not need to send email, and the chain-of-trust decays as it extends. The plane narrows scopes at every hop by policy. Chapter 04.
- **Tenant isolation at the wire.** A call from tenant A's agent to tenant B's agent is a cross-tenant call and needs explicit authorisation, a separate audit record, and — in many deployments — a mesh-level deny by default. The plane enforces this at the routing layer, not at the application layer. Chapter 04.
- **Resilience primitives against cascading failure.** Circuit breakers ([Envoy outlier detection](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/outlier) and [circuit breaking](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/circuit_breaking)), bulkheads (connection pool and concurrency isolation per upstream), retry budgets, hedging, request-level timeouts, and backoff. Applied to inter-agent dependency graphs, not just HTTP service graphs. Chapter 05.
- **The two topology primitives.** The platform supports two inter-agent topology shapes: *handoffs* (agent A transfers the conversation to agent B and exits) and *manager-as-tools* (a manager agent calls subagents as tools and retains the top-level loop). Each shape rests on different platform primitives — handoffs need durable state transfer; manager-as-tools needs per-subagent concurrency budgets and a different fan-out circuit-breaker policy. Chapter 06.
- **The paved-road A2A client and server scaffolds.** The platform ships SDK scaffolds (mod-412) that wrap A2A with mesh-aware clients, baggage-aware servers, scope-narrowing delegate helpers, and the circuit breaker policy as configuration, not as per-team code.

The plane does **not** own:

- **Multi-agent *design*.** Whether the application decomposes into a writer / researcher / critic triplet or a single agent with tools is a product-architecture question. `agentic-systems-architect` (level 48) owns multi-agent design; the plane gives them the primitives to express whatever they design.
- **In-process agent loop logic.** A single agent's reasoning loop is the runtime (mod-402) and the paved-road SDK (mod-412). The plane only enters when the agent calls *another* agent.
- **Agent prompt design, tool selection, or evaluation methodology.** `senior-agentic-ai-engineer` (level 40) owns product-side implementation and prompt engineering. The plane gives them the inter-agent wire; they do the agent.
- **MCP tool calls.** A tool call is a *tool*, not an *agent*. The MCP gateway (mod-403) owns tool call routing. A2A owns agent-to-agent calls. The distinction matters: a tool has a short, stateless contract; an agent has a long, stateful task lifecycle. Chapter 02 is explicit about the line.
- **Policy decisions.** The policy plane (mod-405) decides whether a scope-narrowed delegation is allowed. This plane propagates the scope and enforces the resulting deny; it does not decide the rule.
- **Trace schema.** The observability plane (mod-408) owns `gen_ai.*` and `platform.*` attribute names. This plane emits spans that comply with them; it does not pick the vocabulary.
- **The model-provider wire.** mod-406 owns the call from model gateway to the provider. A2A is agent-to-agent, not agent-to-model.

Drift off this slice and the plane becomes "the platform-engineering team that writes all the agent code", or worse, "the ad-hoc ambassador between every pair of teams." Scope policing is a feature of this module.

## Why a plane and not a framework

A few concrete questions that are hard to answer without an inter-agent routing plane and trivial with one.

- **"My writer agent calls your researcher agent. The researcher hangs. Does my writer hang?"** Without circuit breakers and bulkheads applied at the mesh layer, yes — the writer's thread, slot, or connection pool fills up with pending researcher calls, and the writer's own callers start timing out. Chapter 05.
- **"A user filed a GDPR deletion request. The user talked to agent A, which handed off to agent B, which handed off to agent C. Do you have the full chain of audit records cross-referenced to one user identity?"** Without identity delegation that propagates through every A2A hop with a cryptographic trail, no — you have three per-agent logs and a guess. Chapter 04.
- **"Agent A tried to call agent B with `send:email` scope. Should it have?"** Without scope narrowing at the plane, the application code on agent A decides, which means the answer varies by team and by week. With narrowing, the plane strips scopes that the downstream agent's AgentCard does not declare it needs — the application cannot over-delegate by mistake. Chapter 04.
- **"Can I discover every agent in the fleet that satisfies the `research` capability and pick the cheapest one?"** Without an agent registry backed by AgentCards, the answer is "ask on Slack." With one, the question is a filter over structured metadata. Chapter 03.
- **"Agent A is in tenant 1. Agent B is in tenant 2. Can A call B?"** Without tenant isolation enforced at the resolver, the default answer in most routing stacks is "yes, if it knows the hostname." That is a data-exfiltration path. The plane denies by default and requires an explicit allow. Chapter 04.
- **"Framework X does agent calls one way; framework Y does them another. How do I debug a failure that crosses from X to Y?"** Without a single on-the-wire protocol, you debug two translation layers plus the failure. A2A is the lingua franca; teams pick whatever framework they want on top, but they speak A2A on the wire. Chapter 02.

These are the capabilities the plane delivers. They separate "we have agents that call each other" from "we have an inter-agent plane". Chapters 02–06 build each.

## The two topology modes

Chapter 06 is the full treatment; this section names the shapes so the rest of the chapter has vocabulary.

### Handoffs

Agent A is a routing agent. It receives the user's turn, decides the request belongs to agent B, and *hands off*: it transfers the conversation state to B, exits its own loop, and B becomes the top-level agent for the rest of the session (or until it hands off to C). [OpenAI's Agents SDK](https://openai.github.io/openai-agents-python/) and the earlier [OpenAI Swarm](https://github.com/openai/swarm) pattern are the canonical references; [Anthropic's "Building Effective Agents"](https://www.anthropic.com/research/building-effective-agents) describes the related *routing* pattern.

- **Platform primitives needed.** Durable conversation-state handoff (mod-402's durable execution), identity delegation with full scope transfer at the handoff point, idempotent session hand-off semantics, and a trace that stitches "A's turns" and "B's turns" under one `session_id` with a visible handoff span.
- **Resilience shape.** If B goes down after the handoff, A is gone; the user's session is stuck. The plane needs either a hand-back primitive or a supervisor outside A and B that can route around B.

### Manager-as-tools

Agent A is a supervisor. It receives the user's turn and keeps the loop. When it needs specialised work, it calls agent B as if B were a tool — one request, one response, A still owns the turn. [Anthropic's "Building Effective Agents"](https://www.anthropic.com/research/building-effective-agents) calls this the *orchestrator–workers* pattern. [OpenAI's "A Practical Guide to Building Agents"](https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf) describes the manager pattern with a similar shape.

- **Platform primitives needed.** Request/response A2A with streaming, per-subagent concurrency budgets (one slow subagent should not starve the others), fan-out circuit breaker policy, and scope narrowing per subagent (the supervisor has broad scope; each subagent gets only what it needs).
- **Resilience shape.** The supervisor is a bulkhead by construction — each subagent call is a separate upstream with its own circuit breaker. A failing subagent degrades the manager's answer; it does not take the manager down.

The platform does not pick one shape over the other. Teams pick based on the product. The plane offers the primitives for both, cleanly. Chapter 06 walks the platform primitives per shape.

## The threat model this plane defends against

Four classes of failure dominate what the inter-agent plane sees. They are what chapters 02–06 build mitigations for.

### Cross-boundary cascading failure

Agent A calls agent B. B is slow. A has no timeout, no circuit breaker, and shares a connection pool with every other upstream. A's request-handling slots fill with pending B calls. A's own callers — agents, users, other services — start timing out. The fleet-wide p99 climbs. Then B's own dependency goes slow, and the cascade reaches C.

The plane's defences:

- **Per-upstream circuit breakers.** Envoy's outlier detection + circuit breaker, applied per destination `agent_id`. Chapter 05.
- **Bulkheads.** Connection pool and concurrency limits per upstream, so one slow agent cannot drain the shared pool. Chapter 05.
- **Retry budgets and hedging.** Bounded retry counts per client, with tail-latency hedging where appropriate. Chapter 05.
- **Deadlines, not timeouts, propagated with the request.** The caller's deadline is in the request context; downstream agents refuse work they cannot complete before the deadline. Chapter 03 and 05.

### Identity and permission drift

Agent A calls agent B on a user's behalf. Agent B logs the call as if it were A's own action. Audit records show "agent B did X" and lose the original user. Or: agent A has `send:email` scope, hands off to agent B to summarise, and B's logs show it with `send:email` even though B has no email-sending code path. A compromised B now has an unneeded privilege.

The plane's defences:

- **Token exchange with `act` / `may_act` claims.** The downstream token names agent B as the acting party and the end user as the subject. ([RFC 8693](https://datatracker.ietf.org/doc/html/rfc8693).) Chapter 04.
- **Scope narrowing at the hop.** The plane strips scopes that B's AgentCard does not declare. B cannot receive scopes it does not need. Chapter 04.
- **Workload identity binding.** The outbound call carries A's SPIFFE ID in the mesh's mTLS; B's policy ties the user subject to A's workload identity in its audit. Chapter 04.
- **Cross-tenant deny by default.** The resolver refuses cross-tenant calls without an explicit allow in the registry. Chapter 03 and 04.

### Correlation loss across the mesh

Agent A starts a turn with `trace_id=T`. A calls B over A2A. The A2A client does not know about `traceparent`. B starts a fresh trace. The observability plane (mod-408) now has two disconnected traces for one user interaction, and the Article 72 evidence export (mod-408 chapter 06) is incomplete.

The plane's defences:

- **W3C Trace Context propagation on every A2A hop.** `traceparent` + `tracestate` in the HTTP headers (A2A is HTTP + JSON-RPC with SSE; the headers are a solved problem). Chapter 03 and cross-reference mod-408 chapter 03.
- **Baggage-based cross-cutting attributes.** `tenant_id`, `session_id`, `conversation.id`, `platform.turn_id`. The plane's SDK scaffold writes them on baggage and reads them as span attributes. Chapter 03.
- **A conformance test.** The platform's conformance suite (cross-reference mod-408 chapter 05) runs an A→B→C synthetic and asserts one `trace_id` end to end.

### Semantic drift between frameworks

Team 1 uses the Agents SDK. Team 2 uses LangGraph. Team 3 uses CrewAI. Each framework has its own internal notion of a "handoff" or a "subagent call". If two teams' agents need to call each other and both expose only their native framework's surface, the only way to integrate is per-pair custom glue. The platform grows O(N²) adapters.

The plane's defences:

- **A2A is the on-the-wire contract.** Every agent's external surface is A2A, regardless of what the agent's internal framework is. The glue is one way (framework ↔ A2A) and lives in the paved-road SDK. Chapter 02.
- **AgentCard is the public interface.** `.well-known/agent-card.json` (or the A2A equivalent the spec pins) names what an agent does, what scopes it needs, what tasks it accepts. Chapter 02 and 03.

<!-- needs-research: confirm the current A2A spec's exact filename and location for AgentCard discovery (.well-known/agent-card.json vs. .well-known/agent.json or similar) against the latest spec at https://a2a-protocol.org/. The spec moved to the Linux Foundation in 2025 and may have revised discovery conventions. -->

## The invariant: `A2ACallRecord`

Pin the record shape before writing the plane. Every inter-agent call produces one `A2ACallRecord`, correlated into the mod-408 trace by `trace_id`. The record is invariant across frameworks, deployments, and tenants. Every chapter adds a field to this record.

```json
{
  "call_id": "a2a_01HWX3K2M...",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "parent_span_id": "00f067aa0ba902b7",
  "span_id": "7b3210dcab4511ac",
  "tenant_id": "acme-financial",
  "session_id": "sess_01HV9...",
  "conversation_id": "conv_01HV9...",
  "turn_id": "turn_00005",
  "caller": {
    "agent_id": "writer-v1",
    "agent_revision": "git:abcd1234",
    "workload_id": "spiffe://platform.local/ns/tenants/sa/writer",
    "tenancy": "acme-financial"
  },
  "callee": {
    "agent_id": "researcher-v2",
    "resolved_instance": "researcher-v2-7dc9f.cluster-eu-west",
    "workload_id": "spiffe://platform.local/ns/shared/sa/researcher",
    "tenancy": "shared"
  },
  "a2a": {
    "operation": "tasks/send",
    "task_id": "task_01HV9...",
    "task_state": "working",
    "streaming": true
  },
  "topology": "manager-as-tools",
  "identity": {
    "end_user_subject": "user:01HV9...",
    "delegation_chain": ["writer-v1", "researcher-v2"],
    "scope_granted": ["read:docs"],
    "scope_requested": ["read:docs", "search:web"],
    "scope_narrowed_from": ["read:docs", "write:drafts", "search:web", "send:email"],
    "token_type": "urn:ietf:params:oauth:token-type:jwt"
  },
  "deadline_ms": 15000,
  "resilience": {
    "circuit_breaker_state": "closed",
    "retries_attempted": 0,
    "bulkhead_pool": "researcher-v2",
    "concurrent_calls_at_start": 3
  },
  "outcome": "completed",
  "latency_ms": 2340
}
```

Six invariants the record enforces:

1. **Every inter-agent call carries `trace_id` from the caller.** Correlation loss is a conformance failure. Chapter 03.
2. **Every call names caller and callee by both `agent_id` and workload identity.** Audit answers "which agent, running where" from the record alone. Chapter 04.
3. **`scope_narrowed_from` is populated whenever narrowing happened.** The plane makes scope reduction auditable. Chapter 04.
4. **Deadline is on the call, propagated to the callee.** Timeouts are the caller's responsibility; the callee honours them. Chapter 03 and 05.
5. **Resilience state is explicit.** Circuit breaker open, retry count, bulkhead pool are on the record so SRE (mod-410) can reason about fleet-wide resilience without joining to a side channel. Chapter 05.
6. **Topology mode is declared.** Whether this is a handoff or a manager-as-tools call changes the resilience interpretation and the trace layout. Chapter 06.

Chapters 02–06 build the pieces that produce this record.

## The three obligations

Pin the plane to three non-negotiable obligations. Everything in chapters 02–06 is a decision about how to deliver them.

1. **"One user request on agent A that delegates to agent B produces one trace and one identity chain."** If the auditor pulling a trace sees two disconnected traces or an orphan delegation, the plane has failed its primary job. Correlation and identity propagation are not optional. Chapters 03 and 04.
2. **"No agent can over-delegate privilege it does not need, and no agent can call outside its tenant without explicit authorisation."** Scope narrowing and tenant isolation are mesh-level, not application-level. An application that forgets to narrow does not cause an incident; the plane narrows regardless. Chapter 04.
3. **"A slow or failing downstream agent degrades the caller's output but does not take the caller down."** Circuit breakers, bulkheads, and deadlines applied to the inter-agent graph are the mechanism; the obligation is on the platform, not on each team's code. Chapter 05.

A plane that cannot defend all three is a framework for calling agents, not an inter-agent routing plane.

## Where the plane physically sits

A typical deployment shape:

- **The A2A client and server scaffolds.** Paved-road libraries per language (Python, TypeScript, Go per mod-412). Implement the A2A spec, wrap the mesh-aware HTTP client, write baggage on outbound, read baggage on inbound, participate in the scope-narrowing protocol.
- **The agent registry.** A service that stores one record per `(tenant_id, agent_id)`: the AgentCard, the workload identity, the current version, the deployed instances, the declared scopes, the circuit-breaker tier. Backed by a datastore the mesh can poll.
- **The service mesh control plane.** Istio's `istiod`, Linkerd's control plane, or equivalent. Pushes routing, mTLS, circuit-breaker, and outlier-detection config to the data plane. Reads the agent registry for destination configuration.
- **The data-plane proxies.** Envoy sidecars (Istio) or Linkerd's micro-proxy on every agent pod. Terminate inbound A2A, originate outbound A2A, enforce mTLS, apply retries/timeouts/circuit breakers, emit mesh telemetry.
- **The identity provider.** An OAuth 2.0 authorisation server supporting token exchange ([RFC 8693](https://datatracker.ietf.org/doc/html/rfc8693)). Issues the user's initial token at the edge; mints narrowed tokens at every A2A hop.
- **SPIFFE/SPIRE (or the mesh's equivalent).** Issues short-lived SVIDs to every agent workload. The mesh uses them for mTLS; the plane uses them to bind agent identity to the audit record.
- **The conformance test harness.** Runs A2A protocol conformance against every onboarded agent, scope-narrowing conformance against every delegation path, trace-propagation conformance across the full graph. Cross-reference mod-408 chapter 05 for the pattern.

If you cannot draw this topology on a whiteboard and name the box every A2A call flows through, chapter 02 is the thing to read next.

## Summary

- The inter-agent routing plane owns the A2A wire, the service-mesh mapping, the agent registry, correlation propagation, identity delegation, scope narrowing, tenant isolation, circuit breakers and bulkheads, and the primitives for the two topology modes. It does not own multi-agent design, agent prompt logic, MCP tool calls, policy decisions, trace vocabulary, or the model wire.
- Two topology modes coexist on the plane: *handoffs* (A transfers control to B and exits) and *manager-as-tools* (A keeps the loop and calls B as a tool). Each needs different platform primitives; the plane offers both.
- The failures the plane defends against are cross-boundary cascading failure, identity and permission drift, correlation loss across the mesh, and semantic drift between frameworks. Chapters 02–06 build the mitigations.
- Every inter-agent call produces an `A2ACallRecord` keyed on `trace_id`: caller and callee, A2A operation, delegation chain, narrowed scopes, deadline, circuit-breaker state, outcome. Every chapter adds a piece.
- Three non-negotiable obligations: one trace and one identity chain per user request; no over-delegated privilege and no un-authorised cross-tenant calls; downstream failure degrades output, not the caller.

The next chapter opens the first piece — the A2A protocol surface and how the platform maps it onto the service mesh's existing primitives, so an agent team gets a wire protocol without inheriting sidecar YAML.
