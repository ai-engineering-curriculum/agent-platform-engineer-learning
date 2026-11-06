# Tool Lifecycle — Approval Workflows, Scope Changes, and Deprecation

The previous chapters pinned the three policy languages, the identity primitives, and the broker. This chapter answers the question the paved-road owner lives with: *how does a tool join the plane, how does its scope change get governed, how does a breaking schema change not break production, and how do you pull the kill switch when a tool must go away now*.

The design shape here is a **ladder**. Every tool-lifecycle transition — intake, scope widen, scope narrow, schema change, deprecate, revoke — is a step on the ladder, with an owner, an approval pattern, an evidence requirement, and a rollback story. The plane that cannot name every step of this ladder has no tool lifecycle; it has a sequence of one-off exceptions, each of which is a future audit finding.

## The ladder

Six transitions in a tool's life the plane governs. Treat them as the vocabulary for the rest of the chapter.

| Transition | Who proposes | Who approves | Artefact | Reversible |
|---|---|---|---|---|
| **Intake** (new tool joins) | tool owner | admission reviewer + security partner | admission package | yes (reject or quarantine) |
| **Patch release** (compatible version bump) | tool owner | automatic if admission tests pass | signed version | yes (pin previous) |
| **Scope widen** (new scope requested) | tool owner | admission reviewer + `ai-risk-engineer` | scope-change RFC | yes if rolled out behind flag |
| **Scope narrow** (existing scope retracted) | tool owner or platform | automatic; notification required | deprecation notice | yes short-window, no long-window |
| **Breaking schema change** (incompatible arg/output change) | tool owner | admission reviewer + consumer leads | version branch + deprecation window | yes until old version retires |
| **Emergency revocation** (incident) | on-call or security | on-call authorisation | revocation decision | yes if rolled out behind flag; no if credential burned |

Every transition is a policy change: new policy, scope added to a capability, a `forbid` added to a tenant's Cedar store, a bundle revision. Every transition emits a decision log. The decision envelope from chapter 01 covers all of them.

## Intake — the admission package

A new tool arrives at the plane as a **package**, not a Slack message. The package is what the admission reviewer, the security partner, and the automated tests all consume. Its shape:

- **Tool manifest** — registry entry (mod-403 chapter 03): name, version, owner, declared scopes, declared arguments and output schema, declared tenant and region applicability, declared data classifications touched, declared external services called, declared risk tier.
- **Capability bundle template** — from mod-404 chapter 02: syscall profile, egress policy, filesystem layout, resource quotas, credential needs.
- **Policy fragments** — the OPA capability rules and Cedar policies the tool needs (chapters 02–03). Each fragment is reviewed independently.
- **Secrets manifest** — the broker-side secret kinds the tool needs (chapter 04): database.postgres.readonly, oauth.refresh_token:google_calendar, etc. Each kind is already a broker-side policy; the manifest lists which existing kinds the tool uses.
- **Threat-model attestation** — tool owner's answer to a short questionnaire authored jointly with `ai-risk-engineer`: what harm categories does this tool enable, what safeguards does the tool itself implement, what safeguards does it rely on the plane for.
- **Admission test manifest** — the test-corpus entry points (chapter 06): the regression suite for this tool's policy, the abuse-corpus subset relevant to it, the smoke test the gateway runs on every call shadowing the first N production invocations.
- **Deprecation plan placeholder** — a signed section agreeing in advance to the deprecation-ladder behaviours below. Not every field is filled in at intake; the signatures are.

A minimal package layout:

```
tools/calendar-mcp/
├── v2.4.1/
│   ├── manifest.yaml                 # the registry entry
│   ├── capability-bundle.yaml        # the sandbox bundle
│   ├── policies/
│   │   ├── capability.rego           # OPA capability rules
│   │   ├── capability_test.rego      # regression tests
│   │   └── abac.cedar                # Cedar ABAC policies (if the tool touches tenant data)
│   ├── secrets.yaml                  # the broker manifest
│   ├── threat-model.md               # the attestation
│   ├── admission-tests/              # the test corpus
│   │   ├── golden/
│   │   ├── abuse/
│   │   └── drift/
│   └── deprecation-plan.yaml         # the deprecation contract
├── v2.4.0/   # previous version, preserved for pinning
└── CODEOWNERS  # review routing
```

### The admission tests the reviewer runs

Four admission-test classes are the minimum bar. Chapter 06 scales them.

