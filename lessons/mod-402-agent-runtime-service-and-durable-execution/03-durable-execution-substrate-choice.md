# Durable Execution Substrates: Temporal, Dapr Workflows, Restate, Inngest

The runtime obligations in chapter 01 — at-most-once effective tool calls, bounded cancellation, durable timers and signals — can all be built from scratch on top of Postgres and a worker pool. Most agent platforms should not. Four production-grade durable-execution substrates exist in 2026, each with a different philosophy on determinism, deployment, and developer experience. This chapter is the comparison and the defence — not a tutorial on any one of them.

The output of this chapter, applied to your workload, is a one-page substrate decision that holds up under pushback from an architect (`agentic-systems-architect`, level 48) or an SRE lead (`ai-infra-sre`). Exercise 02 is where you write that page.

## What a durable-execution substrate actually gives you

All four substrates give you roughly the same five primitives, under different names. Before comparing them, name the primitives, because the comparison is about trade-offs on these — not about marketing surface.

1. **Durable functions with replay-based recovery.** Your "workflow" or "service" code is executed deterministically against an append-only event history. On crash, the code is re-played from the history until it reaches the last durable point, and execution resumes. This is the primitive that makes exactly-once *effective* side-effects possible.
2. **Durable timers.** `sleep(duration)` and `sleepUntil(wall_clock_time)` that survive worker restart and fleet scale-down.
3. **External signals.** API-addressable events delivered to a specific running or suspended workflow instance.
4. **Side-effect capture.** A wrapper (`activity`, `side_effect`, `run`, `step`) that lets your code call non-deterministic APIs (HTTP, LLMs, tools) — the substrate records the result in the event history so replay does not re-issue the call.
5. **Child workflows / nested invocations.** Spawn a sub-workflow (sub-agent, A2A conversation) and await its result with the same durability guarantees.

If you remember only one thing: **the primitive is replay-based recovery around an event history, and the activity / side-effect wrapper is how you keep non-determinism out of the replayable code.** Everything else — the clustering story, the SDK ergonomics, the operator model — is implementation choice.

## The four substrates, side by side

The sources cited here are the official docs (see `resources.md`). Feature claims that cannot be grounded in those docs are marked `<!-- needs-research -->` and you should confirm against the current version before quoting them in an architecture review.

### Temporal

- **Shape.** Self-hostable cluster (Temporal Server) or managed (Temporal Cloud). SDKs in Go, Java, TypeScript, Python, PHP, .NET. See <https://docs.temporal.io/>.
- **Determinism model.** Workflow code is strictly deterministic and runs on sticky workers; activities (your tool calls, LLM calls) run anywhere and their results are recorded.
- **Strengths for agent workloads.** Mature, battle-tested, strong query / signal / timer surface, well-defined cancellation semantics, large ecosystem, good observability hooks.
- **Costs.** Operating a Temporal cluster (frontend, matching, history, worker, visibility stores) is a non-trivial platform-engineering commitment. Managed cloud removes this cost but introduces a vendor dependency. Determinism constraints on workflow code need SDK discipline from product teams (mod-412).

### Dapr Workflows

- **Shape.** A Dapr building block that sits on top of Dapr sidecars, backed by a pluggable state store (Redis, Postgres, Azure Cosmos DB, etc.). See <https://docs.dapr.io/developing-applications/building-blocks/workflow/workflow-overview/>. Dapr Agents <https://docs.dapr.io/concepts/agents/> is the agent-specific overlay.
- **Determinism model.** Workflow functions are deterministic; activities wrap side effects. Comparable to Temporal's shape; implemented on top of the Durable Task Framework lineage.
- **Strengths for agent workloads.** Especially strong if the rest of your platform already runs on Dapr (service mesh, pub/sub, state stores). Dapr Agents is explicitly positioned for agent workflows. Pluggable state store is an advantage for teams that want Postgres as the system of record.
- **Costs.** Dapr sidecars add operational surface area. Dapr Workflows is newer than Temporal and the agent-focused docs are still evolving — worth a direct check against the current spec before committing.

