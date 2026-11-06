# exercise-01: Rego Policy for Agent Tool Invocation

**Estimated effort:** 3 hours · **Output:** one Rego policy package (two files minimum), one Rego test file, one `opa test` CI job, and a short README (~1 page) explaining the capability shape · **Pairs with:** chapter [02](../02-opa-and-rego-for-agent-tool-invocation.md); feeds exercises 02, 03, and 04, and projects 402 and 403.

## Objective

Author the OPA / Rego policy that decides, for a single tool invocation, whether the session's capability bundle currently authorises it — enforcing subject predicates, argument-level constraints, version constraints, and expiry. The deliverable is the shape every agent tool call in your plane runs through; chapter 02's examples are the starting point, not the destination.

A well-authored policy makes the chapter-01 decision envelope (envelope, decision, reason code, obligations) produce reliably on well-formed inputs, and fail closed on ill-formed inputs with a reason code the on-call dashboard understands. If your policy produces an allow on an input the chapter-02 abuse list rejects — or produces a deny with a reason code that cannot be grouped — your admission reviewer will send it back.

## Prerequisites

- You have read chapter [01](../01-the-policy-plane-and-the-authorization-triad.md) and chapter [02](../02-opa-and-rego-for-agent-tool-invocation.md).
- You have [OPA](https://www.openpolicyagent.org/docs/latest/) installed locally (`opa version` reports 0.70 or later as of this writing — verify against the current release line; <!-- needs-research: confirm the current OPA stable release at your reading time -->).
- You have the [Rego policy reference](https://www.openpolicyagent.org/docs/latest/policy-reference/) and the [policy-testing guide](https://www.openpolicyagent.org/docs/latest/policy-testing/) open.
- You have chapter 04 of mod-403 and chapter 02 of this module open for the decision-input shape.

## Problem statement

You are writing the policy for a *single tool* on the plane, end to end. Pick one of the following, or pitch your own tool with a similar shape — the acceptance criteria scale.

- **`calendar-mcp/find_free_slot`** — reads a user's calendar availability. Scope `calendar.read`. Argument shape `{attendee: string, range: {start: iso8601, end: iso8601}}`. The subject predicate the capability enforces is `attendee == principal.subject`.
- **`crm-mcp/update_contact_notes`** — writes notes against a CRM contact. Scope `crm.contact.write_notes`. Argument shape `{contact_id: string, note: string, visibility: "private" | "team" | "tenant"}`. The argument constraint the capability enforces is `visibility in session.allowed_visibilities`.
- **`warehouse-mcp/query`** — runs a SQL query against a tenant's data warehouse. Scope `warehouse.query_read`. Argument shape `{sql: string, max_rows: integer}`. The argument constraint enforces `max_rows <= capability.argument_constraints.max_rows_ceiling` and `is_read_only(sql)` (expressed as a Rego helper that permits only SELECT / WITH).

## Requirements

### Part 1 — Input and envelope shape

Start by committing to the input shape your policy consumes. Reuse chapter 02's example as the base. Add any fields your chosen tool requires (e.g. `visibility` for the CRM tool, `max_rows` for the warehouse tool). The shape is the contract between your gateway and the PDP; a missing field fails closed.

Write a short `input-shape.schema.json` (JSON Schema draft 2020-12 or similar) that documents it. The policy does not use this schema at evaluation time — OPA is schema-less by convention — but reviewers and tests do. The schema is also what mod-403 chapter 03's registry entry points at.

### Part 2 — The policy file (main deliverable)

Author `policies/capability.rego` with the following responsibilities:

- `default decision := {deny, "capability.no_match", ...}`.
- `matching_capabilities` rule that finds capabilities covering `(tool_id, requested_scope)`, respecting the `tool_version_constraint` (use `semver.compare`).
- `subject_predicate_satisfied(cap)` that handles the predicate flavours your tool declares; at minimum handle `attendee_match: "principal.subject"` or an equivalent.
- `argument_constraints_satisfied(cap)` that handles the argument constraints relevant to your tool (range-length, max-rows, allowed-values list).
- `allowed_capability := cap` composed from the above.
- `decision := {...}` on both allow and every deny branch, each with a distinct `reason_code` drawn from the closed vocabulary.
- An obligation on allow that at minimum carries `evidence.record` with the field set the compliance stream will capture.
- `deny_reasons contains reason` as a partial rule that emits every applicable reason so the dashboard can distinguish them.

Fail closed on:

- Missing `capabilities` on the session input.
- Missing `expires_at` or an unparseable timestamp.
- Unknown `action.kind`.

### Part 3 — The test file

Author `policies/capability_test.rego`. Write at least:

- **4 positive tests** — different valid capability shapes, including one with no subject predicate (allow) and one with an argument constraint that just barely satisfies (edge-case allow).
- **8 negative tests** — one per reason code you emit (`capability.no_match`, `capability.scope.mismatch`, `capability.subject_predicate_violation`, `capability.argument_constraint_violation`, `capability.expired`, `capability.version.out_of_range`, `capability.exhausted`, `capability.tool.revoked.emergency` if your scenario supports it).
- **3 input-shape tests** — missing fields, unparseable timestamps, unknown action kind. All deny, each with a distinct reason code or a shared `input.malformed` code with a `reason_text` that distinguishes.

Each test uses `with input as {...}` and asserts both `.decision` and `.reason_code`. A test that only asserts `.decision` is a stub and does not count.

### Part 4 — The CI job

Write a minimal CI configuration (GitHub Actions `.github/workflows/opa-test.yml` or equivalent) that:

- Runs `opa fmt --diff` (formatting check).
- Runs `opa check policies/` (static check).
- Runs `opa test policies/` (unit tests).
- Fails the build if any of the above fails.
- Is readable in under 30 lines.

If your plane uses `conftest` rather than raw OPA for CI (many do), author the equivalent `conftest` invocation.

### Part 5 — The README

One page covering:

- Which tool you authored for, and why its capability shape is representative of the plane's tool population.
- The input-shape decisions — anything you chose to represent as a top-level field rather than a nested one, and why.
- The reason-code vocabulary you emit, with a one-line description of each.
- The three failure modes you encode (missing fields, timestamp failures, action-kind mismatch) and how an on-call engineer diagnoses each from a decision log.
- One open question you would route to `ai-risk-engineer` or `agentic-safety-engineer` (chapter 07). Example: "the warehouse tool's argument constraint on `max_rows` is 10,000; do we need a harm-model entry for cross-record aggregation above a threshold?"

## Starter guidance

- Do not try to make your policy reusable across tools yet. One tool, one policy, one set of tests. Chapter 02 intentionally does not scale up; the exercise does not either.
- Prefer `some cap in input.session.capabilities; cap.X == Y` over `input.session.capabilities[_].X == Y`. The former is readable; the latter is Rego's legacy shape and reviewers struggle.
- `semver.compare` and `semver.is_valid` are built-ins — use them for version constraints. Rolling your own semver with regex is a regression trap.
- `time.parse_rfc3339_ns` returns nanoseconds. Compare *nanoseconds to nanoseconds*; mixing `time.now_ns()` with an unparsed string silently coerces and leads to subtle bugs. Parse once at the top of the rule and compare the parsed value.
- The `evidence.record` obligation on allow should list the fields the gateway will attach to the compliance record. Do not list `arguments` as a whole if the arguments contain user content — list `arguments_hash` plus the policy-relevant projection.
- Commit your test input fixtures as separate JSON files under `policies/testdata/` once you have more than four — inline `with input as {...}` becomes unreadable fast.
- Flag `<!-- needs-research: ... -->` on anything you cannot verify against the current OPA release notes (notably WASM-build compatibility for `time.*` and `semver.*` built-ins).

## Acceptance criteria

Mark done when:

- [ ] The policy evaluates deterministically on well-formed inputs and fails closed on each of the three malformed-input classes.
- [ ] The policy emits an obligation on allow that names the fields the gateway will record.
- [ ] `opa test policies/` reports at least 15 tests, all green, in under 5 seconds.
- [ ] Every deny reason the policy can emit has at least one test asserting it fires on the matching input.
- [ ] The input-shape schema documents every field the policy reads.
- [ ] The CI job runs end to end on an untouched clone of the exercise directory.
- [ ] The README names the single tool, the reason-code vocabulary, the failure modes, and one open question for `ai-risk-engineer`.

## Stretch goals

- **Build to WASM.** Compile the policy to WASM (`opa build -t wasm`) and run a single test against the compiled form from a non-Go host (TypeScript via `@open-policy-agent/opa-wasm` or Python via `opa-wasm`). Note any built-ins that did not compile and the workarounds.
- **Add a rate-limit obligation.** On allow, emit a `rate_limit` obligation with window and max; wire a stub PEP that honours it. Write a test proving the obligation ID is echoed on the decision log when the limit is discharged.
- **Decision-log signing.** Configure OPA to emit decision logs to stdout with Cosign-signed JSON (or OPA's bundle-signing key). Write a verifier script that reads a log file and confirms every entry's signature.
- **Negative differential.** Save the current `capability.rego` and intentionally widen one rule (e.g. remove the subject predicate). Run both versions against your test suite with a diff script you write; produce a one-paragraph report of what widened. This previews exercise 04's differential harness.
- **Admission test integration.** Package the policy as the admission bundle a mod-403 registry entry would reference. The exercise 01 of mod-403 exported a schema; pair this policy with that schema so admission CI can run both.
