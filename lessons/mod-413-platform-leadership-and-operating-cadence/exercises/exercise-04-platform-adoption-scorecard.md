# exercise-04: Platform Adoption Scorecard (Four-Instrument Composition)

**Estimated effort:** 2 hours · **Output:** a `metrics.yaml` formalising the DORA + SPACE + adoption + support-MTTR formulas, a synthetic multi-stream dataset, a one-page `scorecard-Q4-2026.md` that composes all four instruments, a five-question survey template, a vanity-metric red-team note, and a one-paragraph reflection · **Pairs with:** chapter [04](../04-measuring-platform-success.md); builds on [`mod-412` chapter 06](../../mod-412-paved-road-sdks-and-developer-experience/06-measuring-adoption-and-cncf-maturity.md) and [exercise 03](../../mod-412-paved-road-sdks-and-developer-experience/exercises/exercise-03-adoption-metric-dashboard.md) in mod-412.

## Objective

Produce the one-page scorecard the quarterly review signs. Chapter 04 pinned the four instruments (DORA, SPACE, platform-adoption KPIs, support-ticket MTTR); [`mod-412` chapter 06](../../mod-412-paved-road-sdks-and-developer-experience/06-measuring-adoption-and-cncf-maturity.md) pinned the paved-road-specific KPIs; this exercise *composes* them into a single artefact plus the discipline (survey template, metrics file, red-team) that keeps the scorecard honest.

Where mod-412 exercise 03 asked you to render the four adoption KPIs on a dashboard, this exercise asks you to roll them up into a scorecard that reads across all four instruments and is read by leadership, not by the platform team.

## Prerequisites

- You have read chapter [04](../04-measuring-platform-success.md).
- You have scanned [`mod-412` chapter 06](../../mod-412-paved-road-sdks-and-developer-experience/06-measuring-adoption-and-cncf-maturity.md); ideally you have completed [`mod-412` exercise 03](../../mod-412-paved-road-sdks-and-developer-experience/exercises/exercise-03-adoption-metric-dashboard.md) and have the manifest / CI / deploy streams available.
- You have completed [exercise 01](exercise-01-platform-roadmap-authored-against-stream-teams.md) so the tenant register and roadmap are known.
- You have skimmed the primary sources in `resources.md` (Google DORA, SPACE paper, DevEx framework). <!-- needs-research: DORA band thresholds move year-over-year in the State of DevOps report; verify against the current release before publishing the scorecard. -->

## Problem statement

Your platform-team is closing Q4. The sponsor review is in two weeks and the tenant council is in four. You own the scorecard that lands in both. The scorecard must:

- Fit on one page.
- Compose DORA, SPACE, platform-adoption KPIs (from mod-412), and support-ticket MTTR.
- Be signed by three parties (platform-team-lead, executive-sponsor, tenant-council-chair).
- Include a narrative paragraph that *reads across* the four instruments and makes one or two load-bearing observations the sponsor should act on.

You also author the metrics file, the survey template, and the vanity-metric red-team note that keep the scorecard honest over multiple quarters.

## Requirements

### Part 1 — The composed metrics file

A `metrics.yaml` that *extends* the mod-412 chapter 06 metrics file (or starts fresh) with the chapter-04 instruments. Each metric carries:

- `id`
- `title`
- `description` (English)
- `instrument` — one of `dora`, `space`, `adoption`, `support`
- `formula` or `source` — SQL or a reference to the stream / survey instrument
- `inputs`
- `dimensions` (tenant, workload_class, platform_vs_tenant_caused, …)
- `window`
- `owner`
- `last_reviewed_at`

Entries you must include:

- The four DORA metrics (Deployment Frequency, Lead Time for Changes, Change Failure Rate, MTTR), measured against tenant deploys, with the platform-caused vs tenant-caused split for Change Failure Rate and MTTR.
- The five SPACE dimensions (Satisfaction, Performance, Activity, Communication, Efficiency), sourced from the quarterly survey (Satisfaction, Performance self-rated, Efficiency) plus the activity log (Activity) plus the office-hours log (Communication).
- The four adoption KPIs from [`mod-412` chapter 06](../../mod-412-paved-road-sdks-and-developer-experience/06-measuring-adoption-and-cncf-maturity.md) (Platform Adoption Rate, Time-to-First-Agent, Deployment Success Rate, Rollback Count).
- The four support-MTTR metrics (Ticket volume by topic, Support MTTR by topic, Reopened rate, Deflection rate).

