# SLIs and SLOs for Agent Workloads

Chapter 01 named what the plane owns. This chapter builds the first piece: the catalogue of service-level indicators the plane measures, the targets it sets on them, and the alerting shape that turns "the graph looks bad" into "we are burning budget 6× faster than planned".

The grounding is the [Google SRE Book, chapter 4 — Service Level Objectives](https://sre.google/sre-book/service-level-objectives/) and the [Google SRE Workbook, chapters 2 (Implementing SLOs) and 5 (Alerting on SLOs)](https://sre.google/workbook/alerting-on-slos/). The specialisation is agents: a workload that is multi-turn, non-deterministic, quality-sensitive, and whose "healthy" can be measured both at the HTTP seam and at the reasoning-and-output seam. A latency graph alone is a lie by omission — an agent can be fast and wrong, or slow and right, and only *both* dimensions describe whether users are being served.

## The SRE vocabulary, in one page

Three terms the rest of the chapter uses literally.

- **SLI (service-level indicator)** — a quantitative measure of some aspect of the service. A fraction, a latency, a rate. "95th percentile of time-to-final-answer in the last 28 days".
- **SLO (service-level objective)** — a target for an SLI, typically expressed as "SLI ≤ X" or "SLI ≥ X" over a window. "p95 time-to-final-answer ≤ 10 s, measured over a rolling 28 days".
- **Error budget** — `1 − SLO` worth of allowable badness. If the SLO is 99 % completion, the error budget is 1 %. Burning it is the signal that reliability work has fallen behind.

The agent additions this chapter formalises:

- **Workload class** — the shape of the agent, not the agent itself. `interactive-chat`, `background-task`, `batch-research`, `code-execution-loop`. The SLO targets vary by class — an interactive chat agent needs a tight time-to-first-tool; a batch research agent needs a generous TTFA but a strict session-completion rate.
- **Measurement source** — the exact attribute (or derived attribute) in the mod-408 semantic layer that produces the SLI series. Pinning the source is how an SLI stops being "whatever the graph says today" and becomes a contract.
- **Window** — a rolling window (28 days is common) or a calendar quarter. The window shapes the budget; too short a window makes transient faults into budget disasters, too long a window hides sustained regressions.
- **Burn rate** — how fast the error budget is being consumed relative to its steady-state rate. Chapter covered in detail below.

## The five SLIs the Agent Platform Engineer must own

The track's spec names five. Each gets a definition, a measurement source (from mod-408's semantic layer), a default window, and a warning on common misuse.

### SLI 1 — Session-completion rate

The fraction of sessions that reach a terminal success state, as opposed to crashing, timing out, being cancelled by the user mid-stream, or stalling in an unrecoverable state.

- **Measurement source.** The runtime (mod-402) emits a `platform.session.lifecycle` event at every state transition. The terminal states are `completed`, `user_cancelled`, `platform_cancelled`, `failed_recoverable`, `failed_terminal`. The SLI is `count(terminal_state="completed") / count(terminal_state IN completed|user_cancelled|platform_cancelled|failed_recoverable|failed_terminal)`.
  - `user_cancelled` *counts* toward the denominator but as a non-success — the user getting bored and closing the tab is a UX signal even if the platform did nothing wrong. Some teams exclude it; document whichever choice you make.
  - `failed_recoverable` is a deliberate exit where the agent surfaced an "I cannot do this" to the user without hallucinating. It counts as a non-success — users do not care that the failure was clean.
- **Default target.** Interactive-chat: 99.0 % over 28 days. Background-task: 99.5 %. Batch-research: 99.9 %.
- **Window.** 28 days rolling is the paved-road default. The reasoning: a 7-day window makes a Friday-afternoon provider blip into a budget emergency; a 90-day window lets a quiet regression accumulate until it hits the quarterly review.
- **Misuse warning.** The denominator *must* include terminal failures and stalls. A fleet-wide bug that hangs sessions indefinitely will show "100 % completion of everything that completed" — the SLI looks green while the actual completion-event volume craters. Use `count(sessions_started last window) − count(sessions_completed last window) − count(sessions_still_in_progress now)` as a cross-check and alert when the gap grows beyond a threshold.