1. **Schema validation.** Does the manifest validate against the registry schema? Does the capability bundle validate against the mod-404 schema? Does each Rego file validate (`opa check`) and each Cedar file validate against the schema?
2. **Scope-use proof.** For every declared scope, the tool owner provides a smoke test that *exercises* the scope. An admitted scope with no exercising test is a scope flagged for the next review cycle (policy T-5 below).
3. **Abuse-corpus pass.** The chapter-06 abuse corpus runs against the proposed policy. Jailbreak-smuggled argument attempts, confused-deputy attempts against the broker, over-privileged scope leak attempts — all must deny with specific reason codes, and the reason codes must match expectations.
4. **Decision-log sanity.** The tool is deployed to the admission-review environment; a traffic replay runs through it; decision logs are reviewed for unexpected reason codes, unexpectedly broad scopes requested at runtime vs. declared, and PII appearing in log payloads.

Admission tests run in CI. The admission reviewer reviews the *human* parts — the threat model, the deprecation plan, the owner signatures — on top of a green CI run. A failing CI is not a thing the reviewer manually overrides.

### Admission reviewer, security partner, tool owner — the roles

Three named roles on an admission. The platform owns the vocabulary:

- **Tool owner** — the engineering team that wrote the tool, responsible for the schema, the correctness of the code, the paging rota if the tool misbehaves.
- **Admission reviewer** — a platform engineer with `tool.admit` permission in the RBAC layer (chapter 02). Checks the package against the plane's standards.
- **Security partner** — `ai-risk-engineer` (level 25) and, for higher-risk tools, `agentic-safety-engineer` (level 40). Checks the threat-model attestation against the harm model and the red-team findings they hold.

For patch releases the security partner's sign-off is on the version's unchanged threat-model attestation (which is still in force). For scope widens, breaking changes, and new intakes the attestation is re-signed.

## Scope changes — widen and narrow

### Scope widen

A tool wants a new scope: the calendar tool now wants `calendar.write`, not just `calendar.read`. The transition:

1. **Scope-change RFC.** The tool owner files an RFC in the tool's directory. The RFC names the new scope, the user-facing consent string, the argument-level constraints the new scope will carry, and the threat-model delta.
2. **Policy change.** The OPA capability policy gains the new scope as a valid entry; the capability-bundle template adds the new scope; the broker manifest, if affected, is updated.
3. **Behind a feature flag.** The new scope is initially gated by a tenant-level feature flag. Early-access tenants opt in; the gateway tracks requests by tenant to catch misuse.
4. **Review window.** After N days (typical: 14) the scope is reviewed by the admission reviewer and security partner against the actual traffic.
5. **General availability.** The flag is removed; the scope is available for every tenant that chooses to grant it in a session.

If the review window surfaces concern, the flag is pulled; the scope is not GA'd. Tenants on the flag receive a deprecation notice (next section).

### Scope narrow

A scope is going away. The transition is more sensitive than the widen because *existing sessions depend on it*. The ladder:

1. **Announcement.** Written to the tenant admins, in the plane's status feed, and on the tool's manifest page. The notice carries a target date — typical minimum horizon is 60 days for user-consent scopes, 14 days for internal scopes.
2. **Warning phase.** The gateway continues to allow the scope, but every call that invokes it carries an obligation `scope.deprecation.warn` on the decision envelope. The gateway surfaces the warning to the calling application.
3. **Blocked phase.** The capability-bundle template no longer includes the scope by default. New sessions do not have it. Existing sessions continue to have the capability on the bundle in-force until session end or until a hard deadline.
4. **Removed phase.** The OPA policy no longer recognises the scope; invocations deny with `capability.scope.removed`. The scope is still in the registry history (never deleted; audit needs it); new tools cannot declare it.

If the scope was being misused, the ladder compresses to the emergency revocation path below.

## Breaking schema changes

A tool's argument or output schema changes in a non-compatible way: the `range` argument becomes two arguments `start` and `end`; an output field is renamed; a type is tightened. Agents that call the tool expect the old shape.

Four options, roughly in order of how often they get used:

### Option A — New version on a new tool ID

The clean path. The breaking change ships as `calendar-mcp/find_free_slot_v2`, not as a new version of `find_free_slot`. Both tools exist in the registry simultaneously. Agents that want the new shape adopt the new tool; agents that call the old shape continue to work. Over time, `find_free_slot` deprecates on the scope-narrow ladder.