Add a `reading_guidance` section: the chapter-04 "read-together discipline" table, adapted for your platform. Focus on three or four conjunctions (not all of them; focus on the ones most likely to appear).

### Part 2 — The synthetic dataset

Extend (or author fresh) a `streams/` directory with:

- `agent_revision_stream.jsonl` — 90 days of manifests (reuse mod-412 exercise 03 or generate fresh for four tenants).
- `ci_event_stream.jsonl` — 90 days of CI runs.
- `deploy_event_stream.jsonl` — 90 days of deploys, with the platform-caused vs tenant-caused attribution on failures.
- `incident_record_stream.jsonl` — ~25 incident records across the 90 days (mix of Sev1 / Sev2 / Sev3), each with MTTR and a platform-vs-tenant root-cause attribution.
- `support_ticket_stream.jsonl` — ~125 tickets across the 90 days with topic clustering (`sdk_bug`, `onboarding`, `eval`, `safety`, `policy`, `other`), opened_at, resolved_at, reopened_at (nullable), resolution_type (`fix`, `docs`, `answered`, `duplicate`, `unresolved`).
- `space_survey_responses.jsonl` — 47 responses across four tenants and two workload classes. Five answers per response plus free-text. Use a plausible rating distribution that has some skew (e.g., three tenants around 4.0–4.4 Satisfaction, one tenant around 3.1).

The synthetic data must exercise:

- At least one platform-caused Change Failure Rate contribution.
- At least one tenant where Satisfaction is materially lower than the average.
- At least one topic cluster whose share of ticket volume has visibly shifted quarter-on-quarter.
- At least one incident where MTTR is elite (< 1 hour) and at least one where it is not.
- At least one SPACE conjunction from chapter 04's read-together table (e.g., Adoption rising + Satisfaction falling → mandate-adoption).

### Part 3 — The one-page scorecard

A `scorecard-Q4-2026.md` that follows the chapter-04 template verbatim, with your synthetic numbers populating it:

- Header: quarter, generation timestamp, owner.
- DORA block (4 metrics, split by platform vs tenant where applicable).
- SPACE block (5 dimensions, N respondents).
- Platform-adoption block (4 KPIs with splits).
- Support block (volume by topic, MTTR, reopened rate, deflection rate).
- Incidents block (counts, MTTR, action-item status).
- Narrative paragraph (~120 words) that reads across all four instruments and states one or two load-bearing observations.
- Signature block (three signers).

The scorecard must fit on one page. If it does not, trim — chapter 04 is explicit on this.

### Part 4 — The quarterly survey template

