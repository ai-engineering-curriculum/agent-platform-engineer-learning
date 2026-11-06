# exercise-04: Incident Playbook for a Runaway Agent Loop

**Estimated effort:** 4 hours · **Output:** a versioned runaway-loop playbook, a reproducible staged incident against a stub agent, a trace snapshot preserved with incident-retention tag, a mod-407-shape fixture promoted into a mock catalogue (including pre-fix fail / post-fix pass verification), a signed post-mortem record referencing the fixture, and a short README (~1 page) · **Pairs with:** chapters [05](../05-incident-response-playbooks-for-agent-failure-modes.md) and [06](../06-post-incident-regression-fixtures-into-the-eval-plane.md); feeds project 403 and project 404.

## Objective

Build the full incident-to-fixture loop for one failure mode — the runaway agent loop — against a stub agent. You will author the playbook, stage the incident, run the playbook, capture a trace, promote a fixture, author the post-mortem, and verify the fixture catches the regression. The lesson is that every piece of this is an artefact, not a vibe; the exercise produces one of each.

Chapter 05 describes what a playbook is; this exercise builds one and exercises it. Chapter 06 describes the fixture pipeline; this exercise runs it end-to-end. The combined output is the muscle memory for the plane's primary obligation: incident response that durably prevents recurrence.

## Prerequisites

