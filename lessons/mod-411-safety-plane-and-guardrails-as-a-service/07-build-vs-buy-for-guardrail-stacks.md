# Build-vs-Buy for Guardrail Stacks

The plane runs engines — chapter 02 — and it measures them — chapter 06. Every engine is a build-or-buy question, sometimes an awkward hybrid. This chapter is the reasoning the Agent Platform Engineer brings to that decision: which guardrail capability runs on an OSS self-hosted engine, which on a vendor-hosted product, which on a model-provider-native feature, and how to swap without fragmenting the `SafetyRecord` stream or burning an incident recovering.

The vendor landscape the spec names: [Robust Intelligence](https://www.robustintelligence.com/) (part of Cisco since 2024), [Lakera](https://www.lakera.ai/), [Arthur](https://www.arthur.ai/), [HiddenLayer](https://hiddenlayer.com/), [Calypso](https://calypsoai.com/). The OSS alternatives: NeMo Guardrails, Guardrails AI, Llama Guard, ShieldGemma, Presidio (chapter 02). Cloud-native options: [AWS Bedrock Guardrails](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html), [Azure AI Content Safety](https://learn.microsoft.com/en-us/azure/ai-services/content-safety/overview), [Google Cloud Model Armor](https://cloud.google.com/security-command-center/docs/model-armor-overview). Model-provider-native: [OpenAI Moderation](https://platform.openai.com/docs/guides/moderation), Anthropic Constitutional-classifier-backed safety, Google safety settings on Gemini.

The chapter is not a product-pick guide — product features and pricing shift per-quarter. It is the decision frame that produces a defensible choice and a maintainable swap path.

## What "build" and "buy" actually mean

Both ends are rarely clean. Three forms of "build" and three forms of "buy":

- **Build — OSS off-the-shelf.** The plane runs Llama Guard / ShieldGemma / NeMo Guardrails on self-hosted infrastructure, using the engine without modification. The platform owns deployment, GPU capacity, version pinning, and the policy pack. Build in the sense of operational ownership; off-the-shelf in the sense of using the engine as-published.
- **Build — OSS plus internal fine-tune.** The platform maintains its own fine-tune of Llama Guard against the platform's red-team corpus. More effective; more operational cost; retraining is an on-going commitment.
- **Build — fully in-house.** A classifier trained from scratch on the platform's data. Rare, expensive, and only the right choice for very specific threat models the public engines cannot cover.
- **Buy — vendor-hosted.** Lakera Guard / Robust Intelligence / Arthur / etc. Egress to a vendor endpoint on every hook invocation. Vendor owns training, corpus updates, model infrastructure.
- **Buy — model-provider-native.** OpenAI Moderation, Anthropic's safety path, Google safety settings, AWS Bedrock Guardrails when the model is on Bedrock. Same egress trade-off but tightly coupled to the completion path.
- **Buy — vendor-provided OSS.** Some vendors (NVIDIA on NeMo Guardrails, Microsoft on Presidio) maintain an OSS project the platform runs itself. Technically a build; organisationally a buy of the maintenance commitment.

A plane almost always runs a mix. The decision is per-slot, not plane-wide.

## The decision axes

Seven axes the plane evaluates per engine slot. The weighting is workload- and tenant-specific; the axes are stable.

### 1. Threat coverage and corpus freshness

Does the engine catch the threats the risk engineers care about? How fast does it update when a new attack lands?

- **Vendor advantage.** Dedicated red teams; corpus updates on a short cadence (weeks). Lakera and Robust Intelligence publicly cite continuous red-teaming as a selling point.
- **OSS reality.** Llama Guard 3 is released in a version; Meta updates on the order of quarters. ShieldGemma similarly. Novel attacks between releases must be caught by the platform's mapping / policy-pack layer or by a sibling engine.
- **In-house advantage.** The platform's corpus is the platform's own; coverage matches exactly what the risk engineers find. Cost is the retraining cadence.

The scorecard (chapter 06) is the measurement; the axis is about what the measurement will show six months from now.

### 2. Latency

The hooks are on the turn's critical path. Model-provider-native usually wins (no additional network hop), self-hosted OSS second (one intranet hop), vendor third (one internet hop, plus the vendor's own latency).

