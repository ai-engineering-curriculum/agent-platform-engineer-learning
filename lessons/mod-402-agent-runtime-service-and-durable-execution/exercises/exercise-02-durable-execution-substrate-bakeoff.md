# exercise-02: Durable Execution Substrate Bakeoff

**Estimated effort:** 3 hours · **Output:** one workload-characteristics sheet + one scored comparison matrix + one decision memo (≈ 1 page) · **Pairs with:** chapter 03.

## Objective

Pick a durable-execution substrate (Temporal, Dapr Workflows, Restate, or Inngest) for a defined agent workload and defend the choice in writing. The point is not to pick the "correct" substrate — none of the four is universally correct. The point is to make the decision *rigorously* against workload numbers and *documentably* so a reviewer (an architect, an SRE lead, a `senior-agentic-ai-engineer` from a stream-aligned team) can push back productively.

## Prerequisites

- You have read chapter [`03`](../03-durable-execution-substrate-choice.md).
- You have skimmed the primary docs for all four substrates (see `../resources.md`): [Temporal](https://docs.temporal.io/), [Dapr Workflows](https://docs.dapr.io/developing-applications/building-blocks/workflow/workflow-overview/), [Restate](https://docs.restate.dev/), [Inngest](https://www.inngest.com/docs).

## Problem statement

Pick a target workload — real or realistic — and run it through chapter 03's decision pattern. Produce three artefacts.

Choose one of these workloads, or pick your own with equivalent specificity:

- **Workload A — Consumer copilot.** 500k sessions/day, median duration 3 minutes, p99 15 minutes. Burst to 50 concurrent sessions per tenant. Low fan-out (≤ 10 tool calls per session). Rare user signals (reply, cancel).
- **Workload B — Long-running research agents.** 2k sessions/day, median duration 30 minutes, p99 6 hours (overnight runs). Heavy fan-out (≤ 500 sub-agent / tool calls per session). Occasional human-in-the-loop approvals.
- **Workload C — Batch document-processing agents.** 1M documents/day, each a short-lived session (median 20 seconds), bursty arrival. No signals; strict cost ceiling per document; heavy throughput-per-dollar.

Or your own — but *write down* the equivalent characteristics first.

## Requirements

### Part 1 — The workload-characteristics sheet (one page)

For the workload you picked, write numbers (not adjectives) for every row in this table:

| Dimension | Your workload |
|---|---|
| Daily session volume | |
| Median session duration | |
| p99 session duration | |
| Concurrent sessions, steady state | |
| Concurrent sessions, 99th-percentile burst | |
| Tool / sub-agent fan-out per session (p50, p99) | |
| External signals per session (p50, p99) | |
| Cancellation deadline target (chapter 02 obligation 2) | |
| Retention window (session, cost ledger, traces) | |
| Multi-tenant posture (shared cluster, isolated clusters, per-tenant quotas) | |
| Determinism budget on product teams (strict / pragmatic) | |
| Existing platform surface you want to be consistent with (Dapr-heavy, JVM-heavy, serverless-heavy, greenfield) | |

If you cannot pin a number, write the range you would stand behind in an architecture review and call out the uncertainty.

### Part 2 — The scored comparison matrix

For each of the four substrates (Temporal, Dapr Workflows, Restate, Inngest), score 1–5 against each characteristic from Part 1 with a one-sentence justification per cell. Use primary docs; cite specific pages or sections as you go.

Minimum shape:

| Dimension | Temporal | Dapr Workflows | Restate | Inngest |
|---|---|---|---|---|
| Fit for session duration | /5 — reason | /5 — reason | /5 — reason | /5 — reason |
| Fit for fan-out | /5 — reason | /5 — reason | /5 — reason | /5 — reason |
| Signal throughput fit | /5 — reason | /5 — reason | /5 — reason | /5 — reason |
| Cancellation primitives | /5 — reason | /5 — reason | /5 — reason | /5 — reason |
| Determinism friction | /5 — reason | /5 — reason | /5 — reason | /5 — reason |
| Deployment ergonomics | /5 — reason | /5 — reason | /5 — reason | /5 — reason |
| Platform consistency | /5 — reason | /5 — reason | /5 — reason | /5 — reason |

Do not aggregate the scores into a single number. The *reasoning* in each cell is the deliverable; the sum would be noise.

### Part 3 — The decision memo (≈ 1 page)

Write a memo structured exactly like this:

1. **Workload one-liner.** One sentence restating the workload.
2. **Choice.** The substrate you picked, in one sentence.
3. **Why this substrate.** Three bullet points citing specific rows from Part 2.
4. **Where this will hurt.** Two to four bullet points on known weaknesses of the choice against *this workload*, with mitigation plans.
5. **Escape hatch.** One paragraph on which substrate you would migrate to if the workload shape drifted 10× in any dimension, and what the migration would cost. Reference the adapter / envelope pattern from chapter 04 — this is partly why you built it.
6. **Open questions / `needs-research`.** Any substrate claim you could not verify from docs at this point in time.

## Starter guidance

- Resist "we pick Temporal because it is the safe default." That may be the right answer, but write the *justification*. If your justification is only "mature", add two more — ecosystem fit, SDK language choice, operations familiarity — or the memo will not survive pushback.
- Treat vendor blog posts as secondary. Score every cell against the primary docs linked in `resources.md`.
- The "where this will hurt" section is the most valuable part of the memo. Reviewers who have shipped production agent runtimes will skip to that section first. If it is empty, the memo reads as marketing.
- Use the three-test from mod-401 chapter 01 if you catch yourself scoring the wrong dimension — "this substrate is good at my product's business logic" is a category error.
- If you cannot find a published number for a substrate's concrete limit (history size, signal throughput, timer precision), flag it `<!-- needs-research: ... -->` and move on. The flag is the correct output, not a guess.

## Acceptance criteria

You can mark this exercise done when all of these are true:

- [ ] Every row in your workload-characteristics sheet has a concrete number or defended range — no "many", "high", "fast".
- [ ] Every cell in your scored comparison matrix has a reason sentence that cites a primary doc section, not a general impression.
- [ ] Your decision memo has all six sections, including the "where this will hurt" section.
- [ ] Your memo names the migration path to a second substrate explicitly, and roughly estimates the cost in engineer-weeks or platform-planes-affected.
- [ ] A reviewer who disagreed with your choice could push back on *specific* cells of your matrix rather than vibes.

## Stretch goals

- **Prototype a tiny agent on your chosen substrate.** One LLM call + one tool call + one signal + one timer. Measure cancellation propagation latency against chapter 02's obligation 2. Attach the numbers to the memo — this is what later work in mod-410 will build on.
- **Repeat the whole exercise** for a *second* workload (pick a different one from the three above) and compare where the pick diverges. The point is that no substrate wins everywhere, and seeing it in two workloads makes the lesson stick.
- **Peer-review swap.** Ask a peer on another track (`senior-agentic-ai-engineer`, `agentic-systems-architect`, `ai-infra-sre`) to read your memo. Note every question they ask that was not answered by the memo; fold the answers in.
