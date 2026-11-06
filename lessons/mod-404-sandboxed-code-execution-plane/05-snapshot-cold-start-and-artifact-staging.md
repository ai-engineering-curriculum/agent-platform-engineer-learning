# Snapshot-Based Cold Start and Artifact Staging

The first tool call in an agent's session feels different from the hundredth. A session that waits 8 seconds for a workspace to come up looks broken; a session that waits 400 milliseconds feels alive. The gap between those two numbers is almost entirely the cold-start and artifact-staging story. A platform that treats cold start as "whatever the substrate gives you" ships the 8-second experience. A platform that owns cold start as a *contract* — single-digit seconds, with a p99 you measure — needs to engineer three things: snapshot management, an artifact-staging pipeline, and a warm-pool layer that converts cold creates into hot resumes.

This chapter pins all three. It builds on chapters 01–03; the capability bundle from chapter 02 is still what the sandbox is admitted against, and the quota and eviction policy from chapter 03 still governs whether a sandbox survives long enough to serve a call.

## What "cold start" actually decomposes into

An honest measurement of the time from "runtime asks for a sandbox" to "tool code can run" is not one number — it is a stack:

1. **Provider dispatch.** Routing (chapter 04), policy consult (mod-405), provider selection.
2. **Substrate instantiation.** Spawning the Firecracker microVM, booting the guest kernel, or starting the gVisor/`runsc` process, or loading the WASM module into the embedder.
3. **Image hydration.** Pulling or mounting the sandbox rootfs; in snapshot pipelines, this is the restore step.
4. **Artifact staging.** Making the tool's inputs, credentials, and declared `/workspace` contents available inside the sandbox.
5. **Workload bootstrap.** Running whatever the sandbox has to run *before* the actual tool invocation (sidecar supervisor, language runtime warm, model-weight cache mount).
6. **Tool invocation.** The actual call lands.

Every optimisation in this chapter reduces or removes a step. The common mistake is to optimise stage 2 (substrate instantiation) when stage 3 or stage 4 is the dominant cost; measure the breakdown before you pick what to fix. Chapter 06's evidence stream is where this breakdown lands.

## Snapshotting, by substrate

Not every substrate supports snapshot restore in the same shape. Pick the pattern per substrate; do not assume one of them.

### Firecracker — persistent snapshots

