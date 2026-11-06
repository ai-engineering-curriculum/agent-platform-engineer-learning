# Operating Model: CNCF Platform Engineering Maturity + Team Topologies

Chapter 01 fixed *what* the Agent Platform Engineer owns. This chapter fixes *how* the team that owns it operates. Two public frames, used together, carry most of the load:

- **Team Topologies** (Skelton & Pais, 2019) — the organisational pattern: a platform team exists to *reduce the cognitive load* of stream-aligned product teams, who are the platform's customers.
- **CNCF Platform Engineering Maturity Model** (CNCF Platforms WG, latest v1.1 at time of writing) — the operating maturity frame: four dimensions (Investment, Adoption, Interfaces, Operations) and four levels (Provisional, Operational, Scalable, Optimizing) to characterise where a platform team currently sits and what the next step is.

Both are public, both are cited in the role's job catalog (see `JOB_REQUIREMENTS.md` → Platform-engineering operating model). You are expected to speak them fluently.

## Team Topologies in one picture

The pattern that matters here:

```
┌──────────────────────┐   ┌──────────────────────┐   ┌──────────────────────┐
│  Stream-aligned      │   │  Stream-aligned      │   │  Stream-aligned      │
│  product team A      │   │  product team B      │   │  product team C      │
│  (refund agent)      │   │  (support copilot)   │   │  (research agent)    │
└──────────┬───────────┘   └──────────┬───────────┘   └──────────┬───────────┘
           │ consume                   │ consume                   │ consume
           ▼                           ▼                           ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      Agent Platform (platform team)                         │
│  Runtime │ MCP registry │ Sandbox │ Policy │ Gateway │ Eval │ Observability │
│  Safety plane │ Paved-road SDKs                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

Three claims from Team Topologies are load-bearing for you.

1. **The platform is a product, and its customers are the stream-aligned teams.** Not an internal audience. Not "developer productivity in the abstract". Specific teams with named lines of business, named failure modes, named frustrations. If you cannot list your top five stream-aligned teams and what each of them complains about, you are not operating as a platform team.
2. **The platform exists to reduce cognitive load, not to add it.** If adopting your paved-road SDK is harder than hand-rolling the agent against Anthropic's SDK + Temporal + Postgres directly, adoption is going to fail. Every requirement the platform imposes (OTel conventions, OPA policy decisions, MCP registry entries, eval gates) must either replace something a stream-aligned team already had to do, or deliver an outcome they could not reasonably build themselves.
3. **Platform teams are one of four team types, alongside stream-aligned, enabling, and complicated-subsystem.** Enabling teams coach stream-aligned teams through a one-off adoption; complicated-subsystem teams own a thing so specialised it would overwhelm any stream-aligned team (think: a serving substrate at a frontier-model operator). You will often have adjacent enabling or complicated-subsystem teams — be clear which you are. Platform teams own a long-lived, broadly-used substrate; if you are coaching one team through a one-off integration, you are acting as an enabling team that week.

Read the Team Topologies book in full if you have not. The practitioner site ([teamtopologies.com](https://teamtopologies.com/)) also publishes short pattern pages that are good secondary references.

## Why the model matters here, specifically

Agent platforms are especially prone to drifting out of the platform-team pattern in two directions:

- **Drifting into stream-aligned work** — "we'll just write the first refund agent for them" (see chapter 01). This is the agent-platform version of a platform team that quietly becomes a feature team for whichever stakeholder is loudest.
- **Drifting into complicated-subsystem work** — a sandbox substrate on Firecracker, a policy engine on OPA with 400 Rego policies, a safety plane with ten guardrail vendors; each of these can become specialised enough that nobody else can approach it. If your interfaces are not stable enough for a stream-aligned team to self-serve against, the subsystem has drifted out of platform-team shape even if it still looks like one on the org chart.

The CNCF maturity model is useful precisely because it names these drift directions explicitly.

## The CNCF Platform Engineering Maturity Model

Four dimensions, four levels. The canonical reference is the CNCF TAG App Delivery whitepaper (`https://tag-app-delivery.cncf.io/whitepapers/platform-eng-maturity-model/`). Internalise the grid; the rest of the track's modules return to it.

### Dimensions

| Dimension | What it measures for an agent platform |
|-----------|----------------------------------------|
| **Investment** | How the platform is funded and staffed. Is this a side project of one engineer, a chartered team with a mandate, or a cost-centred org with product management, docs, and oncall? |
| **Adoption** | Why stream-aligned teams choose to use the platform. Pushed by mandate, pulled because it saves time, or deeply integrated into how product teams work? |
| **Interfaces** | How product teams consume the platform. Ad-hoc tickets, documented APIs / SDKs, self-service portals with golden paths? |
| **Operations** | How the platform is run. Best-effort, with SLOs, with incident response, with error budgets stream-aligned teams opt into? |

### Levels (summarised against this track)

