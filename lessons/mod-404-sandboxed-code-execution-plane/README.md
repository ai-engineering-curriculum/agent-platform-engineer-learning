# mod-404 — Sandboxed Code-Execution Plane for Agents

**Estimated effort:** 16 hours · **Position in track:** fourth module (after [`mod-401`](../mod-401-agent-platform-position-and-paved-road-philosophy), [`mod-402`](../mod-402-agent-runtime-service-and-durable-execution), [`mod-403`](../mod-403-mcp-registry-and-tool-gateway)).

Tools that only read bits are reference tools. Any non-trivial agent eventually writes to disk, forks a subprocess, or executes code the model just generated — and the platform that hosts that execution is the platform that absorbs the blast radius when the input turns adversarial. The sandboxed code-execution plane is where a platform decides *how much* an agent can actually do with a single invocation, and *how little* a compromised invocation can reach. By the end of this module you can pick a sandbox substrate against the OWASP LLM Top 10 and OWASP Agentic AI threat models at engineering depth, engineer the syscall policy / egress rules / filesystem shape / resource quotas for the per-tool capability bundle, integrate E2B / Daytona / Modal as managed fallbacks behind the plane without surrendering your contract, make cold starts land in single-digit seconds via snapshotting and artefact staging, and emit the evidence stream that lets compliance auditors and incident responders answer every question they ask about a sandbox that ran yesterday.

## Learning objectives

- Compare multi-tenant sandbox substrates — Firecracker microVMs, gVisor application kernel, Kata lightweight VMs, WASM / Pyodide — against the threat model from OWASP LLM Top 10 and OWASP Agentic AI Threats and Mitigations.
- Engineer the syscall policy, egress rules, filesystem-capability shape, and resource quotas (CPU, memory, disk, network, file handles) of a per-tenant sandbox fleet; author the eviction policy for leaked sandboxes.
- Integrate E2B, Daytona, and Modal Sandboxes as managed fallbacks behind the platform's sandbox gateway.
- Implement snapshot-based cold-start acceleration and artifact-staging pipelines so an agent's workspace comes up in single-digit seconds.
- Instrument sandbox events against the platform's compliance evidence stream — syscall violations, egress denials, resource-cap hits, OOM events.

## Chapters

1. [`01-substrates-and-the-agent-threat-model.md`](01-substrates-and-the-agent-threat-model.md) — The slice this plane owns, the OWASP-grounded threat model, Firecracker / gVisor / Kata / WASM compared across isolation, latency, density, and compatibility, substrate-to-threat mapping, and the three non-negotiable sandbox-plane obligations.
2. [`02-syscall-egress-and-filesystem-capability.md`](02-syscall-egress-and-filesystem-capability.md) — Capability model on five axes, deny-by-default seccomp-bpf profiles (and the WASI equivalent), egress allowlists with unconditional blocks (IMDS, platform internals, link-local, arbitrary DNS), read-only rootfs + declared `/workspace`, brokered per-call credentials, and the capability-bundle interface.
3. [`03-resource-quotas-and-eviction.md`](03-resource-quotas-and-eviction.md) — Five quota dimensions (CPU, memory, disk, network, process state), tenant-level and fleet-level budgets, the leak-eviction ladder (grace → soft cancel → hard kill → reclamation → attribution), fleet-pressure eviction, and the evidence trail every quota decision must leave.
4. [`04-managed-fallbacks-e2b-daytona-modal.md`](04-managed-fallbacks-e2b-daytona-modal.md) — E2B, Daytona, and Modal Sandboxes as vendors behind the gateway, the provider interface (with `coverage()`), the routing decision shape, egress and credential handling through vendors, the degraded-mode contract, and the build-vs-buy posture.
5. [`05-snapshot-cold-start-and-artifact-staging.md`](05-snapshot-cold-start-and-artifact-staging.md) — Cold start decomposed into six stages, snapshot patterns per substrate (Firecracker diff snapshots, gVisor warm pools, Kata VMM-dependent, WASM AOT caching + instance reuse), the warm-pool contract, the resolve/fetch/place/verify artefact-staging pipeline, credential staging, and the `sandbox_time_to_tool_call_p99` SLI.
6. [`06-sandbox-events-as-compliance-evidence.md`](06-sandbox-events-as-compliance-evidence.md) — The six-event catalogue, the common envelope (`trace_id`, `policy_decision_id`, `capability_bundle_hash`, `received_at`), immutable + content-addressable storage, retention horizons, GDPR Article 17 vs. append-only, PII payload modes, pipeline shape, and cross-plane linkage.

