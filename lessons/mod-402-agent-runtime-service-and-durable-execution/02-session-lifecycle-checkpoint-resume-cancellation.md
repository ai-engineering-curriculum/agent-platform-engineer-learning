# Session Lifecycle: Checkpoint, Resume, Timers, Cancellation, Context Compaction

Chapter 01 pinned the contract: the session object is resumable from its checkpoint, every state change is idempotent, and the whole thing is observable from one query. This chapter makes those properties concrete. It walks through the five mechanical things a runtime has to do with a session — snapshot it, schedule timers against it, deliver signals into it, cancel it on demand, and compact its context when it outgrows the model window — and names the traps in each.

Everything here is substrate-agnostic. The substrate chapter (chapter 03) then shows which durable-execution engines give you which pieces for free.

## The session state machine

```
                       cancel()
             ┌───────────────────────────┐
             │                           │
    create()│                           ▼
     ┌───▶ RUNNING ──signal/tool──▶ WAITING ──resume──▶ RUNNING
     │       │   ▲                     │                    │
     │       │   └──checkpoint──┐      │                    │
     │       ▼                   │     ▼                    ▼
     │   COMPLETED           SUSPENDED ◀──admit──── EXPIRED
     │                                                      ▲
     └──────────────────── expire-timer ────────────────────┘
```

Seven states is enough for most agent runtimes:

- **RUNNING** — a worker is currently executing agent code or an awaited future.
- **WAITING_TOOL** — the agent issued a tool call and is parked on its result.
- **WAITING_SIGNAL** — the agent issued an external-event wait (`waitForSignal("user_reply")`).
- **WAITING_TIMER** — the agent issued a durable sleep ("retry in 5 minutes", "escalate at 10:00").
- **SUSPENDED** — admission control took the session off the hot pool (over quota, over budget, worker shortage); its state is persisted but no worker holds it.
- **CANCELLED** — a cancel signal landed; the runtime is draining pending work.
- **COMPLETED / EXPIRED** — terminal; distinguished for observability (one is success, one is "nothing happened for N hours").

Two properties of this machine are non-negotiable:

1. **Every transition writes a durable event.** `RUNNING → WAITING_TOOL` emits "tool-call-issued"; `WAITING_SIGNAL → RUNNING` emits "signal-delivered"; and so on. The event stream *is* the audit trail (see mod-408 for the OTel-GenAI shape it should take). A transition without a durable event is an invisible state change, and invisible state changes end up as production incidents.
2. **No transition destroys prior history.** A `RUNNING → CANCELLED` move does not delete pending timers; it marks them cancelled. A `COMPLETED` session's checkpoint is retained until retention policy (30, 90, 365 days — your call, mod-411 and legal counsel) expires it. If you compact history, do it in a separate, named step that itself writes an event.

## Checkpoint semantics

A checkpoint is an atomic snapshot of everything a future worker needs to resume. Two questions decide what goes in it:

**1. What is *agent state*?**  Everything that changes across agent steps: the message history (or references to it), the scratchpad, the current plan, retrieval results the agent has already consumed, open tool-call handles, the running cost ledger. Framework-specific pieces (LangGraph's channel state, ADK's session memory, MS Agent Framework's agent runtime object) go in here too — see chapter 04.

**2. What is *not* agent state?** Configuration (prompts, tool manifests, model IDs) lives *outside* the checkpoint and is looked up on resume. If you checkpoint the system prompt, every session carries a copy of it around forever, and the day you fix a prompt typo you cannot roll out the fix without migrating every live session. Configuration is referenced by version; the referenced version is checkpointed, not the content.

### When to snapshot

Pick the smallest number of commit points that still gives you the guarantee in chapter 01's obligation 1 ("if my session committed a tool call, it will not run twice"):

- **Before every side-effecting tool call** — so a replay on worker crash does not re-issue the call.
- **After every LLM response** — because the response itself cost money and you do not want to re-ask on crash.
- **After external signal delivery** — so a crash between "signal received" and "agent consumed signal" does not lose the signal.
- **On explicit `yield`** from agent code — gives agent authors an escape hatch for long computations they want to checkpoint around.

Everything else can be derived. Snapshotting on every single tick is wasteful and makes the state substrate the bottleneck; snapshotting only on completion gives you no durability at all.

### Where to snapshot

Three storage choices, matched to size and churn rate:

| State kind | Storage | Why |
|------------|---------|-----|
| Small, hot (session metadata, cursor, cost ledger) | Postgres row with `FOR UPDATE SKIP LOCKED` | Transactional, indexable, queryable for observability. Don't over-engineer this with Redis — the primary store must survive restart with durability. |
| Medium, structured (message history, agent plan, channel state) | Postgres `jsonb` column or Postgres + object-store pointer | Keep it in Postgres as long as the row fits comfortably (rule of thumb: < 1 MB). Spill to object storage above that, write back only the pointer. |
| Large artifact (uploaded file, tool result blob, long transcript) | Object storage (S3-compatible) | Never in the session row. Store the URL and the content-hash; keep the row small. |

Chapter 05 goes into the substrate side (Postgres vs. Redis vs. object storage, when each belongs on a Kubernetes operator vs. a serverless fabric like Knative) in detail. For now: hot small state goes in Postgres, large payloads go in object storage, Redis is *not* the durable store.

## Resume semantics

Resume must be a pure function of the checkpoint plus the current configuration. Three failure modes to design against:

1. **Config drift between snapshot and resume.** The agent checkpointed against tool manifest v4. By the time it resumes, the registry is at v6, which removed a tool the agent was about to call. The runtime should refuse the resume with a typed error (`ToolManifestIncompatible`), not silently continue and surprise the agent mid-flow. Versioned manifests (mod-403) make this detectable.
2. **Non-deterministic agent code.** Agent step N made a decision using `random()` or `time.now()`. On resume, step N replays with a different value, and the plan diverges. Durable-execution substrates enforce determinism (chapter 03); outside them, the agent-authoring SDK must (mod-412) by providing seeded RNG and virtualised time.
3. **Partial side-effect replay.** The agent issued a tool call, the call succeeded, the worker died before the result was durable. On resume, the runtime must *not* re-issue the call. Two patterns handle this: (a) write the tool-call-issued event before the call, write the tool-result event after, and on resume check which of the two is the last durable event; (b) require the tool to be idempotent via a deduplication key. The runtime provides pattern (a) for free when tools are opaque; mod-403's tool-gateway should also enforce pattern (b).

## Timers

A timer is a promise the runtime makes to deliver a wake event to a session at a wall-clock time. Three qualities a production runtime has to ship:

- **Survives worker restart.** The timer is in durable storage, not a Go `time.AfterFunc` on a specific pod. If the whole runtime fleet is redeployed, the timer still fires.
- **Delivered at-least-once.** Clock skew and GC pauses happen; your delivery guarantee is "fires no earlier than T, fires at most a few seconds after T in the normal case, fires eventually under failure." The agent step the timer wakes is responsible for idempotent handling.
- **Cancellable.** A `CANCELLED` session's pending timers are swept (not just logically ignored, so the delivery pipeline does not thrash).

Patterns that work: a Postgres table of `(fire_at, session_id, timer_id)` indexed on `fire_at`, swept by a pooled worker; Temporal's built-in durable timers; Dapr Workflows' `createTimer`; Restate's awakeables backed by its journal. Do *not* build a timer wheel in Redis as the system of record unless Redis is explicitly your durability tier (which this track argues against — see chapter 05).

## Signals

A signal is an external event addressed at a specific session (`user_reply`, `approval_granted`, `agent-B_response`). The runtime accepts signals via API, persists them, and either delivers them immediately (if the session is in `WAITING_SIGNAL` on the matching name) or buffers them.

Three design calls:

- **Name-addressed, not type-addressed.** Signals have names the agent explicitly waits on (`"user_reply"`), not arbitrary payloads the agent has to parse. This keeps the state machine small — the runtime can validate that a delivered signal matches an awaited name.
- **Buffered with a bound.** If a session is not currently waiting on signal `X` but `X` arrives, buffer it (so a signal that races a resume is not lost). Bound the buffer per session; shed oldest with a warning past the bound.
- **Replay-safe.** Signal delivery writes a durable event *before* the agent consumes it. If the worker dies between durable event and consumption, resume sees the event and continues.

A2A messaging (mod-409) is layered on signals: an inter-agent reply is just a signal with a structured payload and a conventional name.

## Cancellation

Cancellation is the hardest primitive to get right. It has to be *propagating*, *bounded*, and *observable*.

- **Propagating.** When session S is cancelled, every in-flight tool call S has issued is also told to cancel. The tool gateway (mod-403) accepts cancellation tokens; the sandbox (mod-404) honours them; the model gateway (mod-406) cancels in-flight completion requests. Partial propagation means tool spend continues after the user hangs up — a direct violation of obligation 2 in chapter 01.
- **Bounded.** The runtime commits to a propagation deadline (e.g. "cancel is visible across the session within 5 seconds"). After the deadline, outstanding tool calls are considered *leaked* and reported as such. Leaks are an SLO; mod-410 picks the target.
- **Observable.** A single event per session tells the on-call engineer: cancel requested at T₀, propagated to M of N pending calls, N - M leaked. If leaked calls return results after cancellation, the results are discarded and billed to a leakage ledger, not to the session's cost ledger.

Two cancellation reasons to model distinctly:

1. **User-initiated** ("user hangs up", "user clicks cancel"). Fast, no retries, discard partial outputs.
2. **System-initiated** ("budget breached", "tenant quota exhausted", "evicted for higher-priority work"). Can be *checkpointable* — the session goes `SUSPENDED`, not `CANCELLED`, so admission control can admit it again later. Chapter 05 goes into admission.

## Context compaction

LLM context windows are bounded. Every long-running session eventually either overflows or burns tokens on stale history. Compaction is the runtime's job — not the agent's — because only the runtime can enforce consistent policy across tenants and model versions.

Four compaction policies, pick one per session class (not per request):

- **Sliding window.** Keep the last K tool-call turns verbatim. Simplest; fine for short, stateless agents.
- **Summarise-and-drop.** When the context nears the model's limit, call a summariser model on the oldest chunk and replace it with the summary. Costs tokens; preserves long-horizon semantics imperfectly. Record the summariser version in the checkpoint so resume is reproducible.
- **Memory-indexed.** Spill old turns into a memory store (pgvector, see mod-402 chapter 05, and the memory substrate in `agentic-ai-engineer` level-30 material). Retrieve relevant slices on each turn. The right choice for long-running agents with rich memory needs.
- **Hierarchical.** Combine: short-term rolling buffer, mid-term summary, long-term memory index. Needed for multi-day sessions; overkill for request-response.

Make compaction *explicit in the trace*. Every compaction event in the OTel trace (`compaction.applied` with policy, before/after token counts, summariser model if any) so product teams can audit what was dropped. Silent compaction is the fastest way to make product teams lose trust in the runtime.

## Summary

- The session state machine has seven states; every transition writes a durable event; nothing is dropped silently.
- Checkpoint only what is *agent state* (history, scratchpad, cost ledger); keep configuration outside and look it up on resume. Snapshot before side-effecting tool calls, after LLM responses, after signal delivery, and on explicit `yield`.
- Resume is a pure function of checkpoint plus configuration. Design against config drift, non-determinism, and partial side-effect replay explicitly.
- Timers are durable, at-least-once, and cancellable — not built on in-process clocks.
- Signals are name-addressed, buffered with a bound, and replay-safe. A2A messaging (mod-409) is a thin layer on top.
- Cancellation is propagating, bounded, and observable. Leaks are an SLO.
- Context compaction is the runtime's job, picked per session class, and recorded explicitly in traces.

Next chapter: which durable-execution substrate to pick to deliver all of this, and how Temporal, Dapr Workflows, Restate, and Inngest compare on the workload characteristics agent sessions actually have.
