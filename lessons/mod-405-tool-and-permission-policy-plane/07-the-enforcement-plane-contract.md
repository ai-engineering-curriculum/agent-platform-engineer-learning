# The Enforcement-Plane Contract — Partners Upstream of the Policies

The plane does not author the harm model and does not source the adversarial findings. Those are the deliverables of two specialised roles you partner with: **`ai-risk-engineer`** (level 25) who authors the harm model that says *what kinds of harm the organisation refuses to enable*, and **`agentic-safety-engineer`** (level 40) who ships the red-team findings that say *how, specifically, an adversary tries to cause each harm*. Your plane converts their deliverables into enforceable policies, broker-side rules, abuse-corpus entries, and emergency-revocation triggers.

This chapter writes the contract. Not a vague RACI — a specific interface with named artefacts, cadences, and failure modes. A policy plane without this contract will either over-rotate on the risk team's suggestions (freezing the paved road) or quietly diverge from the harm model (eventually finding out in a post-incident review that the policies did not actually cover what the harm model required). Both failures are expensive.

## The three contracts

Three interfaces to formalise:

1. **`ai-risk-engineer` → policy plane.** Harm model, consent model, regulatory constraints. You consume. Your policies enforce.
2. **`agentic-safety-engineer` → policy plane.** Red-team findings, threat primitives, novel attack classes. You consume. Your abuse corpus (chapter 06) encodes.
3. **Policy plane → both.** Decision-log metrics, denied-attack signals, bundle changes requested. You emit. They consume to update the harm model and the red-team priorities.

Each contract has a surface — a named artefact — a cadence, an acceptance criterion, and a failure mode. All six get walked below.

## `ai-risk-engineer` → you — the harm-model handoff

### What `ai-risk-engineer` ships

The harm-model artefact names:

- **Harm categories.** For this organisation, which kinds of harm are refused and which are accepted. Typical examples: *PII exfiltration to unaudited destinations* (refused), *autonomous financial-transaction authorisation above a threshold without human sign-off* (refused), *emission of content that violates the platform's TOS or applicable law* (refused via mod-411 downstream), *agent-driven mass communication without rate controls* (refused).
- **Severity bands.** Low / medium / high / critical, with a definition for each — the plane uses these to pick obligations and approval ladders. A `critical`-severity capability request requires step-up approval, two-person sign-off on the admission, and a mandatory quarterly review.
- **Consent model.** What consent means for each scope, where it is collected, how long it lasts, what revokes it. The ABAC policy T-4 (chapter 03) enforces this.
- **Regulatory constraints.** GDPR Article 5 purpose limitation, EU AI Act Article 72 record-keeping, sectoral constraints (HIPAA, PCI-DSS, FedRAMP) that apply to specific tenants. Each constraint maps to one or more Cedar policies or OPA rules.
- **Exception register.** The time-bound exceptions granted (per tenant, per capability) with their sunset dates. Chapter 05's "important tenant" exception lands in this register.

The shape is not a PowerPoint. It is a signed artefact — JSON or YAML, versioned, in a repository the plane reads at bundle-build time:

```yaml
version: "2026.11"
signed_by: "ai-risk-engineer@acme"
signed_at: "2026-11-01T09:00:00Z"
expires_at: "2027-02-01T09:00:00Z"

harm_categories:
  - id: "pii_exfiltration"
    severity: "critical"
    description: "Movement of personally identifiable information to a destination not previously authorised by the data subject."
    policy_hooks:
      - "platform.abac.purpose_limitation"
      - "platform.abac.destination_allowlist"
      - "platform.capability.scope.narrow"
    decision_obligations:
      - "evidence.record.high_sensitivity"
      - "approval.step_up"

  - id: "autonomous_financial_authorisation"
    severity: "critical"
    description: "Agent-initiated financial transaction authorisation exceeding the tenant's per-transaction limit without human sign-off."
    policy_hooks:
      - "platform.capability.argument_constraint:financial.amount"
      - "platform.approval.two_person_sign_off"

consent_model:
  user_consent_default_ttl_seconds: 3600
  purpose_strings:
    - id: "support_triage"
      description: "Resolve an open support ticket for the consenting user."
      maps_to_scopes: ["calendar.read", "email.read", "notes.read"]
    - id: "marketing_campaign"
      description: "Send promotional content to opted-in recipients."
      maps_to_scopes: ["email.send", "contact_list.read"]
      requires_separate_consent: true

regulatory_constraints:
  - frame: "gdpr"
    article: "5(1)(b)"
    requires: "purpose_limitation"
    policy_hooks: ["platform.abac.purpose_limitation"]

  - frame: "eu_ai_act"
    article: "72"
    requires: "post_market_monitoring_records"
    policy_hooks: ["platform.decision_log.evidence_emission"]
    retention_horizon: "see-mod-404-ch06"

exception_register:
  - tenant: "acme-enterprise"
    capability: "export_customer_list"
    sunset: "2027-01-15"
    rationale: "migration-to-replacement-flow"
    approved_by: ["platform-admin@acme", "ai-risk-engineer@acme"]
```

