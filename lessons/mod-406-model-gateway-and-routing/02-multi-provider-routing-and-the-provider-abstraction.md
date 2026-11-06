# Multi-Provider Routing and the Provider Abstraction

Chapter 01 pinned the slice. This chapter builds the first piece of what lives inside it: the provider-abstraction layer, the routing table that resolves a platform-internal model name to a specific provider endpoint, and the capability matrix that keeps the gateway from issuing a request the chosen backend cannot serve. Fallback, retry, and quota are chapters 03 and 04; here the job is the forward path.

The model gateway's reason to exist, at the simplest layer, is this: an agent says "give me a response from the writer model" and the gateway decides, per call, which provider and which model actually serves. The agent has no business knowing whether the call went to Anthropic's hosted API, to AWS Bedrock-hosted Anthropic, to a self-hosted vLLM instance, or to OpenAI as a fallback. The gateway owns the decision.

## The internal model name

The first design choice: what identifier does the agent use? Three options; one is correct for a platform.

### Option A — The provider's model name

The agent passes `claude-opus-4-7` or `gpt-4o-2024-08-06` directly. The gateway is a thin proxy that routes by the vendor-coded name. This is what `api.openai.com`-compatible proxies look like out of the box.

- **Simple.** Zero translation between the agent SDK and the gateway.
- **Inflexible.** The agent is now tied to that provider's naming. Swapping providers, A/B-testing against a different model, or routing a tenant to a different model family all require application-code changes.
- **Audit-hostile.** Compliance asks "which model answered on 2025-07-14 at 14:22Z"; the answer is whatever the provider mapped that name to at that time, which the gateway did not pin.

### Option B — A generic family name

The agent passes `"fast"`, `"balanced"`, `"strong"`. The gateway picks one specific provider+model per call.

- **Flexible.** The gateway is free to swap the backing model at any time.
- **Opaque.** Agents cannot reason about capability (context length, structured-output support, vision support). The gateway has to encode every capability constraint on the agent's behalf, which it does not always know.
- **Evaluation-hostile.** Mod-407 wants to pin a model name to a given eval score; "balanced" does not pin anything.

### Option C — A platform-coded model identifier

The agent passes an identifier the *platform* owns: `platform:writer-v1`, `platform:coder-v2`, `platform:critic-v1`. The gateway's routing table maps that identifier to one primary and one or more fallback provider-plus-model pairs. The identifier is versioned.

- **The agent is decoupled from the provider.** A routing-table change moves traffic without an application deploy.
- **The capability is pinned by the identifier.** `platform:writer-v1` has a capability declaration — context length, structured-output, vision — that the routing table must satisfy on every mapping.
- **The identifier is auditable.** On any given day, `platform:writer-v1` resolved to a specific provider-plus-model, recorded in the routing config's version history.
- **Evaluation keys on the identifier *and* on the resolved model.** Mod-407 scores both.

The platform-coded identifier is the correct choice. Use it. The routing-table shape below is written against it.

## The routing table

The routing table is a signed, versioned bundle the gateway pulls from the config service on startup and subscribes to updates for. It is the single source of truth for "which provider served which internal model on which day". A representative shape:

```yaml
# routing-table @ version 2026-11-01.3
models:
  - id: platform:writer-v1
    description: General-purpose long-context writer
    capability_declaration:
      context_window_tokens: 200000
      max_output_tokens: 8192
      structured_output: true
      tool_use: true
      vision: false
      caching: provider_managed
    primary:
      provider: anthropic
      model: claude-sonnet-4-6
      region: us-east-1
      key_ref: secret://providers/anthropic/prod/key
    fallbacks:
      - provider: anthropic
        model: claude-sonnet-4-6
        region: eu-west-1
        key_ref: secret://providers/anthropic/prod/key-eu
      - provider: bedrock-anthropic
        model: anthropic.claude-sonnet-4-6-20260815-v1:0
        region: us-east-1
        key_ref: aws://roles/platform-gateway-bedrock
      - provider: openai
        model: gpt-4o-2024-08-06
        region: us-east-1
        key_ref: secret://providers/openai/prod/key
        capability_deviations:
          - structured_output_implementation: json_mode_best_effort
          - caching: unsupported_on_this_mapping
    retry_policy_ref: retry-policies/tool-loop-strict
    cache_policy_ref: cache-policies/writer-v1
    tenancy_allow_list_ref: tenancy/writer-v1
    cost_pricing_ref: pricing/2026-11-01

  - id: platform:critic-v1
    description: Short-output grader; low latency
    capability_declaration:
      context_window_tokens: 32000
      max_output_tokens: 1024
      structured_output: true
      tool_use: false
      vision: false
    primary:
      provider: self-hosted-vllm
      model: llama-3.3-70b-instruct
      endpoint: https://vllm-critic.internal/v1
      key_ref: spiffe://platform.local/gateway/vllm-critic
    fallbacks:
      - provider: openai
        model: gpt-4o-mini
        region: us-east-1
        key_ref: secret://providers/openai/prod/key
```

Four design points that the ecosystem has converged on and that are easy to miss on a first draft:

1. **Primary + ordered fallbacks.** Not a weighted pool. Fallback is a chapter-03 retry decision tree with a deterministic order; weighting across providers mid-fallback leads to non-reproducible billing-event sequences and tangled incident post-mortems.
2. **Explicit region.** A fallback that silently crosses a region boundary is a potential data-residency breach. The routing table carries the region on every entry; chapter 04's tenant allow-list can forbid regions per tenant.
3. **Capability declaration is per-model-identifier, not per-mapping.** The identifier promises `structured_output: true`; every mapping must satisfy that or declare a `capability_deviation`. Deviations are not silent: they are read by the gateway at routing time, and a deviation that intersects the request's requirements changes the mapping's eligibility.
4. **Key references, not key values.** Every `key_ref` is a pointer the gateway resolves via its secret client. Keys never live in the routing table. The key's rotation (mod-405 chapter 04) is invisible to the table.

## The capability matrix

Model providers do not all speak the same shape. A gateway that pretends they do fails on the day a provider returns something the agent did not expect. The capability matrix is the in-memory model of "what does this provider-plus-model actually support", maintained in the gateway's config and consulted on every routing decision.

A minimal row set (not exhaustive):

| Capability | OpenAI `gpt-4o-2024-08-06` | Anthropic `claude-sonnet-4-6` | Google `gemini-1.5-pro-002` | Self-hosted vLLM `llama-3.3-70b-instruct` |
|---|---|---|---|---|
| Max context tokens | 128 k | 200 k | ≥1 M (declared) | Model-config dependent |
| Max output tokens | 16 k | 8 k | 8 k | Model-config dependent |
| Native tool-use | Yes (`tools` / `tool_choice`) | Yes (`tools` / `tool_choice`) | Yes (function calling) | Via guided-decoding / OpenAI-compatible endpoint |
| Parallel tool calls | Yes | Yes | Yes | Depends on serving config |
| Streaming | Yes | Yes | Yes | Yes |
| Structured output | Native JSON mode + response_format schemas | Not native (gateway-side validator) | Native (schema-constrained decoding) | Guided decoding (grammar / regex / JSON schema) |
| Prompt caching | Automatic (routing-dependent) | Explicit cache_control blocks | Explicit (cachedContents) | Gateway-managed on top of KV cache |
| Vision | Yes | Yes | Yes | Varies by checkpoint |

Each cell is a question the gateway may need to answer on a given routing decision. Two implications:

- **A request that requires a capability the fallback does not support cannot route to that fallback.** The gateway refuses the fallback and either picks the next one or returns a routing failure. The alternative — "fall back to a model that cannot do structured output, then silently drop the structured-output enforcement" — is how silent regressions ship to production.
- **The matrix is config, not code.** When a provider announces a new feature, the capability matrix is updated as a config promotion; the gateway reloads; routing rules now include the new mapping. No application changes.

<!-- needs-research: confirm the max-context, max-output, and caching behaviour for each provider/model in the table at reading time; model-level limits have moved repeatedly through 2025-2026. -->

## Provider abstraction, at engineering depth