[Firecracker's snapshot support](https://github.com/firecracker-microvm/firecracker/blob/main/docs/snapshotting/snapshot-support.md) is the mature story for hardware-virt restore. A snapshot is three artefacts: the memory file, the microVM state file, and the VM configuration. Restore pauses the file loader on the memory file (on-demand paging via `userfaultfd` or `uffd` snapshot restoration) and resumes the vCPUs. [Diff snapshots](https://github.com/firecracker-microvm/firecracker/blob/main/docs/snapshotting/snapshot-support.md#diff-snapshots) let you capture only pages changed since a base snapshot.

Snapshot pipeline:

- **Build-time:** boot a clean microVM with the guest image, wait for the workload-bootstrap stage to finish (Python interpreter warm, common imports loaded, model-weight caches primed if applicable), then take a *base snapshot*.
- **Cold-path trigger:** when the gateway cannot serve from the hot pool, restore the base snapshot into a fresh microVM. Restore is in the tens-of-milliseconds range per the [Firecracker snapshotting documentation](https://github.com/firecracker-microvm/firecracker/blob/main/docs/snapshotting/snapshot-support.md); your p99 will be worse because of memory-file read latency over your storage tier.
- **Security floor:** snapshots retain the guest's memory and filesystem state at snapshot time. Treat every snapshot as a potential cross-invocation persistence channel (chapter 01, threat class 4). A snapshot built from a compromised VM poisons every sandbox it hydrates. Protect the snapshot build pipeline the same way you protect a base-image build: signed artefacts, reproducible builds where possible, admission review for any new base snapshot, and automated re-build cadence so stale snapshots do not accumulate exploitable state.
- **Per-call freshness:** the base snapshot is not what the sandbox serves calls on. After restore, the plane overlays the per-call capability bundle — fresh credentials, fresh workspace, fresh network namespace — so that two calls hydrated from the same snapshot still differ on every call-specific axis.

### gVisor — container-image warm pools

gVisor does not have a VMM-shaped snapshot mechanism. The equivalent pattern is **container-image reuse with a hot-pool layer**: build the OCI image with everything a tool needs pre-installed and pre-imported; keep a small pool of warmed sandboxes per image; allocate from the pool on request. Cold creates still happen but only on pool exhaustion.

Equivalent discipline: image build is the security review point, the warm pool is the latency primitive.

### Kata — hypervisor-backed snapshot options

Kata inherits its snapshot support from the VMM it uses. With the Firecracker-backed Kata configuration, snapshot restore is available; with QEMU-backed Kata, snapshot semantics match the VMM's live-migration / save-state support. <!-- needs-research: confirm which Kata VMM back-ends ship snapshot/restore support at your reading time; the Firecracker and Cloud Hypervisor backends have differing maturity. See the Kata repo: https://github.com/kata-containers/kata-containers -->

### WASM / Pyodide — module caching and instance reuse

WASM runtimes do not snapshot a running sandbox the way VMs do, but two primitives give equivalent latency:

- **Precompiled modules.** Wasmtime's [AOT compilation cache](https://docs.wasmtime.dev/cli-cache.html) means the second-and-onward loads of a module skip the compile step. Ship precompiled modules; measure load time with and without the cache.
- **Instance pooling / reuse.** For stateless modules, re-use an instance across calls, resetting memory and capability imports between invocations. For Pyodide, [snapshotting the Python heap](https://pyodide.org/en/stable/usage/) via its own mechanisms can skip the interpreter warm step.

## The warm pool

For every substrate, a hot pool of pre-warmed sandboxes converts the common case from a cold restore to a near-zero-cost claim.

### Pool sizing

Treat pool sizing as a queueing problem, not a guess. Four numbers decide the pool shape per tier:

- **Request rate (λ).** Sandboxes per second, per tier, per tenant (or at least per tier if tenant-level is not separable).
- **Service time (S).** The time from "sandbox claimed by a call" to "sandbox released back to the pool or destroyed."
- **Hot create time (H).** The time to materialise a new sandbox when the pool is empty (restore-from-snapshot for Firecracker; warm-up for gVisor; precompile for WASM).
- **Target hit rate (θ).** Fraction of requests the pool should serve without falling to a cold create. A platform taking the cold-start contract seriously targets θ ≥ 0.99.

Pool size is a function of λ, S, H, and θ. Keep a running measurement of all four and *recompute* the pool size on a schedule the SRE plane (mod-410) agrees. A static pool size tuned at launch is wrong within a quarter.

### Pool invariants

Three properties the pool must preserve regardless of sizing:

- **Call-fresh.** A sandbox served from the pool carries no state from a previous call. The capability bundle, the credentials, the workspace, and the network namespace are reset or recreated before the call lands.
- **Health-checked.** A sandbox the pool offers has passed a liveness check — the supervisor process responds, the workspace mount is writable, the network namespace resolves DNS through the platform resolver. A stale sandbox silently surviving in the pool is the thing that cases a mysterious p99 cliff.
- **Capped.** The pool itself has a quota (chapter 03's fleet headroom). Pools do not grow unbounded in response to demand spikes; above a threshold, new requests get a cold create or route to the managed fallback (chapter 04).

### Pool and eviction interplay

The eviction policy from chapter 03 applies to the pool too. A sandbox that has been in the pool for longer than a max-idle (and is not actively protecting a hot workload) is evicted and replaced with a fresh one on its schedule. Pool sandboxes do not age indefinitely; stale pool members accumulate drift from the base snapshot or image and, more importantly, miss security updates that only land in new base snapshots.

## The artifact-staging pipeline

If snapshot restore gives you a running sandbox in 50 milliseconds but the tool's workspace needs 200 MB of inputs that take 4 seconds to stage, the user still sees a 4-second start. Artifact staging is the other half of the single-digit-seconds contract.

### The staging contract

Staging the inputs for a call is the plane's responsibility, not the tool's. The plane owns four verbs:

1. **Resolve.** Given the call's declared inputs (mounts, files, directories, model-weight references), expand them into a concrete artefact manifest with content hashes.
2. **Fetch.** Pull the artefacts from their source — object store (S3 / GCS / Azure Blob), the session's workspace in the runtime (mod-402), the tool's registry-pinned read-only assets.
3. **Place.** Make the artefacts visible inside the sandbox at the declared paths with the declared modes (chapter 02).
4. **Verify.** Assert, before handing the sandbox to the tool, that every declared input is present, matches its hash, and is at the right mode.

Stages 1–3 are latency; stage 4 is correctness. The tool must not run on an input whose hash does not match what the manifest declared — that is how prompt-injection-poisoned inputs quietly reach tools that would otherwise reject them.

### Patterns that reduce staging latency

A set of techniques, ordered roughly by how much they help:

- **Content-addressable caching.** Every artefact is keyed by its content hash. A fetched artefact lives in a shared cache on the host (or a region-local cache for managed providers); subsequent sandboxes that need the same artefact get it from cache in milliseconds. Standard CAS primitives apply — [BuildKit's content-addressable cache](https://github.com/moby/buildkit/blob/master/docs/dev/cache-api.md) is one implementation reference, [IPFS CIDs](https://docs.ipfs.tech/concepts/content-addressing/) another.
- **Overlay mounts.** Instead of copying inputs into the sandbox's writable filesystem, mount the read-only inputs via an [overlayfs](https://docs.kernel.org/filesystems/overlayfs.html) or virtio-fs layer. A 2 GB read-only input mounted takes milliseconds; a 2 GB read-only input copied takes seconds.
- **Lazy pulling (for large read-only blobs).** For model weights, container images, or large assets, use lazy-pull primitives like [SOCI (Seekable OCI)](https://github.com/awslabs/soci-snapshotter) or [eStargz](https://github.com/containerd/stargz-snapshotter) so that only the blocks actually read by the tool are pulled. First-call latency is dominated by the small subset the tool reads, not the whole artefact size.
- **Staging in parallel with sandbox hydration.** Start the artefact fetch as soon as the call is admitted; complete it concurrently with the snapshot restore. The sandbox pauses in `waiting_for_inputs` only if the fetch is slower than the restore.
- **Pre-stage for predicted calls.** When the runtime signals "likely next call is X" (e.g., a routing agent has decided to invoke tool Y), begin fetching Y's inputs speculatively. Cancel on wrong prediction. This is advanced; measure the hit rate before committing engineering effort to it.
- **Workspace persistence for same-session sequential calls.** If an agent's session makes several tool calls in a row against the same workspace, reuse the workspace rather than re-staging. The session carries a *staging ticket* that the next call presents; the plane extends the workspace's lifetime rather than destroying and recreating.

### Credential staging

Credentials are a staging artefact too, but with distinct properties:

- **Short-lived, minted per call.** A broker (same pattern as mod-403 chapter 05) mints a short-lived token with the exact scope the call needs. The token's `exp` is bounded by the sandbox's `duration_s`.
- **Delivered on a dedicated tmpfs mount.** `/workspace/.credentials/<name>` with mode `0400`, owned by the sandbox user.
- **Destroyed on sandbox teardown.** The tmpfs is unmounted and the underlying memory is zeroed. The broker revokes the token on teardown as defence-in-depth against future replay of a captured token.
- **Attributed in the evidence record.** Chapter 06's record includes the token identifier (not the secret), the broker that issued it, and the scope — enough to answer "what did the sandbox have, when, and why" during an audit.

## Keeping the pipeline attributable

Three invariants make the staging pipeline auditable:

- **Every artefact has a provenance.** Where did the bytes come from? A session-scoped workspace (via the runtime)? A registry-pinned read-only asset? A broker-issued credential? The provenance is in the staging manifest and the evidence record.
- **Every artefact's hash is checked at the sandbox boundary.** A content-addressed artefact whose hash does not match on verify is rejected; the sandbox is destroyed before the tool runs. Record the attempt — a hash mismatch is a strong signal.
- **Every snapshot has a signature and a build provenance.** Treat the snapshot like a container image: signed with Sigstore-shaped tooling ([Cosign](https://docs.sigstore.dev/signing/signing_with_blobs/)), the build is reproducible as much as the substrate allows, and the admission pipeline refuses unsigned snapshots.

If an on-call engineer at 03:00 cannot answer, for a sandbox that ran yesterday, "which snapshot was it restored from, who signed the snapshot, what was in its workspace, what were the artefact hashes, which credentials did it have, and how long did each stage of its start take," the chapter's work is incomplete.

## The cold-start SLI

The sandbox plane's cold-start work earns an SLI the SRE plane (mod-410) consumes. A reasonable shape:

- **Name:** `sandbox_time_to_tool_call_p99`.
- **Definition:** the 99th-percentile elapsed time, per tier per region per provider, from the gateway admitting a sandbox request to the first tool invocation running.
- **Target:** tier-dependent. Code-interpreter tiers in the single-digit-seconds range; low-isolation WASM tiers in sub-second; GPU-attached tiers measured in seconds.
- **Decomposition:** every value is accompanied by its breakdown — dispatch, instantiation, image hydration, staging, bootstrap, invocation. Teams argue about the breakdown, not about the headline number.

Pair it with an honesty SLI: `sandbox_cold_start_breakdown_lag` — the time from an event firing to its appearing in the evidence stream. If the breakdown is not visible in evidence within seconds, the SLI is theatre.

## Summary

- Cold start is a stack, not a number. Measure the breakdown; optimise the dominant stage; do not polish substrate instantiation if staging is the problem.
- Snapshot patterns differ by substrate: Firecracker has first-class snapshot restore with diff snapshots; gVisor leans on container-image warm pools; Kata inherits its VMM's capability; WASM uses AOT caches and instance reuse.
- A warm pool, sized against measured λ, S, H, and target hit rate, converts common-case cold creates into near-zero-cost claims. Pool members are call-fresh, health-checked, capped, and periodically recycled.
- The artefact-staging pipeline owns resolve / fetch / place / verify. Content-addressable caching, overlay mounts, lazy pulls, parallel fetching, and workspace reuse are the techniques that keep staging inside the latency budget.
- Credentials are staging artefacts with their own lifecycle: broker-minted, tmpfs-mounted, destroyed on teardown, attributable in evidence.
- Every snapshot, every artefact, and every staging step is attributable. Signed snapshots, content-hashed artefacts, and provenance-stamped staging manifests are what make the pipeline auditable.
- The `sandbox_time_to_tool_call_p99` SLI with its stage breakdown is what the SRE plane consumes.

The final chapter picks up every event this and earlier chapters have emitted — syscall violations, egress denials, resource-cap hits, OOMs, snapshot and staging telemetry — and pins how they become the compliance evidence stream.