### SLI 2 — Time-to-first-tool (TTFT, or first useful action)

The wall-clock time from session start (or turn start, on follow-up turns) until the agent emits its first tool call, first token in a streamed response, or first side-effecting action. The "something is happening" signal.

- **Measurement source.** `gen_ai.client.operation.duration` from the first chat completion's streaming start event; or the first `gen_ai.tool.call` span's `start_time` minus the session's `start_time`. Both as a histogram metric tagged by `workload_class`, `agent_id`, and `tenant_id`.
- **Default target.** Interactive-chat: p95 ≤ 3 s, p99 ≤ 5 s. Background-task: p95 ≤ 10 s. Batch-research: not a primary SLI; TTFA is enough.
- **Window.** 28 days rolling.
- **Misuse warning.** TTFT is a *user-perceived* latency signal — it should include the queueing time before the agent gets a model gateway slot, not just the gateway's own latency. Measure from when the runtime accepted the turn, not from when the model gateway started the call. The bug people make is measuring TTFT from the gateway's side; the gateway can be fast while the runtime queues a user's request behind a 20-second cold start.

### SLI 3 — Time-to-final-answer (TTFA)

The wall-clock time from session or turn start to the terminal response event. End-to-end latency, including the full tool loop.

- **Measurement source.** The session's terminal event's `timestamp` minus the session's `start_time`. On a per-turn basis, the turn's terminal event minus the turn's start. Histogram metric tagged as per TTFT.
- **Default target.** Interactive-chat: p95 ≤ 10 s, p99 ≤ 30 s. Background-task: p95 ≤ 60 s. Batch-research: p95 ≤ 10 min (the right number depends on the use case; this is a paved-road suggestion).
- **Window.** 28 days rolling.
- **Misuse warning.** TTFA is heavy-tailed for agent workloads. A single "a user asked the research agent to synthesise a report" session that takes 15 minutes will skew the p99 if the fleet is low-volume. Report p95 as the primary; p99 as a secondary; include the heavy-tail cutoff (what fraction of sessions took longer than 2× the p95) as a watch metric.
  - For *streaming* interfaces, the TTFA definition varies: is it time to the first useful token, time to the last token, or time to the user clicking "stop"? Pin the definition in the SLI catalogue and document which event bounds the measurement.

### SLI 4 — Hallucination rate

The fraction of agent outputs that fail a factuality check. The quality-as-reliability signal.

- **Measurement source.** This SLI is *not* a direct trace attribute; it is a derived signal from the evaluation plane (mod-407). Two possible sources, both legitimate:
  - **Offline eval replay** — a sampled fraction of production traces (per mod-407 chapter 04) replayed through a judge pipeline, with the judge's verdict emitted as `platform.eval.factuality_pass`. The SLI is `count(pass=false) / count(total_judged)` over the window. Sampling fraction pinned in the catalogue; typical range 1 %–5 % for high-volume workloads.
  - **Online judge** — a low-latency judge (often a smaller model) that samples and scores production outputs in real time, emitting `platform.eval.factuality_pass` as a span event on the chat span. More expensive but no replay lag.
- **Default target.** Depends heavily on workload. Factuality-critical workloads (research summarisation, finance): hallucination rate ≤ 2 % with a 95 % lower confidence bound reported. Creative workloads (brainstorming, drafting): the SLI may not apply — document the opt-out.
- **Window.** 28 days rolling, with a minimum sample count (e.g., 1000 judged outputs) before the SLI is trusted. Below the floor, report "insufficient data".
- **Misuse warning.** The judge is a model and the judge is wrong. The `ai-eval-engineer` role (level 30) owns judge calibration — the agreement rate between the judge and human raters. The SRE plane should not run a hallucination SLO off an uncalibrated judge. The pattern is: the eval plane publishes a calibrated judge with a human-agreement rate, the SRE plane consumes its verdicts as an SLI, and the judge is on a change-control cadence (quarterly re-calibration minimum). See mod-407 chapter 06 for the handoff.
  - The SLI is *platform reliability-adjacent*, not a product-quality verdict. It tells you "is factuality drifting against the baseline". Product-quality judgement is `ai-eval-engineer` and `senior-agentic-ai-engineer` territory.

### SLI 5 — Safety-refusal rate

