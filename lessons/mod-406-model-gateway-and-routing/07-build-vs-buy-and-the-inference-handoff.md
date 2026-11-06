# Build vs. Buy and the Inference-Engineer Handoff

Chapters 01–06 built the model gateway from the inside out — slice, routing, retry, quotas, cache, schema, redaction. That is the content of the plane. This chapter steps back to the portfolio question: *which of these pieces should the platform write, which should the platform buy, and which should the platform run in a hybrid of both?* It then walks the downstream handoff to the engineer who owns the inference-serving substrate the gateway may be routing to.

Vendor positioning in this space moves fast; the specific product features in this chapter will decay within quarters. The goal is to leave you with the right **questions** to run the evaluation yourself when the time comes, not with a vendor recommendation that ages in a filing cabinet.

## The portfolio frame

Mirror the frame from mod-403 chapter 06: treat the gateway's responsibilities as a portfolio with three tiers.

- **Tier 1 — Own.** The capabilities that are either (a) tenant-boundary-sensitive, (b) differentiating for the platform, (c) tied to internal systems the vendor cannot see, or (d) require coordinated state (quotas, cache namespaces, billing events) with internal consumers. These must be first-class gateway code regardless of what you buy for the rest.
- **Tier 2 — Buy and compose.** The capabilities that are standard, well-understood, and already implemented well in an off-the-shelf tool — provider abstraction, retry policy, OpenAI-compatible wire shape, OpenTelemetry instrumentation. Running a hardened version of a popular library beats re-implementing it at mid-platform scale.
- **Tier 3 — Delegate upstream.** The capabilities that belong to another plane or role. Inference-serving substrate is `ai-infra-inference-engineer` (level 35). Policy decisions are the policy plane. Content-safety on output is the safety plane. The gateway does not reimplement them.

A healthy platform runs all three tiers simultaneously. The failure modes are:

- **All tier 1.** Every capability written in-house. Takes 12–24 engineer-months and ends up with the same silently-wrong edge cases every other team's in-house gateway has. Tempting but expensive.
- **All tier 2.** A commercial gateway operates the whole plane, including the tenant-boundary-sensitive pieces. Risk: when a tenant-isolation bug or a cache-leak bug hits the vendor, you own the breach and the vendor owns the fix cycle. Common in prototypes and early-stage startups; collapses at scale.
- **All tier 3.** The platform owns nothing and routes directly to providers through the agent SDK. Every incident is an agent-code change. This is the "we don't have a gateway" state chapter 01 described.

## The must-own tier

Four capabilities are always tier 1. If a vendor claims to own these, re-read the SLA before trusting the claim.

- **Tenant-scoped cache namespace enforcement.** The cross-tenant cache leak (chapter 05) is the single most specific harm a model gateway can cause. The invariant — namespace composition, key derivation, egress check on request body — must live in code you can audit and roll. A vendor abstracting this away is a vendor whose incident response is your incident.
- **Per-tenant quota and burst budget state.** A quota that is eventually-consistent is a quota that leaks on the day a tenant abuses a race condition. The enforcement store and the admission logic are tier 1.
- **Billing-event schema.** The schema that feeds your FinOps, your chargeback, your price-regression alarms, and your compliance stream is yours. You may emit it from a library that handles OpenAI's wire shape, but the event itself is yours.
- **PII and secret egress enforcement.** The one place "did this request actually leave the trust boundary with X in it" is answerable is the gateway. Enforcement runs where you can reason about it.

Four capabilities are usually tier 2. A hardened library covers 80% of them out of the box; the remaining 20% is integration with the tier 1 surface.

- **Provider adapters.** OpenAI, Anthropic, Google, Bedrock, Azure OpenAI, self-hosted OpenAI-compatible endpoints. Standard translations with well-known edge cases. Buy the library; own the config.
- **Retry policy skeleton.** Exponential backoff, Retry-After honouring, timeouts, circuit-breaker state machine — all standard. The tool-loop invariants from chapter 03 are the gateway's job to layer on top.
- **OpenAI-compatible wire shape.** Many libraries expose this as a drop-in for agent SDKs; it accelerates adoption. Treat it as a feature, not a strategy.
- **Instrumentation glue.** OpenTelemetry trace emission, Prometheus metrics, log formatting — buy the plumbing; configure the naming.

Everything else is a case-by-case call; the questions below drive it.

## The current vendor landscape — categories, not endorsements

Four product categories cover the market at the time of writing. A single vendor often spans two.

