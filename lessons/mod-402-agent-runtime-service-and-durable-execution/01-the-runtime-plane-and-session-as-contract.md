# The Runtime Plane and the Session as a Contract

The first piece of agent-platform substrate every product team consumes is the runtime. If the runtime wobbles — if a session loses its history after a pod restart, if cancellation leaks tool calls after the user hangs up, if a slow tool blocks every other session on a worker — product teams route around it, re-implement it, and your platform stops being a platform. So before any substrate choice (Temporal vs. Dapr, Postgres vs. Redis, stateless vs. stateful workers), the runtime plane has to be defined by *what it guarantees*, not by what it is built on.

This chapter fixes that frame. The rest of the module picks substrates against it.

## The slice this plane owns

The agent-runtime plane sits between the paved-road SDK (mod-412) and every other platform plane. It is where a *session* — one logical agent interaction with a user or upstream caller — lives from the first inbound event to the last tool-call acknowledgement. Concretely, the runtime plane owns:

- **Session lifecycle.** Create, resume, suspend, cancel, complete, expire. Every transition is observable and persistent.
- **Durable progress.** If a worker dies mid-tool-call, the next worker continues from the last committed checkpoint — not from scratch, not from a stale copy that re-executes the LLM call and bills the tenant twice.
- **Context management.** The context window (messages, tool results, retrieved memory) is bounded, and the runtime decides *when* and *how* it gets compacted — not the agent code.
- **Timers and signals.** The runtime schedules future work ("remind me in 10 minutes"), accepts external events ("user cancelled"), and fires both deterministically even if the fleet scales down to zero in between.
- **Cost-aware admission.** The runtime decides which sessions run now, which queue, and which shed load, based on per-tenant quota and live cost telemetry.

The runtime plane does **not** own: tool invocation policy (that is mod-405), model choice and routing (mod-406), prompt or judge content (product teams and `ai-eval-engineer`), eval gating (mod-407), or guardrail logic (mod-411). It calls those planes on behalf of the agent; it does not author them.

## The session as the first-class object

The simplest useful shape of a session has five fields:

```
Session {
  id: string                          # stable across checkpoint / resume
  tenant: string                      # drives quota and policy routing
  state: enum                         # running | waiting_signal | waiting_tool | suspended | cancelled | completed | expired
  checkpoint: opaque bytes            # last committed snapshot of agent state
  pending_timers: list<Timer>         # fire times + target labels
  pending_signals: list<SignalName>   # names the runtime accepts right now
  cost_ledger: CostLedger             # tokens, tool-$, model-$ attributed so far
}
```

Three properties make this object load-bearing:

1. **Resumable from `checkpoint` alone.** A worker that picks up a session after a restart should need nothing else. If resuming also requires reading log files or external cache, your state abstraction is leaking — see chapter 04.
2. **Idempotent transitions.** Every state change (snapshot committed, timer fired, signal delivered) is retryable. Workers crash; the substrate retries; nothing double-charges.
3. **Observable from one query.** An on-call engineer at 03:00 can answer "what is this session doing right now" with a single lookup against the session object. If they have to join against three systems, your observability plane will fail its first incident.

Everything else in the runtime plane — timers, cancellation, concurrency, substrate choice — is a decision about *how* to deliver those three properties.

## The three obligations

Pin the runtime plane's contract to three user-facing obligations. These are what product teams quote back to you when they complain, so you may as well write them down first.

1. **"If my session committed a tool call, it will not run twice."** At-most-once effective side-effects, even if workers die in the middle of the tool call. This is the obligation that forces a durable-execution substrate rather than a bare message queue.
2. **"If my user hangs up, tool calls stop."** Cancellation propagates to in-flight tool invocations within a bounded time, and the runtime stops burning tokens, tool-$, and sandbox time. This is the obligation that forces explicit cancellation semantics instead of relying on request timeouts.
3. **"My session's cost cannot exceed its budget without telling me."** Per-session (and per-tenant) cost is live telemetry, not an after-the-fact report. The runtime can shed load against a budget. This is the obligation that forces the cost ledger onto the session object rather than into a separate billing pipeline.

