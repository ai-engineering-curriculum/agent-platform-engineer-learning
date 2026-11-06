# Telemetry Conformance Testing

Chapters 02–04 defined a contract. This chapter writes down the executable mechanism for keeping that contract. The contract has three parts — semconv vocabulary, trace propagation, content / metadata separation — and three ways to break it: a plane renames an attribute, a plane drops `traceparent` on a hop, or a plane pushes content onto the shared pipeline. All three are detectable before production. All three should fail CI.

A platform without conformance tests is a platform where the dashboards drift silently and the first signal is a reviewer asking "why is `gen_ai.request.model` null on half the writer-v1 traces this week". A platform with them is one where that reviewer's question is the one that never comes up.

## What a conformance test is (and is not)

A conformance test checks that a plane's **emitted telemetry** matches the platform's contract — not that the plane's business logic is correct, not that its latency is acceptable, not that its costs are reasonable. Those are other kinds of tests. A conformance test answers one question: "when this plane handles a request, does the telemetry it emits satisfy the contract chapters 02–04 pinned?"

The scope:

- **In scope.** Attribute names, attribute presence, attribute types, span kinds, span parenting, event names, event attribute presence, metric names, baggage keys, `traceparent` propagation.
- **Out of scope.** Attribute *values* being business-logic-correct (the model the request routed to is a correctness question, owned by mod-406's tests, not this one). Dashboard content (owned by the backend). Business metric aggregation (owned by the eval plane).

The output of a conformance run is a pass / fail and a per-rule report. On fail, the report names the rule, the offending span / event / metric, and a pointer to the semconv doc for the rule.

## Three test levels

The tests come in three concentric scopes.

### Level 1 — SDK-boundary unit tests

Run against a single service in isolation. The service is stubbed at every external dependency; the test exercises one or more code paths; the test asserts on the OTLP export stream.

A sketch, using Python and an in-memory span exporter:

```python
import pytest
from opentelemetry.sdk.trace.export.in_memory_span_exporter import InMemorySpanExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import SimpleSpanProcessor

@pytest.fixture
def captured_spans():
    exporter = InMemorySpanExporter()
    provider = TracerProvider()
    provider.add_span_processor(SimpleSpanProcessor(exporter))
    # Replace the SDK's provider for the duration of the test.
    yield exporter

def test_model_gateway_chat_emits_required_attributes(captured_spans, stub_provider):
    stub_provider.set_response(model="claude-sonnet-4-6", input_tokens=10, output_tokens=5)
    client = ModelGatewayClient(provider=stub_provider)

    client.chat(platform_model="platform:writer-v1", prompt="hi")

    [span] = captured_spans.get_finished_spans()
    assert span.name == "model_gateway.chat"
    assert span.attributes["gen_ai.operation.name"] == "chat"
    assert span.attributes["gen_ai.system"] == "anthropic"
    assert span.attributes["gen_ai.request.model"] == "platform:writer-v1"
    assert span.attributes["gen_ai.response.model"] == "claude-sonnet-4-6"
    assert span.attributes["gen_ai.usage.input_tokens"] == 10
    assert span.attributes["gen_ai.usage.output_tokens"] == 5
    assert "gen_ai.prompt" not in span.attributes       # content never on the span
    assert "gen_ai.completion" not in span.attributes
    assert "platform.tenant_id" in span.attributes
```

A plane's codebase has one of these per operation it emits spans for. They are fast (no network), they fail loudly (a semconv rename flips a specific `KeyError`), and they run on every PR.

The platform's paved-road SDK ships a `conformance.assert_chat_span(span)` helper that encapsulates the check so a plane author writes one line, not twenty:

```python
def test_model_gateway_chat_emits_conforming_span(captured_spans, stub_provider):
    client = ModelGatewayClient(provider=stub_provider)
    client.chat(platform_model="platform:writer-v1", prompt="hi")

    [span] = captured_spans.get_finished_spans()
    conformance.assert_chat_span(
        span,
        plane="model-gateway",
        require_platform_extensions={"platform.tenant_id", "platform.quota_bucket"},
    )
```

The helper is where the semconv version is pinned. Bumping semconv means bumping the helper's version, which propagates by library upgrade; a plane that pins an old helper version fails the collector's semconv-version gate (chapter 04) in integration.

### Level 2 — plane-boundary integration tests

Run against a plane in a docker-compose / k3d / testcontainers harness. Real network. A stub upstream, a stub downstream, a real collector pointed at an in-process OTLP receiver. The test issues a request, reads the collector's output, asserts on the full span set.

The additional things level 2 catches that level 1 does not:

- **Collector-side redaction.** A plane that *does* emit `gen_ai.prompt` is caught at level 1 by the "not in attributes" assertion; a plane that emits it some code paths but not others may slip through level 1's limited coverage. Level 2 confirms the collector redacts it regardless.
- **Baggage propagation across the boundary.** The test sets baggage on the inbound request; asserts the baggage attributes (`platform.tenant_id`) appear on the outbound span from the plane.
- **`traceparent` propagation across the boundary.** The test crafts a specific `trace_id` and `parent_span_id` on the inbound request; asserts that the plane's spans share the `trace_id` and parent to the inbound `parent_span_id`.

A sketch (illustrative, using `testcontainers` with the collector running in a sidecar container):

```python
def test_mcp_gateway_propagates_traceparent(mcp_container, collector_output):
    inbound_trace = "4bf92f3577b34da6a3ce929d0e0e4736"
    inbound_span  = "00f067aa0ba902b7"
    headers = {
        "traceparent": f"00-{inbound_trace}-{inbound_span}-01",
        "baggage": "tenant_id=acme-financial,session_id=sess_01HV9",
    }
    mcp_container.post("/tools/search/invoke", json={"q": "x"}, headers=headers)

    spans = collector_output.read_spans()
    assert all(s.context.trace_id == bytes.fromhex(inbound_trace) for s in spans)
    tool_span = next(s for s in spans if s.name == "mcp.tool.call")
    assert tool_span.parent.span_id == bytes.fromhex(inbound_span)
    assert tool_span.attributes["platform.tenant_id"] == "acme-financial"
```

Level 2 tests live in the plane's own repo (so a PR that breaks propagation fails the plane's CI) *and* in a shared platform-wide test suite (so a change to the SDK shim's propagator can run every plane's test).