## Exercises

Each exercise turns one or more chapters into an artefact you keep and reuse — in the rest of the module, in projects 401 and 404, and in later modules (mod-405, mod-407, mod-408, mod-410).

- [`exercises/exercise-01-firecracker-vs-gvisor-threat-model-tradeoff.md`](exercises/exercise-01-firecracker-vs-gvisor-threat-model-tradeoff.md) — Firecracker vs. gVisor threat-model trade-off memo. Pairs with chapter 01. Feeds exercises 02 and 03 and project 401.
- [`exercises/exercise-02-syscall-policy-and-egress-rules-authoring.md`](exercises/exercise-02-syscall-policy-and-egress-rules-authoring.md) — Syscall policy and egress-rule authoring for a concrete tool. Pairs with chapter 02. Feeds exercise 03 and projects 401 and 404.
- [`exercises/exercise-03-resource-quota-and-eviction-policy-design.md`](exercises/exercise-03-resource-quota-and-eviction-policy-design.md) — Resource-quota profile and eviction-ladder design. Pairs with chapter 03. Feeds project 401 and mod-410.
- [`exercises/exercise-04-snapshot-based-cold-start-prototype.md`](exercises/exercise-04-snapshot-based-cold-start-prototype.md) — Snapshot-based cold-start prototype with the artefact-staging pipeline and the cold-start SLI. Pairs with chapter 05. Feeds project 401 and mod-408.

## Resources

External references — primary sources only — are catalogued in [`resources.md`](resources.md).

## How to approach this module

1. Read the six chapters in order. The capability model (02) presumes the substrate frame (01); quotas and eviction (03) presume the capability model; managed fallbacks (04) presume you can describe your self-hosted contract; snapshot and staging (05) presume both the capability model and the quota shape; the evidence stream (06) picks up events every earlier chapter emits.
2. Do exercise 01 before reading chapter 04 — your Firecracker-vs-gVisor memo is what argues *why* a vendor fallback is placed behind one tier and not another. Exercises 02 and 03 are easier once you have taken a substrate position in 01.
3. Read the OWASP LLM Top 10 and OWASP Agentic AI Threats and Mitigations before chapter 01. The chapter writes against them; a skim will not be enough to argue substrate choices back against your security partner.
4. For the Firecracker and gVisor references, prefer the project docs over blog posts — the Firecracker snapshotting doc, the gVisor compatibility matrix, the Kata containers architecture guide. API-level and security-critical details drift fast in 2026.
5. Mark `<!-- needs-research: ... -->` on any claim you cannot verify against the current substrate / vendor / regulatory text. The MCP-gateway module uses the same flag; it blocks auto-merge, and that is cheaper than silently guessing.

## Where the handoffs land

- **`agentic-ai-engineer` (level 30)** — writes the tool code that runs inside your sandboxes. Your contract to them is the paved-road SDK (mod-412), a capability-bundle template for their tool class, and the admission-test suite that tells them if their code will pass.
- **`agentic-safety-engineer` (level 40) and `ai-infra-security` peers** — own the deep research on sandbox-escape primitives, prompt-injection-to-sandbox-escape chains, and threat-model evolution. You operationalise their findings into tighter seccomp profiles, narrower egress lists, snapshot-build admission gates, and evidence queries. Bring them your event volumes; they bring back threat models.
- **`ai-infra-sre` (peer) and your own future mod-410 work** — consume the SLIs produced in chapters 03, 05, and 06 (`sandbox_reclamation_time_p99`, `sandbox_time_to_tool_call_p99`, `evidence_lag`) and the eviction cause-code histogram. Their SLO proposals land as your admission ceilings.
- **`senior-agentic-ai-engineer` (peer, level 40)** — ships product-side agents that call tools that run in your sandboxes. They care that cold starts are fast, tool calls do not mysteriously fail with cryptic errors, and compliance evidence for their customers' audits comes out of the plane rather than being re-invented per team.
