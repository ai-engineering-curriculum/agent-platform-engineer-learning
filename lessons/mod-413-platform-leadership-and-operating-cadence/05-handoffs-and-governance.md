# Handoffs to Architecture and Engineering Management

Chapters 01–04 covered the operating surface the Agent Platform Engineer owns: the roadmap, the build-vs-buy portfolio, the incident command, and the scorecard. This chapter closes the module by routing what the role *hands off* when the decision rises above the plane layer.

The handoffs matter because the role has a ceiling. The platform engineer at level 40 can stand up an entire agent platform, operate it, measure it, and lead it cross-team at the quarterly cadence. What they cannot — and should not — own is cross-stack infrastructure strategy that spans the agent platform *and* the inference substrate *and* the data platform, agent-system design decisions that re-shape the organisation's agent architecture, or the executive-sponsor relationship that unlocks headcount and budget. Those sit with:

- **[`ai-infra-architect`](../../../ai-infra-architect-learning/) (level 45)** — cross-stack infrastructure strategy across inference, agent platform, data plane.
- **[`agentic-systems-architect`](../../../agentic-systems-architect-learning/) (level 48)** — agent-architecture decisions that span multiple product lines.
- **Engineering management (level 45+)** — roadmap governance, executive sponsorship, headcount, cross-org coordination.

A platform engineer who takes on these roles de facto when there is no one in them is being helpful and, at the same time, losing the ability to run the planes they actually own. The right posture is to write the handoff cleanly and run the operating cadence at level 40 while the architect / manager roles do their work.

This chapter pins what each handoff carries, what the platform engineer stays on the hook for, and the failure modes when the handoffs are missing.

## The three handoff targets

### Handoff 1 — `ai-infra-architect` (level 45)

**Scope:** Cross-stack infrastructure strategy. The agent platform sits atop an inference substrate (hosted model gateways, self-hosted inference clusters, batch-inference pipelines), a data platform (feature stores, vector DBs, data lakes), and shared infrastructure (identity, networking, secret management, observability backends). Decisions that reshape *multiple* of these simultaneously sit with the ai-infra-architect.

**What this role hands off:**

- **Build-vs-buy decisions whose scope spans the agent platform and another substrate.** Example: "should the model gateway be the inference substrate's front-door, or should it be a thin router above a shared inference service that another team owns?" — this is an architecture question, not a platform question.
- **Tool-registry and MCP governance that touches the data plane.** Example: "the MCP server catalog should federate with the data platform's data-catalog" — the governance across catalogs is architect-level.
- **Capacity planning that crosses substrates.** Example: "the agent runtime's bursty workload on the sandbox pool competes with the batch-inference pipeline for GPU budget" — the trade-off is architect-level.
- **Security architecture for the whole stack.** Example: "the policy plane's identity model should be the organisation-wide identity model, not an agent-platform-specific one" — the architect owns the decision, the platform engineer implements the integration.

**What this role keeps:**

- Plane-level decisions inside the agent platform: which runtime substrate, which registry, which sandbox vendor.
- The operating cadence (chapter 01) and the scorecard (chapter 04).
- Incident command (chapter 03) for incidents that are agent-platform-originated.

**The handoff artefact:**

A one- or two-page *architecture ask* document. Three sections:

1. **Context.** The agent-platform situation that is prompting the ask. Include references to the roadmap entries, the scorecard rows, and the build-vs-buy records.
2. **The decision.** The specific choice that needs to be made at the architect level — not "please advise", but "we are proposing X; alternatives are Y and Z; constraints are A and B".
3. **The acceptance criteria.** What outcome, by what date, with what owner, closes the ask. The platform engineer is the proposer; the architect is the decider.

The handoff is not a hand-off-and-forget. The platform engineer participates in the architect's review, implements the chosen direction, and reports back on the scorecard. The architect does not run the platform; the platform engineer does not make cross-stack calls.

### Handoff 2 — `agentic-systems-architect` (level 48)

**Scope:** Agent-architecture decisions that span multiple product lines or multiple organisations within the company. The `agentic-systems-architect` is at the top of the AI-engineering level ladder for the role family; they own how agents *should be built* at the organisation.

**What this role hands off:**

