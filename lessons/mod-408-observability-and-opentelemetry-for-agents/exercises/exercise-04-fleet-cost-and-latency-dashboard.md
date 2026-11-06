# exercise-04: Fleet Cost and Latency Dashboard

**Estimated effort:** 2 hours · **Output:** a Prometheus + Grafana + Tempo stack running in docker-compose, a seeded metrics / traces load from the exercise-02 topology (or a synthetic equivalent), a Grafana dashboard JSON with at least four panels, and a short README (~1 page) · **Pairs with:** chapters [04](../04-backend-federation-and-the-collector-tier.md) and [06](../06-eu-ai-act-article-72-post-market-monitoring.md); feeds project 403 and project 404.

## Objective

Stand up the open-source backend stack from chapter 04 — [Prometheus](https://prometheus.io/docs/), [Grafana](https://grafana.com/docs/), [Grafana Tempo](https://grafana.com/docs/tempo/latest/) — reading the semantic stream the earlier exercises produced, and build a dashboard that answers the three fleet-level questions chapter 01 called out: cost per tenant per model, p95 latency per operation, token volume per cache partition. This is the dashboard a platform engineer and an SRE share; it is also the backbone of chapter 06's compliance-signal dashboard.

A well-authored dashboard reads straight from the semconv `gen_ai.*` metrics and the platform's `platform.*` extensions. No per-service dashboard; no per-tenant copy-paste; one Grafana dashboard, parameterised by tenant and time range.

## Prerequisites

- You have read chapters [04](../04-backend-federation-and-the-collector-tier.md) and [06](../06-eu-ai-act-article-72-post-market-monitoring.md).
- You have the exercise-02 topology (or an equivalent that emits semconv-conformant spans and GenAI metrics). If you did not do exercise 02, a synthetic trace / metric generator is acceptable — produce a load generator that emits a realistic distribution of `gen_ai.*` metrics for at least 2 tenants, 2 models, and 2 cache partitions.
- You have Docker + `docker compose` on your workstation.
- You have Grafana ([docs](https://grafana.com/docs/)), Prometheus ([docs](https://prometheus.io/docs/)), and Tempo ([docs](https://grafana.com/docs/tempo/latest/)) container images available.

## Problem statement

The dashboard must answer three questions from the collected telemetry alone:

1. **Cost per tenant per model, rolled up to a configurable window.** A panel that shows the top 5 tenant × model combinations by total USD spend in the selected window. Reads `platform.gen_ai.cost_usd` (a sum / counter metric; the mod-406 model gateway emits it alongside the semconv token metrics).
2. **p95 latency per operation per model.** A panel that shows the 95th percentile of `gen_ai.client.operation.duration` tagged by `gen_ai.operation.name` × `gen_ai.response.model`. Reads from Prometheus histograms using [`histogram_quantile`](https://prometheus.io/docs/prometheus/latest/querying/functions/#histogram_quantile).
3. **Token volume by cache partition.** A panel that shows input-token volume split by `platform.cache_partition`, for one selected tenant / time range. Reads `gen_ai.client.token.usage` histogram's input-token bucket, tagged by `platform.cache_partition`.

Plus one Article-72-flavoured panel (see Part 4).

## Requirements

### Part 1 — The stack

Author `docker-compose.yaml` bringing up:

- `otel-collector` — the OTel Collector with receivers for OTLP on 4317/4318, a `prometheusremotewrite` exporter to Prometheus, and an `otlp` exporter to Tempo. Config file lives at `collector-config.yaml`.
- `prometheus` — Prometheus with a scrape target for the collector's internal `/metrics` endpoint (fleet-health observability) and a remote-write receiver for the `prometheusremotewrite` exporter. Config at `prometheus.yml`.
- `tempo` — Grafana Tempo with OTLP ingest on 4317 (choose a non-conflicting port if the collector also uses 4317) and local storage (`/tmp/tempo` is fine).
- `grafana` — Grafana with Prometheus and Tempo pre-provisioned as data sources (use the [provisioning docs](https://grafana.com/docs/grafana/latest/administration/provisioning/)) and the dashboard JSON pre-imported.

The collector config includes the `transform/redact_content` processor from chapter 04 even though the exercise's load does not emit content — the lesson is defence in depth.

### Part 2 — The load

Run the exercise-02 topology against the collector for at least five minutes, with:

- At least 100 invocations of the `agent.turn` path.
- Two tenant ids (set via baggage, as in exercise 02).
- Two model identifiers — have the model-gateway stub pick between them deterministically (round-robin, say).
- Two cache-partition values — attached as `platform.cache_partition` on the model-gateway span.
- A deterministic cost emission — the model-gateway stub emits a `platform.gen_ai.cost_usd` metric per call using a canned per-model rate (don't invent real pricing; mark `<!-- needs-research: ... -->` against any specific rate and use an illustrative value).

If you don't have the exercise-02 topology, author a `load_generator.{py,ts,go}` that emits the equivalent metrics stream directly via the OTel SDK. Save the capture to `fixtures/sample-load.jsonl` so the dashboard is reproducible.

### Part 3 — The dashboard

Author `dashboards/fleet-overview.json` (a standard Grafana dashboard JSON). The dashboard has, at minimum, these panels:

- **Panel 1 — Cost per tenant per model (table).** Top 5 rows, sorted by sum of `platform.gen_ai.cost_usd` in the selected window. Columns: `tenant_id`, `response_model`, `total_cost_usd`, `calls`.
- **Panel 2 — p95 latency per operation per model (time series).** Lines for each `(gen_ai.operation.name, gen_ai.response.model)` tuple over the selected window. Query uses `histogram_quantile(0.95, sum by (...)(...))` on the operation-duration histogram.
- **Panel 3 — Token volume by cache partition (time series).** Stacked lines, one per `platform.cache_partition`, for the selected tenant. Query sums `gen_ai.client.token.usage` input-token count.
- **Panel 4 — Article 72 compliance signals (stat panels).** A row of stat panels showing: count of guardrail-flagged traces in window, count of policy-overrides in window, average retention-policy conformance (content-store writes with the expected `retention_window_days` tag), and total serious-incident-candidate traces. Values are stubbed if your exercise-02 topology doesn't emit them; mark the stubs in the panel description.

Each panel has:

- A `description` field naming the semconv / platform attributes the query reads.
- A `templating` variable for `tenant_id` (so the tenant filter is a dropdown at the top of the dashboard).
- A `templating` variable for the time window (default: last 1 hour).
- A link from each panel to the Tempo trace view for a representative trace in the window (use [Grafana's exemplars](https://grafana.com/docs/grafana/latest/fundamentals/exemplars/) if the metric emits them, or a generic Explore link otherwise).

### Part 4 — The Article 72 panel

Chapter 06 described a compliance-signal dashboard. Panel 4 is the minimal prototype. For stretch, panel 4 can read from a `fixtures/article-72-inputs.jsonl` file you author by hand — one row per compliance signal, tagged with the Article it supports. In the exercise-level version, four stat panels are enough; the lesson is that the compliance signals share the dashboard layer with the engineering signals, not that you implement a real Article 72 export here (that is a mod-413 governance-committee artefact).

### Part 5 — The README

One page covering:

- `docker compose up` starts the stack; the load generator runs for five minutes; Grafana at `http://localhost:3000` shows the dashboard.
- A screenshot (or ASCII sketch) of the dashboard with the four panels.
- The PromQL / TraceQL queries behind each panel, one per code block.
- Which semconv attributes and which `platform.*` extensions each panel reads. A reviewer who wants to port the dashboard to Datadog or Langfuse can cross-reference the attribute names.
- One "I was surprised by" observation — e.g. "the histogram quantile is a function over the histogram, not over the raw samples; selecting the wrong aggregation gives a completely different curve."
- One open question for mod-410 SRE — e.g. "which of these panels should drive alerts, versus which are investigation-only?"
- One open question for mod-413 governance — e.g. "the Article 72 panel is engineering-visible here; does leadership want a separate view, or is this the view?"

## Starter guidance

- **Pin the image versions.** Grafana, Prometheus, Tempo, the collector — all pin to specific release tags in the compose file. The APIs are stable; the defaults shift.
- **Use the collector's `prometheusremotewrite` exporter, not Prometheus's OTLP receiver.** <!-- needs-research: Prometheus added native OTLP ingest in recent releases; verify current release-level support before swapping. The remote-write exporter path is the reliable default. -->
- **Pre-provision data sources via Grafana provisioning YAML.** Clicking "Add data source" works for exploration; the exercise's reproducibility depends on provisioning files that live in the repo.
- **Keep the dashboard JSON human-readable.** Grafana's JSON export is noisy by default; strip the panel ids and version history before committing.
- **Mark `<!-- needs-research: ... -->` on any cost figure.** Prices move; a $0.0030 per-1k-token rate in a load generator is an illustrative value, not a quote. Chapter 06 and mod-406 chapter 04 both call out the pattern.
- **Flush the collector before stopping.** `docker compose stop` is usually graceful; a `docker compose down` mid-batch drops tail metrics. The dashboard time-window default (last 1 hour) hides this but a reviewer re-running at 10 minutes may see empty panels.
- **Keep the exemplar links simple.** An "Explore in Tempo" link on each panel is enough. Full click-through-to-trace with correct exemplars is a stretch goal.

## Acceptance criteria

Mark done when:

- [ ] `docker compose up` brings up the stack and the dashboard renders at `http://localhost:3000`.
- [ ] The load generator (or exercise-02 topology) produces ≥5 minutes of metrics / traces across 2 tenants and 2 models.
- [ ] Panel 1 shows the top 5 tenant × model rows by cost.
- [ ] Panel 2 shows p95 latency per `(operation.name, response.model)` tuple.
- [ ] Panel 3 shows input-token volume split by `platform.cache_partition`.
- [ ] Panel 4 shows four Article-72 stat tiles with descriptions linking to the Article the signal supports.
- [ ] The tenant dropdown filters all four panels.
- [ ] The README names the queries, the attributes, the surprises, and the two open questions.
- [ ] No cost / pricing figure in the load generator is presented as authoritative; illustrative values are flagged.

## Stretch goals

- **Mirror the dashboard on Langfuse or Phoenix.** Add the Langfuse or Phoenix exporter to the collector config (per chapter 04), run the same load, and produce a parallel dashboard in that backend. The lesson: same instrumentation, two UIs; and the cases where the dashboards differ are the cases where the backend conventions differ.
- **Add a cost-anomaly alert in Grafana.** [Grafana unified alerting](https://grafana.com/docs/grafana/latest/alerting/) fires when `rate(platform.gen_ai.cost_usd[5m])` by tenant exceeds a configurable threshold. The alert's annotations link to the Tempo trace view. A down payment on mod-410's SRE alerting discipline.
- **Add a replay-attribution panel.** Chapter 03 named `platform.replay_source_trace_id` as the link from an eval replay to its production source. A fifth panel shows which production traces have been replayed recently and how many replays each attracted. Useful for exercise-level verification that chapter 03's link pattern survives round-tripping through the backend.
- **Wire a Promethues scrape of the collector's internal telemetry.** Panel showing the collector's queue depth, batch size, export error rate. Chapter 04 called out that the collector-that-is-blind-to-itself fails silently; the panel is the mitigation.
- **Export the dashboard JSON as a Grafana observability-as-code artefact.** [Grafonnet](https://grafana.github.io/grafonnet/) or the [Grafana Foundation SDK](https://grafana.github.io/grafana-foundation-sdk/) let the dashboard live in a typed source, not a hand-edited JSON. The compile step catches a query against a renamed attribute at build time.
- **Add a third tenant that uses a 100% retention policy; show on the dashboard.** The tenant filter now has three options; the Article-72 retention panel shows the 100%-retention tenant differently. A visual demonstration of chapter 04's per-tenant-partitioned retention.
