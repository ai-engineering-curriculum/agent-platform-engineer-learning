# Resources — mod-412 (Paved-Road SDKs, Templates, and Developer Experience)

All references on this page are primary / authoritative public sources at the time of writing (2026-11-06). Each resource is tagged with the chapter(s) it supports. Prefer the primary source over any blog summary — the agent SDK docs re-rev, the OpenTelemetry GenAI semantic conventions move, and the CNCF Platforms working group updates the maturity model on a declared cadence.

## Chapter 01 — The Paved-Road SDK and What It Owns

### Platform engineering frames (cross-ref mod-401)
- **Skelton & Pais — Team Topologies**: <https://teamtopologies.com/>
- **Team Topologies — Platform Team pattern page**: <https://teamtopologies.com/key-concepts-content/what-is-a-platform-team>
- **CNCF TAG App Delivery — Platforms Whitepaper**: <https://tag-app-delivery.cncf.io/whitepapers/platforms/>
- **CNCF TAG App Delivery — Platform Engineering Maturity Model**: <https://tag-app-delivery.cncf.io/whitepapers/platform-eng-maturity-model/>
- **Internal Developer Platform community**: <https://internaldeveloperplatform.org/>
- **Humanitec — State of Internal Developer Platforms**: <https://humanitec.com/whitepapers>

### Agent platform context (cross-ref mod-401 chapter 03)
- **Anthropic — Building Effective Agents**: <https://www.anthropic.com/research/building-effective-agents>
- **OpenAI — A Practical Guide to Building Agents (PDF)**: <https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf>

### OpenTelemetry semantic contract (cross-ref mod-408)
- **OpenTelemetry — Semantic Conventions for Generative AI**: <https://opentelemetry.io/docs/specs/semconv/gen-ai/>
- **OpenTelemetry — specification**: <https://opentelemetry.io/docs/specs/otel/>
- **W3C Trace Context**: <https://www.w3.org/TR/trace-context/>

## Chapter 02 — Reference Shapes Across Five Frameworks

### Anthropic Agent SDK
- **Anthropic — Agent SDK overview**: <https://docs.anthropic.com/en/api/agent-sdk/overview>
- **Anthropic — Claude Code docs (the harness the Agent SDK packages)**: <https://docs.anthropic.com/en/docs/claude-code/overview>
- **Anthropic Agent SDK on GitHub (Python)**: <https://github.com/anthropics/claude-agent-sdk-python>
- **Anthropic Agent SDK on GitHub (TypeScript)**: <https://github.com/anthropics/claude-code>
- **Anthropic — hooks in the Agent SDK**: <https://docs.anthropic.com/en/docs/claude-code/hooks>

### OpenAI Agents SDK
- **OpenAI — Agents SDK Python docs**: <https://openai.github.io/openai-agents-python/>
- **OpenAI — Agents SDK on GitHub**: <https://github.com/openai/openai-agents-python>
- **OpenAI — Agents SDK: handoffs**: <https://openai.github.io/openai-agents-python/handoffs/>
- **OpenAI — Agents SDK: guardrails**: <https://openai.github.io/openai-agents-python/guardrails/>
- **OpenAI — Responses API docs**: <https://platform.openai.com/docs/api-reference/responses>

### Google Agent Development Kit (ADK)
- **Google — ADK documentation**: <https://google.github.io/adk-docs/>
- **Google — ADK on GitHub (Python)**: <https://github.com/google/adk-python>
- **Google — ADK agent types (LlmAgent, SequentialAgent, ParallelAgent, LoopAgent)**: <https://google.github.io/adk-docs/agents/>
- **Google — ADK Runner & Sessions**: <https://google.github.io/adk-docs/sessions/>
- **Google — Vertex AI Agent Engine**: <https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/overview>

### Microsoft Agent Framework
- **Microsoft — Agent Framework overview**: <https://learn.microsoft.com/en-us/agent-framework/overview>
- **Microsoft — Agent Framework on GitHub**: <https://github.com/microsoft/agent-framework>
- **Microsoft — AutoGen (predecessor, still active)**: <https://microsoft.github.io/autogen/>
- **Microsoft — Semantic Kernel (predecessor, still active)**: <https://learn.microsoft.com/en-us/semantic-kernel/overview/>

