# The Model-Gateway Slice and Its Obligations

The runtime (mod-402) owns the agent loop. The MCP gateway (mod-403) owns the tools. The sandbox (mod-404) owns tool-code execution. The policy plane (mod-405) owns authorisation. That still leaves one hot-path hop on every agent step: the call out to a model. Where exactly does that call go? Which provider, which model name, which region, which key, under which tenant's quota, cached against which prefix, carrying which obligation to redact or validate? That is the slice this module owns.

The naive answer is "the agent SDK talks to `api.openai.com` directly and we are done". A single-provider, single-tenant, no-quota, no-cache, no-cost-attribution gateway is in fact the default shape of most prototypes. It fails the moment you have a second tenant, a second provider, a second model family, an egress audit, a price regression, or an incident that requires failing over off one provider. Every platform with more than one agent on it builds a model gateway. The only question is how much of it you own.

This chapter fixes the slice. The chapters that follow pick the pieces: multi-provider routing (02), fallback and retry tuned to tool-calling loops (03), per-tenant quotas and cost attribution (04), prompt-cache coordination (05), structured-output and PII enforcement (06), and the build-vs-buy call across LiteLLM, Portkey, Envoy AI Gateway, Kong AI Gateway, and Cloudflare AI Gateway (07).

## What the model-gateway plane owns

The plane sits between the agent runtime (mod-402) that drives the loop, the MCP gateway (mod-403) that executes tool calls, the policy plane (mod-405) that authorises those tool calls, the observability plane (mod-408) that collects traces and metrics, and — on the far side — the model providers (hosted APIs and self-hosted inference). It owns:

- **Provider abstraction.** A single internal request shape the runtime speaks, normalised so that a swap from one provider to another is a routing-config change rather than an application change. Chapter 02.
- **Routing.** Per-request selection of which provider, model name, region, and key to use — driven by the request's declared model family, the tenant's provider allow-list, the health of each backend, the current prompt-cache state, and the applicable cost policy. Chapters 02–05.
- **Fallback, retry, and hedging.** When a provider returns a 5xx, a timeout, or a safety-system refusal mid-tool-loop, the gateway decides whether to retry, fail over, switch model, or surface the failure. The *mid-tool-loop* part is what makes this different from a general LLM proxy. Chapter 03.
- **Per-tenant quotas and burst budgets.** Token budgets per tenant per window, with a burst allowance for latency-sensitive paths, enforced before the request hits the provider so the gateway owns the refusal. Chapter 04.
- **Token and cost attribution.** Every request emits a billing event carrying tenant, user, session, project, provider, model, cached-input tokens, non-cached-input tokens, output tokens, and the computed cost. The event is the input to showback, chargeback, FinOps dashboards, and the price-regression alarms. Chapter 04.
- **Prompt-cache coordination.** Prefix hashing, cache partitioning, and cache-affinity routing so that savings actually accrue (same-tenant same-prefix requests land on the same backend) without leaking context (another tenant's cache key never shares a cache namespace with yours). Chapter 05.
- **Structured-output enforcement.** Agents that need JSON (or a tool-call schema) get schema enforcement at the gateway — against the provider's native structured-output feature when one exists, and against a gateway-local validator otherwise. Chapter 06.
- **PII and secret redaction at the egress seam.** Prompts the agent is about to send out to a third-party provider get a last-chance scan; the gateway refuses to egress a prompt that contains a tenant-prohibited class of data. Chapter 06.
- **Observability shape.** Every call emits spans conforming to OpenTelemetry semantic conventions for Generative AI, with the token counts, cache fields, provider metadata, and tenant / session identifiers that mod-408 depends on.

The plane does **not** own:

- **The inference-serving substrate itself.** vLLM, SGLang, TensorRT-LLM, continuous-batching schedulers, KV-cache paging, speculative decoding — that is `ai-infra-inference-engineer` (level 35). The gateway treats self-hosted inference as another provider with a well-known interface. Chapter 07 covers the handoff.
- **The agent loop.** The gateway returns a single model response per call; it does not decide what to do next. That is the runtime (mod-402).
- **Tool dispatch.** The gateway may surface a `tool_use` block from the provider, but it does not call the tool. That is the MCP gateway (mod-403) and the runtime (mod-402).
- **Authorisation of tool calls.** The policy plane (mod-405) decides whether a tool call that the model emits is allowed to run. The model gateway's job ends at returning the model's response.
- **The content-safety decision.** The safety plane (mod-411) owns refusal / redaction decisions on the generative side (toxicity, self-harm, jailbreak-produced content). The model gateway's PII-at-egress role (chapter 06) is specifically pre-egress on prompts; downstream content-safety hooks are the safety plane.
- **The harm model and red-team findings.** Those flow in from `ai-risk-engineer` (level 25) and `agentic-safety-engineer` (level 40) via the policy plane; the model gateway consumes them as routing and redaction obligations, not as policies it authors.

Drift off this slice and you recreate the safety plane or the policy plane, usually incorrectly, at the egress seam. Scope policing is a feature of this module.

## Why a gateway and not direct SDK calls

A few concrete questions that are hard to answer without a gateway and trivial with one. If your current proxy can answer each of them with a one-line config change, you have a gateway. If any one answer involves patching application code, you do not yet.

- **Provider failover.** Anthropic is returning `overloaded_error` on `claude-opus-4-7` for the next 45 minutes. Can your platform roll traffic over to `claude-sonnet-4-6` — or to Bedrock-hosted Claude, or to an OpenAI equivalent for the subset of requests where that is acceptable — without redeploying any agent?
- **Per-tenant model allow-list.** Tenant `acme-financial` is under a data-residency contract that forbids egress to `us-east-1` Anthropic. Can you refuse — at the gateway, not in each agent — any request that would route traffic there, while still letting the same agent code serve another tenant where it is permitted?
- **Runaway tool loop.** An agent has entered a tool-call loop and is burning 40k input tokens per step at Opus rates. The tenant's monthly budget is 70% consumed on day 11. Can you stop it at the gateway without the application noticing anything but a quota-exceeded error?
- **Cache attribution.** Yesterday one tenant moved 60% of their system-prompt tokens into the cached-input bucket. Can you prove it from a billing event rather than from the provider's monthly PDF?
- **Audit replay.** Compliance asks "on 2026-11-03T14:22Z we see an agent decision that reads like it came from an unapproved model — which provider and model name actually served the response?" Can you answer from a log, or do you have to grep agent code?

These are the capabilities the plane delivers, and they are what separate "we have a config variable called `MODEL_URL`" from "we have a model gateway". Chapters 02–06 build each.

## The two deployment shapes

Two deployment shapes dominate; pick deliberately, do not drift.

### In-process library (SDK)

The gateway is a library — LiteLLM Python, Portkey SDK, a thin in-house wrapper — imported into every agent process. Routing config lives in environment variables or a config service. Decisions are local; no extra network hop.

- **Strengths.** Zero added latency (microseconds vs. a network hop). No separate service to operate. Easy to adopt incrementally.
- **Weaknesses.** Per-tenant quotas are either local-only (a single pod cannot enforce "no more than 10M tokens / day across the fleet") or require a shared coordination service — at which point you have reinvented half the standalone shape. Policy and redaction obligations ship as part of the library version; rolling a redaction fix requires rolling every agent.
- **Where it fits.** Early-stage platforms, single-agent prototypes, batch jobs that run as a single process. Mod-401 chapter 04 lists this shape as the "don't platformise yet" point.

### Standalone service (proxy)

The gateway is a standalone HTTP service — one or more replicas behind a load balancer — that the agent runtime calls over the wire. Routing config, quota state, cache index, redaction rules, and provider credentials all live centrally. Agents hold a single internal endpoint and a signed identity to call it.

- **Strengths.** Quotas, budgets, and cache state are coherent across the fleet. Policy and redaction updates roll independently of agent deploys. Provider credentials never leave the gateway. Observability shape is the same across every agent.
- **Weaknesses.** Added network hop (typically single-digit ms in-datacentre, meaningful for interactive paths). Operational load — the gateway is now a tier-0 dependency. Standalone gateways are themselves a piece of software that can have bugs.
- **Where it fits.** Multi-agent, multi-tenant platforms past the prototype stage. This is the shape mod-406 writes against.

A **hybrid** shape — a thin in-process SDK that calls a standalone gateway — is where most mature platforms end up, and is the shape chapter 07 defends. The SDK handles ergonomics and speaks the OpenAI-compatible wire protocol so adoption is cheap; the standalone gateway handles the coordinated state. Each role is clear; neither does the other's work.

## The threat model this plane defends against

Four classes of threat dominate what the gateway sees. They are the ones chapters 03–06 build mitigations for.

### Provider lock-in and silent capability regression

A single-provider gateway is a political and technical lock-in. Every agent in your platform is now a hostage to one vendor's uptime, pricing, deprecation cadence, and policy decisions. The attack surface is not malicious; it is organisational. The platform loses leverage.

Related: a provider silently re-routes `gpt-4o` to a differently-trained successor. A model your evaluation suite (mod-407) validated last quarter now behaves differently on the margins. You discover it from a regression in production, not from a provider announcement.

The plane's defences:

- **Multi-provider routing with pinned model-name mappings.** The gateway's config says `platform:writer-v1 -> openai:gpt-4o-2024-08-06 AND anthropic:claude-sonnet-4-6`, not `writer-v1 -> openai:gpt-4o`. A provider's silent rename does not change the mapping. Chapter 02.
- **Fail-over on refusal, not just on error.** A provider's safety-system refusal is a routing signal, not a terminal state. Chapter 03.
- **Shadow routing for validation.** A small percentage of traffic is sent to the "next provider" for comparison. Chapter 02 builds the primitive; mod-407 owns the evaluation.

### Budget-exhausting loops

An agent enters a tool-calling loop — asks a tool, gets an answer, asks again, loops. Even a modest 20-step loop at Opus rates on a 30 k-token context burns meaningful money per session. A malicious prompt-injection can engineer this deliberately; an under-tested agent can engineer this accidentally. Without a gateway-side budget enforcer, the bill is the first signal.

The plane's defences:

- **Per-session token budget with hard cut.** The gateway counts tokens attributable to a session and refuses further requests past a ceiling. Chapter 04.
- **Per-tenant burst budget separate from monthly.** The burst limits short-window damage; the monthly limits long-tail drift. Chapter 04.
- **Loop-detection heuristics.** Tool-result-identity windows and consecutive-identical-tool-call detection surface the shape of a loop at the gateway, independent of the runtime's own loop guard. Chapter 03.

### Cross-tenant cache leakage

Prompt caching is a real cost win — provider caching discounts are 50–90% off non-cached input tokens on the cached portion, depending on provider and tier (see chapter 05 for current pricing citations). It is also the single most dangerous optimisation in a multi-tenant gateway. If tenant A's system prompt (containing tenant A's private configuration or data) is cached and tenant B lands on the same cache entry, you have leaked across the tenancy boundary.

