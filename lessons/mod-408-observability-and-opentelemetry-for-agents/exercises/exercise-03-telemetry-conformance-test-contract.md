# exercise-03: Telemetry Conformance Test Contract

**Estimated effort:** 2 hours · **Output:** a `rules.yaml` rule catalogue, a rule engine (one language, ≤400 LOC), a per-plane fixture set, three deliberate-violation tests the engine detects, a passing baseline test, and a short README (~1 page) · **Pairs with:** chapter [05](../05-telemetry-conformance-testing.md); feeds exercise 04 and project 403.

## Objective

Translate chapter 05's "executable contract" into a working rule catalogue and a rule engine that runs the catalogue against captured OTLP spans. The catalogue codifies the invariants from chapters 02 and 03; the engine is the mechanism that fails CI when a plane drifts. Three deliberate violations exercise the engine's happy path (it detects them); one passing baseline exercises the control case (it does not raise false positives).

A well-authored rule catalogue is the semantic contract as code. Chapter 05's three-level test shape (SDK unit → plane integration → cross-plane synthetic) is the frame; this exercise builds the level-1 and level-2 pieces and names what level-3 would check.

## Prerequisites

- You have read chapters [02](../02-opentelemetry-semantic-conventions-for-generative-ai.md), [03](../03-distributed-trace-propagation-across-planes.md), and [05](../05-telemetry-conformance-testing.md).
- You have the trace captures from [exercise 02](exercise-02-cross-plane-distributed-trace-propagation.md) — specifically the `green.traces.jsonl` and `broken.traces.jsonl` files. If you did not do exercise 02, you can synthesize equivalent fixtures by hand (a reasonable shortcut for this exercise, but you lose the cross-plane realism).
- You have the semconv version from exercise 01's shim. The rule catalogue targets that version.

## Problem statement

The catalogue encodes at least 12 rules spanning three categories. The engine evaluates spans against the catalogue. The tests demonstrate at least three ways to break the contract and one way to pass it cleanly.

### Rule categories the catalogue must cover

- **Semconv presence / value.** The GenAI chat span carries `gen_ai.operation.name=chat`, `gen_ai.request.model`, `gen_ai.response.model`, `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, `gen_ai.system`. The tool span carries `gen_ai.operation.name=execute_tool`, `gen_ai.tool.name`, `gen_ai.tool.call.id`.
- **Platform extensions.** Every span carries `platform.tenant_id` as an attribute or resource attribute.
- **Cross-span consistency.** For every `tool_use` event on a chat span with a `gen_ai.tool.call.id`, a tool-execution span exists in the same trace carrying the same id. Every non-root span's `parent_span_id` references a span in the same trace. All spans in a capture share one `trace_id` (when the capture is one request).
- **Content separation.** No span carries `gen_ai.prompt`, `gen_ai.completion`, or a span-event attribute `gen_ai.message.content`.

### Rules to break deliberately in the tests

Pick three of the following (or author your own; match the three-category spread):

- A chat span missing `gen_ai.request.model`.
- A tool-execution span whose `gen_ai.tool.call.id` does not match any upstream `tool_use` event in the trace.
- A capture with spans across two `trace_id`s (your `broken.traces.jsonl` from exercise 02 is the natural fixture).
- A span emitting `gen_ai.prompt` as an attribute.
- A span missing `platform.tenant_id`.
- A span emitting a `model_name` attribute instead of (or in addition to) `gen_ai.request.model`.

## Requirements

### Part 1 — The rule catalogue

Author `rules.yaml` in the schema sketched in chapter 05. Minimum coverage:

- **4 semconv-presence rules** (one per operation × one per required attribute class).
- **2 cross-span rules** (tool-call correlation, parent-reference integrity).
- **3 platform-extension rules** (`platform.tenant_id` on every span, no shadowing of `gen_ai.*` under `platform.*`, resource-attribute presence of `service.name`).
- **3 content-off-the-pipeline rules** (absence of `gen_ai.prompt`, `gen_ai.completion`, and content-bearing event attributes).

Every rule carries: a stable id, a human description, an `applies_to` block, an `assert` block, a severity (`error` / `warn` / `info`), and a provenance link (to the chapter / section / semconv page that motivates it).

```yaml
version: "1.0"
semconv_version: "<the version your shim pins>"
rules:
  - id: gen_ai.chat.required.request_model
    description: "A chat span MUST carry gen_ai.request.model"
    severity: error
    provenance: "chapter 02, GenAI semconv — spans — required attributes"
    applies_to:
      attribute: "gen_ai.operation.name"
      equals: "chat"
    assert:
      attribute: "gen_ai.request.model"
      present: true

  # ...
