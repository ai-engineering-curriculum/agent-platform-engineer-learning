# exercise-01: OTel GenAI Attribute Mapping for an Agent

**Estimated effort:** 2 hours · **Output:** one platform SDK shim (one language, ≤300 LOC), a `platform.*` extension registry YAML, a before / after span pair for one operation (JSON), and a short README (~1 page) · **Pairs with:** chapter [02](../02-opentelemetry-semantic-conventions-for-generative-ai.md); feeds exercises 02, 03, 04.

## Objective

Take one existing plane's current instrumentation — whichever one you already have code for from mod-402 / 403 / 404 / 405 / 406 / 407, or a minimal reproducer you write — and map it to the OpenTelemetry Semantic Conventions for Generative AI. Produce a platform SDK shim that does the mapping in one place, a `platform.*` extension registry that names the platform-local attributes the shim emits in addition to semconv, and a concrete before / after pair of spans for one representative operation.

A well-authored shim moves attribute-renaming out of every service's codebase and into one library. Chapter 02's invariants (semconv vocabulary, `platform.*` extensions never shadow `gen_ai.*`, content never on the span) are the shim's contract, not individual services'.

## Prerequisites

- You have read chapters [01](../01-the-observability-plane-and-what-it-owns.md) and [02](../02-opentelemetry-semantic-conventions-for-generative-ai.md).
- You have an OpenTelemetry SDK installed in your language of choice (`opentelemetry-api` + `opentelemetry-sdk` for Python, `@opentelemetry/api` + `@opentelemetry/sdk-trace-node` for Node, equivalent for Go). The [OTel docs](https://opentelemetry.io/docs/languages/) carry the current install instructions per language.
- You have the current [GenAI semantic conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/) open in a browser tab. If the current revision looks different from what chapter 02 cites, pin to the current one and flag `<!-- needs-research: ... -->` anywhere the chapter's names no longer line up.
- You have one plane's codebase (or a 50-LOC reproducer) that already emits spans. If you have none, stub a mock "model gateway" that calls a deterministic fake provider and emits one span per call.

## Problem statement

Pick one of three source operations to map. The shape of your shim follows the operation:

- **`model_gateway.chat`** — a chat / text_completion operation. Maps to `gen_ai.operation.name=chat` with `gen_ai.request.model`, `gen_ai.response.model`, `gen_ai.usage.*`, `gen_ai.response.finish_reasons`, `gen_ai.system`, plus the `platform.*` tenant / quota / price-table extensions.
- **`mcp.tool.call`** — a tool-execution operation. Maps to `gen_ai.operation.name=execute_tool` with `gen_ai.tool.name`, `gen_ai.tool.call.id`, `gen_ai.tool.type`, plus `platform.tool_registry_version` and `platform.policy_decision_id`.
- **`eval.replay.turn`** — an evaluation-plane replay turn (mod-407). Maps to `gen_ai.operation.name=chat` on an `agent.turn` span with `gen_ai.agent.id`, `gen_ai.conversation.id`, plus `platform.replay_source_trace_id` and the eval fixture / run identifiers as `platform.*` extensions.

Your current (pre-shim) instrumentation probably emits a mix of local names (`model_name`, `tool_identifier`, `tenant_token_cost`, etc.). The exercise is the migration.

## Requirements

### Part 1 — The shim's public API

Author `platform_otel_sdk/<language>/gen_ai.py` (or `.ts` / `.go`) exposing, at minimum:

- `start_chat_span(request, tenant, quota)` — enters a context manager / scope that creates the span with the semconv name (`model_gateway.chat` or similar), sets all required `gen_ai.*` attributes it can determine at start time, and attaches the `platform.*` extensions. Returns an object the caller can decorate with the response once it arrives.
- `start_tool_execution_span(tool_call, policy_decision)` — equivalent for the tool-execution case.
- `assert_chat_span(span)` — a conformance helper (for use in tests; exercise 03 will build the general version). On a semconv-conformant span it returns silently; on a missing attribute it raises a `ConformanceError` naming the missing attribute and the chapter it is required by.

The shim never accepts a free-form `**attributes` dict from the caller. Callers pass typed objects (`ChatRequest`, `ChatResponse`, `ToolCall`, `PolicyDecision`) and the shim decides which semconv attribute each field maps to. This is the point of the exercise: concentrate attribute naming in one library.

### Part 2 — The `platform.*` extension registry

Author `platform_extensions.yaml`:

```yaml
version: "1.0"
attributes:
  - key: platform.tenant_id
    type: string
    description: "The tenant on whose behalf the operation runs."
    applies_to: [any_span]
    provenance: "set via W3C Baggage by the API gateway at request entry"

  - key: platform.quota_bucket
    type: string
    description: "The quota bucket drawn down by this operation (mod-406 chapter 04)."
    applies_to: [model_gateway.chat, model_gateway.text_completion, model_gateway.embeddings]
    provenance: "set by the model gateway when the quota is evaluated"

  # ... remainder for your chosen operation
```

The registry is the authoritative list of what the shim can emit beyond semconv. Chapter 02's rule — platform extensions never shadow semconv — is enforceable from the registry: a key that starts with `gen_ai.` is a schema error.

### Part 3 — The before / after span pair

Produce `before.span.json` and `after.span.json`:

- `before.span.json` — the span your current / stub instrumentation emits. Includes whatever local names it uses today.
- `after.span.json` — the same operation emitted through the shim. Uses semconv names for the semconv-defined attributes and `platform.*` names for the extensions.

Keep both in the same file layout (OTLP JSON encoding is the sensible baseline — see the [OpenTelemetry proto definitions](https://github.com/open-telemetry/opentelemetry-proto/tree/main/opentelemetry/proto/trace/v1)). A reviewer should be able to diff the two files and understand exactly which names changed.

### Part 4 — The call-site change

Refactor one real (or stub) call site to use the shim. Expected delta: the call site shrinks by ~5–15 lines; the attribute-naming work moves into the shim. Include the before and after call-site code in the README. The lesson is visible in the diff.

### Part 5 — The README

One page covering:

- Which operation you chose and why.
- The shim's public API, in one code block.
- The before / after span diff — three or four of the most load-bearing name changes called out explicitly.
- Which `platform.*` extensions you added, and which semconv attribute each one sits alongside rather than shadowing.
- One attribute that was ambiguous between semconv candidates — a case where more than one semconv name could have fit, and the reasoning for your choice. (If you did not encounter one, you probably over-simplified the operation.)
- One open question you would route to the semconv working group — e.g. "the finish-reasons enum does not have a value for `provider_safety_refusal`; is the right path a new enum value or a `platform.*` extension?"

## Starter guidance

- **Pin a semconv version on day one.** Pick the current revision of `open-telemetry/semantic-conventions`, note the commit SHA or release tag in the README, and keep that tab open while you code.
- **Resist the temptation to emit both names.** The dual-emit window from chapter 02 is for migrations in production; for the exercise, the shim emits semconv only. The point is to see the names change.
- **Keep the shim small.** The SDK shim is a thin facade over the OTel SDK; it is not a replacement for the SDK. If the shim is more than ~300 lines, you are reinventing OTel.
- **Mock the provider; do not call a live model.** The lesson is attribute mapping, not provider behaviour. A function that returns `ChatResponse(model="claude-sonnet-4-6", usage=Usage(input_tokens=10, output_tokens=5))` is fine.
- **Mark `<!-- needs-research: ... -->` on any attribute name you cannot find in the current semconv revision.** Chapter 02 called out that the spec is still formally experimental; names shift. The flag convention matches mod-406's.
- **Do not put prompt text on the span.** Chapter 02's invariant holds even in the exercise. A `platform.content_ref` stub pointing at a fake `s3://...` URI is the right shape.

## Acceptance criteria

Mark done when:

- [ ] The shim exposes `start_chat_span` / `start_tool_execution_span` (or the equivalent for your chosen operation) that accepts typed objects and emits a semconv-conformant span.
- [ ] The shim's `assert_chat_span` (or equivalent) raises `ConformanceError` with a named missing attribute when any required attribute is missing.
- [ ] `platform_extensions.yaml` names every platform extension the shim emits and rejects any key starting with `gen_ai.`.
- [ ] `before.span.json` and `after.span.json` both exist, are valid OTLP JSON for a single span, and differ precisely in attribute names (not in business-logic values).
- [ ] At least one real call site has been migrated to the shim; the diff is in the README.
- [ ] No attribute key in `after.span.json` carries content — prompts, completions, tool arguments, or reference strings.
- [ ] The README names the chosen operation, the shim's API, the key name changes, the extensions added, one ambiguous attribute, and one open question for the semconv working group.

## Stretch goals

- **Emit the dual-write migration window.** Update the shim to emit both the legacy name and the semconv name for one release-cycle, and show the README how the collector will consume the dual-emit and route only the semconv version downstream. Chapter 04's `transform` processor is the pattern.
- **Generate the shim's attribute setters from the YAML.** The platform extension registry and the semconv YAML together are enough to generate the per-attribute code. A small code generator (jinja or `yq`) produces the setters; the hand-written shim calls into them. Weaver ([the OTel project's generator](https://github.com/open-telemetry/weaver)) is the upstream reference.
- **Add the agent-level attributes.** `gen_ai.agent.id`, `gen_ai.agent.name`, `gen_ai.conversation.id` — the recent additions in semconv's agent model. Emit them as resource attributes (not span attributes) on the SDK's `Resource` and show the diff on a span emitted downstream.
- **Author one language shim and port to one more.** The point: the attribute catalogue is language-neutral; the code is language-specific. A two-language port in the same exercise makes the catalogue's independence obvious and primes the mod-412 paved-road SDK conversation.
- **Build the conformance helper's negative tests.** For the operation you chose, author ten unit tests per shim method — each test violates one required attribute and asserts the helper raises with the right attribute name. The suite is a down payment on exercise 03's rule catalogue.