- **Self-hosted Llama Guard 3 8B on a close-by A10G.** ~30–50 ms p50, ~100 ms p95 under load.
- **OpenAI Moderation.** ~50–150 ms p95 depending on region; essentially free on the request.
- **Lakera Guard / Robust Intelligence.** Published / observed ~50–200 ms p95, region-dependent.

Chapter 06's latency column on the scorecard is the empirical answer per deployment; this is the axis the budget pushes against.

### 3. Cost

Three cost shapes:

- **Vendor per-call.** Per 1 K or 1 M API calls. Linear in traffic. Predictable, but at platform scale (10M+ turns/day with two classifier calls each), the bill is substantial.
- **Self-hosted GPU.** Fixed in GPU hours, scales with throughput, subject to the platform's existing capacity planning. A single A10G serves ~100–300 Llama Guard 3 8B calls/sec depending on sequence length and batching; total GPU hours integrate over the day.
- **Cloud-native.** Billed similarly to vendor per-call; may be bundled with other cloud-platform spend.

Rough regime of break-even: at 100M+ classifier calls/month, self-hosted Llama Guard typically beats vendor per-call. Below 10M calls/month, vendor per-call is usually cheaper than keeping a GPU warm. Between those, both are viable and other axes dominate.

<!-- needs-research: current vendor pricing shifts frequently (Lakera, Robust Intelligence, Arthur published rate cards and bulk-enterprise rates vary). Pin the cost analysis to the specific vendor SKUs and current GPU on-demand rates at purchase time. -->

### 4. Data residency and egress

Can the platform's content leave the trust boundary? Three regimes:

- **Fully air-gapped.** Vendor egress is forbidden. Self-hosted OSS is the only option.
- **Regional egress only.** Vendor egress is allowed to in-region endpoints; vendor must have the region. OpenAI, Anthropic, Lakera have EU regions; smaller vendors may not.
- **Global egress.** Vendor choice is open; data-processing agreement covers the handling.

Tenants in regulated verticals (finance, health, government) often sit in the first or second regime. The plane's build-vs-buy per engine is sometimes dictated by the tenant mix.

### 5. Policy flexibility

Can the engine be configured to reflect the platform's policy?

- **ShieldGemma.** Custom policy prompt is first-class; the engine applies the policy at inference.
- **Llama Guard 3.** Taxonomy is published and stable; custom categories via prompt-prefix are possible but increase drift between engine's reported categories and plane's OWASP mapping.
- **NeMo Guardrails.** Framework-level composability — policy is what the plane writes in Colang; the engine (classifier) is called by the policy. Maximum flexibility.
- **Guardrails AI.** Validators are composable; new validators are PRs to the hub or local plugins.
- **Vendor-hosted.** Flexibility varies; typically category toggling and sometimes custom categories, but the underlying model is the vendor's. Some vendors (Robust Intelligence, Arthur) offer custom fine-tuning for enterprise customers.
- **Model-provider-native.** Limited to what the provider exposes; OpenAI Moderation categories are the provider's; AWS Bedrock Guardrails has a custom-categories feature.

### 6. Operational and lifecycle commitment

Who keeps it running, who patches it, who upgrades it?

- **Vendor-hosted.** Vendor owns uptime, patching, model versioning (opt-in or forced). The platform's risk: vendor churn — acquisition (Robust Intelligence → Cisco), SKU rename, deprecation. The plane's backup engine (chapter 02) is the contingency.
- **Self-hosted OSS.** Platform owns everything — GPU capacity, model versioning, library upgrades, policy packs, CVE response. The cost is real; the control is maximal.
- **Cloud-native.** Cloud provider owns the engine; the platform owns the integration. Similar to vendor-hosted but with the cloud vendor's uptime (often strong).

### 7. Legal and audit posture

For regulated workloads (EU AI Act Article 15 / 72 — chapter 05), the plane may need:

- **Engine version provenance** — pinable and attestable.
- **Classifier training data provenance** — more often required for in-house or OSS (where it is published) than for vendor-hosted (where it is proprietary).
- **Data-processing agreement** — vendor-hosted engines need a DPA; cloud-native engines are covered under the cloud contract; OSS does not need one.
- **Serious-incident disclosability** — an engine whose vendor has gag clauses may constrain the platform's Article 73 reporting. Rarely a blocker but worth checking.

## The decision matrix, by slot

Applying the axes to each slot in chapter 02's assignment:

### Slot — Input filter: prompt-injection classifier

