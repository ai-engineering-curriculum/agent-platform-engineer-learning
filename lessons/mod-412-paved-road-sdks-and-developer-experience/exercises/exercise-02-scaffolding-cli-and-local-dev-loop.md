# exercise-02: Scaffolding CLI and Local-Dev Loop

**Estimated effort:** 2 hours · **Output:** a `platform` (or `aicg`) CLI wrapping Copier with `new`, `update`, `build`, `doctor`, and `dev` subcommands; a Docker Compose definition for the local-dev loop with at least four mocks (model gateway, MCP gateway, sandbox emulator, safety plane); a passing ninety-second smoke test; a short README (~1 page) · **Pairs with:** chapters [03](../03-scaffolding-cli-and-template-catalogue.md), [04](../04-local-dev-loop-sandbox-emulation-trace-replay.md), and [05](../05-ci-templates-and-standardised-instrumentation-hooks.md); feeds projects 401 and 402.

## Objective

Translate chapter 03's scaffolding-CLI design and chapter 04's local-dev-loop topology into a working engineer-facing tool. You will build the CLI that stream-aligned teams type first (`platform new agent`), stand up the Compose-based local-dev loop the scaffolded repo boots, and verify the ninety-second claim from chapter 01 obligation 3.

Chapter 03 fixes the shape; chapter 04 fixes the loop; this exercise is where you discover the fifty small decisions the chapters elided — which flag turns off interactive prompts, how to pass through tenant-specific overlay files, what `platform doctor` prints on a repo with a stale template. The engineer who has never written the CLI wrapper under-estimates by a factor of three the cost of an honest first-run experience.

## Prerequisites

