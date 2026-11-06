# exercise-03: Cross-Team Incident Command Playbook

**Estimated effort:** 2 hours · **Output:** a cross-team ICS playbook (`playbook.md`), a scenario tabletop transcript (`tabletop-scenario-A.md`), the incident record it produces (`inc-20261106-003.md`), the post-incident review notes (`pir-20261106-003.md`), and a one-paragraph reflection · **Pairs with:** chapter [03](../03-cross-team-incident-command.md); feeds the governance log and the regression-fixture pipeline described in [`mod-410` chapter 06](../../mod-410-reliability-engineering-and-sre-for-agents/06-post-incident-regression-fixtures-into-the-eval-plane.md).

## Objective

Build the operational artefact a cross-team ICS actually needs: the playbook document responders open at 2 am, and the record the organisation can read a week later. The exercise forces you to (a) write a playbook that is specific enough to be useful at speed and generic enough to survive five incidents from now, (b) run a tabletop against a realistic cross-plane scenario so you discover the gaps *before* you need the playbook, (c) produce the governance-log artefacts (incident record, post-incident review) that let the next incident find its footing.

The exercise is not a one-team-response exercise. [`mod-410` chapter 05](../../mod-410-reliability-engineering-and-sre-for-agents/05-incident-response-playbooks-for-agent-failure-modes.md) covered that. This exercise is explicitly about the *cross-team* layer — IC, Ops leads from multiple teams, Comms, Planning, severity escalation, SecOps / Safety / Risk involvement, evidence preservation, the three coordination patterns.

## Prerequisites

- You have read chapter [03](../03-cross-team-incident-command.md).
- You have read [`mod-410` chapter 05](../../mod-410-reliability-engineering-and-sre-for-agents/05-incident-response-playbooks-for-agent-failure-modes.md) (the single-team playbook library the cross-team layer composes over).
- You have completed [exercise 01](exercise-01-platform-roadmap-authored-against-stream-teams.md); the tenant register is referenced in the scenario.
- You have skimmed one of the public ICS references in `resources.md` (FEMA IS-100, Google SRE's incident-management chapter, or PagerDuty's incident response docs).

## Problem statement

You are drafting the agent platform's cross-team ICS playbook. The playbook must be:

- **Specific** enough that the person opening it at 2 am can act on it without re-reading chapter 03.
- **Generic** enough that it covers the three coordination patterns in chapter 03 (platform containment while product is in-session; product misbehaviour the platform can only partially contain; suspected data exfiltration with SecOps and Risk in the loop) rather than one specific failure mode.
- **Tested** — a tabletop against at least one scenario surfaces the gaps.

You also author the scenario and run it through the playbook, producing the incident record and the post-incident review as the governance-log artefacts.

## Requirements

### Part 1 — The playbook document

A `playbook.md` with the following sections. The playbook is your paved-road artefact; everything else in this exercise reads from it.

1. **Preamble** — three or four sentences on what this playbook is and is not. Explicitly: this is the cross-team layer; single-team responses use [`mod-410` chapter 05](../../mod-410-reliability-engineering-and-sre-for-agents/05-incident-response-playbooks-for-agent-failure-modes.md).
2. **Declaration criteria** — the five triggers from chapter 03 for when a response becomes cross-team.
3. **The role structure** — IC, Operations lead (one per affected plane; include who fills Platform Ops / Product Ops / SecOps / Safety Ops / Risk Ops), Communications, Planning. For each role, what they own and what they do *not* own.
4. **Severity tiers** — Sev1 / Sev2 / Sev3, the chapter-03 definitions, who must be paged for each.
5. **Declaration lifecycle** — the seven steps from chapter 03 (page → open channel → decide cross-team → name IC → page other Ops → open war room → establish posture).
6. **Status-update template** — the chapter-03 template, verbatim with your own fields where needed. Include a tenant-facing variant.
7. **Coordination pattern 1** — platform containment while product is in-session. One worked example using your tenants from exercise 01.
8. **Coordination pattern 2** — product misbehaviour the platform can only partially contain. One worked example.
9. **Coordination pattern 3** — suspected data exfiltration with SecOps and Risk in the loop. One worked example. Emphasise the evidence-preservation-before-destructive-containment rule.
10. **Post-incident review shape** — four sections (timeline, went-well, went-poorly, action-items). Specify when the review happens (Sev1 within a week; Sev2 within two weeks).
11. **Communication ladder** — three audiences, three cadences, three default postures (chapter 03 table).
12. **Role-handoff criteria** — when IC should hand off to safety / security / risk.
13. **Failure-modes checklist** — one bullet per chapter-03 failure mode (six total), each with the mitigation.
14. **Playbook ownership** — who owns this playbook, where it lives, what the review cadence is (chapter 03 recommends reviewing playbooks after every incident that used them).

