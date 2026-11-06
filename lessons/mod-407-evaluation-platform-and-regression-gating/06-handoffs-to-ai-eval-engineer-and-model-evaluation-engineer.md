# Handoffs to `ai-eval-engineer` and `model-evaluation-engineer`

The eval plane is a platform. A platform without a crisp handoff model is a platform that eats roles it should not — writing judge rubrics in a hallway because nobody else is, interpreting a p-value because nobody else is, deciding whether a 3-point drop on τ-bench is a regression or a trade because nobody else is. Within a year, the plane's team is in the eval-methodology business and the actual methodology owners are watching from a distance. The platform's leverage is gone.

This closing chapter is about the two peer roles the plane exists to serve — `ai-eval-engineer` (level 30) and `model-evaluation-engineer` (level 30) — and how the plane hands off cleanly to each. Chapter 01 names what the plane does not own; this chapter puts contracts on the handoffs.

## Who owns what — the three-way frame

Three roles, three centres of gravity. Platform decisions that cross centres are joint; platform decisions inside one centre are owned.

### Agent Platform Engineer (this role, level 40)

Owns the *platform*. The harness template library (chapter 02), the dataset management plane (chapter 03), the trace / replay store (chapter 04), the eval-gated CI (chapter 05), the judge-router and judge-budget machinery. Produces the paved-road artefacts other teams adopt. Operates the plane. Does not author judge rubrics, benchmark samplers, statistical analyses, or online-eval methodology. Hands methodological questions out and runs the answers back correctly.

### `ai-eval-engineer` (peer, level 30) — the methodology owner for application-layer eval

Owns *what makes an eval good at the application layer*. Judge rubric design and calibration against human labellers. Pairwise versus Likert choices, position-bias mitigations, self-consistency sampling, judge agreement thresholds. RAG eval methodology — retrieval metrics, groundedness measures, faithfulness tests. Online eval methodology — sampling policy on production traces, drift detection, canary analysis. Owns the per-team custom fixtures (chapter 05). Reports regressions back to the authoring stream-aligned team; tracks judge-drift alerts; owns the judge-prompt revision review.

### `model-evaluation-engineer` (peer, level 30) — the statistical and benchmark-methodology owner

Owns *benchmark construction and statistical rigour*. New benchmark design — row selection, difficulty stratification, sampling plan, baseline construction. Power analyses for small-N evals; confidence-interval construction beyond the plane's default bootstrap. Contamination and leakage methodology — is this benchmark still measuring capability or memorisation. Formal reports for leadership and compliance. Not an on-call rota; a designer and auditor.

A fourth role — `ai-risk-engineer` (peer, level 25) — authors the harm model the safety-oriented evals test against (red-team fixtures); mod-411 is the plane that consumes those obligations. The eval plane runs their fixtures but does not author them.

## What the plane hands off to `ai-eval-engineer`

Six concrete handoffs.

### 1. Judge-rubric authoring

The plane does not author `rubric://writer-v1-ragas@v3`. `ai-eval-engineer` does. The plane:

- Hosts the rubric registry (chapter 02) with hash-pinned, versioned rubrics.
- Enforces that a judge call cannot run against an unregistered rubric.
- Routes every judge call through mod-406 with the budget bucket.
- Posts a judge-drift alert when the plane observes a judge model or judge prompt revision without an approving `ai-eval-engineer` principal.

`ai-eval-engineer`'s contract back:

- Authors the rubric content.
- Publishes a calibration card per rubric (judge agreement rate against a held-out human set) and re-runs it on judge-model or rubric revision.
- Reviews judge-drift alerts and either approves (a planned revision) or rolls back (a surprise change).
- Owns the decision of "pairwise versus Likert versus deterministic scorer" for every fixture.

### 2. Per-team custom fixture content

The plane hosts the catalogue (chapter 03). `ai-eval-engineer` populates it. The plane:

- Validates envelope fields on publish (chapter 03).
- Runs contamination warnings at ingest.
- Serves rows through the template library.
- Surfaces coverage gaps on the leaderboard ("no regression fixture for writer-v1 has been updated in 120 days").

`ai-eval-engineer`'s contract back:

- Authors the rows, including the task definition, the reference answer, the rubric reference, the provenance metadata.
- Reviews new rows from the trace-to-catalogue promotion path (chapter 04) and either promotes (useful regression case) or rejects (one-off noise).
- Cuts a new fixture version when the fixture shape changes meaningfully, and attaches a changelog to the lineage pointer.

