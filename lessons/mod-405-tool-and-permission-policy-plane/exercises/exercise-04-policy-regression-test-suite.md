# exercise-04: Policy Regression Test Suite with an Abuse Corpus and Differential Harness

**Estimated effort:** 4 hours · **Output:** a versioned abuse-corpus directory, a differential harness script, a CI job that gates merges on widening changes, and a report (~1 page) against your exercise-01 policy · **Pairs with:** chapter [06](../06-policy-regression-tests-and-the-abuse-corpus.md); feeds project 403.

## Objective

Build the artefacts that prove — not assert — the plane's policies refuse what they were written to refuse. You will construct a versioned abuse corpus spanning three attack classes, a differential harness that compares two bundle versions over the corpus plus a traffic sample, and a CI job that fails a PR when a change widens access without the required annotation. The deliverable is the piece the chapter-07 contract with `agentic-safety-engineer` plugs into.

A well-done exercise produces a harness that catches an intentional widening you commit yourself. If the harness green-lights a widening, your gate is wrong.

## Prerequisites

- You have completed exercise 01 (Rego capability policy) and have its policy and test files at hand. Exercise 02 (Cedar) is useful but not required.
- You have read chapter [06](../06-policy-regression-tests-and-the-abuse-corpus.md).
- You have OPA installed and can run `opa eval` and `opa test` on your policy.
- If you did exercise 02, you have Cedar's analyser available for the SMT-side differential.
- You are comfortable writing shell / Python / Go — the harness is yours to author.

## Problem statement

Build three things:

1. The versioned abuse corpus. Fixtures organised by attack class, with expected decisions.
2. The differential harness. Takes two bundles, runs them against (corpus + traffic sample), emits a diff report.
3. The CI gate. Blocks merge if the diff report contains unannotated widenings.

Then use the harness on yourself: commit a widening to the exercise-01 policy; run the harness; show it catches the widening; propose a required annotation.

## Requirements

### Part 1 — The abuse corpus (versioned fixtures)

Create `abuse-corpus/` with three attack-class directories:

- `jailbreak-smuggled-invocation/`
- `confused-deputy/`
- `over-privileged-scope-leak/`

Author at least four entries per class. Each entry is a pair:

- `<slug>.input.json` — the Rego input the harness will present.
- `<slug>.expected.json` — the expected decision envelope fields.

Minimum entries per class:

**Jailbreak-smuggled-invocation (4+ entries):**
- `subject-swap.input.json` — the capability's `subject_predicate` expects `principal.subject`; the input's argument is a different user. Expected: `deny / capability.subject_predicate_violation`.
- `scope-escalation-via-argument.input.json` — tool permitted with `*.read`; arguments imply write semantics. Expected: `deny / capability.argument_constraint_violation`.
- `expired-capability.input.json` — capability's `expires_at` is before the environment's `timestamp`. Expected: `deny / capability.expired`.
- `version-out-of-range.input.json` — tool_version does not satisfy the capability's `tool_version_constraint`. Expected: `deny / capability.version.out_of_range`.

**Confused-deputy (4+ entries):**
- `broker-missing-decision-id.input.json` — broker.lease action with a `parent_decision.decision_id` not present in the (mock) decision log. Expected: `deny / broker.decision_id_not_allowed`.
- `audience-mismatch.input.json` — an action that references a token whose `aud` is a different resource server than the requested tool. Expected: `deny / token.audience_mismatch`.
- `missing-actor-claim.input.json` — an action where the `session.actor.spiffe_id` is missing. Expected: `deny / session.actor.missing`.
- `trust-domain-mismatch.input.json` — an action where the actor's SPIFFE ID is in an unfederated trust domain. Expected: `deny / svid.trust_domain_unfederated`.

**Over-privileged-scope-leak (4+ entries):**
- `unused-scope-detection.input.json` — a test that runs against the decision-log replay (not the hot-path policy); the fixture includes a replay window and expects a drift-report entry naming `over_privilege.unused_scope`.
- `wildcard-persistence.input.json` — a session capability granted `notes-mcp/*` persists after a bundle change restricting the pattern; expected policy behaviour depends on your plane's commitment (chapter 06 Part 3) — document which and assert.
- `approval-bypass-via-session-persistence.input.json` — a high-severity action invoked once with approval, then re-invoked. Expected: second invocation re-triggers approval obligation.
- `scope-upgrade-via-refresh.input.json` — the gateway refreshes a session-broad token and the authorization server returns a wider scope; expected: the policy still enforces the pre-refresh narrower scope.

Where your exercise-01 policy does not cover an attack (e.g., the broker-side `confused-deputy` entries presume a broker policy you did not write), mark the entry with `skip_reason: ...` in the expected JSON — but *still include it*. A corpus is a living artefact; chapter 07's contract has `agentic-safety-engineer` contributing new entries over time.

### Part 2 — The differential harness

Author `diff/harness.py` (or `.go`, `.sh`; language of your choice). The harness:

- Takes two bundle paths (`--baseline`, `--candidate`).
- Takes a set of input paths (`--inputs path/to/corpus`).
- Runs each input against each bundle using `opa eval -d <bundle> -i <input> 'data.platform.tool.invoke.decision'`.
- Emits a `diff-report.json` with one entry per input:

  ```json
  {
    "input": "path/to/fixture.input.json",
    "baseline": { "decision": "deny", "reason_code": "capability.no_match" },
    "candidate": { "decision": "allow", "reason_code": "capability.scope.exact_match" },
    "direction": "widening",
    "tool_id": "calendar-mcp/find_free_slot",
    "action_kind": "tool.invoke"
  }
  ```

- `direction` is one of `agree | widening | narrowing | reason_code_changed | obligation_changed`.
- Latency is not critical for this exercise, but the harness should scale linearly with corpus size and emit progress every N inputs.
- The harness exits non-zero when the report contains any `widening` entry.

### Part 3 — The CI gate

Author `.github/workflows/policy-diff.yml` (or equivalent). On every PR touching `policies/`:

- Check out the PR branch.
- Build the candidate bundle (`opa build policies/ -o candidate.tar.gz`).
- Fetch the baseline bundle from a known location (for the exercise, the previous `main` commit's bundle built in CI; simulate with a stub if needed).
- Run the harness with both bundles and the corpus.
- If the harness reports widenings:
  - Fail the build with a message listing them.
  - Require an annotation in the PR body — a specific `widening-approved:` line for each widening, plus a reference to an `ai-risk-engineer` review (for this exercise, a stub URL).
- Upload the `diff-report.json` as an artefact on the workflow run.

### Part 4 — Use the harness on yourself

- Commit a trivial widening to the exercise-01 policy: remove the `subject_predicate_satisfied` check from the allow rule.
- Run the harness locally (`./diff/harness.py --baseline main.tar.gz --candidate pr.tar.gz --inputs abuse-corpus/`).
- Confirm the `subject-swap` entry flips from baseline-deny to candidate-allow, reported as `widening`.
- Commit, open a PR, confirm CI fails.
- Revert, confirm CI goes green.

### Part 5 — The report

One page covering:

- **Corpus coverage.** How many entries per attack class, and which plane components each entry exercises.
- **Harness design choices.** Why you chose the inputs-as-files shape (versus an in-memory fixture library), how you handle `skip_reason` entries, what the harness does when the baseline and candidate both deny but with different reason codes.
- **The widening-you-caught.** The subject-predicate removal, the diff-report entry, the CI message a reviewer sees.
- **Limitations.** Three honest limitations: corpus items you could not express, over-privileged-scope-leak tests that require a replay store you do not have, attack classes your corpus does not yet cover.
- **What `agentic-safety-engineer` adds.** One sentence on what the chapter-07 contract expects as a new finding's artefact and how your corpus structure accommodates it.

## Starter guidance

- Keep the harness small. 150 lines of Python is enough for the base exercise; the clever code is in the corpus fixtures.
- `opa eval` on a bundle produces JSON on stdout; parse it, compare to the expected envelope. The decision is at `result[0].expressions[0].value` for the invocation above.
- Fixture inputs are ergonomic when they inherit from a `base.input.json` with the common session shape — your fixtures only need to override the deltas. A tiny `merge.py` makes the corpus half the size and much more readable.
- Do not try to normalise reason codes across bundles if one bundle has a different vocabulary — the whole point of the harness is to flag vocabulary changes. Report them as `reason_code_changed`.
- Running `opa eval` per fixture is slow at scale. For this exercise it is fine. In production, batch via `opa exec` or compile to WASM and evaluate in-process.
- `<!-- needs-research: ... -->` anything you cannot verify against the current OPA release — notably `opa exec` and `opa build` flag names.

## Acceptance criteria

Mark done when:

- [ ] The abuse corpus has at least 12 entries total (4+ per attack class), each with an `.input.json` and an `.expected.json`.
- [ ] The harness runs end to end over the corpus against two bundles and produces a readable `diff-report.json`.
- [ ] CI blocks a PR that widens the policy; the CI message tells the reviewer what widened and what annotation would unblock it.
- [ ] The self-widening test proves the harness catches a real widening.
- [ ] The report names the coverage, the design choices, the caught widening, the limitations, and the hand-off from `agentic-safety-engineer`.

## Stretch goals

- **Narrowing report.** Extend the harness to annotate narrowings: a changed policy that *denies* a previously-allowed input. Render a readable summary — narrowings may be desired (deprecations) or accidental (regressions). Have the CI print them for review but not fail.
- **SMT differential with Cedar.** If you did exercise 02, add a separate CI step using `cedar analyse` on the Cedar policies. Compare the SMT output to the replay-based output on the same change; note the differences. Often SMT catches things the replay misses (because the corpus does not cover them), and vice versa.
- **Canary deny-rate check.** Simulate a canary: run the candidate against a larger traffic sample (synthesise 10,000 inputs), compute the deny-rate by reason code, compare to baseline, flag reason codes whose deny rate changed by more than N%. This is the chapter-06 canary idea at exercise scale.
- **Signed decision logs.** Have the harness emit decision logs for each run and verify signatures across runs. A log that cannot prove it has not been truncated is not an audit log; the exercise previews this.
- **Corpus contribution flow.** Document the shape in which `agentic-safety-engineer` would contribute a new finding to the corpus: PR template, required fields, review responsibilities. Attach as `CONTRIBUTING-corpus.md`.
