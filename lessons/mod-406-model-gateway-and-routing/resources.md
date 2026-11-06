# Resources — mod-406 (Model Gateway and Routing for Agent Workloads)

All references on this page are primary / authoritative public sources at the time of writing (2026-11-06). Each resource is tagged with the chapter(s) it supports. Prefer the primary source over any blog summary — model-provider pricing, caching mechanics, gateway-tool features, and semantic conventions in this space move quarter-by-quarter and secondary sources lag.

## Chapter 01 — The Model-Gateway Slice and Its Obligations

### Platform-engineering background
- **CNCF — Platform Engineering Maturity Model**: <https://tag-app-delivery.cncf.io/whitepapers/platform-eng-maturity-model/>
- **CNCF — Platforms White Paper**: <https://tag-app-delivery.cncf.io/whitepapers/platforms/>
- **Google SRE Book — Request Latency and the Four Golden Signals**: <https://sre.google/sre-book/monitoring-distributed-systems/>

### Agent-platform guidance (cross-ref mod-401)
- **Anthropic — Building Effective Agents**: <https://www.anthropic.com/research/building-effective-agents>
- **OpenAI — A Practical Guide to Building Agents**: <https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf>

### Threat-model artefacts (cross-ref mod-405)
- **OWASP Top 10 for LLM Applications**: <https://genai.owasp.org/llm-top-10/>
- **OWASP Agentic AI — Threats and Mitigations (Agentic Security Initiative)**: <https://genai.owasp.org/initiatives/#agentic-security-initiative>
- **MITRE ATLAS**: <https://atlas.mitre.org/>

## Chapter 02 — Multi-Provider Routing and the Provider Abstraction

### Provider APIs — official references
- **OpenAI — Chat Completions API**: <https://platform.openai.com/docs/api-reference/chat>
- **OpenAI — Responses API**: <https://platform.openai.com/docs/api-reference/responses>
- **OpenAI — Structured Outputs**: <https://platform.openai.com/docs/guides/structured-outputs>
- **OpenAI — Function Calling**: <https://platform.openai.com/docs/guides/function-calling>
- **Anthropic — Messages API**: <https://docs.anthropic.com/en/api/messages>
- **Anthropic — Tool Use**: <https://docs.anthropic.com/en/docs/build-with-claude/tool-use>
- **Anthropic — Prompt Caching**: <https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching>
- **Google — Gemini API Reference**: <https://ai.google.dev/api>
- **Google — Gemini Function Calling**: <https://ai.google.dev/gemini-api/docs/function-calling>
- **Google — cachedContents API**: <https://ai.google.dev/gemini-api/docs/caching>
- **AWS — Bedrock Runtime API (InvokeModel / Converse)**: <https://docs.aws.amazon.com/bedrock/latest/APIReference/>
- **AWS — Bedrock Converse API Overview**: <https://docs.aws.amazon.com/bedrock/latest/userguide/conversation-inference.html>
- **Azure — OpenAI Service REST API reference**: <https://learn.microsoft.com/en-us/azure/ai-services/openai/reference>

### Self-hosted inference (cross-ref `ai-infra-inference-engineer`, chapter 07)
- **vLLM — OpenAI-Compatible Server**: <https://docs.vllm.ai/en/latest/serving/openai_compatible_server.html>
- **vLLM — repository**: <https://github.com/vllm-project/vllm>
- **SGLang — repository**: <https://github.com/sgl-project/sglang>
- **TensorRT-LLM — repository**: <https://github.com/NVIDIA/TensorRT-LLM>
- **Hugging Face — Text Generation Inference**: <https://github.com/huggingface/text-generation-inference>
- **Ollama — repository**: <https://github.com/ollama/ollama>

## Chapter 03 — Fallback, Retry, and the Tool-Calling Loop

### Retry and resilience patterns
- **AWS Builder's Library — Timeouts, retries, and backoff with jitter**: <https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/>
- **AWS Builder's Library — Avoiding fallback in distributed systems**: <https://aws.amazon.com/builders-library/avoiding-fallback-in-distributed-systems/>
- **Google SRE Workbook — Addressing Cascading Failures**: <https://sre.google/sre-book/addressing-cascading-failures/>

### Provider error-handling pages
- **OpenAI — API error codes and handling**: <https://platform.openai.com/docs/guides/error-codes>
- **OpenAI — Rate limits**: <https://platform.openai.com/docs/guides/rate-limits>
- **Anthropic — Errors**: <https://docs.anthropic.com/en/api/errors>
- **Anthropic — Rate limits**: <https://docs.anthropic.com/en/api/rate-limits>
- **Google — Gemini API error codes**: <https://ai.google.dev/gemini-api/docs/troubleshooting>

### HTTP retry primitives
- **RFC 9110 — HTTP Semantics (incl. `Retry-After`)**: <https://datatracker.ietf.org/doc/html/rfc9110>
- **RFC 6585 — Additional HTTP Status Codes (incl. 429)**: <https://datatracker.ietf.org/doc/html/rfc6585>

### Streaming and SSE
- **WHATWG — Server-Sent Events (EventSource)**: <https://html.spec.whatwg.org/multipage/server-sent-events.html>

