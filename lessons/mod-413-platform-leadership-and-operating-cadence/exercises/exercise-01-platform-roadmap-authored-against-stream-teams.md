# exercise-01: Platform Roadmap Authored Against Stream Teams

**Estimated effort:** 2 hours · **Output:** a tenant register (one page), a candidate list (15–25 entries), a scored roadmap with Now / Next / Later columns, and a one-paragraph sponsor-review narrative · **Pairs with:** chapter [01](../01-platform-roadmap-against-stream-teams.md); feeds exercises 02, 03, and 04, and the paired solutions repo.

## Objective

Turn chapter 01's authoring discipline into a concrete, defensible platform roadmap you could hand to an executive sponsor and a tenant council tomorrow. The exercise is deliberately scoped to a *fictional but realistic* organisation — you are not authoring your real employer's roadmap, you are proving you can author *a* roadmap that survives the chapter's six failure modes.

The point of the exercise is the discipline, not the specific entries. By the end you can: (a) name a set of stream-aligned tenants, (b) score candidate investments on pull + maturity + risk + cost, (c) fit the Now column to a realistic capacity budget, (d) articulate the quarter in a single sponsor-review paragraph.

## Prerequisites

- You have read chapter [01](../01-platform-roadmap-against-stream-teams.md) and [`mod-401` chapter 02](../../mod-401-agent-platform-position-and-paved-road-philosophy/02-cncf-maturity-and-team-topologies.md) (Team Topologies + CNCF maturity).
- You have scanned [`mod-412` chapter 06](../../mod-412-paved-road-sdks-and-developer-experience/06-measuring-adoption-and-cncf-maturity.md) (adoption KPIs) and [`mod-412` exercise 03](../../mod-412-paved-road-sdks-and-developer-experience/exercises/exercise-03-adoption-metric-dashboard.md) — the scorecard inputs this roadmap commits to.
- You have the chapter-01 sample roadmap entry shape in mind.

## Problem statement

You have been named platform-team lead for an agent platform mid-rev. The snapshot:

- **Team capacity:** 5 engineers, 4 engineer-weeks per sprint, 2 sprints per quarter = 20 engineer-weeks per quarter (with the chapter-01 20 % buffer, 16 weeks of plannable work).
- **Current CNCF maturity self-assessment:** Investment Level 3, Adoption Level 2, Interfaces Level 2, Operations Level 3 (steady-state oncall, documented SDK, template-less scaffolding).
- **Known tenants (four, drawn from a realistic enterprise shape):**
  - `acme-financial` — T1; refund-automation agent; strict financial-services compliance; 12 production agents; adopted-planes: runtime, policy, model gateway; blockers: "time-to-first-agent too long", "MCP registry search slow".
  - `globex-health` — T1; clinical-trials copilot; HIPAA scope; 5 production agents; adopted-planes: runtime, policy, model gateway, safety; blockers: "eval regression gates not integrated with our fixtures", "a managed-runtime pilot was promised in Q1".
  - `initech-robotics` — T2; internal operations assistant; 3 production agents; adopted-planes: runtime, SDK (just adopted); blockers: "the deprecation of anthropic-v1 breaks our prompt library".
  - `umbrella-ops` — T2; infra ops copilot; 2 production agents; adopted-planes: runtime, SDK, sandbox; blockers: "sandbox start-up latency hurts interactive workloads".
- **Pending mandated work:**
  - SDK rev `v2.0` ships in Q4; deprecation ladder step 2 → 3 for the `anthropic-v1` adapter.
  - OpenTelemetry-GenAI semantic-conventions `gen_ai.*` attribute rev (minor); SDK must follow.
  - Build-vs-buy review for the MCP registry is scheduled (feeds [exercise 02](exercise-02-build-vs-buy-portfolio-rubric.md)).
  - Two Sev2 incidents from Q3 produced four action items (two regression fixtures still to promote).

You are authoring the **Q4 roadmap**. The quarter starts in two weeks.

## Requirements

### Part 1 — Tenant register

A one-page `tenant-register.md` with the chapter-01 columns: `tenant_id`, `stream_name`, `primary_contact` (make up a name), `tier`, `adopted_planes`, `exceptions_open` (invent a plausible count), `blockers` (use the ones above plus one more per tenant that you invent), `last_touchpoint` (ISO-8601 — pick dates in the last 60 days).

The register is for the fictional organisation and will be the working set every later exercise reads from.

### Part 2 — Candidate list

A `candidates.md` or `candidates.yaml` with 15–25 candidate roadmap entries sourced from:

- The four tenant blocker lists above (plus the one-per-tenant you invent).
- The pending mandated work (SDK rev, OTel rev, MCP build-vs-buy review, incident fixture-promotion).
- At least two platform-internal entries (e.g., an SRE playbook gap, a policy-plane refactor, a documentation refresh, a secret-rotation migration).
- At least one entry from each of these categories you invent: a cross-plane investment (two planes have to coordinate), a tenant-asked-but-platform-declines entry (chapter 01 "when not to platformise"; use the mod-401 chapter 04 decision checklist).