The plane's defences:

- **Cache-key partitioning by tenant.** No cache entry is shared across tenants, by construction. Chapter 05.
- **Cache-affinity routing within a tenant.** Same-tenant same-prefix requests land on the same backend (or same cache namespace) so savings actually accrue. Chapter 05.
- **Prefix hashing with tenant-scoped salts.** The cache key is a function of the tenant identifier, not of the prefix content alone. Chapter 05.

### PII and secret exfiltration at the egress seam

An agent, through prompt-injection or coding error, is about to send a prompt out to a hosted provider that contains data the tenant has forbidden from egressing — social-security numbers, API keys, internal URLs, a customer's health information. The provider is not malicious; the data simply has no business leaving the platform. By the time it hits the provider logs, it has already egressed.

The plane's defences:

- **Pre-egress PII scan.** The gateway inspects the serialised prompt against tenant-configured detectors and either redacts (on obligations the policy plane emits) or refuses (on deny-class classes). Chapter 06.
- **Secret-pattern detectors.** API keys, access tokens, PEM blocks match deterministic patterns the gateway can refuse outright. Chapter 06.
- **Provider-specific egress allow-lists.** Some data classes may egress to a self-hosted model but not to a third-party provider; the gateway's routing decision carries the class constraint. Chapter 02.

