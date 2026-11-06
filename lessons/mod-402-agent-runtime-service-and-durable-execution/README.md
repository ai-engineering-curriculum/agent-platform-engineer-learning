# mod-402 — Agent Runtime Service and Durable Execution

**Estimated effort:** 16 hours · **Position in track:** second module (after [`mod-401`](../mod-401-agent-platform-position-and-paved-road-philosophy)).

This module builds the first platform-engineering slice the rest of the track assumes: the multi-tenant agent-runtime plane. By the end, you can defend a substrate choice (Temporal / Dapr Workflows / Restate / Inngest) against a documented workload, project five mainstream agent frameworks (LangGraph, OpenAI Responses, Anthropic Agent SDK, Google ADK, Microsoft Agent Framework) onto a single platform-owned session-state abstraction, and reason precisely about where each class of state lives (Postgres + pgvector, Redis, object storage) and how workers are shaped (stateless, sticky, serverless).

## Learning objectives

- Architect a multi-tenant agent-runtime service — session lifecycle, context compaction, checkpoint / resume, timer and signal handling, cancellation semantics, cost-aware concurrency limits, stateless vs. stateful worker topologies.
- Compare durable-execution substrates (Temporal, Dapr Workflows, Restate, Inngest) for agent workloads and defend a substrate choice against a documented set of workload characteristics.
- Integrate LangGraph checkpoints, OpenAI Responses state, Anthropic Agent SDK session state, Google ADK agent engine, and Microsoft Agent Framework runtime into a unified session-state abstraction the platform exposes.
- Reason about state substrates — Postgres (pgvector for memory indices), Redis (short-term memory / rate-limit state), object storage (large artifacts) — and when each belongs on a Kubernetes operator vs. a serverless fabric like Knative.
- Hand off to `senior-agentic-ai-engineer` (level 40) for product-side subsystem implementation and to `agentic-systems-architect` (level 48) for cross-product architecture.

## Chapters

1. [`01-the-runtime-plane-and-session-as-contract.md`](01-the-runtime-plane-and-session-as-contract.md) — The slice the runtime plane owns, the session as the first-class object, the three non-negotiable obligations (at-most-once effective tool calls, bounded cancellation, live cost ledger), and the ten peer planes it talks to.
2. [`02-session-lifecycle-checkpoint-resume-cancellation.md`](02-session-lifecycle-checkpoint-resume-cancellation.md) — The seven-state session machine, checkpoint timing and placement, resume semantics under failure, durable timers and signals, cancellation that is propagating / bounded / observable, and four context-compaction policies.
3. [`03-durable-execution-substrate-choice.md`](03-durable-execution-substrate-choice.md) — The five primitives every durable-execution substrate ships, side-by-side comparison of Temporal, Dapr Workflows, Restate, Inngest against agent-workload characteristics, and the decision pattern you apply in exercise 02.
4. [`04-framework-state-into-unified-abstraction.md`](04-framework-state-into-unified-abstraction.md) — A platform-owned `SessionState` interface, the versioned-envelope storage shape, and the adapter mapping for LangGraph, OpenAI Responses, Anthropic Agent SDK, Google ADK, and Microsoft Agent Framework.
5. [`05-state-substrates-and-worker-topologies.md`](05-state-substrates-and-worker-topologies.md) — Postgres (system of record, pgvector for memory), Redis (cache / counters), object storage (large blobs); Kubernetes operators vs. Knative-style serverless per substrate; stateless vs. stateful workers; cost-aware concurrency with per-session budget, per-tenant concurrency, fleet back-pressure, and priority tiers.

## Exercises

Each exercise turns one or more chapters into a durable artefact you keep and reuse in later modules.

- [`exercises/exercise-01-session-state-abstraction-contract.md`](exercises/exercise-01-session-state-abstraction-contract.md) — Session state abstraction contract. Pairs with chapters 01, 02, 04.
- [`exercises/exercise-02-durable-execution-substrate-bakeoff.md`](exercises/exercise-02-durable-execution-substrate-bakeoff.md) — Durable-execution substrate bakeoff. Pairs with chapter 03.
- [`exercises/exercise-03-checkpoint-and-resume-under-partial-failure.md`](exercises/exercise-03-checkpoint-and-resume-under-partial-failure.md) — Checkpoint and resume under partial failure. Pairs with chapter 02 and feeds mod-410.
- [`exercises/exercise-04-cost-aware-concurrency-and-admission-control.md`](exercises/exercise-04-cost-aware-concurrency-and-admission-control.md) — Cost-aware concurrency and admission control. Pairs with chapter 05 and feeds mod-406.

## Resources

External references are catalogued in [`resources.md`](resources.md).

## How to approach this module

1. Read the five chapters in order. The substrate chapter (03) only makes sense after you have the contract (01) and the lifecycle (02); the framework abstraction (04) only lands after the substrate; the state-substrate chapter (05) ties them all to what you actually operate.
2. Do the exercises in order. Each one produces an artefact you reuse later — the session-state contract from exercise 01 is the input to every later module that touches a session; the substrate-bakeoff doc from exercise 02 is the input to mod-410's SRE work; the admission-control rules from exercise 04 feed directly into mod-406's model gateway.
3. Flag `<!-- needs-research: ... -->` in any artefact you produce if you cannot verify a specific framework version or substrate feature. The flag is cheaper than silently guessing — the next module reads your artefacts and will notice the gap.

## Where the handoffs land

- `senior-agentic-ai-engineer` (level 40) — reads your session-state abstraction (exercise 01) and builds product-side agents on top of it. If their team wants substrate features your plane does not expose, that is either a platform roadmap item or an explicit opt-out contract (mod-401, chapter 04).
- `agentic-systems-architect` (level 48) — reads your substrate-bakeoff (exercise 02) and the resulting constraints into cross-product architecture decisions. Bring your implementation realities to their design work.
- `ai-infra-sre` (peer) and your own future mod-410 work — reads your admission-control model (exercise 04) and your checkpoint / resume design (exercise 03) as inputs to SLO definition and incident runbooks.
