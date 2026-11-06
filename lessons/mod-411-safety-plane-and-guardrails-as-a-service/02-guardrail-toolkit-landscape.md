# The Guardrail Toolkit Landscape

Chapter 01 named three hook points — input filter, output validator, tool-invocation policy hook — and claimed the plane composes several engines behind one refusal shape. This chapter names the engines, pins what each is good for, and shows how they slot into the plane. The track spec enumerates eight: [NeMo Guardrails](https://docs.nvidia.com/nemo/guardrails/latest/index.html), [Guardrails AI](https://www.guardrailsai.com/docs), [Llama Guard](https://ai.meta.com/research/publications/llama-guard-llm-based-input-output-safeguard-for-human-ai-conversations/), [ShieldGemma](https://ai.google.dev/gemma/docs/shieldgemma/model_card), [OpenAI Moderation](https://platform.openai.com/docs/guides/moderation), [Anthropic Constitutional Classifiers](https://www.anthropic.com/research/constitutional-classifiers), [Lakera Guard](https://www.lakera.ai/), and [Presidio](https://microsoft.github.io/presidio/). The pattern the chapter builds: each engine fills one slot well; the plane's value is the composition contract and the uniform verdict shape that comes out the other side.

## The two axes to classify an engine on

Before picking engines, name the axes. Two are load-bearing for the plane.

- **What *stage* of the pipeline the engine runs at.** Every piece of content that flows through the plane sits at one of four stages: *inbound text* (the raw user turn, a retrieved document, a tool result being re-injected into context), *the model's output text* (the chat message or free-form completion), *the model's output as structured data* (a tool-call argument, a JSON schema instance), or *the tool-invocation decision* (does this `email.send` call run?). An engine is specialised to one or two stages; a plane that uses the wrong engine at the wrong stage pays the price in false positives or misses.
- **What *mode* of detection the engine uses.** Four broad modes, each with different cost/latency/false-positive profile: *pattern / rule* (regex, keyword list, grammar — fast, deterministic, brittle), *small classifier* (dedicated model trained on moderation categories — e.g., Llama Guard, ShieldGemma, OpenAI Moderation — ~tens of milliseconds, calibrated recall, bounded FP), *LLM-as-judge* (a general-purpose model prompted to classify — e.g., Anthropic Constitutional Classifiers in the sense of using Claude's reasoning; NeMo Guardrails' output-rail check — slower, more flexible), and *vendor-hosted ensemble* (Lakera, Robust Intelligence — opaque, often red-team-informed, requires egress).

Pin these two axes and the toolkit map falls out. The next section is the map.

## The paved-road assignment

The plane's paved-road assignment per hook point, naming the engines the spec lists:

| Hook point | Primary responsibility | Default engine(s) | Mode |
| --- | --- | --- | --- |
| Input filter — inbound text | Jailbreak / prompt-injection detection on the user turn; category classification on retrieved content before it enters context | [Llama Guard 3](https://huggingface.co/meta-llama/Llama-Guard-3-8B) or [ShieldGemma](https://ai.google.dev/gemma/docs/shieldgemma/model_card) (OSS); [Lakera Guard](https://www.lakera.ai/) (vendor) | Small classifier |
| Input filter — PII redaction | Detect and redact or hash PII in inbound content before it is sent to the model | [Presidio](https://microsoft.github.io/presidio/) | Pattern + NER |
| Input filter — indirect-injection scanning | Classify retrieved documents and tool results for embedded instructions attacking the agent | Llama Guard 3 (prompt-injection head); [NeMo Guardrails input rails](https://docs.nvidia.com/nemo/guardrails/latest/user-guides/configuration-guide.html); Lakera Guard | Small classifier + rule |
| Dialogue flow control | Topic restriction, canonical-form intent classification, allowed-tools-per-topic policy | [NeMo Guardrails](https://docs.nvidia.com/nemo/guardrails/latest/index.html) (Colang flows) | LLM-as-judge + rule |
| Output validator — structured output shape | Tool-call argument conforms to schema; types and enum domains hold | [Guardrails AI](https://www.guardrailsai.com/docs) validators; [Pydantic](https://docs.pydantic.dev/) / [JSON Schema](https://json-schema.org/) | Rule |
| Output validator — content classification | Detect refusable categories in the model's user-visible message | OpenAI Moderation (for OpenAI-model paths); Llama Guard / ShieldGemma (model-agnostic) | Small classifier |
| Output validator — jailbreak / policy violation | Catch outputs that indicate the input filter missed a successful jailbreak | [Anthropic Constitutional Classifiers](https://www.anthropic.com/research/constitutional-classifiers) (on Anthropic-model paths); NeMo Guardrails output rails | LLM-as-judge |
| Output validator — hallucination / groundedness | Flag answers inconsistent with the retrieved sources | Guardrails AI validators (e.g., `provenance`); custom judge | LLM-as-judge |
| Tool-invocation policy hook | Content-aware policy over the (tool, args, session) tuple; dual-use escalation | [NeMo Guardrails](https://docs.nvidia.com/nemo/guardrails/latest/user-guides/configuration-guide.html) tool-usage rails + rule layer | Rule + LLM-as-judge |

Three points the table flattens.

- **Choose one engine per slot, document the backups.** A plane running two input classifiers in shadow is paying double for the same recall. The paved-road choice is one engine, with a documented backup for when it fails (vendor outage, model deprecated, licence change). Chapter 07 is the build-vs-buy conversation.
- **The engine list is not exhaustive.** [AWS Bedrock Guardrails](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html), [Azure AI Content Safety](https://learn.microsoft.com/en-us/azure/ai-services/content-safety/overview), and [Google Cloud Model Armor](https://cloud.google.com/security-command-center/docs/model-armor-overview) are platform-native equivalents each cloud ships; a cloud-pinned platform may pick the native one and skip a separate OSS engine. The eight named in the spec are the ones the Agent Platform Engineer role must be able to reason about; the cloud-native equivalents are additions.
- **Model-provider-native guardrails have a seat.** Anthropic's Constitutional Classifiers, OpenAI's Moderation API, Google's safety settings, AWS Bedrock Guardrails — each is tightly coupled to its provider and often cheaper (latency and $) than a standalone engine. The plane routes to the native guardrail when the completion is on that provider, and falls through to an OSS classifier when the provider is swapped. Chapter 06's model-gateway integration pattern.

## Engine-by-engine, what each is good for

Enough detail on each to pick one for a slot without reading the entire doc.

### NeMo Guardrails (NVIDIA)

**What it is.** An open-source orchestration framework for programmable guardrails on top of LLMs. Reference: [NeMo Guardrails documentation](https://docs.nvidia.com/nemo/guardrails/latest/index.html) and the project on [GitHub](https://github.com/NVIDIA/NeMo-Guardrails).

**Key mental model.** Policies are expressed in Colang — a declarative DSL for conversational flows. Rails come in four flavours: *input rails* (run on the user turn before it reaches the LLM), *output rails* (run on the LLM response), *dialog rails* (constrain the conversational flow), and *execution rails* (gate tool calls). Each rail can call any engine — a small classifier, a judge LLM, a Python function, a vendor API — and compose results.

**Where it slots.** The orchestrator for the plane's *dialogue flow control* and the *tool-invocation policy hook*. NeMo Guardrails is not a classifier itself; it is the configurable composition point that calls one. A platform that adopts NeMo Guardrails writes Colang flows that invoke Llama Guard for category detection, Presidio for PII, and a Python function for its tenant-specific business rule.

**Strengths.** Framework-agnostic (works with any model provider). Mature tool integration (OpenAI, Anthropic, local via vLLM, LangChain). The rail model is a clean separation between "what to check" and "what to do on failure". Active upstream.

**Weaknesses.** Colang is a DSL — another language in the stack. Dialog rails require a well-defined canonical form for intents; messy intent taxonomies produce brittle rails. Latency depends on which engines the rails call; a rail that calls a judge LLM is a judge-LLM round-trip.

**When not to use it.** If the plane needs only a single-stage input classifier with no composition logic, direct use of Llama Guard or Lakera Guard is lighter. NeMo Guardrails earns its keep when the plane composes three or more engines or when dialog-level flow control is first-class.

### Guardrails AI

**What it is.** An open-source output-validation library for LLM responses. Reference: [Guardrails AI documentation](https://www.guardrailsai.com/docs) and the [Hub](https://hub.guardrailsai.com/) of community validators.

**Key mental model.** Specify the expected output as a RAIL spec (XML dialect) or a Pydantic model; wrap the LLM call; on invalid output, re-ask the LLM, repair, or refuse. Validators are the compositional unit — `ValidChoices`, `ToxicLanguage`, `ProvenanceLLM`, `RegexMatch`, `GibberishText`, and many more.

**Where it slots.** The *output validator* for structured outputs. If the agent emits tool-call JSON, or a structured response that drives downstream automation, Guardrails AI is the repair-and-validate loop. Also a reasonable slot for groundedness checks (`ProvenanceLLM` against retrieved context).

**Strengths.** Battery-included validators for common shapes. Re-ask-on-failure pattern is first-class (and sometimes exactly what the plane wants — e.g., ask the model to repair a slightly-malformed JSON). Model-agnostic.

**Weaknesses.** Re-ask-on-failure doubles the token bill if used naively; the plane needs a policy on how many repairs are allowed before a hard refusal. Not an orchestration framework — compose with NeMo Guardrails or a custom middleware for multi-stage flows.

**When not to use it.** If the structured output is tightly schema-defined, a plain Pydantic validation plus a refusal is lighter. Guardrails AI shines when the response shape is loose enough that automated repair is worth the retry cost.

### Llama Guard (Meta)

**What it is.** A family of small LLM-based safety classifiers released by Meta. Current public versions include Llama Guard 2 (8B), Llama Guard 3 (8B and 1B), and the task-specific [Prompt Guard](https://huggingface.co/meta-llama/Prompt-Guard-86M). References: [Llama Guard paper](https://ai.meta.com/research/publications/llama-guard-llm-based-input-output-safeguard-for-human-ai-conversations/), model card on [Hugging Face — Llama Guard 3 8B](https://huggingface.co/meta-llama/Llama-Guard-3-8B).

**Key mental model.** Fine-tuned Llama variants prompted with a taxonomy of unsafe categories (violence, sexual content, criminal planning, privacy, etc.). Takes a conversation turn (user or assistant) and returns `safe | unsafe` plus the matched categories. Llama Guard 3 is multilingual; Llama Guard 3 Vision is multi-modal; Prompt Guard is the smaller sibling specifically for prompt-injection and jailbreak detection.

**Where it slots.** The default *input-filter content classifier* and the *output-validator content classifier* for the OSS paved road. Prompt Guard is the specialised injection-detector variant.

**Strengths.** Open weights (community-licence; read the terms). Multilingual. Category taxonomy is published and stable within a major version. Reasonable latency on a GPU (tens of milliseconds); CPU-only is possible on the 1B variant for lower-throughput paths. Self-hostable — no third-party egress.

**Weaknesses.** Licence is permissive-with-conditions; read Meta's [Llama Community License](https://llama.meta.com/llama3/license/). The taxonomy may not match a tenant's policy; override categories via the prompt is possible but introduces drift between what the engine reports and what the tenant's policy means. False positives on code snippets and technical content are common.

**When not to use it.** If the deployment cannot host a GPU-backed classifier, a hosted alternative (OpenAI Moderation, Lakera Guard) saves the infra bill. If the taxonomy is a bad fit, a classifier more tunable to the tenant's categories (ShieldGemma allows a custom policy prompt) is a better default.

### ShieldGemma (Google)

**What it is.** A family of open-weight safety classifiers from Google based on Gemma. References: [ShieldGemma model card](https://ai.google.dev/gemma/docs/shieldgemma/model_card), paper [ShieldGemma: Generative AI Content Moderation Based on Gemma](https://arxiv.org/abs/2407.21772), [ShieldGemma 2 release](https://blog.google/technology/developers/gemma-3/) (image understanding).

**Key mental model.** Classifier fine-tuned on Gemma (2B, 9B, 27B sizes) with a structured policy prompt — the operator provides the policy text in-prompt; the model returns a verdict. Supports a small set of harm types (sexual content, dangerous content, harassment, hate speech) and an extensible format.

**Where it slots.** Direct alternative to Llama Guard in the *input-filter* and *output-validator* classifier slots.

**Strengths.** Open weights ([Gemma Terms of Use](https://ai.google.dev/gemma/terms)). Smaller sizes (2B) are deployable on commodity GPUs. Custom-policy support is first-class — the plane can inject a tenant-specific policy string without retraining.

**Weaknesses.** Smaller model family than Llama Guard; multilingual support varies. The specific harm type set is narrower than Llama Guard's.

**When not to use it.** Same shape as Llama Guard; pick one, not both. The practical choice is often driven by which model family the platform already hosts (if the stack already runs Gemma for other purposes, ShieldGemma is zero marginal infra).

### OpenAI Moderation

**What it is.** A hosted moderation endpoint from OpenAI. Reference: [Moderation guide](https://platform.openai.com/docs/guides/moderation) and the current model [`omni-moderation-latest`](https://platform.openai.com/docs/models/omni-moderation-latest).

**Key mental model.** Send text (and, for the omni model, images); receive a category score per harm category (sexual, hate, harassment, self-harm, violence, with "/minors" and "/threatening" sub-types) plus a top-level `flagged` boolean. Free tier (within OpenAI usage).

**Where it slots.** A low-cost *content classifier* for the input filter or output validator when the platform is already an OpenAI customer. Can also serve as the moderation leg for the *output validator* on OpenAI-model completions to match the model-provider's own category definitions.

**Strengths.** Zero infra. Free (billed as an OpenAI API call but historically not counted against spending). Low latency. Multi-modal (omni).

**Weaknesses.** Third-party egress — content leaves the platform's trust boundary. Category set is OpenAI's; a tenant with a bespoke policy still needs another engine. The platform is coupled to OpenAI availability; the plane needs a fallback.

**When not to use it.** Air-gapped deployments, regulated tenants that cannot egress content, or platforms whose guardrail policy is strict enough to require custom-category classification.

### Anthropic Constitutional Classifiers

**What it is.** Anthropic's research on training classifiers from a constitution — a set of natural-language rules — and using them to filter inputs and outputs. Reference: [Constitutional Classifiers: Defending against universal jailbreaks across thousands of hours of red teaming](https://www.anthropic.com/research/constitutional-classifiers). Operationally, deployment on Anthropic's platform surfaces as a safety check inside the Claude Messages API when relevant; the published paper also enables external replication of the training approach.

**Key mental model.** A constitution is a set of rules in plain English describing allowed and disallowed content. Classifiers are trained to score an input or an output against the constitution. The classifiers run as a two-stage pipeline: input classifier blocks jailbreak attempts before the model sees them; output classifier catches cases where the model generated disallowed content despite the input filter.

**Where it slots.** On Claude-backed completions, the Anthropic-side guardrail is the *output validator — jailbreak / policy violation* default. For an air-gapped deployment, the research is a reference the platform can reimplement on its own policy — the *shape* is portable even when the specific classifier is not.

**Strengths.** Trained against a realistic red-team corpus (the paper reports thousands of red-team hours). Natural-language policy specification. Tight integration when the model is already Claude.

**Weaknesses.** Deployed classifier is Anthropic-side; not self-hostable. The "re-implement from the paper" path is a project, not a dependency pull.

**When not to use it.** Non-Anthropic completions. Also: when the platform's policy is radically different from the paper's constitution, re-implementing is the realistic path.

### Lakera Guard

**What it is.** A vendor-hosted guardrail product focused on prompt-injection, PII, toxicity, and policy violation detection. Reference: [Lakera documentation](https://docs.lakera.ai/) and [Lakera Guard](https://www.lakera.ai/products/lakera-guard).

**Key mental model.** HTTPS endpoint; send the content (optionally with session context); receive a verdict per category with a confidence score and a suggested action. Trained on a proprietary red-team corpus that Lakera maintains and extends.

**Where it slots.** The *input-filter prompt-injection* slot, and the *output-validator content classifier* slot. Alternative to Llama Guard / ShieldGemma for platforms that want a vendor-maintained red-team corpus baked in.

**Strengths.** Vendor-maintained up-to-date detection against the current red-team state of the art. Low latency. Multilingual. Optional PII redaction endpoint.

**Weaknesses.** Vendor dependency (egress, licence, pricing). Verdict categories are Lakera-defined; translation into the plane's category identifiers (chapter 03) is a mapping the plane maintains. Proprietary; no insight into false-positive mode changes between releases.

**When not to use it.** Air-gapped deployments, platforms with egress restrictions, and platforms whose scale makes the per-call vendor pricing prohibitive. For those, the paved-road default is Llama Guard / ShieldGemma plus platform-internal fine-tuning. Chapter 07 is the decision frame.

### Presidio (Microsoft)

**What it is.** An open-source PII detection and anonymisation toolkit. Reference: [Presidio documentation](https://microsoft.github.io/presidio/) and [GitHub](https://github.com/microsoft/presidio).

**Key mental model.** Two services: an *analyzer* (NER + pattern recognisers find PII entities — names, emails, credit cards, SSNs, bespoke patterns) and an *anonymizer* (redacts, hashes, encrypts, or replaces detected entities). Recognisers are composable: ship with 50+ built-in (US SSN, UK NIN, credit cards, IBANs, etc.), plus a well-documented custom-recogniser API.

**Where it slots.** The *input-filter PII redaction* default. Also used on the output side to catch cases where the model generates PII that should have been redacted inbound.

**Strengths.** Open-source, self-hosted. Extensible recognisers. Mature set of built-ins. Separate analyze / anonymize steps make it easy to log "we detected PII" without logging the PII itself.

**Weaknesses.** Pattern-based recognisers trade recall for precision; configuration-heavy. NER model is the bottleneck for new entity types — add a model and re-benchmark. Not a replacement for a trained classifier on category detection; it does not know "this is a jailbreak".

**When not to use it.** When the plane needs category detection, not PII. When the deployment is on a cloud platform with a native PII service (e.g., [Google Cloud DLP](https://cloud.google.com/dlp), [Azure AI Language PII detection](https://learn.microsoft.com/en-us/azure/ai-services/language-service/personally-identifiable-information/overview)) and the platform prefers the native path.

## Composition patterns

The engines are necessary; the composition is where the plane earns its name. Four patterns the plane reuses.

### Pattern 1 — Belt and suspenders at the input seam

The input filter runs *two* engines in series: a fast pattern / regex layer that catches exact-match attacks and known prompt-injection strings (cheap, zero-model), then a classifier (Llama Guard / ShieldGemma / Lakera) for everything else. The regex layer catches a long tail of known patterns at ~1 ms; the classifier handles the fuzzy cases at ~30 ms. Both verdicts feed one `SafetyRecord`.

```yaml
# policy-pack/input-filter.yaml
stages:
  - name: known_bad_patterns
    engine: regex
    source: policy-pack/patterns/prompt_injection.yaml
    on_match:
      decision: refuse
      categories: ["LLM01:prompt_injection"]
      template: "paved-road.refuse.prompt_injection.v2"
  - name: content_classifier
    engine: llama_guard_3_8b
    endpoint: "http://llama-guard:8080/v1/classify"
    timeout_ms: 500
    on_match_unsafe:
      decision: refuse
      categories: "${verdict.categories}"
      template: "paved-road.refuse.classifier.v2"
    on_timeout:
      decision: allow
      annotate: "classifier_timeout"
```

The second-stage `on_timeout: allow` is a deliberate choice — the plane fails open on classifier unavailability for latency-sensitive paths and compensates with an increased sampling rate on the output validator. The opposite choice (fail closed) is appropriate for high-risk tenants; it is a per-tenant setting on the policy-pack overlay.

### Pattern 2 — Shape-then-content on the output

The output validator runs shape-validation first (Guardrails AI / Pydantic / JSON Schema) because a malformed tool call cannot be audited for content anyway. If the shape passes, the content classifier runs. Two verdicts; two possible refusal shapes; one record.

```python
def output_validator(agent_response, trace_ctx):
    shape_result = guardrails_validate(agent_response, schema)
    if not shape_result.ok:
        return SafetyVerdict(
            decision="refuse",
            categories=["LLM05:improper_output_handling"],
            template="paved-road.refuse.shape_invalid.v1",
            engine="guardrails-ai",
            hook_point="output_validator",
            trace_ctx=trace_ctx,
        )
    content_result = content_classifier.classify(agent_response.text)
    if content_result.unsafe:
        return SafetyVerdict(
            decision="refuse",
            categories=map_to_owasp(content_result.categories),
            template="paved-road.refuse.classifier.v2",
            engine=content_classifier.name,
            hook_point="output_validator",
            trace_ctx=trace_ctx,
        )
    return SafetyVerdict(decision="allow", ...)
```

The `map_to_owasp` step is load-bearing — every engine's native category set must be mapped into the OWASP LLM identifier set the `SafetyRecord` carries. Chapter 03 pins the identifier set; the mapping is a YAML file in the policy pack.

### Pattern 3 — Provider-native where possible, OSS fallback

On an OpenAI completion, the plane uses OpenAI Moderation for content classification on the output. On an Anthropic completion, the plane uses the Constitutional-classifier-backed safety response path. On a self-hosted model, the plane uses Llama Guard. The middleware is one function; the engine is a routing decision driven by which model the completion came from.

```python
PROVIDER_NATIVE = {
    "openai/*": openai_moderation,
    "anthropic/*": anthropic_safety_passthrough,
    "google/*": google_safety_settings,
    "bedrock/*": bedrock_guardrails,
}
FALLBACK = llama_guard_3

def pick_output_classifier(model_name):
    for pattern, engine in PROVIDER_NATIVE.items():
        if fnmatch(model_name, pattern):
            return engine
    return FALLBACK
```

Three benefits. The platform pays the cheapest classifier per completion. The platform's latency is bounded by the provider's own latency (the native classifier runs on the same request path). And the plane keeps a vendor-neutral fallback for when the provider is swapped or degraded.

One constraint: the mapping from the provider's native categories into the OWASP identifiers has to be curated and versioned. A provider that renames a category (OpenAI has done this — the old `hate` category was split) ripples into the `SafetyRecord` stream; the mapping file is where the ripple is contained.

### Pattern 4 — Composition inside NeMo Guardrails

For a platform that commits to NeMo Guardrails as the orchestrator, the above patterns live inside Colang flows. The composition becomes explicit in configuration and the plane's own code shrinks.

```colang
# Illustrative Colang — exact syntax varies by NeMo Guardrails version.
define flow block jailbreak
  $verdict = execute llama_guard_check(user_message=$user_message)
  if $verdict.unsafe
    bot refuse(template="paved-road.refuse.classifier.v2", categories=$verdict.categories)
    stop

define flow redact pii
  $pii = execute presidio_analyze(text=$user_message)
  if $pii.count > 0
    $user_message = execute presidio_anonymize(text=$user_message, entities=$pii)
    annotate safety_record with redacted_pii=$pii.entity_types
```

The gain is readability — a reviewer sees the full input-rail pipeline in one file. The cost is a DSL dependency; a plane that prefers to stay in the host language can achieve the same composition in Python.

## Deployment and operational shape

A few operational facts the plane has to budget for.

- **Latency.** The in-band hooks sit on the critical path of every turn. Budget: input filter ≤ 50 ms p95 on the regex stage; classifier stage ≤ 150 ms p95 on the GPU path; ≤ 400 ms p95 on the CPU path. Output validator similar. Tool-invocation hook ≤ 20 ms (the plane cannot block tool calls for 200 ms). The SRE plane (mod-410 chapter 02) will catch regressions on the overall TTFT / TTFA; the plane's own dashboard pins the classifier's p95 as a sub-SLI.
- **Throughput.** A 10 K turns/minute workload with two classifier calls per turn (input + output) is ~333 classifier calls/sec. On a single A10G, a batched Llama Guard 3 8B serves roughly that range; the plane runs multiple replicas behind a load balancer. Batching helps — a 32-batch cuts per-call latency only on the throughput margin, not on tail.
- **Cost.** Vendor-hosted classifiers (OpenAI Moderation, Lakera Guard) price per 1 K / 1 M calls. OSS classifiers cost GPU hours. Chapter 07 compares the two at the plane's typical load.
- **Policy-pack hot reload.** Every classifier's policy prompt (ShieldGemma custom policy, NeMo Guardrails rail config, Presidio recognisers, OWASP-mapping YAML) must be reloadable without restarting the service. A naive fix — restart on policy change — stalls the fleet for the rollout window. The paved-road pattern: policy packs are versioned artefacts, the control plane pushes a new version, each hook reloads on a signal.
- **Staleness detection.** The red-team replay harness (chapter 06) also serves as the staleness test — a classifier that passes last week's corpus but misses this week's novel pattern is detected by the harness and flagged for retraining or vendor-SKU bump.

## Common anti-patterns

Four worth naming before the next chapter.

- **"We'll run three classifiers and OR them."** The resulting false-positive rate is the sum of each classifier's false positives. The plane either picks one classifier or runs them in a stacked pattern (one specialises in prompt injection, one in content categories) with a documented merge rule per `SafetyRecord`. OR-of-classifiers is not a composition; it is a tax.
- **"We'll let each tenant pick their own classifier."** The `SafetyRecord` stream fragments; the quarterly scorecard becomes per-tenant; the risk register can no longer group over the fleet. Tenant-specific overlays are fine on top of the paved-road default (adding categories, adjusting thresholds); picking a wholly different engine is an opt-out from the paved road, with the governance cadence mod-413 runs.
- **"The engine's native categories are our categories."** The provider renames a category, the OWASP taxonomy evolves, the risk register loses its join key. The plane maintains a mapping from each engine's native categories to the OWASP LLM identifiers *and* a per-engine version pin. Chapter 03 pins the OWASP identifier set.
- **"We'll block aggressively and sort it out later."** The safety-refusal-rate SLI (mod-410 chapter 02) immediately spikes; the false-positive rate on the quarterly scorecard climbs; users route around the refusals by phrasing differently and the plane trains them to look like attackers. The right default is calibrated — refuse when the classifier's confidence is above a threshold, else redact or annotate. Chapter 04 pins the refusal shape; chapter 06 pins the calibration metric.

## Summary

- The guardrail toolkit is a composition, not a classifier choice. Each engine specialises on one or two stages (inbound text, output text, structured output, tool-invocation decision) and one or two modes (regex, small classifier, LLM-as-judge, vendor ensemble).
- The paved-road assignment per hook point names one default engine per slot and a documented backup. The eight engines the spec lists each fill a distinct role: NeMo Guardrails orchestrates; Guardrails AI validates structured outputs; Llama Guard and ShieldGemma are the OSS classifier default; OpenAI Moderation is the hosted alternative; Anthropic Constitutional Classifiers are the Claude-side output check; Lakera Guard is the vendor red-team-informed default; Presidio handles PII.
- Four composition patterns: belt-and-suspenders on input, shape-then-content on output, provider-native-with-OSS-fallback, and composition inside NeMo Guardrails. The pattern chosen per plane depends on the latency budget, the policy-pack discipline, and the cloud/vendor pin.
- The plane maintains a mapping from each engine's native categories to the OWASP LLM identifier set and pins the engine version in every `SafetyRecord`. Native-category drift is contained by the mapping file; policy-pack changes are hot-reloadable; the red-team replay harness detects staleness on a weekly cadence.

The next chapter opens the OWASP LLM Top 10 and OWASP Agentic AI Threats and Mitigations catalogues and maps each entry onto the plane primitives this chapter named.
