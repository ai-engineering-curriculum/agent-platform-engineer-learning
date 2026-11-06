# Position on the Ladder: What the Agent Platform Engineer Owns

Every role on the AI-engineering level ladder touches the same stack — models, agents, tools, sandboxes, evals, guardrails — but owns a different slice of it. If you cannot draw the slice this role owns and point to the roles that own the slices next to it, you will end up either duplicating someone else's work or leaving an un-owned gap in the paved road.

This chapter fixes the boundary. The rest of the track assumes it.

## The one-sentence scope

**The Agent Platform Engineer builds and operates the internal platform other teams ship agents on.** You are not building an agent product. Your users are the engineers who build agent products, and your deliverables are the runtime, registry, sandbox, policy, gateway, eval, observability, safety, and paved-road SDK substrate those engineers consume.

Three tests for whether a piece of work is yours:

1. **Multi-tenant** — more than one product team depends on it, or will within a quarter. If it serves exactly one product indefinitely, it belongs to the product team, not the platform.
2. **Substrate, not feature** — it is wiring, policy, or a template; it does not encode product-specific business logic. "An agent that answers refund questions" is product scope. "A runtime any agent can run on" is platform scope.
3. **Used through a contract, not forked** — product teams integrate against a stable interface (SDK, API, template), not by reading your source and copying patterns. If the dominant consumption pattern is copy-paste, you have a reference implementation, not a platform.

Fail any of these and the work probably belongs to a peer track.

## The ownership rule

This repo's curriculum applies one rule everywhere: **assign primary coverage to the lowest-level role that genuinely requires the skill; higher-level tracks link to that owner unless additional platform scope, depth, or leadership context is required.** See `JOB_REQUIREMENTS.md` → *Methodology* for the full statement.

Applied to this role:

- Agent *fundamentals* (ReAct loop, tool-call mechanics, single-agent patterns, framework comparison, agent memory basics) are owned by `agentic-ai-engineer` at level 30. This track does not re-teach them — it assumes them.
- Agent *platform* substrate (runtime, MCP registry, sandbox, policy plane, model gateway, eval platform, observability, safety plane, paved-road SDK) is owned here at level 40.
- Agent *system design across product lines* is owned by `agentic-systems-architect` at level 48. That architect designs; this role implements the paved-road substrate the design assumes.

The point of the ladder is that the person reading an Agent Platform Engineer job description will see tasks drawn from each of those altitudes. Your job is to know which altitude the work actually belongs to, so you deliver the platform slice without taking on the product-build slice or the architecture-design slice by mistake.

## The eight peer tracks you must know

Internalise what each of these roles owns, because a missed handoff with any of them becomes a platform defect.

### Down-ladder (your prerequisites)

| Role | Level | What they own | Interface to you |
|------|-------|---------------|------------------|
| `agentic-ai-engineer` | 30 | Build-altitude agent fundamentals: ReAct, tool-call mechanics, framework comparison (LangGraph, CrewAI, AutoGen, OpenAI Agents SDK, Google ADK, Anthropic Agent SDK, smolagents), single-agent and single-MCP-server patterns, basic agent memory | Consumes your paved-road SDK and templates. If an L30 engineer cannot get an agent to a dev loop in an hour using your scaffolding, your platform has failed its lowest-level customer. |
| `ai-eval-engineer` | 30 | Application-layer eval methodology: judge calibration, RAG eval, online eval, dataset quality, cost / latency / quality trade-offs | You ship the eval *platform* (dataset versioning, trace / replay store, CI gates — mod-407). They ship the eval *methodology* that runs on it. If you find yourself tuning a judge rubric, you have crossed into their lane. |
| `ai-risk-engineer` | 25 | Harm-model authoring, guardrail-effectiveness methodology, risk-tier definition | You operate guardrails-as-a-service (mod-411). They specify what the guardrails must block and how effectiveness is measured. If you find yourself writing the harm taxonomy, you have crossed into their lane. |

### Sideways (peers at level 35–40)

| Role | Level | What they own | Interface to you |
|------|-------|---------------|------------------|
| `ai-infra-inference-engineer` | 35 | Inference-serving substrate (vLLM, SGLang, TRT-LLM), autoscaling, KV-cache, SLO engineering for serving | Your model gateway (mod-406) routes to their serving layer. You do not operate serving; they do not operate the gateway. The contract is the gateway's provider adapter plus the latency / cost telemetry both sides agree on. |
| `senior-agentic-ai-engineer` | 40 | Product-side multi-agent subsystem implementation leadership | Your primary stream-aligned customer. They lead product-side subsystems for a line of business; you ship the platform their team's engineers consume. Expect them to push back when a paved-road default does not fit their product — that pushback is the signal you are drifting from substrate into product. |
| `agentic-safety-engineer` | 40 | Frontier-agent red-team methodology, dangerous-capability evaluation | They produce red-team findings and dangerous-capability eval results. You integrate those findings into the platform's safety plane and the regression-eval fixtures (mod-407, mod-411). You do not run the red-team; they do not operate the safety plane. |

### Up-ladder (your escalation and strategy layers)

