# exercise-04: Guardrail Effectiveness vs Red-Team Findings

**Estimated effort:** 3 hours · **Output:** a replay harness that scores the plane against a red-team corpus, a signed quarterly scorecard in the chapter 06 shape, a drift-detection pipeline over the production stream, a fixture-promotion stub that feeds mod-407, and a short README (~1 page) · **Pairs with:** chapter [06](../06-measuring-guardrail-effectiveness.md); feeds project 404. Builds on exercises [01](exercise-01-safety-plane-architecture-and-sla.md), [02](exercise-02-prompt-injection-defence-at-platform-scope.md), and [03](exercise-03-standardised-safety-event-schema.md).

## Objective

Chapter 06 pinned the four metrics — recall, precision, FP rate, latency — and the operating cadence — per-change replay, weekly drift, quarterly scorecard. This exercise forces you to run the whole loop end-to-end, with a real corpus, against your plane from exercises 01–03, and to produce a scorecard you would be comfortable signing in front of the risk engineer, the agentic-safety engineer, and the leadership plane.

The lesson the exercise intends to teach: a plane's effectiveness is a *claim*, not an *assertion*. The claim is signed by three roles; the evidence is the harness output; the artefact is versioned and quarterly. A plane without a scorecard is a plane without a defensible safety posture.

## Prerequisites

- You have completed [exercise 01](exercise-01-safety-plane-architecture-and-sla.md), [exercise 02](exercise-02-prompt-injection-defence-at-platform-scope.md), and [exercise 03](exercise-03-standardised-safety-event-schema.md). The hooks emit real `SafetyRecord`s on a stream.
- You have read chapter [06](../06-measuring-guardrail-effectiveness.md) and chapter [07](../07-build-vs-buy-for-guardrail-stacks.md).
- You have a red-team corpus beyond exercise 02's single-shot injection set. For this exercise the corpus must span at least four OWASP categories — minimum: `LLM01` (both direct and indirect variants), `LLM02` (PII), `LLM06` (excessive agency via tool-call patterns), and one more of your choice (`LLM05`, `LLM07`, `LLM08`, `T01`, `T02`). Acceptable sources include the public corpora from exercise 02 plus your own hand-authored rows in the chapter 06 YAML format.
- You have a benign corpus from exercise 02 or an expanded version. Minimum ~150 rows spanning at least three topic areas (QA, code, multilingual).
- You have a staging environment with the ability to run the replay against the plane without perturbing other work.

## Problem statement

The quarterly scorecard is due. The `ai-risk-engineer` and the `agentic-safety-engineer` have handed you a red-team corpus (either the one you assembled in exercises 01–02 or a richer one). You own the measurement and the artefact. Your output has six parts:

1. A `red_team_corpus/` directory with the rows in the chapter 06 YAML shape — each row has `id`, `author`, `category.owasp`, `category.agentic`, `severity`, `origin`, `scenario`, `expected_verdict`, `notes`.
2. A replay harness `harness/run.py` that replays every row through the plane and produces a scorecard.
3. A scorecard in chapter 06's shape — a Markdown report plus a machine-readable `scorecard.json`.
4. A drift-detection pipeline over the production `SafetyRecord` stream.
5. A fixture-promotion stub that detects reproducible misses and emits a `candidate_fixture_<finding_id>.yaml` file in the mod-407 catalogue shape.
6. A short README explaining the above and listing one quarterly regression with owner and remediation plan.

## Requirements

### Part 1 — The red-team corpus

Under `red_team_corpus/<quarter>/`:

- Minimum 50 rows spanning at least four OWASP categories.
- Each row conforms to the chapter 06 YAML schema.
- At least five rows of `severity: critical`.
- At least 10 rows of `origin: novel` (your own hand-authored).
- At least 10 rows of `origin: public_mirror` — attributed to a public source (Gandalf, PromptBench, garak, Simon Willison running coverage).
- A `MANIFEST.yaml` with the corpus revision, the row count per category, the generation date, and a changelog.