- You have read chapters [01](../01-the-sre-plane-and-what-it-owns.md), [05](../05-incident-response-playbooks-for-agent-failure-modes.md), and [06](../06-post-incident-regression-fixtures-into-the-eval-plane.md).
- You have completed [exercise 01](exercise-01-sli-and-slo-catalog-for-agent-workloads.md) — the stub agent is the subject; the SLIs are detection signals.
- You have a trace backend. Minimum: the stub agent writes newline-delimited span JSON to a file, and your scripts read from that file. Nicer: [Tempo](https://grafana.com/docs/tempo/latest/) or [Jaeger](https://www.jaegertracing.io/) running locally, with the stub agent emitting OTLP. Document whichever you pick.
- You have a mock mod-407 catalogue — a directory on disk under `mock-catalogue/` is sufficient for this exercise. Each fixture is one YAML file.
- You have a `platform-cli` stub — a bash wrapper or 100-LOC Python CLI that implements the subcommands this exercise calls (`session cancel`, `trace show --snapshot`, `eval replay`, `workload rollback`). The real CLI does not exist on your machine; this stub is what makes the exercise run.

## Problem statement

The `writer-v1` agent has an input shape that triggers a runaway loop. Your stub agent will deterministically loop on an input containing a specific marker (a document with two conflicting section headings, or whatever shape you can easily reproduce in 50 LOC). The loop:

- Consumes 30+ turns.
- Repeats a tool call with trivially-different arguments (defeating a naive loop detector).
- Would consume real model budget at wall-clock pace if the model gateway were real.

You will:

1. Author the runaway-loop playbook under `playbooks/runaway-loop-v1.md` using chapter 05's seven-section shape.
2. Stage the incident: run the stub agent with the loop-triggering input until the burn-rate alert from exercise 01 would fire.
3. Execute the playbook's "first five minutes" against the staged incident, capturing a trace snapshot.
4. Apply a containment lever (lower turn cap) and confirm the loop stops.
5. Promote a mod-407 fixture under `mock-catalogue/incident/<id>/row-01.yaml`.
6. Run the fixture against the pre-fix agent revision and confirm it *fails*.
7. Patch the stub agent (tighter loop detector), run the fixture against the post-fix revision, confirm it *passes*.
8. Author the post-mortem referencing the fixture.

## Requirements

### Part 1 — The playbook

`playbooks/runaway-loop-v1.md`. The seven sections from chapter 05:

- **Blurb.** One paragraph. What is a runaway loop, how is it different from a legitimate long session.
- **Detection signals.** The alerts (name + source), the attributes, the decision table. Reference the SLI names from exercise 01.
- **First five minutes.** Five numbered, imperative steps. Each one a specific `platform-cli` command, with expected output and an "if X then Y" branch.
- **Containment.** Five levers with scope / reversibility / blast-radius, matching chapter 05's lever shape.
- **Mitigation.** What to do after containment holds.
- **Recovery.** Specific SLI-threshold criteria — "no runaway sessions detected for 30 minutes", "cost-spike SLI returns to baseline".
- **Post-incident.** The bulleted invariants: record signed, fixture promoted, playbook revised if a step was wrong.

Pin a version (`v1`) and a last-exercised date (populate after the drill run).

### Part 2 — The staged incident

A stub agent and a loop-triggering input. The agent must:

- Emit `platform.session.lifecycle` events (`session_started`, per-turn `session_turn_started`, terminal state at end).
- Emit `gen_ai.tool.call` spans with `platform.agent.tool_arg_hash` attribute on each call.
- Emit `platform.session.cost_usd` as a running attribute on the session span.

The loop-triggering input is deterministic — same input produces the same loop every time. Document the input shape (e.g., a document with specific marker text).

A `scripts/stage-incident.sh` script that runs the stub agent with the loop-triggering input, produces a `trace-incident.ndjson`, and emits a Prometheus metric (`platform_session_turn_count`) that would fire the burn-rate alert from exercise 01.

### Part 3 — The playbook run

Execute the playbook's first-five-minutes against the staged trace file. Each step produces a visible output:

1. `platform-cli trace show <session_id>` → prints the last 10 spans.
2. `platform-cli session show --field agent_revision` → prints `git:<rev>`.
3. `platform-cli session cost <session_id>` → prints the running total.
4. `platform-cli session list --agent writer-v1 --turns-above 30 --last 10m` → prints a list (length 1 for this exercise).
5. Decision: contain this session (`platform-cli session cancel`) and lower the workload turn cap (`platform-cli workload update --max-turns-per-session 20`).

Capture each command's output in `playbook-run/output-<step>.txt`. The run is the evidence that the playbook is executable; a playbook whose steps don't produce the documented outputs is a playbook that needs revision.

### Part 4 — The trace snapshot

Run `platform-cli trace show --snapshot <session_id>` (the stub implementation copies the trace file to `snapshots/<incident_id>/trace.ndjson` and tags it with `retention: incident, retention_until: <today + 2y>`). Document the retention semantics even if the stub does not actually enforce them.

### Part 5 — The fixture row

`mock-catalogue/incident/<incident_id>/row-01.yaml` matching chapter 06's schema:

- `id`, `version`, `source: incident_promotion`.
- `provenance` block with `incident_id`, `signed_by`, `signed_at`, `trace_refs`.
- `target_workload: writer-v1`.
- `input` block with the system prompt version, the user turn, the attached documents (or document content hashes referencing files under `mock-catalogue/content/`).
- `rubric.deterministic_checks` including `turn_count_cap <= 15` and `cost_cap <= 0.50`.
- `rubric.judged_checks` with at least one "answer is non-null" check — stubbed if you do not have a real judge.
- `expected_outcome: pass`.
- `notes` explaining the regression.

### Part 6 — The verification runs

Two runs of the fixture against the stub agent:

1. **Pre-fix run.** `platform-cli eval replay --fixture mock-catalogue/incident/<id>/row-01 --agent-revision <pre_fix_rev>`. The output must show the fixture *fails* (the stub loops; turn-count-cap check fails). Save the output to `verification/pre-fix.txt`.
2. **Post-fix run.** Apply your fix to the stub agent (tighter loop detector — e.g., hash on document-position-range instead of exact-arg). `platform-cli eval replay ... --agent-revision <post_fix_rev>`. The output must show the fixture *passes*. Save to `verification/post-fix.txt`.

The two files together are the chapter 06 integrity-check evidence. A fixture without both is not promoted.

### Part 7 — The post-mortem

`post-mortems/inc-<date>-writer-loop.yaml` matching chapter 06's template:

- `incident_id`, `title`, `timeline` (dated events).
- `root_cause` paragraph.
- `mitigation` with the containment levers used and the recovery steps.
- `fixture_promoted.fixture_id` — the row-01 path from Part 5.
- `action_items` with owners and status.
- `sign_off.role: sre-plane-lead`, your name, today's date.

The post-mortem's `fixture_promoted.fixture_id` field is the enforced link. A post-mortem without it is not signed.

### Part 8 — The README

One page covering:

- How to run the full loop in one command: `./scripts/run-full-exercise.sh` — stages incident, executes playbook, captures snapshot, promotes fixture, runs verification, authors post-mortem.
- A pointer to each artefact (playbook, run outputs, snapshot, fixture, verification runs, post-mortem).
- One "I was surprised by" observation — e.g., "writing the first-five-minutes as `platform-cli` commands revealed that my stub CLI did not have a `session cost` command; I added it, but discovered that the cost attribute was a running total on the session span, not a dedicated metric — which means the real-world command would need to query the span store, not Prometheus."
- One open question for mod-407 or mod-412 — e.g., "my fixture's judge check is a stub; the real catalogue has calibration cards for each judge. Who authors the `judge-cards/presence-of-answer-v1.yaml` and how does calibration affect the fixture's gate behaviour over time?"

## Starter guidance

- **Make the stub agent's loop boringly reproducible.** A 50-LOC Python script that reads the input, counts the marker occurrences, and recursively re-calls itself with a `turn_count+1` is enough. The lesson is in the pipeline around the agent, not in the agent's cleverness.
- **Pick an incident id and use it consistently.** `inc-2026-11-06-writer-loop` or similar. The id threads through the snapshot path, the fixture namespace, and the post-mortem filename.
- **The CLI stub is a thin wrapper.** Each subcommand is 10–20 LOC: parse the args, do the trivial operation, print the result. The point of the CLI shape is reproducibility, not implementation depth.
- **Write the post-mortem *last*, after verification.** Writing it first tends to produce a post-mortem that argues for a fixture, not one that references a fixture that demonstrably works.
- **The redaction step matters even in a stub.** If your loop-triggering input is a document, show the redaction step before storing it as a fixture content — a `redact.sh` that lowercases proper nouns and masks digits is fine. The habit is the lesson.
- **Keep the verification runs simple and binary.** The pre-fix output should contain "FIXTURE FAILED"; the post-fix output "FIXTURE PASSED". The gate reads these; readability here matters.

## Acceptance criteria

Mark done when:

- [ ] `playbooks/runaway-loop-v1.md` exists with all seven sections and uses `platform-cli` commands throughout.
- [ ] The staged incident is reproducible: running `stage-incident.sh` twice produces the same session with the same loop behaviour.
- [ ] Each of the five playbook first-five-minutes steps has a captured output file.
- [ ] The trace snapshot exists under `snapshots/<incident_id>/`.
- [ ] `mock-catalogue/incident/<incident_id>/row-01.yaml` is schema-valid against chapter 06's template.
- [ ] Both verification runs exist: the pre-fix run output contains an unambiguous fixture-fail marker; the post-fix run contains a fixture-pass marker.
- [ ] The post-mortem is signed (your name + today's date) and references the fixture id.
- [ ] The README names each artefact and includes the "surprised by" observation and the open question.
- [ ] No step of the pipeline is manual-only; `run-full-exercise.sh` executes the whole loop end-to-end.

## Stretch goals

- **Author the context-exhaustion playbook (chapter 05, playbook 2).** Same seven-section shape, different detection signals, different levers. Reuse the stub agent with an "oversized attached document" input.
- **Author the tool-cascade meltdown playbook (chapter 05, playbook 4).** Needs a stub tool that degrades; the playbook's containment levers depend on mod-409's circuit-breaker surface, which you may need to stub.
- **Wire the fixture into a mock pre-merge gate.** A script that reads every fixture under `mock-catalogue/incident/*/row-*` and runs each against a candidate agent revision. Simulate a PR that reintroduces the loop and show the gate blocking it.
- **Build the pipeline-health metrics (chapter 06).** Promotion-latency, catch-rate, re-incident-rate as a tiny dashboard — can be a text-mode summary of the mock-catalogue state.
- **Add a `fixture_lifecycle` command.** `platform-cli fixture quarantine <id> --reason flaky` moves the fixture to a quarantined state. Document and implement the state transitions from chapter 06.
- **Cross-plane fixture scope.** The fixture row's gate behaviour should fire not just on `writer-v1` PRs but on PRs changing the paved-road SDK (mod-412) or the policy plane (mod-405). Author the cross-cutting gate configuration (matching chapter 06's "not running fixtures on cross-agent PRs" anti-pattern).
- **Tabletop review the playbook.** Walk a teammate (or a chapter-reading-partner) through the playbook as if they were on-call. Capture their first-run impressions; produce `playbooks/runaway-loop-v2.md` with the revisions. The before/after is the lesson.
