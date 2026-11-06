# Resources — mod-404 (Sandboxed Code-Execution Plane for Agents)

All references on this page are primary / authoritative public sources at the time of writing (2026-11-06). Each resource is tagged with the chapter(s) it supports; several are cited across more than one. Prefer the primary source over any blog summary — kernel-level, substrate-level, and vendor-level details drift fast in 2026 and secondary sources lag.

## Chapter 01 — Substrates and the Agent Threat Model

### OWASP threat models
- **OWASP Top 10 for LLM Applications**: <https://genai.owasp.org/llm-top-10/>
- **OWASP Agentic AI — Threats and Mitigations (Agentic Security Initiative)**: <https://genai.owasp.org/initiatives/#agentic-security-initiative>

### Firecracker
- **Firecracker — home**: <https://firecracker-microvm.github.io/>
- **Firecracker — design document**: <https://github.com/firecracker-microvm/firecracker/blob/main/docs/design.md>
- **Firecracker — getting started**: <https://github.com/firecracker-microvm/firecracker/blob/main/docs/getting-started.md>
- **Firecracker — security**: <https://github.com/firecracker-microvm/firecracker/blob/main/docs/security.md>
- **AWS Lambda + Fargate background (Firecracker in production)**: <https://aws.amazon.com/blogs/aws/firecracker-lightweight-virtualization-for-serverless-computing/>

