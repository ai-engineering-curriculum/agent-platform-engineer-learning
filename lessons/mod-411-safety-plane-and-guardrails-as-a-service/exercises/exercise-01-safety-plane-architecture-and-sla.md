# exercise-01: Safety Plane Architecture and SLA

**Estimated effort:** 3 hours · **Output:** a safety-plane topology diagram, a per-hook SLA document, a stub implementation of the three hook endpoints with a passing integration test against a demo agent, and a short README (~1 page) · **Pairs with:** chapters [01](../01-the-safety-plane-and-what-it-owns.md), [02](../02-guardrail-toolkit-landscape.md), and [07](../07-build-vs-buy-for-guardrail-stacks.md); feeds exercises 02, 03, and project 404.

## Objective

Translate chapter 01's abstract "the plane owns three hooks and emits one record shape" and chapter 02's engine map into a runnable, documented architecture. You will draw the topology, pin the SLA each hook commits to the runtime, implement a stub of the three endpoints that returns a real `SafetyRecord` (chapter 05), and run a demo agent through them.

Chapter 01 is the frame; chapter 02 is the engine map; this exercise is the forcing function that reveals which of the two chapters you have actually internalised vs. memorised. The platform engineer who has never wired a hook into a runtime will under-budget latency on the first production turn and learn the lesson at 3 am.

## Prerequisites

