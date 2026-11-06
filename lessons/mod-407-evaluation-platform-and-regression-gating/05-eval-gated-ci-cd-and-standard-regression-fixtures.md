# Eval-Gated CI/CD and the Standard Regression Fixtures

An eval that does not block a deploy is advisory. Advisory evals rot — the team on-call during a regression reads the dashboard and concludes "the eval is noisy, ignore it", which it is, because nobody has pressure to fix it. An eval that blocks a deploy is a contract. Contracts generate the pressure that keeps them useful.

This chapter is about writing those contracts. Gates read from the catalogue (chapter 03), run through the template library (chapter 02), replay through the trace store (chapter 04), and either refuse a merge or publish a reviewed override. Four standard fixtures — SWE-bench, τ-bench, GAIA, AgentBench — are run as platform-wide baselines; per-team custom fixtures are run on top of the baselines.

## The four gate kinds

Not every gate blocks the same thing. Pin four shapes and map every fixture to one.

### 1. Per-PR regression gate

Runs on every pull request against the branch the PR targets. Blocks the merge if the candidate regresses a fixture with statistical confidence.

- **What triggers it.** A PR touching a watched path — the agent's prompt, the agent's tool schema, the gateway routing table, the judge prompt, a policy bundle.
- **Fixture shape.** Fast enough to fit a PR latency budget. 20–200 rows typically; N=3 runs per row; bootstrap CI on pass rate; threshold is `delta_pass_rate_ci_upper < -0.02`.
- **Cost envelope.** Per-PR, so costs multiply by PR count. Budget typically $1–$5 per PR. The judge-cache (chapter 02) is the single biggest cost lever here.
- **Human override shape.** Allowed, logged, posted as a review comment with the overriding engineer's identity and the reason.

### 2. Pre-merge full-sweep gate

Runs after per-PR gates pass but before merge; longer, more exhaustive. Typically batched on the merge-queue rather than blocking authoring loops.

- **What triggers it.** A PR landing in the merge queue.
- **Fixture shape.** 500–2,000 rows; N=10 runs; broader coverage than per-PR; typically touches one or more standard fixtures (SWE-bench slice, τ-bench slice).
- **Cost envelope.** $10–$100 per sweep. Judge-cache still helps; under-test cache does not (the merge is the thing that changes under-test behaviour).
- **Human override shape.** Allowed, with a 24-hour cool-down before auto-retry, and a required link to a tracking issue.

### 3. Nightly leaderboard gate

Runs every night against production pointers, independent of PRs. Not a merge gate in the tactical sense; its failure pages an on-call and surfaces on the leaderboard.

- **What triggers it.** Scheduler cron.
- **Fixture shape.** Everything the platform considers release-indicative: all standard fixtures, all per-team release-blocking fixtures. 10,000+ rows total; N≥30; tight CIs.
- **Cost envelope.** Significant. Shared across teams; the standard-fixture run is the platform's cost, not the team's.
- **Human override shape.** Not an override — a failure is an incident. On-call opens a ticket, triages, and either rolls back a provider pointer or flags a true regression.

### 4. Release-promotion gate

Runs when a team cuts a release — promoting a candidate agent revision from `staging` to `production`.

- **What triggers it.** A manual promotion via the deploy service.
- **Fixture shape.** The team's release-blocking fixtures at full fidelity. The standard fixtures are not re-run (the nightly caught those); the per-team custom fixtures run at nightly fidelity.
- **Cost envelope.** Per-release, so cost is release-cadence-bounded. Typically $10–$50.
- **Human override shape.** Allowed only with a two-person review — the authoring engineer and a peer — and a required rollback plan.

A platform typically runs all four. A platform that runs only (1) sees regressions land in production; a platform that runs only (3) lets a bad PR into the merge queue first. Separate the kinds; wire the fixtures to the right kind.

## The four standard regression fixtures

The plane runs four standard fixtures across every agent on the fleet. Running them platform-wide, rather than per-team, is where the plane's leverage against provider rolls comes from: every agent sees the same gate against the same frozen fixture when a provider's pointer moves, so a regression is attributable to the provider, not to a single team's code.

### SWE-bench

