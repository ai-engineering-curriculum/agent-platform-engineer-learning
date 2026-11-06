# exercise-02: Per-Tenant Quota and Burst Budget

**Estimated effort:** 2 hours · **Output:** one quota schema, one burst-budget configuration, a reference admission service exercising both against a store, a test suite covering race conditions and estimate reconciliation, and a short README (~1 page) · **Pairs with:** chapter [04](../04-quotas-burst-budgets-and-cost-attribution.md); feeds exercises 03 and 04, and projects 401 / 404.

## Objective

Build the admission layer the gateway consults before routing — specifically, the per-tenant monthly token quota and the per-tenant burst budget, with pre-call estimation, reservation, post-call reconciliation, and the three structured deny reason codes chapter 04 defines. The deliverable is the exact mechanism that stops a prompt-injection-driven loop from burning a tenant's monthly budget in six minutes.

A well-authored admission layer makes concurrent requests from the same tenant race safely, reconciles estimates against actuals on completion, emits reservation events that mod-408 (observability) can follow, and refuses with the right reason code when a quota is actually exhausted — without refusing the request before it when load happens to line up adversarially. If two concurrent requests both succeed against the last 1k tokens of a 50M quota, a reviewer will send it back.

## Prerequisites

- You have read chapter [04](../04-quotas-burst-budgets-and-cost-attribution.md) and understand the split-token model from chapter 04's billing-event schema.
- You have Redis (or Memcached, or a Postgres instance, or a cloud-native strongly-consistent counter you have access to) running locally. Redis is the recommended choice for its Lua-scripting support; `fakeredis` / `miniredis` are acceptable for tests.
- You have a tokenizer available: `tiktoken` for OpenAI, Anthropic's token-counting API or `@anthropic-ai/tokenizer`, Google's `countTokens`. Pre-call estimation uses one of them.
- You have read exercise 01 so you know the shape of the `GatewayRequest` the admission layer consumes.

## Problem statement

Pick one of the following tenant-profile pairs; the shape of your admission layer follows. The exercise covers monthly quota + burst budget only — rate-limiting (short-window request protection) is a separate layer (chapter 04) and is out of scope.

- **`acme` (medium-size paying tenant).** Monthly quota: 50M tokens on `platform:writer-v1`, 20M tokens on `platform:critic-v1`. Burst budget: 500k tokens per 5-minute window, interactive-class. Cost center: `acme/eng/research`. A single user generates up to 200 sessions per month; each session may run a 20-call tool loop.
- **`sigma-labs` (research tenant with a strict per-project cap).** Monthly quota: 10M tokens total across all identifiers (hard cap). Burst budget: 100k tokens per 1-minute window. Cost center: `sigma-labs/experimentation`. Project-level sub-quotas carve the 10M into: `rag-eval` (4M), `agent-sim` (3M), `other` (3M).
- **`globotech` (enterprise tenant with BYOK).** Monthly quota: 500M tokens, soft cap (overages allowed but trigger PagerDuty). Burst budget: 2M tokens per 5-minute window, batch-class. Cost center: `globotech/ai-platform`. Uses their own API key for OpenAI (`key_ref: tenant://globotech/providers/openai/byok`) and the platform's pooled Anthropic key for Anthropic.

## Requirements

### Part 1 — The quota schema

Author `schemas/quota.yaml` (or JSON, or a database DDL — whichever shape matches your store) describing per-tenant quota state. At minimum each row carries:

- `tenant_id`, `model_identifier`, `window_type` (daily / monthly), `window_key` (e.g. `2026-11`), `tokens_budget`, `tokens_reserved`, `tokens_consumed`, `usd_budget`, `usd_reserved`, `usd_consumed`, `updated_at`, `version` (optimistic-lock counter).
- A derived `tokens_remaining = tokens_budget - tokens_reserved` that every read sees consistently.
- A flag `cap_mode: hard | soft`. On `hard`, admission denies when `tokens_remaining < tokens_est`. On `soft`, admission allows but emits a `quota.overage_warning` event.

Document the state transitions on paper or in the README: `reserve → commit (actual ≤ est)`, `reserve → commit (actual > est)`, `reserve → release (request cancelled)`, `reserve → timeout (gateway crash during request)`.

