# Resources — mod-411 (Safety Plane and Guardrails as a Service)

All references on this page are primary / authoritative public sources at the time of writing (2026-11-06). Each resource is tagged with the chapter(s) it supports. Prefer the primary source over any blog summary — the OWASP catalogues re-rev, the EU AI Act gets implementing acts, and the guardrail-vendor product pages ship copy changes almost monthly.

## Chapter 01 — The Safety Plane and What It Owns

### Agent platform context
- **Anthropic — Building Effective Agents**: <https://www.anthropic.com/research/building-effective-agents>
- **OpenAI — A Practical Guide to Building Agents (PDF)**: <https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf>

### Platform engineering and operating model (cross-ref mod-401)
- **Skelton & Pais — Team Topologies**: <https://teamtopologies.com/>
- **CNCF — Platforms White Paper**: <https://tag-app-delivery.cncf.io/whitepapers/platforms/>

### OpenTelemetry semantic contract (cross-ref mod-408)
- **OpenTelemetry — Semantic Conventions for Generative AI**: <https://opentelemetry.io/docs/specs/semconv/gen-ai/>
- **OpenTelemetry — specification**: <https://opentelemetry.io/docs/specs/otel/>
- **W3C Trace Context**: <https://www.w3.org/TR/trace-context/>

## Chapter 02 — The Guardrail Toolkit Landscape

### Orchestration frameworks
- **NVIDIA — NeMo Guardrails documentation**: <https://docs.nvidia.com/nemo/guardrails/latest/index.html>
- **NVIDIA — NeMo Guardrails on GitHub**: <https://github.com/NVIDIA/NeMo-Guardrails>
- **Guardrails AI — documentation**: <https://www.guardrailsai.com/docs>
- **Guardrails AI Hub**: <https://hub.guardrailsai.com/>

### Open-source safety classifiers
- **Meta — Llama Guard paper**: <https://ai.meta.com/research/publications/llama-guard-llm-based-input-output-safeguard-for-human-ai-conversations/>
- **Meta — Llama Guard 3 8B model card (Hugging Face)**: <https://huggingface.co/meta-llama/Llama-Guard-3-8B>
- **Meta — Prompt Guard 86M model card (Hugging Face)**: <https://huggingface.co/meta-llama/Prompt-Guard-86M>
- **Meta — Llama Community License**: <https://llama.meta.com/llama3/license/>
- **Google — ShieldGemma model card**: <https://ai.google.dev/gemma/docs/shieldgemma/model_card>
- **Google — ShieldGemma paper (arXiv)**: <https://arxiv.org/abs/2407.21772>
- **Google — Gemma Terms of Use**: <https://ai.google.dev/gemma/terms>

### Model-provider-native guardrails
- **OpenAI — Moderation guide**: <https://platform.openai.com/docs/guides/moderation>
- **OpenAI — `omni-moderation-latest` model**: <https://platform.openai.com/docs/models/omni-moderation-latest>
- **Anthropic — Constitutional Classifiers research**: <https://www.anthropic.com/research/constitutional-classifiers>
- **Anthropic — Collective Constitutional AI (research)**: <https://www.anthropic.com/research/collective-constitutional-ai-aligning-a-language-model-with-public-input>
- **Google — Gemini safety settings**: <https://ai.google.dev/gemini-api/docs/safety-settings>

### Cloud-native guardrail services
- **AWS — Bedrock Guardrails**: <https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html>
- **Azure — AI Content Safety**: <https://learn.microsoft.com/en-us/azure/ai-services/content-safety/overview>
- **Google Cloud — Model Armor**: <https://cloud.google.com/security-command-center/docs/model-armor-overview>

### PII detection
- **Microsoft — Presidio documentation**: <https://microsoft.github.io/presidio/>
- **Microsoft — Presidio on GitHub**: <https://github.com/microsoft/presidio>
- **Google Cloud — Data Loss Prevention (DLP)**: <https://cloud.google.com/dlp>
- **Azure — AI Language PII detection**: <https://learn.microsoft.com/en-us/azure/ai-services/language-service/personally-identifiable-information/overview>
- **AWS — Comprehend PII**: <https://docs.aws.amazon.com/comprehend/latest/dg/pii.html>

