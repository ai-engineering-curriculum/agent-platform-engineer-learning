# Enablement, Office Hours, Migrations, and the Deprecation Ladder

Chapter 06 pinned the measurement. This chapter is the practice — the weekly, monthly, and quarterly rituals that turn "the paved road exists" into "the paved road is the default road". Tools without a cadence are shelfware; this is the cadence.

Four artefacts carry the enablement work:

1. **The onboarding ritual** — the one-session guided path from "I have a repo idea" to "I shipped my first paved-road agent".
2. **Office hours** — the weekly open session where triage, FAQs, and the exception-register intake happen.
3. **The migration playbook template** — the reusable shape for "we are changing something; here is how a tenant moves".
4. **The deprecation ladder** — the five-rung policy document that makes removing an old adapter, SDK major, or template a predictable, auditable event rather than an argument.

All four feed the exception register — the single source of truth for "this tenant has a documented reason to depart from the paved road, with an owner and an exit criterion".

## The onboarding ritual

The first forty-five minutes a new engineer or a new tenant spends with the paved road. Everything the paved road claims has to feel real inside the forty-five minutes.

### Structure (45 minutes)

- **0–5 min — the one-page map.** A single diagram: the eleven planes in the track, the SDK as the composition layer, the engineer's agent repo at the top, the KPIs at the bottom. The engineer should leave the whole session able to point at each box without looking.
- **5–10 min — `platform new agent`.** Run the scaffolder live; produce a repo. The engineer sees the ninety-second budget met in real time.
- **10–20 min — `make dev` and the first turn.** Bring up the local-dev loop; `POST /turn`; inspect the trace viewer; see the composition the trace confirms matches the map.
- **20–30 min — `make eval`.** Run the fixture replay; see the verdict; inspect the eval report.
- **30–40 min — open the first PR.** Push to a scratch branch; the paved-road CI runs; the engineer sees the gates and the PR comment.
- **40–45 min — the exception register and office hours.** Show the register's web page; show the office-hours calendar; name the next three sessions.

Four properties this format forces:

1. **Everything is a real action, not a slide.** The engineer leaves the session with a working repo in their git history and a PR link.
2. **The map comes first.** An engineer who runs the scaffolder without the map does not know *why* the output has eight files. The map is five minutes; the saved confusion is hours.
3. **The office hours are shown as a place to go.** The onboarding does not promise "you know everything". It promises "you know where to come next".
4. **The session has a repeatable script.** Not a slide deck per host; the ritual is a markdown document in the paved-road docs, used verbatim.

### Who runs it

Options, in order of scale:

- **A paved-road team member runs it ad-hoc.** Works for the first dozen onboardings; does not scale.
- **A volunteer rota of recent adopters runs it.** Works at the next scale; builds social capital in the adopter community; chapter 06's pull signal strengthens.
- **A self-service video version with a Slack-based check-in.** The right form at mature scale; the engineer runs the ritual alone against the docs; a bot checks in on day 2 and day 7.

`time-to-first-agent` (chapter 06) is the measurement of whether the ritual is working. A median above a day suggests the onboarding is broken or absent.

## Office hours

A recurring weekly session — one hour, same day and time, calendar invite to a broad distribution — where anyone can bring a paved-road question. This is the lowest-cost, highest-leverage ritual the enablement cadence runs.

### Shape

- **Weekly, one hour.** Same slot; attendance is not required.
- **Open attendance.** Anyone on a stream-aligned team; anyone from leadership; anyone from adjacent planes.
- **Light structure.** Five minutes of "what shipped this week"; the rest is Q&A against questions filed in advance via a form and questions raised live.
- **Recording and summary.** Recording lives in the paved-road docs; a one-paragraph summary goes to the paved-road channel; the FAQ (below) picks up the recurring questions.
- **A paved-road engineer on duty.** Rotates weekly; the "on duty" person is responsible for the summary and for driving the recurring questions into the FAQ.

### What office hours capture

Three signals the dashboard (chapter 06) misses:

1. **The qualitative lead indicator.** Three weeks in a row of "my build is slow because the attribute-contract test takes thirty seconds" means the test needs a fast path. The dashboard would not have surfaced this until the complaint became a Rate drop.
2. **The exception-register intake.** A tenant who thinks they need an exception is encouraged to bring it to office hours first; the paved-road team can often provide a cheaper path than an exception.
3. **The adjacent-plane handoff.** A question "my eval regression gate is slow" may be a mod-407 issue; the office-hours host triages to the right plane.