### Level 3 — cross-plane end-to-end tests

Run against a staging environment with all planes deployed. A test harness fires a synthetic agent request at the runtime's entry point. The runtime does a turn, which calls the model gateway, which emits `tool_use`, which drives the MCP gateway, which runs a sandboxed tool, which returns, which the runtime feeds to the model gateway again. The test watches the trace backend for the resulting trace and asserts on its shape.

Checks only level 3 catches:

- **One trace end to end.** All spans share one `trace_id`. If any plane drops `traceparent`, level 3 fails.
- **Baggage survives every hop.** `tenant_id` appears on every span, including the sandbox span.
- **No orphan spans.** Every span's `parent_span_id` references another span in the trace. A span pointing at a span that doesn't exist is a propagation bug.
- **Content stays off the shared pipeline.** The test asserts that the backend-visible attributes do not contain prompt text; the test asserts that a content-store file exists with the trace id.

The level 3 harness is a scheduled synthetic (nightly, or more frequent) that reuses the same code as the on-PR level 2 tests. A level 3 failure pages the observability plane's on-call.

## The rule catalogue

A conformance test needs rules. The platform's catalogue — one rule per row — is the executable form of chapters 02–04. Rules are maintained as a YAML file in the semconv package so a new convention lands in one place.

A sketch of the rule-file schema:

```yaml
version: "1.33"      # the semconv version the catalogue targets
rules:
  - id: gen_ai.chat_span.operation_name
    description: "A chat span MUST carry gen_ai.operation.name=chat"
    applies_to:
      span_name: ["model_gateway.chat", "agent.turn"]
    assert:
      attribute: "gen_ai.operation.name"
      equals: "chat"

  - id: gen_ai.chat_span.request_model
    description: "A chat span MUST carry gen_ai.request.model"
    applies_to:
      attribute: "gen_ai.operation.name"
      equals: "chat"
    assert:
      attribute: "gen_ai.request.model"
      present: true

  - id: platform.tenant_id.on_every_span
    description: "Every span MUST carry platform.tenant_id as an attribute or resource attribute"
    applies_to:
      any_span: true
    assert:
      attribute: "platform.tenant_id"
      present: true

  - id: content.not_on_shared_pipeline
    description: "Spans emitted on the shared pipeline MUST NOT carry prompt or completion content"
    applies_to:
      any_span: true
    assert:
      attributes_absent: ["gen_ai.prompt", "gen_ai.completion"]

  - id: tool_call.correlation_id
    description: "A tool-execution span MUST carry gen_ai.tool.call.id matching the originating tool_use event"
    applies_to:
      attribute: "gen_ai.operation.name"
      equals: "execute_tool"
    assert:
      attribute: "gen_ai.tool.call.id"
      present: true

  - id: traceparent.inbound_respected
    description: "A plane that receives a traceparent MUST emit spans with the inbound trace_id"
    applies_to:
      harness: "level-2-integration"
    assert:
      trace_id: "matches_inbound"
```

The rule engine reads this file, matches spans against the `applies_to` block, and runs the assertions. Rule IDs are stable; renaming a rule is a breaking change to downstream reports. New rules land with a "soft-pass" grace period (one release) before they gate CI; the grace period is itself in the rule file as `severity: warn`.

## The rule engine shape

The engine is a small library. One language implementation per paved-road language (Python, TypeScript, Go — mod-412 picks the set). The engine's contract:

```python
from platform_telemetry_conformance import RuleEngine, load_rules

engine = RuleEngine(rules=load_rules("rules.yaml", semconv_version="1.33"))

result = engine.evaluate(spans=captured_spans.get_finished_spans())

if not result.passed:
    for violation in result.violations:
        print(f"[{violation.rule_id}] {violation.message}")
        print(f"  span: {violation.span.name} (id={violation.span.context.span_id.hex()})")
        print(f"  attributes: {dict(violation.span.attributes)}")
    raise SystemExit(1)
```

Features worth building into the engine from the start:

- **Severity.** `error` fails CI; `warn` surfaces in reports but does not block; `info` is always informational. The grace-period mechanism for new rules.
- **Allowlist.** A plane can declare a short-lived allowlist for a specific rule with an expiry date; the plane's PR carries the allowlist; the rule runs green until the expiry, then fails. The allowlist is public (committed in the plane's repo) and reviewed on merge. No silent exemptions.
- **Rule provenance.** Every rule links to the semconv doc, the chapter, or the design decision that motivates it. `git blame` plus the link is the authority a reviewer uses to answer "why does this rule exist".
- **Rule version compatibility.** Rules declare the semconv versions they apply to (`applies_to: { semconv_version_range: ">=1.30" }`); a plane pinned to an older semconv runs only the rules that apply.

Keep the engine small. The lesson is the catalogue; the engine is the glue.

## CI integration

A plane runs conformance in three places.

- **On every PR to the plane's repo.** Level 1 and a subset of level 2. Required to pass; no bypass path.
- **On a nightly schedule.** The full catalogue including rules in `warn` severity, plus level 3 end-to-end. Nightly surfaces rules that will go `error` soon.
- **Pre-release gate.** The platform's monorepo (or the cross-plane release process) runs a full cross-plane conformance sweep before promoting a release train. A release that violates a rule does not promote.

The CI integration carries two reports: the pass / fail decision (for the gate) and the per-rule report (for a human reviewer). Both go to the PR as comments.

The report is read by a human; the gate is read by the merge queue. If the gate runs green but a human later discovers a rule the catalogue missed, the catalogue is amended — a new rule, a soft-pass release, then a hard-fail release.

## The end-to-end synthetic

Level 3's end-to-end test needs a durable harness. The shape:

- **A synthetic agent definition.** A deterministic agent — one prompt, one tool, one model call. Lives in the platform's test repo.
- **A scheduler.** Runs the synthetic every N minutes against staging and, less frequently, against production (with a dedicated test tenant).
- **A trace-backend reader.** Queries the backend (via the Langfuse API, the Phoenix API, the Datadog API, or by reading the content store directly) for the resulting trace.
- **Rule evaluation.** Runs the catalogue against the fetched trace.
- **Alerting.** A failure pages the observability plane's on-call (mod-410 owns the paging; this module owns the signal).

