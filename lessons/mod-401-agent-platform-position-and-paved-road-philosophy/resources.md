# Resources — mod-401 (Position, Scope, and Paved-Road Philosophy)

All references on this page are publicly available primary sources at the time of writing (2026-11-06). Each resource is tagged with the chapter it supports; several are cited across more than one.

## Chapter 01 — Position on the Ladder

- **Track-internal ladder and ownership rule** — [`CURRICULUM.md`](../../CURRICULUM.md), [`JOB_REQUIREMENTS.md`](../../JOB_REQUIREMENTS.md), [`PREREQUISITES.md`](../../PREREQUISITES.md) in this repo. These documents fix the eight peer tracks and the lowest-level-owner ownership rule referenced throughout the chapter.
- **Anthropic — Building Effective Agents** (useful context for what L30–L40 agent engineers know): <https://www.anthropic.com/research/building-effective-agents>
- **OpenAI — A Practical Guide to Building Agents** (same): <https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf>

## Chapter 02 — CNCF Platform Maturity + Team Topologies

- **CNCF — Platform Engineering Maturity Model** (TAG App Delivery whitepaper, latest version): <https://tag-app-delivery.cncf.io/whitepapers/platform-eng-maturity-model/>
- **CNCF — Platforms Working Group**: <https://tag-app-delivery.cncf.io/wgs/platforms/>
- **Team Topologies** (Matthew Skelton & Manuel Pais, IT Revolution Press, 2019): <https://teamtopologies.com/book>
- **Team Topologies — patterns (practitioner site)**: <https://teamtopologies.com/>
- **Google SRE Book** (background for Operations maturity): <https://sre.google/sre-book/table-of-contents/>
- **Google SRE Workbook** (background for error-budget policies referenced at Level 3 Operations): <https://sre.google/workbook/table-of-contents/>
- **DORA — State of DevOps research** (research anchoring the Level 4 scorecard view): <https://dora.dev/research/>
- **GitHub Engineering — SPACE framework paper** (referenced alongside DORA in `mod-413`): <https://queue.acm.org/detail.cfm?id=3454124>

## Chapter 03 — Anthropic + OpenAI Guidance → Platform Obligations

- **Anthropic — Building Effective Agents**: <https://www.anthropic.com/research/building-effective-agents>
- **Anthropic — Claude Agent SDK overview**: <https://docs.anthropic.com/en/api/agent-sdk/overview>
- **OpenAI — A Practical Guide to Building Agents**: <https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf>
- **OpenAI — Agents SDK (Python)**: <https://openai.github.io/openai-agents-python/>
- **OpenAI — Responses API reference** (session-state primitive the SDK uses): <https://platform.openai.com/docs/api-reference/responses>
- **OpenAI — Structured Outputs**: <https://platform.openai.com/docs/guides/structured-outputs>
- **Google — Agent Development Kit (ADK) docs**: <https://google.github.io/adk-docs/>
- **Google — Vertex AI Agent Engine**: <https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/overview>
- **LangGraph** (open-source graph-shaped agent runtime): <https://langchain-ai.github.io/langgraph/>
- **LangGraph Platform**: <https://langchain-ai.github.io/langgraph/concepts/langgraph_platform/>
- **Microsoft — AutoGen framework**: <https://microsoft.github.io/autogen/>
- **Microsoft — Azure AI Foundry Agent Service**: <https://learn.microsoft.com/en-us/azure/ai-services/agents/overview>
- **Hugging Face — smolagents**: <https://huggingface.co/docs/smolagents/>
- **Model Context Protocol — specification**: <https://modelcontextprotocol.io/specification>
- **Model Context Protocol — authorization**: <https://modelcontextprotocol.io/specification/basic/authorization>

## Chapter 04 — When Not to Platformise