Use when: the breaking change is substantial enough that migration is a conscious engineering act, or when concurrent operation is needed indefinitely.

### Option B — Semver major with a deprecation window

The tool's semver major bumps: `find_free_slot` 2.4.1 → 3.0.0. The registry carries both. Capability-bundle templates carry a `tool_version_constraint` (chapter 02); existing session capabilities stay on `2.x`, new session capabilities default to `3.x`. A deprecation window applies; after the window, the `2.x` constraint is no longer satisfiable.

Use when: the change is a clean replacement, not a coexistence; the deprecation window is credible.

### Option C — Dual-mode tool (schema versioning in the argument)

The tool advertises both shapes; the caller indicates which via a `schema_version` field. The registry carries the tool once, with both schemas declared.

Use when: the change is small enough that two code paths in the tool itself is cheaper than two tool entries. Risk: reviewers often miss that the dual-mode tool's policy needs rules for both shapes.

### Option D — Shim at the gateway

The gateway rewrites old-shape calls into the new shape before they reach the tool. The tool itself only knows about the new shape.

Use when: the rewriting is unambiguous and total. Risk: the shim is now a policy-adjacent piece of code with its own correctness story. Use sparingly.

Whichever option is chosen, **the admission reviewer's role on a breaking change is to verify the chosen option is stated in the manifest, that consumers are notified, and that the deprecation plan for the old shape is in place**. Options A and B are the default; C and D are escape hatches with narrow use.

## Deprecation — the general pattern

Every deprecation (scope or tool) runs a four-phase ladder: announce → warn → block → remove. The horizons and the obligations differ by severity.

### Horizons

| Scope kind | Minimum horizon | Required communications |
|---|---|---|
| User-consent scope (visible on consent screen) | 60 days | tenant admins + end-user re-consent prompt |
| Internal / service scope | 14 days | tenant admins |
| Emergency (incident-driven) | 0 days | after-the-fact notification within 24h |

### Obligations during warn

During the warn phase, every invocation of the deprecating thing carries an obligation on the decision envelope:

```json
{
  "kind": "scope.deprecation.warn",
  "deprecating": "calendar.write",
  "removal_date": "2027-01-15",
  "migration_target": "calendar.write_v2",
  "docs_url": "https://docs.platform.local/deprecations/calendar-write-v1"
}
```

