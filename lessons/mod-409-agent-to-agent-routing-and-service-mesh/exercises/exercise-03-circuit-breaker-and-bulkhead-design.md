# exercise-03: Circuit Breaker and Bulkhead Design

**Estimated effort:** 4 hours · **Output:** a four-agent dependency graph with per-edge circuit breakers and bulkheads, a captured cascade under naive config, a captured graceful degradation under hardened config, a comparison report, a one-page design doc on where resilience lives in the platform, and a short README (~1 page) · **Pairs with:** chapter [05](../05-resilience-circuit-breakers-and-bulkheads.md); feeds project 401 / 402 and mod-410.

## Objective

Model a realistic agent dependency graph, run it under a leaf-agent degradation twice — once with no resilience, once with circuit breakers + bulkheads + deadlines + a fan-out policy — and measure the difference. Then write a short platform design doc defending where each resilience primitive lives (mesh vs. scaffold vs. handler) so the next platform engineer knows how to extend it.

Chapter 05 is the theory; this exercise is the muscle memory. A platform engineer who has not personally felt the shape of a cascade reaches for the wrong primitive in the first production incident.

## Prerequisites

- You have completed [exercise 01](exercise-01-a2a-spec-conformance-walkthrough.md) and [exercise 02](exercise-02-identity-and-permission-propagation.md). The two-agent delegated-identity topology from exercise 02 is the starting point.
- You have read chapters [01](../01-the-a2a-plane-and-what-it-owns.md), [03](../03-inter-agent-routing-and-service-discovery.md), and especially [05](../05-resilience-circuit-breakers-and-bulkheads.md).
- You have an HTTP load-generation tool — [`vegeta`](https://github.com/tsenart/vegeta), [`hey`](https://github.com/rakyll/hey), [`k6`](https://k6.io/), or a 30-LOC Python asyncio script. The tool must support a sustained request rate and report p50 / p95 / p99 latencies.
- **Optional but encouraged:** a local Kubernetes cluster ([kind](https://kind.sigs.k8s.io/), [k3d](https://k3d.io/), or [minikube](https://minikube.sigs.k8s.io/)) with [Istio](https://istio.io/latest/docs/setup/getting-started/) installed, so you can configure real `DestinationRule` CRDs. If this is too heavy, implement the circuit breaker and bulkhead *in the application* using a library ([Resilience4j](https://resilience4j.readme.io/) for JVM, [`pybreaker`](https://github.com/danielfm/pybreaker) + a semaphore for Python, [`opossum`](https://github.com/nodeshift/opossum) for Node, [`gobreaker`](https://github.com/sony/gobreaker) for Go). The exercise's lesson works either way; the choice affects what artefacts you commit.

## Problem statement

Model a four-agent dependency graph:

```
       ┌───────────┐
       │ writer-v1 │   (manager-as-tools)
       └─────┬─────┘
             │
     ┌───────┼───────┐
     ▼       ▼       ▼
┌────────┐ ┌────────┐ ┌────────────┐
│researcher-v2│ │drafts-v1│ │ sentiment-v1 │
└─────┬───┘ └─────┘ └────────────┘
      │
      ▼
┌──────────────┐
│fact-checker-v1│
└──────────────┘
```

Writer-v1 fans out to three subagents per turn: researcher-v2, drafts-v1, sentiment-v1. Researcher-v2 depends on fact-checker-v1 (the leaf). The fan-out at writer-v1 is 3; the chain depth to fact-checker-v1 is 3 hops.

Each subagent is a stub returning a deterministic response after a configurable delay:

- Nominal: 200 ms.
- Degraded (what you will trigger on fact-checker-v1): 5000 ms, then 20000 ms.

Writer-v1's handler fans out concurrently to the three subagents, aggregates their responses into a "draft with research + sentiment + facts" output, and returns.

You will run the topology twice:

1. **Naive config.** No circuit breakers, no bulkheads, no deadlines, no fan-out policy. Writer-v1 waits for all three subagents; researcher-v2 waits for fact-checker-v1.
2. **Hardened config.** Per-upstream circuit breakers, per-upstream bulkheads, propagated deadlines, `best_effort` fan-out policy. The exact knobs are chapter 05's T2 defaults — you tune to your implementation.

Under load (10–50 requests per second to writer-v1 for 60 s, depending on your cluster), apply the leaf-agent degradation after 20 s. Measure:

- Writer-v1 end-to-end p50 / p95 / p99.
- Writer-v1 error rate.
- Fan-out partial-success rate.
- Circuit-breaker state timeline (hardened only).
- Observed-cascade reach (naive only — how far did the slowness propagate upstream?).

## Requirements

### Part 1 — The four agents

Build the four stub agents, extending exercise 02's A2A server scaffold. Each exposes a single skill and a configurable response delay via an environment variable `DELAY_MS`. The agents keep identity verification and audit records from exercise 02 — resilience does not replace security.

Writer-v1's handler:

```python
async def handle_draft(request, ctx):
    tasks = [
        client_researcher.tasks_send(session_id=ctx.session_id, message=request),
        client_drafts.tasks_send(session_id=ctx.session_id, message=request),
        client_sentiment.tasks_send(session_id=ctx.session_id, message=request),
    ]
    # Fan-out policy is the knob you will toggle between configs.
    results = await asyncio.gather(*tasks, return_exceptions=True)
    return compose_draft(results)
```

Researcher-v2 calls fact-checker-v1 once per request (single-upstream dependency; not fan-out).

### Part 2 — The naive config

No resilience primitives.

- No circuit breakers.
- No bulkheads (per-upstream unbounded; connection pools shared across all upstreams).
- No propagated deadline. Each hop has a long local timeout (60 s).
- Fan-out policy: `all_required` (writer-v1 fails if any subagent fails).
- Retries: 3 per call, no budget cap.

### Part 3 — The hardened config

Apply chapter 05's primitives. If you are using Istio:

- Per-agent `DestinationRule` with `http1MaxPendingRequests = 50`, `http2MaxRequests = 100`, `maxRetries = 2`, `outlierDetection.consecutive5xxErrors = 5`, `baseEjectionTime = 30s`.
- `VirtualService` with `retries.attempts = 2` and `retries.retryOn = "connect-failure,refused-stream,5xx"`.
- Retry budget via Envoy (approximate as 20 % via Istio's `EnvoyFilter` if `maxRetries` alone is not enough — document how you expressed the budget).

If you are using an in-process library:

- Per-upstream circuit breaker with threshold `consecutive_failures = 5`, `open_duration = 30s`.
- Per-upstream semaphore with `max_concurrent = 50`.
- Per-upstream retry with `attempts = 2`, exponential backoff + jitter, hard cap 2 s total retry time.

In both cases, propagate deadlines:

- Writer-v1 enforces a 15 s turn deadline (write `x-platform-deadline-ms: 15000` at the entry point, or configure in your test).
- Each subagent reads the inbound deadline, decrements by elapsed time at the span start, refuses work if the remaining is < 100 ms.
- Streaming is out of scope for this exercise; use synchronous responses.

Fan-out policy: `best_effort`. Writer-v1 waits up to the deadline for all three subagents, then composes with whatever returned. The AgentCard declares `platform.fan_out_policy = "best_effort"`; the scaffold enforces.

### Part 4 — The load generator

A load generator that:

- Fires `tasks/send` at writer-v1 at a sustained rate (10–50 req/s depending on what your cluster handles).
- Each request is unique (unique `taskId`) to avoid idempotency short-circuits.
- Runs for 60 s.
- After 20 s, SIGUSR1 (or an HTTP admin endpoint) triggers fact-checker-v1 to switch from 200 ms to 5000 ms.
- At 40 s, triggers fact-checker-v1 to switch to 20000 ms (worst case).
- Captures per-request latency + status to a JSONL file.

### Part 5 — The captured runs

Run the load generator twice:

- **Naive run.** Produces `naive.jsonl`. Also capture: writer-v1's thread/connection pool metrics over time (even if just logs of "pool used/pool size"), and the fact-checker-v1 request queue depth.
- **Hardened run.** Produces `hardened.jsonl`. Also capture: per-upstream circuit-breaker state transitions (writer→researcher, writer→drafts, writer→sentiment, researcher→fact-checker), bulkhead saturation over time, deadline-exceeded counts.

### Part 6 — The comparison report

In a `report.md`, produce:

- **Latency comparison table.** Writer-v1's p50 / p95 / p99 across four windows: 0–20 s (nominal), 20–40 s (5 s degradation), 40–60 s (20 s degradation), overall. Both configs side by side.
- **Error / partial-success rate table.** For the hardened config, break down the error class: `deadline_exceeded`, `circuit_open`, `bulkhead_full`, `subagent_failed_handled_by_best_effort`.
- **Cascade observation (naive).** A paragraph describing how far upstream the degradation propagated. Did writer-v1's own callers (if simulated as the load generator's clients) see degraded latency? Did writer-v1's connection pool fill?
- **Circuit-breaker timeline (hardened).** A table or a Mermaid gantt of state transitions: "20.3s — researcher→fact-checker breaker opens; 50.3s — half-open probe; 50.5s — closes." Include the writer→researcher breaker transitions too.
- **One-paragraph conclusion.** What the hardened config bought you and at what cost.

### Part 7 — The design doc

One-page `design.md`:

Motivated by the runs, defend where each resilience primitive lives on the platform:

- **Circuit breaker**: mesh or scaffold? Why? What happens on redeploy (mesh persists state across deploy; scaffold does not)?
- **Bulkhead**: mesh connection pool or application semaphore? Why?
- **Retry budget**: mesh or scaffold? How do you prevent retries at different layers from compounding?
- **Deadline**: who writes it? Who decrements it? Who refuses work?
- **Fan-out policy**: who enforces (`all_required` vs. `best_effort` vs. `quorum`)? Mesh or scaffold?
- **Fallback agent**: when writer→researcher's breaker is open, is there a researcher-v1 fallback? Who routes?

Each decision is one or two sentences. The point is that a future platform engineer reading this doc can tell what to look at when something goes wrong. Chapter 05 supplies the arguments; your job is to commit them to a decision record.

### Part 8 — The README

One page covering:

- How to run the topology (`docker compose up` or `kubectl apply`).
- How to run the load generator against naive config, then the hardened config.
- The four agents' response-delay knobs.
- A pointer to `report.md` and `design.md`.
- One "I was surprised by" observation — e.g. "my naive config did not cascade as visibly as I expected because writer-v1's own callers were my load generator, which is tolerant; I had to add synthetic back-pressure on the generator side to see the shape chapter 05 described."
- One open question for mod-410 — e.g. "the breaker transitions are visible in logs; what's the SLO on 'half-open probe succeeds within 30 s'? Does the alerting rota get paged on half-open attempts, closed-after-open, or only sustained-open?"

## Starter guidance

- **Keep the agents trivially fast and deterministic.** Use `asyncio.sleep(delay_ms)` and a canned response. Your lesson is in the shape of failure, not the agent's behaviour.
- **Add logs every 1 s with the resilience state.** Pool size, in-flight requests, breaker state, deadlines remaining on a sample request. You will stare at these logs for an hour; make them easy to read.
- **If using Istio, start with just two agents and one `DestinationRule` before scaling to four.** The mesh CRD surface has a learning curve; limit the surface you are debugging at once.
- **If using the library path, pick one library per language and stick with it.** Mixing `pybreaker` with a custom semaphore is more work than it is worth.
- **Use the same load generator for both runs.** The comparison is only valid if the inputs are identical.
- **Capture the metric at each layer.** Writer's view (p50/p95/p99 of its own end-to-end), researcher's view, fact-checker's view. Chapter 05's cascading shape is visible only when you can see the latency at each layer.

## Acceptance criteria

Mark done when:

- [ ] The four-agent topology runs and the single `curl` to writer-v1 produces a composed draft response in both configs.
- [ ] The load generator runs for 60 s against each config, with the fact-checker degradation triggered at 20 s and 40 s.
- [ ] `naive.jsonl` and `hardened.jsonl` are captured and loaded by a reader script.
- [ ] `report.md` contains the latency table, error-class table, cascade observation (naive), and circuit-breaker timeline (hardened).
- [ ] `design.md` answers each of the six where-does-this-live questions in one or two sentences.
- [ ] The hardened config's writer-v1 p95 under degradation is *lower* than the naive config's — if it isn't, the resilience is misconfigured and the exercise is not done yet.
- [ ] No handler code implements circuit breakers, bulkheads, retries, or timeouts itself. All resilience lives in the scaffold or the mesh.

## Stretch goals

- **Add a hedging path for the one idempotent, cheap call.** Fan out to researcher-v2 and sentiment-v1 with a hedge policy; measure the p99 impact at the cost of ~1.5× load on those agents. Chapter 05's cautions about expensive upstreams apply.
- **Add a cycle to the graph and watch the registry refuse the deploy.** Modify an AgentCard so sentiment-v1 may call writer-v1. The registry's cycle check rejects. If your registry is a stub, write the check as part of this exercise.
- **Implement the `platform.fallback_agent` primitive.** When writer→researcher's breaker is open, route to researcher-v1 (a cheaper, less-capable alternative). Measure the quality degradation (you need an eval score — stub one for the exercise) vs. the availability improvement.
- **Compare retry strategies.** Run the hardened config three times: `attempts=0`, `attempts=2`, `attempts=4`. Plot the error rate and the load generator's retry-amplification. Chapter 05's "retries without a budget" shape should be visible.
- **Inject a cross-dependency.** Have drafts-v1 also depend on fact-checker-v1. Now fact-checker-v1's degradation affects two of writer-v1's three subagents. Measure how `best_effort` behaves when two of three fan-out children fail.
- **Alert on breaker-open from a synthetic SRE dashboard.** Point Prometheus + Grafana at your mesh (or scrape your library's metrics); build one panel that flashes when any writer→upstream breaker opens. Feed the workflow back into mod-410 by writing a one-sentence alert description per panel.
