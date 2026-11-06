# The Local-Dev Loop: Sandbox Emulation, Trace Viewer, Offline Eval Replay

Chapter 03 got the engineer a repo. This chapter gets them a *feedback loop* — a laptop-only, offline-capable, ninety-second environment that lets them run the agent, see the trace, replay an eval, and iterate without going to the cluster.

The claim of the paved road is not "we run your agent in production". The claim is "the shape you write locally is the shape that ships". The local-dev loop is where that claim is paid: it has to mirror the production composition closely enough that the engineer's feedback is honest, but it has to run entirely on their laptop so the feedback is fast.

Three loops have to work:

1. **The run loop** — "write code, run one turn, see output". Sandbox emulation for mod-404; a local trace viewer against mod-408; a local model (either a mocked one or a small OSS model) against mod-406.
2. **The trace loop** — "inspect the spans the last turn emitted". The engineer sees exactly the `gen_ai.*` and `platform.*` attributes CI will check against (chapter 05).
3. **The eval loop** — "replay the fixture catalogue offline and see the verdict". mod-407 regression fixtures, the mod-407 judge, no cluster, no network.

If any of the three is missing, the engineer either does not get the feedback (runs blind, surprises in CI) or they get the feedback in a way that does not match production (loop works, but the shape is subtly different from what CI runs). The ninety-second budget protects the *likelihood* that the engineer uses the loop — a thirty-minute cold-start loop is a loop no one runs.

## What the loop must not do

Name the anti-shapes first — they are how most "local dev" stories go wrong.

- **It must not require the production cluster.** The whole point is no network to the platform's own infrastructure. VPN, cloud-provider login, bastion jump — any of these and the loop fails the ninety-second budget on first run.
- **It must not require production credentials.** Real model-provider API keys, real OTel collector endpoints, real MCP tool secrets — none of these. Mocks and emulators, with the real plane boundaries unchanged.
- **It must not be a different composition from production.** The order in which mod-411 input filter → mod-406 call → mod-411 tool hook → mod-404 execute runs locally must be the exact order chapter 02 diagrammed. The emulators sit on the same seam the production services sit on.
- **It must not be an afterthought.** A loop that lives in the template as "optional, for development" is a loop that drifts out of sync with production. The paved-road CI (chapter 05) runs the local-dev loop against the sample fixtures *as part of the pipeline*, so drift surfaces on every PR.

## The compose topology

