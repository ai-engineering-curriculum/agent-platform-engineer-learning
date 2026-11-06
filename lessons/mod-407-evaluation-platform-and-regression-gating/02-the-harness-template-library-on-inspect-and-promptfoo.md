# The Harness Template Library on Inspect and Promptfoo

A stream-aligned team authoring a new agent should not need to hand-roll an eval harness. They already have an agent, a judge rubric their product owner cares about, a handful of representative tasks, and a deadline. If authoring an eval that runs against the platform's judge router, reads from the catalogued dataset store, publishes traces to the replay store, and plugs into the regression gate takes two weeks, the platform has failed. If it takes two hours, the platform is working.

This chapter is about how to make it two hours. Two frameworks — [Inspect AI](https://inspect.aisi.org.uk/) from the UK AI Security Institute (AISI), and [Promptfoo](https://www.promptfoo.dev/docs/) from the Promptfoo project — dominate the open-source eval-harness space in late 2026. Each has a different centre of gravity. The plane adopts both and ships templates on top; stream-aligned teams pick the right one by task shape, not by religion.

## Why both frameworks, not one

Each framework optimises for a different eval shape. Forcing one to do both jobs is how bespoke harnesses are reinvented.

**[Inspect AI](https://inspect.aisi.org.uk/)** is the choice for agent evals and research-grade benchmarks. It is written in Python, centres on `Task`, `Solver`, and `Scorer` abstractions, and has native solvers for agent loops, tool-use, chain-of-thought, and model-graded scoring. It ships integrations against OpenAI, Anthropic, Google, AWS Bedrock, Azure, and any OpenAI-compatible endpoint (which covers vLLM, SGLang, Ollama, and the gateway shape in mod-406). Its artefacts are rich: a run writes a `.eval` log that carries every sample, every solver message, every scorer verdict, every token count — replayable, inspectable, diffable. Standard benchmarks are already packaged — SWE-bench, GAIA, GPQA, HumanEval, MMLU, and many more in [`inspect-evals`](https://github.com/UKGovernmentBEIS/inspect_evals).

**[Promptfoo](https://www.promptfoo.dev/docs/)** is the choice for prompt / RAG regression evals and quick CI gates. It is YAML-configured, assertion-first, runs from a single CLI, and its [`promptfoo eval`](https://www.promptfoo.dev/docs/usage/command-line) command is already how most teams in late 2026 wire an eval into GitHub Actions. It has first-class providers for OpenAI, Anthropic, Google, OpenAI-compatible endpoints, HTTP providers (the shape mod-406's gateway speaks natively), and dozens of built-in assertions — contains, equals, javascript, llm-rubric, factuality, latency, cost, model-graded-closedqa, g-eval, and so on. Its artefacts are per-test-case rows that teams cut into dashboards and share-links.

A rough selection rule the template library bakes in:

- **Agent-loop evals with tool use and multi-turn behaviour → Inspect.** The `basic_agent` solver, the tool-sandboxing convention, and the `.eval` trace fidelity are the right fit.
- **Prompt / RAG / single-turn regression → Promptfoo.** The YAML declaration, the assertion library, and the CI ergonomics are the right fit.
- **Both frameworks for the same agent is normal.** An agent will typically have an Inspect agent-loop eval that runs weekly on a custom fixture plus SWE-bench, and a Promptfoo prompt-regression eval that runs on every PR.

The plane does not pick one and forbid the other. The plane ships templates on top of each, routes both through the same judge router, publishes both to the same trace store, and reads both into the same leaderboard.

## What a template encapsulates

A template is not a tutorial. It is a wired artefact a team can clone and run against the real platform — same identity, same judge router, same dataset catalogue, same trace sink — without touching any of those surfaces by hand.

An Inspect template, at minimum, ships:

- **A `Task` definition** pre-wired to the platform's dataset loader so a team references a dataset by catalogue URI rather than a local path.
- **A `Solver` wired to the model gateway.** The template calls the gateway via its OpenAI-compatible endpoint, using a platform-coded model identifier (`platform:writer-v1`), with the SPIFFE workload identity injected from the runner's environment. Agents being evaluated do *not* speak directly to `api.openai.com`.
- **A `Scorer` scaffold with two choices.** A deterministic scorer (exact match / regex / schema-valid) and a model-graded scorer whose judge route is `platform:judge-v1` through the judge router. The template never hardcodes `gpt-4o` as a judge.
- **A `.eval` log sink** that writes to the trace store as well as locally, so a replay-store query against the eval's trace is possible without rerunning.
- **A `pyproject.toml` or `requirements.txt`** pinning Inspect and the template version; a `Dockerfile` reproducing the runner.
- **A `README.md`** naming the fixture, the judge, the gate, the expected pass rate.

A Promptfoo template, at minimum, ships:

- **A `promptfooconfig.yaml`** with the provider pointed at the platform's HTTP provider (the mod-406 gateway), the model identifier platform-coded, the dataset reference a catalogue URI, and the default assertions wired to `platform:judge-v1`.
- **A `tests/` directory** scaffold with example assertion shapes — contains, equals, javascript, llm-rubric, g-eval — each wired to the judge router where it uses a judge.
- **A CI workflow** calling `promptfoo eval` with the right env vars, uploading the share file to the trace store, and posting a comment on the PR.
- **A `fixture-contract.md`** naming what the fixture expects per row (input columns, reference columns, metadata columns).

Both templates ship through a `create-eval` scaffolder:

```sh
pnpm dlx @platform/create-eval \
  --agent writer-v1 \
  --framework inspect \
  --fixture catalogue://writer-v1-ragas@latest \
  --judge platform:judge-v1 \
  --gate writer-v1-nightly
```

The scaffolder, not the engineer, wires the identity, the catalogue URI, the judge route, and the gate name. The engineer writes the solver / assertion logic and the per-row scoring. That is the division of labour the template library delivers.

## The platform-specific surface a template abstracts

Four surfaces must look the same across every eval the plane runs. A template's job is to hide them so a team never gets them wrong.

### 1. Model routing

Agents under test, and judges that score them, route through the model gateway (mod-406). The template's provider configuration is:

```yaml
# promptfoo — under-test provider
providers:
  - id: http
    config:
      url: https://model-gateway.platform.local/v1/chat/completions
      method: POST
      headers:
        Authorization: 'Bearer ${EVAL_RUNNER_TOKEN}'
        X-Platform-Model-Identifier: 'platform:writer-v1'
        X-Eval-Run-Id: '${EVAL_RUNNER_RUN_ID}'
      body: |
        {"model": "{{ model_identifier }}", "messages": {{ messages | dump }}}
```

```python
# inspect — under-test and judge model
from inspect_ai.model import get_model

under_test = get_model(
    "openai/platform:writer-v1",
    base_url="https://model-gateway.platform.local/v1",
    api_key_var="EVAL_RUNNER_TOKEN",
)
judge = get_model(
    "openai/platform:judge-v1",
    base_url="https://model-gateway.platform.local/v1",
    api_key_var="EVAL_RUNNER_TOKEN",
)
```

The engineer never writes a provider's API URL in an eval. The provider identifier is always platform-coded (mod-406 chapter 02). Judge calls and under-test calls carry different headers so the gateway can bucket judge spend separately (mod-406 chapter 04).

### 2. Identity and token

Eval runners have a SPIFFE workload identity (mod-405 chapter 04). The identity is exchanged — once, at job start — for a short-lived bearer token the gateway accepts. The template fetches the token at startup; the engineer never writes or rotates one. Judge budget enforcement, dataset access control, and trace-store writes all key off the identity.

### 3. Dataset reference

Datasets live in the catalogue (chapter 03) and are referenced by `catalogue://<name>@<version>`. The template's dataset loader resolves the URI, fetches the content-addressed objects, and verifies the manifest hash before iterating rows. Reading a dataset by a local file path is a template-level anti-pattern the scaffolder refuses to emit. A reviewer should be able to re-fetch the exact fixture a score was produced against from the catalogue URI alone.

### 4. Trace / log sink

Every run writes:

- The framework's native log (Inspect `.eval`, Promptfoo `share.yaml`).
- A normalised `EvalRecord` per sample (chapter 01) to the EvalRecord stream.
- A trace reference per sample to the trace / replay store (chapter 04), so a reviewer who follows a leaderboard entry back to a run can land on the full trace.

The template ships the three sinks wired through a single configuration block the engineer never edits.

## The judge router and judge-prompt pinning

A judge call is a model call. It routes through mod-406 with four things distinct from a normal agent call:

- **A dedicated model identifier** — `platform:judge-v1`, `platform:judge-cheap-v1` — mapped to a provider / model combination by the routing table, rolled under change control, rolled back under the same change control.
- **A dedicated budget bucket** per eval suite. The gateway (mod-406 chapter 04) tracks judge-bucket spend against a published monthly budget; the eval scheduler refuses new runs when the bucket is exhausted. A runaway `llm-rubric` assertion on a 50k-row fixture is detected at the gateway, not at the invoice.
- **Pinned judge prompts.** The judge's prompt is a hashed, versioned artefact in a repository, not a string in a per-team YAML. A judge-prompt revision is a reviewed release under `ai-eval-engineer`'s change control. The template bakes `judge_prompt_hash` into the `EvalRecord`.
- **A `judge-audit` log** written alongside the under-test trace: judge model identifier, resolved provider, judge prompt hash, judge input tokens, judge output, judge verdict. This is the artefact a reviewer reads when they ask "did the judge change its mind this week?"

The methodology for judge design — rubric construction, calibration against humans, pairwise versus Likert, judge agreement, position-bias mitigations — is `ai-eval-engineer`'s. The plane's job is to run the judge they author correctly, reproducibly, and within budget.

## A concrete template — an Inspect task against a custom fixture

```python
# inspect_templates/writer_ragas.py
from inspect_ai import Task, task
from inspect_ai.dataset import json_dataset
from inspect_ai.model import get_model
from inspect_ai.solver import basic_agent, system_message
from inspect_ai.scorer import model_graded_qa
from platform_eval_sdk import (
    catalogue_uri_resolver,
    eval_record_sink,
    platform_judge,
)

UNDER_TEST = "openai/platform:writer-v1"
JUDGE = "openai/platform:judge-v1"
BASE_URL = "https://model-gateway.platform.local/v1"

@task
def writer_ragas() -> Task:
    dataset = json_dataset(
        catalogue_uri_resolver("catalogue://writer-v1-ragas@latest"),
    )
    return Task(
        dataset=dataset,
        plan=[
            system_message("You are writer-v1, drafting for enterprise customers."),
            basic_agent(
                model=get_model(UNDER_TEST, base_url=BASE_URL),
                tools=[],  # stub — plugs into mod-403 in a real eval
                max_messages=12,
            ),
        ],
        scorer=platform_judge(
            model=get_model(JUDGE, base_url=BASE_URL),
            rubric_ref="rubric://writer-v1-ragas@v3",
        ),
        sinks=[eval_record_sink(suite_id="writer-v1@2026-11-06")],
    )
```

Six things to notice:

- **No provider URL, no API key, no judge prompt literal.** All are indirected through the SDK.
- **The dataset is a catalogue URI.** A reviewer landing on this file can trace exactly which rows ran.
- **The judge is a platform model identifier.** A change to the judge model is a routing-table change, not a code change here.
- **The rubric is a reference.** The platform's rubric registry stores hashed, versioned judge prompts; the `platform_judge` scorer resolves and pins them at run-time.
- **The sinks publish an `EvalRecord`.** The leaderboard reads this stream without parsing `.eval` files.
- **`max_messages=12`** is an explicit budget guard on the agent loop under test, independent of the gateway's per-session guard (mod-406 chapter 04). A template without a budget is a template a reviewer sends back.

## A concrete template — a Promptfoo config against a custom fixture

```yaml
# promptfoo_templates/writer-v1-regression/promptfooconfig.yaml
description: writer-v1 prompt regression — runs on every PR

providers:
  - id: http:platform-gateway-under-test
    config:
      url: ${MODEL_GATEWAY_URL}/v1/chat/completions
      method: POST
      headers:
        Authorization: 'Bearer ${EVAL_RUNNER_TOKEN}'
        X-Platform-Model-Identifier: 'platform:writer-v1'
        X-Eval-Run-Id: '${EVAL_RUN_ID}'
      transformResponse: 'json.choices[0].message.content'

tests:
  - vars:
      _dataset: catalogue://writer-v1-smoke@latest
    assert:
      - type: contains-all
        value: ['customer', 'SLA']
      - type: javascript
        value: |
          output.length >= 200 && output.length <= 1400
      - type: llm-rubric
        provider:
          id: http:platform-gateway-judge
          config:
            url: ${MODEL_GATEWAY_URL}/v1/chat/completions
            headers:
              Authorization: 'Bearer ${EVAL_RUNNER_TOKEN}'
              X-Platform-Model-Identifier: 'platform:judge-v1'
              X-Eval-Rubric-Ref: 'rubric://writer-v1-smoke@v2'

sharing:
  - sink: platform-trace-store
    config:
      suite_id: writer-v1-smoke@2026-11-06
```

Same four surfaces as the Inspect template — platform-coded models, env-injected token, catalogue-URI fixture, trace-store sink. The engineer's code is the test list and the deterministic / rubric asserts; everything else is template.

## Flakiness, seeds, and the N-run default

A single run is a sample, not a measurement. The template library bakes N-run with aggregation by default:

- **N = 3 for PR gates, N = 10 for nightly, N ≥ 30 for release-blocking fixtures.** The numbers are tunable per fixture; the defaults come from `ai-eval-engineer`.
- **Temperature pinning where the methodology admits it.** For deterministic-expected tasks (code-gen with a reference, schema-validation tasks), the template pins `temperature: 0` and surfaces any provider that does not honour the pin as a failure, not a passing row. For judgement-style tasks, temperature is kept at a documented value (typically 0.7) and the variance is explicitly part of the measurement.
- **Seed pinning where the provider supports it.** OpenAI's `seed` parameter and vLLM's `seed` support are template-level knobs; providers without seed support emit a `seed_unsupported` note on the record.
- **Bootstrap confidence intervals, not point comparisons.** Aggregation produces `pass_rate`, `pass_rate_ci_low`, `pass_rate_ci_high`, `mean_score`, `mean_score_ci`. The gate (chapter 05) consumes the CI, not the point.

Methodology on judge agreement rates, inter-rater reliability, and when temperature pinning is defensible comes from `ai-eval-engineer`. The plane is what makes the methodology running by default rather than opt-in.

## Caching and incremental runs

A full re-run of 2,000 rows on every PR is expensive and almost always wrong — the dependency that changed affected a handful of rows, not all of them. The template library caches two levels:

- **Under-test response cache.** Keyed on `(agent_revision, prompt_hash, tool_schema_hash, model_identifier, row_id, seed)`. If nothing changed, the response is served from cache and the row's `under_test_usd` is zero on the record.
- **Judgement cache.** Keyed on `(under_test_response_hash, judge_prompt_hash, judge_model_identifier, reference_hash)`. If the output is byte-identical to a prior run's and the judge has not moved, the verdict is read from cache.

Both caches are content-addressed and tenant-scoped; the cross-tenant leakage concern (mod-406 chapter 05) applies, and the template library inherits the gateway's partitioning. A reviewer can disable both via a scaffolder flag; a CI gate on a release-blocking fixture turns judgement cache off and under-test cache on, by default.

## When *not* to use the template library

Pin honesty: the library is the paved road, not the only road. Three cases merit going off-road:

- **Research-grade work on benchmark construction.** A team evaluating whether to add a new benchmark should prototype outside the template library and promote in once the shape stabilises. The template library optimises for repeatability, which is the wrong optimisation when you are still learning the shape.
- **Online eval on production traffic.** A scheduled offline eval over a fixture is one shape; passive sampling of production traces with an online judge is another. The online case leans on the trace / replay store (chapter 04) and the observability plane (mod-408); the methodology comes from `ai-eval-engineer`. It is not what the template library is for.
- **Statistical benchmarks that require careful power analysis.** A benchmark that needs 5,000-row sample size with stratified sampling across difficulty tiers is an `model-evaluation-engineer` methodology concern. The template library can run it; authoring it is the handoff.

Chapter 06 details both handoffs.

## Summary

- The plane adopts **both** Inspect AI and Promptfoo. Inspect for agent-loop, tool-use, and multi-turn research-grade evals; Promptfoo for prompt / RAG regression in CI. The template library ships on both and routes both through the same judge router, dataset catalogue, and trace store.
- A template is a wired artefact. Four surfaces are hidden from the engineer: model routing (via mod-406), identity and token (via mod-405), dataset reference (via the chapter-03 catalogue), and trace / log sinks (into the chapter-04 store and the EvalRecord stream).
- Judges are model calls. They route through mod-406 with a dedicated identifier, dedicated budget bucket, pinned prompts, and a `judge-audit` log.
- N-run defaults, seed / temperature pinning where defensible, and bootstrap CIs are the baseline. A point comparison is a bug in the gate, not a feature.
- Two levels of cache — under-test and judgement — keep incremental runs cheap. Both are content-addressed and tenant-scoped.
- The template library is the paved road; research-grade benchmark design, online eval, and statistical-benchmark methodology are handoffs out to `ai-eval-engineer` and `model-evaluation-engineer`.

The next chapter opens the dataset management plane — the catalogue the template library resolves URIs against, the provenance model that makes a `row_provenance` field honest, and the access-control model that keeps regulated data out of a public benchmark run.
