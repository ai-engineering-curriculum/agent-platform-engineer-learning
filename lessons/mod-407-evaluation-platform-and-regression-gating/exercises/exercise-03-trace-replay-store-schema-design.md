# exercise-03: Trace / Replay Store Schema Design

**Estimated effort:** 3 hours · **Output:** one schema file (JSON Schema + DDL for the hot index), one sink implementation, one replay engine covering the three replay modes, a worked regression-investigation walkthrough against a seeded dataset, and a short README (~1 page) · **Pairs with:** chapters [04](../04-trace-and-replay-store-for-regression-investigation.md) and [01](../01-the-evaluation-plane-and-what-it-owns.md); feeds exercise 04 and project 403.

## Objective

Design the trace / replay store schema and build the minimum implementation that supports regression investigation. A reviewer with a `trace_id` from a failing eval must be able to replay the session, swap the under-test model or judge, produce a comparable `EvalRecord`, and land on the full session detail from a one-command query against your hot index.

A well-authored store produces chapter-04's session / turn / model-call / tool-call records faithfully, preserves every hash chapter 01 demands, and keeps the three replay modes distinct and labeled on the output record. A store that inlines full tool-output blobs into the index table, or that silently loses the under-test fingerprint on replay, is a store a reviewer sends back.

## Prerequisites

