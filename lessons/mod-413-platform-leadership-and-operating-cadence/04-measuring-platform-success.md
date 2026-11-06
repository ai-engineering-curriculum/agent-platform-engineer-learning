# Measuring Platform Success — DORA, SPACE, Adoption, and Support MTTR

Chapter 03 pinned the incident posture. This chapter pins the long-running counterpart: *how does the platform know, across quarters, whether it is working?*

Three published frames, plus one domain-specific instrument the paved-road track owns, cover the ground:

- **[DORA](https://cloud.google.com/devops/state-of-devops)** — the four delivery-performance metrics (Deployment Frequency, Lead Time for Changes, Change Failure Rate, Mean Time to Restore). The *delivery* lens.
- **[SPACE](https://queue.acm.org/detail.cfm?id=3454124)** — Satisfaction, Performance, Activity, Communication, Efficiency (Forsgren et al., 2021). The *human* lens that complements DORA.
- **Platform-adoption KPIs** from [`mod-412` chapter 06](../mod-412-paved-road-sdks-and-developer-experience/06-measuring-adoption-and-cncf-maturity.md) — platform adoption rate, time-to-first-agent, deployment success rate, rollback count. The *paved-road* lens.
- **Support-ticket MTTR** — the operational-friction lens.

No single lens is sufficient. DORA alone makes a platform look healthy while its engineers burn out. Adoption alone can report success while deploys are failing. SPACE alone cannot tell you whether the roadmap moved. The four together form a cross-checked picture that the quarterly review (chapter 01) and the executive sponsor read.

This chapter pins the metrics, the formulas, the measurement discipline, the vanity-metric failure modes, and the specialisation each metric needs for an agent platform.

## The four instruments, side by side

| Lens | What it measures | Published source | Agent-platform adaptation |
|---|---|---|---|
| **DORA** | Delivery performance of code to production | Google / DORA | Measured on the *paved-road tenants*, not on the platform-team's own deploys |
| **SPACE** | Developer experience across five dimensions | ACM Queue (Forsgren, Storey, Maddila, Zimmermann, Houck, Butler) | Instrumented via a quarterly survey of platform *users* (stream-aligned teams), not the platform team itself |
| **Platform adoption** | Whether the paved road is used, and by whom | CNCF Platform Engineering Maturity Model indicators | `mod-412` chapter 06's four KPIs |
| **Support MTTR** | Operational friction the platform causes | Public ITSM conventions; platform-engineering state-of-the-art | A domain-specific instrument the paved-road track owns |

The read-together posture is what this chapter is about; the measurement discipline for each instrument is below.

## DORA for an agent platform

DORA's four metrics are published, well-defined, and canonical. They were authored for web-service delivery; the adaptation for agent platforms has three subtleties.

### The four DORA metrics

| Metric | Definition | Elite range (public DORA reports) |
|---|---|---|
| **Deployment Frequency** | How often an organisation successfully releases to production | On-demand (multiple deploys per day) |
| **Lead Time for Changes** | Time from code committed to code successfully running in production | Less than one day |
| **Change Failure Rate** | Fraction of deployments that cause a failure in production requiring remediation (rollback, hotfix) | 0–15 % |
| **Mean Time to Restore (MTTR)** | How long it takes to restore service after a production failure | Less than one hour |

The source of truth for the published definitions is the [Google DORA site](https://cloud.google.com/devops/state-of-devops) and the [`dora.dev`](https://dora.dev/) community programme. <!-- needs-research: the Elite / High / Medium / Low thresholds move year to year in the State of DevOps report; verify against the most recent report before quoting in a scorecard. -->

### Adaptation 1 — measure the tenants, not the platform team

The DORA metrics apply to the organisation that ships value to users. For the agent-platform engineer, the organisation is the *stream-aligned tenants*, not the platform team itself.

- Deployment Frequency: deploys of *tenant agents* to production, via the paved-road CI and the organisation's deploy pipeline.
- Lead Time for Changes: tenant-side commit-to-production time.
- Change Failure Rate: tenant-side deploys that get rolled back.
- MTTR: time from a tenant agent's incident declaration to recovery, per chapter 03.

The platform team's own deploys (SDK rev, MCP registry update) are a secondary DORA panel — important, but not the primary scorecard.

### Adaptation 2 — the deploy pipeline is cross-boundary

A tenant agent's deploy traverses: paved-road CI (chapter 05 of mod-412) → tenant's own CD pipeline → the deployment substrate. The platform owns the first step; the tenant owns the second; the organisation owns the third. The DORA panel must attribute each metric to the right owner.

- **Deployment Frequency** is a tenant-owned metric. The platform's role is to make per-deploy cost low enough that frequency is a tenant choice, not a platform limitation.
- **Lead Time for Changes** is split. `T_CI = merge → CI green` is platform-owned; `T_deploy = CI green → prod` is tenant / organisation-owned. Report both.
- **Change Failure Rate** is co-owned. Failures caused by platform regressions (e.g., an SDK rev that broke tenants) are platform-owned; failures caused by tenant-side logic errors are tenant-owned. The incident records (chapter 03) carry the attribution.
- **MTTR** is similar. Incidents with a platform root cause are platform-MTTR; incidents with a tenant root cause are tenant-MTTR.

Reporting the two attributions together shows the platform the parts it can influence and the parts it cannot.

### Adaptation 3 — agent-specific nuances

Three ways agent platforms distort the DORA frame:

1. **"Deployment" is ambiguous.** An agent rev can ship via container image deploy (the CD substrate's view), via prompt-template-only update (sometimes a hot-reload with no container change), or via tool-registry change (an MCP server's capability set shifts without any tenant deploy). DORA's Deployment Frequency is cleanest if it counts *agent revisions reaching production*, defined as a new `AgentRevision` manifest (mod-412 chapter 01) observed in the deploy event stream. Prompt-template-only changes still produce a new manifest; tool-registry changes do not, and should be counted on their own panel.
2. **"Restore" is ambiguous when the failure is a slow regression.** An agent that is now producing lower-quality output after a new prompt template is not "down" in the classical sense; the SRE SLIs catch it (mod-410 chapter 02) but the rollback decision is slower and more deliberate. MTTR is still measurable — from the SLI burn alert to the rollback — but the "restore" moment is not as crisp as a classical web-service restoration.
3. **Change Failure Rate should include eval regressions.** The paved-road CI's eval gate (mod-412 chapter 05) catches many regressions before they ship; a change that *would* have been a failure but was blocked at CI is not a DORA change-failure. A change that shipped and then caused the eval SLI to burn in production *is*.

### Reading DORA alone is insufficient

High Deployment Frequency with low Change Failure Rate and short MTTR looks great. But if the platform's engineers are each on-call every other week and churning at 30 % annually, DORA will not tell you. SPACE does.

## SPACE for an agent platform

SPACE (Forsgren, Storey, Maddila, Zimmermann, Houck, Butler, 2021) is an orthogonal frame. The five dimensions are deliberately overlapping to prevent gaming any one axis.

| Dimension | What it captures | How the platform measures it |
|---|---|---|
| **Satisfaction** | Developer well-being and happiness | Quarterly short survey, five questions, same each quarter |
| **Performance** | Outcomes of the system the developer produced | DORA-adjacent plus the eval and safety SLIs (mod-407, mod-411) |
| **Activity** | Counts of actions — PRs, deploys, commits | Available from Git, CI, deploy pipeline; secondary signal |
| **Communication** | Collaboration and knowledge-sharing | Office-hours attendance, review turnaround, documentation freshness |
| **Efficiency** | Flow state and interruption load | Measured via the survey; cross-check with on-call pager counts |

SPACE deliberately has no single number. It is an *instrument*, not a KPI.

### The quarterly survey

The platform runs a short quarterly survey of its *users* — the stream-aligned engineers building on the paved road. Five questions, same every quarter, split by tenant and workload class. Aligned to SPACE and to the [DevEx framework (Noda, Forsgren, Alexander, Storey, 2023)](https://queue.acm.org/detail.cfm?id=3595878):

1. On a 1–5 scale, how satisfied are you with the paved-road agent SDK?
2. On a 1–5 scale, how fast does the paved-road feedback loop (CI, eval, deploy) let you ship confidently?
3. On a 1–5 scale, how heavy is the cognitive load of adopting the paved road for a new agent?
4. On a 1–5 scale, how well do you understand what the platform owns vs. what you own?
5. Free-text: what single change to the paved road would make your next quarter easier?

The survey is anonymous; the aggregated result is split by tenant and workload class; the free-text responses are clustered and cross-referenced against the office-hours topic log (mod-412 chapter 07).

A trap: do not add questions over time. The same five questions, quarter over quarter, is what lets you read trend. Adding questions is how surveys degrade into vanity instruments. If you need to measure something new, run a separate instrument.

### Satisfaction as the lead indicator

Of the five SPACE dimensions, Satisfaction is the one most people underweight. A platform with declining Satisfaction but stable DORA and Adoption numbers is on a slow fuse: the engineers you are relying on to keep adopting the platform are losing patience, and the next quarter's Adoption numbers will reflect it. Rising Satisfaction with flat numbers means the platform has reached a threshold that the structural metrics have not yet caught up with — expect Adoption to rise next quarter.

### Activity is the easiest to mis-read

Activity is the easiest SPACE dimension to measure and the least informative on its own. A platform engineer who files 40 PRs in a quarter may be prolific or thrashing. Activity is only meaningful alongside Performance and Satisfaction.

A specific anti-pattern: *using PR count as a performance signal*. The SPACE paper is explicit that activity counts are not performance; the platform-leadership scorecard should never promote activity alone.

## Platform adoption KPIs

[`mod-412` chapter 06](../mod-412-paved-road-sdks-and-developer-experience/06-measuring-adoption-and-cncf-maturity.md) pinned four KPIs:

- **Platform Adoption Rate** — fraction of catalogued agents emitting a valid manifest in the rolling window.
- **Time-to-First-Agent** — `T1` + `T2` + `T3`, scaffold → first `make dev` → first green CI → first prod deploy.
- **Deployment Success Rate** — fraction of paved-road deploys succeeding on first attempt.
- **Rollback Count** — count of paved-road deploys rolled back, by trigger.

The formulas and reporting shapes are in mod-412 chapter 06 and are not duplicated here. The leadership-level nuance is how they *compose* with DORA and SPACE.

### The composition

| Metric | Also measures | And disagrees with |
|---|---|---|
| Deployment Frequency (DORA) | Platform Adoption Rate × tenant activity | — |
| Lead Time for Changes (DORA) | `T2` + `T3` from Time-to-First-Agent, roughly | On the SDK-authoring loop that neither DORA nor Adoption captures |
| Change Failure Rate (DORA) | Rollback Count ÷ deploys | A mis-composition — Change Failure Rate counts *failures*, Rollback Count counts *rollbacks*; failures that did not roll back are missed if you equate them |
| Mean Time to Restore (DORA) | Chapter 03 incident MTTR | Only for paved-road-correlated incidents; a non-paved-road tenant's MTTR is a different number |

The composition is where double-counting hides. The scorecard pulls the four KPIs separately and does *not* algebra them into a smaller set.

## Support-ticket MTTR

The fourth instrument is support friction — the volume and resolution time of tickets, questions, and office-hours asks. This is specific to the platform-engineer role and is where the "cognitive-load reduction" claim (Team Topologies) is paid or not paid.

### The ticket stream

Three sources flow into one ticket stream:

- **Formal tickets** — filed against the platform-team's intake queue (JIRA / Linear / ServiceNow).
- **Office-hours questions** — logged in the chapter-07 office-hours log (mod-412).
- **Incident-adjacent asks** — questions raised during incidents that are more "how do I use the platform" than "there is a fire".

Each ticket carries: source, tenant, workload class, topic (clustered — "SDK bug", "policy question", "MCP server registration", "eval fixture authoring", "SRE SLO definition"), opened_at, resolved_at, resolution (fix, docs update, close-as-answered, duplicate, unresolved).

### The four support metrics

1. **Volume by topic.** 100 tickets this quarter, clustered: 40 % SDK onboarding, 20 % MCP registration, 15 % eval fixture authoring, 15 % policy questions, 10 % other. The topic distribution is where the paved-road *gaps* live; a 40 % onboarding share is a signal that the scaffolder is not self-serve enough.
2. **Mean Time to Resolution (support MTTR).** Open-to-close time, by topic. Elite targets: under 1 day for onboarding questions, under 3 days for SDK bugs, under 1 week for feature asks.
3. **Reopened rate.** Fraction of tickets closed and reopened within 30 days. A reopened rate above ~10 % signals that resolutions are not sticking.
4. **"Deflection" rate.** Fraction of questions resolved by pointing to existing documentation vs. by one-off engineer-to-engineer assistance. Low deflection means the documentation (chapter 07 of mod-412) is not keeping pace.

### Why support MTTR is a leadership metric

Support MTTR is the operational-friction instrument. A platform with stellar DORA and Adoption but a 10-day average support-ticket resolution is a platform whose engineers are secretly building their own workarounds and who will route around the paved road the moment a better option appears. The three published frames do not catch this; support MTTR does.

### The office-hours deflection

Office-hours (chapter 07 of mod-412) is where tickets want to be *before* they are tickets. If the office-hours topic log shows the same ten topics month over month, the resolution is *documentation*, not *per-ticket responses*. The platform-team's support MTTR improves when tickets become docs updates; the relevant metric to watch is the shift in the topic distribution from "answered live" to "answered by docs".

## The scorecard

Each quarter, one page. The structure:

```
Platform Scorecard — Q4 2026 — generated 2026-11-06
Owner: agent-platform-team-lead
Signed: platform-team-lead, executive-sponsor, tenant-council-chair

DORA (tenant deploys, rolling 90d)
  Deployment Frequency        3.2 / day avg across tenants      (+0.4 vs Q3)
  Lead Time for Changes       22h 45m p50                       (-4h vs Q3)
    T_CI (platform-owned)     18m p50
    T_deploy (tenant-owned)   21h p50
  Change Failure Rate         9%                                (-2pp vs Q3)
    Platform-caused            3%
    Tenant-caused              6%
  MTTR                        48 min                            (-12m vs Q3)

SPACE (quarterly survey, N=47 respondents across 4 tenants)
  Satisfaction                4.1 / 5                           (+0.2 vs Q3)
  Performance (self-rated)    3.9 / 5                           (+0.1 vs Q3)
  Activity (PRs on paved road) 341                              (+22 vs Q3)
  Communication (free-text → clusters)
    - "docs for new tenant onboarding" (11 mentions)
    - "clearer boundary with the senior-agentic-ai-engineer track" (7 mentions)
  Efficiency (self-rated)     3.6 / 5                           (-0.1 vs Q3)

Platform adoption (mod-412 chapter 06)
  Platform Adoption Rate      72%                               (+4pp vs Q3)
  Time-to-First-Agent p50     20h 55m                           (-6h vs Q3)
  Deployment Success Rate     91%                               (+1pp vs Q3)
  Rollback Count              18                                (-5 vs Q3)

Support
  Ticket volume (quarter)     127                               (+8 vs Q3)
    Onboarding                 38% (down from 44% in Q3)
    SDK bug                    22%
    Eval / safety              18%
    Policy                     12%
    Other                      10%
  Support MTTR p50            1 day 6h                          (-10h vs Q3)
  Reopened rate (30d)         7%                                (-2pp vs Q3)
  Deflection to docs          34% of office-hours asks         (+6pp vs Q3)

Incidents (chapter 03)
  Sev1 (quarter)              2                                 (-1 vs Q3)
  Sev2 (quarter)              7                                 (flat)
  Avg Sev1 MTTR               2h 10m
  Action items from Q3        14 closed, 2 open past due

Narrative (one paragraph)
  DORA and Adoption KPIs continue their steady improvement from Q3; the Scalable
  (Level 3) posture on Interfaces and Adoption is holding. Satisfaction up, but
  Efficiency ticked down — one T2 tenant reported interruption load from the
  rolling deprecation of the anthropic-v1 adapter; chapter 07's migration
  playbook needs a tighter scope for T2 tenants. Support volume up slightly;
  the downward shift in Onboarding share (from 44% to 38%) is the pay-off of
  the Q3 scaffolder investment — the docs are starting to replace live help.
  Two Sev1 incidents both mapped to the MCP registry memory-leak class;
  regression fixtures from both landed in mod-407 before the scorecard was
  signed.
```

One page is the entire point. A scorecard that needs ten pages is not a scorecard; it is a presentation. The narrative paragraph is where the four instruments are *read together*; the numbers are the evidence, the paragraph is the argument.

## Vanity-metric failure modes

Five patterns.

### DORA-only reporting

**Symptom:** The scorecard shows elite DORA numbers. Adoption and Satisfaction are missing.

**Cause:** DORA is the easiest to measure (CI and deploy streams are already instrumented). The others require survey discipline and dashboard work.

**Fix:** The scorecard template above is the minimum; DORA alone is not a scorecard.

### Satisfaction-average-hides-skew

**Symptom:** Satisfaction averages 4.0 / 5 — looks great.

**Cause:** Three of the four tenants are at 4.3; the fourth is at 3.1 and is the one planning to migrate off.

**Fix:** The survey results are split by tenant. The average is a top-of-funnel number; the per-tenant split is where decisions live.

### Adoption-rate-without-exceptions

**Symptom:** Adoption Rate is 85 %, looks excellent.

**Cause:** 25 % of that 85 % is "on paved road with exception" — per-tenant exceptions multiplied to the point where the paved road no longer represents a single shape.

**Fix:** The split from mod-412 chapter 06 — fully-on-paved-road / on-paved-road-with-exception / invisible — is non-negotiable on the scorecard.

### MTTR-mean-hides-tail

**Symptom:** Support MTTR p50 is 1 day. Looks healthy.

**Cause:** p99 is 45 days; a small number of tickets languish forever while the easy ones close fast.

**Fix:** Report p50 *and* p95 *and* the open-older-than-30-days count. Any one of them alone misreads.

### DORA-measuring-the-platform-team

**Symptom:** The DORA panel shows elite numbers. The platform team's own deploys are hourly.

**Cause:** The platform team is measuring itself, not the tenants. Looks good; means nothing for the platform's stated purpose.

**Fix:** The primary DORA panel is *tenant deploys*. A secondary panel for platform-team deploys is fine as a sanity check; promoting it to primary is how DORA becomes a vanity metric.

## The read-together discipline

The scorecard's power is in the *conjunction*. Patterns to practice reading:

| Conjunction | Reading |
|---|---|
| DORA elite + Satisfaction rising + Adoption flat | The engaged tenants are happy; new tenants are not onboarding. Chapter 07 enablement signal. |
| DORA elite + Satisfaction flat + Adoption flat | The platform is at steady state; growth has stalled. Roadmap signal — Next-column entries should be pull-focused. |
| DORA elite + Satisfaction falling + Support MTTR rising | The engineers are drowning in friction that DORA cannot see. Red flag; the platform team may be under-staffed. |
| DORA declining + Change Failure Rate rising + Rollback Count rising | Something structural shifted — a model-provider degrade, an SDK rev with regressions, a safety-plane change. Investigate the cause; this is often the lead signal of an incoming Sev1. |
| Adoption rising + Satisfaction rising + Support MTTR falling | The paved road is working; build-vs-buy decisions should lean "build" on strategic components (chapter 02). |
| Adoption rising + Satisfaction falling | Adoption-by-mandate. The headline is good; the pull signal is weak. Chapter 07 enablement cadence must engage before the numbers reverse. |
| Platform-caused Change Failure Rate rising | A platform regression. The regression-fixture pipeline (mod-410 chapter 06) should already be catching it in the paved-road CI; investigate why it did not. |

The scorecard itself does not draw the reading; the IC-equivalent for metrics (the platform-team lead) does, every quarter. The patterns above are the starter set.

## Instrumentation discipline

Three rules that keep the scorecard honest.

1. **Formulas are pinned in version control.** `platform/scorecard/metrics.yaml` is the source of truth; the dashboard reads from it; CI validates changes. Mod-412 chapter 06 built this; this chapter adds the DORA, SPACE, and support rows to the same file.
2. **Historical panels never rewrite.** A metric definition change is additive — the old formula's panel stays visible for a quarter so the next review can compare apples to apples.
3. **Dashboards and scorecards are two artefacts.** The dashboard is real-time, high-dimensional, explorable. The scorecard is a snapshot, one-page, signed. Never confuse the two. The dashboard is for the platform team; the scorecard is for the executive sponsor and the tenant council.

## When metrics conflict with lived experience

Sometimes the numbers say "elite" and the engineers say "burnout". Sometimes the numbers say "stalled" and the office-hours conversations say "we are about to break through". The rule: *the lived experience is never ignored, even when the numbers disagree.*

- Rising lived-experience-reports + flat numbers → the numbers lag; expect them to move in the next quarter. Investigate pre-emptively.
- Flat lived-experience-reports + rising numbers → the numbers may be measuring something the engineers do not actually value. Re-check the operational relevance of the metric.
- Falling lived-experience-reports + rising numbers → vanity-metric alarm. One of the five failure modes above almost always applies.

The quarterly tenant-council review (chapter 01) is the venue for reconciling these conflicts. The scorecard surfaces the numbers; the council surfaces the lived experience; the roadmap entry *resolves* the conflict.

## Summary

- Four instruments: DORA (delivery), SPACE (human), platform-adoption KPIs (paved-road), support-ticket MTTR (operational friction). No one of them is a sufficient measure; all four together are the leadership-level scorecard.
- DORA is measured on *tenant deploys*, not on the platform team's own deploys; the metric is split by platform-caused vs. tenant-caused when it attributes to a specific owner.
- SPACE is measured via a stable five-question quarterly survey of platform users; the same five questions quarter after quarter are what let you read trend.
- Platform-adoption KPIs are the four from [`mod-412` chapter 06](../mod-412-paved-road-sdks-and-developer-experience/06-measuring-adoption-and-cncf-maturity.md); they split by fully-on-paved-road / on-paved-road-with-exception / invisible.
- Support-ticket MTTR is a domain-specific instrument the platform-engineer owns; it captures the operational friction the other three do not; the topic distribution is where paved-road gaps live.
- The scorecard is one page, signed by three parties, summarising all four instruments plus a narrative paragraph. Dashboards are for the platform team; scorecards are for governance.
- Five vanity-metric failure modes to watch: DORA-only reporting, Satisfaction-average-hides-skew, Adoption-rate-without-exceptions, MTTR-mean-hides-tail, DORA-measuring-the-platform-team.
- The read-together discipline is where decisions live; the conjunctions are what the quarterly review discusses.

The next chapter closes the module by routing where the platform-engineer *stops*: the handoffs to `ai-infra-architect`, `agentic-systems-architect`, and engineering management for cross-platform strategy, roadmap governance, and executive sponsorship.
