# Policy Regression Tests and the Abuse Corpus

The plane's correctness bar is *refused attacks*, not *written rules*. A rule that reads well and does not refuse what it was written to refuse is worse than no rule at all — it makes the plane less safe while telling the auditor it is safer. This chapter builds the test infrastructure that converts that bar into a running CI job.

Three artefacts to construct:

1. **The policy-rule regression suite.** Positive and negative test cases for every rule, co-located with the policy. These catch unintended changes when a rule is edited.
2. **The abuse corpus.** A versioned library of adversarial inputs organised by attack class: jailbreak-smuggled tool invocation, confused-deputy, over-privileged scope leak, and session-hijack adjacent. These catch *gaps* — attacks the current rules are silent on.
3. **The differential harness.** A comparator that runs a candidate bundle and the current-production bundle side by side across replayed traffic and the abuse corpus, emits a widening / narrowing diff, and gates merge on `no-widening-without-explicit-review`.

The three combined are what lets you promote a policy change in minutes with confidence rather than in days with hope.

## The regression suite — rule-level tests

Each policy rule ships with its tests in the same package. Chapter 02 showed the Rego pattern:

```rego
package platform.tool.invoke_test

import future.keywords.if
import data.platform.tool.invoke

test_capability_scope_exact_match_allows if {
    invoke.decision.decision == "allow" with input as {
        "action": {
            "kind": "tool.invoke",
            "tool_id": "calendar-mcp/find_free_slot",
            "tool_version": "2.4.1",
            "requested_scope": "calendar.read",
            "arguments": {
                "attendee": "alice@acme.com",
                "range": {"start": "2026-11-10T09:00:00Z",
                           "end": "2026-11-10T17:00:00Z"}
            }
        },
        "session": {
            "tenant_id": "acme",
            "principal": {"kind": "user", "subject": "alice@acme.com"},
            "capabilities": [{
                "tool_id": "calendar-mcp/find_free_slot",
                "scope": ["calendar.read"],
                "tool_version_constraint": ">=2.4.0,<3.0.0",
                "subject_predicate": {"attendee_match": "principal.subject"},
                "max_calls": 50,
                "expires_at": "2026-11-06T19:10:00Z"
            }]
        },
        "environment": {"timestamp": "2026-11-06T18:22:04Z"}
    }
}

test_capability_scope_mismatch_denies if {
    invoke.decision.decision == "deny"
    invoke.decision.reason_code == "capability.no_match" with input as {
        "action": {"kind": "tool.invoke",
                     "tool_id": "calendar-mcp/find_free_slot",
                     "requested_scope": "calendar.write"},
        "session": {"capabilities": [{"tool_id": "calendar-mcp/find_free_slot",
                                        "scope": ["calendar.read"]}]}
    }
}
```

