# Conformance, Health-Checking, and Failover Routing

The registry says a server speaks MCP 2025-06-18, advertises three tools with these schemas, and requires these scopes. Conformance is the test that says it is telling the truth. Health is the signal that says it is still telling the truth. Failover is what the gateway does when the signal says otherwise.

The baseline tool everyone compares against is the **MCP Inspector** — a debugging client that runs the handshake, lists primitives, calls tools, and reports what it sees. Your production conformance pipeline is "Inspector-shaped": the same kinds of probes, run in CI and in production, emitting results to the registry's health API.

## The three layers of conformance

Draw a line between three questions, because they fail in different ways and cost different amounts to check:

1. **Protocol conformance.** Does the server speak MCP at the wire-and-semantics level the spec requires? Does it accept `initialize`, return its capabilities, respond to `tools/list` with a valid envelope, return `tools/call` results in the right shape, close the session cleanly? Checked cheaply, every time; this is the Inspector layer.
2. **Schema conformance.** Does the schema the server advertises at `tools/list` match the schema it actually validates against on `tools/call`? A server can lie here without noticing: it might accept extra fields the schema forbids, or reject fields the schema allows. Checked by generating valid-and-invalid payloads from the schema and comparing responses.
3. **Semantic conformance.** Given a known input, does the server produce the expected output? A calendar server that returns an empty list for any query is "conformant" at layers 1 and 2 but broken at layer 3. Checked with a small, versioned **golden-fixture** set stored in the registry entry.

Your pipeline runs all three, at different cadences, with different thresholds.

## Protocol conformance — the Inspector-shaped core

Treat these as the non-negotiable probes. For a remote (Streamable HTTP) MCP server:

- **Reachability.** TLS termination, handshake completes within a sane TTL.
- **Initialize handshake.** `initialize` request with current protocol version returns a well-formed response: the server's version, capability flags (`tools`, `resources`, `prompts`, each with `listChanged` flag), server info (`name`, `version`).
- **Protocol version negotiation.** If the probe pretends to speak an older version, the server either accepts or returns a clear error — it does not silently use a different shape.
- **`tools/list`.** Returns a valid envelope; each tool has `name`, `description`, `inputSchema`; inputSchema parses as JSON Schema; names are unique within the response.
- **`resources/list`, `prompts/list`.** Analogous; required iff the server advertised the capability in `initialize`.
- **`tools/call` error paths.** Call with a non-existent tool name → structured error (`-32602` or MCP error code), not a crash. Call with invalid input against the declared schema → structured error with a validation message.
- **Notifications.** If the server advertises `listChanged`, it can send `notifications/tools/list_changed` and the probe observes it.
- **Idle behaviour.** Open a connection, do nothing, verify the server does not kill the session prematurely or leak memory (long-tail probes).
- **Shutdown.** Clean termination.

For a stdio server, the same probes run with the server as a subprocess; `stderr` is captured and inspected for crashes.

These probes are the ones the **MCP Inspector** runs interactively. Your pipeline codifies them in CI and in production as a conformance suite the registry ships.

## Schema conformance — generating payloads from the schema

A server that says `inputSchema = { type: "object", required: ["query"], properties: { query: { type: "string" } }, additionalProperties: false }` has made four testable promises:

1. Calls with no `query` are rejected.
2. Calls with a non-string `query` are rejected.
3. Calls with extra fields (e.g. `{query: "x", extra: 1}`) are rejected.
4. Calls with `{query: "x"}` are accepted (the schema is satisfiable).