## Chapter 04 — Quotas, Burst Budgets, and Cost Attribution

### Rate limiting and bucket algorithms
- **Lamport et al. — The Token Bucket algorithm** (background): <https://en.wikipedia.org/wiki/Token_bucket> (survey — follow cites for primary sources)
- **Stripe Engineering — Scaling your API with rate limiters** (useful operational write-up): <https://stripe.com/blog/rate-limiters>

### Observability — OpenTelemetry (cross-ref mod-408)
- **OpenTelemetry — Semantic Conventions for Generative AI**: <https://opentelemetry.io/docs/specs/semconv/gen-ai/>
- **OpenTelemetry — Traces specification**: <https://opentelemetry.io/docs/specs/otel/trace/>
- **OpenTelemetry — Metrics specification**: <https://opentelemetry.io/docs/specs/otel/metrics/>
- **OpenTelemetry — Logs specification**: <https://opentelemetry.io/docs/specs/otel/logs/>

### FinOps and provider pricing
- **FinOps Foundation — Framework**: <https://www.finops.org/framework/>
- **OpenAI — Pricing**: <https://openai.com/api/pricing/>
- **Anthropic — Pricing**: <https://www.anthropic.com/pricing>
- **Google — Gemini API pricing**: <https://ai.google.dev/pricing>
- **AWS — Bedrock pricing**: <https://aws.amazon.com/bedrock/pricing/>
- **Azure — OpenAI Service pricing**: <https://azure.microsoft.com/en-us/pricing/details/cognitive-services/openai-service/>
<!-- needs-research: prices on each provider page move quarterly and sometimes faster. Re-verify every entry you cite in a billing-event or price-regression test. -->

### Streaming platforms for billing events
- **Apache Kafka — documentation**: <https://kafka.apache.org/documentation/>
- **Redpanda — documentation**: <https://docs.redpanda.com/>
- **Apache Pulsar — documentation**: <https://pulsar.apache.org/docs/next/>
- **NATS JetStream**: <https://docs.nats.io/nats-concepts/jetstream>

## Chapter 05 — Prompt-Cache Coordination Across Tenants

### Provider prompt-caching documentation
- **Anthropic — Prompt caching**: <https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching>
- **OpenAI — Prompt caching**: <https://platform.openai.com/docs/guides/prompt-caching>
- **Google — cachedContents / context caching**: <https://ai.google.dev/gemini-api/docs/caching>
- **AWS — Bedrock prompt caching**: <https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-caching.html>
<!-- needs-research: cache TTLs, minimum-prefix thresholds, write/read multipliers, and billing surfaces for each provider have all moved across 2024-2026. Verify at reading time. -->

### Consistent hashing and partitioning
- **Karger et al. — Consistent Hashing (STOC 1997)**: <https://dl.acm.org/doi/10.1145/258533.258660>
- **Google — Jump Consistent Hash (2014)**: <https://arxiv.org/abs/1406.2294>

### KV-cache and continuous batching (handoff background — cross-ref `ai-infra-inference-engineer`)
- **Kwon et al. — Efficient Memory Management for Large Language Model Serving with PagedAttention** (vLLM paper, SOSP 2023): <https://arxiv.org/abs/2309.06180>
- **vLLM — PagedAttention explainer**: <https://docs.vllm.ai/en/latest/design/arch_overview.html>
- **Zheng et al. — SGLang: Efficient Execution of Structured Language Model Programs**: <https://arxiv.org/abs/2312.07104>

### Multi-tenant isolation (cross-ref mod-405 chapter 03)
- **NIST — Guide to Attribute Based Access Control (SP 800-162)**: <https://csrc.nist.gov/pubs/sp/800/162/upd1/final>

## Chapter 06 — Structured Output and PII Redaction at the Gateway

### Structured-output and guided decoding
- **OpenAI — Structured Outputs (strict mode)**: <https://platform.openai.com/docs/guides/structured-outputs>
- **Google — Gemini structured output / responseSchema**: <https://ai.google.dev/gemini-api/docs/structured-output>
- **Anthropic — Tool use for structured output**: <https://docs.anthropic.com/en/docs/build-with-claude/tool-use>
- **vLLM — Guided decoding (grammar / regex / JSON schema)**: <https://docs.vllm.ai/en/latest/features/structured_outputs.html>
- **Outlines — repository** (structured generation library): <https://github.com/dottxt-ai/outlines>
- **lm-format-enforcer — repository**: <https://github.com/noamgat/lm-format-enforcer>
- **JSON Schema — specification (Draft 2020-12)**: <https://json-schema.org/draft/2020-12/schema>
- **JSON Schema — docs**: <https://json-schema.org/learn/getting-started-step-by-step>

### PII detection libraries
- **Microsoft Presidio — home**: <https://microsoft.github.io/presidio/>
- **Microsoft Presidio — repository**: <https://github.com/microsoft/presidio>
- **Google Cloud — Sensitive Data Protection (DLP)**: <https://cloud.google.com/sensitive-data-protection>
- **AWS — Amazon Comprehend PII detection**: <https://docs.aws.amazon.com/comprehend/latest/dg/pii.html>
- **AWS — Amazon Macie**: <https://aws.amazon.com/macie/>

