# Prompt-Cache Coordination Across Tenants

Chapter 04 pinned split-token accounting. This chapter builds the mechanism that actually drives the cached-token column into the favourable half of the split — prompt caching, coordinated across tenants so that savings accrue without leaking context. It is the single highest-leverage optimisation available to a model gateway and the single most dangerous one if done wrong.

Prompt caching, briefly: when consecutive requests share a long common prefix (a system prompt, a shared document, a tool-definition block), the provider can keep the computed intermediate state for that prefix in memory, re-use it on the next request, and charge a reduced rate for the cached portion of the input. The economics are real — cache-read rates on major providers run between 10% and 50% of the non-cached input rate — and the engineering is subtle.

The subtlety is twofold. **First**, caching is affinity-sensitive: the same cache entry is only hot on the same serving backend. A routing decision that spreads a tenant's requests across backends shreds the cache hit rate. **Second**, caching is tenant-boundary-sensitive: a cache keyed naively on prefix content allows a cross-tenant collision that leaks exactly the kind of context most sensitive to leak (system prompts, instructions, embedded data). The gateway's job is to engineer the first in while engineering the second out.

<!-- needs-research: confirm each provider's current prompt-caching mechanics and pricing at reading time. Caches, discounts, TTLs, and minimum-prefix rules have all moved across 2024-2026; the specifics below are directional. -->

## Where caching lives — provider-managed vs. gateway-managed

Two distinct architectures the gateway composes with. Name both, know which you are in.

### Provider-managed caching

The provider's inference infrastructure owns the cache. The gateway's role is to shape the request so the provider recognises the prefix as cacheable and to route to the backend that already holds the entry.

- **Anthropic prompt caching.** Opt-in via `cache_control` blocks on specific content items in the request. Cached blocks have a 5-minute (default) or 1-hour (extended) TTL from last use. Cache reads are billed at ~10% of non-cached input; cache writes are billed at ~125% (5m TTL) or higher (1h TTL) of non-cached input. The gateway's job: emit `cache_control` on the stable prefix (system prompt, tool definitions, long documents), re-use the same prefix across consecutive requests, and respect a minimum-prefix-size requirement.
- **OpenAI prompt caching.** Automatic for recent-enough models when a request matches a prior request's prefix by a minimum number of tokens (e.g. 1024 at the time of writing). Cache reads are discounted (currently 50% off `gpt-4o`-family input tokens; higher on newer models); writes are free (automatic). The gateway's job: construct requests so the stable prefix is at the start, since the matching is prefix-based and byte-identical through the cut-off.
- **Google Gemini cachedContents.** Explicit resource creation (`cachedContents` API) tied to a model and a region, re-used by resource name. TTLs are configurable. Reads are discounted; the cache-write is a separate API call with a fixed cost. The gateway's job: manage the cachedContents lifecycle, re-use by name, and tear down on TTL expiry.
- **AWS Bedrock.** Varies by model-provider; Bedrock-hosted Anthropic supports the Anthropic-style `cache_control`; Bedrock-hosted other models vary. The gateway treats it as whichever underlying provider's shape applies.

In all provider-managed cases, the gateway does *not* hold the cached content; the provider does. The gateway's responsibility is routing and shape.

### Gateway-managed caching

For self-hosted inference (vLLM, SGLang, TensorRT-LLM), the gateway may own the cache layer explicitly. The serving substrate has a KV-cache; the gateway's role is to *route* requests to the serving instance whose KV-cache already holds the relevant prefix, which turns into a request-affinity problem (see later in this chapter).

Some mature gateways layer an application-level response cache on top (Portkey calls theirs a "semantic cache"), which bypasses the model entirely when a recent request with an identical / highly-similar input exists. That is a different beast from prompt caching — it caches the *response*, not the *prefix state* — and carries its own correctness traps (two requests at different timestamps or with different session context should get different responses even if the prompt text matches). This chapter focuses on prefix caching; response caching is a chapter-06-adjacent decision about whether to let a cached response substitute for a real call, usually governed by policy.

## The tenant-isolation invariant

Pin the invariant before writing anything. Then verify the design against it.

> **No prompt-cache entry visible to or re-usable on behalf of tenant B shall contain content originating from tenant A — regardless of prefix hash collision, byte-identical content, or request routing.**

