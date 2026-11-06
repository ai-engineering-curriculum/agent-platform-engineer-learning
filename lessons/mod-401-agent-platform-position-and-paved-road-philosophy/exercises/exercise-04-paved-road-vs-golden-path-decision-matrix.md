# exercise-04: Paved-Road vs. Golden-Path Decision Matrix

**Estimated effort:** 2 hours · **Output:** one decision matrix + three worked case verdicts + one opt-out contract · **Pairs with:** chapters 03 and 04.

## Objective

Produce a reusable decision matrix that tells you, for any new piece of agent-engineering work, whether it belongs on the platform's paved road, exists as a golden-path reference, stays with the product team, or belongs to a specialist peer. Then apply the matrix to three real-shaped cases and write one opt-out contract for a case that falls off the paved road.

Vocabulary used throughout this exercise:

- **Paved road** — a maintained, versioned, supported platform surface (SDK, template, substrate) that stream-aligned teams consume through a contract. Platform is on the hook for support and deprecation.
- **Golden path** — a reference implementation or template that shows the *recommended* way to do something, but which is not operationally supported substrate. Easier for teams to diverge from without taking a dependency.
- **Product-owned** — stream-aligned product team owns the work entirely.
- **Specialist-peer-owned** — `ai-eval-engineer`, `ai-risk-engineer`, `agentic-safety-engineer`, `ai-infra-inference-engineer`, etc.

The distinction between paved-road and golden-path matters because every paved-road surface pays the maintenance tax from chapter 04; a golden path pays a documentation tax only.

## Prerequisites

- You have read [`03-anthropic-openai-guidance-and-platform-obligations.md`](../03-anthropic-openai-guidance-and-platform-obligations.md) and [`04-when-not-to-platformise.md`](../04-when-not-to-platformise.md).
- You have skimmed Anthropic's *Building Effective Agents* and OpenAI's *Practical Guide to Building Agents*. Links in both chapter 03 and [`resources.md`](../resources.md).
- You have completed exercises 01–03.

## Problem statement

Build a decision matrix that answers four questions about any proposed platform-adjacent work item:

1. **Scope** — Platform / Product / Specialist-peer?
2. **Altitude** (if platform) — Policy plane / Substrate / SDK template / Reference-only golden path?
3. **Readiness** — Can the platform credibly maintain it at the chosen altitude today, under current staffing and operations maturity (from exercise 03)?
4. **Reversibility** — Is the decision easy to revisit next quarter?

Then apply the matrix to three cases and write one opt-out contract.

## Requirements

### 1. The decision matrix

A table of 10–15 rows, one per decision criterion you consider when sorting work. For each criterion, what the answer *implies* for scope / altitude / readiness / reversibility. The criteria below are a starting list — add your own, challenge any that do not fit your context:

- Number of stream-aligned teams that would consume it in the next two quarters.
- Whether the work is substrate (wiring, policy, template) or product feature.
- Whether consumers integrate through a stable contract or by forking source.
- Whether a specialist peer owns the methodology that drives the work.
- Whether the work is covered by one of the OWASP LLM / Agentic AI threat categories (chapter 05).
- Whether the guidance in Anthropic's *Building Effective Agents* or OpenAI's *Practical Guide* recommends this pattern as a default.
- Maintenance cost once shipped — does the platform have the staffing and operations maturity (exercise 03) to support it through deprecation?
- Whether a simpler non-platform alternative exists that the product team could ship themselves.
- Blast radius if the work goes wrong — does one tenant's mistake affect others?
- Audit / compliance surface — does the work produce evidence a regulator will later ask for?

Each row should produce either a pointed instruction ("if ≥2 consumers and substrate-shaped, promote to platform paved-road"), a disqualifier ("if methodology belongs to a specialist peer, defer regardless of consumer count"), or an altitude signal ("if blast radius crosses tenants, policy-plane altitude; otherwise SDK-template altitude is sufficient").

### 2. Three worked case verdicts

Apply the matrix to three real-shaped cases below. For each, write: (a) which criteria fired, (b) the scope / altitude / readiness / reversibility verdict, and (c) the one-paragraph decision rationale a stream-aligned team would read on receiving the verdict.

- **Case A** — "The refund-copilot team wants the platform to ship an opinionated multi-turn clarification pattern — their agent asks up to three clarifying questions before invoking the refund tool. Two other product teams are interested."
- **Case B** — "The legal copilot team wants the platform to ship a per-document citation-required output validator. No other team has asked for this. The compliance team is interested and would like the validator audited."
- **Case C** — "A senior agentic AI engineer leading the customer-support product proposes that the platform add a 'manager / worker with hand-off to humans' orchestration template based on the OpenAI practical guide. The support product needs it now; one other product (sales assist) has a similar roadmap item next quarter."

Each verdict should be one of: *promote to paved-road platform substrate* · *ship as golden-path reference* · *keep with the product team* · *route to a specialist peer*. State the verdict explicitly, then justify it against the matrix.

### 3. The opt-out contract

For any case that *does not* fall onto the paved road but *does* intersect a platform-enforced substrate (e.g. the platform-default safety plane, the OTel observability schema, the policy plane), write an opt-out contract using the template sketched in chapter 04 — bypass, reason, replacement, sign-off, expiry, blast-radius-limits. Case B is the natural candidate if you follow the matrix honestly, but if your matrix lands differently, pick the case that does fall into this shape.

## Starter guidance

- Resist the matrix sprawling past 15 rows. If you add a row, it must do decision work that another row does not already do.
- Let the Anthropic and OpenAI guidance documents push you toward *more* defaults and *fewer* exceptions on patterns both documents name (routing, parallelisation, orchestrator-workers, evaluator-optimizer, human-in-the-loop). A platform that does not paved-road the patterns both model vendors publish as defaults is failing the stream-aligned teams that read those documents on day one.
- If a case verdict is "specialist-peer-owned", your matrix should say who signs off on the methodology before the platform touches it — do not route to "the eval team" in the abstract; route to `ai-eval-engineer` with the specific artefact (judge rubric, dataset, pass-threshold) they need to deliver.
- Reversibility earns its own cell in the matrix because some decisions (introducing a new SDK primitive) are two-way doors; some (changing the identity model at `mod-405` altitude) are not. Treat them differently.

## Acceptance criteria

You can mark this exercise done when all of these are true:

- [ ] Your matrix has 10–15 criteria, each with a concrete implication for scope / altitude / readiness / reversibility.
- [ ] At least one row in the matrix references OWASP LLM Top 10 or OWASP Agentic AI threats and names policy-plane altitude as a consequence.
- [ ] At least one row references the Anthropic or OpenAI guidance as a source of "this should be a paved-road default".
- [ ] Each of the three case verdicts names its scope and altitude explicitly and is defensible against the matrix.
- [ ] Your opt-out contract has all six sections from the chapter-04 template, including an expiry date.
- [ ] A peer could use your matrix on a new case you did not supply and get a non-random answer.

## Stretch goals

- Take an in-flight work item from your current org (or a sibling AICG track's project) and run it through the matrix. Write the one-page memo you would deliver to that team.
- Add a "promote later" lane to your matrix — the chapter 04 pattern where you deliberately leave work with a product team until a second real example appears, then promote. Capture the specific trigger that would move the item from "product-owned" to "paved-road".
- Draft the deprecation ladder for an existing paved-road surface in your platform (or a hypothetical one). Entries: announce, soft-deprecate with migration doc, hard-deprecate with no new onboarding, remove. Who signs off at each step, and how do you communicate to all currently-adopting teams? This connects directly into `mod-412` on paved-road SDKs and DX.