The fraction of sessions in which the safety plane (mod-411) emitted a refusal verdict on at least one agent output. Includes both legitimate refusals ("I cannot help with that") and false-positive refusals ("the agent refused a benign request").

- **Measurement source.** `platform.safety.verdict` span events emitted on every chat span by the safety plane's guardrail hook. The SLI is split into two numbers:
  - **Refusal rate** — `count(verdict="refuse") / count(verdicts_emitted)`.
  - **False-positive refusal rate** — refusals that were later judged (by an offline eval pass or a user feedback signal) as incorrect, over total refusals. This is the operationally important number; a system that refuses every request has a 100 % refusal rate and 100 % false-positive rate.
- **Default target.** The target is *not* "refusal rate ≤ X" — the correct refusal rate depends entirely on the workload and the tenant. The target is "false-positive refusal rate ≤ 10 %" (configurable per workload class) and "refusal rate stays within ±30 % of the 28-day trailing average" (drift alert).
- **Window.** 28 days rolling for the drift; the false-positive rate uses the eval plane's 28-day trailing sample.
- **Misuse warning.** A spike in refusal rate can be a sign of good behaviour (the model correctly refused a new attack pattern), a sign of bad behaviour (over-eager refusal of a legitimate class of query), or a sign of upstream change (a new safety policy deployed). Alert on *drift*, not on absolute level; pair the alert with a mandatory human review step before firing a "safety plane misconfigured" page.
  - This SLI pairs tightly with mod-411 (safety plane). The SRE plane measures and alerts on drift; the safety plane owns the policy. Cross-reference mod-411 chapter on guardrail event emission.

## SLI catalogue: the artefact

The five SLIs above are the paved-road defaults. The actual list lives in a versioned catalogue the plane owns. A paved-road schema:

```yaml
# slis/time_to_final_answer.yaml
name: time_to_final_answer
version: "1.3"
description: >
  End-to-end wall-clock from session (or turn) start to terminal response event.
units: milliseconds
measurement:
  source: otel
  metric: gen_ai.client.operation.duration
  derivation: >
    histogram(gen_ai.client.operation.duration{
      span.name="session",
      workload_class=~"$WORKLOAD_CLASS",
      agent_id="$AGENT_ID"
    })
  reductions:
    - percentile_95
    - percentile_99
    - heavy_tail_fraction  # fraction above 2 * p95
labels:
  required: [tenant_id, agent_id, workload_class, agent_revision]
  optional: [session_mode, streaming, cache_partition]
window:
  type: rolling
  size: 28d
  minimum_sample_count: 1000
owners:
  plane: sre
  co-owner: senior-agentic-ai-engineer
change_control:
  requires_review: [sre, ai-eval-engineer]
  changelog_entry_required: true
```

Each SLI is one file; the directory is the catalogue. The paved-road SDK (mod-412) ships the attribute emitters; this catalogue describes what the SRE plane reads.

## Turning SLIs into SLOs

An SLI is a number. An SLO adds the target, the window, and — crucially — the error budget. The SLO document is what teams sign.

A paved-road template:

```yaml
# slos/writer-v1-interactive.yaml
name: writer-v1-interactive
version: "2.1"
agent_id: writer-v1
workload_class: interactive-chat
tier: T1
window: 28d
objectives:
  - sli: session_completion_rate
    target: 0.99
    direction: >=
  - sli: time_to_first_tool_p95
    target_ms: 3000
    direction: <=
  - sli: time_to_final_answer_p95
    target_ms: 10000
    direction: <=
  - sli: hallucination_rate
    target: 0.02
    direction: <=
    minimum_sample_count: 1000
    judge_calibration_card: judge-cards/factuality-v4.yaml
  - sli: safety_refusal_drift
    target_delta: 0.30
    direction: <=
    pairs_with: safety-refusal-rate-28d-baseline
error_budget_policy: error-budget-policy@v4
opt_in_date: "2026-08-01"
owning_team: writer-agents
on_call_rota: writer-agents-primary
```

Every paved-road workload has one of these. The absence of an SLO document is the signal that the workload is not yet on the paved road.

## Burn-rate alerting (the SRE Workbook model)

