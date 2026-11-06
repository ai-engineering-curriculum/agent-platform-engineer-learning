# exercise-03: Checkpoint and Resume Under Partial Failure

**Estimated effort:** 3 hours · **Output:** one failure-mode catalogue + one resume-correctness test plan + one runbook draft · **Pairs with:** chapter 02 and feeds mod-410.

## Objective

Produce the failure-mode catalogue and resume-correctness test plan that proves your runtime holds chapter 01's obligation 1 — "if my session committed a tool call, it will not run twice" — under realistic partial failures. This exercise is where paper design meets the ugly middle of the state machine. The artefacts you produce feed directly into mod-410 (SRE runbooks) and mod-407 (regression gates).

## Prerequisites

- You have read chapter [`02`](../02-session-lifecycle-checkpoint-resume-cancellation.md).
- You have completed exercise-01 (so you have a session-state contract to test against).
- You have picked a substrate (exercise-02) or can at least name the one you will prototype against.
- Familiarity with chaos / fault-injection tooling is helpful but not required (you can describe the fault injection in prose).

## Problem statement

Enumerate the partial failures that violate chapter 01's three obligations, write a test plan that proves your design handles each one, and sketch the operator runbook for when a test regresses in production.

## Requirements

### Part 1 — The failure-mode catalogue

Produce a table that enumerates at minimum these failure classes, each with a concrete scenario, the obligation it threatens, the chapter-02 mechanism that defends against it, and a short description of the observable damage if the defence fails.

| # | Failure class | Concrete scenario | Obligation threatened | Defence mechanism | Observable damage if defence fails |
|---|---|---|---|---|---|
| 1 | Worker crash mid-LLM-call | Worker dies after the LLM request is in flight but before the result is durable | At-most-once effective | Side-effect wrapper + idempotency on provider side | Double-billed tokens; duplicate agent decisions |
| 2 | Worker crash mid-tool-call | Tool invocation starts; worker dies before result is durable | At-most-once effective | Side-effect wrapper + tool-gateway idempotency key (mod-403) | Double-executed tool; data corruption outside the platform |
| 3 | Worker crash after tool call, before next checkpoint | Tool result returned but not yet committed to session | At-most-once effective | Durable event on activity completion | Replay re-issues tool call |
| 4 | Signal delivered during worker crash | External signal arrives mid-crash | Signal replay-safety | Signal durable write before consumption | Lost signal; session hangs on `WAITING_SIGNAL` forever |
| 5 | Timer fires during fleet restart | Durable timer fires when no worker is up | Bounded delivery | Timer survives in storage; resumed on worker restart | Session misses deadline; stuck in `WAITING_TIMER` |
| 6 | Cancellation during tool call | User cancels while tool is running | Bounded cancellation | Cancellation token propagated to tool gateway | Tool spend continues after hang-up; cost leaks |
| 7 | Config drift between snapshot and resume | Tool manifest advances while session is suspended | Config-drift safety | Versioned manifests + typed `ToolManifestIncompatible` error | Resume silently calls a different tool shape |
| 8 | Framework-adapter upgrade in flight | `adapter_version` bumps while sessions are checkpointed | Adapter versioning | Envelope decoder picks correct `adapter_version` | Resume decodes wrong fields; corrupt state |
| 9 | Clock skew on worker | Worker's wall-clock disagrees with substrate clock | Timer determinism | Substrate-provided virtual time | Timers fire early or late; billing lag |
| 10 | Postgres failover mid-checkpoint | Checkpoint write's transaction fails partway | Durable progress | Transactional write + retry; idempotent commit token | Lost step; session resumes from stale state |

Add any failure class *your* workload or substrate introduces (e.g. hosted-runtime timeouts if you chose Vertex AI Agent Engine or Foundry Agent Service in chapter 04). Aim for 12–15 rows total.

### Part 2 — The resume-correctness test plan

For each row in the catalogue, write a test specification with:

- **Setup.** The starting state of the session (e.g. "RUNNING, one LLM call completed, tool call issued, result not yet durable").
- **Injection.** The fault you inject (e.g. "SIGKILL the worker holding the session"), in enough detail that a chaos-tool user could run it.
- **Expected post-conditions.** What the session state, cost ledger, message log, and side-effect log look like after the system has stabilised.
- **Measurement.** How you observe the post-conditions (SQL query, trace span assertion, external-side check).
- **Pass / fail criterion.** Specific — "cost ledger shows exactly one LLM call charge, not zero or two".

At minimum, the test plan should produce:

- A list of test IDs mapped 1-to-1 to the catalogue.
- A one-paragraph description of the *harness* that runs them (unit, integration, chaos tier). mod-407's eval platform is a plausible home for the integration and chaos tiers; name that in your plan.
- A statement of which tests are blocking on every PR, which run nightly, and which run only on substrate / framework version bumps.

### Part 3 — The runbook draft

Pick two of the most operationally scary failure classes from the catalogue (your call — typical picks are the tool-call-replay class and the leaked-cancellation class) and sketch a one-page operator runbook for each:

- **Alert.** What alert fires? What dashboard shows it?
- **First-five-minutes actions.** Three to five concrete, safe steps the on-call engineer runs to contain blast radius (e.g. "pause admission for tenant T", "mark affected sessions read-only").
- **Investigation queries.** The SQL or trace-query the on-call engineer runs to characterise impact.
- **Remediation.** The path back to a healthy state (replay queue, manual compensation, cost-ledger adjustment).
- **Post-incident artefacts.** What the runbook requires the on-call engineer to produce (incident doc, eval-regression fixture, dashboard change). mod-410 will build on this.

## Starter guidance

- The catalogue is where you should spend the most time; the test plan and runbook are dense but mostly follow once the catalogue is honest.
- A cheap sanity check: for every row in the catalogue, ask "if this happens at 03:00, does the system heal on its own?" If the answer is "no" for more than two rows, revisit the design before writing the test plan. The test plan documents a *working* design; it does not replace one.
- Fault-injection ideas worth trying if you prototype: `pkill -9` on the workflow worker; `iptables` block on the Postgres port; slow / drop packets to the tool gateway; clock-skew the worker with `date -s`; terminate the pod via Kubernetes API with grace=0.
- Do not conflate "tool reported success" with "tool side effect happened". A sent-email tool that logs success but did not actually send is a different bug from one that sent the email twice. Your test plan should capture both halves.
- For any claim you cannot verify (e.g. "the substrate guarantees timer firing within N seconds of wall-clock"), flag `<!-- needs-research: ... -->` and continue. The flag is the correct output.

## Acceptance criteria

You can mark this exercise done when all of these are true:

- [ ] Your catalogue has at least 10 failure rows, each with concrete scenario + defence + damage statement.
- [ ] Every catalogue row maps to a named test in the test plan with specific pass / fail criteria.
- [ ] Your test plan assigns each test to a tier (PR-blocking / nightly / release-gate) and names the harness.
- [ ] Your two runbooks each have alert, first-five-minutes, investigation queries, remediation, and post-incident sections.
- [ ] A new on-call engineer could *follow* the runbook to a safe state without further coaching.
- [ ] You have flagged any unverified substrate guarantee with `<!-- needs-research: ... -->`.

## Stretch goals

- **Actually run the fault injections.** Pick two rows; prototype them against your substrate (even trivially); file the measured numbers into the memo. Numbers beat opinions in SRE reviews.
- **Write the eval-platform fixture** (mod-407) that would detect regression on the checkpoint / resume contract. This is the handoff to `ai-eval-engineer`'s scope — if there is no eval engineer, the fixture is still yours.
- **Draft the SLO definitions** you would propose in mod-410: resume success rate, cancellation propagation p99, leaked-cost ratio. One-liners each, with your current measured baseline. These become mod-410's starting point.