None of these three are optional. A runtime that cannot defend all three in production is not ready to be a platform.

## Where this plane sits in the platform

The runtime plane is the **control plane for one session**. Other planes are the control planes for other things:

| Plane | Controls | Session-level contract to the runtime |
|-------|----------|---------------------------------------|
| MCP registry + tool gateway (mod-403) | Available tools, auth, schema | "Give me a tool client I can call; tell me when it changes." |
| Sandboxed code execution (mod-404) | Short-lived compute for arbitrary tool code | "Run this code with these capabilities, return bytes." |
| Tool / permission policy (mod-405) | Which tools this session may call | "Can this session call tool X with args Y right now? (allow / deny / step-up)" |
| Model gateway (mod-406) | Model routing, quotas, cost attribution | "Send this prompt; charge it to tenant T; retry per my policy, not yours." |
| Eval platform (mod-407) | Traces, replay, regression gates | "Here is a full trace of what I did; here is the input bundle for later replay." |
| Observability (mod-408) | OTel traces, metrics, logs, semantic conventions | "Here is the GenAI-conv-spec span tree for this session." |
| A2A routing (mod-409) | Inter-agent messaging | "Deliver this message to agent Y; let me wait on their reply as a signal." |
| SRE / reliability (mod-410) | SLOs, error budgets, incident response | "These are the session-level SLIs; here is the runbook on breach." |
| Safety plane (mod-411) | Input / output filters, policy gates | "Check this content; return allow / redact / block before I continue." |
| Paved-road SDK (mod-412) | How product teams write agents | "This is the agent-authoring contract; the runtime obeys it." |

Every neighbour has a bounded conversation with the runtime, mediated by the session object. If a neighbour starts holding state the runtime should be holding, the session loses its single-lookup observability. If the runtime starts holding state a neighbour should hold (routing policy, judge rubrics, model choices), the runtime becomes a product and the slice erodes.

## The three handoffs this track makes you responsible for

Later chapters pick substrates and integrate frameworks. Keep three handoffs visible as you do:

- **To `senior-agentic-ai-engineer` (level 40).** They implement product-side agents on top of the runtime. If a product needs an unusual session shape — human-in-the-loop with 24-hour waits, multi-turn negotiation between agents — the runtime either supports it generically or the product gets an opt-out contract (mod-401, chapter 04), *not* a bespoke fork of the runtime.
- **To `agentic-systems-architect` (level 48).** They choose the cross-product orchestration pattern (single-agent with tools, routing-pattern, orchestrator-workers). Your job is to make the pattern they pick executable at platform scale. Bring back implementation constraints — cost ceilings, substrate limits, reliability floors — as inputs to their design, not as polite objections after the fact.
- **To `ai-infra-sre` (peer) and your own future SRE work in mod-410.** The runtime plane's SLOs are *theirs* to consume once declared. Pick SLIs now (session-start latency, resume success rate, cancellation propagation time, cost-ledger lag) even if you only start measuring them properly in mod-410.

## Summary

- The agent-runtime plane owns the session — lifecycle, durable progress, context management, timers / signals, cost-aware admission. Everything else is a neighbour.
- Three obligations are non-optional: at-most-once effective tool calls, bounded cancellation propagation, live cost ledger.
- The session object has five load-bearing fields (id, tenant, state, checkpoint, pending timers / signals, cost ledger) and three properties (resumable from checkpoint alone, idempotent transitions, single-lookup observability).
- The rest of this module is substrate choices against this contract. If a substrate makes an obligation harder to deliver, it loses — regardless of its vendor profile.

The next chapter zooms into the first obligation — durable progress — and the session-lifecycle mechanics (checkpoint, resume, timers, signals, cancellation, context compaction) that make it enforceable.
