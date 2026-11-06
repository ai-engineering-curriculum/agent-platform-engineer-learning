# Syscall Policy, Egress Rules, and Filesystem Capability

A substrate choice (chapter 01) decides the *boundary* of a sandbox; it does not decide what the sandbox is allowed to *do*. Default-open containers with a strong boundary are still catastrophic — they read your cloud metadata endpoint, write to any mounted volume, and open outbound connections to attacker-controlled hosts, all without crossing the boundary at all. The capability shape — which syscalls are permitted, which network destinations are reachable, which filesystem paths are visible, and which credentials flow in — is what converts isolation into a defensible contract.

This chapter pins that shape. It assumes you have read chapter 01 and have an opinion about which substrate is your primary tier.

## The capability model, in one frame

Treat every sandbox as the intersection of **four constrained axes** and **one authenticated inbound surface**. If any axis is default-open, your sandbox leaks on that axis regardless of how strong the others are. The axes are:

1. **Syscall surface** — which host-visible syscalls the sandbox can execute at all.
2. **Network egress** — which off-host destinations the sandbox can reach, by DNS name, IP range, port, and protocol.
3. **Filesystem view** — which host paths the sandbox can see, read, write, or execute from.
4. **Resource ceiling** — CPU, memory, disk, pids, file handles, bandwidth. (Covered in chapter 03.)

The inbound surface:

5. **Capability tokens / credentials.** What identity and scopes the sandbox carries so that any outbound call it is allowed to make is attributable to the session that caused it.

Every call into the sandbox plane specifies a *capability bundle* — a concrete selection along each axis — that the tool's registry entry (mod-403) and the policy plane's decision (mod-405) jointly produced. The sandbox plane enforces the bundle; it does not negotiate it.

## Syscall policy

### The shape your substrate gives you

Each substrate from chapter 01 exposes a different syscall-shaped primitive:

- **Firecracker / Kata (hardware VMs):** The host-kernel syscall surface the guest can reach is exactly what the VMM exposes through its device model and MMIO plus any paravirtualised interfaces. The *guest kernel* then presents its own full syscall surface to the guest userspace. You harden the guest kernel (strip unused modules, disable `ptrace` in production images, strip `kexec`) and apply seccomp-bpf *inside the guest* to the workload.
- **gVisor:** The sandbox's syscall surface is what the gVisor Sentry implements plus the small set of host syscalls Sentry uses underneath (file I/O over Gofer, timekeeping, memory). gVisor publishes a [per-architecture syscall compatibility matrix](https://gvisor.dev/docs/user_guide/compatibility/linux/amd64/) — your policy is a filter *over* that matrix, not independent of it.
- **WASM / Pyodide:** There is no host syscall surface at all. Instead, WASI defines a small, deliberately shaped set of host functions (`fd_read`, `fd_write`, `sock_send`, clocks, random). The host program chooses which of those to import into each module. "Syscall policy" in WASM is *capability granting*: the module simply cannot call what the host did not bind.

The policy you ship on top is a seccomp-bpf profile (for VMs or gVisor-contained workloads), a WASI capability binding (for WASM), and — for every substrate — a declarative description of the allowed operations your platform code generates from the tool's registry entry.

### Writing a seccomp-bpf profile for sandboxed code execution

