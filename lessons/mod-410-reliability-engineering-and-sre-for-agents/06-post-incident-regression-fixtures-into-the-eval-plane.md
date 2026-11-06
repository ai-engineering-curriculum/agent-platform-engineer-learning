# Post-Incident Regression Fixtures into the Eval Plane

Chapters 04 and 05 produced incidents and drills. This chapter closes the loop: every one of them, if the root cause reproduces on a fixed input, becomes a regression fixture in the evaluation plane (mod-407) before the post-mortem is signed. The next deploy that would reintroduce the failure is blocked by the eval gate.

The source for the general practice is the [Google SRE Book's chapter on postmortem culture](https://sre.google/sre-book/postmortem-culture/) and [the Workbook's chapter on postmortem philosophy](https://sre.google/workbook/postmortem-culture/). The specialisation is agents: the regression test is not a unit test or an HTTP integration test — it is an *eval fixture* evaluated by the eval plane's judge pipeline, because the failure mode is often a quality or behaviour failure rather than a transport failure.

An incident that is signed off and closed without producing a fixture is a lesson the fleet will re-learn. This is the obligation chapter 01 named: every reproducible incident becomes a regression fixture before the post-mortem is signed.

## The loop in one page

The mechanics, end-to-end:

```
┌─────────┐        ┌─────────────┐         ┌───────────┐
│Incident │──trace─▶Trace capture │──────▶ │Fixture    │
│response │ snaps  │(mod-408 cold)│promote│promotion  │
└────┬────┘        └─────────────┘  CLI    │CLI        │
     │                                      └─────┬─────┘
     │                                            │
     ▼                                            ▼
┌─────────┐                             ┌────────────────┐
│ Post-   │                             │mod-407 catalogue│
│mortem   │◀────regression_fixture_id──▶│ fixture row    │
│ record  │                             │ under          │
└─────────┘                             │ incident/<id>/ │
                                        └────────┬────────┘
                                                 │
                                                 ▼
                                        ┌────────────────┐
                                        │Next PR's eval   │
                                        │gate blocks the  │
                                        │regression       │
                                        └─────────────────┘
```

Three artefacts drive the loop: the incident's `trace_id` snapshot (preserved in mod-408 with incident-tagged retention), the fixture row (a mod-407 catalogue entry under the `incident/<incident_id>` namespace), and the post-mortem record (which references both).

## What makes a good post-incident fixture

Not every incident produces a fixture. Three criteria:

- **Reproducible.** The failure can be triggered by replaying a specific input (the user turn, the retrieved document, the tool output, the model response). An incident where the failure was a transient provider outage without a specific input signature is *not* fixture-worthy — it is chaos-drill territory (chapter 04).
- **Caught by an eval check.** The failure's root cause is visible in an output that an eval judge (or a hand-written assertion) can score. "The agent revealed its system prompt" is scorable; "the agent ran 2 % slower" is harder to express as a fixture and belongs in the SLO / burn-rate loop instead.
- **Attributable to a change that a gate can block.** The failure was triggered by a specific deploy (prompt change, model change, tool change) or a change that the gate will see on future PRs. "A provider changed its output distribution" is attributable to the model-gateway's version-pin posture; "a tool returned a malformed response" is attributable to the MCP gateway's input-validation posture. Both are gate-blockable; "the on-call missed the page" is not.

The fourth informal criterion: *the team learns something durable from the fixture*. A fixture that will be deleted in two months because it is low-signal is not worth the ongoing maintenance.

## The promotion pipeline, step by step

### Step 1 — Capture, during the incident

The playbook's first-five-minutes instructs the on-call to run `platform-cli trace show --snapshot <session_id>`. The snapshot pins the trace in mod-408's cold store with retention extended beyond the default (e.g., 2 years instead of 30 days). The snapshot writes a `platform.incident.trace_ref` attribute carrying the incident id.

Why "during": mitigation often cancels sessions, rotates logs, or revokes tokens — all of which may destroy the trace. The snapshot is cheap and creates the evidence; without it the post-incident fixture has nothing to replay.

### Step 2 — Identify the minimal reproducer

Not the full trace, but the *input that triggered the failure*. For each failure class:

- **Runaway loop.** The initial user turn + the system prompt + any seed documents retrieved on turn 0. The loop then arises deterministically-ish from that start.
- **Context exhaustion.** The sequence of turns and tool outputs that pushed the context over the limit. Can be a single turn if the input is already huge.
- **Prompt injection.** The injected document or user input. Often one line of text, plus the context in which it arrives (which tool returned it, which retrieval index served it).
- **Tool cascade.** The tool behaviour that triggered the cascade — but this is often better captured as a chaos-drill fixture (class-2) rather than an eval fixture, because the failure is latency-shaped, not output-shaped.

The minimal reproducer is extracted from the trace into a canonical shape:

```yaml
# fixture-source.yaml (incident bundle, pre-promotion)
incident_id: inc-2026-11-04-writer-loop
trace_refs:
  - trace:4bf92f3577b34da6a3ce929d0e0e4736
  - trace:7c28a1ff4891bd44b1dcef2c90a8e133
reproducer:
  agent_id: writer-v1
  agent_revision_at_incident: git:f912abcd
  input:
    system_prompt_version: writer-v1-prompt@v7
    user_turn: "Summarise the attached document."
    attached_documents:
      - content_hash: sha256:bea8c3...
        source: content-store/incident-redacted/doc-f9a2.txt
  expected_fail:
    symptom: runaway_loop
    detection: platform.session.turn_count >= 30
root_cause: >
  The attached document contained conflicting section headings that caused the
  agent's extract-then-summarise loop to re-read the document on every turn,
  with trivially-different tool arguments evading the loop-detector.
mitigation_applied: >
  Lowered max_turns_per_session to 20; patched loop-detector to hash on
  document-position range instead of exact arguments.
proposed_fixture:
  category: regression
  expected_pass_condition: >
    writer-v1 with the mitigation completes the session in ≤ 15 turns and
    produces a non-null summary.
```

The redactor (mod-408 chapter 04's PII-redaction layer) strips PII from the document content before it is stored as a fixture source; the post-incident author reviews the stripped content for accidental sensitive leakage before promotion. Fixture content lives under the content-store with the standard eval-plane access controls (mod-407 chapter 03).

### Step 3 — Author the fixture row

The eval plane's catalogue (mod-407 chapter 03) accepts fixture rows under `incident/<incident_id>/` namespace. The row:

```yaml
# mod-407 catalogue entry
id: incident/inc-2026-11-04-writer-loop/row-01
version: 1
source: incident_promotion
provenance:
  incident_id: inc-2026-11-04-writer-loop
  signed_by: sre-oncall-primary
  signed_at: "2026-11-05T14:21:03Z"
  trace_refs: [trace:4bf92f3577b34da6a3ce929d0e0e4736]
target_workload: writer-v1
input:
  system_prompt_version: writer-v1-prompt@v7
  user_turn: "Summarise the attached document."
  attached_documents:
    - content_ref: content://incident-redacted/doc-f9a2.txt@sha256:bea8c3...
rubric:
  type: deterministic_and_judged
  deterministic_checks:
    - name: turn_count_cap
      assert: "session.turn_count <= 15"
    - name: cost_cap
      assert: "session.cost_usd <= 0.50"
  judged_checks:
    - name: summary_non_null
      judge: judge-cards/presence-of-answer-v1.yaml
      pass_criteria: "verdict == 'present'"
expected_outcome: pass
notes: >
  Regression anchor for the 2026-11-04 writer-v1 runaway-loop incident. The
  deterministic cap is the primary signal; the judged check guards against
  the "answer but empty" mitigation shape.
```

The row is a mod-407 catalogue artefact, not an SRE-plane one. The SRE plane produces the content; the eval plane owns the schema, the judge, the gate behaviour.

### Step 4 — Gate wiring

The gate (mod-407 chapter 05) already includes `catalogue://incident/*` fixtures in the pre-merge sweep for the affected workload. A deploy PR for `writer-v1` runs every `incident/*/row-*` fixture under that workload's bucket; a regression fails the gate.

Nothing new is deployed in the SRE plane here — the gate shape is mod-407's. What this chapter does is make sure the row *arrives* in the catalogue with the right provenance and target, so the gate's existing machinery picks it up.

### Step 5 — Post-mortem link

The post-mortem record references the fixture:

```yaml
# post-mortem-inc-2026-11-04.yaml (SRE plane artefact)
incident_id: inc-2026-11-04-writer-loop
title: writer-v1 runaway loop on conflicting-heading documents
timeline:
  - "2026-11-04T09:12Z — first session enters loop"
  - "..."
root_cause: >
  ... (see fixture-source.yaml for detail)
mitigation:
  containment: [lever_1_cancel_session, lever_2_lower_turn_cap]
  recovery:   [patched_loop_detector, redeployed_v7_with_fix]
fixture_promoted:
  fixture_id: incident/inc-2026-11-04-writer-loop/row-01
  catalogue: mod-407
action_items:
  - owner: sre-plane-lead
    title: promote fixture
    status: done
  - owner: senior-agentic-ai-engineer-writer-team
    title: update agent's loop-detector tests to include fixture content
    status: open
sign_off:
  role: sre-plane-lead
  signed: "2026-11-05T14:30:00Z"
```

The post-mortem sign-off is gated on `fixture_promoted.fixture_id` being set. The paved-road post-mortem template enforces this; a template without the field is the wrong template.

## The fixture lifecycle

Fixtures do not live forever. The lifecycle:

- **Active.** The fixture is part of the regression sweep. Every relevant deploy runs it.
- **Quarantined.** The fixture is flaky (fails intermittently on unchanged code). Quarantined fixtures do not block the gate but still run; the `ai-eval-engineer` role (mod-407 chapter 06) owns the triage.
- **Retired.** The fixture is no longer load-bearing — the regression is now caught by a broader test, or the shape of the agent has evolved past the specific input. Retirement requires sign-off from both the SRE plane and the eval plane's owners.
- **Deleted.** Only after retirement plus a cooling period (typically one quarter). The fixture's content is retained in the cold store with a `retired` tag for audit; the gate stops including it.

A fixture's lifecycle events are logged; the post-mortem's `fixture_promoted.fixture_id` is a stable pointer that still resolves after retirement, so the historical record remains intact.

## Keeping the catalogue honest

A regression fixture is only valuable if the gate actually catches the regression. Two integrity checks the SRE plane owns against the eval plane.

### Pre-promotion check — does the fixture *actually* fail on the un-fixed code?

Before signing the post-mortem, the SRE plane runs the fixture against the agent's pre-mitigation revision (git:f912abcd in the example above) and confirms the fixture *fails*. If it does not, either the reproducer is wrong, or the rubric is wrong, or the mitigation was not the fix. The promotion is blocked until the pre-fix run fails.

Operationally: `platform-cli eval replay --fixture incident/<id>/row-* --agent-revision <pre_fix>`. One command, binary outcome.

### Post-promotion check — does the fixture *pass* on the fixed code?

After promotion, the same command with the post-mitigation revision confirms the fixture passes. A fixture that is red on fixed code is worthless — the gate will block every deploy. Both checks run before the post-mortem is signed.

### Periodic audit — the fixtures are still catching what they claim to

Once a quarter, the SRE plane runs every `incident/*` fixture against a "canary regression" PR — a deliberate reintroduction of each incident's pre-fix shape — and confirms the fixture catches it. Fixtures that no longer detect their original regression are either (a) too loose (the rubric was weakened to pass a different change) or (b) no longer applicable (the agent moved past the shape). Audit outcome: tighten, retire, or quarantine.

## When the fixture cannot be an eval row

Some incidents resist the "replay-the-input" fixture shape. Named explicitly so the reader knows what to do.

- **The incident depended on timing or concurrency.** Example: a race between the model gateway's retry and the sandbox's admission. There is no single input; the failure is emergent. These belong in the chaos-drill library (chapter 04) with a repeatable scenario, not in the fixture catalogue. The post-mortem references the drill scenario instead of a fixture id.
- **The incident depended on load.** Example: the tool-cascade meltdown. A single replay does not reproduce fleet-wide pool saturation. These go in the chaos-drill library's class-2 scenario, often with the specific tool's degradation parameters recorded.
- **The incident revealed a *platform* bug, not an *agent* bug.** Example: the runtime's durable-execution step failed to persist mid-session state on a specific error class. The regression is a *platform* regression; the fixture belongs in the platform's integration test suite, not the agent eval catalogue. The post-mortem references the test id.
- **The fixture's content is sensitive and cannot be preserved even redacted.** The post-mortem documents the incident with the trace references (which stay in cold store with strict access) and declares the fixture as a *restricted* row — stored, gated, but not browsable.

Each case has an escape hatch, but each still produces an artefact the gate can run and a pointer the post-mortem includes. The obligation — "an incident changes the gate" — survives the specific fixture format.

## The chaos-drill feedback loop

Chapter 04's drills also produce fixtures, through the same promotion pipeline. The drill scorecard's `fixture_candidate: true` flag is the signal. A drill that found a weakness (the hypothesis was violated, the playbook was wrong, the SLI did not catch the failure) produces:

- An eval fixture if the weakness is replayable as an input.
- A chaos-drill fixture (an updated or new drill scenario in the library) if the weakness is behavioural / infrastructural.
- A playbook revision if the weakness was a procedure gap.

The feedback loop is the same loop — incident or drill, output is "gate blocks this shape in future" plus "playbook or drill is tighter".

## Metrics on the loop itself

The SRE plane measures the fixture pipeline because a silent degradation of the pipeline is a silent degradation of the plane's main obligation. Three metrics:

- **Promotion latency.** Time from incident signed to fixture promoted. Target: ≤ 7 days paved-road default. A fixture promoted three weeks after the post-mortem is signed is a fixture that nobody owned.
- **Fixture catch rate.** Fraction of post-incident fixtures that have ever blocked a PR. A fixture that has never caught a thing in a year is a candidate for retirement (or evidence of a stable fleet, which is also useful to know).
- **Re-incident rate.** Fraction of incidents whose root cause matches a prior incident (by failure class + reproducer shape). The target is zero — a re-incident is prima facie evidence that the gate missed, which is a defect to investigate.

These three metrics live on the reliability-posture dashboard (chapter 01).

## Handoffs

- **`ai-eval-engineer` (mod-407 chapter 06).** Owns the catalogue schema, the judge calibration, and the gate's rubric handling. The SRE plane submits fixtures; they are the quality gate on the submissions.
- **`senior-agentic-ai-engineer`.** Owns the agent-side fix that the fixture asserts against. The fixture is the contract; the fix is their work.
- **`model-evaluation-engineer`.** For incidents where the fixture rubric needs statistical rigour (e.g., "hallucination rate on this class of input"), the model-evaluation-engineer is the reviewer on the rubric.
- **`agentic-safety-engineer` (mod-411).** Co-owns the fixture-promotion for prompt-injection incidents; the fixture lives in the red-team sub-catalogue and the safety plane's rule set tracks it.
- **mod-402 (runtime).** Where a trace is preserved. The snapshot API is the runtime's and the observability plane's; the fixture-promotion CLI calls it.
- **mod-413 (leadership plane).** Reviews the fixture pipeline's health metrics (promotion latency, catch rate, re-incident rate) quarterly.

## Common anti-patterns

- **"We'll write the fixture later."** Later never comes. The post-mortem is the forcing function because the sign-off blocks on the fixture-id. Enforce the block.
- **Fixture rubric calibrated post-hoc.** A rubric written to pass whatever the fixed code produces is not a regression test — it is a tautology. Author the rubric from the *failure's symptom*, not from the *fix's output*.
- **Fixture that is too specific.** A fixture whose rubric is "exact output byte-equals X" will fail on any legitimate improvement. Rubrics express the behavioural invariant, not the artefact.
- **Fixture that is too generic.** "The agent responds correctly" is not a rubric; it is a wish. The rubric names the failure shape the incident revealed.
- **Fixture library without lifecycle.** Fixtures accumulating indefinitely means the gate takes longer every deploy; retire deliberately.
- **Ignoring the pre-promotion check.** A fixture that passes on pre-fix code catches nothing. The check is the cheapest defect prevention in the whole pipeline; skipping it is why fixtures retire as "it never caught anything".
- **Not running fixtures on cross-agent PRs.** A fixture on `writer-v1` should also run on PRs that change the paved-road SDK (mod-412), the model gateway (mod-406), or the policy plane (mod-405) — shared-dependency regressions are exactly what cross-cutting fixtures catch. Cross-reference mod-407 chapter 05.

## Summary

- Every incident with a reproducible root cause produces a regression fixture in the mod-407 catalogue under the `incident/<id>/` namespace, before the post-mortem is signed.
- The promotion pipeline is five steps: capture-during-incident, identify-minimal-reproducer, author-fixture-row, verify-gate-wiring, link-from-post-mortem. Each has a specific artefact and a specific owner.
- Fixtures have a lifecycle (active, quarantined, retired, deleted) and two integrity checks: fail on pre-fix revision, pass on post-fix revision. Both run before the post-mortem signs off.
- Not every incident fits the "replay an input" shape. Timing / concurrency / load failures become chaos-drill scenarios; platform bugs become integration tests; sensitive-content fixtures are restricted. The obligation — "an incident changes the gate" — survives the specific format.
- Drill scorecards feed the same pipeline. Chapter 04's `fixture_candidate: true` flag triggers promotion via this chapter's steps.
- Three pipeline-health metrics (promotion latency, catch rate, re-incident rate) live on the reliability-posture dashboard. They are the SRE plane's audit of its own obligation.
- Handoffs land cleanly: eval plane (mod-407) owns the schema and judge; safety plane (mod-411) co-owns prompt-injection fixtures; agent engineers own the fix the fixture asserts against; leadership (mod-413) reviews pipeline health.

The module closes here. The reader now has a plane: SLIs and SLOs with pinned formulas (chapter 02), an error-budget policy as a paved-road contract (chapter 03), a chaos-drill programme with four scored classes (chapter 04), four incident-response playbooks (chapter 05), and a closed loop from incident to regression fixture (chapter 06). Every artefact feeds the next; nothing lives only in a slide.
