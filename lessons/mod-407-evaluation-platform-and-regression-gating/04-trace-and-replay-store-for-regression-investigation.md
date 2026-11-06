# The Trace and Replay Store for Regression Investigation

A regression lands. A gate fails. An on-call pages. The first question is always the same: "What did the agent actually do, and what would it do now?" Every minute spent reconstructing the answer from app logs, provider dashboards, and chat screenshots is a minute the fleet is running with the regression unexplained.

This chapter is about the store that makes the answer a one-command query. It is the sibling data plane to the catalogue (chapter 03). The catalogue stores the *inputs* an eval runs against; the trace store stores the *interactions* an agent produces — and lets a reviewer replay any one of them against a candidate change and produce a comparable score.

## What a trace is

Pin the term. Different planes attach different meanings; this chapter uses the unified one.

A **trace** here is the full, hierarchical record of one agent session end-to-end: the user intent that opened it, every turn of the agent loop, every model call through the gateway (mod-406), every tool call through the MCP gateway (mod-403), every policy decision through the policy plane (mod-405), every safety-plane decision (mod-411), every token count, every latency. It is a superset of an OpenTelemetry trace — it carries the same `trace_id` but is a persisted, replayable artefact rather than a volatile observability signal.

The physical model is two layers:

- **Hot layer** — OpenTelemetry spans from mod-408, persisted to the observability backend (Jaeger / Tempo / Honeycomb / Datadog) for 7–30 days. Great for live debugging; not durable enough for regression investigation six months later.
- **Cold / replayable layer** — the trace store this chapter is about. A content-addressed, append-only record of every session, written through a durable sink the runtime (mod-402) and the gateway (mod-406) feed into.

The observability plane (mod-408) owns the hot layer as an operational signal. The eval plane owns the cold layer as a reproducibility artefact. The two share the `trace_id`; querying across them is a join, not a union.

## The three obligations of a replayable trace store

Three non-negotiable obligations. Everything else is a question of implementation.

1. **Every agent interaction a stream-aligned team cares about is captured end-to-end.** No silent drops under load, no "we sample 1%", no "the tool-call input is a string we truncated at 4kB". A trace that is 60% of the real interaction is 0% of a regression investigation. Sampling policy belongs to mod-408 for metrics, not here.
2. **Replay is a method call, not a project.** A reviewer with a `trace_id` can submit a `replay(trace_id, under_test=…)` request and receive an `EvalRecord` back. Replay is the primary read path, not an afterthought.
3. **Retention, redaction, and tenant isolation are first-class.** The store carries PII by construction; the regulated-financial agents' traces are not available to a public-benchmark harness; a tenant's traces are not readable across tenancy boundaries.

A store that cannot defend all three is an observability dashboard, not a replay store.

## The session schema

A session is the unit the store addresses. One row per session at the top level; child rows for turns, model calls, tool calls, policy decisions.

```json
{
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "session_id": "s_01HV8X3M9F5A6R2B4C0D7E8G9H",
  "tenancy": {
    "tenant_id": "acme",
    "principal_subject": "alice@acme.com",
    "runtime_spiffe_id": "spiffe://platform.local/runtime/prod",
    "pii_class_observed": "pii"
  },
  "under_test": {
    "agent_id": "writer-v1",
    "agent_revision": "git:abcd1234",
    "prompt_hash": "sha256:9ce4...",
    "tool_schema_hash": "sha256:1aa2..."
  },
  "started_at": "2026-11-03T14:21:58.004Z",
  "ended_at":   "2026-11-03T14:22:11.721Z",
  "input": {
    "kind": "user_message",
    "text_ref": "s3://traces/4bf9.../input.jsonl#row=0"
  },
  "final_output": {
    "kind": "assistant_message",
    "text_ref": "s3://traces/4bf9.../output.jsonl#row=-1"
  },
  "turn_count": 7,
  "model_call_count": 7,
  "tool_call_count": 3,
  "policy_decision_count": 3,
  "total_cost_usd": 0.0164,
  "outcome": "completed",
  "children_ref": "s3://traces/4bf9.../turns.jsonl"
}
```