- **LLM gateway libraries (self-hosted or SaaS).** Software you run (as a library or as a service) that implements provider abstraction, routing, retry, caching, and observability. Examples: **LiteLLM** (CNCF-adjacent, OSS Python library + Python proxy service), **Portkey** (SaaS + OSS gateway written for the AI stack specifically). This is the category most mod-406 content rhymes with.
- **General-purpose API gateways with AI extensions.** Mature API gateways that have added AI-routing filters as a plugin on top of their core traffic-management features. Examples: **Envoy AI Gateway** (CNCF Envoy ecosystem; adds an `ai-gateway` CRD and filters for provider routing, token-based rate limiting, and semantic routing), **Kong AI Gateway** (plugin suite on top of Kong Gateway that adds AI proxy, prompt decoration, prompt guard, and semantic routing).
- **Edge AI gateways.** Cloud-provider-hosted gateways that proxy from the edge. Examples: **Cloudflare AI Gateway** (routes requests across providers, logs, caches, applies rate limits, bills through Cloudflare's billing). Deployment shape is managed-SaaS with per-request pricing.
- **Managed inference platforms with gateway features.** Products that own the inference substrate and layer a gateway on it. Examples: **AWS Bedrock** (managed Anthropic, Meta, Cohere, Mistral, with Guardrails for safety, Knowledge Bases for retrieval), **Azure AI Foundry / Azure OpenAI**, **Google Vertex AI**. The gateway is one feature of a larger platform; you adopt it as part of adopting the inference platform.

The five tools the module's learning objective calls out — **LiteLLM, Portkey, Envoy AI Gateway, Kong AI Gateway, Cloudflare AI Gateway** — sit in the first three categories; the next section compares them against the questions a build-vs-buy decision asks.

<!-- needs-research: confirm the current feature set, deployment shape, and licensing of each tool listed above at reading time. Each has shipped major changes across 2024-2026 and the comparison table will drift. -->

## The comparison frame

Pick for yourself, with these dimensions. Ignore marketing categories; use these columns.

### Routing semantics

- **Internal model identifier vs. provider-name routing.** Does the tool model a platform-coded identifier (chapter 02's option C) or does it route on the provider's model name directly?
- **Fallback: ordered vs. weighted.** Does it support an ordered fallback walk under declared conditions, or only a load-balancing weighted pool?
- **Capability declarations.** Does it match the tool's capability declarations against the request's capability requirements before routing (chapter 02), or does it fail at the provider's response?
- **Multi-key per provider.** Can it manage multiple API keys per provider and route to spread load / respect rate limits?

### Observability shape

- **OpenTelemetry semantic conventions for generative AI.** Does the tool emit spans conforming to the current conventions, or does it emit proprietary trace fields?
- **Decision-log emission.** Does every routing decision emit a structured log the gateway's audit stream can consume, or is the trace all you get?
- **Billing-event schema.** Does the tool emit an event with the split-token fields from chapter 04 (`input_cached`, `input_non_cached`, `output`), or does it collapse to a single `total_tokens`? Does it include per-attempt cost, or only request-level?
- **Cache-visibility.** Does it emit the cache outcome and the namespace on every event (chapter 05)?

### Deployment model

- **Self-hosted process vs. SaaS.** Does the tool run in your VPC or does it require traffic to leave to a vendor? For enterprise tenants with data-residency constraints, SaaS is often a hard no.
- **Stateful dependencies.** Does the tool require a specific external store (Redis, Postgres, DynamoDB, vendor-managed)? What are the failure modes when it is unreachable?
- **Horizontal scaling and HA.** Is the deployment an active-active stateless shape, or does it have leader-election / coordinator semantics that limit scale?
- **Config-promotion shape.** Routing changes through environment variables, through a config service, through a vendor UI, or through a Kubernetes CRD?

### Multi-tenancy posture

- **Per-tenant isolation primitives.** Does the tool model tenants natively, or is the application responsible for passing a tenant header and hoping?
- **Cross-tenant cache guarantees.** Is cache state tenant-scoped by construction, or does the application have to compose keys?
- **Per-tenant quota support.** Monthly token quota, per-model, out of the box?
- **Audit of per-tenant obligations.** Do redaction and schema events carry the tenant identifier in a form compliance can query?

### Policy integration

- **PII / secret detection.** Built-in detectors, or BYO via plugin? Does the detection emit matches as structured events?
- **Schema enforcement.** Does the tool validate structured output against a schema registry?
- **Content-safety hooks.** Does the tool expose pre- and post-generation hooks that can call out to a policy plane (mod-405)?

### Licensing and legal

- **Open source license, if OSS.** MIT, Apache 2.0, BUSL, SSPL, proprietary — each has different downstream implications. BUSL and SSPL in particular restrict operating the software as a service.
- **SaaS SLA and data-handling.** If the vendor's SaaS sees traffic, what are the data-handling commitments? Is prompt logging on by default?
- **BYOK support.** Can enterprise tenants provide their own provider API keys and have the gateway honour per-tenant key scoping?

### Total cost of ownership

- **Per-request SaaS pricing vs. self-hosted compute.** At your scale, which is cheaper?
- **Operational burden.** How many engineering-hours per quarter does the tool consume in upgrades, incidents, and config changes?
- **Vendor-cutover cost.** Can you leave this tool in 3 months, 6 months, 12 months? What breaks?

## The five tools, read through the frame

What follows is positioning sketches, not endorsements. The purpose is to give you handles for your own evaluation; the specifics will drift.

### LiteLLM

- **Positioning.** Open-source Python library and proxy service. Started as an SDK for calling any provider through an OpenAI-compatible API; grew into a proxy with routing, caching, quotas, and observability.
- **Strengths.** Broad provider coverage (100+ providers claimed); trivial drop-in for Python agents; MIT/Apache-licensed; active community; OpenAI-compatible wire shape.
- **Weaknesses.** Rapid feature evolution means the production surface is still settling; running at high-RPS scale historically required careful tuning; multi-tenancy and quota surfaces are still maturing. Heavy reliance on contributors for provider-specific edge cases.
- **Fit.** Tier 2 for provider adapters and OpenAI-compatible wire shape in a platform that owns tier 1 (quotas, cache namespace, billing events) around it. Risky as a tier-1 vehicle for a large multi-tenant platform.
- **Reference.** <https://docs.litellm.ai/>

### Portkey

- **Positioning.** AI-gateway company offering both an OSS self-hosted gateway and a managed SaaS control plane. Marketing focuses on "AI gateway + observability + prompt management".
- **Strengths.** Opinionated feature set aimed specifically at agent / LLM workloads; configuration-driven routing; built-in semantic caching (response-level, distinct from prompt caching); pre-baked observability; multi-tenancy primitives.
- **Weaknesses.** Managed SaaS data path raises residency and audit questions at enterprise scale; the OSS gateway's feature parity with the SaaS lags; feature-lock-in risk for the proprietary bits.
- **Fit.** Candidate tier-2 for a mid-size platform. Requires a tier-1 wrap for the obligations in chapter 01 that the tool does not own.
- **Reference.** <https://portkey.ai/docs/>

### Envoy AI Gateway

- **Positioning.** A CNCF-ecosystem project (Envoy Gateway + AI extensions) that treats AI routing as a filter in the Envoy data path. Deployed on Kubernetes via Gateway API resources (`HTTPRoute`, `AIGatewayRoute`, `AIServiceBackend`) and standard Envoy filters.
- **Strengths.** Battle-tested data-plane (Envoy); Kubernetes-native; integrates with the rest of the service-mesh stack (mod-409); token-based rate limiting via `ext_proc`; vendor-neutral by construction. Operates in your cluster; no outbound SaaS dependency.
- **Weaknesses.** The AI-specific feature set is young relative to the LLM-gateway tools; semantic / prompt-cache features are less evolved; deeper Kubernetes/Envoy operations knowledge required to operate.
- **Fit.** Strong candidate when the platform already runs Envoy / Istio and the mod-409 service-mesh shape is in place. Natural data-plane for tier 2.
- **Reference.** <https://aigateway.envoyproxy.io/>

### Kong AI Gateway

- **Positioning.** A plugin suite on Kong Gateway that adds AI-specific features (AI Proxy, Request/Response Transformer, Prompt Guard, Prompt Decorator, Semantic Routing, AI Semantic Cache). Deploys as part of a Kong control plane.
- **Strengths.** Rich prompt-shaping plugins out of the box; mature Kong operations and tooling; broad enterprise adoption of Kong itself; integrates with Kong's existing policy, auth, and rate-limiting plugins.
- **Weaknesses.** The AI-specific plugins are newer than the core Kong product; licensing and feature-tier structure (Konnect vs. OSS) can be complex; adopting the AI gateway typically means adopting Kong as the primary gateway.
- **Fit.** Candidate when the platform already runs Kong. The prompt-shaping plugins compose well with chapter 06's gateway-side enforcement; verify the data path matches your residency needs.
- **Reference.** <https://docs.konghq.com/gateway/latest/ai-gateway/>

### Cloudflare AI Gateway

- **Positioning.** Cloudflare's managed edge gateway that proxies LLM API calls through Cloudflare's network. Positions as "observability, caching, and rate limiting for any AI API" with Cloudflare's billing and logging on top.
- **Strengths.** Zero-ops for the gateway itself; built-in logging, analytics, and response caching; cheap for low-RPS workloads; compatible with the major providers' APIs; attractive for teams already on Cloudflare.
- **Weaknesses.** SaaS data-path (prompts transit Cloudflare); limited per-tenant primitives aimed at platform-grade multi-tenancy; residency constraints map imperfectly to Cloudflare's edge footprint; audit and billing are Cloudflare's shape, not yours.
- **Fit.** Strong for single-tenant or small-tenancy workloads with no strict residency constraint. Rarely tier 2 for a multi-tenant agent platform at mid-to-large scale.
- **Reference.** <https://developers.cloudflare.com/ai-gateway/>

## Build vs. buy — the decision

Walk the dimensions in order; every tier-1 answer that is "we must own this" pushes the tool out of tier 2 for that slice.

1. **Does the tool provide a credible internal-identifier-based routing shape?** No tool survives routing by provider name alone at mid-platform scale.
2. **Does its billing-event schema carry the split-token fields chapter 04 requires?** If not, you will emit your own events in parallel — now you have two sources of truth. Prefer tools whose event format can be extended.
3. **Does its cache mechanism allow your namespace composition from chapter 05?** If the tool's cache is keyed by content and tenant-blind, cross-tenant leak is one code path away. Either turn the tool's cache off and layer your own, or pick a different tool.
4. **Does it run in your VPC and under your observability stack?** A SaaS gateway that logs to its own observability is a gateway you cannot trace into mod-408.
5. **Will you want to migrate off it within 12 months?** Vendor lock-in on the data path is heavier than lock-in on config. Prefer tools whose exit is a config regeneration, not a data re-platform.

A credible shape for most mid-platform decisions:

- **Tier 2 — one of LiteLLM or Envoy AI Gateway**, wrapped by platform-owned middleware.
- **Tier 1 — platform-owned** admission, quota, cache namespace, billing-event emission, redaction. Lives either inside the chosen tool's extension points or as a thin platform gateway in front of the tool.
- **Tier 3 — provider SaaS and a dedicated self-hosted substrate**, with the substrate owned by `ai-infra-inference-engineer`.

Document the decision in an ADR. Revisit on an annual cadence. Treat the gateway as a product your own peers can replace in-place when the trade-offs change.

## The handoff to `ai-infra-inference-engineer`

Self-hosted inference is a distinct engineering specialty, owned by `ai-infra-inference-engineer` (level 35) in the curriculum. The gateway routes to self-hosted inference as another provider; the engineer who owns the inference substrate owns everything inside the dashed line below.

```
                 agent runtime (mod-402)
                           │
                           ▼
            ┌────────────────────────────┐
            │   model gateway (mod-406)  │
            └────────────────────────────┘
                   ├──────┬───────┬───────┐
                   ▼      ▼       ▼       ▼
                OpenAI  Anthropic  Google   self-hosted
                                           ┌────────────────────────────┐
                                           │  inference substrate       │
                                           │  (ai-infra-inference-eng)  │
                                           │  - vLLM / SGLang / TRT-LLM │
                                           │  - continuous batching     │
                                           │  - paged KV cache          │
                                           │  - speculative decoding    │
                                           │  - quantisation / LoRA     │
                                           └────────────────────────────┘
```

### What the gateway owes the inference engineer

A clean, versioned contract so that the engineer can optimise the substrate without the gateway caring about the internals.

- **OpenAI-compatible wire shape.** The substrate exposes an OpenAI-compatible HTTP endpoint (vLLM, SGLang, and TensorRT-LLM Inference Server all expose this). The gateway speaks that shape. The engineer is free to swap frameworks underneath.
- **Request hints, not demands.** Guided-decoding grammars, structured-output schemas, speculative-decoding hints — all pass through the request in a documented shape. The substrate decides how to honour them.
- **Backpressure signals, not polling.** The substrate returns `429` with a `Retry-After` when overloaded; the gateway honours. The gateway does not poll a `/health` endpoint on the hot path.
- **Workload-identity auth.** The gateway authenticates to the substrate via SPIFFE (mod-405 chapter 04), not shared secrets.
- **Request tagging.** The gateway tags each request with `tenant_id`, `model_identifier`, and a `request_class` so the substrate can apply its own scheduling and reporting.
- **Cache-affinity hints.** When the gateway runs scheme-3 affinity (chapter 05), it tags the request with the prefix hash so the substrate can route internally to the right KV-cache replica.

### What the inference engineer owes the gateway

A substrate that behaves like a provider from the gateway's perspective.

- **Capability declaration.** For every self-hosted model, the substrate publishes the same capability fields the routing table carries (chapter 02).
- **Stable token-count reporting.** Input and output tokens reported on every response, split cached / non-cached where the substrate's own cache is involved.
- **Deterministic error classes.** Overloaded → 429 or 503; malformed → 400; context-length → 400 with a parseable reason; auth → 401. The gateway's normalisation (chapter 03) depends on this.
- **Published changelog.** Model updates, quantisation changes, LoRA swaps — all announced with lead time so the gateway can route traffic accordingly and so mod-407's eval suite can run.
- **Metrics and traces following gen-ai semantic conventions.** The substrate publishes its own OpenTelemetry spans so the gateway's trace is joinable.

### The interface points that fail first

Three places this contract typically breaks in practice; call them out early.

- **Token counting.** Self-hosted substrates that report `total_tokens` without splitting prompt vs. completion, or that report counts from a different tokenizer than the gateway expects, silently corrupt cost attribution. Fix by making the substrate authoritative and the gateway consume its numbers directly.
- **Streaming parity.** OpenAI-compatible SSE from a self-hosted substrate can emit subtly-different event shapes (missing `[DONE]` sentinel, differing chunk boundaries). The gateway's adapter unit tests cover the quirks; the engineer is the one who fixes them upstream.
- **Grammar / schema enforcement variance.** vLLM, SGLang, and TRT-LLM have different support surfaces for guided decoding. A schema that validates on one engine may not compile on another. The capability matrix declares the subset the substrate supports; the gateway refuses mappings that can't honour the schema.

## Operating cadence

The gateway is a tier-0 dependency once it is up. An operating cadence that catches the two most common failure patterns:

- **Weekly.** Review `gateway_fallback_rate_by_pair` and `cache_hit_rate{namespace}`. Pull the two metrics into one review and read them together; a rising fallback rate with a falling cache hit rate is a routing-table drift.
- **Monthly.** Price-table review (chapter 04). Reconcile provider invoices against the gateway's computed cost. Schedule a config-promotion rehearsal on a staging tenant.
- **Quarterly.** Re-run the build-vs-buy frame. Vendor features move faster than annual review cycles accommodate.
- **On every provider announcement.** Capability matrix review within 48 hours. New feature → routing-table extension; deprecated model → fallback-map rebuild; price change → price-table version bump; safety-policy change → redaction-obligation review.

## Summary

- Treat the gateway as a portfolio of tier-1 (own), tier-2 (buy and compose), and tier-3 (delegate) capabilities. Tenant-scoped cache namespace, per-tenant quota, billing-event schema, and PII/secret egress are always tier 1.
- Compare tools across routing semantics, observability shape, deployment model, multi-tenancy posture, policy integration, licensing, and TCO. LiteLLM, Portkey, Envoy AI Gateway, Kong AI Gateway, and Cloudflare AI Gateway each occupy a different point.
- A credible mid-platform shape is tier-2 LLM gateway (LiteLLM or Envoy AI Gateway) wrapped by platform-owned tier-1 middleware, with providers behind and self-hosted inference alongside.
- The handoff to `ai-infra-inference-engineer` is an OpenAI-compatible wire shape, published capability declarations, deterministic error classes, and stable token reporting. The gateway treats self-hosted inference as another provider; the inference engineer owns the substrate.
- Operate the gateway on a weekly / monthly / quarterly cadence against the SLIs chapters 03–06 define. Vendor landscape moves fast; re-run the frame yearly.

This chapter closes the module. The exercises take the shape of mod-406 content and turn it into artefacts you own — a routing + fallback policy with tool-loop invariants (exercise 01), a per-tenant quota and burst-budget enforcer (exercise 02), a tenant-scoped prefix-cache namespace (exercise 03), and a cost-attribution billing-event pipeline (exercise 04).
