# The SRE Plane and What It Owns

The runtime (mod-402) hosts the agent loop. The MCP gateway (mod-403) catalogues tools. The sandbox (mod-404) runs code. The policy plane (mod-405) authorises what the agent does. The model gateway (mod-406) calls the provider. The evaluation plane (mod-407) verdicts whether a change makes any of them worse. The observability plane (mod-408) produces the shared trace vocabulary. The inter-agent plane (mod-409) carries agent-to-agent calls with circuit breakers and bulkheads. Up to this module, every plane has engineered its own correctness. This module asks a different question: given all the above, how does the platform *operate* under production load, and what does "healthy enough" mean when the thing you are operating is non-deterministic, long-running, and dependent on providers that do not publish their outage calendars?

That is the slice this module owns. It is not a new plane that emits new signals — mod-408 already does — it is the discipline that reads those signals, turns them into service-level objectives and error-budget policy, drills the fleet against the failures it has not yet seen, and runs the incident when one of them happens. The source material is the [Google SRE Book](https://sre.google/sre-book/table-of-contents/) and [SRE Workbook](https://sre.google/workbook/table-of-contents/); the application is agents, which look nothing like a request/response web service when the pager fires at 03:00.

This chapter fixes the slice. The chapters that follow pick the pieces: SLIs and SLOs for agent workloads (02), error-budget policy as a paved-road contract stream-aligned teams opt into (03), chaos drills for the four agent-dependency failure classes (04), incident-response playbooks for the agent-specific failure modes (05), and the post-incident loop that feeds regression fixtures back into the eval plane (06).

## What the SRE plane owns

The plane sits between the observability plane (mod-408) that produces spans, logs, metrics, and events, the evaluation plane (mod-407) that produces verdicts on offline fixtures, the inter-agent plane (mod-409) that emits resilience signals (breaker state, retry rate, bulkhead saturation), the safety plane (mod-411) that emits guardrail verdicts, and — on the other side — the stream-aligned teams whose agents run on the platform plus the leadership committee that reads the reliability posture. The plane owns:

- **The SLI catalogue.** The documented, pinned definitions of the service-level indicators the platform measures for every agent workload. Chapter 02 names the five the Agent Platform Engineer role must own — *session-completion rate*, *time-to-first-tool*, *time-to-final-answer*, *hallucination rate*, *safety-refusal rate* — each as a formula over mod-408's `gen_ai.*` and `platform.*` attributes. Chapter 02.
- **The SLO per tier, per workload class.** A target on each SLI (e.g., "99 % of sessions complete within 60 s", "p95 time-to-first-tool ≤ 3 s"), grounded in user expectation rather than pulled from the current week's measurement. The targets are paved-road defaults; teams can tighten them, but not loosen them below the platform minimum without an architecture review. Chapter 02.
- **The error-budget policy.** The document that says *what happens when the error budget burns* — feature freeze above a burn rate, mandatory reliability work above another, paging a specific rota at a third. This is the artefact that stream-aligned teams opt into when they adopt the paved road; without it, SLOs are just numbers on a dashboard. Chapter 03.
- **The chaos-drill programme.** A quarterly cadence of practised failures against the four agent dependency classes: model-provider outage (a region or a specific family goes down), MCP-server flake (a tool server returns 503s or hangs), sandbox exhaustion (code execution fleet saturates), eval-infra degradation (the eval plane cannot run gates). The programme measures *platform response*, not whether the dependency recovers. Chapter 04.
- **The incident-response playbooks.** One per agent-specific failure mode: runaway agent loop, context-window exhaustion, prompt-injection containment, tool-cascade meltdown. Each names detection signals, containment levers, mitigation steps, recovery criteria, and the post-incident loop. Chapter 05.
- **The post-incident-to-regression-fixture pipeline.** The mechanism that takes the traces captured during an incident and promotes the ones that reproduce the failure into the eval plane's catalogue (mod-407 chapter 04's trace-to-catalogue promotion). The next deploy that would reintroduce the failure is blocked by the gate. Chapter 06.
- **The reliability-posture dashboards and the exec-review cadence.** The aggregated view of SLO attainment, burn rates, open incidents, chaos-drill results, and regression-fixture coverage, reported on a cadence the leadership plane (mod-413) consumes.
- **The on-call rota, the runbooks, and the pager contract.** Who is paged for which signal, what their first five minutes look like, when they escalate. The rota is a platform operation; the runbook is what makes it survivable.

