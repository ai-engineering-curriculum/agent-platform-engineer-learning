# exercise-04: Dataset Versioning and Provenance

**Estimated effort:** 3 hours · **Output:** one catalogue service (schema, publish API, fetch API, diff tooling), one row envelope validator, a seeded two-version dataset with lineage, a Rego access-control policy with tests, and a short README (~1 page) · **Pairs with:** chapters [03](../03-the-eval-dataset-management-plane.md) and [01](../01-the-evaluation-plane-and-what-it-owns.md); feeds project 403.

## Objective

Build the minimum shape of the plane's dataset management plane: a catalogue over content-addressed objects that enforces the mandatory row envelope (provenance, PII class, licence, train-cutoff relevance), produces immutable versions with lineage, labels synthetic data at the schema level, and gates read access through a policy evaluated against a SPIFFE-authenticated caller. The deliverable is the shape every `catalogue://` URI in exercises 01–03 resolves against.

A well-authored catalogue refuses an envelope-defective row at publish, serves a pinned version by hash deterministically, and refuses a read whose SPIFFE identity does not clear the access policy. A catalogue that lets a synthetic row through with `provenance.reviewed_by: null` into a release-blocking fixture is a catalogue a reviewer sends back.

## Prerequisites

- You have read chapters [01](../01-the-evaluation-plane-and-what-it-owns.md) and [03](../03-the-eval-dataset-management-plane.md).
- You have [OPA](https://www.openpolicyagent.org/docs/latest/) installed (or [Cedar](https://www.cedarpolicy.com/) if you prefer; the exercise is in the policy content, not the engine).
- You have a local object store surrogate (filesystem directory, MinIO, or a real cloud bucket).
- Familiarity with [DVC](https://dvc.org/doc) or [LakeFS](https://docs.lakefs.io/) is helpful but not required; this exercise re-implements the thin catalogue layer by hand so the primitives are visible.

## Problem statement

Pick a dataset shape to catalogue — reuse a fixture from exercise 01 if you built one; otherwise:

- **`writer-v1-ragas`** — 30 rows, each a question + reference passage + ground-truth answer. Mix of `human-labelled` (20 rows) and `synthetic:writer-v1-adversarial@v1` (10 rows). Licence split: 20 rows CC-BY-SA-4.0, 10 rows internal-eval-only.
- **`support-agent-regression`** — 50 rows, each a customer query + an ideal response, derived from `replayed:<session-id>` sources. All rows are `pii: high` under the `support-agent` tenant.
- **`extractor-strict-schema`** — 40 rows, each a document + a required JSON output. All rows `human-labelled`, licence `MIT`, `pii: none`.

(Row counts and specific licence choices are illustrative; flag `<!-- needs-research: ... -->` on any specific threshold you cannot defend from first principles or an `ai-eval-engineer` reference.)

## Requirements

### Part 1 — The envelope and schema

Author `schema/row-envelope.schema.json` enforcing the chapter-03 envelope:

- `row_id` — stable string.
- `provenance.kind` — enum (`human-labelled`, `synthetic:<generator-ref>`, `replayed:<source-ref>`, `public-benchmark:<name>`, `partner:<vendor>`). The pattern must *reject* a free-form string.
- `provenance.authored_by` — SPIFFE URI pattern.
- `provenance.authored_at` — RFC 3339 timestamp.
- `pii_class` — enum (`none`, `pii`, `phi`, `financial`, `secret`).
- `licence` — SPDX identifier (or a controlled internal identifier).
- `train_cutoff_relevance.earliest_safe_train_cutoff` — ISO-8601 date.
- `columns` — open object for task-specific fields.

Author `schema/row-writer-v1-ragas.schema.json` (or equivalent for your chosen fixture) extending the envelope with the task-specific columns.

A publish-time validator must reject a row that fails either schema. Commit the validator's test suite alongside — a mix of `publish-should-reject` and `publish-should-accept` fixtures.

### Part 2 — The publish API

Author `catalogue/` with a `publish` endpoint (CLI or HTTP; CLI is simpler) that:

- Accepts a `name`, a `schema_ref`, a row iterator (JSONL file or stdin), an `access_policy_ref`, and a parent version (or `null` for the first version).
- Validates every row against the envelope and task schemas.
- Content-hashes every row (SHA-256 over the canonicalised JSON) and writes it to `objects/<hash>.json`.
- Computes a `version_hash` over the sorted list of `(row_id, row_content_hash, provenance.kind)` tuples.
- Writes a manifest to `manifests/<name>/<version_hash>.yaml` carrying every field in chapter 03's manifest example — row count, publisher SPIFFE id, publication timestamp, parent version pointer, licence split, provenance split, `pii_class`, `access_policy_ref`.
- Refuses to flip the `@latest` tag — that is a separate `promote` command requiring a reviewer identity.

### Part 3 — The fetch API

Author a fetch endpoint that:

- Resolves `catalogue://<name>@<ref>` where `<ref>` is one of `latest`, a semantic `v<N>`, or a `sha256:...` pointer.
- Returns the manifest *plus* a streamed row iterator, with the manifest hash verified against the sorted-tuple reconstruction before any row is yielded (if the manifest and object store disagree, the fetch refuses).
- Takes a SPIFFE workload identity (from a header for the exercise), evaluates the manifest's `access_policy_ref` against it, and refuses on deny (returning a structured refusal reason).
- Emits a `catalogue.read_row` audit event per granted read and a `catalogue.access_refused` event per deny.

### Part 4 — The Rego access policy

Author `policies/catalogue-access.rego` covering at least:

- `allow` when the resource `pii_class == "none"` and the licence is in a public-permissive set.
- `allow` when the resource `pii_class in {"pii", "phi"}`, the subject's tenant matches the resource's `publisher_tenant`, and the subject has an `eval-runner-in-tenancy` role claim.
- `deny` unconditionally when `pii_class == "secret"`.
- `allow` on `publish_version` for subjects with the `eval-author` role.
- `allow` on `publish_tag_latest` only for subjects with the `eval-approver` role (so a reviewer flips `@latest`, not the author).

Write a test suite (`policies/catalogue-access_test.rego`) covering: allow / deny for every clause, a secret-class resource with a mis-written `allow` rule proving the hard-deny wins, a cross-tenant read on a PII fixture proving the tenant-match requirement.

### Part 5 — The diff and promote commands

Author:

- `catalogue diff <name> <v_a> <v_b>` — surfaces added rows, removed rows, content-diffed rows (row ids present in both with different content hashes), and provenance-only changes (row ids present in both with same content hash but different provenance metadata).
- `catalogue promote <name> <version> --tag latest --approver <spiffe-id>` — flips the `@latest` pointer, logging the approver. The command refuses when the approver role does not clear the `publish_tag_latest` policy.

### Part 6 — The seeded two-version dataset

Produce two versions of your chosen fixture:

- **v1** — the initial publish. Verify the manifest hash is reproducible from the object-store contents.
- **v2** — a reviewed revision: add 5 synthetic rows (with proper `provenance.kind: synthetic:<generator>`, a non-null `reviewed_by`, and a `source_material` pointer); correct the reference answer on 2 rows (content-diff); mark 1 row deprecated via removal.
- Run `catalogue diff v1 v2` and commit the output.
- Run `catalogue promote <name> v2 --tag latest --approver spiffe://platform.local/eval/alice`; commit the audit record.

### Part 7 — The README

One page covering:

- Which dataset shape you picked and why.
- The envelope-rejection fixtures and what a bad row looks like.
- The two policy "hard rule" outcomes (secret-class deny wins; cross-tenant deny on PII) with the exact test-run output.
- The v1 → v2 diff narrative.
- One open question for `ai-eval-engineer` — e.g. "we accept synthetic rows with a reviewer SPIFFE id but no evidence of the review; should the schema require a reviewed-at timestamp and a review-note pointer, and what's the operational cost of that?"

## Starter guidance

- **Canonicalise JSON before hashing.** `json.dumps(obj, sort_keys=True, separators=(',', ':'))` is the simplest canonical form; a trailing newline difference can shift every hash.
- **Make the manifest the *only* way to iterate rows.** No code outside the catalogue should scan `objects/`. If you can bypass the manifest to read a row, the provenance story breaks.
- **Treat `@latest` as a pointer, not a version.** The promote command writes a file; the fetch command dereferences it. A reviewer can see the dereferencing by running the fetch twice with a `--pin-latest-at-start` flag and comparing.
- **The Rego test suite is the lesson.** The most instructive bugs in Rego are policies that allow when they should deny — build the test suite before the policy.
- **Flag `<!-- needs-research: ... -->` on licence choices.** CC-BY-SA, CC-BY, internal-eval-only are plausible defaults; the actual licence string for your fixture depends on the source and should be verified.
- **Keep the object store flat.** Content-addressed storage *is* the deduplication; subdirectory sharding beyond a two-byte prefix buys nothing and complicates the `objects/` listing.

## Acceptance criteria

Mark done when:

- [ ] Every row that fails the envelope schema is rejected at publish; the test suite covers at least five rejection cases (missing provenance, free-form provenance kind, bad SPIFFE pattern, invalid licence SPDX, unlabelled synthetic).
- [ ] `catalogue fetch catalogue://<name>@<version-hash>` returns deterministic bytes for a given version, and the hash of the streamed rows reconstructs the manifest hash.
- [ ] `catalogue fetch` refuses a read whose SPIFFE identity fails the policy; the refusal emits an audit event.
- [ ] A fixture with `pii_class: secret` is unfetchable even when a mis-written policy row would otherwise allow (the catalogue's hard-rule guard wins).
- [ ] `catalogue diff` cleanly separates added / removed / content-diffed / provenance-only-changed rows.
- [ ] `catalogue promote` refuses an approver without the `eval-approver` role and succeeds with it; the audit record carries the approver's SPIFFE id.
- [ ] The v1 → v2 walkthrough in the README names the specific diff and the promote audit entry.

## Stretch goals

- **DVC or LakeFS backing.** Replace the hand-rolled object layout with DVC (`.dvc` file per manifest, remote storage push) or LakeFS (branch-and-commit semantics). Prove that `catalogue fetch` of a pinned version is still byte-deterministic.
- **Signed manifests.** Attach a [Sigstore cosign](https://docs.sigstore.dev/cosign/overview/) signature to the manifest at publish time; `catalogue fetch` verifies the signature before yielding rows. A tampered manifest is rejected.
- **Contamination warnings at ingest.** Implement one of the chapter-03 checks (train-cutoff vs. model-cutoff, known-benchmark hash set, n-gram overlap against a prior-version seed). Surface the warning on the manifest and on a `--warnings` flag to fetch.
- **Trace-to-catalogue promotion.** Wire an endpoint that accepts an output from exercise 03's `traces promote` and persists it as a catalogue row with `provenance.kind: replayed:<trace-id>`. Require a reviewer identity on the promote.
- **Hugging Face revision pinning.** Pin an existing published benchmark by its HF revision hash — the catalogue entry is a manifest wrapping the HF reference, not a re-host. Prove the fetch resolves to the HF revision.
- **Per-row encryption with tenant keys.** Row content blobs are encrypted at rest with a tenant-scoped KMS key stub; fetch decrypts on read. Key revocation renders the fixture unfetchable without altering the manifest.
