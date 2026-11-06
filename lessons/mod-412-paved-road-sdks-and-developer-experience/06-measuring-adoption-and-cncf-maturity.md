# Measuring Adoption and CNCF Maturity

Chapter 05 shipped the mechanism — the SDK, the scaffolder, the local-dev loop, the CI template, the instrumentation hooks. This chapter asks the next question: *how does the platform know whether the paved road is actually being used?*

A paved-road team that cannot answer that question is going to lose the argument for its own existence. "We shipped the SDK" is not the same as "teams use the SDK". The CNCF Platform Engineering Maturity Model names this gap explicitly: Interfaces and Adoption are two separate dimensions, and a platform can have mature Interfaces and immature Adoption at the same time — scaffolded a CLI, published docs, nobody onboarded.

The four KPIs this chapter builds are the ones the role spec calls out: **platform adoption rate**, **time-to-first-agent**, **deployment success rate**, and **rollback count**. Each has a pinned formula against the `AgentRevision` manifest stream (chapter 01), the CI pass/fail stream (chapter 05), and the deploy stream (mod-410). The chapter ends with how to read the four together against the CNCF maturity frame from `mod-401` chapter 02 and the common vanity-metric failure modes.

## Where the metrics come from

Three streams, one dashboard.

| Stream | Source | What it carries | Chapter |
|---|---|---|---|
| `AgentRevisionStream` | `platform build` emission from every CI job | One manifest per build, keyed to `git_revision` × `tenant_id` × `agent_id` × `generated_at` | 01, 05 |
| `CIEventStream` | The paved-road CI's own emissions | One record per CI run — gate pass/fail, duration, artefact references | 05 |
| `DeployEventStream` | The organisation's deploy pipeline (Argo CD, Spinnaker, custom) | One record per deploy — tenant, agent, revision, outcome (succeeded / rolled-back / failed) | mod-410 chapter 02 |

All three flow through the same event bus the observability plane (mod-408 chapter 04 collector tier) runs. The adoption dashboard is a set of queries over those streams; chapter 06 does not introduce new infrastructure.

### A note on provenance

The adoption metrics only count what the paved road can *see*. An agent that never emits an `AgentRevision` manifest is invisible to the adoption dashboard; that is by design. If a tenant forks the SDK, their agents fall off the dashboard; the drop surfaces exactly the trend the platform cares about. The dashboard distinguishes "on paved road" from "we lost visibility" explicitly — more below.

## KPI 1 — Platform Adoption Rate

**The question:** What fraction of agents (or tenants, or teams) are on the paved road?

**The formula** (choose the denominator carefully):

```
PlatformAdoptionRate(scope, window)
    = COUNT(DISTINCT agent_id WHERE AgentRevisionStream has an emission in window AND manifest is valid)
    / COUNT(DISTINCT agent_id in scope, discovered via the org catalogue)
```

Four decisions this formula forces:

1. **What is the denominator?** Not "all agents the platform knows about" (which is tautologically 100%). The denominator is the org catalogue — a system-of-record for "what agents exist", typically [Backstage](https://backstage.io/)'s software catalog, a service registry, or an internal spreadsheet reconciled quarterly. Missing from the denominator = invisible agents; chapter 07's enablement cadence works to flush them out.
2. **What counts as "on paved road"?** The manifest emitted in the last window (two weeks is a common pick; a quarter is the governance granularity). An agent whose last manifest emission is six months ago is *was* on paved road but is not *actively* on paved road.
3. **What is "valid"?** `AgentRevision.schema_version` parses; required fields are populated; the exception-register entries reference real register ids. An invalid manifest is a different failure from a missing manifest.
4. **What is the scope?** Tenant, workload class, tier, team. Split the metric by each — the global number is less useful than the per-dimension split that tells you *where* adoption is stuck.

The formula is intentionally honest. A tenant whose agents are on an exception-register entry counts as "on paved road with exception" — a split column on the dashboard, not a bucket that inflates the headline rate.

### Reporting shape

```
Platform Adoption Rate — rolling 14d — as of 2026-11-06
Scope                                   Rate        vs. prior 14d
----------------------------------------------------------------
Global                                  72%          +4pp
  Tenants
    acme-financial                      95%          +2pp
    globex-health                       68%          0
    initech                             42%          +15pp   [onboarding]
  Workload classes
    interactive-chat                    88%          +3pp
    batch-document                      61%          +1pp
    long-running-research               33%          +5pp
  Adapters
    anthropic                           41% of adopters
    openai                              32%
    google-adk                          15%
    microsoft                            7%
    langgraph-temporal                   5%
Fully on paved road                     68%
On paved road with exception            4%
Invisible (in catalog, no manifest)     28%
```

Three things load-bearing in this shape:

1. **The deltas are first-class.** A 72% headline is uninteresting; a 72% + 4pp trend is a decision. Chapter 07's office-hours cadence fires on the deltas.
2. **"Invisible" is named.** The dashboard does not hide what it cannot see — the 28% invisible is the enablement cadence's primary target.
3. **Adapter breakdown is for maintenance, not status.** If one adapter is 60% of adoption and another is 4%, the maintenance cost needs to reflect that. Chapter 02's "retire a framework" conversation reads from this split.

## KPI 2 — Time-to-First-Agent

**The question:** How long does it take a new engineer, starting from `git clone <template-catalogue>`, to get their first paved-road agent deployed?

**The formula:**

```
TimeToFirstAgent(engineer) = (first DeployEventStream record whose agent was scaffolded by this engineer).timestamp
                           - (generation-receipt.generated_at of their first `platform new agent` invocation).timestamp
```

Three sub-metrics the dashboard reports:

- **`T1`** — from `platform new` to first `make dev` success. Chapter 01's third obligation targets 90 seconds. Report p50, p95.
- **`T2`** — from first `make dev` success to first green CI on `main`. The paved-road CI shape sets this at hours, not days.
- **`T3`** — from first green CI to first production deploy. Depends on the organisation's deploy pipeline; the paved road does not own the full distribution but does report it.

The headline number is `T1 + T2 + T3`, broken down.

### Attribution

How does the dashboard know *which engineer* did the scaffolding? Two sources:

1. **The git author of the first commit** after `platform new` — written by the scaffolder into the generation-receipt.
2. **The SSO identity** of the user who authenticated to the paved-road CLI (if the CLI has an auth step). Backstage integrations, `gh auth`, `glab auth` all expose this.

Report both. A "git author" attribution catches cases where a senior engineer onboarded someone else; an SSO attribution is more precise for actual new-hire measurement.

### The ninety-second claim, verified in production

Chapter 01 obligation 3 says "the ninety-second budget is measured on every paved-road release". The CI job in chapter 05 verifies the smoke test against the template catalogue; `time-to-first-agent`'s `T1` dimension verifies it *against real engineers on real laptops*. The two together are the empirical paved-road claim.

### Reporting shape

```
Time-to-First-Agent — engineers onboarded in last 30 days
Onboarded engineers                     14
First-agent-deployed engineers          11  (3 ongoing)
Median T1 (new → make dev success)      78 s     [budget 90s — met]
Median T2 (dev → green CI)              2h 11m
Median T3 (green CI → deploy)           18h 32m
Median T_total                          20h 55m

Outliers:
  Engineer X: T1 142s (laptop had slow Docker start; filed in office-hours)
  Engineer Y: T2 9h 30m (eval regression debug; see fixture catalogue update)
```

## KPI 3 — Deployment Success Rate

**The question:** Of all paved-road builds that reached deploy, how many succeeded on first try?

**The formula:**

```
DeploymentSuccessRate(scope, window)
    = COUNT(DeployEventStream where outcome == succeeded_first_attempt
            AND deploy references an AgentRevision manifest within scope and window)
    / COUNT(DeployEventStream where deploy references an AgentRevision manifest within scope and window)
```

Two definitions that need pinning:

1. **"Succeeded first attempt" vs. "eventually succeeded"** — a deploy that failed, got rolled back, was redeployed with a small fix, and succeeded is *not* a first-attempt success. The paved road's claim is "the CI gates caught the problems"; a deploy that passed all gates and then failed in production is what KPI 4 (`rollback_count`) measures.
2. **"Deploy" means progressive-delivery promotion, not initial rollout**. The paved-road CI's green merge triggers a rollout, which progresses through staging / canary / primary. The `DeployEventStream` carries one record per *stage* transition. A deploy succeeded when it reached the primary stage without rollback.

### Breakdown

```
Deployment Success Rate — rolling 30d — as of 2026-11-06
Scope                                   Rate        vs. prior 30d
----------------------------------------------------------------
Global paved-road                       91%          +1pp
  By tier
    T1                                  96%          +1pp
    T2                                  89%          0
    T3                                  83%          +2pp
  By failure cause (of the 9% that failed)
    Safety-regression caught post-deploy 2%
    Eval regression caught post-deploy   3%
    Cost regression caught post-deploy   1%
    Runtime / infra failure              2%
    Human rollback decision              1%

Non-paved-road comparison (for context, 14-day window)
  Deployment Success Rate               78%  (manifest-less deploys)
```

The non-paved-road comparison is a load-bearing number. The paved road's claim to stream-aligned teams is "we catch problems in CI that would otherwise cost you a rollback". If the paved-road number is not materially better than the non-paved-road number, the claim is not being paid. Chapter 07's conversation with holdout tenants reads from this.

### The feedback loop into CI gates

Every "caught post-deploy" is a signal that a CI gate is miscalibrated. A safety regression that CI passed but production rolled back means the CI's red-team corpus does not cover the production attack. The fixture-promotion pipeline (mod-410 chapter 06) runs on these; the paved-road CI picks up the new fixture; the next build's gate catches the class.

## KPI 4 — Rollback Count

**The question:** How often does a paved-road agent ship a bad build and get rolled back?

**The formula:**

```
RollbackCount(scope, window)
    = COUNT(DeployEventStream where outcome == rolled_back
            AND deploy references an AgentRevision manifest within scope and window)
```

Two sub-metrics:

- **Rollback rate** — rollbacks per deploy (KPI 3's complement for the subset where the deploy *did* reach a stage before rollback).
- **Mean time to rollback** — from the rolled-back deploy's primary-stage entry to the rollback decision. A short MTTR means the SRE plane's SLIs are catching the regressions quickly; a long MTTR means something is taking too long to detect.

### Reporting shape

```
Rollback Count — rolling 30d — as of 2026-11-06
Rollbacks (paved-road)                  18
Rollback rate                           9%
Mean time to rollback                   27 min
  By trigger
    SLO burn-rate alert (mod-410)       8
    Safety-refusal-rate alert (mod-411) 4
    Manual                              4
    Post-incident (mod-410 chapter 05)  2
```

The breakdown by trigger is a direct read on which planes are working. If 70% of rollbacks fire on mod-410's SLO alerts, the SLO definitions are useful; if 70% are manual, the alerting is not sensitive enough.

## Reading the four KPIs together

Each KPI alone is a misread. The four together are a diagnostic.

| Pattern | What it suggests |
|---|---|
| High adoption, long time-to-first-agent, high rollback | Teams are pushed onto the paved road but the DX is bad; rollbacks are the symptom. Fix the DX (chapters 03 and 04). |
| Low adoption, short time-to-first-agent, low rollback | The paved road works for the few who try it, but you have not reached the rest. Enablement cadence (chapter 07) needs more pull signals. |
| High adoption, short time-to-first-agent, high rollback | Adoption is strong but production reality is harder than CI reveals. Strengthen the CI gates — eval regression, cost, latency. Chapter 05. |
| Steady adoption, steady time, steady rollback | Paved road is at steady state; shift investment from scaffolding to depth (adapter coverage, exception reduction). |
| Rising adoption + rising time-to-first-agent | Onboarding is drowning in demand. More enablement capacity (office hours) or more self-serve docs. Chapter 07. |
| Falling adoption + high rollback rate | The paved road is losing confidence. Chapter 07's deprecation ladder and migration playbooks become the triage artefact. |

The patterns are not a decision tree; they are a conversation-starter for the quarterly review (mod-413 owns the cadence).

## Mapping the KPIs to CNCF maturity

`mod-401` chapter 02 pinned the [CNCF Platform Engineering Maturity Model](https://tag-app-delivery.cncf.io/whitepapers/platform-eng-maturity-model/): four dimensions (Investment, Adoption, Interfaces, Operations), four levels (Provisional, Operational, Scalable, Optimizing). Two dimensions are where the paved-road KPIs land directly.

### Interfaces

The Interfaces dimension measures *how* product teams consume the platform. The levels, mapped to the chapters:

| Level | Indicator | Chapter / artefact |
|---|---|---|
| **1 — Provisional** | Copy a sample repo | No scaffolder |
| **2 — Operational** | Documented SDK + API | SDK shipped (chapter 02), docs exist |
| **3 — Scalable** | Self-service portal, scaffolding CLI, local-dev loop, template catalog | Scaffolder shipped (chapter 03), local-dev loop (chapter 04), template catalogue (chapter 03). `time-to-first-agent` below a declared budget. |
| **4 — Optimizing** | Golden paths + opinionated defaults; deprecation ladders run; adoption measured | Chapter 07 ladder operational; chapter 05 CI enforces defaults; adoption dashboard published. |

`time-to-first-agent` is the measurement that distinguishes level 2 from level 3 on Interfaces. A platform with an SDK but no scaffolder sits at level 2 regardless of the SDK quality. A platform whose `time-to-first-agent` median is under an hour has crossed to level 3; sub-fifteen-minutes is the level-4 posture.

### Adoption

The Adoption dimension measures *why* product teams use the platform. The levels, mapped to KPIs:

| Level | Indicator | KPI signal |
|---|---|---|
| **1 — Provisional** | One or two teams try it | Platform Adoption Rate < 10%; a few manifests from pilot tenants. |
| **2 — Operational** | A handful of teams adopt by mandate | Rate 10–40%; high exception-register population; `time-to-first-agent` long. |
| **3 — Scalable** | Most new agents start on the paved road by pull, not push | Rate 50–80%; new agents are near-100% on first scaffold. |
| **4 — Optimizing** | Stream-aligned teams co-author roadmap; paved road assumed | Rate > 80%; exception register near-empty; deprecations completed on cadence; rollback rate below non-paved-road baseline. |

The Rate is the headline Adoption indicator; the exception-register population is the lead indicator that distinguishes pull from push. A high rate with a high exception count is adoption-by-mandate; a high rate with a near-empty exception register is adoption-by-pull.

### The scorecard

Produce one scorecard per quarter. The scorecard is what mod-413 governs; the paved-road plane produces the evidence.

```
Paved-Road Scorecard — Q4 2026 — generated 2026-11-06
Owner: agent-platform-team

CNCF Maturity
  Investment       Level 3 — funded as a product with PM + oncall; see mod-413 roadmap.
  Adoption         Level 3 — Rate 72% (+4pp vs Q3); exception register 11 open (-4).
  Interfaces       Level 3 — scaffolder, local-dev loop, template catalogue operational;
                              time-to-first-agent median 20h 55m, T1 78s (budget 90s met).
  Operations       Level 3 — SLOs defined, error-budget policy signed by 8 tenants;
                              cross-reference mod-410.

KPIs (last 90d)
  Platform Adoption Rate              72%   (+4pp vs Q3)
  Time-to-First-Agent (T_total p50)   20h 55m   (-6h vs Q3)
  Deployment Success Rate             91%   (+1pp vs Q3)
  Rollback Count                      18    (-5 vs Q3)

Advances since Q3
  - agent-chat-microsoft template entered status: active
  - LangGraph+Temporal adapter version 2.4 released
  - anthropic-v1 adapter entered status: deprecated (ladder step 2)

Open risks
  - 28% invisible-agents population in globex-health; enablement scheduled.
  - OpenAI Agents SDK v2 upstream bump requires adapter major; migration playbook drafted.
```

The scorecard is:

- **Short.** One page is the right length; quarterly reviewers read a page, not ten.
- **Signed.** The platform-team lead signs; mod-413 lands the signatures in the governance log.
- **Pinned to the generation timestamp.** The dashboard moves in real time; the scorecard is a snapshot the next review compares against.

## Vanity-metric failure modes

Four common ways the adoption measurement fails. Watch for each; if the dashboard shows the pattern, the measurement is misleading.

### "Hello-world" inflation

**The symptom:** Platform Adoption Rate climbs sharply; time-to-first-agent falls; deployment success rate falls.

**The cause:** New engineers scaffold agents for training / exploration and never deploy them. They inflate the manifest count; they do not represent real production workloads.

**The fix:** Split the Rate by "agents with ≥ N production turns in the window" or "agents whose manifest references at least one real deploy". Keep both numbers on the dashboard — the headline is honest, but the production-workload filter is where the pull signal lives.

### "Mandate" adoption

**The symptom:** Rate is high; exception register is small; `time-to-first-agent` is normal; rollback is high.

**The cause:** Teams are mandated to adopt the paved road and do the minimum to pass the gates; they neither understand nor trust the plumbing. Rollback rate climbs because the agents are not well-integrated into the planes.

**The fix:** Chapter 07's enablement cadence. Pair every new tenant with an office-hours onboarding; make the first production deploy a gated, assisted step; the mandate alone will not stick.

### "Framework-captive" adoption

**The symptom:** Adapter breakdown is extremely skewed; one adapter is 80% of adoption; the others are 5% each.

**The cause:** Either the platform picked one framework and the SDK pretends to support five, or the stream-aligned teams naturally cluster around one and the others' maintenance cost is not justified.

**The fix:** Review the adapter maintenance cost. Chapter 07's deprecation ladder might retire low-adoption adapters; the exception register absorbs the few teams that need them.

### "Metric-chasing" adoption

**The symptom:** Rate is maximised; `time-to-first-agent` is minimised; rollback rate is normal; satisfaction surveys are flat or negative.

**The cause:** The platform is optimising for the dashboard, not the engineer. Scaffolds get shorter, defaults get looser, gates get softer — adoption numbers climb, pull signal falls.

**The fix:** The dashboard is one signal; the pull signal is another. Add qualitative instruments: quarterly developer-experience survey (crossref the [DevEx framework](https://queue.acm.org/detail.cfm?id=3595878) — feedback loops, cognitive load, flow state), office-hours topic counts, exception-register dwell times. Chapter 07.

## Pull vs. push — the central distinction

The CNCF frame distinguishes push adoption (mandated) from pull adoption (chosen). The lead indicator is not the headline Rate; it is the conjunction of three signals:

1. **Exception-register population per tenant is low.** Teams that have adopted by pull are using the defaults; teams that have adopted by push are running the maximum number of exceptions.
2. **Time-to-first-agent is low.** Teams that chose the paved road are using the scaffolder's defaults; teams forced on it are hand-editing the generated repo.
3. **Deployment success rate is high and rollback rate is low.** Pull adopters trust the gates and let them fail; push adopters route around the gates and pay in production.

A platform at CNCF Adoption level 3 shows all three. A platform showing one or two but not the third is at level 2 with a Potemkin-village level-3 headline.

## Satisfaction instruments, briefly

The dashboard captures the structural signals. Two orthogonal instruments capture the human signal:

1. **A short quarterly developer-experience survey.** Five questions, aligned to the DevEx framework (feedback loops, cognitive load, flow state); same five every quarter; split by tenant and workload class.
2. **Office-hours topic clustering.** Chapter 07 runs weekly office hours; the questions that come up are the leading indicator of where the paved road hurts.

Both feed the scorecard's "open risks" section and the chapter-07 enablement roadmap.

## Dashboard operational discipline

Four rules:

1. **Dashboards have an owner.** A panel without an owner is a panel that rots; the adoption dashboard's `owner` is the paved-road plane's maintainers.
2. **Formulas are pinned in a file.** `platform/adoption-dashboard/metrics.yaml` is the source-of-truth for every formula; dashboard panels reference it; chapter 05's CI validates it on change.
3. **Historical panels never rewrite.** A metric definition change does not retroactively edit historical data; the old formula's panel stays live for one quarter so reviewers can compare. The new panel appears alongside; mod-413's quarterly review sees both.
4. **Alerts stay on the SRE plane.** The adoption dashboard does not page people. mod-410 owns paging; chapter 06's signals feed into mod-413's governance review, not into the SRE on-call.

## Common pitfalls

- **Using the headline Rate as the only adoption number.** Covered — split by tenant, workload class, adapter, exception; the headline is the top-of-funnel, the splits are where decisions live.
- **Measuring adoption without measuring satisfaction.** Covered — the DevEx-framework-aligned survey is the orthogonal instrument; never ship the dashboard without it.
- **Confusing the catalog with the production inventory.** The org catalogue may include retired agents, planning placeholders, and test stubs; reconcile quarterly with mod-413.
- **A `deployment_success_rate` that counts the CI, not the deploy.** The gate pass rate is a CI-stream metric (chapter 05); the deployment success rate requires the deploy stream. Mixing them produces a reassuring but irrelevant number.
- **Attributing time-to-first-agent to the SDK alone.** The three sub-components (`T1`, `T2`, `T3`) belong to different owners — paved-road SDK owns `T1`, the paved-road CI owns `T2`, the organisation's deploy pipeline owns `T3`. Report separately; blame fairly.
- **Letting the invisible-agents fraction grow without naming it.** Covered in reporting-shape; the enablement cadence (chapter 07) treats invisible agents as the first thing to flush.
- **Treating the scorecard as a defense exhibit instead of a diagnostic.** The scorecard is for making the next investment decision, not for proving the platform's right to exist. Hide the "proving" instinct and the scorecard improves.

## Summary

- Four adoption KPIs with pinned formulas against the manifest / CI / deploy streams: Platform Adoption Rate, Time-to-First-Agent (`T1` + `T2` + `T3`), Deployment Success Rate, Rollback Count.
- Each KPI has a reporting shape that forces the honest split: tenant / workload class / adapter / exception / tier. The headline numbers inform the trend; the splits inform the decisions.
- The KPIs map onto the CNCF Platform Engineering Maturity Model's Interfaces (scaffolder + local-dev loop + template catalog + time-to-first-agent budget) and Adoption (Rate + exception-register population + rollback delta vs. non-paved-road) dimensions.
- Pull adoption is distinguished from push adoption by the conjunction of low exception-register population, low time-to-first-agent, and low rollback rate — not by the headline Rate alone.
- Four vanity-metric failure modes to watch: hello-world inflation, mandate adoption, framework-captive adoption, metric-chasing adoption. Each has a corresponding fix.
- The adoption dashboard is diagnostic for the governance cadence in `mod-413`; it is not an alerting surface.

The next chapter turns from the measurement to the practice — the office-hours cadence, the migration playbook template, and the five-rung deprecation ladder that turns "the paved road exists" into "the paved road is the default road".