The playbook should fit comfortably in a single markdown file that someone could keep open during an incident.

### Part 2 — The tabletop scenario

A `tabletop-scenario-A.md` that composes *at least two* chapter-03 coordination patterns into one scenario. Suggested shape (feel free to invent your own, but keep it cross-plane):

> At 01:47 UTC on 2026-11-06, the MCP gateway's search endpoint begins returning 503 errors for ~18 % of requests. By 02:05 UTC, two T1 tenants (`acme-financial`, `globex-health`) report their agents producing degraded output. At 02:18 UTC, a responder notices unusually high `fetch_url` tool-call volume from one `acme-financial` agent, with URLs that pattern-match the OWASP LLM Top 10 excessive-agency / data-exfiltration class. SecOps is paged. At 02:32 UTC, SecOps asks for the request logs to be preserved before any further rollback.

Your scenario file must specify, minute by minute:

- The initial signal and the first on-call's response.
- The decision point where single-team becomes cross-team.
- At least four distinct role participations (IC + 3 Ops leads + Comms; SecOps joins mid-stream).
- A severity escalation.
- A tension between containment and evidence preservation.
- A tenant-facing communication.
- A recovery decision with a specific SLI threshold.
- The transition to post-incident review.

The scenario is the *script*; the record (Part 3) is what the fictional incident produced.

### Part 3 — The incident record

An `inc-20261106-003.md` filling the chapter-03 status-update template plus a *timeline*. Format:

```markdown
# Incident #inc-20261106-003 — Sev1

Declared: 02:05 UTC, 2026-11-06, by @on-call-mcp
Severity escalated to Sev1: 02:18 UTC, by @ic-alice (on discovery of exfiltration pattern)

IC: @ic-alice
Ops leads: @on-call-mcp (MCP gateway), @on-call-runtime (runtime, standby),
           @on-call-policy (policy plane), @secops-pat (SecOps),
           @senior-agentic-bob (product-side, acme-financial)
Comms: @comms-eve
Planning: @planning-frank

## Timeline (UTC)
01:47 — MCP gateway search returning 503s; alert fires on error rate
01:51 — @on-call-mcp acknowledges; opens #inc-20261106-003
01:55 — @on-call-mcp confirms single-plane symptom; starts local playbook
02:05 — @acme-financial reports agent-side degradation; cross-team declared
        @on-call-mcp names @ic-alice as IC
02:07 — @ic-alice pages @on-call-runtime, @on-call-policy, @comms-eve
02:10 — First status update sent to #inc-20261106-003; sev set to Sev2
… (fill in minute by minute through the scenario)
02:18 — Exfiltration pattern noticed; SecOps paged; @ic-alice escalates to Sev1
02:22 — @ic-alice directs containment-after-evidence; @secops-pat snapshots
        gateway request logs for forensics
02:35 — Rollback applied after evidence capture; error rate dropping
02:52 — SLI thresholds back to healthy
03:05 — Incident declared recovered
```

Plus the four status updates sent during the incident (02:10, 02:30, 02:45, 03:05) in the chapter-03 template format. And one tenant-facing update, lightly adapted.

### Part 4 — The post-incident review

A `pir-20261106-003.md` with four sections per chapter 03:

- **Timeline** — the minute-by-minute from Part 3, condensed where possible.
- **What went well** — at least two items. Think about the structural moves that worked (early SecOps paging, evidence-before-destructive-containment, scheduled status updates).
- **What went poorly** — at least two items. Think about gaps the tabletop surfaced (the IC was also debugging; the tenant-facing message was delayed; the policy-plane rate-limit was applied too narrowly). The failures must be real given the scenario — do not invent problems the scenario did not have.
- **Action items** — three to six items, each with owner, date, and *proof* (what artefact or SLI shows the action was done). At least one action item is a regression-fixture promotion to [`mod-407`](../../mod-407-evaluation-platform-and-regression-gating/) (via the mod-410 chapter 06 pipeline); at least one is a playbook-rev update.

### Part 5 — Reflection paragraph

A one-paragraph `reflection.md` (~120 words) answering: *the tabletop exposed at least one gap in the playbook you drafted before running it. What was it, and how did you revise the playbook?*

