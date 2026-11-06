# exercise-01: Inspect Harness Template Library

**Estimated effort:** 3 hours · **Output:** one `inspect-template` directory (Task, Solver, Scorer, SDK shim, Dockerfile), one `create-eval` scaffolder stub, two reproducible runs (fault-injected judge, real judge), and a short README (~1 page) · **Pairs with:** chapters [02](../02-the-harness-template-library-on-inspect-and-promptfoo.md) and [01](../01-the-evaluation-plane-and-what-it-owns.md); feeds exercises 02, 03, 04, and project 403.

## Objective

Build the minimum shape of the plane's Inspect-side template: a `Task` + `Solver` + `Scorer` stack that is wired to the platform's model gateway, dataset catalogue, judge router, and trace sink, such that a stream-aligned team can clone the template and have a running eval in under two hours. Wrap the template in a `create-eval` scaffolder so the engineer running it never writes a provider URL, judge prompt literal, or catalogue path by hand.

A well-authored template produces chapter-01's `EvalRecord` on every row, pins every hash the record demands (agent revision, prompt, tool schema, judge prompt, fixture), runs an N-run default with bootstrap CI on pass rate, and keeps judge-cost bounded by a fixture-level budget. If your template hard-codes `gpt-4o` as the judge, or if the judge prompt is a Python string in a `.py` file rather than a `rubric://` reference, a reviewer will send it back.

## Prerequisites

