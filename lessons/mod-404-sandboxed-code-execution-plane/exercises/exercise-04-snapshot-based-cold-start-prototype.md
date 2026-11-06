# exercise-04: Snapshot-Based Cold-Start Prototype

**Estimated effort:** 6 hours · **Output:** one working prototype, one measured breakdown of the cold-start stack, one artefact-staging sketch, one short memo on what you learned · **Pairs with:** chapter [05](../05-snapshot-cold-start-and-artifact-staging.md); feeds project 401 and mod-408.

## Objective

Build a prototype that measurably takes a cold sandbox create from "default substrate behaviour" to "single-digit-seconds experience" by introducing one of the three primitives from chapter 05 — snapshot restore, a warm pool, or an artefact-staging cache. Measure the breakdown before and after. Write down what the breakdown told you.

The prototype's quality is measured by three things: the breakdown is measured, not estimated; the win is attributable to a specific primitive, not to a mystery; the memo names what you would do next on the strength of the data.

## Prerequisites

- You have read chapter [05](../05-snapshot-cold-start-and-artifact-staging.md).
- You have exercises 01, 02, 03 on hand — you need a substrate pick, a capability bundle, and a quota profile. If you did not pick a substrate your local machine can run, pick a surrogate for this exercise (gVisor `runsc` is the easiest to get going locally; Firecracker needs KVM; WASM needs a local Wasmtime).
- You have a machine or VM where you can run your chosen substrate. Firecracker requires KVM; most cloud VMs can run it on nested or bare-metal instances.
- You have the chapter's references to hand: the [Firecracker snapshotting documentation](https://github.com/firecracker-microvm/firecracker/blob/main/docs/snapshotting/snapshot-support.md), the [Wasmtime cache documentation](https://docs.wasmtime.dev/cli-cache.html), and whatever OCI image / tooling ([skopeo](https://github.com/containers/skopeo), [crun](https://github.com/containers/crun), or your platform's runtime) matches your substrate pick.

## Problem statement

Pick one substrate and one primitive, build the prototype, measure it. The combinations:

- **Firecracker + base snapshot + restore.** Boot a clean microVM, let a trivial `supervisor` process initialise, take a snapshot. Build a thin tool that restores from the snapshot and runs a one-line tool call inside it. Measure.
- **gVisor + warm-pool layer.** Build a pool of three warm `runsc` sandboxes pre-started with an OCI image that imports `pandas` and `numpy` on startup. Build a claim/release API around the pool. Measure claim time vs. cold-create time.
- **WASM / Wasmtime + AOT cache + instance reuse.** Compile a Rust or Python WASM module, enable the AOT cache, and build a harness that reuses an instance across three calls, resetting memory between them. Measure load+run time for first call, second call (cache warm), and third call (instance reused).
- **Any substrate + content-addressable artefact-staging cache.** Build a small staging layer that fetches a 100-MB input file from object storage, hashes it on arrival, caches it, and mounts it read-only into the sandbox. Measure cold-fetch vs. cache-hit staging time.

You pick one combination. The point is not "which primitive is fastest overall" — the point is that you measured one and the breakdown is what you work with next.

## Requirements

### Part 1 — Measurement harness (build this first)

Before any optimisation, build a harness that times the six stages from chapter 05:

1. Provider dispatch (your harness is the provider).
2. Substrate instantiation (your substrate start).
3. Image hydration (rootfs mount or image load).
4. Artefact staging (synthetic 50-MB input placed in `/workspace`).
5. Workload bootstrap (import Python stdlib, or load WASM module).
6. Tool invocation (run the one-line tool: `python -c "print(42)"` or equivalent).

The harness emits six timings per run, plus a total. Run 20 iterations on cold starts, 20 on hot starts (if applicable to your primitive), collect p50 / p95 / p99.

### Part 2 — The prototype (the main deliverable)

Implement the chosen primitive. Keep the code small; the point is the measurement, not the elegance:

- **Firecracker snapshot:** a shell script that creates a microVM via the Firecracker API, waits for cloud-init to finish, pauses, snapshots, destroys. A second script restores and invokes.
- **gVisor warm pool:** a tiny Python / Go server that owns three `runsc`-started containers as a pool, exposes `claim` / `release` endpoints, and reaps / replaces pool members on shutdown.
- **WASM AOT + instance reuse:** a Rust or Python harness that loads a module (with AOT cache enabled), creates an instance, runs the call, resets memory, and re-runs.
- **Staging cache:** a small Python daemon that takes a `(bucket, key, expected_sha256)` request, fetches from the object store on cache miss, writes to a local cache, and places a bind-mount target for the sandbox.

Include a toggle to disable the primitive; this is what makes the measurement meaningful.

### Part 3 — Measured breakdown

