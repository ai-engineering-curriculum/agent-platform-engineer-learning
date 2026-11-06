# Quotas, Burst Budgets, and Cost Attribution

A gateway without quotas is a gateway with one open credit card. Chapter 03 built the error path on the assumption that requests the gateway accepts actually get to run; this chapter decides whether to accept the request in the first place, and on the back end, how to attribute the cost it incurred to a tenant, user, session, and model in a form FinOps can trust.

The thread through this chapter is split-token accounting. Prompt caching changes the denominators (the input-token column splits into cached and non-cached) and changes the pricing (the cached bucket is 50–90% cheaper on major providers at the time of writing; chapter 05 gets into the details). A quota or billing event that does not carry the split is a quota or billing event that silently regresses the moment a tenant starts caching effectively. Pin it early, enforce it everywhere.

## What the gateway counts

Three distinct quantities, measured per request and aggregated per tenant / per user / per session / per model-identifier:

- **Tokens** — input (split cached / non-cached), output, tool-use input (the part of input that is a tool-definition block plus tool_result content), tool-use output (the part of output that is a `tool_use` block). The provider's response carries the numbers; the gateway normalises them into the same column shape across providers. Where a provider does not report split cached/non-cached directly, the gateway infers it from its own cache state (chapter 05) and flags the inference.
- **Requests** — one per gateway request (not per attempt). Chapter 03 said the request is the unit of retry; it is also the unit of request-count metrics.
- **Wall-clock time** — end-to-end latency of the gateway request, split by phase (gateway queue, provider call, streaming time). Useful for SLOs, not usually for billing.

Token counts are reported by the provider; the gateway does not re-tokenise. Re-tokenising at the gateway with a different tokenizer than the provider uses is the single most common source of silently-wrong cost dashboards. If you must estimate pre-call (for admission), use the provider's tokenizer or the provider's published rule-of-thumb and correct from the response; never substitute a different tokenizer and call the number authoritative.

<!-- needs-research: confirm whether the specific providers you ship against report split cached/non-cached input tokens in their response envelopes at reading time — this surface has been changing across 2025-2026. Anthropic reports cache_read_input_tokens and cache_creation_input_tokens in usage; OpenAI reports cached_tokens under prompt_tokens_details; Google's gemini API reports cachedContentTokenCount under usageMetadata. Verify field names at reading time. -->

## Quotas vs. budgets vs. rate limits — three distinct concepts

Treat these as three separate layers. Collapsing them is how gateways end up with one giant `rate_limit` variable that cannot answer any question precisely.

- **Rate limit** — short-window request-count or token-count ceiling, enforced to protect the provider (the gateway cannot send more than the provider's TPM / RPM allows) and to protect the gateway itself (a single tenant cannot saturate the gateway's worker pool). Measured in seconds-to-minute windows. Returns 429-class responses. Owned by the gateway for self-protection; parameterised from provider limits.
- **Quota** — longer-window token or cost ceiling with business meaning: "tenant acme is entitled to 50M tokens per month on `platform:writer-v1`". Measured in day-to-month windows. Breach is a tenant-visible event — the agent's call fails with a quota-exceeded reason the application should surface cleanly.
- **Burst budget** — short-window spike allowance that sits between the two: "tenant acme can spend up to 500k tokens in any 5-minute window on interactive paths before being throttled". Protects against sudden loops, prompt-injection-driven spike attacks, and runaway canary deployments.

The semantics are different enough that the enforcement store, the error code, and the recovery path differ. Fold them together only when you understand why the three were separate to begin with.

## The admission decision

A request arrives at the gateway. Before it hits the router (chapter 02) or the retry policy (chapter 03), it passes the admission decision:

