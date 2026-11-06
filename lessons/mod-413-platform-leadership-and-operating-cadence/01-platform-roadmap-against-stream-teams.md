# Platform Roadmap Authored Against Stream-Aligned Teams

Every prior module in this track built one plane and left a seam. [`mod-402`](../mod-402-agent-runtime-service-and-durable-execution) left the runtime. [`mod-403`](../mod-403-mcp-registry-and-tool-gateway) left the tool gateway. [`mod-412`](../mod-412-paved-road-sdks-and-developer-experience) left the SDK and the adoption dashboard. Each ends with an obligation and a handoff. This module is the one that answers the question none of them finish: *given a platform with eleven planes and a finite team, what is the next quarter's work?*

That answer is a **roadmap**, and a roadmap for a platform team is a different artefact from a roadmap for a product team. A product roadmap is a sequence of user-visible features. A platform roadmap is a sequence of *capability investments*, each scored against (i) the needs of a specific set of stream-aligned teams the platform exists to serve, and (ii) the CNCF Platform Engineering Maturity Model's next-step indicators on Investment, Adoption, Interfaces, Operations. The two frames hold the roadmap honest; without them it drifts into whatever is loudest.

This chapter pins the operating frame, the roadmap artefact's shape, the cadence it is reviewed on, and the common failure modes.

## The operating frame recap

[`mod-401` chapter 02](../mod-401-agent-platform-position-and-paved-road-philosophy/02-cncf-maturity-and-team-topologies.md) introduced the two frames the roadmap sits on. A fast recap, because the roadmap is where they become a decision rather than a diagnostic.

