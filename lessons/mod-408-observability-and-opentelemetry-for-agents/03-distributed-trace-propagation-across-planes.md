# Distributed Trace Propagation Across Planes

Chapter 02 pinned the vocabulary — the names every span carries. This chapter pins the thing that makes the vocabulary matter: that every span produced on behalf of one external agent request shares a `trace_id`, is parented correctly, and reaches the same backend as a connected whole. The runtime starts a turn. The turn calls the model gateway. The gateway returns a `tool_use`. The runtime routes the tool call through the MCP gateway. The MCP gateway runs code in the sandbox. The sandbox result comes back; the runtime calls the model again with the result; the model answers. Somewhere in the middle the policy plane decides whether the tool call was allowed; somewhere else the eval plane might replay this trace tomorrow against a candidate model. Five planes, a dozen hops, one request. If any one hop drops the trace context, the trace fragments and chapter 01's first obligation — "one agent request produces one coherent trace" — fails.

The mechanism is [W3C Trace Context](https://www.w3.org/TR/trace-context/), carried by [OpenTelemetry's context propagation API](https://opentelemetry.io/docs/specs/otel/context/api-propagators/), augmented with [OTel Baggage](https://www.w3.org/TR/baggage/) where cross-cutting attributes (like `tenant_id`) need to ride along. This chapter walks every hop between planes, writes down what has to be preserved, and names the failure modes a conformance test (chapter 05) must catch.

## W3C Trace Context in one page

The header-level wire format is two headers: `traceparent` and `tracestate`.

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             ^^ ^                               ^                ^^
             |  |                               |                |
             version-trace_id------------------parent_span_id---trace-flags
```

- `trace_id`: 16 bytes, hex-encoded. Shared across every span on the request.
- `parent_span_id`: 8 bytes, hex-encoded. The span the next hop is a child of.
- `trace-flags`: 1 byte, hex-encoded. Bit 0 is `sampled`.

`tracestate` is a vendor-extensible field for vendor-specific routing context. The platform uses it rarely (chapter 04 covers the one case: backend fan-out routing hints).

Baggage is a separate header (`baggage: key1=value1,key2=value2`) that carries application-level attributes across hops. The platform uses it for `tenant_id`, `session_id`, and `conversation.id` — the cross-cutting identifiers every span should carry as attributes but that are inconvenient to look up per-hop.

The [W3C Trace Context spec](https://www.w3.org/TR/trace-context/) is short and worth reading end to end. Most of this chapter's concrete work is making sure that when a plane receives a request, it reads `traceparent`; when it emits a request, it writes `traceparent`; and that the hop in between — queue, subprocess, durable-execution step — preserves the context.

## The propagator contract

OpenTelemetry defines a `TextMapPropagator` abstraction ([spec](https://opentelemetry.io/docs/specs/otel/context/api-propagators/)) with two operations:

- `inject(context, carrier, setter)` — serialise the context into some carrier (an HTTP headers map, a queue message's metadata, an env-var map).
- `extract(carrier, getter) -> context` — deserialise a context from the carrier.

The platform's SDK shim ships a composite propagator bundling:

- `tracecontext` — the W3C Trace Context propagator. Required; never swap it.
- `baggage` — the W3C Baggage propagator. Required; the platform's tenant / session id ride on it.

That's it. No `b3`, no `jaeger`, no provider-specific propagators. Keeping the propagator list short is how a cross-plane trace stays correlateable; a plane using B3 while every other plane uses W3C means some reverse-translator in the middle, and one day it will be wrong.

```python
# platform_otel_sdk/propagator.py
from opentelemetry import propagate
from opentelemetry.propagators.b3 import B3MultiFormat  # NOT installed
from opentelemetry.trace.propagation.tracecontext import TraceContextTextMapPropagator
from opentelemetry.baggage.propagation import W3CBaggagePropagator
from opentelemetry.propagators.composite import CompositePropagator

propagate.set_global_textmap(
    CompositePropagator([
        TraceContextTextMapPropagator(),
        W3CBaggagePropagator(),
    ])
)
```

The conformance test (chapter 05) checks `propagate.get_global_textmap()` and refuses to run if a plane has registered anything else. Standardisation is cheap when it is executable.

## The hops: where `traceparent` must survive

Every plane boundary is a propagation site. Walk them in order.

### Runtime → model gateway (HTTP / gRPC)

The simplest case. The runtime uses the platform's HTTP client, which the SDK shim wraps with [`opentelemetry-instrumentation-httpx`](https://pypi.org/project/opentelemetry-instrumentation-httpx/) or equivalent. The wrapper calls `inject()` on outgoing headers. The model gateway reads the same shim, which calls `extract()` on incoming headers inside the ASGI / WSGI middleware.

- **Required:** the runtime's HTTP / gRPC client is the shim's. If someone uses `requests` directly, they bypass injection.
- **Required:** the model gateway's server is the shim's. If someone writes a raw `http.server` handler, they bypass extraction.
- **Enforcement:** the conformance suite boots a stub model-gateway that asserts `traceparent` is present on every inbound request. A plane that fails this fails CI.

The MCP gateway → runtime hop (the runtime making a tool-call-driven request to MCP) looks identical.

### MCP gateway → MCP server (HTTP + SSE or stdio)

MCP (the [Model Context Protocol](https://modelcontextprotocol.io/specification)) has two transport modes: HTTP + Server-Sent Events, and stdio (subprocess). Both are propagation sites.

- **HTTP transport.** Same story as above. The MCP gateway's outbound HTTP client injects `traceparent` on every request; MCP servers extract on every request. The MCP spec does not mandate `traceparent`, so the server-side implementation is the platform's responsibility — the paved-road MCP server scaffold (mod-403 chapter 03) includes the extractor. A third-party MCP server that drops `traceparent` is an onboarding gap; chapter 05's conformance test runs against the server before it is allowed on the registry.
- **stdio transport.** The subprocess approach is trickier. `traceparent` can ride in an env var on spawn (`TRACEPARENT`, matching the convention in the [OpenTelemetry environment variable specification](https://opentelemetry.io/docs/specs/otel/configuration/sdk-environment-variables/)) or in the initial JSON-RPC handshake message. The platform's rule: the MCP gateway always injects `TRACEPARENT` into the subprocess environment, and the MCP server scaffold always extracts it on startup. The scaffold is the mechanism that makes this reliable; writing an MCP server without the scaffold is grounds for CI to refuse it.

### MCP server → sandbox (subprocess / container / microVM)

The sandbox plane (mod-404) runs tool code — a Python script, a shell command, a subprocess. Trace context has to survive the exec boundary.

- **Subprocess mode.** The parent process sets `TRACEPARENT` (and the baggage env `OTEL_BAGGAGE`) in the child's environment. The child process's OTel SDK auto-detects and uses them.
- **Container / microVM mode.** Container runtimes (Docker, Firecracker, gVisor) drop host env vars by default. The sandbox launcher explicitly passes `TRACEPARENT` and `OTEL_BAGGAGE` through, either as `--env` arguments or as part of the launch spec.
- **Pyodide / Deno-style in-process sandbox.** The tracer in the sandbox shares the parent process's SDK instance; the launcher sets the current context before invoking the sandboxed function. No env-var dance; just the normal OTel context API.

The sandbox is where propagation most often breaks on first implementation. Chapter 05's cross-plane conformance test is written around this hop specifically.

### Runtime → durable-execution step (temporal / inngest / in-house)

The runtime (mod-402) is built on a durable-execution substrate — [Temporal](https://docs.temporal.io/), [Inngest](https://www.inngest.com/docs), or an in-house equivalent — so agent turns survive process restarts. Durable execution is a propagation site because the workflow task that resumes a turn hours later must carry the trace context of the originating agent request. The pattern:

- **Workflow start.** The runtime captures `traceparent` from the inbound request and persists it as a workflow-input field.
- **Step execution.** Each step begins by extracting the stored `traceparent` and setting it as the current context before running the step body.
- **Step completion.** The step's span closes; the workflow proceeds.

Temporal's [OpenTelemetry integration](https://docs.temporal.io/develop/python/observability) and Inngest's docs cover the canonical patterns. The platform's rule is that the durable-execution client is wrapped by the SDK shim so a workflow author gets this for free.

A subtle point: a long-running workflow may span hours or days. The trace that started it may be gone from the backend by the time the last step runs (retention is bounded). The workflow still correctly carries the `trace_id`; what the reviewer loses is the ability to query the historical span. The content store (chapter 01, chapter 04) is the mitigation — the workflow's structural record lives there regardless of the trace backend's retention.

### Runtime → message queue (Kafka, SQS, Redis Streams)

Agent fleets lean on queues for fan-out, scheduling, and background work. Every queue is a propagation site.

- **Enqueue.** The producer serialises `traceparent` into the message's metadata — a header on Kafka, a message attribute on SQS, a field on a Redis Streams entry. The platform's queue client wrapper does this.
- **Dequeue.** The consumer extracts `traceparent` from the metadata and starts its span as a child.
- **Link semantics.** When one producer emits a message consumed by many — fan-out — the consumer span is linked to the producer span, not parented. Semconv's `SpanLink` captures this; the consumer span's `trace_id` matches the producer's, and the dashboard query stays coherent.

[OpenTelemetry messaging conventions](https://opentelemetry.io/docs/specs/semconv/messaging/) specify the attribute names; the platform's queue client wrapper applies them.

### Evaluation plane: replay vs original

When the eval plane (mod-407) replays a production trace against a candidate agent, the question is "what `trace_id` does the replay use?" Three options; the platform picks the third.

- **Reuse the original `trace_id`.** Tempting; it lets a reviewer pull the replay against the original in one query. Bad idea: replay spans and original spans become indistinguishable, dashboards double-count, retention policies collide.
- **Fresh `trace_id`, no reference.** Clean, but loses the "what was replayed against what" relationship the eval plane needs.
- **Fresh `trace_id` with an explicit link.** The replay span's root carries a `platform.replay_source_trace_id` attribute, and the OTel `SpanLink` on the root points to the original root span. The dashboards see the connection; the retention and routing stay separate. This is the shape.

The attribute convention aligns with mod-407 chapter 04's replay modes. A reviewer filtering for "all replays of trace X" is a `WHERE platform.replay_source_trace_id = 'X'` query over the trace store.

## Baggage: when attributes ride, not spans

Baggage is the mechanism for "this attribute should be on every span in the request, and every plane needs to read it, but the plane that *sets* it is somewhere upstream". The platform uses baggage for exactly three things:

- `tenant_id` — set by the entry point that identifies the tenant (typically the API gateway); every downstream plane reads it off baggage and emits it as `platform.tenant_id` on its spans.
- `session_id` — set by the runtime when the session starts; propagates for the session's lifetime.
- `conversation.id` — the semconv-named identifier; typically the same as `session_id`, but names it in the semconv's own vocabulary.

Baggage is **not** a general-purpose key-value transport. Do not put user-identifiable content on it; it rides through every hop as cleartext header and lands in every observability backend. Baggage carries small, non-sensitive, cross-cutting identifiers. If the attribute is sensitive, do not use baggage; look it up from a context store on each plane.

```python
# At the entry point:
from opentelemetry.baggage import set_baggage

ctx = set_baggage("tenant_id", request.tenant_id)
ctx = set_baggage("session_id", request.session_id, context=ctx)
context.attach(ctx)
```

Every subsequent span the SDK shim creates reads the baggage and attaches `platform.tenant_id` and `session.id` as span attributes. Chapter 05's conformance test asserts both are present on every span.

## Sampling: deciding at the root, not in the middle

Sampling is a propagation concern because an incorrectly sampled trace is a fragmented trace. The default pattern in OTel is head-based sampling with the sampling decision set on the root span and encoded in the `sampled` flag of `traceparent`; downstream planes respect that decision. The platform follows this with one addition.

- **Head sampling at the entry.** The runtime's entry point decides, per request, whether to sample. The decision is in the `sampled` flag; downstream planes do not re-decide.
- **"Always sample serious-incident candidates".** The runtime's sampler consults a predicate — "is this tenant under elevated monitoring", "has this agent tripped a guardrail in the last hour", "is this a replayed trace" — and samples 100% when true. Chapter 04 covers the collector-side tail-sampling that may trim recorded-but-boring traces after the fact.
- **Collector-side tail sampling for cost control.** A processor at the gateway collector may drop traces that recorded no errors, no tool calls, and no guardrail events. This is a decision made by the collector, not the emitting SDK; it is fully controlled by platform ops. Chapter 04 details.

A conformance test asserts: given a sampled inbound `traceparent`, the plane propagates it as sampled. Given an unsampled inbound, the plane does not up-sample. Planes that respect the inbound decision are the only shape compatible with cross-plane coherence.

## Failure modes to test for

The conformance test (chapter 05) targets these specifically.

- **Missing `traceparent` on inbound.** The plane either generates a fresh `trace_id` (acceptable for an external entry; the plane is the trace root) or surfaces an error if the plane is internal and should never be called without context.
- **Mangled `traceparent`.** The plane rejects malformed headers (per [W3C Trace Context section 3.2.2.6](https://www.w3.org/TR/trace-context/#traceparent-header-field-values)) rather than silently ignoring and starting a fresh trace.
- **Parent mismatch.** A hop's span lists a `parent_span_id` that doesn't appear anywhere else in the trace. Usually means a broken wrap on one hop.
- **Sampled-flag flip.** A hop's root span is sampled, but a child span emits unsampled. Means the plane is making its own sampling decision; forbidden.
- **Baggage leakage across tenant boundaries.** Baggage set for tenant A rides into a span emitted on behalf of tenant B. Shouldn't happen; the collector tier (chapter 04) asserts; the conformance test replays a cross-tenant scenario and checks.
- **Context not inherited across subprocess exec.** The sandbox classic. Fix is the launcher's env-var passthrough; check is a synthetic trace that runs a sandboxed tool and asserts the sandbox span's `trace_id` matches the parent.

## A worked example: one external request, six spans, one trace

Pull it together. An external user request arrives at the API gateway. The runtime receives it, calls the model gateway, receives a `tool_use`, invokes the MCP gateway which runs the tool in a sandbox, receives the tool result, calls the model gateway again, receives a final answer. Six spans; one trace:

```
trace_id = 4bf92f3577b34da6a3ce929d0e0e4736

span: agent.turn                                  (runtime)         parent: (root)
  span: model_gateway.chat                        (model-gateway)   parent: agent.turn
    event: gen_ai.tool.message                                      (on model_gateway.chat)
  span: mcp.tool.call                             (mcp-gateway)     parent: agent.turn
    span: sandbox.execute                         (sandbox)         parent: mcp.tool.call
  span: model_gateway.chat                        (model-gateway)   parent: agent.turn
```

Six spans; all parented back to `agent.turn`; `trace_id` identical; `platform.tenant_id` on every one (via baggage); `platform.policy_decision_id` on `mcp.tool.call` referencing the policy-plane record; `platform.sandbox_pod_id` on `sandbox.execute`. A dashboard query for `trace_id = 4bf9...` pulls all six. A regression investigation filters by `gen_ai.response.model = "claude-sonnet-4-6"` and finds every provider call. An Article 72 auditor (chapter 06) asks for all tool invocations for one tenant across one calendar day — a filter on `gen_ai.operation.name = "execute_tool"` and `platform.tenant_id = "acme-financial"`.

If any one hop drops `traceparent`, the above view fragments into two or three disconnected sub-traces. Chapter 05's conformance test is the thing that keeps that from happening.

## Summary

- Trace context propagates on [W3C Trace Context](https://www.w3.org/TR/trace-context/) headers. The platform's SDK shim registers a composite propagator (`tracecontext` + `baggage`) and nothing else.
- Every plane boundary is a propagation site. HTTP / gRPC via middleware; message queues via metadata; durable-execution steps via persisted input; subprocess sandboxes via env var passthrough.
- MCP transports have two propagation shapes — HTTP (headers) and stdio (env var on spawn); both are owned by the paved-road MCP scaffold (mod-403).
- Sandbox propagation is the hop most often missed on first implementation. The sandbox launcher is responsible for env-var passthrough.
- Baggage carries `tenant_id`, `session_id`, `conversation.id` across hops. Not for sensitive data; not a general kv transport.
- Sampling is decided at the root and respected downstream. Tail-sampling and drop decisions happen at the collector (chapter 04), not in the middle of the request path.
- The evaluation plane's replay uses a fresh `trace_id` with `platform.replay_source_trace_id` linking back to the original.

The next chapter picks up the collector tier — how the same semantic stream fans out to Langfuse, Phoenix, Datadog, and Prometheus + Grafana without the emitting planes re-instrumenting per backend.