The plane does **not** own:

- **The semantic contract.** [OpenTelemetry Semantic Conventions for Generative AI](https://opentelemetry.io/docs/specs/semconv/gen-ai/) and the platform's `platform.*` extension namespace belong to the observability plane (mod-408). The SRE plane consumes the vocabulary; it does not pick it.
- **The eval methodology.** What a regression *is* belongs to the evaluation plane (mod-407). The SRE plane promotes incident traces into the catalogue; it does not decide the pass/fail rubric.
- **Agent internal correctness.** Whether an agent's prompt is good, whether the model choice is right, whether the loop is well-structured — those belong to `senior-agentic-ai-engineer` (level 40) and `ai-eval-engineer` (level 30). The SRE plane owns reliability signals; the product-side engineers own the agent's quality.
- **The guardrail policy.** The safety plane (mod-411) decides what a refusal must look like. The SRE plane measures how often refusals fire and alerts on drift; it does not author the rule.
- **Capacity planning as a long-horizon forecast.** That is `ai-infra-inference-engineer` (level 35) and the model-gateway owners' work with the capacity team. The SRE plane owns short-horizon reliability; long-horizon capacity is a separate discipline.
- **The paved-road SDK's resilience code.** Circuit breakers, retries, bulkheads, and deadline propagation live in mod-409's A2A scaffold and mod-412's SDK. The SRE plane consumes the signals those primitives emit; it does not write the primitives.
- **The governance cadence over error-budget-policy exceptions.** Leadership (mod-413) owns the exception register and the signing authority. The SRE plane proposes the policy; leadership approves and audits exceptions.

Drift off this slice and the plane becomes "the people who wake up when something is broken", which is just on-call and does not actually run the fleet. Scope policing is a feature of this module.

## Why a plane and not a per-team on-call

A few concrete questions that are hard to answer without a reliability plane and trivial with one.

- **"The writer agent's p95 time-to-final-answer doubled this week. Is this a user-visible regression or noise?"** Without a defined SLI and a baseline, every team's answer is "the dashboard looks higher"; the user may or may not notice. With an SLI and a pinned SLO target, the answer is "the SLO burns at 2.3× the budget rate; we have five days before the quarterly budget is spent"; now there is a decision to make and a budget to spend making it. Chapter 02.
- **"Should we ship the new prompt that lifts quality by 2 % but regresses latency by 500 ms?"** Without an error-budget policy, the answer is a meeting. With one, the answer is "latency is inside budget this quarter; ship with a monitoring window" or "latency is over budget; freeze non-reliability work". Chapter 03.
- **"The provider's `us-east-1` region just went offline. How does the model gateway behave?"** Without a chaos drill, the first time anyone finds out is the incident that gets paged. With a drill, the platform has a measured time-to-detect, time-to-mitigate, and a documented failover posture. Chapter 04.
- **"An agent entered a tool-call loop and spent $400 of a tenant's budget in five minutes. How was it contained?"** Without a playbook, the answer is "someone noticed in Slack and killed the deployment". With one, the answer is "the loop-detector tripped at turn 20, the model-gateway cost circuit tripped at $50, the runaway-loop playbook paged the on-call, who cancelled the session and reduced the per-session turn cap for that agent". Chapter 05.
- **"We fixed the runaway-loop bug in agent X on 2026-09-04. Have we prevented the same shape of failure in any other agent?"** Without a regression-fixture pipeline, each team re-discovers the lesson. With one, the fixture is in the mod-407 catalogue and the gate blocks the next agent that would ship the same shape. Chapter 06.
- **"Our highest-tier tenant's agent SLO depends on a provider we cannot control. What is our contract?"** Without an explicit error-budget allocation between the platform's budget and the provider's budget, the tenant assumes the platform is 100 % responsible. With one, the SLA is honest, the budget is split, and the provider's outage does not consume the platform's entire reliability margin. Chapter 03.

These are the capabilities the plane delivers. They separate "we have alerts" from "we run the fleet". Chapters 02–06 build each.

## What makes SRE for agents different from SRE for web services

The Google SRE Book was written for request/response services with p99s in milliseconds and binary outcomes. Three things about agents make the vanilla model insufficient without extensions:

- **A single "request" can be minutes long, multi-turn, and non-deterministic.** A session-completion SLI is not a 200/500 count — it is a question about a stateful, often human-in-the-loop interaction. Latency is bimodal (sub-second for cached completions, tens of seconds for a tool-heavy turn). Chapter 02 defines the SLIs with this shape in mind, including the "time to first useful output" (*time-to-first-tool* and streaming's first-token latency) split from the end-to-end time.
- **Quality is a reliability signal, not just a product signal.** A web service that returns a 200 with wrong JSON is a bug; a web service that returns 200 with the right JSON 99 % of the time is on target. An agent that completes 99 % of sessions but hallucinates in 10 % of them is *not* on target, no matter how healthy the latency graph looks. The platform needs SLIs on *hallucination rate* and *safety-refusal rate* that read from the eval plane (mod-407) and the safety plane (mod-411), not from HTTP status. Chapter 02.
- **Failure modes are agent-specific and expensive.** A runaway loop consumes model budget — real money — at wall-clock pace. A context-window exhaustion silently corrupts answer quality. A prompt-injection may exfiltrate data through tool calls. A tool cascade turns one slow MCP server into a fleet-wide saturation. The standard SRE playbook ("restart it, look at the graphs") does not contain these; chapter 05 builds agent-specific playbooks.

The SRE book's primitives — SLIs, SLOs, error budgets, chaos engineering, blameless post-mortems, runbooks, on-call rotations — still apply. The specific *measurements* and *failure modes* that go into them are what this module authors.

## The invariant: `ReliabilityRecord`

Pin the record shape before writing the plane. Every platform decision the SRE plane makes — SLO measurement, burn-rate alert, chaos-drill run, incident response — produces one `ReliabilityRecord`, keyed to the mod-408 `trace_id` where applicable. The record is invariant across incident types, tenants, and workload classes. Every chapter adds a field.

```json
{
  "record_id": "rel_01HWX3K2M...",
  "record_type": "slo_burn_alert",
  "emitted_at": "2026-11-06T14:22:03Z",
  "tenant_id": "acme-financial",
  "workload": {
    "agent_id": "writer-v1",
    "agent_revision": "git:abcd1234",
    "resilience_tier": "T1",
    "workload_class": "interactive-chat"
  },
  "sli": {
    "name": "time_to_final_answer_p95",
    "window": "rolling_28d",
    "current_value_ms": 11400,
    "target_ms": 10000,
    "measurement_source": "otel:gen_ai.client.operation.duration",
    "query_hash": "sha256:7f3c..."
  },
  "slo": {
    "name": "writer-v1-interactive-latency",
    "target": 0.95,
    "window": "28d",
    "budget_remaining_pct": 0.18,
    "burn_rate_1h": 6.4,
    "burn_rate_6h": 2.9
  },
  "trace_refs": [
    "trace:4bf92f3577b34da6a3ce929d0e0e4736",
    "trace:7c28a1ff4891bd44b1dcef2c90a8e133"
  ],
  "incident_ref": null,
  "chaos_drill_ref": null,
  "fixture_promotion_ref": null,
  "outcome": {
    "action_taken": "notify_feature_freeze_candidate",
    "actor": "slo-burn-alert:page-l1",
    "policy_doc": "error-budget-policy@2026-07-15"
  }
}
```

Six invariants the record enforces:

1. **Every reliability decision names the SLI, the SLO, and the policy document it came from.** Audit answers "why did we freeze this team's feature work" from the record alone. Chapter 03.
2. **Every alert ties back to `trace_id` samples.** No alert fires without exhibits; debugging never starts with "whose dashboards do I need to join?". Chapter 02 and cross-reference mod-408 chapter 05.
3. **The burn rate is on the record.** The SRE Workbook's burn-rate alert model — fast-burn (page) vs. slow-burn (ticket) — is machine-readable. Chapter 02.
4. **Chaos-drill outputs and incident outputs reuse the same record shape.** The fleet view is one query, not three. Chapters 04 and 05.
5. **Fixture promotion is a traceable action.** An incident that fed a fixture into the eval plane records the `fixture_promotion_ref`. The next failure attributed to the same shape is a conformance failure of the regression gate. Chapter 06.
6. **Workload metadata is on every record.** Tier, class, agent revision. The quarterly reliability review slices by any of them without joining to the deployment history. Chapter 03.

Chapters 02–06 build the pieces that produce this record.

## The three obligations

Pin the plane to three non-negotiable obligations. Everything in chapters 02–06 is a decision about how to deliver them.

1. **"Every agent workload on the paved road has a declared SLI, a target SLO, and a policy for what happens when the budget burns."** An agent that ships without an SLO is a reliability bet with no stakeholders; the plane does not accept them onto the paved road. Chapters 02 and 03.
2. **"Every failure class the fleet has seen — in incident or in a drill — has a playbook, and every playbook has been exercised at least once per quarter against a chaos drill or a real incident."** Playbooks that live in a document and have never been run are not playbooks; they are hopes. Chapters 04 and 05.
3. **"Every incident whose root cause is reproducible as an eval fixture is promoted into the eval plane's catalogue before the post-mortem is signed off."** An incident that does not change the regression gate is a lesson the next engineer will have to learn again. Chapter 06.

A plane that cannot defend all three is an on-call rota with a dashboard, not a reliability engineering plane.

## Where the plane physically sits

A typical deployment shape:

- **An SLI definitions registry.** A versioned YAML or JSON catalogue (checked into a platform repo) listing every SLI by name, formula, measurement source, and evaluation window. The source of truth for the dashboards and the burn-rate alerts. Chapter 02.
- **An SLO catalogue, keyed by `(workload_class, tier)`.** The targets, the windows, the burn-rate thresholds, the actions. The paved-road default plus per-tenant overrides. Chapter 02 and 03.
- **The burn-rate alerting pipeline.** A rules engine on top of the metrics backend ([Prometheus](https://prometheus.io/docs/alerting/latest/alertmanager/) alertmanager, [Grafana Alerting](https://grafana.com/docs/grafana/latest/alerting/), or a native cloud-provider equivalent) that reads the SLI series, applies the multi-window burn-rate formulas from the SRE Workbook's [Alerting on SLOs chapter](https://sre.google/workbook/alerting-on-slos/), and pages or tickets per the policy.
- **The error-budget policy document.** A plain-English document (versioned) with the five actions on burn and the exception register pointer. The thing a team adopts when they go on the paved road. Chapter 03.
- **The chaos-drill harness.** A combination of [Chaos Mesh](https://chaos-mesh.org/) or [LitmusChaos](https://litmuschaos.io/) (for Kubernetes-level faults), [Toxiproxy](https://github.com/Shopify/toxiproxy) (for network-level faults between agents and dependencies), and a per-drill fault injector that simulates the agent-specific scenarios (fake provider 503s, MCP server latency, sandbox fork-bomb). The drill book and the scorecard. Chapter 04.
- **The incident-management system.** Whatever the organisation uses — [PagerDuty](https://www.pagerduty.com/), [Opsgenie](https://www.atlassian.com/software/opsgenie), [Grafana Incident](https://grafana.com/products/cloud/incident/), [FireHydrant](https://firehydrant.com/) — plus the runbook library, plus the post-incident template. Chapter 05.
- **The fixture-promotion pipeline.** A tool (CLI or CI step) that takes one or more `trace_id`s, pulls the traces from mod-408's cold store, strips PII per the capture-time redaction policy, and emits a mod-407 catalogue entry under the `incident/<incident_id>` namespace. Chapter 06.
- **The reliability-posture dashboard and the exec-review slide.** A Grafana or Looker dashboard summarising SLO attainment, open incidents, drill cadence, and fixture coverage, feeding into the leadership-plane (mod-413) operating cadence.

If you cannot draw this topology on a whiteboard and name the box every reliability decision flows through, chapter 02 is the thing to read next.

## Where this plane sits in the track

The SRE plane is the operational closure over everything that came before. Each prior module left a signal or a lever:

- **mod-402 (runtime)** — turn-level durability, session state, replay primitives. The SRE plane consumes "did the session complete" from the runtime's completion events.
- **mod-403 (MCP gateway)** — tool catalogue health, per-tool latency and error rate. Chaos drills against MCP servers (chapter 04) exercise the gateway.
- **mod-404 (sandbox)** — code-execution capacity, fork-bomb and resource-exhaustion defences. Sandbox saturation is one of the four drill classes. Chapter 04.
- **mod-405 (policy plane)** — deny/permit decisions as span events. The *safety-refusal rate* SLI (chapter 02) reads from these.
- **mod-406 (model gateway)** — routing, token usage, cost signals. The model-provider outage drill (chapter 04) exercises the gateway's failover.
- **mod-407 (eval plane)** — fixture catalogue, regression gate. The post-incident loop (chapter 06) is the primary way the catalogue grows.
- **mod-408 (observability plane)** — the semantic contract and trace propagation. Every SLI formula in chapter 02 is a query over `gen_ai.*` and `platform.*` attributes.
- **mod-409 (inter-agent plane)** — circuit-breaker, retry, bulkhead signals. The tool-cascade meltdown playbook (chapter 05) consumes these as its primary detection signals.

The downstream handoffs:

- **mod-411 (safety plane)** — emits guardrail verdicts. The SRE plane reads the verdict rate as the *safety-refusal rate* SLI and includes safety-related incidents in the playbook library.
- **mod-412 (paved-road SDK)** — packages the SLI emitters (chapter 02's attribute shim), the burn-rate hook (chapter 03's policy integration point), and the incident-reporting helper (chapter 05) so stream-aligned teams opt into the plane by importing the library.
- **mod-413 (platform leadership)** — consumes the reliability-posture report, owns the exception register for error-budget-policy overrides, and runs the operating cadence that pairs this plane with product and eval.

## Summary

- The SRE plane owns the SLI catalogue, SLOs per tier and workload class, the error-budget policy, the chaos-drill programme, the incident-response playbooks, the post-incident fixture pipeline, and the reliability-posture report. It does not own the semantic contract (mod-408), the eval rubric (mod-407), the agent's internal correctness, the guardrail policy (mod-411), long-horizon capacity, the SDK's resilience code (mod-409 / mod-412), or governance signing authority (mod-413).
- Agent SRE extends the Google SRE model with multi-turn, non-deterministic, quality-as-reliability signals and agent-specific failure modes (runaway loops, context-window exhaustion, prompt injection, tool cascade). Chapters 02–05 build those extensions on the SRE book's foundation.
- Every reliability decision produces a `ReliabilityRecord` keyed on `trace_id` where applicable, carrying SLI, SLO, burn rate, policy reference, and the action taken. Chapters 02–06 add fields.
- Three non-negotiable obligations: every paved-road workload has a declared SLO and policy; every known failure class has an exercised playbook; every reproducible incident becomes a regression fixture.

The next chapter opens the first piece — the SLI catalogue for agent workloads, with each of the five indicators defined as a query over mod-408's semantic layer.