The playbook-vs-tabletop gap is the exercise's point. A playbook that survives first contact with a scenario without revision is probably too generic; a playbook that falls apart is probably not detailed enough. The useful feedback is in the middle.

## Starter guidance

- **Write the playbook first.** The temptation is to script the scenario and work the playbook backward. Resist — the playbook must be written to handle incidents you have not yet imagined.
- **Use real roles from exercise 01.** Your tenants (`acme-financial`, `globex-health`, …) and their on-call contacts appear in the playbook's role structure. The playbook is for *your* platform.
- **Make the status-update template exact.** The chapter-03 template has six load-bearing features; your playbook's version has them all, verbatim where appropriate.
- **The tabletop must surface a gap.** If running your scenario through your playbook does not force at least one revision, the scenario is too easy or the playbook is too loose. Try harder.
- **Timestamps are UTC.** This is a convention chapter 03 argues for; your incident record honours it.
- **Evidence preservation is non-negotiable in Part 3.** The scenario has a security dimension; the timeline must show SecOps directing evidence capture *before* destructive containment. If you wrote the scenario and this does not happen, your playbook has a gap that the reflection should name.
- **Action items have proofs.** Chapter 03 is explicit: an action item without a proof is a wish-list entry. "Update playbook" is not a proof; "playbook section 9 updated with the evidence-preservation ordering rule; verified by @safety-oncall reviewing the diff" is.
- **Do not invent organizational consequences.** Chapter 05 is clear that the engineering manager owns organisational-consequence decisions. The incident record does not include "someone should be fired" — those decisions are not in this role's scope.

## Acceptance criteria

Mark done when:

- [ ] `playbook.md` has all 14 sections listed in Part 1.
- [ ] The playbook's role structure names all five Ops lead flavours (Platform / Product / SecOps / Safety / Risk).
- [ ] The playbook includes the three chapter-03 coordination patterns with worked examples.
- [ ] `tabletop-scenario-A.md` composes at least two coordination patterns into a minute-by-minute scenario.
- [ ] The scenario includes a severity escalation and an evidence-vs-containment tension.
- [ ] `inc-20261106-003.md` has IC / Ops / Comms / Planning roles named, a minute-by-minute timeline, and four in-incident status updates plus one tenant-facing update.
- [ ] Timestamps throughout are in UTC.
- [ ] `pir-20261106-003.md` has all four sections (timeline, went-well, went-poorly, action-items) with at least two items in each.
- [ ] At least one action item is a regression-fixture promotion; at least one is a playbook-rev update.
- [ ] Every action item has an owner, a date, and a proof.
- [ ] `reflection.md` names at least one gap the tabletop surfaced and the revision made.

## Stretch goals

- **Run two more scenarios.** Scenario B: platform-containment-while-product-is-in-session (chapter 03 coordination pattern 1, no security dimension — just a model-provider degrade). Scenario C: product misbehaviour the platform can only partially contain (coordination pattern 2 — e.g., a tenant agent spamming a shared email tool). Each scenario produces its own incident record and post-incident review.
- **Author the incident-channel bot.** A short pseudocode (or a real implementation using a messaging platform's API) of a bot that, on `/incident declare sev1 inc-YYYYMMDD-NNN`, posts the playbook's role roster template, pins the status-update schedule, and reminds the IC of the first-status-update deadline.
- **Produce the playbook review cadence.** A `playbook-review-cadence.md` one-pager describing: when the playbook is reviewed (after every Sev1 incident using it; quarterly otherwise); who owns the review (the platform-team lead); how revisions are recorded (git diff + brief rationale).
- **Red-team the IC role.** Walk through each of the chapter-03 six failure modes against your scenario A and answer: did the IC in your scenario fall into this failure mode? If yes, revise the scenario or the playbook. If no, explain what structural lever prevented it.
- **Cross-reference to safety and risk tracks.** For the SecOps / Safety / Risk interactions in your playbook, produce a one-paragraph note per peer track on how your playbook aligns with (or diverges from) their expected handoff shape. The peer tracks are `agentic-safety-engineer`, `ai-risk-engineer`, and the organisation's SecOps function. You will not have the actual peer-track content; invent plausibly but mark `<!-- needs-research: ... -->` where you make substantive claims.
- **Author the tenant-council slide.** A single slide (markdown is fine) summarising to the quarterly tenant council: the Sev1 incident count this quarter, the Sev1 MTTR, the top three action items, and the trend vs. last quarter. The slide is what chapter 04's scorecard surfaces; the exercise is to practise the summary move.
