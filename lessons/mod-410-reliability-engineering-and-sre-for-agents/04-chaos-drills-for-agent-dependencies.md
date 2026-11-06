# Chaos Drills for Agent Dependencies

Chapters 02 and 03 built the measurement side of the plane. This chapter builds the rehearsal side. The source is [Netflix's *Chaos Engineering: System Resilience in Practice*](https://www.oreilly.com/library/view/chaos-engineering/9781492043858/) (Rosenthal & Jones), the [Principles of Chaos Engineering](https://principlesofchaos.org/), and the [AWS Fault Injection Service documentation](https://docs.aws.amazon.com/fis/latest/userguide/what-is.html) — adapted to the four agent-dependency failure classes the agent platform actually sees: model-provider outage, MCP-server flake, sandbox exhaustion, eval-infra degradation.

A platform that has never practised a failure has no idea how long detection, mitigation, and recovery actually take. Incident post-mortems are written after the fact and under pressure; drills surface the same information in a controlled window where it can be fixed. The output of a drill is not "did the system survive" (hopefully yes) but "how long did detection take, where did the alert first fire, which playbook worked, where did the team get stuck".

## The chaos-engineering frame, in one page

Four principles from the Principles of Chaos Engineering:

- **Build a hypothesis around steady-state behaviour.** Not "nothing bad happens" — a specific SLI stays within bounds. Example: "with the US-East model provider offline, writer-v1's TTFA p95 stays below 15 s (vs. 10 s normally) and session completion stays above 97 %."
- **Vary real-world events.** The fault must look like production. A simulated 503 from the model gateway is not the same shape as the real provider's 429-with-retry-after; a 1 % network loss is not a regional outage. The paved-road drill library pins real-world-shaped faults.
- **Run experiments in production** (or as close as possible). Most organisations start in staging; mature practice is to run a GameDay-style experiment in production with explicit scope, the on-call aware, and the fault bounded.
- **Automate experiments to run continuously.** The quarterly drill is a start. The target is continuous (daily or weekly) fault injection against low-impact subsets of production, with the results feeding the reliability-posture dashboard.

This module expects you to reach "quarterly GameDays, plus automated staging drills on every deploy". Full continuous production chaos is a next-level ambition; the Netflix / Chaos Monkey canon is the reference for teams that are ready.

## The four agent-dependency failure classes

Named explicitly because they are the ones the paved-road drill library must cover. Each is described below with its hypothesis, fault injection, detection and response, and scorecard.

### Class 1 — Model-provider outage

The scenario: the primary model provider's inference endpoint returns 503s, 429s, or hangs — for a specific model family, a region, or the whole account.

**Hypothesis.** When the primary provider for `claude-sonnet-4-6` returns sustained 503s, the model gateway (mod-406) fails over to the secondary route within 60 s, the TTFA p95 degrades from 10 s to ≤ 20 s during fail-over, and session-completion stays above 98 %.

**Fault injection.** Three concrete patterns:

- **Mesh-level.** Istio `VirtualService` fault injection against the model-provider's egress endpoint: `fault.abort.httpStatus = 503, fault.abort.percentage.value = 100`. See the [Istio fault-injection guide](https://istio.io/latest/docs/tasks/traffic-management/fault-injection/).
- **Proxy-level.** [Toxiproxy](https://github.com/Shopify/toxiproxy) between the model gateway and the real provider, injecting `timeout` toxic or `latency` toxic (e.g., 30 s delay to simulate the hang shape).
- **Gateway-level.** The paved-road model gateway (mod-406) ships a chaos mode: `POST /admin/chaos` with a fault spec (`{"route": "claude-sonnet-4-6:us-east", "mode": "return_503", "fraction": 1.0, "duration": "10m"}`). The gateway returns the configured fault to upstream callers; the drill is scope-limited to a specific tenant or test workload.

The gateway-level pattern is the paved-road default — it is safe (bounded to a tenant), observable (the gateway logs the injection), and realistic (the fault shape matches what the gateway would surface from the real provider).

<!-- needs-research: confirm the current Istio VirtualService fault-injection field path (`fault.abort.httpStatus` vs. `fault.abort.http_status`); the CRD has shifted field casing between minor versions. -->

**Detection and response.**

- **Expected detection path.** The model-gateway's per-provider error-rate SLI alerts within 60 s (fast-burn against the provider-availability SLI). The gateway's auto-failover kicks in; the provider-dependency-adjustment on the workload's SLO blunts the TTFA hit.
- **Playbook.** "Model-provider outage" playbook — check gateway status, confirm fail-over to secondary, if secondary degraded then enter degraded-mode (reduce max tokens, disable long-context routes, surface a banner).

**Scorecard.** The drill measures:

- **Time to detect (TTD).** Seconds from fault injection to first on-call-eligible alert.
- **Time to mitigate (TTM).** Seconds from alert to the gateway's fail-over completing.
- **Blast radius.** How many workloads saw the degradation in SLI series.
- **User-visible impact.** Fraction of sessions whose TTFA exceeded the SLO target during the window.
- **Playbook score.** 1–5 on "did the playbook steps match what the on-call actually did".

### Class 2 — MCP-server flake

The scenario: an MCP tool server (mod-403) starts returning intermittent 5xx errors or hanging on a subset of requests. The failure is partial, not binary — half of calls succeed, half fail — and the pattern varies minute to minute.

**Hypothesis.** When a single MCP server (e.g., `search-tool-v2`) returns 50 % 503s, the policy plane (mod-405) authorises tool calls as before, the inter-agent plane (mod-409) breaks the circuit after 5 consecutive failures per Envoy outlier detection, the agents that depend on the tool degrade gracefully (fall back to alternative tool or surface a "search unavailable" message), and the fleet-wide session-completion rate drops by ≤ 1 %.

**Fault injection.** The MCP gateway (mod-403) ships a chaos mode analogous to the model gateway's. The fault spec targets a specific tool and specifies a rate (`{"tool": "search-tool-v2", "mode": "return_500", "fraction": 0.5, "duration": "10m"}`). Alternatively, [Chaos Mesh](https://chaos-mesh.org/) HTTPChaos against the MCP server's pod can produce the pattern without a gateway-side admin surface.

**Detection and response.**

- **Expected detection path.** Envoy outlier detection (mod-409 chapter 05's circuit breaker) marks the host as unhealthy; the per-tool error-rate SLI crosses its slow-burn threshold; the fleet's TTFA p99 climbs (because tool retries + fallbacks extend the turn). The slow-burn alert opens a ticket; it does not page.
- **Playbook.** "MCP-tool flake" playbook — identify the failing tool, check if other hosts in its service are healthy, trigger a scale-up or a cordon if one host is the problem, surface the degradation to the dependent agents' owners.

**Scorecard.** Same as class 1, plus:

- **Circuit-breaker trip time.** Seconds from first failure to the breaker going open.
- **Fallback usage.** Fraction of affected tool calls that used a fallback tool vs. surfacing an error.
- **Partial-success tolerance.** Did the fan-out agents (mod-409 chapter 06's manager-as-tools) continue with 2-of-3 subresults, or did they fail the whole turn?

### Class 3 — Sandbox exhaustion

The scenario: the code-execution sandbox (mod-404) runs out of capacity — fork-bomb against the pool, a leak in one tenant's code that holds containers open, or a scheduled batch driving utilisation past the safe limit.

**Hypothesis.** When sandbox utilisation reaches 95 %, the sandbox's admission controller rejects new jobs with a legal error, agents gracefully surface "cannot execute code right now" to the user, and the fleet's completion rate degrades by ≤ 5 % *for workloads that depend on code execution* and 0 % for the rest.

**Fault injection.** Two patterns:

- **Capacity reduction.** Scale the sandbox pool down to 20 % of normal. The controller's admission rate drops; new jobs queue or reject. This is a cleaner test than a fork-bomb because the fault is reversible with a scale-up.
- **Synthetic load.** A tenant-sandboxed worker submits enough jobs to consume the capacity. Realistic shape for the "noisy neighbour" scenario; the paved-road drill library pins this as the standard.

**Detection and response.**

- **Expected detection path.** The sandbox's utilisation SLI (external to this module — mod-404 chapter on capacity) alerts on slow-burn; the admission-reject rate crosses its own threshold; `platform.sandbox.admission` events on affected sessions rise. Agents' error paths surface the condition to users.
- **Playbook.** "Sandbox exhaustion" playbook — check which tenant or workload is driving utilisation, decide between scale-up (if provisioner supports) and per-tenant rate-limit (if the issue is a noisy neighbour), communicate ETA to affected teams.

**Scorecard.** Same as above, plus:

- **Noisy-neighbour isolation.** Did the tenant quota mechanism kick in before fleet-wide impact? (Prior work in mod-404; the drill exercises the mod-404 primitives.)
- **Queue-depth growth rate.** Jobs pending per second during saturation.
- **Recovery curve.** Seconds from scale-up (or rate-limit) to the queue draining.

### Class 4 — Eval-infra degradation

The scenario: the evaluation plane (mod-407) cannot run its gates — the judge model is unavailable, the fixture catalogue's object store is throttling, the CI eval-runners pod is down.

**Hypothesis.** When the judge model (the eval plane's primary judge, e.g., `claude-sonnet-4-6-judge`) is unavailable, the pre-merge eval gate (mod-407 chapter 05) enters a documented failure mode — either "block all merges with a clear error" or "degrade to a smoke-test-only gate" — rather than silently passing everything.

**Fault injection.** Point the eval plane's judge router at a non-existent endpoint, or kill the eval-runners deployment. The drill is distinctive because the failure is *mostly invisible to running agents* — it only affects the gate for new deploys.

**Detection and response.**

- **Expected detection path.** The eval plane's own health SLIs — judge availability, fixture-catalogue latency, runner-pod liveness — alert the eval-plane on-call (mod-407's handoff to SRE plane). The gate behaviour is controlled by the eval plane's degraded-mode configuration; the SRE plane audits that the degradation does not silently become "gate passes everything".
- **Playbook.** "Eval-infra degradation" playbook — confirm which eval component is affected, confirm the gate behaviour is the documented fail-closed shape, communicate to teams waiting on gates, ETA to recovery.

**Scorecard.**

- **Did the gate fail closed?** Pass/fail. A silently-pass gate is the most expensive failure mode — regressions ship under the misconception that they were tested. Fail-closed is the invariant.
- **TTD on the eval-plane health signal.**
- **Downstream merge delay.** Minutes of blocked merges across the fleet during the drill.
- **Communication latency.** Time from degradation start to a team-visible status message.

## The scorecard, as a document

Every drill produces a scorecard. The paved-road template:

```yaml
# drills/2026-Q4-model-provider-outage/scorecard.yaml
drill:
  name: model-provider-outage-us-east
  class: model_provider_outage
  hypothesis: >
    When the primary provider for claude-sonnet-4-6 returns sustained 503s,
    the model gateway fails over to the secondary route within 60 s, TTFA p95
    degrades to ≤ 20 s during fail-over, and session-completion stays ≥ 98 %.
  run_date: "2026-10-22"
  participants: [sre-oncall-primary, model-gateway-owner, writer-agents-lead]
  scope:
    workloads: [writer-v1, researcher-v2]
    tenants: [internal-test, demo]
  duration: 10m
timeline:
  - t: "+0s"
    event: fault injected (gateway chaos mode)
  - t: "+22s"
    event: provider-error-rate SLI crosses fast-burn threshold
  - t: "+28s"
    event: first on-call alert received
  - t: "+41s"
    event: gateway fail-over to secondary route completed
  - t: "+62s"
    event: TTFA p95 recovers to within SLO
  - t: "+600s"
    event: fault removed
measurements:
  time_to_detect_s: 28
  time_to_mitigate_s: 41
  blast_radius_workloads: 2
  sessions_impacted: 143
  sessions_breached_slo: 11
  completion_rate_delta: -0.004
hypothesis_result: confirmed
playbook_score: 4
findings:
  - severity: medium
    title: >
      Playbook step "verify secondary provider's rate-limit headroom" did not
      exist; on-call had to improvise.
    owner: sre-oncall-primary
    fix_pr: null
  - severity: low
    title: >
      Secondary provider's latency during fail-over spiked to 18s p95 before
      stabilising; worth investigating with provider or widening the cache
      warm-up.
    owner: model-gateway-owner
    fix_pr: null
fixture_candidate: true
fixture_id: fixtures/incident/2026-Q4-model-provider-outage@v1
followups:
  - update playbook step for secondary rate-limit
  - add provider-dependency-adjustment data point to writer-v1 SLO
```

The scorecard is the drill's evidence. It is retained with the incident records; the `fixture_candidate` and `fixture_id` fields are the hand-off to chapter 06's regression-pipeline.

## The drill matrix

A matrix lists what has been drilled, when, and with what result. The paved-road version:

| Class                       | Last drilled | Result   | Playbook rev | Fixture generated |
|-----------------------------|--------------|----------|--------------|-------------------|
| model-provider outage       | 2026-10-22   | confirmed| v3           | yes               |
| MCP-server flake            | 2026-10-29   | partial  | v2           | yes               |
| sandbox exhaustion          | 2026-09-03   | confirmed| v2           | no (no new case)  |
| eval-infra degradation      | 2026-08-15   | failed   | v1           | yes (fail-open)   |

Any class whose last drill was more than 90 days ago is overdue. Any class whose last drill `result` is `failed` is on the top of the next drill queue; the chapter 06 fixture-promotion ensures the fail won't happen the same way twice.

## The drill cadence

The paved-road cadence:

- **Weekly.** Automated staging drills triggered on every deploy. The drill library runs class-1 and class-2 faults in staging; the deploy is blocked if the hypothesis is violated.
- **Monthly.** A scheduled, scoped production GameDay — one of the four classes, rotating. The on-call is aware; the scope is bounded (specific tenants or workloads); the result is scored.
- **Quarterly.** Multi-class GameDay. Two classes in sequence (e.g., model-provider outage + MCP flake within the same window). The point is to expose compound-failure modes.
- **Ad-hoc.** After any incident whose root cause is a dependency failure, a drill of that dependency's failure class within 30 days.

Cadence is in the leadership plane's operating rhythm (mod-413). The drill library and the harness are the SRE plane's.

## The drill-safety contract

A drill that causes a real outage is the wrong kind of lesson. The paved-road drills respect four invariants:

- **Scope is bounded.** A tenant filter, a workload filter, or an explicit cluster-level allowlist. The gateway chaos mode refuses to inject if the scope is unset.
- **Duration is bounded.** A maximum duration (10 minutes paved-road default); the harness auto-removes the fault at the limit.
- **The on-call is paged on scope expansion.** If the drill's measurements show blast radius exceeding the scope, the harness auto-aborts and pages.
- **The "stop everything" button exists.** A single `drill:abort` action removes every active fault across every chaos component. The on-call knows the command; the drill lead knows the command; both are in the runbook.

A drill harness without these invariants is indistinguishable from an attack. Pin them before running the first drill.

## Tooling landscape

The paved-road ingredients, concretely:

- **[Chaos Mesh](https://chaos-mesh.org/)** — Kubernetes-native. HTTP chaos, network chaos, pod kill, IOChaos. Strong for class-3 (sandbox) and class-2 (MCP pods) at the pod level.
- **[LitmusChaos](https://litmuschaos.io/)** — Kubernetes-native alternative. Experiment-oriented; good for building reusable drill templates.
- **[Toxiproxy](https://github.com/Shopify/toxiproxy)** — TCP-level fault injection between services. Latency, slow-close, bandwidth limit. Strong for provider-egress simulation when you don't want to touch the real provider.
- **[AWS Fault Injection Service](https://docs.aws.amazon.com/fis/latest/userguide/what-is.html) / [Azure Chaos Studio](https://learn.microsoft.com/en-us/azure/chaos-studio/)** — cloud-provider-managed faults. Useful for infrastructure-level chaos (AZ down, instance termination) that your Kubernetes fabric cannot inject.
- **[Istio fault injection](https://istio.io/latest/docs/tasks/traffic-management/fault-injection/) / [Envoy fault filter](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/fault_filter)** — mesh-level HTTP faults. Clean for class-1 (model provider) and class-2 (MCP gateway).
- **[Steadybit](https://www.steadybit.com/), [Gremlin](https://www.gremlin.com/)** — commercial chaos platforms. Fuller UI and experiment catalogues; evaluate against the open-source tooling and your organisation's buy-vs-build posture.
- **Paved-road gateway chaos modes.** Model gateway (mod-406) and MCP gateway (mod-403) ship `POST /admin/chaos` surfaces. The platform owns these; they are the preferred injection sites because they are observable, bounded, and aligned to the record shapes the plane already emits.

The combination is deliberate: mesh chaos for transport faults, platform-gateway chaos for application-level faults, Kubernetes chaos for infrastructure, and ChaosMesh / Litmus as the orchestrator that ties scheduled experiments together.

## Common anti-patterns

- **Drilling only in staging.** Staging has different load, different data, different cache state. A drill that works in staging and fails in production is the one you wanted to catch — but only running in staging never produces it.
- **Drilling without a hypothesis.** "Let's see what happens if we kill X" is exploration, not a drill. Write the hypothesis; the measurement is "was it true".
- **Drilling the thing you just fixed.** Spending the quarterly drill on the last incident's exact shape is useful once, but over time it leaves the un-drilled classes dark. Rotate.
- **Not scoping blast radius.** A fault without a `fraction` and a `duration` is a production incident. Pin both. Prefer workload / tenant filters to blast-radius guesses.
- **Scorecard-less drills.** A drill with no scorecard produces no evidence; it is a vibe. The scorecard is the artefact.
- **Drilling without feeding the fixture pipeline.** A drill that revealed a bug and did not emit a mod-407 fixture is a half-done drill. Chapter 06.
- **Ad-hoc tooling per drill.** Build the paved-road harness once; every drill uses the same tooling. Novel tooling per drill means every drill is also a tooling exercise.

## Handoffs

The chaos programme does not live only in the SRE plane. The handshake:

- **mod-403 (MCP gateway) and mod-406 (model gateway).** Ship the chaos-mode admin surfaces. Class-1 and class-2 drills depend on them.
- **mod-404 (sandbox).** Owns the capacity SLI and the admission-controller behaviour the class-3 drill targets.
- **mod-407 (eval plane).** Owns the degraded-mode behaviour the class-4 drill tests (fail-closed vs. silently-pass). The SRE plane's audit.
- **mod-408 (observability plane).** The drill emits spans and events as any other platform record; the drill-run metadata is a resource attribute on the generated telemetry.
- **mod-411 (safety plane).** Guardrail outage is a related, parallel failure class — authors a separate drill in that module. The SRE plane co-ordinates the cadence.
- **mod-413 (leadership plane).** Approves the production-drill scope, reviews the drill matrix quarterly, signs off on fleet-wide drills.
- **Stream-aligned teams.** Receive drill notifications, review the affected scorecards, land the fixes for findings attributed to their workloads.

## Summary

- Chaos drills turn hypothetical dependency failures into measured platform behaviour. Four classes drive the paved-road programme: model-provider outage, MCP-server flake, sandbox exhaustion, eval-infra degradation.
- Each drill has a hypothesis (specific SLI bounds expected to hold), a fault injection (mesh-level, proxy-level, or gateway chaos-mode admin surface), a detection-and-response path, and a scorecard (TTD, TTM, blast radius, user-visible impact, playbook score).
- The drill matrix tracks when each class was last run, the result, and whether a fixture was promoted. Any class older than 90 days is overdue.
- Cadence: weekly automated staging drills, monthly scoped production GameDay, quarterly multi-class GameDay, ad-hoc after any incident of the matching class.
- Drill safety is explicit: bounded scope, bounded duration, auto-abort on blast expansion, a universal stop button. A drill without these is an incident.
- The tooling landscape (Chaos Mesh, Litmus, Toxiproxy, Istio/Envoy fault injection, cloud-provider FIS, platform-gateway chaos modes) is composed, not picked one. The paved-road gateway chaos modes are the preferred application-level injection sites.

The next chapter turns from "practising failure" to "running the real one" — the incident-response playbooks for the four agent-specific failure modes: runaway loop, context exhaustion, prompt-injection containment, tool-cascade meltdown.
