# mod-401 — Agent Platform Engineer: Position, Scope, and Paved-Road Philosophy

**Estimated effort:** 10 hours · **Position in track:** first module after orientation (see [`../00-your-relationship-to-the-model`](../00-your-relationship-to-the-model)).

This module fixes the *what* and the *how* of the role before you touch any substrate. The next twelve modules build out specific planes of the agent platform (runtime, MCP registry, sandbox, policy, gateway, eval, observability, A2A, SRE, safety, SDK, leadership); this module is the one that stops you taking on work that is not yours while they do.

## Learning objectives

- Diagram the role's exact scope on the AI-engineering level ladder — what this role owns vs. what `agentic-ai-engineer` (level 30), `senior-agentic-ai-engineer` (level 40), `agentic-systems-architect` (level 48), `ai-infra-architect` (level 45), `ai-infra-inference-engineer` (level 35), `ai-eval-engineer` (level 30), `ai-risk-engineer` (level 25), and `agentic-safety-engineer` (level 40) own.
- Apply the CNCF Platform Engineering Maturity Model and Team Topologies to characterise the platform team's operating model (stream-aligned product teams as customers of a platform team).
- Read Anthropic's *Building Effective Agents* and OpenAI's *Practical Guide to Building Agents* as the engineering guidance stream-aligned teams onboard with — and decide what the platform must therefore paved-road.
- Reason about when NOT to platformise (one-off agent products, experimental prototypes, product-specific orchestrations better owned by `senior-agentic-ai-engineer`).
- Position OWASP Top 10 for LLM Applications and OWASP Agentic AI Threats and Mitigations as the baseline the platform's safety plane must defend against at policy-plane altitude.

## Chapters

1. [`01-position-on-the-ladder.md`](01-position-on-the-ladder.md) — Where this role sits, the eight peer tracks it interfaces with, worked boundary examples, and the common failure modes that erode platform trust.
2. [`02-cncf-maturity-and-team-topologies.md`](02-cncf-maturity-and-team-topologies.md) — The platform-as-a-product shape from Team Topologies, the four CNCF maturity dimensions (Investment, Adoption, Interfaces, Operations), and how to use them to plan the *next step* for the platform.
3. [`03-anthropic-openai-guidance-and-platform-obligations.md`](03-anthropic-openai-guidance-and-platform-obligations.md) — What your users read before onboarding, and the translation table from each piece of guidance to a concrete platform obligation in a later module.
4. [`04-when-not-to-platformise.md`](04-when-not-to-platformise.md) — The four classes of work that should not become platform scope, the decision checklist, and the written opt-out contract pattern.
5. [`05-owasp-baseline-for-the-safety-plane.md`](05-owasp-baseline-for-the-safety-plane.md) — OWASP LLM Top 10 and OWASP Agentic AI Threats and Mitigations mapped onto platform substrate at policy-plane altitude, plus the day-zero safety scope every platform must defend.

## Exercises

Each exercise is designed to take ~2 hours and turns one chapter into a durable artefact you keep for the rest of the track.

- [`exercises/exercise-01-platform-vs-product-boundary-mapping.md`](exercises/exercise-01-platform-vs-product-boundary-mapping.md) — Platform vs. product boundary mapping. Pairs with chapter 01.
- [`exercises/exercise-02-ownership-contract-with-peer-tracks.md`](exercises/exercise-02-ownership-contract-with-peer-tracks.md) — Ownership contract with the eight peer tracks. Pairs with chapter 01 and sets up chapter 03.
- [`exercises/exercise-03-cncf-platform-maturity-self-assessment.md`](exercises/exercise-03-cncf-platform-maturity-self-assessment.md) — CNCF Platform Engineering Maturity self-assessment. Pairs with chapter 02.
- [`exercises/exercise-04-paved-road-vs-golden-path-decision-matrix.md`](exercises/exercise-04-paved-road-vs-golden-path-decision-matrix.md) — Paved road vs. golden path decision matrix. Pairs with chapters 03 and 04.

## Resources

External references are catalogued in [`resources.md`](resources.md).

## How to approach this module

1. Read all five chapters once, then do the exercises in order.
2. Keep the four exercise artefacts — boundary map, ownership contract, CNCF self-assessment, decision matrix — somewhere you can revisit at the end of the track. These are the artefacts you will return to every time a subsequent module raises a scope question.
3. When later modules introduce a new substrate (runtime, registry, sandbox, …), return to chapter 03's translation table to confirm which piece of OpenAI / Anthropic guidance it is implementing.