### The office-hours question record

A machine-readable log — one row per question — makes the enablement roadmap data-driven:

```yaml
# .platform/office-hours-log/2026-11-06.yaml
session_date: 2026-11-06
host: @maya-paved-road
attendees: 14
questions:
  - question: "attribute-contract test takes 30s on our monorepo; can it be made faster?"
    tenant: globex-health
    disposition: filed-issue
    issue_ref: platform/paved-road-sdk#1247
    topic_tags: [ci-performance, attribute-contract]
  - question: "we need to call the Anthropic batch API; the paved-road adapter does not expose it"
    tenant: initech
    disposition: routed-to-exception-register
    exception_ref: ER-2026-0051
    topic_tags: [adapter-extension, anthropic]
```

Chapter 06's scorecard reads the topic tags over the quarter; a tag that repeats is a chapter-07 roadmap item.

## FAQ and self-service documentation

Office hours generate recurring questions; the FAQ is where they live so the next asker finds the answer without another session. Two rules:

1. **The FAQ is maintained from the office-hours log, not written in advance.** The questions are the real ones; the pre-written FAQ is the wish list.
2. **Every FAQ entry has a last-reviewed-on date.** An FAQ entry from 2025 against an SDK that has since rev'd twice is worse than no entry.

Where the FAQ sits (one of):