Three ways a design can violate this, each more subtle than the last.

1. **Shared cache namespace across tenants on the provider.** If the gateway uses a single API key for all tenants on a provider that scopes cache state per-key, both tenants' prefixes land in the same cache namespace. If two tenants ever share a byte-identical prefix (e.g. a common document or prompt template), they share a cache entry — and the provider has no way to know that was wrong.
2. **Cache key keyed on content hash alone.** A gateway-managed cache that keys on `sha256(prefix)` provides the exact same leakage: identical content hashes to the same entry. The attack surface is any shared snippet (a vendor's documentation, a common prompt, the text "You are a helpful assistant"). The fix is the key composition in the next section.
3. **Affinity routing that crosses tenant boundaries.** A cache-affinity routing rule that sends "any request whose prefix hashes to X" to the same backend will send tenant A's and tenant B's X-prefix requests to the same backend, and the provider's per-backend cache holds both. Even if the API-key scoping prevents direct re-use, operational diagnostics and incident recovery can expose it.

The mitigation for all three is to scope the cache by tenant at the key composition, at the API-key selection, and at the affinity routing. The next section is the design.

## The cache namespace composition

The gateway composes a **cache namespace** per request from a deterministic set of fields. The namespace is what distinguishes one cache entry from another; two requests share a cache entry if and only if they share a namespace. A defensible composition:

```
cache_namespace = tenant_id
                | ":"
                | project_tag_or_default
                | ":"
                | model_identifier
                | ":"
                | provider
                | ":"
                | region
                | ":"
                | api_key_fingerprint
```

Then the **prefix hash** — `sha256(serialised_prefix_bytes)` — is computed within the namespace. The cache key the gateway consults is `(cache_namespace, prefix_hash)`. Two requests from different tenants that have the same prefix bytes have different namespaces and therefore different cache keys; they will never collide.

Four design points:

1. **Tenant is the first component.** A namespace that starts with model or provider is one typo in the key constructor away from a cross-tenant collision. Lead with tenant.
2. **Provider and region are included.** The physical cache lives per-provider per-region; a namespace that ignores region treats two caches as one.
3. **API-key fingerprint is included.** When a tenant has its own BYOK (bring-your-own-key) arrangement — chapter 07's enterprise-tier shape — the cache is scoped to the key, not to the gateway's pooled key.
4. **The namespace is logged on every billing event.** Chapter 04's event schema carries `cache.namespace`; this is what makes the invariant verifiable from the audit stream.

## Prefix normalisation

Two requests with the same semantic prefix but different byte representations should hash to the same prefix hash — otherwise the cache hit rate suffers. Two requests with different semantic prefixes but identical bytes after serialisation must not collide. Normalisation lives at the serialiser.

Concrete choices:

- **Fix the JSON serialisation.** Sort keys, use a canonical whitespace (or no whitespace), decide on Unicode normalisation (NFC is the common choice). Two requests whose JSON differs only in key ordering should hash the same.
- **Handle tool-definition blocks deterministically.** If the tool list is a top-level field (Anthropic `tools`, OpenAI `tools`), its serialisation is part of the cached prefix on providers that cache tool definitions. Sort tools by name; sort properties within each tool schema; sort enum values. Otherwise two requests with semantically-identical tools emit different hashes.
- **Normalise system-prompt whitespace cautiously.** Semantic equivalence depends on the model. Trimming trailing whitespace is usually safe; collapsing internal whitespace is usually not — it changes the model's behaviour.
- **Separate volatile content from stable prefix.** The cacheable portion is the part that is stable across requests (system prompt, tool defs, long document). The volatile portion (user message, tool_result) is appended after the cache boundary. On Anthropic, this is literally the position of the last `cache_control` block; on OpenAI, it is whether your stable prefix comes first in the messages array.

The prefix-normalisation rules are part of the gateway config, versioned. A change to the normalisation rule invalidates all caches (because hashes change). Promote carefully and announce.

## Cache-affinity routing

Cache affinity means: given a request, prefer to route it to a backend where the cache for its prefix is hot. For provider-managed caches, "hot on a backend" is a provider-internal concept the gateway only approximates; for gateway-managed caches on self-hosted inference, the gateway can know precisely.

Three affinity schemes, in rising order of specificity:

### Scheme 1 — Per-tenant sticky

Every request from tenant T routes to the same backend (one of the provider's regional endpoints, or one of a pool of self-hosted instances). Simple, obvious, and the first step. It gives cache hits whenever the tenant's prefix has been seen on that backend in the TTL window.

- Works when the per-tenant volume is high enough to keep the cache warm.
- Falls over on low-traffic tenants (cache TTL expires between requests) and on very-large tenants (one backend saturates before another gets load).
- Trivial to implement: consistent-hash on `tenant_id` into the backend pool.

### Scheme 2 — Per-namespace sticky

Route on the full cache namespace, not just on tenant. A tenant with multiple projects, model identifiers, or regions distributes across backends. Within a namespace, every request lands on the same backend.

- Spreads load better than tenant-sticky.
- Still cheap: consistent-hash on the namespace string.
- Preserves cache locality within a (tenant, project, model) stripe, which matches the empirical access pattern.

### Scheme 3 — Per-prefix-hash sticky, scoped by namespace

For each (namespace, prefix_hash) pair, record which backend served it last; prefer that backend on the next request. The gateway maintains an index.

- Maximises cache hit rate — every request tries to land where its exact prefix was last computed.
- Index state must be shared across gateway replicas (Redis, a coordinator service). Operational cost.
- Falls over if the backend went unhealthy since the last request. Fallback is to scheme 2.

Most mature gateways run scheme 2 as the default and scheme 3 for high-value identifiers (high-cost models, large prefixes). Scheme 1 is a prototype shape.

### The index for scheme 3

A representative shape:

```
key:   affinity:{cache_namespace}:{prefix_hash_prefix_16}
value: {
  "backend": "anthropic:us-east-1:key_fingerprint_a3f5",
  "last_seen_at": "2026-11-06T18:22:04Z",
  "ttl_expires_at": "2026-11-06T18:27:04Z",
  "estimated_cached_tokens": 4712
}
```

- Key is namespace-scoped, so one tenant's entries cannot overwrite another's.
- `prefix_hash_prefix_16` is the first 16 hex chars of the hash — short enough to fit in a key efficiently, long enough that collisions within a namespace are negligible.
- TTL on the entry matches the provider's cache TTL; stale entries are either lazily invalidated on health-check failure or actively evicted by the TTL.

## Interaction with the fallback walk

Chapter 03's retry policy walks candidates on failure. Cache affinity reorders candidates on the happy path (chapter 02). Pin the sequence: first apply affinity, then apply health-based reordering, then walk under the retry policy.

Two interactions to internalise:

- **Cache affinity does not override circuit-breakers.** A backend with a hot cache that just opened its circuit is still skipped — the latency and cost of a provider-overloaded backend overwhelm the cache win.
- **A fallback to a different region or provider is a cache miss by definition.** The attempts list on the billing event should distinguish "primary attempt served from cache" from "fallback attempt had no cache opportunity". Price-regression alarms (chapter 04) filter on this when assessing cost drift.

## Cache-shape instrumentation

Six metrics the gateway emits; each answers a question an operator will ask within the first week.

- **`cache_hit_rate{namespace}`** — fraction of requests where `cache.outcome ∈ {hit_partial, hit_full}`. The headline number.
- **`cache_prefix_tokens_served{namespace}`** — total cached input tokens on the window. Multiplied by the cache-read discount from the price table, this is the dollar saving.
- **`cache_write_tokens{namespace}`** — total cache-write input tokens on the window. On Anthropic this is cost added; on OpenAI it is zero. Watch for cache thrash (writes without reads).
- **`cache_miss_cause{namespace, cause}`** — distribution of miss causes: `first_request`, `ttl_expired`, `prefix_changed`, `fallback_to_different_region`, `cache_disabled`. Catches the "why is my cache hit rate low" question without manual diagnosis.
- **`cache_affinity_backend_count{namespace}`** — how many distinct backends served requests within the namespace in the window. Rising count usually means load balancing is beating cache affinity; falls back to scheme 1 / 2.
- **`cache_prefix_collision_rate{namespace}`** — fraction of requests whose prefix hash matched a different prefix hash in another namespace during the window. Should be effectively zero. A nonzero value is either a hashing bug or a namespace collision; both are invariant violations.

## Diagnostics — the four questions

When the cache hit rate is lower than expected, run these in order.

1. **Is the stable prefix identical byte-for-byte across requests?** Compute the prefix hash on two sample requests and diff. A single whitespace change at the top of the system prompt invalidates. If the two hashes differ, normalisation is the fix.
2. **Does the request include a `cache_control` block (or the provider's equivalent) at the right position?** If using Anthropic, the `cache_control` block must sit at the end of the cacheable portion, not the beginning. If using Google, the cachedContents resource must be referenced correctly. The adapter's unit tests should assert the position.
3. **Are requests landing on the same backend?** Check `cache_affinity_backend_count`. If high, the affinity scheme is not sticking. Verify the hash input (is it the namespace or a derivative?) and the backend pool health.
4. **Are requests arriving inside the TTL window?** Check the time between consecutive same-namespace requests against the cache TTL (5 minutes default on Anthropic; longer on Google and OpenAI). If larger than TTL, the cache is cold every time; increasing TTL (where the provider supports it) or moving the identifier to a high-traffic pool is the fix.

## Anti-patterns

- **Keying the cache on content alone.** Guarantees cross-tenant collisions on any shared snippet.
- **Sharing a single API key across all tenants.** Collapses provider-scoped cache state into a single pool. Even if the namespace is correct, operational diagnostics expose keys.
- **Cache-affinity routing without health reordering.** Sends traffic to an open-circuit backend because the cache is hot there; then every request fails over and bills twice.
- **Caching tool results.** Tool results are volatile and session-specific; caching them is caching the wrong thing. The stable prefix is system prompt + tools + long document; the volatile suffix is the conversation and tool results.
- **Hashing a prefix over a non-normalised serialisation.** Guarantees low cache hit rates and makes them a mystery.
- **Letting the TTL expire for a high-value identifier.** A warming loop (a background request every 4 minutes to keep the Anthropic 5m cache hot) can be a cost win if the saved reads outweigh the write; model it before adopting.
- **Mixing provider-managed and gateway-managed caches on the same identifier without accounting.** Billing-event cost fields double-count or miss the discount.

## Security review checklist for the cache

Three checks that should be re-run whenever the cache code changes.

- [ ] **Unit test** — two requests from different tenants with byte-identical prefixes produce different cache keys and different cache namespaces, and the gateway issues two independent provider calls.
- [ ] **Unit test** — the cache namespace is emitted on the billing event and matches the request's tenant.
- [ ] **Integration test** — a cross-tenant leak attempt (a tenant sends a request whose namespace is manipulated to match another tenant's) is refused at the gateway's request-validation step. The namespace is derived by the gateway, not passed by the caller.

If any of the three breaks, the invariant does not hold. Fix before shipping.

## Summary

- Prompt caching is the single highest-leverage optimisation available to a model gateway. The economics come from the split-token accounting of chapter 04; the engineering lives in this chapter.
- Provider-managed caches (Anthropic cache_control, OpenAI automatic, Google cachedContents, Bedrock per-underlying-provider) and gateway-managed caches (on self-hosted inference) require different coordination; both are composable.
- The tenant-isolation invariant is non-negotiable. Compose the cache namespace from tenant + project + model + provider + region + API-key fingerprint; key on (namespace, prefix_hash); never key on content alone.
- Prefix normalisation (canonical JSON, deterministic tool-def ordering, careful whitespace) decides whether cache hits actually happen on byte-identical serialisations.
- Cache affinity has three progressively specific schemes: per-tenant sticky, per-namespace sticky, per-(namespace, prefix_hash) sticky with an index. Most platforms run per-namespace by default.
- Instrument hit rate, prefix tokens served, write tokens, miss cause, backend count, and cross-namespace collision rate. The last metric should be zero; a non-zero value is an invariant violation, not a performance issue.
- Anti-patterns: content-keyed caches, shared API keys, caching tool results, non-normalised serialisations. Each is a known failure mode.

The next chapter picks up schema enforcement and pre-egress PII redaction — the two obligations the gateway runs on request content, separate from caching but often on the same hot path.
