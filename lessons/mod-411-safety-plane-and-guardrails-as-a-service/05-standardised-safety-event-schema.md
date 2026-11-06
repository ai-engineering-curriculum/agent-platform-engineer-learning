# The Standardised SafetyRecord Schema

Chapter 01 declared an invariant. Chapters 02–04 showed engines and attacks. This chapter pins the schema the plane commits to — the `SafetyRecord` — and shows how the stream satisfies the `ai-risk-engineer`'s AI risk register and the EU AI Act Article 15 and Article 72 obligations without a bespoke transformation per consumer.

The design goal: one record shape, keyed on `trace_id`, consumable without engine-specific or vendor-specific translation. If a consumer needs a shape not in this schema, the schema evolves (versioned bump); consumers do not maintain per-engine translators.

> **Regulatory caveat.** This chapter walks the mechanical shape of the schema and its intended audience. Legal interpretation of a specific tenant's obligations under the [EU AI Act — Regulation (EU) 2024/1689](https://eur-lex.europa.eu/eli/reg/2024/1689/oj), applicability of Article 6 / Annex III "high-risk" classification, Article 73 serious-incident thresholds, GDPR interactions, and the forthcoming AI Act implementing acts are counsel's domain. The plane produces mechanical evidence; the counsel interprets it. Cross-reference mod-408 chapter 06 for the broader post-market-monitoring picture.

## The schema

The record, in JSON Schema form.

```yaml
$schema: "https://json-schema.org/draft/2020-12/schema"
$id: "https://platform.local/schemas/safety-record/v1"
title: "SafetyRecord v1"
type: object
required: [record_id, schema_version, emitted_at, trace_id, tenant_id, workload, hook, subject, verdict]
properties:
  record_id:
    type: string
    description: "Globally-unique id. Paved road uses ULID with 'sr_' prefix."
    pattern: "^sr_[0-9A-HJKMNP-TV-Z]{26}$"
  schema_version:
    type: string
    const: "safety-record/v1"
  emitted_at:
    type: string
    format: date-time
  trace_id:
    type: string
    description: "W3C Trace Context trace_id (lowercase hex, 32 chars). From mod-408."
    pattern: "^[0-9a-f]{32}$"
  span_id:
    type: string
    description: "Span under which the hook executed. Lowercase hex, 16 chars."
    pattern: "^[0-9a-f]{16}$"
  session_id:
    type: string
    description: "Agent session id from the runtime (mod-402)."
  tenant_id:
    type: string
  workload:
    type: object
    required: [agent_id, agent_revision, workload_class, tier]
    properties:
      agent_id: { type: string }
      agent_revision: { type: string, description: "Git SHA or equivalent." }
      workload_class:
        type: string
        enum: [interactive-chat, background-task, batch-research, code-execution-loop, agent-to-agent]
      tier:
        type: string
        enum: [T0, T1, T2, T3]
  hook:
    type: object
    required: [point, engine, policy_pack, latency_ms]
    properties:
      point:
        type: string
        enum: [input_filter, output_validator, tool_invocation_hook]
      engine:
        type: string
        description: "Engine identifier with version, e.g., llama-guard-3-8b@2024-07-23"
      policy_pack:
        type: string
        description: "Policy-pack identifier with version, e.g., paved-road@2026-10-01"
      latency_ms:
        type: number
      model_route:
        type: string
        description: "Which model gateway route produced the completion (output_validator only)."
      tool_target:
        type: string
        description: "MCP tool identifier (tool_invocation_hook only)."
  subject:
    type: object
    required: [subject_type, subject_hash, subject_source, trust_tier]
    properties:
      subject_type:
        type: string
        enum: [user_turn, retrieved_document, tool_result, model_output, tool_call_argument, memory_read, memory_write, a2a_message]
      subject_hash:
        type: string
        description: "SHA-256 of the subject content (original bytes)."
        pattern: "^sha256:[0-9a-f]{64}$"
      subject_hash_normalized:
        type: string
        description: "SHA-256 of the normalised subject (post NFKC, zero-width strip, etc.)."
        pattern: "^sha256:[0-9a-f]{64}$"
      subject_source:
        type: string
        description: "Pointer into the trace pipeline, e.g., runtime:turn_events[12] or retrieval:index/doc-123."
      trust_tier:
        type: integer
        enum: [0, 1, 2, 3]
      content_store_uri:
        type: string
        description: "URI of the full content in the tenant-partitioned content store (mod-408 ch. 04)."
  verdict:
    type: object
    required: [decision, categories, confidence, refusal_template]
    properties:
      decision:
        type: string
        enum: [allow, redact, require_human_approval, refuse]
      categories:
        type: array
        description: "OWASP identifier strings. Chapter 03 defines the set and the rev."
        items: { type: string }
      owasp_rev:
        type: object
        properties:
          llm_top_10: { type: string, description: "e.g., '2025'." }
          agentic_threats: { type: string }
      confidence:
        type: number
        minimum: 0
        maximum: 1
      refusal_template:
        type: string
        description: "Template id with version, e.g., paved-road.refuse.prompt_injection.v2. Null if decision=allow."
      native_categories:
        type: object
        description: "Engine-native categories reported before OWASP mapping. For debug / mapping test."
  red_team_refs:
    type: array
    items: { type: string }
    description: "Red-team corpus row ids known to correspond to this verdict. Populated during replay; null in production."
  risk_register_ref:
    type: string
    description: "AI risk register entry id this verdict rolls up to. Populated downstream."
  fixture_promotion_ref:
    type: string
    description: "mod-407 catalogue fixture id if the failure was promoted into a regression fixture."
  atlas_refs:
    type: array
    items: { type: string }
    description: "Optional MITRE ATLAS technique references."
  outcome:
    type: object
    required: [action_taken]
    properties:
      action_taken:
        type: string
        enum: [block_turn, strip_substring, drop_document, pause_for_human, allow_with_annotation, no_op]
      downstream_notified:
        type: array
        items: { type: string }
      hitl_approval_request_id:
        type: string
        description: "If decision=require_human_approval, the ticket/request id."
```