A turn:

```json
{
  "trace_id": "4bf92f...",
  "turn_id": "t_002",
  "parent_turn": "t_001",
  "role": "assistant",
  "model_call_ref": "s3://traces/4bf9.../model-calls/mc_002.json",
  "tool_calls_ref": "s3://traces/4bf9.../tool-calls/turn_002/",
  "policy_decisions_ref": "s3://traces/4bf9.../policy/turn_002.jsonl",
  "started_at": "2026-11-03T14:21:59.102Z",
  "ended_at":   "2026-11-03T14:22:01.414Z"
}
```

A model call (the shape from mod-406's envelope):

```json
{
  "trace_id": "4bf92f...",
  "turn_id": "t_002",
  "gateway_request_id": "gw_01HV...",
  "routing": { "requested_model": "platform:writer-v1", "resolved_provider": "anthropic", "resolved_model": "claude-sonnet-4-6", "attempts": [...] },
  "tokens": { "input_cached": 4712, "input_non_cached": 318, "output": 442 },
  "messages_ref": "s3://traces/4bf9.../model-calls/mc_002.messages.jsonl",
  "cost": { "total_usd": 0.0164, "price_table_version": "2026-11-01" }
}
```

Three invariants:

1. **Everything large is a reference, not an inline blob.** The top-level session row is tens of kilobytes; the content lives in object storage, keyed by content hash. The index can hold a billion session rows; the content is cheap.
2. **Every cross-plane id travels.** `trace_id` from the observability plane, `session_id` from the runtime, `gateway_request_id` from the model gateway. A reviewer can jump sideways into mod-408 dashboards or mod-406 billing events on any of them.
3. **The under-test fingerprint is on the session.** Agent revision, prompt hash, tool-schema hash. A replay for regression investigation requires knowing *what* was running; the fingerprint is how.

## What the store does *not* carry

Scope is a feature here. The store is not:

- **A metrics store.** p99 latency per agent is mod-408's job, not this plane's.
- **A chat-history UI backing store.** Product-side chat UIs use their own persistence; a copy for replay lives here, redacted.
- **A general log aggregator.** Only agent sessions. Application logs that are not part of an agent loop belong elsewhere.
- **A long-tail billing source.** The billing event stream (mod-406 chapter 04) is authoritative; the store carries cost on sessions for correlation only.

Confusing these roles is where storage bills explode.

## Capture — where traces enter the store

Three producers write to the store. Each uses the same sink interface.

- **The runtime (mod-402).** The runtime owns the agent loop and emits a session envelope: session open, each turn start / end, final outcome. It writes session-level metadata and turn-level pointers.
- **The model gateway (mod-406).** On every call, it writes the full model-call record — the chapter-06 envelope — to the model-calls partition.
- **The MCP gateway (mod-403) and the sandbox (mod-404).** On every tool call, it writes the tool-call record — inputs, outputs, policy decisions, resource metrics — to the tool-calls partition.

The sink is OpenTelemetry-compatible for the metadata (so mod-408 can subscribe on the same stream) but writes the full content blobs to object storage, not through OTLP. OTLP is not designed for 50-MB tool outputs.

Two sink shapes dominate:

- **In-process writer plus async batch.** Each producer buffers locally, flushes to the sink service on a timer or at a size threshold. The sink writes to object storage and emits a manifest row to the index. Easy to operate, slight risk of loss on producer crash.
- **Pass-through via the gateway.** Every call already goes through mod-406; mod-406 forks a copy to the store sink. No producer-local buffering. Marginally more latency on the model-call path; much simpler loss story.

Most mature platforms land on the hybrid: gateway pass-through for model calls, in-process writers for runtime and MCP gateway / sandbox, all draining into the same sink.

## Redaction at capture

The store carries PII by construction — real user prompts, real assistant responses, real tool outputs. Redaction policy has four knobs:

- **Pre-store redaction** — the model gateway's egress redactors (mod-406 chapter 06) run over outbound text; the redacted version is what the store sees. Keeps the store clean of low-value PII that would never need to be seen for debugging.
- **At-rest encryption with a tenant-scoped key.** The object store encrypts each session's content blobs with a tenant-scoped KMS key (mod-405 chapter 04's key management plane). A tenant-key revocation invalidates all that tenant's trace content in one call.
- **Post-store redaction on read** — the store's read path can re-run detectors against stored content and surface a redacted view, for example when a trace is being prepared as a catalogue row (chapter 03). The original stays intact in the store; the redacted view is the one the catalogue receives.
- **Hard-class refusal.** `pii_class: secret` content (API keys, PEM blocks) is never persisted to the store. If an egress detector catches one at capture time, the store writes a sentinel row (`content_refused_on_capture`) and discards the content. A session cannot be reconstructed with the secret; a reviewer sees the shape of the session with the body absent.

