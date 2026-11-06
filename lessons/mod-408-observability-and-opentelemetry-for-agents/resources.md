# Resources — mod-408 (Observability and OpenTelemetry for Agent Fleets)

All references on this page are primary / authoritative public sources at the time of writing (2026-11-06). Each resource is tagged with the chapter(s) it supports. Prefer the primary source over any blog summary — the GenAI semantic conventions, the OpenTelemetry Collector's component inventory, and the LLM observability products in this space move quarter by quarter and secondary sources lag.

## Chapter 01 — The Observability Plane and What It Owns

### Platform-engineering background
- **CNCF — Platforms White Paper**: <https://tag-app-delivery.cncf.io/whitepapers/platforms/>
- **CNCF — Platform Engineering Maturity Model**: <https://tag-app-delivery.cncf.io/whitepapers/platform-eng-maturity-model/>
- **Google SRE Book — Monitoring Distributed Systems**: <https://sre.google/sre-book/monitoring-distributed-systems/>
- **Google SRE Workbook — Alerting on SLOs**: <https://sre.google/workbook/alerting-on-slos/>

### Agent-platform observability context
- **Anthropic — Building Effective Agents**: <https://www.anthropic.com/research/building-effective-agents>
- **OpenAI — A Practical Guide to Building Agents**: <https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf>

### OpenTelemetry — foundational references
- **OpenTelemetry — home**: <https://opentelemetry.io/>
- **OpenTelemetry — specification**: <https://opentelemetry.io/docs/specs/otel/>
- **OpenTelemetry — concepts**: <https://opentelemetry.io/docs/concepts/>
- **OpenTelemetry — SDK environment variables**: <https://opentelemetry.io/docs/specs/otel/configuration/sdk-environment-variables/>

## Chapter 02 — OpenTelemetry Semantic Conventions for Generative AI

### The GenAI conventions themselves
- **Semantic Conventions for Generative AI — index**: <https://opentelemetry.io/docs/specs/semconv/gen-ai/>
- **Semantic Conventions — Generative AI spans**: <https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-spans/>
- **Semantic Conventions — Generative AI events**: <https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-events/>
- **Semantic Conventions — Generative AI metrics**: <https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-metrics/>
- **Semantic Conventions — Generative AI attributes**: <https://opentelemetry.io/docs/specs/semconv/attributes-registry/gen-ai/>
- **Semantic Conventions — repository (source of truth)**: <https://github.com/open-telemetry/semantic-conventions>
- **Semantic Conventions — Weaver (tooling)**: <https://github.com/open-telemetry/weaver>
<!-- needs-research: the GenAI semconv is still formally experimental as of writing; before pinning a version to production, read the current repo tags and stability declarations. -->

### Trace, metric, and log specifications
- **OpenTelemetry — Traces specification**: <https://opentelemetry.io/docs/specs/otel/trace/>
- **OpenTelemetry — Metrics specification**: <https://opentelemetry.io/docs/specs/otel/metrics/>
- **OpenTelemetry — Logs specification**: <https://opentelemetry.io/docs/specs/otel/logs/>
- **OpenTelemetry — OTLP (protocol) specification**: <https://opentelemetry.io/docs/specs/otlp/>

### Agent and conversation attributes
- **Semantic Conventions — agent attributes (within GenAI)**: <https://opentelemetry.io/docs/specs/semconv/gen-ai/>
- **OpenInference specification (Arize)**: <https://arize-ai.github.io/openinference/>
- **OpenInference — repository**: <https://github.com/Arize-ai/openinference>

### OpenLLMetry (Traceloop)
- **OpenLLMetry — documentation**: <https://www.traceloop.com/docs/openllmetry/introduction>
- **OpenLLMetry — repository**: <https://github.com/traceloop/openllmetry>

## Chapter 03 — Distributed Trace Propagation Across Planes

### W3C Trace Context and Baggage
- **W3C Trace Context (Recommendation)**: <https://www.w3.org/TR/trace-context/>
- **W3C Baggage**: <https://www.w3.org/TR/baggage/>
- **OpenTelemetry — context propagation API spec**: <https://opentelemetry.io/docs/specs/otel/context/api-propagators/>
- **OpenTelemetry — baggage API spec**: <https://opentelemetry.io/docs/specs/otel/baggage/api/>

### Propagation across transports
- **OpenTelemetry — messaging conventions**: <https://opentelemetry.io/docs/specs/semconv/messaging/>
- **OpenTelemetry — HTTP conventions**: <https://opentelemetry.io/docs/specs/semconv/http/>
- **OpenTelemetry — Python instrumentation libraries**: <https://opentelemetry-python-contrib.readthedocs.io/>
- **OpenTelemetry — JavaScript (Node) instrumentation**: <https://opentelemetry.io/docs/languages/js/libraries/>
- **OpenTelemetry — Go instrumentation**: <https://opentelemetry.io/docs/languages/go/libraries/>

### MCP transport specifics
- **Model Context Protocol — specification**: <https://modelcontextprotocol.io/specification>
- **Model Context Protocol — repository**: <https://github.com/modelcontextprotocol>