| Role | Level | What they own | Interface to you |
|------|-------|---------------|------------------|
| `ai-infra-architect` | 45 | Cross-stack infra architecture, build-vs-buy portfolio strategy across the whole infra stack | Approves platform-shape decisions that touch other infra portfolios (serving, data, SRE). You own the agent-platform portfolio inside their frame. |
| `agentic-systems-architect` | 48 | Agentic-system architecture across product lines: orchestration patterns, framework build-vs-buy at portfolio scope, cross-product consistency | They design; you implement the paved-road that makes the design executable. You bring back implementation constraints (cost, reliability, safety) that must feed back into the design. |

If a work item fits none of these peer tracks and is not substrate, it probably should not exist as a platform team deliverable at all. See chapter 04.

## Boundary examples

Concrete calls are the easiest way to internalise the rule. For each row below, the second column is the primary owner.

| Work item | Primary owner | Why |
|-----------|---------------|-----|
| Stand up a multi-tenant agent runtime on Temporal with checkpoint / resume | `agent-platform-engineer` (you) | Multi-tenant substrate, consumed through an SDK. |
| Write the ReAct loop for the refund-processing agent | `agentic-ai-engineer` | Build-altitude product work. Agent fundamentals, not platform. |
| Lead the multi-agent architecture for the customer-support product line | `senior-agentic-ai-engineer` | Product-side subsystem leadership, not platform. |
| Decide whether the company standardises on LangGraph, OpenAI Agents SDK, or ADK across product lines | `agentic-systems-architect` (with your implementation input) | Portfolio-scope architecture decision. |
| Add a registry entry + conformance tests for a new MCP server | `agent-platform-engineer` | Platform substrate. |
| Author the judge rubric for the refund-answer eval | `ai-eval-engineer` | Eval methodology. |
| Ship the eval platform the judge rubric runs on | `agent-platform-engineer` | Eval substrate. |
| Author the harm taxonomy for the chat product | `ai-risk-engineer` | Methodology, not platform. |
| Ship the input-filter / output-validator service that enforces the harm taxonomy | `agent-platform-engineer` | Platform substrate, policy-plane altitude. |
| Red-team a new Claude or GPT model for dangerous capability | `agentic-safety-engineer` | Frontier-model methodology. |
| Wire the red-team findings into the platform's regression-eval fixtures | `agent-platform-engineer` | Platform substrate consuming the methodology output. |
| Operate the vLLM serving fleet | `ai-infra-inference-engineer` | Serving substrate. |
| Route traffic from agents to the serving fleet with per-tenant quotas and cost attribution | `agent-platform-engineer` | Gateway substrate. |

Take a moment and run the three-test from the top of this chapter against any row you find surprising.

## Common boundary mistakes

Three mistakes are frequent enough to call out explicitly, because every one of them erodes platform trust once it happens.

1. **"We'll write the first agent for them."** You ship a paved-road SDK, a product team cannot get it running, and you end up writing their agent. Now you own the product. Next quarter the paved road breaks and the product team waits for you to fix it rather than fixing it themselves, because it was never their code. Fix: office hours, migration playbooks, and deprecation ladders (mod-412) — not ghostwriting the product.
2. **"We'll run the eval judges for them."** You ship an eval platform, a product team doesn't have an eval engineer, and you end up writing their judge rubrics. Now you own eval methodology for that product line, and your platform roadmap stalls. Fix: hand off to `ai-eval-engineer`; if there is none, escalate up the ladder rather than absorb the gap.
3. **"We'll let this one product bypass the policy plane."** A high-priority product team asks for an exception to the MCP registry or the safety plane, and you grant it as a one-off. The exception is now permanent, the audit trail has a gap, and the next product team cites the precedent. Fix: paved-road versioning and explicit opt-out contracts (chapter 04). An exception that is not documented as an opt-out contract is a platform defect.

## What this means for the rest of the track

The remaining twelve modules build out the slices this chapter named:

- mod-402 — runtime
- mod-403 — MCP registry and tool gateway
- mod-404 — sandboxed code execution
- mod-405 — tool and permission policy
- mod-406 — model gateway
- mod-407 — eval platform
- mod-408 — observability
- mod-409 — A2A routing
- mod-410 — SRE for the agent platform
- mod-411 — safety plane
- mod-412 — paved-road SDKs and developer experience
- mod-413 — platform leadership

Every one of them has an interface to one or more of the peer tracks above. If you hit something during the rest of the curriculum where you cannot tell whether the work is yours or a peer's, come back to this chapter.

## Summary

- The Agent Platform Engineer builds internal substrate other agent engineers consume. Multi-tenant, substrate-not-feature, interface-not-fork.
- Ownership is governed by the lowest-level-owner rule. Agent fundamentals belong down at level 30; platform substrate belongs here at level 40; cross-product design belongs up at level 48.
- Eight peer tracks define your handoffs. Every un-owned gap or duplicated effort is traceable to a missed handoff with one of them.
- The common failure modes are absorbing product build, eval methodology, or safety methodology into the platform. Each one erodes trust and stalls the roadmap.

The next chapter translates this ownership slice into an operating model, using the CNCF Platform Engineering Maturity Model and Team Topologies as the frame.