A real record — the chapter 01 example, filled out against this schema:

```json
{
  "record_id": "sr_01HWX3K2M8R9JW1NXB7MXFKCDQ",
  "schema_version": "safety-record/v1",
  "emitted_at": "2026-11-06T14:22:03.142Z",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "00f067aa0ba902b7",
  "session_id": "sess_01HWX2K1M7R0JW9N0B0MXFKCDQ",
  "tenant_id": "acme-financial",
  "workload": {
    "agent_id": "writer-v1",
    "agent_revision": "git:abcd1234",
    "workload_class": "interactive-chat",
    "tier": "T1"
  },
  "hook": {
    "point": "input_filter",
    "engine": "llama-guard-3-8b@2024-07-23",
    "policy_pack": "paved-road@2026-10-01",
    "latency_ms": 42
  },
  "subject": {
    "subject_type": "user_turn",
    "subject_hash": "sha256:7f3c2a4b8e9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5061728394a5b6c7d8e9f0",
    "subject_hash_normalized": "sha256:9a8b7c6d5e4f3a2b1c0d9e8f7a6b5c4d3e2f1a0b9c8d7e6f5a4b3c2d1e0f9a8b",
    "subject_source": "runtime:turn_events[12]",
    "trust_tier": 1,
    "content_store_uri": "s3://platform-content/acme-financial/2026/11/06/sess_01HWX2K1M7.../turn-12.txt"
  },
  "verdict": {
    "decision": "refuse",
    "categories": ["LLM01:prompt_injection"],
    "owasp_rev": { "llm_top_10": "2025", "agentic_threats": "2025-07" },
    "confidence": 0.94,
    "refusal_template": "paved-road.refuse.prompt_injection.v2",
    "native_categories": { "llama_guard_3": ["S1"] }
  },
  "red_team_refs": null,
  "risk_register_ref": null,
  "fixture_promotion_ref": null,
  "atlas_refs": ["AML.T0051"],
  "outcome": {
    "action_taken": "block_turn",
    "downstream_notified": ["mod-410:safety-refusal-rate-sli", "risk-register:ingest-queue"]
  }
}
```

Nine properties the schema enforces.