The reference stack brought up by `make dev` (or `platform dev`), using Docker Compose as the default orchestrator. Devcontainers, [Dagger](https://dagger.io/), [DevPod](https://devpod.sh/), or the Nix-based `devenv` are drop-in replacements where the organisation already runs them.

```yaml
# docker-compose.dev.yml  — scaffolded by the template; engineer typically does not edit
services:
  agent:
    build: .
    command: paved-road dev --loop
    environment:
      PAVED_ROAD_MODEL_GATEWAY_URL: http://model-gateway:7301
      PAVED_ROAD_MCP_GATEWAY_URL: http://mcp-gateway:7303
      PAVED_ROAD_SANDBOX_URL: http://sandbox:7304
      PAVED_ROAD_SAFETY_URL: http://safety:7311
      OTEL_EXPORTER_OTLP_ENDPOINT: http://otel-collector:4317
    depends_on: [model-gateway, mcp-gateway, sandbox, safety, otel-collector]
  model-gateway:
    image: ghcr.io/platform/local-mocks/model-gateway:2.4.0
    command: --mode mock --profile paved-road-balanced
  mcp-gateway:
    image: ghcr.io/platform/local-mocks/mcp-gateway:2.4.0
    command: --mode local --tools /tools-registry
    volumes: [./tools-registry:/tools-registry:ro]
  sandbox:
    image: ghcr.io/platform/local-mocks/sandbox-emulator:2.4.0
    command: --mode emulator
    # Firecracker/gVisor are not available on every laptop; the emulator provides the SAME API against a safer isolation primitive.
  safety:
    image: ghcr.io/platform/local-mocks/safety-plane:2.4.0
    command: --mode stub --policy-pack paved-road@2026-10-01 --engine regex
  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    command: ["--config=/etc/otelcol/config.yaml"]
    volumes: [./otel/dev-collector.yaml:/etc/otelcol/config.yaml:ro]
  trace-viewer:
    image: jaegertracing/all-in-one:latest
    # or tempo + grafana; or openobserve; or hyperdx-local — pick what the platform standardises on
    ports: ["16686:16686"]
```

Six services, every one of them a mock or emulator that speaks the *production* seam. The agent code is identical to what it will be in production — the only difference is the URLs and the model-provider credentials. Chapter 05's attribute-contract test runs against the trace output from this stack, which is precisely what CI will verify.

### The six seams

| Seam | Local service | Production equivalent | What stays the same |
|---|---|---|---|
| Model completion | `model-gateway` mock — returns a deterministic stub answer or a lightweight model like [Ollama](https://ollama.com/) / [LM Studio](https://lmstudio.ai/) / [`llama.cpp` server](https://github.com/ggml-org/llama.cpp) | mod-406 model gateway | The HTTP / gRPC API, the `gen_ai.*` span attributes, the completion shape, the `paved-road-*` profile names. |
| Tool resolution | `mcp-gateway` local — reads a filesystem registry; invokes local processes or returns stub responses | mod-403 MCP gateway | The tool identifier set, the resolution API, the invocation shape, the tool-health signal. |
| Code execution | `sandbox-emulator` — gVisor-less emulator; executes Python/JS in a restricted process | mod-404 sandbox | The sandbox client API, the policy decisions (egress deny, filesystem scope), the `platform.sandbox.*` attributes. |
| Safety plane | `safety-plane` stub — regex-only input filter, Pydantic output validator, rule-table tool-invocation hook | mod-411 safety plane | The three hook endpoints, the `SafetyRecord` shape, the refusal templates. |
| Observability | OTel collector local — same binary as production, different exporter config | mod-408 collector tier | The semantic conventions (`gen_ai.*`, `platform.*`), the trace propagation, the attribute namespaces. |
| Policy plane | (optional) local OPA with the paved-road policy bundle; usually a trivial stub | mod-405 policy plane | The decision API; the identity-and-permission contract. |

The agents's imports do not change between local and production — the adapter (chapter 02) resolves the URLs at runtime from environment variables. This is the paved-road claim: the loop is the shape.

## The three loops in detail

### Loop 1 — The run loop

```
$ make dev
docker compose up --wait      (25s cold-start, <5s warm-start)
paved-road dev --loop
> listening on http://localhost:7333

$ curl -X POST http://localhost:7333/turn \
    -H 'content-type: application/json' \
    -d '{"user_message": "summarise fixture://golden-path"}'
{"trace_id": "4bf92f3577b34da6a3ce929d0e0e4736", "response": "..."}
```

Four things that are load-bearing here:

1. **The CLI exposes a `POST /turn` endpoint** so the engineer can drive the agent from `curl`, the paved-road CLI (`paved-road turn`), Postman / Bruno, or an in-IDE REST client. The paved-road claim is that the agent is a *service*; running it as a service locally matches production.
2. **The endpoint returns the `trace_id` first.** The engineer copies it into the trace viewer (loop 2) immediately. This is the mod-408 obligation ("every turn produces a trace"); the local-dev loop proves it.
3. **The agent resolves fixture URIs.** `fixture://golden-path` loads a fixture from `fixtures/regression/golden-path.jsonl`; `fixture://smoke/writer/01` loads from the shared fixture catalogue cache. This lets the engineer drive the agent against *the same inputs CI will drive it against*.
4. **Hot reload.** The template's `paved-road dev --loop` watches the `src/` tree and re-imports on save. The engineer edits `agent.py` or `prompts/*.md`; the next `POST /turn` picks up the change. The budget is "first turn in ninety seconds" cold; "turn-after-edit in under five seconds" warm.

### Loop 2 — The trace loop

Open `http://localhost:16686` (Jaeger) or the platform's preferred trace UI. The last turn's trace is visible by `trace_id`; the span tree matches the production shape.

```
turn:writer-v1                            248 ms
├─ safety.input_filter                     12 ms   verdict=allow
├─ gen_ai.openai.responses.create         210 ms   model=claude-3-5-sonnet
│   ├─ gen_ai.tool_call: web.search        45 ms
│   │   ├─ policy.check                     3 ms   allow
│   │   ├─ safety.tool_invocation           5 ms   verdict=allow
│   │   └─ mcp.resolve + invoke            34 ms
│   └─ gen_ai.message.final                 2 ms
├─ safety.output_validator                 10 ms   verdict=allow
└─ reliability.record                       1 ms   turn_outcome=success
```

Four things the engineer uses the trace viewer for:

1. **Attribute verification.** Click on `gen_ai.openai.responses.create`; see the span attributes (`gen_ai.request.model`, `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`); confirm the attribute names match [OpenTelemetry Semantic Conventions for Generative AI](https://opentelemetry.io/docs/specs/semconv/gen-ai/). This is the attribute-contract CI will test; the local viewer is the preview.
2. **Hook composition verification.** The trace's span order is the chapter-02 composition order; a missing span is a missing hook.
3. **Latency observation.** The engineer sees cold-start vs. warm-start latencies locally. "Why is `safety.input_filter` taking 400 ms on first turn?" — because the regex compile happened on the first call; the local trace is where the engineer discovers it before shipping.
4. **Record cross-references.** Click on `safety.input_filter`; the span's `safety.record.id` attribute links to the `SafetyRecord` emitted by the stub safety plane. mod-411's chapter 05 obligation (the record exists per verdict) is visible in the local viewer.

### Loop 3 — The eval loop

```
$ make eval
paved-road eval --fixture-dir fixtures/regression --mode offline
Loading fixtures from fixtures/regression/
  - golden-path.jsonl               12 cases
  - safety-regression.jsonl         4 cases
Running against local stack (http://localhost:7333) ...
Replay complete:
  golden-path:          12/12  passed
  safety-regression:    4/4    passed
  cost-per-session:     avg 0.014 USD (budget 0.050 USD)  [passed]
  latency p95:          1.8 s (budget 3.0 s)              [passed]
Verdict: PASS
Report written to .platform/eval-reports/2026-11-06T14-22-03.json
```

The eval loop reads *the same fixtures CI reads*. Chapter 05's eval-regression gate is this command with `--mode strict` and the budget enforcement turned on. If `make eval` passes locally, CI's eval-regression gate passes; if it fails locally, the engineer can iterate.

Four things to pay attention to:

1. **Offline mode.** The `--mode offline` flag means: do not call real models. The stub model gateway returns the deterministic response the fixture declared; the stub safety plane applies the regex verdicts. The engineer gets a *shape* verdict, not a *quality* verdict. Quality verdicts (did the real model answer well?) are a CI job against the real model gateway, which chapter 05 wires.
2. **Budget enforcement.** The report shows cost and latency against budgets declared in the fixture file. The CI gate (chapter 05) enforces these; the local loop surfaces them.
3. **Report shape mirrors CI.** The `.platform/eval-reports/*.json` file is the same shape CI writes. The engineer who inspects a report locally is reading the same thing a PR comment will later include.
4. **Promotion seam to mod-407.** When a reviewer flags an output locally, the paved-road CLI's `paved-road fixture promote --from-trace <trace_id>` command writes a new fixture entry (mod-410 chapter 06's shape) that can then be pushed into the mod-407 catalogue via its own PR. The local-dev loop is where the promotion starts.

## The sandbox emulator — deeper look

Chapter 01 lists sandbox emulation as one of the three local-dev-loop capabilities. Production sandbox (mod-404) is a serious piece of infrastructure — gVisor or Firecracker or an equivalent, with real network egress controls and resource limits. The emulator cannot be that; what it can be is an API-compatible stand-in.

The emulator ships with four behaviours:

1. **Same HTTP API.** `POST /v1/sandbox/exec` with the same request / response shape as production.
2. **Fake isolation, real policy.** The emulator runs code in a subprocess (not in a VM); the isolation is weak, but the *policy* is real — egress deny, filesystem scope, resource ceilings are enforced the way production enforces them. An attempt to exfiltrate data over HTTP returns the same error locally as it does in production.
3. **Deterministic mode.** When the fixture declares a deterministic execution (common for eval replay), the emulator returns a canned response from the fixture rather than running the code. This is what makes the eval loop fast.
4. **Observability pass-through.** The emulator emits the same `platform.sandbox.*` span attributes production does, so chapter 05's attribute-contract test passes.

A template that relies on the sandbox for anything substantial ships with at least one fixture that drives the sandbox path; the engineer's first `make eval` exercises it.

## The model-gateway mock

The model-gateway mock is where the honesty story matters most. A mock that only returns `"ok"` is a mock whose feedback is useless. Three modes for the mock:

| Mode | What happens | When to use it |
|---|---|---|
| `--mode stub` | Returns a deterministic response per prompt hash. | Fastest cold-start; first-turn demo; smoke tests. |
| `--mode fixture-replay` | Returns the response the fixture declared for this trace. | Offline eval replay; the deterministic branch of CI. |
| `--mode local-model` | Routes to a local [Ollama](https://ollama.com/) / [LM Studio](https://lmstudio.ai/) / [`llama.cpp` server](https://github.com/ggml-org/llama.cpp) running on the laptop. | When the engineer wants a *real* model's behaviour but offline and free. |

The adapter sends `gen_ai.request.model: "paved-road-balanced"` and lets the gateway resolve it. Locally, `paved-road-balanced` maps to the stub; in CI's eval-quality job, it maps to the real model behind the real model gateway; in production, it maps to the configured provider. The agent code does not change.

The `--mode local-model` path is what most engineers want for exploratory work. The template's `docker-compose.dev.yml` keeps it commented out by default (it needs the engineer to have Ollama installed); uncommenting it and running `make dev` picks up the model.

## The safety-plane stub

The safety stub is the one that is most tempting to over-build locally. Resist: the stub exists to exercise the *composition*, not the real defences.

The stub's behaviours:

1. **Input filter** — regex on the user input against the paved-road's policy pack's "known-pattern" list; verdict `allow` / `refuse` based on hit.
2. **Output validator** — Pydantic schema validation if the adapter declares structured output; otherwise `allow`.
3. **Tool-invocation hook** — a tiny rule table loaded from `.platform/safety-rules.local.yaml`, with rules like "email.send to a non-allowlist address returns require_human_approval".
4. **`SafetyRecord` emission** — writes records to `.platform/safety-records.jsonl` so the engineer can inspect them.

Three reasons to keep it lightweight:

- The real engines (Llama Guard, ShieldGemma, NeMo Guardrails) take substantial resources — running them on a laptop breaks the ninety-second budget.
- The *interesting* feedback the stub gives is "did the agent *compose* the hooks". The real quality of the defences is validated in CI against the real engines.
- A heavyweight local stub is a stub engineers will disable, which defeats the purpose.

The template's `safety-rules.local.yaml` ships with a handful of real-sounding-but-benign rules; chapter 05's CI step validates against the real rule set from the production policy-pack repo.

## The trace viewer — which one to pick

The paved road picks one trace viewer and standardises. Options at the time of writing:

| Viewer | Strength | Cost |
|---|---|---|
| [Jaeger](https://www.jaegertracing.io/) — all-in-one container | Mature, well-known, zero config | Only traces; no metrics |
| [Grafana Tempo](https://grafana.com/oss/tempo/) + Grafana | Industry standard; trace / log / metric correlation | Heavier to run locally |
| [Hyperdx](https://www.hyperdx.io/) local | Modern, trace / log / metric in one UI; OTLP-native | Less widely known |
| [SigNoz](https://signoz.io/) | Self-hosted observability stack; OTLP-native | More memory-hungry than Jaeger |
| [OpenObserve](https://openobserve.ai/) | Lightweight; good for a laptop | Newer; some rough edges |

Pick one, pin its image version, and ship it in `docker-compose.dev.yml`. The engineer never sees the choice; they see "the trace viewer at port 16686". Changing it later is a template revision (chapter 03 / 07).

## Devcontainers, DevPod, and the cloud-dev alternative

Some teams cannot run Docker Compose locally (Windows + corporate restrictions, M-series Mac without Rosetta, laptops without enough RAM). Two paved-road escape hatches:

1. **[Devcontainers](https://containers.dev/)** — the `docker-compose.dev.yml` doubles as a devcontainer definition; VS Code / JetBrains / `devcontainer` CLI brings the stack up inside a container; the engineer edits inside the container. The ninety-second budget is harder to hit on a devcontainer cold-start, but warm iterations are the same.
2. **[DevPod](https://devpod.sh/) / [Coder](https://coder.com/) / [Gitpod](https://www.gitpod.io/) workspaces** — the stack runs in a cloud dev environment the engineer connects to. The ninety-second budget still applies, but on a remote host with a cached image it is easier to hit.

The template ships the devcontainer definition alongside the Compose file. The engineer picks; the paved road does not care which one, as long as the *shape* is the same.

## Common pitfalls when authoring the local-dev loop

- **Mocks that have drifted from production.** A mock whose API shape changed without the production shape changing is a mock the paved-road claim cannot make. CI must run a periodic contract test between the mocks and the production services; chapter 05's advisory gates include one.
- **The loop requires the engineer to export secrets.** If the first `make dev` fails because `OPENAI_API_KEY` is unset, the mock is not really a mock. Use fake keys the stub model gateway accepts.
- **The loop uses a different model than production.** Even in `--mode stub` the identifier should be `paved-road-balanced`, not `"gpt-4o-mini"`. The *name* is part of the paved road.
- **Trace propagation broken locally.** If the agent's span tree does not include the model-gateway span or the mcp-gateway span, trace propagation is broken. The local loop surfaces this failure — if it does not, the engineer ships a production-broken configuration. The template's smoke test includes "trace has ≥N child spans".
- **The eval replay only runs locally.** If `make eval` passes locally but `--mode strict` in CI fails, the local mode is doing something CI's mode does not. Keep the mode flag the only difference; if you need more differences, name them in the template README.
- **The `docker-compose.dev.yml` is edited by tenants.** The paved road ships the Compose file; tenant-specific overlays go in `docker-compose.override.yml`. Chapter 03's safe-to-regenerate rule applies.
- **The stub safety plane has no refusal paths.** A stub that always returns `allow` means the composition is never exercised. Ship at least one refusal-producing fixture (`fixtures/regression/safety-prompt-injection.jsonl`); the engineer's first `make eval` sees a refusal.
- **Ignoring the Windows / ARM case.** The paved road's Docker images are multi-arch; the model-gateway mock's local-model path is documented for M-series Mac and Windows WSL2. One architecture-specific bug sinks adoption.

## A worked example: the ninety-second walkthrough

What chapter 03's smoke test actually runs:

```bash
#!/usr/bin/env bash
# templates/agent-chat-anthropic/tests/smoke.sh  — invoked by the catalogue's CI

set -euo pipefail
START=$SECONDS

# 1. Scaffold against the template
copier copy . /tmp/writer-v1 --defaults --trust \
  -d agent_slug=writer-v1 -d tenant_id=acme-financial -d tier=T1

cd /tmp/writer-v1

# 2. Bring up the dev stack
docker compose -f docker-compose.dev.yml up --wait --quiet-pull

# 3. First turn
curl -s --fail -X POST http://localhost:7333/turn \
  -H 'content-type: application/json' \
  -d '{"user_message": "ping"}' | jq -e '.trace_id'

# 4. Eval replay
make eval

# 5. Budget check
if (( SECONDS - START > 90 )); then
  echo "FAIL: ninety-second budget exceeded (${SECONDS}s)"
  exit 1
fi

echo "OK: ninety-second budget met (${SECONDS}s)"
```

Chapter 01's third obligation enforced in a shell script. Every template's smoke test runs this; a template that fails it does not ship. The ninety-second number is a real number, and `time-to-first-agent` (chapter 06) is instrumented against the same clock.

## Running without Docker

Some environments preclude Docker (desktop licensing, corporate restrictions, nested-virt limits). The paved road offers two plan-B shapes:

1. **A `nix` / `devenv` flake** — services run as native processes under `nix develop`. The services are the same binaries; the orchestration is different.
2. **A `platform dev --embedded` mode** — the paved-road CLI runs the mocks as in-process goroutines (or Python threads) inside the agent process. This is the lightest-weight shape but loses the "same seams as production" property for anything that needs real HTTP boundaries.

Both are documented as paved-road alternatives, not as the primary path. The primary path is Compose.

## Summary

- The local-dev loop has three sub-loops: the run loop (`make dev` + `POST /turn`), the trace loop (trace viewer on port 16686), and the eval loop (`make eval` against the fixture catalogue). All three must work on a laptop with no VPN and no cloud credentials.
- The reference stack is Docker Compose with six services (agent, model-gateway mock, mcp-gateway local, sandbox emulator, safety-plane stub, OTel collector) plus a trace viewer. Every service speaks the production API; only the implementation is a mock.
- The sandbox emulator provides API parity against a safer isolation primitive; the model-gateway mock has three modes (stub, fixture-replay, local-model); the safety-plane stub is deliberately lightweight to protect the ninety-second budget.
- Three alternatives for environments that cannot run Compose: devcontainers, DevPod / Coder / Gitpod cloud workspaces, Nix / `devenv` flakes. All ship with the template.
- Common pitfalls: mocks drifting from production, loops that need production credentials, trace propagation broken locally, stub safety plane with no refusal paths, over-building the local stubs. Chapter 05's CI makes drift detectable; the ninety-second smoke test makes slippage visible.

The next chapter opens the CI template and the standardised instrumentation hook library — the pair that turns "the local loop passes" into "the production shape holds under review".
