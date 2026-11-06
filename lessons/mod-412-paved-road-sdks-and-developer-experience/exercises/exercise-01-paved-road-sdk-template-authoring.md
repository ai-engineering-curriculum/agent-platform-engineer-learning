# exercise-01: Paved-Road SDK and Template Authoring

**Estimated effort:** 3 hours · **Output:** a `paved_road_agent` Python package with the `PavedRoadAgent` interface and at least one framework adapter, a Copier template that scaffolds a runnable sample agent, a compile-time `AgentRevision` manifest emitter, and a short README (~1 page) · **Pairs with:** chapters [01](../01-the-paved-road-sdk-and-what-it-owns.md), [02](../02-reference-shapes-across-five-frameworks.md), and [03](../03-scaffolding-cli-and-template-catalogue.md); feeds exercises 02 and 03 and projects 401 / 402 / 403.

## Objective

Translate chapter 01's abstract "one interface, five adapters, one manifest" and chapter 02's `PavedRoadAgent` shape into a runnable, documented SDK. You will author the platform-owned interface (no framework identifiers in its surface), implement at least one framework adapter against it, ship a Copier template that scaffolds a sample agent, and emit a valid `AgentRevision` manifest.

Chapter 01 is the frame; chapter 02 is the shape; this exercise is the forcing function that reveals which of the two chapters you have actually internalised vs. memorised. A platform engineer who has never tried to hide `anthropic.*` / `openai.*` types behind one interface will under-budget the adapter boundary's cost and relearn the lesson the first time an upstream framework ships a renamed hook.

## Prerequisites

