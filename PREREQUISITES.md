# Prerequisites — Agent Platform Engineer

This track assumes the skills that lower-level tracks already own. It does not re-teach them. Learners who have not acquired the material below should follow the linked tracks first.

## Assumed lower-level tracks

- [`ai-infra-junior-engineer-learning`](../ai-infra-junior-engineer-learning) **(level 10)** — Linux, Git, Python packaging, HTTP APIs, basic containers, basic observability.
- [`llm-application-developer-learning`](../llm-application-developer-learning) **(level 25)** — LLM provider APIs (Anthropic, OpenAI, Google), prompts, structured outputs, streaming, token economics, tool / function-call basics.
- [`agentic-ai-engineer-learning`](../agentic-ai-engineer-learning) **(level 30)** — Build-altitude agent fundamentals: ReAct / reason-act loop, tool-call mechanics, framework comparison (LangGraph, CrewAI, AutoGen, OpenAI Agents SDK, Google ADK, Anthropic Agent SDK, smolagents), single-agent and single-MCP-server patterns, basic agent memory.
- [`ai-infra-senior-engineer-learning`](../ai-infra-senior-engineer-learning) **(level 30)** — Kubernetes, containers, cloud-provider primitives (one of AWS / GCP / Azure), service mesh, secrets management at the depth required to run multi-tenant platform services.

Where a module below leans especially hard on one of these tracks (for example, mod-402's Kubernetes-operator exercises), the module links back to the owner track rather than restating it.

## Working knowledge expected

- **Python** — packaging, asyncio, type hints, pytest. Platform-side services ship in Python.
- **TypeScript / Node.js** — modules, async/await, Zod-shaped schemas. Platform SDKs ship in both ecosystems.
- **Databases** — Postgres (including [`pgvector`](https://github.com/pgvector/pgvector)) and Redis at operational depth; comfort reading EXPLAIN plans.
- **HTTP / gRPC** — REST + OpenAPI schemas, server-sent events / streaming, basic gRPC.
- **OAuth 2.0 / OIDC** — enough to recognise token-exchange, scopes, dynamic client registration; mod-403 and mod-405 take this to depth.
- **OpenTelemetry basics** — traces, spans, baggage, context propagation; mod-408 builds the GenAI semantic-convention layer on top.
- **CI/CD** — pipelines, artifact promotion, canary / shadow deploys; mod-407 and mod-412 build the agent-specific slice on top.

## Not taught here — linked out

- **ML model training / fine-tuning** — owned by [`fine-tuning-engineer-learning`](../fine-tuning-engineer-learning) and [`training-pipeline-engineer-learning`](../training-pipeline-engineer-learning).
- **Inference serving (vLLM / SGLang / TRT-LLM)** — owned by [`ai-infra-inference-engineer-learning`](../ai-infra-inference-engineer-learning). The model gateway in mod-406 routes to that peer's serving layer; this role does not operate the serving substrate.
- **Application-layer AI evaluation methodology (judge calibration, RAG eval, online eval)** — owned by [`ai-eval-engineer-learning`](../ai-eval-engineer-learning). mod-407 ships the eval *platform* that templatises that peer's methodology.
- **Harm-model authoring, guardrail-effectiveness methodology, red-team methodology** — owned by [`ai-risk-engineer-learning`](../ai-risk-engineer-learning) (level 25) and [`agentic-safety-engineer-learning`](../agentic-safety-engineer-learning) (level 40). mod-411 ships the safety plane that enforces their findings.
- **Agentic-system architecture across product lines** — owned by [`agentic-systems-architect-learning`](../agentic-systems-architect-learning) (level 48). That architect designs; this role implements and operates.
- **Cross-stack infrastructure architecture and build-vs-buy strategy at portfolio scope** — owned by [`ai-infra-architect-learning`](../ai-infra-architect-learning) (level 45).
- **Product-side multi-agent subsystem implementation leadership** — owned by [`senior-agentic-ai-engineer-learning`](../senior-agentic-ai-engineer-learning) (level 40). That peer leads product-side subsystems; this role ships the platform both peers' teams consume.
- **SOC-side AI incident response / DFIR** — owned by SecOps teams. The platform feeds AI-specific signal to SecOps; it does not handle SOC-side incidents.
- **Legal / privacy / compliance opinion** — owned by legal counsel. The platform produces compliance evidence (EU AI Act Article 72 post-market monitoring, NIST AI RMF measure/manage) that counsel consumes; it does not deliver legal opinion.

## Recommended reading before mod-401

- Anthropic — *Building Effective Agents*: <https://www.anthropic.com/research/building-effective-agents>
- OpenAI — *A Practical Guide to Building Agents*: <https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf>
- Model Context Protocol — *Specification*: <https://modelcontextprotocol.io/specification>
- CNCF — *Platform Engineering Maturity Model*: <https://tag-app-delivery.cncf.io/whitepapers/platform-eng-maturity-model/>
- Google — *SRE Book* (skim): <https://sre.google/sre-book/table-of-contents/>
- OWASP — *Top 10 for LLM Applications* and *Agentic AI Threats and Mitigations*: <https://owasp.org/www-project-top-10-for-large-language-model-applications/> and <https://genai.owasp.org/resource/agentic-ai-threats-and-mitigations/>
- OpenTelemetry — *Semantic Conventions for Generative AI*: <https://opentelemetry.io/docs/specs/semconv/gen-ai/>
