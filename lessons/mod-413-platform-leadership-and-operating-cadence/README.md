# mod-413 — Platform Leadership and Cross-Team Operating Cadence

**Estimated effort:** 10 hours · **Position in track:** thirteenth and final module (after [`mod-401`](../mod-401-agent-platform-position-and-paved-road-philosophy) through [`mod-412`](../mod-412-paved-road-sdks-and-developer-experience)).

Every prior module in this track built one of the eleven planes or the paved-road SDK. This module steps up a level: given a platform with all eleven planes and a finite team, *how do you lead it*? The slice this module owns is the leadership-and-operating-cadence work that keeps the platform coherent across quarters — the roadmap, the build-vs-buy portfolio, the cross-team incident posture, the success metrics that go to the executive sponsor, and the handoffs up the ladder.

The grounding is public: [Team Topologies](https://teamtopologies.com/) and the [CNCF Platform Engineering Maturity Model](https://tag-app-delivery.cncf.io/whitepapers/platform-eng-maturity-model/) from mod-401 chapter 02, the [FEMA Incident Command System (ICS)](https://training.fema.gov/is/courseoverview.aspx?code=is-100.c) and [Google SRE's incident-management chapter](https://sre.google/sre-book/managing-incidents/) for incident command, [DORA](https://cloud.google.com/devops/state-of-devops) and [SPACE](https://queue.acm.org/detail.cfm?id=3454124) for the measurement discipline, and the four mod-412 adoption KPIs (platform adoption rate, time-to-first-agent, deployment success rate, rollback count) plus support-ticket MTTR as the platform-specific instruments. The specialisation is agents — a build-vs-buy portfolio with MCP registry / sandbox / runtime decisions against live vendors, an incident-command discipline that composes platform containment with product-side agent misbehaviour, and a scorecard that reads DORA / SPACE / adoption / support together.

## Learning objectives

- Author a platform roadmap against stream-aligned team needs using the [CNCF Platform Engineering Maturity Model](https://tag-app-delivery.cncf.io/whitepapers/platform-eng-maturity-model/) and [Team Topologies](https://teamtopologies.com/) as the operating frame.
- Run the build-vs-buy portfolio for agent-platform components — self-host vs. managed MCP registry ([Smithery](https://smithery.ai/) / [mcp.so](https://mcp.so/) / [PulseMCP](https://www.pulsemcp.com/) / [Composio](https://composio.dev/)), self-host vs. [E2B](https://e2b.dev/) / [Daytona](https://daytona.io/) / [Modal](https://modal.com/), self-host vs. [LangGraph Platform](https://www.langchain.com/langgraph-platform) / [Vertex AI Agent Engine](https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/overview) / [Azure AI Foundry Agent Service](https://learn.microsoft.com/en-us/azure/ai-foundry/agents/overview) / [AWS Bedrock AgentCore](https://aws.amazon.com/bedrock/agentcore/).
- Command cross-team incidents that span platform outage and product-side agent misbehaviour; coordinate with [`ai-infra-sre`](../../../ai-infra-sre-learning/) (peer, level 30), [`senior-agentic-ai-engineer`](../../../senior-agentic-ai-engineer-learning/) (peer, level 40), [`agentic-safety-engineer`](../../../agentic-safety-engineer-learning/) (peer, level 40), [`ai-risk-engineer`](../../../ai-risk-engineer-learning/) (peer, level 25), and SecOps.
- Measure platform success across four complementary instruments — [DORA](https://cloud.google.com/devops/state-of-devops), [SPACE](https://queue.acm.org/detail.cfm?id=3454124), the four platform-adoption KPIs from mod-412 chapter 06, and support-ticket MTTR.
- Hand off to [`ai-infra-architect`](../../../ai-infra-architect-learning/) (level 45), [`agentic-systems-architect`](../../../agentic-systems-architect-learning/) (level 48), and engineering management (level 45+) for cross-platform strategy, roadmap governance, and executive sponsorship.

## Chapters

1. [`01-platform-roadmap-against-stream-teams.md`](01-platform-roadmap-against-stream-teams.md) — The roadmap as a dated list of capability investments scored against stream-aligned pull and CNCF maturity dimensions, held across three horizons (Now / Next / Later). The tenant register, the entry shape, the authoring steps, the weekly / monthly / quarterly / annual cadence, and the six common failure modes.
2. [`02-build-vs-buy-portfolio.md`](02-build-vs-buy-portfolio.md) — Build-vs-buy as a portfolio-level discipline. The six evaluation axes (strategic fit, regulatory, integration tax, total cost, reversibility, team fit); three worked decisions anchored to late-2026 vendor landscapes (MCP registry, sandbox, runtime); the signed decision-record shape; the portfolio rules that constrain quarterly decision throughput.
3. [`03-cross-team-incident-command.md`](03-cross-team-incident-command.md) — Incident Command System adapted for agent platforms. The role structure (IC, Operations, Communications, Planning), the three severity tiers, the status-update template, three coordination patterns with worked scenarios, the post-incident review shape, six failure modes, and when IC should hand off to a safety / security / risk owner.
4. [`04-measuring-platform-success.md`](04-measuring-platform-success.md) — The four instruments (DORA, SPACE, platform-adoption KPIs, support-ticket MTTR) read together. The adaptations each lens needs for an agent platform; the quarterly survey discipline; the one-page scorecard; five vanity-metric failure modes; the read-together discipline that the quarterly review runs on.
5. [`05-handoffs-and-governance.md`](05-handoffs-and-governance.md) — The three handoff targets (`ai-infra-architect`, `agentic-systems-architect`, engineering management); what each receives, what the platform engineer keeps, the handoff artefacts; the reverse-flow work architects and managers send back; the governance log; postures for when a handoff target role is unfilled; five handoff failure modes.

## Exercises

Each exercise turns one or more chapters into a durable artefact you keep and reuse — in your actual platform-engineer practice, in the operating cadence you set up, and in the governance log the scorecard and the architect asks go into.

- [`exercises/exercise-01-platform-roadmap-authored-against-stream-teams.md`](exercises/exercise-01-platform-roadmap-authored-against-stream-teams.md) — Author a platform roadmap (Now / Next / Later) against a realistic tenant register. Pairs with chapter [01](01-platform-roadmap-against-stream-teams.md).
- [`exercises/exercise-02-build-vs-buy-portfolio-rubric.md`](exercises/exercise-02-build-vs-buy-portfolio-rubric.md) — Score two live build-vs-buy decisions against the six-axis rubric and produce signed decision records. Pairs with chapter [02](02-build-vs-buy-portfolio.md).
- [`exercises/exercise-03-cross-team-incident-command-playbook.md`](exercises/exercise-03-cross-team-incident-command-playbook.md) — Author a cross-team ICS playbook and run a tabletop against a realistic cross-plane incident. Pairs with chapter [03](03-cross-team-incident-command.md).
- [`exercises/exercise-04-platform-adoption-scorecard.md`](exercises/exercise-04-platform-adoption-scorecard.md) — Author the one-page quarterly scorecard that composes DORA, SPACE, adoption, and support-MTTR, and defend it against the vanity-metric failure modes. Pairs with chapter [04](04-measuring-platform-success.md).

## Resources

External references — primary sources — are catalogued in [`resources.md`](resources.md).

## How to approach this module

1. Read chapters 01 through 05 in order. The chapters compose: the roadmap (01) is where the build-vs-buy decisions (02) land; the roadmap and the build-vs-buy records survive incident command (03); the scorecard (04) is how leadership knows the roadmap and incidents are working; the handoffs (05) are how the whole posture scales beyond level 40.
2. Do the four exercises in order. Exercise 01 produces the tenant register and roadmap artefacts that later exercises assume; exercise 03's tabletop needs a plausible platform shape; exercise 04 reads the scorecard against the roadmap you authored in 01.
3. Keep the artefacts the exercises produce in version control. The tenant register, the build-vs-buy records, the incident playbook, and the quarterly scorecard are not exercises you throw away — they are templates for the real platform you will run.
4. Mark `<!-- needs-research: ... -->` on any specific claim that drifts: CNCF maturity-model indicator wording, DORA band thresholds, SPACE instrument details, specific vendor features or pricing tiers, specific MCP registry or managed-runtime capabilities. The vendor landscape as of late 2026 is moving quarterly; the flag blocks auto-merge and is strictly preferable to silent drift.
5. Return to [`mod-401` chapter 02](../mod-401-agent-platform-position-and-paved-road-philosophy/02-cncf-maturity-and-team-topologies.md) when the chapter-01 maturity-level language gets unfamiliar. The two frames (Team Topologies, CNCF maturity) are the spine of the whole module.

## Where the handoffs land

- **[`agentic-ai-engineer`](../../../agentic-ai-engineer-learning/) (peer, level 30)** — stream-aligned product-side consumer of the paved road; appears in the tenant register (chapter 01) and in the SPACE survey (chapter 04). The platform engineer's roadmap entries have them as pullers.
- **[`senior-agentic-ai-engineer`](../../../senior-agentic-ai-engineer-learning/) (peer, level 40)** — subsystem-lead consumer who negotiates exceptions and who co-leads incidents with a product-side root cause (chapter 03).
- **[`agentic-safety-engineer`](../../../agentic-safety-engineer-learning/) (peer, level 40)** — the safety track; becomes IC for incidents with a frontier-safety dimension (chapter 03); co-authors the safety-plane roadmap entries.
- **[`ai-risk-engineer`](../../../ai-risk-engineer-learning/) (peer, level 25)** — risk / compliance track; joins incidents with regulatory implications (chapter 03); feeds the regulatory axis of build-vs-buy decisions (chapter 02).
- **[`ai-infra-sre`](../../../ai-infra-sre-learning/) (peer, level 30)** — SRE track; owns the SRE-plane work (mod-410) that this module's scorecard reports against.
- **SecOps** — joins incidents with a security dimension (chapter 03); feeds the regulatory axis of build-vs-buy decisions (chapter 02).
- **[`ai-infra-architect`](../../../ai-infra-architect-learning/) (level 45)** — upward handoff target for cross-stack infrastructure decisions (chapter 05).
- **[`agentic-systems-architect`](../../../agentic-systems-architect-learning/) (level 48)** — upward handoff target for cross-product agent-architecture direction (chapter 05).
- **Engineering management (level 45+)** — upward handoff target for roadmap governance, executive sponsorship, headcount, cross-org coordination (chapter 05).

## Scope policing

This module owns: the operating cadence over the eleven planes, the roadmap, the build-vs-buy portfolio, cross-team incident command, the four-instrument scorecard, and the governance log. It does *not* own: any single plane's technical content (that is modules 02 through 12), the paved-road SDK's shape (mod-412), cross-stack architecture decisions (`ai-infra-architect`), cross-product agent-architecture direction (`agentic-systems-architect`), executive sponsorship or headcount authority (engineering management). Drift off this slice and the platform engineer either becomes a vague architect or becomes a stand-in manager — both are promotions in disguise and neither leaves the planes well-run.