The naive alert — "page when the SLO is exceeded for the last hour" — is noisy and late. Noisy, because every transient blip pages; late, because once you have exceeded the SLO you have already spent budget. The [SRE Workbook's alerting chapter](https://sre.google/workbook/alerting-on-slos/) formalises the solution: alert on *burn rate* across multiple windows.

The idea: a 1-hour burn rate shows a fast local disaster; a 6-hour burn rate filters out the 20-minute blip that would have paged in the naive model; requiring both shows a real, sustained burn. Multi-window alerts have known false-alarm rates and known page-coverage rates in the Workbook's table.

The paved-road alerting shape for every SLO:

- **Fast-burn page.** `burn_rate_1h ≥ 14.4 AND burn_rate_5m ≥ 14.4`. A 14.4× burn rate consumes 2 % of a 28-day budget in one hour. Pages the primary on-call.
- **Slow-burn ticket.** `burn_rate_6h ≥ 6 AND burn_rate_30m ≥ 6`. Consumes 5 % of the budget in 6 hours. Opens an incident ticket; no page.
- **Budget-exhausted notification.** `budget_remaining ≤ 0.10`. Posts to the owning team's Slack channel; triggers the error-budget policy's "slow down" actions (chapter 03).

Each threshold has a false-alarm analysis in the Workbook; the paved-road defaults above come from the "99 % availability over 30 days" row of the Workbook's table. Teams with tighter or looser SLOs adjust proportionally.

### A concrete PromQL example

Session-completion rate as a Prometheus recording rule plus burn-rate alert:

```yaml
# recording rules
groups:
- name: sli_session_completion
  interval: 1m
  rules:
  - record: sli:session_completion:rate1m
    expr: |
      (
        sum by (tenant_id, agent_id, workload_class) (
          rate(platform_session_terminal_total{terminal_state="completed"}[1m])
        )
      ) /
      (
        sum by (tenant_id, agent_id, workload_class) (
          rate(platform_session_terminal_total[1m])
        )
      )
  - record: sli:session_completion:badness1m
    expr: 1 - sli:session_completion:rate1m

# multi-window burn-rate alert
- alert: WriterV1SessionCompletionBurnRateFast
  expr: |
    (
      max_over_time(sli:session_completion:badness1m{agent_id="writer-v1"}[1h])
        / 0.01  # 1 - 0.99 SLO
      >= 14.4
    )
    and
    (
      max_over_time(sli:session_completion:badness1m{agent_id="writer-v1"}[5m])
        / 0.01
      >= 14.4
    )
  for: 2m
  labels:
    severity: page
    policy: error-budget-policy@v4
  annotations:
    summary: writer-v1 session-completion SLO burning at ≥14.4x
    runbook: https://runbooks.platform.local/session-completion-burn
```

The exact PromQL depends on your metric naming; the pattern (badness rate / `1 - SLO` compared against a multiplier) is portable to Grafana Alerting, Datadog monitors, or any multi-window alerting shape.

<!-- needs-research: confirm the Workbook's exact burn-rate thresholds for a 28-day window; the published table uses 30-day as the anchor and the mapping to 28-day is a trivial arithmetic adjustment that is worth stating correctly in a production document. -->

## Workload classes and tier defaults

Not every agent is a chat agent. The paved-road catalogue defines four workload classes and three tiers; the SLO targets live at their intersection.

| Workload class       | Example                                   | Session-completion | TTFT p95 | TTFA p95     | Notes                                                            |
|----------------------|-------------------------------------------|--------------------|----------|--------------|------------------------------------------------------------------|
| `interactive-chat`   | User-facing chat router                   | 99.0 %             | 3 s      | 10 s         | Streaming is standard; include first-token latency as TTFT.      |
| `background-task`    | Triggered summary, scheduled briefing     | 99.5 %             | 10 s     | 60 s         | No user watching; TTFT is a cost signal, not a UX signal.        |
| `batch-research`     | Multi-step research synthesis             | 99.9 %             | n/a      | 10 min       | Completion matters more than latency; heavy-tail cutoff is key.  |
| `code-execution-loop`| Agent that writes and runs code           | 99.0 %             | 5 s      | 5 min        | Pair with sandbox saturation SLI (chapter 04's drill feeds this).|

The tier (T1 / T2 / T3 — same vocabulary as mod-409 chapter 05's resilience tier) controls how tight the SLOs are:

- **T1 — user-facing, tight.** The default targets above apply exactly.
- **T2 — standard.** 10 % looser on latency, 0.5 % looser on completion.
- **T3 — long-running.** Latency SLOs may be dropped; completion and quality SLOs remain. Budgets are larger.

A team cannot *tighten* above T1 unless they prove the agent can meet it (and take the pager); they cannot *loosen* below T3 without an exec review. The tier assignment is a line in the SLO document and a pointer to the policy.

## SLO review cadence

An SLO is a contract, not a graph. The cadence:

- **Weekly.** The owning team reviews their burn rate and open budget-remaining. A team with ≥ 50 % budget remaining at week 2 of a 28-day window is fine; a team at 0 % by day 10 has a problem the on-call must have been aware of.
- **Monthly.** The SRE plane reports fleet-wide SLO attainment to the leadership plane (mod-413). Workloads consistently missing their SLO are candidates for tier downgrade or product-level intervention.
- **Quarterly.** SLO targets are re-reviewed against user expectation. Targets that have been met for four consecutive quarters are candidates for tightening; targets consistently missed are candidates for loosening (with the conversation about why).

The cadence lands in mod-413's operating-rhythm. The SRE plane produces the inputs.

## Common anti-patterns

Named so the reader recognises them.

- **Measuring what you can, not what matters.** HTTP status codes are easy to count; session completion is a derived signal. A fleet that alerts on "chat endpoint 5xx rate" and nothing else will not notice the day the model gateway serves 200 OK responses with an error payload. The SLIs that matter for agents are session-level and quality-level; the HTTP seam is incidental.
- **Burn-rate alerting without a burn-rate policy.** Pages that fire without a documented response ("feature freeze? paging the VP? nothing?") train the on-call to silence them. Chapter 03 is the paired artefact; an SLO without a policy is decorative.
- **SLIs defined as a graph.** "The thing the dashboard shows" is not a definition. The SLI is a formula over named attributes. The dashboard reads the formula; the formula does not read the dashboard.
- **Uncalibrated-judge-driven hallucination SLOs.** A judge that disagrees with humans 30 % of the time will burn the hallucination budget on signal noise. Partner with `ai-eval-engineer` (mod-407 chapter 06) on calibration cards before standing up the SLO.
- **Safety-refusal rate as a target to minimise.** A platform that treats "refusals are bad" as policy will push teams to turn guardrails down. The target is drift against a pinned baseline, not absolute level. Cross-reference mod-411.
- **Per-tenant SLOs with no tenant context on the metric.** A per-tenant breakdown requires `tenant_id` on the metric; teams that strip it to "reduce cardinality" lose the ability to report tenant SLAs. Keep the label; reduce cardinality in the tenant-routed exporter (mod-408 chapter 04).

## Summary

- Five SLIs define the paved-road reliability surface for agent workloads: *session-completion rate*, *time-to-first-tool*, *time-to-final-answer*, *hallucination rate*, *safety-refusal rate*. Each has a pinned formula, a measurement source in mod-408's semantic layer, and a default target per workload class and tier.
- SLOs are expressed as targets over a window (28 days paved-road default). The SLO document — one per workload — declares the targets, the error-budget policy pointer, and the owning team.
- Burn-rate alerting follows the SRE Workbook's multi-window pattern (fast-burn page, slow-burn ticket, budget-exhausted notification). PromQL and Alertmanager are the paved-road implementation.
- Workload classes (`interactive-chat`, `background-task`, `batch-research`, `code-execution-loop`) and tiers (T1 / T2 / T3) structure the SLO defaults. Teams pick an intersection; the plane publishes the defaults.
- Hallucination SLO depends on a calibrated judge; safety-refusal SLO is drift-based, not level-based; both are quality-as-reliability signals that extend the SRE book's web-service model.
- The SLI catalogue is an artefact — versioned YAML, change-controlled, change-logged. The dashboards read from it; teams do not re-derive.

The next chapter turns the SLO into an operating contract — the error-budget policy that stream-aligned teams opt into when they adopt the paved road, and the actions the policy triggers when the budget burns.
