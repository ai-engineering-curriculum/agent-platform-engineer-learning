# Structured Output and PII Redaction at the Gateway

Two obligations recur in every agent system and are almost always implemented wrong in the first pass: *the response must conform to a schema* and *the request must not egress forbidden data*. Both are naturally placed in the agent, right where the schema or the data is understood — and that is exactly why they belong at the gateway. One central enforcement point, one rolling release, one audit stream, and zero reliance on every agent author remembering the rule.

This chapter builds both as gateway concerns. The split-token accounting from chapter 04 and the cache namespace from chapter 05 will reappear — redaction changes bytes, which changes cache hits; schema failures interact with the retry policy from chapter 03. Order the enforcement correctly or produce regressions in all three directions.

## Why at the gateway and not in the agent

Three reasons, each one sufficient on its own.

- **Uniformity.** A schema is a contract between the agent and the model provider; the gateway enforces every contract the same way. An agent that implements its own JSON validator silently introduces drift from every other agent on the platform. Same for PII redaction: a single detector library on the gateway is the one place compliance can audit.
- **Deploy independence.** When a safety incident requires adding a new detector for a newly-observed secret format, the fix rolls at config-promotion speed (chapter 02) rather than at per-agent redeploy speed. A detector in each agent is a detector you cannot upgrade in a hurry.
- **Gateway owns the egress seam.** The gateway is the one place every prompt passes through on its way to a third-party provider. That seam is the only place where "has this request actually crossed the trust boundary" is answerable without ambiguity. Placing redaction inside the agent assumes every agent is correctly configured; placing it at the gateway assumes only that every call passes through the gateway, which is already invariant one from chapter 01.

## Structured output — the three provider shapes

The gateway's job is to compose a single internal "structured output" shape on top of three materially different provider surfaces.

- **Native JSON mode or schema-constrained decoding.** OpenAI's `response_format: { type: "json_schema", strict: true, schema: ... }` constrains the decoder to emit exactly the schema. Google's `responseSchema` on `generationConfig` does the same. Self-hosted vLLM / SGLang / TensorRT-LLM provide guided decoding (grammar, regex, JSON schema) that is equivalent and typically cheaper. These are **in-decoder** enforcements — the model cannot sample tokens that would violate the schema.
- **Native JSON mode without schema.** Older OpenAI `response_format: { type: "json_object" }` — the output is valid JSON but the shape is not constrained. The gateway must validate against the application schema itself.
- **No native schema support.** Anthropic (at the time of writing) does not accept a response schema directly; the convention is to pass the schema as a tool definition and use tool-choice to force the tool call. The gateway does the schema-as-tool translation for the agent.

<!-- needs-research: confirm each provider's native structured-output surface at reading time. Anthropic in particular may have introduced a native response-schema option after this writing; OpenAI's strict mode has moved repeatedly across 2024-2026. -->

The internal shape the gateway exposes to agents is a single field on the gateway request:

```json
"response_format": {
  "type": "json_schema",
  "schema_ref": "tenant:acme/schemas/writer-v1@3",
  "strict": true,
  "on_invalid": "retry_once_then_fail"
}
```

The gateway's adapter maps this to the provider's native capability where possible; applies a gateway-local validator otherwise. The capability matrix from chapter 02 records which path each mapping uses, and the response envelope records which was applied.

## Composing structured-output with the retry policy

Structured-output failures are a chapter-03 retry decision. A representative extension to the retry policy:

- **`schema.invalid_json`** (parse failed) — `max_attempts: 1`, `reinforce_instruction: true`. The gateway appends a system message to the retry: "your previous response was not valid JSON; emit JSON only, no preamble or trailing content". One rescue attempt before failure.
- **`schema.mismatch`** (parses but violates schema) — same policy. The reinforcement message includes the specific validator error: "field `score` must be a number between 0 and 1".
- **`schema.exceeded_max_output_tokens`** (response truncated before the JSON closed) — the retry increases `max_output_tokens` if the capability allows, otherwise fails.
- **`schema.provider_native_failed_hard`** (OpenAI strict mode returned a refusal because it could not satisfy the schema) — `max_attempts: 0`. A hard-strict refusal usually means the schema is impossible to satisfy for the input; retry makes it worse.