Generate payloads automatically from the JSON Schema (property-based testing tools like Python's `hypothesis-jsonschema`, Node's `json-schema-faker`, or Rust's `schemathesis` all cover this), send them, and compare the server's response class against the schema's claim. Any mismatch is a failing probe — either the schema is wrong or the server is.

Two gotchas:

- **The MCP spec treats `inputSchema` as the input contract; there is no normative `outputSchema` in all tool declarations.** Where the server does ship an output schema (a convention adopted in some tool definitions), run the same probe against observed outputs. Where it does not, the schema-conformance probe only covers inputs.
- **Idempotent-looking probes can do work.** If a tool actually hits a downstream API, sending it synthetic payloads in production is a liability. Isolate schema probes to a staging instance, or ensure the server has a "`dry_run: true`" convention your gateway can set. The admission ladder (chapter 03) runs schema conformance against staging; the production probe degrades to protocol-only for non-idempotent tools.

## Semantic conformance — the golden-fixture layer

The registry entry carries a small, versioned block of **golden fixtures** per tool: `{inputs, expected_outputs_or_properties}`. The semantic probe calls each tool with each input and checks the response against the expected shape. Three realistic patterns:

- **Exact match.** For deterministic tools (a calculator, a hash function), the output is fully predictable.
- **Property match.** For near-deterministic tools (a geocoder, a currency converter), the output has invariants (coordinates within a region, exchange rate within a tolerance).
- **Shape match.** For non-deterministic tools (a web search, an LLM-wrapped service), only the response shape is checkable (array of length ≥ 1, objects with the right keys).

Semantic probes run on a slower cadence (daily, not per-minute). They are expensive and they hit real downstream systems; be deliberate. Where a semantic probe fails, it indicates behavioural drift that schema conformance cannot see — and that is precisely the drift that silently breaks agents.

## Probe cadence and placement

Not every probe runs at every frequency. A reasonable default:

| Probe | Where | Cadence | Failure consequence |
|-------|-------|---------|---------------------|
| Reachability | Production + staging | Every 10–30 seconds (liveness) | Trip circuit breaker, prefer failover |
| Initialize + `tools/list` | Production | Every minute (readiness) | Mark server degraded; discovery hides it from new sessions |
| Schema conformance (idempotent paths) | Staging (always), Production (sample) | Hourly | Fail the version; admission reverts |
| Schema conformance (full) | Staging only | On admission + weekly | Block admission |
| Semantic conformance | Staging (always), Production (sample) | Daily | Open an incident; mark version "behavioural drift suspected" |

The staging instance exists in part so your pipeline can run probes that are unsafe in production. The registry entry carries both the staging and production endpoints; admission promotes staging → production only after the staging probes pass.

## Health signals and the registry's health API

Every probe writes a **HealthEvent** to the registry's health API:

```
HealthEvent {
  server_id       string
  version         semver
  probe_name      string            # e.g. "initialize_handshake", "schema_conformance:search_calendar"
  outcome         enum<pass, fail, degraded>
  details         JSON              # response, error, diff, latency
  observed_at     timestamp
  observer_id     string            # which probe fleet instance wrote this
}
```

The gateway subscribes (via watch, poll, or event bus) and maintains a derived **HealthState** per `(server_id, version)`:

- `up` — recent probes all pass.
- `degraded` — some probes fail but the server is still useful (e.g. latency is high but semantics are intact).
- `down` — critical probes fail; the gateway should not route new calls here.
- `unknown` — no recent observations (stale probe fleet, new deployment).

**Timeouts matter.** A probe that times out is a `fail` for liveness probes and an `unknown` for deep semantic probes (because the slow response may still be in flight). The gateway's rules on `unknown` default to the more cautious interpretation when risk tier is high.

## Failover strategies

Three primary strategies, selected per server in the registry's `RoutingPolicy` (chapter 03):

### Active-passive

One **primary** endpoint; one or more **fallbacks** in order. On a live `down` signal from the primary, or on a per-call timeout, the gateway tries the next fallback. Suitable when a managed-vendor alternative exists for the same tool (chapter 06 build-vs-buy) and the vendor's behavioural envelope is known.

Rules:

- **Idempotent tools only can retry on fallback after a timeout.** For non-idempotent calls, a timeout is a terminal error; otherwise the gateway might cause a duplicate side effect.
- **Fallbacks inherit the schema pin.** The fallback endpoint must serve the pinned tool schema; if its schema diverges, it is not a legal fallback — admission catches this.

### Degraded mode

A server declares what it can still do when partially broken:

- **`read_only`** — if write-type tools are failing, keep serving read-type tools.
- **`best_effort`** — serve what responds within the timeout; return structured errors for the rest.
- **`reject`** — all-or-nothing; if a critical probe fails, remove entirely.

Degraded mode is useful when a server has internal partial failure (e.g. a database write replica is down but reads are fine). It needs the server author's cooperation — the registry entry declares the degraded shape; the probes check whether it is actually deliverable.

### Circuit breaker

Per-endpoint rolling-window state machine: `closed` (normal), `open` (consecutive failures over threshold → short-circuit all calls for cool-down period), `half_open` (after cool-down, try one call; success → closed, failure → open again). Tuned per tool risk tier:

- **T0/T1** (low risk): looser thresholds (5 failures in 60s → open for 30s). Tolerate transient flakes.
- **T2/T3** (higher risk): tighter thresholds (2 failures in 10s → open for 5 minutes, requires manual half-open). Fail safer.

Circuit breakers compose with active-passive: an opened primary immediately diverts to fallback; a half-open test determines when to divert back. Standard library implementations (Resilience4j, Polly, Hystrix-style, `opossum`, cloud service mesh defaults) are fine — write the wiring, do not reinvent the state machine.

## The staleness problem — do not cache a dead server

An agent's `tools/list` is often cached by the gateway for the session. If a server goes `down` after the cache fills, the gateway must not continue to advertise it. Three rules:

1. **Session pins survive but calls fail cleanly.** The pin to `calendar-google@2.3.1` persists on the session; calls to a `down` instance return a `tool_unavailable` error (distinct from `tool_sunset` and `tool_not_found`) so the agent can decide what to do (retry later, ask the user, abandon).
2. **New sessions see current reality.** Discovery responses for new sessions exclude `down` servers. This is where a short cache TTL (10–30s) earns its keep.
3. **Notifications propagate.** If the gateway runs `notifications/tools/list_changed` on behalf of the upstream (or if the upstream itself supports it), subscribed sessions get a signal to refresh. The SDK reconciles.

## What the SRE plane gets from this chapter

The health/conformance pipeline produces the SLIs the SRE plane (mod-410) consumes:

- **Conformance-pass rate**: percentage of probes passing, aggregated per server and per probe type. The SLO is defined per risk tier.
- **Time-to-admit new server**: from submission to GA. A platform SLO on this is what makes teams feel the paved road is fast enough.
- **Failover latency**: from a `down` signal to all gateway instances routing to the fallback. Pick a target (seconds, not minutes).
- **False-positive rate on health probes**: a flappy probe kills your admission velocity. Treat flakiness in the probe suite itself as an incident.

## What the observability plane expects (preview of mod-408)

Every probe emits OTel spans with the GenAI semantic conventions where applicable:

- `gen_ai.tool.name`, `gen_ai.tool.call.id`, `server.address`, `server.port`, `rpc.system=jsonrpc`, `rpc.method=tools/call`.
- Probe-specific attributes: `probe.name`, `probe.outcome`, `probe.observed_at`.
- The health API writes a *span link* onto the server entry's history, so investigators can pivot from "when did this server start failing" to the exact span batch that first noticed.

Chapter 08 of mod-408 picks this up end-to-end; what matters here is that the health signals are already OTel-shaped from this chapter forward.

## Summary

- Conformance is three layers — protocol, schema, semantic — each failing differently and costing differently to check. Your pipeline runs all three on cadences that match the cost.
- Probes write HealthEvents; the gateway derives HealthState; routing decisions read HealthState.
- Failover has three strategies (active-passive, degraded mode, circuit breaker) composed per server in the registry's RoutingPolicy. Idempotency decides whether a timeout is retryable.
- Caching: session pins survive, new sessions see current reality, notifications propagate. Never cache a dead server across session boundaries.
- The pipeline produces the SLIs the SRE plane (mod-410) consumes and emits OTel spans in the shape mod-408 standardises on.

The next chapter addresses the servers that do not fit the chapter-02 model — the ones that do not speak OAuth at all. For those, the gateway runs an auth-wrapping proxy.