### Cadence

- **Quarterly** harm-model review. `ai-risk-engineer` publishes an updated artefact; the plane runs the differential harness (chapter 06) against the policy bundle to show which policies change; a joint review signs off.
- **On-incident.** Any incident that touches a harm category triggers an immediate harm-model review. The post-incident artefact (chapter 05) is authored jointly.
- **On-regulation-change.** A new EU AI Act delegated act, a new state-level privacy regulation, a new sectoral rule — all trigger an off-cycle harm-model update.

### Acceptance criterion

The plane accepts a new harm-model version only when:

- Every `policy_hook` string in the artefact resolves to an actual policy package or rule in the bundle.
- The abuse corpus (chapter 06) covers every harm category with at least one test case per severity band.
- The regulatory constraints each have a decision-log query that an auditor can run.

A hook that does not resolve is a bug: either the harm model names a policy that no one wrote, or the policy name drifted. Either way, the bundle does not promote until it resolves.

### Failure modes

- **Harm model lags reality.** `ai-risk-engineer` has not published an update in a quarter; the artefact is near expiry; no regulatory change has forced an update. The plane's response: the quarterly review itself fires a notification; after the expiry, the bundle refuses to promote without an explicit extension. This creates pressure to keep the artefact current.
- **Harm model exceeds plane capability.** The harm model names a hook the plane cannot enforce (because the attribute is not available, because the broker does not support the obligation). The response: the plane raises a `cannot_enforce` finding on the review; the harm model is amended or the plane gains the capability. Silent acceptance ("we'll get to it") is the failure.
- **Plane enforces beyond the harm model.** The plane's policies refuse a request that the harm model does not require refusing. This is also a finding — not a bug (safety-conservative is safer) but a signal that either the harm model is incomplete or the policy is over-broad. The review resolves either way.

## `agentic-safety-engineer` → you — the red-team handoff

### What `agentic-safety-engineer` ships

Red-team findings arrive as **replayable artefacts**, not Slack screenshots. The minimum fields:

- **Finding ID and severity.** ULID; severity drawn from the harm-model bands.
- **Attack class.** One of the chapter-06 classes (jailbreak-smuggled, confused-deputy, over-privileged-scope, session-hijack-adjacent), or a new class named and defined in the finding.
- **Reproduction package.** The specific inputs — session input, tool input, timing — that exhibit the finding, versioned. Run through the harness; the harness reproduces the attack against the current bundle.
- **Expected post-fix behaviour.** What the fixed plane should return (deny with which reason code, log which evidence, trigger which approval ladder). The abuse corpus (chapter 06) entry is already written.
- **Harm-model category impacted.** The ID from `ai-risk-engineer`'s artefact.
- **Proposed policy change (optional).** A PR against the policy repository the finding team has drafted. The plane owner reviews, iterates, and merges — or writes a different fix if the finding team's fix has unacceptable collateral.
- **Verification hooks.** How the plane will know, after the fix, that the attack is still refused (decision-log queries, monitoring alerts, periodic abuse-corpus runs).

A finding without a reproduction package is a report, not a finding. The policy plane does not write fixes from vague reports.

### Cadence

