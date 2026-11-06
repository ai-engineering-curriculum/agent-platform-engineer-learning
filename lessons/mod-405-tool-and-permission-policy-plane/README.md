# mod-405 — Tool Registry and Permission Policy Plane

**Estimated effort:** 12 hours · **Position in track:** fifth module (after [`mod-401`](../mod-401-agent-platform-position-and-paved-road-philosophy), [`mod-402`](../mod-402-agent-runtime-service-and-durable-execution), [`mod-403`](../mod-403-mcp-registry-and-tool-gateway), [`mod-404`](../mod-404-sandboxed-code-execution-plane)).

The registry (mod-403) decides which tools exist. The sandbox plane (mod-404) decides where tool code runs. In between those two, on every single tool call, this plane answers: *should this caller, in this session, against this tenant's data, with these arguments, be allowed to invoke this tool right now?* By the end of this module you can author tool-authorisation policy as code (OPA / Rego and Cedar), engineer non-human identity (SPIFFE / SPIRE, OAuth 2.1 token exchange, Vault-side credential brokering) across your agents and gateways, design the admission-to-deprecation ladder every tool version rides, write a regression suite that proves your policies refuse an adversarial corpus, and carry the enforcement-plane contract with `ai-risk-engineer` (who authors the harm model) and `agentic-safety-engineer` (who ships red-team findings).

## Learning objectives

- Author tool-authorisation policy as code in OPA / Rego and Cedar — RBAC for human operators, ABAC for per-tenant data boundaries, capability-model shape for agent sessions.
- Engineer non-human identity for agents — SPIFFE / SPIRE workload identity, short-lived tokens via OAuth 2.1 (PKCE, Token Exchange, Resource Indicators), credential brokering for tool-specific secrets.
- Design per-tool approval workflows and deprecation ladders — new-tool intake, scope changes, breaking-schema changes, emergency revocation.
- Write policy regression tests and prove enforcement correctness under a representative abuse corpus (jailbreak-smuggled tool invocation, confused-deputy attack, over-privileged scope leak).
- Carry the enforcement-plane contract with `ai-risk-engineer` (level 25) who authors the harm model and `agentic-safety-engineer` (level 40) who produces red-team findings.

## Chapters

1. [`01-the-policy-plane-and-the-authorization-triad.md`](01-the-policy-plane-and-the-authorization-triad.md) — The slice this plane owns, the three audiences (human operators via RBAC, tenants via ABAC, agents via capabilities), the three defining attack classes (jailbreak-smuggled invocation, confused deputy, over-privileged scope leak), the invariant decision envelope, and the three non-negotiable obligations.
2. [`02-opa-and-rego-for-agent-tool-invocation.md`](02-opa-and-rego-for-agent-tool-invocation.md) — OPA as PDP, Rego in just enough depth, RBAC for operators (role bindings via IdP groups), capability policy for agent sessions (subject predicates, argument constraints, version and expiry), decision logs, integration modes (sidecar HTTP, in-process Go, WASM), bundle lifecycle.
3. [`03-cedar-and-abac-for-per-tenant-data-boundary.md`](03-cedar-and-abac-for-per-tenant-data-boundary.md) — Cedar's four-tuple, schema-first design, the tenant-boundary policy library (tenant ownership, region residency, classification ceiling, purpose limitation), templates for per-tenant variations, deployment shapes, differential analysis via the SMT backend, when Cedar is *not* the right tool.
4. [`04-non-human-identity-spiffe-oauth-and-credential-brokering.md`](04-non-human-identity-spiffe-oauth-and-credential-brokering.md) — SPIFFE / SPIRE deployment, X.509-SVIDs for component mTLS, JWT-SVIDs for bearer flows, OAuth 2.1 Token Exchange with the SVID as `actor_token`, Vault as credential broker (static, dynamic, transit), the full identity chain on a single tool call, rotation and revocation primitives.
5. [`05-tool-lifecycle-approval-workflows-and-deprecation.md`](05-tool-lifecycle-approval-workflows-and-deprecation.md) — The six transitions on the lifecycle ladder (intake, patch, scope widen/narrow, breaking schema, emergency revocation), the admission package shape, the four-phase deprecation ladder (announce → warn → block → remove), four options for breaking schema changes, and the emergency-revocation primitives.
6. [`06-policy-regression-tests-and-the-abuse-corpus.md`](06-policy-regression-tests-and-the-abuse-corpus.md) — Rule-level tests (Rego and Cedar), the versioned abuse corpus organised by attack class, the differential harness (baseline vs. candidate over corpus + replay), the SMT shortcut for Cedar, the CI gate on widening, and the deny-rate canary.
7. [`07-the-enforcement-plane-contract.md`](07-the-enforcement-plane-contract.md) — The three contracts: `ai-risk-engineer` → plane (harm model, consent, regulatory constraints), `agentic-safety-engineer` → plane (red-team findings as replayable artefacts), plane → both (decision-log metrics, weekly digest, quarterly report). The joint incident runbook. The "no surprise policy change" invariant.

