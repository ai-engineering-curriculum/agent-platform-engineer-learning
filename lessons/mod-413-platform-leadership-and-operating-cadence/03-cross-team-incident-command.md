# Cross-Team Incident Command

Chapter 01 pinned the roadmap. Chapter 02 pinned the portfolio. This chapter pins the moment neither of those things can help: *the platform is on fire, a product-side agent is misbehaving, and six teams are in the same Slack channel at 2 am*.

[`mod-410` chapter 05](../mod-410-reliability-engineering-and-sre-for-agents/05-incident-response-playbooks-for-agent-failure-modes.md) built four single-team playbooks — runaway loop, context exhaustion, prompt-injection containment, tool-cascade meltdown. Each assumes one team operating the response. This chapter builds the layer above: the incident-command discipline for when the response *cannot* be one team, because the failure spans the platform and a product-side agent and the cheapest path to containment requires coordination.

The grounding is the [FEMA Incident Command System (ICS)](https://training.fema.gov/is/courseoverview.aspx?code=is-100.c), adopted and adapted by [Google SRE's incident-management chapter](https://sre.google/sre-book/managing-incidents/) and by [PagerDuty's incident-response documentation](https://response.pagerduty.com/). The specialisation here is *agents*: a prompt-injection on one tenant's agent can exfiltrate data via a shared MCP tool; a model-provider latency spike can cascade into A2A loops across tenants; a sandbox breakout is simultaneously a platform incident and a `agentic-safety-engineer` matter. ICS as a structure is public; the roles, severity tiers, communication cadence, and coordination patterns adapted to agents are what this chapter owns.

## When cross-team incident command is the right frame

Not every incident is cross-team. The default posture is a single-team response per [`mod-410` chapter 05](../mod-410-reliability-engineering-and-sre-for-agents/05-incident-response-playbooks-for-agent-failure-modes.md). Escalate to cross-team ICS when *any* of the following is true:

1. **The incident spans platform and product.** The platform's safety plane is refusing turns and a tenant's revenue-critical agent has stopped working; nobody can tell yet whether this is a safety-plane regression or a tenant-side policy violation.
2. **The incident spans multiple tenants.** One shared plane (MCP gateway, model gateway, sandbox pool, policy plane) is misbehaving and a dozen agents across six tenants are affected.
3. **The incident has a safety or security dimension.** A prompt-injection attack is in progress; a sandbox breakout is suspected; a model is producing output that violates a regulatory regime. Any of these requires coordination with `agentic-safety-engineer` or SecOps.
4. **The incident requires legal or compliance decisions in the hour.** A tenant is asking "did customer PII reach the model's logs?"; the answer determines a disclosure obligation.
5. **The incident will last more than two shifts.** A single on-call engineer cannot hold context across a multi-hour incident; cross-team ICS provides the handoff structure.

If none of the five applies, run [`mod-410` chapter 05](../mod-410-reliability-engineering-and-sre-for-agents/05-incident-response-playbooks-for-agent-failure-modes.md) and leave this chapter closed.

## The role structure

ICS uses a pyramid: one Incident Commander (IC), subordinate leads for Operations, Communications, and Planning, and the responders below them. The adaptation for agent platforms:

### Incident Commander (IC)

One person. Named at the moment the incident is declared cross-team. Owns:

- **Priorities.** "Contain data exfiltration" trumps "preserve session state". The IC makes the trade-off calls that each team alone cannot.
- **Severity.** Sev1 / Sev2 / Sev3 (defined below). Severity is a decision, not a sensor reading; the IC owns it.
- **Resource allocation.** Who joins the war room, who stays on their own incident, who is released to sleep.
- **The record.** Timestamps, decisions, hand-offs. The post-incident review draws from the record; the IC owns it or delegates to Planning.

Who the IC is: typically the on-call *platform* lead if the incident is platform-originating; the on-call *senior-agentic-ai-engineer* if the incident is product-originating but requires platform coordination; the on-call *agentic-safety-engineer* if the incident has a frontier-safety dimension. The role is explicit; *the IC is not simultaneously debugging the problem*.

### Operations Lead (Ops)

One person per affected plane. Owns the actual containment work on that plane. Reports to the IC with status; receives priorities from the IC.

- **Platform Ops** — the on-call engineer for each affected platform plane (runtime, MCP gateway, sandbox, policy, model gateway, eval, observability, A2A, safety, SDK).
- **Product Ops** — the on-call engineer for each affected tenant's agent. The `senior-agentic-ai-engineer` or the stream-aligned team's on-call fills this.
- **SecOps** — joins when the incident has a security dimension. Owns the forensic posture, the containment policy, the evidence preservation.
- **Safety Ops** — the `agentic-safety-engineer` on-call. Joins when the incident has a safety dimension (prompt-injection in progress, jailbreak attempt, output producing refusable content at a tenant's agent).
- **Risk Ops** — the `ai-risk-engineer` on-call. Joins when the incident has a regulatory, PII, or audit dimension. For incidents in regulated industries (financial-services, healthcare, government), this role is often mandatory.

### Communications Lead (Comms)

One person. Owns the outward-facing messages: tenants, leadership, status page, legal if applicable.

- **Tenant updates** — a status update to affected tenants every 30 minutes for Sev1, every 60 minutes for Sev2. The template below.
- **Leadership updates** — a terse async update to the executive sponsor every hour; more frequent if severity escalates.
- **Status page** — if the platform has a tenant-facing status page, Comms owns the state transitions.
- **The "no" to premature external statements.** Comms is the one who says "we do not yet have enough information to publish a root cause".

### Planning Lead (Planning)

One person. Owns the incident record and the shift handoffs. The IC sometimes doubles as Planning in Sev2/Sev3; for Sev1 the roles are separate.

- The timeline: every decision with a timestamp, who made it, what evidence supported it.
- The resource roster: who is on the war-room now, who is on the next shift, who has been released.
- The handoff artefact: when the IC rotates, Planning writes the handoff summary the new IC reads before taking the role.

### Responders

The engineers actually running the playbooks. Report to their Ops lead, not directly to the IC. The IC does *not* bypass Ops to direct responders; a structure where everyone is talking to the IC collapses within an hour.

## Severity tiers

Severity is a decision the IC owns. The tiers:

| Tier | Definition | Response posture |
|---|---|---|
| **Sev1** | Multi-tenant impact OR safety/security dimension OR data-handling implication. Revenue-critical agents offline or regulatory obligation at risk. | Full ICS structure; war room opens; executive sponsor paged; Sev1-status to all tenants within 15 minutes; shift rotation enforced at 4-hour boundaries. |
| **Sev2** | Single-tenant significant impact OR a platform plane degraded for a subset. SLO burning fast. | ICS structure with IC + Ops + Comms; war room opens; sponsor informed by email; tenant-affected informed within 30 minutes. |
| **Sev3** | Degraded but non-urgent (e.g., non-paved-road tenants affected; a plane is slower but not down). | Single-team response per mod-410 chapter 05; file an incident record; update status page if tenant-visible. |

Severity can escalate during an incident. A Sev2 that reveals a safety implication becomes Sev1. The *escalation* is itself a decision in the timeline; the IC calls it, Comms sends the update.

A principle: **when in doubt, declare the higher severity**. Downgrading a Sev1 at hour 2 is cheap; upgrading a Sev3 at hour 2 is expensive — the tenants you did not alert are already angry.

## Declaring the incident

The lifecycle:

1. **A page fires.** On-call for one plane sees a burn alert, a safety-refusal spike, a sandbox pool exhaustion, a model-gateway back-pressure.
2. **The on-call opens an incident channel.** Platform convention: `#inc-YYYYMMDD-NNN`, where `NNN` is monotonically increasing per day. Pin the current status, the timeline link, and the role roster.
3. **Within 10 minutes the on-call decides: single-team or cross-team?** The five criteria above answer the question. If cross-team, declare it.
4. **Name the IC.** Convention: the on-call for the originating plane is the IC *by default*, until a more appropriate owner is named explicitly. The handoff is a message in the channel: "@alice taking IC from @bob at 02:14 UTC; context: …".
5. **Page the other Ops.** The IC pages the on-call for every plane and tenant implicated. Pages carry "Sev-N incident #inc-YYYYMMDD-NNN; your plane is affected; join #inc-… within 10 minutes".
6. **Open the war-room.** Video conference for Sev1; text channel suffices for Sev2. The war room is where the IC is; the incident channel is the record.
7. **Establish the initial posture.** The IC states: current hypothesis, current severity, current containment action, current ETA for next status update.

The first status update from the IC, within 15 minutes of declaration, is template-driven.

## The status update template

Every status update follows the same shape so leadership and tenants can parse it in ten seconds.

```
# Incident #inc-20261106-003 — Sev1 — status update 02:30 UTC

Summary: Shared MCP gateway's search endpoint returning 503 for ~18% of requests
since 01:47 UTC. Two T1 tenants (acme-financial, globex-health) see degraded
agent performance; one T2 tenant (initech-robotics) reports session failures.

Known: The MCP gateway's tool-search pagination (shipped in Q3) has a memory
leak that's hitting under the post-release traffic ramp.

Hypothesis: Memory pressure on the gateway pod set is the root cause; policy
plane is unaffected; model gateway and runtime are unaffected.

Containment: Rolled back MCP gateway to v3.4.1 at 02:22 UTC. Error rate
dropping; currently ~4% and falling. Expected sub-1% by 02:45 UTC.

Safety/security: No evidence of data-handling impact. SecOps joined at 02:18 UTC
for forensic review of request logs; nothing flagged.

Next update: 03:00 UTC or sooner if posture changes.

IC: @alice
Ops: @bob (MCP gateway), @carol (runtime, standby), @dave (SecOps)
Comms: @eve
Planning: @frank
```

Six features of this shape are load-bearing:

1. **The severity is at the top.** The reader may be a tenant, a sponsor, a board member. The severity sets their entire reading posture.
2. **Known / hypothesis / containment are separated.** Mixing them is how post-incident reviews end up chasing claims that were never supported.
3. **The safety/security line is present even if negative.** "No evidence" is a positive claim worth recording. Its absence is a red flag.
4. **The next update is scheduled.** Nobody has to ask "when will I hear more?"; the answer is in the previous update.
5. **The role roster is listed.** The channel shows who is doing what; nobody wonders who owns the SecOps angle.
6. **Timestamps are UTC.** This is a convention worth enforcing; incident timelines that mix time zones produce post-incident confusion.

The same template is the tenant-facing update, lightly adapted. The hypothesis and containment lines may be redacted or simplified for tenant audiences; the severity and the next-update time are not negotiable.

## Cross-team coordination patterns

Three concrete patterns the agent-platform IC uses repeatedly.

### Pattern 1 — Platform containment while product is in-session

**Scenario:** A shared model-gateway back-pressure is cascading into stream-aligned agents' sessions. The platform's containment lever is "drain traffic from provider X to provider Y" (mod-406). But some tenants' agents have active multi-turn sessions and a provider switch will reset context unless the SDK handles it.

**Coordination:**

- IC asks Product Ops (each tenant's on-call): "if we switch providers at 02:35 UTC, what is the blast radius on your active sessions?"
- Each Product Ops answers with: expected session failures, expected session continuations, expected customer impact.
- IC decides: switch now (contain the cascade, accept the session resets) or wait (let sessions drain, risk the cascade widening).
- Comms sends a pre-switch tenant notice: "At 02:35 UTC we will switch model providers to contain a latency spike; expect brief session resets for in-flight conversations."

The decision is the IC's; the data is the Product Ops'; the communication is Comms'. The IC does not debate the technical merits of the switch with each Product Ops; the IC collects the input, decides, and the decision is executed.

### Pattern 2 — Product misbehaviour that the platform can only partially contain

**Scenario:** A product-side agent, under what turns out to be prompt injection, is calling `send_email` on a shared MCP server with unusually high volume. The platform can rate-limit the shared tool; the real fix is on the product-side.

**Coordination:**

- IC asks Platform Ops (policy plane, MCP gateway): "what is the minimum rate-limit that stops the active abuse?"
- IC asks Product Ops (the tenant's on-call, plus `senior-agentic-ai-engineer` and `agentic-safety-engineer`): "what is the fix path on the product-side? what is the ETA?"
- IC decides containment: apply rate-limit at the policy plane *now*, scope it narrowly to the affected tenant, keep it in place until the product-side has shipped the fix.
- The rate-limit is reviewed in the post-incident; if it stays in place beyond 48 hours it becomes a durable policy change, which is a separate decision outside the incident scope.

The lever is at the platform; the root cause is at the product. The IC owns both the containment and the forcing function on the product-side fix. The tenant is in the loop via Comms, not via direct engineer-to-engineer messaging that bypasses the ICS structure.

### Pattern 3 — Suspected data exfiltration with SecOps and Risk in the loop

**Scenario:** Mid-incident, a responder notices that the MCP gateway's `fetch_url` tool call volume has spiked from one agent, with URLs that match an exfiltration pattern. This is now a security incident on top of the operational one.

**Coordination:**

- IC pages SecOps *immediately* (page, not message). SecOps joins the war room.
- IC pages Risk Ops (`ai-risk-engineer`) if the tenant is in a regulated industry.
- IC pages the `agentic-safety-engineer` on-call to assess whether this is a known attack pattern from mod-411's threat library.
- IC changes severity if not already Sev1.
- Containment splits into two tracks: (a) stop the immediate exfiltration (cut the agent's network egress, suspend the tool permission in the policy plane); (b) preserve evidence (SecOps controls what gets rolled back and what gets snapshotted first). The IC enforces the order — evidence-preservation steps execute *before* destructive containment where possible.
- Comms notifies leadership with a `may-have-data-exfiltration-implications` flag; the message explicitly does *not* claim a breach until SecOps confirms it.
- Legal notification, if required, is Comms' lever coordinated with SecOps and Risk. The IC does not freelance this.

The pattern is the one that most often goes wrong in incident reviews: the pressure to "just stop the bleeding" bypasses evidence preservation, and the post-incident forensic review is impossible. SecOps' joining the war room is the structural lever; the IC's authority to delay destructive containment until evidence is captured is the discipline.

## The post-incident review

The record the incident produces is the input to the review. The review itself is not a blameful event; it is a learning event, in the shape [Google's SRE post-mortem culture](https://sre.google/sre-book/postmortem-culture/) codified.

The review happens within one week of the incident for Sev1, within two weeks for Sev2. Attendees:

- The IC
- Every Ops lead who joined the incident
- Comms lead
- Planning lead
- Representatives from each affected tenant (optional, invited)
- The platform-team lead (as reviewer, not participant)
- For Sev1 with safety/security dimension: `agentic-safety-engineer`, `ai-risk-engineer`, SecOps

The review's output has four sections:

1. **Timeline.** Pulled from the incident record. One line per decision: timestamp, actor, decision, basis.
2. **What went well.** At least two things. Incident reviews that have no "went well" section corrode team confidence; they are not honest.
3. **What went poorly.** At least two things. Ideally these are not the same two things that showed up in last quarter's reviews — if they are, the remediation is systemic, not incident-specific.
4. **Action items.** Each action has an owner, a date, and a *proof* — what artefact or SLI will show the action was done? Action items without proofs are wish-list entries.

### The regression-fixture handoff

The action items with the longest half-life are the ones that become regression fixtures. [`mod-410` chapter 06](../mod-410-reliability-engineering-and-sre-for-agents/06-post-incident-regression-fixtures-into-the-eval-plane.md) runs this pipeline — the incident's reproducing input promotes to a fixture in the eval plane (mod-407), and the paved-road CI's eval-regression gate (mod-412 chapter 05) blocks any build that regresses on the fixture. This chapter's responsibility is making sure the handoff *happens*; the IC or Planning assigns the fixture-promotion action item before the review closes.

### The playbook-rev handoff

If any step in the incident surfaced a playbook gap (mod-410 chapter 05), the rev goes back to the playbook library the same week. A playbook that was wrong at 2 am will be wrong again at 2 am.

## Common failure modes

Six patterns that recur in cross-team agent-platform incidents.

### The leaderless incident

**Symptom:** Six Slack channels, nobody declared IC, decisions made by whoever shouts loudest.

**Cause:** No convention for who defaults to IC when the originating on-call pages across teams.

**Fix:** The convention in this chapter: the originating-plane on-call is IC by default until a handoff is explicit. The first message in the incident channel names an IC.

### The IC-debugging anti-pattern

**Symptom:** The person named IC is also the senior engineer debugging the root cause. Status updates degrade; coordination suffers.

**Cause:** Incentive to put the smartest person on the hard problem. Correct instinct; wrong role placement.

**Fix:** The IC role is explicitly non-debugging. If the smartest engineer is needed on the technical work, they are Ops lead; a different person is IC. The IC's authority comes from the role, not from technical depth on the current incident.

### The six-conversation incident

**Symptom:** Four war rooms, one per plane. No cross-plane coordination. The IC is in none of them.

**Cause:** Each plane's on-call defaulted to their own channel; nobody consolidated.

**Fix:** The incident channel is the *consolidation point*. Plane-specific threads inside that channel are fine; parallel war rooms are not. The IC enforces the consolidation on declaration.

### The silent-escalation trap

**Symptom:** The incident is Sev1 but was never explicitly declared Sev1; leadership finds out via a tenant complaint.

**Cause:** The on-call flinched at Sev1 because of the "page the sponsor" implication.

**Fix:** The severity decision is time-boxed to 15 minutes from the first page. "I haven't decided severity yet" past the 15-minute mark is itself a flag that the response has stalled.

### The evidence-destruction pattern

**Symptom:** Post-incident, SecOps cannot run forensic analysis because the containment action rolled back the state that had the evidence.

**Cause:** Destructive containment ran before evidence capture; SecOps was not in the war room early enough.

**Fix:** If any responder sees a signal of exfiltration, injection, or security breach, SecOps joins before destructive containment. This is one of the few hard rules of the discipline.

### The post-incident drift

**Symptom:** Action items from the review are open 90 days later. The next review's action items are the same action items.

**Cause:** Action items were wish-list entries without proof artefacts.

**Fix:** Every action item has an owner, a date, and a proof. The weekly roadmap review (chapter 01) scans the open action items; stale ones promote to the Now column of the roadmap or get explicitly closed as "we accept the risk".

## The communication ladder

Incidents expose communication choices that leadership and legal did not pre-authorise in writing. The rule of thumb: **three audiences, three cadences, three default postures.**

| Audience | Cadence | Default posture |
|---|---|---|
| Affected tenants | Every 30 minutes for Sev1; every 60 minutes for Sev2 | Informative, specific about blast radius; vague about root cause until confirmed. |
| Executive sponsor | Every hour; sooner if severity escalates | Terse; "here is the situation, here is who is working on it, here is what I need you to action". |
| Status page (if present) | State transitions only — declared / containing / recovered | Public-facing; written by Comms, approved by IC. |

Internal cross-team messages are separate — they are the war-room record, not a communication channel. Confusing internal coordination with external communication is one of the top-three post-incident findings in most organisations.

## When the platform IC is *not* the right IC

A principle this chapter closes on. The platform-team lead is the right IC when the incident originates in the platform, or when the platform is the primary coordination surface. But:

- **A safety incident with a frontier-model-abuse dimension** — `agentic-safety-engineer` is often the better IC. The platform IC supports.
- **A security incident with a data-breach dimension** — SecOps is the IC. The platform supports.
- **A regulatory incident requiring real-time legal counsel** — `ai-risk-engineer` or legal is the coordination surface; the platform supports.
- **A product-side incident with no platform root cause** — the stream-aligned team's on-call is the IC; the platform is on standby only if its planes are at risk.

Handing over IC is not a demotion. The handover message is terse: "@safety-oncall taking IC from @platform-lead at 02:48 UTC; this incident's primary dimension is frontier-safety; platform plane is standby." The record carries the handover.

## Summary

- Cross-team incident command is an adaptation of ICS for agent platforms: one Incident Commander, dedicated Operations / Communications / Planning leads, severity-driven response posture.
- Escalate to cross-team ICS when the incident spans platform and product, spans multiple tenants, has a safety/security dimension, has a regulatory implication, or will last more than two shifts.
- The IC owns priorities, severity, and the record. The IC does not debug.
- Status updates are template-driven and timeboxed; the severity, the known/hypothesis/containment split, the next-update time, and the role roster are not negotiable.
- Three coordination patterns recur: platform-containment-while-product-is-in-session, product-misbehaviour-that-platform-can-only-partially-contain, suspected-data-exfiltration-with-SecOps-and-Risk-in-the-loop.
- The post-incident review produces timeline / went-well / went-poorly / action-items; action items have owners, dates, and proofs; fixture-promotion and playbook-rev handoffs happen before the review closes.
- Six common failure modes to watch: leaderless incident, IC-debugging anti-pattern, six-conversation incident, silent-escalation trap, evidence-destruction pattern, post-incident drift.
- Hand IC over to the right role when the incident's primary dimension is safety, security, or regulatory; the platform IC supports.

The next chapter turns from the war-room moment back to the quieter one: *how the platform measures its own success* across quarters, using DORA, SPACE, platform-specific adoption, and support MTTR as the four complementary instruments.