```python
def admit(request: GatewayRequest) -> AdmissionResult:
    # 1. Estimate pre-call cost (tokens → cost) for the request.
    estimate = estimate_cost(request)  # returns (tokens_input_est, max_output_tokens, cost_usd_est)

    # 2. Short-window rate limit (per-tenant, per-model-identifier).
    rl = rate_limiter.allow(tenant=request.tenant_id,
                            model=request.model_id,
                            tokens_est=estimate.tokens_total)
    if not rl.allowed:
        return AdmissionResult(deny, reason="rate_limit.exceeded",
                               retry_after=rl.retry_after)

    # 3. Burst budget (per-tenant, per-interactive/batch class).
    bb = burst_budget.try_consume(tenant=request.tenant_id,
                                  path_class=request.path_class,
                                  tokens_est=estimate.tokens_total,
                                  cost_est=estimate.cost_usd_est)
    if not bb.allowed:
        return AdmissionResult(deny, reason="burst_budget.exceeded",
                               retry_after=bb.retry_after)

    # 4. Monthly quota (per-tenant, per-model-identifier).
    q = quota.try_reserve(tenant=request.tenant_id,
                          model=request.model_id,
                          tokens_est=estimate.tokens_total,
                          cost_est=estimate.cost_usd_est)
    if not q.allowed:
        return AdmissionResult(deny, reason="quota.exceeded",
                               retry_after=None)       # no retry; new window required

    # 5. Session-level loop guard (optional, see below).
    sg = session_guard.check(session_id=request.session_id, request=request)
    if not sg.allowed:
        return AdmissionResult(deny, reason=sg.reason,
                               retry_after=None)

    return AdmissionResult(allow, reservation=reservation(rl, bb, q))
```

Three design points:

1. **The admission decision reserves cost against an estimate, not against the actual.** The actual post-call usage is reconciled on the billing event. Over-estimates get refunded to the quota at request completion; under-estimates get surcharged.
2. **The rate-limit, burst-budget, and quota layers share a reservation handle.** If the request is served and costs more than estimated, the reservation is updated — not three separate updates against three different stores racing each other.
3. **The admission decision carries structured reason codes.** The agent's SDK can distinguish `rate_limit.exceeded` (transient, retry with backoff) from `burst_budget.exceeded` (wait a short window) from `quota.exceeded` (admin action required).

## Estimating pre-call cost

A rough estimate is enough; perfect requires the provider's response. Three techniques, in rising order of precision:

- **Character-based.** `input_tokens ≈ len(prompt_text) / 4` is a widely-used rule for English text with major providers. Cheap, wrong on code and on non-English text, not catastrophic for admission decisions.
- **Provider tokenizer.** OpenAI publishes tiktoken; Anthropic publishes a token-counting API; Google exposes `countTokens`. Use the provider's tokenizer when you can; it is accurate and worth a short lookup call for high-cost requests.
- **Historical.** For a given tenant / model-identifier / session pair, the gateway's prior requests give a rolling distribution; use the P95 of recent input-length as the admission estimate if the request shape matches. Especially effective for recurring agent workflows.

The estimate's purpose is to catch pathological requests (a 500 k-token input that would blow the quota) before issuing them; small tenants of the budget do not need high precision. Over-estimate by 10–20% on admission and reconcile on the billing event is a common choice.

## The quota store

Per-tenant per-model-identifier quota state needs three properties:

- **Strongly consistent reads and writes.** Two concurrent requests from the same tenant cannot both succeed against the last 1k tokens of a 50M quota. Redis `INCRBYFLOAT` with a Lua script, or an RDBMS row with a `SELECT ... FOR UPDATE`, or a provider-native counter (DynamoDB conditional writes, Firestore transactions) — pick one with strong semantics.
- **Partitioned by (tenant_id, model_id, window).** The read pattern is "what is this tenant's remaining on this model in this window", a single-key lookup.
- **TTL'd on window.** A daily window resets at the day boundary; the quota row either re-initialises via a scheduled job or expires naturally via TTL and is lazily re-created.

Enforcement at read-time is strongly consistent; aggregate queries (dashboards, FinOps) can hit a replica. Keep the two paths separate.

## Rate limits — the provider ceiling and your own

Two independent rate-limit layers run in a model gateway:

1. **Provider-side ceiling.** Every major provider enforces per-API-key TPM (tokens per minute) and RPM (requests per minute) limits. The gateway tracks remaining capacity per key (often returned in response headers — `x-ratelimit-remaining-tokens`, `x-ratelimit-reset-tokens`) and spreads traffic across keys for the same provider.
2. **Gateway-side per-tenant limit.** A per-tenant cap on tokens-per-minute protects the shared provider pool from a single tenant's spike. This is where the gateway's quota admission denies a request whose rate pushes the tenant over a per-minute cap.

Pin a conceptual model: the provider ceiling is a *supply* constraint; the per-tenant limit is an *allocation* decision. Never confuse the two. A provider 429 means "we exceeded the supply"; a gateway 429 means "we allocated less than you are requesting". The error codes should differentiate.

## Burst budgets

A burst budget is the mechanism that keeps a well-intentioned agent from accidentally consuming the whole month in six minutes. The canonical shape is a token bucket:

- **Capacity** — the maximum burst size (e.g. 500 k tokens).
- **Refill rate** — tokens added per second (e.g. 1 k tokens/s, equivalent to 86.4M tokens/day sustained — above the quota but reachable only in burst).
- **Overflow policy** — requests that would overflow are denied with `burst_budget.exceeded` and a `retry_after` derived from the refill rate.

Burst budgets layer on top of the monthly quota. The quota caps total; the burst caps instantaneous. A tenant with a healthy monthly quota who hits burst limits is still consuming budget — the burst is a spike protector, not a spend protector.

Common per-tenant default shape: separate burst budgets for `path_class: interactive` (small bucket, high refill — protects UX) and `path_class: batch` (large bucket, slow refill — protects monthly spend from a runaway batch).

## Session-level loop guard

An optional admission layer that catches budget-exhausting loops before the quota does. Two heuristics most gateways include:

- **Per-session cumulative token cap.** A single session that has already consumed N tokens on the current logical turn is a session that has probably entered a loop. The cap is high (e.g. 1M tokens per session for a 200 k-context model), but it exists; breach is `session_guard.loop_suspected`.
- **Tool-call repetition detector.** If the last 5 model responses were tool_use blocks naming the same tool with structurally identical arguments, the loop guard flags it. The gateway either denies the next request (strict mode) or appends a system message to the next request ("you have called the same tool with identical arguments 5 consecutive times; are you stuck?") in advisory mode.

The loop guard is the gateway's defence against the budget-exhausting attack from chapter 01's threat model. The runtime has its own loop guard; this is defence in depth, enforced at the gateway so no agent code can be patched to disable it.

## Cost attribution — the billing event

Every completed request emits a single billing event to an append-only stream. The billing-event schema is the single most important artefact of this chapter; it is the truth that FinOps, chargeback, invoice reconciliation, and price-regression alarms all depend on.

```json
{
  "event_id": "be_01HV8X3M9F5A6R2B4C0D7E8G9H",
  "schema_version": 3,
  "gateway_request_id": "gw_01HV8X3M9F5A6R2B4C0D7E8G9H",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "received_at": "2026-11-06T18:22:04.173Z",
  "completed_at": "2026-11-06T18:22:05.417Z",
  "tenancy": {
    "tenant_id": "acme",
    "cost_center": "acme/eng/research",
    "project_tag": "writer-v1-rollout",
    "session_id": "s_01HV8X3M9F5A6R2B4C0D7E8G9H",
    "principal_subject": "alice@acme.com",
    "actor_spiffe_id": "spiffe://platform.local/runtime/prod"
  },
  "routing": {
    "model_id": "platform:writer-v1",
    "resolved_provider": "anthropic",
    "resolved_model": "claude-sonnet-4-6",
    "resolved_region": "us-east-1",
    "mode": "primary"
  },
  "tokens": {
    "input_cached": 4712,
    "input_non_cached": 318,
    "output": 442,
    "tool_use_input_included": 97,
    "tool_use_output_included": 120,
    "input_token_source": "provider_reported",
    "output_token_source": "provider_reported"
  },
  "cost": {
    "input_cached_usd": 0.00071,
    "input_non_cached_usd": 0.00238,
    "output_usd": 0.01326,
    "cache_write_usd": 0.0,
    "total_usd": 0.01635,
    "currency": "USD",
    "price_table_version": "2026-11-01",
    "price_table_hash": "sha256:9ce4..."
  },
  "cache": {
    "namespace": "acme:writer-v1",
    "prefix_hash": "sha256:...",
    "outcome": "hit_partial",
    "cache_ttl_bucket": "5m"
  },
  "attempts": [
    { "provider": "anthropic", "model": "claude-sonnet-4-6",
      "attempt_no": 1, "outcome": "overloaded_error", "latency_ms": 820,
      "tokens_consumed": {"input": 0, "output": 0} },
    { "provider": "anthropic", "model": "claude-sonnet-4-6",
      "attempt_no": 2, "outcome": "success", "latency_ms": 1240,
      "tokens_consumed": {"input": 5030, "output": 442} }
  ],
  "flags": ["success", "fallback_applied", "cache_hit"]
}
```

