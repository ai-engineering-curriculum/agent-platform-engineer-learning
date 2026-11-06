# Resource Quotas and Eviction of Leaked Sandboxes

The capability model in chapter 02 covered the *shape* of what a sandbox is allowed to do. This chapter covers the *amount* — how much CPU, memory, disk, bandwidth, and process state a sandbox may consume — and what the plane does when a sandbox is still alive but nobody is using it any more. Quotas are how you keep a buggy tool from toppling the host; eviction is how you keep a leaked sandbox from quietly burning capacity until an on-call engineer notices your spend graph bent upward.

## Why quotas are a policy artefact, not a config knob

In a traditional web-service platform, resource limits live in a Kubernetes deployment YAML and barely move from quarter to quarter. In an agent platform the picture is different because:

- **Workload is model-generated.** A tool call executing a Python snippet the model wrote can turn into a 10-second `re.findall` on a 5 GB temp file; the tool itself is innocent, the input is adversarial by the threat model, and the sandbox has to survive it without taking the host with it.
- **Per-call shape varies by tool.** A calendar lookup needs tens of MB and a second of CPU; a code-interpreter running a numpy workload needs a few hundred MB and tens of seconds. A single quota profile across the fleet is wrong in both directions — too loose for the calendar case (letting a buggy calendar tool consume code-interpreter budget) and too tight for the code-interpreter case (failing legitimate work).
- **Tenant isolation is a load-bearing property, not a courtesy.** One tenant cannot starve another. Quotas enforce it on the sandbox plane the same way the model gateway (mod-406) does for inference.

Treat every quota as part of the tool's registry entry, pinned per version, reviewed on change. The sandbox plane reads it off the registry at admission and enforces it via the host's cgroups, the VMM, or the WASM runtime's fuel/limit APIs.

## The five quota dimensions

Five dimensions cover the realistic failure modes. Every call carries a value for all five; omitting one means "take the tool's default" (also in the registry), never "no limit."

### CPU

