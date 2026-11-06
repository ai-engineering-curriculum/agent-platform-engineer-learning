# Substrates and the Agent Threat Model

The MCP registry (mod-403) decides *which tools exist* and *who may call them*. The sandboxed code-execution plane decides *where those tools (and any model-generated code they execute) actually run*. The difference matters because the model is non-deterministic, the user is adversarial by default in the threat model, and a tool-call payload is at best untrusted data and at worst an attempt to execute untrusted code with the platform's credentials. If your sandbox plane is weaker than the attacker your agents are exposed to, the rest of the platform is decoration.

This chapter fixes the threat model you build against and compares the substrates the industry has to offer. Later chapters pick the policies (02), the quotas and eviction behaviour (03), the managed fallbacks (04), the cold-start pipeline (05), and the evidence stream (06) against that frame.

## The slice this plane owns

The sandbox plane sits between the tool-gateway (mod-403) — which may have decided *"this tool call needs to execute arbitrary code"* — and the underlying kernel / hypervisor / runtime that will host that execution. It owns:

- **Per-call isolation.** Every tool invocation that executes untrusted content gets a workspace whose compromise cannot reach another tenant, another session of the same tenant, or the platform's own control plane.
- **Capability shape.** What syscalls the workspace may make, what network destinations it may reach, which filesystem paths are readable or writable, and which resources (CPU, memory, disk, pids, file handles) it may consume.
- **Lifecycle.** Cold start, hot resume from snapshot, pause, kill. Including the eviction path for a sandbox whose session has abandoned it (chapter 03).
- **Fallback routing.** When the self-hosted substrate cannot admit a call — capacity, outage, geography, licence — the plane can route to a managed vendor (chapter 04) without changing the contract the runtime sees.
- **Evidence emission.** Every policy-relevant sandbox event (syscall denial, egress block, resource-cap hit, OOM kill) becomes a compliance record (chapter 06).

The sandbox plane does **not** own:

- **Tool authorship.** That is `agentic-ai-engineer` level 30; the plane admits their code into a sandbox but does not write it.
- **Which tool may be called at all.** That is mod-405 (the policy plane). The sandbox plane assumes the policy plane already said yes.
- **The MCP contract with the agent.** That is mod-403. The sandbox plane receives *"run this payload with these capabilities"*; it does not speak MCP.
- **The model's inference.** That is mod-406. The plane may host a tool that calls the model gateway, but the inference itself is not sandboxed by this plane.
- **Guardrail logic over the output.** That is mod-411.

Drift off this slice and you re-import problems the other planes already solved — or worse, duplicate half-solutions that the runtime has to arbitrate between.

## The threat model you build against

Pin the threat model to two published, maintained artefacts:

- **OWASP Top 10 for LLM Applications** — the baseline the whole platform defends against; risks include prompt injection (LLM01), insecure output handling (LLM02), sensitive information disclosure (LLM06), excessive agency (LLM08), and others whose current numbering you should check at your onboarding. <!-- needs-research: confirm the current LLM Top-10 revision number and which risks are IDed as LLMxx in the version you cite; the list reshuffles across major revisions. -->
- **OWASP Agentic AI — Threats and Mitigations** — the agent-specific extension, enumerating threats around tool misuse, memory poisoning, delegated-authority abuse, and autonomous-action blast radius.

Against the sandbox plane specifically, three classes of attack dominate:

1. **Untrusted code execution.** The agent writes or receives code (a Python snippet to analyse a CSV, a shell command to grep logs, an npm install from a tool result) and the platform is asked to run it. The attacker's leverage is to pick code whose side effects — filesystem writes, network calls, syscalls — breach the containing boundary. This is the primary driver for Firecracker/gVisor/Kata-class isolation.
2. **Exfiltration via tool-reachable network.** Even if the code itself is trivially shell-equivalent, the attacker may want to reach outside the sandbox: a cloud metadata endpoint (`169.254.169.254`), a tenant's internal admin service, or an attacker-controlled host on the open internet. This is the driver for egress policy (chapter 02).
3. **Resource consumption / denial of service.** Fork-bombs, gigabyte-tempfile writes, unbounded regex, memory balloons. The attacker's leverage is to make the sandbox so expensive to operate that the plane trims its policy or increases its quotas. This is the driver for the resource-quota work in chapter 03 and the eviction policy for leaked sandboxes.

