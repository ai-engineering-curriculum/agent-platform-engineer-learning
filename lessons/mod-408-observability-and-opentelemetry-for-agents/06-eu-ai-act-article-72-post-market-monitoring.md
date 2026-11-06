# EU AI Act Article 72: Post-Market Monitoring from the Trace Pipeline

Chapters 02–05 built an observability plane that answers engineering questions. This chapter turns that same plane into a regulator-facing artefact. The [EU AI Act — Regulation (EU) 2024/1689](https://eur-lex.europa.eu/eli/reg/2024/1689/oj) obliges providers of high-risk AI systems to maintain a post-market monitoring system; [Article 72](https://eur-lex.europa.eu/eli/reg/2024/1689/oj/eng) codifies the obligation. The platform is not usually itself the "provider" under the Act — the applications built on top are — but the platform's traces, logs, and metrics are what the applications' providers draw from to meet their obligations. A platform that cannot produce the monitoring evidence slice is a platform whose tenants cannot ship high-risk use cases.

This chapter walks what Article 72 actually requires, which pieces of the plane's existing telemetry already cover each requirement, and the specific export / view / documentation surface the platform adds on top so an auditor's question has a one-command answer. The goal is not a legal opinion — that is counsel's — but an engineering frame for "which rows of telemetry under our control satisfy which clause".

> **Scope caveat.** This chapter walks the mechanical shape only. Legal interpretation, applicability to a specific product, the definition of "high-risk" under [Article 6 and Annex III](https://eur-lex.europa.eu/eli/reg/2024/1689/oj/eng), serious-incident reporting under [Article 73](https://eur-lex.europa.eu/eli/reg/2024/1689/oj/eng), the role of the forthcoming post-market monitoring implementing act, and the interaction with the [GPAI code of practice](https://digital-strategy.ec.europa.eu/en/policies/general-purpose-ai-code-practice) are counsel's domain, not platform engineering's. <!-- needs-research: the Commission is expected to publish implementing acts under Article 72(3) specifying template and operational details for the post-market monitoring plan; the detailed format changes if and when those land. Re-verify against the latest EUR-Lex text and against counsel before shipping an evidence export. -->

## What Article 72 asks for

Article 72 of the AI Act, in its current text, requires providers of high-risk AI systems to:

- **Establish and document a post-market monitoring system** proportionate to the nature of the technology and the risks of the system.
- **Actively and systematically collect, document, and analyse relevant data** on the performance of the system throughout its lifetime. The data is primarily data provided by the deployer (user / operator) but may be collected directly by the provider where appropriate.
- **Evaluate continuous compliance** with the Act's requirements on high-risk systems (data governance, risk management, technical documentation, record-keeping, transparency, human oversight, accuracy/robustness/cybersecurity — Articles 8–15).
- **Base the system on a post-market monitoring plan** that is part of the technical documentation ([Article 11](https://eur-lex.europa.eu/eli/reg/2024/1689/oj/eng) and [Annex IV](https://eur-lex.europa.eu/eli/reg/2024/1689/oj/eng) carry the detailed scope).

[Article 73](https://eur-lex.europa.eu/eli/reg/2024/1689/oj/eng) extends Article 72 with an explicit obligation: **serious incidents** are reported to the competent market-surveillance authority within strict deadlines (typically 15 days, shorter for incidents involving widespread or irreversible harm). Serious-incident evidence is partially sourced from the same monitoring pipeline.

The platform's job, mechanically, is to make the monitoring plan implementable without bespoke per-application pipelines. A platform tenant building a high-risk agent product should be able to:

1. **Query** the platform for the monitoring data their plan references, by tenant / use case / time window.
2. **Export** the query result in a documented, version-stable format.
3. **Retain** the underlying evidence long enough to satisfy the plan's data-retention commitments.
4. **Produce**, on demand, a serious-incident evidence package for a specific trace / session.
5. **Document** what the platform guarantees and what the provider must do themselves.

Each of (1)–(5) is a view over the trace pipeline the plane already runs.

## The five pieces, mapped

Map the Article 72 asks onto pieces of the plane.

### 1. Performance and interaction data

Chapters 02–04 already produce this. For every agent request, the platform has:

- Which model answered (`gen_ai.response.model`).
- Which tools were called (`gen_ai.tool.name`, `gen_ai.tool.call.id` on tool-execution spans).
- How long each operation took (`gen_ai.client.operation.duration`).
- Token / cost accounting (`gen_ai.usage.*`, `platform.gen_ai.cost_usd`).
- Policy decisions (`platform.policy_decision_id`, mod-405).
- Guardrail verdicts (chapter 02 events, mod-411).
- The end-to-end trace that ties it together (chapter 03).
- Full-fidelity prompt / completion content in the tenant-partitioned content store (chapter 04).

The platform's added surface: a **time-window / tenant / use-case query** that aggregates the above. For a tenant "export all agent interactions for use case `writer-v1-high-risk` in 2026-Q3", the query returns a structured record per interaction with the attributes above and a pointer to the full content.

### 2. Compliance-relevant signals

Article 72 wants continuous compliance evidence against the Act's high-risk obligations (Articles 8–15). Several of those obligations are naturally observable on the trace pipeline:

- **Accuracy, robustness, cybersecurity ([Article 15](https://eur-lex.europa.eu/eli/reg/2024/1689/oj/eng)).** The eval plane's regression gate (mod-407 chapter 05) produces an EvalRecord per fixture per deploy. Published to the backend as a metric; linked from the trace where a replay touched the production session.
- **Human oversight ([Article 14](https://eur-lex.europa.eu/eli/reg/2024/1689/oj/eng)).** Policy decisions where a human escalation path is invoked produce a span event; the aggregate is a count.
- **Transparency and provision of information ([Article 13](https://eur-lex.europa.eu/eli/reg/2024/1689/oj/eng)).** The system-prompt disclosure (identifying the AI system to the user) is a tool-level concern; the span attribute `platform.transparency.disclosure_applied` can be captured on the agent.turn span by the paved-road SDK.
- **Record-keeping ([Article 12](https://eur-lex.europa.eu/eli/reg/2024/1689/oj/eng)).** The trace pipeline itself is the record-keeping artefact. Retention windows per tenant / per use case live on the content-store bucket's lifecycle rules. The retention policy is a per-tenant platform setting; a tenant under high-risk obligations configures the window to match their plan.

The platform's added surface: a **compliance-signal dashboard** that aggregates these per tenant per use case, with the per-signal metric tied back to the Article it supports. The dashboard is a Grafana board (chapter 04's Grafana + Prometheus + Tempo stack); each panel links to the trace query that drilldowns to raw evidence.

### 3. Retention and integrity

Record-keeping under [Article 12](https://eur-lex.europa.eu/eli/reg/2024/1689/oj/eng) implies evidence that is retained and that cannot be mutated retroactively. The platform's existing substrate:

- **Content-store retention policies.** Per-tenant, per-use-case lifecycle rules on the object store (S3 object lock, GCS retention policies, equivalents). Writes are append-only; mutation is forbidden by bucket policy.
- **Trace retention in the backend.** Backend-specific. Datadog / Langfuse / Phoenix each have their own retention knobs; chapter 04's collector tier is the thing that lets the platform mirror the tenant's content-store retention to the backend's retention.
- **Integrity signal.** Chapter 04's content-store write is tagged with a SHA-256 over the content; a reviewer re-fetching the content verifies the hash against a separately-recorded value. The hash itself is written to a tamper-evident log (an append-only database table, or [Rekor](https://docs.sigstore.dev/rekor/overview/)-style transparency log) that a regulator can read.

The platform's added surface: a **content-provenance record** that, given a trace id, returns the content-store URI, the content hash, the write timestamp, the retention window, and the current lifecycle status. A regulator asking "was this record retained in accordance with the plan" reads the record.

### 4. Serious-incident evidence package

[Article 73](https://eur-lex.europa.eu/eli/reg/2024/1689/oj/eng)'s serious-incident notification requires, for a given incident, the provider to describe what happened, when, who was affected, and what the system's role was. The platform's job is to produce, for one or more trace ids, a self-contained evidence package.

The package shape:

```
incident-{incident_id}/
├── manifest.json                   # incident_id, tenant_id, use_case, time_window, hashes
├── traces/
│   ├── {trace_id_1}.jsonl          # full OTLP trace content
│   ├── {trace_id_2}.jsonl
│   └── ...
├── content/
│   ├── {trace_id_1}.session.jsonl  # full prompt / completion content from content store
│   └── ...
├── policy_decisions/
│   └── {decision_id}.json          # policy plane records (mod-405)
├── eval_records/
│   └── {eval_run_id}.json          # any eval replay against these traces (mod-407)
├── provider_mapping.json           # which provider / model served each span
└── README.md                       # human-readable walkthrough
```

Produced by a command the tenant's compliance team runs: `platform-oversight export-incident --tenant acme-financial --time 2026-11-03T14:00/14:30Z --output ./incident-xyz`. Signed with the platform's key; the regulator verifies the signature against a published key.

### 5. The monitoring plan

Article 72 requires the plan itself to be documented. The platform's job is **not** to author the plan — the tenant's provider role does that — but to document **what the platform guarantees** so the tenant's plan can reference it rather than re-inventing it.

The platform publishes, as part of its own technical documentation:

- **A list of the signals it produces** (`gen_ai.*` attributes, `platform.*` extensions, the content store, the eval record stream, the policy decision stream, the guardrail verdict stream).
- **The retention guarantee per tenant tier** (default 30 / 90 / 180 / 365 days; configurable per tenant).
- **The export surface** (the queries, the incident-package command, the signing key, the export file formats).
- **The assurance scope** (what the platform guarantees about the completeness of the pipeline — level-3 synthetic coverage, uptime SLOs, incident-handling procedures).

A tenant writing their monitoring plan references this document and augments it with their application-specific signals (e.g. their domain-specific rubric scores from the eval plane). The reference is the paved road; the augmentation is the stream-aligned-team's own concern.

## The evidence view: a worked example

Walk through an auditor's request. A tenant's provider reports a serious incident: on 2026-11-03T14:22Z, a writer-v1 agent produced content that triggered a regulatory concern. The auditor asks for the evidence.

```sh
platform-oversight export-incident \
  --tenant acme-financial \
  --use-case writer-v1-high-risk \
  --session sess_01HV9... \
  --time 2026-11-03T14:20:00Z/2026-11-03T14:25:00Z \
  --output ./incident-2026-11-03-xyz

# Produces incident-2026-11-03-xyz/ with:
# - 3 traces (the session's three turns)
# - 3 session content files from the content store
# - 7 model-call spans with gen_ai.response.model recorded
# - 2 tool-execution spans with gen_ai.tool.name and policy decision refs
# - 2 policy-decision records (mod-405)
# - 1 guardrail verdict (the one that flagged the interaction)
# - the SHA-256 hashes of each file written to the content store
# - the platform's signing manifest
# - a README summarising what each piece is and which Article it supports
```

The manifest.json carries the Article 72 clause each artefact supports:

```json
{
  "incident_id": "inc-2026-11-03-xyz",
  "generated_at": "2026-11-06T09:14:22Z",
  "platform_version": "2026.11.03",
  "semconv_version": "1.33",
  "artefacts": [
    {
      "kind": "trace",
      "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
      "content_hash": "sha256:5e7f9a...",
      "supports": ["Article 72 — performance data", "Article 12 — record-keeping"]
    },
    {
      "kind": "content",
      "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
      "content_hash": "sha256:bb1a24...",
      "retention_window_days": 365,
      "supports": ["Article 12 — record-keeping"]
    },
    {
      "kind": "policy_decision",
      "decision_id": "pd_01HV9Z...",
      "supports": ["Article 14 — human oversight"]
    },
    {
      "kind": "guardrail_verdict",
      "span_id": "7b3210dcab4511ac",
      "verdict": "flagged",
      "supports": ["Article 15 — accuracy / robustness"]
    }
  ],
  "signature": {
    "algorithm": "ed25519",
    "key_id": "platform-oversight-2026-11",
    "signature": "...",
    "verification_url": "https://platform.internal/oversight/keys/platform-oversight-2026-11.pem"
  }
}
```

The signed manifest is what the regulator verifies. The content (traces, content-store files, policy decisions) is hash-pinned by the manifest. A regulator who receives the package can verify it without the platform's cooperation — that is the point of the signature.

## What the platform does NOT claim to produce

Three categories of thing explicit in the Article 72 / Article 73 landscape that live outside the platform's scope.

- **Risk management documentation ([Article 9](https://eur-lex.europa.eu/eli/reg/2024/1689/oj/eng)).** The provider's risk management system — the identification of risks, the mitigations, the residual-risk assessment — is a document the provider authors. The platform's telemetry feeds into it but is not it.
- **Fundamental rights impact assessment ([Article 27](https://eur-lex.europa.eu/eli/reg/2024/1689/oj/eng)).** For deployers of certain high-risk systems. Not platform scope.
- **Training data governance ([Article 10](https://eur-lex.europa.eu/eli/reg/2024/1689/oj/eng)).** The provider owns training data provenance. The platform's eval dataset plane (mod-407 chapter 03) governs eval data, not training data; the distinction matters.

A tenant's provider role is accountable for producing these. The platform is a dependency that makes some of them easier; it is not an accountable party for them in its own right.

## The periodic review loop

Article 72 implies a review cadence — the monitoring data is collected, analysed, and acted upon over the system's lifetime. The platform's shape:

- **Quarterly tenant review.** Each tenant with a high-risk use case receives a quarterly export summarising their signals: top anomalies, regression trends, incident count, override count on policy decisions, guardrail flag rate. The tenant's compliance function reads the export; their monitoring plan references the format.
- **Annual platform-wide review.** The observability plane publishes an annual report on pipeline health — trace coverage, backend availability, content-store retention compliance, conformance-test pass rates. Read by platform leadership (mod-413) and shared with tenants.
- **Change control on the evidence export.** The export format and the manifest schema are versioned. A change requires one release of soft-mode before the gate requires the new schema. A change is not a mutation of previously-exported manifests — those remain valid at their version.

## Where this connects back to the rest of the track

The Article 72 obligation does not sit in isolation on the platform; it reads from the other planes.

- **Policy plane (mod-405)** supplies the human-oversight signal (`platform.policy_decision_id` with a `requires_human` flag).
- **Model gateway (mod-406)** supplies `gen_ai.response.model` and cost attribution.
- **Evaluation plane (mod-407)** supplies the EvalRecord that satisfies the accuracy / robustness continuous-compliance signal and supplies the replay capability for incident investigation.
- **Safety plane (mod-411)** supplies the guardrail verdict for the content-moderation portion of Article 15.
- **Reliability plane (mod-410)** supplies the SRE operating cadence that defines what counts as a serious incident on the platform side.
- **Paved-road SDKs (mod-412)** package the per-service onboarding — every agent at a tenant goes through the SDK; the SDK is where disclosure attributes land on spans.
- **Platform leadership (mod-413)** owns the governance committee that reviews the tenants' monitoring plans against the platform's documented guarantees.

If the platform is serving tenants with high-risk use cases, every one of those planes contributes to Article 72 evidence. This chapter is the view that pulls the contributions together.

## Summary

- The platform is not usually the "provider" under the EU AI Act, but its tenants are, and their post-market monitoring plans draw on the platform's telemetry.
- Article 72 asks for performance / interaction data, continuous-compliance signals, retained evidence, serious-incident reporting under Article 73, and a documented monitoring plan.
- Chapters 02–05 already produce the signals. The platform adds: a time-window / tenant / use-case query, a compliance-signal dashboard, a content-provenance record with integrity hashes, a signed incident export package, and a published technical document of what the platform guarantees.
- The incident export is a self-contained, signed, hash-pinned package of traces, content, policy decisions, eval records, and guardrail verdicts. The signature is verifiable without platform cooperation.
- What the platform does NOT claim to produce: the provider's risk management documentation, fundamental rights impact assessments, training-data governance. Those are provider-authored artefacts.
- The periodic review loop — quarterly tenant exports, annual platform report, versioned manifest schema — is what makes the evidence slice durable across platform and tenant changes.
- Every other plane in the track contributes a signal into this view; the chapter is the pull-together of the full track's telemetry.

The module closes here. The observability plane is now walkable end-to-end: vocabulary pinned (02), context propagated (03), collectors fanning out to any tenant's chosen backend (04), conformance tests keeping the whole thing honest (05), and the regulator-facing export produced from the same pipeline (06). The exercises build the pieces; the resources page catalogues the primary sources.