Five invariants this schema enforces:

1. **Split-token counts on every event.** `input_cached` and `input_non_cached` are separate columns. A single `input_tokens` column hides every caching regression.
2. **Cost is computed, not provider-reported.** The gateway applies `price_table_version` and `price_table_hash` locally. Provider invoices come weeks later and cannot drive real-time dashboards; the gateway's computed cost is the ground truth for operations and the provider invoice is the external reconciliation that catches gateway bugs.
3. **Cost-center and project tags travel on every event.** FinOps asks "how much did the research org spend on `platform:writer-v1`"; the answer is a group-by on `tenancy.cost_center` and `routing.model_id`. These tags come from the request (from the agent's identity and config), not from the gateway's inference.
4. **Attempts land on the event, not on a separate stream.** The sum of per-attempt token consumption equals the request-level totals. Mod-408's trace can join on `gateway_request_id`.
5. **`token_source` is explicit.** `provider_reported` means the provider's response carried the number; `gateway_estimated` means the gateway inferred it (typically for streaming interruption) and flagged the lower confidence. Every FinOps query should be able to filter out estimated events.

The billing-event stream is append-only. Corrections take the shape of compensating events (a new event with `type: adjustment` and a reference to the original), not mutations.

## The price table

A versioned, signed artefact that the gateway loads at startup and reloads on config promotion. A representative shape:

```yaml
# pricing/2026-11-01
version: "2026-11-01"
hash: sha256:9ce4...
effective_from: 2026-11-01T00:00:00Z
currency: USD
models:
  - provider: anthropic
    model: claude-sonnet-4-6
    region: us-east-1
    input_non_cached_per_mtok: 3.00
    input_cached_read_per_mtok: 0.30
    input_cached_write_per_mtok: 3.75
    output_per_mtok: 15.00
    tool_use_billing: included_in_io
  - provider: anthropic
    model: claude-opus-4-7
    region: us-east-1
    input_non_cached_per_mtok: 15.00
    input_cached_read_per_mtok: 1.50
    input_cached_write_per_mtok: 18.75
    output_per_mtok: 75.00
    tool_use_billing: included_in_io
  - provider: openai
    model: gpt-4o-2024-08-06
    region: us-east-1
    input_non_cached_per_mtok: 2.50
    input_cached_read_per_mtok: 1.25
    input_cached_write_per_mtok: null       # cache writes free on this model
    output_per_mtok: 10.00
    tool_use_billing: included_in_io
```

<!-- needs-research: confirm each per-model unit price against the provider's current pricing page at reading time. Model pricing has been the single fastest-moving surface across 2024-2026 and any numeric values here will be stale within quarters. -->

Two notes:

- **Cache-write and cache-read have different prices.** Anthropic's "prompt caching" typically charges a premium to *write* the cache and a discount to *read* it. OpenAI's prompt caching is often free to write (automatic) and discounted on read. Google's cachedContents is explicit. The price table carries all three columns so that the billing event shows exactly what happened.
- **Regional prices can differ.** `us-east-1` and `eu-west-1` may share a price today and diverge tomorrow. Carry the region on every price row even if the current values match.

Changing the price table is a config promotion. A new `price_table_version` propagates to the gateway through the config service; new requests use the new table; completed events reference their table version so the backfill is bounded.

## Price-regression alarm

The price-regression alarm is the FinOps equivalent of an uptime SLO. A representative shape, evaluated on a rolling window of billing events:

- **Compare `total_usd / output_tokens` at the group level** (tenant, model_id, path_class) across the current window and the baseline window (same group, previous window of equal length).
- **Alert on a ratio above 1.15** (15% cost-per-output-token regression) after filtering out requests where `cache.outcome` changed category (a tenant who stopped caching will show a step function).
- **Suppress noise by requiring a minimum event count** per group (e.g. 500 events / window).

The alarm catches two canonical failure modes: a provider's silent price change not reflected in the table, and a routing change that moved traffic to a more expensive mapping without planning. Both are findable in minutes rather than quarters.

## Billing-event consumers

The stream is consumed by at least four downstream services, each with different requirements:

- **Real-time dashboards** — stream-processing (Flink, Beam, Materialize, or a serverless alternative) aggregating by group with second-level freshness. Drives the operational FinOps view.
- **Daily / monthly aggregates** — batch job partitioning by day and tenant; writes to a warehouse (Snowflake, BigQuery, Redshift) for chargeback and invoice generation.
- **Price-regression alarms** — the fast-path consumer above.
- **Provider-invoice reconciliation** — a monthly job that compares the sum of gateway-computed cost against the provider's invoice line items. Discrepancies above a threshold trigger a pricing-table audit.

Treat the stream as the source of truth; do not let any dashboard connect to the quota store (chapter's enforcement side) or to the application's own usage metrics. One source means one definition of a token and one definition of a dollar.

## Showback vs. chargeback — a design decision, not a technology choice

Both terms refer to attributing cost; they differ in who owes.

- **Showback.** Costs are attributed to a cost center and surfaced on dashboards; the cost center does not actually get invoiced. Useful for driving behaviour without the operational overhead of internal billing.
- **Chargeback.** Costs are attributed and the cost center is debited in some internal-accounting ledger.

The gateway does not know which mode the organisation operates in; it emits the event with enough information to support either. The decision — and the ledger — lives in the FinOps layer downstream.

## Common anti-patterns

- **One `total_tokens` column.** Collapsing input/output and cached/non-cached into a single number makes every caching discussion ambiguous.
- **Provider-reported cost.** Taking the provider's reported cost at face value means the gateway cannot be the source of truth. Compute at the gateway; reconcile provider-side.
- **Session-only quotas with no tenant cap.** A user with 500 sessions can bypass per-session caps. Always cap per-tenant; per-session is an additive guard, not a substitute.
- **Mutable billing events.** Updating a past event in place destroys audit. Use compensating events.
- **Quota checks on an aggregate dashboard.** The dashboard is eventually consistent; admission must be strongly consistent against the enforcement store.
- **Pre-call estimation with a different tokenizer than the provider.** Cost dashboards silently drift; only post-call reconciliation catches it.
- **Wall-clock budgets confused with token budgets.** A wall-clock SLO guards latency; a token budget guards cost. Both are useful, neither substitutes for the other.

## Summary

- The gateway counts tokens (split cached / non-cached, plus tool-use shares), requests, and wall-clock time. It emits one billing event per completed request to an append-only stream.
- Rate limits, burst budgets, and quotas are three distinct layers with distinct semantics, distinct stores, distinct error codes. Rate limits protect short-window supply; burst budgets protect spike cost; quotas protect window-level spend.
- Admission reserves estimated cost against each layer before routing; reconciliation at request completion updates reservations against actual usage. Over-estimates refund; under-estimates surcharge.
- Session-level loop guards (cumulative cap, repetition detector) are the gateway's defence against budget-exhausting loops; they complement the runtime's own guard.
- The billing-event schema carries the split-token counts, the per-attempt list, the computed cost against a versioned price table, and the cost-center and project tags. It is the source of truth for FinOps.
- Price regression alarms, provider-invoice reconciliation, and real-time FinOps dashboards all consume the same stream.

The next chapter picks up the cache side. Split-token accounting makes the economic case for prompt caching concrete; prompt caching with cross-tenant isolation is the technical problem chapter 05 solves.