- **[Team Topologies](https://teamtopologies.com/)** — the organisational pattern. A *platform team* exists to reduce the cognitive load of *stream-aligned* product teams, who are its customers. The platform's mandate is not "build cool infrastructure"; it is "remove friction from the teams listed in your tenant register".
- **[CNCF Platform Engineering Maturity Model](https://tag-app-delivery.cncf.io/whitepapers/platform-eng-maturity-model/)** — the diagnostic frame. Four dimensions (Investment, Adoption, Interfaces, Operations); four levels (Provisional, Operational, Scalable, Optimizing). Used here to ask *what is the next step* on each dimension.

A roadmap authored against these two frames reads differently from one authored against engineer enthusiasm. Each roadmap entry must answer *which stream-aligned team pulls this work and which maturity-dimension level does it advance*. An entry that answers neither is cut.

## What a platform roadmap actually is

A roadmap is a dated list of capability investments plus the stream-aligned teams and maturity dimensions each one serves. It is not a backlog, not a plan of record for every ticket, not a vision document. It is the artefact mod-413 leadership returns to every quarter to make the next investment decision defensible.

The roadmap has three time horizons:

| Horizon | Length | Content | Governance |
|---|---|---|---|
| **Now** | 1 quarter | Capabilities in flight; one owner each; acceptance criteria written | Reviewed weekly with the platform team; monthly with the executive sponsor |
| **Next** | 2–3 quarters | Capabilities committed to with named stream-aligned pullers; maturity-level target per dimension | Reviewed quarterly with the executive sponsor and the stream-aligned tenant council |
| **Later** | 4+ quarters | Capability *bets* — not yet committed, not yet staffed; include build-vs-buy decisions still open (chapter 02) | Revisited every quarter; items promote to Next only after the pull signal is explicit |

Three horizons is the deliberate compromise. Fewer and you either lose the quarterly rhythm (one horizon) or lose the forcing function for cross-team commitment (two horizons). More and the Later column becomes a wish-list nobody reads.

The CNCF maturity frame is attached to every row, not to the roadmap as a whole. An Interfaces investment that raises the scaffolding CLI to Level 3 does not raise Adoption; they are independent, and the roadmap must not pretend otherwise.

## The tenant register — the required prerequisite

You cannot author a stream-aligned roadmap without naming the stream-aligned teams. The **tenant register** is a one-page living document the platform team owns. Minimum columns:

| Column | What it carries |
|---|---|
| `tenant_id` | Stable identifier used in the SDK manifests, the policy plane, the eval catalogue, the SRE SLOs |
| `stream_name` | The line-of-business the tenant serves (e.g., "refund automation", "support copilot", "clinical research") |
| `primary_contact` | Named engineer and engineering manager on the stream-aligned team |
| `tier` | T1 (critical), T2 (production), T3 (experimental) — tied to mod-410's SLO tiers |
| `adopted_planes` | Which of the eleven planes the tenant is actively using |
| `exceptions_open` | Count of active exception-register entries (mod-412 chapter 07) |
| `blockers` | The top two things the tenant is waiting on the platform for |
| `last_touchpoint` | ISO-8601 of the last structured conversation (office hours, roadmap review, incident retro) |

The register is maintained in version control, reviewed monthly, and is the input to every roadmap decision. A roadmap entry without a tenant in the register pulling it is a tell: you are either building for a hypothetical user or for the platform team's own satisfaction.

A tenant register with fewer than three real tenants is a sign the platform has drifted from platform-team shape into complicated-subsystem shape. See [`mod-401` chapter 02](../mod-401-agent-platform-position-and-paved-road-philosophy/02-cncf-maturity-and-team-topologies.md) for the drift pattern.

## The roadmap entry shape

Every roadmap entry fits one row. The row has five load-bearing fields. If you cannot fill all five, the entry is not ready.

```yaml
- id: "RM-2026-Q4-07"
  title: "Scaffolding CLI reaches Interfaces Level 3"
  horizon: now
  owner: "@platform-plane-sdk"
  pullers:
    - tenant_id: acme-financial
      stream_name: refund automation
      reason: "Hand-rolling first-agent setup consumed 3 engineer-weeks per new agent in Q3"
    - tenant_id: globex-health
      stream_name: clinical-trials copilot
      reason: "Time-to-first-agent currently > 2 weeks; see quarterly adoption scorecard"
  maturity_target:
    dimension: Interfaces
    from_level: 2
    to_level: 3
    indicator: "Scaffolding CLI + local-dev loop + template catalogue operational; time-to-first-agent p50 < 24h"
  cost:
    engineer_weeks: 6
    opportunity: "Delays MCP registry pagination work (RM-2026-Q4-11) by one sprint"
  acceptance:
    - "platform new agent works end-to-end for Anthropic and OpenAI adapters"
    - "time-to-first-agent median across new engineers crosses 24h"
    - "Generation-receipt schema validates in CI (chapter 05, mod-412)"
  risks:
    - "Backstage Software Templates version pin lags upstream; migration playbook needed"
```

Walked field by field:

- **id** — stable across horizons; does not re-number when an entry promotes from Later → Next → Now. Used in incident records and retros to reference the investment.
- **horizon** — now / next / later. Promotion is a decision, not a drift; promotion is recorded in the governance log (see cadence section below).
- **owner** — one engineer or sub-team, by handle. "Platform team" is not an owner; "@platform-plane-sdk" is.
- **pullers** — at least one tenant from the register, with a *reason* (not a title). The reason is the forcing function: if the tenant could articulate it differently from what the platform team wrote, the pull is weak and the entry is either mis-scoped or speculative.
- **maturity_target** — the CNCF dimension this advances, the current level, the target level, and the indicator that proves the move. "Raises Adoption" is a claim; "raises Adoption from Operational to Scalable by ensuring new agents start on paved road by pull" is a decision.
- **cost** — engineer-weeks and *opportunity cost*. The latter is what the entry displaces. Without it, every entry looks free.
- **acceptance** — three to five specific, testable statements. "Teams are happier" is not acceptance; a measurable KPI delta or a dashboard state is.
- **risks** — one to three real risks the owner has thought about. Not a comprehensive risk register; the lead-indicator risks.

Entries without pullers are *platform-internal* entries — they still exist (maintenance, security hardening, migrations imposed by upstream) but are tagged explicitly as such. Platform-internal entries should not exceed roughly a quarter of the Now column; if they do, the platform has drifted into complicated-subsystem shape and stream-aligned teams are no longer the primary customer.

## Authoring the roadmap

Four steps. Each is a meeting or a document; the steps are serial.

### Step 1 — Tenant interviews

Not a survey. A 45-minute structured conversation with each T1 and T2 tenant's primary contact, held at most twice a year. The agenda is fixed:

1. What is your team shipping in the next two quarters?
2. What on the platform is making that hard?
3. What on the platform are you actively routing around? (This surfaces exceptions the register may be stale on.)
4. What would you build on top of the platform if you had the time and it were available?
5. What is one thing we should *stop* doing?

The last question is the one that forces honesty. A tenant that cannot name something to stop is either very happy, very polite, or very disengaged. The transcript or notes are committed to the tenant register's log.

### Step 2 — Maturity self-assessment

The platform team runs [`mod-401` exercise 03](../mod-401-agent-platform-position-and-paved-road-philosophy/exercises/exercise-03-cncf-platform-maturity-self-assessment.md) once a quarter. Each of Investment, Adoption, Interfaces, Operations gets a current level and the single next-step indicator that would raise it. The assessment is dated and signed; the next quarter's compares against it.

### Step 3 — Candidate list

Every tenant complaint, every maturity next-step indicator, every known migration (SDK version bump, OTel semconv rev, model-provider API change) is a candidate roadmap entry. The candidate list is draft-stage — no estimates, no acceptance criteria, no horizon assigned. Typically 20–40 candidates per quarterly refresh.

### Step 4 — Scoring and selection

Each candidate is scored on four dimensions:

| Dimension | 1 | 2 | 3 |
|---|---|---|---|
| **Pull signal** | Speculative | One tenant asked | Two or more tenants pulling with named cost |
| **Maturity move** | No dimension advances | Advances one level on one dimension | Advances one level on two dimensions, or unblocks a Level-4 indicator |
| **Risk of delay** | Low — can wait a quarter | Medium — tenants are starting to route around | High — a mandated migration deadline or a blocking incident risk |
| **Cost** | ≤ 2 engineer-weeks | 2–8 engineer-weeks | > 8 engineer-weeks |

Scores are not summed into a single number — that would hide the trade-offs. Instead, the Now column is filled with high-pull, high-maturity-move, high-risk-of-delay entries first. Cost is the budget constraint, not the ranking criterion.

A principle: **the Now column must fit the team's capacity minus 20 %**. The 20 % buffer is for incidents (chapter 03), security patches, and the unforeseeable. Teams that fill 100 % of capacity on planned work ship about 60 % of it and then feel permanently behind.

## The roadmap-review cadence

A roadmap is not a document, it is a *cadence*. The document is what the cadence writes to. The default cadence for this track:

### Weekly — platform team

30 minutes. One agenda:

- Any entry in Now blocked?
- Any new pull signal (incident, tenant complaint, support ticket) that changes a priority?
- Any Next entry that needs a design spike to be promotable?

Output: updated status on every Now entry; up to one Next entry promoted to Now, if capacity has opened.

### Monthly — with executive sponsor

60 minutes. Agenda:

- Adoption scorecard review (mod-412 chapter 06): where are the four KPIs moving?
- Any Now entry slipping by more than a sprint?
- Any Later entry promotable to Next? Record the decision.
- Any investment ask (hiring, tooling budget) the sponsor needs to action?

Output: a monthly roadmap snapshot, signed by the platform-team lead, archived in the governance log.

### Quarterly — with the tenant council

90 minutes. Attended by the primary contact from each T1 tenant (optional for T2). Agenda:

- Review last quarter's Now column: what landed, what did not, why.
- Review the four KPIs (adoption, time-to-first-agent, deployment success rate, rollback count) plus support MTTR (chapter 04).
- Walk the next quarter's proposed Now column; invite pushback.
- Walk one Later item that is being actively scoped (often a build-vs-buy decision; see chapter 02).
- Record attendance and action items; attendance is a signal in its own right.

Output: a signed quarterly scorecard (format from [`mod-412` chapter 06](../mod-412-paved-road-sdks-and-developer-experience/06-measuring-adoption-and-cncf-maturity.md)) plus the next quarter's Now column locked.

### Annual — strategy refresh

Once a year the roadmap is torn up and rebuilt from the tenant interviews and the maturity self-assessment. The entries that survive the rebuild are the ones that had both pull and maturity moves. The entries that die are the ones that were on the Later column for three consecutive quarters without promotion — a diagnostic that the pull was never real.

## A worked example — the Q4 roadmap

A concrete scenario, to make the shape tangible. The platform is on the back half of this track. Four tenants on the register (T1: `acme-financial`, `globex-health`; T2: `initech-robotics`, `umbrella-ops`). CNCF self-assessment: Investment Level 3, Adoption Level 2, Interfaces Level 2, Operations Level 3.

The Now column for Q4 lands at (capacity 20 engineer-weeks, buffered to 16):

| id | Title | Horizon | Owner | Pullers | Maturity | Weeks |
|---|---|---|---|---|---|---|
| RM-2026-Q4-07 | Scaffolding CLI reaches Interfaces Level 3 | now | @platform-plane-sdk | acme-financial, globex-health | Interfaces 2→3 | 6 |
| RM-2026-Q4-11 | MCP registry tool-search pagination | now | @platform-plane-mcp | acme-financial | Interfaces 2→3 (indicator: self-service) | 3 |
| RM-2026-Q4-14 | SLO burn-rate runbook for model-provider degrade | now | @platform-plane-sre | globex-health, umbrella-ops | Operations 3→4 (indicator: regression fixtures) | 2 |
| RM-2026-Q4-19 | Deprecate anthropic-v1 adapter (ladder step 2 → 3) | now | @platform-plane-sdk | — (platform-internal) | Interfaces 3 maintained | 1 |
| RM-2026-Q4-22 | Build-vs-buy review: MCP registry self-host vs managed | now | @platform-lead | council of 4 tenants | Investment Level 3 indicator | 2 |
| *(buffer)* | Incidents, migrations, security | — | rota | — | — | 2 |

Four entries are stream-aligned-pulled; one is platform-internal (adapter deprecation); one is a governance artefact (the build-vs-buy decision review that chapter 02 covers). The buffer is explicit.

The Next column holds 10–15 entries with named pullers. The Later column holds 8–12 bets (self-host vs managed registry resolution, agent-service migration pilot, A2A substrate switch).

The paragraph that lands in the monthly sponsor meeting: *"Q4 Now column is six entries at 20 engineer-weeks with a 2-week incident buffer. Four entries are directly pulled by acme-financial or globex-health (T1); one is a ladder migration imposed by the SDK rev cadence; one is a scheduled build-vs-buy decision. The headline moves are Interfaces from Level 2 to Level 3 (scaffolder shipped) and Operations from Level 3 to the Level 4 indicator (regression fixtures flowing from incidents). Investment and Adoption are not advancing this quarter — Adoption follows Interfaces by roughly a quarter, and the Q1 roadmap plans the push."*

That paragraph is the deliverable. The YAML entries behind it are the evidence.

## Common failure modes

The ways roadmaps fail are the ways platform teams lose credibility. Each has a corresponding fix.

### The wish-list roadmap

**Symptom:** The Now column has fifteen entries; the Next and Later columns have thirty each; nothing finishes on time.

**Cause:** Entries were never cost-scored against the team's capacity. Scope expanded to match engineer enthusiasm and leadership expectation.

**Fix:** Capacity-minus-20 % cap on the Now column, enforced in the weekly review. Reject entries without acceptance criteria at the promotion step.

### The platform-internal roadmap

**Symptom:** Every entry in the Now column is a refactor, a migration, or a security patch. Zero tenant pullers.

**Cause:** The platform team is optimising for its own clean house; stream-aligned teams have stopped routing requests through it.

**Fix:** Cap platform-internal entries at ~25 % of the Now column. If internal work genuinely dominates, call it a *remediation quarter* explicitly and set the quarter-after goal of returning to normal mix. Transparency protects credibility.

### The vendor-roadmap-import roadmap

**Symptom:** Every entry maps to a line from a vendor's whitepaper: "ship AgentOps feature X", "adopt new managed service Y", "integrate with evaluation vendor Z".

**Cause:** The platform is steering by vendor marketing rather than tenant pull. Easy to spot because the vendor's slide deck predicts the roadmap.

**Fix:** The pullers field catches this. A vendor's whitepaper is not a puller. If no tenant can articulate the reason the vendor's capability matters to *their* work, cut the entry.

### The architecture-first roadmap

**Symptom:** Long, prose-heavy architecture documents; no dated entries; no acceptance criteria.

**Cause:** The platform team has drifted into architect posture. Architecture is a handoff target (chapter 05), not the Now column's output.

**Fix:** Every entry commits to acceptance criteria measurable in the current quarter. Architecture diagrams live outside the roadmap, as reference material for the handoff to `ai-infra-architect` / `agentic-systems-architect`.

### The hero-engineer roadmap

**Symptom:** One or two engineers own most Now entries; promotion / absence risk is extreme.

**Cause:** The team scaled faster on commitment than on ownership distribution.

**Fix:** The owner field is per-entry, but the governance log reviews *distribution* quarterly. If one handle owns more than ~30 % of entries, the quarterly review flags it and the sponsor conversation includes a hiring or reallocation ask.

### The silent-handoff roadmap

**Symptom:** Entries in Later never promote; ownership is unclear; tenants ask about them and get an "we are thinking about it" response for three quarters.

**Cause:** The Later column became a graveyard. No decision rule promotes entries; nothing gets killed.

**Fix:** The annual strategy refresh kills Later entries that have been there for three consecutive quarters. A Later entry that cannot find a puller in a year was never real.

## What the roadmap is explicitly *not*

- **Not a product backlog.** Backlogs are for tickets; roadmaps are for capability investments. Keep them separate; let backlogs serve the Now column, not define it.
- **Not a vision.** Visions are longer-horizon and seldom dated. A platform vision ("agents at this organisation run on a paved road") belongs in the handoff to engineering management and the architect tracks (chapter 05).
- **Not a budget.** Budgets allocate dollars across engineer-weeks and tooling line-items; roadmaps allocate engineer-weeks across capability investments. The two interlock but are not the same artefact.
- **Not a Gantt chart.** Dependencies between entries exist (RM-2026-Q4-19 depends on RM-2026-Q3-04 shipping, say), but the roadmap records them as a `blocked_by` field, not a timeline bar. The quarterly rhythm is the timeline; tighter timing is for sprint planning.
- **Not a hiring plan.** Hiring sits one level up, with the sponsor. The roadmap informs it (which planes need owners next quarter?) but does not substitute for it.

## Summary

- A platform roadmap is a dated list of capability investments scored against stream-aligned pull and CNCF maturity dimensions, held across three horizons (Now / Next / Later).
- The tenant register is the required prerequisite. A platform with fewer than three named tenants cannot author a stream-aligned roadmap; see the drift patterns in [`mod-401` chapter 02](../mod-401-agent-platform-position-and-paved-road-philosophy/02-cncf-maturity-and-team-topologies.md).
- Every entry carries five load-bearing fields: owner, pullers, maturity_target, cost (including opportunity cost), acceptance. Entries without all five are not ready for the Now column.
- The cadence writes the roadmap: weekly with the platform team, monthly with the executive sponsor, quarterly with the tenant council, annually a strategy refresh.
- The Now column is capped at team capacity minus 20 %; platform-internal entries are capped at ~25 % of Now. Both caps are enforcement levers, not suggestions.
- Six common failure modes to watch: wish-list, platform-internal, vendor-import, architecture-first, hero-engineer, silent-handoff. Each has a corresponding discipline.

The next chapter turns from authoring the roadmap to one specific category of roadmap decision that appears in every quarter: *build versus buy*. Three live sub-decisions — self-host MCP registry vs. managed, self-host sandbox vs. E2B/Daytona/Modal, self-host orchestration vs. LangGraph Platform / Vertex Agent Engine / Azure AI Foundry Agent Service — anchor the chapter.