### Secret-scanning primitives
- **GitHub — Secret scanning patterns**: <https://docs.github.com/en/code-security/secret-scanning>
- **TruffleHog — repository**: <https://github.com/trufflesecurity/trufflehog>
- **Gitleaks — repository**: <https://github.com/gitleaks/gitleaks>

### Safety-plane cross-references (cross-ref mod-411)
- **OWASP Top 10 for LLM Applications**: <https://genai.owasp.org/llm-top-10/>
- **NIST — AI Risk Management Framework (Generative AI Profile)**: <https://www.nist.gov/itl/ai-risk-management-framework>

## Chapter 07 — Build vs. Buy and the Inference-Engineer Handoff

### LLM gateway tools
- **LiteLLM — documentation**: <https://docs.litellm.ai/>
- **LiteLLM — repository**: <https://github.com/BerriAI/litellm>
- **Portkey — documentation**: <https://portkey.ai/docs/>
- **Portkey — Gateway (OSS) repository**: <https://github.com/Portkey-AI/gateway>

### General-purpose API gateways with AI features
- **Envoy AI Gateway — documentation**: <https://aigateway.envoyproxy.io/>
- **Envoy AI Gateway — repository**: <https://github.com/envoyproxy/ai-gateway>
- **Envoy Gateway — documentation**: <https://gateway.envoyproxy.io/>
- **Kong AI Gateway — documentation**: <https://docs.konghq.com/gateway/latest/ai-gateway/>
- **Kong Gateway — documentation**: <https://docs.konghq.com/gateway/latest/>

### Edge AI gateways
- **Cloudflare AI Gateway — documentation**: <https://developers.cloudflare.com/ai-gateway/>

### Managed inference platforms (consult for tier-3 delegation)
- **AWS Bedrock — User Guide**: <https://docs.aws.amazon.com/bedrock/latest/userguide/what-is-bedrock.html>
- **AWS Bedrock — Guardrails**: <https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html>
- **Azure AI Foundry — docs**: <https://learn.microsoft.com/en-us/azure/ai-foundry/>
- **Google Vertex AI — Generative AI documentation**: <https://cloud.google.com/vertex-ai/generative-ai/docs>

### Self-hosted inference (the handoff surface — cross-ref `ai-infra-inference-engineer`)
- **vLLM — OpenAI-compatible server**: <https://docs.vllm.ai/en/latest/serving/openai_compatible_server.html>
- **SGLang — OpenAI-compatible API**: <https://docs.sglang.ai/>
- **TensorRT-LLM — Triton inference server backend**: <https://github.com/triton-inference-server/tensorrtllm_backend>
- **NVIDIA NIM (Inference Microservices)**: <https://docs.nvidia.com/nim/>

### Workload identity (cross-ref mod-405 chapter 04)
- **SPIFFE — home**: <https://spiffe.io/>
- **SPIRE — repository**: <https://github.com/spiffe/spire>

### CNCF landscape and governance
- **CNCF — Platforms White Paper**: <https://tag-app-delivery.cncf.io/whitepapers/platforms/>
- **CNCF Landscape — AI + Data**: <https://landscape.cncf.io/>

## Cross-cutting references

- **Model Context Protocol — specification** (cross-ref mod-403): <https://modelcontextprotocol.io/specification>
- **Anthropic — Building Effective Agents**: <https://www.anthropic.com/research/building-effective-agents>
- **OpenAI — A Practical Guide to Building Agents**: <https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf>
- **OpenTelemetry — Semantic Conventions for Generative AI**: <https://opentelemetry.io/docs/specs/semconv/gen-ai/>
- **JOB_REQUIREMENTS.md** (at the track root): the normalised list of citations the curriculum is written against.

## Notes on using these references

- Every entry on this page is a **primary** or **authoritative-public** source at the time of writing. Model-provider pricing pages, caching mechanics documentation, and gateway-tool releases are the fastest-moving surfaces on the track; if the source has a version tag — SDK version, API version, pricing-page revision — read the version that matches what you deploy against.
- The specific feature sets of LiteLLM, Portkey, Envoy AI Gateway, Kong AI Gateway, and Cloudflare AI Gateway have shipped major changes across 2024–2026 and will continue to. Before quoting a specific feature in a build-vs-buy decision, confirm against the vendor's current documentation and release notes.
- Provider pricing and caching discount rates move independently of the surrounding product. A price-table entry you cite today may be stale next quarter; the price-regression alarm in chapter 04 is the operational mitigation.
- Regulatory artefacts (EU AI Act, state privacy laws, sectoral rules) are refined by delegated acts and amendments. Before quoting a specific record-keeping horizon or compliance requirement in a billing-event retention policy, verify against the latest text and against counsel at your organisation.
- Mark `<!-- needs-research: ... -->` on any derived artefact where a citation on this page does not match the version you are shipping against. The flag convention is consistent with mod-403 / 404 / 405; it blocks auto-merge, and that is cheaper than silently guessing.
