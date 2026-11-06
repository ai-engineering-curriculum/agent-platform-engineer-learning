# exercise-02: Error-Budget Policy for Platform Tenants

**Estimated effort:** 3 hours · **Output:** a paved-road error-budget-policy document, a CI-gate implementation of at least one mechanical action, a scripted budget-burn simulation driving the gate, a tabletop exception-register review capturing three worked requests, a 1-page "policy change log" explaining your defaults, and a short README (~1 page) · **Pairs with:** chapter [03](../03-error-budget-policy-as-paved-road-contract.md); feeds exercise 03 and project 403.

## Objective

Turn chapter 03's policy template into a concrete, enforced contract. You will author the document, wire at least one of its mechanical actions into a CI pipeline so it actually fires, simulate a burn against the SLO from exercise 01 so the gate responds, run a tabletop review of three exception requests, and produce the change-log that would accompany a real v1 → v4 iteration.

Chapter 03 is where the words live; this exercise is where the policy becomes a thing teams cannot avoid. A policy that lives on a wiki page and depends on social enforcement is a policy teams learn to work around. A policy wired into the merge queue is a policy they design against.

## Prerequisites

- You have read chapters [01](../01-the-sre-plane-and-what-it-owns.md), [02](../02-slis-and-slos-for-agent-workloads.md), and especially [03](../03-error-budget-policy-as-paved-road-contract.md).
- You have completed [exercise 01](exercise-01-sli-and-slo-catalog-for-agent-workloads.md) and have a working `writer-v1` SLO document + burn-rate alerts.
- You have a Git-hosted CI platform of choice ([GitHub Actions](https://docs.github.com/en/actions), [GitLab CI](https://docs.gitlab.com/ee/ci/), [Jenkins](https://www.jenkins.io/doc/), [CircleCI](https://circleci.com/docs/)) with permission to add workflows and required checks to one repo. A local Git repo with Git hooks is acceptable if CI access is not available — document the choice.
- You have a merge-queue-style mechanism available: either [GitHub merge queues](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/incorporating-changes-from-a-pull-request/merging-a-pull-request-with-a-merge-queue), [GitLab merge trains](https://docs.gitlab.com/ee/ci/pipelines/merge_trains.html), or a hand-rolled required-check gate.

## Problem statement

You own the paved-road error-budget policy for the fictional multi-tenant Agent Platform. Two workload teams — `writer-agents` (from exercise 01) and `researcher-agents` — are the first adopters. The platform leadership plane has asked you to produce:

1. A document (v4) they can sign and the two teams can opt into.
2. A mechanical enforcement surface wired into CI so the "feature-freeze on budget-exhausted" action fires when the condition is met, without anyone needing to notice.
3. A simulated budget-burn demonstrating the gate working — a PR that would have been allowed on healthy budget is blocked when the budget is exhausted.
4. A tabletop exception-register review walking through three worked exception requests, ending with one approved, one approved-with-conditions, and one denied.
5. A change log showing the policy's evolution from v1 to v4 — what you tuned, why, and what evidence you had.

## Requirements

### Part 1 — The policy document

`policy/error-budget-policy-v4.yaml` following chapter 03's template. Must include:

- `name`, `version: "4"`, `effective_date`, `supersedes`, `scope`.
- The five threshold bands (`budget_healthy`, `budget_under_watch`, `budget_constrained`, `budget_exhausted`, `burning_fast`), each with explicit `condition` and `actions` arrays.
- `opt_in.requires` enumerating the paved-road prerequisites.
- `exception_register` pointing to a file path and declaring review cadence.
- `review.cadence: quarterly` with the reviewers list.
- A dated `effective_date` of your choice (use the current calendar date for the v4 pin).

The actions in each band must map to concrete artefacts on your platform:

- `feature_freeze_on_non_reliability_work` → the CI check in Part 2.
- `auto_rollback_last_deploy_eligible` → describe the hook (you do not have to implement rollback for this exercise; document how it would integrate).
- `page_tier1_oncall` → the alert label from exercise 01's burn-rate alerts.
- `weekly_review_mandatory` → a cadence item that is documented, not enforced.

### Part 2 — The CI-gate implementation

Wire one of the policy's mechanical actions into CI. The minimum is `feature_freeze_on_non_reliability_work`:

- A workflow (YAML file for the chosen CI platform) that runs on every PR to the `writer-v1` repo.
- The workflow calls a small service (or a static-file lookup — acceptable for this exercise) that returns the current budget band for the workload.
- If the band is `budget_constrained` or `budget_exhausted`, the workflow fails the check *unless* the PR has the `reliability-override-approved` label.
- The failure message explains which band the workload is in, which policy version is in effect, and how to request the override.

The budget-band source is `GET /slo/writer-v1/state` from a tiny stub service (a 50-LOC Python/Flask or Go HTTP handler returning a hard-coded band per request parameter). For this exercise, you can also read the band from a committed YAML file that your burn simulation modifies — document whichever shape you pick.

The workflow must:

- Run on every PR, not just main-branch pushes.
- Expose the band value in the check's annotations for debuggability.
- Reference the policy document by version in the check's description.

### Part 3 — The burn simulation

A scripted scenario that walks through the gate:

1. Baseline — band is `budget_healthy`. Open a PR with no reliability tag. The gate passes.
2. Switch the stub band source to `budget_constrained`. Open a second PR with no reliability tag. The gate fails. The failure message is readable.
3. Add the `reliability-override-approved` label to the second PR. Re-run the check (close/reopen, re-trigger, or whatever your CI needs). The gate passes with an override annotation.
4. Switch the stub source to `burning_fast`. The deploy workflow (a second workflow — or the same workflow extended) blocks the deploy regardless of label.

Each step produces a screenshot (CI log view, or the PR check panel). Attach them in a `burn-simulation/` folder.

### Part 4 — The exception register

`exceptions/register.yaml` with the paved-road schema. Author three worked requests. Suggested shape:

```yaml
- request_id: exc-2026-Q4-001
  workload: writer-v1
  requested_at: "2026-11-05T09:11:00Z"
  requested_by: writer-agents-lead
  justification: >
    Customer feature X is contractually due 2026-11-12 and partial delivery
    risks the deal. Currently in budget_constrained; override requested to
    merge PRs tagged feature-X.
  mitigation_plan: >
    Features gated behind a kill-switch; rollout to 1% initial cohort; burn-
    rate alerts primary on-call.
  decision: approved_with_conditions
  conditions:
    - kill_switch_required
    - 1pct_cohort_initial_rollout
    - burn_rate_alert_review_daily_until_budget_recovers
  signed_by: [sre-plane-lead, product-council]
  expires_at: "2026-11-19T00:00:00Z"
```

Three requests, covering:

- One **approved** (clear justification, low risk, short window).
- One **approved-with-conditions** (reasonable justification, risk mitigated by explicit conditions).
- One **denied** (weak justification, over-reach, or no mitigation plan).

Each request includes `request_id`, `workload`, `requested_at`, `requested_by`, `justification`, `mitigation_plan`, `decision`, (optional) `conditions`, `signed_by`, `expires_at`.

### Part 5 — The policy change log

`policy/CHANGELOG.md` with four entries, v1 → v4, each explaining:

- What changed between versions (e.g., v2 → v3: added `burning_fast` band separately from `budget_exhausted`).
- Why (the evidence: an incident, a drill result, a stakeholder request).
- Who signed off.

The log can be *fictional* for this exercise — the point is that you understand the change-control shape for a policy that teams depend on.

### Part 6 — The README

One page covering:

- What the policy says in two sentences (the elevator pitch for a team being asked to sign).
- How to run the burn simulation locally (`./scripts/simulate-burn.sh`).
- A pointer to each major artefact (policy file, exceptions, change log, CI workflow).
- One "I was surprised by" observation — e.g., "writing the CI check revealed that my stub band source had no concept of per-workload bands; the real implementation needs a key per workload, which I had glossed over in chapter 03's policy template."
- One open question for mod-413 — e.g., "the exception register's `signed_by` includes `product-council` but the policy does not specify who sits on that council in which organisational contexts; who owns the policy's reviewer list?"

## Starter guidance

- **Treat the policy as an API.** The fields in the YAML are the contract; the document body is the readable explanation. Keep both in sync.
- **Stub the band source before building the CI check.** A 10-line Flask handler returning a configurable band is enough; it decouples the policy wiring from the real SLO backend.
- **Write the failure message as if a non-SRE engineer will see it.** "Your PR is blocked because the writer-v1 SLO is in the `budget_constrained` band per policy v4. Add the `reliability-override-approved` label (requires team lead), or wait for the budget to recover (see https://dashboard...)." A cryptic failure is a policy nobody respects.
- **Make the override auditable.** The CI workflow should post a comment or annotation that the override fired, with timestamps and the labeller's identity. The exception register is the receipt.
- **Keep the burn simulation deterministic.** A simulation that works "most of the time" is a demo, not an artefact. Pin the inputs.
- **The three exception requests do not need to be realistic.** They need to exercise the register's fields and the decision shape. A deliberately-weak "denied" case is pedagogically useful.

## Acceptance criteria

Mark done when:

- [ ] `policy/error-budget-policy-v4.yaml` exists, is schema-valid against chapter 03's template, and lists the five bands with explicit actions.
- [ ] A CI workflow exists that reads the band and blocks/allows a PR accordingly on at least one of the two mock workloads.
- [ ] The `reliability-override-approved` label is honoured, and the override action is annotated in the check output.
- [ ] The burn simulation runs end-to-end and produces the four screenshots (or equivalent evidence).
- [ ] The exception register contains three worked cases, one in each decision category.
- [ ] The change log has four entries with the what / why / who shape.
- [ ] A reader can tell, from the README alone, how to run the simulation and where every artefact lives.
- [ ] No action in the policy is purely social (either enforced mechanically, documented as a cadence, or explicitly marked `informational`).

## Stretch goals

- **Wire the fast-burn page into a real alert channel.** Point the `burning_fast` band action at a Slack webhook or a test-mode PagerDuty incident. Confirm the message carries the policy version.
- **Wire the auto-rollback hook.** A deploy workflow that watches for a `budget_exhausted` band within the correlation window of a recent deploy and triggers a rollback. Use [Argo Rollouts](https://argoproj.github.io/argo-rollouts/) or a stub rollback script.
- **Build the provider-dependency-adjustment.** The SLO document from exercise 01 has a `provider_dependency_adjustment: null`. Populate it: feed a mock mod-406 provider-outage record stream into the stub band source and show that provider-attributable minutes do not burn platform budget.
- **Author a conformance test for the policy document.** A script that reads `policy/error-budget-policy-v4.yaml` and asserts every band has at least one action, every action references either a known CI check name or an `informational` tag, and the opt-in prerequisites are not empty.
- **Tier-adjusted thresholds.** Author the T1 and T3 variants of the policy. Document the thresholds that shifted and why (per chapter 03's "resilience tier and error-budget allocation" section).
- **Governance review simulation.** Run a 15-minute tabletop with teammates (or alone, with a written script) that walks through one month of exception requests. Capture the discussion outcomes as a PR comment on each exception.
