# The Evaluation Plane and What It Owns

The runtime (mod-402) owns the agent loop. The MCP gateway (mod-403) owns the tools. The sandbox (mod-404) runs tool code. The policy plane (mod-405) authorises what the agent does. The model gateway (mod-406) owns the call out to a model. Every one of those planes shipped a decision today — a new prompt, a new tool, a new router entry, a new quota, a new policy. Which of those decisions makes the fleet better, which makes it worse, and which is a silent regression nobody will notice until it hurts a customer? That is the slice this module owns.

The naive answer is "each team runs its own eval notebook before merge". A small-team, single-agent, no-fixture, no-trace-store, no-gate shape is the default at most start-ups. It fails the moment you have a second agent, a second team, a second model provider rolling under you, a nightly regression someone needs to block, or a compliance reviewer who asks "when did this change, and against what fixture did it pass?" Every platform with more than one agent on it ends up building a shared evaluation platform. The only question is how much of it you own.

This chapter fixes the slice. The chapters that follow pick the pieces: the harness template library over Inspect and Promptfoo (02), the dataset management plane with versioning, provenance, synthetic-data labelling, and access control (03), the trace / replay store that captures every agent interaction (04), eval-gated CI/CD against SWE-bench / τ-bench / GAIA / AgentBench and per-team custom fixtures (05), and the handoffs out to `ai-eval-engineer` and `model-evaluation-engineer` (06).

## What the evaluation plane owns

The plane sits between the agent runtime (mod-402) that produces interactions, the model gateway (mod-406) that produces billing / routing / token facts on every call, the MCP gateway (mod-403) that logs tool-call outcomes, the policy plane (mod-405) that logs deny / permit decisions, the safety plane (mod-411) that logs guardrail decisions, the observability plane (mod-408) that collects OpenTelemetry traces, and — on the far side — the stream-aligned teams that author agents and the researchers who author new benchmarks. The plane owns:

- **The harness template library.** A library of task-shaped templates on top of [Inspect AI](https://inspect.aisi.org.uk/) and [Promptfoo](https://www.promptfoo.dev/docs/), each wired to the platform's identity, logging, dataset access, and judge-model egress rails, so a team can onboard a new eval in hours. Chapter 02.
- **The eval dataset management plane.** A versioned, provenance-tracked, access-controlled store for every dataset an eval reads — public benchmarks, replayed production traces, synthetic generations (labelled as such), and golden sets. Chapter 03.
- **The trace / replay store.** A content-addressable log of every agent interaction (session → turn → model call → tool call → policy decision) that is deep enough for a reviewer to replay a single turn against a new model, a new prompt, or a new tool-schema and produce a comparable score. Chapter 04.
- **The eval-gated CI/CD hooks.** Deployment pipelines block a merge if the candidate change regresses a standard fixture (SWE-bench / τ-bench / GAIA / AgentBench) or a per-team custom fixture by more than a published threshold, with statistical evidence of regression rather than a single-run swing. Chapter 05.
- **The judge-model routing and spend cap.** Judge calls are themselves model calls routed through mod-406, with a dedicated budget bucket, a frozen prompt version, and a replayable log so the judge is auditable. Chapter 02 details; the methodology sits with `ai-eval-engineer`.
- **The leaderboard and regression dashboard.** A single surface that answers "is the writer-v1 agent better this week than last" for every stream-aligned team, pulling from the harness library, the trace store, and the CI gates. Chapter 05.
- **The eval SDK and paved-road onboarding.** A small library and `create-eval` scaffolder that produces a correctly-wired eval, a dataset reference, a judge contract, and a CI gate in one command. Cross-reference mod-412.

The plane does **not** own:

- **Eval methodology itself.** What makes a good judge rubric, how to calibrate a pairwise judge, how to design a RAG eval that measures retrieval versus generation, how to run online eval on live traffic — these are `ai-eval-engineer` (level 30). This plane templatises their methodology so it scales; it does not author it.
- **Benchmark design and statistical methodology.** How to construct a benchmark, how to control for leakage, how to design confidence intervals and power analyses for small-N evals — these are `model-evaluation-engineer` (level 30). This plane runs their benchmarks correctly; it does not design them.
- **The agent loop under test.** The runtime (mod-402) owns the loop; the eval plane calls into it as a black box.
- **The model under test.** The model gateway (mod-406) owns the call; the eval plane requests a routing identifier and gets back a trace.
- **Harm model authoring.** The policy plane (mod-405) ingests the harm model from `ai-risk-engineer`; the eval plane consumes the plane's decision as a column on a trace.
- **The online / production trace back-feed as an SRE discipline.** Mod-410 owns the SRE operating cadence that feeds regression fixtures from incidents.

Drift off this slice and you reinvent eval methodology — usually worse than `ai-eval-engineer` would — or you reinvent benchmark construction — usually worse than `model-evaluation-engineer` would. Scope policing is a feature of this module.

## Why a platform and not per-team notebooks

A few concrete questions that are hard to answer without a shared platform and trivial with one. If your current eval setup can answer each one in a single command, you have a platform. If any one requires running a notebook, you do not yet.

- **Cross-agent regression after a provider roll.** Anthropic rolls the production pointer for `claude-sonnet-4-6` to a new weights revision. Can every team's eval re-run against the new weights overnight, with pass / fail per fixture, without a single human intervention?
- **Judge-model drift.** The judge used in writer-v1 scoring was `gpt-4o-2024-08-06`. It is now pointing at `gpt-4o-2024-11-20`. Can the platform tell you how many tests changed verdict on the swap, from a log — not from each team's chat window?
- **Trace replay.** A production incident at 2026-11-03T14:22Z touched one agent session. Can a reviewer replay that session, step by step, against a candidate fix — same inputs, same tool-call shape, same timestamps — and score the fix on the exact regression that happened?
- **Dataset provenance.** An auditor asks "what is in the writer-v1 fixture, where did each row come from, and which rows are synthetic?" Can you answer from a manifest, or do you have to grep notebooks?
- **Gate-on-regression.** A pull request to the planner-v1 prompt drops τ-bench pass rate by 4 points at p < 0.05. Does CI refuse to merge, or does the drop land in production and get surfaced by an on-call?

These are the capabilities the plane delivers, and they are what separate "we ran evals" from "we have an evaluation platform". Chapters 02–05 build each.

## The two deployment shapes

Two deployment shapes dominate; pick deliberately, do not drift.

### In-repo scaffold (SDK)

The platform ships as a template repository and a thin SDK. Each team's agent repo pulls the template, authors evals in their own CI, uses the platform's judge-routing and dataset-fetch SDK calls, and publishes traces to a shared store. No standalone service runs evals; the team's existing CI (GitHub Actions, Buildkite, GitLab CI) does.

- **Strengths.** Zero added service to operate. Easy to adopt incrementally. Teams own their eval velocity.
- **Weaknesses.** Shared regression fixtures (SWE-bench / τ-bench) are N-way duplicated; each team re-downloads, re-sets-up, re-runs. Judge-cost coordination is hard. Flaky-eval triage is a hallway conversation. Cross-team leaderboards require log stitching.
- **Where it fits.** Early-stage platforms, one or two agents, teams with strong eval culture already.

### Standalone platform (service)

The platform ships as a service — an eval scheduler, a dataset catalogue, a judge router, a trace/replay store, and a leaderboard — fronted by an API and a UI. Teams register evals as declarations; the platform schedules, runs, judges, stores, and reports.

- **Strengths.** Standard fixtures run once, cached, shared. Judge-model spend is a budgeted pool with a dashboard. Trace store is one place, searchable across teams. Leaderboard is first-class. Compliance can read one audit log.
- **Weaknesses.** The platform is now a tier-1 dependency. Rolling a judge-prompt change becomes a release. Teams must speak the platform's eval-declaration schema.
- **Where it fits.** Multi-agent, multi-tenant platforms past the prototype stage. This is the shape mod-407 writes against.

A **hybrid** shape — a thin in-repo SDK over a standalone service — is where most mature platforms end up. The SDK handles ergonomics and local dev; the service handles the coordinated state (datasets, judges, traces, gates). Each role is clear; neither does the other's work.

## The threat model this plane defends against

Four classes of failure dominate what the eval plane sees. They are what chapters 02–05 build mitigations for.

### Silent regression on a shared dependency

A provider rolls a model pointer. A prompt template gets edited. A tool schema widens. Any one of these can shift behaviour across every agent on the fleet. Without a scheduled, judgement-stable eval against a frozen fixture, the regression is detected by a customer, not by a test.

The plane's defences:

- **Pinned fixtures with content hashes.** Every eval references a dataset version by hash; a roll is a commit, not a rename. Chapter 03.
- **Scheduled nightly runs independent of merge pressure.** The eval scheduler runs a known set every night against the current pointers, not just on a PR. Chapter 05.
- **Judge prompts and judge models pinned by hash.** Changing the judge is a reviewed release, not a config edit. Chapter 02.

### Dataset leakage and contamination

A benchmark whose test set has entered the model's training corpus is no longer a benchmark — it is a memorisation test. A dataset mutated in place loses its ability to detect yesterday's regression tomorrow. An unlabelled synthetic row in a human-labelled fixture biases every score it touches.

The plane's defences:

- **Immutable content-addressed dataset objects.** A row cannot be mutated; only added as a new version. Chapter 03.
- **Synthetic-data labelling as a dataset-level invariant.** Every row carries a `provenance` field; synthetic rows are visible by schema, not by convention. Chapter 03.
- **Contamination checks at ingest.** Datasets record publication date and train-cutoff relevance; the catalogue surfaces when a benchmark postdates the model it is being used to evaluate. Chapter 03.

### Flaky evals and stochastic noise

A judge that scores 73% on one run and 69% on the next run is not a judge; it is a coin flip with a bias. A CI gate wired to single-run thresholds will either block too often (if the threshold is tight) or block too late (if the threshold is loose). The real question is whether the candidate is statistically worse, not whether one run crossed a line.

The plane's defences:

- **N-run default with bootstrap confidence intervals.** The gate is `change in pass rate significant at p < 0.05`, not `pass rate > 0.80`. Chapter 05.
- **Seed and temperature pinning per fixture.** Where the methodology admits it, the plane pins seeds and temperatures to kill avoidable noise. Chapter 02.
- **Judge calibration cards.** A per-judge per-rubric agreement rate against a held-out human-labelled set, re-computed on judge changes. Methodology sits with `ai-eval-engineer`; the plane is what *runs* the calibration card.

### Judge-cost runaways

A judge that reads full agent traces at Opus rates costs real money per evaluated row. A fixture of 2,000 rows judged on every PR becomes a line item. Without a budget enforcer, the first signal is the invoice.

The plane's defences:

- **Judge-router with a dedicated budget bucket.** Judge calls route through mod-406 with a per-fixture burst budget. Chapter 02.
- **Trace-slice contracts.** The judge sees only the slice it needs — the final answer, the final `tool_use` sequence, a reference answer — not the entire trace. Chapter 04.
- **Cached judgements on unchanged rows.** If the candidate agent output and the reference are unchanged, the judgement is read from cache. Chapter 04.

## The eval record

Pin the shape before writing the plane. Every eval run — one row, one fixture, one agent, one model — produces an `EvalRecord`. The judge's score lives *inside* the record; the record itself is invariant across Inspect, Promptfoo, and any custom harness the plane adopts later.

```json
{
  "eval_run_id": "er_01HV8X3M9F5A6R2B4C0D7E8G9H",
  "eval_suite_id": "writer-v1@2026-11-06",
  "fixture": {
    "name": "writer-v1-ragas",
    "version_hash": "sha256:82c1...",
    "row_id": "row_00178",
    "row_provenance": "human-labelled"
  },
  "under_test": {
    "agent_id": "writer-v1",
    "agent_revision": "git:abcd1234",
    "model_identifier": "platform:writer-v1",
    "resolved_provider": "anthropic",
    "resolved_model": "claude-sonnet-4-6",
    "tool_schema_hash": "sha256:1aa2...",
    "prompt_hash": "sha256:9ce4..."
  },
  "trace": {
    "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
    "trace_store_uri": "s3://traces/4bf9.../session.jsonl",
    "turn_count": 7,
    "tool_calls": 3
  },
  "scoring": {
    "judge_model": "platform:critic-v1",
    "judge_prompt_hash": "sha256:7b1c...",
    "judge_response_id": "msg_01A2b3C4d5E6",
    "metrics": {
      "task_success": 1,
      "faithfulness": 0.82,
      "helpfulness": 4
    }
  },
  "cost": {
    "under_test_usd": 0.0164,
    "judge_usd": 0.0021,
    "total_usd": 0.0185,
    "price_table_version": "2026-11-01"
  },
  "verdict": "pass",
  "gate_decision": {
    "gate": "writer-v1-nightly",
    "baseline_run_id": "er_01HV1...",
    "delta_pass_rate": 0.0,
    "delta_significant_at_p05": false
  }
}
```

Five invariants the record enforces:

1. **Everything referenced is hash-pinned.** Fixture, row, agent revision, tool schema, prompt, judge prompt. A reviewer can re-fetch the exact inputs that produced this score.
2. **The judge is itself a routed model call.** No ambient judge; the judge route, the cost, and the response id are on the record. Judge drift becomes a reviewable event.
3. **Trace reference is a pointer, not an inline blob.** The record stays small enough for a leaderboard query; the trace is pulled on demand from the trace store (chapter 04).
4. **Provenance is first class on every row.** `human-labelled`, `synthetic:<generator>`, `replayed:<prod-trace-id>` — every row says what it is.
5. **The gate decision is part of the record.** Not derived later; the decision about whether this run blocks a deploy is persisted alongside the score.

Chapters 02–05 build the pieces that produce this record.

## Where the plane physically sits

A typical shape for the physical deployment:

- **Eval scheduler:** A workflow service (typically on the same durable-execution substrate as mod-402) that schedules fixtures against candidate revisions, nightly against production pointers, and on-demand from CI.
- **Harness runners:** Stateless workers that execute Inspect / Promptfoo tasks. Jobs are queued; runners scale to the queue. SPIFFE-authenticated outbound to the model gateway (mod-406) and the MCP gateway (mod-403).
- **Dataset catalogue:** A metadata store (typically Postgres) over an object-store (S3 / GCS / R2) for the raw rows, with a convention like [DVC](https://dvc.org/doc) or [LakeFS](https://docs.lakefs.io/) for content-addressed versioning. Chapter 03.
- **Trace / replay store:** An append-only store for interactions — typically object storage for the long tail with a hot index (ClickHouse / BigQuery / Iceberg) for search. Chapter 04.
- **Leaderboard and dashboard:** A read-only service over the EvalRecord stream, aggregating per-agent, per-fixture, per-model scores over time.
- **Judge router:** A thin layer in front of mod-406 that enforces the judge budget, pins the judge prompt, and writes a judge-audit entry alongside each score.
- **CI integration surface:** A webhook / action that CI calls to request an eval run and receive a gate decision; the actual gate is enforced by the merge-queue system, not by the eval plane. Chapter 05.

If you cannot draw this topology on a whiteboard and name the box every decision flows through, chapter 02 is the thing to read next.

## The three obligations

Pin the plane to three non-negotiable obligations. Everything in chapters 02–06 is a decision about how to deliver them.

1. **"Every eval run is reproducible from a hash-pinned record. Fixture, row, agent revision, prompt, tool schema, judge prompt, judge model."** If a reviewer cannot reproduce a score a week later from the record alone, the record is defective. Reproducibility is the compliance contract and the regression-investigation contract.
2. **"Every production deploy passes a published set of regression gates or is marked an override with an attached reason."** The gate is the contract; overrides are allowed but logged. A platform that is advisory — "the eval says this regresses, do what you want" — is not a gate.
3. **"Dataset provenance and synthetic-data labelling are schema-level, not conventional."** A row that is synthetic but not labelled is a defect of the dataset system, not of the eval. A dataset whose train-cutoff relevance is not recorded is a defect of the catalogue, not of the methodology.

A plane that cannot defend all three is an evaluation platform in name only.

## Summary

- The evaluation plane owns the harness template library, the dataset management plane, the trace / replay store, the eval-gated CI hooks, the judge router, the leaderboard, and the paved-road onboarding. It does not own eval methodology, benchmark design, the agent loop, the model under test, or the harm model.
- Two deployment shapes exist — in-repo scaffold and standalone platform; a hybrid of the two is where most mature platforms end up. The plane in this module is written against the standalone shape.
- The failures the plane defends against are silent regression on a shared dependency, dataset leakage and contamination, flaky-eval noise, and judge-cost runaways. Chapters 02–05 build the mitigations.
- Every eval run produces an `EvalRecord`: hash-pinned fixture, hash-pinned agent, hash-pinned judge, trace pointer, cost split, verdict, and gate decision. Every chapter adds a piece to this record.
- Three non-negotiable obligations: reproducible-from-record; gated-or-override-logged; provenance-and-synthetic-labelling as schema, not convention.

The next chapter opens the first piece — the harness template library over Inspect AI and Promptfoo that produces most of the `under_test` and `scoring` fields on the record.