- **Shape-of-paved-road questions that require a cross-product answer.** Example: "should all customer-facing agents adopt a consistent A2A handoff model, or should each product line define its own?" — this is a cross-product agent-architecture call.
- **Multi-year bets on agent paradigms.** Example: "should the platform optimise for single-agent workflows or multi-agent collectives?" — this is a design-direction call, not a plane-level decision.
- **Platform federation.** Example: "the acquisition we closed last quarter has their own agent platform; should we federate them, migrate them, or run both?" — this is an org-level architecture decision.
- **Research-into-practice handoff.** Example: "a recent frontier-safety result changes what the safety plane must defend against; what is the organisation's response?" — the agentic-systems-architect coordinates with the `agentic-safety-engineer` track and the platform implements.

**What this role keeps:**

- The implementation of the chosen agent architecture at platform-plane altitude.
- Making the paved road *this* paved road — the specific SDK adapters, the specific policy decisions, the specific eval fixtures.
- The scope-policing of what the platform will and will not build (chapter 01 roadmap discipline).

**The handoff artefact:**

A *design-direction paper*. Two or three pages; typically authored by the architect, with the platform engineer as a named contributor. Sections:

1. **The paradigm.** What shape of agents is the organisation aiming for over 2–3 years.
2. **The implication for each plane.** How the runtime, registry, sandbox, policy, model gateway, eval, observability, A2A, SRE, safety, SDK planes must change to support it.
3. **The platform-engineer's roadmap translation.** The architect does not author the roadmap; the platform engineer translates the design direction into roadmap entries. The paper and the roadmap refer to each other.

The relationship: the platform engineer *implements* the architect's direction and *pushes back* when the direction is unimplementable at the current maturity. The push-back is not insubordination; it is data. An architect who insists on an un-implementable direction is a bad architect; a platform engineer who implements an un-implementable direction becomes the owner of its failure.

### Handoff 3 — Engineering management (level 45+)

**Scope:** The organisational surface of the platform — headcount, sponsorship, cross-org coordination, performance management of the team, budget authority. Engineering management is not usually a single track in the technical-ladder sense; it is a distinct ladder (manager, director, VP). The platform engineer at level 40 reports to an engineering manager at level 45+.

**What this role hands off:**

- **Headcount asks.** The platform engineer identifies the need from the roadmap and the scorecard; the engineering manager approves, posts, and makes the offer.
- **Executive-sponsor relationship.** The platform engineer runs the monthly sponsor review (chapter 01); the engineering manager owns the sponsor relationship between reviews — the sponsor-to-sponsor conversations, the sponsor's cross-org asks.
- **Cross-org coordination at scale.** The platform engineer runs the tenant council (chapter 01); the engineering manager coordinates with *other* engineering managers (data platform, inference infrastructure, security engineering) at a leadership cadence.
- **Performance management of the platform team.** The engineering manager owns promotions, performance reviews, compensation, and the career ladder of the platform engineers.
- **Budget authority.** Build-vs-buy decisions above a dollar threshold are the manager's call to authorise; the platform engineer authors the decision record.
- **Hiring plans and attrition mitigation.** The manager owns the workforce planning that makes the roadmap executable.

**What this role keeps:**

- The roadmap *authorship* (the manager approves, but the platform engineer authors).
- The scorecard *reading* (the manager signs the scorecard; the platform engineer writes it).
- The technical decisions at plane-level altitude.
- Incident command when the incident is technical; the manager owns the *organisational* consequences of incidents (crisis communication, legal, post-incident org change).

**The handoff artefact:**

The *monthly one-on-one* and the *quarterly leadership review*. The one-on-one is the operating cadence; the quarterly review is where the scorecard surfaces to the manager's own leadership.

## When one or more handoff targets does not exist

A pragmatic reality: in many organisations, one or more of these roles is unfilled or combined with another role. The platform engineer at level 40 may be the senior-most AI-engineering role present. What then?

Three postures, in order of preference.

### Posture 1 — Make the gap visible

Name the missing role in the roadmap's "open risks" section. "This platform currently has no architect role; the following decisions are stacking up that require level-45 authority: …". The manager and the sponsor should see the gap and *either* staff it, *or* explicitly delegate the authority to the platform engineer for a defined period.