The redaction model inherits the detectors from mod-406 chapter 06 and the audit stream from mod-405. The eval plane does not author detectors; it runs them against capture.

## Retention

Four retention tiers; wire them to the row's attributes, not to a global default.

- **90 days hot, 7 days of full search.** Mod-408 handles this; traces here reference the hot layer.
- **13 months full-fidelity in the store.** Regression investigations routinely reach back a year. Anything shorter leaks regression narrative.
- **5 years metadata-only.** Session index rows (sans content) for compliance questions "did the agent do X on Y date". Content blobs are purged at 13 months; metadata rows stay.
- **Per-tenant overrides.** A regulated-financial tenant may require longer or shorter content retention; a `pii_class: phi` session inherits a shorter retention automatically.

Retention changes are reviewed; a shortening on a fixture referenced by an active CI gate is a gate-failing event (chapter 05).

## Replay — the primary read path

Replay is the API a reviewer runs when a gate failure needs explaining. Three replay modes.

### 1. Exact replay

Take a persisted session, re-run the model calls and tool calls against the *original* model, prompt, and tool-schema. Outcome should be byte-identical for deterministic providers, within a provider-specified variance for the rest.

Exact replay is mostly a debugging and smoke-test primitive. It answers "did my capture fidelity regress?" It does not answer "is my candidate better?"

### 2. Under-test-swap replay

Take a persisted session's *inputs* — user message, tool outputs, system prompt — and re-run the agent loop against a candidate model, prompt, or tool-schema. The agent's responses and new tool calls diverge from the original; the replay engine decides what to do with tool calls that no longer appear in the recorded trace.

Three policies dominate for new tool calls:

- **Record-and-replay only** — if the candidate's tool call does not match a recorded one, the replay fails the session as `divergence_outside_replay`. Tight, but useful when the candidate's tool shape is supposed to be identical.
- **Live-dispatch** — the replay dispatches the tool call live against a sandbox (mod-404) under test. Realistic, but expensive and non-deterministic.
- **Simulator-dispatch** — a per-tool simulator (built from the aggregate of recorded tool outputs) answers the candidate's tool call. Cheaper than live; introduces its own bias.

The template library ships record-and-replay and simulator-dispatch; live-dispatch is a per-team opt-in. The replay engine records which policy fired and emits it on the `EvalRecord`.

### 3. Judge-swap replay

Take the recorded under-test outputs as-is and re-score them with a new judge. Cheap, useful when a judge rubric is being revised and the authoring `ai-eval-engineer` wants to see how many verdicts move. The under-test model is not re-called; only the judge is.

Judge-swap replay is the most common replay mode for release-blocking fixtures. It is what chapter 05's gate calls under the hood when a judge version changes.

## Trace-to-catalogue promotion

A replayable trace is the raw material for a fixture (chapter 03). Promotion has four steps:

1. **A reviewer flags a session as fixture-worthy.** The leaderboard or a chat command in on-call tooling sends the trace id to the catalogue sink.
2. **Redaction pass** — the trace is re-redacted with the stricter "catalogue egress" detector profile. The original stays in the store; the catalogue only sees the redacted copy.
3. **Shape normalisation** — the session is transformed into the catalogue's row schema (chapter 03). The `input` column is the user message, the `reference` column is either the original final output (if the session succeeded) or the human-authored correction (if the session was a failure being added as a regression case). The `tool_outputs` columns are the slice of tool outputs that matter for replay.
4. **Publish** — the catalogue publishes a new row (or new version) with `provenance.kind = replayed:<session-id>`, carrying the original tenant and licence, and routing through the publish review workflow.

