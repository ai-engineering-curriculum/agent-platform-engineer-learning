# Sandbox Events as Compliance Evidence

Everything in chapters 02–05 ends with "the plane emits an event." This is the chapter that explains what those events are, how they land in the platform's compliance evidence stream, and what external auditors and incident responders actually do with them. Without this chapter, the capability model, the quota ladder, and the cold-start work are claims; with it, they are falsifiable statements the plane can be asked about.

The target shape is the same one mod-403 chapter 07 established for tool invocations: a per-event, immutable, content-addressable, attributable record that other planes can cross-reference. This chapter adds the sandbox-specific record types and the sandbox-specific audit queries.

## The event catalogue

Six event types cover the sandbox plane's obligations. Every event carries a common envelope and a type-specific payload.

### Common envelope

```yaml
event:
  id: evt-01JXAB...                     # ULID; append-only
  type: sandbox.resource_cap_hit        # see catalogue below
  occurred_at: 2026-11-06T10:14:03.248Z
  received_at: 2026-11-06T10:14:03.299Z
  provider: firecracker-primary         # or gvisor-high-density, e2b, modal, ...
  sandbox_id: sbx-01JXAB...
  session_id: s-01JX9N...               # from the runtime (mod-402)
  tenant_id: t-acme
  user_id: alice@acme.com               # from the policy plane
  tool: calendar-google
  tool_version: 2.3.1
  trace_id: 7c1d...                     # OTel trace id; cross-ref mod-408
  policy_decision_id: pd-01JXAB...      # cross-ref mod-405
  capability_bundle_hash: sha256:…      # hash of the bundle this call ran under
  payload: { … type-specific … }
```

Four fields make the envelope load-bearing:

- **`trace_id`** — correlates the sandbox event with the runtime's trace (mod-408), the tool-gateway's call record (mod-403 chapter 07), and the eval replay (mod-407).
- **`policy_decision_id`** — the exact decision from mod-405 that admitted the call. "The policy plane allowed this" is checkable retroactively.
- **`capability_bundle_hash`** — the hash of the enumerable capability bundle (chapter 02) the sandbox ran under. An auditor asking "what was this sandbox allowed to do" gets a single hash to look up.
- **`received_at`** — the ingestion timestamp into the evidence pipeline, separate from `occurred_at`. The lag between them is an SLI; a growing lag is a compliance signal.

### The six event types