Run the harness in both modes (primitive on / primitive off). Produce a table:

| Stage | Primitive OFF (p50 / p95 / p99) | Primitive ON (p50 / p95 / p99) | Delta |
|---|---|---|---|
| Provider dispatch | | | |
| Substrate instantiation | | | |
| Image hydration | | | |
| Artefact staging | | | |
| Workload bootstrap | | | |
| Tool invocation | | | |
| **Total** | | | |

Numbers in milliseconds. Cite the hardware you measured on (host specs, VM shape, storage type), the iteration count, and the single-digit variance pattern you observed. Numbers without hardware context are not interpretable.

### Part 4 — Attribution and honesty

Three paragraphs after the table:

- **What specifically changed and why did it?** "Substrate instantiation dropped from 450 ms (p95) to 55 ms (p95) because Firecracker's uffd-based memory restore skipped guest-kernel boot" is the shape — specific stage, specific mechanism.
- **What did *not* change and why not?** If staging is still 1200 ms out of your 1500 ms budget, the primitive you chose did not touch that stage. Own it.
- **The next primitive you would add.** On the strength of the breakdown, what is the dominant remaining stage? Pick the primitive from chapter 05 that would address it and name what you would expect.

### Part 5 — Artefact-staging sketch (even if you did not implement it)

Regardless of which primitive you chose for the prototype, sketch the end-to-end artefact-staging pipeline for a representative call. Chapter 05's four verbs (resolve, fetch, place, verify) become concrete steps: where does the manifest come from, where does the cache live, which hashes are checked, which steps run in parallel with substrate hydration. Sketch as a sequence diagram or as a labelled flow.

### Part 6 — Memo (~1 page)

Short write-up:

- What primitive you picked and why.
- What the measurement harness told you (one chart or table).
- What surprised you (if nothing surprised you, the measurement granularity is probably too coarse).
- What you would need to productionise this prototype: pool sizing against measured λ and S, snapshot build-pipeline hygiene, admission-test for new snapshots, evidence-stream integration (chapter 06), SLI definitions.

## Starter guidance

- Keep the harness minimal and *trustworthy*. A harness that uses coarse `time()` calls will tell you nothing; use monotonic nanosecond-precision timers and emit timings per stage.
- Measure *clean* starts. Warm up the kernel's file cache before the first iteration so that iteration 1 is not artificially slow; then clear it (`echo 3 > /proc/sys/vm/drop_caches`) when you want to measure actually-cold starts.
- For snapshot restore (Firecracker), the published ~tens-of-milliseconds restore time comes from reading the memory file from fast local storage. If you are restoring from an object store or a networked FS, your numbers will be very different. Say which you measured.
- For warm-pool implementations, your pool's correctness is more important than its speed. A pool that hands out a stale sandbox is worse than a pool that is a hair slower. Include at least one health check per pool member before issuing it.
- `<!-- needs-research: ... -->` your way past anything the project docs do not confirm — especially around uffd semantics, gVisor's handling of `clock_gettime` in restored sandboxes, or WASM AOT cache semantics across Wasmtime versions. The upstream story moves.

## Acceptance criteria

Mark done when:

- [ ] The harness emits per-stage timings and runs cleanly on cold and warm starts (where applicable).
- [ ] The prototype runs end-to-end with the primitive on and off; the toggle works.
- [ ] The breakdown table has p50/p95/p99 for all six stages in both modes, with hardware context cited.
- [ ] The attribution paragraphs name the specific stage and mechanism responsible for the measured win (or the honest absence of one).
- [ ] The artefact-staging sketch covers resolve / fetch / place / verify, including the hash check at the sandbox boundary.
- [ ] The memo names what you would need to productionise, including the SLI from chapter 05 (`sandbox_time_to_tool_call_p99`) and the evidence hook from chapter 06.

## Stretch goals

- **Second primitive.** Based on the breakdown, add the primitive your memo recommended next and re-run the harness. Three columns instead of two in the table.
- **Evidence wiring.** Emit the `sandbox.created` event (chapter 06) with the breakdown as payload, write it to a local JSONL file, and run a query to pull all events for a given synthetic session id. This is the first draft of what chapter 06's pipeline consumes.
- **Hot-pool sizing exercise.** Measure your workload's λ, S, and H from the harness. Compute, from the formula in chapter 05, the pool size you would run for θ ≥ 0.99. Sanity-check against your Part 2 prototype.
- **Managed-vendor comparison.** Submit the same workload to one of E2B / Daytona / Modal via their SDK. Record their observed cold-start time. Compare against your self-hosted number and your chapter-04 routing rule — does the vendor win on cold start? If so, should your routing weight shift? If not, is the vendor's isolation-boundary trade-off worth the cost difference?
