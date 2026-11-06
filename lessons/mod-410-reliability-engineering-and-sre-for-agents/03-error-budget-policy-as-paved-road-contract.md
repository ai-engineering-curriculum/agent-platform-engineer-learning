# Error-Budget Policy as a Paved-Road Contract

Chapter 02 built the SLIs and SLOs. This chapter builds the thing that turns them from measurements into operating decisions: the error-budget policy. The source is the [Google SRE Workbook chapter on implementing SLOs](https://sre.google/workbook/implementing-slos/) and [chapter 5, Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/), plus the "error budget" idea from the [SRE Book's chapter on service-level objectives](https://sre.google/sre-book/service-level-objectives/). The specialisation is agents on a multi-team platform: the policy has to work for teams the SRE plane does not directly manage, which means it has to be an *opt-in contract*, not a directive.

A platform that measures SLOs without an error-budget policy ends up with pretty dashboards and the same ship-whenever-you-want behaviour it had before. The policy is the thing that makes the SLO expensive to miss — expensive in a way that aligns incentives back to reliability work.

## Why "policy" and not "a Slack message from the SRE team"

The platform engineering team is not the authority on every stream-aligned team's roadmap. Even if it were, tribal enforcement scales to about five teams and then collapses. The alternative — a signed, dated, versioned policy document the team adopted when they went on the paved road — scales to thirty teams and does not require the SRE on-call to argue priorities at 11 pm.

The policy is a Team Topologies–style X-as-a-service contract (see [Skelton & Pais, *Team Topologies*](https://teamtopologies.com/)). The paved-road platform offers the SLO measurement, the burn-rate alerting, and the automatic actions as a service; the stream-aligned team signs up, in writing, in exchange for the operating guarantees the platform makes back. The policy is the specific slice of that contract that governs what happens when the budget burns.

## The policy document (paved-road template)

A single policy — versioned at the platform level — governs every workload on the paved road. Per-workload overrides are allowed but reviewed. The document is two to four pages, in plain English; here is the structural skeleton.

```yaml
# error-budget-policy@v4.yaml
name: paved-road-error-budget-policy
version: "4"
effective_date: "2026-09-01"
supersedes: paved-road-error-budget-policy@v3
scope: >
  Applies to every agent workload with a signed SLO document under slos/.
  Per-workload overrides are enumerated in slos/<name>.yaml.exceptions and
  require SRE + product sign-off.

thresholds:
  - name: budget_healthy
    condition: budget_remaining > 0.50
    actions: [none]
  - name: budget_under_watch
    condition: budget_remaining <= 0.50 AND budget_remaining > 0.25
    actions:
      - weekly_review_mandatory
      - reliability_tag_on_next_sprint
  - name: budget_constrained
    condition: budget_remaining <= 0.25 AND budget_remaining > 0.10
    actions:
      - feature_freeze_on_non_reliability_work
      - exec_review_at_30d
  - name: budget_exhausted
    condition: budget_remaining <= 0.10 OR fast_burn_alert_active
    actions:
      - reliability_work_only
      - auto_rollback_last_deploy_eligible
      - page_tier1_oncall
      - incident_on_next_fast_burn
  - name: burning_fast
    condition: burn_rate_1h >= 14.4 AND burn_rate_5m >= 14.4
    actions:
      - page_primary_oncall
      - auto_freeze_deploys_for_workload

opt_in:
  requires:
    - signed_slo_document
    - adopted_paved_road_sdk_version: ">=1.8"
    - conformance_tests_green_last_7d
  team_acknowledgement: >
    By opting into the paved road, the owning team agrees to the actions
    above when the corresponding condition fires. Deviations are logged in
    the exception register (mod-413).

exception_register:
  location: platform-leadership/error-budget-exceptions.yaml
  requires: [product_lead_signoff, sre_signoff]
  review_cadence: monthly

review:
  cadence: quarterly
  reviewers: [sre-plane-lead, platform-leadership, product-council]
  minimum_data_retention: 90d
```

Every clause is doing work. Walk them.

## The five threshold bands

The bands come from the SRE Workbook's "multi-window multi-burn-rate" model, translated into operational steps. Each band is defined by a *condition over the SLO's state*, and each triggers a *mechanical action* the platform executes.

### Budget healthy (`> 50 %`)

The normal state. The team ships what it wants, when it wants. No reliability-triggered actions are in effect. The SRE plane's job in this band is to stay out of the way.

### Budget under watch (`≤ 50 %, > 25 %`)

Signal: the SLO is on track to miss unless something changes. Actions:

- **Weekly review becomes mandatory.** The owning team adds a five-minute "SLO status" slot to its weekly standup. The weekly review is a conversation, not a meeting with the SRE team; the team reviews with its own leads.
- **The next sprint plans for reliability work.** At least one ticket on next sprint is tagged `reliability` with explicit reference to the burn. The paved-road's automated sprint-tagging hook (if the organisation uses one) reads the SLO state and files the tag.

This is *not* a freeze. It is an attention signal. The error-budget policy's job in this band is to make the team notice early, when the fix is cheap.

### Budget constrained (`≤ 25 %, > 10 %`)

Signal: the SLO is likely to miss this quarter. Actions:

- **Feature-freeze on non-reliability work.** New feature commits to the workload's repo require an SRE or team-lead override label (`reliability-override-approved`). Reliability work (fixes to the burning SLI, test additions, performance tuning) is unaffected.
- **Exec review at 30 days.** If the team has not recovered budget in 30 days, the next quarterly review surfaces the workload to the product council. This is not a punishment; it is a "we need resource help or we need to re-tier this workload" conversation.

The freeze is mechanical — a CI gate on commits that lack the override label — not social. The point is that feature engineers reliably know they are frozen (because their PR is red) rather than guessing from a Slack message they missed.

### Budget exhausted (`≤ 10 % OR fast-burn active`)

Signal: we are out of budget. Actions:

- **Reliability-work-only.** Every new PR to the workload requires the `reliability-override-approved` label *or* is tagged `reliability`. All other work is blocked at the CI gate.
- **Automatic rollback of the last deploy, if the last deploy correlated with the budget drop.** The paved-road SDK's deploy-tracker (mod-412) tags each deploy; the SRE plane has a correlation window (e.g., 1 hour post-deploy) in which a budget-burn fast-alert triggers an automatic rollback. The rollback is reversible; the team can roll forward with explicit override.
- **Primary on-call is paged.** The on-call starts an incident regardless of whether the burn is also a traditional outage.
- **The next fast-burn event creates an incident record** in the incident-management system automatically.

These are not punishments. They are operational actions the team pre-agreed to when opting into the paved road. A mechanical, pre-agreed action removes the "whose decision is this?" meeting at 2 am.

### Burning fast (`burn_rate_1h ≥ 14.4 AND burn_rate_5m ≥ 14.4`)

Signal: something is actively on fire. Actions:

- **Primary on-call paged.** First line of defence.
- **Auto-freeze deploys for the workload.** The CI gate blocks every merge to the workload's deploy branch until the burn subsides, independent of the budget-remaining state. The point is to stop making the problem worse while people investigate.
- **The playbook (chapter 05) is attached to the page.** The on-call sees "SLO fast-burn on writer-v1 TTFA" with a link to the TTFA-regression playbook.

The 14.4× multiplier is the SRE Workbook's standard for a 1-hour page. The paired 5-minute check filters out single-blip noise; both must be true.

## The actions, as a CI contract

A policy that reads nicely but cannot be enforced is decorative. The actions on the budget bands are CI-level checks and merge-queue states:

- **The merge queue reads the SLO state.** Every push to `main` on a workload repo checks the SLO state before accepting the commit. The state source is a platform-local API (`GET /slo/<workload>/state`) backed by the Prometheus / Alertmanager pipeline.
- **The `reliability-override-approved` label.** Applied by a team lead (not the PR author). The label is auditable; the label-application event lands in the exception register.
- **The automatic rollback.** A Kubernetes operator or a deployment controller (whichever the organisation uses) watches the fast-burn alert and the deploy-tracker event stream. When a fast-burn fires within the correlation window of a deploy, it triggers a rollback workflow (e.g., [Argo Rollouts](https://argoproj.github.io/argo-rollouts/) `promote --rollback`, or the equivalent in [Flagger](https://flagger.app/)).
- **The deploy freeze.** The deploy pipeline reads the SLO state and refuses to deploy if the state is `budget_exhausted` or `burning_fast`. The refusal is *overridable* (with the `reliability-override-approved` label on the deploy PR), but the override is logged to the exception register.

The failure mode a policy prevents is the one where these actions are *suggested* and someone decides case-by-case. Case-by-case is where incentives drift; mechanical enforcement is where they align.

## Resilience tier and error-budget allocation

Not every budget band is the same size. The three resilience tiers from mod-409 chapter 05 — T1 (tight), T2 (standard), T3 (long-running) — translate into error-budget allocation:

- **T1 workloads** have *smaller* budgets (tighter SLOs → less room). Burn-rate alerts fire sooner. Policy actions are more aggressive (feature freeze at 25 %, not 10 %).
- **T2 workloads** have *standard* budgets. The paved-road policy above is calibrated to T2.
- **T3 workloads** have *larger* budgets (looser SLOs). The thresholds shift: the `under watch` band might sit at `≤ 70 %, > 40 %`; the `constrained` band at `≤ 40 %, > 20 %`.

The important invariant: a team that *tightens* its tier takes on a smaller budget and tighter policy actions. A team that *loosens* loosens both. The tier is one knob that moves the whole operating contract.

## The provider-budget-split problem

Agent workloads depend on external providers (model APIs, hosted search, hosted vector stores) that have their own availability. If the provider is 99.5 % available and the platform's SLO is 99 %, the platform has essentially *no* internal budget — the provider's bad days consume all of it.

The paved-road policy handles this explicitly:

- **Per-workload SLO documents may declare a `provider_dependency_adjustment`.** The adjustment is the known, measured availability of the critical provider over the SLO window. The SLI series is divided into *platform-attributable* and *provider-attributable* badness, and the budget is split proportionally.
- **A provider outage does not burn platform budget** if the mod-406 model-gateway's outage-detection event fires. The gateway's fail-over to a secondary provider is the primary mitigation; the budget split is the honest reporting. Cross-reference mod-406.
- **The combined-SLO is published to tenants** so they see the real reliability they can expect, not a number the platform cannot deliver on.

This is not an excuse mechanism — the budget split is *measured*, from the mod-406 gateway's provider-outage records, not asserted by the team. A workload that over-states its provider dependency has a reconciliation at the quarterly review.

## Opt-in and the exception register

The policy's scope is "workloads with a signed SLO document". Teams that have not signed one are not on the paved road and the policy does not govern them (they get no error-budget protection either — the plane does not run alerts for them, and their SLA to tenants is between them and their tenants).

Teams that *have* signed are bound by the actions above. Exceptions — "we need to ship this feature during an exhausted-budget window" — are not refused; they are recorded.

- **Request path.** PR with `reliability-override-approved` label. Team lead signs off. The label, the PR, the deploy, and the burn state at the time are written to the exception register.
- **Review path.** The exception register is reviewed monthly by the leadership plane (mod-413). A team with three exceptions in a month gets a conversation; a team with eight gets an intervention.
- **Audit path.** The register is retained for 90 days minimum (longer in regulated contexts) and is an input to the quarterly reliability review.

The register exists because policies without exception mechanisms either get violated silently or make the policy impossible to live with. Explicit exceptions are better than quiet workarounds.

## When the policy itself is wrong

A policy that fires every day is wrong; a policy that never fires is also wrong. The quarterly review asks:

- **Did the policy fire?** Zero fires in a quarter suggests SLOs are too loose or the workload is over-provisioned. The data point is useful; it is not a crisis.
- **Did the policy's actions help?** The post-incident template (chapter 05) includes a "did the budget-band action correlate with recovery?" question. If the feature freeze did not change the trajectory, the freeze may be mis-calibrated — or the burn may have been driven by something the freeze cannot address (upstream provider, data distribution shift).
- **Were the overrides warranted?** Review the exception register. Teams with routine overrides may need a tier downgrade, a product-level conversation, or a tighter workload boundary.
- **Is the burn-rate multiplier right?** The SRE Workbook publishes the default multipliers; a platform whose fleet has a different latency distribution may tune them. Any tuning is logged and versioned in the policy document.

The policy is a versioned object. Changes go through change-control; the policy pointer in each SLO document is a version pin. Cross-reference mod-413 on the governance cadence.

## Common anti-patterns

- **Policy as a wiki page.** A policy nobody can cite the current version of is a policy nobody is bound by. Version it, pin it in the SLO document, and put the version number in the alert's Slack message.
- **Policy without actions.** "The team should consider feature freeze" is a suggestion. "The CI gate is red" is an action. Pick actions.
- **Policy with punitive framing.** The policy is a service the team opted into for its own protection; the framing is "here is what we agreed we would do in this state", not "here is what we are forcing you to do". Teams that experience the policy as punishment find ways to not sign up. Make the trade-off — "you get the paved road's observability, resilience, and incident support" — visible.
- **Policy tuned for the past.** The thresholds in the template above assume a 28-day window and a T2 SLO. A workload with a tighter SLO needs tighter thresholds; the paved-road template gives teams a starting point, not a one-size.
- **Provider-adjustment as a fig leaf.** A team that attributes all its badness to the provider has probably stopped looking. The adjustment comes from mod-406's measured records; the SRE plane audits the ratio.
- **Exception register without review.** Writing exceptions to a file nobody reads is the same as having no mechanism. Monthly review, by the leadership plane, is the pairing. Chapter mod-413.

## Handoff to leadership (mod-413)

The error-budget policy is the SRE plane's artefact. The *exception register* and the *policy governance cadence* are the leadership plane's. The handshake:

- **SRE plane publishes.** The policy, versioned. SLO attainment reports monthly. Budget-band history per workload.
- **Leadership plane reviews.** The exception register monthly. The policy itself quarterly. Approval of major policy changes before they land.
- **Both planes operate the cadence together.** Monthly review is a 30-minute meeting; quarterly is a half-day. The cadence is in mod-413 chapter on operating rhythms.

The policy without leadership backing turns into "SRE's policy nobody follows". Leadership without SRE's data turns into opinion. The joint cadence makes it a contract.

## Summary

- An error-budget policy is the operating contract that converts SLO attainment into actions. Without it, SLOs are decorative; with it, reliability is on the roadmap.
- The paved-road policy defines five threshold bands — healthy, under watch, constrained, exhausted, burning fast — each with pre-agreed mechanical actions executed by the CI and deployment pipelines.
- Teams opt in by signing an SLO document and adopting the paved-road SDK. Opt-in is explicit; the plane does not impose the policy on teams that have not signed.
- Resilience tier (T1 / T2 / T3) shapes the budget size and action thresholds. Provider dependency is handled with a measured budget split against mod-406's outage records, not an unverifiable assertion.
- The exception register captures overrides, review is monthly (leadership plane), and the policy itself is versioned and reviewed quarterly.
- Anti-patterns are the ones every policy framework falls into: wiki-page policies, suggestions instead of actions, punitive framing, frozen thresholds, unreviewed exceptions. Avoid each explicitly.

The next chapter turns the policy outward — from "what do we do when the budget burns" to "which failures do we practise before they happen", the chaos-drill programme against the four agent-dependency failure classes.
