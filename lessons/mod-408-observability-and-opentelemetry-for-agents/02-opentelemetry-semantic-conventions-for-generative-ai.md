# OpenTelemetry Semantic Conventions for Generative AI

Chapter 01 pinned the slice. The first job inside the slice is the vocabulary: which attribute names, event shapes, and span kinds every plane emits. That vocabulary is not something the platform invents. The [OpenTelemetry Semantic Conventions for Generative AI](https://opentelemetry.io/docs/specs/semconv/gen-ai/) are the public standard, maintained by the [OpenTelemetry project's Semantic Conventions working group](https://github.com/open-telemetry/semantic-conventions), and they are what the platform's instrumentation contract rests on.

This chapter walks the standard, picks the pieces the platform actually commits to, writes down the extensions the platform must author beyond it, and defines versioning rules so the vocabulary evolves without silent breakage. The output is what chapter 05's conformance tests check against.

> **Status note.** The OTel GenAI conventions are, as of writing, still formally in `development` / `experimental` status in the [semantic-conventions repo](https://github.com/open-telemetry/semantic-conventions/tree/main/model/gen-ai). They are the best public contract that exists, and the major LLM observability vendors (Langfuse, Phoenix, Datadog, OpenLLMetry) emit or consume them. But the exact attribute inventory shifts between spec revisions. <!-- needs-research: confirm the current stability level and attribute inventory against the latest tag of open-telemetry/semantic-conventions before committing a semconv version to production. --> Chapter 05's conformance tests pin a version and fail CI when a plane drifts; chapter 04's collector enforces the pinned version at export.

## What semconv gives you

OpenTelemetry Semantic Conventions define three kinds of thing:

- **Resource attributes** describe the entity emitting the telemetry — a service, a process, a host. They live on every span the entity emits. `service.name`, `service.version`, `deployment.environment`, and GenAI additions like `gen_ai.agent.id` / `gen_ai.agent.name` are the ones that matter here.
- **Span attributes** describe one operation — a chat completion, an embedding call, a tool invocation. `gen_ai.operation.name`, `gen_ai.request.model`, `gen_ai.response.model`, `gen_ai.usage.*`, `gen_ai.tool.name`, `gen_ai.tool.call.id`.
- **Events** (span events in OTel parlance, or increasingly span-scoped log records) describe something that happened inside the operation. GenAI defines a family of events for prompt messages, completion messages, tool calls, and tool results.

All three are pinned to names. A platform that follows the conventions emits `gen_ai.request.model`, not `model_name` or `model_identifier`. The conformance tests (chapter 05) check the names; the dashboards (chapter 04) query the names; the audit view (chapter 06) joins on the names. Chapter 01's invariant that every plane speaks the same vocabulary is a direct statement of "pick semconv, pin the version, fail CI when something drifts".

## The GenAI span: shape and required attributes

A GenAI operation — a chat completion, a text completion, an embedding call — is represented as one span. The [`Generative AI spans` convention](https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-spans/) pins the shape. Core attributes, with the semconv names:

| Attribute | Meaning | Platform contract |
| --- | --- | --- |
| `gen_ai.operation.name` | The kind of operation — `chat`, `text_completion`, `embeddings`, `execute_tool`. | Required on every GenAI span. |
| `gen_ai.system` | The provider identifier — `openai`, `anthropic`, `aws.bedrock`, `azure.ai.openai`, `vertex_ai`, etc. | Required on every provider-facing span; omit for platform-internal spans (e.g. the runtime's turn span). |
| `gen_ai.request.model` | The model identifier *as requested*. On the platform this is the internal identifier (`platform:writer-v1`), not the provider's name. | Required. |
| `gen_ai.response.model` | The model identifier *as the provider responded with*. This is the provider's string (`claude-sonnet-4-6`, `gpt-4o-2024-08-06`). | Required when the provider returns one. |
| `gen_ai.usage.input_tokens` | Input token count. | Required for billable calls. |
| `gen_ai.usage.output_tokens` | Output token count. | Required for billable calls. |
| `gen_ai.response.finish_reasons` | Array of finish reasons — `stop`, `length`, `tool_use`, `content_filter`. | Required when the provider returns finish reasons. |
| `gen_ai.response.id` | The provider-side response id (`msg_01A2b3C4d5E6` from Anthropic, `chatcmpl-...` from OpenAI). | Required when the provider returns one. |
| `server.address`, `server.port` | The provider endpoint actually called. | Required on provider-facing client spans. |
| `error.type` | Error class on failure (mapped from provider error codes). | Required on failed spans. |

Platform-scoped attributes the platform authors as extensions (naming them under `platform.*` and NOT masquerading as semconv):

- `platform.tenant_id` — the tenant on whose behalf the call runs.
- `platform.quota_bucket` — the quota bucket the request drew against (mod-406 chapter 04).
- `platform.cache_partition` — the prompt-cache partition (mod-406 chapter 05).
- `platform.price_table_version` — the price table used to compute cost (mod-406 chapter 04).

An important discipline: **do not shadow a semconv attribute with a platform-prefixed one**. If semconv defines `gen_ai.response.id`, do not also emit `platform.response_id` carrying the same value. The join is harder, the schema is noisier, and the first reviewer who pulls the trace wastes time deciding which to trust.

### Concrete example: one provider call from the model gateway

A pseudocode sketch of the model-gateway (mod-406) emitting a chat span. The example uses the Python OTel SDK API; the attribute names are what matters.

```python
with tracer.start_as_current_span(
    name="model_gateway.chat",
    kind=SpanKind.CLIENT,
    attributes={
        "gen_ai.operation.name": "chat",
        "gen_ai.system": "anthropic",
        "gen_ai.request.model": request.platform_model_id,     # platform:writer-v1
        "server.address": "api.anthropic.com",
        "server.port": 443,
        "platform.tenant_id": tenant.id,
        "platform.quota_bucket": tenant.quota_bucket_id,
    },
) as span:
    response = provider.call(request)
    span.set_attribute("gen_ai.response.model", response.model)        # claude-sonnet-4-6
    span.set_attribute("gen_ai.response.id", response.id)              # msg_01A2b3C4d5E6
    span.set_attribute("gen_ai.usage.input_tokens", response.usage.input_tokens)
    span.set_attribute("gen_ai.usage.output_tokens", response.usage.output_tokens)
    span.set_attribute(
        "gen_ai.response.finish_reasons",
        [b.type for b in response.content if b.type in {"text", "tool_use"}],
    )
```

A reviewer reading this span can answer: which model was asked for, which model answered, how many tokens, which tenant paid, which quota bucket drew down, which price table version applied. A dashboard reading these spans can aggregate across every plane in the fleet using the same attribute names.

## The tool-call span and events

Agent platforms live and die by tool calls. The MCP gateway (mod-403) invokes a tool per `tool_use` the model emits; the sandbox (mod-404) runs the tool code; the policy plane (mod-405) authorises or refuses. Semconv has a dedicated operation for this: `gen_ai.operation.name=execute_tool`. The [tool-execution span convention](https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-spans/#execute-tool-span) defines the attributes.

Core attributes on a tool-execution span:

| Attribute | Meaning |
| --- | --- |
| `gen_ai.operation.name` | `execute_tool`. |
| `gen_ai.tool.name` | The tool's name as the model referenced it (e.g. `search`, `read_file`). |
| `gen_ai.tool.type` | Semconv-defined tool category — `function`, `extension`, or custom. |
| `gen_ai.tool.call.id` | Correlation id from the model's `tool_use` block. The same id appears on the originating chat span's event. |

Platform extensions carried on tool-execution spans:

- `platform.tool_registry_version` — the MCP registry version that served the tool schema (mod-403 chapter 02).
- `platform.policy_decision_id` — the policy-plane decision id (mod-405) that permitted or denied the call.
- `platform.sandbox_pod_id` — the sandbox identifier (mod-404) if the tool ran in a sandbox.

### Linking the chat span to the tool span: the `tool_call` event

The chat span emits a `gen_ai.tool.message` span event (or, in semconv's log-record formulation, a scoped log record) per tool call the model requests. The event carries the same `gen_ai.tool.call.id` as the subsequent tool-execution span, which lets a reviewer answer "did this specific `tool_use` from the model actually lead to a tool-execution span, and what did that span do". Without the correlation, a trace with concurrent tool calls becomes an unparseable pile.

```python
# Inside the chat span:
span.add_event(
    name="gen_ai.tool.message",
    attributes={
        "gen_ai.tool.call.id": tool_use.id,
        "gen_ai.tool.name": tool_use.name,
    },
)

# The runtime then launches the tool-execution span as a child:
with tracer.start_as_current_span(
    name="mcp.tool.call",
    kind=SpanKind.CLIENT,
    attributes={
        "gen_ai.operation.name": "execute_tool",
        "gen_ai.tool.name": tool_use.name,
        "gen_ai.tool.call.id": tool_use.id,
        "platform.policy_decision_id": policy.decision_id,
    },
) as tool_span:
    ...
```

Two spans; one correlation id; one coherent trace. A dashboard query for "all tool calls on this trace" is a filter by `gen_ai.operation.name=execute_tool` and `trace_id=…`, not a timestamp heuristic.

## Prompt and completion events

Semconv represents the *content* of a conversation — system messages, user prompts, assistant completions, tool-use blocks, tool results — as a sequence of events (or logs, depending on which part of the standard one reads) on the chat span. The relevant event names and their payload keys evolve with semconv revisions; the current naming is in the [`events` subsection of the GenAI conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-events/).

<!-- needs-research: the event schema (names like `gen_ai.user.message`, `gen_ai.assistant.message`, `gen_ai.tool.message`, `gen_ai.system.message`, `gen_ai.choice`) has been refined across semconv revisions; before committing to a payload shape, re-read the current spec and the `semantic-conventions` repo's `model/gen-ai/events.yaml`. -->

The platform's rule for prompt / completion events is firm:

- **Capture the structure, not the content, on the shared pipeline.** The event tells the dashboard the shape of the conversation (how many turns, which roles, which tool calls). The content (the prompt text, the completion text, the tool arguments) is a pointer to the content store, not an inline payload.
- **Full-fidelity content lives in the tenant-partitioned content store.** Chapter 04 details. A session-store URI appears on the root span as an attribute; the event references it.
- **Redaction decisions are semconv-aware.** A redactor that strips `gen_ai.prompt.*` content attributes from spans but preserves `gen_ai.usage.*` knows the vocabulary, not "any string that looks long".

A sketch of a prompt event on the chat span:

```python
span.add_event(
    name="gen_ai.user.message",
    attributes={
        "gen_ai.message.index": i,
        "gen_ai.message.role": "user",
        "platform.content_ref": f"s3://sessions/{trace_id}/msg_{i}.json",
        # "gen_ai.message.content": prompt_text,    # NOT emitted — content lives in the store.
    },
)
```

The redactor's rule is readable as code: "events named `gen_ai.*.message` must not carry `.content` or `.body` attributes on the shared pipeline; if one is present, drop it and log a redaction event." Chapter 04 makes that enforceable at the gateway collector.

## Metrics: the aggregates semconv already names

Alongside spans and events, semconv defines a handful of GenAI metrics ([`Generative AI metrics`](https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-metrics/)). The ones the platform will emit:

- `gen_ai.client.token.usage` — a histogram of input / output tokens per call, tagged by `gen_ai.request.model`, `gen_ai.response.model`, `gen_ai.operation.name`, `gen_ai.system`.
- `gen_ai.client.operation.duration` — a histogram of per-operation latency.
- Provider-level metric counters where semconv extends them (`gen_ai.server.*` on the model gateway side).

Two platform extensions worth authoring as metrics, not just span attributes (because the aggregation story is different):

- `platform.gen_ai.cost_usd` — a sum, tagged the same way as `gen_ai.client.token.usage` plus `platform.tenant_id` and `platform.price_table_version`.
- `platform.gen_ai.quota_decision` — a counter tagged by decision (`permitted`, `throttled`, `denied`), tenant, bucket.

Prometheus + Grafana (chapter 04) reads these metrics directly; the Langfuse / Phoenix / Datadog surfaces reconstruct them from the span stream. Chapter 04 details which backend expects which form.

## Agent-level attributes: naming the thing under observation

GenAI semconv recently added a vocabulary for the agent itself ([`Generative AI agent` conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/#agent-attributes)): `gen_ai.agent.id`, `gen_ai.agent.name`, `gen_ai.conversation.id`. The platform's convention:

- `gen_ai.agent.id` — the stable platform identifier (`writer-v1`). Pinned to the runtime's registration; does not change across revisions.
- `gen_ai.agent.name` — human-readable name. May change; the dashboards use `agent.id` for joins.
- `gen_ai.conversation.id` — the platform's `session_id`. Pins a cross-turn conversation to one identifier.
- `platform.agent.revision` — git SHA or version string. The agent id is stable; the revision is not. A regression investigation needs both.

Every plane emits all four (as resource attributes where the plane is agent-scoped, as span attributes where it is not).

## Versioning the vocabulary

Semconv is versioned; the platform pins the version. The rules:

1. **One semconv version per release of the platform SDK shim.** A service upgrading to a new shim is upgrading to a new semconv version. Both changes ship together, reviewed together.
2. **Mixed-version fleets are temporary and bounded.** During a rollout the fleet carries two versions; the collector tier (chapter 04) handles the common attributes and gates the diverging ones per emitter. The window is days, not months.
3. **Attribute-rename migrations get a dual-emit window.** When semconv renames `gen_ai.prompt_tokens` to `gen_ai.usage.input_tokens` (a historical rename), the SDK emits both for one release so downstream readers can migrate at their own pace.
4. **Removal is a breaking change.** Dropping a platform extension is a reviewed change, announced, and gated by the conformance test one release in advance.

The semconv rev the platform pins is a visible fact — on the service's `/health` endpoint, on the resource attributes (`telemetry.sdk.version`, `platform.semconv_version`), in the paved-road SDK's release notes. A dashboard that depends on an attribute that moved semconv versions can read the resource attribute and decide which query shape to run.

## What the platform extends semconv with

Four classes of extension the platform owns, with naming discipline.

- **Tenant and quota attributes** (`platform.tenant_id`, `platform.quota_bucket`, `platform.price_table_version`). These are platform facts, not provider facts; they do not belong under `gen_ai.*`. Everything multi-tenant hangs off them.
- **Policy decision references** (`platform.policy_decision_id`, `platform.policy_version`). The policy plane (mod-405) owns the decisions; the observability plane carries the reference so a reviewer can pull the decision record.
- **Content-store pointers** (`platform.content_ref`). Chapter 01's invariant — content lives in the store, not the pipeline — is a single attribute pointing into the store.
- **Cost attribution** (`platform.gen_ai.cost_usd`). Computed by the model gateway (mod-406 chapter 04). Attached as a span attribute and emitted as a metric so FinOps can aggregate from either direction.

Naming discipline: **platform extensions always live under `platform.*`**. Never prefix a platform extension with `gen_ai.*`; the project's semantic-conventions working group owns that namespace, and shadow-naming makes future upgrades a merge nightmare. If a platform extension turns out to be useful enough that semconv adopts it, the SDK shim emits the new semconv name and keeps the `platform.*` alias for one release.

## Conformance expectations on each plane

This chapter is the vocabulary; chapter 05 is the executable contract. The rough shape of what the contract requires per plane:

- **Runtime (mod-402).** Emits `agent.turn` spans with `gen_ai.operation.name=chat`, `gen_ai.agent.id`, `gen_ai.conversation.id`, `service.name`, `platform.tenant_id`. One span per agent turn.
- **Model gateway (mod-406).** Emits chat / text_completion / embeddings spans with the full semconv inventory above, plus `platform.quota_bucket`, `platform.price_table_version`, and `platform.gen_ai.cost_usd`.
- **MCP gateway (mod-403).** Emits `execute_tool` spans with `gen_ai.tool.name`, `gen_ai.tool.call.id`, `platform.tool_registry_version`.
- **Sandbox (mod-404).** Emits a child span under the tool-execution span carrying `platform.sandbox_pod_id`, a code-execution kind, and resource usage.
- **Policy plane (mod-405).** Emits a span (or span event on the triggering span) carrying `platform.policy_decision_id`, `platform.policy_version`, and the decision verdict.
- **Evaluation plane (mod-407).** Emits `agent.turn` spans during replay with `platform.replay_source_trace_id` pointing at the replayed production trace.
- **Safety plane (mod-411).** Emits span events on the chat span carrying guardrail verdicts.

A plane that fails any of these on CI fails to merge. Chapter 05 is the mechanism.

## A note on the OpenInference convention

The [OpenInference specification](https://arize-ai.github.io/openinference/) — Arize's trace convention — predates much of the current GenAI semconv shape and remains the native language of Arize Phoenix. OpenInference and semconv overlap heavily but are not identical: attribute names differ, event shapes differ, the agent model differs. The platform's rule:

- **Semconv is the platform's primary vocabulary.** All instrumentation emits semconv.
- **OpenInference is a compatibility concern at the exporter.** Chapter 04 details the Phoenix exporter path, which translates semconv attributes into OpenInference attributes where they diverge. The SDK does not emit OpenInference directly; the collector translates.

The two standards are converging — the [OpenInference project](https://github.com/Arize-ai/openinference) tracks semconv for new attributes, and the OTel GenAI working group reviews OpenInference proposals. The platform assumes convergence; the exporter translator is scoped as a temporary artefact. <!-- needs-research: as of writing, the convergence story is incomplete and the exporter translator is still load-bearing. Re-evaluate before each SDK shim release. -->

## Summary

- The vocabulary is the [OpenTelemetry Semantic Conventions for Generative AI](https://opentelemetry.io/docs/specs/semconv/gen-ai/). Every plane emits `gen_ai.*` attributes, not local synonyms.
- A GenAI chat / text_completion / embeddings span carries `gen_ai.operation.name`, `gen_ai.system`, `gen_ai.request.model`, `gen_ai.response.model`, `gen_ai.usage.*`, `gen_ai.response.*`, `server.address`. A tool-execution span carries `gen_ai.tool.name`, `gen_ai.tool.call.id`. Events carry the message structure; content lives in the content store.
- Semconv-defined metrics — `gen_ai.client.token.usage`, `gen_ai.client.operation.duration` — give aggregation surfaces every backend can consume.
- Platform extensions live under `platform.*`, never under `gen_ai.*`. Tenant id, quota bucket, policy decision id, content pointer, cost — platform concerns, platform namespace.
- Semconv versions are pinned per SDK shim release; renames get dual-emit windows; removals are breaking changes gated by the conformance tests.
- OpenInference is a compatibility concern at the Phoenix exporter path, not a second vocabulary emitted by the SDK.

The next chapter picks up the second piece of the slice — how the vocabulary rides across plane boundaries so the many spans a single request produces land in one trace.