### Vendor products (chapter 02 and chapter 07)
- **Lakera — documentation**: <https://docs.lakera.ai/>
- **Lakera — Guard product page**: <https://www.lakera.ai/products/lakera-guard>
- **Robust Intelligence (Cisco) — platform**: <https://www.robustintelligence.com/>
- **Arthur — Shield**: <https://www.arthur.ai/arthur-shield>
- **HiddenLayer — AISec Platform**: <https://hiddenlayer.com/aisec-platform/>
- **Calypso AI**: <https://calypsoai.com/>

<!-- needs-research: vendor product pages, SKUs, and pricing shift frequently. Pin to the exact URL and dated snapshot when a specific product capability is referenced. -->

## Chapter 03 — OWASP LLM Top 10 and Agentic AI Threats

### OWASP catalogues (primary)
- **OWASP GenAI Security Project — landing page**: <https://genai.owasp.org/>
- **OWASP — Top 10 for LLM Applications (project page)**: <https://genai.owasp.org/llm-top-10/>
- **OWASP — LLM01: Prompt Injection**: <https://genai.owasp.org/llmrisk/llm01-prompt-injection/>
- **OWASP — Agentic AI Threats and Mitigations**: <https://genai.owasp.org/resource/agentic-ai-threats-and-mitigations/>

### Cross-catalogue references
- **MITRE ATLAS — Adversarial Threat Landscape for AI Systems**: <https://atlas.mitre.org/>
- **MITRE ATT&CK**: <https://attack.mitre.org/>
- **NIST — AI 100-2 Adversarial Machine Learning: A Taxonomy and Terminology**: <https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.100-2e2023.pdf>
- **NIST — AI Risk Management Framework (AI RMF)**: <https://www.nist.gov/itl/ai-risk-management-framework>
- **NIST — AI RMF 1.0 playbook**: <https://airc.nist.gov/AI_RMF_Knowledge_Base/Playbook>

<!-- needs-research: the exact string form of OWASP category identifiers (e.g., `LLM01:2025:prompt_injection` vs. `LLM01_prompt_injection@2025`) has varied across community implementations. The official OWASP GenAI Security Project page is authoritative — re-verify before pinning. -->

## Chapter 04 — Prompt Injection Defence at Platform Scope

### Primary research and surveys
- **Simon Willison — Prompt injection explained**: <https://simonwillison.net/2022/Sep/12/prompt-injection/>
- **Simon Willison — ongoing prompt-injection coverage**: <https://simonwillison.net/tags/prompt-injection/>
- **Greshake et al. — Not what you've signed up for: Compromising Real-World LLM-Integrated Applications with Indirect Prompt Injection (arXiv)**: <https://arxiv.org/abs/2302.12173>
- **Perez & Ribeiro — Ignore Previous Prompt: Attack Techniques For Language Models (arXiv)**: <https://arxiv.org/abs/2211.09527>
- **Microsoft — Defending Against Indirect Prompt Injection Attacks (Spotlighting paper, arXiv)**: <https://arxiv.org/abs/2403.14720>

### Red-team corpora and probes
- **Lakera — Gandalf prompt-injection challenge**: <https://gandalf.lakera.ai/>
- **Microsoft — PromptBench**: <https://github.com/microsoft/promptbench>
- **Microsoft — LLMail-Inject dataset**: <https://aka.ms/LLMail-Inject>
- **garak — LLM vulnerability scanner**: <https://garak.ai/>
- **NVIDIA — garak on GitHub**: <https://github.com/NVIDIA/garak>

### Standards and reviews
- **NIST — AI 100-2 (adversarial ML taxonomy, cross-ref chapter 03)**: <https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.100-2e2023.pdf>
- **OWASP — LLM01: Prompt Injection (cross-ref chapter 03)**: <https://genai.owasp.org/llmrisk/llm01-prompt-injection/>

## Chapter 05 — The Standardised SafetyRecord Schema

### OpenTelemetry (cross-ref mod-408)
- **OpenTelemetry — Semantic Conventions for Generative AI**: <https://opentelemetry.io/docs/specs/semconv/gen-ai/>
- **OpenTelemetry — Span events**: <https://opentelemetry.io/docs/concepts/signals/traces/#span-events>
- **W3C — Trace Context**: <https://www.w3.org/TR/trace-context/>

### JSON Schema
- **JSON Schema — Draft 2020-12**: <https://json-schema.org/draft/2020-12/release-notes>
- **JSON Schema — Learn**: <https://json-schema.org/learn/getting-started-step-by-step>