- **Trace-id keyed.** Every consumer joins to mod-408 traces on `trace_id`. No bespoke cross-reference.
- **Engine and policy-pack versioned, inline.** A query "which records came from Llama Guard 3 before the policy pack update" is a point query, no deployment-history join.
- **Subject hashed, not embedded.** The record never carries PII. The content lives in the tenant-partitioned content store (mod-408 chapter 04); the record carries only the URI and the hash.
- **Trust tier is first-class.** Forensics on indirect-injection failures groups by `subject.trust_tier`.
- **OWASP identifiers and rev are both pinned.** Chapter 03's mapping is honoured; a rev bump is a visible schema change, not a silent drift.
- **Native categories preserved for debug.** The engine's raw output is retained; a mapping test can replay old records when the mapping file is updated.
- **Refusal template versioned.** The user-facing wording is reproducible; two refusals from the same template version are identical to the user.
- **Downstream notification list declared.** Who gets this record — risk register, SRE plane, A2A circuit breaker, HITL queue — is on the record, not an out-of-band deployment.
- **Optional refs are null, not absent.** Downstream consumers never have to handle "missing field vs. null"; the schema requires either a value or `null`.

## Where the record lives

Two homes.

- **An OpenTelemetry span event, inline on the trace.** Every hook emits a `platform.safety.verdict` span event on the chat span (for input filter / output validator) or the tool-invocation span (for tool-invocation hook), with attributes mirroring the top-level record fields. Mod-408 chapter 02 pins the attribute namespace. The span event gives the SRE plane (mod-410 chapter 02) the refusal-rate signal and gives the trace-backend view a one-click "what happened here" answer.
- **A dedicated event stream (SafetyEvent topic).** The same record, in full, published to the platform's event bus (Kafka / Pub/Sub / Kinesis). The risk register consumes from this stream; the EU AI Act export reads from it; the red-team replay harness writes to it. The stream is the long-term record; the span event is the trace-local shadow.

Both homes carry the same `record_id`. The stream is authoritative; the span event is a convenient join point.

```python
def emit_safety_record(record: SafetyRecord, span: trace.Span):
    span.add_event(
        name="platform.safety.verdict",
        attributes={
            "platform.safety.record_id": record.record_id,
            "platform.safety.decision": record.verdict.decision,
            "platform.safety.categories": record.verdict.categories,
            "platform.safety.hook.engine": record.hook.engine,
            "platform.safety.hook.policy_pack": record.hook.policy_pack,
            "platform.safety.confidence": record.verdict.confidence,
            "platform.safety.subject.trust_tier": record.subject.trust_tier,
            "platform.safety.refusal_template": record.verdict.refusal_template,
        },
    )
    safety_event_producer.send(topic="safety.events.v1", key=record.trace_id, value=record.as_json())
```

The attribute namespace is `platform.safety.*`. Mod-408 chapter 02 is the full declaration; this chapter uses it.

## Who consumes the stream

Five consumers the schema is designed to serve without transformation.

### Consumer 1 — AI risk register (`ai-risk-engineer`, level 25)

The `ai-risk-engineer` role maintains a register of residual risks, mitigations, owners, and review cadence. The register's rows are keyed by an identifier (`risk-register:R-YYYY-NNNN`); the plane's `SafetyRecord.risk_register_ref` points at the row.

The register's primary consumption pattern:

- **Group by `verdict.categories` × `tenant_id` × `workload_class`** to produce "how many LLM01 refusals did acme-financial's writer-v1 agent see this quarter".
- **Join to `red_team_refs`** to answer "which refusals correspond to known red-team findings vs. novel detections".
- **Drill to `trace_id`** for a specific incident's full context.
- **Filter on `confidence` and `decision`** to find high-confidence refuses vs. low-confidence redactions — the borderline is where risk-register entries often live.

The plane does not write to the register; the register's ingester consumes the stream and groups. Chapter 06 is where the scorecard view is built; this chapter pins the shape the register joins over.

### Consumer 2 — EU AI Act post-market monitoring

