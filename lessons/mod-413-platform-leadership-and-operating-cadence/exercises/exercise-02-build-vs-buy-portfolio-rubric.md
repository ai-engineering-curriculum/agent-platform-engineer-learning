# exercise-02: Build-vs-Buy Portfolio Rubric

**Estimated effort:** 2 hours · **Output:** a `rubric.yaml` (or markdown table) formalising the six-axis rubric, two signed decision records (`BvB-2026-Q4-02-runtime.md` and `BvB-2026-Q4-07-mcp-registry.md`), a one-paragraph portfolio summary, and a one-paragraph reflection · **Pairs with:** chapter [02](../02-build-vs-buy-portfolio.md); feeds exercise [01](exercise-01-platform-roadmap-authored-against-stream-teams.md) (Later column entries) and the module's governance log.

## Objective

Turn chapter 02's rubric and decision discipline into two concrete decision records that a sponsor, a tenant council, and an architect could read and either sign or send back. The exercise forces you to (a) formalise the six-axis rubric in a form you can re-apply, (b) score two different real vendor landscapes against the rubric, (c) write the decision record as a durable artefact — vendors considered, axis scores, tenant accommodations, triggers for re-decision, signatures.

The exercise explicitly asks you to produce *two* records with *different outcomes*. One lands "build"; one lands "hybrid". The contrast is the point — the rubric is a decision instrument, not a build-or-buy preference.

## Prerequisites

- You have read chapter [02](../02-build-vs-buy-portfolio.md).
- You have completed [exercise 01](exercise-01-platform-roadmap-authored-against-stream-teams.md) (the tenant register is referenced in both decision records).
- You have skimmed the vendor landscape as of late 2026 via the links in chapter 02 and `resources.md`. For any specific feature or pricing claim you make, verify against the vendor's current documentation.

## Problem statement

Your platform-team is at the Q4 build-vs-buy review (chapter 02 §portfolio view). Two decisions are open on the roadmap's Later column that have been pulled hard enough by tenants to move to a decision this quarter:

### Decision A — The agent runtime / orchestration layer

Your platform currently self-hosts its agent runtime on Temporal. Two signals have forced the review:

- `globex-health` is scaling their clinical-trials copilot and their compliance officer has asked whether the runtime can be migrated to a managed service with an explicit regulatory posture.
- A new executive has suggested the organisation should "reduce the number of self-hosted critical systems"; the agent runtime is on the shortlist.
- Candidate vendors: [LangGraph Platform](https://www.langchain.com/langgraph-platform), [Vertex AI Agent Engine](https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/overview), [Azure AI Foundry Agent Service](https://learn.microsoft.com/en-us/azure/ai-foundry/agents/overview), [AWS Bedrock AgentCore](https://aws.amazon.com/bedrock/agentcore/).

### Decision B — The MCP server registry

Your platform currently maintains a private MCP server registry (built in [`mod-403`](../../mod-403-mcp-registry-and-tool-gateway/)). Two signals:

- `acme-financial` wants to use a widely-adopted public MCP server (a search tool), and the governance-by-copy approach the private registry supports is wearing thin.
- A couple of public registries have reached a maturity worth evaluating.
- Candidate vendors: [Smithery](https://smithery.ai/), [mcp.so](https://mcp.so/), [PulseMCP](https://www.pulsemcp.com/), [Composio](https://composio.dev/), plus the current self-host.

## Requirements

### Part 1 — The rubric file

A `rubric.yaml` (or `rubric.md`) formalising the six axes from chapter 02:

```yaml
axes:
  - id: strategic_fit
    description: "Does the component express an opinion the platform exists to hold?"
    scale:
      1: "Pure commodity"
      3: "Partially strategic; the edges matter"
      5: "Strategic; the component expresses the platform's identity"
  - id: regulatory
    description: "Residency, BYOK, attestations the component must satisfy"
    scale:
      1: "Vendor meets all requirements out of the box"
      3: "Partial; compensating controls required"
      5: "Structural gap; vendor cannot meet"
  # … repeat for integration_tax, total_cost, reversibility, team_fit
```

Each axis has a 1 / 3 / 5 scale with the chapter-02 wording as the anchors. 2 and 4 are interpolations; use them freely.

Add a `reading_guidance` section: three or four short paragraphs explaining how the scored axes are *read together* (chapter 02 §reading the rubric). This is the piece that keeps the rubric from being read as a calculator.

### Part 2 — Decision record A (the runtime)

A `BvB-2026-Q4-02-runtime.md` using the chapter-02 record shape:

- `component` — "Agent runtime / orchestration"
- `date` — ISO-8601 (pick a date in Q4 2026)
- `owner` — "@platform-lead"
- `decision` — one of "build (continue self-host)", "buy (migrate)", "hybrid", "defer"
- `vendors_evaluated` — four or five vendors with a one-sentence note each: what they offer, what makes them a serious candidate, what disqualifies or limits them. Cite specific product pages where you make a claim.
- `axis_scores` — 1–5 on each of the six axes, with a one-sentence justification per score.
- `summary` — three or four sentences explaining the decision.
- `tenant_accommodations` — how `globex-health` (or any tenant with special needs) is accommodated under the decision.
- `triggers_for_re_decision` — two to four specific, observable triggers.
- `re_decision_horizon` — ISO-8601 (chapter 02 recommends 12 months for runtime).
- `signed_by` — invent three signers: platform-lead, executive-sponsor, tenant-council-chair.

Honor chapter 02's guidance: the typical shape for a runtime decision in late 2026 is *build with per-tenant accommodation*. You may land somewhere else, but the record must explain *why* departing from the typical shape.

### Part 3 — Decision record B (the MCP registry)

A `BvB-2026-Q4-07-mcp-registry.md` with the same shape. Here the typical shape is *hybrid* (self-host private + subscribe to a public registry via federation). Your record should land hybrid and articulate the federation posture — which tools go in which registry, how the policy plane decides, what happens when a public registry entry changes or disappears.

### Part 4 — Portfolio summary (one paragraph)

A `portfolio-summary.md` paragraph (~150 words) that reads *across* the two decisions. The paragraph must cover:

- What the portfolio looks like after Q4 — which planes are "build", which are "buy", which are "hybrid".
- Whether the portfolio respects chapter 02's constraints (no more than ~30 % of planes on a managed service; no more than one decision resolving per quarter; every managed service has a re-decision horizon).
- One sentence on what the *next* portfolio decision is likely to be (and in which quarter).

### Part 5 — Reflection paragraph

A `reflection.md` paragraph (~120 words) answering: *where did the rubric give you a clear answer, and where did the rubric force you to make a judgement call that the scores alone could not justify?*

The reflection is the exercise's point — the rubric is a decision *instrument*, and instruments do not decide for you.

## Starter guidance

- **Score the axes before writing the decision.** The temptation is to work backward from the answer you already have. Work forward from the axes and *notice* when the scores point somewhere you did not expect.
- **Do not sum the scores.** Chapter 02 is explicit. A "total" is a crutch; the axis-by-axis read is the decision.
- **Verify specific vendor claims.** For any sentence like "vendor X supports BYOK" or "vendor Y is priced at …", cite the vendor's own page and mark `<!-- needs-research: ... -->` if you cannot find a current source. The late-2026 vendor landscape moves fast.
- **Make the tenant accommodation specific.** A record that says "accommodate `globex-health` on Vertex" is weaker than one that says "`globex-health`'s clinical-trials copilot runs on Vertex AI Agent Engine via the paved-road SDK's Vertex adapter; the manifest tags it; the policy plane applies the HIPAA-scope policies regardless of substrate."
- **Triggers for re-decision are observable.** "If a managed runtime gets better" is not a trigger. "If a managed runtime speaks OTel-GenAI session conventions natively" is. Write triggers a quarterly review can scan in five minutes.
- **The signed-by line matters.** Three signers (platform-lead, sponsor, tenant-council-chair) is the chapter-02 convention. Record their names (invented), their role, and the date. A reversal becomes a new record; signatures are durable.

## Acceptance criteria

Mark done when:

- [ ] `rubric.yaml`/`rubric.md` has all six axes with 1 / 3 / 5 anchors from chapter 02.
- [ ] The rubric includes a `reading_guidance` section explaining the read-together discipline.
- [ ] `BvB-2026-Q4-02-runtime.md` has all chapter-02 fields, including axis_scores with one-sentence justifications.
- [ ] `BvB-2026-Q4-07-mcp-registry.md` has all chapter-02 fields.
- [ ] One of the two decisions lands "build (with accommodation)"; the other lands "hybrid".
- [ ] Both records cite at least three vendor-specific sources (links to product pages, documentation, or whitepapers).
- [ ] Both records specify a `re_decision_horizon` in ISO-8601 form.
- [ ] Both records specify at least two observable `triggers_for_re_decision`.
- [ ] `portfolio-summary.md` paragraph lands in ~150 words and covers portfolio shape, chapter-02 constraints, and the next likely decision.
- [ ] `reflection.md` paragraph names at least one axis that gave a clear answer and at least one that required a judgement call.

## Stretch goals

- **Author the architect ask.** Chapter 05's handoff artefact shape. The runtime decision touches cross-stack infrastructure (hosted model gateways, shared compute); author a two-page `arch-ask-2026-Q4-01-runtime.md` that hands the context to the `ai-infra-architect` and asks them to confirm the direction.
- **Model a cost scenario.** For the runtime decision, estimate total cost under two scenarios (self-host at current scale; managed at the same scale). Use the vendor's self-serve pricing where available and mark `<!-- needs-research: ... -->` where enterprise pricing is unavailable. Produce a `cost-scenarios.md` one-pager.
- **Add a third decision — the sandbox.** Run the full rubric on [E2B](https://e2b.dev/) vs [Daytona](https://daytona.io/) vs [Modal](https://modal.com/) vs self-host, and produce `BvB-2026-Q4-11-sandbox.md`. Note this violates chapter 02's "one decision per quarter" rule; make the violation explicit and justify (or decline) in the portfolio summary.
- **Red-team your records.** Against each of chapter 02's six failure modes (vendor-asked-for-a-meeting, we've-always-built-it, cheapest-at-the-top-of-funnel, integration-tax-paid-once, regulatory-posture-at-signing, exit-cost-unexamined), identify the weakest point in each decision record and write a one-sentence mitigation.
- **Author the re-decision trigger-scan.** A short script or checklist that reads a decision record and prints the next scheduled re-decision date plus the trigger list to be scanned. This is the piece the quarterly roadmap review runs every quarter.
- **Draft the migration playbook for the alternative decision.** If your runtime decision landed "build", write the one-page migration playbook you *would* follow if a trigger fired in year 2 and the re-decision flipped to "buy". If the decision landed "buy", write the exit-migration playbook. Chapter 02 argues exit cost must be an explicit axis; this stretch makes it tangible.
