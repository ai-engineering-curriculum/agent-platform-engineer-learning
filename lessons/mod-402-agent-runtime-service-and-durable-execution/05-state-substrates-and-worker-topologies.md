# State Substrates and Worker Topologies: Postgres, Redis, Object Storage, Stateless vs. Stateful Workers

Chapters 01–04 fixed the contract, the lifecycle, the durable-execution substrate, and the framework abstraction. One decision remains: where on the storage surface each class of state lives, and how the compute fleet that reads and writes that state is shaped. This chapter walks the three state substrates (Postgres with pgvector, Redis, object storage), the deployment choice between a Kubernetes operator and a serverless fabric like Knative, the stateless-vs-stateful worker trade-off, and the admission-control model the fleet enforces to keep cost under control.

Everything here directly feeds exercise 04 (cost-aware concurrency and admission control). If anything in this chapter feels theoretical, run the numbers against your own workload — the point of the exercise is to force that.

## The three state substrates

### Postgres as the system of record

Postgres is where the *durable* parts of the session object (chapter 01) live: the small hot row, the cost ledger, pending timers, and (for most installations) the pgvector memory index.

**What it is good at:**

- **Transactional writes.** The session row + cost-ledger delta + timer-table delete can be one atomic transaction. Consistency across these three is what lets you answer "how much did this session spend?" from one query.
- **Queryable observability.** Mod-408's observability plane wants to query *"all suspended sessions over budget in tenant T"*. SQL on jsonb columns handles that cleanly; a key-value store does not.
- **pgvector for memory indices.** The pgvector extension (<https://github.com/pgvector/pgvector>) gives you ANN search inside the same database as the session row, letting one transaction update the session, write a message, and index its embedding. No two-store consistency problem.
- **Operational maturity.** Every SRE team on the planet has a Postgres runbook. That matters at 03:00.

**What it is not good at:**

- **High write rate on a single hot row.** A session with 200 tool calls in 30 seconds will contend with itself if you naïvely update the same row per step. Fix with append-mostly logs plus occasional row materialization — not with "put it in Redis."
- **Very large blobs.** Even with TOAST, Postgres is not the right place for multi-megabyte artifacts. Push them to object storage; keep the pointer.
- **Unbounded history.** Session history grows forever if you let it. Partition by month or tenant; archive to object storage on retention expiry.

**Right-sizing rules of thumb for this plane:**

- One Postgres cluster per environment (dev, staging, prod), not one per product team.
- Separate the session store from the retrieval-corpus store if either one grows faster than the other.
- pgvector on the same cluster as session state is fine until the memory index becomes the bottleneck; then split into a dedicated cluster with its own capacity model.

### Redis / Valkey as the short-term / rate-limit tier

Redis (<https://redis.io/docs/>) and its fork Valkey are the right tool for state that is *small, hot, and acceptable to lose*.

Three uses in the runtime plane:

1. **Short-term memory tier that is a cache, not a source of truth.** The last N turns pinned in Redis for sub-millisecond latency, backed by Postgres for durability. A Redis outage degrades latency; it does not corrupt sessions.
2. **Rate-limit and quota counters.** Per-tenant, per-model, per-tool counters tick in Redis (or in the model gateway's own stateful fabric — mod-406). Rate-limit drift on a restart is survivable; Postgres contention on this path is not.
3. **Admission-control bookkeeping.** "How many concurrent sessions is tenant T currently running?" is cheap in Redis and expensive in Postgres. Sub-second accuracy is enough.

**Three traps to avoid:**

- **Do not promote Redis to the system of record.** Agent session state has retention, audit, and legal requirements (mod-411, mod-408). Redis's persistence modes (RDB, AOF) are weaker than Postgres's transaction log and operationally harder to run with real durability at the sizes you will hit.
- **Do not use Redis as the durable timer store.** Timer semantics in chapter 02 are a Postgres (or substrate-native) job. Redis sorted sets look like a timer wheel but are not durable enough to anchor cancellation and billing guarantees.
- **Expect to evict.** Give every key in Redis a TTL. A Redis cluster that grows unboundedly is a time bomb.

### Object storage for large artifacts

S3-compatible object storage (AWS S3, GCS, Azure Blob, MinIO <https://min.io/docs/>) is for anything bigger than "the session row comfortably holds it."

Three classes of artifact belong here:

- **Uploaded user content** (documents, images, audio) referenced by the session.
- **Tool-produced blobs** (generated images, PDFs, large scraped pages) that the agent may re-reference.
- **Long-transcript archives** of completed sessions after a retention threshold, so the hot Postgres cluster stays small.

**Design rules:**

- **Hash-addressed, not session-addressed.** Store by content hash and deduplicate; the session's row holds the hash + size + content-type. Two sessions that reference the same uploaded file should not double-store it.
- **Explicit retention.** Each artifact has a retention tier ("hot 30d, cold 90d, delete at 365d") driven by the compliance posture mod-411 and legal counsel define. The retention is encoded in object tags / lifecycle rules, not in application code.
- **Cross-region considerations.** If your session can resume in a different region, the artifact store must be accessible from both — or the resume path has to copy on demand. Decide this early; the "default copy" choice has an egress cost you can quantify.

### The three together: a worked example

For a session that uploads a 10MB file, runs ten tool calls, and ends with a response:

```
Postgres rows:
  session_row       (~5 KB — status, cost ledger, pointers)
  message log       (~20 KB — ten tool calls + replies, text and ids only)
  pgvector memory   (~N embeddings of ~6 KB each, as agents retain information)
  pending_timers    (empty by now)

Redis keys (all TTL'd):
  session:<id>:hot_turns     (last 4 turns, 3-minute TTL, refreshed on read)
  tenant:<t>:concurrency     (counter, 60-second TTL)
  tool-call:<id>:in_flight   (cancellation handle, TTL = tool deadline)

Object storage:
  blob:<sha256>   (the 10MB file)
  blob:<sha256>   (any tool-produced large artifacts)
```

Nothing is duplicated. Each substrate holds exactly the class of state it is best at. Resume reads from Postgres (for truth) and warms Redis (for latency).

## Kubernetes operator vs. serverless fabric

Each substrate has a deployment choice between a Kubernetes operator (self-managed, long-lived, stateful) and a serverless fabric like Knative (<https://knative.dev/docs/>) (on-demand, autoscaled, stateless).

### Postgres: operator by default

Run Postgres on a mature operator (CloudNativePG, Crunchy, Zalando). The reasons:

- Databases have identity, storage attachment, failover semantics, and backup / PITR operations that need a long-lived controller. Serverless gives you none of those.
- Vendor-managed (RDS, Cloud SQL, Azure Database for PostgreSQL) is also legitimate — the "operator-style" argument applies to the shape of the responsibility, not literally to which binary runs the controller.
- Agents produce bursty read/write patterns; a serverless Postgres model would thrash connection setup / teardown and pay for the churn.

### Redis: operator by default, with caveats

Same reasoning — stateful service, identity-aware, needs an operator or managed service (ElastiCache, Memorystore, Azure Cache). Serverless Redis offerings exist and can fit cache-only use cases. Verify the durability profile matches *what you are using Redis for* (cache ≠ source of truth).

### Object storage: managed service, essentially always

Object storage is where the Kubernetes-operator-vs-serverless question does not really apply — you consume an S3-API, hosted either by a cloud provider or by a stateful service (MinIO on Kubernetes with its own operator, Ceph via Rook). Agents do not care.

### The runtime workers themselves

This is the interesting choice, and it is where Knative comes in.

**Stateless HTTP / gRPC front door → serverless (Knative, Cloud Run, Fargate):**

- The ingress that accepts a new agent request, mints a session, and hands it to the durable-execution substrate is fine on a scale-to-zero fabric. Throughput is bursty, work per request is small, and no local state is held. Knative serving is a good fit.

**Durable-execution workers → stateful or sticky deployment, not scale-to-zero:**

- Temporal workers, Dapr Workflow apps, Restate handlers, Inngest workers are all long-polling against the substrate's work queue. Scale-to-zero breaks that pattern — the worker needs to be *present* for the substrate to assign work.
- Run as Kubernetes `Deployment` or `StatefulSet`, autoscaled with KEDA (<https://keda.sh/docs/>) on the queue depth / pending-sessions metric.

**Sandboxed tool executors → on-demand (ephemeral pods, Firecracker-microVM, Knative, Lambda-style):**

- mod-404 covers this plane specifically. Sandboxes are the classic "burst and dispose" workload serverless fabrics are built for.

**Memory / retrieval services → long-lived, stateful:**

- A pgvector query service, a reranker, an embedding service are all long-lived and benefit from warm caches and model weights resident in memory. Scale-to-zero is counterproductive here.

The right fleet shape is therefore *hybrid*: scale-to-zero at the edges, long-lived in the middle, ephemeral for sandboxed compute. "Everything on Knative" or "everything on operators" is a sign the trade-off has not been made.

## Stateless vs. stateful workers

Two topologies exist for the durable-execution workers; both ship in production. Pick based on your substrate and your scale.

### Stateless workers (default)

- A worker picks up a session's workflow task, replays the event history to reconstruct state, executes one step, commits the result, lets go of the session.
- Any worker can pick up any session. Load balancing is trivial.
- **Cost:** replay of long histories can be expensive on worker restart. Substrates mitigate with sticky routing / history caching, but a long-tail session with a 10,000-event history will feel it.

### Stateful workers (sticky)

- Each session is affinitised to a specific worker; the worker holds the reconstructed state in memory across steps.
- **Benefit:** no replay cost per step. Low latency between steps.
- **Cost:** worker death loses the local cache; next worker has to replay. Load balancing is harder. Rolling deploys are harder.
- **When to pick:** sessions with high step density (A2A chat, long-running research agents) where replay cost dominates. Verify with a measurement, not with vibes.

Temporal and Dapr Workflows lean stateless with history caching on hot workers. Restate's virtual objects are stateful-per-key by design. Inngest's step model is closer to stateless. The point is that *you* make the pick; blindly accepting the substrate's default is a bet, not a decision.

## Cost-aware concurrency and admission control

Obligation 3 from chapter 01 — "my session's cost cannot exceed its budget without telling me" — is where this all comes together. Four controls, in order of blast radius:

### 1. Per-session budget

- Every session is created with a token / tool / wall-clock budget. Exceeding it is a bounded event — the session is cancelled (user-initiated class in chapter 02) or paused for approval (step-up).
- Enforced at the places that spend money: model gateway (mod-406) checks before each call, tool gateway (mod-403) checks before each tool call, the runtime checks before every scheduled wake.

### 2. Per-tenant concurrency and QPS

- "Tenant T may have at most N concurrent running sessions" and "Tenant T may start at most M new sessions per minute."
- Enforced at admission time. The runtime's `admit()` function decides: if below both caps, admit; if above either, either queue (WAITING_ADMIT) or reject (if the queue is full).
- Counters live in Redis (fast, approximate) with a periodic Postgres reconciliation (durable, exact). Sub-second drift is acceptable; cross-restart drift is not.

### 3. Fleet-wide back-pressure

- When the overall fleet is at capacity (worker-pool saturation, substrate queue depth over threshold, model-gateway upstream throttling), admit *no new* sessions. Queue them or reject, by priority tier.
- Signal the back-pressure out of band (chapter on observability, mod-408) so dashboards and alerts show *why* throughput dropped.

### 4. Priority tiers

- Not all sessions are equal. Interactive user sessions outrank batch A2A sessions; paying-tenant sessions outrank free-tier. Priority is a tag on the session and a dimension of admission.
- Avoid more than three or four tiers — more becomes unmanageable. Anti-pattern: letting every product team add a tier. Pattern: platform defines tiers; product teams map onto them.

### The admission decision as code

```python
def admit(request: NewSessionRequest, now: Instant) -> AdmissionDecision:
    tenant = request.tenant
    tier = request.priority_tier

    # Budget pre-check — don't admit sessions that cannot possibly stay in budget.
    if request.estimated_cost > tenant_budget_remaining(tenant):
        return Reject(reason="tenant_budget_exhausted")

    # Concurrency cap.
    if concurrent_sessions(tenant) >= limit_for(tenant, tier):
        return Queue(reason="tenant_concurrency_cap", tier=tier)

    # Fleet back-pressure.
    if fleet_pressure(now) > pressure_threshold(tier):
        return Queue(reason="fleet_back_pressure", tier=tier)

    return Admit()
```

Four properties this function has to have:

- **Pure-ish.** It reads from counters; it does not write. The write happens after the caller commits "I am going to run this now." That separation is what makes admission auditable.
- **Fast.** Admit decisions are in the request path. Keep the counters in Redis; refresh from Postgres periodically.
- **Observable.** Every Reject / Queue decision emits a span with the reason. Mod-408's observability plane turns this into dashboards; mod-410's SRE plane turns it into SLOs.
- **Degradable.** If the admission service is down, the runtime should either fail closed (safer) or fall back to a conservative static limit (more available). Decide per-product; whichever you pick, document it.

## Where cost telemetry comes from

Three sources feed the cost ledger:

1. **Model gateway (mod-406).** Token counts in / out, price per token for the chosen model, provider surcharges.
2. **Tool gateway (mod-403).** Tool invocation count × per-invocation price (where priced), plus surcharges (storage, egress, compute time on sandboxed tools).
3. **Sandbox plane (mod-404).** Per-second compute cost for sandboxed executions attributed to the triggering session.

All three stream events into the cost ledger with a stable `session_id`; the ledger reconciles against provider billing (mod-408's billing pipeline) nightly.

## Summary

- Postgres is the system of record (session rows, cost ledger, pending timers, pgvector memory). Redis is the cache / counter tier. Object storage is for large blobs. Each substrate is used for what it is best at — no cross-use.
- Postgres and Redis belong on operators or managed services (stateful, long-lived). Agent runtime workers are hybrid — scale-to-zero at the edges (Knative), long-lived durable-execution workers in the middle, ephemeral sandboxes (mod-404) on-demand.
- Stateless vs. stateful workers is a measured trade-off: default stateless, pick sticky when step density makes replay cost dominate, and verify with a benchmark.
- Admission control has four layers: per-session budget, per-tenant concurrency / QPS, fleet-wide back-pressure, priority tiers. The decision is pure, fast, observable, and degradable.
- Cost telemetry is a platform concern that feeds the per-session ledger from model gateway, tool gateway, and sandbox plane. The ledger is live, not nightly.

This module ends here on the runtime plane. The next module (mod-403) picks up the tool-call side — MCP registry and the tool gateway the session calls into. The admission control hooks in this chapter feed directly into that plane's quota and policy enforcement.