- **OSS (Llama Guard 3, Prompt Guard, ShieldGemma).** Default for platforms with GPU capacity and > 10M calls/month. Threat coverage behind vendor; corpus freshness limited; policy flexibility medium.
- **Vendor (Lakera Guard, Robust Intelligence).** Choice when corpus freshness is the dominant axis. Latency and cost penalties.
- **Model-provider-native.** Not primarily this slot; most providers do not expose prompt-injection detection as a standalone API.

**Paved-road default:** OSS (Llama Guard 3 8B) on self-hosted GPU, with Prompt Guard 86M as a fast prefilter. Vendor is the backup; swap is a config change.

### Slot — Input filter: PII redaction

- **OSS (Presidio).** Default. Deterministic, extensible, free.
- **Cloud-native (Google Cloud DLP, Azure AI Language PII, AWS Comprehend).** Choice when the platform is cloud-pinned and the native service is more mature than Presidio for the PII set in scope.
- **Vendor.** Rarely a differentiator for PII.

**Paved-road default:** Presidio; cloud-native when the platform is single-cloud and the native service has materially better recognisers.

### Slot — Input filter: indirect-injection scanning on retrieved content

- Same choices as prompt-injection classifier. The scenarios are harder; the corpus is harder to build; the measurement (chapter 06) is where the choice is validated.

**Paved-road default:** Same engine as the user-turn classifier, tighter threshold.

### Slot — Dialogue flow control

- **OSS (NeMo Guardrails).** Default when the plane needs composable flow control across multiple engines. The orchestrator slot.
- **In-house middleware.** Choice when the plane prefers to stay in the host language and composition is simple.

**Paved-road default:** NeMo Guardrails for platforms adopting a multi-engine composition; in-house middleware for platforms with simpler needs.

### Slot — Output validator: structured-output shape

- **OSS (Guardrails AI, Pydantic, JSON Schema).** Default. Framework-agnostic, free, flexible.
- **Model-provider structured-output features.** Complementary; the provider enforces the schema server-side (OpenAI's `response_format=json_schema`, Anthropic's tool-use with input_schema).

**Paved-road default:** Model-provider structured-output when the model supports it; Guardrails AI or Pydantic as a belt-and-suspenders validator.

### Slot — Output validator: content classification

- **OSS (Llama Guard 3, ShieldGemma).** Default; same engine as input filter is efficient (one inference endpoint, two call sites).
- **Model-provider-native (OpenAI Moderation on OpenAI paths).** Low-cost; use alongside OSS.
- **Vendor.** For platforms that already pay vendor per-call on the input side, extending to output is incremental.

**Paved-road default:** Same classifier as input filter + model-provider-native complement when applicable.

### Slot — Output validator: jailbreak / policy violation (hard case)

- **Anthropic Constitutional classifiers (on Claude paths).** The provider-native path when the completion is Claude.
- **LLM-as-judge via NeMo Guardrails on non-Claude paths.** More expensive; used sparingly on high-risk turns.

**Paved-road default:** Provider-native on Claude paths; LLM-as-judge on a sampled fraction (10–20%) of other paths; always-on only for T1 tenants.

### Slot — Output validator: groundedness / hallucination

- **Guardrails AI `ProvenanceLLM` validator.** Default.
- **Model-provider citation features (Anthropic citations, OpenAI structured citations).** Complementary.
- **Vendor (Arthur Shield has a hallucination detector).** Choice when the platform wants a dedicated hallucination SKU; costs vendor per-call.

**Paved-road default:** Guardrails AI `ProvenanceLLM`; provider citations when available; vendor only if the platform's eval plane (mod-407) cannot cover the gap.

### Slot — Tool-invocation policy hook

- **In-house rule engine (plus NeMo Guardrails for composition).** Default. The rules are platform-specific; no vendor has the context.
- **Cloud-native (AWS Bedrock Guardrails tool filters, Azure AI Content Safety with tool-call integration).** Complementary; sometimes covers argument-pattern checks.

**Paved-road default:** In-house rule engine on top of mod-405's policy decisions; NeMo Guardrails tool-usage rails for composition; no vendor dependency.

## The decision document

The plane maintains a versioned decision document — one page per slot — pinned to the paved road.

