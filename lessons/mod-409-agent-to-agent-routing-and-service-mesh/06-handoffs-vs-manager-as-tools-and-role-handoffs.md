# Handoffs vs. Manager-as-Tools and Role Handoffs

Chapter 01 named the two inter-agent topology modes. Chapters 02–05 built the plane without picking one. This chapter finishes the module by comparing the two topologies in platform terms — what each needs from the plane, what each costs, what each fails at — and then hands off the role boundary to `agentic-systems-architect` (level 48) for multi-agent *design* and `senior-agentic-ai-engineer` (level 40) for product-side subsystem implementation.

The frame to keep: the inter-agent plane does not decide which topology an application uses. It gives both topologies equally first-class primitives so the choice is a *product-architecture* call, not a *platform constraint*.

## The two topologies

Both shapes are well-documented in the agent literature. [Anthropic's "Building Effective Agents"](https://www.anthropic.com/research/building-effective-agents) describes the *routing* and *orchestrator–workers* patterns; [OpenAI's "A Practical Guide to Building Agents"](https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf) describes the *manager* pattern alongside explicit handoffs. [OpenAI's Agents SDK](https://openai.github.io/openai-agents-python/) elevates handoffs to a first-class primitive; [LangGraph](https://langchain-ai.github.io/langgraph/) exposes supervisor-and-workers patterns; [CrewAI](https://docs.crewai.com/) favours manager-as-tools with hierarchical crews. The names vary; the shapes are the same.

### Handoffs

Agent A receives a turn. A decides the request belongs to agent B. A *transfers* the conversation: subsequent turns in the same session go to B, not back to A. A is out of the loop until B hands back or hands off further.

```
User → A: turn 1 → A responds, hands off to B.
User → B: turn 2 → B responds.
User → B: turn 3 → B responds, hands off to C.
User → C: turn 4 → C responds.
```

The conversation's *current agent* is a per-session fact. Chapter 03's session-affinity record tracks it.

Representative frameworks: [OpenAI Swarm](https://github.com/openai/swarm) (deprecated in favour of the Agents SDK), [OpenAI Agents SDK](https://openai.github.io/openai-agents-python/) with its `handoffs=[...]` primitive.

### Manager-as-tools

Agent M (the manager) receives a turn. M decides it needs specialised work. M calls agents B, C, D *as tools* — one request-response per call, with M still owning the top-level loop. B, C, D produce outputs, M composes them into its reply.

```
User → M: turn 1
    M → B: subagent call, B returns.
    M → C: subagent call, C returns.
    M → composes final answer → M responds.
User → M: turn 2 (M still the top-level agent)
```

The conversation's *current agent* is M, always. Subagents are invoked; they do not take over the session.

Representative frameworks: Anthropic's orchestrator–workers pattern, [LangGraph's supervisor pattern](https://langchain-ai.github.io/langgraph/concepts/multi_agent/), [CrewAI](https://docs.crewai.com/).

## The platform primitives each one needs

The plane has to deliver both topologies. Each needs a different subset of the primitives chapters 02–05 built.

### Handoffs need

- **Durable conversation-state handoff.** When A hands off to B, the conversation's *state* has to move: the turn history, the context window, any artifact attachments, the pending task if there is one. The platform's primitive is the durable-execution store (mod-402) plus the session-affinity record (chapter 03). The handoff is a transaction against both.
- **Session-affinity rewrite.** Chapter 03's record. A handoff is an idempotent write against `session_affinity[session_id] = new_current_agent`. The entry gateway reads this on every subsequent turn.
- **Full-chain scope delegation.** Chapter 04. B receives a token whose scope covers everything B might need to do for the rest of the session — not just the first turn.
- **Hand-back and hand-forward.** B must be able to hand off further (to C) or hand back (to A, or to a supervisor). The mechanism is the same: another session-affinity rewrite.
- **A trace that stitches A's turns and B's turns under one `session_id`.** mod-408's trace has to show the handoff as a visible span (`gen_ai.operation.name=handoff`), with attributes naming the source and destination agents, so an auditor can read the chain.
- **A fallback when the handoff target is unhealthy.** A handed off to B; B is now down. The session is stuck. The plane needs either:
  - a *hand-back* primitive that returns the session to A when B fails (requires A to be running and reachable), or
  - a *supervisor* above A and B that routes around B and picks a sibling.

### Manager-as-tools need

- **Request/response A2A with streaming.** M calls B; B returns. No session-affinity change.
- **Per-subagent concurrency budgets.** Chapter 05's bulkheads, applied per upstream `agent_id`. A slow B does not starve M's calls to C.
- **Fan-out circuit-breaker policy.** M's five concurrent subagent calls are five independent breakers; one open breaker degrades M's answer but does not cascade into the others.
- **Per-subagent scope narrowing.** Chapter 04, applied per call. Each subagent gets only what *it* needs; M keeps its full scope.
- **A declared fan-out cap.** The AgentCard's `platform.fan_out_cap` (chapter 05). M cannot spawn unbounded subagent calls per turn.
- **A graceful-degradation policy.** `platform.fan_out_policy` (chapter 05): `all_required`, `best_effort`, or `quorum`. M's handler respects it.

### Shared across both

Both topologies share the plane's baseline: A2A on the wire (chapter 02), service discovery via the registry (chapter 03), identity delegation and tenant isolation (chapter 04), per-edge circuit breakers and bulkheads (chapter 05), the `A2ACallRecord` (chapter 01). The differences above are additive on top of the baseline.

## What goes wrong with each (and how the plane helps)

Both topologies have characteristic failure modes. The platform's job is to make both recoverable, not to pick a winner.

### Handoff failure modes

- **"Lost session."** A hands off to B, the handoff transaction partially commits (session-affinity updated, state not transferred), B serves the next turn but has no context. The platform's fix: the handoff is one transaction (state + affinity written atomically), or compensated as a saga (if state write fails, affinity rolls back). mod-402's durable execution is the substrate.
- **"Oscillating handoffs."** A hands to B, B immediately hands to A. If no circuit-breaker is on the handoff edge, the loop is infinite. The plane's fix: a per-session handoff budget (max N handoffs per session, max M handoffs per turn); exceeding it surfaces `failed: handoff_budget_exceeded` to the user.
- **"Stranded user."** A hands to B. B is down. Session-affinity points to B. Every subsequent turn fails. The plane's fix: a hand-back supervisor that the entry gateway consults when B is unreachable, with a recovery policy on the AgentCard (`platform.handoff_fallback`).
- **"Permissions decay."** B inherits A's scope but B's AgentCard declared fewer. The scope is narrowed at handoff. The handoff may fail *in the middle of the session* when B tries to do something it can't — from the user's perspective, the conversation stalls. The plane's fix: at handoff time, the scaffold checks the *projected* skills B will need and refuses the handoff if the narrowed scope cannot cover them.

### Manager-as-tools failure modes

- **"The manager's prompt explodes."** M aggregates subagent outputs into its context window; a verbose subagent blows the budget. The plane does not solve this directly — it is the model-gateway's context-budget feature (mod-406). The plane's role is capturing per-subagent token counts in the `A2ACallRecord` so the eval plane can diagnose.
- **"Subagent latency amplification."** Sequential subagent calls serialise; M's turn latency is the sum. The platform's fix is the fan-out primitive — concurrent subagent calls where the work is independent. The team's handler decides what is parallel.
- **"Fan-out stampede."** M fans out to five subagents each of which fans out to five more. 25 concurrent calls per manager turn across the fleet × 1000 managers = 25 000 calls in flight. The plane's fix: the AgentCard's `platform.fan_out_cap` is enforced per level, and the scaffold emits `a2a_fan_out_concurrency`; mod-410's alarm fires on sustained stampede.
- **"Subagent context asymmetry."** Each subagent sees only M's instruction, not the broader conversation. If M under-briefs a subagent, the subagent produces a plausible-but-wrong answer. This is a *design* problem — a prompt-engineering and agent-decomposition question — that the plane does not solve. It is explicitly on `senior-agentic-ai-engineer`'s side of the handoff (below).

## When to use which

The decision is contextual. Three heuristics.

- **Who should own the user's context after the next turn?**
  - If it should be the specialist, use handoff. The user is talking to the specialist; subsequent turns continue with that specialist.
  - If the manager should keep the thread (because it will aggregate the specialist's output into its own reply or because it coordinates multiple specialists), use manager-as-tools.
- **How tight is the specialisation?**
  - A domain-specific specialist with its own long-running state (a code-generation agent with session memory, a legal agent with case context) tends to benefit from handoff.
  - A stateless specialist (a classifier, a reranker, a one-shot translator) fits manager-as-tools.
- **What is the recovery story if the specialist fails?**
  - Handoff puts a full session at stake; failure needs a hand-back path. If there is no supervisor to hand back to, think again.
  - Manager-as-tools puts one subagent call at stake; a bulkhead contains the failure to one call and the manager's `best_effort` policy composes a degraded answer.

A combined pattern is common: a *router agent* at the top handles handoffs between a small number of top-level specialists (sales, support, admin); each specialist is a *manager* that fans out to its own subagents (manager-as-tools). The platform handles both edges identically.

## The handoff handshake, concretely

Because handoffs are the less familiar primitive in service-mesh terms, a worked handshake:

```python
# writer-v1 handler decides the request is better handled by researcher-v2
from platform_a2a import handoff

async def handle(request, ctx):
    if is_pure_research(request):
        await handoff(
            to="researcher-v2",
            context=ctx.session_context,
            reason="user_asked_for_research_only",
        )
        # The handoff suspends this handler. The session is now owned by researcher-v2.
        return
    ...
```

Under the hood, `handoff(...)`:

1. **Verifies the target agent exists on the registry** and declares `handoff` in `platform.topology_modes`.
2. **Checks the projected scope.** Writer-v1's current token scope narrowed against researcher-v2's `declared_scopes` and the handoff policy; refuses if the resulting scope is empty or missing a projected-necessary scope.
3. **Writes the session state to the durable store.** A2A conversation history, pending artifacts, open task IDs. Keyed by `session_id`.
4. **Atomically rewrites session-affinity.** `session_affinity[session_id] = researcher-v2`. If any step up to here fails, the handoff aborts and the handler continues.
5. **Emits a `handoff` span.** `gen_ai.operation.name=handoff`, `platform.handoff_from=writer-v1`, `platform.handoff_to=researcher-v2`, `platform.handoff_reason=user_asked_for_research_only`, `platform.handoff_budget_remaining=4`.
6. **Returns success to the handler.** The handler returns its last response to the user (typically "I'm handing you to a researcher for this question."). The next user turn routes to researcher-v2 via the entry gateway's session-affinity lookup.

On the next turn, the entry gateway:

1. Reads `session_affinity[session_id]` → `researcher-v2`.
2. Narrows the user's edge token for researcher-v2 (chapter 04).
3. Fetches session state (A2A conversation history + any state writer-v1 attached).
4. Calls researcher-v2's `tasks/send` with the user's new message and the recovered state.

The same primitives apply to hand-back and hand-forward. Every handoff is the same transaction against (durable state store, session-affinity record).

## The limits of the plane

The plane cannot decide:

- **What tasks an agent should take on vs. delegate.** That is prompt engineering and decomposition.
- **Which subagents a manager should fan out to for a given input.** That is routing logic inside the agent.
- **Whether an agent's output is good.** That is evaluation (mod-407).
- **Whether the overall multi-agent design is sensible for the product.** That is systems architecture.

These are the handoff lines to the two roles this module hands off to.

## Handoff to `agentic-systems-architect` (peer, level 48)

Multi-agent *design* is the architect's domain. The platform engineer's job is to give the architect primitives that express their designs without hidden costs; the architect's job is to pick designs the platform can serve well. Four concrete handoffs.

### 1. The topology choice

The plane engineer owns whether handoff and manager-as-tools are *supported*. The architect owns which one a given multi-agent system *uses*. The platform engineer's contract to the architect:

- Both topologies are available with parity in observability, identity, resilience, and governance.
- The architect can combine them (router-at-top, managers-below).
- Platform metrics distinguish the two in the `A2ACallRecord`'s `topology` field; the architect can observe how their design performs.

The architect's contract back:

- When the design is written down, the topology for each edge is explicit.
- The design accounts for the failure modes above (lost session, oscillating handoffs, fan-out stampede).
- Cross-tenant or cross-organisation delegation is called out explicitly in the design document; the registry's `cross_tenant_allow` is a reviewable artefact.

### 2. The AgentCard review

An AgentCard is simultaneously a public API, a resilience posture, and a security posture. New AgentCards (or substantive changes to existing ones) go through a joint review:

- The platform engineer reviews the scope declarations, the tenancy, the resilience tier, and the fan-out policy against the platform's defaults.
- The architect reviews the skills, the topology modes, the fan-out shape, and the potential-caller surface against the system design.

The review's output is the AgentCard. Shipped PR; versioned registry entry.

### 3. The dependency-graph review

New agent → agent edges change the dependency graph. The registry's directed-edge list (chapter 05) grows; new cycles or fan-out shapes become possible. The architect owns the graph's *semantic* review ("should this agent be able to call that one?"); the platform engineer owns the *operational* review ("will this edge survive a degradation on the leaf?"). A joint sign-off before the edge is activated on the registry.

### 4. The escalation path

When a product team wants a pattern the plane does not support (long-running "always on" background agents that talk to each other on a cadence, cross-organisation agent federations, agent "teams" with shared state), the architect is the entry point to decide whether the pattern is worth adding to the plane. The platform engineer's role is to estimate the platform cost; the architect's role is to weigh it against the product need.

## Handoff to `senior-agentic-ai-engineer` (peer, level 40)

Product-side subsystem implementation — the agent's own prompts, tools, loop logic, evaluation harness, and framework choice — is `senior-agentic-ai-engineer`'s domain. The platform engineer's contract is to make the paved road cheap enough that the product engineer never reaches for a bespoke framework-level integration. Four concrete handoffs.

### 1. The paved-road scaffolds

The product engineer imports the A2A client scaffold and the A2A server scaffold (chapter 02). They do *not* re-implement token exchange, mTLS, bulkheads, or circuit breakers. The platform engineer's contract:

- The scaffolds work with the common frameworks the product engineers are using (LangGraph, LlamaIndex, CrewAI, OpenAI Agents SDK, custom). The scaffold is framework-agnostic at the wire; adapters bridge framework handoff primitives to A2A.
- SDK upgrades are non-breaking across minor versions. A platform-level propagation fix lands as a patch bump; teams run `uv lock --upgrade` and are correct.
- The scaffold emits the `A2ACallRecord`; the product engineer never needs to log an inter-agent call manually.

### 2. The AgentCard as a contract

The product engineer writes the agent's handler; the platform engineer and architect review the AgentCard. The handler's behaviour must match the card:

- The handler must not accept skills not declared on the card. Scaffold enforces.
- The handler must not require scopes not declared on the card. Scaffold enforces.
- The handler must honour `platform.fan_out_cap` and `platform.fan_out_policy` when manager-as-tools. Scaffold enforces.
- The handler must not initiate handoffs to agents not listed in `platform.may_handoff_to`. Scaffold enforces.

### 3. The conformance test suite

Before the product engineer's agent lands on the registry, the platform's conformance test (chapter 03, plus the full mod-408 chapter 05 suite) runs against it. The product engineer's contract:

- The agent passes conformance. Red fails CI.
- Failures surface as actionable errors, not as cryptic mesh YAML.
- The platform engineer is the escalation when a conformance rule looks wrong for the agent; the review decides whether the rule is wrong or the agent is.

### 4. The on-call interface

When an agent degrades in production, the SRE (mod-410) rotation is the first responder. Serious or persistent incidents escalate to the owning `senior-agentic-ai-engineer`. The platform engineer's contract:

- SRE has the dashboards (mod-408, mod-410) to answer "which agent, in which chain, at which hop." The product engineer does not need to re-derive this.
- The platform provides a replay primitive (mod-407's trace-and-replay store) that reproduces the failing turn against the agent's current version. The product engineer uses it to repro.

The platform does not do the product engineer's debugging. It gives them a clean surface to debug on.

## The explicit non-handoffs

Three things that look like handoffs but are not.

- **Multi-agent *framework* selection.** Not an architect decision, not a product engineer decision — it is the product engineer's call, informed by the architect, with the platform engineer only intervening if the framework cannot be wrapped by the paved-road scaffold. The plane is framework-neutral by construction.
- **Prompt engineering for the individual agent.** The platform engineer never writes the handler's prompts. The product engineer owns them; the eval engineer (mod-407's handoffs) owns whether they are good.
- **Agent-level model selection.** Which model backs a given agent is the model gateway's routing policy (mod-406). The architect may advise on cost/latency tradeoffs; the platform engineer's role is "make sure the model gateway can serve the choice."

## Where this module hands off to the rest of the track

A map for the reader.

- **To mod-410 (Reliability Engineering and SRE for Agents).** The resilience signals from chapter 05 feed mod-410's dashboards and alerting. Breaker-state gauges, retry rates, deadline-exceeded counts, bulkhead saturation — the plane emits; SRE consumes.
- **To mod-411 (Safety Plane and Guardrails as a Service).** Guardrail verdicts on inter-agent handoffs (a handoff that would send a user to a less-safe agent) ride the same event stream as single-agent guardrails. The plane's job is the hook; mod-411 is the policy.
- **To mod-412 (Paved-Road SDKs and Developer Experience).** The A2A client and server scaffolds *are* the paved-road SDK for inter-agent calls. mod-412 is where they land in the developer's hands and the onboarding story is written.
- **To mod-413 (Platform Leadership and Operating Cadence).** The governance cadence reviews the registry's `cross_tenant_allow`, the fleet-wide fan-out shape, and the audit stream from chapter 04. The architect's AgentCard reviews feed into the quarterly governance committee.
- **Back to mod-408 (Observability and OpenTelemetry for Agent Fleets).** Every chapter in this module assumes `traceparent` works and the `gen_ai.*` semconv is pinned. The A2A spans, the handoff spans, the fan-out spans — all compliant with mod-408's vocabulary.

## Summary

- Two inter-agent topology modes — handoffs and manager-as-tools — are first-class on the plane. Each needs different primitives; the plane delivers both.
- Handoffs need: durable state transfer, session-affinity rewrite, full-chain scope delegation, hand-back / hand-forward support, a handoff-visible trace, and a fallback when the target is unhealthy.
- Manager-as-tools need: request/response A2A with streaming, per-subagent concurrency budgets, fan-out circuit-breaker policy, per-subagent scope narrowing, declared fan-out cap, graceful degradation policy.
- Characteristic failure modes: lost session, oscillating handoffs, stranded user, permissions decay (handoffs); prompt explosion, latency amplification, fan-out stampede, subagent context asymmetry (manager-as-tools). The plane catches the operational ones; the design and eval roles catch the semantic ones.
- Topology choice is contextual: who should own the user's context, how tight the specialisation is, what the recovery story is when the specialist fails. A combined pattern (router handoff at top, managers below) is common.
- The handoff handshake is a single durable transaction against (state store, session-affinity record), with explicit scope-projection check. Every handoff emits a handoff span; every rewrite is auditable.
- The plane hands off to `agentic-systems-architect` for topology choice, AgentCard review, dependency-graph review, and pattern escalation; to `senior-agentic-ai-engineer` for paved-road scaffolds, AgentCard conformance, pre-registry conformance tests, and the on-call interface.
- The explicit non-handoffs: framework selection, prompt engineering, agent-level model selection. The plane stays out of these by construction.

This closes mod-409. The reader now has: the A2A wire protocol and its mesh mapping, service discovery and correlation propagation, identity delegation and permission narrowing, circuit breakers and bulkheads for agent graphs, and the two topology modes with a clean role-handoff story. The next module (mod-410) picks up the SRE operating cadence on top of these signals.
