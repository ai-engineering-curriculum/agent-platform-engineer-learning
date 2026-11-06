# exercise-02: Promptfoo-Based Regression Gate

**Estimated effort:** 3 hours · **Output:** one Promptfoo project directory, a CI workflow that calls it, a gate-decision JSON file, a baseline / candidate run pair with bootstrap CI, and a short README (~1 page) · **Pairs with:** chapters [02](../02-the-harness-template-library-on-inspect-and-promptfoo.md) and [05](../05-eval-gated-ci-cd-and-standard-regression-fixtures.md); feeds exercises 03, 04, and project 403.

## Objective

Wire a per-PR regression gate using Promptfoo against the platform's HTTP provider shape, such that a pull request that regresses the fixture at `delta_pass_rate_ci_upper < -0.02` refuses to merge, and a pull request that is noise-equivalent to the baseline passes cleanly. The deliverable is the shape every "a prompt or RAG tweak just landed" PR runs through on your plane.

A well-authored gate consumes the N-run defaults from chapter 02, compares against a persisted baseline from the trace store, surfaces the gate record on the PR as a review comment, and honours a documented override shape (signed engineer + reason + fixture-flake-rate link). A gate that compares one run against one run, or that refuses to override without a two-person review on a per-PR scope, is wrong in the opposite direction; a reviewer will push back on either.

## Prerequisites

