# Framework Session State: One Abstraction, Five Frameworks

Agent frameworks each ship their own idea of "session state": a checkpoint in LangGraph, a response id in the OpenAI Responses API, a session in the Anthropic Agent SDK, a session store in Google ADK, a runtime object in the Microsoft Agent Framework. If your platform exposes any of these directly to product teams, two things happen. First, product teams get coupled to one framework. Second, every substrate migration (chapter 03) or substrate upgrade (chapter 05) becomes an N-way product-team migration. The platform's job is to prevent both.

This chapter defines a platform-owned session-state abstraction, shows how each framework maps into it, and names the trade-offs you have to take on when you do the mapping. You are not throwing away the frameworks — you are *framing* them so the substrate choice and the framework choice decouple.

> Frameworks evolve fast. Every framework-specific claim here cites the current primary doc in `resources.md`. Before acting on a specific feature name, verify it against the version you are deploying — this track marks anything unverifiable with `<!-- needs-research -->`.

## The platform-owned abstraction

The abstraction has three layers: an interface the SDK (mod-412) exposes to agent code, a serialization format the runtime stores, and an adapter per framework that bridges the two.

### Interface — what agent code sees

```python
class SessionState:
    """Platform-owned abstraction. Agent frameworks are adapted into this."""

    @property
    def session_id(self) -> str: ...

    @property
    def tenant(self) -> str: ...

    @property
    def cost_ledger(self) -> CostLedger: ...

    def messages(self) -> Iterable[Message]: ...
    def append_message(self, msg: Message) -> None: ...

    def memory(self, namespace: str) -> MemoryStore: ...     # long-term, usually pgvector

    def scratchpad(self) -> MutableMapping[str, Any]: ...    # small KV, cleared on completion

    def artifact(self, key: str) -> ArtifactRef: ...         # object-store pointer, not inline bytes

    def checkpoint(self, reason: str = "yield") -> None: ...
    # explicit yield-point; runtime decides whether to actually snapshot
```

Four things make this interface load-bearing:

1. **Nothing in the interface is a framework type.** `Message`, `CostLedger`, `MemoryStore`, `ArtifactRef` are platform types, not `langchain_core.messages.AIMessage`.
2. **Memory and artifacts are references, not inline.** Chapter 02 said: keep the hot row small. Agents fetch memory and artifacts through the `SessionState` API so the platform decides where they live.
3. **Scratchpad is opt-in and small.** It exists so product teams can stash per-session state the framework does not model. It is cleared on completion so it cannot accidentally become a parallel history.
4. **Checkpointing is explicit but advisory.** Agent code says "here is a good yield point"; the runtime decides whether to actually snapshot (back-off, rate-limit, batch). Agent code does not get to *force* a snapshot every tick — that is a platform choice, not a product one.

### Serialization — the on-disk shape

The serialized session is a versioned blob plus small typed columns:

```
session_row:
  session_id        text primary key
  tenant            text not null, indexed
  state             text not null                 # RUNNING / WAITING_* / ...
  schema_version    int  not null                 # evolves independently
  updated_at        timestamptz
  cost_ledger_json  jsonb                         # queryable for dashboards
  checkpoint        bytea | jsonb                 # opaque body (see below)

session_memory:
  session_id × namespace × embedding + payload    # pgvector; chapter 05
session_artifacts:
  session_id × key  →  object_store_url, sha256, bytes
```

`checkpoint` is where framework-specific state lives, wrapped in a thin envelope:

```json
{
  "schema": "aps.session.v1",
  "framework": "langgraph",
  "framework_version": "0.2.x",
  "adapter_version": 3,
  "payload": { /* adapter-produced blob */ }
}
```

The envelope exists for one reason: when you change adapters (new framework version, new field, new encoding) you rewrite the adapter and bump `adapter_version`, and the resume path can pick the right decoder. Without the envelope, every adapter change is a destructive migration against live sessions.

### Adapters — one per framework

An adapter is a small, well-tested translator in each direction:

```python
class FrameworkAdapter(Protocol):
    framework_name: str
    adapter_version: int
    def to_checkpoint(self, framework_state: Any) -> bytes: ...
    def from_checkpoint(self, blob: bytes) -> Any: ...
    def apply_platform_scratchpad(self, framework_state, kv: Mapping) -> Any: ...
    def extract_messages(self, framework_state) -> list[Message]: ...
    def extract_cost(self, framework_state, provider_response) -> CostDelta: ...
```