**[SWE-bench](https://www.swebench.com/)** — introduced by Jimenez et al. (2024) — measures a language model's ability to resolve real GitHub issues by producing a patch that passes the repository's test suite. Variants include the full benchmark, [SWE-bench Lite](https://www.swebench.com/lite.html) (reduced subset for faster iteration), and [SWE-bench Verified](https://openai.com/index/introducing-swe-bench-verified/) (human-filtered for solvability). Inspect AI ships an implementation in `inspect-evals`.

The plane's job:

- Pin the dataset version by hash. SWE-bench rows have a `base_commit`; the catalogue manifests which commit set is in scope.
- Run each row in an isolated sandbox (mod-404) so patches cannot escape.
- Attribute cost to the under-test model, the judge-of-the-tests (deterministic, since SWE-bench tests are code), and the sandbox substrate.
- Surface per-repo breakouts so a regression on `django` is distinguishable from a regression on `sympy`.
- Flag the train-cutoff relevance (chapter 03) — SWE-bench rows predate many provider training cutoffs; a specific `publication_date >= model.train_cutoff` subset is the leakage-safe slice.

Gate wire-up: per-PR on a Lite slice; pre-merge on a broader slice; nightly on the full Verified set.

### τ-bench

**[τ-bench](https://github.com/sierra-research/tau-bench)** — introduced by Yao, Shunyu et al. at Sierra — measures an agent's ability to complete customer-interaction tasks correctly while following a policy. Rows are structured conversations with a simulated user; the agent uses tools, the simulated user may change their mind mid-conversation, and the final state is scored against a reference final state (database state, in the current implementation).

The plane's job:

- Pin the user-simulator model identifier the same way the judge is pinned (chapter 02). A τ-bench run where the user-simulator moved is not comparable to a prior run.
- Isolate tool calls against the τ-bench reference environment (airline, retail).
- Score via the included final-state comparator; no model-graded judge needed. Cheaper than SWE-bench.
- Surface the policy-violation rate separately from the task-completion rate; both are blocking thresholds.

Gate wire-up: pre-merge on both environments; nightly on both at full fidelity.

### GAIA

**[GAIA](https://huggingface.co/datasets/gaia-benchmark/GAIA)** — "General AI Assistant", Mialon et al. — measures general assistant capability with real-world questions that require multi-step reasoning, web browsing, multi-modal retrieval, and tool use. Three difficulty tiers (Level 1–3). Final answers are scored by exact-match against a reference.

The plane's job:

- Pin the dataset revision (GAIA is published on Hugging Face; the catalogue carries the revision hash).
- Provide the tool suite — web browser, file reader, code interpreter — through the MCP gateway (mod-403) and sandbox (mod-404). GAIA's own repo gives a reference tool suite; the plane's job is to provide an equivalent through its own tools so the measurement is of the agent, not of a bolted-on browser.
- Score with the published exact-match grader; retain the full trace for failure-mode analysis.
- Surface per-level results; a Level-3 regression is a stronger signal than a Level-1 regression.

Gate wire-up: nightly on Level 1 and Level 2; weekly on Level 3 (Level 3 is expensive and high-variance).

### AgentBench

**[AgentBench](https://github.com/THUDM/AgentBench)** — introduced by Liu, Xiao et al. — measures agent capability across eight distinct environments (operating system, database, knowledge graph, digital card game, lateral thinking puzzles, house-holding, web shopping, web browsing). The breadth is the value.

The plane's job:

- Pin the dataset and the environments; several of the eight require their own test harnesses the plane runs under the sandbox (mod-404).
- Surface per-environment scores; a web-shopping regression is not a database regression.
- Rate-limit the environments requiring external calls (web browsing); mirror the sites into the sandbox substrate where licensing permits.
- Flag environment version drift; AgentBench's environments have evolved across releases, and the catalogue manifest pins which.

Gate wire-up: pre-merge on a representative subset; nightly on the full environment set.

A note on all four: benchmark revisions and leaderboards drift. Pricing of the models they were originally scored against drifts. Specific version pins, score baselines, and leakage states that are correct today will be stale next quarter. Flag `<!-- needs-research: ... -->` on every baseline score you cite from a specific revision.

## Per-team custom fixtures

Standard fixtures measure general capability; they do not measure "does writer-v1 still do the specific thing our customers ask for?" That is what per-team custom fixtures are for.

The pattern the plane encourages:

- **A smoke fixture** — 20–50 rows, runs on every PR, fast, deterministic scorers where possible, no judge calls unless unavoidable. Catches obvious regressions.
- **A quality fixture** — 200–500 rows, runs pre-merge and nightly, model-graded with a stable judge rubric. Catches the softer regressions.
- **A regression fixture** — grows over time from incidents. Every time the team ships a fix, the triggering case becomes a row in the regression fixture, with a provenance of `replayed:<incident-session-id>` (chapter 04 → chapter 03). The fixture is append-only; rows do not get retired.

The plane does not author per-team fixtures. The plane provides the catalogue that stores them, the templates that run them, and the gate infrastructure that enforces them. `ai-eval-engineer` writes them.

## The statistical frame — why one run is not enough

A single run of a probabilistic system is one sample. The gate is a hypothesis test, not a threshold cross.

The default the plane ships:

- **N runs per row, aggregated per fixture.** N=3 for per-PR; N=10 for pre-merge; N≥30 for nightly and release-promotion.
- **Bootstrap confidence interval on pass rate.** Resampling 1,000 bootstraps over the N runs produces a `pass_rate_ci_low` and `pass_rate_ci_high`. The gate reads the CI, not the point.
- **Comparison against a baseline.** Every fixture has a baseline — the last known-good run on the current production pointer. The gate's decision is `delta_pass_rate_ci_upper < -epsilon` for a declared epsilon (typically 0.02).
- **Flaky-row suppression with evidence.** A row that fails in 1 of 10 runs is a 10% flake rate; the gate records it but does not treat it as a regression unless the N-run aggregate crosses the CI threshold.
- **Pairwise rather than absolute, where methodology permits.** For judge-scored rubrics, a pairwise judge ("is candidate A better than baseline B") is lower-variance than an absolute Likert score. The template library supports both; `ai-eval-engineer` chooses per-fixture.

An engineer who has not seen their CI gate reject a flake they could not reproduce has not yet shipped under the real one. Document the flake rate as part of the fixture's calibration card so a reviewer can distinguish "the gate is wrong" from "the gate is noisy".

## The gate decision shape

The gate outputs a single decision per run, persisted to the `EvalRecord` stream (chapter 01) and posted to the triggering CI.

```json
{
  "gate_run_id": "gate_01HV8...",
  "gate": "writer-v1-nightly",
  "trigger": {
    "kind": "scheduled_nightly",
    "scheduled_at": "2026-11-06T00:00:00Z"
  },
  "under_test": {
    "agent_revision": "git:abcd1234",
    "resolved_model": "claude-sonnet-4-6",
    "prompt_hash": "sha256:9ce4..."
  },
  "fixtures": [
    {
      "fixture": "swe-bench-verified@2026-10-01",
      "n_runs": 30,
      "baseline_run_id": "gate_01HV1...",
      "baseline_pass_rate": 0.432,
      "candidate_pass_rate": 0.412,
      "candidate_pass_rate_ci": [0.401, 0.423],
      "delta_pass_rate_ci_upper": -0.009,
      "verdict": "regress"
    },
    {
      "fixture": "writer-v1-regression@2026-11-02",
      "n_runs": 10,
      "baseline_pass_rate": 1.0,
      "candidate_pass_rate": 1.0,
      "candidate_pass_rate_ci": [0.95, 1.0],
      "verdict": "pass"
    }
  ],
  "overall_verdict": "regress",
  "overall_action": "block_merge",
  "override": null
}
```

Three invariants:

1. **A gate decision names the baseline.** The reader can replay the baseline from the trace store on the record alone.
2. **A regress verdict carries CI, not just a point.** `-0.009` is the upper bound of the CI; a reader who disagrees with the decision has the data to argue against it.
3. **An override is explicit and persisted.** If an engineer overrides a `regress` verdict, the override, the engineer's identity, and the reason land on the record. "I overrode because the flake rate on this row is 15%, tracked in INCIDENT-402" is the shape.

## Override logging — the escape valve

Gates must be overrideable. A platform where no one can override has a platform problem the moment the gate is wrong. A platform where overrides are untracked has a compliance problem the moment the gate is right.

The pattern:

- **Overrides are signed.** The overriding engineer's identity is on the record. A bot cannot override.
- **Overrides require a reason.** Free-text, logged, surfaced on the leaderboard with the gate record.
- **Overrides have a half-life.** A fixture that is overridden more than twice in a week surfaces a `fixture.flaky_or_wrong` alert to `ai-eval-engineer`. The gate is not the goal; the gate is the signal.
- **Overrides on release-promotion are two-person.** As noted above.

An override is a conversation between the gate and the engineer, with the record as the transcript.

## The CI/CD integration surface

The plane does not run the merge queue; it provides decisions to it. Three integration points that cover the common systems:

- **GitHub Actions / GitLab CI / Buildkite.** The `platform-eval-gate` action / step calls the plane's gate API with the fixture set, the under-test revision, and the trigger. It blocks on the decision; it posts a PR comment with the CI breakdown.
- **The merge queue.** Mod-412's paved-road SDK ships a merge-queue integration that reads the plane's gate stream directly; the merge queue holds a PR until the pre-merge sweep clears.
- **The deploy service.** The release-promotion gate is a precondition the deploy service checks. A failing gate refuses promotion unless a two-person override is attached.

Three things the integration surface specifically does not do:

- **It does not decide the policy of blocking.** Whether a specific `regress` verdict blocks a merge, or just warns, is the fixture's wire-up policy, not the plane's. The plane produces the verdict; the fixture's gate configuration maps verdict → action.
- **It does not store the override policy.** Policies about who may override which fixture are mod-405's — the plane consults them, does not own them.
- **It does not retry on failure beyond a bounded policy.** A gate that retries indefinitely hides a flake; the plane retries once-with-different-seed on `terminal.infrastructure` errors and no further.

## The leaderboard — the visibility layer

A gate is a go / no-go. A leaderboard is a trend line. Both are necessary; neither replaces the other.

The leaderboard the plane ships:

- **One agent per row.** The current pass rate on every wired fixture, over the last 30 / 90 / 365 days.
- **One fixture per column.** The current pass rate across every agent that runs the fixture.
- **One bar per cell showing N-run variance.** The reviewer sees the CI, not just the point.
- **A regression flag per cell when the CI contracts below the baseline.** The flag links to the gate record.
- **A hover-over with the fixture's calibration card.** Judge agreement rate against humans, flake rate, train-cutoff warnings.
- **An audit trail per promotion.** Each `@latest` promotion of a fixture appears as a vertical line on the trend chart; the leaderboard's reader can see when a step up or down is "the fixture changed", not "the agent changed".

The leaderboard is not the gate; the leaderboard is the context for the gate.

## When to override your own gate

Last practical note. The plane makes gates easy to add and easy to override; it does not make them easy to delete. Deletion is reviewed, because a deleted gate is a silent regression tomorrow.

Three cases where override-and-track is correct:

- **The fixture is wrong.** A row's reference was updated incorrectly. Override, open an issue to fix the row, land the PR, re-baseline.
- **The judge is wrong.** A rubric revision is more strict than intended and false-positive rate spikes. Override, revert the judge-prompt revision, re-baseline.
- **The provider rolled under you and the regression is real but accepted.** A provider's new weights are 2 points worse on one axis and 4 points better on another, and the platform's chosen trade is the roll. Override, log the trade, re-baseline against the new pointer.

Three cases where override is wrong:

- **It is the end of the quarter.** Rushed overrides at release time are how real regressions land in production.
- **The authoring engineer is also the reviewer.** Two-person on release-promotion overrides exists for this.
- **The override has been invoked more than three times for the same fixture this quarter.** The fixture is telling you something — either it is flaky and `ai-eval-engineer` needs to fix it, or there is a real regression you are repeatedly overlooking.

## Summary

- Four gate kinds: per-PR regression, pre-merge full-sweep, nightly leaderboard, release-promotion. Each has its own fixture shape, cost envelope, and override model. A platform typically runs all four.
- Four standard fixtures — [SWE-bench](https://www.swebench.com/), [τ-bench](https://github.com/sierra-research/tau-bench), [GAIA](https://huggingface.co/datasets/gaia-benchmark/GAIA), [AgentBench](https://github.com/THUDM/AgentBench) — run platform-wide and give provider-roll attribution. Per-team custom fixtures — smoke, quality, regression — run on top.
- The statistical frame is N-run with bootstrap CI, compared to a baseline on the current production pointer. The gate is a hypothesis test; `delta_pass_rate_ci_upper < -epsilon` is the decision.
- The gate decision is a persisted record naming the baseline, carrying the CI, and surfacing any override. An override is signed, reasoned, half-lifed, and two-person on release-promotion.
- The CI/CD integration surface is thin — the plane produces decisions, the merge queue / deploy service acts on them. Policy about overrides is mod-405.
- The leaderboard is the visibility layer; it is not the gate. Both trend and go/no-go are necessary.

The next and final chapter closes out the module — the handoffs to `ai-eval-engineer` and `model-evaluation-engineer`, the operating cadence, and the open questions the plane will have to grow into.