```markdown
# Build-vs-Buy Decision — Input Filter: Prompt-Injection Classifier

**Decision owner:** @agent-platform-engineer-lead
**Rev:** 2026-10-01
**Signed-by:** @agent-platform-engineer-lead, @ai-risk-engineer-lead, @agentic-safety-engineer-lead

## Decision
Paved-road default: Llama Guard 3 8B @ version 2024-07-23, self-hosted on A10G pool `safety-plane-inference`.
Fast prefilter: Prompt Guard 86M, same pool.
Backup: Lakera Guard; swap is a one-line policy-pack change; documented in runbook `rb-safety-engine-swap-v3`.

## Rationale
- Threat coverage: scorecard Q3 2026 showed 91% recall on severity=high LLM01 rows at the current threshold; Lakera Guard scored 94% on the same corpus. The 3pp gap is within the exception register.
- Latency: p95 94 ms (budget 150 ms); Lakera Guard p95 observed 180 ms (over budget).
- Cost: at current traffic (300M classifier calls/month), self-hosted is ~$18K/month in GPU; Lakera Guard at list price would be ~$X/month (redacted).
- Egress: 30% of tenants are air-gapped; vendor cannot serve them.
- Policy flexibility: taxonomy is stable; OWASP mapping is a known file.
- Lifecycle: Meta releases quarterly; the platform's retraining cadence is semi-annual.

## Swap path
- Triggered by: scorecard regression > 5pp recall, vendor outage > 1h, cost crossover.
- Mechanism: policy-pack change `input-filter.engine: lakera-guard`; redeploy; scorecard replay on the first week.
- Unknown-category handling: mapping file `owasp-mapping-lakera-guard.yaml` is pre-written and kept current.

## Risks accepted
- Novel-attack catchup lag: ~1 week vs. vendor. Mitigation: red-team corpus weekly drift replay.
- GPU capacity volatility: tracked by SRE plane (mod-410 chapter 02 TTFT SLI).

## Next review
Q1 2027 quarterly review; triggered earlier on any accepted-risk event.
```

Five rules for the decision document.

- **One slot per document.** Stacking multiple slots into one doc hides where the tradeoff is.
- **Signed by the three roles.** Build-vs-buy is not only platform; the risk engineer and the agentic safety engineer have equity in the coverage tradeoff.
- **Swap path is a precondition, not an afterthought.** A decision with no swap path is a decision the platform will regret.
- **Risks accepted are named.** The exception register (mod-413) tracks them.
- **Next review is scheduled.** Build-vs-buy is reviewed quarterly; the rev string is the pin.

## Swap discipline

The plane avoids big-bang engine swaps. The pattern:

1. **Deploy the new engine in shadow.** New engine receives every input; produces a `SafetyRecord` with `outcome.action_taken = "no_op"` and a distinctive `hook.policy_pack` suffix. Production continues to use the incumbent.
2. **Replay the corpus against both.** Produce a side-by-side scorecard. Chapter 06's harness, two engines.
3. **Compare production distributions.** Over a week, how often do the two engines disagree? What is the disagreement distribution per category?
4. **Decide on the swap.** The three signatories review the scorecard and the production disagreement; sign the decision document rev bump.
5. **Deploy the new engine as the primary.** Keep the old engine running in shadow for one further week — enables rollback without redeploying.
6. **Decommission the old engine.** Only after the shadow week shows no regression.

Six steps; one week shadow; one week parallel-primary; one week shadow-of-old. The month of discipline avoids the single incident caused by a surprise regression on swap day.

## The OpenAI / Anthropic / cloud-provider special cases

Three special cases worth calling out explicitly.

- **OpenAI Moderation.** Free (within the OpenAI usage contract). Low latency on OpenAI-model paths (co-located). Category drift is possible as OpenAI updates the model (`omni-moderation-latest` vs. a pinned version). The paved road: use it alongside Llama Guard on OpenAI paths; monitor for drift; the mapping file absorbs category renames.
- **Anthropic Constitutional classifier path.** Built into the Claude Messages API when relevant; the platform benefits transparently and sees refusals returned by the API. The platform's input filter can be lighter on Claude paths (the provider is doing part of the work) but still needs to run — belt and suspenders; chapter 04 is explicit.
- **AWS Bedrock Guardrails / Azure AI Content Safety / Google Cloud Model Armor.** Cloud-native options for platforms pinned to one cloud. Typically stronger integration with the cloud's model-provider mesh; typically narrower policy flexibility than OSS. Right default for single-cloud platforms with modest policy customisation; wrong default for platforms needing deep custom categories.