A fifth class — **structured-output malformation** — is a near relative of the first four and is addressed in chapter 06: a provider's native JSON mode is best-effort on most providers, and a tool-call that cannot be parsed is an agent-side error the gateway can catch before the agent sees it.

## The request envelope

Pin the shape before writing the plane. Every request to the gateway returns to the caller inside a single envelope. The provider-specific response bodies live *inside* the envelope; the envelope itself is invariant across OpenAI, Anthropic, Google, and self-hosted backends.

```json
{
  "gateway_request_id": "gw_01HV8X3M9F5A6R2B4C0D7E8G9H",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "received_at": "2026-11-06T18:22:04.173Z",
  "routing": {
    "requested_model": "platform:writer-v1",
    "resolved_provider": "anthropic",
    "resolved_model": "claude-sonnet-4-6",
    "resolved_region": "us-east-1",
    "attempts": [
      { "provider": "anthropic", "model": "claude-sonnet-4-6",
        "attempt_no": 1, "outcome": "overloaded_error",
        "latency_ms": 820 },
      { "provider": "anthropic", "model": "claude-sonnet-4-6",
        "attempt_no": 2, "outcome": "success",
        "latency_ms": 1240 }
    ],
    "fallback_reason_code": "provider.overloaded"
  },
  "tenancy": {
    "tenant_id": "acme",
    "session_id": "s_01HV8X3M9F5A6R2B4C0D7E8G9H",
    "principal_subject": "alice@acme.com",
    "actor_spiffe_id": "spiffe://platform.local/runtime/prod"
  },
  "tokens": {
    "input_cached": 4712,
    "input_non_cached": 318,
    "output": 442,
    "tool_use_input": 97,
    "tool_use_output": 120
  },
  "cost": {
    "input_cached_usd": 0.0007,
    "input_non_cached_usd": 0.0024,
    "output_usd": 0.0133,
    "total_usd": 0.0164,
    "price_table_version": "2026-11-01"
  },
  "cache": {
    "prefix_hash": "sha256:9ce4...",
    "cache_namespace": "acme:writer-v1",
    "cache_outcome": "hit_partial",
    "cached_prefix_tokens": 4712
  },
  "obligations": {
    "applied": [
      { "kind": "redact.pii_high",
        "fields": ["phone", "email"],
        "source": "policy_plane_decision:01HV8X..." },
      { "kind": "schema.validate",
        "schema_ref": "tenant:acme/schemas/writer-v1@3",
        "outcome": "valid" }
    ],
    "refused": []
  },
  "response": {
    "provider_response_id": "msg_01A2b3C4d5E6",
    "finish_reason": "end_turn",
    "content": [ /* provider-native content blocks */ ]
  }
}
```

Five invariants the envelope enforces:

1. **A deterministic gateway request ID, independent of the provider's.** The gateway owns correlation across retries and fallbacks; mod-408 uses `gateway_request_id` to join per-attempt spans into one logical request.
2. **The routing attempt list is complete.** Every attempt — primary, retry, fallback — lands here with its outcome and latency. A fallback that returns success is still a reportable event; the cost side of the house wants to see failovers because they drive tail latency and blended-cost reporting.
3. **Token counts split cached vs. non-cached.** Blended counts hide the whole point of caching. Chapter 04 shows why the split is the essential unit of billing-event design.
4. **Cost is computed at the gateway against a versioned price table.** Not fetched later from the provider invoice. The `price_table_version` makes the cost replayable and the price-regression alarm (chapter 04) possible.
5. **Obligations the plane applied are listed.** A redaction that fired, a schema that validated, a safety hook that ran — every obligation leaves a trace. A missing obligation list is a bug, not an optimisation.

