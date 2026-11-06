# The Observability Plane and What It Owns

The runtime (mod-402) owns the agent loop. The MCP gateway (mod-403) owns the tools. The sandbox (mod-404) runs tool code. The policy plane (mod-405) authorises what the agent does. The model gateway (mod-406) owns the call out to a model. The evaluation plane (mod-407) owns the verdict on whether a change makes any of them worse. Every one of those planes emits telemetry today — spans, logs, metrics, events, cost records — and almost none of it agrees on vocabulary. Which attribute names a prompt. Which field names the model. Which span is the "root" of an agent turn. Which event closes a tool call. That is the slice this module owns.

The naive answer is "each service ships its own structured logs and we glue them together in Kibana". That shape is the default in most platforms up to the first incident where a reviewer needs to answer "which model was called inside this tool invocation that was spawned by that sub-agent". Without a shared semantic contract and shared trace propagation, the reviewer is reading five different log formats and guessing which record is which. Every multi-plane agent platform ends up with an observability plane. The only question is whether it is coherent or accidental.

This chapter fixes the slice. The chapters that follow pick the pieces: the OpenTelemetry Semantic Conventions for Generative AI as the platform's instrumentation contract (02), distributed trace propagation across runtime → model gateway → MCP server → sandbox → eval harness so one request produces one coherent trace (03), backend integration (OpenLLMetry, Langfuse, Arize Phoenix, Datadog LLM Observability, Grafana + Prometheus) against the same semantic layer (04), conformance tests that keep stream-aligned teams' telemetry honest (05), and the EU AI Act Article 72 post-market-monitoring evidence slice produced from the trace pipeline (06).

## What the observability plane owns

The plane sits between the runtime (mod-402) that produces spans on every agent turn, the MCP gateway (mod-403) that produces spans on every tool invocation, the sandbox (mod-404) that produces spans on every code execution, the policy plane (mod-405) that produces spans on every authorisation decision, the model gateway (mod-406) that produces spans on every provider call, the evaluation plane (mod-407) that produces spans on every eval row, the safety plane (mod-411) that produces spans on every guardrail decision, and — on the far side — the tenant backends (Langfuse, Phoenix, Datadog, Grafana) that read the resulting telemetry. The plane owns:

- **The semantic contract.** A pinned set of attribute names, event shapes, and span kinds every plane emits — rooted in the [OpenTelemetry Semantic Conventions for Generative AI](https://opentelemetry.io/docs/specs/semconv/gen-ai/), extended only where the platform's own facts require it, versioned the way an API is versioned. Chapter 02.
- **The SDK and collector configuration.** A paved-road OTel SDK configuration — language shim, resource attributes, sampler, exporter, redactor — that every plane's codebase imports. A collector topology (agent → gateway collector → tenant-routed exporters) that routes the same semantic stream to any tenant's backend without re-instrumentation. Chapter 04.
- **Context propagation.** A `traceparent` / `tracestate` ([W3C Trace Context](https://www.w3.org/TR/trace-context/)) pass-through across every plane boundary — HTTP, gRPC, message queue, durable-execution step, sandboxed subprocess — so that one external agent request produces one coherent trace from entry to tool completion. Chapter 03.
- **The redaction and PII posture at the telemetry seam.** Prompts, completions, tool inputs, and tool outputs carry potential PII and tenant secrets. The plane decides what is recorded where (full prompt in session store, hashed in metrics, elided in cross-tenant aggregates) and enforces it in the collector before export. Chapter 04.
- **The conformance-test suite.** A declarative contract — "a span with `gen_ai.operation.name=chat` MUST carry `gen_ai.request.model` and `gen_ai.response.model`; any `tool_use` event MUST carry `gen_ai.tool.name` and `gen_ai.tool.call.id`" — that stream-aligned teams run in CI against their own instrumentation. Chapter 05.
- **The post-market monitoring evidence slice.** The EU AI Act [Article 72](https://eur-lex.europa.eu/eli/reg/2024/1689/oj) obligation for providers of high-risk systems to maintain post-market monitoring is partially satisfied by the trace pipeline the plane already runs. The plane publishes a documented view over its own telemetry that an auditor can read. Chapter 06.
- **Backend federation.** The plane does not pick one backend. It ships the same semantic stream to OpenLLMetry → Langfuse, Phoenix, Datadog LLM Observability, and a Prometheus + Grafana + Tempo stack, so each tenant (or each use case — product analytics, ops, compliance) picks its preferred surface without the emitting team re-instrumenting. Chapter 04.
- **The paved-road onboarding.** A `platform-otel init` command that wires a new service into the SDK, the collector, the redactor, and the conformance tests in one invocation. Cross-reference mod-412.

The plane does **not** own:

- **Eval methodology or verdicts.** The evaluation plane (mod-407) owns what a regression is. The observability plane produces the trace that lets someone investigate one; it does not say whether a change is better or worse.
- **Policy decisions.** The policy plane (mod-405) authors deny / permit rules. The observability plane records the decision as an event on a span; it does not make it.
- **SRE on-call, SLOs, alerting runbooks.** Those sit in mod-410. The observability plane provides the signals; the SRE operating cadence consumes them.
- **The content-safety verdict.** The safety plane (mod-411) owns guardrail outcomes. The observability plane records the verdict as an event; it does not author the policy.
- **Dashboard methodology for product analytics.** Which business metrics matter and how they roll up to leadership is product engineering, not platform engineering.
- **Model and inference-level introspection.** Attention weights, hidden states, token probabilities — that is `ai-infra-inference-engineer` (level 35) and researcher territory, not platform telemetry.

Drift off this slice and you recreate the eval plane at the dashboard layer (usually with worse statistics) or the policy plane at the collector layer (usually with silent denials nobody can audit). Scope policing is a feature of this module.

## Why one semantic contract and not per-plane conventions

A few concrete questions that are hard to answer without a shared semantic contract and trivial with one. If your current telemetry setup can answer each in a single query, you have an observability plane. If any one requires joining three log sources by timestamp heuristics, you do not yet.

- **End-to-end trace for one agent request.** A customer at 2026-11-03T14:22Z filed a complaint about an agent output. One external request. Can you pull a single trace that shows the runtime turn, the model-gateway call, every MCP tool invocation, every policy decision, every sandbox run, and the eval row the trace was replayed against — with timestamps, token counts, and tool arguments — from one `trace_id`?
- **Model attribution across a tool loop.** An agent made six model calls, two tool calls, and one sub-agent handoff on one turn. Which calls used which model, which tenant quota, which cached-prefix bucket? Can every one of them show up in a single query over `gen_ai.request.model` and `gen_ai.response.model` without joining to the model gateway's billing log?
- **Cross-tenant cost comparison.** Tenant A spent 4× tenant B's token budget last week. Can you aggregate `gen_ai.usage.input_tokens` and `gen_ai.usage.output_tokens` across every plane for each tenant from one metrics store, or do you have to crack open each team's dashboards?
- **Regression replay with telemetry.** The eval plane (mod-407) replays yesterday's production trace against a candidate model. Can the replay produce a trace that is structurally comparable to the original — same span kinds, same attribute names, same event shapes — so a reviewer can diff them without a transformation step?
- **Compliance audit.** An auditor asks "for this high-risk use case, show me the serious-incident candidate traces from last quarter — the ones where the agent returned an output a guardrail later flagged." Can you answer from a saved trace query, or do you have to write custom glue?

These are the capabilities the plane delivers. They separate "we have logs" from "we have an observability plane". Chapters 02–06 build each.

## The two deployment shapes

Two deployment shapes dominate; pick deliberately, do not drift.

### SDK-only (library in every service)

The observability plane ships as a language shim over the OpenTelemetry SDKs plus a shared semantic-conventions package. Every service imports the shim, which configures the SDK, applies resource attributes, and installs an OTLP exporter pointed at whatever backend the tenant wants. No collector; the SDK talks OTLP straight to the backend.

- **Strengths.** Zero added service to operate. Easy to adopt incrementally. Backends receive native OTLP.
- **Weaknesses.** Redaction, sampling, and tenant routing all have to be correct in every service. Rolling a redaction fix means rolling every service. Cross-tenant telemetry federation (one agent request spanning two tenants' agents) is impossible to coordinate from inside the emitting process.
- **Where it fits.** Early-stage platforms, single-tenant deployments, use cases where the tenant owns its backend.

### SDK + collector tier (shared pipeline)

The observability plane ships the SDK plus a two-tier collector topology: a per-node / per-pod [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/) agent, and a shared gateway collector. The SDK speaks OTLP to the local agent; the local agent forwards to the gateway; the gateway applies redaction, sampling, tenant routing, and multi-exporter fan-out before shipping to the tenant backends.

- **Strengths.** Redaction and sampling are a central concern, not a per-service concern. Backend changes (adding Phoenix for one tenant, migrating another off Datadog) are a gateway config change, not a service redeploy. Aggregations (fleet-wide cost, latency, token counts) happen on a coherent stream. Compliance holds the gateway accountable, not twenty teams.
- **Weaknesses.** Added network hops (two, typically single-digit milliseconds in-cluster). The collector is now a tier-0 dependency; if the collector fails, the platform goes blind. Headroom and HA need to be a thing you plan for.
- **Where it fits.** Multi-tenant, multi-plane platforms past the prototype stage. This is the shape mod-408 writes against.

A **hybrid** shape — SDK speaks OTLP directly to local agent collectors on dev / CI, and through the gateway in production — is where most mature platforms end up. Chapter 04 details the deployment picture.

## The threat model this plane defends against

Four classes of failure dominate what the observability plane sees. They are what chapters 02–06 build mitigations for.

### Semantic drift across planes

The runtime team names a model attribute `model_name`. The model gateway names it `provider.model`. The MCP gateway names it `model_identifier`. Every plane is correct in isolation; together they are a cross-team join nightmare, and every dashboard has a per-plane alias table that goes stale the moment a convention shifts.

The plane's defences:

- **One canonical vocabulary.** The [OpenTelemetry Semantic Conventions for Generative AI](https://opentelemetry.io/docs/specs/semconv/gen-ai/) pin the names. Every plane emits `gen_ai.request.model`, `gen_ai.response.model`, `gen_ai.operation.name`, not a local synonym. Chapter 02.
- **Versioned conventions.** The semconv package is pinned to a version; a plane upgrading to a new semconv revision ships the upgrade as a reviewable change, not a silent rename. Chapter 02.
- **Conformance tests in CI.** The semantic contract is executable. A plane that emits `model_name` instead of `gen_ai.request.model` fails CI. Chapter 05.

### Broken trace propagation

The runtime starts a trace. The call to the model gateway is over HTTP with `traceparent` set — so far so good. The MCP gateway invokes a tool through a message queue, but the queue consumer does not extract `traceparent` from the message headers. The sandbox exec'd by the tool is a fresh subprocess with no inherited context. The eval harness replays the trace but starts a new root. Result: five disconnected trace segments. A reviewer cannot answer "what happened on this agent request" without timestamp matching.

The plane's defences:

- **Context propagation at every hop.** HTTP (`traceparent` header), gRPC (binary carrier), queues (headers / message metadata), durable-execution steps (step input metadata), subprocess (env var passthrough). Chapter 03.
- **Instrumentation libraries wrapped at the SDK.** The platform's SDK wraps the HTTP client, the queue client, the durable-execution client, and the subprocess spawner. A team cannot opt out of propagation without opting out of the SDK. Chapter 03.
- **A propagation conformance test.** An end-to-end synthetic trace runs on a schedule across every plane; the test fails if any hop drops `traceparent`. Chapter 05.

### Telemetry PII leakage

Prompts carry user text. Completions carry model output. Tool arguments carry call metadata. Model responses carry sub-document quotes that may include tenant secrets. Dumping all of it to every observability backend is a GDPR / HIPAA / tenant-contract breach waiting to happen. Hashing it loses the thing a reviewer needs to debug. Dropping it loses the thing a judge needs to score.

The plane's defences:

- **A redactor at the collector.** Full prompt and completion content is captured on a tenant-partitioned content store (session store with tight RBAC); the spans that ship to shared backends carry hashes and metadata only. Chapter 04.
- **Semantic-convention-aware redaction.** The redactor knows which attributes are content (`gen_ai.prompt.*`, `gen_ai.completion.*`, tool call arguments) and which are metadata (`gen_ai.request.model`, `gen_ai.usage.*`). It redacts the former; it preserves the latter. Chapter 04.
- **Tenant-partitioned retention policies.** Different tenants have different retention windows; the collector enforces them before export. Chapter 04.

### Collector failure modes that go blind

The collector drops events on back-pressure. The collector reorders spans. The collector double-exports. The collector shards wrong and splits a trace across two exporters. The collector is itself a tier-0 service that needs its own observability — the irony is a known pattern.

The plane's defences:

- **Per-pipeline health signals.** The collector emits its own `otelcol_*` Prometheus metrics ([OpenTelemetry Collector internal telemetry](https://opentelemetry.io/docs/collector/internal-telemetry/)). An external scraper watches for queue-drop and export-failure rates. Mod-410 owns the alerting.
- **Tail-sampling at the gateway, head-sampling at the SDK only for non-root spans.** Root traces are either fully sampled or sampled by a deterministic policy that preserves end-to-end coverage for serious-incident candidates. Chapter 04.
- **Replay from the content store.** The session store (full-fidelity content) is the authoritative record; the trace pipeline is the index. A collector outage loses index, not evidence. Chapter 04 and cross-reference mod-407 chapter 04.

## The telemetry record

Pin the shape before writing the plane. Every agent interaction produces one or more `TraceRecord`s — one per span, aggregated by `trace_id`. The semantic-convention attributes live on each span; the record is a view over the aggregate. The record is invariant across OpenLLMetry, Langfuse, Phoenix, Datadog, Grafana, and any custom backend the plane adopts later.

```json
{
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "entry_span_id": "00f067aa0ba902b7",
  "tenant_id": "acme-financial",
  "session_id": "sess_01HV9...",
  "turn_id": "turn_00005",
  "agent_id": "writer-v1",
  "agent_revision": "git:abcd1234",
  "agent_framework": "custom-runtime",
  "spans": [
    {
      "span_id": "00f067aa0ba902b7",
      "parent_span_id": null,
      "name": "agent.turn",
      "kind": "server",
      "attributes": {
        "gen_ai.operation.name": "chat",
        "gen_ai.agent.id": "writer-v1",
        "gen_ai.conversation.id": "sess_01HV9...",
        "session.id": "sess_01HV9...",
        "service.name": "runtime"
      }
    },
    {
      "span_id": "1a4e96c13c7d4f32",
      "parent_span_id": "00f067aa0ba902b7",
      "name": "model_gateway.chat",
      "kind": "client",
      "attributes": {
        "gen_ai.operation.name": "chat",
        "gen_ai.request.model": "platform:writer-v1",
        "gen_ai.response.model": "claude-sonnet-4-6",
        "gen_ai.system": "anthropic",
        "gen_ai.usage.input_tokens": 1842,
        "gen_ai.usage.output_tokens": 273,
        "gen_ai.response.finish_reasons": ["tool_use"],
        "service.name": "model-gateway"
      }
    },
    {
      "span_id": "7b3210dcab4511ac",
      "parent_span_id": "00f067aa0ba902b7",
      "name": "mcp.tool.call",
      "kind": "client",
      "attributes": {
        "gen_ai.operation.name": "execute_tool",
        "gen_ai.tool.name": "search",
        "gen_ai.tool.call.id": "call_a1b2c3",
        "service.name": "mcp-gateway"
      }
    }
  ],
  "content_store_uri": "s3://sessions/4bf9.../session.jsonl",
  "redaction": {
    "prompt_captured_at_collector": false,
    "prompt_captured_at_content_store": true,
    "redactor_version": "v3"
  },
  "sampling": {
    "decision": "recorded",
    "reason": "serious_incident_candidate"
  }
}
```

Five invariants the record enforces:

1. **Every span uses the canonical semconv vocabulary.** `gen_ai.request.model`, not `model_name`. `gen_ai.tool.name`, not `tool_identifier`. Chapter 02.
2. **Every span carries `tenant_id`, `session_id`, and `agent_id` as attributes or resource attributes.** Multi-tenant partitioning and session-level analysis are query-time, not join-time. Chapter 04.
3. **Content is captured at the content store, not at the shared trace pipeline.** The record's `content_store_uri` is a pointer; the pipeline carries hashes and metadata. Chapter 04.
4. **`traceparent` propagated unbroken from the entry span.** All descendant spans share `trace_id`; `parent_span_id` references point into the same trace. Chapter 03.
5. **Sampling decision is explicit on the record.** A reviewer reading the record knows whether this trace was captured because it was sampled, because it was a serious-incident candidate, or because the tenant policy requires full capture. Chapter 04.

Chapters 02–06 build the pieces that produce this record.

## Where the plane physically sits

A typical shape for the physical deployment:

- **The semconv package.** A small language-neutral vocabulary file (YAML) and per-language code generators built on [OpenTelemetry's semantic-conventions pipeline](https://github.com/open-telemetry/semantic-conventions). Published as a library under a pinned version.
- **The platform OTel SDK shim.** One package per supported language (Python, TypeScript, Go are the mod-412 defaults). Configures the SDK, applies resource attributes, installs instrumentation libraries, and wires the exporter.
- **Node-local collector agents.** An [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/) running on each node or as a sidecar. Receives OTLP from local SDKs; forwards to the gateway collector.
- **The gateway collector tier.** A horizontally scaled collector deployment. Applies redaction, sampling, tenant routing, and fan-out to backend exporters.
- **The content store.** A tenant-partitioned object store (S3 / GCS / R2) for full-fidelity prompts, completions, tool arguments. RBAC gated.
- **The backend fan-out.** OTLP exporters to OpenLLMetry → Langfuse, Phoenix, Datadog LLM Observability, and a Prometheus + Grafana + Tempo stack. Chapter 04.
- **The conformance test runner.** A library and CI integration that runs the semantic-contract suite against any service's emitted telemetry. Chapter 05.
- **The Article 72 evidence view.** A query and export surface over the content store and the aggregate metrics. Chapter 06.

If you cannot draw this topology on a whiteboard and name the box every attribute flows through, chapter 02 is the thing to read next.

## The three obligations

Pin the plane to three non-negotiable obligations. Everything in chapters 02–06 is a decision about how to deliver them.

1. **"One agent request produces one coherent trace from entry to tool completion."** If a reviewer cannot pull all the work done on behalf of one external request from one `trace_id`, the pipeline has failed its primary job. Context propagation is not optional. Chapter 03.
2. **"Every plane speaks the same semantic vocabulary at a pinned version."** If a dashboard query needs a per-plane alias table to join two spans, the semantic contract is incomplete. Chapter 02 and chapter 05.
3. **"Full-fidelity content is tenant-partitioned, time-bounded, and queryable for post-market monitoring."** The content store is the authoritative record; retention policies are tenant-specific; the Article 72 view is a documented, auditable export. Chapter 04 and chapter 06.

A plane that cannot defend all three is a dashboard, not an observability plane.

## Summary

- The observability plane owns the semantic contract, the SDK and collector configuration, trace propagation, telemetry redaction, the conformance tests, backend federation, and the post-market monitoring evidence view. It does not own eval verdicts, policy decisions, SRE on-call, content-safety verdicts, or model-level introspection.
- Two deployment shapes exist — SDK-only and SDK + collector tier; a hybrid is where most mature platforms land. The plane in this module is written against the SDK + collector shape.
- The failures the plane defends against are semantic drift across planes, broken trace propagation, telemetry PII leakage, and collector failure modes that go blind. Chapters 02–06 build the mitigations.
- Every interaction produces a `TraceRecord` keyed on `trace_id`: canonical semconv attributes, tenant / session / agent identifiers, pointers to content-store full fidelity, sampling decision, and redaction posture. Every chapter adds a piece to this record.
- Three non-negotiable obligations: one request → one coherent trace; one vocabulary pinned across planes; tenant-partitioned content with a documented post-market monitoring view.

The next chapter opens the first piece — the OpenTelemetry Semantic Conventions for Generative AI, and why the platform's instrumentation contract is nothing more and nothing less than "emit what semconv says, versioned, in every plane".