### LangGraph and Temporal
- **LangChain — LangGraph documentation**: <https://langchain-ai.github.io/langgraph/>
- **LangChain — LangGraph on GitHub**: <https://github.com/langchain-ai/langgraph>
- **LangChain — LangGraph checkpoints**: <https://langchain-ai.github.io/langgraph/concepts/persistence/>
- **Temporal — documentation**: <https://docs.temporal.io/>
- **Temporal — Python SDK**: <https://docs.temporal.io/develop/python>
- **Temporal — Workflows**: <https://docs.temporal.io/workflows>
- **Temporalite — dev-server binary**: <https://github.com/temporalio/temporalite>

## Chapter 03 — Scaffolding CLI and Template Catalogue

### Template engines
- **Cookiecutter — documentation**: <https://cookiecutter.readthedocs.io/>
- **Cookiecutter on GitHub**: <https://github.com/cookiecutter/cookiecutter>
- **Copier — documentation**: <https://copier.readthedocs.io/>
- **Copier on GitHub**: <https://github.com/copier-org/copier>
- **Copier — updating a project**: <https://copier.readthedocs.io/en/stable/updating/>

### Backstage and self-service portals
- **Backstage — Software Templates**: <https://backstage.io/docs/features/software-templates/>
- **Backstage — Software Templates built-in actions**: <https://backstage.io/docs/features/software-templates/builtin-actions>
- **Backstage — TechDocs**: <https://backstage.io/docs/features/techdocs/techdocs-overview/>
- **Backstage on GitHub**: <https://github.com/backstage/backstage>

### CLI toolkits
- **Click (Python)**: <https://click.palletsprojects.com/>
- **Typer (Python, over Click)**: <https://typer.tiangolo.com/>
- **Cobra (Go)**: <https://github.com/spf13/cobra>
- **PyInstaller — build single-file Python binaries**: <https://pyinstaller.org/>

## Chapter 04 — The Local-Dev Loop

### Dev loops and containerisation
- **Docker Compose documentation**: <https://docs.docker.com/compose/>
- **Development Containers specification**: <https://containers.dev/>
- **Dev Containers extension for VS Code**: <https://code.visualstudio.com/docs/devcontainers/containers>
- **DevPod**: <https://devpod.sh/>
- **Coder**: <https://coder.com/>
- **Gitpod**: <https://www.gitpod.io/>
- **Nix & devenv**: <https://devenv.sh/>
- **Dagger**: <https://dagger.io/>

### Local trace viewers
- **Jaeger — all-in-one quickstart**: <https://www.jaegertracing.io/docs/latest/getting-started/>
- **Grafana Tempo**: <https://grafana.com/oss/tempo/>
- **SigNoz**: <https://signoz.io/docs/>
- **Hyperdx (local)**: <https://www.hyperdx.io/docs/>
- **OpenObserve**: <https://openobserve.ai/docs/>

### Local model runners
- **Ollama**: <https://ollama.com/>
- **LM Studio**: <https://lmstudio.ai/>
- **llama.cpp server**: <https://github.com/ggml-org/llama.cpp>

### Sandbox substrates (cross-ref mod-404)
- **gVisor**: <https://gvisor.dev/>
- **Firecracker**: <https://firecracker-microvm.github.io/>
- **Kata Containers**: <https://katacontainers.io/>

## Chapter 05 — CI Templates and Standardised Instrumentation Hooks

### Reusable CI workflows
- **GitHub Actions — reusing workflows**: <https://docs.github.com/en/actions/using-workflows/reusing-workflows>
- **GitLab CI — `include` keyword**: <https://docs.gitlab.com/ee/ci/yaml/includes.html>
- **Buildkite — pipeline templates**: <https://buildkite.com/docs/pipelines/templates>
- **Dagger — reusable modules**: <https://docs.dagger.io/>

### OpenTelemetry Python & semconv
- **OpenTelemetry Python documentation**: <https://opentelemetry.io/docs/languages/python/>
- **OpenTelemetry Python semantic-conventions package (PyPI)**: <https://pypi.org/project/opentelemetry-semantic-conventions/>
- **OpenTelemetry — Semantic Conventions for Generative AI**: <https://opentelemetry.io/docs/specs/semconv/gen-ai/>
- **OpenTelemetry Collector**: <https://opentelemetry.io/docs/collector/>