### EU AI Act (primary legal text)
- **EU AI Act — Regulation (EU) 2024/1689 on EUR-Lex (English)**: <https://eur-lex.europa.eu/eli/reg/2024/1689/oj/eng>
- **EU AI Act — Article 15 (Accuracy, robustness and cybersecurity)**: <https://artificialintelligenceact.eu/article/15/>
- **EU AI Act — Article 72 (Post-market monitoring)**: <https://artificialintelligenceact.eu/article/72/>
- **EU AI Act — Article 73 (Reporting of serious incidents)**: <https://artificialintelligenceact.eu/article/73/>
- **EU AI Act — Article 12 (Record-keeping)**: <https://artificialintelligenceact.eu/article/12/>
- **EU AI Act — Article 11 (Technical documentation) and Annex IV**: <https://artificialintelligenceact.eu/article/11/>
- **EU AI Act — Article 6 (Classification) and Annex III**: <https://artificialintelligenceact.eu/article/6/>
- **European Commission — AI Act page**: <https://digital-strategy.ec.europa.eu/en/policies/regulatory-framework-ai>
- **European Commission — General-Purpose AI Code of Practice**: <https://digital-strategy.ec.europa.eu/en/policies/general-purpose-ai-code-practice>

### GDPR cross-references
- **GDPR — Regulation (EU) 2016/679 on EUR-Lex**: <https://eur-lex.europa.eu/eli/reg/2016/679/oj>
- **GDPR — Article 32 (Security of processing)**: <https://gdpr-info.eu/art-32-gdpr/>

### Integrity and provenance (for the content-hash / tamper-evident log)
- **Sigstore — overview**: <https://www.sigstore.dev/>
- **Sigstore — Rekor (transparency log)**: <https://docs.sigstore.dev/rekor/overview/>

<!-- needs-research: The European Commission is expected to publish implementing acts under Article 72(3) specifying template and operational details for the post-market monitoring plan. Re-verify against the latest EUR-Lex text and member-state regulator guidance before shipping an export. -->

## Chapter 06 — Measuring Guardrail Effectiveness

### Benchmark and red-team datasets
- **Lakera — Gandalf public corpus (cross-ref chapter 04)**: <https://gandalf.lakera.ai/>
- **Microsoft — PromptBench**: <https://github.com/microsoft/promptbench>
- **Microsoft — LLMail-Inject**: <https://aka.ms/LLMail-Inject>
- **garak — vulnerability scanner probes**: <https://garak.ai/>
- **LMSYS — Chatbot Arena (for benign baselines)**: <https://lmarena.ai/>
- **SimpleQA (OpenAI benchmark)**: <https://openai.com/index/introducing-simpleqa/>

### SLO / SLI methodology (cross-ref mod-410)
- **Google — SRE Book, Service Level Objectives (chapter 4)**: <https://sre.google/sre-book/service-level-objectives/>
- **Google — SRE Workbook, Implementing SLOs (chapter 2)**: <https://sre.google/workbook/implementing-slos/>
- **Google — SRE Workbook, Alerting on SLOs (chapter 5)**: <https://sre.google/workbook/alerting-on-slos/>

### Judge and evaluation frameworks (cross-ref mod-407)
- **UK AISI — Inspect**: <https://inspect.aisi.org.uk/>
- **Promptfoo — documentation**: <https://www.promptfoo.dev/docs/>
- **Langfuse — datasets and evaluation**: <https://langfuse.com/docs/datasets>
- **Arize Phoenix**: <https://arize.com/docs/phoenix/>

## Chapter 07 — Build-vs-Buy for Guardrail Stacks

### Vendor product pages (cross-ref chapter 02)
- **Lakera**: <https://www.lakera.ai/>
- **Robust Intelligence (now part of Cisco)**: <https://www.robustintelligence.com/>
- **Cisco — Robust Intelligence acquisition announcement**: <https://blogs.cisco.com/news/cisco-announces-intent-to-acquire-robust-intelligence>
- **Arthur**: <https://www.arthur.ai/>
- **HiddenLayer**: <https://hiddenlayer.com/>
- **Calypso AI**: <https://calypsoai.com/>

