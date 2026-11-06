# exercise-04: Deprecation Ladder and Migration Playbook

**Estimated effort:** 2 hours · **Output:** a signed five-rung deprecation-ladder policy document, a worked migration playbook for an SDK rev (e.g., a `platform.session.id` → `platform.session.uuid` attribute rename), a tabletop transcript of running the playbook against three fictional tenants, a populated exception register with at least three entries, and a short README (~1 page) · **Pairs with:** chapter [07](../07-enablement-office-hours-migrations-and-deprecation-ladder.md); feeds `mod-413`.

## Objective

Translate chapter 07's enablement cadence into operable artefacts — the deprecation-ladder policy document, a worked migration playbook for a realistic SDK rev, and the exception register that absorbs the tenants who cannot migrate on the ladder's schedule.

Chapter 07 fixes the shape; this exercise is where you discover how much of the shape is *policy* (your call, binding) and how much is *procedure* (mechanical, derived from the manifest stream). The platform engineer who has never written the ladder as a signed document under-estimates how much of the ladder is political: a dwell window too short loses tenant trust, a dwell too long multiplies the SDK's support surface, and the exception register is the valve that keeps both failures from exploding.

## Prerequisites

- You have read chapter [07](../07-enablement-office-hours-migrations-and-deprecation-ladder.md).
- You have a conceptual `AgentRevision` manifest stream (from [exercise 03](exercise-03-adoption-metric-dashboard.md)) — or you will stub one for this exercise with three to six tenant entries.
- You have a text editor and a willingness to role-play. This exercise is mostly *writing* and *thinking*; the mechanisms are light.

## Problem statement

The platform-team lead has committed: "we will publish a deprecation-ladder policy and run every SDK major through a migration playbook". You own the first ladder and the first playbook.

