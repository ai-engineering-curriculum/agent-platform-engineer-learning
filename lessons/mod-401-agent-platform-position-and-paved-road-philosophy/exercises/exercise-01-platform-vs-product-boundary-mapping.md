# exercise-01: Platform vs. Product Boundary Mapping

**Estimated effort:** 2 hours · **Output:** one boundary-map diagram + one ownership-rule write-up (≈1 page) · **Pairs with:** chapter 01.

## Objective

Produce a concrete boundary map that shows what the Agent Platform Engineer owns, what each peer track owns, and every handoff between them — for a real agent system you can read about in depth. The map and the write-up become the reference you consult every time a later module raises a "whose work is this?" question.

## Prerequisites

- You have read [`01-position-on-the-ladder.md`](../01-position-on-the-ladder.md).
- You have skimmed [`CURRICULUM.md`](../../../CURRICULUM.md) and [`JOB_REQUIREMENTS.md`](../../../JOB_REQUIREMENTS.md).
- You can access one real agent system in enough detail to describe its substrates. One of the following works well:
  - An open-source agent platform (e.g. [LangGraph Platform](https://langchain-ai.github.io/langgraph/concepts/langgraph_platform/), [Dapr Agents](https://docs.dapr.io/concepts/agents/), [Vertex AI Agent Engine](https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/overview), [AWS Bedrock Agents](https://docs.aws.amazon.com/bedrock/latest/userguide/agents.html)).
  - A vendor case study or engineering blog post that describes an agent system in production detail.
  - Your employer's own agent system, if you are already working on one.

## Problem statement

Pick one real agent system. Draw its architecture, overlay the ownership slices from this module onto it, and write down every handoff that crosses a slice boundary. Then apply the three-test from chapter 01 — multi-tenant, substrate-not-feature, interface-not-fork — to each substrate and note where the real system passes, fails, or is ambiguous.

## Requirements

Your deliverable has three parts.

### 1. The boundary-map diagram

A diagram (hand-drawn, [tldraw.com](https://tldraw.com/), Excalidraw, Mermaid, draw.io — whatever you prefer) that shows:

- The agent product in the middle: the specific agent(s) product teams build in this system.
- The platform substrate around it: at minimum runtime, MCP / tool registry, sandbox, policy plane, model gateway, eval platform, observability, safety plane, SDK.
- The peer tracks around that: `agentic-ai-engineer`, `senior-agentic-ai-engineer`, `agentic-systems-architect`, `ai-infra-architect`, `ai-infra-inference-engineer`, `ai-eval-engineer`, `ai-risk-engineer`, `agentic-safety-engineer`.
- Every arrow across a boundary labelled with *what* crosses it (an artefact, a decision, a contract, a telemetry stream).

Colour-code or shade the Agent Platform Engineer's slice so it is unambiguous.

### 2. The ownership-rule write-up (≈1 page)

For each of these ten questions about the real system, write one or two sentences:

1. Who owns the agent's prompts, business logic, and judge rubrics?
2. Who owns the runtime substrate the agent runs on?
3. Who owns the tool registry and the schema-governance policy?
4. Who owns the sandbox syscall policy and egress rules?
5. Who owns the per-tenant model-routing policy and cost attribution?
6. Who owns the eval harness and the regression gates?
7. Who owns the OTel trace schema and the observability backends?
8. Who owns the harm taxonomy the safety plane enforces?
9. Who owns the red-team fixtures that gate releases?
10. Who owns the serving substrate the model gateway routes to?

If the real system has a different team shape than this track assumes, say so — note the mismatch and explain how work would re-slot under the track's ownership rule.

### 3. The three-test audit

Pick three substrates in the real system and apply the three-test from chapter 01 (multi-tenant · substrate-not-feature · interface-not-fork) to each one:

| Substrate | Multi-tenant? | Substrate not feature? | Interface not fork? | Verdict |
|-----------|---------------|------------------------|---------------------|---------|
| … | … | … | … | Platform / Product / Ambiguous |

Note every "ambiguous" with a one-sentence explanation of why the three-test does not cleanly answer the question for this system.

## Starter guidance

- Start by listing every component in the real system. Do not try to fit it into the ownership grid yet. Once the list is complete, *then* colour in who owns what.
- If a component in the real system clearly fits a peer track's scope (serving, eval methodology, safety methodology, product business logic), the arrow into / out of it must be labelled. Those are the most important boundaries to get right.
- If a component does not fit any peer track or the platform, mark it. That is where the three-test from chapter 01 does its job.
- When in doubt, prefer specific over general: "refund agent prompt" is a better label than "agent logic", "pgvector retrieval store" is better than "retrieval".

Read [Team Topologies — platform team pattern](https://teamtopologies.com/) for a short refresher on what a healthy stream-aligned ↔ platform boundary looks like.

## Acceptance criteria

You can mark this exercise done when all of these are true:

- [ ] Your diagram is readable in under 60 seconds by someone who has not read the real system's docs.
- [ ] Every arrow that crosses a boundary has a concrete label — artefact, decision, telemetry, or contract — not "collaborates with".
- [ ] You can defend each of the ten write-up answers in two sentences without looking at the chapter again.
- [ ] Your three-test audit flags at least one ambiguity. If every substrate is a clean platform-or-product verdict, the audit is probably too lenient — try three different substrates.
- [ ] You can name, from memory, the peer track you would route each ambiguity to and the specific contract you would propose with them.

## Stretch goals

- Re-do the exercise for a *second* real agent system from a different domain (customer-support copilot vs. coding agent vs. research agent). Compare where the boundaries fell differently and why.
- Draft an opt-out contract (chapter 04 pattern) for the most ambiguous substrate in your audit — written as if a product team wanted to bypass it. A later exercise (exercise-04) builds on this.
- If your employer runs an agent platform, bring the boundary map to a 1:1 with a peer on another track and see whether they agree with the labels.
