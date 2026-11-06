# exercise-03: CNCF Platform Maturity Self-Assessment

**Estimated effort:** 2 hours · **Output:** one 4×4 maturity-grid assessment + a next-quarter plan · **Pairs with:** chapter 02.

## Objective

Use the CNCF Platform Engineering Maturity Model to assess where a real agent platform currently sits, on each of the four dimensions (Investment, Adoption, Interfaces, Operations), and write the next-quarter plan that moves at least two dimensions up by one level. This is the diagnostic routine `mod-413` later builds into a quarterly cadence; this exercise is the first run of it.

## Prerequisites

- You have read [`02-cncf-maturity-and-team-topologies.md`](../02-cncf-maturity-and-team-topologies.md).
- You have read the [CNCF Platform Engineering Maturity Model](https://tag-app-delivery.cncf.io/whitepapers/platform-eng-maturity-model/) whitepaper directly, not just the chapter summary.
- You can describe one real platform in enough depth to assess it on all four dimensions. Options:
  - Your own employer's agent platform, if you work on one.
  - A publicly-described platform (`LangGraph Platform`, `Vertex AI Agent Engine`, `Azure AI Foundry Agent Service`, `AWS Bedrock Agents`, `Dapr Agents`) using only what their public docs say — you will have to flag what the public record cannot tell you.
  - A hypothetical platform at your current project's maturity if you are earlier in your career. Say "hypothetical" explicitly in the write-up.

## Problem statement

Characterise the platform you picked on each of the CNCF model's four dimensions (Investment, Adoption, Interfaces, Operations) at one of four levels (Provisional, Operational, Scalable, Optimizing). Cite the evidence for each level. Then write the next-quarter plan.

## Requirements

### 1. The 4×4 self-assessment

A single grid like the one below, filled in. For each dimension, cite three pieces of evidence that justify the level. Evidence can be a doc, a telemetry metric, an org-chart fact, or a cadence artefact (e.g. "Q3 planning doc shows the platform ran a per-team paved-road adoption review on dd/mm/yyyy"). If the evidence is private, describe it generically; if the evidence is public, link to it.

| Dimension | Current level (1–4) | Evidence #1 | Evidence #2 | Evidence #3 |
|-----------|---------------------|-------------|-------------|-------------|
| **Investment** | | | | |
| **Adoption** | | | | |
| **Interfaces** | | | | |
| **Operations** | | | | |

### 2. The pull-vs-push test for Adoption

For the Adoption dimension specifically, answer in one paragraph: *if leadership lifted the mandate to use the platform tomorrow, how many stream-aligned teams would still pick the paved road?* Cite the evidence. This is the single most important test of true Adoption maturity, and the one most commonly overstated in internal reporting (chapter 02 — "common pitfalls").

### 3. The next-quarter plan

Pick the two dimensions where moving up one level is both most valuable and most achievable in a single quarter. For each one:

- **Target level** — the next CNCF level, named.
- **What changes in practice** — the concrete changes a stream-aligned team would notice. Not "we will get better at X"; instead "we will ship a scaffolding CLI such that a product engineer can start a new agent against the paved road in under 15 minutes."
- **Module coverage** — which modules in this track teach the substrate you will need. Cite the module slug.
- **Measurement** — what you will measure at end-of-quarter to know you moved up a level. One or two metrics, not a dashboard.
- **Dependency risk** — one thing that could prevent the move (resourcing, dependency on a peer track that is understaffed, a product-team adoption requirement). One sentence.

### 4. The dimension you decided *not* to move

Name the dimension you deliberately parked this quarter and explain in one paragraph why moving up on it is lower-leverage this cycle. This is a Team Topologies discipline — a healthy platform team does not try to level up on every dimension simultaneously.

## Starter guidance

- The CNCF model is diagnostic, not prescriptive. Resist the urge to inflate your self-assessment — the point of this exercise is that your plan is only as useful as the honesty of your level assignments.
- If you are assessing a public platform, note explicitly what the public record cannot answer. For example, you can usually read Interfaces (docs, CLI, templates) and some of Operations (public SLA / support statements) from the outside, but Investment and true pull-vs-push Adoption are rarely visible externally.
- Pull-vs-push is the test that separates Level 2 from Level 3 Adoption. A platform where all adoption is mandated is almost certainly Level 2 even if the technical interfaces look Level 3.
- If your platform is at Level 1 across the board, that is normal for an early-stage platform. The next-quarter plan should still identify two dimensions to move; a plan that moves zero dimensions is not a plan.
- Sketch two levels of resolution: the headline grid (what you present to leadership) and the per-cell evidence (what you present to a peer platform team when you compare notes).

## Acceptance criteria

You can mark this exercise done when all of these are true:

- [ ] Every cell of the 4×4 grid has a level and three cited pieces of evidence.
- [ ] Your pull-vs-push paragraph cites evidence, not opinion. (If you cannot cite evidence, the level is probably lower than you first thought.)
- [ ] The two-dimension next-quarter plan names a *behaviour change a user would notice*, not an internal milestone.
- [ ] Each dimension in the plan cites a specific module (`mod-402` … `mod-413`) that teaches the substrate the move requires.
- [ ] The parked dimension has a defensible reason, not "we didn't have time."
- [ ] You have been honest where the public record (or your access) did not let you assess a dimension cleanly.

## Stretch goals

- Repeat the assessment for a second platform at a very different maturity level — e.g. your employer's platform plus LangGraph Platform plus your side-project agent. Compare the grids and notice which dimensions look similar and which look different.
- Draft the "scorecard" view a Level 4 platform would publish for its stream-aligned customers (adoption rate, SLO burn by tenant, cost by tenant, paved-road-vs-off-road percentage, safety-event rate). Even if your current platform is Level 1, drafting the Level-4 scorecard tells you what data pipelines you will need by then (`mod-408` + `mod-413`).
- Compare your plan with [Spotify's internal platform engineering posts](https://engineering.atspotify.com/) or [Netflix's paved-road writing](https://netflixtechblog.com/). Both run the Team Topologies + maturity-model pattern openly. What do they do at a level above yours that you can borrow?
