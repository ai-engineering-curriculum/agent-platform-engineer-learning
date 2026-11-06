# exercise-03: Adoption Metric Dashboard

**Estimated effort:** 2 hours · **Output:** a formulas file pinning the four KPIs, a generator for a synthetic `AgentRevisionStream` + `CIEventStream` + `DeployEventStream`, a dashboard (Grafana / Metabase / Streamlit — your choice) rendering the four KPIs over the synthetic data, a signed one-page quarterly scorecard, and a short README (~1 page) · **Pairs with:** chapter [06](../06-measuring-adoption-and-cncf-maturity.md); feeds `mod-413` and project 401.

## Objective

Translate chapter 06's four KPIs and the CNCF-maturity frame into a running dashboard and a quarterly scorecard. You will pin the formulas in code (not in dashboard-config glue), generate a synthetic event stream rich enough to exercise the KPIs' interesting shapes, render the dashboard against the stream, and produce the one-page scorecard the quarterly review will consume.

Chapter 06 pinned the measurement; this exercise is where the measurement becomes operational. The platform engineer who has never written the formulas as code under-estimates how much ambiguity hides in the English ("what counts as an active agent", "what is the right window", "is that a rollback or a canary-stop"). Each ambiguity is a decision; your formulas file is where you record them.

## Prerequisites

