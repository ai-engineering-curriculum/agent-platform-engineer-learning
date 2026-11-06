# Managed Fallbacks — E2B, Daytona, and Modal Sandboxes

There are three honest reasons to put a managed sandbox vendor behind your gateway rather than running every call on self-hosted substrate:

1. **Capacity you have not built yet.** You are launching the plane; the self-hosted Firecracker / gVisor fleet is weeks away; product teams need sandboxes now.
2. **Spikes, outages, geographies.** Your self-hosted fleet is right-sized for steady state. A managed vendor absorbs demand when a product team ships a feature that triples your call volume at 09:00 on a Monday, or when a region you do not run in lights up, or when the primary substrate is in a degraded state.
3. **Workloads you do not want to host.** A code interpreter with full Jupyter, a Playwright-headful browser automation sandbox, a shell with 8 GB RAM and 30-second life for arbitrary developer prompts. Building and operating these in-house is weeks of work for a workload the vendor already runs at scale.

What the vendors are *not* for is replacing the gateway. They sit *behind* your sandbox plane, exposed through the same contract the runtime already uses. The plane keeps its three obligations (chapter 01) and its capability model (chapter 02). The vendor is an implementation of the sandbox-create verb, not a product the runtime talks to directly.

This chapter pins how to integrate three concrete managed sandboxes — E2B, Daytona, and Modal Sandboxes — as fallbacks or primary routes behind the gateway.

## The vendor landscape, at the shape a platform engineer cares about

Three vendors cover most of the design space agent platforms see today. The vendor landscape moves fast; verify current scopes and SDKs at your onboarding. <!-- needs-research: vendor product scopes and SDK surface change quickly; re-verify E2B, Daytona, and Modal Sandboxes at the time of reading. -->

### E2B — [`e2b.dev`](https://e2b.dev/)

