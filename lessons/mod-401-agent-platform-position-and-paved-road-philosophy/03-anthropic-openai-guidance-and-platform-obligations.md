# The Engineering Guidance Your Users Read: Anthropic + OpenAI

Stream-aligned teams building agents in 2026 arrive onboarding your platform having read two documents:

- **Anthropic — *Building Effective Agents*** (Schluntz & Zhang, Anthropic Engineering blog): <https://www.anthropic.com/research/building-effective-agents>
- **OpenAI — *A Practical Guide to Building Agents*** (OpenAI, business guides and resources): <https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf>

Both are short, both are opinionated, and both are the engineering vocabulary your users will use when they describe what they want to build. If your platform's defaults contradict them, you will spend every onboarding call defending the discrepancy rather than shipping. If your platform's defaults implement them, onboarding is a question of pointing the user at the paved-road template that matches the pattern they already have in mind.

Read both documents before continuing with this chapter.

## The takeaways the user will arrive with

### From Anthropic's *Building Effective Agents*

1. **"Agent" is a spectrum, not a binary.** On one end, *workflows* — LLM calls orchestrated along predictable paths. On the other end, *agents* — systems where the LLM dynamically directs its own processes and tool use. The right choice depends on the task.
2. **Start simple.** Use the simplest pattern that works. Reach for frameworks and agents only when the task genuinely requires them, because every layer of abstraction between the developer and the model is an extra place to debug.
3. **A small set of composable patterns covers most of what teams need.** Prompt chaining, routing, parallelisation, orchestrator-workers, evaluator-optimizer, and the "augmented LLM" building block (model + tools + retrieval + memory).
4. **Autonomous agent loops are the pattern that needs the most engineering investment.** They require good tooling (clear tool descriptions, environment affordances the model can discover), sandboxing, and strong observability.

### From OpenAI's *Practical Guide to Building Agents*

1. **An agent is a system that uses an LLM to accomplish tasks on behalf of a user.** The guide names three building blocks: a model, a set of tools, and instructions.
2. **Pick the smallest-capable model that works, and only escalate when the task demands it.** Model choice is a first-class design decision.
3. **Tools are the agent's interface to the world; their descriptions and schemas are a first-class design artefact.**
4. **Orchestration patterns range from single-agent to manager/worker to handoff topologies.** The guide gives concrete decision criteria for which to use.
5. **Guardrails are mandatory, not optional.** The guide names input/output filtering, relevance classification, safety classification, PII filtering, moderation, and tool-risk classification as baseline categories.
6. **Human-in-the-loop is the escape hatch for high-risk actions, and should be designed in, not bolted on.**

These two documents do not contradict each other on anything substantive. Treat them as the same guidance in two voices.

## The translation table: guidance → platform obligation

Every point above has a "what must the platform paved-road" answer. The table below is the core artefact of this chapter. Each row maps a piece of guidance that your users read to a concrete platform obligation and the module where this track teaches you how to deliver it.

| User-facing guidance | Platform obligation | Track coverage |
|----------------------|---------------------|----------------|
| "Start with the simplest pattern." | Ship paved-road templates for prompt-chain and single-agent patterns alongside the full autonomous-agent template, so simple is actually the easy default. | `mod-412` |
| "Pick the smallest-capable model." | Model gateway must make model choice declarative (per-step, per-tenant) and attribute cost and latency per model so teams can measure the trade-off they chose. | `mod-406` |
| "Tool descriptions and schemas are a first-class artefact." | MCP registry enforces schema governance, versioning, deprecation, and conformance tests. Not optional. | `mod-403` |
| "Autonomous loops need sandboxing." | Multi-tenant sandboxed code-execution plane with syscall policy, egress rules, resource quotas, and leak-eviction policy. Not a developer-provided dev container. | `mod-404` |
| "Autonomous loops need observability." | OTel GenAI semantic conventions on every span, cross-plane trace propagation (runtime → gateway → MCP → sandbox → eval), standard dashboards and alerting profiles. | `mod-408` |
| "Guardrails are mandatory." | Platform safety plane — input filters, output validators, tool-invocation policy enforcement, standardised refusal behaviour — as a shared service, not re-implemented per product. | `mod-411` |
| "Tool-risk classification and approval for high-risk actions." | Tool and permission policy plane (OPA / Rego, Cedar, SPIFFE), approval workflows, capability model enforcement. | `mod-405` |
| "Human-in-the-loop as a first-class pattern." | Agent runtime supports pause / resume / interrupt / approval checkpoints as durable-execution primitives, not as application-level state that products reinvent. | `mod-402` |
| "Routing, parallelisation, orchestrator-workers topologies." | Agent runtime and A2A substrate support these topologies with correlation-ID propagation, identity delegation, bulkheads, and circuit breakers. | `mod-402`, `mod-409` |
| "Evaluator-optimizer pattern." | Eval platform exposes templates for the pattern plus a trace / replay store the optimizer loop can read. | `mod-407` |
| "Prompt + tool + retrieval + memory = the augmented LLM building block." | Paved-road SDK composes these cleanly; memory and retrieval have platform-backed defaults. | `mod-412` (with `mod-402` for session state) |
| "Choose model and framework per task." | Gateway abstraction and SDK template plumbing let teams switch models and frameworks without forking the paved road. | `mod-406`, `mod-412` |