## Exercises

Each exercise turns one or more chapters into an artefact you keep and reuse — in the rest of the module, in projects 402 and 403, and in later modules (mod-407, mod-408, mod-410, mod-411).

- [`exercises/exercise-01-rego-policy-for-agent-tool-invocation.md`](exercises/exercise-01-rego-policy-for-agent-tool-invocation.md) — End-to-end Rego capability policy for one concrete tool, with tests and CI. Pairs with chapter 02. Feeds exercises 03 and 04 and projects 402 and 403.
- [`exercises/exercise-02-cedar-abac-for-per-tenant-data-boundary.md`](exercises/exercise-02-cedar-abac-for-per-tenant-data-boundary.md) — Cedar schema, five tenant-boundary policies, a template-instantiated per-tenant exception, tests, memo. Pairs with chapter 03. Feeds exercise 04 and projects 402 and 403.
- [`exercises/exercise-03-spiffe-spire-workload-identity-for-agents.md`](exercises/exercise-03-spiffe-spire-workload-identity-for-agents.md) — Local SPIRE deployment, three attested workloads, mTLS gateway↔PDP, Token Exchange with JWT-SVID as `actor_token`, diagnostics memo. Pairs with chapter 04. Feeds project 402.
- [`exercises/exercise-04-policy-regression-test-suite.md`](exercises/exercise-04-policy-regression-test-suite.md) — Versioned abuse corpus across three attack classes, a differential harness, a CI gate on widening, and a self-caught widening. Pairs with chapter 06. Feeds project 403.

## Resources

External references — primary sources only — are catalogued in [`resources.md`](resources.md).

## How to approach this module

1. Read the seven chapters in order. The authorisation triad (01) is the frame everything else fits in; OPA (02) and Cedar (03) are the two policy engines, paired in that sequence because the capability shape in 02 is the most immediate; identity (04) is what makes 02 and 03's `principal` / `actor` trustworthy; lifecycle (05) is where the plane governs *change*; the regression suite (06) is where the plane proves enforcement; the contract (07) is where the plane engages with the partners who make it work in context.
2. Do exercise 01 before reading chapter 03. Writing a capability policy first gives you the hands-on grounding for why Cedar's ABAC shape is a different problem (and why both engines earn their place).
3. Do exercise 03 before reading chapter 05. Standing up SPIFFE locally makes the lifecycle ladder's identity assumptions concrete — a scope revocation without the broker and SVID picture is abstract.
4. Read OWASP Agentic AI Threats and Mitigations before chapter 01 and keep it open through chapter 06. The attack classes in 01 and the corpus in 06 are expressed in that document's vocabulary.
5. Mark `<!-- needs-research: ... -->` on any claim you cannot verify against the current OPA, Cedar, SPIRE, or OAuth 2.1 revision. The policy-plane track uses the same flag convention as mod-403 and mod-404; it blocks auto-merge, and that is cheaper than silently guessing.

## Where the handoffs land

- **`ai-risk-engineer` (level 25)** — authors the harm model you consume. Your contract to them is enforceable policy hooks for every harm category they name, decision-log evidence they can audit against, and a bundle-promotion gate that refuses hooks the policy set does not implement. See chapter 07 for the artefact shape and cadence.
- **`agentic-safety-engineer` (level 40)** — ships red-team findings as replayable artefacts. Your contract to them is the abuse-corpus structure (chapter 06) that accepts their findings as contributions, a differential harness that confirms their fixes, and a weekly / monthly cadence that surfaces which findings are in which fix state.
- **`agentic-ai-engineer` (level 30) and `senior-agentic-ai-engineer` (peer, level 40)** — write the agents that call tools the plane governs. Your contract to them is the paved-road SDK (mod-412), a predictable capability-bundle shape per tool class, and decision-envelope error codes they can present to end users without having to reverse-engineer them.
- **`ai-infra-sre` (peer) and your own future mod-410 work** — consume the policy-plane SLIs (decision p99, deny-rate by reason code, bundle-propagation lag, emergency-revocation time-to-stop) and the `exception_register` sunsets. Their SLO proposals become your admission ceilings.
- **`ai-infra-security` (peer)** — owns the deep research on jailbreak primitives, token-exchange misuse patterns, SPIFFE federation attacks, and credential-broker threat models. You operationalise their findings into tighter policies, broker-side rules, and corpus entries. Bring them your decision-log anomalies; they bring back threat models.