The upcoming SDK rev is small-but-real: `platform.session.id` is renamed to `platform.session.uuid` to align with [W3C Trace Context](https://www.w3.org/TR/trace-context/) conventions. The rename ships as follows:

- SDK v3.0: emits both attributes (the shim).
- SDK v4.0: removes the shim; the attribute-contract test (chapter 05) checks only `platform.session.uuid`.

You must publish the deprecation ladder, author the migration playbook for this rev, run a tabletop against three fictional tenants, and record the exception-register entries that survive the tabletop.

The output has five parts:

1. The ladder policy document (`deprecation-ladder.md`).
2. The migration playbook for the attribute rename (`migrations/platform-session-id-to-uuid.md`).
3. The tabletop transcript (`tabletop-2026-11-06.md`).
4. The populated exception register (`exception-register/*.yaml`, at least three entries).
5. A short README (~1 page) with trade-offs, two "I was surprised by" observations, and one open question for mod-413.

## Requirements

### Part 1 — The deprecation-ladder policy document

A signed `deprecation-ladder.md` that includes:

- A one-paragraph preamble stating what the ladder is *for*.
- The five rungs with platform action, tenant signal, and minimum dwell. Tier-sensitive dwell (T1/T2/T3) is required.
- The exit criterion from each rung to the next, including the requirement that rung 5 depends on the manifest stream showing zero active references.
- The exception path at each rung.
- The approval and signature block (chapter 07 says mod-413 owns the signature; record who would sign).
- A versioning section — the policy has its own version (`ladder-policy/v1`), and future edits go through mod-413.

The policy is prescriptive. It says "T1 tenants get a minimum of X weeks of dwell at rung 1"; the number is your call but must be justified in a short rationale (two sentences max per tier).

### Part 2 — The migration playbook

Author `migrations/platform-session-id-to-uuid.md` using the chapter-07 playbook template. Specifically:

- Summary (one sentence describing the change).
- Reason (why — align with W3C Trace Context).
- Impact (enumerated from the manifest stream — for this exercise, invent 6-12 affected agents across 3-4 tenants; list by `tenant_id` and `agent_id`).
- Timeline — the five-rung schedule with real dates, grounded in the ladder-policy dwell numbers.
- What tenants need to do — concrete steps.
- Rollback plan — pinning back to v3 for 48h.
- Exception path — points to the exception register with the fields required.

The playbook must be *runnable*: a tenant reading it can act. Avoid "we recommend you review this" — prescribe the commands.

### Part 3 — The tabletop transcript

A facilitated tabletop exercise against three fictional tenants with different postures:

- **Tenant A — `acme-financial`** (T1): paved-road adopter, pulls the SDK rev willingly, migrates in the first week.
- **Tenant B — `globex-health`** (T2): has 8 agents and a compliance review cycle; cannot migrate all on the first-rung dwell; needs an exception.
- **Tenant C — `initech`** (T3): has an internal-tool agent on a fork of the SDK that pre-dates the paved-road rollout; owner is unreachable for two weeks.

Write the transcript as a dialog: the platform-team lead walks the migration; each tenant responds; the register captures the action. The transcript is ~1-2 pages. For each tenant, the transcript lands:

- What communication channel was used (office hours, 1:1, email).
- What the tenant's response was.
- What the register / ladder outcome is (migrate in time, filed ER, waited to retry contact).

### Part 4 — The exception register

At least three YAML files under `exception-register/`, one per surviving exception. Follow the chapter-07 shape:

- `ER-2026-0051.yaml` — `globex-health`'s blanket extension on their 8 agents due to the compliance review cycle. Named exit criterion is "re-review after the compliance sign-off, expected 2026-12-15".
- `ER-2026-0052.yaml` — `initech`'s fork-of-SDK agent. Named exit criterion is "owner returns 2026-11-20; migration lands by 2026-12-04 OR agent is decommissioned".
- `ER-2026-0053.yaml` — invent one additional entry: e.g., a workload that uses `platform.session.id` as a database key and cannot rename the column in the migration window; named exit criterion is "schema migration ships in Q1 2027".

Each ER must have: filed_at, filed_by, tenant, agents, rule violated, summary, justification, mitigations, exit criterion (type + description + target_date), owner, reviewer, status, and at least one review entry.

### Part 5 — README (~1 page)

- The dwell numbers you chose per tier, and a two-sentence rationale.
- Two "I was surprised by" observations — candidates: the ladder's rung-5 depends on the manifest stream showing zero active references; this turns out to be a surprisingly strong constraint. Or: the exception register's "target date" field is harder to fill in than it looks, because the target date is a function of the tenant's own internal schedule.
- One open question for mod-413 — e.g., "the ladder treats dwell as tier-sensitive; should it also be sensitivity-sensitive (e.g., a safety-plane change gets longer dwell than a template-cosmetic change)?".

## Starter guidance

- **Pick the dwell numbers before you write the playbook.** The playbook's timeline is the ladder's dwell applied to the specific dates; starting the playbook first means retrofitting the policy.
- **Keep the playbook's "what tenants need to do" short.** Chapter 07 stresses that the primary case should be `platform update`; everything else is exception territory. If your playbook has ten steps, you have missed the paved-road claim.
- **Write the tabletop in dialog form, not prose.** A dialog surfaces the moments where the policy has a tacit decision (e.g., "we don't extend an ER past 90 days without re-reviewing"). Prose glosses over those.
- **Make the exceptions *real*.** The temptation is to write three carbon-copy ERs; resist. Each tenant has a different reason; each ER has a different exit criterion. The register is only useful if the diversity of reasons is captured.
- **The signature block does not need real names.** `@platform-lead`, `@safety-plane-lead`, `@mod-413-chair` are fine role placeholders.
- **The "rung 5 depends on zero manifests" rule is the most interesting one.** Walk the policy document through why that rule exists; the README's "surprised by" item probably lives here.

## Acceptance criteria

Mark done when:

- [ ] `deprecation-ladder.md` includes all five rungs with tier-sensitive dwell numbers and a two-sentence rationale per tier.
- [ ] `migrations/platform-session-id-to-uuid.md` follows the chapter-07 template; a tenant reading it could migrate without asking further questions.
- [ ] `tabletop-2026-11-06.md` is ~1-2 pages of dialog covering the three tenants and three dispositions.
- [ ] `exception-register/` contains at least three ERs with different exit-criterion types and different dates.
- [ ] Every ER has a named owner, a named reviewer, and at least one review entry (status approved or needs-more-info).
- [ ] `deprecation-ladder.md` is signed (the signature block is filled in with role names).
- [ ] README includes the two "surprised by" observations and the open question.
- [ ] No ER has an indefinite / blank exit date.

## Stretch goals

- **Write the "zero manifests" query.** The SQL (or DuckDB) that reads the manifest stream from exercise 03 and reports whether any agent still references the deprecated attribute. Attach it to the playbook as the mechanical rung-5 check.
- **Write the FAQ delta.** Four or five entries that would land in the paved-road docs' FAQ as a result of this migration (chapter 07's "FAQ maintained from office-hours log" shape).
- **Run a second tabletop with harder tenants.** A tenant whose workload class is not yet on the paved road; a tenant whose previous ER expired without re-review; a tenant whose primary engineer resigned. These are the cases the first tabletop usually misses.
- **Author the mod-413 review summary.** A ~200-word summary of the exercise's output as it would appear in the quarterly review packet; chapter 07's quarterly cadence consumes it.
- **Propose a change to the policy.** Based on the tabletop, propose one amendment to `deprecation-ladder.md` (e.g., "add a 24-hour no-rollback window at rung 3 for T1 tenants"). Record it as a proposed `ladder-policy/v2` diff.
- **Model a bad path.** A tabletop where the policy *fails* — the dwell was too short, two tenants were surprised, the register gained five entries in a week. Write the retro. Chapter 07's "common ladder failure modes" is the source of candidate scenarios.
- **Draft the sunset notice.** The actual communication that goes out at rung 1 (release notes, email to tenants). Chapter 07 mentions this but does not show the shape; produce one.
