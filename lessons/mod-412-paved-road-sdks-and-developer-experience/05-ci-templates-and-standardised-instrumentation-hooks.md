# CI Templates and Standardised Instrumentation Hooks

Chapter 04 brought the local-dev loop to the engineer's laptop. This chapter is the second half of the same bet: the CI pipeline that turns "the local loop passes" into "the production shape holds under review", and the instrumentation hook library that makes the shape honest without requiring the engineer to hand-write OpenTelemetry attribute strings.

A reusable CI workflow is not an optional convenience. The CI is where the five paved-road gates — lint, unit, eval regression, safety regression, cost — get *enforced*, where the `AgentRevision` manifest is *emitted*, and where the attribute-contract test *fails* the build if the agent stopped emitting the attributes it claimed. If any of those happens inline in tenant repos rather than in a reusable workflow, each tenant will diverge within a quarter and the paved road splinters.

## What the CI template owns

Five concrete responsibilities. Each is a *gate* — a job that returns pass / fail and that the pull-request merge policy honours.

| Gate | What it runs | Pass criterion | Required or advisory |
|---|---|---|---|
| **Lint** | Style, type checks, paved-road import-rule enforcement (chapter 02 rule 2) | No lint errors | Required |
| **Unit** | The agent repo's own unit tests, including adapter-level unit tests | All tests pass | Required |
| **Eval regression** | `paved-road eval --mode strict` against the fixture pins (mod-407) | All fixtures pass; PR curve within tolerance | Required (advisory on first adoption; chapter 07's promotion path) |
| **Safety regression** | Replay the paved-road's red-team corpus (mod-411) against the agent's hooks | Precision / recall deltas within policy-pack tolerance | Required (advisory on first adoption) |
| **Cost / latency smoke** | `paved-road eval --mode cost-smoke` — a small, representative set of fixtures run against the real model | p95 latency and avg cost per session within workload-class budget | Required for T1/T2; advisory for T3 |

Plus two *build-time emissions*:

- The `AgentRevision` manifest emission (chapter 01).
- The attribute-contract test — a verification that the agent's actual span emissions match the semantic conventions it declared (below).

Plus three *advisory signals* — not pass/fail, but published on the PR:

- SBOM diff ([CycloneDX](https://cyclonedx.org/) or [SPDX](https://spdx.dev/)) — paved-road dependency graph against the previous commit.
- Trace-sampled preview — a selection of spans from the eval-regression run attached as artefacts, so a reviewer can inspect them.
- Adoption-manifest snapshot — the `AgentRevision` manifest attached to the PR so a reviewer can see the SDK / adapter / template versions at a glance.

## The reusable workflow shape

The gold-standard shape is a reusable workflow that tenant repos import with a single line. Three CI systems cover most of the mainstream:

### GitHub Actions — reusable workflows

```yaml
# .github/workflows/paved-road.yml  — in the tenant repo
name: paved-road
on: [push, pull_request]
jobs:
  paved-road:
    uses: platform/paved-road-ci/.github/workflows/paved-road.yml@v3
    with:
      agent_slug: writer-v1
      tenant_id: acme-financial
      workload_class: interactive-chat
      tier: T1
    secrets:
      model_gateway_token: ${{ secrets.MODEL_GATEWAY_TOKEN }}
```

The callee lives in a single platform-owned repo (`platform/paved-road-ci`) and implements every gate as a separate `job`. Tenant repos never copy the gates inline. See [GitHub docs — reusing workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows).

### GitLab CI — `include:`

```yaml
# .gitlab-ci.yml  — in the tenant repo
include:
  - project: platform/paved-road-ci
    ref: v3
    file: /paved-road.gitlab-ci.yml

variables:
  AGENT_SLUG: writer-v1
  TENANT_ID: acme-financial
  WORKLOAD_CLASS: interactive-chat
  TIER: T1
```

GitLab's `include:` resolves at pipeline-generation time; the included file defines the stages and jobs. See [GitLab docs — include keyword](https://docs.gitlab.com/ee/ci/yaml/includes.html).

### Buildkite — pipeline templates

```yaml
# .buildkite/pipeline.yml  — in the tenant repo
steps:
  - label: ":paved: paved-road"
    pipeline:
      from: platform/paved-road-ci/paved-road.buildkite.yml@v3
      variables:
        agent_slug: writer-v1
        tenant_id: acme-financial
        workload_class: interactive-chat
        tier: T1
```

Buildkite pipeline templates work through dynamic pipelines; see [Buildkite docs — pipeline templates](https://buildkite.com/docs/pipelines/templates).

### The workflow version pin is the deprecation surface

The `@v3` tag is where the deprecation ladder (chapter 07) attaches. A tenant repo stuck on `@v2` after `v3` ships emits a warning in `platform doctor` (chapter 03) and a lint note in the PR; after the grace period, `@v2` is deleted from the platform repo and the tenant's CI fails hard. The ladder and the grace-period calculus are chapter 07; the mechanism is the workflow version pin.

## The five gates — what each one actually runs

### Gate 1 — Lint

Four classes of check. All ride on standard tooling; the paved-road contribution is the **import-rule config** that enforces chapter 02's adapter-boundary rule.

```yaml
# paved-road-ci calls these per-language:
- ruff check .                       # Python style + import-sort
- mypy --strict .                    # Python types
- paved-road lint imports            # the paved-road import rule
- paved-road lint manifest           # .platform/manifest-template.yaml validity
```

The `paved-road lint imports` command reads the SDK's allow-list and refuses to pass if the agent imports a framework-native identifier (`anthropic.*`, `openai.*`, `google.adk.*`, `agent_framework.*`, `langgraph.*`, `temporalio.*`) outside an explicitly tagged escape-hatch block (e.g., `# paved-road: framework-escape [EXCEPTION-ID-0043]`). The exception id references the exception register (chapter 07) — a tagged escape without a matching register entry fails lint.

### Gate 2 — Unit

The template's own unit tests plus two adapter-level additions:

- **`PavedRoadAgent` conformance** — a pytest fixture (shipped by the SDK) that asserts the agent's class satisfies `isinstance(agent, PavedRoadAgent)` and that each lifecycle hook is callable.
- **Hook-order test** — with the stub safety plane, the stub policy plane, and the mock model gateway, drive one turn and assert the span tree order matches chapter 02's composition diagram.

```python
# tests/test_paved_road.py  — scaffolded by the template
def test_agent_is_paved_road():
    assert isinstance(WriterAgent(), PavedRoadAgent)

def test_hook_order(paved_road_test_stack, trace_recorder):
    agent = WriterAgent()
    with paved_road_test_stack(safety_pack="paved-road-test", policies="allow-all"):
        asyncio.run(agent.run("hello", session=stub_session()))
    assert trace_recorder.span_order() == [
        "turn.start",
        "safety.input_filter",
        "gen_ai.completion",
        "safety.output_validator",
        "reliability.record",
        "turn.end",
    ]
```

### Gate 3 — Eval regression

The hardest gate to get right, and the one that most differentiates an agent paved road from a microservice paved road. Cross-reference mod-407.

Three modes of eval in CI:

| Mode | What runs | When |
|---|---|---|
| `strict` | Full fixture catalogue pinned in `AgentRevision.fixtures.catalog_pins`; each fixture run against the real model gateway. Verdict from the paved-road judge (mod-407). | On every PR merge to `main`; nightly. |
| `fast` | A subset of fixtures (`smoke/*`) against the real model gateway; the full catalogue against the mocked model gateway for shape-only verification. | On every PR open / push. |
| `offline` | The mock model gateway returns fixture-declared responses; validates composition without exercising model quality. | Pre-commit hook; `make eval` locally. |

The PR default is `fast`. The merge gate is `strict`. The local-dev loop is `offline`. This matches the triangle: tight feedback on PR, honest feedback on merge.

The eval gate's pass criterion is not "no fixture failures". It is:

- **Deterministic fixtures** (shape, structured output, tool-call argument assertion): must pass 100%.
- **Judge-scored fixtures** (final-answer quality): judge verdict must be within the policy-declared tolerance band — "no regression below last green's judge score within statistical CI". Reviewing a PR that regresses the judge score three points is chapter 07's office-hours territory.
- **Regression against a logged incident** (mod-410 chapter 06's promoted fixtures): must pass 100% for the lifetime of the fixture.

### Gate 4 — Safety regression

Replay the mod-411 red-team corpus against the agent's hooks. Scored precision / recall / FP rate; must stay within the policy pack's declared tolerance (mod-411 chapter 06). The CI emits a scorecard record (JSON) attached to the PR and ingested into the safety plane's dashboard.

Three levels of enforcement by workload class / tier:

| Workload class / tier | Safety regression enforcement |
|---|---|
| T1 interactive-chat (user-facing, paying tenant) | Required; any regression on safety precision/recall > policy tolerance blocks merge. |
| T2 batch-document | Required; higher tolerance (e.g., 2 percentage-point regression allowed if accompanied by a signed exception). |
| T3 internal-tool | Advisory; posted to PR; owner can override with sign-off. |

The policy tolerance is not the paved road's choice — it is the safety plane's (mod-411 chapter 06). The CI gate reads the tolerance from the policy pack pinned in the manifest.

### Gate 5 — Cost / latency smoke

Run `paved-road eval --mode cost-smoke` — a small set of fixtures against the real model gateway — and assert:

- `avg cost per session ≤ workload_class.cost_budget.avg`
- `p95 latency ≤ workload_class.latency_budget.p95`
- `avg input/output token counts within declared bands` (mod-406's cost model depends on these)

The budgets live in `paved-road-ci/budgets.yaml` per workload class and tier:

```yaml
# platform/paved-road-ci/budgets.yaml
workload_classes:
  interactive-chat:
    T1:
      cost_budget_usd:
        avg_per_session: 0.05
        p95_per_session: 0.20
      latency_budget_s:
        ttft_p95: 3.0
        ttfa_p95: 12.0
    T2:
      cost_budget_usd:
        avg_per_session: 0.10
      latency_budget_s:
        ttfa_p95: 60.0
  batch-document:
    T2:
      cost_budget_usd:
        avg_per_session: 0.25
      latency_budget_s:
        ttfa_p99: 600.0
```

The budgets cross-reference mod-410's SLO document templates and mod-406's cost model. The CI gate is the enforcement point; chapter 06 counts the breaches in `deployment_success_rate`.

## The attribute-contract test

The attribute-contract test is a separate, dedicated job inside the paved-road CI. It is what makes the SDK's instrumentation hook library (below) *trustworthy* — a dependency bump of the SDK that silently stopped emitting a required attribute would otherwise go unnoticed.

```
$ paved-road ci attribute-contract
Reading contract: paved-road-sdk/contracts/gen_ai-v1.37.0.yaml
Driving one deterministic turn against the offline stack ...
Captured spans: 7
Checking required attributes:
  gen_ai.request.model                  ✓ present on 2 spans
  gen_ai.request.max_tokens             ✓ present on 2 spans
  gen_ai.response.model                 ✓ present on 2 spans
  gen_ai.usage.input_tokens             ✓ present on 2 spans
  gen_ai.usage.output_tokens            ✓ present on 2 spans
  gen_ai.response.finish_reasons        ✓ present on 2 spans
  platform.session.id                   ✓ present on all spans
  platform.tenant.id                    ✓ present on all spans
  platform.agent.id                     ✓ present on all spans
  platform.workload.class               ✓ present on all spans
  platform.workload.tier                ✓ present on all spans
  platform.safety.record.id             ✓ present on 3 spans
PASS — attribute contract gen_ai-v1.37.0 + platform-v2026-10-01 satisfied.
```

Four invariants:

1. **The contract is pinned in the manifest.** `AgentRevision.instrumentation.gen_ai_semconv_rev` and `platform_semconv_rev` declare the revs the build targeted. The test reads the contract for those revs from the SDK's shipped contract files.
2. **The test drives real spans through a real OTel collector.** Not a static source-code scan — a live trace from the offline stack. A hook that declares the attribute but does not actually emit it fails.
3. **The test reads the SDK's contract, not the production collector's.** This keeps the test offline and independent of operations.
4. **Changes to the contract are versioned.** Cross-reference mod-408 chapter 05.

The test's output is itself a span-event attached to a receipt span, so the collector can gate ingestion on the contract version (mod-408 chapter 06).

## The standardised instrumentation hook library

The library — call it `paved_road_agent.instrumentation` — is where every `gen_ai.*` and `platform.*` attribute originates. The library is imported by every adapter (chapter 02) and, as a consequence, is imported by every paved-road agent transitively.

Its surface is small on purpose.

```python
# paved_road_agent/instrumentation/__init__.py — conceptual outline
from contextlib import contextmanager
from opentelemetry import trace
from opentelemetry.semconv._incubating.attributes import gen_ai_attributes as semconv_gen_ai

_tracer = trace.get_tracer("paved_road_agent", version=__version__)

def _platform_attributes(session) -> dict[str, str]:
    return {
        "platform.session.id": session.session_id,
        "platform.tenant.id": session.tenant,
        "platform.agent.id": session.agent_id,
        "platform.agent.revision": session.agent_revision,
        "platform.workload.class": session.workload_class,
        "platform.workload.tier": session.tier,
        "platform.sdk.version": __version__,
    }

@contextmanager
def gen_ai_span(name, session, model, **kwargs):
    with _tracer.start_as_current_span(name) as span:
        for k, v in _platform_attributes(session).items():
            span.set_attribute(k, v)
        span.set_attribute(semconv_gen_ai.GEN_AI_REQUEST_MODEL, model)
        try:
            yield span
        finally:
            # record token usage, finish reasons on span exit
            pass

def record_tool_call(span, tool_name, mcp_resolution_id):
    span.set_attribute("gen_ai.tool.name", tool_name)
    span.set_attribute("platform.mcp.resolution_id", mcp_resolution_id)

def record_safety_verdict(span, verdict):
    span.set_attribute("platform.safety.record.id", verdict.record_id)
    span.set_attribute("platform.safety.decision", verdict.decision)

def emit_reliability_record(session, turn_outcome, latency_ms, tokens, cost):
    record = {
        "schema_version": "reliability-record/v1",
        "trace_id": session.trace_id,
        "session_id": session.session_id,
        "turn_outcome": turn_outcome,
        "latency_ms": latency_ms,
        "input_tokens": tokens.input,
        "output_tokens": tokens.output,
        "cost_usd": cost,
    }
    _reliability_record_emitter.emit(record)
```

Four design choices:

1. **Attribute names are imported from the semconv package, not strings.** `semconv_gen_ai.GEN_AI_REQUEST_MODEL` is the imported constant from the [OpenTelemetry GenAI semconv Python package](https://pypi.org/project/opentelemetry-semantic-conventions/). When semconv revs, the import breaks at the SDK level — not silently at runtime in every agent. `<!-- needs-research: confirm the exact Python import path at the semconv rev we pin -->`
2. **Platform attributes are a single dict.** `_platform_attributes(session)` is the one function that knows what the platform's attributes are; change the shape in one place and every span updates.
3. **Records are first-class.** `emit_reliability_record` (mod-410), `emit_safety_record` (mod-411), `emit_eval_record` (mod-407) all live here. The adapter calls them; the records flow to the collector or to a local JSONL file depending on config.
4. **The tracer is named after the SDK.** mod-408 chapter 05's instrumentation-scope attribute surfaces `"paved_road_agent"` with the SDK version; an agent instrumented by a non-SDK path is distinguishable.

### The hooks between agent and adapter

The library also exposes `before_turn`, `after_turn`, `before_tool_call`, `after_tool_call`, `on_error` as mix-ins the adapter composes. An agent repo that wants to extend a hook does so by overriding the method; the standardised attributes stay on because the default implementation calls the library.

```python
# agent repo — overriding after_turn
class WriterAgent(AnthropicAgentAdapter, PavedRoadAgent):
    async def after_turn(self, session, output):
        await super().after_turn(session, output)        # keeps the paved-road emissions
        # tenant-specific additional metric
        self._my_feature_counter.inc()
```

A hook override that does not call `super()` produces a warning on the CI's attribute-contract test. Chapter 07's office-hours triage sees these as "you probably meant to call super".

## PR annotations — what the gates write on the PR

Every gate posts a comment / check-run. The paved-road comment-shape is pinned so reviewers see the same thing on every tenant's PR.

```
Paved-road CI — writer-v1 — pass

Gates:
  lint                    ✓
  unit                    ✓
  eval-regression (fast)  ✓  12/12 fixtures, judge score 0.86 (Δ+0.01 vs main)
  safety-regression       ✓  precision 0.94, recall 0.88 (within paved-road@2026-10-01 tolerance)
  cost-latency-smoke      ✓  avg $0.014/session (budget $0.05), p95 TTFA 1.8s (budget 3.0s)
  attribute-contract      ✓  gen_ai v1.37.0 + platform 2026-10-01

Manifest:
  SDK:         paved_road_agent @ 2.4.1
  Adapter:     AnthropicAgentAdapter @ 2.4.1
  Template:    agent-chat-anthropic @ 3.2.0 (3.3.0 available)
  Tier:        T1

Artefacts:
  AgentRevision manifest     agent-revision.yaml
  Eval report                eval-report.json
  Safety scorecard           safety-scorecard.json
  Trace sample (3 fixtures)  traces.tgz
```

Four things:

1. **Pass/fail per gate**, not a single aggregate.
2. **Deltas vs. `main`** — the eval judge score, the safety precision/recall, the latency numbers. A reviewer can see what moved.
3. **Manifest summary** — SDK, adapter, template versions. The deprecation ladder (chapter 07) reads these.
4. **Artefacts** — attached so a reviewer can download the eval report, the scorecard, and the trace sample without clicking through to the CI UI.

## The tenant override surface

Tenants sometimes need a gate behaviour the paved road does not ship. Three kinds of override:

| Override | Shape | Governance |
|---|---|---|
| Add a tenant-specific extra gate | `paved-road-ci.extra-gates:` entry in the workflow config | None — tenant concerns stay in their repo. |
| Loosen a paved-road gate (e.g., allow one failing fixture) | `.platform/exception-overrides.yaml` with an `exception_id` from the register | Requires a chapter-07 exception-register entry; CI reads and validates. |
| Change the paved-road gate shape (e.g., lower the cost budget) | *Not available via tenant override* | Must go through the budgets-file PR in the paved-road CI repo, reviewed by the platform. |

The three layers protect the paved-road claim: tenants can *add* to the gates freely; they can *loosen* only via the exception register; they cannot *change* the gates themselves.

## Secrets and credentials in CI

Three rules:

1. **Model-provider credentials belong to the model gateway, not to the agent repo.** The `cost-latency-smoke` gate calls the model gateway with a short-lived token; the paved-road CI vault issues the token at job start. The token is scoped to this tenant, this workload class, and this budget. If the token is leaked, the budget caps blast radius.
2. **Safety-plane policy packs are read-only in CI.** The paved-road CI pulls the pack from the platform's policy-pack repo on every job; the tenant repo never contains the pack.
3. **Fixtures with sensitive content are reference-only.** A fixture that needs sensitive input content references a secret-managed URI (`secretfixture://...`); the paved-road CI resolves it from a vault; the fixture metadata (expected shape, verdict) is public and lives in the repo.

The paved-road CI's job of being the *one* place credentials are handled is what keeps tenant repos free of the operational responsibility.

## When to split the paved-road CI repo

A single paved-road CI repo serves the platform for a long time, but at some size the following splits become real:

- **Per-language** — Python, TypeScript, .NET live in separate repos; the paved-road CLI selects the right one.
- **Per-CI-system** — GitHub Actions, GitLab CI, Buildkite workflows live in separate repos; the scaffolder picks the right scaffolder-generated workflow file.
- **Per-gate** — the eval-regression gate becomes complex enough to live in its own repo with its own release cadence; the paved-road CI depends on it at pinned versions.

Each split is governed the same way chapter 07 governs the SDK — SemVer, deprecation ladder, exception register.

## Common pitfalls when authoring the CI template

- **Inlining the gates in each tenant repo.** A tenant repo with its own `pytest` + `ruff` + `paved-road eval` invocations is off the paved road the moment the paved-road eval command changes. Everything goes through the reusable workflow.
- **The CI passes locally but fails on the platform runner.** Different runner means different environment; this is where `docker-compose.dev.yml` and the paved-road CI's container drift silently. The template's smoke test runs `platform ci simulate` which executes the reusable workflow locally; chapter 04's loop includes it.
- **A failing advisory gate blocks merge.** Advisory means advisory. If the organisation's merge policy marks all CI jobs required, split the advisory gates into a separate check-run name that the merge policy knows is advisory.
- **Not pinning the reusable workflow version.** `uses: platform/paved-road-ci/.github/workflows/paved-road.yml@main` is a floating pin; every main-branch update to the CI breaks tenant PRs that were green an hour ago. Always pin a tag (`@v3`, `@v3.2`). Chapter 07 governs the moves.
- **Attribute-contract test as a source-code scan.** A static scan passes because the identifier is there; the live span tree is where the attribute actually has to appear. Always drive a real turn through a real collector.
- **Policy packs loaded at image build time.** The pack is pulled per-job from the policy-pack repo; building it into the CI image means every pack revision requires a CI-image rebuild. Dynamic fetch is the paved-road default.
- **Cost gate against a model the agent does not use.** The budget is parameterised by workload class and tier; the `cost-latency-smoke` call to the model gateway resolves the same profile the agent uses. Reading a tenant-specific `cost-budget.yaml` is tempting and wrong — the budget comes from the paved-road `budgets.yaml` with the manifest's declared profile.

## A worked example: a PR through the paved-road CI

An engineer on the `writer-v1` agent opens a PR that renames the system prompt. Walk the pipeline:

1. **`lint` (8 s)** — ruff + mypy + paved-road import check. Pass.
2. **`unit` (12 s)** — pytest, including the hook-order test. Pass.
3. **`eval-regression (fast)` (24 s)** — smoke fixtures against the real model gateway; the full catalogue in offline mode. Pass.
4. **`safety-regression` (18 s)** — mod-411 corpus replay against the stubbed-for-CI safety plane. Pass (identical to main).
5. **`cost-latency-smoke` (31 s)** — three representative fixtures against the real model gateway on `paved-road-balanced`. Pass.
6. **`attribute-contract` (6 s)** — one offline turn, check attribute contract. Pass.
7. **`build-and-emit` (4 s)** — emit the `AgentRevision` manifest and the receipt into the artefact bundle.

Total: ~103 s wall-clock on a warm runner; ~160 s cold. The PR comment appears; the reviewer approves on the shape visible in the comment.

Now a different PR that drops the `after_turn` super-call on an override: `lint` passes; `unit`'s hook-order test passes; `attribute-contract` fails — a `platform.safety.record.id` attribute is missing from one span. The engineer reads the failing contract-test output, re-adds `super()`, and the PR goes green. Chapter 07's office hours sees this as a common new-hire mistake; the paved-road doc pins the pattern.

## Summary

- The paved-road CI is a reusable workflow imported with one line from tenant repos (GitHub Actions / GitLab CI / Buildkite). Five gates: lint, unit, eval regression, safety regression, cost / latency smoke; plus the `AgentRevision` manifest emission and the attribute-contract test.
- The attribute-contract test is what keeps the SDK's instrumentation library trustworthy — it drives a real turn through a real collector and asserts the semantic conventions the manifest declared are actually emitted.
- The standardised instrumentation hook library (`paved_road_agent.instrumentation`) is the one place `gen_ai.*` and `platform.*` attributes originate. Attribute names are imported from the semconv package; records (`ReliabilityRecord`, `SafetyRecord`, `EvalRecord`) are emitted via first-class functions.
- Tenant overrides have three shapes: add a gate (freely), loosen a gate (only via exception-register entry), change a gate (must go through the paved-road CI repo).
- PR annotations are a pinned shape: pass/fail per gate, deltas vs. main, manifest summary, artefacts. The reviewer sees the same thing on every tenant's PR.

The next chapter moves from the mechanism to the measurement — the four adoption KPIs the CNCF Platform Engineering Maturity Model scores Interfaces and Adoption against, how to compute them from the manifest stream, and how to tell adoption-by-mandate from adoption-by-pull.