The gateway speaks a single internal request shape — call it the **gateway request** — to its handlers and emits a single internal response shape — the **gateway response**. For each provider, a thin adapter translates both directions. The internal shape is close enough to the OpenAI shape that many platforms adopt the OpenAI chat-completions schema as the internal shape; close enough to the Anthropic messages shape that the translation to Anthropic is cheap; and expressive enough to carry provider-specific extensions (cache_control blocks, system role vs. system message, tool-choice directives).

Three concrete translation points worth internalising.

### The system prompt

- **OpenAI** — a `system` message in the messages array (or a `developer` role in newer SDKs).
- **Anthropic** — a top-level `system` field, not an element of `messages`. Multiple system blocks are valid and are the natural cache-partition boundary (chapter 05).
- **Google (Gemini)** — a `systemInstruction` field at the request top level.
- **vLLM OpenAI-compatible** — the OpenAI shape.

The adapter translates in both directions. A naive adapter that copies the OpenAI shape to Anthropic's messages array as a `user` message with a `role: system` tag is exactly the shape that silently disables Anthropic's prompt caching. Read the adapter carefully.

### Tool calls

- **OpenAI** — tool definitions in a `tools` array; the model returns `tool_calls` entries with `function.name` and `function.arguments` (a JSON string). The runtime echoes tool results back as `role: tool` messages referencing the call ID.
- **Anthropic** — tools in a `tools` array; the model returns `tool_use` content blocks with `name` and `input` (structured). The runtime echoes results as `tool_result` content blocks referencing the `tool_use_id`.
- **Google** — `functionDeclarations` in `tools`; model returns `functionCall` parts; results go back as `functionResponse` parts.

The adapter normalises to one internal shape — most platforms adopt a shape close to Anthropic's structured `input` (cleaner than OpenAI's string-encoded `arguments`, less encoded than Google's parts) — and translates on the way out and back. Argument strings vs. structured input are the single most common silent bug in a new adapter; a schema check on the translated arguments catches it.

### Finish reasons

- **OpenAI** — `finish_reason`: `stop`, `length`, `content_filter`, `tool_calls`, `function_call`.
- **Anthropic** — `stop_reason`: `end_turn`, `max_tokens`, `stop_sequence`, `tool_use`.
- **Google** — `finishReason`: `STOP`, `MAX_TOKENS`, `SAFETY`, `RECITATION`, `OTHER`.

The internal shape normalises to a closed vocabulary — `end_turn`, `max_tokens`, `tool_use`, `stop_sequence`, `content_filter`, `other` — with the provider-native value preserved under `provider_native_finish_reason` for audit. Chapter 03's retry policy branches on the normalised value.

## Routing selection, on the hot path

A single call flows through the gateway roughly like this. The code is a sketch; a real gateway adds metrics, circuit breakers, obligation hooks, and typed error types.

```python
def route(request: GatewayRequest) -> GatewayResponse:
    # 0. Validate request against capability declaration.
    model_entry = routing_table.lookup(request.model_id)  # platform:writer-v1
    missing_caps = capability_check(request, model_entry.capability_declaration)
    if missing_caps:
        raise CapabilityNotSupported(request.model_id, missing_caps)

    # 1. Resolve candidate list: primary + fallbacks, filtered by tenancy.
    candidates = [model_entry.primary, *model_entry.fallbacks]
    candidates = [c for c in candidates
                  if tenancy_allowed(request.tenant_id, c)
                  and capability_matrix.supports(c, request)]
    if not candidates:
        raise NoEligibleBackend(request.model_id, request.tenant_id)

    # 2. Reorder by cache-affinity (chapter 05).
    candidates = reorder_for_cache_affinity(request, candidates)

    # 3. Reorder by health (chapter 03's circuit-breaker state).
    candidates = reorder_for_health(candidates)

    # 4. Walk candidates until one succeeds or all fail.
    attempts: list[Attempt] = []
    for candidate in candidates:
        attempt = execute(candidate, request)   # chapter 03 handles the retry policy
        attempts.append(attempt)
        if attempt.outcome == "success":
            return finalise(request, candidate, attempt, attempts)
        if not fallback_eligible(attempt.outcome, model_entry.retry_policy):
            raise ProviderFailure(request.model_id, attempts)

    raise AllCandidatesExhausted(request.model_id, attempts)
```

