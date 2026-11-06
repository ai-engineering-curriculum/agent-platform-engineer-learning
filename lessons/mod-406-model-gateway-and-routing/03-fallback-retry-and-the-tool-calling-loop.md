# Fallback, Retry, and the Tool-Calling Loop

A chat-completion proxy retries. That is almost the whole retry story for a classical request/response HTTP service. A model gateway for agents cannot stop there, because the requests are not independent — they are turns in a tool-calling loop where retry safety depends on whether a tool side-effect has already executed, on whether the model has already observed a result, and on whether a safety refusal is a transient capacity signal or a durable content decision. The retry policy that is correct for a chat proxy is unsafe for an agent loop. This chapter picks the one that is not.

Chapter 02 gave the forward path. This chapter picks the error-path pieces: the failure taxonomy, retry classification, circuit-breaking, fallback to a different model, and the loop-specific invariants that keep tool side-effects from double-firing.

## The tool-calling loop in two frames

Fix two pictures before writing policy. They look the same and behave differently.

- **Single-shot generation.** The runtime sends a prompt, the gateway calls the model, a response comes back, the runtime renders it. If the call fails, retrying the exact same request is safe; the only side-effect was consuming tokens.
- **Tool-calling loop.** The runtime sends a prompt. The model returns a `tool_use` block. The runtime dispatches the tool (through mod-403), gets a result, appends it to the conversation, and asks the model to continue. That second model call now depends on the first having emitted the tool_use, and on the runtime having actually run the tool. If the gateway retries that second call, is the model going to see the tool_result in the input? Yes — the tool_result is already in the appended conversation. Is the model going to re-emit a tool_use? Possibly — but the tool it asks for next time might be the same tool with the same arguments. The runtime's job is to deduplicate; the gateway's job is to make sure that when it retries a call the model did not complete, it does not somehow cause the runtime to see two different tool_use blocks that look different because they were sampled at different temperatures.

Three invariants fall out immediately:

1. **Idempotency at the gateway seam is on the *request*, not on the *response*.** A single gateway request may be attempted multiple times; the runtime sees exactly one response. The gateway handles the "collapse multiple attempts into one response" part.
2. **The gateway never calls the tool.** It returns the tool_use; the runtime (mod-402) dispatches it. If the gateway thinks it owns the tool-call, retry semantics for tool loops become impossible to reason about.
3. **The idempotency key is on the gateway request, not on the provider request.** The gateway may issue the same logical request to Provider A and then to Provider B; both attempts share one idempotency key from the runtime's perspective.

## The failure taxonomy

Providers return a wide variety of errors. The gateway normalises them into a closed vocabulary before the retry policy branches. A representative mapping, drawn from current provider error surfaces:

| Normalised code | Semantics | Example provider signals |
|---|---|---|
| `transient.network` | TCP/TLS failure, connection reset, read timeout before response started | ECONNRESET, SSL handshake failed, dial timeout |
| `transient.overloaded` | Provider capacity exhausted, retry with backoff | Anthropic `overloaded_error` (HTTP 529 / 503), OpenAI 503 server_error, Google 503 UNAVAILABLE |
| `transient.rate_limited` | Hit provider rate limit, retry after `Retry-After` | OpenAI 429 rate_limit_exceeded, Anthropic 429 rate_limit_error, Google 429 RESOURCE_EXHAUSTED |
| `transient.timeout` | Request started, provider did not complete within budget | Read timeout after request, upstream deadline exceeded |
| `terminal.client_error` | The request itself is malformed | 400 invalid_request_error, 404 model_not_found, 413 request_too_large |
| `terminal.auth` | Credential failure — do not retry with the same key | 401 invalid_api_key, 403 forbidden |
| `terminal.content_filter` | Provider's safety system refused | OpenAI `content_filter` finish_reason, Anthropic response with `stop_reason: end_turn` after content filter (varies), Google `SAFETY` finish reason |
| `terminal.context_length` | Request exceeded provider's context window | 400 context_length_exceeded, Anthropic `invalid_request_error` with context-length text |
| `terminal.output_malformed` | Response parse failed (provider returned incomplete JSON, invalid tool_call arguments) | Downstream parser error |
| `success` | Provider returned a parseable response | 200 with validated body |

Two notes:

- **`terminal.content_filter` is terminal at the gateway but not necessarily at the runtime.** A safety-driven refusal should fall over to a different provider only if the platform has decided that the refusal is spurious (chapter 06). The default is *do not route around content filters*; the runtime surfaces the refusal to the agent.
- **`terminal.output_malformed` can be a provider-side bug or an agent-side schema mismatch.** Chapter 06 handles schema enforcement; if the schema fails on the gateway's own validator, the gateway may retry with a reinforced instruction once before failing over.

