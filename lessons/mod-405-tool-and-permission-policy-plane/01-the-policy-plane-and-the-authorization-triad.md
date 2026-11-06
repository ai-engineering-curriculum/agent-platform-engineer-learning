# The Policy Plane and the Authorization Triad

The MCP registry (mod-403) decides *which tools exist*. The sandbox plane (mod-404) decides *where tool code runs*. In between those two sits the question this plane answers on every single tool call: *should this caller, in this session, against this tenant's data, with these arguments, be allowed to invoke this tool right now?* The answer has to be fast enough to not stall the agent, defensible enough to survive a security review, and reproducible enough to replay six months later when a compliance auditor asks why a specific decision came out the way it did.

That is the slice this module owns. By the end of the module you can name who is being authorised, which policy language and runtime answers each slice of the question, how a non-human identity for an agent actually works, how a tool version joins and leaves the paved road, and how you prove — not assert — that your policies refuse the attacks your harm model names.

This chapter fixes the frame. The next six chapters pick the pieces.

## The slice this plane owns

The permission-policy plane sits between the runtime (mod-402) that holds the session, the registry / gateway (mod-403) that routes tool calls, the sandbox (mod-404) that executes tool code, and the identity / secret infrastructure that mints tokens and brokers credentials. It owns:

- **Authorisation decisions.** For every tool invocation, admin action on the platform, and credential-brokering request, the plane returns *allow / deny / allow-with-obligations*, plus a reason code and a decision ID.
- **Policy authorship and change control.** Policies are code. They live in a git-tracked repository, go through code review, run through a test suite (chapter 06), and ship to production through an audited release pipeline.
- **Non-human identity.** Agents, runtime workers, tool sidecars, and policy decision points themselves have workload identities (chapter 04) distinct from the end user's identity. The plane issues, rotates, and binds them.
- **Credential brokering.** Tools need secrets — an OAuth refresh token for Google Calendar, a database password for a tenant warehouse, a signing key for a payment gateway. The plane holds those and hands out *per-call* short-lived credentials bound to the caller, the tool, and the decision ID.
- **Tool-lifecycle governance.** Admission of new tools, scope changes, breaking schema changes, deprecations, emergency revocation. The policy plane is where the paved-road gate lives.
- **Decision evidence.** Every decision — allow or deny — is a record on the compliance stream that mod-408 picks up and mod-404's evidence pipeline co-indexes.

The plane does **not** own:

- **The harm model.** That is `ai-risk-engineer` (level 25). The plane consumes the harm model as inputs to policy. See chapter 07.
- **Red-team findings and jailbreak primitives.** That is `agentic-safety-engineer` (level 40). The plane operationalises findings into new policy rules and regression tests. See chapter 07.
- **Which tools exist / what their schemas are.** That is the registry (mod-403). The plane reads from the registry.
- **How the tool executes.** That is the sandbox (mod-404). The plane precedes and follows sandbox execution but does not run the code.
- **End-user authentication.** That is the identity provider the tenant brings. The plane consumes the IdP's assertions.
- **Inference-side guardrails.** That is the safety plane (mod-411). Output filtering, PII redaction, refusal ladders — they run against model output, not against tool authorisation.

Drift off this slice and you recreate work that lives in the other planes, usually incorrectly. Scope policing is a feature of this module.

## The authorization triad

Three different audiences use this plane, and each needs a different authorisation model. Picking one model for all three is the mistake that collapses the plane.

### Human operators — RBAC

A platform engineer deploys a new tool. An on-call engineer revokes a tool's capability-bundle in an incident. A tenant admin adds a user to a role that can configure a specific MCP server. These are human-operator decisions, and the policy question is:

> "Does *this user*, in *this role*, in *this organisation*, have permission to perform *this administrative action* on *this resource*?"

The right model here is **role-based access control (RBAC)**. Users have roles. Roles grant permissions. Permissions are action-plus-resource tuples. The permission vocabulary is small (dozens, not thousands) and stable across tenants; the role vocabulary is bigger but still bounded. Most decisions can be answered by walking a graph.

You will pick an identity provider the tenant already uses (Okta, Entra ID, Google Workspace), sync group membership down, and bind those groups to internal roles in your RBAC store. Chapter 02 picks the policy language that expresses these rules in a form a security reviewer can read.

### Per-tenant data boundaries — ABAC

An agent running inside *tenant A*'s session needs to call a tool that reads customer records. The customer record has attributes: `owner_tenant: A`, `region: eu-west-1`, `classification: pii.high`. The call's context has attributes too: the acting user's tenant, the session's configured region, the tool version, the capability bundle in effect, the request arguments. The policy question becomes:

> "Does *this request* satisfy *the attribute predicates* that gate access to *this resource*, considering the full set of context attributes on both sides?"

RBAC cannot answer this cleanly. Roles do not scale to "allow if resource.region matches session.region and resource.classification is at or below session.max_classification and request.arguments.limit is at or below role.max_rows". **Attribute-based access control (ABAC)** does — it is explicitly an expression language over attributes on the principal, the resource, the action, and the environment.