A note on authorship: writing novel rows is the risk engineer's job; for this exercise you are briefly playing that role. Keep the novel rows true to the category — a row tagged `LLM06:excessive_agency` should actually exercise the excessive-agency path, not the LLM01 classifier.

### Part 2 — The replay harness

`harness/run.py` that:

- Takes `--corpus-rev`, `--policy-pack`, `--engine-version`, `--output-dir`.
- Iterates every row and calls the appropriate hook endpoint from exercises 01–02 with the row's scenario.
- Compares the emitted `SafetyRecord.verdict` against the row's `expected_verdict`.
- Records the match outcome: `caught` (verdict matches expected), `missed` (expected refuse/redact/hitl but got allow), `over-refuse` (expected allow but got refuse/redact), `wrong-category` (correct decision but wrong category set).
- Records `hook.latency_ms` for each call.
- Produces `scorecard.json` grouped by category, severity, trust tier, and origin.
- Runs the benign corpus through the same hooks and produces FP metrics.
- Runs in CI-friendly mode (exit code non-zero if the deploy-gating rules from chapter 06 fail).

Deploy-gating rules (chapter 06):

- Any regression on `severity: critical` rows (row was caught at the prior corpus rev, missed at the current) fails the run.
- Overall recall drops more than 2pp vs. the prior scorecard fails the run.
- FP rate rises more than 2pp vs. the prior scorecard fails the run.
- Per-hook p95 latency exceeds the SLA from exercise 01 fails the run.

For this exercise you can author a baseline `scorecard.prior.json` by hand; it stands in for the "prior scorecard" the gate compares to.

### Part 3 — The quarterly scorecard

A `scorecards/<quarter>/SCORECARD.md` matching chapter 06's shape:

- Headline numbers (overall recall at severity=critical, high, medium; overall precision; FP rate; p95 latencies).
- A category-sliced table.
- A regressions section with owner and remediation plan — invent at least one plausible regression and document it realistically.
- A new-categories section if any novel categories appeared.
- A build-vs-buy signal section if any engine is near-budget on latency or showing recall drift (chapter 07 feed).
- Three signature lines: `@agent-platform-engineer:you`, `@ai-risk-engineer:(placeholder)`, `@agentic-safety-engineer:(placeholder)`.

The scorecard MUST be Markdown that reads as an auditable artefact. "Numbers in a tabular format with explanations" is the bar; prose without numbers is a fail.

### Part 4 — The drift-detection pipeline

A `drift/monitor.py` that:

- Subscribes to the production `SafetyRecord` stream (from exercise 03).
- Maintains a 28-day trailing baseline of the three chapter 06 drift signals: refusal rate, novel-category frequency, confidence-distribution per category.
- Alerts (prints to stdout, writes a `drift-alerts.jsonl`, posts to a Prometheus `/metrics` endpoint — your choice; configurable) when any signal crosses a threshold.
- The thresholds are configurable; the defaults should match chapter 06's ±30% drift bound and a reasonable novel-category appearance threshold (e.g., > 0.1% of records for a category not in the top-10 the previous week).

Produce a demo scenario:

- Script that generates `SafetyRecord`s for two weeks of synthetic traffic (say, 10k records) at a steady-state distribution.
- Script that then injects a week of records at 2× the baseline refusal rate.
- Show the drift monitor detecting the injected anomaly and emitting an alert within one day of simulated drift.

### Part 5 — The fixture-promotion stub

A `fixture_promoter/` module that:

- Reads the harness output and the drift alerts.
- For each `missed` row in the scorecard with `severity: high` or `severity: critical`, emits a `candidate_fixture_<row_id>.yaml` in the mod-407 catalogue shape (your call on the exact fields; include `fixture_id`, `source` as `safety/<row_id>`, `input` as the row's scenario, `expected_verdict`, and `owner`).
- For each drift alert that is `confirmed` (you can mark alerts as confirmed by hand for this exercise), emit a candidate fixture.
- Does NOT automatically write to the real catalogue — the fixture is a *candidate*; adopting it is the risk engineer's call. Produce the files in `candidates/`.
- Each candidate fixture's `safety_record_ref` points to the `SafetyRecord.record_id` or the `red_team_refs[0]` on the originating row.

### Part 6 — The README

One page covering:

- How to run the harness (`python harness/run.py --corpus-rev 2026-10`).
- How to generate the scorecard and where it lands.
- How to run the drift demo (the two scripts).
- How the fixture promoter works and how a candidate fixture is adopted.
- Two "I was surprised by" observations — e.g., "the scorecard's overall-recall number was 90% but the severity=critical recall was 70%; averaging over severity hides the number that matters."
- One open question for the leadership plane — e.g., "the drift monitor fires when refusal rate is +30% of baseline; our launch cadence is weekly; what is the operating cadence for responding to drift alerts when a tenant's own cadence is at odds with the plane's response SLA?"

## Starter guidance

- **Start with the gate.** Write the deploy-gating rules before the metrics computation; the gate is the thing that will catch your regression in a month.
- **Include at least one critical-severity regression you catch.** Add a row to the corpus; verify it is `missed`; document the regression and the remediation in the scorecard. The scorecard is unconvincing without at least one real regression to show how it is handled.
- **The benign corpus is where the hard calibration lives.** Expect the first scorecard to show over-refusal on code and multilingual content; that is the chapter 06 lesson.
- **Keep the harness deterministic.** Pin seeds; pin engine versions; pin corpus revs. The scorecard is a signed artefact; "ran it again, got different numbers" is a failed artefact.
- **Measure latency realistically.** The classifier's p95 on your laptop is not the production p95. Note the discrepancy in the scorecard explicitly.
- **Reuse the emitter from exercise 03.** Every replayed verdict emits a `SafetyRecord` with `red_team_refs` populated. The schema constraint doubles as a replay-integrity check.

## Acceptance criteria

Mark done when:

- [ ] The red-team corpus has 50+ rows spanning 4+ categories; at least five rows at `severity: critical`; at least 10 `origin: novel`.
- [ ] `harness/run.py` runs end-to-end and produces `scorecard.json`.
- [ ] The quarterly `SCORECARD.md` exists in chapter 06's shape with three signature lines.
- [ ] The scorecard reports at least one regression with an owner and a remediation plan.
- [ ] `drift/monitor.py` runs the demo scenario; the injected anomaly is detected; the alert lands in the configured sink.
- [ ] The fixture promoter produces at least one candidate fixture in the `candidates/` directory; the fixture is valid against your mod-407 catalogue shape.
- [ ] The harness fails the deploy-gating rules deterministically against a known-bad policy-pack (demonstrate by introducing a regression in one engine version).
- [ ] README includes the two "surprised by" observations and the open question for leadership.
- [ ] All emitted records from the harness carry `red_team_refs` populated; a test asserts this.

## Stretch goals

- **Severity-weighted recall.** Report a single "weighted recall" number using the chapter 06 implied weights (critical = 4, high = 2, medium = 1, low = 0.5). Shows how the scorecard's weighted view differs from the flat view.
- **Two engines side-by-side.** Replay the corpus against both your incumbent classifier and a candidate (chapter 07 swap discipline). Produce a comparative scorecard and a recommendation.
- **Production-trace replay.** Sample 100 production records; replay them through the current and candidate configurations; report divergence. The point is to show that lab-replay and production-replay can disagree.
- **False-negative hunt.** Seed the benign corpus with five records that *should* have been refused (deliberate mis-labels); run the harness; verify your pipeline detects the mislabel. Discipline over the corpus's own quality.
- **Automate the promotion loop.** Wire the fixture promoter to actually call a mock mod-407 catalogue API (any HTTP POST) when the risk engineer "confirms" a candidate (a flag file, a CLI flag, your choice).
- **Add a Rogue-Agent (OWASP Agentic T13) scenario.** A multi-agent setup where one agent is a plant; measure how the plane detects behavioural drift on it; score the detection.
- **Tenant-opt-in production samples.** Simulate a tenant contributing 50 production rows annotated as benign; fold them into the benign corpus for the next scorecard; show the FP rate adjustment.
