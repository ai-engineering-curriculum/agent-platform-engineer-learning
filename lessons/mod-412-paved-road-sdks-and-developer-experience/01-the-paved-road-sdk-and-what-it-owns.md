# The Paved-Road SDK and What It Owns

The runtime (mod-402) exposes a session contract. The MCP gateway (mod-403) catalogues tools. The sandbox (mod-404) runs code. The policy plane (mod-405) authorises. The model gateway (mod-406) routes completions. The evaluation plane (mod-407) catalogues fixtures. The observability plane (mod-408) fixes the semantic vocabulary. The inter-agent plane (mod-409) carries A2A edges. The SRE plane (mod-410) runs the fleet. The safety plane (mod-411) owns the three hook invocations around every turn.

Every one of those planes stops short of a question the platform still has to answer: *how does a stream-aligned product team adopt all eleven planes without wiring eleven integration points into every agent repo by hand, and how does the platform know whether its paved road is actually being used?* A product engineer who has to read eleven READMEs and copy eleven integration stanzas before writing their first `def run(...)` line is going to fork a sample repo and drift off the road the first time a plane rev changes. The planes beneath this one give you the *capabilities*; this module gives you the *shape* in which those capabilities become easy to adopt and hard to bypass.

This chapter fixes the slice this module owns. The chapters that follow pick the pieces: the reference shapes across the five mainstream agent frameworks (02), the scaffolding CLI and template catalogue (03), the local-dev loop (04), the CI templates and standardised instrumentation hooks (05), the adoption metrics and the CNCF maturity frame (06), and the enablement operating cadence that keeps the road the *default* road rather than the *mandated* road (07).

## What the paved-road SDK owns

The plane sits between the eleven planes below it and the stream-aligned teams whose agents consume them. It owns:

- **The `PavedRoadAgent` reference shape.** The interface, the lifecycle hooks, the metadata object, and the one-call `run(...)` entry point every paved-road agent compiles against. Chapter 02 maps this onto the five framework adapters. The platform owns the interface; the adapters wrap framework-native shapes to implement it.
- **The framework adapters.** One adapter per mainstream framework — [Anthropic Agent SDK](https://docs.anthropic.com/en/api/agent-sdk/overview), [OpenAI Agents SDK](https://openai.github.io/openai-agents-python/), [Google Agent Development Kit](https://google.github.io/adk-docs/), [Microsoft Agent Framework](https://learn.microsoft.com/en-us/agent-framework/overview), and [LangGraph](https://langchain-ai.github.io/langgraph/) on [Temporal](https://docs.temporal.io/). Each adapter is a thin, well-versioned shim whose job is "take framework X's native shape and expose it as a `PavedRoadAgent`". Chapter 02.
- **The scaffolding CLI.** A single command (`platform new agent`, `aicg new`, whatever your naming convention is) that generates a complete, runnable repo from a named template — framework choice, workload class, tier, and policy pack supplied as flags or interactive prompts. Chapter 03 names the three template-engine options ([Cookiecutter](https://cookiecutter.readthedocs.io/), [Copier](https://copier.readthedocs.io/), [Backstage Software Templates](https://backstage.io/docs/features/software-templates/)) and the per-workload-class catalogue.
- **The template catalogue.** A versioned, GitOps-managed catalogue of templates — one per `{framework, workload_class, tier}` cell where the matrix is interesting. Each template ships: starter code, a sample fixture (mod-407), a sample `SessionState` wiring (mod-402), a `manifest.yaml`, the CI config (mod-410 / mod-411 gates), a local-dev-loop `Makefile`, and a `README.md` whose first ninety seconds are "clone, `make dev`, see the first turn succeed".
- **The local-dev loop.** A set of commands (`make dev`, `make trace`, `make eval`) that give an engineer, on a laptop with no cluster, three concrete feedback loops: sandbox emulation against mod-404's contract, a trace viewer against mod-408's vocabulary, and offline eval replay against mod-407's fixture catalogue. Chapter 04.
- **The CI template.** A reusable pipeline — [GitHub Actions](https://docs.github.com/en/actions/using-workflows/reusing-workflows) reusable workflows, [GitLab CI](https://docs.gitlab.com/ee/ci/yaml/includes.html) `include`, [Buildkite](https://buildkite.com/docs/pipelines/templates) pipeline templates, or whatever the organisation's standard is — that implements the five paved-road gates: lint, unit, eval regression (mod-407), safety (mod-411), cost (mod-406). Chapter 05.
- **The standardised instrumentation hooks.** A library imported by every adapter that fills `gen_ai.*` and `platform.*` span attributes automatically against the mod-408 semantic contract, and emits the required records (`ReliabilityRecord` for mod-410, `SafetyRecord` for mod-411, `EvalRecord` for mod-407). An agent that imports the SDK is instrumented by default; an agent that hand-rolls its instrumentation is off the paved road. Chapter 05.
- **The `AgentRevision` manifest.** A machine-readable record emitted by every scaffolded repo at every build, keyed to a git revision, declaring: the SDK version, the adapter in use, the workload class, the tier, the policy pack, the model-gateway profile, the attached fixtures, and the generation-receipt reference. The manifest is what the adoption KPIs (chapter 06) count over and what the deprecation ladder (chapter 07) compares against. Pinned below.
- **The adoption-KPI emitter and dashboard.** Four metrics — platform adoption rate, time-to-first-agent, deployment success rate, rollback count — with pinned formulas against the manifest and the CI stream. The dashboard belongs to the plane; the review cadence belongs to mod-413. Chapter 06.
- **The enablement operating cadence.** Onboarding sessions, weekly office hours, quarterly deep-dive clinics, migration playbooks, and the deprecation ladder. The cadence is a *product* in the sense of the [CNCF Platforms whitepaper](https://tag-app-delivery.cncf.io/whitepapers/platforms/) — a repeated thing with a release calendar, not a Slack channel. Chapter 07.
- **The exception register.** When a tenant genuinely cannot take the paved road for a documented reason (a framework the SDK does not yet wrap, a workload class the catalogue does not yet cover, a tier whose SLA the paved road cannot hit), the plane records the exception, the owner, and the exit criterion that puts the tenant back on the road. Chapter 07.

The plane does **not** own:

- **The capabilities themselves.** The SDK does not implement sessions (mod-402), tool resolution (mod-403), sandboxing (mod-404), policy decisions (mod-405), model routing (mod-406), eval fixtures (mod-407), the semantic-conventions spec (mod-408), A2A routing (mod-409), SLIs / SLOs (mod-410), or guardrail engines (mod-411). It *composes* those planes behind one call; it does not replace them. A bug in the sandbox is mod-404's bug, not the SDK's.
- **The product-side agent correctness.** Whether the agent's system prompt is clear, whether the model choice is a good fit for the workload, whether the structured-output schema matches the business domain — these are [`senior-agentic-ai-engineer`](../../../senior-agentic-ai-engineer-learning/) (level 40) and [`agentic-ai-engineer`](../../../agentic-ai-engineer-learning/) (level 30) decisions. The SDK is the shape, not the substance.
- **The upstream framework roadmaps.** Anthropic, OpenAI, Google, Microsoft, and the LangGraph / Temporal communities move at their own cadence. The SDK's adapters track their releases; the SDK's job is to insulate stream-aligned teams from each framework's versioning policy, not to influence it.
- **The CNCF maturity model itself.** The model (chapter 06) is a diagnostic frame the SDK uses; the plane does not own the frame. If the CNCF Platforms working group re-revs the model, the SDK's dashboards are updated; the frame is not the plane's artefact.
- **The leadership cadence.** The adoption review, the roadmap sign-off, and the deprecation-ladder final decisions belong to `mod-413`. The SDK produces the evidence; leadership makes the calls.
- **Legal and licensing interpretation.** Whether an upstream framework's licence is compatible with the platform's shipping model, whether a vendor-side adapter can be bundled, whether a tenant's workload crosses an export-control threshold — these are counsel's calls. The plane names the facts; counsel picks the interpretation.

Drift off this slice and the SDK becomes "the people who write the first agent for every product team" — a feature team masquerading as a platform plane. Scope policing is a feature of this module, and chapter 07 is where the policing lives in practice.

## Why a plane and not a per-agent sample repo

A few concrete questions that are hard to answer without a paved-road SDK plane and trivial with one.

- **"The tenant's finance agent and their HR agent both need the three safety hooks (mod-411). Did both wire them the same way?"** Without a plane, the finance team copied sample-agent-v1 and the HR team copied sample-agent-v2; one wrapped the hook invocations in a try/except that swallows failures, the other did not. With one, both agents call `PavedRoadAgent.run(...)` and the hooks are composed behind the interface; the mod-411 obligation 1 ("every paved-road agent routes every seam through the plane") is mechanically satisfied. Chapter 02 and chapter 05.
- **"OpenTelemetry's GenAI semantic conventions rev from v1.37 to v1.38 and the `gen_ai.request.model` attribute is renamed. How many of our agents need code changes, and when?"** Without a plane, the rename is a sweep across every agent repo and a different team deals with it on a different timeline. With the standardised instrumentation hook library (chapter 05), the rename is a point release of the SDK; stream-aligned teams see it as a dependency bump. Cross-reference mod-408 chapter 05.
- **"A VP of Engineering asks, 'how many agents are on the paved road?'. Can we answer?"** Without a plane, the answer is "probably most of them, let me ask around". With the `AgentRevision` manifest (below) and the adoption-KPI dashboard (chapter 06), the answer is a number with a date on it, split by tenant and workload class.
- **"We deprecate Anthropic Agent SDK v1 adapter after the vendor's v2 release. How long until it is safe to delete?"** Without a plane, the answer is "whenever the last team notices"; the delete happens six months late and nobody can tell whether it is safe. With the deprecation ladder (chapter 07), the manifest tells you which agents are still on `v1` on every build, the ladder's policy ties the cutoff to a declared criterion (e.g., zero manifests referencing `v1` for four consecutive weeks), and the delete is a mechanical event.
- **"A senior engineer wants to add a hook the SDK does not expose. Is that a reasonable platform roadmap ask or a one-off exception?"** Without a plane, the senior engineer forks the SDK and the next SDK upgrade is a merge-conflict project. With the exception register (chapter 07), the request is a documented record with an owner and an exit criterion, and the SDK roadmap conversation is open rather than ad-hoc.
- **"A new hire builds their first agent on their laptop. Does it work before the first standup?"** Without a plane, the answer depends on whose Slack messages the new hire read. With the scaffolding CLI and the local-dev loop (chapters 03 and 04), there is a measured, repeated "ninety seconds from clone to first turn" acceptance criterion, and `time-to-first-agent` (chapter 06) is a weekly tracked metric.

These are the capabilities the plane delivers. They separate "we have a sample repo" from "we have a paved-road SDK". Chapters 02–07 build each.

## What makes a paved road for agents different from a paved road for microservices

The platform-engineering literature — [Team Topologies](https://teamtopologies.com/), the [CNCF Platforms whitepaper](https://tag-app-delivery.cncf.io/whitepapers/platforms/), and the [Internal Developer Platform](https://internaldeveloperplatform.org/) community — was written largely around microservice / web-service / data-pipeline paved roads. An agent workload breaks three of that model's assumptions, and chapter 02–05 return to each:

- **The runtime is non-deterministic and long-running.** A microservice paved road can usually assume "the request is stateless, the response fits in one round-trip". An agent workload has a session (mod-402), can yield for hours, can call tools (mod-403), can run code (mod-404), and can emit a different trajectory on identical input. The SDK has to expose a lifecycle (chapter 02's `PavedRoadAgent`) that models yield-and-resume natively, not as an afterthought.
- **The CI gates include eval, safety, and cost — not just lint and unit.** A microservice paved road's "quality gate" is a unit-test suite and maybe a load test. An agent paved road's quality gate also runs the fixture catalogue (mod-407), the safety-regression corpus (mod-411), and a cost-per-session smoke test (mod-406). The five-gate CI template (chapter 05) is the paved-road-specific extension.
- **The "framework" choice is first-class and plural.** A microservice paved road usually assumes one web framework per language (Flask or FastAPI or Spring or Rails). An agent paved road has to span [Anthropic Agent SDK](https://docs.anthropic.com/en/api/agent-sdk/overview), [OpenAI Agents SDK](https://openai.github.io/openai-agents-python/), [Google Agent Development Kit](https://google.github.io/adk-docs/), [Microsoft Agent Framework](https://learn.microsoft.com/en-us/agent-framework/overview), and [LangGraph](https://langchain-ai.github.io/langgraph/) because stream-aligned teams have real reasons (model-provider affinity, prior investment, feature coverage) to pick each. Chapter 02's adapter pattern is how the SDK presents one shape without picking one framework.

The microservice paved-road toolkit — [Backstage](https://backstage.io/), [Cookiecutter](https://cookiecutter.readthedocs.io/), reusable CI workflows, trace viewers — composes on top of those extensions. You are not inventing a new toolkit; you are picking the shapes those tools produce and adding the agent-specific pieces. Chapter 03 names the composition.

## The invariant: `AgentRevision` manifest

Pin the manifest shape before writing the SDK. Every paved-road agent emits one `AgentRevision` record per build, keyed to the git revision, carrying the SDK version and framework adapter it compiled against, the workload class and tier it declares, and the references to every paved-road artefact it consumes. The manifest is the record every adoption KPI (chapter 06) counts over and every deprecation decision (chapter 07) compares against.

```yaml
# .platform/agent-revision.yaml  — emitted by `platform build` into the build artefact
schema_version: agent-revision/v1
generated_at: 2026-11-06T14:22:03Z
agent:
  agent_id: writer-v1
  tenant_id: acme-financial
  git_revision: abcd1234ef567890
  repo: github.com/acme-financial/writer-agent
sdk:
  paved_road_sdk_version: 2.4.1
  adapter: anthropic-agent-sdk
  adapter_version: 2.4.1
  framework_version: 0.3.x  # Anthropic Agent SDK python package version
workload:
  workload_class: interactive-chat
  tier: T1  # resilience tier (mod-410 chapter 02)
  policy_pack: paved-road@2026-10-01  # safety-plane pack (mod-411)
  model_gateway_profile: paved-road-gpt4o-balanced  # (mod-406)
scaffolding:
  template: templates/agent-chat-anthropic
  template_version: 3.2.0
  generation_receipt: receipts/writer-v1-2026-09-14.json
  copier_answers_file: .copier-answers.yml
instrumentation:
  gen_ai_semconv_rev: 1.37.0
  platform_semconv_rev: 2026-10-01
  required_attributes_test: passed
gates:
  lint: passed
  unit: passed
  eval_regression: passed
  safety_regression: passed
  cost_budget: passed
fixtures:
  catalog_pins:
    - mod-407-fixture-catalog@2026-10-28:regression/writer-v1/*
overrides:
  exception_register_entries: []
  # or:
  # - exception_id: ER-2026-0043
  #   reason: framework_not_yet_adapted
  #   expires_at: 2026-12-31
  #   owner: team-financial-writer
```

Seven invariants the manifest enforces:

1. **Every agent names the SDK version, adapter, and framework version it compiled against.** The deprecation ladder (chapter 07) answers "how many agents are still on SDK v1.x" from this field alone.
2. **Every agent names the template and generation-receipt it came from.** The scaffolding CLI's receipt file (chapter 03) is the provenance pointer for "was this agent scaffolded or hand-rolled". `time-to-first-agent` (chapter 06) joins from the receipt's timestamp.
3. **Every agent names the workload class, tier, and policy pack it declares.** The paved road is parametric by `{workload_class, tier}`; the manifest is where the parameters are bound. The safety-plane policy pack (mod-411) and the SRE tier (mod-410) are both looked up on this.
4. **Every agent names the CI gates it passed on this build.** `deployment success rate` (chapter 06) is the number of builds whose manifest shows all five gates passing. A build that skips gates is on the exception register, not on the paved road.
5. **Every agent pins the semantic-conventions rev it instruments against.** When mod-408's `gen_ai.*` or the platform's `platform.*` namespace rev moves, the manifest declares the rev the build targeted. The attribute-contract test (chapter 05) fails the build if the rev does not match the instrumented attributes.
6. **Every agent names its fixture-catalog pins.** The eval-regression gate (mod-407) ran against a named pin; the deprecation of a fixture is a cross-reference against this field.
7. **Overrides are structural, not free-text.** When an agent is operating under an exception-register entry, the entry id, reason, expiry, and owner are first-class fields. The adoption dashboard (chapter 06) splits "fully on paved road" from "on paved road with active exception" on this field.

Chapters 02–07 produce this manifest on every scaffolded build.

## The three obligations

Pin the SDK plane to three non-negotiable obligations. Everything in chapters 02–07 is a decision about how to deliver them.

1. **"Every paved-road agent is on every plane with one call."** A stream-aligned engineer imports the SDK, writes their `run(...)` body, and is wired into the eleven planes by default. A plane the SDK does not compose into `PavedRoadAgent` is a plane the paved-road claim cannot make; chapters 02 and 05 enforce this.
2. **"Every paved-road build emits an `AgentRevision` manifest that the adoption dashboard, the deprecation ladder, and the governance cadence all consume without transformation."** A build without a manifest is not on the paved road, however many planes it integrates behind the scenes; chapter 06 enforces this.
3. **"The ninety-second path from `clone` to first turn is measured on every paved-road release, and a regression against that budget blocks the release."** A paved road a new hire cannot walk in ninety seconds is a paved road nobody walks by choice; chapter 04 enforces this.

A plane that cannot defend all three obligations is a sample repo, not an SDK.

## Where the SDK physically sits

A typical deployment shape:

- **A Python package** (`paved_road_agent`, or the equivalent name the organisation picks) — or one per language the platform supports. It exposes `PavedRoadAgent`, the framework adapters, the `SessionState` adapter against mod-402, the tool-resolution client against mod-403, the model-gateway client against mod-406, the safety-hook clients against mod-411, the A2A client against mod-409, and the standardised instrumentation hook library against mod-408. The package is versioned with a strict SemVer policy (chapter 02's "adapter version" rule). Published to a private registry; tenants pin on major-minor.
- **A scaffolding CLI** (`platform` or `aicg` or the organisation's name) distributed as a binary (built with [PyInstaller](https://pyinstaller.org/) or [Go](https://go.dev/) or whatever the organisation standardises on) *and* as a Python package (for developers who already have Python). Reads a GitOps-managed template catalogue.
- **A template catalogue repository** (a git repo on the platform's SCM) containing one subdirectory per `{framework, workload_class, tier}` cell. Each template is a [Cookiecutter](https://cookiecutter.readthedocs.io/) or [Copier](https://copier.readthedocs.io/) template, or a [Backstage Software Template](https://backstage.io/docs/features/software-templates/) that invokes one. Chapter 03.
- **A reusable CI workflow repository** — one per CI system the platform supports (GitHub Actions, GitLab CI, Buildkite). Stream-aligned teams `uses:` or `include:` the paved-road workflow. Chapter 05.
- **A local-dev-loop shell** — a Docker Compose definition (or `devcontainer.json`, or a `Taskfile`) that brings up the sandbox emulator (mod-404), the local trace viewer (Jaeger or the lightweight [Hyperdx](https://www.hyperdx.io/) option or a `tempo` single-binary), and the offline eval harness against the fixture-catalogue cache. Chapter 04.
- **An `AgentRevision` manifest ingestor** — a small service (or the mod-408 collector tier with a dedicated receiver) that accepts manifest emissions from CI jobs and streams them into the manifest warehouse the adoption dashboard queries. Chapter 06.
- **The adoption dashboard** — one tab on the platform's engineering dashboard (Grafana / Looker / Backstage custom plugin). Queries the manifest warehouse plus the CI-event stream plus the deploy stream. Chapter 06.
- **The enablement cadence artefacts** — office-hours calendar, onboarding guide, migration-playbook template, deprecation-ladder policy document, and the exception register itself. These live in the paved-road documentation site (a [Docusaurus](https://docusaurus.io/), [MkDocs Material](https://squidfunk.github.io/mkdocs-material/), or [Backstage TechDocs](https://backstage.io/docs/features/techdocs/techdocs-overview/) site) and in the ticket system the exception register writes into. Chapter 07.

If you cannot draw this topology on a whiteboard and name the box every agent build flows through, chapter 02 is the thing to read next.

## Where this plane sits in the track

The SDK plane sits downstream of every substrate module and upstream of the leadership module. Each prior module left a hook or a signal the SDK consumes:

- **mod-402 (runtime)** — the `SessionState` interface and the framework adapters (mod-402 chapter 04) are the ground truth the SDK's adapters compose. The SDK's `PavedRoadAgent` interface *extends* the `SessionState` adapter surface.
- **mod-403 (MCP gateway)** — the tool-resolution API. The SDK's tool-call hook imports this client.
- **mod-404 (sandbox)** — the sandbox client and the sandbox emulator used in the local-dev loop (chapter 04).
- **mod-405 (policy plane)** — identity-and-permission decisions. The SDK's tool-call hook invokes the policy plane *before* mod-411's tool-invocation safety hook, as mod-411 chapter 01 requires.
- **mod-406 (model gateway)** — the routing client and the model-profile naming convention.
- **mod-407 (eval plane)** — the fixture catalogue. The local-dev-loop offline replay (chapter 04) and the CI eval-regression gate (chapter 05) both read from mod-407.
- **mod-408 (observability plane)** — the semantic-conventions contract. The standardised instrumentation hooks (chapter 05) fill `gen_ai.*` and `platform.*` attributes against this vocabulary.
- **mod-409 (inter-agent plane)** — the A2A client with its circuit breaker and retry budget.
- **mod-410 (SRE plane)** — the SLI vocabulary, the `ReliabilityRecord` emitter, the burn-rate alert templates, the SLO document template, and the resilience-tier identifier set.
- **mod-411 (safety plane)** — the three hooks (input filter, output validator, tool-invocation policy). The SDK's adapter composes them around every turn, satisfying mod-411 chapter 01 obligation 1 by construction.

The downstream handoffs:

- **[`agentic-ai-engineer`](../../../agentic-ai-engineer-learning/) (level 30)** — the primary consumer. Imports the SDK, fills in `run(...)`, ships.
- **[`senior-agentic-ai-engineer`](../../../senior-agentic-ai-engineer-learning/) (level 40)** — the subsystem-lead consumer. Writes the system prompt, picks the model profile, negotiates exceptions against the paved road when a workload genuinely needs them.
- **mod-413 (platform leadership)** — owns the adoption scorecard cadence (chapter 06), the deprecation-ladder sign-offs (chapter 07), the operating-model report, and the quarterly roadmap review.

## Summary

- The paved-road SDK plane owns: the `PavedRoadAgent` reference shape, the five framework adapters, the scaffolding CLI and template catalogue, the local-dev loop, the CI template with its five gates, the standardised instrumentation hook library, the `AgentRevision` manifest, the four adoption KPIs and their dashboard, and the enablement operating cadence including the exception register. It does *not* own the capabilities themselves (those belong to the eleven planes below it), product-side agent correctness, upstream framework roadmaps, the CNCF maturity model itself, the leadership cadence, or legal / licensing interpretation.
- Paved roads for agents differ from paved roads for microservices in three ways: the runtime is non-deterministic and long-running, the CI gates include eval and safety and cost (not just lint and unit), and the framework choice is first-class and plural.
- The invariant `AgentRevision` manifest is the connective tissue: every paved-road build emits it; every downstream consumer (adoption dashboard, deprecation ladder, governance cadence) reads it without transformation. Chapters 02–07 add fields.
- Three non-negotiable obligations: every paved-road agent is on every plane with one call; every paved-road build emits an `AgentRevision` manifest; the ninety-second path from `clone` to first turn is measured on every release and a regression blocks the release.

The next chapter opens the first piece — the `PavedRoadAgent` reference shape and the five framework adapters.