- **Shape:** a milli-CPU budget (`cpu_m`) that acts as a weight and ceiling, plus a wall-clock duration ceiling (`duration_s`). `cpu_m` throttles the busy-looping case; `duration_s` catches the "I did nothing for 20 minutes" case.
- **Enforcement:** on Linux, `cpu.max` and `cpu.weight` under the [cgroup v2](https://docs.kernel.org/admin-guide/cgroup-v2.html) hierarchy. On Firecracker / Kata, vCPU count + VMM-level throttles. On WASM, the runtime's fuel / epoch interruption mechanism (see [Wasmtime fuel](https://docs.wasmtime.dev/examples-fuel.html) and [epoch interruption](https://docs.wasmtime.dev/examples-interrupting-wasm.html)).
- **Pattern:** fail-closed. When the duration ceiling trips, the sandbox is killed, the policy plane is notified, the evidence stream (chapter 06) gets a `resource_cap_hit` event. There is no "soft extension" at the sandbox plane — a tool that genuinely needs more time is renegotiating through the runtime (mod-402) and the policy plane, not quietly running longer.

### Memory

- **Shape:** a byte ceiling (`memory_mb`). Optionally a soft ceiling that triggers a pressure signal before the hard one triggers an OOM kill.
- **Enforcement:** `memory.max` under cgroup v2 ([memory controller docs](https://docs.kernel.org/admin-guide/cgroup-v2.html#memory)), VMM-level `--mem-size` on Firecracker, WASM runtime limits. Enable the kernel OOM killer *inside* the sandbox and let the host OOM killer be the second line of defence.
- **Pattern:** OOM kill at the ceiling; the sandbox plane records the OOM with the (tenant, session, tool, args-summary) tuple before the kill is acknowledged. OOM is a signal, not a mystery — it should land in the evidence stream faster than the user notices.
- **Note on swap:** disable swap for sandboxed workloads. Letting a sandbox consume swap turns a bounded failure into a host-slowdown.

### Disk

- **Shape:** a byte ceiling per declared writable mount (`/tmp` size, `/workspace` size, overlay-upper size).
- **Enforcement:** tmpfs `size=` mount options for ephemeral writable paths; XFS or ext4 project quotas for persistent overlays; VMM-level virtio-blk size caps. Monitor free-inode state in addition to byte count — a fork-bomb's file creations can exhaust inodes well before bytes.
- **Pattern:** write failures bubble to the tool as normal ENOSPC. The sandbox plane does not silently extend the budget. A tool whose legitimate workload needs more disk declares a larger ceiling on its registry entry, reviewed.

### Network

- **Shape:** sustained and burst bandwidth caps per sandbox (`ingress_mbps`, `egress_mbps`), plus a per-destination connection count ceiling (`max_connections`). Some sandboxes want a byte-volume cap per call (`egress_mb`); the exfiltration threat model drives this.
- **Enforcement:** Linux [tc](https://man7.org/linux/man-pages/man8/tc.8.html) with HTB or `tbf` per network namespace; VMM-level rate limiting on Firecracker's [rate-limiting API](https://github.com/firecracker-microvm/firecracker/blob/main/docs/api_requests/patch-network-interface.md); eBPF with [Cilium bandwidth manager](https://docs.cilium.io/en/stable/network/kubernetes/bandwidth-manager/) when the substrate is Kubernetes.
- **Pattern:** quiet throttling, not kills. A tool that hits its bandwidth ceiling slows down and gets an audit record; it does not terminate. Egress byte-volume is a compliance signal (chapter 06) regardless of whether a cap triggered.

### Process / handle state

- **Shape:** `max_pids`, `max_open_files`, `max_threads`.
- **Enforcement:** `pids.max` under cgroup v2 ([pids controller](https://docs.kernel.org/admin-guide/cgroup-v2.html#pid)); `RLIMIT_NOFILE`, `RLIMIT_NPROC` via prlimit; VMM-level file-handle caps.
- **Pattern:** fork-bomb resistance. A tool that exceeds its pid budget stops being able to fork; the plane records the attempt. Treat a pid-ceiling hit as a security signal, not just an operational one.

### The registry fragment, together

```yaml
# excerpt from a ServerVersion carried by the registry (mod-403)
resources:
  defaults:
    cpu_m: 1000
    duration_s: 60
    memory_mb: 512
    disk:
      /tmp:       {size_mb: 128}
      /workspace: {size_mb: 1024}
    network:
      egress_mbps: 50
      ingress_mbps: 50
      max_connections: 32
      egress_mb_per_call: 200
    pids:
      max_pids: 128
      max_open_files: 512
      max_threads: 128
  per_call_overrides:
    # the policy plane may raise or lower bounded values; the ceiling in `caps`
    # is the absolute max.
    cpu_m:         {min: 100, max: 4000}
    duration_s:    {min: 5,   max: 600}
    memory_mb:     {min: 64,  max: 4096}
```

The `per_call_overrides` block is what the policy plane (mod-405) is allowed to adjust; nothing exceeds the ceiling in `defaults`. A request that asks for more than the ceiling is rejected at admission, not silently clamped — the caller needs to know their request was wrong.

## Tenant-level and fleet-level quotas

Per-sandbox quotas keep one workload from taking the host. Tenant-level and fleet-level quotas keep one tenant (or one runaway deployment) from taking the plane. Pick two numbers per tenant:

- **Steady-state sandbox budget.** Maximum concurrently running sandboxes allocated to the tenant. A tenant hitting this limit queues or is shed upward to the managed fallback (chapter 04), not refused to the user.
- **Hot-path admission rate.** Sandboxes/second the tenant may create. The number exists to protect the plane against a stuck caller spawning sandboxes in a tight loop.

Pick two numbers per fleet:

- **Headroom.** Fraction of substrate capacity reserved for hardware-failure drain and for managed-fallback failback. 10-20% is a common starting point; your SRE plane (mod-410) will refine with evidence.
- **Hot-standby pool size.** For snapshot-restore substrates (chapter 05), the number of pre-warmed sandboxes kept idle per tier so that the common case is restore-from-snapshot, not cold-create.

These numbers live in your platform-level configuration, not in individual tool entries; the SRE plane tracks their consumption.

## The leak problem

A sandbox has three honest lifecycles:

1. **Complete normally.** Tool returned, output drained, sandbox destroyed.
2. **Fail.** Tool errored, quota hit, the plane emitted an event, sandbox destroyed.
3. **Abandoned.** The session that owned the sandbox is gone — the runtime crashed, the user hung up, the agent looped forever and the runtime cancelled the session but the cancellation did not propagate for some reason, the client never sent `done`. The sandbox has no work to do, nobody cares about its output, and it keeps consuming its quota until something notices.

Case 3 is the leak you engineer against. Leaks happen; the plane assumes they will.

### Detection

Three signals catch a leak early:

- **No owning session.** The sandbox-to-session binding is renewed by a heartbeat from the runtime. If the heartbeat lapses beyond a threshold, the sandbox is a *suspected* leak. A reasonable threshold is 2× the runtime's checkpoint interval (mod-402, chapter 02) so that momentary runtime pauses do not count.
- **No live tool call on an idle timer.** A sandbox that is holding the capability bundle but has not been in an active tool invocation for longer than a per-tool threshold is suspect. For code-interpreter-shaped tools the threshold is minutes; for long-running analysis tools it might be hours. The threshold is on the tool's registry entry.
- **Owning session reports completed / cancelled / expired.** The runtime's own state transition should fire a sandbox-release signal; if the signal is delivered, the sandbox is a *confirmed* leak if it is still alive after the release deadline.

### The eviction policy

Write the eviction policy as a ladder:

1. **Grace period.** From the first "suspected leak" signal, wait one grace interval (e.g. 60 seconds). The signal clears automatically if the runtime resumes its heartbeat; the ladder stops there.
2. **Soft cancel.** Deliver a cancellation to the sandbox — SIGTERM to the top-level process in the guest; on VMs, invoke the VMM's shutdown API with a bounded wait. Record a `soft_cancel_initiated` evidence event.
3. **Hard kill.** On soft-cancel timeout (e.g. 10 seconds), kill the sandbox outright. SIGKILL to the process; `KillMicroVM` / equivalent to the VMM. Record a `hard_kill` event and the reason code.
4. **Resource reclamation.** Tear down the network namespace, unmount the tmpfs, destroy the overlay, return the sandbox slot to the pool. Record `reclaimed` with the reclaimed bytes and cpu-seconds.
5. **Attribution.** Every eviction is attributed to a cause code: `heartbeat_lost`, `idle_timeout`, `session_cancelled`, `session_completed_without_release`, `tenant_quota_exceeded`, `fleet_pressure`. The policy plane (mod-405) ingests the cause-code histogram; recurring causes are reviewed.

The time from "the session goes away" to "the sandbox is reclaimed" is a load-bearing SLI. Give it a name — e.g. `sandbox_reclamation_time_p99` — and track it in mod-410. Common targets are measured in single-digit minutes for mass-consumption tools and seconds for high-churn ones.

### Eviction under fleet pressure

The ladder above describes *individual* leak eviction. When the fleet is under pressure — the headroom buffer is consumed, capacity is close to exhausted — a second ladder runs:

1. **Admission shedding.** The sandbox gateway starts refusing new sandbox requests above a tenant-quota-prorated threshold, routing them to the managed fallback (chapter 04) with an explicit reason.
2. **Idle-first eviction.** Sandboxes with the longest idle time (per their idle timer) are evicted first, in LRU order within tenants, respecting tenant floors.
3. **Over-quota-tenant eviction.** If a single tenant is consuming more than its steady-state budget and the fleet is under pressure, that tenant's sandboxes are evicted before other tenants'.
4. **Never evict for substrate-health reasons silently.** If a host is being drained, the affected sandboxes get an informative shutdown signal and the eviction is attributed to `substrate_drain`, not `tenant_quota_exceeded` or `idle_timeout`. Attribution is what makes an audit possible.

The pressure ladder is policy that the SRE plane reads; it is not something a single on-call engineer tunes during an incident without leaving a record.

## The quota decision as evidence

Three things make this chapter's work auditable:

- **Every quota value is traceable to the registry entry that set it.** The tool version, the version hash, the admission PR that landed the version, and the policy-plane decision for the specific call. Chapter 06 picks this thread up.
- **Every quota-cap hit emits an event.** `resource_cap_hit` with the dimension, the configured value, the observed value at hit, the sandbox id, the session id, the tenant id, the tool id, and the timestamp. The event lands in the compliance evidence stream.
- **Every eviction emits an event.** `eviction` with the ladder stage, the cause code, and the reclaimed-resource totals. The histogram over time is the number you use to argue for tighter — or looser — budgets.

If you cannot answer, for an arbitrary sandbox that ran yesterday, "what quota did it have, why was it chosen, did it hit a cap, if so which, and when was the sandbox reclaimed," the chapter's work is incomplete.

## Summary

- Quotas on five dimensions — CPU, memory, disk, network, process state — carried in the tool's registry entry, enforced by cgroups, VMM, or WASM-runtime limits.
- Tenant-level budgets, fleet headroom, and a hot-standby pool keep the plane from being taken by one tenant or one runaway caller.
- Leaks are assumed. The eviction ladder (grace → soft cancel → hard kill → reclamation → attribution) runs on three detection signals (heartbeat lapse, idle timeout, session-release).
- Fleet pressure has its own ladder (admission shedding → LRU-within-tenant → over-quota-tenant eviction → attributed substrate drains).
- Every quota decision and every eviction is attributable to a registry entry, a policy decision, a cause code — feeding chapter 06's evidence stream.

The next chapter opens up the managed fallbacks — E2B, Daytona, Modal — that absorb the overflow and the tiers you do not want to host yourself.
