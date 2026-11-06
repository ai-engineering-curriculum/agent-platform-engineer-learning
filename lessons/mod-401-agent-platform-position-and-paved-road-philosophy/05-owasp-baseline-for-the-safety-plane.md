# The OWASP Baseline Your Safety Plane Must Defend

Every stream-aligned team that onboards your platform inherits a security posture from it. If the paved road ships without a credible baseline against known LLM and agentic-AI threats, you are shipping a known-unsafe default and distributing the cost of fixing it across every product team.

Two public documents fix the baseline the agent platform must defend against:

- **OWASP — *Top 10 for Large Language Model Applications*** (OWASP GenAI Security Project): <https://owasp.org/www-project-top-10-for-large-language-model-applications/>
- **OWASP — *Agentic AI Threats and Mitigations*** (OWASP GenAI Security Project): <https://genai.owasp.org/resource/agentic-ai-threats-and-mitigations/>

These are the accepted industry taxonomies in 2026. This chapter positions them at *policy-plane altitude* — what the platform's safety plane (mod-411), policy plane (mod-405), and registry / gateway substrates (mod-403, mod-406) must defend against — rather than at a product-prompt level. Chapter 03 established the rule: *the platform runs the policy; stream-aligned teams and specialist peers author it.* This chapter is the policy the platform must be *able* to run.

Read both OWASP documents at least once before continuing. The category labels will evolve across versions; the shape of the threat model will not.

## The two documents, and what each owns

- **OWASP LLM Top 10** covers threats that exist as soon as an LLM is in the system — prompt injection, sensitive-information disclosure, supply chain, model / data poisoning, improper output handling, excessive agency, system-prompt leakage, vector / embedding weaknesses, misinformation, and unbounded consumption (2025 edition category shape — check the current version). These apply to every LLM-backed product, agent or not.
- **OWASP Agentic AI Threats and Mitigations** builds on top with threats that emerge from *agentic* behaviour specifically — autonomous loops, tool invocation, memory, inter-agent routing, long-running goals, human-in-the-loop manipulation, cascading hallucination across steps, and multi-agent attack surface.

Read them in that order. The LLM Top 10 names the primitive threats; the Agentic guide names the compound threats that only exist because an agent can loop, plan, call tools, and talk to other agents.

Where you need the exact, current list of categories, go to the OWASP pages directly — they publish updates. The rest of this chapter speaks about the threats by theme so the mapping onto platform substrate holds even as the categories shift.

## Policy-plane altitude vs. prompt altitude

The single most important framing move for a platform engineer reading these documents: **not every mitigation belongs at the same altitude.** A product team can and should defend against some of these threats inside the prompt and UX of their specific agent. The platform must defend against others at policy-plane altitude so that *every* agent on the platform inherits the defence.

A simple test for each threat: *if two products on the platform must both defend against this, can one product get it wrong without affecting the other?* If the answer is "yes, product-local," the mitigation can live at prompt / UX altitude in that product. If the answer is "no, one product getting this wrong leaks policy violation / data / privilege across tenants," the mitigation belongs at policy-plane altitude in the platform.

- **Policy-plane altitude:** platform-enforced, every tenant gets it, opt-out requires a written contract (chapter 04). Lives in mod-403 / mod-404 / mod-405 / mod-406 / mod-411 substrate.
- **Prompt / UX altitude:** product-authored, per-feature, platform exposes hooks but does not enforce.

## Mapping OWASP threats onto platform substrate

The table below assigns each OWASP LLM Top 10 theme and each OWASP Agentic AI threat theme to the primary platform substrate that defends against it, plus the specialist-peer whose methodology authors the policy the platform runs. Specific category names will shift with OWASP revisions; adapt the mapping to the current version of each document.

### OWASP LLM Top 10 — platform mapping