The [EU AI Act — Regulation (EU) 2024/1689](https://eur-lex.europa.eu/eli/reg/2024/1689/oj) obliges providers of high-risk AI systems to collect, document, and analyse performance data over the system's lifetime. Two articles anchor the obligation:

- **[Article 15](https://artificialintelligenceact.eu/article/15/)** — Accuracy, robustness, and cybersecurity. Among other things, requires that high-risk systems "perform consistently throughout their lifecycle" and that incidents / malfunctions are logged. The `SafetyRecord` stream is a large piece of the cybersecurity evidence (prompt-injection and tool-misuse rates are cybersecurity signals) and the stream's time-series feeds the "performs consistently" view.
- **[Article 72](https://artificialintelligenceact.eu/article/72/)** — Post-market monitoring. Providers must establish and document a post-market monitoring system proportionate to the risks; actively and systematically collect performance data; evaluate continuous compliance. The `SafetyRecord` stream is the operational artefact. Mod-408 chapter 06 is the broader post-market-monitoring picture; the safety plane contributes the safety slice.

The plane's export surface:

- **Per-tenant, per-use-case, per-time-window export.** `safety.events.v1` filtered by tenant / workload / time, serialised as JSONL, with a manifest listing hashes and content-store URIs.
- **Serious-incident evidence package.** Under [Article 73](https://artificialintelligenceact.eu/article/73/), providers must notify the market surveillance authority of serious incidents within strict deadlines. For a specific `trace_id` or `incident_id`, the plane emits a self-contained package: all `SafetyRecord`s on the trace; the full content from the content store (with the retention tag); the mod-408 trace export; the applicable policy pack; the engine version in play; the refusal-template contents.
- **Retention.** The `SafetyRecord` stream is retained per tenant per use case for the plan window the tenant commits to. Content-store retention is covered by mod-408 chapter 04; stream retention is a per-topic lifecycle rule.

```yaml
# Illustrative per-tenant export command
platform safety export \
  --tenant acme-financial \
  --use-case writer-v1-high-risk \
  --from 2026-07-01T00:00:00Z \
  --to 2026-09-30T23:59:59Z \
  --output s3://acme-financial-export/2026-q3-ai-act-article-72/ \
  --include-content-store \
  --include-trace-manifest
```

The command runs across the `safety.events.v1` stream and the content store; produces one tarball per week; the manifest lists every hash and URI. Chapter 05-content is designed so the tarball is directly loadable into the tenant's own compliance tooling.

### Consumer 3 — SRE plane (mod-410 chapter 02 SLI 5)

The safety-refusal-rate SLI reads from the stream. Two queries:

- **Refusal rate.** `count(decision IN refuse|redact) / count(*)` grouped by tenant and workload over 28-day rolling.
- **Drift.** The rate's deviation from its 28-day trailing baseline. Alerts on drift beyond ±30 %.

Mod-410 chapter 02 is the full treatment; the safety plane's contribution is the stream the SLI queries.

### Consumer 4 — Red-team replay harness (chapter 06)

The harness replays red-team corpus entries against the plane in a batch job. Each replay produces a `SafetyRecord` with the `red_team_refs` field populated. Chapter 06 is the scorecard methodology; the record shape is what makes the methodology mechanical rather than bespoke.

### Consumer 5 — Fixture promotion into mod-407

When a safety incident is reproducible, it promotes into mod-407's catalogue under `safety/<finding_id>`. The fixture carries the `SafetyRecord`'s `trace_id` and its `subject_source` so the fixture's inputs are reproducible. The record's `fixture_promotion_ref` fills in once the fixture is in the catalogue; a future record on the same shape joins to the fixture id and the deploy gate blocks the regression.

Cross-reference mod-410 chapter 06 — the same promotion pipeline the SRE plane uses for reliability fixtures.

## Mapping onto EU AI Act Article 15 and Article 72 obligations

A table the plane keeps alongside the schema. Each obligation is mapped to one or more fields and / or one or more operational commitments.

| Obligation (paraphrased) | Field / commitment |
| --- | --- |
| Article 15 — High-risk systems perform consistently and log malfunctions | `safety.events.v1` stream retention; the SRE plane's refusal-rate SLI; the quarterly scorecard (chapter 06) |
| Article 15 — Cybersecurity — resilience against unauthorised third-party attempts | LLM01 / LLM07 / T02 records are the evidence; the red-team replay harness is the operational test |
| Article 72 — Establish post-market monitoring system | `safety.events.v1` stream; per-tenant export command; this schema |
| Article 72 — Actively and systematically collect performance data | Every verdict → one record; retention per tenant plan; mod-408 chapter 06 integration |
| Article 72 — Evaluate continuous compliance with Articles 8–15 | `verdict.categories` × time dashboards; the quarterly scorecard |
| Article 73 — Report serious incidents within deadline | Serious-incident evidence package command; `trace_id` → tarball |
| Article 12 — Record-keeping | Stream retention + content-store retention; integrity hash (`subject_hash`) + tamper-evident log cross-ref |

Ancillary (not AI Act but plane-relevant):

- **GDPR Article 32 (security of processing).** PII handling is captured in `subject.subject_hash` and the content-store retention policy; the plane never emits unredacted PII inline in the record.
- **GDPR Article 15 / 17 (subject access and erasure).** On an erasure request, the content store removes the subject's content; the `SafetyRecord` carries only the hash and is retained as evidence (hashes are not personal data). Cross-reference mod-408 chapter 04.

<!-- needs-research: the exact wording of Article 15 and Article 72 shifts between the Regulation text, the forthcoming implementing acts, and the operator-facing guidance from member-state regulators. Pin the version date on the mapping file when shipping; re-verify annually with counsel. -->

## Schema evolution

A versioned schema has to evolve. Three rules.

- **Backward-compatible additions are a minor-version bump.** `safety-record/v1.1`. New optional fields are added; the `v1` validator still accepts the new records if it ignores unknown fields.
- **Breaking changes are a major-version bump.** `safety-record/v2`. Consumers migrate on their own cadence; the plane emits both for a documented transition window and drops `v1` after.
- **Schema changes are reviewed in the policy-pack change-control cadence.** Chapter 07's governance board; mod-413's operating cadence.

The `schema_version` field is literal — `safety-record/v1` for every record emitted under this schema. A consumer that reads a record MUST check the `schema_version` before deserialising.

## Common anti-patterns

Four worth naming.

- **"We'll emit the record only on refuse."** The stream is a time-series of all verdicts, not an alert log. Refuse-only records cannot answer "what fraction of inputs are triggering the input filter" because the denominator is lost. Emit allow records at a sample rate if emitting all is too expensive; chapter 06 pins the sampling.
- **"We'll embed the content in the record."** PII. Breakage under GDPR and under tenant policy. The record carries hashes and URIs; the content lives behind the retention policy.
- **"We'll let each engine emit its own record shape."** The whole point of the schema is one shape. Engines emit their native verdict; the plane's adapter produces the standardised record. The adapter is a few lines per engine; the saving across consumers is massive.
- **"We'll version only on breaking changes."** Minor-version bumps are cheap for consumers and give the plane a changelog without renegotiating the major version. Emit minor bumps for additions; keep majors for breaks.

## Summary

- The `SafetyRecord` schema is one versioned shape — `safety-record/v1` — keyed on `trace_id`, emitted by every hook on every verdict. Lives both as a span event (inline on mod-408 traces) and as a dedicated event stream (authoritative, for long-term consumption).
- Five consumers are served without transformation: the AI risk register (`ai-risk-engineer`), the EU AI Act post-market-monitoring export (Article 72 + Article 15 + Article 73), the SRE plane's safety-refusal-rate SLI (mod-410 chapter 02), the red-team replay harness (chapter 06), and the mod-407 fixture-promotion pipeline.
- The record never carries unredacted content — hashes and content-store URIs only. The content store (mod-408 chapter 04) is the retention substrate; the stream is the evidence log.
- Schema evolution is versioned; additions are minor bumps, breakages are major with a declared transition window. Policy-pack change control reviews schema changes alongside engine-swap and category-mapping changes (chapter 07, mod-413 operating cadence).

The next chapter opens measurement — how the plane scores its own effectiveness against the red-team findings `ai-risk-engineer` and `agentic-safety-engineer` produce.