- You have read chapters [01](../01-the-evaluation-plane-and-what-it-owns.md), [04](../04-trace-and-replay-store-for-regression-investigation.md), and [03](../03-the-eval-dataset-management-plane.md).
- You have a local object store surrogate (a directory on disk is sufficient; MinIO, LocalStack, or a real cloud bucket if you prefer) and a local columnar store (DuckDB over Parquet is the recommended minimum; ClickHouse / Postgres work).
- If you did exercise 01 or 02, you have a few real traces to seed the store with; otherwise this exercise ships a seed-generation script in the starter guidance below.
- You have the [OpenTelemetry Semantic Conventions for Generative AI](https://opentelemetry.io/docs/specs/semconv/gen-ai/) open while you wire cross-plane ids.

## Problem statement

Design the trace store for one agent — pick a shape and keep it:

- **`writer-v1` writer agent.** Multi-turn, with retrieval tool calls. Sessions typically 4–12 turns, 1–4 tool calls. Light on tool-call payload (text retrieval chunks).
- **`planner-v1` planner agent.** Multi-turn, high tool-call volume. Sessions typically 10–25 turns, 5–20 tool calls. Heavier tool-call payload (structured results, up to a few hundred kB).
- **`support-agent` customer-support agent.** Multi-turn, with database tool calls. Sessions typically 3–8 turns, 2–5 tool calls. PII-heavy in the user messages.

The shape you pick drives your partitioning and retention choices. Flag `<!-- needs-research: ... -->` on any session-size assumption you cannot verify against a real production trace.

## Requirements

### Part 1 — The schema

Author `schema/` containing:

- `session.schema.json` — JSON Schema for the session row. At minimum: the fields in chapter 04's session example — `trace_id`, `session_id`, `tenancy.*`, `under_test.*`, timestamps, `input.text_ref`, `final_output.text_ref`, counts, outcome, `children_ref`.
- `turn.schema.json`, `model_call.schema.json`, `tool_call.schema.json`, `policy_decision.schema.json` — JSON Schemas for each child-row shape. Model-call rows match the mod-406 envelope fields; tool-call rows carry `tool_id`, `arguments_hash`, `output_hash`, `latency_ms`, `resource_counts`; policy-decision rows carry the decision shape from mod-405 chapter 03.
- `hot-index.ddl.sql` — the DDL for the columnar store. One table per shape above; references (`children_ref`, `messages_ref`, etc.) are string columns, not foreign keys to the content.
- `README.schema.md` — one page documenting the schema choices, the indexed columns, and the partitioning scheme.

Two requirements the schema must meet:

- No large payload (message body, tool-call output, model-call message history) is inlined in the hot-index table. All large payloads are object-store references by content hash.
- Every row carries `trace_id` and the cross-plane ids it belongs to (`gateway_request_id` on model-call rows, policy-decision ids on policy rows). A reviewer can join across tables by `trace_id` without extra rules.

### Part 2 — The capture sink

Author `sink/` implementing:

- A `TraceSink` interface with `write_session(session)`, `write_turn(turn)`, `write_model_call(call)`, `write_tool_call(call)`, `write_policy_decision(decision)`, `close(session_id)`.
- A `FileSystemSink` implementation that writes: content blobs under `{bucket}/{yyyy}/{mm}/{dd}/{session_id}/` with content-hash filenames; index rows to Parquet files (one partition per day) consumable by DuckDB.
- Hash computation at write time; the row carries the hash, the blob is keyed on it. Reading the same session twice produces the same hashes.
- A `redaction_profile` knob: `off` for the exercise's default, `strict` for the post-store redaction described in chapter 04 (stretch goal part 6).

### Part 3 — The seed dataset

Either:

- Reuse traces from exercise 01 / 02 if you built them.
- Use the seed-generation script pattern below: write a `seed.py` that generates 50 synthetic sessions covering four shapes — all-success, mid-session-provider-overloaded-and-retried, tool-loop-exhausted, content-filter-refusal. Each synthetic session should have a plausible user message, 2–8 turns, and a believable final output. The payload does not need to be semantically meaningful; the *shape* has to be right so your replay engine exercises the real code paths.

Persist the seed and commit it alongside the code — a reviewer should get the same traces on `make seed`.

### Part 4 — The query surface

Wire a small CLI (or REST handlers; CLI is simpler) exposing:

- `traces ls --agent writer-v1 --since 1d --outcome tool_loop_exhausted` — scan the hot index, return session rows.
- `traces show <trace_id>` — full session walk-through: input, turn-by-turn model calls and tool calls, outcomes, cost.
- `traces replay <trace_id> --mode <exact|under-test-swap|judge-swap> [options]` — invoke the replay engine (part 5) and emit a new `EvalRecord`.
- `traces slice <trace_id> --slice-contract final-answer-and-last-2-turns` — produce a reduced view of the trace suitable for a judge call.

### Part 5 — The replay engine

Implement all three replay modes described in chapter 04:

- **Exact replay.** Re-fetch the recorded model calls and tool calls; re-execute against the same under-test configuration. Cheap sanity check; produce a diff against the recorded outputs on a provider-specified tolerance (exact match for your stubbed under-test model).
- **Under-test-swap replay.** Re-run the agent loop using the original inputs (user message + recorded tool outputs) but a new under-test model, prompt, or tool-schema. Decide on new tool-call divergence per chapter 04's three policies and implement at least `record-and-replay` and `simulator-dispatch`. Record which policy fired on the output record.
- **Judge-swap replay.** Re-score the recorded under-test outputs with a new judge model / judge prompt. Under-test is not re-called; judge cost is the only cost.

Each replay emits a `replay-output/<replay_id>.evalrecord.jsonl` file with the `EvalRecord` shape, carrying a `replay.mode`, `replay.original_trace_id`, and (for under-test-swap) `replay.new_tool_call_policy` field.

### Part 6 — The worked walkthrough

Pick one failing session from the seed (the `tool_loop_exhausted` shape is ideal) and walk through the full regression-investigation path:

- `traces ls` to locate the failure.
- `traces show` to read the session.
- `traces replay --mode under-test-swap --under-test platform:writer-v1-candidate` to test a candidate fix.
- Produce a diff narrative (in the README) explaining what the candidate did differently and whether it resolved the tool loop.

### Part 7 — The README

One page covering:

- Which agent shape you picked and why.
- The schema's partitioning (date-first vs. tenant-first) and the rationale.
- The three replay modes you implemented and the specific divergence policy under-test-swap uses.
- The walkthrough from part 6.
- One open question for `ai-eval-engineer` or `ai-infra-sre` — e.g. "when a replay shows the candidate loses the loop but on a slightly different error path, is the regression 'fixed' or 'moved'?"

## Starter guidance

- **DuckDB is enough.** A Parquet file per day under a `hot-index/` directory, queried with DuckDB, hits the chapter's expected query shapes without a cluster. Reserve ClickHouse / BigQuery for scaling up.
- **Content-hash paths kill a class of race conditions.** Writing to `{session_id}/{content_hash}.json` means a repeated write is a no-op, not a corruption. Lean on it.
- **The simulator-dispatch policy is where stretch work lives.** The cheap implementation is "return the first recorded output that matches the tool name"; a better one keys on tool name + arguments hash. Even the cheap one is instructive.
- **Pin the replay-contract version.** When you change the divergence policy, bump a `replay_contract_version` string on the replay output. Chapter 03 relies on this version being stable for trace-to-catalogue promotion.
- **Redact at capture, not at replay.** The cleanest story in the exercise is to not store secrets in the first place. A stretch goal exercises post-store redaction; the default exercise should not.
- **`traces replay` is the primary read path.** If it feels like an afterthought, redesign.

## Acceptance criteria

Mark done when:

- [ ] The JSON Schemas validate the recorded rows from your seed; a session with a missing required field is rejected at write time.
- [ ] The hot index DDL loads into DuckDB; a `SELECT trace_id FROM session_sessions WHERE outcome = 'tool_loop_exhausted'` returns the expected rows.
- [ ] `traces ls`, `traces show`, `traces replay`, and `traces slice` each work against the seed; replay modes produce distinct `EvalRecord` outputs with correctly populated `replay.*` fields.
- [ ] Under-test-swap replay with the `record-and-replay` divergence policy refuses a session whose candidate makes a tool call not present in the recording and records the refusal in the output.
- [ ] Judge-swap replay produces a new verdict for every row in the seed without calling the under-test model again (verify via a cost counter).
- [ ] The walkthrough in the README shows a credible investigation path from failure to fix-candidate score.
- [ ] No content blob is inlined in the hot-index tables; every reference is a content-hash object-store path.

## Stretch goals

- **Partition by tenant.** Re-partition the hot index by tenant first, date second. Measure the query-pushdown differences for a single-tenant scan and a cross-tenant one.
- **Post-store redaction.** Implement the `redaction_profile=strict` path: on read, run detectors over the message blobs and emit a redacted view. Prove the original stays intact; prove the catalogue-bound view does not carry the raw PII.
- **Trace-to-catalogue promotion.** Wire a `traces promote <trace_id> --fixture writer-v1-regression` command that transforms the session into a catalogue row and sends it through an `ai-eval-engineer` review stub.
- **At-rest encryption.** Encrypt content blobs with a per-tenant key fetched from a KMS stub. A tenant-key revocation invalidates every blob for that tenant.
- **Cross-plane id navigation.** Build a `traces jump --from <trace_id> --to mod-406-billing` command that resolves the gateway_request_ids on the session's model-call rows and queries the mod-406/exercise-04 billing event stream.
- **Simulator-dispatch learning.** When the simulator is asked about a (tool_name, arguments_hash) pair it has not seen, record the request and surface it on the next leaderboard refresh so a reviewer can decide whether to extend the simulator or admit it as live-dispatch.
