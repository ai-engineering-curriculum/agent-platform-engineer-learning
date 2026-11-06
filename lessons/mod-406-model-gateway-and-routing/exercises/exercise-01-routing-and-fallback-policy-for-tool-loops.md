# exercise-01: Routing and Fallback Policy for Tool Loops

**Estimated effort:** 3 hours · **Output:** one routing-table file, one retry-policy file, one capability-matrix file, a reference adapter plus invariant unit tests, and a short README (~1 page) · **Pairs with:** chapters [02](../02-multi-provider-routing-and-the-provider-abstraction.md) and [03](../03-fallback-retry-and-the-tool-calling-loop.md); feeds exercises 02, 03, 04, and project 401 / 404.

## Objective

Design the routing-table, capability-matrix, and retry-policy artefacts the gateway uses for a single platform-coded model identifier under a tool-calling loop — and write the invariant tests that prove retry does not re-sample tool_use blocks, does not shred tool schemas on fallback, and does not re-stream after the first chunk. The deliverable is the shape every tool-loop request through your plane runs under.

A well-authored policy produces chapter-03's request envelope attempts list predictably on well-formed inputs, refuses incompatible fallbacks on capability grounds, and preserves tool-loop invariants under every error normalised in the chapter-03 taxonomy. If your policy allows a retry to re-sample a different `tool_use` or silently falls back from Anthropic structured-output-via-tools to OpenAI json_mode and calls it "the same capability", a reviewer will send it back.

## Prerequisites