A `survey-template.md` with the chapter-04 five questions (adapted to your platform's specifics). Include:

- Platform-specific framing in the preamble (e.g., "this survey goes to every stream-aligned engineer who committed to a paved-road repo in the last quarter").
- The anonymisation and aggregation policy (anonymous per-response; aggregated by tenant and workload class).
- The cadence (quarterly, same five questions every quarter, next review in the Q1 scorecard).
- The free-text clustering methodology one-pager (how you cluster: topic-by-hand for N<100 responses; add `<!-- needs-research: ... -->` on any specific NLP-cluster tool you reference).

### Part 5 — The vanity-metric red-team note

A short `vanity-red-team.md` (~300 words) that walks through chapter 04's five vanity-metric failure modes and answers *for your scorecard specifically*:

- DORA-only reporting — are the other three instruments visible?
- Satisfaction-average-hides-skew — is the per-tenant split shown?
- Adoption-rate-without-exceptions — is the fully-on-paved-road / with-exception / invisible split shown?
- MTTR-mean-hides-tail — is p50 *and* p95 *and* open-older-than-30-days shown for support MTTR?
- DORA-measuring-the-platform-team — is the primary DORA panel on tenant deploys, not platform-team deploys?

For each failure mode, name the specific mitigation in your scorecard. If a failure mode is not mitigated, fix the scorecard.

### Part 6 — Reflection paragraph

A one-paragraph `reflection.md` (~120 words) answering: *the four instruments disagreed somewhere in your synthetic data. Where? What did the narrative paragraph say about it? What decision did it inform?*

The disagreement-across-instruments is where the scorecard earns its keep. Numbers that all agree are easy; numbers that disagree force the read-together discipline chapter 04 argues for.

## Starter guidance

- **Lift mod-412 exercise 03's stream generator if available.** This exercise is additive: you are composing the chapter-04 instruments *onto* the paved-road KPIs that exercise already renders. Do not re-author from scratch.
- **Pin formulas before generating data.** Chapter 04's instrumentation discipline is explicit — formulas live in the metrics file, not in the data generator. Write them first.
- **Make the survey small.** Five questions. Not ten. Chapter 04 is explicit; adding questions is how surveys degrade.
- **The scorecard is for the sponsor, not the team.** The platform team gets the dashboard; the sponsor gets this one page. A sponsor-scorecard that shows internal infrastructure metrics is failing its audience.
- **The narrative paragraph is where the value is.** The numbers are the evidence; the paragraph is the argument. Spend half your Part 3 time on the paragraph.
- **Use real DORA band language sparingly.** The Elite / High / Medium / Low bands come from the State of DevOps report; if you quote a specific threshold, cite the current report and mark `<!-- needs-research: ... -->` where the threshold may have moved. The band labels drift; the four metrics do not.
- **The disagreement-across-instruments is the exercise.** A synthetic dataset where DORA, SPACE, adoption, and support all point the same direction is either too clean or too lucky. Engineer at least one disagreement into the data.
- **Signatures matter.** Even in a fictional scorecard, write the three signers' names and dates. The scorecard becomes an artefact in the governance log; the format is the format.

## Acceptance criteria

Mark done when:

- [ ] `metrics.yaml` has all 17 metrics listed in Part 1, each with instrument / formula / inputs / dimensions / window / owner / last_reviewed_at.
- [ ] The metrics file includes a `reading_guidance` section with at least three chapter-04 conjunctions explained.
- [ ] `streams/` contains the five JSONL files with realistic Q4 data over 90 days.
- [ ] At least one platform-caused Change Failure Rate contribution is present.
- [ ] At least one tenant is materially lower on Satisfaction than the average.
- [ ] At least one topic cluster shift is visible in the support-ticket stream.
- [ ] `scorecard-Q4-2026.md` fits on one page and covers DORA / SPACE / adoption / support / incidents / narrative / signatures.
- [ ] The scorecard shows platform-caused vs tenant-caused splits on Change Failure Rate and MTTR.
- [ ] The scorecard shows fully-on-paved-road / with-exception / invisible splits on Adoption Rate.
- [ ] The scorecard shows p50 and p95 and open-older-than-30-days on support MTTR.
- [ ] `survey-template.md` has five questions with preamble, anonymisation policy, and cadence.
- [ ] `vanity-red-team.md` walks all five chapter-04 failure modes with specific mitigations.
- [ ] `reflection.md` names at least one instrument disagreement in your data and the decision it informed.

## Stretch goals

- **Author the scorecard-versus-last-quarter diff.** Generate a Q3 dataset as well and produce a `scorecard-Q3-to-Q4-diff.md` showing the changes. Chapter 04's scorecard emphasises deltas; this makes them tangible.
- **Build the sponsor-facing slide.** A single-slide (markdown or a real slide tool) summary of the scorecard. Three bullets plus the narrative paragraph. The exercise is to practise the compression: a one-page scorecard already prunes; the single slide prunes further.
- **Automate the scorecard generation.** A short script (`scorecard/generate.py`) that reads `metrics.yaml` and the streams, computes every metric, renders the markdown, and leaves the narrative paragraph as a `# TODO: author the narrative` placeholder the platform-team-lead writes by hand. The script is exactly as automated as it should be — the numbers are generated, the reading is not.
- **Add the DevEx framework cross-check.** Chapter 04 cites the DevEx framework (Noda, Forsgren, Alexander, Storey). For each of your SPACE dimensions, cross-reference which DevEx component (feedback loops, cognitive load, flow state) it maps to. Produce a short `devex-cross-reference.md` note.
- **Author an "instrument disagreement" decision record.** Chapter 04 §read-together discipline is explicit that disagreements are the signal. Take the one disagreement your synthetic data produced, author a short `instrument-disagreement-2026-Q4.md` that the platform-team-lead would bring to the sponsor review. Three sections: which instruments disagree, what each is likely telling you, what action is being proposed.
- **Red-team the SPACE Activity metric.** Chapter 04 is explicit: Activity is the easiest to mis-read. Write a short note on the Activity number in your scorecard and defend why it is reported at all. If you cannot defend it, drop it; the scorecard improves.
- **Build the attendance log for the quarterly council.** Chapter 01 says "attendance is a signal in its own right". Produce a plausible attendance log for the Q4 council: who was invited, who attended, who sent a delegate, who did not respond. The attendance alone does not go in the scorecard; the summary line in the "open risks" section does.
