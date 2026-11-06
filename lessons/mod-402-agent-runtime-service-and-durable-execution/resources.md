# Resources — mod-402 (Agent Runtime Service and Durable Execution)

All references on this page are primary / authoritative public sources at the time of writing (2026-11-06). Each resource is tagged with the chapter(s) it supports; several are cited across more than one. If you find a secondary blog post summarising one of these, go to the primary source — the chapter prose is written against the primary versions and summaries lag.

## Chapter 01 — The Runtime Plane and the Session as a Contract

- **CNCF — Platform Engineering Maturity Model** (background on the platform-as-a-product shape this chapter inherits from mod-401): <https://tag-app-delivery.cncf.io/whitepapers/platform-eng-maturity-model/>
- **Team Topologies — platform team pattern**: <https://teamtopologies.com/>
- **OpenTelemetry — Semantic Conventions for Generative AI** (the trace shape every session plane is measured in; preview for mod-408): <https://opentelemetry.io/docs/specs/semconv/gen-ai/>
- **Anthropic — Building Effective Agents** (the agent-authoring context the runtime hosts): <https://www.anthropic.com/research/building-effective-agents>
- **OpenAI — A Practical Guide to Building Agents** (same): <https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf>

## Chapter 02 — Session Lifecycle: Checkpoint, Resume, Timers, Cancellation, Context Compaction

### Durable progress / checkpoint primitives
- **Temporal — Workflows (replay, history, side effects)**: <https://docs.temporal.io/workflows>
- **Temporal — Timers**: <https://docs.temporal.io/develop/python/timers>
- **Temporal — Signals**: <https://docs.temporal.io/sending-messages#send-signal>
- **Temporal — Cancellation**: <https://docs.temporal.io/cancellation>
- **Dapr Workflows — overview**: <https://docs.dapr.io/developing-applications/building-blocks/workflow/workflow-overview/>
- **Dapr Workflows — features and concepts**: <https://docs.dapr.io/developing-applications/building-blocks/workflow/workflow-features-concepts/>
- **Restate — journal, invocations, awakeables**: <https://docs.restate.dev/concepts/invocations>
- **Inngest — durable functions and steps**: <https://www.inngest.com/docs/features/inngest-functions>

### Context management / compaction
- **Anthropic — Prompt caching** (a caching primitive compaction policies must be aware of): <https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching>
- **OpenAI — Managing context for the Responses API**: <https://platform.openai.com/docs/api-reference/responses>
- **LangGraph — Message handling / summarisation patterns**: <https://langchain-ai.github.io/langgraph/concepts/persistence/>

## Chapter 03 — Durable Execution Substrate Choice

- **Temporal — docs home**: <https://docs.temporal.io/>
- **Temporal — Python SDK**: <https://docs.temporal.io/develop/python>
- **Temporal — Determinism and non-deterministic errors**: <https://docs.temporal.io/workflows#deterministic-constraints>
- **Dapr Workflows — overview (restated)**: <https://docs.dapr.io/developing-applications/building-blocks/workflow/workflow-overview/>
- **Dapr Agents — concept**: <https://docs.dapr.io/concepts/agents/>
- **Restate — docs home**: <https://docs.restate.dev/>
- **Restate — Virtual Objects** (keyed, single-writer actor-ish primitive): <https://docs.restate.dev/concepts/services#virtual-objects>
- **Restate — Durable handlers and journal**: <https://docs.restate.dev/concepts/durable_execution>
- **Inngest — docs home**: <https://www.inngest.com/docs>
- **Inngest — Concurrency, throttling, rate limiting**: <https://www.inngest.com/docs/guides/concurrency>
- **Durable Task Framework** (Dapr Workflows lineage, useful background): <https://github.com/Azure/durabletask>

## Chapter 04 — Framework Session State: One Abstraction, Five Frameworks

### LangGraph
- **LangGraph — overview**: <https://langchain-ai.github.io/langgraph/>
- **LangGraph — Persistence (Checkpointer)**: <https://langchain-ai.github.io/langgraph/concepts/persistence/>
- **LangGraph Platform** (hosted variant): <https://langchain-ai.github.io/langgraph/concepts/langgraph_platform/>

### OpenAI
- **OpenAI — Responses API reference**: <https://platform.openai.com/docs/api-reference/responses>
- **OpenAI — Agents SDK (Python)**: <https://openai.github.io/openai-agents-python/>
- **OpenAI — Agents SDK sessions (Python)**: <https://openai.github.io/openai-agents-python/sessions/>
- **OpenAI — Structured Outputs**: <https://platform.openai.com/docs/guides/structured-outputs>