- You have read chapters [01](../01-the-model-gateway-slice-and-its-obligations.md), [02](../02-multi-provider-routing-and-the-provider-abstraction.md), and [03](../03-fallback-retry-and-the-tool-calling-loop.md).
- You have an OpenAI-style HTTP client library in your preferred language (Python with `httpx` is a common choice; Go with `net/http`; TypeScript with `undici`). The adapter is small enough that you can author it in a few hundred lines.
- You have accounts and API keys for at least two of OpenAI, Anthropic, Google Gemini — or a local vLLM / SGLang / Ollama endpoint — so you can exercise real provider errors (429, 529, timeout). If you can only use one, the invariant tests still exercise correctly against a fault-injecting mock, but you lose the "did my policy actually survive a real `overloaded_error`" check.
- You have read the [OpenTelemetry Semantic Conventions for Generative AI](https://opentelemetry.io/docs/specs/semconv/gen-ai/) and know which attributes a span should carry.

## Problem statement

Pick one platform-coded model identifier to author for; the shape of your policy follows:

- **`platform:writer-v1`** — a general-purpose writer for long-context document drafting. Primary: Anthropic `claude-sonnet-4-6` (`us-east-1`); fallback: Anthropic `claude-sonnet-4-6` (`eu-west-1`), then AWS Bedrock-hosted Anthropic, then OpenAI `gpt-4o-2024-08-06`. Capability requirements: 200k context, structured output, tool-use, streaming. Agents use this to draft long-form text with retrieval tool calls.
- **`platform:critic-v1`** — a short-output grader used in evaluation pipelines. Primary: self-hosted vLLM serving `llama-3.3-70b-instruct`; fallback: OpenAI `gpt-4o-mini`. Capability requirements: 32k context, JSON mode, no tool-use, non-streaming. Agents use this to score a document on a rubric; output is a strict JSON blob.
- **`platform:planner-v1`** — a strong-reasoning planner used at the start of complex multi-tool workflows. Primary: OpenAI `o4`-class or Anthropic `claude-opus-4-7`; fallback: Google `gemini-1.5-pro-002`. Capability requirements: 200k context, tool-use, streaming, parallel tool calls. Agents use this in a loop with up to 20 tool calls per session.

(Pricing details, exact model IDs, and current capabilities drift — flag them with `<!-- needs-research: ... -->` if you cannot verify against the current provider documentation when you author.)

## Requirements

### Part 1 — The routing table

Author `config/routing-table.yaml` (or JSON, if you prefer) containing:

- **The chosen model identifier** with its capability declaration (chapter 02).
- **A primary and at least two ordered fallbacks** across at least two distinct providers (not just two regions of one provider; a region-only fallback hides provider-lock-in risk).
- **Explicit `region`, `key_ref`, and `capability_deviations` on every mapping.** Declared deviations drive the capability check in part 4.
- **References by name** to the retry-policy file, capability-matrix file, and (stub-only, no need to author) cache-policy, tenancy allow-list, and price-table.

The file must be hand-authorable by a reviewer and must not contain secret values — only references.

### Part 2 — The capability matrix

Author `config/capability-matrix.yaml` covering each provider-plus-model pair used in your routing table. For each pair, record:

- Max input and max output tokens.
- Tool-use support (yes / no / schema-as-tool-hack).
- Streaming support.
- Structured-output support (native strict / native best-effort / unsupported).
- Prompt-caching support (automatic / explicit-cache_control / unsupported) with a `cache_write_cost_multiplier` and `cache_read_cost_multiplier` row as chapter 05 defines them.
- Parallel-tool-calls support.

Values should match published provider documentation at the time of authoring. Anything you cannot verify gets a `# needs-research` comment.

### Part 3 — The retry policy

Author `config/retry-policies/tool-loop-strict.yaml` with the full shape chapter 03 defines:

- Per-normalised-error-code policy for at least: `transient.network`, `transient.overloaded`, `transient.rate_limited`, `transient.timeout`, `terminal.client_error`, `terminal.auth`, `terminal.content_filter`, `terminal.context_length`, `terminal.output_malformed`.
- For each code: `max_attempts`, `backoff` (deterministic — `honour_retry_after_or`, `exponential_jitter(initial, cap)`, or `0ms`), and `fallback_eligible` (`true`, `false`, or `conditional`).
- An `overall` block with `max_attempts_total` and `max_wall_clock_ms`.
- The `stop_on_partial_stream: true` invariant.
- A `reinforce_instruction: true` on `terminal.output_malformed`.

Values you choose must be defensible — a `max_attempts: 5` on `transient.overloaded` without a `max_wall_clock_ms` budget guard is a thundering-herd generator, and a reviewer will reject it.

### Part 4 — The reference adapter and router

Write a minimal reference implementation (one source file is enough; ~400–700 lines) exercising:

- A `GatewayRequest` dataclass / struct matching the fields from chapter 01.
- A `Router` that:
  1. Loads the routing table, capability matrix, retry policy.
  2. Resolves candidates (capability check → stub tenancy check → stub cache-affinity reorder → stub health reorder).
  3. Walks candidates under the retry policy.
  4. Returns a `GatewayResponse` with a populated `routing.attempts` list.
- Two provider adapters (any two of OpenAI, Anthropic, Google, or a self-hosted OpenAI-compatible endpoint) that translate the internal request shape into provider-native and back, including tool-definitions, tool-use blocks, and finish reasons.
- A fault-injection layer under the adapter that can simulate `transient.overloaded`, `transient.network`, `terminal.content_filter`, and `terminal.output_malformed` on demand.

### Part 5 — The invariant tests

Write a test file `tests/invariants_test.*` that exercises **each** of the tool-loop invariants from chapter 03 against the fault-injection layer. At minimum:

- **Invariant 1 — request-is-unit-of-retry.** A retry caused by `transient.overloaded` on the primary must produce *one* response to the caller, with two entries in `routing.attempts`, and the same `gateway_request_id` across attempts. The provider adapter must have been called twice; the caller-facing response must be present once.
- **Invariant 2 — tool_use is not re-sampled across attempts on partial stream.** When streaming is enabled and the first chunk already carried a `tool_use` content block, a subsequent transient failure must *not* produce a retry that re-samples. The test asserts that either (a) the request fails with a `stream_broken_after_partial` outcome, or (b) the retry is to a different candidate but the attempts list carries the `stop_on_partial_stream` flag.
- **Invariant 3 — tool-schema does not change under fallback.** Primary has a tool with signature `{query: string}`; fallback has a widened schema `{query: string | array<string>}`. The router must refuse the fallback mapping (capability mismatch) or the adapter must emit the narrow schema even to the widened fallback. Either is defensible; the test asserts the chosen behaviour.
- **Invariant 4 — streaming commits once.** Under a successful retry on a non-streaming request, the final response carries one set of token counts equal to the final successful attempt's usage, not the sum across attempts.

Each test asserts both the outcome (success / terminal / refused) and at least one field on the attempts list. A test that only asserts the outcome is a stub.

### Part 6 — The README

One page covering:

- Which model identifier you chose and why.
- The fallback order and the single most interesting edge case (e.g. "OpenAI fallback drops cache support — cache_hit_rate SLO trades off on fallback").
- The reason-code normalisation table your adapter uses (which provider error → which normalised code).
- The three invariants you exercised and how the tests shaped them.
- One open question you would route to `ai-risk-engineer`, `agentic-safety-engineer`, or `ai-infra-inference-engineer`. Example: "the Anthropic `terminal.content_filter` outcome in our test passes through as a terminal response; should we route around it in any case the harm model allows?"

## Starter guidance

- **Keep the adapter small.** Two providers is enough. Four is interesting; six is yak-shaving. The lesson is in the translation points chapter 02 names (system prompt placement, tool-call encoding, finish-reason normalisation), not in coverage.
- **Pin the normalised-error vocabulary before writing code.** If the vocabulary changes partway through, every test and every policy row changes with it. A short enum at the top of the file keeps you honest.
- **Honour `Retry-After` from day one.** Providers return it; your `honour_retry_after_or` helper should parse both the delta-seconds and HTTP-date forms (`Retry-After: 5` and `Retry-After: Sat, 06 Nov 2026 18:30:00 GMT` are both valid).
- **The streaming adapter is where quietly-wrong behaviour lives.** Record the moment of first-chunk-emission explicitly and consult it in the retry decision. Hiding it behind a generator makes the code elegant and the bugs permanent.
- **Shadow routing is tempting to add and out of scope here.** Primary + fallback only; shadow and canary are chapter 02 concerns that exercise 02 does not need.
- **Flag `<!-- needs-research: ... -->` on every capability value you cannot verify.** Provider rate-limit shapes and model-level maximums have moved multiple times in the last 18 months; a reviewer expects the flag.
- **Treat quota / cache / redaction as external calls to interfaces that stub out for this exercise.** Exercises 02, 03, and (next module) 04 flesh them in.

## Acceptance criteria

Mark done when:

- [ ] The routing-table YAML validates (lints) and references the capability matrix and retry policy by name.
- [ ] Every mapping declares its capability deviations against the identifier's capability declaration; the router refuses a mapping whose deviations intersect the request's requirements.
- [ ] The retry policy covers all eight normalised codes with defensible `max_attempts`, backoff, and `fallback_eligible` values; the policy has overall `max_attempts_total` and `max_wall_clock_ms` guards.
- [ ] The adapter reports a normalised outcome on every attempt; the attempts list on the response is complete and correct.
- [ ] All four invariant tests pass; each asserts the outcome and at least one attempts-list field.
- [ ] A fault-injected run that exceeds `max_wall_clock_ms` surfaces as a terminal response with `reason_code: budget.wall_clock_exhausted`.
- [ ] A single provider 401 in the middle of a fallback walk does *not* retry on the same key (invariant from chapter 03).
- [ ] The README names the model identifier, the fallback decision, the normalisation table, the invariants, and one open question.

## Stretch goals

- **Per-region tenancy allow-list.** Add a `tenancy.yaml` that marks `eu-west-1` as denied for a `tenant_id: residency-eu-only`; verify the router refuses the fallback on that tenancy.
- **Circuit-breaker state machine.** Implement the open / half-open / closed state per `(provider, model, region)` triple on top of your attempts log; a third consecutive `transient.overloaded` opens the circuit for a cool-down window and the router skips the candidate.
- **OpenTelemetry integration.** Emit a span per attempt conforming to the gen-ai semantic conventions; include `gen_ai.request.model`, `gen_ai.response.model`, `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, `gen_ai.request.gateway.model_identifier`, and a provider-native-finish-reason field.
- **Shadow probe.** Even when a circuit is open, send 1% of requests to the open candidate as a shadow (return the primary / fallback result to the caller). Use the shadow success rate to drive the half-open transition.
- **Hedging (dangerous).** On the latency-sensitive identifier (`platform:critic-v1`), implement bounded-pair hedging within a single provider — issue a request to the primary, and after `hedge_delay_ms` fire the same request against a second key of the same provider. First response wins; cancel the other. Prove the invariants still hold.
