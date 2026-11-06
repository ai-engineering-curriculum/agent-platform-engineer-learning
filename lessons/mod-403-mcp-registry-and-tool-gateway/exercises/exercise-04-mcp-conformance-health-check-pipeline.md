# exercise-04: MCP Conformance and Health-Check Pipeline

**Estimated effort:** 2 hours · **Output:** one probe-suite spec + one failover policy per server tier + one SLI/SLO proposal · **Pairs with:** chapter [04](../04-conformance-health-and-failover.md); feeds projects 401 and 403, hands SLIs into mod-410.

## Objective

Design the pipeline that keeps the registry's claims honest. For each server the registry admits, the gateway's conformance and health infrastructure must continuously verify: does this server still speak MCP correctly, does its declared schema still match its behaviour, does it meet its golden fixtures, and what does the gateway do when it does not?

Chapter 04 laid out the three conformance layers (protocol, schema, semantic), the probe cadences, the health signals, and the three failover strategies. This exercise is the concrete spec.

## Prerequisites

- You have read chapter [04](../04-conformance-health-and-failover.md) in full.
- You have exercise 01's `ServerEntry` schema (specifically, the fields `conformance_ref`, `RoutingPolicy`, `SLO targets`) available. Your probe records write to that shape.
- You have looked at the **MCP Inspector** source and its test flows (<https://github.com/modelcontextprotocol/inspector>). The Inspector is both a debugging tool and the de facto reference for what a reasonable protocol-conformance probe looks like.
- You are familiar, at least superficially, with a property-based testing or schema-fuzzing library. Python's `hypothesis-jsonschema`, Node's `json-schema-faker`, Rust's `schemathesis`, or Go's equivalents are all fine.

## Problem statement

Design a pipeline that:

- Runs protocol conformance probes continuously in production and staging.
- Runs schema conformance probes against admission and periodically.
- Runs semantic conformance probes on a daily cadence.
- Writes HealthEvents to the registry's health API.
- Drives the gateway's RoutingPolicy decisions (failover, circuit break, degraded mode).
- Emits OTel spans in the shape mod-408 will consume.
- Produces SLIs the SRE plane (mod-410) can set SLOs against.

The deliverable is three parts.

## Requirements

### Part 1 — Probe-suite specification

Specify the three probe layers as a concrete test spec. For each probe, write:

- **Name.** A stable identifier (`probe_name` field in the HealthEvent).
- **Layer.** Protocol / schema / semantic.
- **Trigger.** Admission-gate / production-liveness / production-readiness / scheduled.
- **Cadence.** E.g. every 30 seconds, hourly, daily.
- **Target.** Staging-only, production-only, or both.
- **Input.** What the probe sends (`initialize`, `tools/list`, a generated payload, a golden fixture).
- **Assertions.** What the probe checks on the response.
- **Timeout.** Per-probe timeout before `fail` or `unknown` outcome.
- **Side effects.** Whether the probe causes any downstream effect (reads a record, writes a record, hits a vendor API). Side-effecting probes are staging-only.

At minimum, define:

**Protocol layer (6 probes).**

1. `reachability` — TLS handshake and HTTP 2xx on the base endpoint.
2. `initialize_handshake` — Full `initialize` → `initialized` round-trip; capabilities parsed.
3. `protocol_version_negotiation` — Verifies the server returns a protocol version the client understands.
4. `tools_list_shape` — `tools/list` returns a valid envelope; inputSchemas parse as JSON Schema; names are unique.
5. `error_shape_invalid_tool` — `tools/call` with a non-existent tool name returns a structured error (not a 500, not a crash).
6. `shutdown_clean` — Explicit close for stdio, connection teardown for HTTP; no lingering resources.

**Schema layer (3 probes, written generically over all tools).**

7. `schema_required_fields` — Call each tool missing one required input field; expect a structured validation error.
8. `schema_type_violations` — Call each tool with a field of the wrong type; expect a structured validation error.
9. `schema_additional_properties` — Call each tool with an extra field (if the schema has `additionalProperties: false`); expect a validation error.

**Semantic layer (1 probe per tool, driven by registry fixtures).**

10. `golden_fixture:<tool_name>:<case_name>` — Call the tool with a known input from the registry's fixture block; compare response against the expected (`exact_match`, `property_match`, `shape_match` mode per the chapter 04 taxonomy).

For each, write the assertion in prose precise enough to implement. "`error_shape_invalid_tool` — response is a JSON-RPC error with `code=-32602` *or* an MCP tool error with `isError: true` and a message field; neither a 200 success nor a transport-level 500" is a spec; "the response is reasonable" is not.

### Part 2 — Failover policy per server tier

Chapter 03 defined risk tiers T0 (lowest) through T3 (highest). Chapter 04 defined three failover strategies. Author the default `RoutingPolicy` for each tier — a template the registry uses when an owner does not override.

For each tier, specify:

- **Circuit breaker thresholds** (failure count, window, cool-down).
- **Default degraded mode** (`reject`, `read_only`, `best_effort`).
- **Retry policy** (idempotent-only, with max retries and backoff).
- **Timeout** (per-call, before falling over).
- **Fallback behaviour** (named explicit fallback, graceful degradation, or none).
- **Probe-driven routing** (how fast a `fail` signal propagates to the gateway's routing layer: seconds for T3, longer tolerance acceptable for T0).

Format as a table or a YAML block; the registry entry format from exercise 01 should accept it.

Justify, in a one-paragraph note per tier, why the thresholds are what they are. "Tight for T3 because the cost of a bad call is high" is the shape; be specific about which bad call.

### Part 3 — SLI/SLO proposal

Chapter 04 and chapter 03 together name five SLIs this pipeline produces. Pick each, and for each propose:

- The precise measurement definition (which events over which window, with which filter).
- A proposed SLO value — an objective, not an aspiration. Pick something you would defend in an SRE review in mod-410.
- An alerting cadence (what breach latency triggers a page vs. a ticket).

The five SLIs:

1. **Conformance-pass rate.** Percentage of probes passing, aggregated per server and probe-layer, rolling 24 hours.
2. **Time-to-admit new server.** From admission submission to GA promotion, 50th and 95th percentiles.
3. **p99 discovery latency.** The gateway's `tools/list` response time at p99.
4. **Failover latency.** From a confirmed `down` HealthEvent to all gateway replicas routing to the fallback.
5. **False-positive rate on health probes.** HealthEvents that mark `fail` but were caused by probe-side issues (network to probe, probe itself crashing), per probe, per week.

For each, write the proposal as a self-contained block the mod-410 SLO definition work can pick up without clarification.

## Starter guidance

- The MCP Inspector is the reference client; a probe suite that cannot reproduce Inspector's green/red output for a known-good server is not working yet. Use Inspector locally against a staging server as the smoke-check for your probe design.
- Property-based schema probes are easy to over-engineer. Start with the three in the required set; add more only when you have evidence a real-world server would benefit.
- Not every probe belongs in production. Semantic probes that actually cost vendor-API call volume are scheduled and staging-biased. Say which probes you *refuse* to run in production, not just which ones you include.
- SLO numbers are a negotiation with the SRE plane (mod-410). Propose them here; do not mistake the proposal for a committed SLO.
- OTel attribute shape: chapter 04 and mod-408 specify `rpc.system=jsonrpc`, `rpc.method=tools/call`, `gen_ai.tool.name`, plus your probe-specific attributes (`probe.name`, `probe.outcome`). Include these in your spec.

## Acceptance criteria

Mark done when:

- [ ] The probe suite has at least the ten required probes, each with name / layer / trigger / cadence / target / input / assertions / timeout / side effects.
- [ ] The schema-conformance probes name a concrete property-based testing library and give the generation strategy (e.g. "generate 10 inputs per tool per probe, seeded for reproducibility").
- [ ] The semantic probes make it clear which tools get an `exact_match`, which `property_match`, and which `shape_match`, with examples.
- [ ] Each of T0–T3 has a complete default `RoutingPolicy` with justification.
- [ ] The five SLIs each have a precise measurement definition, a proposed SLO value, and an alerting cadence.
- [ ] The probe runner's HealthEvent shape conforms to chapter 04 (`server_id`, `version`, `probe_name`, `outcome`, `details`, `observed_at`, `observer_id`).
- [ ] The spec documents which probes are staging-only versus production-safe and why.
- [ ] You have flagged any unverified MCP-spec detail or Inspector detail with `<!-- needs-research: ... -->`.

## Stretch goals

- **CI gate for admission.** Write the GitHub Actions / GitLab CI / Buildkite step that runs the probe suite against a staging server on an admission PR. The step should fail the PR if any protocol or schema probe fails, and warn (not fail) if a semantic probe fails on a new fixture.
- **Probe observability.** Sketch the dashboard the SRE plane uses to watch the five SLIs — which panels, which aggregations, which alert thresholds. Preview for mod-408 and mod-410.
- **Post-incident hook.** When an incident reveals a behavioural drift that the semantic probes did not catch, how do you add a new golden fixture? Spec the flow from incident response to a merged registry entry change.
- **Vendor reality check.** Pick one tier-2 (vendor-fronted) tool from exercise 01's worked entries; walk through how its probes differ from a self-hosted tool. Vendors often disallow synthetic traffic; what does your pipeline do differently?
- **Chaos drill plan.** Write a one-paragraph chaos drill that tests the failover from a working primary to a standby — what you inject, what you expect, how you measure (feeds mod-410).