### Supply-chain / SBOM
- **CycloneDX specification**: <https://cyclonedx.org/>
- **SPDX specification**: <https://spdx.dev/>
- **SLSA framework**: <https://slsa.dev/>
- **Sigstore**: <https://www.sigstore.dev/>

### Linting and type checking (Python examples)
- **Ruff**: <https://docs.astral.sh/ruff/>
- **mypy**: <https://mypy-lang.org/>
- **pyright**: <https://microsoft.github.io/pyright/>

## Chapter 06 — Measuring Adoption and CNCF Maturity

### Operating-model and DX frames
- **CNCF TAG App Delivery — Platform Engineering Maturity Model**: <https://tag-app-delivery.cncf.io/whitepapers/platform-eng-maturity-model/>
- **Noda, Forsgren, Alexander, Storey — DevEx: What Actually Drives Productivity (ACM Queue, 2023)**: <https://queue.acm.org/detail.cfm?id=3595878>
- **Google / DORA — State of DevOps Reports**: <https://cloud.google.com/devops/state-of-devops>
- **SPACE framework paper (Forsgren, Storey, Maddila, Zimmermann, Houck, Butler, 2021)**: <https://queue.acm.org/detail.cfm?id=3454124>
- **Puppet — State of Platform Engineering**: <https://www.puppet.com/resources/state-of-platform-engineering>

### Dashboard tools
- **Grafana**: <https://grafana.com/docs/grafana/latest/>
- **Metabase**: <https://www.metabase.com/docs/>
- **Streamlit**: <https://docs.streamlit.io/>
- **DuckDB**: <https://duckdb.org/docs/>

### Related reading
- **Spotify — Backstage announcement and principles**: <https://backstage.io/blog>
- **Team Topologies — Platform as a Product**: <https://teamtopologies.com/key-concepts>
- **CNCF — case studies of platform teams**: <https://www.cncf.io/case-studies/>

## Chapter 07 — Enablement, Office Hours, Migrations, and the Deprecation Ladder

### Docs sites and knowledge bases
- **Docusaurus**: <https://docusaurus.io/>
- **MkDocs Material**: <https://squidfunk.github.io/mkdocs-material/>
- **Backstage TechDocs**: <https://backstage.io/docs/features/techdocs/techdocs-overview/>

### Deprecation and versioning references
- **Semantic Versioning (SemVer)**: <https://semver.org/>
- **Kubernetes — Deprecation Policy**: <https://kubernetes.io/docs/reference/using-api/deprecation-policy/>
- **Kubernetes — API versioning**: <https://kubernetes.io/docs/reference/using-api/>

### Migration-playbook anatomy (industry priors)
- **Stripe — Rigorous deprecation notice patterns (API changelog)**: <https://stripe.com/docs/upgrades>
- **Twilio — API deprecation patterns**: <https://www.twilio.com/docs/usage/api/changes>
- **GitHub REST API — deprecations**: <https://docs.github.com/en/rest/overview/api-versions>

## Exercise cross-references

- **Exercise 01** — paved-road SDK template authoring. Primary references: chapter 02 adapters (Anthropic Agent SDK, OpenAI Agents SDK, Google ADK, Microsoft Agent Framework, LangGraph + Temporal), chapter 03 Copier template, chapter 01 `AgentRevision` manifest shape.
- **Exercise 02** — scaffolding CLI and local-dev loop. Primary references: chapter 03 CLI subcommands, chapter 04 Docker Compose topology, Jaeger / OTel Collector / Ollama.
- **Exercise 03** — adoption-metric dashboard. Primary references: chapter 06 KPI formulas and CNCF maturity frame; DuckDB / Grafana / Streamlit; DevEx framework.
- **Exercise 04** — deprecation ladder and migration playbook. Primary references: chapter 07 ladder rungs and exception register; SemVer; Kubernetes deprecation policy.

## Note on verification

This module references upstream agent SDKs, framework version numbers, OpenTelemetry semantic-conventions revs, and CNCF maturity-model wording. All of these move. Before quoting a specific class name, method name, decorator, attribute identifier, or maturity-level description in the text or in a scaffolded template, verify it against the version you are packaging. The chapters use `<!-- needs-research: ... -->` on claims that cannot be verified at the time of authoring; the flag blocks auto-merge and is strictly preferable to silent drift.