### Cloud-native comparisons
- **AWS — Bedrock Guardrails pricing**: <https://aws.amazon.com/bedrock/pricing/>
- **Azure — AI Content Safety pricing**: <https://azure.microsoft.com/en-us/pricing/details/ai-content-safety/>
- **Google Cloud — Model Armor pricing / overview**: <https://cloud.google.com/security-command-center/docs/model-armor-overview>

### Operational references
- **OpenAI — Moderation (free / rate-limit terms)**: <https://platform.openai.com/docs/guides/moderation>
- **OpenAI — pricing**: <https://openai.com/api/pricing/>
- **Anthropic — pricing**: <https://www.anthropic.com/pricing#anthropic-api>

<!-- needs-research: vendor pricing (Lakera, Robust Intelligence, Arthur, HiddenLayer, Calypso) is often not posted publicly or shifts quarterly. For a production decision, obtain a current quote directly and pin the quote date in the decision document. -->

## Cross-cutting references

### Within this track
- **mod-402 — Agent Runtime Service and Durable Execution**: `../mod-402-agent-runtime-service-and-durable-execution/`
- **mod-403 — MCP Registry and Tool Gateway**: `../mod-403-mcp-registry-and-tool-gateway/`
- **mod-404 — Sandboxed Code Execution Plane**: `../mod-404-sandboxed-code-execution-plane/`
- **mod-405 — Tool and Permission Policy Plane**: `../mod-405-tool-and-permission-policy-plane/`
- **mod-406 — Model Gateway and Routing**: `../mod-406-model-gateway-and-routing/`
- **mod-407 — Shared Evaluation Platform and Regression Gating**: `../mod-407-evaluation-platform-and-regression-gating/`
- **mod-408 — Observability and OpenTelemetry for Agent Fleets**: `../mod-408-observability-and-opentelemetry-for-agents/`
- **mod-409 — Agent-to-Agent Routing and Inter-Agent Service Mesh**: `../mod-409-agent-to-agent-routing-and-service-mesh/`
- **mod-410 — Reliability Engineering and SRE for the Agent Platform**: `../mod-410-reliability-engineering-and-sre-for-agents/`
- **mod-412 — Paved-Road SDKs and Developer Experience**: `../mod-412-paved-road-sdks-and-developer-experience/`
- **mod-413 — Platform Leadership and Operating Cadence**: `../mod-413-platform-leadership-and-operating-cadence/`

### Role boundaries (within the Agent Career Curriculum ecosystem)
- **Track overview (this repository)**: `../../README.md`
- **Job requirements (citations the curriculum writes against)**: `../../JOB_REQUIREMENTS.md`

## Notes on using these references

- OWASP's Top 10 for LLM Applications re-rolls — the "2025" rev is the current labelling at the time of writing; prior community implementations referenced the "2023" rev. Any reference to a specific LLM0X identifier in a production `SafetyRecord` should state the rev and the mapping-file pin.
- The OWASP Agentic AI Threats and Mitigations catalogue is newer than the Top 10 and has evolved its taxonomy more recently. The T-series identifiers used in chapter 03 reflect the current project publication; re-verify against the OWASP GenAI Security Project's page when pinning a production mapping.
- The EU AI Act is Regulation (EU) 2024/1689. The regulation entered into force on 1 August 2024 with phased application dates; Article 72 obligations on providers of high-risk AI systems come into application alongside the broader high-risk-system rules per the regulation's timeline. Implementing acts expected under Article 72(3) will clarify operational details — check with counsel and the latest EUR-Lex text before relying on a specific operational interpretation.
- The guardrail-tooling landscape moves fast. Llama Guard, ShieldGemma, NeMo Guardrails, Guardrails AI, Presidio, OpenAI Moderation, and the vendor APIs each ship updates on a monthly-to-quarterly cadence. Chapter 02's engine capabilities and chapter 06's performance observations should be re-benchmarked against the current release before shipping a production artefact. Pin the engine version string in every `SafetyRecord`.
- The vendor product capabilities and pricing cited in chapter 07 are a snapshot. Vendor contracts and SKUs change; the paved-road decision document is the versioned artefact; the resources on this page are inputs, not authority.
- Mark `<!-- needs-research: ... -->` on any derived artefact where a citation on this page does not match the version you are shipping against. The flag convention is consistent with mod-403 / 404 / 405 / 406 / 407 / 408 / 409 / 410; it blocks auto-merge, which is cheaper than silently guessing.