### 3. Judge-model calibration and periodic re-calibration

The plane routes judge calls and logs their verdicts. `ai-eval-engineer` interprets.

- The plane surfaces per-judge statistics: agreement rate against held-out humans, position-bias rate (for pairwise judges), self-consistency rate (judge disagreeing with itself across temperature-0 re-runs on the same input).
- `ai-eval-engineer` sets the thresholds — "if judge-agreement drops below 0.78, the judge is retired and the fixtures it scored are re-baselined" — and reviews when the plane alerts on a threshold crossing.

### 4. Online eval

The plane captures production traces (chapter 04) and provides a sampling API. `ai-eval-engineer` writes the sampling policy and the online judge configuration.

- The plane does not decide what fraction of production traffic to sample, which strata to sample over, or how the sampled population is partitioned for canary analysis.
- The plane *runs* the sampling policy the methodology owner writes, enforces the judge budget, and feeds the `EvalRecord` stream.

### 5. Flaky-fixture triage

The plane surfaces flake rates and override counts (chapter 05). `ai-eval-engineer` decides what to do about them.

- Fixture X has an override rate > 3 per week — is the fixture wrong, is the judge wrong, or is there a real regression people are ignoring?
- Methodology on the three-way distinction is `ai-eval-engineer`'s. The plane runs their fix (re-baseline, judge revision, fixture retirement).

### 6. Judge-prompt change control

Every judge-prompt revision is a reviewed release. The plane enforces the review mechanically; `ai-eval-engineer` reviews the content.

- A judge-prompt PR must list the fixtures the rubric is used in.
- The plane runs a `judge-prompt-regression` pass — the new rubric applied to the last N nightly runs — and surfaces the delta in verdict for every fixture.
- `ai-eval-engineer` reviews the delta and either approves, revises, or abandons.

## What the plane hands off to `model-evaluation-engineer`

Four concrete handoffs.

### 1. New benchmark construction

Adding a new standard fixture to the platform — a new SWE-bench variant, a new multi-modal benchmark, a new tool-use benchmark — is `model-evaluation-engineer`'s methodological call.

- The plane hosts the fixture in the catalogue once authored and runs it through the template library.
- `model-evaluation-engineer` writes the sampling plan, the difficulty stratification, the baseline construction, the public-vs-private split, and the statistical test that constitutes "regression".
- Authoring is the handoff; running is the plane's.

### 2. Power analysis and confidence-interval design

The plane's bootstrap CI is a reasonable default, not the right choice for every fixture.

- For a small-N high-stakes fixture (50 rows, release-blocking), bootstrap may under-report uncertainty; a Wilson interval or Clopper-Pearson interval on the pass rate may be more appropriate.
- For a pairwise fixture, a Bradley-Terry model is a stronger aggregation than a simple win rate.
- `model-evaluation-engineer` chooses per fixture; the plane runs the chosen aggregation.

### 3. Contamination methodology

The plane runs ingest-time warnings (chapter 03). `model-evaluation-engineer` interprets.

- Is a 2% n-gram overlap a concern or not? Does the known-benchmark hash set need updating? Is a publication-date-vs-train-cutoff warning a real concern for this fixture, or is the benchmark re-published often enough that the latest rows are safe?
- The plane produces the signal; the methodology owner reads it.

### 4. Formal reports

Leadership asks "is the platform's agent fleet getting better over time?" An answer longer than a sentence requires methodology.

- The plane's leaderboard (chapter 05) is a trend line, not a report.
- `model-evaluation-engineer` writes the quarterly capability report, choosing the fixtures that constitute "capability" over time, controlling for provider rolls, and producing a decision-grade document.

## What a clean handoff looks like

A clean handoff has four properties. The template library's `create-eval` scaffolder (chapter 02) encodes each.

- **The handoff has a schema.** The rubric reference is `rubric://...`; the fixture reference is `catalogue://...`; the judge model is a platform-coded identifier. A handoff that is "send a Google doc with the rubric text" has no schema and leaks into the plane.
- **The handoff is revision-controlled.** Rubrics, fixtures, judge models are versioned the same way agent code is. A handoff that is "just update the prompt in the YAML" has no revision control and leaks.
- **The handoff has a reviewer.** The approving principal's identity is on every change. A handoff where "anyone with repo write can change the rubric" has no reviewer and leaks.
- **The handoff has a bounded time to interpret.** A judge-drift alert the plane fires expects an `ai-eval-engineer` response within 24 hours; an override frequency alert expects a response within a week. A handoff with no SLA leaks by default.