Three points from this sketch worth pulling out:

- **Capability check is the first gate.** A request that asks for structured output when the resolved mapping has `structured_output: false` is a routing failure, not a silent degradation.
- **Tenancy filter runs before cache reordering.** You never evaluate affinity for a backend the tenant is not allowed to use; that leak (chapter 05) is prevented by construction.
- **Cache affinity and health are two independent reordering passes, not one.** Mixing them conflates "stick to the backend that already has your prefix" with "avoid the backend that just failed"; treat them as composable sorts.

## Shadow routing and canary routing

A gateway owns three distinct routing modes. Name them explicitly in config; do not collapse them.

- **Primary routing** — the response the agent sees. This is the mode above.
- **Shadow routing** — a second provider serves the same request in parallel, and the response is captured and compared offline. The agent never sees the shadow response. Mod-407 uses shadow traffic for provider-swap evaluation; chapter 03 uses it for "is the fallback still viable" probing. Shadow routing is budgeted and sampled — 1% to 5% of eligible traffic is a common range.
- **Canary routing** — a defined percentage of production traffic is served by a different mapping for the primary, with a kill-switch to roll back. The agent sees the canary response. Canary deployments of a new model-name mapping run through this mode. The canary-vs-control comparison is a mod-407 job; the gateway's job is clean, auditable splitting.

The three modes share infrastructure but have different billing semantics. Shadow traffic consumes provider cost that must be attributed to a *shadow* budget line (chapter 04), not to the tenant's primary budget. Canary traffic bills against the tenant because the tenant saw the response. Mixing them in the billing-event stream is how FinOps review meetings turn into forensics sessions.

## Common routing anti-patterns

Four shapes that look right on a whiteboard and do not survive a quarter of production.

- **Random weighted routing across providers.** "20% Anthropic, 80% OpenAI" is a shape that eliminates the ability to pin a model to an evaluation score or to debug a regression. Use canary routing for intentional cutovers; do not treat permanent load-splitting as a routing policy.
- **Fallback loops that span regions silently.** A fallback entry that sits in a region the tenant forbids is a breach the next time the primary's region degrades. The tenancy allow-list (chapter 04) is the mitigation; enforce it in `tenancy_allowed()` on every candidate.
- **Per-request provider selection by cost.** A gateway that picks "the cheapest currently-eligible provider" per request is a gateway that produces non-reproducible evaluations and makes cache affinity (chapter 05) unachievable. Cost-driven routing belongs at the model-identifier choice (which platform model identifier does the agent use), not at the per-call provider choice.
- **Opaque provider adapters.** An adapter that logs `"provider call succeeded"` with no provider-native fields is an adapter that cannot be debugged. Every adapter emits structured events with the provider-native request ID, the finish reason, and the rate-limit headers.

## Summary

- Use a platform-coded internal model identifier (`platform:writer-v1`), not a provider name and not a generic family label. Agents reference the identifier; the gateway owns the mapping.
- The routing table is a versioned, signed config bundle with a primary and ordered fallbacks per identifier, explicit regions, explicit key references, and explicit capability declarations. Config changes roll independently of application deploys.
- The capability matrix is the gateway's model of what each provider+model supports. Routing refuses a fallback that cannot satisfy the request's capability needs.
- Provider abstraction lives at the adapter layer: system prompt, tool-call shapes, and finish reasons all translate to a single internal shape. The adapters are where the single most common silent bugs ship.
- Routing selection on the hot path: capability check → tenancy filter → cache-affinity reorder → health reorder → walk candidates. Attempts are logged; the attempt list is part of the response envelope.
- Three routing modes — primary, shadow, canary — are distinct and billed distinctly. Random weighted routing is an anti-pattern.

The next chapter picks up the retry policy that each attempt runs under and the fallback semantics tuned to the tool-calling loop specifically — where naive proxy retries behave worst.
