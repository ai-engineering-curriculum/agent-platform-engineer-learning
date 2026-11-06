# OWASP LLM Top 10 and OWASP Agentic AI Threats

Chapter 02 named the engines. This chapter names the threats and maps each onto the hooks the plane runs. Two catalogues do most of the work: the [OWASP Top 10 for LLM Applications](https://genai.owasp.org/llm-top-10/) (the general-purpose LLM-app catalogue, maintained by the OWASP GenAI Security Project) and the [OWASP Agentic AI — Threats and Mitigations](https://genai.owasp.org/resource/agentic-ai-threats-and-mitigations/) catalogue (the agent-specific extension from the same project). The pair is not the whole threat landscape — [MITRE ATLAS](https://atlas.mitre.org/), [NIST AI 100-2](https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.100-2e2023.pdf), and the [NIST AI Risk Management Framework (AI RMF)](https://www.nist.gov/itl/ai-risk-management-framework) overlap substantially — but it is the catalogue most practitioners cite and the one the `ai-risk-engineer` and `agentic-safety-engineer` roles anchor their red-team methodology to. Pin the catalogue, pin the identifiers the plane emits in `SafetyRecord`, and the risk register can group over the fleet.

> **Rev caveat.** OWASP LLM Top 10 is versioned. The published revisions include "2023" and "2025" rolls; categories renumber between revs. The identifiers below use the LLM Top 10 for LLM Applications 2025 rev where the official project page is the source of truth. A production `SafetyRecord` stream pins the exact rev string (e.g., `LLM01:2025`); the mapping file in chapter 02's policy pack is the place the rev lives. <!-- needs-research: confirm the exact string form the project ships (e.g., `LLM01:2025:prompt_injection` vs. `LLM01_prompt_injection@2025`). The forms have varied across community implementations; the official OWASP GenAI Security Project page is authoritative — re-verify before pinning. -->

## The LLM Top 10 for LLM Applications

Ten categories; each is a shape of harm, not a specific attack. The plane's mapping is "which hook catches which category, and what is the primary mitigation". Numbers follow the 2025 rev as published on the [OWASP GenAI Security Project — LLM Top 10](https://genai.owasp.org/llm-top-10/) page.

### LLM01 — Prompt Injection

**What it is.** Untrusted content — a user turn, a retrieved document, a tool result, an image the model can read — instructs the model to perform actions the deployer did not sanction. The direct form is a malicious user turn ("ignore all previous instructions and ..."); the indirect form is an attacker-controlled document the agent is asked to summarise.

**Where the plane catches it.**
- **Input filter.** Primary detection via the content classifier (Llama Guard 3 / Prompt Guard / Lakera Guard) and the known-pattern regex stage. Chapter 02 pattern 1.
- **Input filter on retrieved content.** The retrieval pipeline must route tool results and retrieved documents through the same classifier, not just the user turn. Chapter 04 is the full-chapter treatment of direct and indirect injection.
- **Output validator.** Belt-and-suspenders. A successful injection often surfaces in the model's output ("I will now call `email.send`...") before the tool call fires. The output validator is the second line of defence.
- **Tool-invocation policy hook.** The last line of defence. Even if the input filter missed and the output validator missed, the tool-invocation hook can refuse the resulting dangerous call pattern.

**SafetyRecord category identifier.** `LLM01:prompt_injection` (direct) and `LLM01:indirect_prompt_injection` (indirect — the plane distinguishes these because the mitigation differs).

### LLM02 — Sensitive Information Disclosure

**What it is.** The model reveals information it should not — training-data memorisation, retrieved PII, tenant-A data to tenant-B, operator system-prompt content, credentials or API keys the agent handled in context.

**Where the plane catches it.**
- **Input filter — PII redaction.** Presidio redacts PII on the inbound path so the model never sees it in the first place. The gold-standard mitigation: don't let sensitive data reach the model unless the agent genuinely needs it.
- **Output validator.** Scans the response for PII patterns (Presidio again, this time on the output), for secrets patterns (API keys, JWTs, credential strings via [TruffleHog](https://github.com/trufflesecurity/trufflehog)-style regexes), and for cross-tenant data leaks via the retrieved-source provenance check.
- **Groundedness check.** A `ProvenanceLLM` validator (Guardrails AI) or a judge flags outputs whose claims do not match the retrieved sources — a common path for cross-tenant leaks via training-data memorisation.
- **Cross-reference mod-408.** The content-store retention and the tenant-partitioned storage from mod-408 chapter 04 is the perimeter; the safety plane is the last-mile check.

**SafetyRecord category identifier.** `LLM02:sensitive_info_disclosure` with sub-category labels for `pii`, `credentials`, `cross_tenant`.

### LLM03 — Supply Chain

**What it is.** A model, dataset, or dependency in the chain is compromised — a poisoned fine-tune, a backdoored tokenizer, a model-weight injection from an untrusted repository, an MCP server that is itself malicious.

**Where the plane catches it.**
- **Not the input/output filter's primary job.** The plane cannot classify "was this model's training data poisoned" by looking at a single turn. The mitigation is largely upstream — model-gateway (mod-406) provenance, MCP registry (mod-403) attestation, SBOMs, signed artefacts via [Sigstore](https://www.sigstore.dev/).
- **Output drift detection.** The output validator, operated across the fleet, can detect sudden shifts in refusal rate, in grounding failure rate, or in content-classifier scores following a model-weight or dependency bump. The SRE plane's safety-refusal-rate SLI (mod-410 chapter 02) is the channel; the safety plane's quarterly scorecard (chapter 06) is the forensic view.
- **The `SafetyRecord` itself.** By pinning `hook.engine` and `workload.agent_revision` on every record, a post-hoc query "did refusals shift when we upgraded to model X v2" is trivial. The plane contributes evidence; mod-406 and mod-403 own the control.

**SafetyRecord category identifier.** `LLM03:supply_chain` — rarely emitted directly by the plane; mostly flagged by cross-plane audit.

### LLM04 — Data and Model Poisoning

**What it is.** Training data or fine-tuning data is manipulated to introduce a backdoor, a bias, or a specific harmful behaviour.

**Where the plane catches it.**
- **The plane does not detect poisoning at training time.** That is the `ai-risk-engineer`'s and `model-evaluation-engineer`'s territory, covered by the eval plane (mod-407) with trigger fixtures. The plane is a backstop — the output validator catches the harmful behaviour at inference time whether it came from a prompt-injection, a jailbreak, or a poisoned weight.
- **Red-team corpus coverage.** The quarterly scorecard (chapter 06) includes a slice of fixtures derived from known trigger patterns. Coverage gaps surface as an owner action in the risk register.

**SafetyRecord category identifier.** `LLM04:data_poisoning` — flagged by output-side detectors when the symptom is observed; the root-cause attribution happens outside the plane.

### LLM05 — Improper Output Handling

**What it is.** The *application* downstream of the LLM fails to sanitise the model's output before using it — rendering the output as HTML (XSS), executing it as SQL (SQLi), passing it to a shell (RCE), feeding it as a URL to a network client (SSRF).

**Where the plane catches it.**
- **Output validator — shape and content.** The output must conform to the declared schema; any shell meta-character in an argument that is eventually passed to a shell is a plane concern.
- **Tool-invocation policy hook.** Catches the dangerous *consequence*. A URL argument with `file://` or an internal RFC1918 address is refused.
- **Cross-reference mod-404 (sandbox) and mod-405 (policy plane).** Shell execution and arbitrary-URL fetching are sandbox-side concerns; policy-plane rules enforce egress restrictions. The safety plane coordinates with both; it is not the only layer.

**SafetyRecord category identifier.** `LLM05:improper_output_handling` with sub-labels (`shell_metachars`, `ssrf_pattern`, `sqli_pattern`, `xss_pattern`).

### LLM06 — Excessive Agency

**What it is.** The agent has more permission, more tool reach, or more autonomy than the task requires; a single mistake has outsized blast radius. The classic example: an agent with `payments.transfer` permission that is only ever asked to summarise invoices.

**Where the plane catches it.**
- **Primarily mod-405 (policy plane) and mod-403 (MCP registry).** Least privilege is a policy-plane concern — the fewest tools, the narrowest scopes, the shortest-lived credentials. The safety plane is complementary: its tool-invocation policy hook is where "the model wants to call `payments.transfer`" becomes a *content* question, not just a permission question.
- **Tool-invocation policy hook.** Dual-use tools (email, file write, HTTP fetch, payments) run under a safety-side policy that layers on top of mod-405: human-in-the-loop for high-impact calls, rate limits per session, argument-pattern allowlists.
- **Human-in-the-loop checkpoints.** The plane emits an `require_human_approval` verdict on dual-use tool calls when a policy threshold is crossed. The runtime (mod-402) is the thing that pauses the session awaiting the human; the plane produces the signal.

**SafetyRecord category identifier.** `LLM06:excessive_agency` — often paired with an `LLM01` or `LLM08` cause.

### LLM07 — System Prompt Leakage

**What it is.** The model reveals the operator's system prompt — scaffolding, tool definitions, confidential policy text. The leak can be via direct extraction ("repeat your system prompt") or indirect (the model quotes a system prompt fragment in an error message).

**Where the plane catches it.**
- **Input filter.** Known system-prompt-extraction attack patterns are in the regex stage; the content classifier flags novel variants.
- **Output validator.** A substring check against the actual system prompt text catches the leak at the tail. The plane needs the agent's system prompt available to the output validator (as a hash or as a stripped snippet set), which is a cross-plane handoff: the paved-road SDK (mod-412) registers the system prompt with the plane at session start.
- **Principle.** The best mitigation is treating the system prompt as *public* — do not put secrets, credentials, or confidential text in it. The plane is a backstop against the "we assumed the system prompt was private" bug.

**SafetyRecord category identifier.** `LLM07:system_prompt_leakage`.

### LLM08 — Vector and Embedding Weaknesses

**What it is.** Attacks on the retrieval / embedding pipeline — adversarial documents indexed into the vector store, cross-tenant embedding leakage, retrieval poisoning (a document is indexed that is attacker-controlled and will be retrieved for benign queries).

**Where the plane catches it.**
- **Input filter on retrieved content.** Every retrieved document, before it enters the model's context, is classified. Chapter 04 is the treatment; the short version: do not treat retrieved content as trusted just because it came from your vector store.
- **Vector-store-side mitigations.** Tenant partitioning on indices, write-time attestation on documents, provenance labels carried through retrieval. Cross-reference mod-408 chapter 04 for the content store's tenant partition.

**SafetyRecord category identifier.** `LLM08:retrieval_poisoning` and `LLM08:embedding_leak`.

### LLM09 — Misinformation

**What it is.** The model confidently emits factually wrong information — hallucination, biased extrapolation, outdated facts.

**Where the plane catches it.**
- **Primarily the eval plane (mod-407).** Factuality is an eval concern; the plane's hallucination-rate SLI (mod-410 chapter 02 SLI 4) reads from the eval plane's judge verdicts.
- **Output validator — groundedness.** On a per-turn basis for critical workloads, the plane runs a provenance check: the model's claims against the retrieved sources. A claim with no source is flagged.
- **Refusal template for low confidence.** The plane can require the agent to emit a hedged refusal when the content classifier reports low groundedness.

**SafetyRecord category identifier.** `LLM09:misinformation` — rarely emitted at the per-turn hook rate (expensive); emitted at the sampling rate the eval plane configures.

### LLM10 — Unbounded Consumption

**What it is.** Model calls, tool calls, or compute consumed without bound — intentional abuse (DoS via prompt-engineering a long-running loop), prompt-amplification (asking the model to produce 10 M tokens), or runaway-loop bugs.

**Where the plane catches it.**
- **Primarily mod-406 (model gateway) and mod-404 (sandbox).** Cost limits, token limits, per-session caps, rate limits are gateway and sandbox concerns.
- **Tool-invocation policy hook.** The plane can refuse tool calls that pattern-match abuse — the 7th `shell.execute` in 30 seconds with a `curl` to an external address, the 20th `search` call with a different query each time to drain a vendor API quota.
- **SRE plane cross-ref.** Runaway loops are covered by mod-410 chapter 05 playbook 1. The safety plane is complementary: it produces the detection signal for the content-pattern side of the cascade.

**SafetyRecord category identifier.** `LLM10:unbounded_consumption` — typically emitted by the tool-invocation hook when it refuses a pattern.

## The Top 10 in one table, mapped

| OWASP ID | Shape | Primary hook | Backup hook | Primary engine |
| --- | --- | --- | --- | --- |
| LLM01 — Prompt Injection | Untrusted content hijacks instructions | Input filter | Output validator; tool-invocation hook | Llama Guard / Prompt Guard / Lakera Guard |
| LLM02 — Sensitive Info Disclosure | Model reveals sensitive data | Input filter (redact) and output validator (scan) | Groundedness check | Presidio + output classifier |
| LLM03 — Supply Chain | Compromised model / dep / tokenizer | Mostly mod-406 and mod-403 | Plane's output-drift signal | N/A — plane as evidence source |
| LLM04 — Data and Model Poisoning | Training-data manipulation | Mostly mod-407 (triggers) | Plane's output validator (symptom detection) | Eval-plane fixtures |
| LLM05 — Improper Output Handling | Downstream fails to sanitise | Output validator (shape + content); tool-invocation hook | Mod-405 and mod-404 | Guardrails AI + Pydantic |
| LLM06 — Excessive Agency | Over-permissioned agent | Tool-invocation hook + mod-405 | Human-in-the-loop checkpoint | NeMo Guardrails tool-usage rails |
| LLM07 — System Prompt Leakage | Model reveals system prompt | Input filter + output validator (substring match) | — | Content classifier + plane's substring check |
| LLM08 — Vector and Embedding Weaknesses | Retrieval pipeline attacks | Input filter on retrieved content | Vector-store tenant partition | Content classifier + mod-408 tenant partition |
| LLM09 — Misinformation | Hallucination / wrong facts | Mostly mod-407 (judge) | Plane's groundedness check | Judge LLM + Guardrails AI `ProvenanceLLM` |
| LLM10 — Unbounded Consumption | Unbounded calls / tokens / compute | Mostly mod-406 / mod-404 | Tool-invocation hook (pattern refuse) | Rule engine |

Three observations from the table.

- **Half the Top 10 is not primarily the safety plane's job.** LLM03, LLM04, LLM09, LLM10 live mostly in other planes. The safety plane's value in those cases is *contributing evidence* — a `SafetyRecord` row that joins to the root-cause plane — not being the primary mitigation. Scope the plane to the rows where it is primary and partner on the rest.
- **The plane's load-bearing rows are LLM01, LLM02, LLM05, LLM06, LLM07, LLM08.** The next chapter opens LLM01 in depth; LLM02 is covered by the Presidio integration in chapter 02; LLM05 by the structured-output validator; LLM06, LLM07, LLM08 by combinations of the three hooks.
- **The identifiers go on the `SafetyRecord`.** The verdict's `categories` field uses these strings literally (with the rev pin). A query over the record stream slices by category without engine-specific translation.

## The Agentic AI threats catalogue

The [OWASP Agentic AI — Threats and Mitigations](https://genai.owasp.org/resource/agentic-ai-threats-and-mitigations/) catalogue extends the Top 10 to agent-specific threats. The identifiers (as of the current revision) are the `T` series (T01, T02, ...). A pragmatic subset for platform scope, each mapped onto the plane's primitives:

- **T01 — Memory Poisoning.** An attacker writes into the agent's long-term memory (summary, scratchpad, RAG index) in a way that persists across sessions.
  - **Plane role.** Input filter on every memory read (treat memory content as untrusted input); output filter on memory writes (classify what the agent is about to persist). Mapping to `SafetyRecord.categories`: `T01:memory_poisoning`.
- **T02 — Tool Misuse.** An agent uses a legitimately-permissioned tool in an unintended way (abuse of a bulk-email tool, abuse of a search tool for DoS).
  - **Plane role.** Tool-invocation policy hook with argument-pattern allowlist and rate budget. Mapping: `T02:tool_misuse`.
- **T03 — Privilege Compromise.** The agent gains broader scope than intended via a credential-escalation path.
  - **Plane role.** Primarily mod-405 (policy plane) and mod-404 (sandbox). The safety plane contributes detection on the output seam when it sees the agent requesting scope-escalation patterns.
- **T04 — Resource Overload.** DoS via compute / token / cost consumption.
  - **Plane role.** Mostly mod-406 and mod-404; plane emits the content-pattern side.
- **T05 — Cascading Hallucination Attacks.** Hallucinations propagate through agent-to-agent chains (mod-409) and become ground truth downstream.
  - **Plane role.** Groundedness check at each A2A boundary (the plane's output validator runs on each sending side); `source_citation` enforcement.
- **T06 — Intent Breaking and Goal Manipulation.** Attacker reshapes the agent's goal via adversarial inputs; agent believes its new goal is legitimate.
  - **Plane role.** Input filter on every turn with a *goal-check* rail (compare the apparent intent against the operator's declared agent purpose). Harder in practice; this is where chapter 06's red-team scorecard earns its keep.
- **T07 — Misaligned and Deceptive Behaviour.** Agent appears to comply while pursuing an orthogonal goal.
  - **Plane role.** Output validator; cross-reference with the eval plane's behavioural test fixtures.
- **T08 — Repudiation and Untraceability.** Actions taken are not attributable to a principal or a trace.
  - **Plane role.** The `SafetyRecord` schema in chapter 05 is the evidence; mod-408's trace propagation is the perimeter.
- **T09 — Identity Spoofing and Impersonation.** An agent impersonates a human or another agent.
  - **Plane role.** Output validator checks for impersonation claims; mod-405 (policy plane) enforces principal identity.
- **T10 — Overwhelming Human-in-the-Loop (HITL).** The plane raises so many approval requests that humans rubber-stamp them.
  - **Plane role.** The plane maintains a *quality bar* for HITL requests — the request has to include the decision context, the recommended action, and a one-click revoke. The plane can also auto-batch low-risk HITL requests.
- **T11 — Unexpected RCE and Code Attacks.** Code generated by the agent contains injection payloads.
  - **Plane role.** Output validator on code-generating paths; mod-404 (sandbox) is the containment. Cross-reference with [OWASP ASVS](https://owasp.org/www-project-application-security-verification-standard/).
- **T12 — Agent Communication Poisoning.** Malicious messages across A2A channels poison downstream agents.
  - **Plane role.** Input filter at every A2A boundary.
- **T13 — Rogue Agents in Multi-Agent Systems.** A compromised agent in the fleet.
  - **Plane role.** Behavioural-drift detection on the `SafetyRecord` stream; mod-409 circuit breakers.
- **T14 — Human Attacks on Multi-Agent Systems.** Humans target the agent ecosystem via social engineering, phishing, etc.
  - **Plane role.** Contributes evidence; mod-405 and the organisation's broader security programme are primary.
- **T15 — Human Manipulation.** The agent manipulates users (over-compliance, flattery, pressured consent).
  - **Plane role.** Output validator with a *manipulation detector* classifier; often cited as a social-impact threat the plane contributes evidence on.

The complete, canonical identifier list and definitions are on the OWASP project page; use that as the authoritative mapping source.

## The composite OWASP map, as the plane uses it

The `SafetyRecord.categories` field is a list of these identifier strings. A single verdict can match multiple — an indirect injection against an over-permissioned agent is `["LLM01:indirect_prompt_injection", "LLM06:excessive_agency", "T02:tool_misuse"]`. The plane does not de-duplicate; the risk register's grouping logic is where the aggregate rolls up.

```yaml
# policy-pack/owasp-mapping.yaml
schema_version: owasp-mapping/v1
rev:
  llm_top_10: "2025"
  agentic_threats: "2025-07"   # pin to the exact publish date
engine_mappings:
  llama_guard_3:
    "S1": ["LLM01:prompt_injection"]
    "S2": ["LLM02:sensitive_info_disclosure"]
    "S3": ["LLM05:improper_output_handling"]
    "S14": ["LLM10:unbounded_consumption"]
  lakera_guard:
    "prompt_injection": ["LLM01:prompt_injection"]
    "pii": ["LLM02:sensitive_info_disclosure/pii"]
    "unsafe_language": ["LLM02:sensitive_info_disclosure"]
  presidio_pii:
    "US_SSN": ["LLM02:sensitive_info_disclosure/pii"]
    "CREDIT_CARD": ["LLM02:sensitive_info_disclosure/pii"]
    "EMAIL_ADDRESS": ["LLM02:sensitive_info_disclosure/pii"]
```

Five rules for the mapping file.

- **Pin the rev.** Every mapping has a rev string. The record's `verdict.owasp_rev` echoes it. A change to the mapping is a versioned change to the policy pack.
- **Engine-native category names are case-sensitive and version-pinned.** The left-hand side of the mapping matches the engine's reported category literally. A new engine version that renames a category is a mapping-file change.
- **One native category can map to multiple OWASP ids.** A classifier that reports `unsafe_content` without further decomposition maps to a broad set (`LLM02`, maybe `LLM05`) with a lower confidence.
- **Unknown engine categories map to `UNKNOWN:<engine>:<category>`.** Never silently drop. The quarterly scorecard surfaces these; the owner adds a mapping entry.
- **The mapping is tested.** The red-team replay harness (chapter 06) covers the mapping file — a red-team row tagged with an OWASP id has to produce a `SafetyRecord` carrying that id when replayed through the plane. The test fails on unmapped categories.

## Cross-catalogue hooks: MITRE ATLAS and NIST

Two cross-references worth pinning even though the plane does not emit against them as first-class identifiers.

- **[MITRE ATLAS](https://atlas.mitre.org/)** — Adversarial ML Threat Matrix. Organises attacks by tactic and technique (reconnaissance, initial access, execution, evasion, impact). Useful when the `ai-risk-engineer` is translating a red-team finding into a defence-in-depth posture; the SafetyRecord can carry an optional `atlas_refs` field to pin a specific finding to an ATLAS technique.
- **[NIST AI 100-2 — Adversarial Machine Learning Taxonomy](https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.100-2e2023.pdf)** and the **[NIST AI RMF](https://www.nist.gov/itl/ai-risk-management-framework)** — regulator-facing framework for AI risk. The plane contributes the "Measure" and "Manage" functions' evidence. A `SafetyRecord` stream plus the mod-408 trace pipeline is a large fraction of what the AI RMF asks for.

Both are cross-references, not primary category sources. The plane emits OWASP identifiers; the risk register maps those into ATLAS and NIST views as needed.

## Common anti-patterns

Three worth naming before the next chapter.

- **"We just use the engine's native categories."** The record stream fragments by engine. A swap from Llama Guard to Lakera breaks every downstream dashboard. The mapping file is non-negotiable.
- **"We'll emit one category per verdict."** A single incident can sit at the intersection of LLM01, LLM06, and T02. Emit all; let the risk register group.
- **"We'll keep the categories open-ended."** A `categories: ["custom.tenant_rule_42"]` without any OWASP mapping is unauditable in the fleet view. If a tenant has a bespoke rule, map it to the closest OWASP id and add the tenant-specific label as a sub-identifier (`LLM02:sensitive_info_disclosure/tenant_rule_42`). The quarterly scorecard can still slice both ways.

## Summary

- The plane emits OWASP LLM Top 10 and OWASP Agentic AI identifier strings in `SafetyRecord.categories`, with a pinned rev string. The mapping from each engine's native categories to the OWASP identifiers lives in a versioned policy-pack file; unknown categories map to `UNKNOWN:<engine>:<category>` and are swept by the quarterly scorecard.
- Six of the Top 10 are primarily the plane's job: LLM01 (prompt injection), LLM02 (sensitive info), LLM05 (improper output handling), LLM06 (excessive agency), LLM07 (system prompt leakage), LLM08 (vector/embedding weaknesses). The other four are contributions to other planes' primary mitigations.
- The Agentic AI catalogue extends the picture with session-persistence (T01), tool misuse (T02), A2A (T05, T12), HITL (T10), code-generation (T11), and multi-agent threats (T13–T15). Each maps onto hooks the plane already runs; the identifiers go in the same `SafetyRecord.categories` list.
- MITRE ATLAS and NIST AI 100-2 / AI RMF are cross-references; the plane does not emit against them as primary identifiers but the `SafetyRecord` carries optional refs.

The next chapter opens LLM01 in depth — direct and indirect prompt injection, the trust boundaries the plane enforces, the defensive techniques (spotlighting, structured-input discipline, retrieval sanitisation), and the refusal shape.