[E2B](https://e2b.dev/docs) builds a hosted secure sandbox service aimed at AI-agent code execution. Each sandbox is an isolated microVM (Firecracker under the hood, by E2B's own description) with a configurable template: a base image with the packages, binaries, and ports the tool needs. The API is a small SDK (Python, TypeScript) that creates sandboxes, uploads files, runs commands or language-specific `runCode` calls, streams output, and tears sandboxes down. [E2B templates](https://e2b.dev/docs/sandbox-template) let you ship your own base image. E2B documents its isolation model (Firecracker + custom runtime) and offers persistent and ephemeral modes.

E2B's shape:

- **Isolation:** Firecracker microVMs per sandbox.
- **Latency:** sub-second to small-digit-second sandbox creation, depending on template size and region.
- **Interface:** HTTPS API + SDK; file transfer, command execution, port forwarding, timed sandboxes.
- **Observability:** per-sandbox logs the SDK can stream; metadata is per-session.
- **When it fits:** code-interpreter tier, developer-prompt tier, anything that wants a short-lived full Linux shell with hardware-virt isolation and you do not want to operate Firecracker yourself.

### Daytona — [`daytona.io`](https://www.daytona.io/)

[Daytona](https://www.daytona.io/docs) is a development-environment platform that has an ambient story for agent-driven workloads: on-demand workspaces with a Git source, pre-installed devcontainers, SDK access from agent code. [Daytona Sandboxes](https://www.daytona.io/docs/getting-started) and the agent-facing SDK expose creation, file I/O, and command execution similarly to E2B but with workspace-shaped defaults (persistent until explicitly destroyed, devcontainer-based images, VS Code attachment surfaces).

Daytona's shape:

- **Isolation:** workspace-shaped VMs; self-hosted and hosted options exist.
- **Latency:** several seconds for a workspace; hot-pool configurations drop it lower.
- **Interface:** SDK + CLI; Git-aware workspace lifecycle.
- **Observability:** workspace events, resource usage per workspace.
- **When it fits:** agent workloads that are workspace-shaped — running test suites, editing a code tree, invoking devcontainer-provided tooling across multiple steps of a session.

### Modal Sandboxes — [`modal.com`](https://modal.com/)

[Modal](https://modal.com/docs) is a serverless compute platform whose [Sandbox primitive](https://modal.com/docs/guide/sandbox) exposes short-lived, programmatically created sandboxes with the language-aware hooks Modal already provides (Python SDK, GPU attach, large data volumes). Modal Sandboxes let you run arbitrary commands or snippets on managed infrastructure with the same quota and attached-storage model Modal uses for its other jobs.

Modal's shape:

- **Isolation:** container/VM hybrid, Modal-managed; GPU-attached sandboxes available.
- **Latency:** varies — several seconds in common cases, longer with GPU attach or large image pull.
- **Interface:** Python-first SDK; Modal Volume attachment; GPU and large-memory classes.
- **Observability:** per-sandbox logs and metrics in Modal's dashboard.
- **When it fits:** sandboxes that need GPU, large data, or Modal-shaped storage; sandboxes that benefit from Modal's compute marketplace (specific instance classes).

A short comparison:

| Dimension | E2B | Daytona | Modal Sandboxes |
|---|---|---|---|
| Primary shape | Short-lived microVM for AI code execution | Workspace / devcontainer | Short-lived sandbox with GPU / large-memory options |
| Headline isolation | Firecracker microVM (per E2B docs) | VM / container | Modal-managed sandbox |
| Latency profile | Sub-second to few seconds | Few seconds | Few seconds; longer with GPU |
| Persistence options | Ephemeral + persistent | Persistent by default | Ephemeral with Modal Volume attach |
| GPU support | Not standard | Not standard | Yes (named classes) |
| Interface | REST + Python / TS SDK | SDK + CLI | Python SDK |

For every claim, the vendor's current docs at your onboarding are the authority. Marketing terminology on vendor front pages compresses details you need.

## Where the vendor plugs into your plane

The sandbox gateway is already the single thing the runtime (mod-402) and the MCP gateway (mod-403) talk to. The managed vendor is an additional *provider implementation* behind that same interface.

```
Runtime / MCP gateway
        │
        ▼
Sandbox Gateway (your plane)
   │   │   │
   │   │   └──▶ SelfHosted Firecracker/gVisor/Kata provider
   │   └─────▶ WASM / Pyodide provider
   └─────────▶ Managed provider: {e2b | daytona | modal}
                   │
                   ▼
                Vendor API
```

Three things keep this architecture honest:

- **The gateway's contract does not change.** Callers pass a sandbox request (chapter 02's capability bundle). The provider implementation is a strategy; the caller does not pick it, the gateway does.
- **The capability bundle is translated into the vendor's shape at the provider boundary.** `resources.cpu_m: 1000` becomes an E2B `template.cpu` or a Modal `cpu=1.0`; `egress.allow` becomes whatever the vendor's egress configuration permits. Mismatches are handled explicitly, not silently.
- **The evidence stream (chapter 06) still gets attributed per call.** The vendor's own events are ingested, correlated with the session, and written to the same compliance pipeline.

The provider's responsibilities are narrow:

1. **Translate the capability bundle.** Produce the vendor-native request, carrying every field that maps, failing admission on fields that do not (chapter 02's `syscall_profile` does not map to Modal; the gateway decides what to do about that).
2. **Hold credentials.** The vendor API key lives in the broker (same pattern as mod-403 chapter 05). The provider fetches it per call and never caches it in long-lived memory.
3. **Convert vendor responses to the gateway's shape.** Status, logs, outputs, errors.
4. **Emit evidence events.** For every vendor call that corresponds to a `sandbox_created`, `tool_invoked`, `sandbox_destroyed`, `cap_hit`, `egress_denied` platform event, the provider writes the equivalent evidence record with vendor-specific fields in `provider_details`.

## Building the provider abstraction

A minimal interface — the kind you ship in Go or Python — captures the sandbox plane's view:

```python
from typing import Protocol

class SandboxProvider(Protocol):
    name: str  # "firecracker-primary" | "gvisor-high-density" | "e2b" | "daytona" | "modal"

    async def create(self, req: SandboxRequest) -> SandboxHandle: ...
    async def invoke(self, handle: SandboxHandle, call: ToolCall) -> ToolResult: ...
    async def destroy(self, handle: SandboxHandle, reason: str) -> None: ...
    async def health(self) -> ProviderHealth: ...
    def translate(self, req: SandboxRequest) -> ProviderNativeRequest: ...
    def coverage(self) -> ProviderCoverage: ...
```

`coverage()` is the key method. It returns which parts of the gateway's `SandboxRequest` the provider implements and which it cannot. The provider explicitly reports which syscall-profile features, egress-policy constraints, filesystem-mount modes, and resource dimensions it can enforce. The routing layer (below) consults coverage *before* it picks a provider — a tool whose policy requires `rw-persist` for a workspace path cannot land on a provider whose coverage reports no persistent-mount support.

A provider that lies about its coverage is the hole in the plane; the admission tests in chapter 06 verify coverage claims against the vendor's actual behaviour.

## Routing — when to pick which provider

The sandbox gateway makes the provider choice per call. The inputs are:

- **Policy plane decision.** The policy plane (mod-405) may stamp `provider_requirement` on the call: `self_hosted_only` for compliance-sensitive workloads, `no_third_country_vendor` for data-residency constraints.
- **Registry metadata.** The tool's registry entry may stamp preferences: `preferred_provider: wasm-compact` for a small-fast tool.
- **Fleet state.** Self-hosted capacity, queue depth, hot-pool availability.
- **Provider health.** Each provider's `health()` result. If the primary is degraded, the secondary is picked.
- **Call shape.** A call requesting GPU goes to a GPU-capable provider or fails; a call with tight cold-start requirements goes to a hot-pool-backed provider.

A reasonable decision shape, expressed as an ordered list:

1. **Hard constraints first.** Policy, residency, forbidden-provider lists. These are filters, not scores.
2. **Coverage.** Discard providers whose `coverage()` cannot serve the capability bundle.
3. **Capacity / health.** Discard providers that are saturated or degraded.
4. **Preferred-provider stamp.** If present and still viable, pick it.
5. **Fleet-level routing policy.** Self-hosted first until a threshold (steady-state budget), managed fallback above.
6. **Tie-breaker.** Latency / cost / tenant affinity — your SRE plane (mod-410) tunes this.

The decision, with its reasons, is attached to the evidence record. "Why did this call land on E2B rather than self-hosted?" is a question the on-call engineer and the compliance audit both ask.

## Egress and credential handling with managed vendors

The capability model from chapter 02 does not evaporate when you cross the vendor boundary. Two translation points matter.

### Egress

Managed vendors offer varying egress controls. Three concrete postures:

- **Vendor supports allowlist egress at sandbox creation time.** Translate the capability bundle's `egress.allow` list directly. Verify behaviour with a conformance test per vendor per policy — the admission test in chapter 06 confirms the vendor's egress enforcement matches what the capability bundle asked for.
- **Vendor supports only broad-ranged egress controls.** Treat the vendor's sandbox as sitting behind your own egress proxy: route the sandbox's outbound traffic through a platform-controlled HTTP or SOCKS proxy that enforces the real allowlist. The vendor's own egress-permissive stance is irrelevant if every outbound call goes through your proxy.
- **Vendor supports no fine-grained egress.** Only allow this vendor for tiers where egress is already restricted upstream (e.g., tools that only talk to the model gateway, which is already fronting an allowlist).

The capability bundle's `block_unconditional` list (cloud metadata, platform internals, link-local) must be enforced on every vendor regardless of their native controls, by the same proxy pattern.

### Credentials

Three rules:

1. **No shared vendor API key across tenants.** Use the vendor's workspace / project / team primitives to separate tenants where possible; where not, mint a per-tenant short-lived vendor-API session.
2. **No long-lived credentials delivered into the vendor's sandbox.** Pass short-lived tokens into the sandbox at creation time (via the vendor's secret injection or environment variable), scoped to the specific call. The token lifetime is bounded by the sandbox's declared `duration_s`.
3. **No vendor API key in agent-reachable paths.** The vendor API key is the gateway's secret; a compromised sandbox should not be able to spawn further sandboxes. Treat the vendor API key as a control-plane credential.

## Degraded-mode contract

Vendors degrade. Design the plane's response up-front:

- **Vendor rate-limit returns 429.** Back off, surface the delay to the runtime (mod-402), continue retrying with jitter, open a circuit after a threshold. Record `provider_backpressure` per attempt.
- **Vendor region outage.** Fail over to another configured vendor region if available; otherwise shift traffic to the other vendors in the routing order; otherwise shift back to self-hosted at degraded quality.
- **Vendor-wide outage.** The gateway's degraded-mode policy fires — some tiers may become unavailable, and the gateway surfaces structured errors to the runtime with the reason. The policy plane and the SRE plane (mod-410) share the outage event.
- **Vendor silent failures.** Conformance probes (same shape as mod-403 chapter 04's MCP conformance) run against each vendor continuously. A vendor that passes probes but is quietly dropping egress destinations is the thing that bites in post-mortems; probe egress enforcement, not just reachability.

Design the gateway so that failing back from vendor to self-hosted is a *tested* path, not a belief. A quarterly drill in which the vendor path is forcibly broken is cheaper than discovering the failback is a myth at 03:00.

## Build-vs-buy anchors

The build-vs-buy decision here is the same shape mod-403 chapter 06 covered for tool vendors — portfolio framing, Tier-1 to Tier-3 triggers, ten-question evaluation checklist — applied to sandboxing. The platform-engineering position is:

- **Own the gateway.** The contract the runtime and tool-gateway see belongs to you.
- **Own enough substrate to not depend on any one vendor.** Even if the primary substrate is managed, a self-hosted baseline in at least one tier keeps you from being lifted by vendor pricing or outages.
- **Buy capacity, GPU, and specialised shapes.** Running a GPU-attached sandbox pool for one tenant's weekly workload is a worse use of platform-team time than paying Modal.
- **Buy *behind* the gateway, never in front of it.** No product team's agent code imports `@e2b/sdk` directly; they import the paved-road SDK (mod-412), which talks to the gateway, which talks to E2B.

A vendor whose scope the plane hands over without wrapping is a vendor that owns your platform. The gateway is the thing that keeps the platform yours.

## Summary

- Managed sandbox vendors sit behind the sandbox gateway as additional provider implementations, not alongside it. The capability model and the gateway contract do not change.
- E2B, Daytona, and Modal Sandboxes cover most of the design space agent platforms need: AI-code-execution microVMs, workspace-shaped persistent environments, GPU / large-memory sandboxes.
- Every provider implements a narrow interface: `create`, `invoke`, `destroy`, `health`, `translate`, and — critically — `coverage`. Routing rejects a provider whose coverage cannot serve the capability bundle.
- Egress and credentials are enforced by the plane, even behind a managed vendor: route through a platform egress proxy if the vendor cannot do allowlists; mint short-lived per-call credentials; never share vendor API keys across tenants.
- Design and *test* the degraded-mode path from vendor back to self-hosted. Vendor outages are expected; losing the failback is the thing that burns you.
- Build the gateway; own enough substrate to not depend on any one vendor; buy capacity, GPU, and specialised shapes behind the gateway.

The next chapter handles the other primary way the plane earns its latency budget: snapshot-based cold starts and artifact staging.