The synthetic is also the mechanism that catches collector failures the per-plane tests don't: if the collector drops spans on back-pressure, the synthetic's assertion that the trace contains six spans fails. Level 3 is the only test that sees the complete pipeline.

## Working with existing frameworks

Two existing OSS projects do pieces of this; the platform should know what they do and where they fit.

- **[Promptfoo](https://www.promptfoo.dev/docs/) assertion plugins.** Promptfoo supports custom assertions on agent output. The platform can author a Promptfoo plugin that invokes the conformance engine on a captured trace — useful for teams writing evals that want conformance as part of the eval row. Promptfoo is not a replacement for CI-level conformance; it is a convenience for eval authors.
- **[OpenTelemetry's semconv weaver](https://github.com/open-telemetry/weaver).** The OTel project ships a tool (Weaver) that generates per-language types, docs, and (increasingly) runtime checks from the semconv YAML. The platform's rule catalogue is consistent with Weaver's generated artefacts where they overlap; where Weaver extends into runtime checking, the platform's engine adopts rather than duplicates. <!-- needs-research: Weaver is actively evolving; its runtime-check capabilities and coverage of the GenAI conventions change release-to-release. Re-check before committing to custom rule infrastructure. -->

Neither replaces a bespoke rule catalogue. The platform's catalogue is where platform-specific extensions (`platform.*` attributes, cross-plane invariants, content-off-the-pipeline rules) live.

## Common failure modes the tests should catch

The specific failures the test suite is designed to catch — the "if these ever make it to production, the test suite failed" list.

- **Renamed attribute.** A plane emits `model_name` where `gen_ai.request.model` was required. Caught by a presence rule in level 1.
- **Null required attribute.** A plane sets `gen_ai.request.model = None` when it cannot determine the model. Required attributes must be present with a non-null value; set the attribute to a sentinel (`unknown`) and open an issue rather than leaving it null.
- **Tool-call correlation break.** A `tool_use` event emits `gen_ai.tool.call.id` = `"call_a1"`; the subsequent tool-execution span emits `"a1"` without the prefix. Caught by a correlation rule in level 2.
- **Trace fragmentation across the sandbox.** The sandbox launcher forgets to pass `TRACEPARENT` through. Caught by level 3's "all spans share one `trace_id`" rule.
- **Content on the shared pipeline.** A plane adds `gen_ai.prompt` to a span attribute for "debugging". Caught by level 1's absence rule and by level 2's collector check.
- **Baggage leakage across tenants.** A plane re-uses a context across requests and ends up with tenant A's baggage on a tenant B span. Caught by a per-span tenant assertion in level 3 that correlates against the request's intended tenant.
- **Resource-attribute omission.** A plane forgets to set `service.name` on startup, and every span is labelled `unknown_service`. Caught by a resource-attribute presence rule.

The list grows. Every production incident where "the dashboards lied" produces one new rule; the catalogue is a living artefact.

## Summary

- Conformance tests are the executable form of the contracts in chapters 02–04. They check names, presence, types, propagation, and content separation — not business-logic correctness.
- Three levels: SDK unit tests (fast, in-process), plane integration tests (collector in the loop), and cross-plane end-to-end synthetics (staging, scheduled).
- The rule catalogue is a YAML file in the semconv package; the rule engine is a small library per paved-road language. Rules carry severity (`error`, `warn`, `info`), semconv-version applicability, and provenance.
- CI integration: PR-blocking on every plane repo, nightly expansion with warn-severity rules, pre-release cross-plane sweep.
- Level 3 synthetics are the only tests that exercise the full pipeline; collector failure modes that the per-plane tests miss surface here.
- Weaver and Promptfoo plugins complement but do not replace the platform's own catalogue; platform extensions live in the platform's catalogue.

The next chapter picks up the last obligation — turning the trace pipeline into a documented post-market-monitoring evidence slice under the EU AI Act's Article 72, so the same telemetry that catches a regression also satisfies the regulator.
