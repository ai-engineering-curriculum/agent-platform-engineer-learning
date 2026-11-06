# exercise-01: Session State Abstraction Contract

**Estimated effort:** 3 hours · **Output:** one interface specification + one envelope schema + two adapter sketches · **Pairs with:** chapters 01, 02, 04.

## Objective

Write the platform-owned `SessionState` contract that your agent runtime will expose to product teams. This is the artefact every later exercise in the module (and several modules downstream — mod-403, mod-407, mod-412) will reference. If you get it right once, you re-use it forever; if you get it wrong, every subsequent handoff is a renegotiation.

## Prerequisites

- You have read chapters [`01`](../01-the-runtime-plane-and-session-as-contract.md), [`02`](../02-session-lifecycle-checkpoint-resume-cancellation.md), and [`04`](../04-framework-state-into-unified-abstraction.md).
- You have skimmed one or more of: [LangGraph Persistence](https://langchain-ai.github.io/langgraph/concepts/persistence/), [OpenAI Responses API reference](https://platform.openai.com/docs/api-reference/responses), [Anthropic Agent SDK overview](https://docs.anthropic.com/en/api/agent-sdk/overview), [Google ADK docs](https://google.github.io/adk-docs/).
- You have access to a text editor and any language you are comfortable writing an interface sketch in (Python, TypeScript, Go, Java, Rust — all fine).

## Problem statement

Produce the specification for the session-state abstraction your runtime exposes. The specification is intentionally language-agnostic in its contract but concrete enough that two independent engineers could implement a framework adapter without further clarification.

The deliverable is three parts; every part is required.

## Requirements

### Part 1 — The interface spec (≈ 1 page)

Write the `SessionState` interface, including every method and property, each with a one-sentence contract. Cover at minimum:

- Identity and tenancy (session id, tenant, priority tier).
- Cost ledger access (read running totals, append a cost delta — the *write* is reserved for the model / tool / sandbox gateways; product agents only read).
- Message log access (iterate, append; projection into platform `Message` type).
- Memory access (`memory(namespace)` returning a platform `MemoryStore`, which itself is a separate small interface).
- Scratchpad access (`scratchpad()` returning a bounded mutable mapping; document the bound).
- Artifact access (`artifact(key)` returning an `ArtifactRef` — URL + content hash + size, never inline bytes).
- Explicit yield / checkpoint hint (`checkpoint(reason)` — advisory, platform decides whether to actually snapshot).
- Any lifecycle observables the agent code is allowed to read (current state, cancellation token).

For each method, write:

- **Pre-conditions.** What state must the session be in?
- **Post-conditions.** What is observably true after the call returns?
- **Error classes.** Named exceptions (e.g. `BudgetExhausted`, `SessionCancelled`, `AdapterIncompatible`). Match the three obligations from chapter 01.
- **Determinism note.** Does the method have to be replay-safe inside a durable-execution workflow body?

### Part 2 — The envelope schema

Define the on-disk envelope from chapter 04 as a schema. Pick a schema language you would actually use (JSON Schema, Protobuf, Avro — all fine). Include at minimum:

- `schema_version` (integer, independent of adapter version).
- `framework` (enum of supported adapter names).
- `framework_version` (string, framework-reported version).
- `adapter_version` (integer, bumped per adapter implementation change).
- `payload` (bytes or framework-specific sub-schema).
- `created_at`, `updated_at` (timestamps).

Write a one-paragraph migration policy answering:

- When `schema_version` changes, what happens to in-flight sessions?
- When `adapter_version` changes, what happens to in-flight sessions?
- When `framework_version` changes (framework upgrade), what happens to in-flight sessions?

### Part 3 — Two adapter sketches

Pick *two* frameworks from chapter 04 (LangGraph, OpenAI Responses, Anthropic Agent SDK, Google ADK, Microsoft Agent Framework). For each:

- Sketch the five adapter methods (`to_checkpoint`, `from_checkpoint`, `apply_platform_scratchpad`, `extract_messages`, `extract_cost`) at code level — pseudocode is fine, but every line should be defensible.
- Identify at least one field in the framework's native state that *does not* map cleanly onto the platform abstraction, and write one sentence on how you would handle it (project it, drop it with a warning, or force an opt-out contract per mod-401 chapter 04).
- Identify the framework-specific version pins the adapter needs, and how you will catch an upstream breaking change in CI.

## Starter guidance

- Start with `chapter 04`'s `SessionState` sketch and *change* it. The shape in the chapter is a starting point, not a specification; the exercise's value is the decisions you add on top (error classes, determinism notes, migration policy).
- When in doubt whether a method belongs on `SessionState`, ask the chapter 01 three-test: is it multi-tenant? substrate-not-feature? interface-not-fork? If the answer is "no" on any of them, it probably belongs on a neighbour plane (tool gateway, model gateway, memory service).
- Do not design features for frameworks you have not read. If you pick LangGraph as one of your two frameworks, actually read the LangGraph persistence docs before writing the adapter sketch. The whole point of the exercise is that the adapter is *lossless* against the framework's real contract.
- Mark `<!-- needs-research: ... -->` wherever you cannot verify a current-version detail. The flag is cheaper than silently guessing.

## Acceptance criteria

You can mark this exercise done when all of these are true:

- [ ] A peer engineer could implement your interface in a language of their choice from your spec alone, without asking you questions.
- [ ] Every method's error classes map onto one of the three chapter 01 obligations (at-most-once effective tool calls, bounded cancellation, live cost ledger) or are justified as a fourth obligation you added explicitly.
- [ ] Your envelope schema answers the three migration questions in Part 2 in writing, not implicitly.
- [ ] Your two adapter sketches each name at least one native-framework field that does not map cleanly, with a documented handling decision.
- [ ] Your adapter sketches each include the CI gate that catches an upstream breaking change (a replay test, a schema diff, a version pin — pick one).
- [ ] You have flagged any unverified framework detail with `<!-- needs-research: ... -->`.

## Stretch goals

- Add a *third* adapter sketch for a framework you have less experience with — the mismatch between your mental model and the framework's real shape is where you will learn the most.
- Prototype the adapter in code against a trivial agent (one tool call, one LLM call) and write out the checkpoint payload. Compare the serialised size across two frameworks — the difference is often an eye-opener.
- Draft the two handoff documents your adapter produces: one for `senior-agentic-ai-engineer` (product teams using the abstraction) and one for your own future mod-410 work (the SLIs the adapter exposes). These will be re-used in those modules.