### Anthropic
- **Anthropic — Agent SDK overview**: <https://docs.anthropic.com/en/api/agent-sdk/overview>
- **Anthropic — Tool use (block semantics)**: <https://docs.anthropic.com/en/docs/build-with-claude/tool-use>
- **Anthropic — Prompt caching (restated)**: <https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching>

### Google
- **Google — Agent Development Kit (ADK)**: <https://google.github.io/adk-docs/>
- **Google — ADK sessions, state, memory**: <https://google.github.io/adk-docs/sessions/>
- **Google — Vertex AI Agent Engine**: <https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/overview>

### Microsoft
- **Microsoft — Agent Framework entry point** (verify current URL; <!-- needs-research: confirm canonical Microsoft Agent Framework documentation URL at your onboarding -->): <https://learn.microsoft.com/en-us/agent-framework/>
- **Microsoft — Azure AI Foundry Agent Service**: <https://learn.microsoft.com/en-us/azure/ai-services/agents/overview>
- **Microsoft — AutoGen** (predecessor / lineage): <https://microsoft.github.io/autogen/>
- **Microsoft — Semantic Kernel** (predecessor / lineage): <https://learn.microsoft.com/en-us/semantic-kernel/overview/>

### Abstraction patterns (background)
- **Model Context Protocol — specification** (adjacent abstraction, relevant to adapter boundaries): <https://modelcontextprotocol.io/specification>

## Chapter 05 — State Substrates and Worker Topologies

### Postgres + pgvector
- **PostgreSQL — official docs**: <https://www.postgresql.org/docs/>
- **pgvector** (vector-similarity extension): <https://github.com/pgvector/pgvector>
- **CloudNativePG** (Postgres operator for Kubernetes): <https://cloudnative-pg.io/docs/>
- **Crunchy Postgres Operator**: <https://access.crunchydata.com/documentation/postgres-operator/latest/>
- **Zalando Postgres Operator**: <https://postgres-operator.readthedocs.io/en/latest/>

### Redis / Valkey
- **Redis — docs**: <https://redis.io/docs/>
- **Valkey** (OSS fork maintained under Linux Foundation): <https://valkey.io/>
- **Redis Operator** (one of several): <https://github.com/OT-CONTAINER-KIT/redis-operator>

### Object storage
- **AWS S3 — User Guide**: <https://docs.aws.amazon.com/AmazonS3/latest/userguide/>
- **GCP Cloud Storage**: <https://cloud.google.com/storage/docs>
- **Azure Blob Storage**: <https://learn.microsoft.com/en-us/azure/storage/blobs/>
- **MinIO** (S3-compatible on-prem): <https://min.io/docs/>
- **Rook / Ceph** (for self-hosted object storage on Kubernetes): <https://rook.io/docs/rook/latest/>

### Compute fabric
- **Kubernetes — StatefulSet**: <https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/>
- **Kubernetes — Deployment**: <https://kubernetes.io/docs/concepts/workloads/controllers/deployment/>
- **Knative — docs**: <https://knative.dev/docs/>
- **KEDA — docs** (event-driven autoscaling, useful for queue-depth-driven worker scaling): <https://keda.sh/docs/>

### Model / tool gateway (preview of mod-406 / mod-403)
- **LiteLLM / LiteLLM Proxy** (rate-limit and quota primitives this chapter's admission control hands off to): <https://docs.litellm.ai/>
- **Envoy — rate limiting**: <https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/rate_limit_filter>
- **Portkey — AI gateway**: <https://portkey.ai/docs>

## Secondary / cross-cutting references

- **Google SRE Book — chapters on handling overload / graceful degradation** (background for chapter 05's shed ladder): <https://sre.google/sre-book/handling-overload/>
- **Google SRE Workbook — alerting on SLOs** (background for chapter 02's cancellation-propagation SLI and exercise-03's SLO definitions): <https://sre.google/workbook/alerting-on-slos/>
- **OpenTelemetry — Semantic Conventions for Generative AI** (restated): <https://opentelemetry.io/docs/specs/semconv/gen-ai/>
- **CloudEvents** (common event-envelope pattern worth internalising for signals and A2A): <https://cloudevents.io/>

## Notes on using these references

- Everything on this page is a *primary* or *authoritative-public* source. If the source has a version tag, read the version that matches the one you deploy against — API-level and determinism-level details in particular drift fast in 2026.
- The MS Agent Framework, Google ADK, Anthropic Agent SDK, and Restate entries are in a particularly fast-moving phase; before quoting a specific feature or primitive in an architecture memo or exercise artefact, verify against the current doc and mark `<!-- needs-research: ... -->` if it does not match this page.
- `JOB_REQUIREMENTS.md` at the track root and `.aicg/job-requirements.json` carry the normalised, machine-readable form of the track's citations.