### Restate

- **Shape.** A single-binary distributed runtime with a journal-per-invocation log. SDKs in TypeScript, Java, Kotlin, Python, Go, Rust. See <https://docs.restate.dev/>.
- **Determinism model.** Durable handlers (functions) with the same replay-based recovery model; "virtual objects" give you a keyed actor-ish primitive (stateful, single-writer per key) that fits per-session state well.
- **Strengths for agent workloads.** Simpler cluster story than Temporal (one binary); virtual objects map cleanly onto the "session is the first-class object" framing from chapter 01; strong HTTP-first ergonomics. Journal-based model gives fine-grained observability out of the box.
- **Costs.** Smaller ecosystem than Temporal, newer; some integrations your team expects (vendor connectors, enterprise SSO) may not exist yet. Worth checking against the roadmap.

### Inngest

- **Shape.** Developer-platform-style service with a cloud-hosted control plane and self-hosted workers; SDKs in TypeScript, Python, Go. Built around durable "functions" / "steps". See <https://www.inngest.com/docs>.
- **Determinism model.** Steps are the side-effect capture primitive; function code is deterministic across the step boundaries.
- **Strengths for agent workloads.** Lowest onboarding cost — write a function, import the SDK, done. Strong concurrency / rate-limiting / throttling primitives on the per-function / per-key dimension, which maps nicely onto per-tenant admission control (chapter 05).
- **Costs.** The cloud-hosted control plane is the common deployment; self-hosted options exist but check the current support matrix before committing a regulated workload to self-host. SDK breadth is narrower than Temporal.

## The workload characteristics that actually decide

A substrate bakeoff that compares feature lists is noise. A bakeoff that compares *your* workload characteristics against the four substrates' constraints is a decision. Here is the characteristic list to run your workload through:

| Characteristic | Why it matters for the substrate choice |
|---|---|
| **Session duration distribution.** Are 99th-percentile sessions seconds, minutes, hours, or days? | A session that spans a week needs cheap, durable sleep. Temporal and Dapr Workflows handle multi-day sleeps well. For shorter sessions, Inngest's cloud model's cost profile matters more. |
| **Fan-out shape.** How many child tool calls / sub-agents per session? | High fan-out per session stresses the event-history / journal substrate. Restate's per-invocation journal and Temporal's history-size limits both have practical ceilings (confirm against docs). |
| **Signal / A2A rate.** Are signals rare (human replies) or dense (sub-agent chatter)? | Signal throughput per workflow instance has engine-specific ceilings; dense A2A traffic may want a different pattern (A2A broker, mod-409) in front of the runtime. |
| **Cancellation deadline.** Chapter 02's obligation 2 target. | Substrates differ in how quickly cancellation propagates into running activities. Verify against docs and your own measurements in exercise 02. |
| **Determinism budget on product teams.** Can you require product teams to write deterministic workflow code? | Temporal, Dapr Workflows, Restate all require it. Inngest's step abstraction hides more of it. Determinism is the main friction for product teams; mod-412's SDK layer can smooth it, but it never disappears. |
| **Deployment ergonomics.** Self-host vs. managed. | Managed (Temporal Cloud, Inngest Cloud) is faster to onboard but introduces a vendor dependency for a system-of-record. Self-host (Dapr, Restate, Temporal OSS) is more operational work. |
| **Existing platform surface.** What does the rest of the organisation already run? | A Dapr-everywhere org should default-prefer Dapr Workflows. A JVM-heavy org gets more mileage from Temporal's Java SDK. Boring consistency with the rest of the platform is undervalued. |

## A worked micro-example: "can I replay this safely?"

Across all four substrates, the mental model you need to internalise is the same. Agent step N looks like:

```python
# Pseudocode. Real SDK names differ per substrate.

async def agent_turn(session):
    user_msg = session.last_user_message()

    # Non-deterministic — MUST be an activity / step / side_effect.
    plan = await step("plan", lambda: call_llm("planner", user_msg))

    # Deterministic — fine in workflow code.
    tool_name, tool_args = parse_tool_call(plan)

    # Side-effecting — MUST be an activity / step / side_effect.
    tool_result = await step(
        f"tool-{tool_name}",
        lambda: call_tool(tool_name, tool_args, session.tenant),
    )

    # Non-deterministic (model) — activity.
    reply = await step("compose", lambda: call_llm("composer", tool_result))

    # Deterministic persistence back onto the session.
    session.append(reply)
```

Three rules make this correct across every substrate:

1. **Every LLM call, every tool call, every HTTP call is inside a `step` / `activity`.** The substrate records the result; replay reads the result, does not re-execute.
2. **No `random()`, no `time.now()`, no `uuid4()` in the workflow body.** Use substrate-provided deterministic versions, or inject the value from a `step`.
3. **Keep the workflow body small.** Long, branchy workflow code grows the replay cost; push work into activities. Replays run on every worker restart — this matters.

If product teams follow those three rules in agent code (the SDK should make it hard to break them — mod-412), the substrate is doing its job no matter which of the four you picked.

## The decision pattern

For exercise 02, apply this pattern end-to-end:

1. **Write down the workload characteristics** — concrete numbers (session duration percentiles, fan-out, signal rate, cancellation deadline, retention) for the agents you expect to host in the first six months. Do not skip the numbers. A bakeoff with no numbers decays into opinion.
2. **Score each substrate on those characteristics.** Not a pass/fail — a 1–5 fit score with a one-sentence justification per cell.
3. **Add deployment and ecosystem constraints.** Managed vs. self-host, SDK language fit, consistency with the rest of the platform.
4. **Pick one — and write the failure modes.** No substrate is a universal fit. The decision doc should include "we picked X; here is where we expect it to hurt; here is how we plan to mitigate." An architect who cannot articulate the pain is not ready to commit.
5. **Define the migration escape hatch.** If the workload shape drifts by 10× in any dimension, which substrate do you migrate to and how expensive is it? This is not throwaway thinking — chapter 04's unified session-state abstraction is partly designed to make this cheaper.

## Non-substrate choice: rolling your own

Writing a bespoke durable-execution engine on top of Postgres is a trap this track names explicitly. It is tractable for a quarter, drains the team for a year, and ships late. The three obligations in chapter 01 are harder than they look at scale, and every substrate above encodes multi-year hard-won lessons (event-history compaction, worker stickiness, visibility stores, idempotency tokens, replay cache, upgrade semantics).

Legitimate reasons to roll your own are narrow:

- You have a strict operational constraint none of the four satisfies (air-gapped, FedRAMP-high, on specific hardware).
- Your workload is truly tiny and lives inside one process; a thin Postgres-backed state machine is enough and you have no plan to scale it.
- You are *augmenting* a chosen substrate with a thin layer, not replacing it.

Otherwise: pick one of the four, write down the escape hatch, move on.

## Summary

- Durable-execution substrates give five primitives: replay-based durable functions, durable timers, external signals, side-effect capture, and nested invocations. All four of Temporal, Dapr Workflows, Restate, and Inngest provide them with different trade-offs.
- Compare them against *your* workload characteristics — session duration, fan-out, signal rate, cancellation deadline, determinism budget on product teams, deployment ergonomics, and existing platform surface — not feature grids.
- Agent code's correctness across substrates reduces to three rules: every side effect in an activity / step; no non-determinism in workflow bodies; keep the workflow body small.
- Rolling your own durable-execution engine is almost never the right call. When it is, say so explicitly and name the constraint.

Next chapter: how five mainstream agent frameworks (LangGraph, OpenAI Responses, Anthropic Agent SDK, Google ADK, Microsoft Agent Framework) each model session state, and how to collapse them under one platform-owned abstraction so the substrate choice above does not couple to any single framework.