| Level | Investment | Adoption | Interfaces | Operations |
|-------|-----------|----------|------------|-----------|
| **1 — Provisional** | One engineer, no mandate | One or two teams try it | Copy a sample repo | Best effort |
| **2 — Operational** | Chartered team, platform PM | A handful of teams adopt by mandate | Documented SDK + API | On-call, basic SLOs |
| **3 — Scalable** | Funded as a product | Most new agents start on the paved road by pull, not push | Self-service portal, scaffolding CLI, local-dev loop, template catalog | Per-service SLOs, error-budget policy, chaos drills |
| **4 — Optimizing** | Portfolio investment; build-vs-buy run continuously | Stream-aligned teams co-author roadmap; paved road assumed | Golden paths + opinionated defaults; deprecation ladders run; adoption measured | DORA + SPACE + platform scorecards; regression fixtures flow back from incidents |

Map these to this track's modules:

- Investment and operations maturity are the back-of-track topics — `mod-410` (SRE for agents) and `mod-413` (platform leadership and cadence).
- Interfaces maturity is `mod-412` (paved-road SDKs and developer experience).
- Adoption maturity is the lens for every module. Even a technically excellent runtime (mod-402) will be a provisional-level platform if stream-aligned teams only adopt it under mandate.

A platform can sit at different levels on different dimensions — it is common for an early-stage agent platform to be level 2 on Operations (an on-call rota exists) while still level 1 on Interfaces (there is a sample repo but no scaffolding CLI). The maturity model is diagnostic, not a pass/fail grade.

## A worked example: evolving Operations maturity for the agent runtime

Take mod-402's runtime as a running example, and walk its Operations dimension through the levels:

- **Level 1 (Provisional)** — one engineer runs the runtime on a staging cluster, restarts it manually when it drops. No SLO. "It worked when I last checked."
- **Level 2 (Operational)** — a team owns the runtime. There is an on-call rota, a staging + prod cluster, and a documented `session_completion_rate` SLO. Incidents go through a lightweight postmortem.
- **Level 3 (Scalable)** — per-tenant SLOs. Error budget policy says a stream-aligned team that burns their budget on a bad deploy must pause releases until the budget recovers; teams opt in to this policy knowingly. Chaos drills for model-provider outage, MCP-server flake, sandbox exhaustion are run on a schedule.
- **Level 4 (Optimizing)** — incidents automatically produce regression fixtures that land in mod-407's eval harness; the next release cannot ship until the fixture passes. Platform scorecards publish per-tenant adoption, SLO burn, cost, and safety-refusal-rate deltas. Build-vs-buy of the runtime substrate is reviewed on cadence (quarterly, say) against what Temporal / Dapr / Restate / Inngest ship upstream.

The point of walking the grid is to make the *next step* legible. From level 2 Operations, the next step is "per-tenant SLOs and error budget policy" — not "DORA scorecard". The maturity model gives you an ordering.

## Put the two frames together

Team Topologies tells you *who* the platform is for and *what shape* the team must hold to keep delivering. CNCF maturity tells you *where you are* on the journey and *what the next step is* on each operating dimension.

Used together they produce a defensible operating-model story you can hand to leadership:

> "We are a platform team (Team Topologies) for the N stream-aligned agent product teams listed below. On the CNCF model we are currently Level 2 Operational on Investment and Operations, Level 1 Provisional on Interfaces, and Level 2 Operational on Adoption. Our next-quarter plan raises Interfaces to Level 2 by shipping the scaffolding CLI in mod-412 and a template catalog, and raises Adoption to Level 3 by onboarding product teams A, B, and C by pull."

That paragraph is the cadence story that mod-413 drills into.

## Common pitfalls when applying these frames

- **Treating "platform team" as a title rather than a shape.** If every stream-aligned team asks you for custom work, you are a stream-aligned team with a confused name. Rotate the request back onto their backlog, offer a paved-road improvement that makes it self-serve, or explicitly swap into an enabling-team shape for a short, named engagement.
- **Aspiring to Level 4 on everything before achieving Level 2 on the basics.** DORA scorecards do not fix a platform whose interfaces are "copy this sample repo". Climb the dimensions in order.
- **Confusing adoption-by-mandate with adoption-by-pull.** A product team that uses your platform because leadership told them to is a Level 1 or 2 adoption signal, not a Level 3. True Level 3 means new teams would pick the paved road over hand-rolling even if the mandate were lifted. Measure both.
- **Letting the maturity model substitute for user research.** Stream-aligned teams are the platform's customers; interview them, measure them, and let their actual complaints drive the next-step plan. The maturity model is a diagnostic lens on top of real user research, not a replacement for it.

## Summary

- Team Topologies gives you the team shape: a platform team serving stream-aligned product teams, measured by how much cognitive load the platform removes.
- CNCF Platform Engineering Maturity gives you four dimensions (Investment, Adoption, Interfaces, Operations) and four levels to diagnose the platform's current state and plan the next step.
- Both frames are prerequisites for the leadership and cadence work in `mod-413`, and both are the lens you should apply to every substrate module in this track.
- The two most common drifts for agent platforms specifically are drifting into stream-aligned work (writing the first product-side agent) and drifting into complicated-subsystem work (building something only the platform team can approach). Name the drift early and route out of it.

The next chapter turns from the operating model to the *engineering guidance* stream-aligned teams onboard with, and reads Anthropic's *Building Effective Agents* and OpenAI's *Practical Guide to Building Agents* as the source documents that should drive what your platform paved-roads.