### Durable execution
- **Temporal — observability (OpenTelemetry integration)**: <https://docs.temporal.io/develop/python/observability>
- **Temporal — concepts**: <https://docs.temporal.io/concepts>
- **Inngest — documentation**: <https://www.inngest.com/docs>
- **Restate — documentation**: <https://docs.restate.dev/>

### Sampling
- **OpenTelemetry — sampling spec**: <https://opentelemetry.io/docs/specs/otel/trace/sdk/#sampling>
- **OpenTelemetry — tail sampling processor (contrib)**: <https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/tailsamplingprocessor>

## Chapter 04 — Backend Federation and the Collector Tier

### The OpenTelemetry Collector
- **OpenTelemetry Collector — documentation**: <https://opentelemetry.io/docs/collector/>
- **OpenTelemetry Collector — configuration reference**: <https://opentelemetry.io/docs/collector/configuration/>
- **OpenTelemetry Collector — internal telemetry**: <https://opentelemetry.io/docs/collector/internal-telemetry/>
- **OpenTelemetry Collector — repository (core)**: <https://github.com/open-telemetry/opentelemetry-collector>
- **OpenTelemetry Collector — repository (contrib)**: <https://github.com/open-telemetry/opentelemetry-collector-contrib>
- **OTTL — OpenTelemetry Transformation Language**: <https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/pkg/ottl>

### Backend: Langfuse
- **Langfuse — documentation**: <https://langfuse.com/docs>
- **Langfuse — OpenTelemetry integration**: <https://langfuse.com/docs/opentelemetry>
- **Langfuse — self-hosting guide**: <https://langfuse.com/docs/deployment/self-host>
- **Langfuse — repository**: <https://github.com/langfuse/langfuse>

### Backend: Arize Phoenix
- **Arize Phoenix — documentation**: <https://docs.arize.com/phoenix>
- **Arize Phoenix — tracing concepts**: <https://docs.arize.com/phoenix/tracing/llm-traces>
- **Arize Phoenix — repository**: <https://github.com/Arize-ai/phoenix>
- **OpenInference specification**: <https://arize-ai.github.io/openinference/>

### Backend: Datadog LLM Observability
- **Datadog — LLM Observability documentation**: <https://docs.datadoghq.com/llm_observability/>
- **Datadog — OpenTelemetry documentation**: <https://docs.datadoghq.com/opentelemetry/>
- **Datadog — OpenTelemetry exporter (contrib)**: <https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter/datadogexporter>

### Backend: Grafana + Prometheus + Tempo
- **Prometheus — documentation**: <https://prometheus.io/docs/>
- **Prometheus — remote-write specification**: <https://prometheus.io/docs/concepts/remote_write_spec/>
- **Grafana — documentation**: <https://grafana.com/docs/>
- **Grafana Tempo — documentation**: <https://grafana.com/docs/tempo/latest/>
- **Grafana Tempo — metrics generator**: <https://grafana.com/docs/tempo/latest/metrics-generator/>
- **Prometheus remote write exporter (contrib)**: <https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter/prometheusremotewriteexporter>

### Redaction, PII, and content handling
- **Microsoft Presidio — home**: <https://microsoft.github.io/presidio/>
- **Google Cloud — Sensitive Data Protection (DLP)**: <https://cloud.google.com/sensitive-data-protection>
- **AWS — Amazon Comprehend PII detection**: <https://docs.aws.amazon.com/comprehend/latest/dg/pii.html>

### Content-store object storage (cross-ref mod-407 chapter 04)
- **AWS S3 — Object Lock**: <https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html>
- **Google Cloud Storage — bucket lock**: <https://cloud.google.com/storage/docs/bucket-lock>
- **Cloudflare R2 — documentation**: <https://developers.cloudflare.com/r2/>
- **Rekor (Sigstore transparency log)**: <https://docs.sigstore.dev/rekor/overview/>

## Chapter 05 — Telemetry Conformance Testing

### Semantic-convention tooling
- **OpenTelemetry Weaver**: <https://github.com/open-telemetry/weaver>
- **Semantic Conventions — contribution guide**: <https://github.com/open-telemetry/semantic-conventions/blob/main/CONTRIBUTING.md>

### Testing harness references
- **OpenTelemetry Python — in-memory exporter**: <https://opentelemetry-python.readthedocs.io/en/latest/sdk/trace.export.html#opentelemetry.sdk.trace.export.in_memory_span_exporter.InMemorySpanExporter>
- **OpenTelemetry Python — SDK API**: <https://opentelemetry-python.readthedocs.io/>
- **OpenTelemetry JavaScript — testing**: <https://opentelemetry.io/docs/languages/js/>
- **Testcontainers — documentation**: <https://testcontainers.com/>

### Adjacency: eval-side assertions
- **Promptfoo — documentation**: <https://www.promptfoo.dev/docs/>
- **Promptfoo — custom assertions**: <https://www.promptfoo.dev/docs/configuration/expected-outputs/>
- **Inspect AI — documentation**: <https://inspect.aisi.org.uk/>

