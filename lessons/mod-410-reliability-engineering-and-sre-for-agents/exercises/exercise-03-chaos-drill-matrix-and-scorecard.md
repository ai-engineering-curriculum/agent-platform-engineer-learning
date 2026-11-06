# exercise-03: Chaos-Drill Matrix and Scorecard

**Estimated effort:** 4 hours · **Output:** a drill library with at least one fully-authored class-1 (model-provider outage) drill, a runnable fault-injection harness scoped to a local stub platform, a scorecard produced from a live run, a drill matrix (Markdown table) with four rows (one per class, three of them planned-but-not-run), a drill-safety contract document, and a short README (~1 page) · **Pairs with:** chapter [04](../04-chaos-drills-for-agent-dependencies.md); feeds exercise 04 and later modules' chaos work.

## Objective

Design and run one chaos drill end-to-end against a stub agent platform. Produce the scorecard, the matrix entry, and the safety contract. The runnable drill is the muscle memory; the matrix and safety contract are the governance artefacts.

Chapter 04 lays out four classes of agent-dependency failure. This exercise picks the one that is easiest to stand up locally (class 1, model-provider outage) and walks it through hypothesis → fault injection → detection → scorecard → fixture-candidate. You can add the other classes as stretch goals; the single-class walk-through is the required path.

## Prerequisites

