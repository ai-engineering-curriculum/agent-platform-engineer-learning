# exercise-04: Cost Attribution and Billing Events

**Estimated effort:** 2 hours · **Output:** one billing-event JSON Schema, one versioned price table, a reference event emitter that computes cost from token splits, a reconciliation harness against a simulated provider invoice, a price-regression alarm, and a short README (~1 page) · **Pairs with:** chapter [04](../04-quotas-burst-budgets-and-cost-attribution.md) (and chapter [05](../05-prompt-cache-coordination-across-tenants.md) for the split-token fields); feeds project 404.

## Objective

Build the end-to-end cost-attribution pipeline chapter 04 names: the billing-event schema that FinOps consumes, the versioned price table the gateway applies, the emitter that computes cost at the gateway (not from the provider invoice), and the two downstream consumers that make the pipeline worth maintaining — the provider-invoice reconciliation job and the price-regression alarm. The deliverable is the exact artefact that lets the platform answer "how much did acme spend on `platform:writer-v1` this week, and did we get it right".

A well-authored pipeline produces billing events whose sum, grouped by tenant and model, reconciles with the provider's invoice line items to within a small tolerance (gateway-side rounding and tokenizer approximation account for the rest). A pipeline that emits a single `total_tokens` column cannot distinguish a tenant who reduced cache usage from a provider who raised prices. A reviewer will send such a pipeline back.

## Prerequisites

