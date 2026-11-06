# exercise-03: Resource Quota and Eviction Policy Design

**Estimated effort:** 3 hours · **Output:** one quota-profile set (three tiers), one eviction-policy document (two ladders), one SLI / SLO proposal (1 page), and a short admission-test sketch · **Pairs with:** chapter [03](../03-resource-quotas-and-eviction.md); feeds project 401 and mod-410.

## Objective

Produce the quota and eviction policy your sandbox plane will actually operate against. The deliverable is the shape that an on-call engineer, a security reviewer, and the SRE plane (mod-410) all consume: per-tier default quotas, per-tool override ceilings, tenant-level and fleet-level budgets, and the two eviction ladders (leak and fleet pressure) with their cause codes.

A quota profile written as "give each sandbox 2 GB of memory and 1 CPU" is not the deliverable. The deliverable is the thing that answers "why 2 GB, bounded how, overridable when, evicted on what cause, surfaced where" — because those are the questions the on-call engineer and the compliance audit ask.

## Prerequisites

- You have read chapter [03](../03-resource-quotas-and-eviction.md).
- You have exercise 02's capability-bundle fragment on hand; this exercise fills in the `resources` section you stubbed.
- You have skimmed the [cgroup v2 documentation](https://docs.kernel.org/admin-guide/cgroup-v2.html), specifically the `cpu`, `memory`, and `pids` controllers.
- If your primary substrate is Firecracker, you have read the [Firecracker rate-limiting API](https://github.com/firecracker-microvm/firecracker/blob/main/docs/api_requests/patch-network-interface.md) and the microVM resource documentation. For WASM, skim the [Wasmtime fuel](https://docs.wasmtime.dev/examples-fuel.html) and [epoch interruption](https://docs.wasmtime.dev/examples-interrupting-wasm.html) docs.

## Problem statement

Design the full quota and eviction policy for a sandbox fleet serving three tiers. The tiers are the ones you likely picked in exercise 01:

- **Primary tier** — bulk workload (e.g., gVisor or Firecracker with a general-purpose profile).
- **Elevated tier** — higher-risk workload (Firecracker with a hardened image, if primary was gVisor; same substrate with tighter policy, otherwise).
- **Specialised tier** — optional (WASM/Pyodide for narrow workloads, or a GPU-attached tier via managed fallback — see chapter 04). If your platform does not run a third tier, say so and skip the specialised rows.

The deliverable has four parts; every part is required.

## Requirements

### Part 1 — Per-tier default quota profiles (the main deliverable)

Author a quota profile per tier. Each profile covers all five dimensions from chapter 03: CPU (`cpu_m`, `duration_s`), memory (`memory_mb`, optional soft ceiling), disk (per declared writable mount, byte and inode caps where applicable), network (`egress_mbps`, `ingress_mbps`, `max_connections`, `egress_mb_per_call`), process state (`max_pids`, `max_open_files`, `max_threads`).

For every number in the profile, deliver:

- The value.
- The reasoning for the value — observed data from a proxy workload, a comparable platform's published default, or a well-argued starting-point assumption. "This feels right" is not reasoning; "LangChain's Playwright wrapper benchmarks at 900 MB resident across its warm set, so 1500 MB gives headroom plus 40% for fragmentation" is reasoning (and make up your own numbers honestly).
- The override ceiling — the maximum the policy plane (mod-405) is allowed to raise this value to for a per-call override. A profile without ceilings is a profile that drifts.

Include at least **one override example per tier**: a tool you would allow to raise its memory to the ceiling, naming the tool class and the rationale. This exercises whether your ceilings are workable at all.

### Part 2 — Tenant-level and fleet-level budgets

Add the two numbers per tenant (steady-state sandbox budget, hot-path admission rate) and the two numbers per fleet (headroom, hot-standby pool size per tier) from chapter 03.

For each:

- Pick a starting value. Make the reasoning explicit (projected tenant workload, expected failover-drain capacity need, etc.).
- Name the signal that tells you the value is wrong — queue depth at the gateway, pool-miss rate, over-quota evictions, fleet pressure events per hour.
- Name the owner of the number (the SRE plane consumes it; who proposes changes to it?).

### Part 3 — Eviction policy document (two ladders)

Write the two ladders from chapter 03 as executable policy:

- **Leak-eviction ladder.** From suspected leak (grace) → soft cancel → hard kill → reclamation → attribution. For each stage, name the timer, the signal that advances to the next stage, the evidence event emitted, and the cause-code catalogue.
- **Fleet-pressure ladder.** Admission shedding → idle-first eviction → over-quota-tenant eviction → never-silent substrate-drain. For each stage, name the trigger (fleet-pressure signal), the shedding rule (tenant-quota-prorated threshold), the evidence event, and the never-silent-substrate-drain policy.

Each ladder is a policy document, not code; clarity is the acceptance test. An on-call engineer at 03:00, reading just this document, must be able to tell what the plane is doing.

### Part 4 — SLI / SLO proposal and admission-test sketch

#### SLI / SLO (1 page)

Propose three SLIs for the SRE plane (mod-410) to consume:

- `sandbox_reclamation_time_p99` — from leak-detected to reclaimed.
- `sandbox_resource_cap_hit_rate` — hits per 1000 sandboxes, by dimension.
- `sandbox_eviction_cause_rate` — ladder causes per 1000 sandboxes, by cause code.

For each, write: the exact definition (including window), the proposed SLO value and your reasoning for it, the alerting policy you propose (burn-rate alerts at what windows), and what you would do with the data (tighten a quota? widen the headroom? fire a tenant-level review?).

#### Admission test sketch

A short checklist (≈10 items) that an admission pipeline runs on any new tool version that brings its own quota overrides. Examples:

- Override values are within tier ceilings.
- Override values are no looser than the previous version without an attached rationale.
- Egress-byte-per-call override is justified against the exfiltration threat model.
- `duration_s` override is bounded by the runtime's own session-length limit (mod-402).
- OOM behaviour of the proposed memory profile has been exercised (an OOM test ran and the kill was clean).
- No override removes a dimension entirely (nothing goes to `unlimited`).

## Starter guidance

- Resist the urge to pick round numbers. Round numbers signal "I did not do the work." A memory ceiling of 1024 MB is an invitation to ask "why not 896 or 1280?"; a memory ceiling of 1500 MB with the reasoning attached deflects the question.
- The override-ceiling rule is where quotas live or die. Without ceilings, tools negotiate upward until the plane is uncapped; with ceilings, the plane holds a line and a tool that genuinely needs more negotiates a tier change (which is a reviewed event).
- Tenant-level budgets are not one-size-fits-all. If you know your tenant distribution has a long tail — one or two power users, many small — build that in. "$3 per hour per tenant maximum" is a starting conversation, not a ship target.
- Pick the alerting shape before the SLO value. If you cannot write a sensible multi-window burn-rate alert for an SLI, the SLI is not actionable, and the SLO is cosmetic.
- For the admission-test sketch, every item should be an automation hook, not an exhortation. "Reviewer agrees the quota is reasonable" is not a test; "override values are within tier ceilings" is.

## Acceptance criteria

Mark done when:

- [ ] Three tier profiles are complete across all five dimensions (CPU, memory, disk, network, process state), with override ceilings, with at least one override example per tier.
- [ ] Every number in every profile has reasoning attached (not a blanket "sensible default").
- [ ] Tenant-level (×2) and fleet-level (×2) budgets are proposed with signals-of-wrongness and owners.
- [ ] Both ladders (leak, fleet-pressure) are documented stage-by-stage with timers, signals, events, and cause codes.
- [ ] Three SLIs are proposed with definitions, SLO values, burn-rate alerting, and the actions the data triggers.
- [ ] The admission-test checklist has at least 10 items and every item is automatable.

## Stretch goals

- **OOM dry-run.** Take the Part 1 memory ceilings and run a workload that deliberately approaches them under cgroup v2. Attach the kernel's OOM-kill trace to the memo and verify the sandbox plane's evidence stream (chapter 06) would record the event correctly (trace-id joinable, cause-code correct).
- **Multi-cloud translation.** If your platform runs across clouds, translate the fleet-level headroom number into each cloud's specific instance-class reservation. A fleet-level number is useless if the SRE plane has to re-derive it in three provisioning surfaces.
- **Burndown exercise.** Simulate a 5-minute 10×-traffic spike against your pool-size / headroom numbers. Does admission shedding fire where you predicted? Does the hot-standby pool drain to zero? Write the retro.
- **Comparison with a published quota.** Find a published platform's quota page (e.g., a managed-vendor cold-start page, a Kubernetes default quota) and write a one-page comparison: what are their numbers, what do yours differ from, and why? Grounds your reasoning against someone else's evidence.