- You have read chapters [01](../01-the-paved-road-sdk-and-what-it-owns.md), [02](../02-reference-shapes-across-five-frameworks.md), [03](../03-scaffolding-cli-and-template-catalogue.md), and skimmed the [`OpenTelemetry Semantic Conventions for Generative AI`](https://opentelemetry.io/docs/specs/semconv/gen-ai/) page.
- You have Python 3.11+, [Poetry](https://python-poetry.org/) (or [`uv`](https://docs.astral.sh/uv/)), and [Copier](https://copier.readthedocs.io/) installed locally.
- You have at least one upstream framework installed: [`anthropic`](https://pypi.org/project/anthropic/) with Agent SDK access, the [OpenAI Agents SDK](https://openai.github.io/openai-agents-python/), the [Google ADK](https://google.github.io/adk-docs/), the [Microsoft Agent Framework](https://learn.microsoft.com/en-us/agent-framework/overview), or [LangGraph](https://langchain-ai.github.io/langgraph/) with [Temporal](https://docs.temporal.io/). Pick one. If you cannot install the real package (air-gapped environment, licensing), use a stub package you implement to mirror the public API; you MUST annotate this choice in the README.
- You have a mod-402 `SessionState` stub available — the one from [exercise 01 of mod-402](../../mod-402-agent-runtime-service-and-durable-execution/exercises/exercise-01-session-state-abstraction-contract.md) is ideal; otherwise implement an in-memory stand-in with the four fields the adapter needs (`session_id`, `tenant`, `agent_id`, `trace_id`).

## Problem statement

You own the paved-road SDK for the fictional platform `acme-platform`. A product engineer on `acme-financial` is asking "I want to write an interactive-chat agent and get on every plane with one call". The deliverable is the SDK they will import plus the template they will scaffold from.

The output has five parts:

1. The `paved_road_agent` package exposing `PavedRoadAgent` and the `@agent(...)` decorator.
2. At least one framework adapter (`AnthropicAgentAdapter`, `OpenAIAgentAdapter`, `GoogleADKAdapter`, `MicrosoftAgentAdapter`, or `LangGraphTemporalAdapter`). The adapter composes the eleven planes in the chapter-02 order.
3. The `AgentRevision` manifest emitter and its JSON Schema.
4. A Copier template that scaffolds a sample `writer-v1` agent from the paved-road SDK.
5. A short README (~1 page) that describes the design choices, two "I was surprised by" observations, and one open question for the SDK's next review.

## Requirements

### Part 1 — The `paved_road_agent` package

A Python package with the following modules:

- `paved_road_agent/__init__.py` — exports `PavedRoadAgent`, `agent`, and the one adapter you implement.
- `paved_road_agent/interface.py` — the chapter-02 `PavedRoadAgent` Protocol. NO framework identifiers in the type signatures.
- `paved_road_agent/types.py` — `Message`, `TurnEvent`, `ToolInvocation`, `AgentInput`, `AgentOutput`. All platform types.
- `paved_road_agent/decorator.py` — the `@agent(...)` decorator that reads the metadata and writes the compile-time manifest.
- `paved_road_agent/adapters/<framework>.py` — the adapter you implement (below).
- `paved_road_agent/instrumentation/__init__.py` — a minimal version of the chapter-05 library: `gen_ai_span(...)`, `_platform_attributes(session)`, `record_tool_call(...)`, `emit_reliability_record(...)`. Minimal is fine — enough that the adapter does not hand-write span attributes.
- `paved_road_agent/manifest.py` — the manifest emitter.
- `tests/test_conformance.py` — a pytest test that asserts an instance of the adapter satisfies `isinstance(instance, PavedRoadAgent)` and that `instance.manifest` is a valid `AgentRevision`.

The `@agent(...)` decorator at minimum accepts: `workload_class`, `tier`, `model_profile`, `system_prompt` (path), `tools` (list of strings). Record the values on the class so the manifest emitter can read them.

### Part 2 — The framework adapter

Pick one framework and implement its adapter. The adapter must:

1. Implement `PavedRoadAgent.stream(...)` and `PavedRoadAgent.run(...)`.
2. Compose the eleven planes in chapter-02 order. For this exercise you can stub mod-403 / mod-404 / mod-405 / mod-406 / mod-408 / mod-411 behind thin HTTP clients or in-process functions — the point is the *ordering* and the *composition*, not the real planes. At minimum you must:
   - Call `safety_input_filter(session, input)` before the first model call.
   - Call `safety_output_validator(session, output)` before yielding the final response.
   - For each tool call: call `policy_check(...)` *before* `safety_tool_invocation_hook(...)` *before* `resolve_tool(...)`.
   - Open a `gen_ai_span(...)` around the model call with the framework-specific name.
   - Emit a `ReliabilityRecord` at turn completion.
3. Never leak framework-native types outside its own module. A `from anthropic import Message` in your agent code fails the exercise.
4. Have a corresponding adapter unit test driving one turn with a stub model and asserting the span order matches chapter-02's diagram.

### Part 3 — The `AgentRevision` manifest emitter

- Author `paved_road_agent/schemas/agent-revision-v1.json` — a JSON Schema for the manifest shape in chapter 01.
- `paved_road_agent/manifest.py` exposes `build_manifest(agent_class, git_revision, ci_data) -> dict` that returns the manifest with all required fields populated. `ci_data` is a dict containing the CI gate results — your emitter accepts the shape the chapter-05 CI template would produce.
- The emitter writes `.platform/agent-revision.yaml` to the repo on `platform build` (or an equivalent `paved-road build` command you implement).
- A validator (`paved_road_agent/manifest.py::validate(manifest) -> list[str]`) returns the list of schema errors; empty list means valid.

### Part 4 — The Copier template

- A `templates/agent-chat-<framework>/` directory layout as in chapter 03.
- `copier.yml` with the parameters from chapter 03: `agent_slug`, `tenant_id`, `workload_class`, `tier`, `model_profile`.
- The scaffolded repo contains:
  - `src/agent.py` — the `@agent(...)` decorator + adapter inheritance in a single ~15-line file.
  - `prompts/system.md` — a placeholder system prompt.
  - `fixtures/regression/golden-path.jsonl` — one fixture that drives the agent against a stub model and expects a specific response.
  - `.platform/manifest-template.yaml` — the compile-time manifest seed.
  - `pyproject.toml` — depends on the paved-road SDK.
  - `.copier-answers.yml` — auto-maintained by Copier.
  - A trivial `Makefile` with at least `make test` (runs pytest) and `make build` (runs `paved-road build`).
  - A `README.md` — scaffolded README for the agent, with a ninety-second walkthrough.

Verify: `copier copy templates/agent-chat-<framework> /tmp/writer-v1 --defaults --trust -d agent_slug=writer-v1 -d tenant_id=acme-financial -d tier=T1` succeeds; `cd /tmp/writer-v1 && pip install -e . && pytest` passes.

### Part 5 — README (~1 page)

Document:

- The framework you chose and why (one sentence).
- The adapter-boundary rule you enforced (e.g., which identifiers you guarded against). If you have a lint rule, name it.
- Two "I was surprised by" observations. Candidates: the framework's hook model does not map cleanly onto the `before_turn` / `after_turn` shape; the `AgentRevision` manifest grew a field you did not expect while writing the emitter; the ninety-second budget was easier / harder than expected for scaffolded repo first-boot.
- One open question for the SDK's next review — e.g., "my adapter needs to call `anthropic.messages.create` in a non-streaming code path for one tool-call retry; is the escape-hatch block the right shape for that, or does this suggest a `run_single_call(...)` method on the interface?".

## Starter guidance

- **Start with the interface and the manifest shape.** Before writing any adapter code, write `interface.py` and `schemas/agent-revision-v1.json`. If either of those shapes shifts after you start the adapter, you will rewrite the adapter.
- **Write the stub-plane functions first.** `safety_input_filter`, `policy_check`, `resolve_tool`, `safety_tool_invocation_hook`, `safety_output_validator` can all be in-process no-op stubs for this exercise. Compose the adapter against the stubs; replace them with real calls later.
- **Use the OpenTelemetry Python SDK's in-memory exporter for tests.** `InMemorySpanExporter` lets you capture the span tree your adapter emits and assert its order without running a collector.
- **Accept that the first adapter is messy.** Chapter 02's six-item adapter-authoring checklist is what a *shipped* adapter looks like. For this exercise, satisfy the first four items (interface, no leaks, versioning, instrumentation) and leave the last two (full production integration tests, documented escape hatches) for exercise 02.
- **Validate the manifest early.** A unit test that calls `build_manifest(...)` and then `validate(manifest)` and asserts an empty error list catches schema drift immediately.
- **Keep the sample agent trivial.** A `WriterAgent` that returns "hello" against the stub model is fine. The point is the *composition*, not the agent logic.
- **Mark `<!-- needs-research: ... -->` on anything you cannot verify.** Framework version pins, exact attribute names in GenAI semconv, OpenAI Agents SDK method names — all of these move. The flag is cheaper than silently guessing.

## Acceptance criteria

Mark done when:

- [ ] `paved_road_agent/interface.py` defines `PavedRoadAgent` with no framework-native identifiers in its type signatures.
- [ ] `paved_road_agent/adapters/<framework>.py` implements the adapter; `isinstance(WriterAgent(), PavedRoadAgent)` is `True`.
- [ ] The adapter composes the eleven planes in chapter-02 order; a unit test drives one turn and asserts the span order.
- [ ] A separate unit test asserts that no framework-native identifier appears in the scaffolded repo's `src/agent.py` (grep against the known identifier list).
- [ ] `paved_road_agent/schemas/agent-revision-v1.json` exists; `validate(build_manifest(WriterAgent, ...))` returns an empty error list.
- [ ] The Copier template scaffolds a working agent; `pip install -e . && pytest` passes in the scaffolded repo.
- [ ] `.platform/agent-revision.yaml` is written by `paved-road build` with all required fields populated.
- [ ] README includes the framework choice, the enforced adapter-boundary rule, two "I was surprised by" observations, and the open question.
- [ ] No step relies on a VPN, cluster, or production credentials.

## Stretch goals

- **Implement a second adapter.** The SDK's claim is "five adapters, one shape"; even one additional adapter proves the shape is honest.
- **Implement the `paved-road lint imports` command.** A CLI subcommand that reads the scaffolded repo and fails if a framework-native identifier appears outside a tagged escape-hatch block. Chapter 05.
- **Add the attribute-contract test.** Capture the span tree in the unit test and assert the required `gen_ai.*` and `platform.*` attributes are present. This is the mini version of chapter 05's attribute-contract gate.
- **Author a second template** for a different `{workload_class, tier}` cell (e.g., `agent-batch-<framework>` for `batch-document`).
- **Hook up a real safety-plane stub.** Replace the in-process `safety_input_filter` with an HTTP call to the exercise-01 stub from `mod-411`; verify the span tree and the `SafetyRecord` emission.
- **Author the SemVer-pin policy document.** A one-page ADR in `docs/ADR/` explaining how the adapter's version tracks the framework's; chapter 07's deprecation ladder references it.
- **Verify the ninety-second budget.** Write a smoke test that times `copier copy` + `pip install -e .` + `make test` + first `paved-road turn` against the stub model; assert under ninety seconds. This is the chapter-04 claim verified on your own machine.