ABAC is where tenant isolation lives. A bug here is a cross-tenant data leak. Chapter 03 picks Cedar as the policy language for this slice and walks through why its schema-first design matches the "lots of attributes, formally reviewable" shape.

### Agent sessions — capability model

An agent is not a human. It does not have a persistent set of permissions it carries across every interaction. It has a *session*, and the session has a *bag of capabilities* that was handed to it when the session was opened: this tool, with this scope, bound to this user's consent, valid for this long, usable up to this many times. When the session ends, the capabilities end with it. When the capability bundle narrows (because a scope-exchange step narrowed it), new calls run against the narrowed set.

This is the **capability model** (see Dennis & Van Horn, 1966, "Programming Semantics for Multiprogrammed Computations"). The policy question is:

> "Does *this session's capability bundle* currently authorise *this specific operation* on *this specific target*, and if so, with which obligations (rate limit, approval requirement, redaction)?"

Why the capability model and not another layer of ABAC: capabilities carry the *entire* authorisation context for a tool call in a single bearer artefact that the gateway can verify locally. The session's identity is one part of the capability, not the whole thing. Revoking a capability is a local operation on the session, not a search across role graphs.

The capability model also matches how OAuth 2.1 scopes work on the wire — a scope on an access token *is* a narrow capability the resource server will accept. The plane's job is to map user consent (on the IdP side) and policy decisions (on your side) into the right scoped token for the call. Chapter 04 goes deep on this.

### One triad, three policy languages (usually)

A mature plane uses more than one policy language, and that is fine. Typical shape:

- **Rego / OPA** for RBAC and for the capability check at the tool-call seam. Chapter 02.
- **Cedar** for per-tenant ABAC on tenant data operations. Chapter 03.
- **OAuth 2.1 scopes + SPIFFE SVIDs** as the on-the-wire capability format, with the above languages generating and validating them. Chapter 04.

You can collapse Cedar into Rego if you want one language everywhere, and some platforms do. You can also split differently. What you cannot do is pretend one audience's model fits another's: an RBAC answer to an ABAC question misses tenant attributes; an ABAC answer to a capability question drags cross-session state into a decision that must be session-local.

## The threat model this plane defends against

Three classes of attack dominate what this plane sees. They are the ones chapter 06 builds the abuse corpus against.

### Jailbreak-smuggled tool invocation

The model, through prompt injection or a compromised document in the agent's context, is persuaded to call a tool it should not call — or to call a tool it may call *with arguments* it should not pass. The attack lives at the model-to-tool seam. The attacker's leverage is that the model is, by construction, suggestible.

The plane's defences:

- **The policy decision is independent of the model's intent.** Whatever the model asked for, the plane's rules run against the session's capabilities, the user's consent, and the tenant's constraints. "The user told me to" is not an argument the plane listens to.
- **High-risk arguments trigger an approval ladder.** Chapter 05 walks the shapes: step-up consent, two-person approval, cool-down windows.
- **Argument-level scope narrowing.** A tool that advertises `files.read` without subject constraints is a plane-level admission failure. See chapter 02 on how the capability bundle carries per-argument predicates.

### Confused deputy

The classic confused-deputy attack: a privileged service that holds credentials is tricked into using them on behalf of a less-privileged caller. In an agent setting, the deputy is often *your gateway* or *your tool sidecar*, which holds broad credentials to a downstream API and is now being asked to use them for an operation the calling session should not be able to perform directly.

The plane's defences:

- **Resource Indicators (RFC 8707)** on every token the gateway requests; every resource server validates `aud`. See mod-403 chapter 02 for the OAuth-layer treatment and chapter 04 here for the brokering layer.
- **Actor claims (RFC 8693) in tokens.** When a service acts on behalf of a user, the token names both. Audit queries can distinguish "the user did X" from "the gateway did X for the user".
- **No ambient authority inside tools.** Tools do not read raw platform credentials. Secrets arrive per-call from the broker, bound to the decision ID. See chapter 04.
- **Policy decisions are logged with the invoking principal AND the actor.** Replaying a log without the actor claim is how a confused-deputy misuse disappears into the aggregate.

### Over-privileged scope leak

A tool was admitted with a scope broader than its code actually needed. A capability bundle was authored with a filesystem mount broader than any of its advertised operations used. A role was granted to a group that contained more members than the human reviewer realised. The attack is a scope that looked tight on paper and leaked in practice.

The plane's defences:

- **Admission tests at the registry gate.** Chapter 05's admission workflow includes "list the scopes you intend to use and prove you need each one".
- **Decision-log analysis.** Regularly diff *requested-and-granted* scopes against *actually-used-at-call-time* scopes across production traffic. The gap is the over-privilege list.
- **Deprecation ladder for unused scope.** A scope present on a token for 90 days without a single invocation is a scope to remove. Chapter 05.
- **Capability-bundle narrowing on scope change.** When a scope narrows, every active session using it is either refreshed on a narrowed bundle or cut. Chapter 05.

A fourth class — **session hijack / identity confusion** — is a near relative of the first three and is addressed primarily at the identity layer (chapter 04): workload identities that mutually authenticate, short-lived SVIDs, refresh-token reuse detection. The plane assumes mTLS between components.

