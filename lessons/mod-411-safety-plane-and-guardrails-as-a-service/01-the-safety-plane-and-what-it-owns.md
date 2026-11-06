# The Safety Plane and What It Owns

The runtime (mod-402) hosts the agent loop. The MCP gateway (mod-403) catalogues tools. The sandbox (mod-404) runs code. The policy plane (mod-405) authorises what a given principal is allowed to do. The model gateway (mod-406) brokers the completion. The evaluation plane (mod-407) verdicts regressions. The observability plane (mod-408) fixes the vocabulary. The inter-agent plane (mod-409) carries agent-to-agent calls. The SRE plane (mod-410) runs the fleet.

Every one of those planes stops short of a question the platform still has to answer: *is this input safe to feed to the model, is this output safe to return to the user, is this tool call safe to execute, and when the answer is "no", what does the refusal look like*. mod-405 answers "is the principal allowed to invoke this tool" — a question about identity and permission. The safety plane answers "is the *content* of this interaction acceptable" — a question about what the content means. A principal that is allowed to call `email.send` is still not allowed to use that call to exfiltrate a tenant's PII, and a principal that is refused a request still deserves a refusal that is auditable, consistent, and legally defensible.

This chapter fixes the slice this module owns. The chapters that follow pick the pieces: the guardrail toolkit landscape (02), OWASP LLM Top 10 and OWASP Agentic AI Threats mapped onto plane primitives (03), prompt-injection defence at platform scope (04), the standardised `SafetyRecord` schema and the EU AI Act Article 15 / 72 obligations (05), measuring effectiveness against red-team findings (06), and build-vs-buy reasoning across the vendor and OSS landscape (07).

## What the safety plane owns

The plane sits between the model gateway (mod-406) on the model-call seam, the MCP gateway (mod-403) on the tool-invocation seam, the policy plane (mod-405) on the authorisation seam, the observability plane (mod-408) on the telemetry seam, the evaluation plane (mod-407) on the red-team-corpus seam, the SRE plane (mod-410) on the refusal-drift seam, and the stream-aligned teams whose agents call all of them. It owns:

- **The input filter.** The in-band hook that inspects every agent-bound prompt — the user turn, the system prompt fragments assembled from retrieval, and any tool-result content re-injected into context — before it reaches the model. Classifies against a configurable set of categories (jailbreak attempt, prompt-injection pattern, disallowed topic, PII leakage upstream, ...). Returns `allow | redact | refuse` with a verdict record. Chapter 02 maps this onto NeMo Guardrails input rails, [Llama Guard](https://ai.meta.com/research/publications/llama-guard-llm-based-input-output-safeguard-for-human-ai-conversations/), [ShieldGemma](https://ai.google.dev/gemma/docs/shieldgemma/model_card), [OpenAI Moderation](https://platform.openai.com/docs/guides/moderation), [Lakera Guard](https://www.lakera.ai/), and [Presidio](https://microsoft.github.io/presidio/) in the PII-redaction slot.
- **The output validator.** The in-band hook that inspects every model-produced message — final answer, tool-call arguments, streamed tokens as a tail-pass — before it reaches the user or a downstream tool. Validates against shape (structured output conforms to a schema), content (no leaked secrets, no disallowed category, no hallucinated fact against retrieval), and consequence (the output does not instruct a tool call that the policy plane would refuse). Chapter 02 maps this onto [Guardrails AI](https://www.guardrailsai.com/docs) validators, NeMo Guardrails output rails, [Anthropic Constitutional Classifiers](https://www.anthropic.com/research/constitutional-classifiers), and the model-provider-side moderation APIs.
- **The tool-invocation policy hook.** The in-band check that inspects every tool call — its target, its arguments, its surrounding session context — against safety-side rules that complement mod-405's authorisation. mod-405 answers "is this principal allowed to call this tool with these scopes"; the safety plane answers "does this *content* (the argument, the retrieved context, the inferred intent) match a known-dangerous pattern". Dual-use tools (email, file write, shell, external HTTP) are the usual residents. Returns `allow | require_human_approval | refuse`.
- **The standardised refusal behaviour.** When any of the three hooks refuses, the plane emits a refusal in a *consistent* shape — the same verdict code, the same user-facing wording family, the same refusal-event record — regardless of which guardrail engine produced the decision. Tenants see one refusal story, not a patchwork. Chapter 04 pins the refusal shape; chapter 05 pins the event shape.
- **The `SafetyRecord` emitter.** Every verdict the plane produces — allow, redact, require_human_approval, refuse — produces one `SafetyRecord` keyed to the mod-408 `trace_id`. The record is the invariant this module defends. Chapter 05 pins the schema; mod-408 chapter 02 declares the span-event attributes it surfaces on.
- **The guardrail configuration surface.** A paved-road configuration (policy packs, category lists, PII rules, refusal templates) a tenant opts into when their agent adopts the platform. Overrides are a per-tenant YAML file with change control; drift is detected by the conformance test in chapter 05 and by the regression harness in chapter 06.
- **The red-team integration point.** The plane publishes, on a declared cadence, a scorecard of its verdicts against the red-team corpus `ai-risk-engineer` (level 25) and `agentic-safety-engineer` (level 40) maintain. The plane does not author the corpus — those peers do — but it operates the harness that replays it and emits precision/recall/latency/coverage. Chapter 06.
- **The guardrail-stack build-vs-buy decision record.** A document, versioned, naming which guardrail capability sits on which engine (OSS or vendor), the latency / cost / coverage tradeoff that justified it, and the rotation plan if the vendor is swapped. The plane does not block tenants from bolting on their own guardrails, but the paved-road default is a single documented choice per slot. Chapter 07.

The plane does **not** own:

- **The identity-and-permission decision.** Whether a principal (an agent, a user, a tenant) is *allowed* to call a given tool with given scopes is the policy plane's call — mod-405. The safety plane runs on top of that decision; a tool call that mod-405 has already refused never reaches the safety hook. Dual-use tools where the policy plane is permissive by design are precisely where the safety plane earns its keep.
- **The red-team methodology.** What attack to try, which jailbreak families to cover, how often to refresh the corpus — those are the `ai-risk-engineer` and `agentic-safety-engineer` roles' domain. The plane consumes their output; it does not produce it. Chapter 06 pins the handoff.
- **The AI risk register itself.** The curated register of residual risks, mitigations, owners, and review cadence is the `ai-risk-engineer`'s artefact. The safety plane feeds it `SafetyRecord` rows the register can group over; the register's structure and its signing authority belong to risk engineering.
- **The eval-plane fixture catalogue.** When a safety failure reproduces as a fixture, the fixture lives in mod-407's catalogue under the `safety/<finding_id>` namespace. The safety plane *promotes* fixtures the same way the SRE plane (mod-410 chapter 06) does; it does not own the catalogue.
- **The trace semantic vocabulary.** [OpenTelemetry Semantic Conventions for Generative AI](https://opentelemetry.io/docs/specs/semconv/gen-ai/) and the platform's `platform.safety.*` attribute namespace belong to mod-408. The safety plane emits against the vocabulary; it does not pick it.
- **The product-side agent correctness.** Whether an agent's prompt is well-structured, whether its system prompt is clear about what it will not do, whether the model choice is a good fit — those are `senior-agentic-ai-engineer` (level 40) and `ai-eval-engineer` (level 30) concerns. The safety plane is a backstop, not a substitute for a well-designed agent.
- **The legal interpretation.** Whether a tenant's use case is "high-risk" under the [EU AI Act — Regulation (EU) 2024/1689](https://eur-lex.europa.eu/eli/reg/2024/1689/oj) Article 6 and Annex III, whether a particular refusal shape meets a specific regulator's expectation, whether a serious-incident disclosure under Article 73 has been triggered — those are counsel's calls. The plane produces the mechanical evidence that supports those calls; it does not make them.

Drift off this slice and the plane becomes "the people who write the moderation code into every agent", which is just a shared library. Scope policing is a feature of this module.

## Why a plane and not a per-agent library

A few concrete questions that are hard to answer without a safety plane and trivial with one.

- **"The tenant's finance agent and their HR agent both deal with PII. Do they redact PII the same way?"** Without a plane, the finance agent's team wrote a regex, the HR agent's team integrated [Presidio](https://microsoft.github.io/presidio/), and the two refusal messages read differently. With one, both agents call the same input-filter hook and get the same verdict shape, the same redaction template, and the same `SafetyRecord` row the risk register can audit. Chapter 02.
- **"A novel jailbreak was reported in the corpus on Monday. How many of our agents are vulnerable, and how many are patched by Friday?"** Without a plane, the patch is a PR against every agent repo. With one, the jailbreak pattern becomes a policy-pack update; the regression harness in chapter 06 reports coverage; the rollout is one deploy that covers the fleet. Chapter 04.
- **"Our `email.send` tool was invoked in a session that looked, in hindsight, like an indirect injection via a retrieved document. How do we know whether the plane caught it or let it through?"** Without a plane, the evidence is scattered across the agent's logs, the MCP gateway's logs, the policy plane's decisions, and somebody's Slack message. With a `SafetyRecord` per verdict, keyed to the `trace_id`, the audit is one query. Chapter 05.
- **"We moved from Llama Guard 2 to Llama Guard 3. Did refusals shift? Is any tenant newly over-refused?"** Without a plane, the swap is per-agent and every team discovers the regression at different times. With one, the swap is behind the hook, the drift is on the SRE plane's safety-refusal-rate SLI (mod-410 chapter 02), and the fast-burn alert fires within hours of the change. Chapter 06 pairs with mod-410 chapter 02.
- **"A regulator asks for post-market-monitoring evidence for a high-risk tenant's agent over Q3. Can we produce it?"** Without a plane, each agent's telemetry is bespoke and the export takes weeks. With one, the `SafetyRecord` stream plus the mod-408 trace export already satisfies Article 72's "actively and systematically collect, document, and analyse" and the serious-incident cut under Article 73 is one `trace_id → incident-package` command away. Chapter 05.
- **"Do we build the sensitive-content classifier or buy it from Lakera / Robust Intelligence / Arthur?"** Without a plane, each team answers this independently and the platform's cost-per-session and latency-per-session both drift. With a plane, the decision is a single, auditable record (chapter 07), and swapping the engine is a configuration change rather than a code migration across a fleet of agents.

These are the capabilities the plane delivers. They separate "we have moderation" from "we have a safety plane". Chapters 02–07 build each.

## What makes safety for agents different from safety for LLMs

The "LLM guardrails" literature was written for single-turn chat completions: user sends a message, filter it, model answers, filter the answer. An agent workload breaks three of that model's assumptions.

- **A session is multi-turn and context-laden.** The model sees not just this turn's user input but a growing stack of previous turns, retrieved documents, tool results re-injected as content, and system-prompt fragments composed from policy packs. A single-turn moderation call misses the attack that lives in the *retrieved document* the agent is about to summarise, or in the *tool output* the agent is about to act on. Chapter 04 makes direct-vs-indirect prompt injection the central threat model.
- **The output is often a tool call, not a human-readable message.** Classical moderation ("is this text offensive?") does not cover "is this tool-call argument an exfiltration attempt?". An output validator that only runs on user-facing text misses exactly the dangerous output — the `email.send({to: attacker, body: secret_data})` JSON. Chapter 02 forces the output validator to run on structured output and tool-call arguments, not just chat-visible text.
- **Agency compounds the blast radius.** A single misuse of a chat model is a single bad message. A single misuse of an agent that has `shell.execute`, `database.query`, and `payments.transfer` is a cascade. The [OWASP Agentic AI Threats and Mitigations](https://genai.owasp.org/resource/agentic-ai-threats-and-mitigations/) catalogue (and the [OWASP LLM Top 10 for LLM Applications](https://genai.owasp.org/llm-top-10/) category LLM06 "Excessive Agency") codify the pattern. Chapter 03 maps each OWASP category onto the plane's primitives.

The OSS guardrail ecosystem — NeMo Guardrails, Guardrails AI, Llama Guard, ShieldGemma — handles the single-turn moderation piece well. The agent-specific pieces (tool-arg validation, indirect-injection defence, dual-use-tool policy, cross-turn drift) are what the plane has to compose on top. Chapter 02 names the composition.

## The invariant: `SafetyRecord`

Pin the record shape before writing the plane. Every verdict the plane produces — input-filter, output-validator, tool-invocation-policy-hook — produces one `SafetyRecord`, keyed to the mod-408 `trace_id`. The record is invariant across engine, tenant, workload class, and verdict type. Every chapter adds a field; chapter 05 pins the final schema.

```json
{
  "record_id": "sr_01HWX3K2M...",
  "schema_version": "safety-record/v1",
  "emitted_at": "2026-11-06T14:22:03Z",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "00f067aa0ba902b7",
  "tenant_id": "acme-financial",
  "workload": {
    "agent_id": "writer-v1",
    "agent_revision": "git:abcd1234",
    "workload_class": "interactive-chat",
    "tier": "T1"
  },
  "hook": {
    "point": "input_filter",
    "engine": "llama-guard-3-8b",
    "policy_pack": "paved-road@2026-10-01",
    "latency_ms": 42
  },
  "subject": {
    "subject_type": "user_turn",
    "subject_hash": "sha256:7f3c...",
    "subject_source": "runtime:turn_events[12]"
  },
  "verdict": {
    "decision": "refuse",
    "categories": ["LLM01:prompt_injection"],
    "confidence": 0.94,
    "refusal_template": "paved-road.refuse.prompt_injection.v2"
  },
  "red_team_refs": [
    "red-team-corpus:row-2026-10-22-0471"
  ],
  "risk_register_ref": "risk-register:R-2026-0113",
  "fixture_promotion_ref": null,
  "outcome": {
    "action_taken": "block_turn",
    "downstream_notified": ["mod-410:safety-refusal-rate-sli"]
  }
}
```

Seven invariants the record enforces:

1. **Every verdict names the engine, policy pack, and hook point it came from.** Audit answers "which classifier produced this refusal" from the record alone. Chapter 02.
2. **Every verdict ties back to a trace / span.** No refusal fires without its debug exhibits; the risk register never needs a cross-system join. Cross-reference mod-408 chapter 05.
3. **The verdict lists the categories it matched.** The OWASP LLM01..LLM10 and OWASP Agentic AI category identifiers are first-class strings. Chapter 03 pins the identifier set.
4. **Refusal templates are named, not free-text.** Two refusals from two engines against the same category produce the same user-facing wording family. Chapter 04.
5. **Red-team and risk-register references are structural.** A verdict known to correspond to a logged red-team row records that row id; the risk register joins over it. Chapter 06.
6. **Fixture promotion is traceable.** When a safety failure reproduces as a fixture in mod-407's catalogue, the `fixture_promotion_ref` fills in and the next deploy that would reintroduce the failure is gated out.
7. **The subject is hashed and sourced, not embedded.** The record carries a SHA-256 of the subject (the user turn, the tool-call arguments, the retrieved document) and a pointer to where the full content lives in the trace pipeline. Nothing in the record itself is a PII leak. Chapter 05.

Chapters 02–07 build the pieces that produce this record.

## The three obligations

Pin the plane to three non-negotiable obligations. Everything in chapters 02–07 is a decision about how to deliver them.

1. **"Every agent on the paved road routes every user turn, every tool invocation, and every model response through the plane, and the refusal shape is identical across engines."** An agent that bypasses the plane is a reliability bet with no stakeholders and an audit gap. Chapter 02 and chapter 04.
2. **"Every verdict emits a `SafetyRecord` keyed on `trace_id`, consumable by the AI risk register and by the EU AI Act post-market-monitoring export without a bespoke transformation."** A verdict that only lives in a Slack alert is a verdict the next auditor cannot find. Chapter 05.
3. **"Every quarter, the plane publishes a scorecard against the current red-team corpus, with precision, recall, latency, coverage, and a diff against the prior quarter."** A plane that has no measured effectiveness is a plane that will silently regress. Chapter 06.

A plane that cannot defend all three is a collection of moderation libraries, not a safety plane.

## Where the plane physically sits

A typical deployment shape:

- **An input-filter sidecar or middleware.** A gRPC / HTTP endpoint invoked by the runtime (mod-402) before each model call. In Python, often a FastAPI service wrapping [NeMo Guardrails](https://docs.nvidia.com/nemo/guardrails/latest/index.html) or a direct call to [Llama Guard](https://huggingface.co/meta-llama/Llama-Guard-3-8B) / [ShieldGemma](https://huggingface.co/google/shieldgemma-9b) on a dedicated inference endpoint, with a short-circuit cache for exact-match prompts.
- **An output-validator middleware.** The symmetric hook on the return path from the model gateway. Often a composition of a shape validator ([Guardrails AI](https://www.guardrailsai.com/docs) RAIL spec or Pydantic schema), a content classifier (the same engine reused, or a smaller sibling like [OpenAI Moderation](https://platform.openai.com/docs/guides/moderation)), and a refusal-template renderer.
- **A tool-invocation policy hook.** Called by the MCP gateway (mod-403) before each tool invocation, after mod-405's policy plane has approved the identity-and-permission check. Reads the tool name, the arguments, and relevant session context; returns `allow | require_human_approval | refuse` against a safety-side rule set (e.g., "`email.send` to any address outside the tenant's allowlist requires human approval").
- **A policy-pack repository.** A versioned repository (GitOps-friendly) of rule files, category lists, PII rule sets ([Presidio](https://microsoft.github.io/presidio/) recognisers), and refusal templates. Pulled by every hook at startup; hot-reloadable via a control-plane push. One policy pack per environment; per-tenant overlays sit beside.
- **The `SafetyRecord` emitter.** A library (or SDK hook) each guardrail service uses to publish its verdict record. The record flows to the observability plane's event bus (mod-408 chapter 04's collector tier) and from there to the risk-register consumer, the SRE plane's SLI pipeline, and the mod-407 catalogue's safety fixture promoter.
- **The red-team replay harness.** A batch job (per chapter 06) that replays the current red-team corpus against the plane on each policy-pack version bump and on a weekly cadence against the unchanged pack (drift detection). Emits a scorecard record.
- **The paved-road SDK integration.** The mod-412 SDK packages the hook invocations so a stream-aligned team's agent is on the plane by importing the library — no hand-wired integration per agent. Cross-reference mod-412.

If you cannot draw this topology on a whiteboard and name the box every verdict flows through, chapter 02 is the thing to read next.

## Where this plane sits in the track

The safety plane is the operational guard rail over everything that came before. Each prior module left a hook or a signal:

- **mod-402 (runtime)** — turn events. The input filter is wired into the runtime's turn-dispatch; the output validator into the turn-finalise step.
- **mod-403 (MCP gateway)** — tool-invocation span. The tool-invocation policy hook runs after mod-405's allow and before the gateway forwards the call.
- **mod-404 (sandbox)** — the code-execution plane. The plane's output validator checks tool-call arguments destined for the sandbox against dangerous patterns (shell meta-characters, egress patterns).
- **mod-405 (policy plane)** — identity-and-permission decisions. The safety plane runs after mod-405 says yes; a tool call mod-405 refuses never reaches the safety hook.
- **mod-406 (model gateway)** — chat-completion spans. The input filter runs before the gateway's completion call; the output validator runs after.
- **mod-407 (eval plane)** — fixture catalogue. Safety failures promote fixtures under `safety/<finding_id>`.
- **mod-408 (observability plane)** — semantic vocabulary and trace propagation. Every `SafetyRecord` is a span event emitted against this plane's conventions. Chapter 05 pins the attribute names.
- **mod-409 (inter-agent plane)** — agent-to-agent span events. The plane's hooks run on every A2A edge the same way they run on the user edge.
- **mod-410 (SRE plane)** — SLI pipeline. The *safety-refusal rate* and its drift alert in mod-410 chapter 02 read directly off the `SafetyRecord` stream.

The downstream handoffs:

- **mod-412 (paved-road SDK)** — packages the three hook invocations so the plane is importable, not integrable-per-agent.
- **mod-413 (platform leadership)** — owns the operating cadence over the quarterly red-team scorecard, the policy-pack change-control board, and the exception register when a tenant requests a temporary pack override.

## Summary

- The safety plane owns the input filter, the output validator, the tool-invocation policy hook, the standardised refusal behaviour, the `SafetyRecord` emitter, the policy-pack configuration surface, the red-team integration point, and the build-vs-buy decision record. It does not own identity-and-permission decisions (mod-405), red-team methodology (`ai-risk-engineer`, `agentic-safety-engineer`), the risk register itself, the eval fixture catalogue (mod-407), the trace vocabulary (mod-408), product-side agent correctness, or legal interpretation.
- Agent safety extends the classical LLM-moderation model in three ways: a session is multi-turn and context-laden, outputs are often tool calls (not chat text), and agency compounds blast radius. Chapters 02–07 build each extension on top of the OSS and vendor engines named in the module spec.
- Every verdict emits a `SafetyRecord` keyed on `trace_id`, carrying engine, policy pack, categories, refusal template, red-team and risk-register references, and the action taken. Chapters 02–07 add fields.
- Three non-negotiable obligations: every paved-road agent routes every seam through the plane with one refusal shape; every verdict emits a `SafetyRecord` the risk register and the EU AI Act export both consume without transformation; every quarter the plane publishes a scorecard against the red-team corpus.

The next chapter opens the first piece — the guardrail toolkit landscape, mapping each named engine (NeMo Guardrails, Guardrails AI, Llama Guard, ShieldGemma, OpenAI Moderation, Anthropic Constitutional Classifiers, Lakera Guard, Presidio) onto the three hook points.