- **The paved-road docs site** ([Docusaurus](https://docusaurus.io/), [MkDocs Material](https://squidfunk.github.io/mkdocs-material/), [Backstage TechDocs](https://backstage.io/docs/features/techdocs/techdocs-overview/)).
- **A dedicated Backstage plugin page that overlays the template catalogue.**

Both are fine; what matters is that `platform doctor` (chapter 03) and the office-hours summaries link to the FAQ by stable URL.

## Migration playbooks

The paved road changes. A new adapter, a renamed `gen_ai.*` attribute, a template major bump, a dependency upgrade — each is a change that tenants have to absorb. Without a playbook, each tenant discovers the change at a different time, in a different way, and the aggregate cost of the change is multiplied by the tenant count.

### The migration-playbook template

Every paved-road change that requires tenant action ships a playbook in the same shape. The template:

```markdown
# Migration playbook: {{ change_name }}

## Summary
- Change: {{ one-sentence description }}
- Reason: {{ why this is happening — upstream rev, incident, policy }}
- Impact: {{ which agents are affected; read from AgentRevisionStream }}
- Timeline: {{ announce-date / deadline / end-of-grace }}
- Owner: {{ paved-road engineer running the migration }}

## Who is affected

A query against AgentRevisionStream that enumerates the affected manifests, by tenant and agent id.
Attach the query output as of the playbook's publish date.

## What the change does

A plain-English description of the before and after, plus a pointer to the SDK /
template /  CI change that implements it.

## What tenants need to do

Step-by-step. If `platform update` plus a template-catalogue bump covers it, say so.
If a hand-edit is required, show the diff.

### 1. Verify your repo is affected
```
$ platform doctor
# observe the "update available" or "migration required" line
```

### 2. Apply the update
```
$ platform update
# review the diff
# re-run `make test` and `make eval`
```

### 3. (If applicable) Hand-edits
Describe any manual steps the regeneration does not cover. Reference the
exception-register process if your repo is one of the documented exceptions.

### 4. Open the migration PR
Standard paved-road CI applies. Reference this playbook in the PR description.

## Timeline

| Rung | Date         | What happens |
|------|--------------|--------------|
| Announce | 2026-11-06 | Playbook published; office-hours slot added |
| Warn     | 2026-11-20 | `platform doctor` warns; CI advisory note |
| Default-off | 2026-12-04 | New scaffolds no longer get the old shape |
| Removed-from-scaffolder | 2026-12-18 | Old template retired from catalogue; new only |
| Deleted | 2027-01-15 | Old code deleted; builds against it fail hard |

## Rollback plan

If the migration goes wrong. What a tenant does to pin back to the previous rung for
24-48h while the platform team investigates. Usually: `platform update --to v2` with
a documented downgrade note.

## Exception path

If a tenant genuinely cannot migrate by the deadline. Points to the exception-register
form; names the expected exit criteria.
```

Four properties:

1. **Everyone affected is enumerated at publish time.** The playbook is not "broad guidance"; it is a specific instrument with named addressees.
2. **The timeline is the deprecation ladder** (below). The playbook is where the ladder becomes concrete for one specific change.
3. **The rollback path is first-class.** A migration without a rollback path is a migration that will stall on the first tenant surprise.
4. **The exception path is named.** The playbook does not leave "what if I cannot do this" implicit; chapter 07 is explicit about when an exception is the right answer.

### Worked example: SDK v2 → v3 attribute rename

A worked outline of what a non-trivial paved-road migration looks like:

- **Change:** `platform.session.id` attribute renamed to `platform.session.uuid` to align with [W3C Trace Context](https://www.w3.org/TR/trace-context/) conventions.
- **Reason:** Observability plane (mod-408) chapter 05 rev.
- **Impact:** 142 paved-road agents; 7 tenants.
- **Timeline:**
  - 2026-11-06 — v3 released with both attributes emitted (shim).
  - 2026-11-20 — `platform doctor` warns when `.session.id` is referenced; CI advisory.
  - 2026-12-04 — new scaffolds use `.session.uuid` only; existing dashboards updated.
  - 2026-12-18 — the shim is removed from the SDK; agents that have not run `platform update` fail the attribute-contract gate.
  - 2027-01-15 — v2 SDK packages are deleted from the registry.
- **Tenant steps:** `platform update`; verify dashboards; re-run `make eval`; open the PR.
- **Rollback:** pin to SDK v2.9.x via `paved_road_sdk = "~=2.9"` for 48h while filing an issue.
- **Exception:** if the tenant has a custom dashboard that cannot be updated in the window, file an ER with evidence and the exit date.

The playbook is one markdown file in the paved-road docs; the deprecation-ladder table above is the governance contract.

## The deprecation ladder

A five-rung ladder. Each rung has a declared duration (a minimum grace window) and a specific action the paved-road takes. The ladder is why tenants trust the paved road: they know that nothing goes away without this sequence, and they know that everything eventually completes the sequence.

### The five rungs

| Rung | Platform action | Tenant signal | Minimum dwell |
|---|---|---|---|
| **1. Announced** | Deprecation announced in release notes, FAQ, office hours. Migration playbook published. | None that blocks; a note in `platform doctor`. | 2 weeks (T3), 4 weeks (T2), 8 weeks (T1 by default). |
| **2. Warn** | `platform doctor` warns; CI prints an advisory note on PR; office-hours tracks adoption. | A visible notice; no CI failure. | 2 weeks (T3), 4 weeks (T2), 8 weeks (T1). |
| **3. Default-off** | New scaffolds no longer produce the deprecated shape; existing agents continue to build. | `platform new` cannot select the old shape; existing agents unaffected. | 2 weeks (T3), 4 weeks (T2), 8 weeks (T1). |
| **4. Removed-from-scaffolder** | Template removed from catalogue (`status: retired`); SDK symbol deprecated but still importable. | New scaffolds of the old shape fail; existing agents see a deprecation warning at import. | 2 weeks (T3), 4 weeks (T2), 8 weeks (T1). |
| **5. Deleted** | Symbol / template / adapter deleted from the codebase and the package registry. | Builds against the deleted shape fail hard; exception-register entries required for any remaining usage. | — |

Four load-bearing details:

1. **Dwell is tier-sensitive, not calendar-fixed.** A T1 tenant deserves longer notice than a T3 internal-tool tenant because the consequences of a surprise are higher. The platform's commitment is minimum dwell, not maximum; tier is the modifier.
2. **Dwell is tied to zero-active-manifests, not a wall-clock date.** Each rung's exit is "the policy's dwell has elapsed AND the manifest stream shows zero active references to the deprecated shape". If one tenant lingers at rung 3, the ladder pauses at rung 3 for that tenant while the paved-road team contacts them; the global ladder advances only when the manifests allow.
3. **Rung 5 is genuinely destructive.** Deletions are not reversible from the tenant side; by the time you reach rung 5, no manifest references the shape and the exception register covers the few that do.
4. **The ladder is a document.** The ladder's policy sits in the paved-road docs as a signed document; mod-413 governs the signature; the specific migration playbook (above) is the instantiation.

### The ladder in practice

Three questions the ladder answers for every change.

- **When do we announce?** Rung 1 happens when the change is merged. The announcement is a release note; the playbook is simultaneously published. Never announce a deprecation whose replacement is not yet shipped.
- **When do we remove?** Rung 5 happens when the manifest stream shows zero active references for two consecutive weekly windows, AND the dwell on rung 4 has elapsed. Both conditions must hold; either alone is insufficient.
- **What if a tenant blocks?** A tenant whose manifest shows the deprecated shape past rung 3's dwell gets a named contact: an office-hours slot, a one-off migration-assistance offer, and an exception-register entry if the migration cannot land in time. Rung 5 is not reached while a non-exception tenant is still on the shape; the ladder is a commitment.

### Common ladder failure modes

- **Deprecation announced without playbook.** The tenants have no action to take; adoption of the replacement stalls; the ladder gets stuck at rung 2. Always ship the playbook with the announcement.
- **Dwell too short.** T1 dwell measured in weeks rather than quarters; tenant trust evaporates. Dwell is a commitment; make it a generous one and honour it.
- **Dwell too long.** Rung 4 takes a year; deprecations stack; the SDK has a dozen shims simultaneously. The paved-road team's own cost grows; the "clean" story erodes. Keep dwell generous *and* bounded.
- **No telemetry.** Rung 5 is decided by rumour ("I think everyone has moved") instead of the manifest stream. Always wait for the zero-reference signal.
- **Exceptions treated as permanent.** An exception-register entry whose exit criterion is "indefinite" is a fork, not an exception. Chapter 07's exception policy enforces a dated exit; absent a date, the ER is not approved.

## The exception register

The single source of truth for "this tenant has a documented reason to depart from the paved road, with an owner and an exit criterion".

### Shape

```yaml
# exception-register/ER-2026-0051.yaml
schema_version: exception/v1
id: ER-2026-0051
filed_at: 2026-11-04
filed_by: team-initech-ops
tenant: initech
agents: [internal-ops-copilot]
paved_road_rule_violated: adapter-boundary
summary: Direct Anthropic batch API call for cost-sensitive batch job; paved-road adapter does not yet expose this.
justification: >
  The batch API cuts our cost per session by ~40% for a workload that runs daily
  overnight. The paved-road adapter's streaming interface is not applicable.
mitigations:
  - Scoped the escape-hatch block in src/agent.py with the explicit tag.
  - The attribute-contract gate still passes (we fill the required attributes manually).
  - The CI's cost-latency smoke gate exercises this path.
exit_criterion:
  type: paved-road-change
  description: The adapter adds a batch interface; the paved road is restored via `platform update`.
  target_date: 2027-03-01
owner: @initech-ops-lead
reviewer: @paved-road-sdk-maintainers
status: approved
reviews:
  - date: 2026-11-04
    reviewer: @paved-road-sdk-maintainers
    disposition: approved
    notes: Scoped and reviewable; batch API is on the roadmap for Q1 2027.
```

Five invariants:

1. **A rule is named.** Not "the paved road"; the specific rule (adapter-boundary, cost-budget override, template-mandatory-field, deprecated-shape-use). The register reads back the rule set.
2. **A tenant is named.** No "general" exceptions; each is tied to specific agents.
3. **An exit criterion is required.** Either a paved-road-change (the paved road absorbs the case), a tenant-change (the tenant eliminates the dependency), or a retirement (the agent is decommissioned before the criterion must land).
4. **A target date is required.** Not "indefinite"; the date may be extended in a re-review, but it is never blank.
5. **The register is reviewed on cadence.** Quarterly (mod-413 owns it). An ER whose target date has passed is reviewed; either the paved road ships the change, the tenant ships theirs, or the ER is re-approved with an extended date and a written reason.

### Adoption-dashboard interaction

Chapter 06's dashboard splits "Fully on paved road" from "On paved road with exception". An agent with an active ER counts in the second column; the headline Rate is the sum. This is what prevents an ER bloat from masking as adoption.

## Office-hours playbook — the host's checklist

A reusable checklist for the on-duty engineer. The ritual is public; the checklist is internal.

- [ ] Pull the office-hours intake form; triage questions into topic tags.
- [ ] Open with "what shipped this week" — one-line summary of merges to paved-road repos, new template versions, SDK releases.
- [ ] Answer intake questions in order; route adjacent-plane questions to those owners live.
- [ ] Capture any question that is a candidate ER; walk the asker through the register form during the session.
- [ ] Capture any question that is a candidate roadmap item (recurring, or significant-impact single asker); file the issue during the session.
- [ ] Write the YAML question record (above); publish the recording and the one-paragraph summary.
- [ ] Update the FAQ for any question that recurred (second occurrence creates an entry; third occurrence promotes it to the top of the FAQ).

## The quarterly cadence

A once-per-quarter review that mod-413 governs. The paved-road plane's contribution:

- **Publish the scorecard** (chapter 06).
- **Publish the exception register summary** — total open, newly filed, closed, ER-dwell-distribution.
- **Publish the deprecation ladder's state** — what is at each rung; expected next moves.
- **Publish the enablement-cadence summary** — number of onboardings, office-hours attendance and topic distribution, FAQ changes.
- **Name one roadmap commitment for the next quarter.** Not four; one. The forcing function is "what is the biggest lever for `time-to-first-agent`, pull-signal strength, or deployment success rate over the next thirteen weeks".

The cadence is the mod-413 artefact; the paved-road plane's artefacts are the inputs. Chapter 06 and chapter 07 together produce everything the quarterly review consumes.

## Common pitfalls when operating enablement

- **"We don't need office hours."** The absence of office hours routes every question into DMs; the paved-road team's load spikes; the FAQ never materialises. Office hours are cheap and high-leverage; always run them.
- **The onboarding is a slide deck.** The forty-five minutes are a *working session*; the engineer leaves with a real agent and a real PR. A slide deck is a failed onboarding.
- **Migration playbooks that assume the deprecation ladder.** An ad-hoc migration without a playbook is an unsigned deprecation; tenants cannot tell whether the ladder applies. Always publish a playbook per migration.
- **Dwell cut short because leadership wants the fleet-wide cleanup now.** The ladder is a commitment. Cutting dwell once costs the paved road more trust than the cleanup saves.
- **Exception-register entries that never close.** Quarterly review them; close any whose exit criterion has been met; extend any whose reason is still live; retire any whose agent has been decommissioned.
- **Mixing enablement with on-call.** Enablement questions ("how do I upgrade the SDK") go to office hours. Production incidents ("my agent is 500-ing") go to the mod-410 on-call. One channel per concern.
- **Treating the paved road as "finished".** The scaffolder, the SDK, the CI, the local-dev loop — each rev twice a quarter in a healthy platform. A paved road that has not shipped a change in two quarters is drifting out of relevance.
- **Letting the "we'll rewrite the SDK" temptation win.** When a major rev is tempting, chapter 02 rule 1 (additive within a major) is your discipline; chapter 07's ladder is your audit. Major rewrites are a last resort; migrations are the norm.

## Summary

- Enablement is a cadence, not a document set. Four artefacts: the onboarding ritual, weekly office hours, migration playbooks per change, the five-rung deprecation ladder.
- The onboarding ritual is forty-five minutes of real action, not slides; `time-to-first-agent` (chapter 06) measures its effectiveness.
- Office hours are weekly, one hour, open, with a machine-readable question log that feeds the FAQ and the roadmap.
- Migration playbooks follow a single template: summary, who is affected (enumerated from the manifest stream), what to do, the timeline (the deprecation ladder's rungs), the rollback path, and the exception path.
- The deprecation ladder has five rungs — Announced, Warn, Default-off, Removed-from-scaffolder, Deleted — with tier-sensitive dwell and a hard dependency on the manifest stream showing zero active references before rung 5.
- The exception register is the single source of truth for documented departures from the paved road. Every ER has a named rule, named tenant, named exit criterion, named target date, and quarterly review.
- Four roles in the quarterly cadence: paved-road plane publishes the scorecard, the ER summary, the ladder state, and the enablement summary; mod-413 governs the review.
- Common failure modes: no office hours, slide-deck onboarding, playbook-less migration, cut-short dwell, zombie exceptions, mixed enablement / on-call, drift-into-rewrite. Each has an obvious fix; the discipline is naming them before they happen.

The paved road is now a plane. Chapter 06's dashboard says who is on it; this chapter says how they stay on it; chapters 01–05 say what "on it" means. The track's next module, [`mod-413`](../mod-413-platform-leadership-and-operating-cadence/), reads these artefacts into the leadership and operating cadence that governs the platform as a whole.