| Type | When it fires | Minimum payload fields |
|---|---|---|
| `sandbox.created` | A sandbox is admitted and is about to accept its first tool call | `substrate_tier`, `provider`, `snapshot_id` (if restored), `cold_start_ms` with breakdown |
| `sandbox.invocation` | A tool call runs inside the sandbox | `tool_call_id`, `duration_ms`, `exit_code`, `bytes_in`, `bytes_out`, `artifact_hashes` |
| `sandbox.syscall_violation` | A syscall denied by seccomp (or an equivalent on WASM) | `syscall_name`, `action` (`ERRNO`/`KILL_PROCESS`), `pid_in_guest` |
| `sandbox.egress_denied` | An outbound network attempt denied by policy | `destination` (fqdn/cidr), `port`, `protocol`, `enforcement_point` (nftables/eBPF/proxy) |
| `sandbox.resource_cap_hit` | A quota dimension hit (CPU time, memory, disk, pids, bandwidth) | `dimension`, `configured_value`, `observed_value`, `action` (`throttle`/`kill`) |
| `sandbox.lifecycle` | Transitions outside normal start/call/stop — suspension, eviction, provider-degraded routing, hard-kill | `transition`, `cause_code`, `reclaimed_resources`, `ladder_stage` (from chapter 3's eviction ladder) |

Two specialised subtypes of `sandbox.lifecycle` deserve explicit catalogue entries because incident responders look for them by name:

- `sandbox.lifecycle.oom_kill` — memory controller killed the process; separate because OOMs are the single most-asked question during incidents.
- `sandbox.lifecycle.eviction` — chapter 3's eviction ladder fired, with `cause_code` and the full ladder stage trail (`soft_cancel_initiated → hard_kill → reclaimed`).

Keep the catalogue short and closed. A new event type is a reviewed change; teams do not smuggle a seventh type in. If existing types do not answer a question, the fix is to extend a payload with reviewed fields, not to add a type.

## Attribution — the fields auditors actually ask about

Auditors and incident responders ask a surprisingly small set of questions. If your records answer these at a single query, your plane is doing its job:

1. **"Who did this?"** → `user_id`, `tenant_id`, `session_id`.
2. **"Who approved it?"** → `policy_decision_id` resolves to the mod-405 record.
3. **"What was it allowed to do?"** → `capability_bundle_hash` resolves to the bundle.
4. **"What did it actually do?"** → `sandbox.invocation` records plus every child event under the same `trace_id`.
5. **"Did any of its attempts fail policy?"** → `sandbox.syscall_violation`, `sandbox.egress_denied`, `sandbox.resource_cap_hit` records.
6. **"Did it leave anything behind?"** → the provenance chain of any `rw-persist` output and the destruction record for the ephemeral paths.
7. **"When did the session release the sandbox?"** → `sandbox.lifecycle.eviction` or `sandbox.destroyed`.
8. **"How long did the evidence take to show up?"** → `received_at - occurred_at` across every event in the window.

Every one of these should resolve from one query against the evidence store keyed on either `session_id` or `trace_id`. If an auditor has to join against three systems, your scheme is not evidence — it is log data.

## Shape of the record store

Three practical requirements shape the storage layer.

### Immutability

Records are **append-only**. No in-place update is possible by design, not just by convention. Three common substrates:

- **Object storage with object lock.** Write each record (or batch) to S3 / GCS / Azure Blob with the object-lock / bucket-lock primitive engaged in compliance mode for the retention horizon. (S3 Object Lock, GCS Bucket Lock, Azure Blob immutable-storage.)
- **Append-only ledger / WORM database.** Dedicated primitives — AWS QLDB (deprecated at some providers, verify) <!-- needs-research: confirm current status of AWS QLDB and equivalent WORM ledger products at your reading time. --> — or a Kafka-shaped log that ingests the records with compacted retention.
- **Hash-chained batches.** Each batch's digest incorporates the previous batch's digest ([RFC 6920](https://datatracker.ietf.org/doc/html/rfc6920)-style naming). A tampered batch breaks the chain and the integrity check fails on read.

Use at least two of these in combination: the object lock stops accidental deletion, the hash chain detects non-accidental tampering.

### Content-addressability

The capability bundle (chapter 02), the syscall profile, the snapshot id, and the artefact hashes are content-addressable references. The record stores the hash; the full bundle lives in a separate store addressed by that hash. Two benefits:

- **Deduplication.** The same capability bundle used across thousands of sandboxes stores once.
- **Immutable cross-reference.** A record referencing `capability_bundle_hash: sha256:abcd…` cannot drift relative to what the bundle was at that time.

### Partitioning for the questions auditors actually ask

Primary index: `session_id`. Secondary: `tenant_id`, `tool`, `event_type`, `occurred_at`. A query like "all `sandbox.egress_denied` events for tenant `t-acme` between 10:00 and 11:00 yesterday" must be cheap. The internal shape depends on your storage substrate (a partitioned table, a time-series store, an OpenSearch index); the shape of the queries is the thing you design against.

## Retention horizons

Retention is driven by the regulatory frame you sit inside, not a technical preference. Common horizons:

- **Incident response.** 90 days of searchable history.
- **Audit (SOC 2, ISO 27001).** 1 year typically, 3 years in some scopes. Check your current controls with your assurance team.
- **Regulatory (EU AI Act Article 72, sector-specific rules).** 6 or 10 years depending on the system's risk classification. The AI Act's post-market monitoring record horizon is at the long end of the range. <!-- needs-research: confirm the current record-keeping horizon required by EU AI Act Article 72 and any associated delegated acts at your audit cycle. See the regulation text: https://eur-lex.europa.eu/eli/reg/2024/1689/oj -->
- **Internal governance.** Driven by your platform's own promises.

The horizons are a per-tenant policy, not a fleet default. A tenant subject to EU AI Act high-risk classification has a longer horizon than one running internal-only workloads. The record store partitions by horizon so that older records age out of tenants that do not need them while being preserved for tenants that do.

## Right-to-erasure vs. append-only immutability

A user invoking GDPR Article 17 right-to-erasure reaches into every immutable store and asks: delete my data. You cannot delete from an object-locked store. The right pattern is the same as mod-403 chapter 07:

- **Separate the payload from the envelope.** The envelope stays (needed for integrity of the hash chain); the payload — inputs and outputs that could contain personal data — lives in a separate store, referenced by hash.
- **Erasure is a tombstone on the payload.** The payload store supports delete; the record keeps the hash and now points at a tombstone. The hash chain remains valid; the audit trail still proves "this sandbox ran at this time under this policy decision"; the sensitive content is no longer retrievable.
- **Negotiate the scope with legal.** Some regulators may demand the envelope itself also be removable in specific scopes; this is a legal call, not an engineering one.

The point: design the payload-envelope separation up-front. Retrofitting erasure onto a monolithic record format is a project.

## PII in payloads

Sandbox inputs and outputs may contain personal data — customer emails, internal documents, model-generated text that quoted something sensitive. Three postures:

1. **Raw payload retained.** Highest utility for incident response and eval replay (mod-407), highest privacy risk. Use only for tenants whose data-processing agreements explicitly allow it.
2. **Hashed payload with length and schema.** The record retains a hash of the payload, its byte length, and a shape summary (e.g., schema id). Enough to detect replay, enough to prove "this specific payload was processed," useless for reading the content.
3. **Redacted payload.** Payload passes through a PII-detection + redaction step ([Presidio](https://microsoft.github.io/presidio/), cloud DLP) before storage. Readable, but sensitive spans are masked.

The per-tenant / per-tool payload mode is a policy decision carried on the registry entry. Chapter 02's capability bundle references the payload mode so the sandbox plane knows how to emit events before they reach the evidence store.

## Pipeline shape

A workable ingest pipeline has four stages:

```
[sandbox plane] ──► [buffer / ingest] ──► [enrich / redact] ──► [immutable store]
                                                                    │
                                                                    ├──► hash-chain writer
                                                                    ├──► OTel exporter (mod-408)
                                                                    └──► indexer (search layer)
```

Stage roles:

- **Buffer / ingest.** Durable queue (Kafka, Pub/Sub, Kinesis). Records survive a downstream outage; backpressure to the sandbox plane is bounded.
- **Enrich / redact.** Attach `received_at`, fetch and attach `policy_decision_id` details if missing, pass payloads through the per-tenant PII mode. Emit `received_at - occurred_at` lag metrics.
- **Immutable store.** Write to object-locked storage; emit the hash of each batch; chain the hash with the previous batch.
- **Indexer.** Build the search layer auditors and incident responders query.

Every stage has its own SLI: ingest lag, enrichment lag, write-success rate, indexer freshness. Chapter 05's `sandbox_cold_start_breakdown_lag` and this chapter's `evidence_lag` share a shape.

## Cross-plane linkage

The evidence stream is useful in proportion to how well it joins to the rest of the platform's telemetry:

- **Traces (mod-408).** Every event carries the OTel `trace_id`; the span tree is the first thing an incident responder reads.
- **Tool-gateway audit (mod-403 chapter 07).** The `tool_call_id` links the sandbox invocation to the gateway's per-tool record. A sandbox-side view of a call and a gateway-side view of the same call sit next to each other.
- **Eval replay store (mod-407).** The input and output hashes let the eval platform re-constitute a call for regression testing or incident reproduction.
- **Policy-plane decisions (mod-405).** The `policy_decision_id` is the join key; policy decisions are stored with the same immutability properties.
- **Runtime session store (mod-402).** The `session_id` is the join key; session state, cost ledger, and checkpoint history sit next to sandbox evidence.

An on-call engineer pivots across these from a single query. A platform that forces them to re-key by hand is a platform whose on-call is slower than it needs to be.

## Example: tracing one abuse attempt end-to-end

To make the catalogue concrete, trace one specific scenario — a tool call that attempted to exfiltrate credentials — through the evidence stream.

1. **`sandbox.created`** fires. Fields include the `capability_bundle_hash`, the chosen `substrate_tier: firecracker-hardened`, the `snapshot_id`, the `cold_start_ms` with its stage breakdown.
2. **`sandbox.invocation`** fires once the tool call starts.
3. Inside the sandbox, the model-generated code tries to read `/etc/ssh/ssh_host_rsa_key`. The file is not present (rootfs is read-only and the path is not in the image); `open()` returns `ENOENT`. This is *not* a policy violation; no event fires at the sandbox plane.
4. The code tries to `connect()` to `169.254.169.254:80`. Egress policy denies; **`sandbox.egress_denied`** fires with `destination: 169.254.169.254`, `enforcement_point: nftables`.
5. The code forks aggressively, trying to exhaust pids. The `pids.max` controller fires `ENOMEM` on `clone`; **`sandbox.resource_cap_hit`** fires with `dimension: pids`, `configured_value: 128`, `observed_value: 128`, `action: throttle`.
6. The tool invocation completes with a non-zero exit code; **`sandbox.invocation`** closes with `exit_code: 1` and the output bytes.
7. The runtime signals session complete; **`sandbox.destroyed`** fires.
8. The evidence pipeline enriches each event with the `policy_decision_id` (already present), attaches `received_at`, passes through PII redaction on the payload, and writes the batch. The batch's hash chains to the previous batch.

A query against the evidence store by `session_id` returns the full sequence. Three independent policy-enforced events (egress deny, pid-cap hit) in a single session is a strong signal the safety plane (mod-411) will act on; the audit record makes the signal actionable instead of gut-feel.

## Summary

- Six event types cover the plane's obligations: `created`, `invocation`, `syscall_violation`, `egress_denied`, `resource_cap_hit`, `lifecycle` (with `oom_kill` and `eviction` subtypes).
- Every event carries an envelope with the four cross-plane join keys — `trace_id`, `policy_decision_id`, `capability_bundle_hash`, and `received_at`.
- The record store is immutable (object lock + hash-chained batches), content-addressable (bundles, snapshots, payloads referenced by hash), and partitioned for the questions auditors ask.
- Retention is per-tenant, driven by the regulatory frame. The payload-envelope separation makes right-to-erasure compatible with append-only immutability.
- PII in payloads has three modes (raw / hashed / redacted) carried on the registry entry.
- The pipeline (buffer → enrich → immutable store → index) has its own SLIs; `received_at - occurred_at` is the honesty number.
- Cross-plane linkage to mod-402 (session), mod-403 (tool audit), mod-405 (policy), mod-407 (replay), and mod-408 (traces) is what makes the evidence stream auditable *and* investigable.

That closes the chapter sequence. The four exercises turn the chapters into artefacts — a threat-model memo, a syscall-and-egress policy, a quota-and-eviction design, and a snapshot cold-start prototype — you keep and reuse in projects 401 and 404 and in later modules (mod-405, mod-407, mod-408, mod-410).
