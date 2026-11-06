# exercise-02: Cross-Plane Distributed Trace Propagation

**Estimated effort:** 3 hours · **Output:** a running 3+-service docker-compose topology, one captured trace file (OTLP JSON) with every span sharing a `trace_id`, a deliberate-propagation-break demonstration, a Mermaid diagram of propagation sites, and a short README (~1 page) · **Pairs with:** chapter [03](../03-distributed-trace-propagation-across-planes.md); feeds exercises 03 and 04.

## Objective

Stand up a minimal cross-plane topology — at least a runtime, a model gateway, and an MCP server, with one of the hops crossing a subprocess or container boundary — and wire W3C Trace Context + Baggage end to end so that one external request produces one coherent trace. Then deliberately break propagation at one hop and capture the fragmented result so you have the before / after.

A well-wired topology holds one `trace_id` across every hop and carries `tenant_id` on every span via baggage. A broken one has disconnected trace segments and loses the cross-cutting attribute. Chapter 03's invariant — one request → one trace — is executable here, not theoretical.

## Prerequisites

- You have read chapters [01](../01-the-observability-plane-and-what-it-owns.md), [02](../02-opentelemetry-semantic-conventions-for-generative-ai.md), and [03](../03-distributed-trace-propagation-across-planes.md).
- You have the SDK shim from [exercise 01](exercise-01-otel-genai-attribute-mapping-for-an-agent.md) — or a 50-LOC stand-in that at least sets `gen_ai.operation.name` and `platform.tenant_id`.
- You have Docker (and `docker compose`) on your workstation. The [Docker documentation](https://docs.docker.com/compose/) covers the setup.
- You have an OTel Collector binary / image available ([contrib distribution](https://github.com/open-telemetry/opentelemetry-collector-releases)).
- You have the [W3C Trace Context](https://www.w3.org/TR/trace-context/) spec open in a tab, specifically §3.2 (`traceparent`) and §3.3 (`tracestate`).

## Problem statement

Build a three-service pipeline that mirrors the first three hops of chapter 03's worked example. The services can be the thinnest-possible stubs — the lesson is in propagation, not in business logic.

- **Service A — `runtime-stub`.** Accepts an HTTP POST at `/invoke` with a JSON body. Starts an `agent.turn` span with `gen_ai.operation.name=chat`. Calls Service B over HTTP. Receives B's response. If B's response contains a `tool_use` block (deterministic — have B always return one), calls Service C. Returns.
- **Service B — `model-gateway-stub`.** Accepts an HTTP POST at `/v1/chat`. Starts a `model_gateway.chat` span. Returns a deterministic `tool_use` response with `gen_ai.tool.call.id = "call_a1b2c3"`.
- **Service C — `mcp-server-stub`.** Accepts an HTTP POST at `/tools/search/invoke`. Starts an `mcp.tool.call` span with `gen_ai.operation.name=execute_tool`. In addition, **spawns a subprocess** (one of: a short-lived `python -c "..."` script, a `sh -c` command, or a fresh container) to compute the tool result. The subprocess starts its own span (`sandbox.execute`), which must be parented to the `mcp.tool.call` span.

All three services push OTLP to a single collector; the collector exports to an in-memory sink (an `file` exporter is the simplest option — [see contrib docs](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter/fileexporter)).

## Requirements

### Part 1 — The services and the SDK plumbing

Author each service with a full SDK configuration:

- `TracerProvider` instantiated at startup.
- OTLP exporter pointed at the collector.
- Global propagator set to `CompositePropagator([TraceContextTextMapPropagator(), W3CBaggagePropagator()])` — and nothing else.
- Inbound HTTP middleware that extracts `traceparent` and `baggage`.
- Outbound HTTP client that injects `traceparent` and `baggage`.
- Baggage attribute setters on service A: `tenant_id` and `session_id` go on baggage from the inbound request body.
- Every span emitted by any service carries `platform.tenant_id` as an attribute, sourced from baggage.

Pseudocode is in chapter 03; translate to your language of choice. The working services are the mechanical floor; the lesson is the glue.

### Part 2 — The subprocess hop

Service C spawns a subprocess per invocation. The subprocess must:

- Receive `TRACEPARENT` and `OTEL_BAGGAGE` (or your SDK's equivalent env var) from the parent's environment.
- Boot an OTel SDK (or use an in-process shim that reads the env vars).
- Emit a `sandbox.execute` span parented correctly to `mcp.tool.call`.

The hard part is the env-var passthrough on spawn. `subprocess.Popen(..., env={**os.environ, "TRACEPARENT": current_traceparent()})` is the pattern. The subprocess can be as thin as a Python one-liner that prints a canned result; what matters is the span emission.

### Part 3 — The collector + capture

Minimum collector config (file-exporter sink):

```yaml
receivers:
  otlp: { protocols: { grpc: { endpoint: 0.0.0.0:4317 }, http: { endpoint: 0.0.0.0:4318 } } }
processors:
  batch: { timeout: 1s, send_batch_size: 1024 }
exporters:
  file: { path: /captures/traces.jsonl }
  # A `logging` exporter alongside is handy for live reading during dev.
service:
  pipelines:
    traces: { receivers: [otlp], processors: [batch], exporters: [file] }
```

<!-- needs-research: the exact exporter / processor option fields shift across collector minor versions; verify this YAML against the current contrib distribution before running. -->

### Part 4 — The green run

Invoke Service A once with a tenant-id-bearing payload:

```sh
curl -X POST http://localhost:8001/invoke \
  -H 'Content-Type: application/json' \
  -d '{"tenant_id": "acme-financial", "session_id": "sess_01HV9", "prompt": "hi"}'
```

Capture the collector's `traces.jsonl` output into `green.traces.jsonl`. Walk the file and assert:

- **Exactly one `trace_id` across all spans.** A script that reads `green.traces.jsonl`, extracts every span's `trace_id`, and asserts the set is `{1}`.
- **Every span's `platform.tenant_id` equals `"acme-financial"`.**
- **Every non-root span's `parent_span_id` references another span in the capture.** No orphans.
- **`sandbox.execute` is a child of `mcp.tool.call`.** The hop across the subprocess boundary held.

Automate these assertions as a `check_green.py` (or `.ts`, `.go`) script. Checked in; runs on `make verify` or equivalent.

### Part 5 — The deliberate break

Produce a second run with one of the following deliberate breaks:

- **Drop the subprocess env-var passthrough.** Service C spawns without `TRACEPARENT`. The subprocess starts a fresh root trace.
- **Replace the propagator on Service B.** Use `jaegerpropagator` (or any non-W3C format) — Service A injects W3C and B does not extract it. B's spans start a fresh trace.
- **Swallow `traceparent` on Service A's outbound.** Remove the injection middleware on one outbound call.

Capture the resulting `broken.traces.jsonl`. Walk it and show:

- The set of `trace_id`s is `> 1`.
- The handoff point is identifiable — one plane's span has a `trace_id` matching A's; the next plane's spans have a different `trace_id`.

Author a `check_broken.py` that explicitly asserts fragmentation. It is a demonstration script, not a CI gate — the point is to reason about the detection pattern.

### Part 6 — The Mermaid diagram

Produce a Mermaid sequence diagram in the README showing every propagation site:

```mermaid
sequenceDiagram
    participant Client
    participant A as runtime-stub
    participant B as model-gateway-stub
    participant C as mcp-server-stub
    participant S as sandbox-subprocess
    Client->>A: POST /invoke (no traceparent — root)
    A->>B: POST /v1/chat (traceparent: 00-{tid}-{sid_A}-01, baggage: tenant_id=...)
    B->>A: 200 (response with tool_use)
    A->>C: POST /tools/search/invoke (traceparent: 00-{tid}-{sid_A}-01, baggage: tenant_id=...)
    C->>S: spawn subprocess (env TRACEPARENT=00-{tid}-{sid_C}-01)
    S->>C: exit 0 (tool result; span emitted)
    C->>A: 200 (tool result)
    A->>Client: 200 (final response)
```

Call out each propagation site by name in the README prose: HTTP header on A→B, HTTP header on A→C, env var on C→S. One line of explanation each.

### Part 7 — The README

One page covering:

- Which three services you built and how to run them (`docker compose up`; one `curl`).
- The Mermaid diagram, with the propagation-site call-outs.
- The three green assertions and how to run them (`python check_green.py`).
- The deliberate break you implemented and how `check_broken.py` detects it.
- One "I was surprised by" observation from the exercise — e.g. "the subprocess hop was the one that broke first; the HTTP hops worked out of the box because the instrumentation library did the injection for me."
- One open question for mod-410 SRE — e.g. "if the subprocess span is emitted but the subprocess exits before the OTel SDK flushes, how does the SRE on-call detect the dropped span?"

## Starter guidance

- **Use the SDK's auto-instrumentation libraries where available.** [`opentelemetry-instrumentation-httpx`](https://pypi.org/project/opentelemetry-instrumentation-httpx/), [`opentelemetry-instrumentation-fastapi`](https://pypi.org/project/opentelemetry-instrumentation-fastapi/), the equivalents in Node / Go — they handle the HTTP header injection and extraction for free. Writing the middleware by hand is a learning exercise but not required for this one.
- **Pin your SDK version in the README.** The propagation behaviour is stable; the configuration surface moves. A reviewer wanting to reproduce needs to know the version.
- **Keep the services thin.** Each one is 30–80 LOC. The temptation to add real behaviour (an actual Anthropic call, a real tool execution) is a trap; the point is the trace shape.
- **Flush before exit.** `TracerProvider.shutdown()` or `force_flush()` at process exit. Dropped spans masquerading as propagation failures are the most frustrating hour of this exercise.
- **Read `green.traces.jsonl` by hand before writing assertions.** One manual pass teaches you the OTLP JSON layout; the assertions follow from the structure.
- **Use `docker compose logs -f` liberally.** The collector's `logging` exporter on `debug` verbosity is the fastest way to catch a propagator misconfiguration.

## Acceptance criteria

Mark done when:

- [ ] `docker compose up` brings up three services + a collector + a captures volume, and the single `curl` invocation succeeds.
- [ ] `green.traces.jsonl` contains at least four spans (`agent.turn`, `model_gateway.chat`, `mcp.tool.call`, `sandbox.execute`) sharing one `trace_id`.
- [ ] `check_green.py` passes: one `trace_id`; every span has `platform.tenant_id = "acme-financial"`; no orphan parents; `sandbox.execute` parents to `mcp.tool.call`.
- [ ] `broken.traces.jsonl` exists and `check_broken.py` demonstrates the fragmentation.
- [ ] The Mermaid diagram names every propagation site and the README's prose matches it.
- [ ] The subprocess hop uses explicit env-var passthrough (no inherited env relying on shell).
- [ ] No service has registered any propagator other than `tracecontext` + `baggage`.

## Stretch goals

- **Add a durable-execution hop.** Insert Temporal, Inngest, or a hand-rolled job queue between Service A's entry and Service B's call. The job stores `traceparent` as a workflow input; the step-execution span reads it back and parents correctly. Chapter 03's durable-execution section has the pattern.
- **Add a message-queue hop.** Kafka, SQS, Redis Streams. Producer injects `traceparent` into headers / metadata; consumer extracts and starts a child span linked to the producer via a `SpanLink`. The [OTel messaging conventions](https://opentelemetry.io/docs/specs/semconv/messaging/) name the attributes.
- **Replace the file exporter with a real backend.** Spin up Langfuse, Phoenix, or a Prometheus + Tempo stack in the same compose file. The trace shows up in a UI; the exercise's lesson gains a tangible payoff.
- **Instrument the collector with a tail-sampling policy.** Add a `tail_sampling` processor that keeps the trace only if any span has `error.type` set. Fire a request that fails in the subprocess; watch the keep decision; fire a successful one; watch it drop. Chapter 04 covers the pattern.
- **Add a cross-tenant leak test.** Fire two concurrent requests with different `tenant_id`s on baggage. Assert via `check_green.py` that each trace's spans carry the correct tenant. The classic concurrency bug the chapter warned about.
- **Wire the OpenInference exporter as a parallel pipeline.** For the Phoenix adopter, route the same trace stream through a translator processor that renames `gen_ai.*` to `openinference.*`. Chapter 04's translator example is the starting point.