- You have read chapters [01](../01-the-sre-plane-and-what-it-owns.md), [02](../02-slis-and-slos-for-agent-workloads.md), and especially [04](../04-chaos-drills-for-agent-dependencies.md).
- You have completed [exercise 01](exercise-01-sli-and-slo-catalog-for-agent-workloads.md) — the stub agent, the SLIs, and the burn-rate alerts are the detection surface this drill exercises.
- You have a fault-injection tool of choice: [Toxiproxy](https://github.com/Shopify/toxiproxy) for TCP-level faults, a mocked model-gateway endpoint that supports a `POST /admin/chaos` surface (acceptable if you build it as a 50-LOC stub), or [Istio fault injection](https://istio.io/latest/docs/tasks/traffic-management/fault-injection/) if you already have Istio running from a prior exercise.
- You have a stub model gateway. If you have not built one, a 100-LOC Python/Go HTTP handler that proxies to a local model (or just returns a canned response) and that supports a chaos endpoint is sufficient.
- You have a load generator ([k6](https://k6.io/docs/), [vegeta](https://github.com/tsenart/vegeta), [hey](https://github.com/rakyll/hey), or a 30-LOC asyncio script).

## Problem statement

Class 1 drill: model-provider outage. The target is `writer-v1` running with a mock model gateway configured to route `claude-sonnet-4-6` to a primary endpoint and fall over to a secondary.

You will:

1. Author the drill plan (hypothesis, scope, fault-injection spec, expected detection path).
2. Author the drill-safety contract (scope bounds, duration cap, abort procedure).
3. Run the drill against the stub platform with the load generator providing baseline traffic.
4. Capture the scorecard — TTD, TTM, blast radius, user-visible impact, playbook score, hypothesis result.
5. Author the drill matrix (markdown table) with class-1 populated from your run and classes 2–4 populated as planned-but-not-run with hypotheses and target dates.
6. Produce the drill-safety contract as a separate signed document.

## Requirements

### Part 1 — The drill plan

`drills/2026-Q4-model-provider-outage/plan.yaml`:

```yaml
drill:
  name: model-provider-outage-us-east
  class: model_provider_outage
  hypothesis: >
    When the primary provider for claude-sonnet-4-6 returns sustained 503s,
    the model gateway fails over to the secondary route within 60 s, TTFA p95
    degrades to ≤ 20 s during fail-over, and session-completion stays ≥ 98 %.
  planned_date: "2026-11-12"
  participants: [sre-oncall-primary, model-gateway-owner, writer-agents-lead]
  scope:
    workloads: [writer-v1]
    tenants: [internal-test]
    environment: local-stub
  duration_cap: 10m
  fault_injection:
    target: mock-model-gateway
    spec:
      route: "claude-sonnet-4-6:primary"
      mode: return_503
      fraction: 1.0
    expected_mitigation: "gateway auto-failover to secondary route"
  success_criteria:
    - name: time_to_detect
      operator: "<="
      value: 60
      unit: seconds
    - name: time_to_mitigate
      operator: "<="
      value: 120
      unit: seconds
    - name: completion_rate_during_window
      operator: ">="
      value: 0.98
```

### Part 2 — The drill-safety contract

`drills/safety-contract.md`:

- **Scope bounds.** Whose workloads, whose tenants, which environments can be affected. The invariant that cross-boundary faults abort.
- **Duration cap.** The maximum drill duration (10 minutes default); the harness auto-removes the fault at the cap.
- **Abort procedure.** The command (or button) that removes every active fault across every chaos component. A sentence on how it is tested.
- **Blast-radius auto-abort.** If measurements show the drill affecting more than the declared scope (e.g., a tenant not on the allow-list sees degradation), the harness aborts automatically and pages.
- **Communication.** Pre-drill notification to on-call. Post-drill summary.

Sign it (your name and date) at the bottom. In a real platform the contract is reviewed by the leadership plane; for this exercise your signature stands in for the review.

### Part 3 — The fault-injection harness

A runnable harness under `drills/harness/`:

- `inject.sh <drill-name>` — reads the plan YAML, applies the fault to the mock gateway (via the chaos endpoint or Toxiproxy command).
- `measure.sh <drill-name>` — runs during the drill window; samples SLI series from Prometheus (or logs) every 5 s; writes to `measurements.ndjson`.
- `abort.sh` — removes all faults; sends an abort-complete marker.
- `score.py <drill-name>` — reads `measurements.ndjson` and produces the scorecard YAML.

The scripts can be bash + Python; keep each one short and single-purpose. The important invariant is that `abort.sh` is callable *during* the drill and unconditionally stops everything.

### Part 4 — The drill run

Execute the drill end-to-end:

1. Start the baseline load (at least 10 req/min for 2 minutes before the fault; 10 minutes total).
2. At T+2:00, run `inject.sh`.
3. At T+2:30, confirm the fault is being emitted by the stub gateway (a `curl` showing the 503 response).
4. Observe the detection — the burn-rate alert from exercise 01 should fire; capture the alert timestamp as the TTD.
5. Observe the mitigation — if the stub gateway supports auto-failover, measure when the SLI recovers; this is the TTM. If the stub does not have failover built in, simulate the mitigation (manually re-route) and treat the manual action as the mitigation step, documenting the shortcut.
6. At T+10:00, `abort.sh` auto-fires; the drill ends; the fault is removed.
7. Run `score.py` to produce the scorecard.

### Part 5 — The scorecard

`drills/2026-Q4-model-provider-outage/scorecard.yaml` matching chapter 04's template:

- Timeline (T+0 to T+10 with the key events).
- Measurements (TTD, TTM, blast radius, sessions impacted, sessions breached SLO, completion-rate delta).
- Hypothesis result (`confirmed`, `partial`, `violated`).
- Playbook score (1–5; chapter 04 asks "did the playbook match what you actually did"; for this exercise, score honestly even without a formal playbook — this exposes what the playbook needs to contain).
- Findings (severity, title, owner, fix-PR placeholder).
- `fixture_candidate: true | false` with a one-line justification.
- Follow-ups.

### Part 6 — The drill matrix

`drills/matrix.md`:

| Class                       | Last drilled | Result   | Playbook rev | Fixture generated | Target next drill |
|-----------------------------|--------------|----------|--------------|-------------------|-------------------|
| model-provider outage       | <your date>  | <result> | n/a          | <yes/no>          | 2027-Q1           |
| MCP-server flake            | never        | planned  | n/a          | n/a               | 2026-Q4           |
| sandbox exhaustion          | never        | planned  | n/a          | n/a               | 2027-Q1           |
| eval-infra degradation      | never        | planned  | n/a          | n/a               | 2027-Q2           |

For each of the three planned-but-not-run classes, a 2–3 sentence paragraph below the table naming the hypothesis you would test and what the primary detection signal would be.

### Part 7 — The README

One page covering:

- How to run the drill end-to-end in one command (`./drills/run.sh model-provider-outage`).
- How to invoke `abort.sh` and the expected safety behaviour.
- A pointer to `matrix.md`, `scorecard.yaml`, and `safety-contract.md`.
- One "I was surprised by" observation — e.g., "my stub gateway returned 503s instantly rather than the real provider's `timeout-then-503` shape; the TTD was unrealistically low because the burn-rate alert fired immediately. Real-world faults are slower to surface and the SLI threshold needs to assume that."
- One open question for mod-406 or mod-413 — e.g., "my stub gateway's failover was a manual step; what is the paved-road's story for automatic failover when the primary and secondary providers charge different prices? Who signs off on the cost-impact of a drill-triggered failover that stays in effect post-drill?"

## Starter guidance

- **Keep the stub model gateway simple.** The chaos endpoint is the important part; the rest can be a `return 200` canned response. The drill is about the platform's reaction, not the model's output.
- **Fix the load generator's input distribution.** A reproducible drill needs reproducible inputs; a varying load pattern will add noise to the TTD measurement.
- **Treat the first run as a pilot.** The first time you run this, you will find missing instrumentation (the alert does not include a timestamp you expected, the scorecard script chokes on an empty window). That is the point. Fix, re-run, keep both scorecards as artefacts.
- **Document the shortcuts.** "I simulated failover by re-routing manually at T+1:45" is an honest note; a scorecard that pretends the automation exists is a worse artefact than one with a known shortcut.
- **Make the abort obvious.** The abort script's output should name every fault it removed. If the drill aborts and the user is not sure whether the production fault is gone, the contract is broken.
- **If you have time, run the class-2 (MCP flake) drill too.** It is the most common failure in a mature platform and the muscle memory is useful. Treat it as stretch.

## Acceptance criteria

Mark done when:

- [ ] `drills/2026-Q4-model-provider-outage/plan.yaml` exists with all fields populated.
- [ ] `drills/safety-contract.md` exists, is signed and dated, and names the abort procedure.
- [ ] The harness runs `./inject.sh`, `./measure.sh`, `./abort.sh`, `./score.py` in sequence; the end-to-end drill script produces the scorecard.
- [ ] `scorecard.yaml` is populated from the live run with timeline entries, measurements, hypothesis result, and findings.
- [ ] `drills/matrix.md` exists with the class-1 row populated from the real run and classes 2–4 populated as planned-but-not-run with hypotheses.
- [ ] `abort.sh` is callable *during* the drill and demonstrably stops the fault within 5 seconds.
- [ ] The README names each artefact and includes the "surprised by" + open question.

## Stretch goals

- **Run the class-2 (MCP flake) drill end-to-end.** Stub a mock MCP server; inject partial-5xx; measure whether the paved-road inter-agent plane's breaker (if you have it — otherwise, a library-level breaker) catches it. Produce a second scorecard.
- **Add the `fixture_candidate: true` promotion.** If the drill revealed a weakness, author the mod-407 fixture row that would catch it (cross-reference chapter 06 and exercise 04).
- **Build a drill scheduler.** A cron (or GitHub Actions scheduled workflow) that runs a selected drill nightly against staging. The matrix updates automatically with the most-recent-run date.
- **Add the production-scope safety check.** The harness refuses to inject faults when the environment label is `production` unless a `drill-authorised-<date>` pre-shared token is present. The token is a leadership-plane artefact (mod-413).
- **Multi-class compound drill.** Two faults in sequence — e.g., model-provider outage + MCP flake within the same window. The scorecard captures the compound shape.
- **Author the chaos-mode admin surface for the model gateway.** A proper REST endpoint (`POST /admin/chaos`, `DELETE /admin/chaos`) with authentication, scope validation, and automatic expiry. The interface becomes the paved-road artefact for future drills.