The gateway attaches this to the response headers (or the runtime's equivalent); mod-412 (paved-road SDKs) surfaces it to the agent developer.

### Block-phase reason codes

During the block phase, new sessions deny with specific reason codes:

- `capability.scope.deprecated` — scope is announced-for-removal; new sessions do not get it; existing sessions retain until end-of-session.
- `capability.scope.removed` — scope is past the removal date; no sessions get it.
- `tool.version.deprecated` — tool version constraint cannot match any supported version; caller must adopt newer version.

Dashboards track the rate of these denies; a spike near a removal date means a consumer has not migrated.

## Emergency revocation

An incident lands: a tool is actively misbehaving, a credential has leaked, a red-team finding is severe enough that the tool must stop immediately. The deprecation ladder is too slow. The emergency path:

### Primitives

- **Pull the tool's registry entry.** The registry's state becomes `revoked`. The gateway refuses every new session capability for the tool. In-flight capabilities still work until the next primitive fires.
- **Invalidate the policy bundle.** A new bundle removes the tool from the capability-template defaults and from the OPA policy's allow set. In-flight sessions that evaluate against the new bundle deny with `capability.tool.revoked.emergency`.
- **Revoke the broker leases.** Chapter 04's `vault lease revoke -prefix=tools/<tool_id>`. All active database users, dynamic certs, and signed artefacts are void. In-flight tool calls that attempt to refresh their lease fail.
- **Burn the credential.** For a leaked credential, rotate at the back-end (new Google OAuth client, new signing key, new mutual-TLS cert). The old credential becomes dead on arrival.

### Authorisation for the emergency

The RBAC layer (chapter 02) grants `tool.revoke` to `platform_admin` and to `on_call`. The latter typically carries an `approval_required: platform_admin` constraint with a `max_delay_seconds` ceiling — the on-call can act immediately, but a platform admin must co-sign within the delay window. The decision log records both.

A dry run of the emergency ladder is part of the plane's on-call rehearsal cadence (chapter 07; also mod-410 SRE). The first time the on-call runs the ladder should not be during the actual incident.

### Post-incident

Every emergency revocation triggers a post-incident review:

- The decision logs are replayed to confirm the ladder fired in the right order.
- The abuse corpus (chapter 06) gains a test case covering the specific mode that necessitated the revocation.
- The tool's deprecation plan is updated (if the tool is being brought back) or the removal is completed (if it is not).
- The threat-model attestation is re-signed if the tool returns.

## Decision records and the compliance trail

Every transition on the ladder emits a decision log with the chapter-01 envelope. The specific `action.kind` values form a closed vocabulary:

- `tool.intake.approve` / `tool.intake.reject`
- `tool.scope.widen` / `tool.scope.narrow`
- `tool.schema.change_approved`
- `tool.deprecation.announce` / `tool.deprecation.warn_phase_start` / `tool.deprecation.block_phase_start` / `tool.deprecation.remove`
- `tool.revoke.emergency` / `tool.revoke.rollback`

Each carries the tool ID, the version (or version range), the actor, the approving principals, the attached artefact hash (manifest, policy fragment, deprecation plan), and the decision ID. mod-408 picks these up on the compliance index. An auditor's "why does this tool exist in production" query resolves to the chain of intake → scope-widen → patch-releases decisions.

## Interaction with the sandbox plane

Many of these transitions interact with mod-404's capability bundle:

- A **scope widen** that implies a new egress target updates the egress policy fragment.
- A **breaking schema change** that affects the tool's syscall surface requires a new sandbox-plane admission test (the strace baseline of mod-404 exercise 02).
- An **emergency revocation** that follows a sandbox-escape finding couples with mod-404 chapter 06's event stream to identify in-flight sandboxes to be reclaimed.

Treat the capability bundle and the policy fragment as **co-versioned**: a tool's manifest references both by hash; a change to either bumps the tool's bundle revision; the gateway's admission check re-runs the admission tests when either changes.

## When the ladder breaks down

Three failure modes to recognise:

### The "important tenant" exception

A high-value tenant pushes for a scope outside the plane's harm model. The reviewer is pressured to approve. The failure mode is not the policy; it is the pressure. The response is procedural:

- The tool owner proposes the change.
- The admission reviewer and `ai-risk-engineer` jointly review. If the change exceeds the plane's harm model, they say no; the tenant's escalation path is the account team, not the plane.
- If the business decides the exception is worth the harm-model drift, the reviewer records the exception with a decision ID, a time-bound sunset, and the explicit statement of what harm-model constraint is being relaxed. Chapter 07 covers the contract with `ai-risk-engineer` for formal harm-model exceptions.

A plane without this procedure is a plane that will either refuse legitimate asks or admit illegitimate ones; both are expensive.

### Deprecation-horizon shrinkage under load

A tool is deprecating; a migration is late; the removal date approaches; someone proposes extending the horizon. The ladder behaviour: the horizon extension is itself a decision, logged, with an owner. The second extension triggers a review of why the first failed. Horizon extensions are visible on the deprecation-plan page; every extension lowers the plane's credibility, and the cost is explicit.

### Emergency without rehearsal

An emergency revocation fires; the ladder's primitives were untested; one primitive fails mid-incident. The fix is not the incident response; it is the rehearsal cadence. Chapter 07 covers the on-call cadence that `agentic-safety-engineer` helps drive. For now, the invariant: every emergency primitive has a pre-approved test that runs on a cadence in a non-production environment and emits a decision log.

## Summary

- The tool lifecycle is a ladder of six transitions: intake, patch release, scope widen, scope narrow, breaking schema change, emergency revocation. Every transition has a proposer, approver, artefact, and rollback story.
- Intake arrives as a package: manifest, capability bundle, policy fragments, secrets manifest, threat-model attestation, admission tests, deprecation plan. Admission tests run in CI; reviewers sign off on top of green CI.
- Scope widens behind a feature flag with a review window. Scope narrows follow the four-phase deprecation ladder (announce → warn → block → remove) with horizons by scope kind.
- Breaking schema changes get one of four treatments (new tool ID, semver major, dual-mode, gateway shim); options A and B are the default.
- Emergency revocation has four primitives (registry revoke, bundle invalidation, lease revocation, credential burn) with on-call authority and platform-admin co-signature.
- Every transition emits a decision record on the compliance stream; capability bundles and policy fragments are co-versioned; the ladder is rehearsed, not just documented.

The next chapter picks up the proof side — how you demonstrate, not assert, that the policies refuse the attacks the harm model names.
