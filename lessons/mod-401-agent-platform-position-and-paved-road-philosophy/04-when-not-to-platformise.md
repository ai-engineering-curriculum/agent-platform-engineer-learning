# When Not to Platformise

A platform team's instinct is to platformise. That instinct is correct most of the time — it is why the team exists — but there is a tax on every platform abstraction, and some work is cheaper and more honest when it is left out of the paved road. This chapter is about the *out* decision: how to spot work that should not become platform scope, how to say no without blocking the stream-aligned team, and how to write down the opt-out so the decision does not quietly undo itself next quarter.

Anthropic's *Building Effective Agents* is explicit about this instinct: "when building applications with LLMs, we recommend finding the simplest solution possible, and only increasing complexity when needed." Replace "application" with "platform" and the same rule holds for platform work.

## The cost of platformising

Everything you paved-road has to be maintained forever, documented, versioned, supported, deprecated through a ladder, and secured against misuse. Concretely, platformising a thing pays an ongoing tax across:

- **Maintenance** — regressions, upgrades, library churn, model-vendor breaking changes.
- **Documentation** — onboarding guides, templates, troubleshooting pages, migration notes.
- **Support** — office hours, Slack triage, oncall interruptions.
- **Security and policy** — the platform substrate attracts audit scope the one-off code never would.
- **Opportunity cost** — every hour you spend on a marginal paved-road feature is an hour not spent on the Interfaces or Adoption dimension of the CNCF maturity model (chapter 02).

If the benefit the stream-aligned teams get back is smaller than that tax, the right call is to leave the work *outside* the platform.

## Four classes of work that should not become platform scope

### 1. One-off agent products

A single product-line agent that will never have a sibling (the model-upgrade copilot for the internal infra team, the one-shot migration agent that runs for a quarter and then retires, the demo agent for a single conference). These should run on the platform's substrate — they consume runtime, gateway, sandbox, policy, observability, eval — but their *agent logic* belongs to the product team, not to a paved-road template.

The right behaviour is for the product team to use the paved-road SDK template as a starting point and then diverge as the product demands. The wrong behaviour is for the platform to commit to a per-product template fork that it then has to maintain.

Rule: **if you cannot name a second team that would use the template, do not promote it from a product repo into the platform.**

### 2. Experimental prototypes

Research spikes, hackathon prototypes, feasibility experiments, "can we even get this to work" one-week bets. Platformising these is a category error — the point of a prototype is to answer a question cheaply, and every integration with the paved road is a cost the prototype did not need to pay.

The right behaviour is to make the platform friendly to prototyping without being required for it. Local-dev loop (mod-412), sandbox emulation, trace viewer, offline eval replay — these let a prototype use platform tooling without becoming a platform tenant. The eval and observability substrate should be opt-in for prototypes, not mandatory.

Rule: **do not block a two-week prototype on anything the platform would make a six-month tenant do.** Prototypes graduate into platform tenants once they are headed to production, not before.

### 3. Product-specific orchestration that belongs to a senior agentic engineer

This is the most common category in practice, and the hardest to see when you are inside it. Signs:

- The pattern is specific to one product's customer behaviour, one business workflow, or one data model.
- Generalising it to two products would require knobs the second product does not need and the first product does not want to pay for.
- `senior-agentic-ai-engineer` (level 40, your peer) is the right owner — it is product-side multi-agent subsystem leadership.

Example: a multi-agent orchestration that routes refund-eligible customers to one tool-call path and non-refund-eligible customers to another, using a product-specific scoring rule. The scoring rule is product business logic. The *routing substrate* (A2A routing, correlation-ID propagation, circuit breakers) is platform. The *specific routing rule* is product.

Rule: **if generalising the pattern would require the stream-aligned team to document their business logic as platform API, the pattern is not platform scope.** Keep the substrate, decline the specific.

### 4. Policy or compliance work that belongs to a specialist peer

Harm taxonomies belong to `ai-risk-engineer`. Red-team methodology belongs to `agentic-safety-engineer`. Eval judge design belongs to `ai-eval-engineer`. Serving-fleet operation belongs to `ai-infra-inference-engineer`. Legal / privacy opinion belongs to counsel.

The platform integrates the output of all of these — the harm taxonomy becomes guardrail policy (mod-411), the red-team findings become regression fixtures (mod-407), the judge rubrics run on the eval platform, the serving fleet is behind the gateway (mod-406). But the *authoring* of those methodology artefacts is not platform work, and attempting to absorb it is one of the biggest trust-eroding failure modes of agent platforms (see chapter 01).

Rule: **the platform runs the policy; the specialist peer authors it.** If you find yourself writing the thing rather than running infrastructure for it, escalate to the right peer.

## Decision checklist

Before promoting a new thing onto the paved road, walk this checklist. Any "no" is a strong signal to decline or defer:

1. **Is there more than one stream-aligned team that will consume this within the next two quarters?**
2. **Is this substrate — wiring, policy, templates — rather than a product feature?**
3. **Will consumers integrate through a stable interface rather than fork your source?**
4. **Would the specialist peer owner (eval, risk, safety, inference, legal) sign off that the authoring stays with them?**
5. **Can we commit to maintaining this through deprecation (not just launch) with the staffing and operations maturity we have today (chapter 02)?**
6. **Is the alternative — leaving this with the product team or the specialist peer — materially worse for the organisation, not just slightly less consistent?**

A platform roadmap that only adds items passing this checklist stays coherent. A platform roadmap that adds items with two or three "no"s accumulates maintenance debt and surface area that the team eventually has to amputate.

## Opt-out contracts

Sometimes the right answer is "the platform exists, this work stays outside it." Make that an explicit, written contract — not a verbal exception. Write it down because a verbal exception is a platform defect that will be cited as precedent by the next team.

An opt-out contract documents:

- **What is opted out** — which substrate (safety plane, policy plane, eval gate, MCP registry) and for which workload.
- **Why** — the product reason the paved-road default does not fit.
- **What replaces it** — the alternative the opted-out team commits to (in-product guardrails, dedicated sandbox, non-shared eval, direct-call tools with their own policy).
- **Who signs off** — platform owner + risk / safety sign-off if the opt-out touches guardrails.
- **Expiry** — when the contract is reviewed and either renewed, retired, or upgraded back onto the paved road.
- **Blast-radius limits** — explicit "this does not set precedent for the broader platform; policy still applies to other workloads."

A standard opt-out contract template belongs in the platform's internal docs. mod-412 covers the documentation surface; mod-413 covers the cadence that reviews opt-outs.

Example (sketch):

```
title: Opt-out: Legal copilot uses product-local guardrails
workload: legal-copilot (team: LegalTech)
opts_out_of: Platform safety plane (mod-411 standardised guardrails)
reason: The copilot's output must preserve exact statutory quotations,
        which the platform's default output filter redacts as a PII-shaped pattern.
replacement: Legal-authored output validator, reviewed by ai-risk-engineer,
             running in the same Guardrails AI pipeline slot as the platform default.
sign-off: agent-platform-engineer (lead), ai-risk-engineer, Legal counsel
expiry: 2027-05-01 — reviewed quarterly; replaced by platform-level per-tenant
        validator policy when that ships (mod-411 roadmap).
blast-radius: Applies only to this workload. All other tenants continue under
              the platform default. Precedent not set.
```

Three properties make this a *good* opt-out: it names what is bypassed, it names what replaces it with equivalent rigor, and it has an expiry date.

## Reversibility: platformise *later*

The two-way-door form of this decision is: push the work to the product team now, watch what they build, and when a second product team builds something structurally similar, *then* promote the common shape onto the paved road. The promotion happens after the pattern is proven, not before.

Three concrete reasons this works better than preemptive platformising:

- **The abstraction is designed from two real examples**, not from one example extrapolated. Two examples catch roughly the same shape; one example extrapolated catches the shape of the one example and nothing else.
- **The deprecation ladder has somewhere to start.** The first product keeps its code; new entrants get the paved-road version; the first product migrates on a timeline the platform team can staff.
- **You stop subsidising a template no one else will adopt.** If a second team never arrives, the pattern stays with the first team and the platform saves the maintenance tax.

The practical implication for your roadmap: treat "promote a reference implementation into a paved-road template" as a normal, quarterly-cadence operation, not a rare event. mod-412 and mod-413 both lean on this.

## Common pushback and how to handle it

- **"But if you don't take this on, we'll ship a shadow version and you'll have no visibility."** True of a few things (observability, safety plane, policy, registry) — those are exactly the substrates where you should *not* give an opt-out without a replacement. Also wrong of most things: a product-line orchestration that stays in a product repo is not a shadow platform. See it as a product, not a threat to yours.
- **"Why can't you just add this one feature?"** Because the next team will ask for a second one-off, and the one after that will ask for a third, and the sum is an incoherent platform. Point at the decision checklist, say "if we can show a second team that will use this, we'll promote it," and write the request as a backlog item that lists expected second consumers.
- **"We're on a deadline."** Fine — but the deadline is not the platform's deadline. Offer substrate help (sandbox access, gateway tenancy, trace viewer) and a paved-road template that gets them 70% there; the specific 30% stays theirs. Capture what the 30% would look like as a paved-road template, and bring it back to the roadmap after the deadline.

These answers are not confrontations — they are the mechanism that keeps the platform coherent. Say them calmly and write them down.

## Summary

- Platformising has a cost. Not everything that *could* be platformised *should* be.
- Four classes of work routinely should not become platform scope: one-off products, prototypes, product-specific orchestration, and specialist-peer authoring work.
- Use a decision checklist: ≥2 consumers, substrate-not-feature, stable interface, specialist-peer sign-off, maintainable through deprecation, materially better than the alternative.
- When something stays outside the platform, write an opt-out contract — bypass, reason, replacement, sign-off, expiry, blast-radius.
- Prefer platformising *later*: wait for two real examples, then promote the common shape.

The next and final chapter of this module turns to the baseline the platform's *safety plane* must defend against from day one, using OWASP's LLM Top 10 and Agentic AI Threats and Mitigations as the frame.