Three design points:

1. **Reinforcement reinforces, it does not re-architect.** One shot. If the one-shot fails, the gateway surfaces a `terminal.output_malformed` outcome and the runtime decides. The gateway is not going to iterate forever to bully the model into the schema.
2. **Reinforcement is applied as an additional system or user message, not as a mutation of the prior one.** The conversation history on retry is intact, with the reinforcement appended. This preserves the agent's own invariants about conversation state.
3. **The reinforcement message is part of the audit log.** The billing event records that a rescue was attempted; the span carries the message content under `gen_ai.gateway.rescue_message`.

## Schema registration and versioning

Schemas live centrally, same pattern as the routing table and price table (chapters 02, 04).

- **Schema storage.** `tenant:{tenant_id}/schemas/{name}@{version}`. A versioned, signed JSON Schema artefact. New versions are additive admissions; a change that breaks existing agents is a new name.
- **Validation at admission.** Schemas are themselves validated against JSON Schema meta-schemas (draft 2020-12 is a reasonable default) and against a platform-level style guide (e.g. all string enums have a `description`; all arrays have a `maxItems`). Invalid schemas do not promote.
- **Reference by ref, not value, in requests.** The agent names the schema by `schema_ref` on the request; the gateway resolves it at runtime from its config bundle. This decouples schema evolution from agent deploys.

