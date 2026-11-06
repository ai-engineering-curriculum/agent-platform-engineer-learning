# Backend Federation and the Collector Tier

Chapter 02 pinned the vocabulary. Chapter 03 pinned the context propagation. This chapter picks up the third invariant: the same semantic stream reaches every backend each tenant (or each use case) chooses, without the emitting planes re-instrumenting per destination. The runtime team should write `gen_ai.request.model` once; whether a tenant reads it on [Langfuse](https://langfuse.com/docs), [Arize Phoenix](https://docs.arize.com/phoenix), [Datadog LLM Observability](https://docs.datadoghq.com/llm_observability/), [OpenLLMetry](https://www.traceloop.com/docs/openllmetry/introduction), or a [Prometheus](https://prometheus.io/docs/) + [Grafana](https://grafana.com/docs/) + [Tempo](https://grafana.com/docs/tempo/latest/) stack is a routing concern, not an instrumentation concern.

The mechanism is the [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/). This chapter walks the two-tier topology, the exporters per backend, the redaction and sampling processors, the content-store split, and the tenant-routing fan-out. Chapter 05 writes the conformance tests against it; chapter 06 reads from it for the Article 72 view.

## Why a collector tier and not SDK-direct OTLP

The simplest topology is "each service's SDK pushes OTLP straight to the backend of choice". It works until any of the following lands on the board.

- **Two tenants with two backends.** Tenant A wants Datadog; tenant B wants Langfuse. A given service might serve both in a single process. SDK-direct means the SDK must know per-request which backend to target — a request-scoped exporter decision, not a service-startup one.
- **Redaction.** Prompts and completions carry PII. A redactor inside each service is N redactor implementations to keep in sync; a redactor at a shared collector is one implementation with one audit log.
- **Semantic translation at the exporter seam.** Phoenix speaks [OpenInference](https://arize-ai.github.io/openinference/); Datadog LLM Observability wants its own tag namespace on top of OTLP; Langfuse ingests OTel-native spans and also offers its own SDK. A translator per backend lives somewhere; a collector is the sane place.
- **Tail sampling.** The decision "keep this trace because something interesting happened" requires seeing the whole trace. An SDK can't tail-sample — by the time it decides, the spans have already been exported.
- **Observability of the observability pipeline.** The collector emits its own metrics ([internal telemetry](https://opentelemetry.io/docs/collector/internal-telemetry/)); SRE (mod-410) watches them; chapter 01's "collector failure modes that go blind" threat is where this visibility matters.

Enter the two-tier collector shape: a local agent collector (per node or per pod sidecar) and a gateway collector (a horizontally scaled central deployment). The SDK speaks OTLP to the local agent; the local agent forwards to the gateway; the gateway does the shared work and fans out.

## The topology

```
┌─────────────┐  OTLP  ┌─────────────┐  OTLP  ┌─────────────────────┐
│ service SDK │──────► │ node-local  │──────► │   gateway collector │
│  (any lang) │        │ collector   │        │  (HA, scaled)       │
└─────────────┘        │  (daemon /  │        │                     │
                       │   sidecar)  │        │  processors:        │
                       └─────────────┘        │   - redact content  │
                                              │   - tail sample     │
                                              │   - tenant route    │
                                              │   - semconv version │
                                              │     gate            │
                                              │                     │
                                              │  exporters:         │
                                              │   - Langfuse        │
                                              │   - Phoenix         │
                                              │   - Datadog         │
                                              │   - Prometheus      │
                                              │   - OTLP → Tempo    │
                                              │   - content store   │
                                              │     (S3)            │
                                              └─────────────────────┘
```

Roles:

- **Node-local collector.** Terminates SDK connections, buffers on back-pressure, forwards upstream. Minimal processing — a `batch` processor and a `memory_limiter` is the typical config. It exists for reliability (the SDK's export does not fail if the gateway is briefly unavailable) and for easier network policy.
- **Gateway collector.** Does the work. Receives OTLP from node agents, applies redaction and sampling, routes per tenant, exports to backends and to the content store.

The gateway collector is a tier-0 service. It has an SRE runbook (mod-410) and its own observability. Chapter 01's threat model called out that an unobserved collector goes blind silently; the mitigation is `otelcol_*` Prometheus metrics (queue depth, batch size, export errors, dropped spans) scraped by the platform's main Prometheus and alerted on.

## The pipeline, in `otelcol` config terms

The OpenTelemetry Collector is configured with [`receivers`, `processors`, `exporters`, and `pipelines`](https://opentelemetry.io/docs/collector/configuration/). A sketch of a gateway collector config (illustrative — pin versions and verify against the current [`contrib` distribution](https://github.com/open-telemetry/opentelemetry-collector-contrib) before deploying):

```yaml
receivers:
  otlp:
    protocols:
      grpc: { endpoint: 0.0.0.0:4317 }
      http: { endpoint: 0.0.0.0:4318 }

processors:
  memory_limiter:
    check_interval: 1s
    limit_mib: 4096
  batch:
    send_batch_size: 8192
    timeout: 1s
  transform/redact_content:
    log_statements:
      - context: span
        statements:
          - delete_key(attributes, "gen_ai.prompt")
          - delete_key(attributes, "gen_ai.completion")
      - context: spanevent
        statements:
          - delete_key(attributes, "gen_ai.message.content")
  tail_sampling:
    decision_wait: 10s
    policies:
      - name: error-traces
        type: status_code
        status_code: { status_codes: [ERROR] }
      - name: guardrail-flagged
        type: string_attribute
        string_attribute:
          key: platform.safety.flagged
          values: ["true"]
      - name: sample-rest
        type: probabilistic
        probabilistic: { sampling_percentage: 10 }
  routing/by_tenant:
    from_attribute: platform.tenant_id
    default_exporters: [otlp/langfuse_default]
    table:
      - value: acme-financial
        exporters: [datadog/llm, otlp/langfuse_acme]
      - value: contoso
        exporters: [otlp/phoenix]

exporters:
  otlp/langfuse_default:   { endpoint: "cloud.langfuse.com:4317", ... }
  otlp/langfuse_acme:      { endpoint: "selfhosted.langfuse.acme.com:4317", ... }
  otlp/phoenix:            { endpoint: "phoenix.internal:4317", ... }
  datadog/llm:             { api: { key: "${DD_API_KEY}" }, ... }
  prometheusremotewrite:   { endpoint: "https://prom.internal/api/v1/write" }
  otlp/tempo:              { endpoint: "tempo.internal:4317" }
  awss3:
    s3uploader:
      s3_bucket: platform-sessions
      s3_prefix: "{{ .Attributes['platform.tenant_id'] }}/{{ .TraceID }}"

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, transform/redact_content, tail_sampling, batch, routing/by_tenant]
      exporters: [otlp/langfuse_default, otlp/phoenix, datadog/llm, otlp/tempo, awss3]
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [prometheusremotewrite, datadog/llm]
```

<!-- needs-research: the exact processor names, option fields, and exporter inventory in the contrib distribution evolve across minor versions; before copying this config, verify against the current `opentelemetry-collector-contrib` README and the specific release's component list. -->

Three features worth calling out:

- **The `transform/redact_content` processor strips content attributes.** The semantic-convention-aware redactor lives here. A chapter 02 invariant ("content never ships on the shared pipeline") becomes one block of config. Spans arrive with attributes; the processor deletes `gen_ai.prompt` and `gen_ai.completion` on every span and `gen_ai.message.content` on every span event. The content got to the content store via a separate path (next section).
- **`tail_sampling` keeps the interesting traces.** Errors and guardrail-flagged traces survive at 100%; everything else at 10%. The numbers are per tenant / per cost target; <!-- needs-research: default sampling percentages are a cost / coverage tradeoff that depends on fleet volume and backend ingest cost. Tune per deployment and document the chosen value. -->.
- **`routing/by_tenant` fans out per tenant.** Two tenants on Langfuse, one on Phoenix, all on Datadog. One pipeline; routing decides destinations.

## The content-store split

Chapter 01 said: content lives in the content store, metadata on the trace pipeline. The collector enforces the split.

- **At the SDK.** The emitting plane never writes full prompt / completion text into a span attribute. The chapter 02 convention: prompts and completions live in the content store; the span carries `platform.content_ref` pointing at the store.
- **At the collector (defence in depth).** Even if a plane violates the SDK convention, the `transform/redact_content` processor strips content before export. A violator surfaces via a log signal (the processor emits a counter); the conformance test (chapter 05) fails the plane in CI.
- **At the content store.** A tenant-partitioned S3 / GCS / R2 bucket, write-only from the collector's `awss3` exporter, read-only to scoped services (trace-replay from mod-407 chapter 04, Article 72 export from chapter 06 below). Tight RBAC; audit log on reads.

The content store's path layout matches the trace model: `s3://platform-sessions/{tenant_id}/{trace_id}/session.jsonl`. One file per trace; appended-to as spans close. The format is OTLP's [JSON-encoded trace protobuf](https://github.com/open-telemetry/opentelemetry-proto/blob/main/opentelemetry/proto/trace/v1/trace.proto) with the content attributes preserved. Retention is tenant-specific (30 / 90 / 180 / 365 days typical); the collector tags the file with lifecycle metadata the object store enforces.

The split is the primary PII posture. The shared backends (Langfuse, Phoenix, Datadog) see structure; the content store sees content; access is partitioned by tenant; retention is bounded. A reviewer working on a specific trace reads the content store; a dashboard aggregating across tenants reads the backends.

## Backend integration: four shapes

Each backend consumes the semantic stream differently. The platform supports five; here are the four meaningful shape-classes.

### OpenLLMetry and Langfuse: OTLP-native + GenAI semconv

[OpenLLMetry](https://www.traceloop.com/docs/openllmetry/introduction) (by Traceloop) is a set of OTel-native instrumentations plus a convention over GenAI telemetry that predates / overlaps with OTel's own GenAI semconv. [Langfuse](https://langfuse.com/docs) ingests OTLP natively ([Langfuse's OTel documentation](https://langfuse.com/docs/opentelemetry)) and reads `gen_ai.*` attributes to populate its "Trace" and "Observation" models.

The exporter is a vanilla `otlp` exporter pointed at the Langfuse endpoint. No translation layer required:

```yaml
exporters:
  otlp/langfuse:
    endpoint: "cloud.langfuse.com:4317"
    headers:
      authorization: "Basic ${LANGFUSE_API_KEY}"
```

The platform's contract to Langfuse is "the spans we send carry the GenAI semconv attributes". Langfuse displays them. For tenants that run self-hosted Langfuse (behind their VPC), the exporter targets their endpoint instead. Per-tenant endpoint selection is the `routing/by_tenant` processor's job.

OpenLLMetry ships both an SDK and a traceloop-cloud destination. The platform does not use the OpenLLMetry SDK directly (it would be a competing emission path alongside the platform's own); it uses the OTel SDK plus the platform's semconv package. The OpenLLMetry contribution the platform does adopt is their instrumentation libraries (`opentelemetry-instrumentation-openai`, etc.) where their coverage exceeds the main contrib repo's. <!-- needs-research: the overlap between `openllmetry` instrumentation packages and upstream `opentelemetry-python-contrib` shifts quarter by quarter; pick per package based on current maintenance status. -->

### Arize Phoenix: OTLP + OpenInference translation

[Arize Phoenix](https://docs.arize.com/phoenix) is OTel-native on ingestion but speaks [OpenInference](https://arize-ai.github.io/openinference/) as its canonical span convention. Many `gen_ai.*` attributes map to OpenInference attributes (`openinference.*`, `input.value`, `output.value`, `llm.*`); some are Phoenix-specific. Phoenix accepts semconv-shaped spans and translates server-side where it can.

The exporter path:

```yaml
exporters:
  otlp/phoenix:
    endpoint: "phoenix.internal:4317"

processors:
  transform/openinference_compat:
    trace_statements:
      - context: span
        statements:
          # Illustrative — the exact mappings depend on Phoenix / OpenInference
          # versions. See the OpenInference specification for authoritative names.
          - set(attributes["llm.model_name"], attributes["gen_ai.request.model"])
            where attributes["gen_ai.request.model"] != nil
          - set(attributes["llm.token_count.prompt"], attributes["gen_ai.usage.input_tokens"])
            where attributes["gen_ai.usage.input_tokens"] != nil
```

<!-- needs-research: OpenInference / Phoenix attribute mapping is tracked in the OpenInference spec and the Phoenix docs; the illustrative statements above should be regenerated from the current spec before use. -->

A tenant that chose Phoenix sees traces with the familiar Phoenix layout; the emitting planes did no Phoenix-specific work. Translation is a collector processor the platform owns. When OpenInference and semconv fully converge, the processor drops to a no-op.

### Datadog LLM Observability: OTLP ingestion + Datadog tag namespace

[Datadog LLM Observability](https://docs.datadoghq.com/llm_observability/) is a Datadog product layered over spans and metrics. Ingestion is via [Datadog's OTLP ingest](https://docs.datadoghq.com/opentelemetry/) or via the Datadog exporter in the collector. Datadog's convention maps OTel attributes to Datadog tags; the exporter does the translation automatically.

```yaml
exporters:
  datadog/llm:
    api:
      key: "${DD_API_KEY}"
      site: "datadoghq.com"
    traces:
      compute_stats_by_span_kind: true
    hostname: ${HOSTNAME}
```

The platform's contract to Datadog is "the spans we send carry the GenAI semconv attributes"; the exporter translates. A nuance: Datadog's LLM Observability product consumes certain Datadog-specific attributes (`llm.model`, `llm.application`) for product-level panels. The exporter is configured to synthesise these from semconv names, so the emitting planes continue to speak semconv. <!-- needs-research: Datadog periodically updates which attributes LLM Observability consumes natively. Re-verify mappings against current product docs before each exporter config change. -->

### Grafana + Prometheus + Tempo: the open-source stack

The open-source combination is: [Prometheus](https://prometheus.io/docs/) + [`prometheusremotewrite` exporter](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter/prometheusremotewriteexporter) for metrics, [Grafana Tempo](https://grafana.com/docs/tempo/latest/) + OTLP exporter for traces, [Grafana](https://grafana.com/docs/) for the dashboards, and a Prometheus scrape of the collector's internal telemetry for pipeline health.

```yaml
exporters:
  prometheusremotewrite:
    endpoint: "https://prometheus.internal/api/v1/write"
    resource_to_telemetry_conversion: { enabled: true }
  otlp/tempo:
    endpoint: "tempo.internal:4317"
    tls: { insecure: false }
```

The GenAI metrics (`gen_ai.client.token.usage`, `gen_ai.client.operation.duration`, and the platform extensions `platform.gen_ai.cost_usd`, `platform.gen_ai.quota_decision`) land in Prometheus and become the backbone of chapter 06's fleet-level dashboards. Tempo stores the full trace for drilldown. Grafana cross-links the two ([Tempo-Prometheus integration](https://grafana.com/docs/tempo/latest/metrics-generator/)).

This is the shape the exercise-04 dashboard builds against. It is also the shape mod-410 (SRE) reads for its SLO dashboards.

## Sampling at the gateway: strategies

Head sampling (chapter 03) decides at the root; tail sampling decides at the collector once the whole trace is visible. The platform uses both.

- **Head sampling at the entry (runtime).** 100% for serious-incident candidates (listed in chapter 03); a configurable probability for the rest (10–50% typical depending on fleet volume).
- **Tail sampling at the gateway (chapter's `tail_sampling` processor).** Keep every trace with an error, a guardrail flag, a tool-call failure, a policy denial, or a cost anomaly; drop a configurable fraction of "boring" traces.
- **Backend-specific sampling where ingest cost justifies.** Datadog's LLM Observability may receive only the keep set; Prometheus always receives the full metric stream (metrics are already aggregated, not per-trace).

Sampling rules are a tenant-visible config. A tenant under compliance obligations may require 100% retention in their content store regardless of the collector's sampling. The content-store exporter and the trace exporter are independent; the content store's sampling is always 100% for tenants under such obligations, driven by the baggage-propagated `tenant_id`.

## Semantic-version gating at the collector

Chapter 02 pinned semconv versions. The collector enforces them:

- **Per-emitter version check.** Every span carries `platform.semconv_version` as a resource attribute. The collector runs a processor (an OTTL statement or a custom processor) that drops spans with an unsupported version and emits a `platform.semconv_version_rejected` counter.
- **Mixed-version windows.** During rollout, the collector supports N and N-1 simultaneously. Both get exported. The conformance tests (chapter 05) exercise both paths.
- **Backfill translation.** When semconv renames an attribute, the collector's processor can translate inbound-N-1 to outbound-N for backends that only know N. The translation is a bounded, documented list.

Semconv version is a platform-visible fact; the gating is executable.

## Operational concerns: the gateway collector is tier-0

Three ops concerns to call out — mod-410 owns the SRE discipline, but the module writes these down as platform obligations.

- **HA deployment.** The gateway collector is horizontally scaled with at least two replicas across availability zones. Local agents are configured with multiple upstream endpoints; failover is at the DNS / load balancer layer.
- **Back-pressure handling.** `memory_limiter` is required. On pressure, the collector refuses new ingest; the local agent buffers up to its limit and then drops. Dropped spans emit a counter. SRE alerts on the counter.
- **Config rollout as a release.** Collector config is in a GitOps repo; changes PR'd; reviewed by at least one platform engineer; rolled with a canary (one replica first, 15-minute soak, then rest).

And the self-observation loop: the collector emits its own metrics via a Prometheus scrape of the collector's `/metrics` endpoint. Dashboards live in Grafana. Alerts live in mod-410. The collector that is blind to itself is the one that fails silently.

## A tenant onboards a new backend: the shape of the change

Walk through the change control for a new tenant arriving with "we want our traces in Langfuse self-hosted".

1. **The tenant provisions the backend.** They deploy Langfuse in their VPC or cloud account. They produce an OTLP endpoint and an API key.
2. **The platform provisions a per-tenant exporter.** Adds `otlp/langfuse_tenantX` to the gateway collector's exporter list, pointing at the tenant's endpoint, with the API key stored in a secret.
3. **The platform adds a routing rule.** The `routing/by_tenant` processor gains a row: `tenantX → otlp/langfuse_tenantX`.
4. **The change is PR'd.** One config file, two stanzas. No application code changes anywhere. The emitting planes never knew.
5. **The rollout is observed.** The platform's dashboards watch export-error rate for the new exporter; back-pressure counters stay clean; the tenant confirms traces arrive.

Compare to the alternative shape — if each tenant required a per-service SDK change, this onboarding would touch every service. The collector tier is what makes backend federation a config change.

## Summary

- The collector tier does the shared work: redaction, tail sampling, tenant routing, semconv-version gating, and per-backend fan-out. The emitting planes speak OTLP and semconv; the collector translates.
- Two-tier topology: node-local agents handle buffering; gateway collectors handle processing and export. The gateway is tier-0 and has its own SRE discipline.
- Content and metadata split at the collector: shared backends receive metadata and attribute hashes; the tenant-partitioned content store receives the full-fidelity prompts and completions.
- Backend integration shapes: OpenLLMetry + Langfuse are OTLP-native; Phoenix is OTLP + OpenInference translation; Datadog LLM Observability is OTLP via the Datadog exporter; Grafana + Prometheus + Tempo is the open-source stack with native OTel ingest.
- Sampling: head at the entry, tail at the gateway, with "keep every interesting trace" policies encoded in the tail-sampling processor.
- A new tenant backend is a collector config change — no code change, no re-instrumentation.

The next chapter writes the conformance tests that keep the whole system honest — the executable contract that fails CI when a plane drifts off semconv, drops `traceparent`, or emits content on the shared pipeline.
