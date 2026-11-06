# The Scaffolding CLI and Template Catalogue

Chapter 02 pinned the `PavedRoadAgent` shape and the five adapters. This chapter is the first touch point between a stream-aligned engineer and the paved road — the moment they type one command and get a working repo back.

The scaffolding CLI is not a convenience. It is the mechanism by which *every* paved-road convention lands in a repo correctly on day one: the `AgentRevision` manifest shape, the CI gates, the fixture directory, the instrumentation hooks, the trust-tier policy pack, the model-profile pin. A repo scaffolded with the CLI is on the paved road by construction. A repo hand-rolled is off the paved road until it satisfies the attribute-contract test and emits a manifest — both of which the scaffolder sets up for free.

This chapter walks the CLI's design, the three template-engine options, the template-catalogue shape, the generation-receipt file, versioning and update discipline, and the acceptance criteria that gate a template before it ships.

## What the CLI must produce in ninety seconds

The forcing-function budget, pulled from chapter 01's third obligation: *ninety seconds from `clone` to first turn*.

That is the whole-loop budget. Inside it, the scaffolder owns:

1. `platform new agent` prompts and runs the template (≤10 s).
2. The generated repo installs its dependencies (≤45 s on a warm cache; the template controls what "dependencies" means — the paved-road default is the SDK plus the adapter and the mod-407 fixture client).
3. `make dev` (chapter 04) stands up the local-dev loop (≤25 s: sandbox emulator cold-start is the usual bottleneck).
4. The first agent turn against a local mock model returns successfully (≤10 s).

If the generated repo requires environment variables the engineer has to hunt down, or network access to a VPN, or a Docker image they do not yet have pulled, the budget is lost before the loop starts. The scaffolder's job is to produce a repo whose `make dev` is self-contained against mocks and fixtures.

## The three template-engine options

The paved-road SDK's scaffolder is a thin wrapper around a template engine. Three mainstream options; pick one per platform (switching later is a non-trivial migration).