```

### Part 2 — The rule engine

Author `conformance_engine/<language>/engine.{py,ts,go}` (one language, ≤400 LOC). The engine:

- Loads the catalogue from `rules.yaml`.
- Accepts OTLP JSON (or protobuf) as input — a trace fixture, a captured span list.
- Evaluates every rule against every applicable span (`applies_to` is the filter).
- Returns a result: `passed: bool`, `violations: List[Violation]` with `rule_id`, `message`, `span_id`, `trace_id`, `attributes`.
- Exits non-zero when any `error`-severity rule fails; zero otherwise.

The engine supports the following assertion kinds (minimum):

- `present: true` — the named attribute exists and is non-null on the span.
- `equals: <value>` — the attribute equals the literal value.
- `attributes_absent: [list]` — none of the listed attributes are present.
- `trace_id: matches_inbound` — all spans share one trace id (applied at the trace level rather than the span level).
- `correlation: <attribute>` — the attribute's value matches a span-event attribute of the same name on another span in the trace.

Severity `error` fails the run; `warn` surfaces in output but does not fail; `info` is logged. The grace-period mechanism chapter 05 describes.

### Part 3 — The fixtures

Author at least five fixtures under `fixtures/`:

- `fixtures/baseline/green.traces.jsonl` — a well-formed capture (exercise 02's `green.traces.jsonl` is the natural seed).
- `fixtures/violations/missing_request_model.traces.jsonl` — a chat span without `gen_ai.request.model`.
- `fixtures/violations/orphan_tool_call.traces.jsonl` — a tool-execution span whose `gen_ai.tool.call.id` does not match any `tool_use` event.
- `fixtures/violations/trace_fragmentation.traces.jsonl` — spans across two `trace_id`s (exercise 02's `broken.traces.jsonl`).
- `fixtures/violations/content_on_span.traces.jsonl` — a span with `gen_ai.prompt` set.

Fixtures are checked in. A reviewer re-running `conformance-engine fixtures/<path>/` reproduces the exact pass / fail outcome.

### Part 4 — The tests

Author `tests/test_conformance.py` (or equivalent) with:

- **One passing test** — the baseline fixture; asserts `result.passed is True` and `result.violations == []`.
- **Three failing tests** — each targets one of the violation fixtures; asserts `result.passed is False`, the specific rule id appears in `result.violations`, and the violation's `message` references the attribute at fault.

Tests run in `pytest` (or equivalent); they are the engine's own level-1 harness.

### Part 5 — The README

One page covering:

- The rule catalogue's coverage — count by category, the 12+ rules you authored, the three violation fixtures.
- The engine's architecture — load, filter, evaluate, report. One paragraph.
- A `pytest -v` run of the tests, with the pass / fail output copied in.
- Three things the engine can *not* catch at level 1 / level 2 — things that require a level-3 synthetic. Chapter 05 names a few; your list may overlap.
- One ambiguity in the semconv spec you ran into — e.g. "`gen_ai.response.finish_reasons` is an array; semconv does not specify whether the array can be empty on a successful response, so the rule treats present-with-any-value as pass."
- One grace-period rule you added at `severity: warn` to demonstrate the staged-rollout mechanism.

## Starter guidance

- **The catalogue is the lesson; the engine is the glue.** Spend time on the rules; keep the engine small. If the engine is getting to 500+ LOC, revisit.
- **Use OTLP JSON, not OTLP protobuf, for the fixtures.** The spec shape is identical, but JSON is diff-friendly and `jq`-friendly. The [OTLP spec](https://opentelemetry.io/docs/specs/otlp/) carries the mapping.
- **Keep the fixture captures small.** Three to six spans per file is plenty. Full real captures are noisy and obscure the violation.
- **Write the fixtures by copying green and mutating one attribute.** Hand-editing JSON is the fastest path; don't run a full service to produce a one-span violation.
- **The correlation assertion is the hardest.** It requires reading events on other spans, not just attributes on the current span. Prototype it on paper before writing the code.
- **Flag the semconv spec version in `rules.yaml`.** The catalogue is tied to a revision; the engine refuses to run against a fixture without that version on the resource attributes. Chapter 04 called out this gate.
- **Mark `<!-- needs-research: ... -->` on any rule whose motivating semconv attribute you cannot locate in the current spec revision.** The flag convention matches mod-406's.

## Acceptance criteria

Mark done when:

- [ ] `rules.yaml` contains at least 12 rules spanning semconv presence, platform extensions, cross-span consistency, and content separation.
- [ ] Every rule has `id`, `description`, `severity`, `provenance`, `applies_to`, `assert`.
- [ ] The engine loads the catalogue and evaluates against OTLP JSON input.
- [ ] `pytest` shows one passing baseline test and three failing-fixture tests, each failing on the expected rule id.
- [ ] A violation's message names the specific attribute at fault; a reviewer reading the output can act on it without reading the engine's source.
- [ ] The engine exits non-zero on any `error`-severity failure and zero on `warn`-only reports.
- [ ] The README names the coverage, the three level-3-only checks, one semconv ambiguity, and one grace-period rule.

## Stretch goals

- **Wire a rule that follows OTel Weaver's generated output.** [Weaver](https://github.com/open-telemetry/weaver) can emit per-language types from semconv. Generate a type for the chat span; use it in the engine; show that the generated type catches a renamed attribute at compile time, not runtime.
- **Author a level-3-style end-to-end harness.** Spin up the exercise 02 topology under compose; fire one request; fetch the trace; run the catalogue. Package the end-to-end as a `make conformance-e2e`.
- **Build the allowlist mechanism.** A rule-id → expiry date table. A fixture with an active allowlist for one rule passes; the same fixture after the expiry fails. Chapter 05's allowlist pattern made executable.
- **Publish per-plane per-rule pass rates.** Run the engine across a corpus of captured traces (yours or synthesised). Produce a markdown table — plane × rule → pass rate — of the kind a platform leadership review (mod-413) consumes. Call out the three lowest rates as "where to invest next quarter".
- **Add a Promptfoo custom-assertion plugin.** A short JavaScript / Python plugin that runs the conformance engine on a captured trace and surfaces violations as Promptfoo assertions. Pairs with the mod-407 Promptfoo harness; the lesson is "the same catalogue, two consumers".
- **Translate one rule into a collector OTTL statement.** Chapter 04's `transform/redact_content` is one example. Author a rule — e.g. "if a span has `gen_ai.prompt`, drop the attribute and emit a counter" — as an OTTL statement; show it running in a sidecar collector. The lesson: conformance-at-CI and defence-in-depth-at-the-collector are the same catalogue in two places.