## The retry policy

A retry policy is a per-request-plus-per-attempt decision: given the normalised code, the attempt number, the elapsed time, and the model identifier's declared retry policy, do we retry on the same backend, fail over to the next, or give up? A representative policy declaration:

```yaml
# retry-policies/tool-loop-strict
retry_policy:
  id: tool-loop-strict
  per_backend:
    transient.network:
      max_attempts: 2
      backoff: exponential_jitter(initial=200ms, cap=2s)
    transient.overloaded:
      max_attempts: 1          # fail over quickly; do not hammer an overloaded provider
      backoff: honour_retry_after_or(1s)
    transient.rate_limited:
      max_attempts: 1
      backoff: honour_retry_after_or(1s)
    transient.timeout:
      max_attempts: 1
      backoff: 0ms
    terminal.auth:
      max_attempts: 0
    terminal.client_error:
      max_attempts: 0
    terminal.content_filter:
      max_attempts: 0
      fallback_eligible: false   # do not route around safety refusals
    terminal.context_length:
      max_attempts: 0
      fallback_eligible: conditional   # if a fallback has a larger context window
    terminal.output_malformed:
      max_attempts: 1
      reinforce_instruction: true
  overall:
    max_attempts_total: 4       # across all backends
    max_wall_clock_ms: 20000    # budget for the full fallback walk
    stop_on_partial_stream: true
```

Four design points inside this policy:

1. **Max-attempts per code is separate from max-attempts total.** A pathological scenario (two providers overloaded, a third timing out, a fourth returning auth errors) can quickly rack up ten attempts; the `max_attempts_total` cap terminates. Without it a single degraded platform-wide incident produces a thundering herd.
2. **`stop_on_partial_stream` is non-negotiable in streaming mode.** If the provider has already sent any tokens over the stream, the gateway must not retry — the agent has already observed partial output. Collapse to failure or fail over to a different backend with the understanding that duplicate generation is now possible if the streaming started. Most mature gateways disable fallback entirely once the first streamed chunk has been emitted.
3. **`honour_retry_after_or` respects the provider's `Retry-After` header and falls back to a fixed value if absent.** Anthropic's 529 and OpenAI's 429 both carry the hint; respecting it is table stakes.
4. **`reinforce_instruction` for `terminal.output_malformed` is a one-shot retry with a rewritten system message that says, essentially, "your last response failed schema validation; emit JSON only". It is not a loop; it is a single rescue. Chapter 06 details the enforcement side.

## Fallback across providers

Chapter 02 defined the candidate list. The retry policy decides when to walk off a candidate; this section says what "walk off" means.

### Health-based reordering

Each backend has a running health signal — a circuit-breaker state machine on top of the attempt log.

- **Closed.** Recent attempts have mostly succeeded. Normal routing.
- **Open.** Recent attempts have failed above a threshold. Routing *skips* this backend for a cooldown window.
- **Half-open.** Cooldown has elapsed. The next request is allowed through as a probe; success closes the circuit, failure reopens it with a longer cooldown.

Common tuning:

- **Failure window and threshold on a rolling second-level clock,** not request count. A low-traffic model identifier is harmed by "50 consecutive failures" thresholds; "50% failure rate over the last 30s with at least 10 requests" is the shape most gateways adopt.
- **Separate circuits per `(provider, model, region)` triple,** not per provider. A single provider's `us-east-1` can be degraded without `eu-west-1` being affected; collapsing the circuit at provider level over-reacts.
- **Shadow probes on open circuits.** Even while a circuit is open to production traffic, a small shadow sample continues to probe so the half-open transition is data-driven, not clock-driven alone.

### Fallback eligibility

A failure that returns `fallback_eligible: true` moves to the next candidate. A failure with `fallback_eligible: false` surfaces as the gateway's response.

Two edge cases that burn new gateways:

- **A provider whose context window is smaller than the current request.** `terminal.context_length` is `fallback_eligible: conditional` — the next candidate is eligible *if* its declared context window fits the request. The gateway cannot just try and hope; it reads the capability declaration from chapter 02. A failed capability-eligibility check here is a routing failure with `reason_code: fallback.context_window_insufficient`.
- **A provider whose structured-output semantics differ from the primary.** If the primary supports native JSON mode and the fallback does not, the gateway either applies its own validator (chapter 06) or refuses the fallback. The declaration in the routing table says which; the request's capability requirements and the mapping's capability deviations drive the decision.

## The tool-loop invariants

This is the section that separates a model gateway from an LLM proxy. Four invariants, each with a concrete failure mode that motivates it.