If a piece of guidance is not in your platform's paved-road, your users will build it themselves — six times, incompatibly. That is the failure mode this chapter exists to prevent.

## What *not* to paved-road

Both documents are also useful for spotting work you should deliberately not platformise. Anthropic's guide is explicit that frameworks add abstraction cost, and that teams should "measure performance and instrument your agents" rather than relying on framework defaults. Three concrete implications for the platform:

- **Do not paved-road a single agent framework as the only option.** Teams will have reasons to pick LangGraph, OpenAI Agents SDK, Anthropic Agent SDK, Google ADK, Microsoft Agent Framework, or smolagents based on the task. The platform should make several work well, not pick one and foreclose the rest. (The paved-road SDK layer in `mod-412` is the integration surface that keeps this trade-off tractable.)
- **Do not paved-road agent prompts.** Prompts are product. The eval harness scaffolding (mod-407) and observability spans (mod-408) are platform.
- **Do not paved-road the agent's business logic or its judge rubrics.** Both are product and eval-engineer-owned (see chapter 01).

Chapter 04 goes deeper on where the "do not platformise" line sits.

## A worked example: onboarding a new team against the guidance

A stream-aligned team arrives wanting to ship a research-assistant agent. Their team lead has read both guidance documents. The onboarding conversation goes something like this, with every hand-off pointing at a platform substrate:

> **Team lead:** "We want a routing step at the front, then parallel subagents for different source types, then an orchestrator that synthesises."
>
> **You:** "That is OpenAI's manager/worker + Anthropic's orchestrator-workers pattern. There is a paved-road template for it in the SDK catalog — this scaffold here. It runs on the agent runtime (mod-402) with checkpoint / resume on every step."
>
> **Team lead:** "We want it to be able to run code to query APIs."
>
> **You:** "Tools go through the MCP registry (mod-403). The sandbox substrate (mod-404) runs the code. The policy plane (mod-405) decides which tools this agent can call and whether any need approval. Register your tool here, the registry CLI will run the conformance tests."
>
> **Team lead:** "We want to pick Claude for the orchestrator and a cheaper model for the subagents."
>
> **You:** "Model choice is per-step in the SDK template; the gateway (mod-406) does the routing and cost attribution."
>
> **Team lead:** "How do we know it's getting better over time?"
>
> **You:** "The eval platform (mod-407) has templates for evaluator-optimizer and a trace / replay store. Your `ai-eval-engineer` writes the judges; we host the infrastructure. CI gates on the eval."
>
> **Team lead:** "What about guardrails?"
>
> **You:** "The safety plane (mod-411) wraps every step by default. If you need a product-specific guardrail on top, it plugs into the same pipeline. OWASP LLM Top 10 and OWASP Agentic AI threats are the baseline — chapter 05 of mod-401 covers which ones the platform defends at policy altitude and which ones your product has to defend at prompt / UX altitude."
>
> **Team lead:** "How do we see what the agent is doing?"
>
> **You:** "OTel GenAI conventions (mod-408) are on by default; the dashboards and the trace viewer come for free."

That whole conversation is the paved road in action. Every answer is a module in this track, and the user has not had to read any of them because the SDK template encodes the answer.

Internalise that this is the operating target. A user who has read the two guidance documents and finds every pattern they need already paved-roaded by your platform is a Level 3 adoption signal (see chapter 02). A user who has to read the twelve substrate modules and wire it all together themselves is a Level 1 adoption signal.

## Keeping up as the guidance evolves

Both documents are living artefacts; the model vendors will update them. Two consequences for the platform:

1. **Treat the latest guidance as a backlog signal.** When Anthropic or OpenAI adds a new recommended pattern (think: the addition of orchestrator-workers to the pattern catalog, or future additions like structured planning or background-task execution), ask whether the paved-road still covers it. If it does not, open a platform roadmap item.
2. **Track breaking-change signals in SDK releases.** The guidance documents are downstream of what the SDKs ship. When Anthropic's Agent SDK or OpenAI's Agents SDK changes a default, the onboarding conversation above changes. Keep your paved-road templates on a release cadence that tracks these SDKs.

The platform-leadership module (`mod-413`) builds this into the quarterly cadence.

## Summary

- Stream-aligned teams onboard your platform having read Anthropic's *Building Effective Agents* and OpenAI's *Practical Guide to Building Agents*. Both are short, opinionated, and drive the vocabulary your users use.
- Every substantive point in those documents maps to a platform obligation — runtime primitives, registry, sandbox, gateway, policy, eval, observability, safety, SDK. The table above is the translation.
- Do not paved-road things the guidance explicitly says should stay flexible: single-framework lock-in, prompts, business logic, judge rubrics.
- The onboarding target is: a user who has read the guidance finds every pattern already paved-roaded. That is what Level 3 adoption looks like.

The next chapter turns the question around: given that platformising has a cost, when should you deliberately *not* platformise?
