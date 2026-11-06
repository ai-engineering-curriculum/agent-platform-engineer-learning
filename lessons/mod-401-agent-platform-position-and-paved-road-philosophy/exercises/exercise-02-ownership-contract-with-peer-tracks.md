# exercise-02: Ownership Contract With Peer Tracks

**Estimated effort:** 2 hours · **Output:** eight short written contracts (one per peer track) · **Pairs with:** chapter 01 (and sets up chapter 03).

## Objective

Turn the ownership rule from chapter 01 into eight concrete, signable contracts — one for each peer track the Agent Platform Engineer interfaces with. Every contract should name what the platform delivers, what the peer delivers back, and what the handoff looks like in an incident. These are the operating-cadence artefacts that stop the "whose work is this?" argument from happening in the moment.

## Prerequisites

- You have read [`01-position-on-the-ladder.md`](../01-position-on-the-ladder.md).
- You have completed [`exercise-01-platform-vs-product-boundary-mapping.md`](exercise-01-platform-vs-product-boundary-mapping.md) — your boundary map is the raw material this exercise formalises.
- You have skimmed the CURRICULUM.md / PREREQUISITES.md pages that describe the eight peer tracks. (If any peer track's purpose is still unclear to you, re-read chapter 01's peer-track table.)

## Problem statement

For each of the eight peer tracks below, write a one-page *ownership contract* that names the interface between the Agent Platform Engineer and that peer. Each contract is the working-agreement version of the boundary your exercise-01 diagram drew.

The eight peer tracks:

1. `agentic-ai-engineer` (L30) — build-altitude agent fundamentals; consumer of your paved-road SDK.
2. `ai-eval-engineer` (L30) — application-layer eval methodology.
3. `ai-risk-engineer` (L25) — harm taxonomy and guardrail-effectiveness methodology.
4. `ai-infra-inference-engineer` (L35) — inference-serving substrate.
5. `senior-agentic-ai-engineer` (L40) — product-side multi-agent subsystem leadership.
6. `agentic-safety-engineer` (L40) — frontier red-team methodology.
7. `ai-infra-architect` (L45) — cross-stack infra architecture and portfolio build-vs-buy.
8. `agentic-systems-architect` (L48) — agentic-system architecture across product lines.

## Requirements

Each contract must have all eight of the following sections, kept short — one or two lines per section is usually enough.

1. **Peer track** — the name, level, family (Engineering / Infra / Governance).
2. **What the platform delivers to them** — the specific substrate, API, SDK template, or evidence pipeline they consume from the platform.
3. **What they deliver to the platform** — the artefact, methodology output, policy, or signal the platform integrates.
4. **Where the contract lives** — the substrate module in this track where the interface is implemented. Cite the module slug (e.g. `mod-407` for eval platform).
5. **How disagreements are resolved** — the escalation path when platform and peer disagree on scope, timeline, or policy. Usually an architect track (`agentic-systems-architect` or `ai-infra-architect`).
6. **Incident handoff** — in an incident spanning platform and peer scope, who holds incident command, who holds the comms, and what each side owes the other during the active incident.
7. **Cadence** — how often you talk to this peer when nothing is on fire (quarterly roadmap sync, monthly op review, release train, etc.). If the peer track does not yet exist in your org, say so and note where the responsibility lands until it does.
8. **Failure modes** — the two or three most likely ways the contract breaks down (platform absorbs the peer's methodology work, peer bypasses the platform substrate, etc.) and the early signal to watch for.

## Starter guidance

- Do not invent interfaces that this track's twelve technical modules do not deliver. For each contract, the "what the platform delivers" column should cite one or more real substrate modules.
- Use chapter 01's peer-track tables as the starting scaffold — those tables already name the interface shape for each peer.
- For peer tracks that your org has not staffed (common for `agentic-safety-engineer` and `ai-eval-engineer` in early-stage orgs), be explicit about who covers the peer's slice in the interim and what the trigger for staffing the role would be. Do not simply fold it into the platform — that is exactly the failure mode chapter 01 flags.
- The contracts do not need to be formal RACI matrices, and a one-sentence section is better than a half-page one. The goal is a document a peer can read in five minutes and recognise as fair.
- A short worked example is often useful: for the `ai-infra-inference-engineer` contract, the "incident handoff" section might read something like "gateway-side errors (routing, auth, quota) — platform IC. Serving-side errors (p99 latency, timeout rate, 5xx from serving) — inference-engineer IC. If we cannot agree in 10 minutes, escalate to `ai-infra-architect`."

## Acceptance criteria

You can mark this exercise done when all of these are true:

- [ ] All eight contracts exist, each on one page or shorter.
- [ ] Every "what the platform delivers" is tied to a specific substrate module (`mod-402` … `mod-413`) rather than vague ("we deliver the platform").
- [ ] Every "what they deliver back" is a concrete artefact — a dataset, a harm taxonomy, a judge rubric, a red-team report, an SLO, a design doc — not a feeling.
- [ ] The incident-handoff section for each contract names *who holds IC* and *who holds comms* unambiguously.
- [ ] You have identified at least one peer track your org has not yet staffed (if applicable) and written down the interim arrangement.
- [ ] You can walk a colleague through any one of the eight contracts in under three minutes.

## Stretch goals

- Take one of the eight contracts and redraft it as if it were a real internal document — complete with "owners", "last reviewed", "next review", and a specific first stakeholder on the peer side (named role, not a person).
- Compare two contracts that superficially look similar — for example, `ai-eval-engineer` vs. `ai-risk-engineer`, or `senior-agentic-ai-engineer` vs. `agentic-systems-architect`. Articulate in one paragraph how the two contracts are *different* even though both peers consume platform substrate.
- Draft the opposite document: the "contract from the peer's side" for one peer, written as if you were the peer explaining what the platform owes them. Read it back as the platform engineer. Did your platform-side contract match?