- **Team Topologies — team-interaction modes** (`collaboration`, `X-as-a-Service`, `facilitating`): <https://teamtopologies.com/key-concepts-content/team-interaction-modes>
- **ThoughtWorks — paved-road discussion** (secondary source, treats paved-road as pattern): <https://www.thoughtworks.com/insights/blog>
- **Netflix Technology Blog — "paved road" writing** (search for *paved road*; Netflix's engineering blog is one of the long-running public sources on the pattern): <https://netflixtechblog.com/>
- **Spotify Engineering — Backstage + platform patterns**: <https://engineering.atspotify.com/>
- **Anthropic — Building Effective Agents** (the "start simple" guidance that chapter 04 generalises from): <https://www.anthropic.com/research/building-effective-agents>

## Chapter 05 — OWASP Baseline for the Safety Plane

- **OWASP — Top 10 for Large Language Model Applications**: <https://owasp.org/www-project-top-10-for-large-language-model-applications/>
- **OWASP — GenAI Security Project (top-level)**: <https://genai.owasp.org/>
- **OWASP — Agentic AI Threats and Mitigations**: <https://genai.owasp.org/resource/agentic-ai-threats-and-mitigations/>
- **MITRE ATLAS** (adversarial-ML taxonomy; complementary to OWASP): <https://atlas.mitre.org/>
- **NIST — AI Risk Management Framework (AI 100-1)**: <https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.100-1.pdf>
- **NIST — Generative AI Profile (AI 600-1)**: <https://airc.nist.gov/AI_RMF_Knowledge_Base/AI_RMF/Uses/GenAI-Profile>
- **EU AI Act — regulation text** (Article 15 technical requirements; Article 72 post-market monitoring): <https://eur-lex.europa.eu/eli/reg/2024/1689/oj>
- **OpenTelemetry — Semantic Conventions for Generative AI** (the trace / evidence pipeline the regulatory evidence sits on): <https://opentelemetry.io/docs/specs/semconv/gen-ai/>

## Guardrail and safety tooling (previewed in chapter 05; depth in `mod-411`)

- **NVIDIA NeMo Guardrails**: <https://docs.nvidia.com/nemo/guardrails/>
- **Guardrails AI**: <https://www.guardrailsai.com/docs>
- **Meta — Llama Guard** (paper): <https://ai.meta.com/research/publications/llama-guard-llm-based-input-output-safeguard-for-human-ai-conversations/>
- **Google — ShieldGemma**: <https://ai.google.dev/gemma/docs/shieldgemma>
- **OpenAI — Moderation API**: <https://platform.openai.com/docs/guides/moderation>
- **Anthropic — Constitutional Classifiers (research)**: <https://www.anthropic.com/research/constitutional-classifiers>
- **Lakera Guard**: <https://www.lakera.ai/>
- **Microsoft Presidio** (PII detection and redaction): <https://microsoft.github.io/presidio/>

## Secondary / supporting references used across multiple chapters

- **Open Policy Agent (OPA) — docs**: <https://www.openpolicyagent.org/docs/latest/>
- **Cedar — policy language**: <https://docs.cedarpolicy.com/>
- **SPIFFE / SPIRE — non-human identity**: <https://spiffe.io/docs/>
- **Temporal — docs** (durable-execution substrate referenced in chapter 03 translation table): <https://docs.temporal.io/>
- **Dapr Workflows**: <https://docs.dapr.io/developing-applications/building-blocks/workflow/workflow-overview/>
- **Dapr Agents**: <https://docs.dapr.io/concepts/agents/>
- **Restate**: <https://docs.restate.dev/>
- **Inngest**: <https://www.inngest.com/docs>
- **Firecracker microVM** (sandbox substrate referenced in chapter 05): <https://firecracker-microvm.github.io/>
- **gVisor**: <https://gvisor.dev/docs/>
- **Kata Containers**: <https://katacontainers.io/docs/>
- **LiteLLM / LiteLLM Proxy** (model gateway substrate referenced in chapters 03 and 05): <https://docs.litellm.ai/>
- **Langfuse** (observability backend for GenAI telemetry): <https://langfuse.com/docs>
- **Arize Phoenix**: <https://docs.arize.com/phoenix>
- **UK AISI — Inspect eval framework**: <https://inspect.aisi.org.uk/>
- **Promptfoo**: <https://www.promptfoo.dev/docs/intro/>
- **A2A — Google Agent2Agent protocol**: <https://github.com/a2aproject/A2A>

## Notes on using these references

- Everything on this page is a *primary* or *authoritative-public* source. If you find a secondary blog post summarising one of them, go to the primary source — the guidance above is written against the primary versions and the summaries will lag.
- Both OWASP documents and the CNCF maturity model are versioned. Check the version at the top of each page before citing; the chapter prose treats the current public version as authoritative.
- The AICG job-requirement catalog at [`.aicg/job-requirements.json`](../../.aicg/job-requirements.json) is the normalised, machine-readable form of the citations above.