The explicit delegation is the key move. Operating as if you have authority you do not have is how decisions get reversed six months later. Operating with explicitly-delegated authority is a role expansion, time-limited, and reviewable.

### Posture 2 — Operate below the level that should own the decision

Where the authority is not delegated, the platform engineer runs the decision *at plane level* and flags the architecture implication. Example: a build-vs-buy decision for the sandbox substrate is run as a plane-level decision; the record notes "a cross-stack review with the inference substrate is advisable; no architect currently; decision stands until that review is scheduled."

The decision gets made; the record makes clear what it was *not*. Future revisions have the context.

### Posture 3 — Decline the decision

Some decisions cannot be made at level 40 without becoming the owner of their failure. A multi-year agent paradigm commit, a mandatory-use-managed-service decision from an executive, a federation decision for an acquisition — each is a level-45+ decision that should not be made by a level-40. The platform engineer's move is to decline until the right role is present or the authority is explicitly delegated.

"I am a level-40 and this decision is a level-48 decision; I cannot run it" is a legitimate professional stance. The manager's job is to either (a) delegate, (b) escalate, or (c) accept the decision stays open.

## The reverse flow — what comes back from the architects and management

The handoffs are bi-directional. Each role sends work *back* to the platform engineer that is theirs to execute.

### From `ai-infra-architect`

- **Cross-substrate standards to adopt.** Example: a new organisation-wide identity model; the platform engineer integrates the policy plane.
- **Capacity allocations.** Example: a GPU budget assigned to the agent runtime's sandbox pool; the platform engineer plans against it.
- **Reference-architecture updates.** Example: a new org-wide secret-management pattern; the platform engineer migrates the plane configurations.

### From `agentic-systems-architect`

- **Design-direction updates.** Example: a shift toward multi-agent paradigms; the platform engineer prioritises the A2A plane's roadmap entries.
- **Research-into-practice directives.** Example: a mandated adoption of a new OTel-GenAI attribute set; the platform engineer updates the semantic conventions in the SDK.

### From engineering management

- **Roadmap approvals and vetoes.** The manager signs the Now column; cuts the entries the sponsor pushes back on.
- **Team structure changes.** Example: splitting the team into an eval-focused sub-team and a runtime-focused sub-team; the platform engineer re-owns the roadmap row structure.
- **Budget allocations.** Example: a new quarterly budget for managed services; the platform engineer re-runs the build-vs-buy portfolio.

The reverse flow is not visible to the tenants. From the stream-aligned teams' perspective, the platform's capabilities and commitments are the platform engineer's; the invisible architecture and management work is what makes the commitments durable.

## The governance log

Every handoff generates an artefact that lands in the platform's *governance log*. The log is a single directory in version control, append-only, timestamped. Minimal structure:

```
platform-governance/
├── roadmap/
│   ├── 2026-Q4-roadmap.md
│   ├── 2026-Q3-roadmap.md
│   └── …
├── build-vs-buy/
│   ├── BvB-2026-Q4-02-runtime-decision.md
│   └── …
├── incident-records/
│   ├── inc-20261106-003.md
│   └── …
├── architect-asks/
│   ├── arch-ask-2026-Q4-01-mcp-registry-federation.md
│   └── …
├── design-direction-papers/
│   ├── ddp-2026-01-multi-agent-paradigm.md
│   └── …
├── scorecards/
│   ├── 2026-Q4-scorecard.md
│   └── …
└── tenant-register.md
```

Every artefact that leaves the plane level goes into this directory. The directory is read-only to consumers; append-only to the platform-engineer and the architect / manager roles. A six-month retrospective on the platform reads the log; nothing happens outside it.

A principle: *the governance log is not for performance review*. Nobody's PR comes from the log; its purpose is institutional memory. The moment the log is weaponised for performance management, it will degrade into a presentation surface and lose its diagnostic value.

## Where this track stops

The role described in this track is bounded. The track handles:

- Standing up and operating all eleven planes.
- Authoring the paved-road SDK and measuring its adoption.
- Running the quarterly operating cadence.
- Commanding cross-team incidents.
- Measuring success across the four instruments.

It does not handle:

- Designing the next paradigm of agent architecture for the organisation (`agentic-systems-architect`).
- Making cross-stack infrastructure trade-offs between the agent platform and the inference substrate, data plane, or shared infrastructure (`ai-infra-architect`).
- Executive sponsorship, headcount authority, cross-org management (engineering management).
- Product-side agent correctness and shipping decisions (`senior-agentic-ai-engineer` / `agentic-ai-engineer`).
- Model-provider relationships at commercial scale (typically the architect or engineering management).

The practitioner who is strong at level 40 may promote into one of these — most often into `ai-infra-architect` for the infrastructure-leaning path, or into engineering management for the people-leaning path. The ladder is explicit so the roles above the platform engineer are legible targets, not vague abstractions.

## Common failure modes

### The absent-architect trap

**Symptom:** A platform engineer makes multi-year paradigm commits, cross-stack trade-offs, and federation decisions because the architect role is unfilled. Six months later the decisions reverse and the engineer is the one holding the bag.

**Cause:** The gap was not made visible; authority was not explicitly delegated.

**Fix:** Posture 1 above. Name the gap in the roadmap; either get explicit delegation or decline the decision.

### The architect-overreach trap

**Symptom:** An architect makes plane-level decisions the platform engineer should own — picks the registry vendor, writes the policy rules, authors the eval fixtures.

**Cause:** The architect is excited, or has not understood the altitude of the role.

**Fix:** The handoff artefacts (architecture ask, design-direction paper) carry *what the architect decides* and *what the platform engineer decides*. The two sections are explicit. A design-direction paper that reads like a plane-level spec is a signal to re-scope.

### The manager-as-decider trap

**Symptom:** The engineering manager makes technical decisions the platform engineer should own — picks the sandbox substrate, authors the SDK adapter list.

**Cause:** The manager is a former engineer and cannot resist; or the platform engineer has deferred too much.

**Fix:** The roadmap is authored by the platform engineer; the manager approves. Technical decisions are flagged in one-on-ones as "this is my call; I want your buy-in, not your authorship."

### The handoff-as-escalation trap

**Symptom:** Every tricky decision escalates up the ladder; the platform engineer never decides, waits for the architect or manager.

**Cause:** Avoidance of responsibility, or unclear authority.

**Fix:** The platform engineer's authority at level 40 is explicit. Decisions at plane level, decisions within the roadmap's capacity budget, incident command, scorecard authorship — these do not require escalation. If the engineer is escalating them, the problem is professional posture, not organisational structure.

### The silent-reverse-flow trap

**Symptom:** Architects and managers send work back to the platform engineer that is not visible in the roadmap or the scorecard. The platform engineer absorbs it informally.

**Cause:** The reverse flow does not enter the governance log.

**Fix:** Architect-directed work and manager-directed work appear as roadmap entries with the architect / manager as the *puller*. The pull is formal, visible, and scored on the rubric. Hidden reverse-flow work is how roadmaps slip quietly.

## Summary

- The platform-engineer role at level 40 hands off three things: cross-stack infrastructure strategy (to `ai-infra-architect`), cross-product agent-architecture direction (to `agentic-systems-architect`), and organisational scope — headcount, sponsorship, cross-org coordination — (to engineering management at level 45+).
- Each handoff has an artefact: the architecture ask, the design-direction paper, the monthly one-on-one plus quarterly leadership review.
- The reverse flow is real: architects and managers send work back that the platform engineer executes. Every piece of reverse-flow work lands as a roadmap entry with the architect / manager as puller.
- When a handoff target role is unfilled, the posture is make-the-gap-visible → operate-below-the-decision → decline. Operating with undelegated authority is how decisions reverse.
- The governance log is the institutional memory: roadmaps, build-vs-buy records, incident records, architect asks, design-direction papers, scorecards, tenant register. Append-only, in version control.
- Five common failure modes: absent-architect trap, architect-overreach trap, manager-as-decider trap, handoff-as-escalation trap, silent-reverse-flow trap. Each has a corresponding discipline.

The platform-engineer role is bounded by design. The practitioner who runs it well — at every level of the four maturity dimensions, across every one of the eleven planes, with the four scorecard instruments honest — is the one who grows into the architect or management role that sits above. Running it *beyond* its bounds is how practitioners burn out and platforms stall.

This closes the module and the track.