| Threat theme (OWASP LLM Top 10) | Primary platform substrate | Mitigation posture | Specialist-peer input |
|---------------------------------|-----------------------------|--------------------|-----------------------|
| **Prompt injection** (direct + indirect) | Safety plane (`mod-411`) — input filters, tool-input isolation; policy plane (`mod-405`) — capability model constrains what a compromised prompt can cause | Platform defends tool-invocation side-effects; product defends feature-specific reasoning about input provenance | `ai-risk-engineer` (harm taxonomy), `agentic-safety-engineer` (red-team fixtures) |
| **Sensitive information disclosure** | Safety plane — output validators, PII detection (Presidio-shaped); gateway (`mod-406`) — output filtering before the response leaves the tenant boundary | Platform enforces baseline PII / secret / credential redaction; product declares feature-specific additional categories | `ai-risk-engineer` |
| **Supply chain** (models, datasets, third-party components) | MCP registry (`mod-403`) — schema governance, conformance tests, deprecation ladder; model gateway (`mod-406`) — provenance of model versions, pinning, rollback; eval platform (`mod-407`) — regression gates on supplier change | Platform enforces registration, conformance, provenance; product chooses which registered suppliers it uses | n/a — this is platform scope |
| **Data and model poisoning** | Eval platform (`mod-407`) — regression gates, trace / replay store; observability (`mod-408`) — drift detection | Platform enforces regression gating on every release; product authors the judges | `ai-eval-engineer` (judge methodology) |
| **Improper output handling** (unsanitised output flowing into downstream sinks) | Safety plane — output validators; policy plane — sink-specific capability constraints (e.g. no raw-HTML sink without sanitisation) | Platform enforces default sanitisation; product opts in to raw-sink paths with explicit capability | n/a — platform |
| **Excessive agency** (agent has more authority than the task needs) | Policy plane (`mod-405`) — capability model, RBAC / ABAC, approval workflows; runtime (`mod-402`) — human-in-the-loop checkpoints | Platform enforces least-privilege by default; product declares the specific capabilities each session needs | `ai-risk-engineer` (risk-tier → required capabilities) |
| **System-prompt leakage** | Safety plane — output filter detects and strips platform-preamble patterns; gateway — redaction before response egress | Platform defends its own paved-road preambles; product responsible for its product-specific prompt confidentiality | n/a — split by provenance |
| **Vector / embedding weaknesses** | MCP registry (for retrieval tools) + policy plane — tenant isolation on retrieval stores; eval platform — retrieval quality gates | Platform enforces per-tenant isolation; product authors retrieval judges | `ai-eval-engineer` |
| **Misinformation** (hallucination of unverifiable claims) | Eval platform — hallucination-rate SLIs (see `mod-410` SLI set); safety plane — citation-required output validators where applicable | Platform exposes the SLI and the validator slot; product sets the threshold for its risk tier | `ai-eval-engineer`, `ai-risk-engineer` |
| **Unbounded consumption** (runaway loops, context-window exhaustion, cost spikes) | Runtime (`mod-402`) — concurrency + step caps, timeout / budget enforcement; gateway (`mod-406`) — per-tenant quotas, burst budgets, cost attribution; SRE (`mod-410`) — circuit breakers | Platform enforces caps and quotas; product sets its specific budgets within platform defaults | n/a — platform |

### OWASP Agentic AI Threats — platform mapping

The agentic document covers threats that only exist because of agentic behaviour. The current (2026) release names threats in themes including memory poisoning, tool misuse, privilege compromise, resource overload, cascading hallucination, intent / goal manipulation, misaligned or deceptive behaviour, repudiation / untraceability, identity spoofing, overwhelming the human-in-the-loop, unexpected code execution, agent-to-agent communication poisoning, rogue agents, human attacks on multi-agent systems, and human-manipulation-by-agent patterns. Cross-reference the current OWASP page for the authoritative list.

The platform mapping:

| Agentic threat theme | Primary platform substrate | Mitigation posture |
|----------------------|---------------------------|--------------------|
| **Memory poisoning** (persisted session state / long-term memory corrupted by adversarial input) | Runtime (`mod-402`) — schema-validated session state, checkpoint integrity; safety plane — input filtering on memory-write path | Platform enforces schema + integrity checks on the memory substrate; product authors domain-specific memory validators |
| **Tool misuse** (agent invokes tools in unintended, but still-authorised, ways) | Policy plane (`mod-405`) — capability model, per-call ABAC, approval workflows for high-risk tool classes; MCP registry — tool-risk classification | Platform runs the policy; `ai-risk-engineer` classifies tool risk |
| **Privilege compromise** (agent escalates or inherits authority beyond its session) | Policy plane — SPIFFE / SPIRE non-human identity, short-lived tokens, credential brokering; gateway — identity propagation through the stack | Platform enforces identity + scope on every hop; no mitigation is product-side |
| **Resource overload** (agent-driven DoS on compute, model tokens, or downstream APIs) | Runtime concurrency / budget caps; gateway quotas; SRE circuit breakers | Platform enforces caps; product sets its tier within the caps |
| **Cascading hallucination** (one hallucinated step corrupts subsequent reasoning) | Observability (`mod-408`) — cross-step trace propagation; eval platform — step-level regression fixtures; A2A routing (`mod-409`) — bulkheads | Platform exposes the trace surface; `ai-eval-engineer` authors the step-level judges |
| **Intent / goal manipulation** (adversarial input redirects the agent's goal mid-run) | Safety plane — input filters; runtime — plan-step validation checkpoints | Platform exposes the hooks; product + `ai-risk-engineer` author the plan-coherence check |
| **Misaligned / deceptive behaviour** | Eval platform — behavioural regression fixtures; observability — telemetry for `agentic-safety-engineer` red-team reports | Platform infrastructure; `agentic-safety-engineer` owns detection methodology |
| **Repudiation / untraceability** | Observability (OTel GenAI conventions, `mod-408`) — immutable trace record; registry / gateway — audit trail of every tool call and model call | Platform makes non-repudiation the default; product cannot opt out without a written opt-out contract |
| **Identity spoofing / impersonation** (agent claims an identity it does not hold) | Policy plane — SPIFFE-rooted non-human identity; A2A — identity propagation across inter-agent hops | Platform enforces identity; no product-side opt-out for cross-tenant hops |
| **Overwhelming the human-in-the-loop** (approval fatigue, bypassed reviews) | Runtime — approval checkpoint primitives; policy plane — approval-workflow rate limits; safety plane — anomaly detection on approval-throughput patterns | Platform exposes the primitives; product sets the specific approval policy within platform limits |
| **Unexpected code execution** | Sandbox (`mod-404`) — syscall policy, egress rules, filesystem capabilities, resource quotas, leak-eviction | Platform owns the sandbox; no product opt-out |
| **Agent-to-agent communication poisoning** | A2A substrate (`mod-409`) — identity propagation, message-schema validation, correlation-ID integrity | Platform enforces the transport contract |
| **Rogue agents** (unregistered or policy-bypassing agent processes in the mesh) | Registry (`mod-403`) + runtime — registration required to join the mesh; policy plane — denies traffic from unregistered principals | Platform enforces registration; no product opt-out |
| **Human attacks on multi-agent systems** / **human-manipulation by agent** | Safety plane — standardised refusal behaviour; observability — anomaly detection on human-interaction patterns; `agentic-safety-engineer` methodology | Platform exposes the surface; `ai-risk-engineer` + `agentic-safety-engineer` author the specifics |

If any of these have shifted or been renamed in the current OWASP revision, update the mapping. The platform *substrate* will stay the same — the policy the substrate runs is what OWASP revises.

## Shared responsibility: who defends what

Think of the platform's security posture as a shared-responsibility model, similar to a cloud-provider shared-responsibility matrix. The platform defends the substrate; the product team defends its product-specific logic; the specialist peer authors the methodology both sides run.

- **Platform defends:** identity, tenant isolation, sandbox, registry governance, gateway quotas, observability trace integrity, baseline input filtering, baseline output validation, policy-engine execution, approval-primitive mechanics.
- **Product team defends:** its own system-prompt confidentiality, its feature-specific input contracts, its domain judges, its risk-tier choices within platform defaults.
- **Specialist peer authors:** the harm taxonomy (`ai-risk-engineer`), the dangerous-capability red-team methodology (`agentic-safety-engineer`), the judge rubrics (`ai-eval-engineer`), the serving-side mitigations (`ai-infra-inference-engineer` for inference-substrate threats).

The onboarding document your platform gives new teams should spell this out. It is one of the highest-leverage docs on the platform: it tells every product team what they can trust the platform to defend and where they still have work to do.

## The regulatory surface this maps onto

Platform engineers do not author regulatory opinion (chapter 01 — that belongs to legal counsel). But the OWASP baseline aligns closely with two regulatory frames that will show up in platform roadmap conversations:

- **NIST AI Risk Management Framework (AI 100-1)** and its **Generative AI Profile (AI 600-1)** — maps the OWASP threat themes onto Govern / Map / Measure / Manage functions. Your platform is the Measure and Manage substrate.
- **EU AI Act Article 15** — accuracy, robustness, and cybersecurity obligations for high-risk AI systems. Your platform is where the technical measures required by Article 15 live.
- **EU AI Act Article 72** — post-market monitoring. Your observability substrate (`mod-408`) is the Article 72 evidence pipeline for every agent that runs on the platform.

Chapter 05 is not trying to teach these frames — the modules downstream do. The point here is that an OWASP-aligned platform is also where the organisation's regulatory evidence comes from. Platform substrate choices (observability schema, audit trail retention, policy-engine logging) are the ones legal and compliance will later ask for. Make them defensible from day one.

## Concrete day-zero checklist for the safety plane

A new agent platform does not need to defend every OWASP category perfectly on day one. It does need to defend a credible minimum. The list below is a reasonable day-zero scope. Each item has a module where it is taught at depth.

1. **Identity on every hop** — non-human identity (SPIFFE / SPIRE), tenant-scoped tokens, short-lived credentials. (`mod-405`)
2. **Registry-only tool invocation** — no agent calls a tool that is not registered with a schema and conformance tests. (`mod-403`)
3. **Sandboxed code execution** — no agent runs code outside the sandbox substrate; egress and syscall policy on by default. (`mod-404`)
4. **Capability-model policy engine** — least-privilege per session, approval workflow for high-risk tool classes. (`mod-405`)
5. **Gateway quotas and budgets** — per-tenant token / cost ceilings, burst budgets, circuit breakers. (`mod-406`)
6. **Trace integrity** — OTel GenAI conventions on every span, immutable trace store, audit trail. (`mod-408`)
7. **Baseline input filter + output validator** — prompt-injection screening and PII redaction before egress. (`mod-411`)
8. **Standardised refusal behaviour** — a shared refusal protocol every product inherits. (`mod-411`)
9. **Regression gate on release** — eval platform refuses to ship a change that regresses the safety fixtures. (`mod-407`, feeding from red-team output owned by `agentic-safety-engineer`)
10. **Written opt-out contract process** — a documented way to deviate from any of the above with sign-off and expiry (chapter 04).

Anything short of this list and your platform is distributing known-OWASP-category risk across every tenant.

## Common pitfalls

- **Treating OWASP categories as a checklist to pass once.** The threats mutate as agent capabilities evolve (the Agentic doc only exists because the LLM Top 10 could not describe what new threats emerged once agents could loop and call tools). Re-read both documents each release cycle.
- **Pushing all defence to prompt altitude.** Prompts drift, product teams rewrite them, and a defence that lived in a prompt disappears quietly. Platform-altitude defences do not evaporate on refactor.
- **Pushing all defence to platform altitude.** Over-platformisation imposes cognitive load on product teams and slows them down. The shared-responsibility model is the whole point.
- **Ignoring the Agentic doc because it is newer and less familiar.** The agentic threats are the ones unique to what this platform exists to run. They are the most important ones for this track.
- **Treating opt-outs as exceptions rather than contracts.** See chapter 04. A policy-plane opt-out without a written contract is a platform defect.

## Summary

- OWASP's *Top 10 for LLM Applications* and *Agentic AI Threats and Mitigations* are the baseline threat taxonomies your platform must defend against in 2026.
- Not every threat defends at the same altitude. Platform-altitude mitigations belong in the registry, sandbox, policy plane, gateway, safety plane, and observability substrates. Product-altitude mitigations belong in prompts and feature UX.
- Specialist peers author the threat-specific methodology (`ai-risk-engineer`, `agentic-safety-engineer`, `ai-eval-engineer`) — the platform runs it.
- The day-zero safety-plane scope is non-negotiable; opt-outs require written contracts (chapter 04).
- The same substrate that defends against OWASP threats produces the regulatory evidence that NIST AI RMF and EU AI Act Article 72 ask for later.

This closes the *position-and-philosophy* module. You now have: the ownership slice (chapter 01), the operating model (chapter 02), the engineering guidance your users arrive with (chapter 03), the discipline to decline work that is not platform scope (chapter 04), and the security baseline the platform must defend from day one (chapter 05). The next module (`mod-402`) starts building the runtime substrate.