The replay-contract version is stamped onto the row so that future replays reproduce the same inputs under the same semantics. If the replay semantics change (live-dispatch becomes the default, say), the catalogue manifest notes the breaking change and the gate (chapter 05) re-baselines.

## The schema — hot index and cold storage

A practical implementation uses two stores.

### Hot index (search)

A columnar store — ClickHouse, BigQuery, or Iceberg-on-object-storage with a DuckDB or Trino front-end — holding:

- `session_sessions` — one row per session with all the metadata fields and references to content blobs.
- `session_turns` — one row per turn.
- `session_model_calls` — one row per model call, duplicating a subset of the gateway billing-event stream for correlation.
- `session_tool_calls` — one row per tool call with tool id, arguments hash, output hash, latency, resource counts.
- `session_policy_decisions` — one row per policy decision.

A reviewer runs:

```sql
SELECT trace_id, started_at, total_cost_usd
FROM session_sessions
WHERE tenancy.tenant_id = 'acme'
  AND under_test.agent_id = 'writer-v1'
  AND outcome = 'tool_loop_exhausted'
  AND started_at BETWEEN '2026-11-01' AND '2026-11-03'
ORDER BY total_cost_usd DESC
LIMIT 20;
```

and gets back the worst twenty cost-exhausted writer-v1 sessions in a window, followed by a one-click into replay.

### Cold storage (content)

Object storage, content-addressed. One prefix per session, one object per message / tool-call input / tool-call output / policy decision body. Parquet for the structured tables (turns, model calls, tool calls, policy decisions) when batch-published; JSONL for append-friendly writes.

Partitioning by date (`YYYY/MM/DD/<session-id>/`) is defensible for query-pushdown and retention. Partitioning by tenant pays off for cross-tenant blast-radius and for KMS key management.

## What the store does for the eval plane specifically

Three uses beyond regression investigation:

- **Replay-based fixture authoring.** The catalogue's `replayed:...` provenance kind is only possible because the store can serve the replay. Chapter 03's trace-derived datasets flow through here.
- **Online-eval sampling.** `ai-eval-engineer`'s online eval methodology (chapter 06 handoff) samples live production traces, routes them through a judge, and emits an `EvalRecord`. The store is what the sampler reads.
- **Trace-slice contracts for judges.** A judge does not need the whole trace; it needs the final answer and maybe the last two turns. The store serves a `slice(trace_id, slice_contract)` endpoint so judge cost stays bounded (chapter 02).

## Summary

- The trace / replay store is a persisted, content-addressed record of agent sessions. It is the cold-layer sibling to mod-408's hot observability layer. Traces carry a `trace_id`, `session_id`, `gateway_request_id`, and the under-test fingerprint (agent revision, prompt hash, tool-schema hash).
- Three obligations: end-to-end capture of every session a team cares about; replay as a first-class API; retention, redaction, and tenant isolation enforced by construction.
- Capture is through a shared sink the runtime, the model gateway, and the MCP gateway / sandbox all write into. Redaction runs at egress on the model gateway so the store never sees avoidable PII; at-rest encryption is tenant-key scoped; hard-class content is refused at capture.
- Replay has three shapes — exact, under-test-swap (record-and-replay / live-dispatch / simulator-dispatch), and judge-swap. The replay engine records which policy fired on the `EvalRecord`.
- Traces become catalogue rows through the trace-to-catalogue promotion path; the replay-contract version is stamped onto the row so future runs are reproducible.
- The store is two layers physically — a hot columnar index for search and a cold object-storage store for content blobs — with cross-plane ids on every row for jump-into-mod-408 and jump-into-mod-406-billing.

The next chapter opens the gate — the CI/CD surface that reads from the catalogue, kicks off replays through the template library, and refuses a merge when a candidate change regresses a fixture with statistical confidence.