For each candidate: a title, a one-sentence description, which tenant(s) would pull it (or `platform-internal`), and a rough engineer-week estimate.

### Part 3 — Scored roadmap

A `roadmap-Q4-2026.yaml` with each candidate promoted into Now, Next, or Later (or marked `declined` with a one-sentence reason).

Use the chapter-01 entry shape (fields: id, title, horizon, owner, pullers, maturity_target, cost, acceptance, risks). Score each candidate on the four dimensions in chapter 01 (pull signal, maturity move, risk of delay, cost) before assigning a horizon.

Constraints to respect:

- Now column total engineer-weeks ≤ 16 (the 20 % buffer).
- Platform-internal entries ≤ 25 % of the Now column by engineer-weeks.
- At least one entry explicitly advances each of Interfaces and Adoption maturity dimensions in the Next column.
- At least one Later entry is a build-vs-buy decision awaiting pull strengthening.
- At least one candidate is `declined` with a clear "when not to platformise" rationale.

### Part 4 — Sponsor-review paragraph

A one-paragraph narrative (~120 words, no more than 160) that you would read in the monthly executive-sponsor review at the start of Q4. The paragraph must mention:

- The capacity budget and the planned utilisation.
- The maturity moves targeted for the quarter (dimension × level change).
- The tenants whose work is primarily being pulled.
- The single risk you most want the sponsor to be aware of.

Use the chapter-01 worked-example paragraph as a reference shape, not a template to copy.

## Starter guidance

- **Build the tenant register first.** It is the forcing function — every roadmap entry must reference a tenant in it. If a candidate has no tenant, it is either platform-internal (declare it) or speculative (drop it).
- **Score ruthlessly on pull.** The most common failure is the wish-list roadmap (chapter 01). An entry whose puller is "we think teams would want this" is speculative; it goes in Later or is declined.
- **Platform-internal entries are first-class citizens — but capped.** The cap is a lever, not a suggestion. If internal work dominates, name the quarter a "remediation quarter" explicitly in the sponsor paragraph.
- **Use real engineer-week estimates.** A quarter with seven 2-week entries is honest; a quarter with fifteen 1-week entries is almost always under-estimated. Scope entries at *delivery granularity*, not task granularity.
- **Respect the horizon semantics.** Now = in flight, with named owner; Next = committed to, pullers named; Later = bets, not yet staffed. Promotion between horizons is a decision you record in the roadmap file (e.g., `promoted_from: next; promoted_on: 2026-10-15`).
- **Mark `<!-- needs-research: ... -->` on CNCF-maturity-indicator wording** you cannot verify against the current whitepaper — chapter 01 pins the practice.
- **Do the declined entry seriously.** A fictional-but-plausible "we won't build this and here's why" entry is often the strongest signal of discipline on the whole roadmap. The chapter-04 of mod-401 decision checklist gives you the frame.

## Acceptance criteria

Mark done when:

- [ ] `tenant-register.md` names four tenants with all required columns populated.
- [ ] `candidates.md`/`candidates.yaml` has 15–25 entries covering the sources listed in Part 2.
- [ ] `roadmap-Q4-2026.yaml` has every candidate assigned to Now / Next / Later / declined.
- [ ] Each Now entry has the five load-bearing fields (owner, pullers, maturity_target, cost, acceptance); each pullers list references a tenant from the register.
- [ ] Now column total engineer-weeks ≤ 16; platform-internal share ≤ 25 %.
- [ ] At least one declined entry has a clear "when not to platformise" rationale.
- [ ] At least one Later entry is a build-vs-buy decision pointing to [exercise 02](exercise-02-build-vs-buy-portfolio-rubric.md).
- [ ] The sponsor paragraph lands in ≤ 160 words and covers capacity, maturity moves, pullers, and one risk.

## Stretch goals

- **Wire the roadmap to a real tracking system.** Export the Now column into a GitHub Projects board, a Linear project, or a JIRA query. Each entry's `id` matches the issue key.
- **Add a dependency graph.** Some Now entries depend on others (e.g., the scaffolder depends on the SDK rev). Produce a small [Mermaid](https://mermaid.js.org/) `graph TD` diagram showing dependencies.
- **Run the chapter-01 interview questions against yourself.** Pick one tenant, play both the primary contact and the platform-engineer, and work through the five interview questions. Record the transcript. Reflect in a one-paragraph note on which questions were hardest to answer honestly.
- **Author the Q4 candidate list for Q1.** Carry three to five entries forward into the Later column for Q1 with an articulated promotion-trigger (what signal, if it fires, promotes it to Next).
- **Produce the quarterly tenant-council agenda.** Chapter 01's quarterly-cadence section specifies the agenda shape. Author a 90-minute agenda the council would receive, with named attendees and three discussion-prompt bullets.
- **Red-team your own roadmap.** Against each of chapter 01's six failure modes (wish-list, platform-internal, vendor-import, architecture-first, hero-engineer, silent-handoff), write one sentence identifying the *weakest* point in your roadmap on that failure mode, and the mitigation.