## The operating cadence

Rituals the plane runs with its handoff peers. Pick a subset at startup; grow the list as the platform matures.

- **Weekly judge-drift review** with `ai-eval-engineer`. Fifteen minutes. The plane reports judge-model revisions, rubric revisions, and threshold crossings; `ai-eval-engineer` dispositions each.
- **Bi-weekly flake-rate and override review** with `ai-eval-engineer`. Thirty minutes. Walk fixtures over 1% flake rate and over 3-per-week override rate; decide per fixture.
- **Monthly new-benchmark review** with `model-evaluation-engineer`. Thirty minutes. Review proposed new standard fixtures, methodology changes, and contamination warnings.
- **Quarterly capability report sign-off** with `model-evaluation-engineer` and the platform's leadership. Longer. `model-evaluation-engineer` presents, the plane's team answers infrastructure questions.
- **Incident-driven fixture additions** on no cadence but on every incident. The trace-to-catalogue promotion path (chapter 04 → chapter 03) is the mechanical surface; the review at promotion is the human one.

## Hand-offs out to adjacent platform roles

The handoffs above are the two the module names. Three more land naturally in the plane's day-to-day.

- **`ai-infra-sre` (peer) and your own future mod-410 work** consume the leaderboard and gate-failure stream as operational signals; mod-410's incident cadence back-feeds regression fixtures through the trace-to-catalogue promotion path.
- **`agentic-safety-engineer` (peer, level 40)** authors red-team fixtures that live in the catalogue the same way per-team custom fixtures do. The plane runs them; `agentic-safety-engineer` interprets.
- **`agentic-ai-engineer` (level 30) and `senior-agentic-ai-engineer` (peer, level 40)** are the primary customers. They author the agents that the plane evaluates. Their contract to the plane is "I will not bypass the gate"; the plane's contract to them is "the gate will not be noise".

Mod-401 chapter 04 details the paved-road social contract these handoffs sit inside.

## The open questions the plane will grow into

Last practical note. The surface this plane covers is moving fast. Three questions the plane does not yet have perfect answers to, and that will shape the module over the next two revisions:

- **How do you evaluate an agent that calls another agent?** Mod-409's agent-to-agent routing creates eval shapes where the "agent under test" invokes other platform agents. Is that a system eval (score the outcome) or a composition eval (score each hop)? The methodology is immature; the plane will need new primitives in the trace store and the replay engine when it stabilises.
- **How do you run online eval without burning the production trace store on judge calls?** Judge-at-production-rate is expensive. The current pattern — sample a fraction, judge offline — leaves a long tail of untested production behaviour. Novel methodology from `ai-eval-engineer` will drive plane features here.
- **How do you detect that a benchmark is contaminated *after* ingest?** The ingest-time warnings are shallow. Deeper methodology — cross-checking outputs against published solutions, measuring per-row revealed-knowledge — is a `model-evaluation-engineer` concern. The plane will need to support their tooling as it emerges.

Mark `<!-- needs-research: ... -->` on any specific answer you commit to on these three. The flag convention from mod-406 holds here; it blocks auto-merge, and that is cheaper than a confident wrong answer.

## Summary

- The plane owns the platform; `ai-eval-engineer` owns application-layer eval methodology; `model-evaluation-engineer` owns benchmark construction and statistical rigour.
- Six handoffs to `ai-eval-engineer` — rubric authoring, per-team custom fixtures, judge calibration, online eval policy, flaky-fixture triage, judge-prompt change control. Four handoffs to `model-evaluation-engineer` — new benchmark construction, power analysis and CI design, contamination methodology, formal reports.
- A clean handoff has a schema, revision control, a reviewer, and a bounded SLA. The `create-eval` scaffolder encodes these; a handoff that is "update the Google doc" leaks into the plane.
- Operating cadence rituals — weekly judge-drift, bi-weekly flake / override review, monthly new-benchmark, quarterly capability report, incident-driven fixture additions — keep the plane and the methodology owners in sync.
- Three open questions — agent-to-agent eval, scalable online eval, post-ingest contamination — will shape the plane's next two revisions. Flag them with `<!-- needs-research: ... -->` on any specific commitment.

The module is done. Four exercises walk the key deliverables; project 403 (Eval-Gated Agent Fleet) composes them into a live plane.