Five methods is enough; resist adding more. The adapter's *only* job is lossless translation and cost extraction — it is not where agent policy lives.

## Mapping each framework

Below is the mapping from each framework's own state model into the abstraction. Cite each against the primary doc before you implement — and flag `<!-- needs-research -->` where the current version diverges from what is written here.

### LangGraph (open-source graph runtime)

LangGraph's state model is a graph of nodes communicating via *channels* (named state slots). It ships a `Checkpointer` abstraction (in-memory, SQLite, Postgres implementations) that persists the channel state after each super-step, plus a built-in `MemorySaver` concept layered on top. See <https://langchain-ai.github.io/langgraph/concepts/persistence/>.

**Map:**

- **Channel state → `payload`.** LangGraph's checkpointer gives you a serializable snapshot per step; the adapter stores it as the payload.
- **Message history → `messages()`.** LangGraph channels often include a messages channel; the adapter projects it into the platform message type, not vice-versa.
- **LangGraph checkpoint thread_id → platform `session_id`.** One-to-one; the adapter keeps the map.
- **Human-in-the-loop (interrupt) → `WAITING_SIGNAL` with a conventional name.** LangGraph interrupts map onto the signal model from chapter 02; the resume payload carries the human response.

**Watchpoints:**

- LangGraph version churn is real; pin exact versions per adapter and gate upgrades with a replay test suite (chapter 02's resume semantics).
- Do not use LangGraph's own Postgres checkpointer as *your* system of record — layer the adapter on top of the platform's own Postgres schema so you keep the envelope.

### OpenAI Responses API and Agents SDK

The OpenAI Responses API (<https://platform.openai.com/docs/api-reference/responses>) exposes a server-side response object you can thread via `previous_response_id`. The OpenAI Agents SDK (<https://openai.github.io/openai-agents-python/>) builds on top.

**Map:**

- **`response_id` → part of `payload`.** The platform's session is the system of record; the `response_id` is cached inside the payload so the adapter can resume by referencing it on the next turn.
- **Server-side state ≠ platform state.** Do not treat OpenAI's server-side session as durable from the platform's perspective. The platform must still own the checkpoint. Treat server-side state as a performance optimisation — if it disappears or the retention expires, the adapter must be able to reconstruct the conversation from the platform's message log.
- **Structured outputs / tool-use schemas → platform tool-gateway (mod-403), not stored in session.** Configuration lives outside the checkpoint.

**Watchpoints:**

- Rate-limit headers from OpenAI are cost-ledger inputs — the adapter's `extract_cost` is the right place to parse them.
- Policy and retention on OpenAI's side are not under your control. Document the fallback path (reconstruct from platform message log) and test it.

### Anthropic Agent SDK

The Anthropic Agent SDK (<https://docs.anthropic.com/en/api/agent-sdk/overview>) exposes session state primitives for its agent loop.

**Map:**

- **Agent SDK session state → `payload`.** Same shape as LangGraph: serialize the SDK's session object, store as payload, bump `adapter_version` on SDK upgrades.
- **Message history → platform messages.** Anthropic's message types project to the platform type; keep the projection lossless (tool_use blocks, tool_result blocks, cache control markers).
- **Context compaction (prompt caching, chapter 02's compaction) → runtime concern.** The adapter surfaces cache markers in messages so the runtime's compaction policy can decide what to drop; the policy choice is platform-owned, not agent-owned.

**Watchpoints:**

- The Anthropic Agent SDK lineage includes the earlier Claude Agent SDK naming; verify the version your adapter targets and track breaking changes.
- Tool-use and tool-result blocks include ids the SDK threads together. The adapter must preserve them verbatim to keep resume valid.

### Google Agent Development Kit (ADK) and Vertex AI Agent Engine

Google's ADK (<https://google.github.io/adk-docs/>) models agents with session state, memory, and events. Vertex AI Agent Engine (<https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/overview>) is Google's hosted runtime for ADK-style agents.

**Map:**

- **ADK Session (events, state, artifacts) → `payload` + platform memory / artifact stores.** The adapter projects:
  - ADK session.state (short-term KV) → platform scratchpad for durable keys; otherwise into `payload`.
  - ADK session.events (ordered event log) → platform message log.
  - ADK artifact service → platform artifact store (object storage), not inline.
- **ADK's memory service → platform memory (pgvector via `session.memory(ns)`).** Do not let two memory stores exist for the same session.
- **If you use Agent Engine as the runtime**, the platform is *wrapping* a hosted service. The envelope and the adapter still apply, but the "resume" semantics now include "is this session still alive on the hosted side?" Document the health check.

**Watchpoints:**

- ADK is young; expect API churn. Pin versions and gate adapter upgrades with the replay suite.
- The hosted Agent Engine trades operational cost for vendor coupling. Chapter 03's escape-hatch discussion applies: write down the migration path off it before committing.

### Microsoft Agent Framework (and AutoGen / Semantic Kernel lineage)

Microsoft's Agent Framework consolidates earlier work (AutoGen, Semantic Kernel) into a single Microsoft-supported runtime for agent code. Verify current docs at <https://learn.microsoft.com/en-us/agent-framework/> (replace with the active entry point if the URL changes) and Azure AI Foundry Agent Service (<https://learn.microsoft.com/en-us/azure/ai-services/agents/overview>). <!-- needs-research: confirm the current canonical Microsoft Agent Framework docs URL and its exact session-state primitives at your onboarding time -->

**Map:**

- **Framework runtime state → `payload`.** Same envelope pattern.
- **Message history → platform messages.** Lossless projection.
- **If the framework exposes a hosted runtime** (Foundry Agent Service), the same wrapping pattern applies as with Google Agent Engine — platform envelope, platform scratchpad, hosted runtime becomes a substrate rather than a state owner.

**Watchpoints:**

- The AutoGen / Semantic Kernel consolidation means names and APIs shift; the adapter is where you absorb that churn.
- Microsoft-specific telemetry (Application Insights, OTel) should feed into mod-408's observability plane via the same adapter boundary.

## Cross-cutting rules the abstraction enforces

Every framework's adapter has to respect four rules the platform-owned interface dictates:

1. **No framework owns the session id.** The platform mints it; adapters translate inward.
2. **No framework owns the cost ledger.** Adapters extract per-call cost (from provider response headers, token counts, tool-gateway billing events) into the platform ledger.
3. **No framework owns the message log.** The platform message log is the audit surface (mod-407, mod-408). Framework messages are projected in; framework memory is projected in; nothing stays exclusively inside a framework store.
4. **No framework owns the configuration.** Prompts, tool manifests, model IDs are versioned platform configuration referenced by id in the payload, not embedded in it.

If an adapter cannot respect one of these rules, the framework does not fit the platform yet — either upstream the fix or scope the framework to a product-specific opt-out contract (mod-401, chapter 04).

## What this buys the platform

With the abstraction in place:

- **Substrate migration is tractable.** You can swap durable-execution substrates (chapter 03) without product teams noticing, because the envelope and adapters separate "where the bytes live" from "what the bytes mean".
- **Framework version upgrades are gated.** `adapter_version` bumps go through the replay suite; any divergence is caught before touching production sessions.
- **Multi-framework agents are possible without heroics.** A product team using LangGraph can call a sub-agent written on the Anthropic Agent SDK; the platform's A2A plane (mod-409) ferries messages through the platform types, not framework types.
- **Observability is uniform.** Every trace, every eval replay (mod-407), every incident (mod-410) reads the same message log and cost ledger regardless of which framework wrote it.

## Common failures and their fix

- **"We just use the framework's own checkpointer."** You will regret it the first time you upgrade the framework and need to migrate live sessions, or the first time a different framework shows up in the fleet. Keep the envelope.
- **"We will add a framework method directly on `SessionState`."** No — the adapter is the only place framework types appear. If you want a capability (e.g. prompt-caching marker) on the interface, it is a platform concept by the time it crosses the boundary.
- **"We let the agent write directly into Postgres from the framework."** That bypasses the envelope, the version marker, the scratchpad bound, and the observability wiring. Every direct-write is a future migration bill.
- **"We cache messages inside the framework and project only when asked."** Divergence between framework view and platform view is the single hardest class of bug to debug in an agent runtime. Project every change through the adapter eagerly.

## Summary

- Expose one platform-owned `SessionState` interface to agent code. Nothing on it is a framework type.
- Store a versioned envelope with a thin adapter per framework. `adapter_version` lets you evolve translators without destructive migration.
- Map LangGraph, OpenAI Responses, Anthropic Agent SDK, Google ADK, Microsoft Agent Framework through adapters only — never expose their native state objects to product teams.
- The abstraction's payoff is substrate migration, framework upgrade gating, multi-framework agents, and uniform observability.

Next chapter: the state substrates themselves — Postgres (with pgvector), Redis, object storage — and whether each belongs on a Kubernetes operator or on a serverless fabric like Knative.