The chapters that follow build the pieces that produce this envelope.

## Where the plane physically sits

A typical shape for the physical deployment:

- **Gateway service:** Horizontally scaled behind a load balancer. Stateless for the hot path; all coordination state (quotas, cache index, rate limiters) lives in a shared store (typically Redis Cluster or an equivalent) or is partitioned by consistent-hash key. SPIFFE-authenticated inbound from the runtime (mod-405 chapter 04); SPIFFE or provider-API-key outbound.
- **Config service:** Routing rules, model-name mappings, tenant allow-lists, price table — all versioned, signed, promoted through envs. The gateway pulls the active bundle on start and subscribes to updates. No hand-edits in prod.
- **Billing-event bus:** Every request emits a billing event to an append-only stream (Kafka, Kinesis, Pub/Sub). The stream is the source of truth for chargeback and for price-regression alarms. Chapter 04.
- **Cache coordination state:** Per-tenant prefix-hash index pointing at backend affinities. For a provider-managed cache (Anthropic's prompt caching, OpenAI's prompt caching), the gateway's role is primarily *routing* so that the same tenant's same-prefix requests land on a backend that will see the cache. For a gateway-managed cache (self-hosted inference where the gateway owns the KV paging hint), the gateway owns the entry. Chapter 05.
- **Secret stores:** Provider API keys live in a secret manager (Vault or cloud-native), fetched once on startup, rotated by the broker (mod-405 chapter 04). The gateway never writes keys to logs.
- **Observability sinks:** OpenTelemetry traces to a span collector; metrics (request rate, p50/p95/p99 latency per provider / per model, cache hit rate, quota-refusal rate) to a metrics backend; decision logs to the audit stream.

If you cannot draw this topology on a whiteboard and name the box each decision flows through, chapter 02 is the thing to read next.

## The three obligations

Pin the plane to three non-negotiable obligations. Everything in chapters 02–07 is a decision about how to deliver them.

1. **"No model call runs outside the gateway, and every call produces a replayable billing event with cost, provider, model, and token-split fields."** No direct provider calls from agents. No ambient model credentials inside the agent process. Replayability is the compliance and FinOps contract.
2. **"Routing, retry, quota, cache, and redaction obligations are enforced at the gateway, not in agent code — and config changes to any of them roll independently of agent deploys."** If a safety-driven redaction fix requires an application redeploy, the gateway has lost its reason to exist. Config promotion is the primary release surface.
3. **"The cache never crosses the tenant boundary, and the cost of a call is correct to the cached-token-split line."** Cross-tenant cache leak is the single specific breach a model gateway can cause. The split-token cost field is the single most common silent FinOps regression. Both must be first-class invariants, not optimisations.

A plane that cannot defend all three is a model gateway in name only.

## Summary

- The model-gateway plane owns provider abstraction, routing, fallback and retry, quotas and burst budgets, token and cost attribution, prompt-cache coordination, structured-output enforcement, and pre-egress PII / secret redaction. It does not own inference-serving, tool dispatch, authorisation, content safety on model output, or the harm model.
- Two deployment shapes exist — in-process SDK and standalone service; a hybrid of the two is where most mature platforms end up. The plane in this module is written against the standalone shape.
- The threats the plane defends are provider lock-in and capability regression, budget-exhausting loops, cross-tenant cache leakage, and pre-egress PII / secret exfiltration. Chapters 03–06 build the mitigations.
- Every call returns the same request envelope: routing attempts, tenancy, split-token counts, versioned cost, cache outcome, applied obligations, and the provider-native response inside. Every chapter adds a piece to this envelope.
- Three non-negotiable obligations: every call through the gateway with a replayable billing event; routing / retry / quota / cache / redaction as gateway config, not application code; and tenant-scoped cache with correct split-token cost on every event.

The next chapter opens the first piece — the provider abstraction and multi-provider routing table that the envelope's `routing` block is a report on.