### Part 2 — The burst-budget configuration

Author `config/burst-budgets.yaml` encoding the token-bucket parameters chapter 04 names, per tenant, per `path_class`:

```yaml
burst_budgets:
  - tenant_id: acme
    path_class: interactive
    capacity_tokens: 500000
    refill_tokens_per_second: 1667   # 500000 tokens / 300s window
    overflow_action: deny
    overflow_retry_after_ceiling_s: 60
  - tenant_id: acme
    path_class: batch
    capacity_tokens: 5000000
    refill_tokens_per_second: 1000
    overflow_action: deny
    overflow_retry_after_ceiling_s: 300
```

### Part 3 — The admission service

Write a reference admission service (~200–400 lines in your preferred language) exposing a single function / endpoint:

```
admit(request: GatewayRequest, estimate: CostEstimate) -> AdmissionResult
```

The service implements, in order:

1. **Pre-call estimation.** A helper `estimate(request)` returning `(tokens_input_est, tokens_output_est_max, cost_usd_est)` using the provider's tokenizer for input and `request.max_output_tokens` (bounded by the capability declaration) for output. Over-estimate by 10% as the admission padding; record the raw and padded values.
2. **Burst-budget check.** Token-bucket semantics; denies with `burst_budget.exceeded` and a computed `retry_after_s`.
3. **Monthly-quota reservation.** A single strongly-consistent operation that (a) reads `tokens_reserved`, (b) checks `tokens_budget - tokens_reserved >= tokens_est`, (c) atomically `tokens_reserved += tokens_est` and returns a `reservation_id`. Lua script on Redis or a conditional `UPDATE ... WHERE version = X` on Postgres is the shape.
4. **Response.** On deny, a structured `AdmissionResult(outcome=deny, reason_code=..., retry_after_s=..., reservation_id=null)`. On allow, `AdmissionResult(outcome=allow, reservation_id=..., ...)`.

Also implement:

- `reconcile(reservation_id, actual_usage)` — called by the gateway after the provider response is parsed. Replaces the reservation with the actuals: `tokens_reserved -= tokens_est`, `tokens_consumed += tokens_actual`, same for USD. For a `soft` cap with overage, emits a `quota.overage_warning` event.
- `release(reservation_id, reason)` — called when a request was refused downstream (chapter 03 fallback walk exhausted, chapter 06 refusal). Releases the reservation without consuming.
- `expire_stale_reservations()` — a background sweeper that releases reservations older than `reservation_ttl_s` (recommend 60–120s; longer than any reasonable request duration).

### Part 4 — The test suite

Write tests covering at least:

- **Happy path.** Admit a request with estimate 10k, reconcile with actual 9.5k, assert `tokens_consumed = 9500` and `tokens_reserved` back to 0.
- **Overage under estimate.** Admit a request with estimate 10k, reconcile with actual 12k, assert `tokens_consumed = 12000` and the surplus accounted for.
- **Concurrent race.** Two concurrent `admit()` calls from the same tenant, estimates that together exactly consume the remainder of the quota. Exactly one admits, exactly one denies with `quota.exceeded`. Repeat under 50 concurrent calls; assert the final `tokens_reserved` is bounded by `tokens_budget`. (Use goroutines / threads / asyncio tasks against the real store; a mock will not catch the race.)
- **Burst denial.** Fill the token bucket; the next request denies with `burst_budget.exceeded` and a `retry_after_s` that matches the refill rate (± 1s).
- **Cancelled request releases.** Admit → release → admit again; the second admit succeeds.
- **Stale reservation sweeper.** Admit, simulate a gateway crash (no reconcile / release), wait `reservation_ttl_s + 1`, run the sweeper, verify the reservation is released and `tokens_reserved` returns to baseline.
- **Soft-cap overage.** Configure `acme` with `cap_mode: soft`; exceed the quota; assert admission still allows and an overage event fires.
- **Reason-code vocabulary.** Every deny reason the service emits has at least one test asserting the specific `reason_code` and `retry_after_s` shape.

### Part 5 — The README

One page covering:

- Which tenant profile you authored for, and why its quota shape stresses the implementation (e.g. the per-project sub-quotas for `sigma-labs` force a hierarchical accounting model).
- The store you chose (Redis, Postgres, cloud-native) and the single most interesting tradeoff you hit (atomic script vs. transaction, TTL semantics, replication).
- The state-transition table for a reservation, with the ingress events (admit, reconcile, release, expire) and the resulting row state.
- The reason-code vocabulary your admission service emits.
- One open question you would route to `ai-infra-sre` (your future mod-410 peer) or `ai-risk-engineer`. Example: "soft-cap tenants currently emit a warning but still consume; should a repeat-soft-cap breach auto-promote to a hard cap?"

## Starter guidance

- **Make the reservation atomic or do not ship it.** `GET-then-SET` from your application loses the race under any contention. If your store is Redis, write a Lua script. If Postgres, use `UPDATE ... WHERE version = X RETURNING ...`. The race test will catch a non-atomic implementation within 50 iterations.
- **Keep the estimator naive but honest.** Character-count heuristics are fine for the admission check; the chapter 04 point is that the *actual* numbers come from the provider. Over-estimating by 10% is cheaper than under-estimating.
- **Treat `release` and `reconcile` as the same code path with different inputs.** Both close the reservation; reconcile also writes consumption numbers. Separating them into unrelated branches leads to silent drift.
- **The burst-budget bucket and the monthly quota are independent.** A tenant with ample monthly quota can still be burst-denied; a tenant with ample burst can still be quota-denied. Test each combination.
- **The sweeper is the backup for your own bugs.** Any reservation you forget to close should eventually be swept. Set `reservation_ttl_s` well above your p99 request latency plus the retry policy's `max_wall_clock_ms`.
- **Use a monotonic clock for TTL comparisons, not wall-clock.** NTP adjustments can corrupt elapsed-time math. Your language's monotonic clock primitive is the correct choice.
- **Instrument from day one.** The admission service should emit a span per call (allow or deny) and a counter per deny reason code. Chapter 04's price-regression alarm consumes a sibling event stream — keep the shapes compatible.

## Acceptance criteria

Mark done when:

- [ ] The admission service exposes `admit`, `reconcile`, `release`, and `expire_stale_reservations`.
- [ ] A reservation under the race test (50+ concurrent admits) never over-admits past `tokens_budget`.
- [ ] Reconcile correctly accounts for over-estimates (releases surplus) and under-estimates (surcharges).
- [ ] Burst budgets honour the token-bucket semantics; the reported `retry_after_s` is within ±1s of the refill time.
- [ ] Soft-cap tenants emit an overage warning rather than denying; hard-cap tenants deny.
- [ ] Every deny reason emitted has a test asserting both the `reason_code` and the appropriate `retry_after_s` shape.
- [ ] The sweeper reclaims a crashed reservation after the TTL elapses; the quota returns to the pre-reservation state.
- [ ] The README documents the state-transition table and one open question for `ai-infra-sre` or `ai-risk-engineer`.

## Stretch goals

- **Hierarchical quotas.** For the `sigma-labs` profile, implement sub-quotas per project. A request names the project; admission reserves both at the project level and at the tenant level; both reservations commit on reconcile or roll back together on release.
- **Cost (USD) accounting in addition to tokens.** Drive the admission by estimated USD against a tenant USD budget; reconcile against the gateway's computed cost (chapter 04). A tenant's USD budget tightens before their token budget on a model with high per-token cost.
- **Quota rollover on window boundary.** A background job that resets `tokens_reserved` and `tokens_consumed` at the window boundary (daily at 00:00 UTC, monthly on the 1st). Ensure in-flight reservations straddling the boundary are handled.
- **Showback event emission.** Every reservation / reconcile emits an event to a mock chargeback stream; verify the sum per tenant per window matches the expected spend.
- **Fair-queueing deny alternative.** Instead of denying a burst-exceeded request, enqueue it with a bounded wait up to the `retry_after_s` — if the bucket refills in time, admit; otherwise deny. Document the correctness-vs-complexity tradeoff.
- **Admin-override endpoint.** A privileged endpoint that raises a tenant's monthly quota mid-window; verify the ledger reflects the change and new admits respect the new ceiling without racing in-flight reservations.