### Invariant 1 — The request is the unit of retry

The *gateway request* — a specific conversation state, a specific sampling config, a specific idempotency key — is the unit. One request, zero or more attempts, exactly one response to the runtime. The runtime does not know how many attempts happened; the attempts list on the envelope (chapter 01) records them for audit.

Failure mode if violated: the runtime retries at its layer *while the gateway is also retrying*, producing a cross-stripe of duplicate attempts to the provider. This surfaces as weird per-session billing spikes and non-deterministic partial streams.

Mechanism: the runtime's call is an HTTP request with an idempotency key the gateway honours. If the runtime retries the same key, the gateway returns the last recorded outcome rather than issuing a new provider call.

### Invariant 2 — Tool-use blocks are not re-sampled across attempts

When a retry successfully reaches a provider that returns a `tool_use` block, the gateway must *not* return a different `tool_use` than the previous attempt returned — unless the previous attempt did not reach the point of emitting one. A provider's sampling at temperature >0 is non-deterministic; the same prompt can emit a different tool call.

Failure mode if violated: the runtime executes tool A on attempt 1 (whose response was later discarded on a stream break), then on the retry sees tool B, and the agent proceeds with inconsistent state.

Mechanism: the gateway's retry collapses partial outcomes before emitting a successful response. A partial stream that emitted a tool_use is not retried (`stop_on_partial_stream: true`); it is surfaced as a terminal outcome and the runtime handles it.

### Invariant 3 — Fallback-across-providers does not change the tool schema

The tools array (chapter 02) is translated per-provider. The tool *schemas* are identical across providers; the wire shape differs. A fallback to a different provider does not change what tool the model may call or what argument shape it must emit.

Failure mode if violated: the model sees different tool signatures on retry, which the runtime dispatches differently, which corrupts the conversation.

Mechanism: the adapter is lossless on tool schemas; the capability matrix refuses a fallback that cannot represent the primary's tool-call shape.

### Invariant 4 — Streaming latency is observable, but the response is still atomic

Streaming responses stream, but the gateway still produces exactly one normalised response per request. Mid-stream events are observable to the runtime; the final "response is complete" event is the atomic commit point for billing and for the attempts list.

Failure mode if violated: the stream commits token counts at every chunk; a retry that discards the stream has already billed. Two attempts on the same request produce two billing events.

Mechanism: the streaming envelope commits exactly once, at the end of the final successful attempt. Token counts accumulate on each attempt and the attempts list records them, but the per-request total is the final successful attempt's counts.

## Hedging

A legitimate technique at specific latency SLO points, a trap at the general case. Hedging means: issue the request to the primary, and after `hedge_delay_ms` elapses without a complete response, issue it in parallel to the fallback. The first response wins; the other is cancelled.

- **When to use.** Latency-sensitive interactive paths (user-visible UI responses) on a model identifier where the fallback is an acceptable substitute at the p99. The gateway must be able to cancel the losing attempt cleanly and must bill only the winner.
- **When not to use.** Tool-calling loops in general, unless the hedge pair is to the same provider with different keys (a very narrow case). Hedging to a different provider means two different sampled tool_use responses race; the loser is cancelled but not before it may have begun emitting a different tool call. The invariants above fail.
- **When it is subtly wrong.** Hedging when either provider's response triggers side-effects downstream (e.g. a provider that logs prompts to training data under a legal-basis distinction from the primary). You cancelled the request on your side; the provider already handled it.

Most mature gateways offer hedging as a per-model-identifier capability that defaults to off. Turn it on only after a latency measurement proves the need and after confirming the invariants above hold.

## Streaming, partial output, and the retry decision

Streaming interacts with retry in subtle ways worth naming.

- **Before the first chunk**, the attempt has produced no agent-visible output. Retry is safe.
- **After the first chunk**, the agent may have rendered partial output. Retry is not safe by default: the retry would restart the stream from scratch, producing double output. Options are (a) fail the request (the agent sees the partial response and a failure trailer; the runtime decides whether to resubmit), (b) fall back only in a non-streaming shape (stop streaming, assemble, retry, return the full non-streamed body), (c) accept the duplication risk and mark it explicitly. Most gateways pick (a).

The gateway's streaming adapter explicitly marks the moment the first chunk goes out. All retry decisions after that moment check the flag.

## Observability on the error path

Three metrics are the minimum on the error path. Mod-408 goes deeper; here they are the SLIs the gateway owns.

- **`gateway_attempts_per_request`.** A histogram. Alert when p99 rises; it is a leading indicator of provider trouble.
- **`gateway_fallback_rate_by_pair`.** The fraction of requests on model identifier X served by fallback N rather than the primary. Alert when it jumps.
- **`gateway_circuit_state_change`.** Every circuit transition, with reason. Dashboards look at this before anything else during an incident.