Cedar has an equivalent via its [testing framework](https://github.com/cedar-policy/cedar#policy-tests):

```cedar-test
# Policy T-2: region residency
scenarios {
    permits {
        name: "same-region-reads-allowed",
        request: {
            principal: Session::"s_01HV",
            action:    Action::"ReadRecord",
            resource:  Record::"r_01HV",
            context:   { declared_region: "eu-west-1", declared_purpose: "support" }
        },
        entities: {
            Session::"s_01HV": { region: "eu-west-1", purpose: "support" },
            Record::"r_01HV":  { region: "eu-west-1", classification: "pii.low" }
        }
    }
    forbids {
        name: "cross-region-reads-denied",
        request: {
            principal: Session::"s_01HV",
            action:    Action::"ReadRecord",
            resource:  Record::"r_01HV",
            context:   { declared_region: "eu-west-1" }
        },
        entities: {
            Session::"s_01HV": { region: "eu-west-1" },
            Record::"r_01HV":  { region: "us-east-1" }
        },
        reason: "policy-T-2:tenant_region_mismatch"
    }
}
```

Three rules of practice:

1. **Every new rule ships with at least one permit test and one forbid test.** If you cannot think of a case the rule permits (or a case it forbids), you have authored a rule that cannot fire or cannot deny; both are bugs.
2. **Reason codes are asserted, not just the decision.** `deny with reason capability.no_match` and `deny with reason capability.subject_predicate_violation` are two different behaviours. A test that only asserts `deny` masks a regression that changes which rule did the denying.
3. **Tests use the production input shape.** A test input that works in the test but is not of the shape the gateway sends produces green CI and red production. Chapter 02's input example is the canonical shape; every test builds on it.

The suite is run with `opa test` (or `cedar test`) in CI. Latency budget: the whole suite for a tool must finish in seconds, not minutes. If it does not, decompose the tests by rule, or compile policies to WASM for faster evaluation.

## The abuse corpus — gap-finding tests

Rule-level tests catch regressions in rules the author thought about. They do not catch rules the author did not write. The abuse corpus is the second-order instrument: a library of inputs representing *what an adversary would actually try*, organised by attack class, and run against the full bundle. A deny with a specific reason code is the expected outcome; any other outcome (allow, deny with the wrong reason code, timeout) is a finding.

### Attack class 1 — Jailbreak-smuggled tool invocation

The model is persuaded (via prompt injection, a poisoned document in context, or a chained tool output) to call a tool with arguments it should not pass. The corpus instances:

- **Subject swap.** The session's user is Alice; the agent attempts `find_free_slot` with `attendee: bob@acme.com`. The capability's `subject_predicate` should deny with `capability.subject_predicate_violation`.
- **Scope escalation via argument.** The agent requests `calendar.read` but passes arguments that imply a write (`range.start: null, delete: true` on a tool that interprets null range as "all"). Argument-level constraint denies with `capability.argument_constraint_violation`.
- **Classification bypass.** The session's `max_classification` is `pii.low`; the agent requests a record with `classification: pii.high`. The Cedar classification-ceiling policy denies with `abac.classification_ceiling_exceeded`.
- **Purpose smuggling.** The session's `declared_purpose` is `support_triage`; the agent calls a tool that the user has not consented to for that purpose. The Cedar purpose policy denies with `abac.purpose_mismatch`.
- **Tenant-cross via argument.** The agent (in tenant A's session) attempts to pass a resource ID from tenant B. The Cedar tenant-ownership policy denies with `abac.tenant_ownership_violation`.
- **Deprecated-tool-use.** The agent invokes a tool whose scope is in the block phase of the deprecation ladder. The OPA policy denies with `capability.scope.deprecated` (or `capability.scope.removed` post-ladder).
- **Over-time capability use.** A capability that was valid at session-open has expired; the agent retries. The capability policy denies with `capability.expired`.

Corpus instances are not hand-written on the fly. They are versioned fixtures:

```
abuse-corpus/
└── jailbreak-smuggled-invocation/
    ├── subject-swap/
    │   ├── calendar-find-free-slot.input.json
    │   └── calendar-find-free-slot.expected.json
    ├── scope-escalation-via-argument/
    │   ├── files-read-null-scope.input.json
    │   └── files-read-null-scope.expected.json
    └── ...
```

Each `.expected.json` names the expected decision, reason code, and the policy ID that should have fired. The test harness compares; any divergence is a finding.

### Attack class 2 — Confused deputy

The attacker exploits the plane's own privilege. The corpus instances:

- **Actor-claim stripping.** The gateway's token exchange call omits the `actor_token`; the authorization server issues a token without an `act` claim; the downstream tool cannot distinguish "the user" from "the gateway acting for the user". The broker-side policy must deny with `broker.missing_actor_claim`; equivalently, the downstream tool's own policy must refuse tokens without `act` for cross-service calls.
- **Audience widening.** A tool request references a token whose `aud` is a different resource server. The receiving tool's token-validation refuses with `token.audience_mismatch`.
- **Lease-replay via decision_id.** The broker is called with a `decision_id` that the decision log does not show (fabricated) or whose result was `deny`. The broker-side policy denies with `broker.decision_id_not_allowed`.
- **Credential elevation at the tool.** The MCP server's sidecar presents its broker lease to a back-end API for a different operation than the lease permits. The back-end API's own policy denies; the back-end's logs show the failed elevation attempt.
- **SPIFFE spoofing.** A workload presents an X.509 cert not signed by SPIRE; mTLS refuses; no policy runs. The test here is at the transport layer.
- **Trust-bridge misuse.** A gateway in trust domain A attempts to use its SVID to authenticate to a service in trust domain B without an explicit federation. The receiving service refuses with `svid.trust_domain_unfederated`.

These require the test harness to simulate not just a policy input but a *plane configuration* — the transport, the token format, the broker call shape. Fixtures carry the simulated setup; the harness mounts them against local instances of OPA, Cedar, and a mock broker.

### Attack class 3 — Over-privileged scope leak

Not an attack at the moment of invocation but a slow leak. The corpus instances:

- **Request-vs-use divergence.** A session capability includes `calendar.write`; the 30-day decision log shows no call that invoked write. The corpus check: does the plane flag this capability as over-granted? Expected: an entry in the drift report with reason code `over_privilege.unused_scope`.
- **Wildcard capability drift.** A capability granted `notes-mcp/*` on an older bundle persists after a bundle change restricted the pattern. The check: does the next bundle-pull narrow the live capability, or does it continue with the old shape until session end? Both are defensible; the check asserts whichever the plane committed to.
- **Approval-bypass via session persistence.** An action that requires step-up approval is invoked once with approval, then the capability is extended; subsequent calls skip approval. The capability policy must re-require approval for high-severity actions; the test asserts the obligation re-fires.
- **Scope-upgrade via refresh.** The gateway refreshes a session-broad token; the authorization server returns a token with a *wider* scope than requested. The gateway must honour the returned scope, not the requested one; the per-call scope narrowing (chapter 04) must still enforce the policy. The test asserts the gateway does not silently accept the widening.
- **Admitted-but-unexercised.** Admission accepted a scope declared with a smoke test; the smoke test passes in CI but the production traffic never exercises the scope. Corpus check: does the drift report flag the scope as `admitted.unexercised`?

Over-privilege tests often cannot pass or fail on a single decision — they are *statistical* assertions over the decision log. The harness runs them as periodic reports against the replayed-decision store, not as blocking tests on every PR.

### Attack class 4 — Session-hijack adjacent (lighter touch)

Session hijack is primarily the identity layer's problem (chapter 04). The policy plane's role is to make hijacks visible. Corpus instances:

- **Capability reuse across sessions.** A capability_id appearing on decisions from two different session_ids. The capability policy must deny the second use; the test asserts.
- **Actor change mid-session.** A session's X.509-SVID subject changes between calls (e.g., a different gateway replica). The capability policy accepts (replicas are expected) but logs the change; the test asserts the log entry exists.
- **Clock-skew window exploitation.** A capability with `expires_at` in the past is presented by a workload with clock skew. The policy must use the authoritative time (from the environment input); the test asserts.

## The differential harness

The regression suite and the abuse corpus run against a single bundle. The differential harness runs *two* bundles — the candidate and the baseline — and reports what the candidate would decide differently.

### Shape

```bash
policy-diff \
  --baseline bundles/prod/current.tar.gz \
  --candidate bundles/pr-4823.tar.gz \
  --inputs traffic-replay/last-7d.jsonl,abuse-corpus/**/*.input.json \
  --output diff-report.json
```

The output names every input where the decision differs. For each divergence:

- `baseline.decision` and `candidate.decision`
- `baseline.reason_code` and `candidate.reason_code`
- `direction: widening | narrowing | reason_code_changed | obligation_changed`
- a context fingerprint: tool_id, tenant category, scope, action.kind

A **widening** change (baseline denied, candidate allows) is the thing to gate on. By default, CI fails the PR if any widening appears. The PR author must provide an explicit review and an annotation on each widening — a brief statement of why the widening is intended and a reference to the harm-model change (chapter 07) that supports it.

A **narrowing** change (baseline allowed, candidate denies) is usually fine, but still reported — some narrowings are desired (deprecating a scope), others are accidental (a typo in a regex). The PR author annotates.

A **reason_code change** (same allow/deny outcome, different code) is a dashboard concern — downstream alerts and metrics depend on the vocabulary. Annotate with the reason-code vocabulary update.

### What you need to run this

- A **replay store** of recent decision inputs, hashed (so payloads can be matched to policy decisions but not freely exported). Chapter 02's decision-log emission already carries the input hash; the replay store is the hash → canonical-input map, retained per the regulatory horizon.
- A **sandboxed PDP**. The harness spins up fresh OPA and Cedar instances per bundle, isolated from production, with no PIP side-effects.
- A **merge gate**. CI blocks the PR until either (a) no widenings or (b) widenings have the required annotations and sign-off from `ai-risk-engineer`.

### Cedar's analyser as a cheaper first pass

Cedar's [differential analyser](https://github.com/cedar-policy/cedar-analysis) can answer "does policy B permit any request policy A denies, under the schema?" without needing a replay. It is SMT-backed; fast; closed-form. For the Cedar policies specifically, run it before the replay-based differential. A clean SMT result lets you skip the slower replay for the ABAC layer, though the OPA side still needs the replay-based harness.

## Running the suite in CI

A good CI pipeline for a policy change:

1. **Syntax and schema.** `opa fmt`, `opa check`, `cedar validate`. Seconds.
2. **Unit tests.** `opa test`, `cedar test`. Seconds.
3. **Abuse-corpus pass.** Harness runs the full abuse corpus against the candidate bundle. Reports expected deny with reason code. Minutes.
4. **Differential (SMT).** Cedar's analyser compares candidate and baseline over the schema. Seconds-to-minutes.
5. **Differential (replay).** Harness runs candidate and baseline across a last-N-days replay. Minutes.
6. **Human sign-off.** Any widenings require `ai-risk-engineer` sign-off; any structural change (new schema field, new scope) requires admission-reviewer sign-off.
7. **Build and sign.** `opa build`, sign with Cosign, push to bundle service.
8. **Deploy.** Bundle service, PDP pull, blue-green with automatic rollback if deny-rate regresses.

Steps 1–5 are the gate. Step 6 is the human loop. Step 7 is mechanical. Step 8 is the deploy, with a safety net.

## The deny-rate canary

A policy change can look clean in differential (no widening, no regressions in tests) and still break production if it introduces unintended denies on real traffic. The canary:

- Deploy the new bundle to a canary PDP serving 1% of traffic.
- Measure `deny_rate` on that PDP vs. production PDPs, by reason code.
- A deny-rate spike (p95 of `deny_rate_by_reason[reason_code]` on canary exceeds baseline by configurable delta) triggers automatic rollback.
- Human-driven rollback command (chapter 05) is always available.

A bundle that doubles the global deny-rate because of an unintended tightening is the common failure the differential cannot catch. The canary is where it surfaces.

## What the corpus does not catch

Honesty: the corpus catches the attacks it is written to catch. New attack classes — a novel jailbreak primitive, a previously-unknown token-exchange misuse — require someone (usually `agentic-safety-engineer`, chapter 07) to write the corpus entries. The gap between "an attack exists" and "the corpus includes it" is operational risk.

Three practices narrow the gap:

- **Every red-team finding ships with a corpus entry.** `agentic-safety-engineer`'s deliverable to the plane is not "we found a bug" — it is "we found a bug and here is the input that exhibits it, along with the expected post-fix behaviour". Chapter 07 writes this into the contract.
- **Every incident ships with a corpus entry.** Chapter 05's post-incident review produces a new abuse-corpus test case covering the mode that necessitated the revocation.
- **The corpus is reviewed quarterly against the harm model.** `ai-risk-engineer` reviews the harm model; the corpus review asks "does the corpus still exercise every category". The review is itself a decision logged on the plane.

## Summary

- The regression suite covers per-rule behaviour; the abuse corpus covers gap-finding across known attack classes; the differential harness covers unintended widening against a previous bundle.
- Rule-level tests assert both decision and reason code. Tests use the production input shape.
- The abuse corpus organises by attack class: jailbreak-smuggled invocation, confused deputy, over-privileged scope leak, session-hijack adjacent. Each entry names expected decision and reason code.
- The differential harness runs candidate and baseline over traffic replay plus the corpus. Widenings gate merge; narrowings and reason-code changes are annotated.
- CI runs syntax, unit tests, abuse corpus, SMT analysis (Cedar), replay differential, human sign-off, build, canary. A deny-rate canary catches unintended tightenings differential cannot.
- The corpus is only as complete as what is written into it. Every red-team finding, every incident, and every quarterly harm-model review feeds new entries.

The next and final chapter writes the contract across the two partner roles the plane carries: `ai-risk-engineer` (harm model upstream) and `agentic-safety-engineer` (red-team findings upstream).