- You have read chapters [01](../01-the-evaluation-plane-and-what-it-owns.md), [02](../02-the-harness-template-library-on-inspect-and-promptfoo.md), and [04](../04-trace-and-replay-store-for-regression-investigation.md).
- You have [Inspect AI](https://inspect.aisi.org.uk/) installed (`pip install inspect-ai`) and have run at least one example task.
- You have access to the platform's model gateway from exercise mod-406/exercise-01 — or any OpenAI-compatible endpoint (gateway, LiteLLM proxy, vLLM, local Ollama) that you can call with a bearer token and route against a platform-coded model identifier.
- You have the [OpenTelemetry Semantic Conventions for Generative AI](https://opentelemetry.io/docs/specs/semconv/gen-ai/) open while you wire spans.

## Problem statement

Pick one of three fixture shapes to author the template against; the shape of your template follows the fixture:

- **`writer-v1-smoke`** — 20 rows, each a one-paragraph brief and a reference paragraph. Rubric: `faithfulness`, `tone`, `length-bounds`. Deterministic length-bounds scorer plus a model-graded faithfulness / tone scorer. Pass-rate style verdict.
- **`planner-v1-toolloop`** — 30 rows, each a multi-step task with a reference final state. Tools are stubs (`search`, `lookup`, `commit`) that return canned outputs the template ships. Deterministic final-state comparator; no judge. Pass / fail verdict.
- **`critic-v1-rubric`** — 50 rows, each an input/output pair plus a ground-truth rubric score. Model-graded with `platform:judge-v1`; verdict is agreement with the ground-truth rubric (Cohen's kappa across rows, pass-rate threshold).

(The exact rubric wording, length bounds, and judge thresholds you pick are illustrative; flag any specific number as `<!-- needs-research: ... -->` when your authoring choice is not backed by an `ai-eval-engineer` reference.)

## Requirements

### Part 1 — The template directory

Author `inspect-template/<your-fixture>/` containing:

- `task.py` — the Inspect `Task` definition. The task calls into the SDK shim for dataset load, model configuration, and scoring. The engineer's code is the task's structural decisions (max_messages, solver choice), not the plumbing.
- `rubrics/` — one file per rubric used by the task, in the shape `rubric://<name>@<version>.yaml`. The file is referenced by hash; the SDK fetches and pins at run-time.
- `fixture-contract.md` — names the row schema the fixture expects (input columns, reference columns, metadata columns) and the provenance expectations (chapter 03).
- `Dockerfile` — reproducible runner image; the engineer should be able to `docker build && docker run` and get the same answer as in CI.
- `README.md` — the usage card for the engineer cloning the template.

### Part 2 — The SDK shim

Author `platform_eval_sdk/` with at minimum:

- `catalogue_uri_resolver(uri) -> Dataset` — resolves a `catalogue://<name>@<version>` URI against your local stub catalogue (a directory of JSONL files is sufficient for this exercise). Verifies the manifest hash before iterating rows; errors loudly on mismatch.
- `platform_judge(model, rubric_ref) -> Scorer` — a scorer wrapper around Inspect's `model_graded_qa` that: (a) resolves the rubric reference against a local stub rubric registry; (b) pins the rubric's content hash on the sample metadata; (c) emits a `judge-audit` row to a sink; (d) respects a judge-budget interface (part 4).
- `eval_record_sink(suite_id)` — a sink that writes one `EvalRecord` per sample (chapter 01 schema) to a JSONL file at the end of the run.
- A `trace_sink` stub that writes `.eval` plus a session JSONL under `./traces/<trace_id>/` so a replay from exercise 03 is possible.

### Part 3 — The `create-eval` scaffolder

Write a one-command scaffolder (bash + jinja, or a small Node / Python script) that generates a new template instance:

```sh
./create-eval \
  --agent writer-v1 \
  --framework inspect \
  --fixture catalogue://writer-v1-smoke@latest \
  --judge platform:judge-v1 \
  --gate writer-v1-nightly
```

Output: a copy of `inspect-template/` with:

- The agent id, fixture URI, judge identifier, and gate name baked into `task.py`, `README.md`, and the CI YAML.
- A generated `.env.example` listing the env vars (`MODEL_GATEWAY_URL`, `EVAL_RUNNER_TOKEN`, `CATALOGUE_URL`).
- A generated CI config (GitHub Actions is fine) that calls `inspect eval task.py` and uploads the `EvalRecord` JSONL as an artefact.

A scaffolder that outputs a template referencing an un-pinned judge or a local-path fixture fails this part — the whole point is to prevent that class of error.

### Part 4 — The judge-budget interface

Judges are model calls. Wire a budget guard into `platform_judge`:

- Before every judge call, check a per-run budget bucket (`JUDGE_BUDGET_USD`, defaulted to `0.50` in the template).
- Estimate the cost of the judge call (token count × price-table rate — your gateway's billing event from exercise mod-406/exercise-04 is the ideal source, a flat per-call estimate is acceptable for the exercise).
- Refuse the call when the bucket is exhausted and emit a sentinel `EvalRecord` row with `verdict: judge_budget_exhausted`.
- Emit a running total to stderr every N calls so an engineer running locally sees the spend.

### Part 5 — Two reproducible runs

Produce two end-to-end runs and record both as `run-01.evalrecord.jsonl` and `run-02.evalrecord.jsonl`:

- **Run 01 — fault-injected judge.** The judge model is stubbed to return a fixed verdict table (half of rows pass, half fail) and a known cost per call. Verifies the plumbing — EvalRecord, hash pinning, budget accounting — without burning real tokens.
- **Run 02 — real judge.** The judge is called against a real provider (your platform gateway if you have one, an OpenAI-compatible endpoint otherwise). Verifies the live path.

Both runs must produce byte-identical EvalRecords on the structural fields (hashes, routing, fixture version) when re-run against the same inputs. The scoring fields (`verdict`, `scoring.metrics`) may vary for run 02 within the pass-rate CI; the structural fields must not.

### Part 6 — The README

One page covering:

- Which fixture you chose and why.
- The `create-eval` command and the surface it generated.
- The three hashes the template pins (fixture version, judge prompt, under-test revision) and where each is surfaced on the `EvalRecord`.
- How the judge-budget guard behaves on exhaustion (what the sentinel row looks like, what the exit code is).
- One open question you would route to `ai-eval-engineer` — e.g. "when is a length-bounds scorer appropriate on its own vs. needing a judge in combination?"

## Starter guidance

- **The SDK shim is the lesson, not Inspect.** Inspect gives you Task / Solver / Scorer; the exercise is in wrapping them so the engineer never writes a provider URL. Keep Inspect-level code thin.
- **Pin the rubric content hash as early as possible.** If your judge prompt is sourced from a YAML file read at runtime, hash it at load and attach the hash to every `EvalRecord` the judge touches.
- **Stub the catalogue.** A local directory of JSONL files with a `manifest.json` carrying a SHA-256 over the sorted row contents is enough. The lesson is the manifest-hash check, not a full catalogue service.
- **Do not bake secrets into the template.** The scaffolder must emit `.env.example`, not a working `.env`. A template that commits a key is a template a reviewer sends back on sight.
- **N=3 for the local run.** The default in the chapter is PR=3 / pre-merge=10 / nightly=30; the exercise's two runs are PR-scale.
- **Flag `<!-- needs-research: ... -->` on every provider-specific latency / cost figure.** Prices and endpoint shapes move; the flag convention matches mod-406's.

## Acceptance criteria

Mark done when:

- [ ] `inspect eval task.py` runs green on a stub judge locally and produces a well-formed `EvalRecord` per row.
- [ ] The `EvalRecord` carries `fixture.version_hash`, `scoring.judge_prompt_hash`, `under_test.prompt_hash`, `under_test.tool_schema_hash`, and `under_test.agent_revision`; none is `null`.
- [ ] The scaffolder emits a template copy for a new fixture without leaving any TODOs — the generated template runs end-to-end with the right env vars.
- [ ] The judge-budget guard refuses calls past the configured budget and writes a `judge_budget_exhausted` sentinel row.
- [ ] Both run-01 and run-02 artefacts are reproducible on the structural fields; the `pass_rate` on run-01 is deterministic; the `pass_rate` on run-02 falls inside a pre-declared CI.
- [ ] The README names fixture, scaffolder surface, the three hashes, the budget-exhaustion behaviour, and one open question for `ai-eval-engineer`.

## Stretch goals

- **Add a Promptfoo-sibling template.** Author a `promptfoo-template/<fixture>/` with a `promptfooconfig.yaml` wired to the same judge router. Run the same fixture through both frameworks; compare per-row verdicts. The lesson is where the two frameworks agree and where they diverge — the per-framework drift is itself a signal.
- **Wire an OpenTelemetry span per model call.** Emit `gen_ai.*` attributes per the semantic conventions. The under-test span and the judge span should be distinguishable by attribute. Mod-408 will consume these.
- **Judgement cache.** Content-address the judge's input (`(under_test_response_hash, judge_prompt_hash, judge_model_identifier, reference_hash)`); serve a cached verdict when all four match a prior run. Prove that a second run with no changes consumes zero judge dollars.
- **Rubric-change diff.** Add a `rubric-diff v2 v3` command that runs both rubric versions over the last run's traces and surfaces which rows changed verdict. This is the backbone of chapter 02's judge-prompt change control.
- **Replay integration.** On a run completion, the template submits the top-5 lowest-scoring rows to a replay queue (a JSONL file for this exercise) so exercise 03's trace / replay store can promote them into a regression fixture.