- You have read chapters [02](../02-the-harness-template-library-on-inspect-and-promptfoo.md) and [05](../05-eval-gated-ci-cd-and-standard-regression-fixtures.md).
- You have [Promptfoo](https://www.promptfoo.dev/docs/) installed (`npm install -g promptfoo` or `npx promptfoo`) and have run the getting-started example at least once.
- You have an OpenAI-compatible model endpoint you can POST to with a bearer token (your gateway, LiteLLM, vLLM, OpenAI directly). Both the under-test and judge providers can route to the same endpoint for the exercise.
- You have GitHub Actions available (any CI equivalent works — GitLab CI, Buildkite, CircleCI — the exercise focuses on gate decisions, not CI vendor).

## Problem statement

Pick one prompt-regression fixture to gate. If you did exercise 01, reuse the same fixture so you can cross-check between the Inspect-side and Promptfoo-side verdicts. Otherwise:

- **`writer-v1-smoke`** — 20 rows, each a brief → paragraph. Mix of deterministic assertions (length-bounds, contains-all, regex) and an `llm-rubric` judged on `faithfulness` / `tone`.
- **`support-agent-canned-responses`** — 30 rows, each a customer query → an approved answer. Deterministic `contains-any` on policy terms plus an `llm-rubric` on empathy.
- **`extractor-strict-json`** — 25 rows, each a document → a required JSON schema. `is-valid-openai-function-call` or `is-json` with schema validation; no judge needed.

(Specific rubric wording and judge thresholds are illustrative; flag `<!-- needs-research: ... -->` on any specific number you cannot defend from an `ai-eval-engineer` reference.)

## Requirements

### Part 1 — The Promptfoo project

Author `promptfoo-gate/<your-fixture>/` containing:

- `promptfooconfig.yaml` — the assertion list, the HTTP provider configuration pointing at your gateway / endpoint, and the judge-rubric provider block.
- `tests/` — the row set as individual `.yaml` files or as a `datasets.yaml` referencing a JSONL file (a stub catalogue is fine; see exercise 01).
- `rubrics/` — the judge prompt as a versioned file, with a `rubric-version.json` manifest carrying the content hash.
- `.env.example` — the env vars the CI step sets.

Four requirements the project must meet:

- The under-test provider is HTTP, pointing at an endpoint you can override with `MODEL_GATEWAY_URL`; the `X-Platform-Model-Identifier` header carries a platform-coded id.
- The judge provider uses a separate HTTP provider block with a distinct header so the gateway could bucket judge spend separately (even if your local endpoint is one instance).
- Every row carries a `_meta.fixture_version` and `_meta.rubric_version` field; the gate record (part 3) reads these.
- The config pins `temperature` on the under-test provider and documents the rationale in a `rationale.md` next to the config.

### Part 2 — The N-run harness

Promptfoo supports multi-run via `promptfoo eval --repeat N`. Wrap it so the harness:

- Runs N=3 for PR, N=10 for pre-merge, N=30 for nightly, picked via a `GATE_KIND` env var.
- Reads the shared baseline run (a `baseline.json` file in the project) and, after the candidate runs, computes a per-fixture `pass_rate`, `pass_rate_ci_low`, `pass_rate_ci_high`, `delta_pass_rate_ci_upper` using a 1,000-sample bootstrap over the N runs.
- Emits a `gate-decision.json` matching the shape in chapter 05 — `gate_run_id`, `trigger`, `under_test`, `fixtures[*]`, `overall_verdict`, `overall_action`, `override`.

### Part 3 — The CI workflow

Author `.github/workflows/eval-gate.yml` (or your CI equivalent) that:

- Triggers on pull_request against watched paths (prompt file, tool-schema file, routing config).
- Checks out the PR branch, installs Promptfoo, runs the N-run harness.
- Uploads `gate-decision.json` and the Promptfoo `share.yaml` as workflow artefacts.
- Posts a PR comment (via `actions/github-script` or `gh pr comment`) with: the per-fixture pass-rate, the CI, the delta, the verdict, and a link to the artefact.
- Exits non-zero when `overall_action == "block_merge"`, zero otherwise.

A workflow that posts a static "eval passed" comment regardless of the gate decision is a workflow a reviewer sends back.

### Part 4 — Override handling

Add an override surface. The engineer must be able to override a blocking verdict *from the PR itself*, without editing CI code:

- A PR label like `eval-override:<fixture>:<reason-ref>` triggers the gate to accept the regress verdict but write the override into the gate record.
- The override requires two pieces of evidence: the overriding identity (GitHub actor on the label event) and a `reason-ref` resolvable to a ticket / incident id.
- Overrides on `release-promotion` gates are rejected without a two-person review — but per-PR gates accept a single-person override. The exercise is per-PR; stretch covers release-promotion.
- Every override lands on `gate-decision.json` as a signed record.

### Part 5 — The baseline / candidate run pair

Produce two concrete runs:

- **Baseline run.** Capture and commit `baseline.json` for the fixture against a known-good prompt. Persist the baseline's `EvalRecord` set under `runs/baseline/`.
- **Candidate run (regress).** Modify the prompt deliberately to regress the fixture (e.g. remove a key instruction, shorten the system message). Open a PR; the CI should produce a `regress` verdict and block.
- **Candidate run (pass).** Revert the regressive change; the next CI run should pass cleanly.

Commit the three `gate-decision.json` files (baseline, regress candidate, passing candidate) so a reviewer can walk the trace.

### Part 6 — The README

One page covering:

- Which fixture, which three assertion kinds (at least one deterministic, at least one judged).
- The N / bootstrap settings per `GATE_KIND`.
- The override label format and what the override record looks like.
- The baseline-regress-pass pair, with the exact delta and CI numbers you observed.
- One open question for `ai-eval-engineer` — e.g. "our `llm-rubric` judge agrees with itself 82% across re-runs on the same input; what is the right threshold to retire it?"

## Starter guidance

- **Keep the HTTP provider small.** A single block pointing at `${MODEL_GATEWAY_URL}` with a header for the platform-coded model identifier is all you need. Resist the temptation to re-implement Promptfoo's provider internals.
- **Promptfoo's `--repeat` is your friend.** Wrapping it shell-side rather than re-running N times keeps the bootstrap honest.
- **Bootstrap is cheap; implement it.** A 1,000-sample bootstrap over 30 rows × 3 runs is milliseconds. Do not round to "pass rate > 0.80" for the exercise; the lesson is the CI.
- **Pin `temperature: 0` on deterministic-expected fixtures only.** On `llm-rubric` judged fixtures, pin the judge-side temperature (typically 0) and document the under-test temperature choice.
- **The baseline is content-addressed.** Record the under-test model, prompt hash, tool-schema hash, and fixture version on the baseline file. A baseline whose provenance is "whatever was in main last Thursday" cannot support a gate.
- **Flag `<!-- needs-research: ... -->` on every threshold.** `0.02` is the chapter's illustrative epsilon; your fixture may need a different one, and the choice is `ai-eval-engineer`'s.

## Acceptance criteria

Mark done when:

- [ ] `promptfoo eval` runs green on the baseline configuration and produces a `gate-decision.json` matching the chapter-05 shape.
- [ ] The regress candidate produces `overall_verdict: regress` with a CI upper bound strictly below `-0.02` of the baseline.
- [ ] The passing candidate produces `overall_verdict: pass`.
- [ ] The CI posts a PR comment with the per-fixture breakdown, uploads artefacts, and sets a non-zero exit code on block.
- [ ] A test PR with the override label is accepted; the override lands on `gate-decision.json` with the actor identity and reason reference.
- [ ] The N / bootstrap behaviour is driven by `GATE_KIND` and visible in the gate record.
- [ ] The README names fixture, assertions, N configuration, override format, the three runs, and one open question for `ai-eval-engineer`.

## Stretch goals

- **Two-person override on `release-promotion`.** Extend the override surface so a `release-promotion` gate requires a second GitHub reviewer identity in-band (e.g. a `/approve-override` chat command on the PR) before the override is accepted.
- **Judgement cache.** Content-address the judge input (`under_test_response_hash + rubric_hash + judge_model`); serve a cached verdict when all three match a prior run. Prove that re-running the passing candidate consumes zero judge calls.
- **Share-file ingestion into the trace store.** Have the workflow upload the Promptfoo `share.yaml` to the trace store stub (JSONL file for this exercise) so a reviewer who follows the leaderboard entry back to a run lands on the full trace.
- **Nightly pointer-roll detection.** Add a scheduled workflow that re-runs the baseline (no code change) against the current `platform:*` identifier. If the nightly `pass_rate_ci` diverges from the baseline's, open an issue titled `provider-roll-suspected: <fixture>`.
- **Multi-fixture overall verdict.** Wire a second fixture into the same gate; the overall verdict is `block_merge` if any fixture regresses, and the PR comment lists both fixtures' deltas side-by-side. Prove the override is per-fixture, not global.