A fourth, less obvious, but severe class the OWASP Agentic model names is **persistence between invocations** — an attacker who successfully compromises one sandbox and leaves a payload that will be picked up by the *next* agent reusing that image or that pool. Every snapshot pipeline (chapter 05) must assume this is attempted and design against it.

## The four substrate families

Four substrates cover the design space your plane will realistically pick from. Any vendor product (chapter 04) is, under the covers, one of these.

### Firecracker — microVMs on KVM

[Firecracker](https://firecracker-microvm.github.io/) is an AWS-originated open-source Virtual Machine Monitor built for serverless and container-like workloads. Each sandbox is a full KVM virtual machine with its own kernel, booted from a stripped-down image; the VMM runs in userspace and is designed to a minimal attack surface (a small device model, a slimmed-down VirtIO set). AWS Lambda and Fargate are the headline users.

Isolation boundary: **hardware-virtualisation**. A guest kernel exploit has to cross the KVM / VMM boundary before it touches the host. The threat model is roughly the one your compliance auditor already accepts for multi-tenant cloud compute.

Characteristics:

- Boots in low hundreds of milliseconds without snapshots and in tens of milliseconds with [snapshot restore](https://github.com/firecracker-microvm/firecracker/blob/main/docs/snapshotting/snapshot-support.md) (chapter 05 goes deep on this).
- Memory overhead per VM is small but non-zero; packing density is lower than container-class substrates.
- Requires KVM on bare metal or on a KVM-capable cloud instance type; nested virtualisation is not a supported story.
- Device model is minimal — no attached accelerator passthrough in the common case, no USB, no nested vt-x in the guest.
- The guest kernel is yours to pick and patch; the attack surface you ship is explicit.

Use when: multi-tenant, untrusted code, latency budget allows a sub-second boot; you can run on bare metal or a KVM-capable instance class.

Do not use when: you need GPU-attached execution for arbitrary tool code, nested virtualisation, or very high density on a non-KVM host.

### gVisor — application kernel in userspace

[gVisor](https://gvisor.dev/) is a Google-originated sandbox that implements a Linux-compatible application kernel in userspace. Instead of letting the guest talk to the host kernel through the full syscall surface, gVisor's `runsc` intercepts syscalls and services most of them itself, in Go, with a dramatically reduced host-kernel surface. Google Cloud Run and App Engine standard are headline users.

Isolation boundary: **reduced-syscall-surface, in-userspace reimplementation**. Compromising the sandbox requires an exploit in the gVisor application kernel (not nothing — it is a kernel) *and* a further exploit in the limited host-kernel surface gVisor uses underneath. There is no hardware-virtualisation boundary.

Characteristics:

- Cold start is container-class — low tens of milliseconds for a `runsc` container; the architecture does not require a guest kernel to boot.
- Density is container-class; many sandboxes per host.
- Compatibility is good for POSIX-shaped programs, imperfect for anything that pokes unusual syscalls (certain JITs, certain FUSE-shaped filesystems, `io_uring` is historically limited). The [supported-syscalls matrix](https://gvisor.dev/docs/user_guide/compatibility/linux/amd64/) is the thing to read before committing.
- OCI-runtime compatible; drops into existing container tooling.
- Networking is user-mode by default (Netstack), which is where egress policy (chapter 02) is easiest to enforce.

Use when: high density, container-shaped payloads, latency budget wants tens-of-ms starts, you can accept compatibility gaps for exotic workloads.

Do not use when: the workload needs hardware-virtualisation separation for compliance or regulatory reasons, needs kernel features gVisor does not implement, or runs arbitrary user-supplied native binaries with unknown syscall patterns.

### Kata Containers — lightweight VMs under the OCI runtime

[Kata Containers](https://katacontainers.io/) is a CNCF-hosted project that runs OCI containers inside lightweight VMs — historically on QEMU, increasingly with [Firecracker or Cloud Hypervisor](https://github.com/kata-containers/kata-containers/tree/main/docs) as the VMM — exposing a `kata` OCI runtime handler drop-in. From the Kubernetes-pod operator's view it is just a `RuntimeClass`; underneath, each pod is a VM.

Isolation boundary: **hardware-virtualisation**, same category as Firecracker, delivered through the OCI/Kubernetes shape of the platform.

Characteristics:

- Boot times in hundreds of milliseconds, slower than Firecracker alone because of the OCI shim layer; depends heavily on the VMM backend and on whether direct-volume mounts or virtio-fs is used.
- Native fit for Kubernetes: `RuntimeClass: kata`, no changes to pod spec beyond that in most cases; the operator story leans on existing CNI, CSI, and scheduler work.
- Strong compatibility — a VM is a VM, and the guest kernel can be whatever Linux the image expects.
- Overhead is the price of per-pod VMs: measurable memory and CPU floor per sandbox.

Use when: your platform is Kubernetes-native and you want hardware-virt isolation without rebuilding every scheduling, networking, and volume-mount piece around raw Firecracker.

Do not use when: you need to go below the Kubernetes abstraction for density reasons or because your runtime (mod-402) is not pod-shaped.

### WASM and Pyodide — in-process sandboxes

[WebAssembly](https://webassembly.org/) with [WASI](https://wasi.dev/) is a portable, deny-by-default, capability-shaped execution environment. A host program (e.g. your runtime worker) embeds a WASM runtime ([Wasmtime](https://wasmtime.dev/), [Wasmer](https://wasmer.io/), [WasmEdge](https://wasmedge.org/)) and loads a `.wasm` module. The module has no syscalls except what the host explicitly grants through WASI; filesystem and network access are not ambient but delegated capabilities.

[Pyodide](https://pyodide.org/) is CPython and most of the scientific-Python stack compiled to WASM; it enables Python-shaped code to run in a WASM sandbox (in a browser, in Node, or embedded).

Isolation boundary: **language-runtime deny-by-default**. There is no process boundary, no VM, no reduced-syscall surface — there is a module whose capabilities are those the host hands it. If the host's capability-granting code is wrong, there is no deeper layer to catch it; if the WASM runtime itself has a sandbox-escape bug, the host process is the breach radius.

Characteristics:

- Startup is sub-millisecond for pre-compiled modules; this is the fastest substrate on the chart.
- Density is the host process's density — thousands per host is plausible.
- Language coverage is narrower: WASI-compatible Rust, Go (TinyGo or wasip2-shaped), C/C++; via Pyodide, Python; via dedicated efforts, a slice of JavaScript. Arbitrary binary workloads do not fit.
- The WASM runtime's security track record is good but not long; the OWASP Agentic model specifically notes that the WASM-with-host-capabilities layer is where a lot of real agent code lives and must be reviewed with that in mind.

Use when: the tool code can be compiled to WASM, cold-start latency is single-digit milliseconds, you need massive density, and the capability surface the host hands the module is the small surface you can reason about.

Do not use when: the tool code is arbitrary language / native binary / anything-goes, the workload needs GPU or full POSIX, or the compliance boundary your auditor wants is a hardware or syscall boundary.

## Comparison matrix

| Dimension | Firecracker | gVisor | Kata | WASM / Pyodide |
|---|---|---|---|---|
| Isolation boundary | Hardware (KVM) | Reduced host-syscall surface in userspace | Hardware (KVM, usually Firecracker/Cloud Hypervisor) | Language-runtime capability model (no process/VM boundary) |
| Cold start (no snapshot) | ~hundreds of ms | ~tens of ms | ~hundreds of ms+ (OCI shim) | sub-ms |
| Cold start (with snapshot) | ~tens of ms | n/a (restored via container-image reuse) | varies by VMM | sub-ms |
| Density per host | Low–mid | High | Low–mid | Very high |
| Kernel compatibility | Any Linux you ship as guest | Linux-compatible, with gaps | Any Linux you ship as guest | WASI (narrower, growing) |
| GPU support | Not standard | Limited | VMM-dependent | Not applicable |
| Fits Kubernetes natively | Through Kata or custom scheduling | Yes (`runsc` OCI runtime) | Yes (`RuntimeClass: kata`) | In-process, no scheduler fit |
| Primary deployment references | AWS Lambda / Fargate | Google Cloud Run / App Engine std | OpenShift sandboxed containers, Confidential Containers | Fastly Compute@Edge, Cloudflare Workers, Shopify Functions, Pyodide-based notebook sandboxes |
| What an escape costs | Guest kernel + VMM + KVM exploits | gVisor Sentry + limited host-syscall exploit | Guest kernel + VMM + KVM exploits | Host-process capability bug or runtime escape |

(`Primary deployment references` are the headline production users of each substrate; when the vendor landscape changes, verify.) <!-- needs-research: verify the current state of GPU support in Firecracker and Kata and the WASM-component-model-based exposure of GPU-adjacent resources through WASI at the time of reading. -->

Numbers are *orders of magnitude* and *relative orderings* — not benchmark figures to quote. The absolute numbers you ship against must come from your own measurements on your own hardware. Vendor-published boot times generally reflect best-case snapshot-restore in their best-case configuration; your fleet's p99 will differ.

## Mapping substrates to the threat model

Not every substrate defends every threat equally. The honest mapping:

| Threat class | Firecracker | gVisor | Kata | WASM / Pyodide |
|---|---|---|---|---|
| Untrusted-code sandbox escape | Strong (hardware boundary) | Moderate (relies on gVisor's app kernel + minimal host surface) | Strong | Weak-to-moderate (relies on runtime + host capability code) |
| Egress exfiltration | Equally strong across all if the egress policy is enforced at the plane; weak everywhere if it is not | Same | Same | Same — plus WASM's capability model forces explicit grants, which is helpful |
| Resource DoS | Quotas are external to the substrate; all four rely on the host's cgroups / resource caps | Same | Same | Same, but WASM runtimes often enforce fuel/limits natively |
| Cross-invocation persistence | Depends on the snapshot / artifact-staging pipeline (chapter 05), not the substrate | Same | Same | Same |

In other words: the *substrate* decides your sandbox-escape posture. Your *policy and quota engineering* (chapters 02, 03) decides your egress and DoS posture. Your *staging pipeline* (chapter 05) decides your cross-invocation posture. Picking Firecracker does not fix a weak egress policy; picking WASM does not force a weak one if the host carefully hands out capabilities.

## The decision shape

The platform's substrate choice is rarely one substrate. A typical shape is:

- **Primary tier (bulk of code-execution calls):** gVisor or Kata — container-shaped, Kubernetes-native, acceptable isolation for the dominant workload (model-generated Python/JS snippets executing against a tenant's data).
- **Elevated tier (anything explicitly higher-risk):** Firecracker microVMs with a hardened guest image and tighter egress. Triggered by tool metadata (`risk_tier: T1`) carried over from the registry (mod-403).
- **Specialised tier (specific workloads with strict latency or density needs):** WASM / Pyodide for code that fits (data munging in Python, tool code written in Rust, policy helpers). Explicit opt-in; capability grants reviewed.
- **Managed fallback (capacity, outage, geography):** E2B / Daytona / Modal (chapter 04). The sandbox gateway routes to them with the same contract.

The registry (mod-403) and the policy plane (mod-405) are what *drive* this routing. The sandbox plane admits the call, selects the tier, and emits the attribution. The runtime (mod-402) never sees the substrate name — it sees the sandbox gateway.

## The three obligations

Pin the sandbox plane to three obligations. Everything in chapters 02–06 is a decision about how to deliver them.

1. **"Nothing a compromised sandbox can do reaches another tenant, another session, or the control plane."** Isolation is non-negotiable, and the strength of the boundary is a published property of the plane. This obligation forces the substrate choices in this chapter and the capability engineering in chapter 02.

2. **"The sandbox cannot make a reach my policy did not pre-authorise."** No ambient filesystem access, no ambient network, no ambient credentials — everything is a declared capability, logged on grant and logged on use. This obligation forces the policy work in chapter 02, the eviction behaviour in chapter 03, and the evidence stream in chapter 06.

3. **"A sandbox the session abandoned stops costing the platform within a bounded time."** Leaks happen — the runtime crashes, the user hangs up, the agent loops forever. The plane evicts, reports what was evicted and why, and does not quietly bleed capacity. This obligation forces the quota and eviction work in chapter 03.

A plane that cannot defend all three is a sandbox gateway in name only.

## Summary

- The sandbox plane isolates per-call code execution for the agent fleet. It does not own tool authorship, policy decisions, the MCP contract, inference, or guardrails.
- The threat model is OWASP LLM Top 10 plus OWASP Agentic AI Threats and Mitigations. The sandbox-specific attack classes are untrusted-code escape, egress exfiltration, resource DoS, and cross-invocation persistence.
- Four substrate families cover the design space: Firecracker microVMs (hardware boundary, serverless-origin), gVisor (reduced-syscall userspace kernel), Kata (OCI-shaped lightweight VMs), WASM/Pyodide (in-process capability model).
- The substrate decides the sandbox-escape posture. Policy, quotas, and staging pipelines decide the egress, DoS, and cross-invocation posture.
- Three obligations are non-negotiable: isolation that blocks tenant/session/control-plane blast radius; no un-authorised reaches; bounded cost of leaked sandboxes.

The next chapter pins the capability shape — syscall policy, egress, filesystem — that converts a substrate choice into a defensible per-call policy.
