# exercise-01: SLI and SLO Catalogue for Agent Workloads

**Estimated effort:** 3 hours · **Output:** a versioned SLI catalogue (YAML) for one workload, matching Prometheus recording rules (or equivalent) with sample data, a Grafana dashboard JSON (or equivalent) with four panels, two multi-window burn-rate alerts, a signed SLO document, and a short README (~1 page) · **Pairs with:** chapter [02](../02-slis-and-slos-for-agent-workloads.md); feeds exercises 02, 03, and project 403.

## Objective

Translate chapter 02's abstract SLI catalogue into a concrete, runnable artefact for one workload. You will author the catalogue entries, implement the measurement in a metrics backend of your choice, build the dashboard a team actually looks at, wire the SRE Workbook's multi-window burn-rate alerts, and sign the SLO document that binds the workload to the paved-road operating contract.

Chapter 02 is the theory; this exercise is the forcing function that reveals which parts you can actually express as a query and which parts are still waving hands. The platform engineer who has never written a burn-rate alert will reach for a 1-minute threshold on the first production incident and learn the lesson at 3 am.

## Prerequisites

- You have read chapters [01](../01-the-sre-plane-and-what-it-owns.md) and [02](../02-slis-and-slos-for-agent-workloads.md).
- You have [Prometheus](https://prometheus.io/) + [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/) + [Grafana](https://grafana.com/docs/grafana/latest/) available locally (compose file, `kind` cluster, or hosted equivalent) — OR a comparable metrics backend ([Grafana Mimir](https://grafana.com/docs/mimir/latest/), [VictoriaMetrics](https://docs.victoriametrics.com/), [Datadog](https://docs.datadoghq.com/metrics/), cloud-provider native). The paved-road assumption is Prometheus; adapt if your stack differs and note the choice in the README.
- You have a stub agent that emits the mod-408 semantic attributes on span-level — either from [exercise 02 of mod-408](../../mod-408-observability-and-opentelemetry-for-agents/exercises/exercise-02-cross-plane-distributed-trace-propagation.md) or a 50-LOC Python stub that emits `session_started` / `session_terminal` events, a histogram for `gen_ai.client.operation.duration`, and `platform.safety.verdict` + `platform.eval.factuality_pass` events.
- You have an OTel collector (or a shim) translating the span attributes into Prometheus metrics. Cross-reference mod-408 chapter 04.

## Problem statement

You own the SLO story for a single workload: `writer-v1`, an interactive-chat agent running on the paved road for the fictional tenant `acme-financial`. The agent helps users draft long-form documents; it uses a research tool and a drafts tool. A product manager has just signed off on a target SLA with acme-financial and now asks you to "set up the SLOs". Your output is:

1. A versioned SLI catalogue with at least three of the five paved-road SLIs populated.
2. Recording rules (or equivalent) that produce the SLI series from stub-agent telemetry.
3. A dashboard showing the SLIs, the SLO targets, and the budget-remaining state.
4. Two multi-window burn-rate alerts (fast-burn and slow-burn) per the SRE Workbook.
5. A signed SLO document (`slos/writer-v1-interactive.yaml`) ready to attach to the error-budget policy (exercise 02).

## Requirements

### Part 1 — The SLI catalogue

Author three catalogue files under `slis/` using chapter 02's schema. Minimum three of the five; author the one of hallucination-rate and safety-refusal-drift only if your stub agent emits those events. The three you MUST author:

- `slis/session_completion_rate.yaml`
- `slis/time_to_first_tool.yaml`
- `slis/time_to_final_answer.yaml`

Each file must declare:

- `name`, `version`, `description`, `units`.
- `measurement.source`, `measurement.metric`, `measurement.derivation` (the Prometheus expression as a string).
- `measurement.reductions` (`percentile_95`, `percentile_99`, `heavy_tail_fraction` where relevant).
- `labels.required` and `labels.optional`.
- `window.type` (`rolling`), `window.size` (`28d`), `window.minimum_sample_count`.
- `owners`, `change_control`.

Pin the measurement derivations to Prometheus functions where possible (`histogram_quantile`, `rate`, `sum by`). Where a function does not exist (e.g., "heavy_tail_fraction"), document how you would compute it — a recording rule, a derived metric, a dashboard-only panel — in the catalogue entry's `derivation` field.

### Part 2 — The recording rules

Produce a Prometheus `rules.yaml` file with the recording rules that materialise each SLI. Minimum coverage:

- `sli:session_completion:rate1m` — per-minute completion rate.
- `sli:session_completion:badness1m` — `1 - rate` for burn-rate math.
- `sli:ttft:p95_5m` — rolling 5-minute p95 TTFT histogram-quantile.
- `sli:ttfa:p95_5m` and `sli:ttfa:p99_5m` — rolling 5-minute p95 and p99 TTFA.
- `sli:ttfa:heavy_tail_5m` — fraction above `2 * p95`.

Add resource labels for `tenant_id`, `agent_id`, `workload_class`, `agent_revision` on every rule's output (via `by` clauses).

### Part 3 — The sample telemetry run

Run the stub agent for at least 10 minutes with realistic traffic:

- A baseline load (10 req/min) with nominal latency.
- At minute 5, a 3-minute degradation (one of: inject a 2-second wait before tool-call for 30 % of sessions; drop 10 % of sessions with a terminal `failed_recoverable`).
- A return to baseline at minute 8.

The run produces a `telemetry-trace.ndjson` or OTLP-captured file you can replay; attach a one-command runner script.

### Part 4 — The dashboard

Produce one Grafana dashboard JSON (or equivalent for your stack) with four panels:

1. **Current SLI value vs. target** — a time series for session-completion rate, TTFT p95, TTFA p95 overlaid with the target lines. Show "last 1h" by default.
2. **Error budget remaining** — a single-number panel showing `budget_remaining = (SLO_target - SLI_current_over_window) / (1 - SLO_target)` for one primary SLI (session-completion or TTFA). Include a colour-change threshold at 50 %, 25 %, 10 %.
3. **Burn-rate dual-window** — a time series showing the 1h burn rate and the 6h burn rate for session-completion, with the SRE Workbook thresholds (14.4× and 6×) as marker lines.
4. **Heavy-tail watch** — a time series showing the `ttfa:heavy_tail_5m` fraction over 24h. The panel's description explains why the heavy-tail is a watch metric, not a page-eligible one.

The dashboard is an artefact the owning team will look at weekly; make it readable.

### Part 5 — The burn-rate alerts

Two `PrometheusRule`s (or equivalent) under `alerts/`:

- **Fast-burn page.** `(1h_badness / (1 - SLO_target) >= 14.4) AND (5m_badness / (1 - SLO_target) >= 14.4)` on session-completion. `severity: page`. `for: 2m`. Annotations include the runbook link, the policy reference, and the current SLI value.
- **Slow-burn ticket.** `(6h_badness / (1 - SLO_target) >= 6) AND (30m_badness / (1 - SLO_target) >= 6)` on session-completion. `severity: ticket`. `for: 15m`.

Validate with [Promtool](https://prometheus.io/docs/prometheus/latest/command-line/promtool/) `test rules` or [Alerting-rule tests](https://prometheus.io/docs/prometheus/latest/configuration/unit_testing_rules/) using a hand-written test dataset that triggers each alert deterministically.

### Part 6 — The SLO document

`slos/writer-v1-interactive.yaml`, matching chapter 02's template. Must include:

- `name`, `version`, `workload_class: interactive-chat`, `tier: T1`, `window: 28d`.
- `objectives:` array referencing the SLIs you authored.
- `error_budget_policy: error-budget-policy@v4` (the policy you will author in exercise 02).
- `opt_in_date`, `owning_team`, `on_call_rota`.
- A `provider_dependency_adjustment` block — for this exercise, set it to `null` with a comment that the adjustment will be populated once mod-406's outage records are wired.
- At least one objective pinned to a `minimum_sample_count` threshold (document which).

### Part 7 — The README

One page covering:

- How to run the stub agent (`docker compose up` or `python stub_agent.py --scenario default`).
- How to replay the sample telemetry into Prometheus and check that the recording rules materialise.
- Which `promtool test` commands to run and what their expected output is.
- A screenshot (or link to the Grafana export) of the dashboard in both nominal and degraded states.
- Two "I was surprised by" observations — e.g., "the heavy-tail-fraction was zero in my scenario because my stub's degradation injected a *mean* shift, not a *tail* shift; the real fleet has both and the heavy-tail panel is why you see one before the other."
- One open question for mod-413 — e.g., "the SLO document references a policy that does not yet exist; what is the paved-road's story for workloads that sign an SLO before the policy is drafted?"

## Starter guidance

- **Start with the simplest SLI — session-completion.** It is a rate of counters and it is what every other SLI will be structurally similar to. TTFA with histograms is harder.
- **Seed the stub agent's telemetry with a known distribution.** Deterministic seeds make the dashboard screenshots reproducible and the alert tests obvious.
- **Keep the recording-rule outputs stable across runs.** Avoid timestamp-dependent labels; use the dashboard's time range to select the window, not the recording rules.
- **Write the `promtool test` cases before the alert.** A test that demonstrates "if we force 1h_badness to 15 % and the SLO target is 99 %, the fast-burn fires" is the clearest expression of what you mean.
- **Do not add SLIs the workload cannot emit.** If your stub does not emit safety verdicts, skip the safety-refusal SLI. A catalogue entry whose measurement.source does not exist is a lie waiting to happen.
- **Pin a policy-version reference even though exercise 02 has not run yet.** The forward reference is the right shape; the policy file will fill in when exercise 02 produces it.

## Acceptance criteria

Mark done when:

- [ ] Three SLI catalogue files exist under `slis/` and each has the chapter 02 fields populated with Prometheus-expression derivations.
- [ ] `rules.yaml` includes recording rules for every SLI; the rules materialise against the sample telemetry replay.
- [ ] A one-command replay produces a Grafana dashboard that shows baseline, degradation, and recovery clearly.
- [ ] The two burn-rate alerts exist with `promtool test` cases that demonstrate they fire on the intended inputs and do not fire on nominal.
- [ ] `slos/writer-v1-interactive.yaml` exists, is validated against the template, and references at least three objectives pinned to the SLIs.
- [ ] README includes the two "surprised by" observations and the open question.
- [ ] No SLI's measurement derivation references an attribute the stub agent does not emit.

## Stretch goals

- **Add the hallucination-rate SLI.** Stub a low-latency judge that scores a fraction of sessions and emits `platform.eval.factuality_pass`. Document the minimum-sample-count floor you chose and why.
- **Add the safety-refusal-drift SLI.** Stub safety verdicts; build the 28-day trailing baseline as a recording rule; alert on drift beyond 30 % of the baseline. Document the "first 28 days we have no baseline" boundary condition.
- **Build the exception-register consumer.** A second dashboard panel that reads from a static YAML file listing SLO overrides and surfaces which objectives have active exceptions.
- **Add a per-tenant breakdown.** Group the panels and alerts by `tenant_id` so that `acme-financial`'s burn is visible independently of other tenants. Watch the cardinality cost.
- **Translate one alert into Grafana Alerting.** Reproduce the fast-burn alert's semantics as a Grafana managed alert rule; document the shape differences (annotation handling, label mutation, routing).
- **Author a conformance test for the SLI catalogue.** A script that reads every `slis/*.yaml` file and asserts (a) the `measurement.source` is a known metric name, (b) required labels are consistent across entries, (c) the window field matches the paved-road default or has a documented reason for differing. The conformance shape is cross-reference mod-408 chapter 05.
