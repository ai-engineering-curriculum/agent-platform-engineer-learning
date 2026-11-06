# The Eval Dataset Management Plane

An eval is only as reliable as the dataset it runs against. A dataset that mutates in place cannot detect yesterday's regression tomorrow. A benchmark whose test set has leaked into training weights is a memorisation test. A "human-labelled" fixture that is 40% silently-synthetic is a judge-calibration disaster. A customer-replayed trace dropped into an evaluator's `s3://evals/` bucket has probably taken PII with it.

The dataset management plane answers four questions on behalf of every eval the platform runs:

1. **Versioning.** Which exact rows were this run's inputs, and can I re-fetch them a week from now, a year from now, on a laptop in a different continent?
2. **Provenance.** Where did each row come from — a public benchmark, a replayed production trace, a human-labelled gold set, a synthetic generation — and who authored it?
3. **Synthetic-data labelling.** Which rows are synthetic, by what generator, from what source material, labelled by whom?
4. **Access control.** Can this runner, with this identity, read this dataset — and if not, how is the refusal logged?

This chapter is the plane that answers them. Chapter 02's templates reference the plane by `catalogue://<name>@<version>` URIs; this chapter is what those URIs resolve to.

## The three storage layers

Separate three concerns so that no single system has to be the source of truth for all of them. Conflating them is where in-house dataset plumbing goes wrong.

### 1. Content-addressed object store

The raw bytes of every row live in object storage (S3 / GCS / Azure Blob / R2) keyed by content hash. A row is a canonicalised JSON or JSONL blob — serialised with a stable key ordering, no trailing whitespace — hashed with SHA-256. Mutating a row is impossible: a mutation is a new hash, a new object.

Two conventions are mature and either is defensible:

- **[DVC](https://dvc.org/doc)** — the git-companion for data. DVC stores `.dvc` metadata files in git and pushes content-addressed blobs to a remote. DVC's `dvc.lock` becomes the hash manifest; `dvc pull` is the fetch primitive the template library wraps.
- **[LakeFS](https://docs.lakefs.io/)** — a git-like control plane over object storage. LakeFS exposes branches, commits, and merges over an S3-compatible store, and its `lakefs://repo/ref/path` URI is the direct inspiration for the `catalogue://` scheme.

A third option — [Hugging Face Datasets](https://huggingface.co/docs/datasets) — is the right choice when a benchmark is already published there (SWE-bench, GAIA, etc.) and the plane's catalogue entry is a *manifest* that pins a HF revision hash. Do not re-host what Hugging Face hosts; pin the revision and audit the publisher.

### 2. Manifest and schema registry

The catalogue proper is a metadata store (Postgres is the default; any ACID RDBMS works) mapping a dataset name and version to:

- A **row schema** — JSON Schema or similar — specifying the columns a row must carry.
- A **manifest hash** — a hash over the sorted list of `(row_id, row_content_hash, provenance)` tuples, so a version can be validated as unmodified end-to-end with one check.
- A **publisher identity** — the SPIFFE id that published the version, the time, and the signed commit reference.
- A **lineage pointer** — the parent version this one was derived from, if any.
- An **access-control block** — the roles / principals permitted to read, with the explicit policy reference.

The catalogue's API is small: `resolve(uri) -> ManifestRef`, `fetch_rows(manifest_ref) -> Iterator<Row>`, `publish(schema, rows, provenance) -> ManifestRef`, `list_versions(name)`, `diff(a, b)`. No other code should touch the object store directly for eval purposes; the catalogue API is the egress seam.

### 3. Lifecycle index

A separate index — typically ClickHouse / BigQuery / DuckDB over the manifest stream — answers operational questions:

- Which fixtures were published in the last 7 days?
- Which fixtures have not been refreshed in 180 days?
- Which fixtures were touched by which `ai-eval-engineer` principal last quarter?
- Which fixtures are referenced by at least one active CI gate?

The lifecycle index is read-only. Its job is dashboards and alerts, not source of truth.

## The row schema and the mandatory fields

Every row in every dataset the plane catalogues carries a required metadata envelope independent of the task-specific fields. The envelope is non-negotiable; the task fields are the dataset author's.

```json
{
  "row_id": "writer-v1-ragas:row_00178",
  "provenance": {
    "kind": "human-labelled",
    "authored_by": "spiffe://platform.local/eval/alice",
    "authored_at": "2026-10-14T19:03:00Z",
    "source_ref": "linear:EVAL-1142"
  },
  "pii_class": "none",
  "licence": "CC-BY-SA-4.0",
  "train_cutoff_relevance": {
    "earliest_safe_train_cutoff": "2026-07-01"
  },
  "columns": {
    "input": "...",
    "reference": "...",
    "rubric_ref": "rubric://writer-v1-ragas@v3"
  }
}
```

Six fields the envelope enforces:

1. **`row_id`** — stable across versions. If the row content changes, the content hash changes, but the row id stays — so historical eval records keep pointing at "the same question" even as its text is refined.
2. **`provenance.kind`** — one of `human-labelled`, `synthetic:<generator>`, `replayed:<source>`, `public-benchmark:<name>`, `partner:<vendor>`. The enum is closed; a dataset emitting a free-form provenance string fails the publish check.
3. **`provenance.authored_by`** — a SPIFFE id, not a human name. Humans have SPIFFE ids too (via a dev workload identity plane); the field is machine-correlatable.
4. **`pii_class`** — one of `none`, `pii`, `phi`, `financial`, `secret`. The access-control plane (below) keys off this; a row with `pii: high` cannot be surfaced to a public-benchmark run even if the ACL would permit it.
5. **`licence`** — SPDX identifier. A dataset that mixes licences gets a per-row licence; an eval report flags per-licence split.
6. **`train_cutoff_relevance.earliest_safe_train_cutoff`** — the earliest model-training date at which this row cannot have leaked into training. Chapter 05 reads this when gating; if the under-test model's training cutoff postdates it, the row is ineligible and the gate refuses.

A dataset failing any envelope check on publish is a publish-time error, not a run-time error. The catalogue refuses to version a defective dataset.

## Versioning and immutability

Treat a dataset like a signed, reviewed artefact. The pattern:

- **Versions are immutable.** A published version's manifest hash never changes. Fixing a bad row is a *new* version with its own hash and its own lineage pointer to the parent.
- **The `@latest` tag is a mutable pointer, not a version.** `catalogue://writer-v1-ragas@latest` is a convenience; `catalogue://writer-v1-ragas@sha256:abcd...` or `@v7` is the reproducible reference. Release-blocking gates and compliance audits dereference `@latest` to the underlying hash at run start and record it on the `EvalRecord`.
- **Promotion is reviewed.** A new version flips `@latest` only when an `ai-eval-engineer` principal approves the publish — because the point of `@latest` is to be the "known-good".
- **Diffs are first-class.** `catalogue diff v6 v7` surfaces per-row content diffs, added rows, removed rows, and provenance changes. A reviewer who sees a surprise gate failure asks "did the fixture change?" first; the diff answers in one command.
- **Retention is explicit.** Historical versions never disappear by default; a prune policy is an opt-in with a long tombstone window. A silently-deleted version is a reproducibility leak.

The practical shape in late 2026 is DVC or LakeFS backing the object store; Postgres as the manifest store; a small `catalogue` service wrapping both with the review and ACL glue.

## Provenance and the synthetic-data labelling model

The most dangerous dataset is a "human-labelled" fixture with silent synthetic rows. A model trained on cheap synthetic rows is cheap to score; the human-labelled fixture becomes a judge-of-itself rather than a judge-of-the-model.

Label synthetically-generated rows as synthetic. Schema-level, not conventional.

The canonical shape:

```json
{
  "provenance": {
    "kind": "synthetic:writer-v1-adversarial@v2",
    "generator": {
      "model_identifier": "platform:planner-v1",
      "resolved_provider": "openai",
      "resolved_model": "gpt-4o-2024-08-06",
      "prompt_hash": "sha256:7b1c...",
      "generator_run_id": "gen_01HV..."
    },
    "source_material": [
      { "kind": "public-dataset", "uri": "catalogue://writer-v1-seed@v1" }
    ],
    "reviewed_by": "spiffe://platform.local/eval/bob",
    "reviewed_at": "2026-10-20T11:14:00Z"
  }
}
```

Three invariants:

1. **Every synthetic row names its generator.** The generator is itself a platform-coded model identifier with a pinned prompt. If a synthetic generator rolls, the dataset's descendants are known.
2. **Every synthetic row names its source material.** A row synthesised from no source material is suspect; the catalogue surfaces rows with empty `source_material` in a report so a reviewer can intervene.
3. **Human review is a schema field.** A `reviewed_by` of `null` on a synthetic row that participates in a release-blocking gate is a publish-time error. An unreviewed synthetic row can participate in exploratory runs, not in gates.

Two things the plane does *not* do:

- It does not refuse synthetic data. Synthetic generation is a productivity win for rare-case coverage and for expanding small golden sets.
- It does not pretend a generator-based synthetic review is a substitute for a human labeller for the golden set. The template library's leaderboard shows the two in different columns; the methodology on when each is defensible is `ai-eval-engineer`'s.

## Replayed production traces as a dataset source

Production traces (chapter 04) are a goldmine of realistic fixtures — real prompts, real tool-call shapes, real failure modes. They are also PII. Treating a replayed trace as a dataset row requires three things:

1. **Redaction before ingest.** The trace plane runs the same detectors the model gateway runs on egress (mod-406 chapter 06) over the trace, and the redacted trace is what enters the catalogue. The redaction decision is logged; reviewers can see what was removed.
2. **Consent and licence attribution.** A tenant's terms-of-service must permit the replay use. The catalogue carries the licence string the tenant's TOS grants (`internal-eval-only`, `anonymised-public`, etc.), and the access-control plane reads the licence as part of the ACL evaluation.
3. **Reproducibility of the replay shape.** Chapter 04 details how to replay a trace against a new model / prompt / tool-schema. The dataset manifest carries the trace reference and the replay-contract-version so a run a year later produces the same inputs under the same replay semantics.

A trace-derived fixture is often the most valuable fixture a platform has — and the one most likely to leak. Build the redaction, consent, and replay-reproducibility story first, not after you have five of them.

## Access control for datasets

Not every eval runner can read every dataset. A regulated-financial fixture should not be readable by a public-benchmark harness; a customer-replayed fixture should not be readable outside the agent that generated it; a judge-calibration fixture should not be exposed broadly (or it becomes a training leak itself).

The plane's access model is attribute-based and cedes to the policy plane (mod-405 chapter 03). Three attributes:

- **Subject.** The SPIFFE workload identity of the eval runner, often with an authored-by human identity in-band for logging.
- **Resource.** The dataset's `name`, `version`, `pii_class`, `licence`, `publisher_tenant`.
- **Action.** `read_row`, `read_manifest`, `publish_version`, `publish_tag_latest`, `diff`.

A policy example:

```rego
package catalogue.read_row

default allow = false

allow {
  input.resource.pii_class == "none"
  input.resource.licence in {"CC-BY-SA-4.0", "MIT", "Apache-2.0"}
}

allow {
  input.resource.pii_class in {"pii", "phi"}
  input.subject.tenant == input.resource.publisher_tenant
  input.subject.roles[_] == "eval-runner-in-tenancy"
}

deny[msg] {
  input.resource.pii_class == "secret"
  msg := "secret-class datasets never egress eval boundary"
}
```

Every refusal is logged (mod-405's audit stream). A refusal on a scheduled gate is a `catalogue.access_refused` event the leaderboard surfaces so a team is not surprised by a gate it does not have permission to run.

Three practical rules:

- **Default-deny.** A new dataset starts unreadable; access is explicitly granted.
- **No ambient catalogue tokens.** A runner's catalogue read is authenticated by SPIFFE identity, not by a shared API key. A shared key leaks; an identity does not.
- **Deny for `pii_class: secret` is a hard rule, not a policy row.** The catalogue refuses to serve the row at all for these classes; a mis-written policy that `allow`s them is overridden by the catalogue. "Defence in depth" is the engineering phrase; "no silent policy regression" is the operational phrase.

## Contamination checks at ingest

A benchmark that postdates the model you are testing is a useful gauge. A benchmark that predates the model's training-cutoff is a memorisation test — the model may have been trained on it.

The catalogue runs three contamination checks at ingest and surfaces the results on the dataset page, so the authoring team sees them before a gate wires the fixture in.

- **Train-cutoff versus publication date.** The catalogue records the dataset's publication date (or the latest row's publication date for aggregated benchmarks) and cross-references it against known provider training cutoffs. A `publication_date <= model.train_cutoff` flags a warning. Chapter 05 uses the flag.
- **Hash collision against known contaminated sets.** A set of hashes of widely-published benchmark rows is maintained (the [Open Benchmark Contamination](https://github.com/EleutherAI/elk) style of practice); the catalogue cross-references new rows against the set.
- **Near-duplicate detection via n-gram / embedding overlap.** For large published sets, a cheap n-gram overlap check (shingling over 10-grams) surfaces rows that may have leaked. Methodology on how to interpret the overlap number is `model-evaluation-engineer`'s; the plane provides the signal.

None of these is definitive — proving a benchmark is not in training weights is hard. The checks are warnings a reviewer acts on, not refusals.

## Concrete schema — the manifest and the publish API

The manifest the catalogue stores:

```yaml
name: writer-v1-ragas
version: v7
version_hash: sha256:82c1...
schema_ref: schema://writer-v1-row@v2
row_count: 487
published_by: spiffe://platform.local/eval/alice
published_at: 2026-10-14T19:03:00Z
parent_version: v6
licence_split:
  CC-BY-SA-4.0: 420
  internal-eval-only: 67
provenance_split:
  human-labelled: 300
  synthetic:writer-v1-adversarial@v2: 120
  replayed:prod-writer-v1: 67
pii_class: pii
train_cutoff_warnings:
  publication_date_vs_known_cutoffs:
    - { provider: openai, model_family: gpt-4o, cutoff: 2024-04-01, ok: true }
    - { provider: anthropic, model_family: claude-sonnet-4, cutoff: 2025-01-01, ok: true }
access_policy_ref: policy://writer-v1-ragas-access@v3
integrity_signature: sig:cosign:...
```

The publish API (sketched):

```python
from platform_catalogue import Catalogue, Row, Provenance

cat = Catalogue.from_env()
rows = [
    Row(
        row_id="writer-v1-ragas:row_00178",
        provenance=Provenance.human_labelled(
            authored_by="spiffe://platform.local/eval/alice",
            source_ref="linear:EVAL-1142",
        ),
        pii_class="none",
        licence="CC-BY-SA-4.0",
        train_cutoff_relevance="2026-07-01",
        columns={"input": "...", "reference": "...", "rubric_ref": "rubric://writer-v1-ragas@v3"},
    ),
    # ...
]
manifest = cat.publish(
    name="writer-v1-ragas",
    schema_ref="schema://writer-v1-row@v2",
    rows=rows,
    access_policy_ref="policy://writer-v1-ragas-access@v3",
)
print(manifest.version_hash)
```

A reviewer runs `catalogue diff v6 v7` and sees exactly what changed; `catalogue promote writer-v1-ragas v7 --tag latest` is the separate, reviewed step that makes the new version the default.

## Summary

- The plane separates three concerns — content-addressed object store (DVC / LakeFS / HF revision pins), catalogue manifest (Postgres), lifecycle index (ClickHouse / BigQuery). Versions are immutable; `@latest` is a reviewed, mutable pointer.
- Every row carries a mandatory envelope — `row_id`, `provenance.kind`, `provenance.authored_by`, `pii_class`, `licence`, `train_cutoff_relevance`. The envelope is enforced at publish, not at run.
- Synthetic rows name their generator, their source material, and their reviewer. An unreviewed synthetic row cannot participate in a release-blocking gate.
- Production traces can become fixtures, but only after redaction, consent / licence attribution, and a replay-contract version. The redaction decision is logged.
- Access control is attribute-based, SPIFFE-identified, default-deny. Hard rules (`pii_class: secret`) are enforced by the catalogue, not by policy, so a mis-written policy row cannot override them.
- Contamination checks (train-cutoff warnings, known-benchmark overlap, n-gram overlap) run at ingest and surface as warnings — reviewer-interpreted, not catalogue-refused.

The next chapter opens the trace / replay store — the sibling data plane to the catalogue, holding every interaction a replayed-trace dataset row points at, and the store a reviewer walks when a gate failure needs explaining.
