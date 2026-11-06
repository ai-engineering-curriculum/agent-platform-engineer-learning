# Compliance Audit Evidence for Tool Invocations

An agent called a tool is not an audit record. "At `2026-11-06T14:22:11Z`, user `alice@acme.com` in tenant `acme`, in session `s_abc123` of agent `cx-triage@3.1.0`, invoked tool `erp.lookup_invoice` (server `erp-mcp@2.3.1`, schema hash `sha256:…`) with scope `erp.read_invoice` authorised by policy decision `p_def456`, with inputs hashed `sha256:…` and outputs hashed `sha256:…`, served by vendor route `self-hosted`, trace id `t_ghi789`, gateway request id `r_jkl012`, completed with outcome `success`" is an audit record.

This chapter is the specification for producing the second kind. It is the thing auditors read; it is the thing incident responders pivot on; it is the thing the EU AI Act Article 72 post-market monitoring obligation lives inside. The other chapters of this module are preconditions for this one.

## Who reads these records, and why

- **External auditors (SOC 2, ISO 27001, regulatory).** Sample-test "show me every invocation of a high-risk tool in Q3." Need immutability, retention, chain of custody, redaction of PII without destroying evidential value.
- **EU AI Act Article 72 obligations.** Post-market monitoring for high-risk AI systems requires an evidence trail of outputs, incidents, and corrective actions. Your tool-invocation records are one strand of the evidence pack project-403 assembles. <!-- needs-research: confirm the current scope of Article 72 record-keeping requirements in effect as of your audit cycle; delegated acts have been evolving. -->
- **Incident responders.** "A token was leaked; what did it touch?" or "A tool returned bad data; who saw it and how?" They query by principal, by tool, by time, by trace.
- **Product teams.** "Our agent is misbehaving; why?" They query by session, cross-reference with traces (mod-408) and replays (mod-407).
- **Platform governance.** "Is scope X being used? By whom? Should we deprecate it?" They query aggregates.

Design the record for the hardest reader (the external auditor), and the others get served for free.

## The record schema

```
ToolInvocationAudit {
  # Identity
  record_id             uuid          # unique per record, server-assigned, never reused
  record_version        int           # schema version of this record type
  record_sha256         hex           # self-hash computed after assembly, used for tamper-evidence

  # Timing
  started_at            timestamp     # UTC, ISO-8601 with nanosecond precision where available
  completed_at          timestamp
  duration_ms           int

  # Principals
  user_id               string?       # null for service-initiated calls
  user_display          string?       # for audit UX; not load-bearing
  tenant_id             string
  agent_id              string        # the agent definition (template)
  agent_version         semver
  session_id            string        # ties to the runtime plane's session object (mod-402)

  # What was called
  server_id             string        # registry's slug
  server_version        semver
  tool_name             string
  tool_schema_sha256    hex           # content hash of the pinned schema
  transport             enum<stdio, streamable_http, websocket>
  route_class           enum<self_hosted, vendor, auth_wrapper>
  route_detail          string        # endpoint identifier (not URL in audit; URL may leak)

  # Authorization
  scope                 list<string>  # scopes used on this call
  authorization_server  string?       # issuer URL
  access_token_id       hex?          # hash of the access token or 'jti' claim; never the token itself
  audience_claim        string?       # aud value observed on the token
  consent_record_id     string?       # the consent record this token descends from

  # Policy
  policy_decision_id    string        # mod-405 decision id for allow/deny/step-up
  policy_decision       enum<allow, step_up, allow_with_constraints>
  policy_constraints    JSON?         # e.g. ABAC filters applied

  # Payloads
  input_mode            enum<raw, hashed, redacted>
  input_payload         JSON?         # raw JSON if input_mode=raw; may be very large
  input_sha256          hex           # always present
  input_pii_class       enum<none, low, medium, high>
  output_mode           enum<raw, hashed, redacted>
  output_payload        JSON?
  output_sha256         hex
  output_pii_class      enum<none, low, medium, high>

  # Outcome
  outcome               enum<success, user_error, tool_error, policy_denied, timeout, cancelled>
  error_class           string?       # structured error code
  error_message_sha256  hex?          # hash; raw message under raw-mode only

  # Observability linkage
  trace_id              hex
  span_id               hex
  parent_span_id        hex?
  gateway_request_id    string
  idempotency_key       string?

  # Vendor (when applicable)
  vendor_name           string?
  vendor_record_id      string?       # the vendor's own id, for cross-reference

  # Cost (where known)
  tokens_in             int?
  tokens_out            int?
  tool_cost_currency    string?
  tool_cost_amount      decimal?

  # Lineage
  supersedes_record_id  uuid?         # for records corrected after the fact
  correction_reason     string?
}
```