### Contract-testing precedent (concept)
- **Pact — documentation**: <https://docs.pact.io/>
- **OpenAPI — specification**: <https://spec.openapis.org/>
- **JSON Schema — documentation**: <https://json-schema.org/>

## Chapter 06 — EU AI Act Article 72: Post-Market Monitoring

### The AI Act text
- **EU AI Act — Regulation (EU) 2024/1689 (EUR-Lex, English text)**: <https://eur-lex.europa.eu/eli/reg/2024/1689/oj/eng>
- **EU AI Act — overview page (European Commission)**: <https://digital-strategy.ec.europa.eu/en/policies/regulatory-framework-ai>
- **EU AI Act — implementation timeline (European Commission)**: <https://digital-strategy.ec.europa.eu/en/policies/ai-act>
- **European AI Office**: <https://digital-strategy.ec.europa.eu/en/policies/ai-office>
<!-- needs-research: the Commission is expected to publish implementing acts under Article 72(3) specifying post-market monitoring plan templates and the Article 73 reporting form; re-verify current status before shipping an export. -->

### Adjacent regulatory artefacts
- **GPAI — General-Purpose AI Code of Practice (Commission)**: <https://digital-strategy.ec.europa.eu/en/policies/general-purpose-ai-code-practice>
- **ISO/IEC 42001 — AI management systems standard**: <https://www.iso.org/standard/81230.html>
- **NIST — AI Risk Management Framework (AI RMF 1.0)**: <https://www.nist.gov/itl/ai-risk-management-framework>
- **NIST — AI RMF Generative AI Profile**: <https://www.nist.gov/itl/ai-risk-management-framework>
- **OECD — AI principles**: <https://oecd.ai/en/ai-principles>

### Data protection intersections (verify with counsel)
- **GDPR — Regulation (EU) 2016/679**: <https://eur-lex.europa.eu/eli/reg/2016/679/oj>
- **EDPB — guidelines**: <https://www.edpb.europa.eu/our-work-tools/general-guidance/guidelines-recommendations-best-practices_en>
- **HIPAA — Privacy Rule overview**: <https://www.hhs.gov/hipaa/for-professionals/privacy/>

### Signing, provenance, and integrity
- **Sigstore — documentation**: <https://docs.sigstore.dev/>
- **cosign — documentation**: <https://docs.sigstore.dev/cosign/overview/>
- **in-toto — attestation framework**: <https://in-toto.io/>
- **SLSA — Supply-chain Levels for Software Artifacts**: <https://slsa.dev/>

## Cross-cutting references

- **OpenTelemetry — Semantic Conventions for Generative AI**: <https://opentelemetry.io/docs/specs/semconv/gen-ai/>
- **OpenTelemetry Collector — configuration**: <https://opentelemetry.io/docs/collector/configuration/>
- **W3C Trace Context**: <https://www.w3.org/TR/trace-context/>
- **Model Context Protocol — specification** (cross-ref mod-403): <https://modelcontextprotocol.io/specification>
- **Inspect AI** (cross-ref mod-407): <https://inspect.aisi.org.uk/>
- **Promptfoo** (cross-ref mod-407): <https://www.promptfoo.dev/docs/>
- **JOB_REQUIREMENTS.md** (at the track root): the normalised list of citations the curriculum is written against.

## Notes on using these references

- Every entry on this page is a **primary** or **authoritative-public** source at the time of writing. The OTel GenAI semconv, the Collector contrib component inventory, and the LLM observability products are the fastest-moving surfaces on the track; if the source has a version tag (SDK version, semconv revision hash, contrib component version) read the version that matches what you deploy against.
- The GenAI semantic conventions are still formally experimental as of writing. The chapters pin a version; the resources page points at the current upstream. Before quoting an attribute name or event shape, read the YAML in `open-telemetry/semantic-conventions/model/gen-ai/` at the revision you target.
- The observability-backend landscape (Langfuse, Phoenix, Datadog LLM Observability, OpenLLMetry) is adjacent, overlapping, and shifting. Each product's native convention and OTel ingestion surface shifts across releases; the collector translation layer is where the platform absorbs the churn.
- The EU AI Act — and specifically the implementing acts under Article 72(3) and the Article 73 reporting form — is actively being refined by the Commission and the AI Office. Regulatory artefacts cited here are refined by delegated acts and amendments; before quoting a specific retention horizon or reporting deadline, verify against the latest text and against counsel at your organisation.
- Benchmark fixtures and model-provider pricing (cross-ref mod-406 chapter 04 and mod-407 chapter 05) move on independent cadences. A per-token rate cited in a dashboard panel or in an exercise worked example may be stale next quarter; the mod-406 price-regression alarm is the operational mitigation.
- Mark `<!-- needs-research: ... -->` on any derived artefact where a citation on this page does not match the version you are shipping against. The flag convention is consistent with mod-403 / 404 / 405 / 406 / 407; it blocks auto-merge, and that is cheaper than silently guessing.
