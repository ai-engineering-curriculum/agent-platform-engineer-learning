# exercise-04: Cost-Aware Concurrency and Admission Control

**Estimated effort:** 2 hours · **Output:** one admission-decision spec + one quota and tier policy + one load-shedding playbook · **Pairs with:** chapter 05 and feeds mod-406.

## Objective

Design the admission-control function and the quota / tier policy your runtime uses to hold chapter 01's obligation 3 ("my session's cost cannot exceed its budget without telling me") under realistic traffic. The artefacts you produce feed directly into mod-406 (where the model gateway enforces the same policy on the spend side) and mod-410 (SRE SLOs on admission latency and shed-rate).

## Prerequisites

- You have read chapter [`05`](../05-state-substrates-and-worker-topologies.md).
- You have completed exercise-01 (so you have the session-state abstraction the cost ledger plugs into) and exercise-02 (so you know your substrate).
- Familiarity with per-tenant rate-limiting patterns (token bucket, leaky bucket, concurrency gauges) helps but is not required.

## Problem statement

Specify the admission-control decision, the quota / priority-tier policy, and the load-shedding playbook for a workload with multiple tenants of different spend profiles. The deliverables below are what a reviewer from `ai-infra-sre`, `agentic-systems-architect`, or a product team's `senior-agentic-ai-engineer` would read to decide whether your runtime is ready to host their traffic.

Use the same target workload you picked in exercise-02, or pick one of:

- **Workload A.** Three paying tenants (large, medium, small) + free tier. 500k sessions/day across tenants; spend ratios roughly 100:20:5:1.
- **Workload B.** Single-tenant platform with internal teams as the sub-tenants; team quotas defined by finance, not the platform.
- **Workload C.** Multi-tenant SaaS with ten similar-size tenants and spiky bursts (e.g. end-of-month accounting agents).

## Requirements

### Part 1 — The admission-decision spec

Write the pseudocode (or real code) for your `admit()` function, derived from chapter 05's sketch. The spec must cover:

- **Signature.** `admit(request, now) -> Admit | Queue | Reject`. Define what `request` carries (tenant id, priority tier, estimated cost, requested session options).
- **Counter sources.** Where each counter lives (Redis vs. Postgres vs. substrate-native vs. model-gateway telemetry) and its staleness bound.
- **Order of checks.** Explicit order: budget → per-tenant concurrency → fleet back-pressure → tier-specific caps. Argue the order — the wrong order either leaks admissions or rejects admissions that would have fit.
- **Decision semantics.** What a `Queue` returns to the caller (position, estimated wait, cancellability); what a `Reject` returns (reason, retry-after if any).
- **Observability.** The exact span / log fields every admission decision emits (tenant, tier, decision, reason, numeric values used in the comparison).
- **Degradation mode.** What `admit()` returns when its counter sources are unavailable (fail-closed vs. conservative-static-limit) and why.
- **Pure-ness.** Argue that the function is pure-ish (reads counters, does not write). Identify where the actual write happens and what commits "this admission is spoken for".

Keep the spec under one page. Dense is fine; long is a smell.

### Part 2 — The quota and tier policy

Define the policy document a product team or a tenant admin reads to understand "what am I allowed to run?". Cover:

- **Priority tiers.** Three or four tiers (e.g. `interactive`, `background`, `batch`, `free`). For each: a one-sentence definition, the kinds of sessions that belong in it, and the per-tier behaviour under back-pressure (never shed / shed last / shed first / shed immediately).
- **Per-tenant budget shape.** How a tenant's monthly / daily / per-session budget is set, who sets it (platform ops? sales? self-serve?), and how overage behaves (shed vs. page vs. overage-billed).
- **Per-tenant concurrency and QPS caps.** How defaults are chosen, how a tenant requests an increase, who approves (the loop between tenant → platform ops → finance → back to tenant).
- **Noisy-neighbour protection.** What prevents a single tenant from consuming the fleet. Concurrent-session cap per tenant, per-tenant rate on expensive tools, per-tier fleet share.
- **Burst policy.** How much headroom a tenant gets over its steady-state cap for short bursts; what the token-bucket refill looks like.
- **Governance.** Who owns the policy document, how exceptions are logged, where the audit trail lives.

