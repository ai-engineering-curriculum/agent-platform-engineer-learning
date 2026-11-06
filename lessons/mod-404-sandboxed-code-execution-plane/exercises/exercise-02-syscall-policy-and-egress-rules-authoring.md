# exercise-02: Syscall Policy and Egress Rules Authoring

**Estimated effort:** 4 hours · **Output:** one seccomp-bpf profile (full file), one egress-policy document, one capability-bundle fragment for the tool's registry entry, and a short rationale memo (~1 page) · **Pairs with:** chapter [02](../02-syscall-egress-and-filesystem-capability.md); feeds exercise 03 and projects 401 and 404.

## Objective

Author the per-tool capability bundle for a single concrete tool — a seccomp-bpf profile narrow enough to reflect the tool's actual syscall surface, an egress policy that passes the chapter's unconditional-block set, and a registry-fragment that names both and maps them to the chapter's capability-bundle schema. If exercise 01 picked your substrates, this exercise is the first place where that pick becomes concrete executable policy.

The deliverable is the shape an admission reviewer at your platform sees on every new tool version. If this exercise produces a profile and policy a reviewer could not read in 15 minutes, the shape is wrong.

## Prerequisites

- You have read chapter [02](../02-syscall-egress-and-filesystem-capability.md). The capability-bundle example at the end of the chapter is the schema you are writing against.
- You have the Linux [seccomp-bpf documentation](https://www.kernel.org/doc/html/latest/userspace-api/seccomp_filter.html) and the [Docker default seccomp profile](https://github.com/moby/moby/blob/master/profiles/seccomp/default.json) open as references.
- You have the [libseccomp manpages](https://github.com/seccomp/libseccomp/tree/main/doc/man/man3) for the `SCMP_ACT_*` action semantics and for the specific `_arg` and `_mask` filter shapes.
- You have picked a substrate from exercise 01; if the primary tier is gVisor, read the [gVisor syscall compatibility matrix](https://gvisor.dev/docs/user_guide/compatibility/linux/amd64/) for your architecture before you start — your profile is a filter *over* what gVisor supports.
- You have access to a Linux environment where you can run `strace` and `perf trace` on the tool's typical workload for Part 1.

## Problem statement

Pick one tool from the list below (or pitch your own; the admission criteria are the same). Author its capability bundle.

- **A Python code interpreter** that executes model-generated Python against staged CSVs. Imports allowed: `pandas`, `numpy`, `scipy`, `requests` (read-only against the model gateway endpoint). Writes: output CSV and PNG charts to `/workspace/output`.
- **A browser-automation tool** that drives [Playwright](https://playwright.dev/) in headful mode against an allowlist of customer-facing URLs. Writes: screenshots and a trace JSON to `/workspace/output`.
- **A `git` + `pytest` runner** for a CI-adjacent agent — clones a customer's repository, runs tests, uploads results.
- **A video-transcription tool** that invokes `ffmpeg` and a locally-running Whisper model against a staged media file; writes a JSON transcript.
- **Your-choice tool** with a non-trivial syscall surface. It has to be more than one syscall class; a pure-compute transform does not exercise the policy work.

## Requirements

### Part 1 — Baseline the real syscall surface

Run the tool against three representative workloads. Collect:

- A syscall histogram (names + counts) via `strace -c -f` or `perf trace -s`.
- The specific `clone` / `clone3` flag mask (if applicable) via `strace -e trace=clone,clone3 -v -f`.
- The specific `openat` dirfd / flags combinations (if applicable) via `strace -e trace=openat -f`.

You will not be able to enumerate *every* syscall the tool could make across all possible workloads. Pick three workloads deliberately — one golden-path, one with heavier I/O, one adversarial (a malformed input the tool will fail on). Deliver a short summary of what each workload exercises.

### Part 2 — Seccomp-bpf profile (the main deliverable)

Author the profile. Default action is `SCMP_ACT_ERRNO` (deny by default). The profile has three zones:

- **ALLOW list.** Every syscall from your Part 1 histogram, plus a reasoned set of known-necessary-but-untraced syscalls (process lifecycle, signals). Each entry is grouped and commented with its purpose (file I/O, networking, process lifecycle, memory).
- **KILL list.** Explicitly-killed syscalls (chapter 02's list: `kexec_load`, `init_module`, `delete_module`, `mount`, `umount2`, `reboot`, `ptrace`, `bpf` where you are not using eBPF, `perf_event_open`, `keyctl`, `add_key`, `request_key`, `userfaultfd`, `name_to_handle_at`, `open_by_handle_at`, `pivot_root`, `setns`, `unshare`). Delete entries from this list only with a one-line rationale per deletion.
- **AUDIT / log.** Any syscall observed in Part 1 but that you are not comfortable allowing permanently. Mark `SCMP_ACT_LOG` so the first weeks in staging populate a real tail.

Include architecture restrictions (x86_64 at minimum; add arm64 if your fleet runs it).

Specific requirements:

- Deliver the profile as a single file: JSON (Docker-style) or YAML (CRI-style) or a shell-executable `libseccomp` Python script that produces an equivalent BPF. Pick one.
- `clone` / `clone3` entries must include an argument filter that forbids `CLONE_NEWUSER` unless unprivileged userns is explicitly your threat model (and if it is, write the one-line rationale and link to your decision).
- `socket` must include an argument filter that restricts the domain to `AF_INET`, `AF_INET6`, `AF_UNIX`, denying anything else (no `AF_NETLINK`, no `AF_PACKET`).
- Every allowed syscall is grouped with a comment naming the functional cluster it belongs to.

### Part 3 — Egress policy document

Write the egress policy as a YAML fragment that mirrors chapter 02's example. At minimum:

- The destination allowlist with `fqdn`, `port`, `protocol`, and `rationale` on every entry.
- The unconditional block list including all cloud-provider IMDS endpoints (IPv4 and provider-specific IPv6 / alternate), link-local (`169.254.0.0/16`, `fe80::/10`), platform-internal CIDRs (parameterise as `<PLATFORM_INTERNAL>`), and `kubernetes.default.svc` if your substrate lives in Kubernetes.
- The DNS section: resolvers the sandbox is allowed to use, with `block_other: true`.

Then, in one short prose section, describe the **enforcement topology**: where the policy is enforced — nftables on the host side of a dedicated veth? eBPF via Cilium? A user-mode HTTP proxy for a managed-vendor tier? — and what the failure mode looks like when a tool tries to reach a blocked destination (ENETUNREACH? Connection reset? Delayed null? The failure mode affects how a misbehaving tool's bug is diagnosed).

### Part 4 — Capability-bundle fragment for the registry

Combine Parts 2 and 3 into a capability-bundle fragment the registry (mod-403) would carry for this tool version. Mirror the schema from chapter 02's example. Include:

- `syscall_profile: <profile-id>` and a reference to where the profile lives.
- The full `egress` object.
- A `filesystem` section with `/workspace/input` (`ro`), `/workspace/output` (`rw-persist`), and `/tmp` (`rw`, with a sized tmpfs).
- A `resources` section as a stub — exercise 03 will fill it in.
- A `credentials` section if your tool needs any (the Playwright tool does not; the code interpreter calling the model gateway does).

### Part 5 — Rationale memo (~1 page)

Short memo explaining:

- The three workloads you baselined and what each exercised.
- The two or three syscalls on your ALLOW list you were least comfortable allowing, and why they are there.
- Any syscall from the chapter-02 KILL list that you had to *not* kill (with one-line rationale and a security-partner review line).
- Any egress destination you allowed that you expect your security partner to push back on, and the argument for keeping it.

## Starter guidance

- Treat `strace` output as a *hint*, not a spec. A workload-driven histogram misses the cold-path syscalls your tool makes once per process lifetime (e.g., setup, teardown). Add the usual suspects (`exit_group`, `rt_sigaction`) even if your histogram does not show them.
- The ALLOW list grows if you let it; prefer tightening than loosening. If your code interpreter calls `ioctl` with many `request` values, filter by the exact `request` values that matter (TTY ioctls for Python REPLs, nothing else).
- For the egress policy, write the `rationale` fields as if your security partner will read them in a week without your Slack history. "needed for PyPI" is not a rationale; "the tool's `pip install` phase fetches wheels from `files.pythonhosted.org`, CDN endpoints documented at <https://status.python.org/>" is.
- If you pick the browser-automation tool, remember that Playwright under the hood spawns a headful browser process with a very different syscall surface than Python-only code. Your Part 1 trace will be *large*. Decide early whether the browser process itself runs in the same sandbox or in a child sandbox with its own profile.
- `<!-- needs-research: ... -->` your way past anything you cannot verify in the current kernel release notes or project docs — notably anything touching `io_uring`, `userfaultfd`, or `bpf`. The upstream posture on these moves across revisions.

## Acceptance criteria

Mark done when:

- [ ] Part 1 syscall baselines are attached (three workloads, histograms + argument detail for `clone*` and `openat`).
- [ ] The seccomp profile loads cleanly into your test environment (`seccomp_load` returns success) and the tool's golden-path workload runs under it without `SIGSYS` / `ERRNO` on allowed syscalls.
- [ ] Every KILL-list omission has a one-line rationale and your-security-partner-review placeholder.
- [ ] The egress policy includes the full chapter-02 unconditional block set (IMDS IPv4 + IPv6, link-local, platform internals, provider-specific metadata FQDNs) and uses platform-only DNS resolvers.
- [ ] The capability-bundle fragment validates against the chapter-02 example schema.
- [ ] The rationale memo names at least two uncomfortable ALLOW decisions and at least one controversial egress entry with explicit reasoning.
- [ ] Any unverifiable kernel-feature claim is flagged `<!-- needs-research: ... -->`.

## Stretch goals

- **Tighten the profile against the tail.** Rebuild the histogram with `SCMP_ACT_LOG` for the previously-un-allowed syscalls across a longer window (overnight run against more workloads); produce a diff of what you would add or still deny.
- **Admission test.** Write a shell / Python test that loads the profile, runs a script attempting each KILL-list syscall, and asserts the process is terminated. Add it to the tool's admission pipeline.
- **gVisor equivalent.** If your primary substrate was gVisor in exercise 01, re-express the profile as a filter over the gVisor syscall matrix: list which of your ALLOWs are known supported, which are limited (per the matrix), and what the fallback behaviour is when a limited syscall is called.
- **Managed-vendor egress mapping.** For the egress policy, write a translation note: how each allow entry and unconditional block translates to E2B's / Daytona's / Modal's configuration primitives, including what to do when the vendor cannot enforce a specific rule (the proxy-your-own-traffic pattern from chapter 04).