- You have read chapters [01](../01-the-safety-plane-and-what-it-owns.md), [02](../02-guardrail-toolkit-landscape.md), and skimmed [05](../05-standardised-safety-event-schema.md) and [07](../07-build-vs-buy-for-guardrail-stacks.md).
- You have Docker + Docker Compose available locally, or a Kubernetes dev cluster (`kind` / `k3d` / Minikube). The paved-road assumption is Docker Compose for this exercise.
- You have a stub agent available — either the runtime stub from [exercise 02 of mod-402](../../mod-402-agent-runtime-service-and-durable-execution/exercises/) or a 100-LOC FastAPI agent that calls [OpenAI](https://platform.openai.com/docs/api-reference) / [Anthropic](https://docs.anthropic.com/) / a local [Ollama](https://ollama.com/) model and emits the mod-408 semantic attributes on each turn (`gen_ai.*`, `platform.session.lifecycle`).
- You have Python 3.11+ and the ability to pull a small OSS guardrail model. The exercise can substitute a mock classifier for air-gapped environments — you MUST annotate this choice in the README.

## Problem statement

You own the architecture for the safety plane that the fictional tenant `acme-financial`'s `writer-v1` interactive-chat agent will route through. A VP of Engineering has asked you to produce an architecture document and a running proof of concept. The output has four parts:

1. A topology diagram naming every box the plane runs, every seam it crosses, and every stream it emits to.
2. A per-hook SLA table binding the plane to the runtime's TTFT / TTFA budget (mod-410 chapter 02 numbers).
3. A stub implementation of the three hooks (input filter, output validator, tool-invocation hook) with a real `SafetyRecord` emitted per verdict.
4. A demo in which the stub agent calls the hooks, produces at least one of each verdict type (`allow`, `redact`, `require_human_approval`, `refuse`), and the `SafetyRecord`s land on a Kafka topic / local file bus the next exercise can replay from.

## Requirements

### Part 1 — Topology diagram

An `architecture.md` (or PNG / SVG) showing:

- The runtime (mod-402), the MCP gateway (mod-403), the model gateway (mod-406), the observability plane's collector (mod-408), and the policy plane (mod-405) as external boxes.
- The safety plane's three hook endpoints as separate services (even if they share deployment — logical separation is required).
- The engines each hook calls (chapter 02's map). For this exercise it is acceptable to run a single OSS classifier (Llama Guard 3 1B works on CPU; Prompt Guard 86M on CPU; or ShieldGemma 2B on a modest GPU). Document the choice.
- The policy-pack repository (local filesystem, git, or S3 bucket — your call).
- The `SafetyRecord` event stream (local Kafka via Docker Compose, Redpanda, or a flat-file JSONL for a lighter setup).
- The three verdict paths — allow, redact, require_human_approval, refuse — annotated with what each produces downstream (agent continues, agent continues with redacted content, runtime pauses, agent emits templated refusal).

Call out which boxes you are running vs. mocking. A mock "model gateway" is fine; a mock "safety-plane hook" is the point of the exercise and must be real.

### Part 2 — The per-hook SLA table

A table under `sla.md`:

| Hook | Budget p95 | Budget p99 | Fail-open or fail-closed on timeout | Degraded-mode behaviour |
| --- | --- | --- | --- | --- |
| input_filter | ... | ... | ... | ... |
| output_validator | ... | ... | ... | ... |
| tool_invocation_hook | ... | ... | ... | ... |

Rules:

- The SLA budgets must be grounded in mod-410 chapter 02's TTFT / TTFA targets. If your stub agent targets p95 TTFT ≤ 3 s, the input filter cannot eat more than X ms of that budget, and the number must be justified in the document.
- Fail-open vs. fail-closed is a policy decision, not a default. Justify the choice per hook. (Chapter 02 pattern 1 fails open on the classifier stage for latency-sensitive paths; chapter 04 tightens this for T1 tenants.)
- Degraded-mode behaviour includes: what the hook returns on classifier unavailability, what `SafetyRecord.outcome.action_taken` the hook emits, and what downstream signal (SRE plane, ops) is produced.

### Part 3 — Stub implementation of the three hooks

Three endpoints — HTTP or gRPC, your call. Minimum behaviours:

**Input filter** (`POST /v1/input-filter/evaluate`):
- Accept a JSON body with `content`, `trust_tier`, `trace_id`, `span_id`, `tenant_id`, `workload` (sub-object), and optional `subject_source` and `session_context`.
- Call at least one real classifier. A small Prompt Guard or Llama Guard model is acceptable; ShieldGemma 2B is acceptable; even OpenAI Moderation via its SDK is acceptable (document the egress choice).
- Apply at least one known-pattern regex (chapter 04 pattern 2).
- Return a `verdict` matching chapter 05's schema.
- Emit a full `SafetyRecord` to the event stream.

**Output validator** (`POST /v1/output-validator/evaluate`):
- Accept the model's response (text or structured) and the originating `trace_id` / `span_id`.
- Run a shape validator (Pydantic or [Guardrails AI](https://www.guardrailsai.com/)) if the content is structured.
- Run a content classifier (can reuse the input-filter engine).
- Return and emit as above.

**Tool-invocation hook** (`POST /v1/tool-invocation-hook/evaluate`):
- Accept `tool_name`, `tool_args`, `trace_id`, `span_id`, `session_id`, and recent-turn context.
- Apply at least two rules from the plane's rule set. Suggested:
  - `email.send` outside the tenant's allowlist of `to:` addresses returns `require_human_approval`.
  - Any tool call containing a `file://` or RFC1918 URL in its arguments returns `refuse` with `LLM05:improper_output_handling/ssrf_pattern`.
- Return and emit.

Each endpoint must:

- Measure its own latency and populate `hook.latency_ms` on the record.
- Apply a configurable timeout (deadlines propagated from the caller).
- Produce the OWASP category identifier set correctly (chapter 03's mapping file).
- Emit to a configured sink (file, Kafka topic, or Redpanda — your choice; make it configurable).

### Part 4 — Demo: stub agent routed through the hooks

An end-to-end run:

- `scripts/demo.sh` starts the Compose stack (runtime stub, three hook services, Kafka / flat-file sink, classifier model).
- `scripts/demo-run.sh` runs a scripted agent session that exercises each verdict path:
  1. A benign turn (`allow`).
  2. A turn containing known-pattern injection text (should produce `refuse` or `redact`).
  3. A turn containing a PII pattern (should produce `redact`).
  4. A tool call to `email.send` outside the allowlist (should produce `require_human_approval`).
  5. A tool call containing a `file://` URL (should produce `refuse`).
- The demo prints each `SafetyRecord` as JSON and asserts the expected `verdict.decision`.

Attach a `demo.log` capturing one successful run.

### Part 5 — README (~1 page)

- How to start and stop the stack.
- How to run the demo and what to expect.
- The engine choice for each hook and the trade-offs you chose (reference chapter 07's axes).
- Two "I was surprised by" observations — e.g., "the classifier's p95 on CPU was 450 ms on first-cold-start, dropping to 180 ms warm; our SLA table has to account for cold start explicitly."
- One open question for the risk engineer — e.g., "the known-pattern regex catches 'ignore all previous instructions' but misses 'ignore the above'; what is the risk-engineer's cadence for extending the pattern list, and how do we avoid the regex bloating without bounds?"

## Starter guidance

- **Start with the fake classifier.** Return "unsafe" on any input containing the literal string `JAILBREAK` and "safe" otherwise. Wire the hook end-to-end — the Kafka stream, the `SafetyRecord` shape, the demo agent's call path. Only then swap in a real classifier.
- **Measure the fake classifier's latency.** A hook that returns in 2 ms behind a fake classifier will suddenly return in 200 ms with the real one; the SLA numbers you commit to in Part 2 need to be reality-tested before you sign them.
- **The `SafetyRecord` must validate against the JSON Schema in chapter 05.** Write the schema file (`schemas/safety-record-v1.json`) first; validate every emission. This catches shape bugs early and makes exercise 03 nearly free.
- **Keep the OWASP mapping file real.** `policy-pack/owasp-mapping.yaml` with the engine-native-category → OWASP-id entries you actually use. Even if small, the file is the scaffolding exercise 04 extends.
- **Use OpenTelemetry trace IDs.** Even if you are not running a full OTel collector, generate real W3C Trace Context ids in the demo. The `SafetyRecord.trace_id` has to be a real 32-char hex; exercise 03's export expects it.
- **Separate the policy-pack reload path.** Even if the stub "reload" is just "re-read a file", have a documented reload mechanism. Chapter 02 pins this; a stub without reload is a stub that teaches the wrong habit.

## Acceptance criteria

Mark done when:

- [ ] `architecture.md` exists and names every box; no box is a drawing without a corresponding config or service.
- [ ] `sla.md` has the table filled in with budgets justified against mod-410's SLIs, fail-open/closed named per hook, degraded-mode behaviour documented.
- [ ] `services/input-filter`, `services/output-validator`, and `services/tool-invocation-hook` each expose the endpoint, each call a real (not mocked) engine or classifier for at least one of their checks, each emit a `SafetyRecord`.
- [ ] The `SafetyRecord` emissions validate against `schemas/safety-record-v1.json`.
- [ ] `scripts/demo.sh` and `scripts/demo-run.sh` start the stack and exercise all four verdict decisions; `demo.log` captures the run.
- [ ] README includes the two "surprised by" observations and the open question for the risk engineer.
- [ ] Each hook's `hook.latency_ms` is populated with a measured value, not hard-coded.
- [ ] No `SafetyRecord` emits an engine-native category without an OWASP mapping (unmapped entries must surface as `UNKNOWN:<engine>:<category>` per chapter 03).

## Stretch goals

- **Add the dialog flow controller.** Wire [NeMo Guardrails](https://docs.nvidia.com/nemo/guardrails/latest/index.html) as the orchestrator; express at least one Colang flow and invoke a classifier from inside it.
- **Add a second engine in shadow.** Deploy a second classifier (e.g., ShieldGemma alongside Llama Guard) in shadow mode; produce a side-by-side verdict log. Chapter 07's swap discipline, one-week compressed to ten minutes.
- **Harden the SLA — add a circuit breaker.** When the classifier's p95 exceeds the budget for a rolling window, open a circuit breaker and route to a fallback (regex-only). Emit `hook.outcome.action_taken = "allow_with_annotation"` and a separate telemetry event.
- **Integrate with the paved-road SDK scaffold.** Either use the actual mod-412 SDK scaffold or produce a thin wrapper that lets an agent opt in by `from safety_plane import SafetyClient; client = SafetyClient()`; call sites are three lines.
- **Add a tenant overlay.** A per-tenant YAML file that overrides thresholds; `acme-financial` tightens LLM01 refuse threshold to 0.85; the demo shows the different behaviour.
- **Write a conformance test for the architecture.** A script that queries each hook's `/healthz`, verifies the engine version and policy-pack version are what the decision document declares, verifies the emitted `SafetyRecord`s match the schema. Appends to `demo.log`.