The versioning discipline matters most on strict-schema providers: a schema change is an in-decoder behaviour change. Capture which schema version was in effect for a given billing event (chapter 04's event already includes it as an obligation).

## PII redaction — the obligation-driven enforcement model

The second half of this chapter. The redaction decision is not made at the gateway; it is applied at the gateway. The decision comes from the policy plane (mod-405) as an obligation on the gateway request:

```json
"obligations": {
  "redact": [
    { "kind": "pii.email", "action": "mask", "mask_token": "<email>" },
    { "kind": "pii.ssn", "action": "refuse" },
    { "kind": "secret.api_key_prefix_sk", "action": "refuse" },
    { "kind": "pii.phone", "action": "mask", "mask_token": "<phone>" }
  ]
}
```

The policy plane produced this list based on the tenant's classification policy, the destination provider, and the harm model. The gateway's job is to:

1. **Detect.** Run detectors over the serialised request content (prompt, tool definitions, documents in cache blocks).
2. **Decide per match.** For each match, apply the obligation — mask or refuse.
3. **Enforce.** If any match says `refuse`, the request is denied with `gateway.egress_refused_by_obligation` and never reaches the provider. If matches are all `mask`, the request body is rewritten with the masked bytes and the rewritten body is what hits the provider.
4. **Record.** The billing event (chapter 04) lists every obligation that applied, with match counts, and never the original matched bytes.

## Detectors — three tiers of precision

- **Pattern detectors.** Regexes and token-classifier shortcuts for well-structured identifiers: `sk-...`-style API keys, PEM blocks, JWTs, IPv4/IPv6 addresses, credit-card numbers (with Luhn validation), SSN patterns, phone numbers with E.164 shapes. Fast, deterministic, auditable. Caveat: regex-based detectors are prone to false positives on code and logs; tune per-detector.
- **Named-entity detectors.** Classifier models (small, often distilled from BERT-family) that tag spans of text as person names, locations, organisations, medical conditions. Higher recall on free-form text; lower precision; cost measurable per request.
- **Context-aware detectors.** LLM-based detection (a small model, often a 7B-class) applied to the prompt, asked whether it contains a specific class of content. Last-resort, high cost. Useful on tenant-configured custom classes where the taxonomy is domain-specific.

A gateway typically runs tier 1 on every request, tier 2 on requests above a size threshold or on tenants whose policy requires it, tier 3 on request classes explicitly configured. The policy obligation drives which tier applies per class.

Industry-standard baselines include [Microsoft Presidio](https://microsoft.github.io/presidio/) (open-source PII analyser), [Google Cloud DLP](https://cloud.google.com/sensitive-data-protection), and AWS Macie / Comprehend — each shippable as a sidecar or library.

## Where redaction runs in the request flow

Pin the order; drift costs money or correctness.

```
request arrives
  ├─ admission (chapter 04)                       // deny early on quota
  ├─ obligation pull from policy-plane decision   // the redact list
  ├─ redaction pass (this chapter)
  │   ├─ detect
  │   ├─ decide-per-match
  │   └─ refuse OR mask (mutates request body)
  ├─ cache namespace / prefix hash (chapter 05)   // after redaction!
  ├─ routing + capability check (chapter 02)
  ├─ attempt walk + retry (chapter 03)
  └─ response → structured-output validation → billing event
```

The ordering has three consequences worth internalising:

1. **Redaction happens *before* cache-key composition.** The cache key is derived from the redacted body, not the original. Otherwise two requests whose only difference is an embedded email address hash differently and miss the cache. Redaction normalises inputs in the same way prefix normalisation does (chapter 05) — after redaction, the gateway hashes.
2. **Redaction happens *before* routing selection.** A detector that triggers `refuse` must stop the request from reaching any backend. Routing after redaction ensures that.
3. **Response-side redaction (if any) happens after the provider response, not before.** The symmetric case — the response may contain PII the model surfaced from an uploaded document — is a chapter-11 safety-plane concern. The gateway can run detectors on the response body, but the policy is usually a safety-plane obligation handed down separately.

## The cache + redaction interaction

Two concrete scenarios worth walking through.

### Scenario A — same tenant, different users, same prefix

The system prompt is tenant-wide; two different users under the tenant send requests. Redaction is applied to each user's message. The stable prefix (tenant-wide system prompt, tools, documents) is unchanged by redaction because it contains no PII. The cache works: both users' requests hit the same cached prefix.

### Scenario B — same tenant, different users, system prompt contains user context

If the tenant's system prompt is templated with the user's identity — "You are helping Alice Smith with her calendar" — then redaction (which may mask user names) rewrites the system prompt bytes, and the cache key diverges per user. The cache hit rate collapses. The fix is architectural: push user-specific context out of the stable prefix and into the volatile suffix, so the cached portion is truly stable across users. This is the single most common "why is our cache hit rate low" diagnosis from the fourth question in chapter 05.

The lesson: redaction is downstream of agent design. Agents authored well — stable prefix stable across users — compose cleanly with redaction and caching. Agents authored poorly — user context woven into system prompt — compose badly, and the gateway makes the tradeoffs visible.

## Refuse semantics

A `refuse` obligation is terminal. The request does not go to the provider; no attempt is made; the response to the runtime is a structured error:

```json
{
  "gateway_request_id": "gw_...",
  "outcome": "refused",
  "refusal": {
    "reason_code": "gateway.egress_refused_by_obligation",
    "matched_class": "pii.ssn",
    "match_count": 1,
    "obligation_id": "obl_...",
    "policy_decision_id": "01HV8X..."
  },
  "cost": { "total_usd": 0 },
  "tokens": { "input_cached": 0, "input_non_cached": 0, "output": 0 }
}
```

Three design points:

1. **Refusal is zero-cost.** No tokens were sent; no tokens are billed. The billing event records a zero-cost refusal so FinOps can count refused requests without a cost gap.
2. **The refusal carries the `obligation_id` and `policy_decision_id`.** The runtime can surface a specific reason to the user ("this message contains a tenant-prohibited data class; please redact and retry") and can route the audit trail back to the policy plane's original decision.
3. **The refusal never carries the matched content.** Not in the response, not in the audit log. Logging the SSN that was matched is exactly the leak the obligation prevented.

## Mask semantics

A `mask` obligation rewrites the request body in place, replacing matched spans with a mask token. The rewritten body goes to the provider.

Two subtle points:

- **The mask token should be stable.** Use a fixed string (e.g. `<email>`) rather than a per-match UUID — otherwise semantically-identical content hashes differently each time, defeating cache coordination.
- **The audit log records the match count and the class, not the matched content.** An operator can see that 3 emails were masked on a request without seeing the original emails.

Some obligations use **tokenisation** rather than masking — the detector replaces a match with a reversible token the agent can refer to, with the mapping held server-side so downstream processing can reify. Tokenisation is a specialised case; most model-gateway redaction uses plain masking.

## Instrumentation

Minimum metrics the gateway emits for this chapter:

- **`redaction_match_count{class}`** — number of detector matches per class per window. Trending this detects a tenant whose data shape changed.
- **`redaction_refusal_count{tenant, class}`** — requests refused outright. A tenant seeing a lot of refusals is a signal — malicious injection, misconfigured agent, or an unreasonable policy.
- **`schema_validation_failure_rate{model_identifier, schema_ref}`** — fraction of responses that failed schema validation. A rising value is a model capability regression or a schema change.
- **`schema_rescue_success_rate{model_identifier, schema_ref}`** — fraction of rescue retries that succeeded. If this is low, rescue is noise; if high, rescue is doing real work.

Every refusal and every schema failure is a span in the trace with the full obligation / rescue context.

## Anti-patterns

- **Running redaction after cache-key composition.** Two requests that differ only in an email address collide in the cache namespace with different prefix hashes, and the cache goes cold.
- **Logging matched content.** The purpose of redaction is to prevent that content from going anywhere, including the log. Match class and count, nothing else.
- **Per-agent schema validators.** Every agent reimplements, each slightly differently, and schema regressions become archaeology.
- **No reinforcement, infinite rescue loops, or multiple rescues.** Each is wrong in different ways; a single reinforcement is the right policy.
- **Refusing on classes the tenant has not opted into.** A refusal that the application cannot anticipate is a broken contract; obligations are tenant-configured and policy-plane-emitted, not gateway-default.
- **Asymmetric schema enforcement by provider.** Agents cannot know that the OpenAI path enforces strictly while the Anthropic path enforces advisorily; the gateway normalises so the agent sees the same behaviour regardless of mapping.
- **Masking with rotating tokens.** Breaks cache-key stability; introduces non-determinism.

## Summary

- Schema enforcement and PII redaction belong at the gateway for uniformity, deploy independence, and because the gateway is the one place that owns the egress seam.
- Structured output composes three provider shapes: in-decoder schema enforcement (OpenAI strict mode, Google responseSchema, vLLM guided decoding), JSON-mode without schema (gateway validates), and schema-as-tool-definition (Anthropic). The internal shape the gateway exposes is one field with an on-invalid policy.
- Rescue retries are one-shot with an appended reinforcement message; multi-shot rescue is not a strategy.
- Schemas live in versioned config, referenced by `schema_ref`. The billing event carries the ref.
- PII redaction runs as a sequence of detectors driven by obligations the policy plane emits. Three detector tiers: pattern, NER, LLM-based.
- Redaction must run before cache-key composition and before routing. A `refuse` is a zero-cost terminal response with the obligation id for audit; a `mask` uses a stable token to preserve cache-key stability.
- Cache + redaction interact: a well-authored agent keeps user-specific content out of the stable prefix so redaction does not shred cache hit rates.
- Instrument match counts, refusal counts, schema failure rate, and rescue success rate. Log match class and count only — never matched content.

The next chapter closes the module with the build-vs-buy call across the current gateway ecosystem — LiteLLM, Portkey, Envoy AI Gateway, Kong AI Gateway, Cloudflare AI Gateway — and the handoff to `ai-infra-inference-engineer` for the self-hosted substrate the gateway may be routing to.