| Engine | Strengths | Weaknesses | When it fits |
|---|---|---|---|
| **[Cookiecutter](https://cookiecutter.readthedocs.io/)** | Simple, mature, Jinja-based, huge community. Low ceremony. | One-shot: regenerating a repo against a new template version is a manual diff exercise. No state carried forward. | Platforms with low regeneration frequency and a tolerant tenant base. |
| **[Copier](https://copier.readthedocs.io/)** | First-class *updates*: `copier update` re-runs the template against the repo, applying new template changes to an existing project. Answers file (`.copier-answers.yml`) persists user choices. | Smaller ecosystem than Cookiecutter; a few more concepts to learn. | Platforms that expect to rev the template often (quarterly or faster) and want the regeneration path to be first-class. **The paved-road default.** |
| **[Backstage Software Templates](https://backstage.io/docs/features/software-templates/)** | Integrates with Backstage catalog; template steps can include non-file actions (create GitHub repo, register in catalog, publish TechDocs). Multi-step actions like "scaffold + create repo + register in catalog" are declarative. | Requires a Backstage install; template authoring is YAML-plus-nested-actions; more operational weight. | Organisations that already run Backstage; benefit is the catalog integration, not the templating itself. Often invokes Copier or Cookiecutter underneath. |

The common pattern:

- **Small platforms** standardise on Copier. The scaffolding CLI is `platform new`, a thin `copier copy` wrapper.
- **Backstage organisations** standardise on Backstage Software Templates, which call Copier underneath via the `fetch:template` action. The scaffolding CLI is `platform new` *and* the Backstage portal's "Create" button; both land the same template.

In the rest of this chapter the examples assume Copier under a `platform new` CLI; the Backstage overlay is a straightforward wrap.

## The CLI shape

The CLI reads the template catalogue and prompts for the parameters each template declares. The paved-road default is five parameters; a tenant-specific template may add more.

```
$ platform new agent
? Agent slug (kebab-case, e.g. writer-v1):  writer-v1
? Tenant id (from org catalogue):             acme-financial
? Workload class:  [interactive-chat / batch-document / long-running-research / agent-to-agent / tool-driven-automation]
  interactive-chat
? Resilience tier (mod-410 chapter 02):  [T1 / T2 / T3]
  T1
? Framework adapter:  [anthropic / openai / google-adk / microsoft / langgraph-temporal]
  anthropic
? Model profile (mod-406):  [paved-road-balanced / paved-road-fast / paved-road-high-quality / paved-road-cheap]
  paved-road-balanced

Scaffolding templates/agent-chat-anthropic@v3.2.0 into ./writer-v1 ...
  - .copier-answers.yml  ⦿ records your choices for `copier update` later
  - .platform/manifest-template.yaml  ⦿ chapter 01 AgentRevision seed
  - src/agent.py  ⦿ PavedRoadAgent subclass
  - prompts/writer.md  ⦿ system prompt placeholder
  - fixtures/regression/  ⦿ mod-407 fixture directory
  - Makefile  ⦿ local-dev-loop entry points
  - .github/workflows/paved-road.yml  ⦿ reusable CI workflow import
  - docs/README.md  ⦿ ninety-second walkthrough

Done. Next:
  cd writer-v1
  make dev         # boots sandbox emulator, trace viewer, local model; runs the first turn
  make eval        # offline replay against the seed fixtures
  git init && gh repo create --private --source=. --remote=origin --push
```

Four pieces are load-bearing:

1. **The parameters bind the manifest.** Every prompt here maps to an `AgentRevision` field (chapter 01). The template never asks for a choice the manifest does not surface.
2. **The scaffold is an immediately runnable repo.** Not "run `make install` then edit three TODOs". The engineer gets *a working agent* that returns a response on a mock model inside the local-dev loop.
3. **The receipt is machine-readable.** `.copier-answers.yml` is the Copier state file. The scaffolder also writes `.platform/generation-receipt.json` — the paved-road record (below).
4. **The next-step commands are paved-road commands.** `make dev` is chapter 04. `make eval` is chapter 05. The engineer does not have to find a wiki to know what to run next.

### CLI subcommands beyond `new`

The CLI's full shape — this is representative, not prescriptive:

| Subcommand | Purpose | Chapter |
|---|---|---|
| `platform new agent` | Scaffold a new agent from a template. | This chapter. |
| `platform new tool` | Scaffold a new MCP tool server with the registry metadata and health-check wiring. | mod-403 (template owned by that plane, exposed here). |
| `platform update` | Rerun the template against an existing repo (Copier update). | This chapter (regeneration). |
| `platform build` | Produce the `AgentRevision` manifest from the compile-time manifest + CI data. | Chapter 01 and 05. |
| `platform dev` | Shortcut for `make dev`; brings up the local-dev loop. | Chapter 04. |
| `platform eval` | Run the offline eval replay against the fixture pins. | Chapter 04 and 07. |
| `platform doctor` | Diagnose paved-road health: SDK version, adapter version, attribute-contract test, manifest presence. | This chapter and chapter 07. |
| `platform exception` | File / inspect / close an exception-register entry. | Chapter 07. |
| `platform migrate` | Run a named migration playbook (chapter 07) against the current repo. | Chapter 07. |

A single binary with these verbs is what makes the paved road feel like a product rather than a wiki page.

## The template catalogue

The template catalogue is a git repository on the platform's SCM. One top-level directory per template; each template is Copier-compatible (`copier.yml` + the file tree). A sketch:

```
template-catalogue/
├── catalog.yaml                                 # the catalogue index — metadata about each template
├── templates/
│   ├── agent-chat-anthropic/                    # {framework=anthropic, workload=chat}
│   │   ├── copier.yml
│   │   ├── README.md
│   │   ├── {{ _copier_conf.answers_file }}.yml.jinja
│   │   ├── src/
│   │   │   └── agent.py.jinja
│   │   ├── prompts/
│   │   │   └── writer.md.jinja
│   │   ├── fixtures/
│   │   │   └── regression/
│   │   │       └── golden-path.jsonl.jinja
│   │   ├── Makefile.jinja
│   │   ├── docker-compose.dev.yml.jinja
│   │   ├── .github/
│   │   │   └── workflows/
│   │   │       └── paved-road.yml.jinja
│   │   ├── .platform/
│   │   │   └── manifest-template.yaml.jinja
│   │   └── pyproject.toml.jinja
│   ├── agent-chat-openai/
│   │   └── ...
│   ├── agent-chat-google-adk/
│   │   └── ...
│   ├── agent-chat-microsoft/
│   │   └── ...
│   ├── agent-batch-langgraph-temporal/
│   │   └── ...
│   ├── agent-research-langgraph-temporal/
│   │   └── ...
│   ├── agent-a2a-anthropic/
│   │   └── ...
│   └── tool-mcp-python/
│       └── ...
└── docs/
    ├── CONTRIBUTING.md                          # the template authoring guide
    └── ADR/                                     # template-specific architecture decision records
```

The `catalog.yaml` index is the CLI's lookup. The CLI reads it at startup to populate the `platform new --template` tab-completion and the "pick a template" interactive prompt.

```yaml
# catalog.yaml
schema_version: template-catalog/v1
templates:
  - id: agent-chat-anthropic
    title: Interactive Chat Agent — Anthropic Agent SDK
    description: Paved-road PavedRoadAgent subclass on the Anthropic Agent SDK.
    adapter: anthropic
    workload_classes: [interactive-chat]
    tiers: [T1, T2]
    version: 3.2.0
    paved_road_sdk_min_version: 2.4.0
    paved_road_sdk_max_version: 3.0.0
    owners: ["@platform-sdk-maintainers"]
    status: active        # active | deprecated | retired
    repo_path: templates/agent-chat-anthropic
  - id: agent-chat-openai
    ...
```

Four load-bearing properties:

1. **Templates are indexed by `(adapter, workload_class, tier)`.** Not every cell is interesting; the catalogue covers only the cells the platform wants to support. A workload a tenant proposes that is not covered is an exception-register entry, which may upgrade into a new template if the exception is repeated.
2. **Templates declare the SDK range they are compatible with.** When the SDK rev moves, the template's `paved_road_sdk_min_version` is the gate. A template referencing an SDK-removed shim (chapter 07) is marked `deprecated` in `status` and `platform doctor` warns on it.
3. **Templates have owners.** Scope policing (chapter 01): a template that nobody owns is a template nobody updates. The owners are a `CODEOWNERS` entry on the template directory and a group-chat ping on the template's issues.
4. **Templates have ADRs.** Every non-trivial design choice in the template (why Copier not Cookiecutter, why one directory structure, why this `pyproject` policy) is a small ADR. When a tenant asks "why does the template do X", there is a document. Chapter 07's migration playbook template builds on this.

### The template-authoring acceptance checklist

A template cannot enter the catalogue until it satisfies:

- [ ] The scaffolded repo builds and runs on a laptop with no VPN and no cloud credentials inside the ninety-second budget.
- [ ] The scaffolded repo passes the paved-road attribute-contract test (chapter 05) on first `make test`.
- [ ] The scaffolded repo emits an `AgentRevision` manifest on `platform build` with all required fields populated.
- [ ] The scaffolded repo's CI config is a one-line `uses: paved-road-ci/.github/workflows/paved-road.yml@v3` (or GitLab `include:` or Buildkite template reference) — not a hand-rolled workflow.
- [ ] The template's `README.md` walks the ninety-second loop explicitly: `make dev`, expected output, trace-viewer URL, `make eval`, expected verdict.
- [ ] The template's `copier.yml` prompts are a strict subset of the platform-wide parameter set (no bespoke prompts outside `extras:`).
- [ ] The template declares its owner and its ADRs; the catalogue entry is present.
- [ ] A smoke test (`make smoke` in CI) scaffolds the template and runs the ninety-second loop on every pull request against the catalogue repo.
- [ ] The template has at least one end-to-end regression fixture in `fixtures/regression/` demonstrating a successful turn against the mock model.

A template that cannot pass the checklist does not ship. The pressure to add a template "quickly" is the usual way paved roads lose their quality; chapter 01 obligation 3 is what the checklist enforces.

## The generation receipt

Every `platform new` invocation writes two artefacts into the generated repo:

- `.copier-answers.yml` — Copier's native state. Used for `copier update` / `platform update`.
- `.platform/generation-receipt.json` — the paved-road record.

```json
{
  "schema_version": "generation-receipt/v1",
  "generated_at": "2026-11-06T14:22:03Z",
  "scaffolder_version": "platform-cli@2.4.1",
  "template": {
    "id": "agent-chat-anthropic",
    "version": "3.2.0",
    "repo_path": "templates/agent-chat-anthropic",
    "catalogue_commit": "github.com/.../template-catalogue@a1b2c3d"
  },
  "answers": {
    "agent_slug": "writer-v1",
    "tenant_id": "acme-financial",
    "workload_class": "interactive-chat",
    "tier": "T1",
    "adapter": "anthropic",
    "model_profile": "paved-road-balanced"
  },
  "paved_road_sdk_version_at_scaffold": "2.4.1",
  "fixtures_seeded": [
    "fixtures/regression/golden-path.jsonl"
  ]
}
```

Three invariants this record enforces:

1. **Provenance.** The build (chapter 05) joins the receipt against the CI stream; `time-to-first-agent` (chapter 06) reads the receipt's `generated_at` and the first-build `generated_at` to compute the delta.
2. **Reproducibility.** `platform update` can regenerate from the exact template commit if a tenant needs to rebuild from scratch.
3. **Audit.** When a migration playbook (chapter 07) runs, it reads the receipt to know which template was used and whether this repo is subject to the migration.

The receipt is not secret; it ships in the repo. Anything a tenant would not want in version control (secrets, keys) does not go through the scaffolder at all.

## The `platform update` path — Copier's regeneration story

When the template revs — new fixtures, updated CI config, SDK minor-bump adjustment — every agent on the paved road should get the update without a hand rewrite. Copier is built for this: `platform update` re-runs the template against the existing repo, applying the new template version's changes over the user's modifications.

```
$ cd writer-v1
$ platform update
Updating from template agent-chat-anthropic v3.2.0 → v3.3.0 ...
  - .github/workflows/paved-road.yml      updated (CI gates revision)
  - Makefile                              updated (new `make trace` target)
  - .platform/manifest-template.yaml      updated (added `instrumentation.platform_semconv_rev`)
  - src/agent.py                          unchanged (user-modified region preserved)
  - prompts/writer.md                     unchanged (user-modified)
Conflict in .platform/manifest-template.yaml: your version added a custom field.
  Resolved via three-way merge; review the result.

Next steps:
  git diff          # review the update
  make test         # re-run the paved-road attribute-contract test
  git commit -am "chore: paved-road template update 3.2.0 → 3.3.0"
```

Four things this buys the platform:

1. **Updates are a single command.** Chapter 07's migration playbook can prescribe `platform update` as the first step for the common case.
2. **User modifications are preserved by default.** Only the regions marked `_safe_to_regenerate` in the template (CI config, Makefile, the paved-road hook wiring, the manifest template) are overwritten without a conflict marker.
3. **The update is auditable.** The receipt's `template.version` field changes to `3.3.0`, which the adoption dashboard (chapter 06) picks up — "60% of agents have adopted the v3.3.0 template" becomes a real datapoint.
4. **The deprecation ladder has a mechanical step.** When a template is deprecated (chapter 07), `platform doctor` warns; when it is removed, `platform update` fails with a prescribed migration path.

### The three places template changes land

Not every edit to a template is the same kind of change. Three categories, three different update stories:

| Category | Example | Update story |
|---|---|---|
| **Safe-to-regenerate** (CI config, Makefile, the paved-road hook wiring) | "Add `make trace` target" | `platform update` overwrites without prompting; CI re-tests. |
| **User-owned** (`src/agent.py`, `prompts/*.md`, `fixtures/*`) | User edits their system prompt | `platform update` never overwrites; user modifications persist. |
| **Semi-structured** (`.platform/manifest-template.yaml`) | Template adds a new required field | Three-way merge; conflict surfaces to the user; `platform doctor` prescribes the fix. |

Mark the categories explicitly in the template's `copier.yml` (Copier supports `_subdirectory`, `_templates_suffix`, and conditional blocks for this). A template that treats everything as safe-to-regenerate will stomp user prompts; a template that treats everything as user-owned will never ship updates.

## Versioning the template catalogue

Templates version with SemVer. Three rules:

1. **Patch** — a bugfix (typo in README, formatting fix in generated CI config) that cannot break an existing repo on `platform update`.
2. **Minor** — a new file added (new fixture, new CI gate that is advisory on first version) that does not require the user to change anything in their repo.
3. **Major** — a change that requires the user to take action after `platform update` (e.g., a renamed `.platform/manifest-template.yaml` field that the paved-road attribute-contract test now enforces). Majors carry a migration playbook (chapter 07).

The paved-road CLI's `platform update` surfaces the SemVer transition ("updating from v3.2.0 to v4.0.0 (major) — see migration playbook `templates/migrations/v3-to-v4.md`"). Major updates against the catalogue are coordinated by the enablement cadence (chapter 07).

## A worked example: the chat-anthropic template end to end

Walk the template from the catalogue entry through to the scaffolded repo.

**`catalog.yaml` entry:**

```yaml
- id: agent-chat-anthropic
  title: Interactive Chat Agent — Anthropic Agent SDK
  description: Paved-road PavedRoadAgent subclass on the Anthropic Agent SDK.
  adapter: anthropic
  workload_classes: [interactive-chat]
  tiers: [T1, T2]
  version: 3.2.0
  paved_road_sdk_min_version: 2.4.0
  owners: ["@platform-sdk-maintainers"]
  status: active
  repo_path: templates/agent-chat-anthropic
```

**`copier.yml` prompts (subset):**

```yaml
_min_copier_version: "9.0.0"
_subdirectory: "{{ agent_slug }}"
_templates_suffix: ".jinja"

agent_slug:
  type: str
  help: "Agent slug (kebab-case)"

tenant_id:
  type: str
  help: "Tenant id from org catalogue"

workload_class:
  type: str
  choices: [interactive-chat]
  default: interactive-chat

tier:
  type: str
  choices: [T1, T2]

model_profile:
  type: str
  choices: [paved-road-balanced, paved-road-fast, paved-road-high-quality, paved-road-cheap]
  default: paved-road-balanced

paved_road_sdk_version:
  type: str
  default: "2.4.1"
```

**`src/agent.py.jinja`:**

```python
from paved_road_agent import PavedRoadAgent, agent
from paved_road_agent.adapters.anthropic import AnthropicAgentAdapter

@agent(
    workload_class="{{ workload_class }}",
    tier="{{ tier }}",
    model_profile="{{ model_profile }}",
    system_prompt="prompts/system.md",
    tools=["web.search", "notes.read", "notes.write"],
)
class {{ agent_slug | kebab_to_pascal }}(AnthropicAgentAdapter, PavedRoadAgent):
    pass
```

**`.platform/manifest-template.yaml.jinja`:**

```yaml
schema_version: agent-revision/v1
agent:
  agent_id: {{ agent_slug }}
  tenant_id: {{ tenant_id }}
sdk:
  paved_road_sdk_version: {{ paved_road_sdk_version }}
  adapter: anthropic
workload:
  workload_class: {{ workload_class }}
  tier: {{ tier }}
  policy_pack: paved-road@{{ _now.strftime('%Y-%m-01') }}
  model_gateway_profile: {{ model_profile }}
```

**`Makefile.jinja`:**

```makefile
.PHONY: dev eval test smoke build doctor
dev:
	docker compose -f docker-compose.dev.yml up --wait
	poetry run paved-road dev
eval:
	poetry run paved-road eval --fixture-dir fixtures/regression
test:
	poetry run pytest -q
build:
	poetry run paved-road build
doctor:
	poetry run paved-road doctor
smoke:
	timeout 90 bash -c 'make dev && curl -s localhost:7333/ping | grep OK'
```

**`.github/workflows/paved-road.yml.jinja`:**

```yaml
name: paved-road
on: [push, pull_request]
jobs:
  paved-road:
    uses: platform/paved-road-ci/.github/workflows/paved-road.yml@v3
    with:
      agent_slug: {{ agent_slug }}
      tenant_id: {{ tenant_id }}
      workload_class: {{ workload_class }}
      tier: {{ tier }}
```

Chapter 05 opens the reusable CI workflow. The template's contribution is to import it with the right parameters, not to inline the gates.

## Template-catalogue governance

Four rules keep the catalogue healthy.

1. **The platform team owns the "paved-road" templates** (`templates/agent-*`, `templates/tool-*`). Tenant-authored templates live under `templates/x-<tenant>/...` with their own owners. The catalogue's CI blocks a tenant template from claiming a workload class the paved-road templates already serve — the catalogue is one road with named exits, not a bazaar.
2. **Every pull request against the catalogue runs the smoke test for every affected template.** A README typo fix does not trigger the suite; a template file change does. The catalogue repo's CI runtime is bounded; templates whose smoke tests exceed a budget (e.g., 2 minutes per template) go back to the author.
3. **Template deprecations go through the deprecation ladder** (chapter 07). A `status: deprecated` entry is announced, left for one review cycle, then moved to `status: retired`, then finally removed from the repo. During `deprecated`, `platform new` emits a warning; during `retired`, it refuses.
4. **A new paved-road template requires a proposal document** that names the workload-class niche it fills, the SLO the resulting repo is expected to hit, the migration path from any existing template, and the maintenance burden it adds. The proposal lands in the catalogue's `docs/ADR/` directory before the template.

## Common pitfalls when authoring templates

- **Prompts that only make sense to the author.** "Is this tenant GDPR-affected? [y/N]" is a bad scaffolder prompt — the answer shapes a repo the tenant will inherit forever. Make the prompt either structural (workload class, tier) or deferred (filled in `.platform/manifest.yaml` after scaffolding by a tenant-admin, not by the engineer scaffolding).
- **Starter code that does not work.** A template whose scaffolded repo does not pass its own tests on first `make test` is a template that teaches the wrong habit (failing tests are normal, then the engineer stops looking at them). The smoke test (above) is the enforcement.
- **Auto-generated docs that nobody updates.** A template that writes a 400-line README is a README no engineer reads; only the first ninety seconds matters. Short and precise.
- **Encoding the author's preferred non-paved-road choices.** If the author prefers `uv` over `poetry` and the paved road defaults to `poetry`, the template follows the paved road. Chapter 01 scope policing applies to templates; the catalogue is not a demo space for individual preferences.
- **Prompts that could have been defaults.** Every prompt is a cognitive load on the engineer. If the paved road has a right answer for a parameter (e.g., the policy pack name is always `paved-road@<current month>`), encode it as a default or an implicit value, not a prompt.
- **One mega-template instead of per-cell templates.** A template with a dozen prompts and conditional blocks for every workload class is harder to maintain than four targeted templates. The catalogue is cheap; the maintenance per template is where the cost lives.
- **Skipping `.copier-answers.yml`.** Without the answers file, the update path breaks and `platform update` cannot cleanly diff against the current template. Always ship Copier's state file.
- **Forgetting the fixtures.** A scaffolded agent with no fixtures cannot run the eval-regression gate (mod-407 / chapter 05), so the paved-road claim is broken from day one. Every template ships at least one golden-path fixture.

## The `platform doctor` command

A diagnostic CLI subcommand that reads the current repo and tells the engineer where they are on the paved road.

```
$ platform doctor
Repo:                writer-v1
Template:            agent-chat-anthropic @ 3.2.0  [UPDATE AVAILABLE: 3.3.0]
Paved-road SDK:      paved_road_agent @ 2.4.1      [CURRENT]
Adapter:             AnthropicAgentAdapter        [CURRENT]
Manifest template:   .platform/manifest-template.yaml  ✓ valid
Attribute contract:  ✓ passing
CI config:           .github/workflows/paved-road.yml  ✓ paved-road reusable workflow at v3
Exception register:  none active
Local-dev loop:      ✓ `make dev` targets exist

Recommendations:
  - Run `platform update` to pick up template version 3.3.0 (adds `make trace` and the new `platform_semconv_rev` manifest field).
```

Three design choices:

1. **The doctor is the engineer's self-service tool.** Chapter 07's office-hours triage starts with "run `platform doctor` and paste the output".
2. **Warnings have explicit next steps.** An "update available" note includes the command to run; a "manifest invalid" error includes the schema file and the offending field.
3. **The doctor emits a record.** A JSON snapshot of the output lives at `.platform/doctor-report.json`; the CI picks it up to decorate the build status.

## Summary

- The scaffolding CLI is the first touch point between a stream-aligned engineer and the paved road; its ninety-second budget is a chapter 01 obligation. Every parameter the CLI asks for maps to an `AgentRevision` manifest field.
- Three template-engine options cover the mainstream: Cookiecutter (simplest), Copier (first-class updates — paved-road default), Backstage Software Templates (when the organisation already runs Backstage). Pick one per platform.
- The template catalogue is a GitOps-managed repo with one template per `{adapter, workload_class, tier}` cell. Every template has an owner, an ADR, and a smoke test; a template that cannot scaffold a repo that boots inside the ninety-second budget does not ship.
- Every `platform new` writes two artefacts: `.copier-answers.yml` (Copier state for updates) and `.platform/generation-receipt.json` (the paved-road provenance record). The receipt is what `time-to-first-agent` (chapter 06) joins against.
- `platform update` makes template revisions a single command. Three categories of file (safe-to-regenerate, user-owned, semi-structured) control what the update does to each.
- Template versioning is SemVer; majors require a migration playbook (chapter 07). `platform doctor` is the engineer's self-service diagnostic.

The next chapter opens the local-dev loop — what `make dev` actually brings up, and why the ninety-second budget is as much about the loop as the scaffolder.
