# exercise-01: A2A Spec Conformance Walkthrough

**Estimated effort:** 2.5 hours · **Output:** a minimal A2A server, an executable conformance test suite (one language), a per-call `A2ACallRecord` capture, a conformance matrix (Markdown table) mapping each A2A concept to a service-mesh primitive, and a short README (~1 page) · **Pairs with:** chapters [02](../02-the-a2a-protocol-and-service-mesh-mapping.md) and [03](../03-inter-agent-routing-and-service-discovery.md); feeds exercises 02 and 03.

## Objective

Read the [A2A protocol specification](https://a2a-protocol.org/) in enough depth to implement a credible A2A server, and author a conformance test suite that catches the common ways an A2A implementation can be wrong. Then produce a conformance matrix that maps each A2A concept to the service-mesh primitive the platform would use to operationalise it. Chapter 02's mapping table becomes a thing you built rather than a thing you read.

A conforming server is the entry ticket onto the platform's agent registry. An implementation that passes the mandatory-surface tests is correct enough to be callable from the paved-road client scaffold; an implementation that fails them is a liability regardless of how clever its handler is.

## Prerequisites

- You have read chapters [01](../01-the-a2a-plane-and-what-it-owns.md), [02](../02-the-a2a-protocol-and-service-mesh-mapping.md), and [03](../03-inter-agent-routing-and-service-discovery.md).
- You have the [current A2A spec](https://a2a-protocol.org/) open in a tab. If the hosted spec and the [a2aproject repository](https://github.com/a2aproject) disagree on a detail, pin your implementation to one and note it in your README.
- You have a language of choice (Python / TypeScript / Go — the mod-412 defaults) with an HTTP server framework and a JSON-RPC library (or a 30-LOC hand-rolled JSON-RPC 2.0 handler).
- You have `curl` and `jq` for manual poking.

## Problem statement

Build a stub A2A server for a single agent — `greeter-v1` — that implements enough of the mandatory surface to be a real conformance target. "Enough" means:

- Serves an AgentCard at the spec-mandated well-known path.
- Implements `tasks/send` returning a task object with a legal state.
- Implements `tasks/get` returning the task by `taskId`.
- Implements `tasks/sendSubscribe` (or the current spec's equivalent streaming method) returning an SSE stream with at least one `working` update and a terminal `completed` update.
- Validates JSON-RPC 2.0 shape (version, method, id, params).
- Rejects an unauthenticated request with a legal error.
- Returns `task_id` idempotency on repeat `tasks/send` with the same `taskId`.

The agent itself does one skill: `greet(name) -> "Hello, <name>!"`. The content of the handler is incidental; the conformance targets the wire contract.

Then, author a conformance test runner that fires requests against the server and asserts against the spec. The runner is the artefact; the server is the target. Finally, produce the mapping table.

## Requirements

### Part 1 — The AgentCard

Serve `GET /.well-known/agent-card.json` (or whatever path the current spec mandates — document which revision you pinned) returning a valid AgentCard. Include:

- `name`, `description`, `url`, `version`.
- `capabilities.streaming = true`, `capabilities.pushNotifications = false`.
- `authentication.schemes = ["bearer"]`.
- `defaultInputModes = ["text"]`, `defaultOutputModes = ["text"]`.
- One skill with `id = "greet"`, inputs `text`, outputs `text`.
- A `platform` extension block (chapter 02): `tenancy`, `declared_scopes = ["greet:anyone"]`, `workload_id` (a SPIFFE-shaped string — you do not need SPIFFE to be live for this exercise), `resilience_tier = "T2"`, `topology_modes = ["manager-as-tools"]`.

<!-- needs-research: confirm the mandatory AgentCard fields against the current spec; some fields have moved between optional and required across revisions. -->

### Part 2 — `tasks/send` and the task lifecycle

Implement the synchronous `tasks/send` method. On receipt:

- Validate JSON-RPC 2.0 shape. Return the appropriate JSON-RPC error on malformed input.
- Validate the task has an `id` (client-generated) and a `sessionId`. Return a legal error on missing fields.
- Create or fetch the task record. Idempotency: if a task with the same `id` already exists, return it unchanged.
- Transition the task from `submitted` → `working` → `completed` and return the completed task in the response. Store the task in an in-memory map keyed by `(sessionId, id)`.
- Attach the output artifact — the greeting text — in the format the spec requires.

### Part 3 — `tasks/get` and `tasks/cancel`

Implement:

- `tasks/get(id)` returning the task by id, including its full state and artifacts.
- `tasks/cancel(id)` transitioning a `working` task to `canceled`. For this exercise the work completes in <100 ms, so cancel will rarely fire mid-work — that is fine; the handler must still accept the method and return a sensible response.

### Part 4 — Streaming

Implement the spec's streaming method (`tasks/sendSubscribe` or the current name). The server returns an SSE stream with:

- At least one `working` event with partial progress.
- A terminal `completed` event with the final artifact.
- Legal event framing per [the SSE standard](https://html.spec.whatwg.org/multipage/server-sent-events.html).

Pin your SSE framing in the README; different libraries format events slightly differently and the conformance test needs to know.

### Part 5 — Authentication and rejection

Implement a minimal bearer-token check: the server accepts `Authorization: Bearer <token>` and rejects without with a JSON-RPC error whose shape matches the spec's authentication-failure convention. You do not need to verify the token against a real JWKS for this exercise — a static expected string is enough — but the rejection path must be present.

Also implement a scope check: the token's `scope` claim (which you can stub into the token payload) must include `greet:anyone`. A token missing the scope is rejected with the spec's authorisation-failure error.

### Part 6 — The conformance test runner

Write a test runner (one file, 150–300 LOC) that fires requests against `http://localhost:8080` and asserts. Each assertion is one test; a failing test reports which assertion failed and what the response was.

Minimum test set (expand as you catch things while writing the server):

1. **AgentCard is reachable and well-formed.** `GET /.well-known/agent-card.json` returns 200 with the right content-type. The JSON parses. `name`, `url`, `version`, `capabilities`, `skills`, `authentication` are present.
2. **AgentCard declares the `greet` skill.** The `skills` array has at least one entry with `id="greet"`.
3. **AgentCard declares the platform extension.** `platform.tenancy`, `platform.declared_scopes`, `platform.workload_id`, `platform.resilience_tier` are present.
4. **`tasks/send` with valid input returns a task.** POST a well-formed JSON-RPC request; expect a 200 with a JSON-RPC response whose `result` is a task object in state `completed`.
5. **`tasks/send` idempotency.** POST the same request twice with the same `taskId`; expect identical responses and the task to appear only once in `tasks/get`.
6. **`tasks/send` with malformed JSON-RPC fails.** Missing `jsonrpc` version or `method` returns a JSON-RPC parse / invalid-request error.
7. **`tasks/send` with missing taskId fails.** Expect a JSON-RPC invalid-params error.
8. **`tasks/get` returns the task.** After a `tasks/send`, `tasks/get` by the same id returns the task.
9. **`tasks/get` for an unknown id returns an error.** A legal JSON-RPC error.
10. **`tasks/cancel` returns success on a terminal task.** The spec may define the exact semantics; document what yours does.
11. **Streaming emits at least one `working` and one terminal event.** Open the SSE stream; collect events; assert.
12. **Requests without `Authorization` are rejected.** 401 or JSON-RPC authentication error.
13. **Requests with a token missing `greet:anyone` scope are rejected.** JSON-RPC authorisation error.

The runner should exit non-zero on any failure, print a per-test summary, and be runnable via `make conformance` or equivalent.

### Part 7 — The conformance matrix

In the README, produce a Markdown table with four columns:

| A2A concept | Spec reference | Platform / mesh primitive | Enforcement layer |
| --- | --- | --- | --- |

Fill in at least:

- HTTPS transport
- JSON-RPC 2.0 request/response
- SSE streaming
- Push-notification webhook (even if your stub does not implement it)
- AgentCard well-known path
- `tasks/send` idempotency via `taskId`
- Bearer authentication
- Scope-based authorisation
- `tasks/cancel`
- Task state machine

For each row, name the mesh primitive (Envoy VirtualService, DestinationRule's circuit breaker, Istio AuthorizationPolicy, etc.) and the enforcement layer (mesh / scaffold / handler / registry). Chapter 02's mapping table is the model; your matrix should be at least as complete.

### Part 8 — The `A2ACallRecord` capture

Have the server log one `A2ACallRecord` (chapter 01's schema) per handled request to stdout or a file. You can stub fields you do not have (`caller.workload_id` can be a placeholder since you are not yet on a mesh). The point is that the shape is correct — later exercises add the real values.

### Part 9 — The README

One page covering:

- How to run the server (`docker compose up` or `python -m greeter`).
- How to run the conformance tests (`make conformance` or equivalent).
- The spec revision you pinned and any places you deviated.
- The conformance matrix (Part 7).
- One "I was surprised by" observation — e.g. "the spec says `tasks/send` is synchronous but allows the response to come after an arbitrary delay; I had to decide how long my test runner waits before failing."
- One open question for exercise 02 — e.g. "my bearer check is a static token; does the AgentCard's `authentication.schemes` field constrain how the mesh's AuthorizationPolicy is written?"

## Starter guidance

- **Write the AgentCard by hand first.** Serialise it to JSON; validate the shape by eye against the spec. The one-pager AgentCard is the whole contract — getting it right unlocks everything else.
- **Use a JSON-RPC 2.0 library, or write a 30-LOC handler.** Both are fine. The point of the exercise is the surface, not the plumbing.
- **Keep the task store in memory.** A `dict` keyed by `(sessionId, taskId)` is enough. Durability is mod-402's problem.
- **Tackle streaming last.** The synchronous methods are easier to debug; SSE framing is finicky. Get `tasks/send` + `tasks/get` green before touching streaming.
- **Fire one `curl` by hand for every method before writing the test runner.** The runner is easier to write when you have seen the response shape.
- **Diff your server against the spec one method at a time.** When a test fails, re-read the relevant spec section. Half of the value of this exercise is building confidence that you have read the spec.

## Acceptance criteria

Mark done when:

- [ ] The server is reachable, serves the AgentCard, and handles all four methods (`tasks/send`, `tasks/get`, `tasks/cancel`, streaming).
- [ ] `make conformance` (or equivalent) passes all 13 tests from Part 6.
- [ ] A deliberate break — e.g., temporarily remove the `taskId` idempotency check — causes the relevant test to fail. The runner catches regressions.
- [ ] The conformance matrix in the README is complete (no TBD cells).
- [ ] Every handled request emits an `A2ACallRecord` to stdout or a file.
- [ ] The README names the A2A spec revision pinned and the SSE framing convention used.

## Stretch goals

- **Implement push-notification webhooks.** Register a webhook URL via `tasks/pushNotification/set`; the server posts task-state transitions to it; the conformance runner stands up a stub webhook receiver and asserts delivery.
- **Add a second skill and test skill-level routing.** The `platform.declared_scopes` mapping becomes per-skill; the server rejects a call to a skill the token's scope does not cover.
- **Write the mesh YAML.** Even if you are not running a mesh locally, author the `VirtualService` + `DestinationRule` + `AuthorizationPolicy` CRDs that would route to your agent in Istio. Commit them. Exercise 03 uses the same shape.
- **Add a `state_transition_history` endpoint.** Some task recoveries depend on it; the spec's exact semantics are version-sensitive.
- **Fuzz the JSON-RPC parser.** Feed it malformed inputs (extra fields, wrong types, missing `jsonrpc` version) and assert the server returns legal JSON-RPC errors — not crashes, not HTML error pages. This is a conformance class the mesh does not help with.
- **Generate the AgentCard from code with a type-checked library.** A hand-authored JSON is fine for the exercise; a generated AgentCard is what teams do in production.