For Linux-guest substrates the policy lingua franca is [seccomp-bpf](https://www.kernel.org/doc/html/latest/userspace-api/seccomp_filter.html). Docker's [default seccomp profile](https://docs.docker.com/engine/security/seccomp/) is the readable reference; it blocks roughly 50 of the ~400 Linux syscalls by default and is a useful *starting template*. It is a starting template, not a ship target: your workloads are narrower than *all containers*, and you can afford to be tighter.

A workable default for *code-execution* sandboxes (i.e. sandboxes hosting arbitrary short-lived user or model-generated code, not full services) is:

- **Allow:** file I/O (`open*`, `read`, `write`, `close`, `fstat`, `lseek`, `pread64`, `pwrite64`), process-lifetime (`fork`/`clone` with a bounded `CLONE_*` mask, `execve` for specific paths, `exit`, `wait`), memory (`mmap`, `munmap`, `mprotect`, `brk`), signals (`rt_sigaction`, `rt_sigprocmask`), sockets (`socket`, `connect`, `recvfrom`, `sendto` — with egress policy enforced at the network layer, not here).
- **Deny:** everything kernel-adjacent that the workload does not plausibly need: `kexec_load`, `init_module`, `delete_module`, `swapon`, `swapoff`, `mount`, `umount2`, `reboot`, `ptrace`, `bpf` (unless you are building your own), `perf_event_open`, `keyctl`, `add_key`, `request_key`, `userfaultfd`, `clone3` with `CLONE_NEWUSER` unless unprivileged userns is your explicit threat model.
- **Audit and log:** anything the workload invokes that is not on either list at first. Shipping in `SCMP_ACT_LOG` mode (see the [libseccomp docs](https://github.com/seccomp/libseccomp/blob/main/doc/man/man3/seccomp_rule_add.3)) before flipping to `SCMP_ACT_ERRNO` is how you discover the real tail your workloads need.

```yaml
# Illustrative, not a ship target. Treat this as a shape you own and
# narrow against your workload baseline — see the "Tightening" subsection.
defaultAction: SCMP_ACT_ERRNO    # deny-by-default
architectures:
  - SCMP_ARCH_X86_64
syscalls:
  - names: [read, write, open, openat, close, fstat, lseek, mmap, munmap,
            mprotect, brk, rt_sigaction, rt_sigprocmask, rt_sigreturn,
            ioctl, pread64, pwrite64, readv, writev, access, faccessat2,
            pipe, pipe2, dup, dup2, dup3, poll, epoll_create1, epoll_wait,
            epoll_ctl, exit, exit_group, wait4, waitid, clock_gettime,
            clock_gettime64, nanosleep, getpid, gettid, getuid, getgid,
            getcwd, chdir, fchdir, stat, lstat, newfstatat, statx, uname,
            getdents64, fcntl, fsync, fdatasync, prlimit64]
    action: SCMP_ACT_ALLOW
  - names: [socket, connect, bind, listen, accept, accept4, sendto,
            recvfrom, sendmsg, recvmsg, shutdown, getsockname, getpeername,
            setsockopt, getsockopt]
    action: SCMP_ACT_ALLOW
    # network egress is enforced at the firewall layer, not here
  - names: [clone, clone3, execve, execveat, fork, vfork]
    action: SCMP_ACT_ALLOW
    # process-lifetime gating happens via CLONE_* masks + pid quota (chapter 03)
  - names: [bpf, kexec_load, init_module, delete_module, mount, umount2,
            swapon, swapoff, reboot, ptrace, perf_event_open, keyctl,
            add_key, request_key, userfaultfd, name_to_handle_at,
            open_by_handle_at, pivot_root, setns, unshare]
    action: SCMP_ACT_KILL_PROCESS
    # KILL, not ERRNO — these are "do not continue" signals
```

The `SCMP_ACT_KILL_PROCESS` block is where your defence-in-depth lives: if the workload tries anything on this list, do not let the process keep running. Kill it, record the violation to the evidence stream (chapter 06), and surface the event to the policy plane so the pattern can inform future admission decisions.

### Tightening past the baseline

The template above is a *fleet default*. To get tighter, instrument the fleet first:

- Run every tool in `SCMP_ACT_LOG` mode in staging for one to two weeks.
- Collect the syscall histogram per tool version.
- For tools with a stable tail (data transforms, HTTP clients, SDK wrappers), ship a tool-specific seccomp profile that is the observed set plus a short justified-additions list.
- For tools whose workload is genuinely arbitrary (REPLs for user code), ship the broader default.

Carry the tightened profile in the tool's registry entry (mod-403, `ServerVersion.sandbox_profile`) and *pin* it to the version. A new version proposing a looser profile is a review-grade admission event.

### A note on `io_uring`, `userfaultfd`, and the moving list of "controversial" syscalls

The set of syscalls that Linux upstream has flagged as attack-surface-sensitive moves. `io_uring` has been enabled, disabled, re-enabled, and re-restricted across container platforms repeatedly as new exploits appear and new mitigations land; the Kubernetes default container runtime's position on it has shifted. `userfaultfd`, `clone3`, and `bpf` have similar histories. <!-- needs-research: at your onboarding, review the latest CRI / kernel / seccomp changelog for the current posture on io_uring, userfaultfd, bpf, and clone3; the defaults here may have moved. -->

The posture your plane takes is: deny by default, allow by evidence. If a workload needs `io_uring`, require a registry-level opt-in with a documented rationale and automatic review by the security partner (chapter 06 and mod-405).

## Egress rules

Network egress is where a sandbox escape becomes a *successful* sandbox escape. A compromised workload with no egress is a localised failure; a compromised workload that can reach `169.254.169.254` or `s3.amazonaws.com/attacker-bucket` is an incident.

### Deny-by-default, allow by destination

Design egress as a *destination allowlist* per tool. The allowlist is part of the tool's registry entry; it becomes concrete firewall policy at sandbox-creation time. The baseline for any new tool is:

- **No egress.** The tool starts with no network reachability at all. A tool that needs the internet declares it.
- **Localhost / in-sandbox loopback allowed.** For tool stacks that bind local HTTP servers as part of their own implementation.
- **Explicit destinations.** Each permitted destination is an `(fqdn|cidr, port, protocol)` triple. Prefer FQDNs over CIDRs where the DNS is stable; prefer CIDRs where the FQDN has an excessively broad or attacker-controllable DNS response.
- **TLS required for anything off-host.** Plaintext egress to non-localhost destinations is refused.

### What to block unconditionally

Four categories are blocked regardless of what the tool declares:

- **Cloud instance-metadata endpoints.** `169.254.169.254` (the IMDS address used across AWS / GCP / Azure / OpenStack) is the single most exfiltration-productive destination on the internet for cloud-hosted sandboxes. A sandboxed workload has no legitimate reason to query IMDS. Block the IPv4 address *and* the IPv6 equivalent (`fd00:ec2::254` and other provider-specific addresses) at the sandbox's network layer, not in application code. <!-- needs-research: confirm the current IMDS and alternative-IMDS endpoints across AWS, GCP, Azure, and other cloud providers; the IPv6 and alt addresses have evolved. See AWS IMDSv2 docs: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html -->
- **Platform internal CIDRs the sandbox has no business reaching.** The platform's own admin APIs, Kubernetes API server (`kubernetes.default.svc`), internal service meshes, databases, Redis / Valkey, control-plane message buses. These are enumerated per platform; the sandbox's network namespace reaches none of them unless a specific tool entry both declares the destination and passes policy-plane review.
- **Link-local and private ranges unless explicitly carved out.** `169.254.0.0/16`, `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `fc00::/7`, `fe80::/10`. The tool declares the specific CIDR it needs.
- **DNS exfiltration channels.** Block egress to arbitrary DNS servers (allow only the platform's own resolvers). Many exfiltration patterns ride on DNS TXT or CNAME queries. See the [SANS DNS exfiltration primer](https://www.sans.org/blog/) and your own threat model. <!-- needs-research: find the primary SANS or similar reference for DNS-tunneling as exfiltration pattern that your security partner endorses. -->

### How to enforce it

The enforcement layer depends on the substrate, but all of them should land as policy configuration, not application code:

- **Firecracker / Kata:** A per-sandbox network namespace with a dedicated veth or TAP pair. [nftables](https://wiki.nftables.org/wiki-nftables/index.php/Main_Page) or eBPF ([Cilium](https://cilium.io/)) rules on the host side enforce the allowlist. DNS is forced through the platform's resolver.
- **gVisor:** Netstack is a userspace network stack inside the Sentry; the enforcement point is a filter on the Sentry's `netfilter`-shaped layer *plus* the host-level firewall as defence-in-depth.
- **WASM / Pyodide:** Egress exists only to the extent the host grants a socket capability. The host must only grant sockets to pre-validated destinations; the module cannot name a destination the host did not pre-allow.

Pick one enforcement topology for the platform and make every substrate converge on it. Multiple topologies mean multiple failure modes and multiple audit paths.

### Sample egress policy, declared with the tool

```yaml
# excerpt from a ServerVersion carried by the registry (mod-403)
egress:
  default: deny
  allow:
    - fqdn: api.openai.com
      port: 443
      protocol: tls
      rationale: "Tool calls the OpenAI chat completions endpoint."
    - fqdn: ".githubusercontent.com"
      port: 443
      protocol: tls
      rationale: "Tool fetches raw files from GitHub as inputs."
  block_unconditional:
    - cidr: 169.254.0.0/16       # link-local, IMDS
    - cidr: fd00::/8             # provider-local IPv6
    - cidr: 10.0.0.0/8           # platform-internal (overridable for carve-outs)
    - fqdn: metadata.google.internal
    - fqdn: metadata.azure.com
  dns:
    resolvers: [platform-resolver.internal]
    block_other: true
```

The `rationale` field is read on admission by the policy plane (mod-405) and by the security partner at review. It is not decoration; it is evidence.

## Filesystem capability

A sandbox's filesystem view is where you decide what the tool code can *see* and what it can *change*. Get this wrong and the strongest syscall policy and tightest egress rule are decoration — the tool read the credentials off disk.

### The default view

The baseline filesystem for a code-execution sandbox should be:

- **Rootfs: read-only.** The guest OS image is mounted read-only. The tool cannot modify binaries, libraries, or shipped configuration. Any write attempt is denied by the mount flag, not by permissions.
- **`/tmp`, `/home/<agent-user>`: writable, bounded, per-sandbox.** The sandbox gets a fresh tmpfs-or-equivalent with a size cap (chapter 03). It is destroyed on sandbox teardown.
- **`/workspace` (or equivalent): the explicit I/O surface.** The staged inputs (chapter 05) land here on start; the declared outputs are read from here on completion. Nothing else is persisted.
- **No bind mounts from the host.** Not `/var/run/docker.sock`, not the Kubernetes service account token path, not the host's `~/.ssh`. These get agents to arbitrary code execution on the host every quarter in some new post-mortem.

The user account inside the sandbox is a dedicated non-root (`agent` with a non-zero, non-reused UID) with no `sudo`, no `setuid` binaries on the writable paths, and umask 0077. The rootfs ships no `setuid` binaries the policy has not reviewed.

### Capabilities as declared paths, not ambient mounts

Every path the tool is supposed to touch is declared in the tool's registry entry, with a mode:

- `ro` — read-only, present at sandbox start, destroyed at teardown.
- `rw` — writable, bounded, destroyed at teardown.
- `rw-persist` — writable *and* its contents are read out at teardown (used for tool outputs; see the staging pipeline in chapter 05).

There is no fourth mode. There is no "the tool mounts whatever it wants." A tool that needs a new path files an admission change on its registry entry.

### Hardening the guest

Three hardenings that compound:

- **`noexec` on all writable paths.** `/tmp`, `/home`, `/workspace` are mounted with `noexec`. The tool cannot stage an executable payload and run it.
- **`nosuid` and `nodev` on everything the tool touches.** No setuid escalation path; no device nodes materialising.
- **Stripped `/dev`.** `/dev/null`, `/dev/zero`, `/dev/urandom`, `/dev/random`, maybe `/dev/stdin`/`/dev/stdout`. That is the list. No `/dev/kmsg`, no `/dev/mem`, no `/dev/kvm`, no raw block devices.

The guest image passes an admission test on every build: a short script asserts the mount flags, the `/dev` whitelist, the absence of setuid binaries on writable paths, and the non-root default user. The sandbox plane runs this test as part of image acceptance.

## Inbound credentials

The last axis: what identity the sandbox carries. Three rules:

1. **No ambient cloud credentials.** The sandbox does not have an EC2 instance role, a GCP service account, or an Azure managed identity automatically bound. If the tool needs cloud access, it gets a short-lived credential brokered per invocation (same model as chapter 05 of mod-403), scoped to the specific operation.
2. **No shared long-lived tokens.** If the tool needs to talk to an internal API, the gateway or the auth-wrapping proxy issues a per-session, short-lived token with the exact scope of the policy decision that admitted the call.
3. **Tokens delivered out-of-band, not baked into the image.** The sandbox image does not contain a credential. Tokens are injected via a tmpfs-mounted file on the sandbox's `/workspace/.credentials/` with 0400 permissions, destroyed on teardown.

The tool's code reads from the path; the token's lifetime matches the sandbox's. A credential that outlives its sandbox is a credential your post-mortem will discuss.

## The capability bundle as the plane's interface

Pulling the four axes together: every sandbox-creation call carries a declarative bundle.

```yaml
sandbox_request:
  tool: calendar-google
  tool_version: 2.3.1
  session_id: s-01JX9…
  tenant_id: t-acme
  substrate_tier: gvisor-default   # or firecracker-hardened, wasm-compact, …
  capability_bundle:
    syscall_profile: calendar-google-seccomp-v3
    egress:
      allow:
        - fqdn: www.googleapis.com
          port: 443
          protocol: tls
      block_unconditional: [169.254.0.0/16, 10.0.0.0/8, kubernetes.default.svc]
      dns: {resolvers: [platform-resolver.internal], block_other: true}
    filesystem:
      - path: /workspace/input
        mode: ro
        source: staging://s-01JX9/input
      - path: /workspace/output
        mode: rw-persist
      - path: /tmp
        mode: rw
        size_mb: 128
    resources: {cpu_m: 500, memory_mb: 512, disk_mb: 1024, pids: 128, fds: 512, duration_s: 60}
    credentials:
      - name: GOOGLE_API_TOKEN
        mount: /workspace/.credentials/google
        source: broker://google/{tenant}/{session}
        max_age_s: 60
```

Two properties matter: the bundle is **enumerable** (no ambient capability gets in by accident) and **attributable** (every field has a provenance from the registry, the policy plane, or the broker). Chapter 06 shows how these fields become the compliance evidence stream.

## Summary

- Isolation (chapter 01) decides the boundary; capability engineering (this chapter) decides what the sandbox is allowed to do inside it.
- Syscall policy is a deny-by-default seccomp-bpf profile (for Linux-guest substrates) or an explicit WASI capability binding (for WASM). Default-container profiles are templates, not ship targets; tighten per tool against observed behaviour.
- Egress is destination-allowlist from the tool's registry entry, unconditionally blocking cloud metadata endpoints, platform internals, link-local and private ranges, and arbitrary DNS. Enforcement lives in the network namespace, not the application.
- Filesystem is read-only rootfs plus a declared `/workspace` I/O surface; no bind mounts from the host; `noexec`/`nosuid`/`nodev` on writable paths; stripped `/dev`.
- Inbound credentials are brokered per invocation, mounted on tmpfs, destroyed on teardown; there are no ambient cloud roles.
- The whole policy ships as a capability bundle driven by the registry and the policy plane. Enumerable and attributable are the properties that let chapter 06 produce evidence.

The next chapter pins the fifth axis — resource quotas — and the eviction policy for sandboxes the session abandons.