- You have read chapter [06](../06-measuring-adoption-and-cncf-maturity.md) and [`mod-401` chapter 02](../../mod-401-agent-platform-position-and-paved-road-philosophy/02-cncf-maturity-and-team-topologies.md).
- You have an `AgentRevision` manifest shape from [exercise 01](exercise-01-paved-road-sdk-template-authoring.md) (or can borrow the schema from chapter 01).
- You have access to a dashboard tool: [Grafana](https://grafana.com/) (open-source), [Metabase](https://www.metabase.com/), [Streamlit](https://streamlit.io/), or a Jupyter notebook with [`panel`](https://panel.holoviz.org/) or [`plotly`](https://plotly.com/python/) — your call. Streamlit is the lowest-friction option for this exercise.
- Python 3.11+ with [`duckdb`](https://duckdb.org/) or [`polars`](https://pola.rs/) for the formula computations. [`sqlglot`](https://github.com/tobymao/sqlglot) is handy if you want to co-author formulas as SQL you can validate.

## Problem statement

The platform-team lead has committed the following to leadership: "within one quarter, we will publish a weekly adoption scorecard with four KPIs and a CNCF-maturity narrative". You own the mechanism.

The output has five parts:

1. The formulas file (`metrics.yaml`) pinning each KPI's definition.
2. A synthetic data generator that produces three streams — manifest, CI, deploy — covering 90 days, 6 tenants, 2-3 workload classes, and the chapter-06 KPI patterns.
3. The dashboard rendering the four KPIs against the synthetic streams.
4. The signed quarterly scorecard document (one page) that mod-413 would consume.
5. A short README (~1 page) with trade-offs, two "I was surprised by" observations, and one open question for the CNCF-maturity narrative.

## Requirements

### Part 1 — The formulas file

`metrics.yaml` with one entry per KPI. Each entry declares:

- `id` — e.g., `platform_adoption_rate`.
- `title` — the human-readable name.
- `description` — the chapter-06 English definition.
- `formula` — SQL or a code-callable function name computing the metric from the streams.
- `inputs` — the stream and fields consumed.
- `dimensions` — the splits (tenant, workload_class, tier, adapter, exception).
- `window` — the time window (e.g., `rolling_14d`).
- `owner` — the formula's owner (`@paved-road-plane`).
- `last_reviewed_at` — ISO-8601.

Example:

```yaml
- id: platform_adoption_rate
  title: Platform Adoption Rate
  description: >
    Fraction of catalogued agents that emitted a valid AgentRevision manifest
    in the rolling window. Covers pull-signal distinguishability via the
    exception-register split (chapter 06).
  formula: |
    WITH catalog AS (SELECT agent_id FROM org_catalog),
         emitted AS (SELECT DISTINCT agent_id FROM agent_revision_stream
                     WHERE generated_at >= NOW() - INTERVAL '14 days'
                       AND manifest_valid = TRUE),
         exceptioned AS (SELECT DISTINCT agent_id FROM emitted e
                         JOIN manifest_exception_overrides o
                         ON o.agent_id = e.agent_id
                         WHERE o.status = 'approved')
    SELECT
      (SELECT COUNT(*) FROM emitted) * 1.0 / (SELECT COUNT(*) FROM catalog) AS rate,
      (SELECT COUNT(*) FROM emitted) - (SELECT COUNT(*) FROM exceptioned) AS fully_on_paved_road,
      (SELECT COUNT(*) FROM exceptioned) AS on_paved_road_with_exception,
      (SELECT COUNT(*) FROM catalog) - (SELECT COUNT(*) FROM emitted) AS invisible
  inputs:
    - agent_revision_stream.agent_id
    - agent_revision_stream.generated_at
    - agent_revision_stream.manifest_valid
    - org_catalog.agent_id
    - manifest_exception_overrides.agent_id
    - manifest_exception_overrides.status
  dimensions: [tenant_id, workload_class, tier, adapter]
  window: rolling_14d
  owner: "@paved-road-plane"
  last_reviewed_at: "2026-11-06"
```

Repeat for `time_to_first_agent`, `deployment_success_rate`, `rollback_count`. For `time_to_first_agent`, split into `T1`, `T2`, `T3` sub-metrics as chapter 06 requires.

### Part 2 — The synthetic data generator

A script (`scripts/generate_streams.py`) that produces JSONL files for three streams:

- `agent_revision_stream.jsonl` — ~2000 manifests across 90 days, 6 tenants, 2 workload classes. Vary the SDK version, adapter, template version; include some invalid manifests (missing fields) and some referencing exception-register entries.
- `ci_event_stream.jsonl` — ~5000 CI runs. 85% pass; 15% fail on various gates (eval, safety, cost). Correlated with the manifests.
- `deploy_event_stream.jsonl` — ~1000 deploys. 91% successful first attempt; 9% rolled back for various triggers (SLO burn, safety refusal, manual, post-incident).

Plus one static file:

- `org_catalog.jsonl` — 250 agents across the six tenants; some have manifests in the stream, some do not (the "invisible" fraction).

The generator must also produce a seed for one of each KPI pattern from chapter 06's "reading the four KPIs together" table — e.g., one tenant with "high adoption, short time-to-first-agent, high rollback", another with "steady adoption, steady time, steady rollback". The scorecard (below) comments on these patterns, so the data must exercise them.

### Part 3 — The dashboard

The dashboard renders each KPI at the detail chapter 06 describes:

- **Platform Adoption Rate** panel: headline rate; delta vs. prior 14d; breakdown by tenant, workload class, adapter; split by fully-on-paved-road / with-exception / invisible.
- **Time-to-First-Agent** panel: median `T1` / `T2` / `T3`; outliers table.
- **Deployment Success Rate** panel: rate overall and by tier; breakdown by failure cause; non-paved-road comparison line.
- **Rollback Count** panel: count; rate; MTTR; breakdown by trigger.

Each panel reads the formulas from `metrics.yaml` and queries the streams via DuckDB (or the equivalent). The panels update when you re-run `scripts/generate_streams.py --seed <different>` — the dashboard is a function of the streams, not a tableau of hard-coded numbers.

One additional panel: **CNCF Maturity Narrative** — a markdown panel that reads a file (`current_maturity.md`) summarising where the platform sits on Investment / Adoption / Interfaces / Operations. This is the hand-authored piece; the dashboard surfaces it alongside the numbers.

### Part 4 — The quarterly scorecard

A one-page markdown document `scorecard-Q4-2026.md`:

- Header: the quarter, the generation timestamp, the signing owner.
- CNCF-maturity levels for each of the four dimensions, each one line.
- The four KPIs with their values and deltas.
- "Advances since Q3" — three bullet points.
- "Open risks" — three bullet points.
- Signature block (mod-413 governs the signatures; in this exercise, record who *would* sign).

The scorecard quotes real numbers from your synthetic streams. The narrative is the hand-authored piece that explains what the numbers mean in your data.

### Part 5 — README (~1 page)

- The dashboard tool you chose and why.
- The one ambiguity in the chapter-06 definitions you had to resolve (there will be at least one — write how you resolved it).
- Two "I was surprised by" observations — candidates: the chosen window hides a specific pattern; the exception-register split turned out to matter more than the headline rate; `T1` is easier to measure than `T2` because template-provenance is explicit.
- One open question for the CNCF-maturity narrative — e.g., "we are Scalable on Interfaces but Operational on Adoption; the maturity model says you can be mixed across dimensions, but should the scorecard call out the lag explicitly?".

## Starter guidance

- **Write the formulas before the generator.** Otherwise you will tune the generator to produce numbers you want to see, and the exercise becomes "building a dashboard to tell yourself what you already think".
- **Use DuckDB over JSONL.** `SELECT * FROM 'agent_revision_stream.jsonl'` is native; no ingestion step.
- **Make the generator seedable.** `--seed 42` produces deterministic streams for the smoke test; `--seed random` produces a fresh set for exploratory exercises.
- **Include at least one "pattern" per tenant.** Hello-world inflation, mandate adoption, framework-captive adoption, metric-chasing adoption — the generator should let one tenant exhibit one of chapter 06's vanity-metric failure modes so your dashboard can surface it.
- **Resist pre-aggregating.** Keep the streams detailed; the formulas aggregate. If you pre-aggregate in the generator, you cannot answer new questions without regenerating.
- **Keep the scorecard short.** One page is the whole point. If it does not fit on one page in the mod-413 governance review, it will not be read.
- **Mark `<!-- needs-research: ... -->` on CNCF-maturity-model indicator wording** — the whitepaper revs; quote your source commit.

## Acceptance criteria

Mark done when:

- [ ] `metrics.yaml` pins all four KPIs with inputs, dimensions, window, and owner.
- [ ] `scripts/generate_streams.py --seed <N>` produces the three JSONL streams plus the catalog file.
- [ ] The dashboard renders all four KPIs with the chapter-06 breakdowns.
- [ ] At least one "pattern" from chapter 06's "reading the four KPIs together" table is observable in the dashboard.
- [ ] The exception-register split is first-class on the Platform Adoption Rate panel.
- [ ] The CNCF Maturity Narrative panel is present and the content lands on one screen.
- [ ] `scorecard-Q4-2026.md` fits on one page and is signed.
- [ ] README includes the two "surprised by" observations and the open question.
- [ ] Re-running the generator with a different seed produces different dashboard values without any manual recalculation.

## Stretch goals

- **Add a satisfaction survey ingestion.** Chapter 06 calls for an orthogonal instrument (DevEx-aligned, five questions). Generate 50 synthetic survey responses tied to tenants; add a sixth panel reading them.
- **Add an office-hours question log.** Chapter 07's format: tag, tenant, disposition. Generate 20 entries; add a panel showing topic counts.
- **Implement the fairness-attribution split for `time_to_first_agent`.** Chapter 06 calls out that `T1` / `T2` / `T3` belong to different owners; add a panel attributing each to its owner so the scorecard conversation can focus on the right lever.
- **Add a "paved-road-vs-not" comparison panel.** The `DeploymentSuccessRate`'s non-paved-road line is especially valuable; extend the generator to produce non-paved-road builds and show the delta.
- **Author a non-vanity-metric unit test.** A pytest test that runs each chapter-06 vanity pattern through the generator and asserts the dashboard surfaces the failure mode (e.g., Hello-world inflation shows a spike with a flat production-filter number).
- **Alert on invisibility.** If the "invisible" column grows week-over-week, the dashboard emits a Slack notification (or writes a report). Chapter 06 is explicit that this is not a paging surface, but a weekly digest is fair game.
- **Write the CNCF-maturity assessment script.** A script that reads the KPIs and `current_maturity.md` and asserts a consistency check (e.g., "if Rate > 70% and `time_to_first_agent` median < 24h, Interfaces and Adoption should both be at level 3 or higher").