- You have read chapter [04](../04-quotas-burst-budgets-and-cost-attribution.md) and chapter [05](../05-prompt-cache-coordination-across-tenants.md).
- You have exercise 03's cache-namespace composition available, or a stub that returns plausible namespaces.
- You have a stream platform available — Kafka, Redpanda, NATS JetStream, Pulsar, or a local file as an append-only JSONL. The pipeline writes to an append-only sink; which sink matters less than the append-only property.
- You have a SQL-capable sink available for reconciliation (DuckDB, SQLite, Postgres). An in-process DuckDB is a lightweight default.
- You have read [the OpenTelemetry Semantic Conventions for Generative AI](https://opentelemetry.io/docs/specs/semconv/gen-ai/) and know which `gen_ai.*` attributes correspond to the billing-event fields.

## Problem statement

Build the billing pipeline for a gateway serving three tenants (reuse the profiles from exercise 02 or exercise 03 — the pipeline must work across whichever subset you pick). Requirements apply to all tenants; the three must appear in the reconciliation and price-regression tests.

## Requirements

### Part 1 — The billing-event schema

Author `schemas/billing-event.v3.schema.json` as a JSON Schema (draft 2020-12 or the version your validator supports) encoding the chapter-04 schema. At minimum:

- `event_id` — ULID / UUID.
- `schema_version` — integer; the schema file declares `3` as its version.
- `gateway_request_id`, `trace_id`.
- `received_at`, `completed_at` — ISO 8601 UTC with `Z`.
- `tenancy.*` — `tenant_id`, `cost_center`, `project_tag`, `session_id`, `principal_subject`, `actor_spiffe_id`.
- `routing.*` — `model_id`, `resolved_provider`, `resolved_model`, `resolved_region`, `mode` (`primary` / `fallback` / `shadow` / `canary`).
- `tokens.*` — `input_cached`, `input_non_cached`, `output`, `tool_use_input_included`, `tool_use_output_included`, `input_token_source` (`provider_reported` / `gateway_estimated`), `output_token_source`.
- `cost.*` — `input_cached_usd`, `input_non_cached_usd`, `output_usd`, `cache_write_usd`, `total_usd`, `currency`, `price_table_version`, `price_table_hash`.
- `cache.*` — `namespace`, `prefix_hash`, `outcome` (`miss` / `hit_partial` / `hit_full` / `disabled`), `cache_ttl_bucket`.
- `attempts[]` — one entry per provider attempt (chapter 03), each with `provider`, `model`, `attempt_no`, `outcome`, `latency_ms`, `tokens_consumed`.
- `flags[]` — a closed vocabulary (`success`, `refused`, `fallback_applied`, `cache_hit`, `cache_miss`, `refund`, `adjustment`).

The schema includes `$id`, `$schema`, and `additionalProperties: false` at the top level and at each nested object.

Write one positive validator test (a canonical event passes) and five negative ones (missing `tokens.input_cached`, missing `cost.price_table_version`, extra top-level property, wrong-type `attempts`, negative token count).

### Part 2 — The versioned price table

Author `config/pricing/2026-11-01.yaml` following chapter 04's shape:

- `version`, `hash` (recomputed on read and verified), `effective_from`, `currency`.
- `models[]` — one row per `(provider, model, region)` triple with `input_non_cached_per_mtok`, `input_cached_read_per_mtok`, `input_cached_write_per_mtok` (nullable), `output_per_mtok`, `tool_use_billing` (`included_in_io` / `separate`).
- Cover at least the mappings used in your routing table (exercise 01).

Values must come from the provider's current pricing pages at the time of authoring. Anything you cannot verify gets a `# needs-research` comment with the specific field named.

Also author `config/pricing/2026-11-15.yaml` representing a hypothetical price increase 2 weeks later — Anthropic `claude-sonnet-4-6` input price rises 15%. This is what the regression alarm catches.

### Part 3 — The event emitter

Write an emitter (one file, 200–400 lines) that, given the gateway's internal state after a request, produces a validated billing event and writes it to the sink.

Responsibilities:

- Consume the gateway's `GatewayResponse` from exercise 01 and the cache `namespace` / `outcome` from exercise 03.
- Compute cost from the token splits × the price table rows for the resolved mapping.
- Validate the event against the JSON Schema before writing; on validation failure, write a telemetry record and refuse to emit (bad data is worse than missing data on this stream).
- Write to the append-only sink with a monotonically-increasing `event_id`.
- Emit an OpenTelemetry span conforming to the gen-ai semantic conventions carrying `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, `gen_ai.cost.total_usd`, and the custom `gen_ai.cache.namespace` attribute.

Test cases:

- A `cache.outcome: miss` event produces a cost equal to `(input_non_cached / 1_000_000) * input_non_cached_per_mtok + (output / 1_000_000) * output_per_mtok`.
- A `cache.outcome: hit_partial` event with 4712 `input_cached` and 318 `input_non_cached` produces the two-column cost correctly and sums to `total_usd`.
- An Anthropic `cache_write` event (first time the prefix is seen; Anthropic charges 125% on write) produces a visibly higher cost than a steady-state read.
- An OpenAI `cache_write` event (free write) produces no `cache_write_usd`.
- A `fallback_applied` event has `attempts.length >= 2`; the sum of per-attempt `tokens_consumed.input` matches the final `tokens.input_non_cached + tokens.input_cached` for the successful attempt (not sum across attempts).

### Part 4 — The reconciliation harness

Write a job that consumes the billing-event stream and reconciles against a simulated provider invoice.

- Load the events for a window into a SQL table (DuckDB is fine).
- Aggregate by `(resolved_provider, resolved_model, resolved_region)` and compute `sum(total_usd)`.
- Load a simulated invoice file `fixtures/invoice-2026-11.json` containing the provider's per-model monthly total.
- Compute the delta: `gateway_sum - provider_sum`, per row.
- Pass if abs(delta) / provider_sum < 2% (a defensible tolerance for tokenizer approximation and provider rounding); fail with structured output otherwise.

Seed the fixture with values close to your emitter output; the test is that the harness runs end-to-end and the numbers reconcile, not that your fixture is magic. Then intentionally over-report a token count on one event — assert the reconciliation fails on that row with the correct delta.

### Part 5 — The price-regression alarm

Implement the chapter 04 alarm.

- Rolling-window job that computes `avg(total_usd / output_tokens)` per `(tenant_id, model_id, path_class)` over the current window and the previous window of equal length.
- The windows have the same length (e.g. 24h); the cache-outcome distribution should be filtered to events with the same dominant outcome bucket (`hit_partial` with ≥50% cached tokens vs. `miss` are separate groups) — otherwise a cache behaviour change dominates the signal.
- Alert if the ratio `current / previous > 1.15` with an event count ≥ 500 per group.
- Emit the alert as a structured event: `{ "alert": "cost.per_output_token.regression", "tenant_id": "...", "model_id": "...", "ratio": 1.21, "events_in_window": 712, "likely_causes": ["price_table_version_changed", "fallback_rate_increase"] }`.

Test with:

- A seeded stream where the price table switches from `2026-11-01` to `2026-11-15` mid-window. Assert the alarm fires with `price_table_version_changed` as a likely cause.
- A seeded stream where the price is stable but the fallback rate spikes (a lot of events with `mode: fallback` on a more expensive mapping). Assert the alarm fires with `fallback_rate_increase` as a likely cause.
- A seeded stream where neither changes but the user prompt length rose organically. Assert the alarm does *not* fire (the per-output-token cost is roughly stable; the absolute cost rose but the ratio did not).

### Part 6 — The README

One page covering:

- The schema you adopted and the one or two fields you added or dropped from the chapter 04 reference (and why).
- Your price-table structure and how you handle the Anthropic cache-write surcharge.
- The reconciliation tolerance you chose and the single most likely cause of a 2% drift (common answers: tokenizer approximation, provider-side rounding, streaming-interruption estimates).
- The price-regression alarm's grouping dimensions and the two false-positive shapes your tests exercise.
- One open question you would route to `agentic-systems-architect`, `ai-infra-sre`, or the finance team. Example: "when a soft-cap tenant goes into overage, should the overage events flag differently so FinOps dashboards do not double-count them against their budget?"

## Starter guidance

- **Decide now whether cost is computed at event-emit time or at event-consume time.** Chapter 04 says emit-time, because the gateway owns the price table version at that moment. Consume-time computation re-opens the question of which price applied when — usually not what you want.
- **Keep the price table dumb YAML.** Smart-templating a price table creates one more thing to debug when numbers go wrong. Each row is self-contained.
- **Validate the event against the schema before writing.** An invalid event in the append-only stream is harder to clean than refusing to emit.
- **The reconciliation tolerance is a product decision; 1–3% is a reasonable range.** Document the choice; do not hide it behind a magic constant.
- **The price-regression alarm's signal is noisy on small tenants.** Require a floor of events in the window; otherwise a single large request dominates the ratio.
- **Treat refunds and adjustments as new events with a compensating sign and a reference back to the original.** Mutating past events in the stream is the single most common FinOps bug.
- **Keep the sink append-only.** If your stream is a file, use `O_APPEND`; if Kafka, topics are naturally append-only; if Postgres, insert, never update.
- **Flag `<!-- needs-research: ... -->` on every price value you cannot verify.** Pricing has moved multiple times across 2024–2026; your reviewer expects the flag on anything not sourced from the provider's current pricing page.

## Acceptance criteria

Mark done when:

- [ ] The schema validates one canonical positive event and rejects the five negative cases.
- [ ] The price table loads, hashes consistently, and every mapping used by the router has a row.
- [ ] The emitter computes cost from token splits × price-table rows; the `hit_partial` test produces a visibly reduced cost vs. a miss.
- [ ] A `cache_write` event on Anthropic reflects the surcharge; the same scenario on OpenAI reflects the free write.
- [ ] The reconciliation harness passes on the seeded fixtures and fails with a correct structured delta on the intentionally-inflated event.
- [ ] The price-regression alarm fires on both of the true-positive scenarios and does not fire on the organic-growth false-positive scenario.
- [ ] Every billing event in the stream validates against the schema; the test suite spot-checks 100 events for conformance.
- [ ] The README documents the schema, price-table shape, reconciliation tolerance, alarm dimensions, and one open question.

## Stretch goals

- **Shadow-mode billing.** Shadow-traffic events (chapter 02) are tagged with `mode: shadow` and attributed to a shared *shadow-cost* ledger, not to the tenant. Add tests that a 1% shadow sample does not inflate tenant totals but does show up in the shadow-cost dashboard.
- **BYOK cost attribution.** For a tenant using their own API key (`globotech` in exercise 03), the `cost.total_usd` is still computed at the gateway, but the tenant is *not* billed through the platform. Emit the event with `cost.billing_mode: byok` and a separate test that verifies BYOK events are excluded from the FinOps chargeback aggregate while still appearing in operational dashboards.
- **Streaming-interruption estimates.** Chapter 03's `stream_broken_after_partial` outcome bills whatever tokens made it over the stream; implement a conservative estimator when the provider does not report a final count and flag the `input_token_source: gateway_estimated` on the event. The reconciliation harness should treat those as a separate reconciliation group with a wider tolerance.
- **Monthly-boundary-straddling events.** A request that starts at 23:59:59 UTC on the 30th and completes at 00:00:03 UTC on the 1st — which month does it bill in? Decide (completed_at is conventional), document, and add a test that verifies behaviour at the boundary.
- **Alarm dashboard.** Publish the regression-alarm output to a dashboard sink; include trend lines for `cost / output_token` per (tenant, model) over the trailing 30 days. The dashboard is the artefact that makes the alarm useful to the operating-cadence review from chapter 07.