## Vendor-specific notes

Short, operational notes on each vendor named in the spec.

### Robust Intelligence (Cisco)

Acquired by Cisco in August 2024. Product line includes [AI Validation](https://www.robustintelligence.com/platform/ai-validation) (pre-deployment red-teaming of models) and [AI Protection](https://www.robustintelligence.com/platform/ai-protection) (runtime guardrails). Enterprise-focused, often sold as part of broader AI security programmes. Red-team expertise is a stated differentiator. Appropriate when the platform wants a managed partner relationship beyond a classifier API.

### Lakera

Startup, SaaS-first. [Lakera Guard](https://www.lakera.ai/products/lakera-guard) focuses on prompt-injection, PII, and content moderation. Also publishes [Gandalf](https://gandalf.lakera.ai/) — a public prompt-injection challenge the plane's benign and red-team corpora often draw on. Appropriate when the platform wants a focused, developer-friendly guardrail with a continuously-updated red-team corpus.

### Arthur

AI observability vendor with a guardrails product line ([Arthur Shield](https://www.arthur.ai/arthur-shield)) that includes hallucination detection, prompt-injection detection, and content moderation. Appropriate when the platform wants to combine guardrails with broader model monitoring under one vendor.

### HiddenLayer

AI security vendor focused on adversarial ML threats — model-weight attacks, inference-time attacks. [HiddenLayer AISec Platform](https://hiddenlayer.com/aisec-platform/) is their broader product; runtime guardrails are one component. Appropriate when the platform needs coverage against the MITRE ATLAS threat spectrum beyond LLM prompt-injection — model-theft, backdoors, etc.

### Calypso AI

Vendor focused on moderation and policy for enterprise GenAI deployments. [Calypso AI](https://calypsoai.com/) product line emphasises model-agnostic deployment and policy orchestration. Appropriate when the platform wants a policy-first approach with less investment in self-hosted OSS orchestration.

The product details move; the axes stay. Match the axes, measure against the scorecard, and swap when the signal justifies it.

## Common anti-patterns

Five worth naming before the module's exercises.

- **"We'll buy the most-expensive vendor; must be the best."** Scorecard measurement is what decides. "Most-expensive" is not an axis.
- **"We'll build everything ourselves; vendors are a trap."** A fully self-hosted plane is viable; so is a plane with a vendor on the slot where corpus freshness matters most. The dogma hurts the user.
- **"We signed a vendor contract; we're committed."** The decision document is a versioned artefact; vendor contracts have terms; the plane's swap path is live even when the contract is. Lock-in is a risk to be measured, not a fait accompli.
- **"We'll let each tenant pick their vendor."** The `SafetyRecord` stream fragments; the scorecard becomes per-tenant; the fleet view breaks. Per-tenant policy packs are fine; per-tenant engines are an opt-out from the paved road.
- **"We'll not document the decision; everyone knows the current state."** The next quarterly review will not know; the next platform engineer will not know; the audit will not know.

## Summary

- Build-vs-buy is per-slot, not plane-wide. The plane almost always runs a mix.
- Seven axes: threat coverage and corpus freshness, latency, cost, data residency / egress, policy flexibility, operational / lifecycle commitment, legal / audit posture. Weight depends on workload and tenant mix.
- The paved-road default per slot: OSS self-hosted (Llama Guard / ShieldGemma / Presidio / Guardrails AI / NeMo Guardrails) for most platforms; model-provider-native (OpenAI Moderation, Anthropic Constitutional classifiers) where the completion is on that provider; cloud-native (Bedrock Guardrails / Content Safety / Model Armor) for single-cloud platforms with modest customisation; vendor (Lakera / Robust Intelligence / Arthur / HiddenLayer / Calypso) on the slot where corpus freshness is the dominant axis.
- The decision document is a versioned artefact per slot, signed by three roles, with a declared swap path. Swap discipline: one week shadow, review, primary with incumbent in shadow, decommission. The month of discipline avoids the incident caused by a surprise regression on swap day.
- Common anti-patterns: assuming price correlates with effectiveness, dogmatic build or buy, treating a vendor contract as lock-in, per-tenant engine choice, and leaving the decision undocumented.

With this chapter closed, the module's scope is complete. The exercises exercise each chapter against a concrete artefact; `resources.md` catalogues the primary sources the chapters cite.
