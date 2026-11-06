# Circuit Breakers, Bulkheads, and Cascading-Failure Prevention

Chapter 04 pinned who the request is. This chapter pins what happens when the downstream agent is slow, failing, or dead — and why "retry and hope" is the last remaining way platforms destroy themselves in production.

The lesson is older than this module. [Michael Nygard's *Release It!*](https://pragprog.com/titles/mnee2/release-it-second-edition/) names the pattern: a slow dependency fills up the caller's thread pool, then the caller fills up *its* caller's thread pool, then the whole system is wedged, then the on-call engineer reboots everything and prays. The famous AWS post-mortem on S3's 2017 US-EAST-1 outage ([*Summary of the Amazon S3 Service Disruption*](https://aws.amazon.com/message/41926/)) shows the pattern at infrastructure scale. Netflix's response — [Hystrix](https://github.com/Netflix/Hystrix) in 2012, and [Resilience4j](https://resilience4j.readme.io/) as its modern successor — productised the escape: *isolate* the slow dependency, *fail fast* while it recovers, *fan out safely* under load.

Inter-agent graphs are vulnerable to this cascade with two accelerants the service-mesh literature is only now catching up to: agent calls are longer (seconds to minutes, not milliseconds), and the fan-out at each hop is wider (one manager calls five subagents, each of which may call five more). A naive deployment of three agents in a chain can wedge the whole fleet within one minute of a 2× latency increase on the leaf.

## The patterns the plane uses

Four classic patterns, applied to the inter-agent graph.

### Circuit breaker

A state machine per upstream that fails fast when the upstream is unhealthy, probes periodically, and restores traffic when the probe succeeds. States:

- **Closed.** Normal; calls go through. The breaker counts failures over a sliding window.
- **Open.** Failure rate (or consecutive 5xx count) exceeded the threshold; calls fail fast locally without touching the upstream. Open-state dwell is bounded (e.g., 30 s).
- **Half-open.** After the open-state dwell, a small number of trial calls are allowed. Success restores closed; failure re-opens.

The thing the circuit breaker gives up is the chance that the upstream has recovered. The thing it protects is the caller: fast failure is a feature, not a bug.

Envoy's [outlier detection](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/outlier) ejects individual hosts from the upstream pool when they fail; Envoy's [circuit breaker](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/circuit_breaking) limits total concurrent operations against the upstream (connection count, pending requests, retries). The plane uses both — they are complementary, not alternatives. Chapter 05's "Configuration" section lists the Envoy/Istio surface.

### Bulkhead

Ship compartments in a sinking ship: an isolated resource pool per upstream so one dependency cannot drain the shared pool. If writer-v1 can only hold twenty concurrent researcher-v2 calls and ten concurrent fact-checker-v1 calls, a slow researcher-v2 can starve its own pool but cannot starve fact-checker-v1's pool. Writer-v1's own request-handling threads stay free to accept new work.

The bulkhead primitive on the mesh is Envoy's connection pool limits in the `DestinationRule`'s `connectionPool.http` block (`http1MaxPendingRequests`, `http2MaxRequests`, `maxRequestsPerConnection`, `maxConnections`). In application code, it is a semaphore per upstream — the client scaffold's job, not the handler's. [Resilience4j's bulkhead](https://resilience4j.readme.io/docs/bulkhead) is the reference implementation.

### Retries with budget

Retries are the right answer for transient transport failures (connection reset, 5xx without a response body, idempotent operation timed out). They are a disaster without a budget. A single user request that generates three retries at each of four hops is `3^4 = 81` upstream calls — the amplification is the thing that causes cascades in the first place.

The platform's rule:

- **Mesh retries** are for transport failures only. Max two attempts, idempotent methods only (and A2A `tasks/send` is idempotent by `taskId`). [Istio's `HTTPRetry`](https://istio.io/latest/docs/reference/config/networking/virtual-service/#HTTPRetry) is the surface.
- **Retry budget** caps the retry rate at the client. Envoy's [retry budget](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/cluster/v3/circuit_breaker.proto#envoy-v3-api-msg-config-cluster-v3-circuitbreakers-thresholds-retrybudget) is a percentage of total requests. 20 % is a common starting point — if retries exceed 20 % of requests, something upstream is wrong and amplifying is making it worse.
- **Application retries** are the client scaffold's business, with exponential backoff + jitter and a hard cap on wall-clock spend. The [AWS Architecture Blog post on jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/) is the standard reference.

### Hedging (speculative execution)

For latency-sensitive, idempotent calls, firing a second request after a p95-latency delay and taking whichever returns first. Trades a small amount of extra load for p99 compression. The pattern is Jeff Dean's "tail at scale" ([Dean & Barroso, *Communications of the ACM*, 2013](https://research.google/pubs/the-tail-at-scale/)).

Hedging is powerful when the upstream is cheap and the tail is heavy. It is a disaster when the upstream is expensive (model calls — hedging a $0.50 call to double its load is fifty dollars a thousand requests the second you flip the switch). The plane's rule: hedging is off by default; opt in per agent with a declared budget, and a mesh-level cap (Envoy's `RetryPolicy` with `retry_on: response-headers` and a `hedge_on_per_try_timeout` fraction — [Envoy's hedge policy](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/router_filter#x-envoy-upstream-rq-per-try-timeout-ms)).

<!-- needs-research: confirm the current state of Envoy's hedge-policy stability as of the pinned Envoy version; the feature has moved between alpha and stable several times across releases. -->

## Why the graph matters

A single pair of agents calling each other is not the dangerous shape. The dangerous shape is a graph with:

- **Shared downstream dependencies.** Two different agents both call fact-checker-v1. Fact-checker-v1 slows down; both callers feel it; their callers feel it.
- **High fan-out.** A manager-as-tools pattern with five concurrent subagent calls. One slow subagent pins the manager at the fan-out join if the handler waits for all five.
- **Deep chains.** User → router → writer → researcher → fact-checker. Five hops. Each hop's own timeout compounds; the user's effective deadline is unknowable without propagated deadlines (chapter 03).
- **Circular potential.** Agent A can call agent B; agent B *also exports a skill A would use*. If the routing allows the cycle, a bug cascades into infinite recursion.

The plane's defences, applied to the graph:

- **Per-upstream circuit breakers at every edge.** The mesh enforces per-destination state; the breaker for writer→researcher is independent of the breaker for writer→fact-checker.
- **Per-upstream bulkheads at every edge.** A slow researcher-v2 starves its own pool at writer-v1, not fact-checker-v1's pool.
- **Deadlines propagate.** Chapter 03's `x-platform-deadline-ms`. A hop that cannot complete before the user's deadline refuses the work.
- **Cycle detection in the registry.** The registry has a directed-edge list of "agent X declares that it may call agent Y" (from AgentCard's `platform.may_call`). A deployment that would introduce a cycle requires an explicit override, logged and reviewed.
- **Fan-out caps at the scaffold.** A manager cannot fan out more than N subagent calls concurrently without declaring it on its AgentCard; the mesh enforces concurrent-call counts via bulkhead; the scaffold enforces per-turn fan-out via semaphore.

## Resilience tiering

Every agent registers a resilience tier on its AgentCard (`platform.resilience_tier`). The tier is what the mesh config generator consumes to render circuit-breaker and bulkhead settings. The platform ships three tiers; teams pick one per agent.

- **T1 — tight.** Low fan-out budget, low concurrency per upstream, low retry attempts, short deadlines. For agents that drive user-facing latency: an interactive chat router, a first-hop agent on a web-facing surface.
- **T2 — standard.** Moderate fan-out, moderate concurrency, two retries, medium deadlines. The default.
- **T3 — long-running.** High concurrency allowed, longer deadlines, lower retry rate (because the work is expensive). For research agents, background drafting, agentic coding tasks.

Concrete values are platform-tuned; the point is that *the tier is the knob*, not thirty individual YAML fields per agent. A team moving an agent from T2 to T3 is a reviewable AgentCard change.

Example T2 default (Istio `DestinationRule`):

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: researcher-v2
  namespace: shared
spec:
  host: researcher-v2.a2a.svc.cluster.local
  trafficPolicy:
    connectionPool:
      http:
        http1MaxPendingRequests: 50
        http2MaxRequests: 100
        maxRequestsPerConnection: 10
        maxRetries: 2
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 10s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
      minHealthPercent: 50
    tls:
      mode: ISTIO_MUTUAL
```

<!-- needs-research: verify the Istio CRD field names against the version your cluster runs; the `http1MaxPendingRequests` / `maxRequests` fields have been renamed across versions. -->

## Deadlines are the tactical difference

Chapter 03 introduced the deadline-not-timeout rule. For resilience, deadlines are the single highest-value primitive — more than circuit breakers, more than bulkheads, more than retries.

Why: a 60-second user deadline split naively across a 5-hop chain is 12 seconds per hop. A chain that *propagates* the remaining deadline refuses work the moment the deadline is unachievable, freeing the rest of the chain to do something else. A chain that uses per-hop local timeouts wastes work on every hop.

The plane's rule is uniform:

- The entry gateway writes `x-platform-deadline-ms` on the first-hop call from the user's requested budget (defaulted per endpoint).
- Every A2A hop decrements the deadline by the time already spent on the current span.
- Every server scaffold refuses to start work it cannot finish before the deadline; returns `failed: deadline_exceeded` immediately.
- Streaming tasks that pass the deadline mid-stream emit a final `failed: deadline_exceeded` and close the stream; the mesh's stream timeout is a *backstop*, not the primary mechanism.

The failure-mode question to ask in review: "If this agent were 2× slower than usual, what would happen to the user's deadline?" If the answer is "the user would wait indefinitely," the deadline is not propagating.

## Agent-specific failure modes

Not every failure in an agent graph is a classic distributed-systems failure. A short catalogue.

### Semantic failure (the agent returns, but wrong)

The agent responded within the deadline, but the output is semantically wrong — hallucinated, incomplete, unsafe. The mesh sees a 200; the breaker does not fire.

- **Where it is caught.** The evaluation plane (mod-407) and the safety plane (mod-411). Not this chapter's circuit breaker.
- **The plane's role.** Emit the trace so the eval plane can replay the failure; the breaker does not protect against this and should not be the one trying.
- **The one exception.** A persistent *structural* failure — "the agent always returns malformed JSON" — is catchable at the server scaffold's output validation. Treat it as a 5xx for the breaker's purposes.

### Loop failure (the agent won't stop)

The agent is in an infinite tool-calling loop. It would complete, eventually, but is burning tokens and racking up cost.

- **Where it is caught.** The runtime's turn budget (mod-402) and the model gateway's cost cap (mod-406). The inter-agent plane sees a long-running task and should trust the deadline — if the runtime enforces `max_steps_per_turn` and the model gateway enforces per-turn token budgets, the inter-agent deadline is the external backstop.
- **The plane's role.** Honour the deadline. Cancel via `tasks/cancel` when the breaker opens or the deadline passes. The server scaffold propagates cancellation into the handler, which the runtime surfaces as "stop the loop now."

### Streaming stall (the task stays `working` forever)

The task is in `working` state. The stream emits no updates. Nothing is explicitly failed; the client is holding an open connection.

- **Where it is caught.** The plane's per-update deadline: `x-platform-max-silence-ms` (or an equivalent policy), enforced by the client scaffold on the stream. No update within the window → treat as a failure, cancel the task, surface `failed: stream_stalled`.
- **The mesh's role.** Mesh stream timeouts are a backstop; the primary mechanism is the client scaffold's logical health check.

### Fan-out partial failure

Manager-as-tools calls five subagents. Three complete. Two fail or time out.

- **Where it is caught.** The manager's handler, with a declared policy on its AgentCard (`platform.fan_out_policy`):
  - `all_required` — any failure fails the manager's turn.
  - `best_effort` — surface what came back; the handler composes a degraded answer.
  - `quorum` — require N of M; the manager's loop decides on the result.
- **The plane's role.** Capture the per-subagent `A2ACallRecord` and the aggregate outcome; the manager's handler is where the business decision is made.

### Authorisation failure (chapter 04)

A downstream agent refuses because the scope doesn't cover the invoked skill.

- **Where it is caught.** The server scaffold returns 403. The breaker does *not* count this as a failure — it is a correctly-functioning denial, not a reliability failure. The breaker-threshold rule is "5xx and connect errors only."

## The one thing the mesh cannot do alone

The service mesh is very good at per-hop resilience. It is less good at *graph-level* policy — "this whole turn, across every agent it touches, costs at most $0.10 and completes within 30 s." Graph-level policy is the plane's own resilience layer, implemented in the client scaffold and informed by the registry:

- **Turn-wide deadline.** From the entry gateway; propagated via `x-platform-deadline-ms`.
- **Turn-wide cost budget.** The scaffold reads the per-tenant budget from the model gateway (mod-406) and declines to initiate an A2A call if the projected cost of the chain would exceed it.
- **Fan-out cap.** The scaffold caps concurrent calls per manager turn; the mesh's bulkhead backs it up, but the semantic knob is at the application.
- **Cycle break.** The registry's cycle detection; the scaffold's deployment-time check that no AgentCard introduces a cycle.
- **Graceful degradation path.** Each agent's AgentCard declares a `platform.fallback_agent` where meaningful. The scaffold routes to the fallback when the primary's breaker is open; the audit record captures the fallback event.

The pattern is: the mesh defends the wire; the scaffold defends the graph.

## A concrete scenario: researcher-v2 starts to slow

Put everything together with a worked example. The initial state: writer-v1 (T2) calls researcher-v2 (T2) as part of every drafting turn. Researcher-v2 depends on fact-checker-v1 (T3). Cluster latency is nominal.

**Minute 0 — nominal.** Writer-v1's breaker for researcher-v2 is closed. Breaker metric: `envoy_cluster_circuit_breakers_default_cx_open=0`. P99 for the researcher-v2 call is 1200 ms. Deadlines: 15 s turn deadline; 14 s remaining when the researcher call starts.

**Minute 5 — fact-checker-v1 begins to degrade.** A downstream index refresh caused fact-checker-v1's p99 to double. Researcher-v2's own p99 climbs — it waits on fact-checker-v1.

**Minute 6 — researcher-v2 begins returning `failed: deadline_exceeded` for calls that cannot finish in time.** Writer-v1's breaker for researcher-v2 counts these as application-level failures *only if* the scaffold is configured to (platform default: yes, on `failed` outcomes). The mesh's outlier detection sees no 5xx yet (the server is responding; it is returning a semantic failure).

**Minute 7 — researcher-v2's pool fills.** New requests pile up in the connection pool. Envoy starts returning 503 for overflow. Mesh outlier detection kicks in — researcher-v2 pods with the most errors are ejected from the upstream pool. Writer-v1's load concentrates on the remaining healthy pods, which may also saturate.

**Minute 8 — writer-v1's breaker opens for researcher-v2.** `consecutive5xxErrors=5` is hit. For 30 s, writer-v1's calls to researcher-v2 fail fast. Writer-v1's handler, which declared `platform.fan_out_policy=best_effort`, emits a degraded answer ("I could not confirm these facts; please verify before sending").

**Minute 8:30 — breaker half-opens.** A trial call to researcher-v2 goes through.

- If researcher-v2 has recovered: trial succeeds, breaker closes, normal traffic resumes.
- If not: trial fails, breaker re-opens, dwell extends.

**What the plane protected.**

- Writer-v1 did not lock up. Its own callers still got answers (degraded).
- Fact-checker-v1's degradation did not propagate beyond researcher-v2. Writer-v1's calls to other agents (fact-checker-v2, if deployed, or a completely different agent) continued unaffected because the bulkheads are per-upstream.
- The user got a visible, bounded degradation ("I could not confirm these facts") rather than a stuck UI.
- The audit chain, trace, and `A2ACallRecord` captured every step — SRE (mod-410) has the evidence for the post-mortem.

**What the plane did not do.**

- It did not solve fact-checker-v1's degradation. SRE is paged on the leaf's SLO breach.
- It did not retry forever. The retry budget prevented amplification; the user did not wait 30 s because the mesh was heroically re-attempting.
- It did not fail-open. Researcher-v2's failure did not silently drop the authorisation check or the identity chain.

## SRE-facing signals the plane emits

The resilience posture is useless without signals to tune against. Every signal below lands in the observability plane (mod-408) with the `A2ACallRecord` schema; the SRE module (mod-410) owns the alerting.

- **`a2a_circuit_breaker_state_total{caller,callee,state}`** — gauge of the breaker's state.
- **`a2a_retry_rate{caller,callee}`** — fraction of calls that triggered a retry. Alarm > 10 %; alarm > 20 % as paging.
- **`a2a_deadline_exceeded_total{callee,reason}`** — count of deadline failures. A sudden spike at a leaf is a canary for the next cascade.
- **`a2a_pool_pending_requests{caller,callee}`** — bulkhead depth. Saturation precedes cascade.
- **`a2a_cross_tenant_calls_total{src_tenant,dst_tenant,allowed}`** — chapter 04 signal; a non-zero `allowed=false` is a security incident.
- **`a2a_fan_out_concurrency{caller}`** — current fan-out; alarm when it approaches the AgentCard's declared cap.

These are complementary to the GenAI semantic conventions (mod-408 chapter 02) — the latter describes *what happened in the agent turn*; these describe *how the inter-agent plane behaved*.

## The anti-patterns to stay away from

Short catalogue of the shapes that look like resilience but aren't.

- **Infinite retries with linear backoff.** Looks like persistence; is amplification. Use retry budgets and exponential backoff with jitter.
- **Hedging without a budget.** Every call becomes two. Cost doubles. The caller's own capacity halves.
- **Global timeout at the entry gateway only.** Downstream hops don't know the deadline; they keep working after the user has given up.
- **Fail-open authorisation.** When the authz server is down, letting the call through. The audit chain is broken; a compromised downstream now operates without restriction. Fail-fast instead.
- **Circuit breaker per *call site* instead of per *upstream destination*.** Two call sites at the same caller to the same callee should share one breaker, or one broken upstream opens two breakers independently and the math gets weird.
- **Retry-on-any-error.** `retry_on: 5xx,4xx` is a configuration smell. A 400 is not a transient failure; retrying it just means the caller sees the 400 N+1 times.
- **"Just restart everything."** The on-call anti-pattern. If the system cannot recover without a human restart, it has no resilience posture at all.

## Summary

- Four classic patterns apply: circuit breakers (fail fast on bad upstreams), bulkheads (isolate pools per upstream), retries with budget (bounded, transport-failure only), and hedging (opt-in, per-agent-budgeted). The mesh is the primary enforcement; the client scaffold layers graph-level policy on top.
- The dangerous graph shapes are shared downstream dependencies, high fan-out, deep chains, and cycle potential. Each has a specific defence (per-edge breaker, per-edge bulkhead, propagated deadline, registry-level cycle detection).
- Agents are tiered (T1 tight, T2 standard, T3 long-running) on the AgentCard; the mesh config generator renders tier-specific `DestinationRule`s. Teams pick a tier, not individual knobs.
- Deadlines (propagated via `x-platform-deadline-ms`) are the single highest-leverage primitive. Every hop refuses work it cannot finish in time; wasted work is the enemy.
- Agent-specific failure modes — semantic failure, loop failure, streaming stall, fan-out partial failure — have specific catches. Semantic failure is the eval plane's problem, not the breaker's; stream stall is the scaffold's silence-deadline; fan-out partial failure is the AgentCard's declared policy.
- The mesh defends the wire; the scaffold defends the graph. Turn-wide deadline, turn-wide cost, fan-out cap, fallback routing, cycle break — all live in the scaffold informed by the registry.
- SRE-facing signals (`a2a_circuit_breaker_state`, `a2a_retry_rate`, `a2a_deadline_exceeded_total`, `a2a_pool_pending_requests`, `a2a_cross_tenant_calls_total`) emit to mod-408; mod-410 owns the alerting.

The next chapter closes the module by picking up the topology question — handoffs vs. manager-as-tools — and the role handoffs to `agentic-systems-architect` and `senior-agentic-ai-engineer`.