### gVisor
- **gVisor — home**: <https://gvisor.dev/>
- **gVisor — architecture guide**: <https://gvisor.dev/docs/architecture_guide/>
- **gVisor — Linux syscall compatibility (amd64)**: <https://gvisor.dev/docs/user_guide/compatibility/linux/amd64/>
- **gVisor — security model**: <https://gvisor.dev/docs/architecture_guide/security/>
- **Netstack (gVisor's userspace network stack)**: <https://github.com/google/gvisor/tree/master/pkg/tcpip>

### Kata Containers
- **Kata Containers — home**: <https://katacontainers.io/>
- **Kata Containers — documentation**: <https://github.com/kata-containers/kata-containers/tree/main/docs>
- **Kata Containers — architecture**: <https://github.com/kata-containers/kata-containers/blob/main/docs/design/architecture/README.md>
- **Confidential Containers (CNCF project, Kata-adjacent)**: <https://confidentialcontainers.org/>

### WebAssembly and Pyodide
- **WebAssembly — home**: <https://webassembly.org/>
- **WASI — home**: <https://wasi.dev/>
- **WASI specification**: <https://github.com/WebAssembly/WASI>
- **Wasmtime — home**: <https://wasmtime.dev/>
- **Wasmtime — security**: <https://docs.wasmtime.dev/security.html>
- **Wasmer**: <https://wasmer.io/>
- **WasmEdge**: <https://wasmedge.org/>
- **Pyodide**: <https://pyodide.org/>
- **Shopify Functions (production WASM reference)**: <https://shopify.dev/docs/apps/build/functions>
- **Cloudflare Workers runtime (production WASM / isolate reference)**: <https://developers.cloudflare.com/workers/runtime-apis/>
- **Fastly Compute@Edge (production WASM reference)**: <https://www.fastly.com/documentation/guides/compute/>

## Chapter 02 — Syscall Policy, Egress Rules, and Filesystem Capability

### Seccomp and syscall policy
- **Linux kernel — seccomp-bpf documentation**: <https://www.kernel.org/doc/html/latest/userspace-api/seccomp_filter.html>
- **libseccomp — man pages**: <https://github.com/seccomp/libseccomp/tree/main/doc/man/man3>
- **Docker — default seccomp profile (JSON)**: <https://github.com/moby/moby/blob/master/profiles/seccomp/default.json>
- **Docker — seccomp overview**: <https://docs.docker.com/engine/security/seccomp/>
- **Kubernetes — seccomp documentation**: <https://kubernetes.io/docs/tutorials/security/seccomp/>
- **systemd — SystemCallFilter unit setting** (an operational cousin of seccomp): <https://www.freedesktop.org/software/systemd/man/latest/systemd.exec.html#SystemCallFilter=>

### Linux capabilities and user namespaces
- **Linux kernel — capabilities(7)**: <https://man7.org/linux/man-pages/man7/capabilities.7.html>
- **Linux kernel — user_namespaces(7)**: <https://man7.org/linux/man-pages/man7/user_namespaces.7.html>
- **OCI runtime spec — Linux-specific configuration**: <https://github.com/opencontainers/runtime-spec/blob/main/config-linux.md>

### Egress policy
- **nftables wiki**: <https://wiki.nftables.org/wiki-nftables/index.php/Main_Page>
- **Cilium — network policies and eBPF**: <https://docs.cilium.io/en/stable/security/policy/>
- **AWS IMDSv2 documentation**: <https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html>
- **GCP VM metadata server documentation**: <https://cloud.google.com/compute/docs/metadata/overview>
- **Azure Instance Metadata Service**: <https://learn.microsoft.com/en-us/azure/virtual-machines/instance-metadata-service>

### Filesystem capability and image hardening
- **Linux kernel — overlayfs documentation**: <https://docs.kernel.org/filesystems/overlayfs.html>
- **Linux kernel — tmpfs documentation**: <https://docs.kernel.org/filesystems/tmpfs.html>
- **virtio-fs (host-to-guest shared filesystem)**: <https://virtio-fs.gitlab.io/>
- **Chainguard Images (minimal, reproducible OCI images)**: <https://www.chainguard.dev/chainguard-images>
- **distroless (minimal container images)**: <https://github.com/GoogleContainerTools/distroless>

### Credential brokering (cross-ref mod-403 chapter 05)
- **HashiCorp Vault — documentation**: <https://developer.hashicorp.com/vault/docs>
- **SPIFFE / SPIRE**: <https://spiffe.io/docs/latest/>

## Chapter 03 — Resource Quotas and Eviction

### Linux resource controllers
- **Linux kernel — cgroup v2 documentation**: <https://docs.kernel.org/admin-guide/cgroup-v2.html>
- **systemd — resource control**: <https://www.freedesktop.org/software/systemd/man/latest/systemd.resource-control.html>
- **prlimit(1)**: <https://man7.org/linux/man-pages/man1/prlimit.1.html>
- **getrlimit / setrlimit**: <https://man7.org/linux/man-pages/man2/getrlimit.2.html>

### VMM-level resource controls
- **Firecracker — rate-limiting API**: <https://github.com/firecracker-microvm/firecracker/blob/main/docs/api_requests/patch-network-interface.md>
- **Firecracker — microVM configuration**: <https://github.com/firecracker-microvm/firecracker/blob/main/docs/api_requests/actions.md>
- **Cloud Hypervisor**: <https://www.cloudhypervisor.org/docs/>

### WASM-runtime controls
- **Wasmtime — fuel (per-instruction accounting)**: <https://docs.wasmtime.dev/examples-fuel.html>
- **Wasmtime — epoch interruption (cooperative pre-emption)**: <https://docs.wasmtime.dev/examples-interrupting-wasm.html>

### Traffic shaping (network quotas)
- **Linux `tc` manpage**: <https://man7.org/linux/man-pages/man8/tc.8.html>
- **Cilium — bandwidth manager**: <https://docs.cilium.io/en/stable/network/kubernetes/bandwidth-manager/>

### SRE background for quota policy
- **Google SRE Book — Handling Overload**: <https://sre.google/sre-book/handling-overload/>
- **Google SRE Workbook — Alerting on SLOs**: <https://sre.google/workbook/alerting-on-slos/>

## Chapter 04 — Managed Fallbacks: E2B, Daytona, Modal

Vendor product scopes and SDK surfaces change quickly. Verify against the vendor's current documentation at the time of reading.

### E2B
- **E2B — home**: <https://e2b.dev/>
- **E2B — documentation**: <https://e2b.dev/docs>
- **E2B — sandbox templates**: <https://e2b.dev/docs/sandbox-template>

### Daytona
- **Daytona — home**: <https://www.daytona.io/>
- **Daytona — documentation**: <https://www.daytona.io/docs>
- **Daytona — getting started**: <https://www.daytona.io/docs/getting-started>

### Modal
- **Modal — home**: <https://modal.com/>
- **Modal — documentation**: <https://modal.com/docs>
- **Modal — Sandbox primitive**: <https://modal.com/docs/guide/sandbox>
- **Modal — Volume (attached storage)**: <https://modal.com/docs/guide/volumes>

### Adjacent / cross-reference (mod-403, mod-406)
- **Model Context Protocol — specification** (the tool-side plane this sandbox plane serves): <https://modelcontextprotocol.io/specification>
- **LiteLLM / LiteLLM Proxy** (adjacent to the model-gateway layer): <https://docs.litellm.ai/>

## Chapter 05 — Snapshot Cold Start and Artifact Staging

### Firecracker snapshotting
- **Firecracker — snapshot support**: <https://github.com/firecracker-microvm/firecracker/blob/main/docs/snapshotting/snapshot-support.md>
- **Firecracker — diff snapshots**: see the snapshotting doc above.
- **userfaultfd(2) (Linux API for on-demand paging used in restore)**: <https://man7.org/linux/man-pages/man2/userfaultfd.2.html>

### OCI image, pull optimisation, and warm pools
- **containerd documentation**: <https://containerd.io/docs/>
- **BuildKit — cache**: <https://github.com/moby/buildkit/blob/master/docs/dev/cache-api.md>
- **SOCI (Seekable OCI) Snapshotter**: <https://github.com/awslabs/soci-snapshotter>
- **eStargz / stargz-snapshotter**: <https://github.com/containerd/stargz-snapshotter>

### Content-addressable storage and artefact integrity
- **IPFS — content addressing**: <https://docs.ipfs.tech/concepts/content-addressing/>
- **Sigstore — Cosign (signing container images and blobs)**: <https://docs.sigstore.dev/>
- **in-toto / SLSA (supply-chain provenance)**: <https://slsa.dev/>

### WASM caching and instance reuse
- **Wasmtime — CLI cache**: <https://docs.wasmtime.dev/cli-cache.html>
- **Pyodide — usage and lifecycle**: <https://pyodide.org/en/stable/usage/>

### SRE / observability for cold-start SLIs
- **OpenTelemetry — traces specification**: <https://opentelemetry.io/docs/specs/otel/trace/>
- **OpenTelemetry — Semantic Conventions for Generative AI**: <https://opentelemetry.io/docs/specs/semconv/gen-ai/>

## Chapter 06 — Sandbox Events as Compliance Evidence

### Immutable / append-only storage primitives
- **AWS S3 — Object Lock**: <https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock-overview.html>
- **GCP Cloud Storage — Bucket Lock**: <https://cloud.google.com/storage/docs/bucket-lock>
- **Azure Blob — immutable storage**: <https://learn.microsoft.com/en-us/azure/storage/blobs/immutable-storage-overview>

### Content-addressable hashing and chain-of-custody
- **RFC 6920 — Naming Things with Hashes**: <https://datatracker.ietf.org/doc/html/rfc6920>
- **RFC 8785 — JSON Canonicalization Scheme** (for deterministic record hashing): <https://datatracker.ietf.org/doc/html/rfc8785>

### Regulatory / standards baselines
- **EU AI Act — Regulation (EU) 2024/1689**: <https://eur-lex.europa.eu/eli/reg/2024/1689/oj>
- **EU AI Act — Article 72 (post-market monitoring)**: in the regulation above; cross-reference any delegated acts in force at your audit cycle. <!-- needs-research: confirm whether specific record-keeping delegated acts under Article 72 have entered into force at your audit cycle. -->
- **GDPR — Regulation (EU) 2016/679** (Article 17 right to erasure): <https://eur-lex.europa.eu/eli/reg/2016/679/oj>
- **ISO/IEC 27001:2022**: <https://www.iso.org/standard/27001>
- **AICPA — Trust Services Criteria (SOC 2)**: <https://www.aicpa-cima.com/resources/landing/system-and-organization-controls-soc-suite-of-services>
- **NIST — AI Risk Management Framework**: <https://www.nist.gov/itl/ai-risk-management-framework>

### PII detection and redaction
- **Microsoft Presidio**: <https://microsoft.github.io/presidio/>
- **AWS Comprehend — PII detection**: <https://docs.aws.amazon.com/comprehend/latest/dg/pii.html>
- **GCP Data Loss Prevention**: <https://cloud.google.com/sensitive-data-protection/docs>

### Observability / trace linkage (cross-ref mod-408)
- **OpenTelemetry — Semantic Conventions for Generative AI**: <https://opentelemetry.io/docs/specs/semconv/gen-ai/>
- **OpenTelemetry — Traces specification**: <https://opentelemetry.io/docs/specs/otel/trace/>

## Secondary / cross-cutting references

- **CNCF — Platform Engineering Maturity Model** (background on the platform-as-a-product shape from mod-401): <https://tag-app-delivery.cncf.io/whitepapers/platform-eng-maturity-model/>
- **Anthropic — Building Effective Agents** (agent-authoring context the plane hosts): <https://www.anthropic.com/research/building-effective-agents>
- **OpenAI — A Practical Guide to Building Agents**: <https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf>
- **Model Context Protocol — specification** (cross-ref mod-403): <https://modelcontextprotocol.io/specification>
- **A2A (Agent-to-Agent) protocol** (preview for mod-409): <https://a2aprotocol.ai/> <!-- needs-research: confirm the canonical spec URL for A2A at your reading time; this has been moving between hosts. -->
- **JOB_REQUIREMENTS.md** (at the track root): the normalised list of citations the curriculum is written against.

## Notes on using these references

- Every entry on this page is a **primary** or **authoritative-public** source. If the source has a version tag — kernel release, substrate commit, vendor doc dated — read the version that matches the one you deploy against. Firecracker's snapshot semantics, gVisor's syscall matrix, and the vendor product scopes have all moved across releases in 2025–2026.
- The OWASP LLM Top 10 and OWASP Agentic AI lists reshuffle across major revisions. Before quoting a specific item ID (`LLMxx`), verify the current revision.
- The EU AI Act's record-keeping requirements under Article 72 are refined by delegated acts that are still appearing. Before quoting a specific retention horizon in a compliance memo, verify against the latest delegated act.
- Mark `<!-- needs-research: ... -->` in any derived artefact where a citation on this page does not match the version you are shipping against. The flag blocks auto-merge; it is cheaper than silently guessing.