## What a policy decision actually looks like

Pin the shape before picking a language. Every decision from this plane returns the same envelope. The chapter-02 Rego and chapter-03 Cedar policies are different *inside* the envelope; the envelope itself is invariant.

```json
{
  "decision_id": "01HV8X3M9F5A6R2B4C0D7E8G9H",
  "decision": "allow" | "deny" | "allow_with_obligations",
  "reason_code": "capability.scope.exact_match" | "abac.tenant_region_mismatch" | ...,
  "reason_text": "session capability includes files.read_tenant_scope exactly; request within scope",
  "obligations": [
    { "kind": "approval.step_up",
      "approver_role": "tenant_admin",
      "max_delay_seconds": 300 },
    { "kind": "redact.pii_high",
      "fields": ["ssn", "dob"] },
    { "kind": "rate_limit",
      "key": "session_id",
      "window_seconds": 60,
      "max_requests": 5 }
  ],
  "evidence": {
    "input_hash": "sha256:...",
    "policy_bundle_hash": "sha256:...",
    "evaluator": "opa@1.3.0",
    "latency_ms": 1.8
  },
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "received_at": "2026-11-06T18:22:04.173Z"
}
```

Four invariants the envelope enforces:

1. **A deterministic decision ID.** Carried into every downstream record (gateway, sandbox, observability). Mod-408 uses this to answer "what policy decision allowed this call?" months later.
2. **A reason code from a closed vocabulary.** Free-text `reason_text` is for humans; the code is for dashboards, alerts, and the regression suite (chapter 06).
3. **Obligations are machine-actionable.** An obligation is a structured directive the caller must enforce — a user-prompted approval, a redaction, a rate limit. The caller echoes the obligation ID back when the obligation is discharged. The plane records the discharge.
4. **The input hash and the policy bundle hash make the decision replayable.** Six months later, you fetch the input by hash and the policy bundle by hash, re-run the evaluator, and get the same decision ID (within evaluator version). This is how you answer an auditor without archiving every input in cleartext.

The chapters that follow build the policies that produce this envelope.

## Where the plane physically sits

A typical shape for the physical deployment:

- **Policy decision point (PDP):** OPA (chapter 02) and Cedar (chapter 03) running as sidecars next to every gateway and runtime pod that needs sub-millisecond decisions, fed by a policy-bundle service that pushes signed bundles. Fallback mode when the bundle is stale is *deny* with a specific reason code.
- **Policy admin (PAP):** A git-backed policy repository; CI runs tests (chapter 06), signs bundles, and promotes through envs. No one hand-edits a bundle in prod.
- **Policy information point (PIP):** Attribute lookups — tenant properties, user properties, resource metadata — are pre-staged into the decision input where possible, called out of band when necessary. A policy that calls back to a database on every decision is a policy that will fail closed under load.
- **Policy enforcement point (PEP):** The gateway, the runtime, the broker. Enforcement happens at the seam; the PDP only decides.
- **Workload identity:** SPIRE (chapter 04) issuing SVIDs to every component, with mTLS enforced between them. The PDP checks who is calling before it evaluates the policy.

If you cannot draw this topology on a whiteboard and name the box each decision flows through, chapters 02–04 are the thing to read.

## The three obligations

Pin the plane to three non-negotiable obligations. Everything in chapters 02–07 is a decision about how to deliver them.

1. **"No tool call runs without a decision from this plane, and every decision is replayable."** No ambient authorisation, no implicit permits. A missing decision is a bug, and the gateway refuses the call. Replayability is the compliance contract.
2. **"The agent session's authority is a bag of capabilities bounded by the user's consent, the tenant's policy, and the plane's harm-model constraints — intersect, not union."** When three sources disagree, the narrowest wins. Over-privilege is a bug; refusing is correct.
3. **"The plane carries proof, not just assertion, that its rules enforce the harm model."** The regression suite (chapter 06) exercises the abuse corpus on every bundle change, and the suite is as versioned as the policies it exercises.

A plane that cannot defend all three is a policy plane in name only.

## Summary

- The permission-policy plane owns authorisation decisions, non-human identity, credential brokering, tool-lifecycle governance, and decision evidence. It does not own the harm model, red-team findings, tool schemas, sandbox execution, end-user authentication, or output guardrails.
- Three audiences get three authorisation models: RBAC for human operators, ABAC for per-tenant data boundaries, capability for agent sessions. Expect to use OPA / Rego, Cedar, and OAuth 2.1 scopes + SPIFFE SVIDs in combination.
- The defining threats are jailbreak-smuggled tool invocation, confused deputy, and over-privileged scope leak. Chapter 06's abuse corpus is the artefact that proves the plane refuses them.
- Every decision returns a standard envelope: decision, reason code, obligations, evidence hashes, decision ID. Everything that comes after is policy inside that envelope.
- Three non-negotiable obligations: every call carries a replayable decision; session authority is the intersection of consent, tenancy, and harm-model; the plane proves enforcement under an abuse corpus.

The next chapter opens the first policy language — OPA and Rego — and walks the authoring of the tool-invocation decision.