- You have completed [exercise 01](exercise-01-paved-road-sdk-template-authoring.md) (or have an equivalent `paved_road_agent` package and at least one Copier template to point at).
- You have read chapters [03](../03-scaffolding-cli-and-template-catalogue.md), [04](../04-local-dev-loop-sandbox-emulation-trace-replay.md), and skimmed [05](../05-ci-templates-and-standardised-instrumentation-hooks.md).
- You have Docker + Docker Compose installed locally. If Docker is unavailable, a [Podman Compose](https://podman.io/) substitute is acceptable; you MUST annotate this in the README.
- You have Python 3.11+ and either [`click`](https://click.palletsprojects.com/) / [`typer`](https://typer.tiangolo.com/) for the CLI, or you can implement it in Go / Rust if your platform prefers (the exercise does not require Python).
- You have a trace viewer image available locally. [Jaeger all-in-one](https://hub.docker.com/r/jaegertracing/all-in-one) is the paved-road default.

## Problem statement

A product engineer joins the `acme-financial` team and is told: "scaffold your first paved-road agent". They open their laptop, type `platform new agent`, and expect to be running against a live local trace within ninety seconds.

The output has four parts:

1. The `platform` CLI with at least five verbs: `new`, `update`, `build`, `doctor`, `dev`.
2. The `docker-compose.dev.yml` the template emits, with the four mocks running.
3. A passing ninety-second smoke test.
4. A short README (~1 page) with trade-offs, two "I was surprised by" observations, and one open question for the next DX review.

## Requirements

### Part 1 — The `platform` CLI

Five subcommands, at minimum:

**`platform new agent`**
- Reads a template catalogue index (a local YAML file is fine for this exercise — the `catalog.yaml` shape from chapter 03).
- Interactive by default; `--template <id>` and `--values key=value...` for non-interactive invocation.
- Calls `copier copy` with the right arguments.
- Writes `.platform/generation-receipt.json` to the scaffolded repo with the chapter-03 shape.
- Prints a next-step summary that names the paved-road commands (`make dev`, `make eval`, `make build`).

**`platform update`**
- Calls `copier update` in the current repo.
- Verifies the new template version is within the SDK's compatibility range (from `.copier-answers.yml`).
- Prints the diff summary and the recommended `platform doctor` invocation after.

**`platform build`**
- Reads `.platform/manifest-template.yaml`, the exercise-01 `build_manifest(...)`, and a stub CI-data file.
- Writes `.platform/agent-revision.yaml`.
- Returns non-zero if any required manifest field is missing.

**`platform doctor`**
- Reads the current repo and prints the chapter-03 diagnostic block: template version (and whether an update is available), SDK version, adapter version, attribute-contract status (stub: "passing" if the attribute-contract test file is present), CI config (stub: "paved-road reusable workflow detected" if `.github/workflows/paved-road.yml` exists), exception-register entries (empty or listed from `.platform/exception-overrides.yaml`), local-dev-loop status (whether `docker-compose.dev.yml` is syntactically valid).
- Non-zero exit code when any required artefact is invalid.

**`platform dev`**
- `docker compose -f docker-compose.dev.yml up --wait`.
- Starts the agent's own process (either as a subcommand of the paved-road CLI or by invoking a project-specific entry point).
- Prints the ingress URL and the trace-viewer URL.

All subcommands write a one-line log to stderr with ISO-8601 timestamps so a smoke-test harness can parse their output.

### Part 2 — The local-dev Compose definition

Create `docker-compose.dev.yml` in the scaffolded template's output. Four mocks at minimum:

- **`model-gateway` mock** — a FastAPI / Flask / Node service that exposes a `/v1/completions` endpoint returning a deterministic stub response on a paved-road profile name. Mode flag `--mode stub`.
- **`mcp-gateway` mock** — a service that accepts a tool-resolution POST and returns a canned tool metadata plus an invocation endpoint that returns a canned response. Reads a local `tools-registry/` directory of YAML files that declare tool names and stub responses.
- **`sandbox-emulator`** — a service that accepts a `POST /v1/sandbox/exec` with code + args and either executes it in a `subprocess.run(...)` with a strict allow-list OR returns a fixture-declared response when the request includes a `deterministic_hash` header. This is a stand-in for mod-404; the point is the API shape, not real isolation.
- **`safety-plane` stub** — a service exposing `/v1/input-filter/evaluate`, `/v1/output-validator/evaluate`, `/v1/tool-invocation-hook/evaluate`. Each uses a regex table loaded from `.platform/safety-rules.local.yaml`. Writes a `SafetyRecord` JSON Lines file on every verdict. (Cross-reference the exercise-01 shape from mod-411.)

Plus one observability service:

- **OTel collector** — `otel/opentelemetry-collector-contrib` with a config that exports traces to a local Jaeger instance (or writes JSON to disk for inspection).

Plus the trace viewer:

- **Jaeger all-in-one** on port 16686.

The scaffolded agent's `docker-compose.dev.yml` references all of them with explicit image tags. Each mock must have `--mode`, `--port`, and `--log-level` flags.

### Part 3 — The ninety-second smoke test

A script that reproduces chapter 04's forcing function:

```bash
#!/usr/bin/env bash
# tests/smoke.sh
set -euo pipefail
START=$SECONDS
TEMP=$(mktemp -d)
cd "$TEMP"

platform new agent --template agent-chat-<framework> \
  --values agent_slug=writer-v1 \
  --values tenant_id=acme-financial \
  --values workload_class=interactive-chat \
  --values tier=T1 \
  --values model_profile=paved-road-balanced

cd writer-v1
docker compose -f docker-compose.dev.yml up --wait --quiet-pull
# first turn
RESP=$(curl -s --fail -X POST http://localhost:7333/turn \
  -H 'content-type: application/json' \
  -d '{"user_message": "ping"}')
echo "$RESP" | jq -e '.trace_id' >/dev/null

ELAPSED=$((SECONDS - START))
echo "elapsed=${ELAPSED}s"
if (( ELAPSED > 90 )); then
  echo "FAIL: ninety-second budget exceeded (${ELAPSED}s)"
  exit 1
fi
echo "PASS: ninety-second budget met (${ELAPSED}s)"
```

Run it; capture the output; attach the resulting log file (`smoke.log`) to the deliverable. Note: cold Docker pulls will exceed ninety seconds on first run; the budget is a *warm-cache* budget. The smoke test must document that assumption and either (a) pre-pull images in a separate step or (b) warm the cache on first invocation and re-run.

### Part 4 — README (~1 page)

- The CLI framework / language you chose (one sentence).
- The five mock services and the trade-offs you made (which were trivial stubs vs. which had policy).
- Two "I was surprised by" observations — e.g., "`copier update`'s three-way merge produced a surprising conflict on `.copier-answers.yml` itself; we solved it by excluding that file from the user-owned category". Or: "the Jaeger all-in-one image is 300MB and dominated cold-start; we switched to [Hyperdx-local](https://www.hyperdx.io/) and the budget held".
- One open question — e.g., "the `platform doctor` command currently treats the attribute-contract status as pass/fail; should it also show a historical trend (recent runs) so a reviewer can see drift?".

## Starter guidance

- **Stand up the mock services first.** Without the mocks the CLI has nothing to point at. A single-file FastAPI service per mock is enough; keep them under 50 lines each.
- **Pre-pull images in CI.** The ninety-second budget is realistic only on a warm cache. Your smoke-test harness should pull the images once (or include a `platform warmup` subcommand) and then run the clock.
- **Use `click`'s `pass_context` or `typer`'s dependency injection** to keep subcommands composable. `platform dev` might call `platform build --no-write` internally.
- **Pin image tags in `docker-compose.dev.yml`.** `latest` is where reproducibility goes to die; the paved-road ships with pinned tags and updates them on template rev.
- **Keep the safety-plane stub's rule set small and real.** A regex table of three patterns is enough — "JAILBREAK", a PII pattern like an SSN, and `file://` URLs in tool arguments. Resist the temptation to re-implement NeMo Guardrails locally.
- **Instrument the scaffolded agent with the OpenTelemetry Python SDK's OTLP exporter.** Point `OTEL_EXPORTER_OTLP_ENDPOINT` at the local collector; set the service.name to the agent slug.
- **Write the smoke test last.** It is the forcing function but also the acceptance criterion; write it once the pieces run.
- **Mark `<!-- needs-research: ... -->` on anything you cannot verify.** The exact Jaeger image tag, the OTel collector config fields, the FastAPI version pins all move.

## Acceptance criteria

Mark done when:

- [ ] `platform new agent --template <id> --values ...` scaffolds a runnable repo in under 10 seconds (clock from CLI start to prompt return).
- [ ] `platform update` is a working wrapper over `copier update` and surfaces the version transition.
- [ ] `platform build` emits `.platform/agent-revision.yaml` with the exercise-01 schema validated.
- [ ] `platform doctor` prints the diagnostic block from chapter 03 and exits non-zero on at least one intentional failure case (e.g., manifest missing).
- [ ] `platform dev` brings up the Compose stack and the scaffolded agent process in under 45 seconds on a warm-cache machine.
- [ ] `docker-compose.dev.yml` runs all five services (model-gateway mock, mcp-gateway mock, sandbox emulator, safety-plane stub, OTel collector + Jaeger) with pinned image tags.
- [ ] `curl POST /turn` returns a JSON body with a valid `trace_id`.
- [ ] Opening the trace viewer URL shows the turn's span tree with at least five spans (safety input, gen_ai completion, at least one tool call, safety output, reliability record) in the chapter-02 order.
- [ ] `tests/smoke.sh` executes end-to-end in under ninety seconds on a warm-cache machine; `smoke.log` is attached.
- [ ] README includes the two "surprised by" observations and the open question.

## Stretch goals

- **Add a `platform exception` subcommand.** Reads / writes `.platform/exception-overrides.yaml` with the chapter-07 shape; `platform exception list` prints open entries.
- **Add a `platform eval` subcommand.** Thin wrapper over the exercise-01 fixture harness; invokes `paved-road eval --mode offline` against the scaffolded repo.
- **Implement the cold-start "warm-up" pathway.** `platform dev --warm` pre-pulls all images and primes the first-turn cache; measure the cold vs. warm delta and document it.
- **Switch the trace viewer.** Try [Hyperdx-local](https://www.hyperdx.io/), [SigNoz](https://signoz.io/), or [OpenObserve](https://openobserve.ai/) instead of Jaeger; report the pros/cons in the README.
- **Add a devcontainer definition.** A `.devcontainer/devcontainer.json` that brings up the same stack inside VS Code's Dev Container feature; verify the ninety-second budget holds (cold-start is harder here).
- **Add a `platform ci simulate` subcommand** that runs the chapter-05 gates locally against the scaffolded repo. Wires exercise-02 into exercise-03's measurement without requiring a real CI runner.
- **Implement the model-gateway's `--mode local-model`** against [Ollama](https://ollama.com/) and verify the loop still holds the ninety-second budget. (Pull a small model like `qwen2.5:0.5b`; the budget is tighter but achievable on a warm cache.)
- **Write a one-page ADR** on why you picked Compose over devcontainers / DevPod / Nix-based `devenv`. Chapter 04 lists the alternatives; your ADR justifies the paved-road default.