Required-fields vs. optional reflects what the gateway can guarantee. `user_id` is nullable because service-initiated calls have no user; `access_token_id` is nullable when the server is stdio with no OAuth; `vendor_*` fields are populated only for tier-2 routes. Everything else is required.

Exercise expectations and any downstream consumers (project-403's evidence pack) read this schema; stabilise it under `record_version` and migrate explicitly.

## What goes in the payload field, and what does not

This is the single most important decision in the record design, because it trades off evidence completeness against privacy and data-minimisation obligations.

Three modes per payload (input and output), picked by policy at write time:

- **`raw`.** The full JSON is stored. Evidence is maximal. Usable for sessions that handle no sensitive data (e.g. public-information lookups). Default off for anything above T0 risk.
- **`hashed`.** Only the SHA-256 of the canonical-form JSON is stored. Reproducibility is possible if the caller still has the input bytes; privacy is maximal. Default for T1–T2.
- **`redacted`.** A policy-defined transformation runs: PII is replaced with typed placeholders (`"<EMAIL>"`, `"<PHONE>"`, `"<NAME>"`), remaining structure is preserved. The redactor is either rule-based (regex), ML-based (Presidio-shaped — a preview of mod-411 safety plane), or both. Evidence is moderate; privacy depends on redactor quality.

The record always carries the SHA-256 of the pre-redaction payload. This means an auditor can prove "the agent saw X" if the caller still has the original bytes (usually from the replay store, mod-407), without the audit record itself retaining the sensitive data.

The `*_pii_class` field is set by the same classifier that drives redaction. It is itself part of the audit record (how PII was handled matters, not just whether it was there).

## Immutability and tamper-evidence

External auditors need to trust the record was not edited after the fact. Three layers of immutability, from cheapest to strongest:

1. **Append-only storage.** The audit stream writes to a store that forbids updates (object storage with Object Lock, an append-only database table with `REVOKE UPDATE`, a transaction log in a database with no API for mutation). Already catches accidental overwrites.
2. **Content hash and chain.** Each record has `record_sha256` computed after assembly; records are written in sequence and each record's hash incorporates the previous record's hash (hash chain). Tampering with any record breaks the chain at that point forward. An external witness (a daily exported hash to a separate system) proves the state of the chain at a point in time.
3. **Cryptographic signatures.** For high-assurance records (T3 tools, regulated data), each record is signed with an audit-signing key held by the audit service (not the gateway). The signing key is rotated on a schedule; the public keys are published so auditors can verify independently.

Pick a layer appropriate to the risk tier the records cover. Most platforms land at layer 2 for everything, with layer 3 for a classified subset.

## Retention, redaction, and the right-to-erasure tension

Audit records are not the place for data-minimisation-by-deletion. They are the place for data-minimisation-by-design (hashing and redaction at write time) and lawful retention (policy-driven per record class).

Two realistic retention shapes:

- **Short-horizon (operational).** 90–180 days of records queryable for incident response and on-call. Stored hot; indexed by every principal and every resource.
- **Long-horizon (compliance).** 7 years (or the regulatory retention period) of records queryable for audit. Stored colder; indexed by the subset of fields auditors query.

**Right-to-erasure requests.** When a user invokes GDPR Art. 17 or an equivalent right, the gateway must have a response. The response is rarely "delete the audit record" (because the lawful-basis-for-retention of an audit record usually trumps); it is "show the user what we hold, redact what we lawfully may, keep the structural record." The redacted-payload mode is what makes this feasible: the record keeps the fact that an invocation occurred (with principals and times) while the payload is already a hash.

Document the retention policy per `(record_version, pii_class, risk_tier)` triple. The policy is the registry entry's responsibility to declare (chapter 03); the audit pipeline enforces it.

## Linking to traces, replays, and the policy plane

The record is not a standalone artefact; it is a node in a graph the platform already builds.

- **Trace id → OTel trace (mod-408).** Every record carries `trace_id` and `span_id`; the audit record and the OTel span are the same event in two shapes. An investigator pivots between the audit query (who called what) and the span view (what else happened in the same session) seamlessly.
- **Session id → session object (mod-402).** The session has its full history, cost ledger, and checkpoint; the audit record is one facet of one tool call inside that session.
- **`tool_schema_sha256` → registry (chapter 03).** The exact schema an agent saw at the moment of invocation is retrievable; the audit answers "did the schema change between agent saw it and tool was called" (no, because the gateway enforces chapter 01 obligation 2).
- **`policy_decision_id` → policy plane (mod-405).** The reason the call was allowed (or step-upped) is in mod-405's decision log; the audit record is a cross-reference.
- **`replay_bundle_id` (optional field) → replay store (mod-407).** If the session is in the replay store, the full input/output bytes are recoverable from there; the audit record points at them without duplicating them.

A record that cannot be pivoted into trace + session + schema + policy + replay is not an audit record; it is a log line.

## Query shapes to design for

Think about the queries auditors and investigators will actually run; make them cheap.

- **By principal.** "All tool calls by user U in [T1, T2]." Index on `(user_id, started_at)`.
- **By tool.** "All invocations of tool `erp.update_invoice` in [T1, T2]." Index on `(server_id, tool_name, started_at)`.
- **By scope.** "Every session that exercised scope `files.write`." Index on `(scope, started_at)` (multi-value scope requires an inverted index).
- **By policy outcome.** "Every `step_up` decision this week." Index on `(policy_decision, started_at)`.
- **By trace.** "Audit records for trace T." Index on `trace_id`.
- **By tenant for compliance export.** Range scan on `(tenant_id, started_at)` ordered.

These queries also drive the storage shape. A document store (opaque payload JSON, structured headers, separate indexes) works. A columnar store (BigQuery, Snowflake, ClickHouse) works if you denormalise. A hybrid (headers in a fast store, payloads in object storage) is common at scale. Avoid designing an audit store that cannot run "all records in Q3 for tenant X" in a reasonable time window; that is the auditor's first query.

## PII handling at write time

The gateway composes a small redaction pipeline:

1. **Classify.** Inspect the input payload (and later the output); assign a `pii_class` using a shared classifier library. Rule-based for structured fields (email regex, phone regex, named-entity for `name`/`address` keys), ML-based for free text (Presidio or equivalent, preview of mod-411). The classifier emits spans: offsets and types.
2. **Decide mode.** Policy rule, keyed on `(tool_name, risk_tier, pii_class, tenant_policy)`, picks `raw` / `hashed` / `redacted`.
3. **Apply.** If `redacted`, rewrite the payload by replacing the spans; if `hashed`, compute the hash, drop the bytes.
4. **Write.** The record carries the resulting payload (if any), the SHA of the pre-redaction payload, the pii_class, and the redactor version.

Storing the redactor version matters: if a classifier false-negatives a record, you need to know which records that version wrote so you can reclassify (or at least flag) the affected set.

## Operational concerns

- **Write path reliability.** Audit writes cannot be best-effort. The gateway's tool-call path blocks on audit write (or writes to a durable write-ahead log that it can prove will be committed). "The tool call succeeded but we lost the audit record" is a compliance-breaking outcome; design for at-least-once audit with idempotency keyed on `record_id`.
- **Write path throughput.** Tool-call volume is high. Use a queue or stream between the gateway and the audit store; size it against peak concurrent calls times retention-window read volume.
- **Backfill.** New fields added to the record schema require a migration plan. Backfill the old records to the new schema, or carry `record_version` and query across versions. Pick one; document it.
- **Cross-region and cross-account.** For multi-region deployments, write audit locally and replicate; a region that lost connectivity should still record (then catch up). For multi-account (per-tenant) deployments, run one audit pipeline per boundary — cross-account trust is a careful decision.

## The Article 72 evidence pack

Project-403 of this track produces an EU AI Act Article 72 evidence slice. The pieces from this chapter that feed that pack:

- **Inventory.** The registry (chapter 03) answers "what tools are in scope, at what risk tier, under what approvals."
- **Operational logs.** This record shape, aggregated per tool and per tenant.
- **Incident log.** The audit records where `outcome` is `policy_denied` or `tool_error` are a starting point; mod-410 adds incident records.
- **Corrective actions.** When a tool is deprecated or sunset in response to behaviour, the governance record (chapter 03) links to the audit query that justified the action.

Design the record to fit this export; project-403 reuses it rather than re-plumbing.

## Summary

- An audit record is a structured tuple, not a log line. The schema in this chapter is the thing the gateway produces, every tool call, every time.
- Payloads carry one of three modes (raw, hashed, redacted). Hashing preserves provenance without retaining content; redaction keeps structure visible; raw is reserved for low-risk tools.
- Immutability has layers (append-only, hash chain, signature). Pick appropriate to risk tier; most records live on the hash-chain layer.
- Retention has two horizons (operational, compliance). Right-to-erasure is handled by redaction-by-design plus retention policy, not by deletion.
- The record is a node in a graph (trace, session, schema, policy decision, replay). Pivoting across all five is the defining audit capability.
- Design for the auditor's query first; the investigator and the product team queries fall out for free.

That is the module. The registry, the auth path, the conformance pipeline, the auth-wrapping proxy, the vendor topology, and the audit record are the six substrates every subsequent module in this track assumes. Project-401 and project-402 stress-test them against a real workload; the exercises below take you to the point where each one is yours to defend.