### Part 3 — The load-shedding playbook

A one-page playbook for the on-call engineer to run when the fleet is in distress. Cover:

- **Signals of distress.** The dashboards and alerts that indicate shedding is needed (admission p99, substrate queue depth, model-gateway upstream throttling, per-worker CPU / memory, Postgres connection pool pressure).
- **Shed ladder.** The ordered sequence of shedding actions, each with a one-line description of blast radius and reversibility. A typical ladder:
  1. Drop `free` tier admissions.
  2. Queue `batch` tier; let the queue grow.
  3. Reject `batch` tier with retry-after.
  4. Queue `background` tier.
  5. Cap `interactive` tier concurrency per tenant.
  6. Full brown-out: reject all new admissions; let in-flight sessions run.
- **Exit criteria.** When each rung is safe to climb back down; the metric and target that must hold for N minutes before you de-escalate.
- **Communication.** Who gets notified at each rung (internal channels, status page, affected tenants).
- **Post-incident.** Artefacts to produce (shed-event log, cost-ledger impact, tenant comms). Mod-410 will reuse this.

## Starter guidance

- Resist the temptation to invent new tiers. Three or four is the right number. Every additional tier is a new conversation with finance and a new row on the on-call dashboard.
- Be explicit about which counters are *approximate* (Redis, sub-second drift acceptable) and which are *exact* (Postgres, on a slow path, reconciles). Mixing them silently is a recipe for stealing admissions from one tenant to give to another.
- A good test: imagine two tenants each 90% of their quota, both submitting bursts at the same time. Trace your spec through that scenario by hand. If one of them is starved for reasons you cannot explain, your ordering is wrong.
- The shed ladder should be *rehearseable*. Pick one rung and ask "can we drill this in staging next week?" If not, it is too abstract.
- Costs drift. Model prices change; tool prices change. Note in your policy document how a price change propagates — do live sessions get their budgets re-priced mid-flight, or does the new price apply only to the next session? Both answers can be defended; the policy has to pick.
- If your substrate or model gateway has concurrency primitives you plan to rely on (Inngest's `concurrency` and `throttle`, Temporal's rate limiters, LiteLLM Proxy's rate limits — see `resources.md`), name them and the documented semantics. Flag `<!-- needs-research: ... -->` for anything you cannot verify at this point in time.

## Acceptance criteria

You can mark this exercise done when all of these are true:

- [ ] Your admission-decision spec names counter sources, staleness bounds, and degradation behaviour explicitly.
- [ ] Your quota and tier policy has at most four tiers and a one-sentence definition for each, plus a governance owner.
- [ ] Your shed ladder has at least five ordered rungs with exit criteria.
- [ ] The two-tenants-each-at-90% thought experiment runs cleanly through your spec with a defensible outcome.
- [ ] You can explain, in two sentences, how the price-change propagation decision affects in-flight sessions and why.
- [ ] You have flagged any unverified substrate / gateway guarantee with `<!-- needs-research: ... -->`.

## Stretch goals

- **Prototype the admission function.** Implement `admit()` in a toy service (Redis + Postgres + a handful of counters) and benchmark it under load. Target: p99 admission latency < 5 ms. If you miss, say why; the number is the artefact.
- **Hook the function into the exercise-01 abstraction.** The same session-state object exposes the cost ledger the admission function reads from. Trace the data flow end-to-end.
- **Write the mod-406 handoff.** What does the model gateway (mod-406) need to receive from the admission plane to enforce budget per LLM call, and what does the gateway send back to keep the cost ledger live? One page; this directly seeds that module.
