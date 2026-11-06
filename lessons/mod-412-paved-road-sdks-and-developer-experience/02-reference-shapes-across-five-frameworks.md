# Reference Shapes Across Five Frameworks

Chapter 01 pinned the plane's slice: one `PavedRoadAgent` reference shape, five framework adapters, all eleven planes behind one call. This chapter opens the shape itself.

Five frameworks dominate the mainstream agent ecosystem at the time of writing (2026-11-06). Each has strong reasons a stream-aligned team might pick it — model-provider affinity, workflow model, feature coverage, prior investment — and none of them is a clear winner for every workload. The paved-road SDK has to let a team pick its framework and still get one shape, one instrumentation surface, and one adoption story.

The adapter pattern is how this is done. The platform defines one interface. Each framework gets a thin, well-versioned adapter whose job is "take the framework's native shape and expose the platform's shape". Stream-aligned teams write against the platform's shape. The adapter boundary is where framework change lands.

## The five reference frameworks

| Framework | Primary docs | What it ships | Workload affinity |
|---|---|---|---|
| **Anthropic Agent SDK** | [docs.anthropic.com/en/api/agent-sdk/overview](https://docs.anthropic.com/en/api/agent-sdk/overview) | TypeScript and Python SDKs that package Claude Code's agent harness as a reusable loop — context management, tool use, subagents, hooks. | Teams whose primary model is Claude; workloads that benefit from Anthropic's built-in file / bash / web-search tools and from the Agent SDK's hook model. |
| **OpenAI Agents SDK** | [openai.github.io/openai-agents-python](https://openai.github.io/openai-agents-python/) | A Python-first SDK for building agents with handoffs, guardrails, sessions, and tool-call tracing, built on the OpenAI Responses API. | Teams whose primary model is OpenAI's family; workloads that use handoffs between specialised sub-agents. |
| **Google Agent Development Kit (ADK)** | [google.github.io/adk-docs](https://google.github.io/adk-docs/) | An open-source, multi-agent, model-agnostic Python framework with a clear agent / tool / session / runner separation; deploys to Vertex AI Agent Engine or any container runtime. | Teams on GCP or Vertex AI; multi-agent systems where the ADK's `SequentialAgent` / `ParallelAgent` / `LoopAgent` composition fits. |
| **Microsoft Agent Framework** | [learn.microsoft.com/en-us/agent-framework/overview](https://learn.microsoft.com/en-us/agent-framework/overview) | A .NET and Python framework, open-source, that merges the Semantic Kernel and AutoGen lineages; agents, workflows, runtimes, and plugins. | Teams on the Microsoft stack; Enterprise .NET agents; workloads that need a deterministic workflow orchestration surface alongside the LLM. |
| **LangGraph + Temporal** | [langchain-ai.github.io/langgraph](https://langchain-ai.github.io/langgraph/) · [docs.temporal.io](https://docs.temporal.io/) | LangGraph gives you an explicit graph of nodes and edges over an LLM runtime; Temporal gives you a durable-execution substrate underneath (workflow / activity / signal / timer). Compose: the LangGraph loop runs *inside* a Temporal workflow. | Teams whose workloads are long-running, require durable state and retries, or benefit from a visible graph of the agent's reasoning steps. Cross-reference mod-402 chapter 03 on durable-execution substrates. |

These are the five the SDK's reference shapes (chapter 01) explicitly cover. The adapter mechanism is the same for any additional framework a tenant demands, but each additional adapter is a maintenance cost; chapter 07 discusses the "add adapter" governance.

> Every claim in this chapter about a framework's specific class, decorator, or method name is current at the time of writing (2026-11-06). Before shipping a specific identifier into an adapter, verify it against the version you are packaging — this track uses `<!-- needs-research -->` for anything unverifiable.

## The platform interface — `PavedRoadAgent`

The one interface everything compiles against. Nothing in it is framework-native.

```python
# paved_road_agent/interface.py
from __future__ import annotations
from typing import AsyncIterator, Protocol, runtime_checkable
from .types import (
    AgentManifest,           # mirrors AgentRevision but build-time
    Message,                 # platform message type
    TurnEvent,               # platform turn-event type (mod-402)
    ToolInvocation,          # platform tool-invocation type (mod-403 / mod-405 / mod-411)
    AgentInput,              # user input shape
    AgentOutput,             # final response shape, including structured output
    SessionState,            # mod-402 chapter 04
)

@runtime_checkable
class PavedRoadAgent(Protocol):
    """Platform-owned paved-road agent interface. Frameworks adapt into this."""

    manifest: AgentManifest

    async def run(
        self,
        input: AgentInput,
        session: SessionState,
    ) -> AgentOutput: ...

    async def stream(
        self,
        input: AgentInput,
        session: SessionState,
    ) -> AsyncIterator[TurnEvent]: ...

    # Lifecycle hooks — override to extend. Called by the adapter around every turn.
    async def before_turn(self, session: SessionState, input: AgentInput) -> None: ...
    async def after_turn(self, session: SessionState, output: AgentOutput) -> None: ...
    async def before_tool_call(self, session: SessionState, invocation: ToolInvocation) -> None: ...
    async def after_tool_call(self, session: SessionState, invocation: ToolInvocation) -> None: ...
    async def on_error(self, session: SessionState, exc: BaseException) -> None: ...
```

Four design choices are load-bearing here:

1. **The only external types are platform types.** `Message`, `TurnEvent`, `ToolInvocation`, `AgentInput`, `AgentOutput`, `SessionState` are all defined in `paved_road_agent.types`. No `anthropic.*`, no `openai.*`, no `langgraph.*` type ever appears in the interface. This is what makes the framework a *detail* rather than the *contract*.
2. **`run(...)` returns an `AgentOutput`; `stream(...)` returns `TurnEvent`s.** The two methods are equivalent in capability; `stream` is the superset. Adapters implement `stream` and the base class implements `run` on top. This maps cleanly onto the Anthropic Agent SDK's query/stream distinction, the OpenAI Agents SDK's `Runner.run` / `Runner.run_streamed`, the Google ADK's `Runner.run_async`, the Microsoft Agent Framework's thread/stream model, and the LangGraph `astream` interface.
3. **Lifecycle hooks are platform hooks, not framework hooks.** The adapter's job is to invoke these hooks at the right moments regardless of the framework's own hook model. `before_turn` fires once per user-turn boundary; `before_tool_call` fires once per tool invocation. The standardised instrumentation hook library (chapter 05) *subscribes* to these hooks to emit span attributes and records. A product engineer who only wants defaults never touches them.
4. **`SessionState` is passed in; the agent does not create it.** The runtime (mod-402) owns session lifecycle. The adapter calls `run(input, session)` with a session the runtime materialised. The agent manipulates the session via the mod-402 chapter 04 interface. This is what lets the runtime do checkpoint, resume, timer, and cancellation without the agent code noticing.

An agent-side developer rarely overrides the hooks. The typical shape is:

```python
# writer-agent/src/agent.py
from paved_road_agent import PavedRoadAgent, agent
from paved_road_agent.adapters.anthropic import AnthropicAgentAdapter

@agent(
    workload_class="interactive-chat",
    tier="T1",
    model_profile="paved-road-claude-balanced",
    system_prompt="prompts/writer.md",
    tools=["web.search", "notes.read", "notes.write"],
)
class WriterAgent(AnthropicAgentAdapter, PavedRoadAgent):
    pass  # defaults are enough for the paved-road path
```

The decorator `@agent(...)` reads the metadata and emits the compile-time portion of the manifest (`.platform/agent-manifest.yaml`). The build step emits the full `AgentRevision` (chapter 01) by joining that with the CI-stream data. No framework type appears in the agent repo.

## Adapter 1 — Anthropic Agent SDK

The [Anthropic Agent SDK](https://docs.anthropic.com/en/api/agent-sdk/overview) exposes an agent loop with hooks, subagents, and tool-use primitives. The SDK ships a `query` / `streaming` interface and a `hooks` interface. The adapter's job is to compose these.

Reference shape of the adapter (concepts, not literal source — version-pin against the SDK release you package):

```python
# paved_road_agent/adapters/anthropic.py  — conceptual outline
from anthropic.agent_sdk import query, streaming, SubagentDefinition, ToolSpec
from paved_road_agent.interface import PavedRoadAgent
from paved_road_agent.types import TurnEvent, AgentOutput
from paved_road_agent.instrumentation import gen_ai_span, platform_span
from paved_road_agent.hooks.safety import safety_input_filter, safety_output_validator
from paved_road_agent.hooks.policy import policy_check
from paved_road_agent.hooks.tooling import resolve_tool  # mod-403

class AnthropicAgentAdapter(PavedRoadAgent):
    """Adapts Anthropic Agent SDK's query/stream into PavedRoadAgent."""

    async def stream(self, input, session):
        system_prompt = self._compose_system_prompt(session)
        tools = [await resolve_tool(t) for t in self.manifest.tools]
        with gen_ai_span("anthropic.agent_sdk.query", session=session, model=self.model):
            async for event in streaming(
                user_prompt=await safety_input_filter(session, input),
                system_prompt=system_prompt,
                tools=tools,
                on_tool_call=self._wrap_tool_call(session),  # policy → safety → mod-403 dispatch
                on_message=self._wrap_message(session),
            ):
                for pr_event in self._translate_event(event, session):
                    yield pr_event
                if pr_event.kind == "final_response":
                    pr_event.text = await safety_output_validator(session, pr_event.text)
```

Four things the adapter is doing for you:

1. **Composing the three safety hooks** (mod-411) in the right order — input filter before the model call, output validator after the final response, tool-invocation hook inside the tool-call interceptor.
2. **Composing the policy plane** (mod-405) before the tool-invocation safety hook — mod-411 chapter 01 requires the order.
3. **Translating Anthropic's event shape into `TurnEvent`s** so the `stream` consumer never sees `anthropic.*` types.
4. **Opening the right spans** for the mod-408 vocabulary — one span around the agent's run, one per model call, one per tool call. The standardised instrumentation hook library (chapter 05) supplies the `gen_ai_span` / `platform_span` helpers.

The adapter has one load-bearing constraint: *it must track the Anthropic Agent SDK's minor-version cadence as patch-level releases of the adapter*. If Anthropic releases a new SDK with a renamed `on_tool_call` hook, the adapter version bumps patch; the paved-road SDK's major-minor stays stable; stream-aligned teams see a dependency bump and nothing else. Chapter 07's deprecation ladder is how larger adapter shifts (a `v2` Anthropic Agent SDK with a non-migratable surface change) are handled.

## Adapter 2 — OpenAI Agents SDK

The [OpenAI Agents SDK](https://openai.github.io/openai-agents-python/) models agents as objects with instructions, model, and tools; runs them via `Runner.run(...)` or `Runner.run_streamed(...)`; supports handoffs, guardrails, and sessions. The adapter composes `Agent` and `Runner` into `PavedRoadAgent`.

Reference shape:

```python
# paved_road_agent/adapters/openai.py  — conceptual outline
from agents import Agent, Runner
from paved_road_agent.interface import PavedRoadAgent

class OpenAIAgentAdapter(PavedRoadAgent):
    def _build_native_agent(self, session):
        return Agent(
            name=self.manifest.agent_id,
            instructions=self._compose_system_prompt(session),
            model=self._model_for_profile(self.manifest.model_profile),
            tools=self._wrap_tools_for_openai(self.manifest.tools),
            guardrails=self._paved_road_guardrails(),  # bridges to mod-411 input filter
        )

    async def stream(self, input, session):
        agent = self._build_native_agent(session)
        runner = Runner()
        with gen_ai_span("openai.agents.run_streamed", session=session, model=agent.model):
            result = runner.run_streamed(agent, await safety_input_filter(session, input))
            async for event in result.stream_events():
                yield from self._translate_event(event, session)
            final = await safety_output_validator(session, result.final_output)
            yield final_response_event(final)
```

Two framework-specific details:

- **Guardrails bridge.** The OpenAI Agents SDK has a native guardrails concept. The adapter wires those into the SDK's guardrail slot but delegates the actual decision to mod-411's safety plane hooks — the OpenAI guardrail is a thin adapter *into* the platform's safety plane, not a parallel safety stack. One decision-making body; one `SafetyRecord` emission.
- **Handoffs.** The OpenAI Agents SDK supports handoffs between agents. The adapter maps handoffs onto mod-409's A2A primitives; a handoff is an A2A call with the same `trace_id`. Cross-reference mod-409.

Adapter version follows the OpenAI Agents SDK's minor cadence on patch; major shifts (e.g., a non-backwards-compatible change to `Runner` or the Responses-API wiring) go through the deprecation ladder.

## Adapter 3 — Google Agent Development Kit

The [Google ADK](https://google.github.io/adk-docs/) models agents via `LlmAgent` (and the compositional `SequentialAgent`, `ParallelAgent`, `LoopAgent`); runs them via a `Runner` with explicit `Session` management; integrates with Vertex AI Agent Engine. The adapter wraps these into `PavedRoadAgent`.

```python
# paved_road_agent/adapters/google_adk.py  — conceptual outline
from google.adk import Agent, LlmAgent, SequentialAgent, Runner
from paved_road_agent.interface import PavedRoadAgent

class GoogleADKAdapter(PavedRoadAgent):
    def _build_native_agent(self):
        return LlmAgent(
            name=self.manifest.agent_id,
            model=self._model_for_profile(self.manifest.model_profile),
            instruction=self._compose_system_prompt(),
            tools=self._wrap_tools_for_adk(self.manifest.tools),
        )

    async def stream(self, input, session):
        native = self._build_native_agent()
        runner = Runner(agent=native, session_service=self._session_service(session))
        with gen_ai_span("google.adk.run_async", session=session, model=native.model):
            async for ev in runner.run_async(
                user_id=session.tenant,
                session_id=session.session_id,
                new_message=await safety_input_filter(session, input).to_adk_content(),
            ):
                yield from self._translate_event(ev, session)
```

Three framework-specific details:

- **Session service bridge.** The ADK's `SessionService` is replaced by a mod-402 bridge — the ADK operates against an in-memory shim over the platform's `SessionState` so the ADK's own session APIs continue to work while the real state is stored in mod-402.
- **Multi-agent composition.** `SequentialAgent` / `ParallelAgent` / `LoopAgent` compositions are preserved; the adapter wires each sub-agent into mod-411's safety hooks individually, so a multi-agent workload still emits one `SafetyRecord` per hook invocation.
- **Model-agnostic.** ADK supports Gemini, OpenAI, Anthropic, Llama, and more through its model abstraction; the adapter's `_model_for_profile` resolution is identical to the other adapters, routing through mod-406 regardless of the ADK-side model identifier.

## Adapter 4 — Microsoft Agent Framework

The [Microsoft Agent Framework](https://learn.microsoft.com/en-us/agent-framework/overview) is a .NET and Python framework that unifies the Semantic Kernel and AutoGen lineages. Agents expose a `ChatAgent` shape; a `Workflow` surface composes them via a graph / typed-edge model.

```python
# paved_road_agent/adapters/microsoft.py  — conceptual outline
from agent_framework import ChatAgent, Workflow, WorkflowBuilder
from paved_road_agent.interface import PavedRoadAgent

class MicrosoftAgentAdapter(PavedRoadAgent):
    def _build_native_agent(self, session):
        return ChatAgent(
            name=self.manifest.agent_id,
            instructions=self._compose_system_prompt(session),
            model_client=self._client_for_profile(self.manifest.model_profile),
            tools=self._wrap_tools_for_mscaf(self.manifest.tools),
        )

    async def stream(self, input, session):
        native = self._build_native_agent(session)
        thread = native.get_new_thread()
        with gen_ai_span("agent_framework.chat_agent.run_stream", session=session):
            async for chunk in native.run_stream(
                await safety_input_filter(session, input),
                thread=thread,
            ):
                yield from self._translate_chunk(chunk, session)
```

Three framework-specific details:

- **Thread bridge.** The Microsoft Agent Framework's `thread` concept is a conversation scope; the adapter maps `thread` to `SessionState` with one-to-one semantics — the first `get_new_thread()` call is translated to a session load.
- **Workflow surface.** When a workload uses the Workflow surface (multi-agent compositions over a typed graph), the adapter composes the mod-409 A2A plane under each inter-agent edge, so the workflow continues to work locally and in production without the agent author writing A2A calls by hand.
- **.NET parity.** For organisations that write agents in C#, a sibling adapter exists (same interface, C# implementation). The paved-road SDK supports multiple languages; the manifest shape and the obligations are language-independent.

## Adapter 5 — LangGraph + Temporal

[LangGraph](https://langchain-ai.github.io/langgraph/) gives you an explicit graph of nodes over an LLM-driven agent; [Temporal](https://docs.temporal.io/) gives you durable-execution primitives. Composed — LangGraph's node loop *inside* a Temporal workflow — the pair is what many "long-running agent" workloads actually want. Cross-reference mod-402 chapter 03.

```python
# paved_road_agent/adapters/langgraph_temporal.py  — conceptual outline
from langgraph.graph import StateGraph, END
from temporalio import workflow, activity
from paved_road_agent.interface import PavedRoadAgent

@workflow.defn
class _PavedRoadAgentWorkflow:
    @workflow.run
    async def run(self, input, session_ref):
        session = await activity.execute_activity(load_session, session_ref)
        async for event in _langgraph_stream(self.graph, input, session):
            await activity.execute_activity(emit_turn_event, event)
            if event.kind == "tool_call":
                await activity.execute_activity(policy_then_safety_tool_hook, event)

class LangGraphTemporalAdapter(PavedRoadAgent):
    def _compile_graph(self):
        g = StateGraph(self._state_type)
        g.add_node("agent", self._agent_node)
        g.add_node("tool", self._tool_node)
        g.add_conditional_edges("agent", self._should_continue, {"tool": "tool", END: END})
        g.add_edge("tool", "agent")
        return g.compile()

    async def stream(self, input, session):
        handle = await self._temporal_client.start_workflow(
            _PavedRoadAgentWorkflow.run,
            args=[input, session.ref()],
            id=self._workflow_id(session),
            task_queue=self.manifest.task_queue,
        )
        async for turn_event in self._tail_history(handle):
            yield turn_event
```

Four framework-specific details:

- **Workflow determinism.** Temporal's workflow code is deterministic; all non-determinism (LLM calls, tool calls, time) goes into activities. The adapter puts the LangGraph loop inside a workflow but places every LLM call, every tool call, and every mod-411 hook invocation behind activities; the local-dev loop (chapter 04) runs this through a Temporalite single-binary.
- **Checkpoint integration.** LangGraph's own checkpointer is bridged to the Temporal workflow state and to mod-402's `SessionState`; the agent can resume across Temporal worker restarts, LangGraph node boundaries, and Kubernetes rollouts without the agent author doing anything.
- **Task queue naming.** The paved-road SDK's convention is one task queue per `{tenant, workload_class}` — mod-402 chapter 05 goes into the worker-topology shape.
- **Signals and cancellation.** Temporal signals surface as `session.cancel_signal` / `session.pause_signal` on the platform interface; the agent code never writes `workflow.wait_condition(...)` directly.

## The adapter boundary — the policy

Three rules keep the adapter boundary healthy.

### Rule 1 — The platform interface is additive within a major version

Once `PavedRoadAgent` ships `v1`, new methods or new hook points are additive. A `v1` agent compiled against a future `v1.4` SDK keeps working because the new hook has a default implementation in the base class.

A *removal* is a breaking change; it lives on the deprecation ladder (chapter 07), with at least one minor version of coexistence (the method is kept as a deprecated shim), and the deprecation emits a warning on every build that references it.

### Rule 2 — Framework identifiers never cross the adapter boundary

If a framework-native type appears in an agent repo, the paved-road claim is broken. The adapter is the only place `anthropic.*`, `openai.*`, `google.adk.*`, `agent_framework.*`, `langgraph.*`, `temporalio.*` identifiers exist. The agent repo imports `paved_road_agent.*` only.

Enforce with a lint rule in the paved-road CI template (chapter 05): a disallowed-import list that fails the build if an agent repo imports a framework-native identifier outside an explicitly tagged escape-hatch block.

### Rule 3 — Adapters version with their framework; the SDK versions with the platform

| Change | Adapter version impact | SDK version impact |
|---|---|---|
| Patch release of the upstream framework (bugfix) | Adapter patch bump | None |
| Minor release of the upstream framework (new feature, backwards-compatible) | Adapter minor bump, SDK minor bump that includes it | SDK minor bump |
| Breaking change in the upstream framework | Adapter major bump; new adapter name (`anthropic_v2`) coexists with old; old enters deprecation ladder | SDK minor bump that *adds* the new adapter; later SDK major that *removes* the old adapter |
| Adding a new framework | New adapter at `v1.0.0`; SDK minor bump that adds it | SDK minor bump |
| Retiring a framework | Adapter enters deprecation ladder; removed only in an SDK major | SDK major bump |

The rule the ladder enforces is "a stream-aligned team cannot be forced into a framework change by an SDK minor-version bump". If Anthropic ships a `v2` SDK, the paved-road SDK's minor bump *adds* `AnthropicV2AgentAdapter`; the `AnthropicAgentAdapter` continues to work until the deprecation ladder removes it. Chapter 07 walks the ladder.

## Composition of all eleven planes behind `PavedRoadAgent`

Every adapter composes the same stack around every turn. Draw this on a whiteboard before writing adapter code; the ordering matters.

```
PavedRoadAgent.run(input, session)
  │
  ├─ before_turn hook (user-defined, optional)
  │
  ├─ mod-411 input filter         ─────────────────────┐
  │     (safety plane hook 1)                           │
  │                                                     │
  ├─ gen_ai span start            ─ mod-408             │
  │   turn_begin TurnEvent        ─ mod-402             │
  │                                                     │
  ├─ for each model call:                               │
  │     mod-406 route             (model gateway)       │
  │     model call                                      │
  │     gen_ai span event         ─ mod-408             │
  │                                                     │
  ├─ for each tool call proposed:                       │
  │     before_tool_call hook                           │  Every edge
  │     mod-405 policy check      (identity+permission) │  emits records:
  │     mod-411 tool-invoc hook   (safety plane hook 3) │  SafetyRecord,
  │     mod-403 resolve tool                            │  ReliabilityRecord,
  │     [mod-404 sandbox exec]    when tool is code-run │  gen_ai spans
  │     after_tool_call hook                            │
  │     turn_event TurnEvent      ─ mod-402             │
  │                                                     │
  ├─ final response emit:                                │
  │     mod-411 output validator  (safety plane hook 2) │
  │     final_response TurnEvent  ─ mod-402             │
  │     gen_ai span end           ─ mod-408             │
  │                                                     │
  ├─ after_turn hook (user-defined, optional)           │
  │                                                     │
  └─ ReliabilityRecord emit        ─ mod-410  ─────────┘
```

Four consequences of this ordering:

1. **mod-405 policy runs *before* mod-411's tool-invocation hook.** mod-411 chapter 01 is explicit: the safety plane runs on top of the policy plane's allow. A tool call that mod-405 refuses never reaches the safety plane.
2. **mod-411 input filter runs *before* the first model call.** Catching the attack before it reaches the model is cheaper than catching it after.
3. **mod-411 output validator runs *before* `final_response` is yielded.** A tenant never sees a response the validator would have refused.
4. **mod-410's `ReliabilityRecord` emits on turn completion**, carrying token counts (from mod-406's completion record), latency (from mod-408's gen_ai span duration), and the verdict shape. mod-410 chapter 02's SLIs query this record.

An agent that composes the planes in a different order is off the paved road regardless of how many planes it integrates — chapter 05's attribute-contract test fails the build. The paved-road obligation is "one call, correct order, every plane".

## Picking an adapter: the decision matrix

Stream-aligned teams ask "which framework do I pick". The paved road's answer is a matrix, not a mandate:

| If your workload... | ...the paved road recommends | Why |
|---|---|---|
| Is primarily Claude-backed and uses Anthropic's native tool suite (bash, file edit, web search) | `AnthropicAgentAdapter` | Lowest friction; the native Anthropic Agent SDK hooks compose cleanly into the paved-road hooks. |
| Is primarily OpenAI-backed and uses handoffs between specialised sub-agents | `OpenAIAgentAdapter` | Handoff model is first-class; guardrails bridge gives the shortest path to the safety plane. |
| Runs on GCP / Vertex AI and benefits from ADK's multi-agent compositions | `GoogleADKAdapter` | ADK's `SequentialAgent` / `ParallelAgent` / `LoopAgent` maps onto workflows cleanly; deploys to Vertex AI Agent Engine with less bespoke infra. |
| Lives in a .NET shop or already has Semantic Kernel plugins | `MicrosoftAgentFrameworkAdapter` | Only adapter that supports .NET natively; existing SK plugin inventory ports. |
| Is long-running, requires durable state, needs retry / signal / timer primitives | `LangGraphTemporalAdapter` | Only adapter with durable-execution underneath; explicit graph makes the trajectory legible for review. |

Teams that don't fit one of the five pick the closest cell *or* file an "add-adapter" request against the exception register (chapter 07). The SDK's roadmap conversation is the forcing function back onto the five; the exception register is the pressure release when the forcing function is wrong.

## A worked example: the writer agent, four adapters

Take the same agent — a writing-assistant that uses `web.search`, `notes.read`, `notes.write`, and emits a markdown response — and expose it against four adapters. The goal of the exercise is not to pick one; the goal is to see what the adapter boundary buys you.

```python
# writer-agent/src/agent.py  — only this line changes between adapters
from paved_road_agent.adapters.anthropic import AnthropicAgentAdapter as A
# from paved_road_agent.adapters.openai import OpenAIAgentAdapter as A
# from paved_road_agent.adapters.google_adk import GoogleADKAdapter as A
# from paved_road_agent.adapters.microsoft import MicrosoftAgentAdapter as A

from paved_road_agent import PavedRoadAgent, agent

@agent(
    workload_class="interactive-chat",
    tier="T1",
    model_profile="paved-road-balanced",   # the profile resolves to the right model
    system_prompt="prompts/writer.md",
    tools=["web.search", "notes.read", "notes.write"],
)
class WriterAgent(A, PavedRoadAgent):
    pass
```

Four observations:

1. **Only one import line changes.** The agent code is adapter-agnostic because `PavedRoadAgent` is the contract.
2. **The model profile resolves to a framework-appropriate model.** `paved-road-balanced` is a mod-406 profile; it may resolve to Claude on the Anthropic adapter and GPT-4o on the OpenAI adapter. The agent code does not care.
3. **Tools are platform-identifier strings, not framework types.** `web.search` is an mod-403 tool identifier; each adapter resolves it against mod-403 and wraps it in the adapter's own tool-call shape.
4. **Manifest declares the tier.** The scaffolder (chapter 03) uses the tier to pick the policy pack and the SLO template; the manifest records the choice.

The LangGraph + Temporal case is one more line, because it needs the task queue:

```python
from paved_road_agent.adapters.langgraph_temporal import LangGraphTemporalAdapter as A

@agent(
    workload_class="batch-document",   # more appropriate for LangGraph+Temporal
    tier="T2",
    model_profile="paved-road-balanced",
    task_queue="writer-batch",         # Temporal task queue per tenant+workload
    system_prompt="prompts/writer.md",
    tools=["web.search", "notes.read", "notes.write"],
)
class WriterAgent(A, PavedRoadAgent):
    pass
```

If a team genuinely needs features from two frameworks — say, LangGraph's graph visualisation *and* OpenAI's native handoff — that is an exception register entry, not a two-adapter merge. Chapter 07.

## The adapter authoring checklist

Before shipping a new adapter or a major-bumped revision of an existing one, check:

- [ ] The adapter implements `PavedRoadAgent.stream(...)` and satisfies the plane-composition ordering diagrammed above.
- [ ] The adapter never leaks framework-native types outside its own module. CI lint rule enforced.
- [ ] The adapter version is pinned to the framework version it wraps; the pin is on the adapter's `pyproject.toml` (or equivalent) with explicit lower and upper bounds.
- [ ] The adapter's instrumentation uses `paved_road_agent.instrumentation.*` helpers; no bespoke OTel attribute names outside the mod-408 vocabulary.
- [ ] The adapter composes mod-405 policy *before* mod-411's tool-invocation hook, and mod-411 input filter *before* the first model call.
- [ ] The adapter emits `ReliabilityRecord` on turn completion with all mod-410 chapter 02 required fields populated.
- [ ] The adapter has an end-to-end sample in the template catalogue (chapter 03) and a passing integration test against a stubbed mod-402 / mod-403 / mod-406 / mod-411.
- [ ] The adapter's documentation names the framework-specific escape hatches (where an agent can access framework-native types under an explicit tag) and the exception register process for any usage of them.

Chapter 03 opens the scaffolding CLI that turns "the adapter exists" into "a stream-aligned team uses it in ninety seconds".

## Summary

- The paved-road SDK exposes one interface, `PavedRoadAgent`, with no framework-native types in its surface. Adapters wrap framework-native shapes into this interface.
- Five adapters cover the mainstream ecosystem: Anthropic Agent SDK, OpenAI Agents SDK, Google ADK, Microsoft Agent Framework, LangGraph + Temporal. Each adapter composes all eleven planes behind one call, in a documented order: mod-411 input filter → mod-408 span → mod-406 completion → (per tool-call) mod-405 policy → mod-411 tool hook → mod-403 resolve → mod-404 execute → mod-411 output validator → mod-410 `ReliabilityRecord`.
- Three rules keep the adapter boundary healthy: the platform interface is additive within a major version; framework identifiers never cross the adapter boundary; adapters version with their framework while the SDK versions with the platform.
- The decision matrix for picking an adapter is a *recommendation*, not a mandate; workloads that do not fit one of the five go through the exception register (chapter 07), which is how the "add a sixth adapter" conversation is governed.

The next chapter opens the scaffolding CLI and the template catalogue — the first touch point between a stream-aligned engineer and the paved road.
