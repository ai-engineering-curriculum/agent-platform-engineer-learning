# exercise-03: Standardised Safety-Event Schema

**Estimated effort:** 3 hours · **Output:** a versioned `SafetyRecord` JSON Schema, an emitter library wired into the three hooks, a Kafka / event-stream subscriber that performs the risk-register consumer's primary query, an EU AI Act Article 72 export command, a conformance-test suite, and a short README (~1 page) · **Pairs with:** chapter [05](../05-standardised-safety-event-schema.md); feeds exercise 04 and project 404. Builds on exercises [01](exercise-01-safety-plane-architecture-and-sla.md) and [02](exercise-02-prompt-injection-defence-at-platform-scope.md).

## Objective

Chapter 05 pinned the schema. This exercise forces you to commit to it — write the file, validate every emission, expose two real consumers over the resulting stream, and prove the schema is tight enough that a consumer can read records without engine-specific transformation.

The lesson the exercise intends to teach: schemas that *almost* validate are schemas that break their consumers at 3 am. The time to tighten is before a tenant exports a quarter of records for an auditor.

## Prerequisites

- You have completed [exercise 01](exercise-01-safety-plane-architecture-and-sla.md) and [exercise 02](exercise-02-prompt-injection-defence-at-platform-scope.md), or an equivalent plane that emits `SafetyRecord`s on each hook.
- You have read chapter [05](../05-standardised-safety-event-schema.md), and have skimmed chapter [06](../06-measuring-guardrail-effectiveness.md).
- You have a working event stream from exercise 01 (Kafka, Redpanda, flat-file bus, or equivalent).
- You have a minimal sense of the [EU AI Act — Regulation (EU) 2024/1689](https://eur-lex.europa.eu/eli/reg/2024/1689/oj), specifically [Article 15](https://artificialintelligenceact.eu/article/15/), [Article 72](https://artificialintelligenceact.eu/article/72/), and [Article 73](https://artificialintelligenceact.eu/article/73/). You will not interpret them (counsel's job); you will produce the mechanical evidence.
- You have familiarity with [JSON Schema Draft 2020-12](https://json-schema.org/draft/2020-12/release-notes) or equivalent validation.

## Problem statement

The `ai-risk-engineer` has asked for the following: a stream of `SafetyRecord`s they can consume without a translator per engine, and a quarterly export command the counsel and the compliance officer can hand to a regulator. Your output has five parts:

1. The `SafetyRecord` v1 JSON Schema in a canonical location (`schemas/safety-record-v1.json`), with every field constrained as chapter 05 describes.
2. An emitter library (Python, Go, or TypeScript — your choice) that wraps the three hooks' emit paths and validates each record before publishing to the stream.
3. A subscriber that implements the risk register's primary grouping query (`group by verdict.categories × tenant_id × workload_class` over a 28-day window) and publishes a dashboard view.
4. An EU AI Act post-market-monitoring export command — `platform safety export --tenant X --use-case Y --from ... --to ...` — that produces a tarball with the schema chapter 05 describes.
5. A conformance-test suite that fails on any emission that violates the schema or breaks downstream-consumer assumptions.

## Requirements

### Part 1 — The schema file

`schemas/safety-record-v1.json` — JSON Schema Draft 2020-12 (or your framework's canonical format). It MUST:

- Use `$id: "https://platform.local/schemas/safety-record/v1"` or an equivalent stable, documented URI.
- Pin `schema_version: const: "safety-record/v1"`.
- Declare every required field from chapter 05.
- Constrain hex fields (`trace_id`, `span_id`) with the exact regex.
- Constrain enums (`workload.workload_class`, `workload.tier`, `hook.point`, `subject.subject_type`, `subject.trust_tier`, `verdict.decision`, `outcome.action_taken`) to the lists chapter 05 declares.
- Require `subject.subject_hash` as a SHA-256 string matching `"sha256:[0-9a-f]{64}"`.
- Require `verdict.confidence` in [0, 1].
- Require `hook.engine` and `hook.policy_pack` to include version identifiers (regex check — your choice of pattern).

Produce an `examples/` directory with at least five valid records (one per verdict decision + one for each hook point not already covered) and at least three invalid records (missing required field, wrong `schema_version`, invalid `trace_id` format). The conformance test loads these.

### Part 2 — The emitter library

A package `safety_record_emitter` (or your framework equivalent) with:

- A `SafetyRecord` type (dataclass, pydantic model, protobuf — your choice; must round-trip to the schema).
- A `SafetyRecordEmitter` class that:
  - Takes a sink URI (`kafka://localhost:9092/safety.events.v1` / `file:///tmp/safety-events.jsonl` / your choice).
  - Validates every record against the schema before publishing (configurable; default on).
  - Emits two surfaces: the stream publish, and a span event on the current OpenTelemetry span (chapter 05's dual-home pattern).
  - On validation failure: logs the failure, emits a metric, and MUST NOT silently drop. Choose between (a) raising and rejecting the verdict, (b) publishing to a dead-letter queue — document the choice.
- Three convenience functions matching the three hooks: `emit_input_filter(...)`, `emit_output_validator(...)`, `emit_tool_invocation(...)`. Each accepts the minimum required fields and computes `record_id`, `emitted_at`, `hook.latency_ms`.

The three hooks from exercise 01 must be migrated to use this library. Remove any ad-hoc record construction from the hook endpoints.

### Part 3 — The risk-register subscriber

A service `risk_register_consumer` that:

- Subscribes to the `safety.events.v1` stream (or your equivalent).
- Maintains a rolling 28-day aggregation: `count` grouped by `(verdict.categories, tenant_id, workload_class, verdict.decision)`.
- Exposes an HTTP `/v1/risk-register/summary` endpoint that returns the current aggregate as JSON.
- Exposes a `/v1/risk-register/drill?trace_id=...` endpoint that returns all records for a given trace.
- Writes a flat `risk-register-view.json` on a 1-minute cadence for a mock downstream BI tool.

The aggregation does not need to be production-grade — an in-process dict refreshed on consume is fine for the exercise. If you want a stretch, back it with [DuckDB](https://duckdb.org/) or [ClickHouse](https://clickhouse.com/) and show a real 28-day sliding window.

### Part 4 — The EU AI Act export command

A CLI `platform safety export` (name it what you like) that:

- Accepts `--tenant`, `--use-case`, `--from`, `--to`, `--output-dir`, `--include-content-store` (flag).
- Queries the stream (or the stream's long-term archive — you can simulate with the on-disk JSONL from Part 2).
- Produces a tarball with:
  - `manifest.json` — tenant, use case, time window, record count, content-store URI base, generated-at timestamp.
  - `records/YYYY-WW.jsonl` — records partitioned by ISO week.
  - `policy-packs/` — the policy-pack versions referenced by the records.
  - `engines/` — engine identifiers and versions referenced.
  - `refusal-templates/` — the refusal templates referenced.
  - (If `--include-content-store`) `content/` — pointers to (or fetched copies of) the content-store URIs on the records. Be mindful of size.
  - `README.md` — a short Article 72-aligned description of what the package is and how to verify it.
- Emits an integrity hash over the tarball; writes it to `manifest.json` and to a separate `checksum.sha256` file.

The command is the mechanical evidence; the regulator-facing *interpretation* is counsel's job. Do not pretend to make the legal call; do produce the artefact.

### Part 5 — Conformance tests

A test suite `tests/conformance/` that:

- Validates every example record against the schema.
- Validates every emitted record over a run of the demo from exercise 01 against the schema.
- Asserts the risk-register subscriber's aggregate matches a known-good value against a canned input.
- Asserts the export command produces a tarball whose manifest, schema version, record count, and integrity hash are self-consistent.
- Asserts the schema's enum sets match chapter 05's canonical lists — a drift between the chapter's definitions and your file is a test failure.

Minimum ~15 test cases.

### Part 6 — The README

One page covering:

- Where the schema file lives and its `$id`.
- How to run the emitter library against the hooks from exercise 01.
- How to subscribe (`python -m risk_register_consumer`).
- How to export a tarball and verify it (`platform safety export --tenant demo --from ...`).
- Two "I was surprised by" observations — e.g., "the `emitted_at` timestamp was initially in local time; the schema's `format: date-time` passed but the downstream consumer's 28-day window rolled incorrectly; the record MUST be in UTC."
- One open question for the risk engineer — e.g., "the schema requires `red_team_refs` as an array of strings but the risk engineer's current corpus id format uses non-ASCII characters; what is the canonical id format?"

## Starter guidance

- **Write the schema first; let everything else fail against it.** The order matters — writing the emitter before the schema produces an emitter that enforces whatever shape you happened to code.
- **Validate on emit, not just on consume.** A record that reaches the stream and only fails on the consumer is a record you have to clean up by hand. Chapter 05's "validate before publish" pattern is not optional.
- **The `subject_hash` is the test case most people get wrong.** It is a hash of the *original* content, not the normalised one; chapter 04 and chapter 05 are explicit. The normalised hash is a separate field.
- **Don't fight the UTC.** All timestamps are UTC (`Z` suffix). Everything downstream assumes it.
- **Simulate retention.** Even if your JSONL "archive" is on local disk, mimic the per-tenant partition: `safety-events/<tenant>/<YYYY>/<MM>/<DD>/*.jsonl`. The export command has a much easier life.
- **Keep the example records realistic.** The five examples in the schema's `examples/` directory are what downstream readers learn from; a toy example with "example_trace_id" instead of a real hex id teaches the wrong habit.

## Acceptance criteria

Mark done when:

- [ ] `schemas/safety-record-v1.json` exists with the chapter 05 fields, constraints, and enum sets. The `$id` is documented.
- [ ] The emitter library validates every record before publish; the three hooks from exercise 01 use it; ad-hoc record construction is removed.
- [ ] `risk_register_consumer` runs, aggregates, and exposes the two endpoints.
- [ ] `platform safety export` produces a tarball whose manifest is self-consistent and whose contents match chapter 05's export shape.
- [ ] Conformance tests exist and all pass. At least one failing test exists for each type of invalid record (missing field, wrong schema version, invalid format, enum violation).
- [ ] The README includes the two "surprised by" observations and the open question for the risk engineer.
- [ ] Every timestamp in every emitted record is UTC (literal `Z` suffix); a test asserts this.
- [ ] No record in the stream carries unredacted content — the subject hash and content-store URI are the only references.

## Stretch goals

- **Version the schema.** Add a `safety-record/v1.1` with one new optional field (e.g., `verdict.severity_estimate`); write the migration test showing `v1` consumers still work.
- **Wire the span-event emission.** Produce a real OTel trace with the `platform.safety.verdict` span event; verify the attributes carry the record_id, decision, categories, and are picked up by a collector (chapter 05 dual-home pattern).
- **Harden the export.** Use [Sigstore](https://www.sigstore.dev/) / [cosign](https://docs.sigstore.dev/cosign/overview/) to sign the tarball; include the signature in the package. A regulator-facing package benefits from attestation.
- **Build a tenant portal view.** A small web UI (one page) that shows a tenant their own `SafetyRecord` aggregates over the last 28 days — a self-service post-market-monitoring read.
- **Add a mod-408 integration.** Wire the risk-register subscriber to also publish a `platform.safety.aggregate.*` metric to a Prometheus endpoint; cross-reference mod-408 chapter 05.
- **Author the mod-407 fixture promotion stub.** When the risk register sees a cluster of records with the same `categories` and `subject_hash_normalized`, emit a candidate-fixture record for chapter 06's loop into mod-407.