Every attempt, successful or not, writes a span with `gen_ai.request.model`, `gen_ai.response.model`, `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, provider-native fields namespaced under `gen_ai.provider.*`, and the normalised outcome. The OpenTelemetry Semantic Conventions for Generative AI (mod-408, cross-ref) define the names; the gateway follows them.

## A worked example

Pull it together. A representative fault in production, walked through the policy.

A tenant's agent is in a tool-calling loop. The gateway serves `platform:writer-v1`, primary Anthropic `us-east-1`. On step 7 of the loop:

1. Attempt 1 to Anthropic `us-east-1` with stream=true. The stream begins; 12 tokens arrive; the TCP connection drops with ECONNRESET.
2. Normalised code: `transient.network`. Policy: `max_attempts: 2`. But `stop_on_partial_stream: true` is set and the first chunk already emitted. Streaming retry is not allowed.
3. Policy decides: do not retry. Surface a terminal outcome.
4. Attempts list: 1 entry, `outcome: stream_broken_after_partial`. Response envelope: `response.finish_reason: other` with `provider_native_finish_reason: stream_broken_after_partial`. Token counts: the 12 tokens are billed because the stream committed them.
5. Runtime decides, by its own policy, whether to re-submit. If it does, it is a *new* gateway request with a *new* idempotency key, and the loop proceeds. The duplicate possibility (the first stream may have emitted a tool_use the agent began acting on) is a runtime-side concern — the gateway's invariants kept the tool_use from being resampled.

Zoom out one level and consider a non-streaming failure in the same flow:

1. Attempt 1 to Anthropic `us-east-1`, non-streaming. Response is `529 overloaded_error` with `Retry-After: 2`.
2. Normalised code: `transient.overloaded`. Policy: `max_attempts: 1`. Backoff: honour `Retry-After` = 2s. But the policy's `max_attempts: 1` means "attempt once, then fall over". The gateway does not retry on the same backend; it moves to the next candidate.
3. Attempt 2 to Anthropic `eu-west-1`. The tenancy allow-list permits `eu-west-1`. Response: 200 success.
4. Attempts list: 2 entries. Response envelope: success. `routing.fallback_reason_code: provider.overloaded`.

The agent code does not change on either path; the runtime observes either the normal success or the terminal outcome; the gateway owns the retry + fallback decision tree.

## Common anti-patterns on the error path

- **Retrying on `terminal.content_filter`.** Routing around safety refusals is a policy you should make with your safety partner, not a default. See chapter 06.
- **Infinite in-gateway retry loops.** Without `max_attempts_total` and `max_wall_clock_ms`, a provider that returns `429` forever with a `Retry-After: 60` burns one gateway worker per stuck request until the worker pool exhausts.
- **Shared idempotency keys across the fleet.** The runtime generates the key; the gateway respects it; neither reuses it for unrelated requests.
- **Silent tool-schema widening on fallback.** A fallback provider that accepts a slightly-wider argument schema (e.g. an array where the primary accepts only a single string) will produce `tool_use` blocks the runtime cannot dispatch. The adapter refuses to widen.
- **Health probes on the hot path.** A health check that sits on the hot path (request comes in, gateway pings provider, then routes) adds a per-request round trip for no benefit. Health signals come from the attempt log, not from synthetic probes on the critical path.

## Summary

- Fallback and retry for a model gateway are a tool-loop problem, not a classical RPC retry problem. The gateway's request is the unit of retry; the response is atomic; tool_use blocks are not re-sampled; tool schemas do not change under fallback; streaming commit happens once.
- Normalise every provider error into a closed vocabulary before the retry policy branches. The policy declares per-code `max_attempts`, per-code `fallback_eligible`, an overall `max_attempts_total`, a wall-clock budget, and the stream-safety invariant.
- Circuit-breakers per `(provider, model, region)` with a rolling-window failure rate and a half-open probe are the health signal. Probes may be shadow-sampled to avoid starving an open circuit.
- Hedging is a per-identifier capability that defaults off. Use only for latency SLOs, never for tool loops, and never across providers whose side-effects differ.
- Streaming interacts with retry — after the first chunk, retry is not safe; most gateways fail the request rather than re-stream.
- Instrument attempts-per-request, fallback-rate-by-pair, and circuit-state-change as the three error-path SLIs. Every attempt is a span with the OpenTelemetry gen_ai fields.

The next chapter picks up quotas, burst budgets, and cost attribution — the tenant-facing surface that decides whether a request is even allowed to enter the retry policy above in the first place.