- **Continuous.** Red-team findings arrive asynchronously.
- **Severity-triggered handling.** Critical findings go to emergency-revocation triage (chapter 05); high findings go to the next bundle; medium / low findings join the backlog with a response-time target (e.g. two weeks for medium).
- **Monthly red-team review.** `agentic-safety-engineer` presents the findings cohort; the plane owner presents the resolution status; both sides re-prioritise.

### Acceptance criterion

A finding is accepted when:

- The reproduction package exists and the harness reproduces it on the current bundle.
- The expected post-fix behaviour is unambiguous.
- The abuse-corpus entry is authored and the test is wired into CI.

A finding without reproduction is returned to `agentic-safety-engineer` for completion. A finding whose expected behaviour the plane and the safety team disagree on goes to a joint review with `ai-risk-engineer` to resolve — this is often a harm-model gap, not an enforcement one.

### Failure modes

- **Finding-volume outstrips capacity.** The policy team cannot absorb the finding rate. Response: triage severity; medium / low findings queue; the harm-model review surfaces the backlog so the risk and safety teams can argue the priorities explicitly rather than letting the policy team silently drop them.
- **Reproduction drifts.** A finding's reproduction worked six months ago; now the inputs are stale (tool names changed, scopes renamed) and the attack no longer reproduces even on an un-fixed bundle. Response: the finding is re-reproduced on the current bundle; if the attack no longer works, the abuse-corpus entry stays (as regression coverage) and the finding is closed. If the attack still works but needs a new reproduction, the reproduction is updated.
- **Fix removes a required behaviour.** A fix for a confused-deputy finding tightens the broker-side policy such that a legitimate flow breaks. The differential harness (chapter 06) catches the narrowing; the fix is iterated with the originating tool owner.

## You → both — the decision-log and metrics upstream

What you ship back is the evidence that the plane is enforcing what the harm model says and refusing what the red-team finds.

### Metrics

A core set of metrics the plane exposes on the compliance / observability stream (mod-408):

- **`policy.decision.rate{action.kind}`** and **`policy.decision.rate{action.kind, decision}`** — baseline health.
- **`policy.decision.deny_rate_by_reason`** — a vocabulary-indexed deny rate. Spikes on a specific reason code (e.g. `capability.subject_predicate_violation`) are red-team signals and `agentic-safety-engineer` watches them.
- **`policy.harm_category.denies{category, severity}`** — denies tagged with the harm category the firing policy enforces. `ai-risk-engineer` watches these to confirm harm-model coverage is operationally effective.
- **`policy.exception_register.active_count{tenant}`** — exceptions in force. Watchlist for sunset compliance; alerts fire N days before sunset.
- **`policy.bundle.widening_merges{period}`** — merges that widened policy (annotated per chapter 06). Trend watchlist — a quarter with many widenings is a harm-model-evolution signal.
- **`policy.admission.rejects{reason}`** — new tool admissions rejected by reason. `ai-risk-engineer` watches for patterns (categories of tool consistently rejected for a specific reason) that imply a harm-model refinement.

### Cadence of reporting

- **Real-time dashboards** on the metrics above (mod-408).
- **Weekly summary** — a short written digest sent to `ai-risk-engineer` and `agentic-safety-engineer`. Keep it focused: top five reason-code movements, open exceptions nearing sunset, bundle changes shipped.
- **Quarterly report** — the review input for both partner cadences. Captures harm-model coverage, regression-suite coverage, abuse-corpus growth.

### Failure modes

- **Metrics without context.** A deny-rate spike without the reason-code vocabulary is noise. The dashboards group by reason code, tool, tenant, and time; metrics without those dimensions are not useful to the partners.
- **Dashboards no one watches.** A plane that emits metrics but has no one looking at them is not getting value from the handoff. The weekly digest is the forcing function that turns metrics into discussion.

## A joint incident model

Some incidents touch all three roles. A rough runbook:

1. **Incident detected** — abnormal deny-rate spike, an active attack reported by `agentic-safety-engineer`, a tenant complaint of unexpected behaviour.
2. **On-call triage** — plane on-call starts the clock. Chapter 05's emergency primitives are available.
3. **Joint channel opened** — plane on-call, `ai-risk-engineer`, `agentic-safety-engineer`, affected tool owner. Explicit roles: plane on-call is incident commander; risk engineer is harm-model authority; safety engineer is attack-primitive authority.
4. **Stop the bleed** — emergency revocation primitives (chapter 05). Scope narrowed, bundle rolled back, lease revoked, credential burned.
5. **Preserve evidence** — the decision log for the window is tagged; mod-404 chapter 06's evidence stream is referenced.
6. **Root cause**:
   - Was the harm model silent on this harm? → `ai-risk-engineer` deliverable.
   - Was the harm model clear but the policies did not enforce it? → plane deliverable.
   - Was the attack novel? → `agentic-safety-engineer` deliverable; corpus grows.
7. **Fix** — PR, differential harness pass, abuse-corpus entry, canary, promote.
8. **Post-incident** — joint review within one week. Deliverables: updated harm model (if needed), new abuse-corpus entries, policy change references, timeline, lessons.

The thing the runbook encodes: at no point does one role carry the incident alone. The plane sits between the model and the findings; the incident response reflects that.

## The "no surprise policy change" invariant

A final invariant that cuts across all three contracts:

**No policy change that widens enforcement, narrows enforcement for a critical harm category, or changes a reason-code vocabulary ships without `ai-risk-engineer` sign-off.**

This is the thing that keeps the plane honest. A change the plane owner thinks is routine may have harm-model implications only the risk engineer can see. A reason-code rename breaks dashboards the safety engineer depends on. A narrowing that looks like a quality improvement to the plane is a harm-model loosening to the risk engineer.

Implementation:

- Chapter 06's CI gate blocks merges on widenings and vocabulary changes without the sign-off.
- The sign-off is recorded on the decision log of the bundle promotion.
- The sign-off's absence is itself a finding on the next quarterly review.

## What the contract is not

Three things to avoid.

### It is not a committee

The contract does not require every change to go through a meeting. The cadences are the meetings; the artefacts are what pass between them in the intervals. Routine policy changes (new rule, new test case) ship through the normal CI gate. The contract applies at the shape-level: who authors what, who signs what, what the failure modes look like.

### It is not a handoff of ownership

The policy plane remains the authority on *how* policies are written, tested, and shipped. The harm model is the input; the red-team findings are the input; the plane's authorship judgement over the Rego and Cedar code is not delegated to either partner.

### It is not a shield against responsibility

A plane owner cannot deflect an incident by saying "the harm model did not say." The plane shares ownership for the operational reality of enforcement, and part of that ownership is pushing back when the harm model is silent on something the plane sees in traffic. The contract works in both directions.

## Summary

- The policy plane consumes the harm model from `ai-risk-engineer` and the red-team findings from `agentic-safety-engineer`, and emits decision-log metrics back to both.
- The harm model is a signed, versioned artefact with harm categories, severity bands, a consent model, regulatory constraints, and an exception register. The plane's bundle will not promote unless every policy hook the artefact names resolves.
- Red-team findings arrive as replayable artefacts with reproduction, expected post-fix behaviour, harm-model category, and verification hooks. A finding without reproduction is returned.
- The plane emits metrics keyed by reason code and harm category, a weekly digest, and a quarterly report. Both partners act on these.
- The incident runbook names three roles and six steps; no role carries the incident alone.
- The "no surprise policy change" invariant blocks widenings, critical-category narrowings, and reason-code changes without `ai-risk-engineer` sign-off.

This is the final chapter of mod-405. The module's six-chapter arc:

1. The authorization triad (RBAC / ABAC / capability).
2. OPA and Rego for capability and operator RBAC.
3. Cedar and ABAC for per-tenant data boundaries.
4. SPIFFE / SPIRE and the credential broker for non-human identity.
5. The tool lifecycle ladder (intake → deprecation → emergency revocation).
6. The regression suite and the abuse corpus that prove enforcement.
7. The contract with the risk and safety partners who make the plane work in context.

The paired projects (project-402 `governed-mcp-gateway` and project-403 `eval-gated-agent-fleet`) build increasingly realistic versions of this plane; the exercises in this module produce the artefacts those projects compose. Carry the paper trail forward.
