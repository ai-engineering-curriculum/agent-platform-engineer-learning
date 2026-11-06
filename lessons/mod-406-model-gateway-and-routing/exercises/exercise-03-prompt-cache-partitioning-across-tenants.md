# exercise-03: Prompt-Cache Partitioning Across Tenants

**Estimated effort:** 3 hours · **Output:** one namespace-composition module with tests, one prefix-normalisation serialiser with tests, one cache-affinity routing strategy against a stub provider, one cross-tenant leakage test suite (the main deliverable), and a short README (~1 page) · **Pairs with:** chapter [05](../05-prompt-cache-coordination-across-tenants.md); feeds exercise 04 and project 404.

## Objective

Build the three artefacts that make prompt caching safe to run in a multi-tenant gateway: the tenant-scoped **cache namespace composer**, the deterministic **prefix-normalisation serialiser**, and a **cache-affinity router** that keeps same-tenant same-prefix requests on the same backend. Then prove — with tests, not assertion — that the chapter-05 tenant-isolation invariant holds: no cache entry is re-usable across the tenancy boundary under any construction a reasonable adversary can produce.

A well-authored cache layer makes the savings from chapter 05 accrue (visible as rising cache_hit_rate and rising input_cached in chapter 04's billing events) without leaking context (visible as a zero cross-namespace collision rate, by construction). If your implementation lets a tenant control the namespace from request input, or hashes on content alone, or lets the cache state outlive the tenant boundary, a reviewer will send it back.

## Prerequisites

- You have read chapter [05](../05-prompt-cache-coordination-across-tenants.md) and understand the two invariants (namespace composition and prefix normalisation).
- You have the exercise-01 router available (or a stub router that walks a candidates list). The affinity router in this exercise extends the one from exercise 01.
- You have read [Anthropic's prompt-caching documentation](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching), [OpenAI's prompt-caching documentation](https://platform.openai.com/docs/guides/prompt-caching), and [Google Gemini's cachedContents API](https://ai.google.dev/gemini-api/docs/caching) at reading time.
- You have Redis (or any strongly-consistent K/V store) available for the affinity index.

## Problem statement

Three tenants share a gateway and all use `platform:writer-v1` (Anthropic `claude-sonnet-4-6`). Each has a representative stable prefix.

- **`acme`** — system prompt + a 40k-token internal knowledge-base extract, loaded at the start of every session. Volatile portion: the user's message and an appended tool-result loop. The KB extract and system prompt total ~45k tokens; expected cache hit rate: >80%.
- **`sigma-labs`** — system prompt + tool-definitions block. Volatile portion: the user's message. No embedded documents; stable prefix is ~3k tokens; expected cache hit rate: >60% on repeat sessions within TTL.
- **`globotech`** — system prompt + user-identity preamble templated per-user ("You are assisting Alice Smith..."). Volatile portion: the message and tool_result loop. Stable prefix *appears* to be ~5k tokens but shreds across users because of the preamble.

Your cache layer serves all three tenants concurrently without a single cache entry crossing the tenancy boundary. `globotech`'s shape is explicitly in-scope because chapter 05 says agent-design interacts with cache: your deliverable surfaces the shred as a diagnostic, not a silent regression.

## Requirements

### Part 1 — The namespace composer

Author a module (one file, 50–150 lines) that exposes a single pure function:

```
compose_namespace(request: GatewayRequest) -> CacheNamespace
```

Invariants:

- The namespace is derived from the request's **gateway-authoritative** fields (`tenant_id`, `project_tag`, `model_identifier`, `resolved_provider`, `resolved_region`, `api_key_fingerprint`). No field in the namespace may be read from caller-supplied input directly — the gateway derives each.
- `tenant_id` is the first component of the namespace string. A test confirms this ordering is invariant across refactors.
- `api_key_fingerprint` is a short deterministic hash of the resolved key (e.g. first 16 hex chars of `sha256(key)`). Keys themselves never appear in the namespace.
- The function is pure: same inputs produce same output.

Tests for this module must include:

- Two requests with identical business content but different tenants produce different namespaces.
- Two requests with identical tenants but different regions produce different namespaces.
- A hostile request attempting to set `tenant_id` via a header or a JSON field other than the gateway-derived field does not influence the namespace — the fuzz test tries 20+ injection shapes.

### Part 2 — The prefix-normalisation serialiser

Author a serialiser (one file, 100–250 lines) that turns a gateway request's stable-prefix portion into deterministic bytes. "Stable-prefix portion" is defined as:

- The `system` prompt (string or list of blocks, per provider adapter).
- The `tools` array, if present.
- Any content blocks explicitly marked `cache_control: {"type": "ephemeral"}` (Anthropic) or an equivalent "cache up to this point" marker. Content after the last marker is the volatile suffix and is *not* included.

Normalisation rules:

- JSON serialisation uses sorted keys and no insignificant whitespace.
- Strings are Unicode NFC-normalised.
- Tool-definition arrays are sorted by tool name; each tool's JSON-schema properties are sorted by key; enum values are sorted; `description` text is NFC-normalised.
- `cache_control` markers themselves are stripped from the normalised bytes (they are metadata, not content); the serialiser's output is the content that gets hashed, not including the markers.
- The output is `bytes`, not a Python / JS string — hashing happens on bytes.

Expose:

```
serialise_stable_prefix(request: GatewayRequest) -> bytes
prefix_hash(request: GatewayRequest) -> str   # "sha256:..."
```

Tests for this module must include:

- Two requests with the same system prompt serialised in different-key-order JSON produce the same hash.
- Two requests differing only in trailing whitespace on the system prompt produce the same hash (if your normalisation strips trailing whitespace) — document the choice either way.
- A request whose tools list differs only in ordering produces the same hash.
- A request with a different user message but identical stable prefix produces the same hash.
- A request where the `cache_control` marker is placed earlier (less content cached) produces a different hash than the full prefix.
- Fuzz test: generate 1000 random (stable-prefix, volatile-suffix) pairs; assert `prefix_hash` depends only on the stable prefix.

### Part 3 — The cache-affinity router

Extend the exercise-01 candidate-walk with an affinity-aware reorder step. Implement scheme 2 from chapter 05 (per-namespace sticky) as the baseline, and scheme 3 (per-(namespace, prefix_hash) sticky with an index) as an opt-in per model identifier.

- The baseline scheme uses a consistent-hash function on the namespace string to pick a preferred backend from the candidate list.
- The opt-in scheme queries a Redis index `affinity:{namespace}:{prefix_hash_prefix_16}` to find the backend that last served this prefix, and prefers it if still healthy.
- Affinity *reorders* the candidate list; it does not add or remove candidates. Health signal (chapter 03's circuit-breaker state) is consulted after affinity reordering; an open-circuit backend is skipped even if it is the affinity-preferred one.
- On a successful response, the router writes `affinity:{namespace}:{prefix_hash_prefix_16} -> backend_id` with a TTL matching the provider's cache TTL (5 minutes for Anthropic default; configurable).

Test scenarios:

- A sequence of 10 requests with the same namespace and same prefix hash land on the same backend (baseline + opt-in).
- Across two namespaces, requests distribute across backends by the consistent-hash, not all landing on one.
- An open-circuit backend is skipped even when it is the affinity-preferred one; the next-preferred backend is chosen.
- A TTL expiry re-randomises affinity on the next request — the test fast-forwards the Redis TTL.

### Part 4 — The cross-tenant leakage test suite (main deliverable)

Author `tests/cross_tenant_leakage_test.*` exercising every realistic path for a cross-tenant leak. Each test is adversarial; passing requires that the gateway prevents the leak by construction, not by chance.

At minimum:

- **Byte-identical stable prefix, different tenants.** Tenant A and tenant B send requests with the *exact same system prompt bytes*. Assert: different namespaces, different cache keys, two independent provider calls.
- **Byte-identical tool definitions, different tenants.** Same assertion.
- **Request-injected tenant override attempt.** Tenant B sends a request with a `tenant_id: acme` header. The gateway authentication layer (stub) resolves the actual tenant from the SPIFFE identity. Assert: the authoritative tenant in the derived namespace is `sigma-labs` (or whatever the caller actually is), ignoring the injected header.
- **API-key-fingerprint isolation.** Tenant A uses the platform-pooled Anthropic key; tenant C uses BYOK. Two requests with identical bytes produce different namespaces due to the differing `api_key_fingerprint`.
- **Namespace-component ordering.** Rearrange the composition order experimentally; verify the resulting namespaces are different and no accidental collision appears.
- **Fuzz test.** 10,000 random request pairs with random stable-prefix bytes and random tenant assignments. Assert: for all pairs, if the tenants differ, the namespaces differ, even when the prefix bytes are coincidentally identical.

Each test both asserts the outcome and, where possible, demonstrates the mechanism — e.g. logs the derived namespace values.

### Part 5 — Diagnostics for the `globotech`-shape anti-pattern

Add a diagnostic emitter that, when the `cache_affinity_backend_count{namespace}` metric for a namespace rises or the `cache_miss_cause` includes a large fraction of `prefix_changed` despite similar user-visible behaviour, emits a structured warning:

```json
{
  "diagnostic": "cache.volatile_content_in_prefix_suspected",
  "namespace": "globotech:writer-v1:anthropic:us-east-1:a3f5...",
  "window": "15m",
  "distinct_prefix_hashes": 847,
  "distinct_request_count": 1024,
  "advice": "stable prefix is varying across requests under the same namespace; check whether user-specific content is in the cacheable portion"
}
```

The emitter should surface the `globotech` shape (user-identity in the system prompt) within the first few hundred requests in a simulated load, not after a human review.

### Part 6 — The README

One page covering:

- Which three tenants you simulated and the stable-prefix shape of each.
- The namespace composition you chose, with a one-line rationale per component.
- The normalisation rules you adopted (and any you deliberately rejected, like whitespace-collapsing on the system prompt — chapter 05 warns against it).
- The tenant-isolation invariant from chapter 05, restated in your own words, and how each of the five tests in part 4 corresponds to a specific threat model.
- The `globotech` diagnostic — how you detect the shape and what you advise.
- One open question you would route to `ai-risk-engineer` or the registry team (mod-403). Example: "the current cache namespace does not carry the policy bundle hash; if a tenant's redaction obligations change, should we invalidate their cache?"

## Starter guidance

- **Namespace is derived at the gateway from authoritative fields; never read from request input.** This is the single most common beginner bug. Write the fuzz test early; let it catch any attempt to shortcut.
- **Serialise, then hash.** Hashing a Python `dict` directly uses whatever ordering the dict happens to have; it will compare equal within a process and differ across restarts. Always serialise to canonical bytes, then hash.
- **Prefer `sha256`.** Shorter hashes (md5, sha1) are fine for cache coordination but you lose the audit property — reviewers expect sha256 on security-adjacent hashing.
- **Do not include the user-visible message in the "stable prefix" by accident.** The split is explicit: the system prompt, tools, and content explicitly marked as cacheable are stable; everything else is volatile. Your serialiser's unit tests should exercise the split explicitly.
- **Consistent-hashing library choice matters less than you think.** Any reasonable library (Python `uhashring`, Go `cespare/xxhash`-based rings, Node `hashring`) works. Pin the library and the ring configuration.
- **Think about the TTL you choose.** Anthropic 5-minute default is the minimum useful value for cache-affinity; longer values keep the index stale longer. 5 minutes matches the provider; use it as the default.
- **The `globotech` diagnostic is a product feature.** It is how your platform surfaces an agent-design anti-pattern to the application team. Mock traffic until the diagnostic reliably fires.

## Acceptance criteria

Mark done when:

- [ ] `compose_namespace` is pure and derives every component from gateway-authoritative fields; the fuzz test (20+ injection shapes) is green.
- [ ] `serialise_stable_prefix` and `prefix_hash` are deterministic and produce identical hashes for semantically-equivalent inputs; the fuzz test (1000 pairs) is green.
- [ ] The cache-affinity router sticks same-namespace requests to the same backend baseline-scheme or opt-in-scheme correctly; affinity never overrides health signal.
- [ ] All six cross-tenant leakage tests pass; the fuzz test (10,000 pairs) finds no leak.
- [ ] The `globotech` diagnostic fires on a seeded load replicating the user-identity-in-system-prompt anti-pattern within 200 requests.
- [ ] Every cache entry's billing event (stub is fine — chapter 04 full event is exercise 04's job) carries the derived `cache.namespace`.
- [ ] The README documents the composition, the invariants, the five threat scenarios, and one open question.

## Stretch goals

- **Multi-provider affinity index.** When the same `platform:writer-v1` identifier has fallback mappings on multiple providers, the affinity index is per-provider; a tenant with an Anthropic cache entry and an OpenAI cache entry has *two* affinity records, and the router picks the one matching the chosen candidate.
- **Cache warming for high-value identifiers.** A background job re-sends the stable prefix every 4 minutes to keep an Anthropic cache hot across TTL. Instrument the cost of the write against the saved reads; show when warming pays back.
- **Namespace-change rollover.** A tenant changes their project tag (part of the namespace). The old cache entries do not automatically re-align; the next request with the new tag misses. Document the behaviour explicitly in the diagnostic so the application team is not surprised.
- **Encrypted-prefix hashing.** For an enterprise tenant with a residency-sensitive stable prefix, include a tenant-specific HMAC key in the namespace (`api_key_fingerprint` + tenant-controlled HMAC). A gateway operator reading logs sees opaque namespaces; the tenant can still reason about their own.
- **Shadow-cache exercise.** Allow the gateway to maintain its own hash-table in parallel with the provider's cache (i.e. "we think the provider should have hit, did it?"). Emit a mismatch metric; run against real Anthropic traffic for an hour and see how often the gateway's expectation matches reality.
