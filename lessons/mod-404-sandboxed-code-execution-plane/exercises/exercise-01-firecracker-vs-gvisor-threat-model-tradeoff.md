# exercise-01: Firecracker vs. gVisor Threat-Model Trade-off

**Estimated effort:** 3 hours · **Output:** one architecture memo (3–5 pages) with a decision, a trade-off table, and a two-tier routing rule · **Pairs with:** chapter [01](../01-substrates-and-the-agent-threat-model.md); feeds exercises 02 and 03, and project 401.

## Objective

Produce the memo your platform team would send to a security partner (and your own on-call team) to defend a two-tier sandbox-substrate choice: a **primary tier** that most code-execution calls land on and an **elevated tier** that higher-risk tools route to. The memo argues the choice back against OWASP LLM Top 10 and OWASP Agentic AI Threats and Mitigations — not back against vendor marketing.

A well-written memo is the artefact that keeps this decision from being re-litigated every quarter. The chapter is a map of the design space; the memo is your plane's position on it, written down so the next engineer picking a workload for the plane does not have to re-derive it.

## Prerequisites

- You have read chapter [01](../01-substrates-and-the-agent-threat-model.md) at full depth.
- You have skimmed, at a minimum, [Firecracker's design overview](https://github.com/firecracker-microvm/firecracker/blob/main/docs/design.md) and the [gVisor architecture guide](https://gvisor.dev/docs/architecture_guide/). Reading the gVisor [supported-syscalls matrix](https://gvisor.dev/docs/user_guide/compatibility/linux/amd64/) at least once is required.
- You have the current [OWASP Top 10 for LLM Applications](https://genai.owasp.org/llm-top-10/) open in another tab, plus the [OWASP Agentic AI Threats and Mitigations](https://genai.owasp.org/initiatives/#agentic-security-initiative) landing page. Note the current revision numbers before you start.
- You have a working picture of the kinds of tool workloads your platform will host: code interpreters, shell commands, browser automation, data transforms. Chapter 04 of mod-403 and the project catalogue at the track root are reference points.

## Problem statement

Your platform is standing up its sandbox plane. You have been handed one non-negotiable: the primary tier will not be a managed vendor — some workloads must run on self-hosted substrate for compliance reasons. You still have to decide which self-hosted substrate is the primary tier and which — if any — is the elevated tier for higher-risk workloads.

Write a memo that:

1. Picks a substrate for each tier.
2. Argues the pick against the OWASP threat models at engineering depth (not "security is important").
3. Writes down the routing rule the gateway will apply to decide which tier a given call lands on.
4. Names the trade-offs you are accepting and what you will do if one of them bites.

## Requirements

### Part 1 — Threat-model trade-off table (the main deliverable)

Author a table whose rows are the four threat classes from chapter 01 plus at least four specific items from the current OWASP LLM Top 10 and three from the OWASP Agentic AI list that are relevant to sandbox-execution. Columns are **Firecracker** and **gVisor**. In each cell, state concretely what the substrate does and does not defend against *for that specific threat*. Avoid both marketing superlatives ("state-of-the-art isolation") and hand-wave hedges ("some protection"). If a cell says "defends," it should answer "against what specifically, through which mechanism, under what assumptions."

At a minimum the rows cover:

- Untrusted-code sandbox escape.
- Egress exfiltration (specifically IMDS and arbitrary off-host destinations).
- Resource DoS (CPU, memory, disk, pids, bandwidth).
- Cross-invocation persistence via snapshot or image state.
- LLM01 Prompt Injection, LLM02 Insecure Output Handling, LLM06 Sensitive Information Disclosure, LLM08 Excessive Agency (verify the current item numbering; the list reshuffles across revisions).
- At least three OWASP Agentic items that your workload exposure makes material — e.g., tool misuse, delegated-authority abuse, autonomous-action blast radius, memory poisoning.

Where a defence depends on something *outside* the substrate (egress policy, broker-minted credentials, snapshot build hygiene), say so explicitly and name the chapter / mechanism that owns it.

### Part 2 — Tier assignment and routing rule

Pick a primary tier and (optionally) an elevated tier. For each:

- Name the substrate.
- Name three categories of workload that land on it.
- State the capability-bundle defaults that apply (chapter 02 preview).
- State the admission tests that must pass before a tool is allowed on the tier.

Then write the routing rule, in pseudo-config, that the gateway applies. At minimum the rule reads from the tool's registry entry (mod-403) a `risk_tier` label and a `requested_capabilities` object. The rule's output is `tier: primary` or `tier: elevated` (or `tier: refused` if the request cannot be served).

Explain, in prose, what *triggers* an elevated-tier placement — what fields, from where, decided it. This is the thing the policy plane (mod-405) and the security partner will read.

### Part 3 — Trade-offs you are accepting

Three honest short paragraphs:

- **What am I giving up on the primary tier?** Density, cold start, compatibility — whichever is the actual trade.
- **What am I giving up on the elevated tier?** Cost, latency, operational cost.
- **What happens when the trade-off bites — what is the mitigation ladder?**

"None" is not an answer. If you cannot name a trade-off on a tier, you have not picked a tier.

### Part 4 — What I would need to re-open this decision

A short list of signals that, if observed, would make you re-open the substrate pick. Examples: a published sandbox-escape in the chosen substrate; a workload class whose characteristics your tier assumptions did not anticipate; a vendor-side change that makes a managed-fallback-first posture (chapter 04) economically different; a regulatory change that forces a hardware-boundary tier for a workload class currently on the reduced-syscall tier. The point is to pre-commit to the conditions under which you will revisit — so the decision does not quietly calcify.

## Starter guidance

- Writing against *specific* threats is the exercise's whole point. A cell that says "Firecracker provides strong isolation" is weaker than a cell that says "Firecracker's KVM boundary means a guest-kernel exploit has to also exploit the VMM or KVM to reach the host; the published history of such exploits is X; our assumption is Y." You may not have the data for X; name that gap and estimate what you would need to fill it.
- The Firecracker vs. gVisor decision is not Firecracker vs. gVisor in isolation — it is Firecracker vs. gVisor for *your* workload mix. If your primary workload is model-generated Python snippets against bounded inputs, that is a very different decision than if your primary workload is user-prompted shell commands against production data.
- Do not over-rotate on cold-start numbers in Part 1. Cold start is a chapter-05 problem; it does not defend or attack the threat model. If a cold-start consideration shows up in Part 1, re-read it — it almost certainly belongs in Part 3.
- Mark `<!-- needs-research: ... -->` on any claim you cannot verify against the project docs or the current OWASP revision. The MCP track uses this convention; the sandbox track inherits it.

## Acceptance criteria

Mark done when:

- [ ] The trade-off table has at least 11 rows and each cell is concrete about what the substrate does or does not defend against for that specific threat.
- [ ] A substrate is named for each tier and the routing rule, in pseudo-config, resolves `risk_tier` + `requested_capabilities` to a tier or to `refused`.
- [ ] At least three capability-bundle defaults per tier are specified (even if they are placeholders for the chapter 02 exercise to fill in).
- [ ] The trade-offs section names a trade on each tier and a mitigation ladder for each.
- [ ] The "re-open this decision" section names at least three concrete signals.
- [ ] Every citation to OWASP or to a project doc uses the current revision; unverifiable claims are flagged.

## Stretch goals

- **Add a third tier.** If your workload mix justifies it (e.g., high-density WASM / Pyodide for a specific tool class), add that tier to the memo with its own threat-model row-set, routing predicate, and trade-off paragraph.
- **Attach the first draft of the admission test suite.** Chapter 02 will formalise seccomp profiles and egress policy; your memo can preview the three admission checks per tier that must pass before a tool can land on it.
- **Draft the quarterly re-review calendar.** The signals you named in Part 4 need an owner and a cadence; propose both.
- **Red-team your own memo.** Write a one-paragraph counter-argument from the perspective of your security partner and name how your memo would need to change if they win that argument.
