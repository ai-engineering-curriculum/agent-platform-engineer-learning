# The MCP Server Registry — Versioning, Deprecation, Approval, Schema Governance

The gateway terminates the protocol; the *registry* is the system of record for which MCP servers exist, what version they are on, which scopes they declare, who approved them, and when they get retired. Without a registry, the gateway is a smart proxy with no memory. With a registry, the gateway becomes a governance surface — product teams see a tool catalog, security reviews have a hook, compliance has an inventory, and the paved-road SDK has a thing to read.

This chapter is the data-model chapter. Everything the gateway does at runtime (chapter 04 conformance, chapter 05 auth-wrapping, chapter 07 audit) reads from or writes to the registry; so getting its schema right is the single highest-leverage design decision in the module. Exercise 01 ships the artefact you reuse for the rest of the track.

## What a registry is for

Three jobs, in priority order:

1. **Describe every tool precisely enough that an agent (or the SDK on an agent's behalf) can call it correctly without reading its source code.** Discovery is not just "the tool exists"; it is "the tool exists, with this exact JSON Schema input, this output shape, these required scopes, this risk tier, this version."
2. **Carry the governance state of every tool so the platform's humans do not have to carry it in their heads.** Which tools are experimental, GA, deprecated, sunset. Who approved them. What review artefacts exist. What the deprecation ladder looks like.
3. **Be queryable by every neighbouring plane for its own purposes.** The policy plane (mod-405) queries by risk tier to decide whether step-up auth is needed. The audit plane (chapter 07) queries by version to reconstruct what an agent saw. The SRE plane (mod-410) queries by health state to build SLO dashboards. The paved-road SDK (mod-412) queries by capability to help a developer pick the right tool.

A registry that does any one of these well but not the others will be replaced. Design for all three.

## The four surfaces

Treat the registry as four interfaces over one storage layer:

- **Discovery API.** Read-only, high-traffic, latency-sensitive. Called by every agent session on every turn that may invoke a tool. Served from a hot cache; backed by the storage. Returns server entries at the version an agent is pinned to.
- **Admission API.** Write path for "I have a new server I want admitted." Accepts a submission, runs automated gates (schema validity, conformance against a staging instance, security scan, licence check), routes to the human approval workflow, and finally promotes to GA.
- **Governance API.** Read/write for lifecycle operations: deprecate, set sunset date, pin failover policy, update scope description, rotate auth-server trust. Audited; every change leaves a signed trail.
- **Health API.** Write by probes (chapter 04), read by the gateway's routing layer. High write volume (every probe emits), low read volume (gateway reads once on cache miss), eventually consistent is fine.

Keep these conceptually separate even if they share a database. Admission writes do not race with runtime discovery reads because they target different objects (a submission is not a GA entry until promoted). Health writes are isolated so a flappy server does not touch the governance record.

## The registry entry — the schema that matters

The core object is one **server entry** per registered MCP server, with embedded version records. Minimum fields:

```
ServerEntry {
  id                 string                 # stable platform id (slug), e.g. "calendar-google"
  display_name       string                 # human-facing, used in consent + catalog UIs
  description        string                 # one-paragraph, used in catalog UIs
  owner              { team, oncall }       # who maintains this server; who gets paged
  risk_tier          enum<T0, T1, T2, T3>   # T0 lowest, T3 highest; drives approvals + policy
  tags               list<string>           # faceting for the catalog UI
  auth_model         AuthModel              # OAuth 2.1 details or auth-wrap descriptor
  transport          enum<stdio,
                          streamable_http,
                          websocket>
  endpoint_url       string                 # for remote transports
  image              ImageRef               # container image ref if the registry runs it
  versions           list<ServerVersion>    # history
  active_version     semver                 # the version served at the `latest` cursor
  deprecation        DeprecationState       # non-null iff deprecated
  managed_by         ManagedBy              # self-hosted | vendor:{name} | hybrid
  slo_targets        SLOTargets             # per-tool latency/error targets
  created_at         timestamp
  updated_at         timestamp
}

ServerVersion {
  version            semver
  released_at        timestamp
  tool_schemas       list<ToolSchema>       # the frozen schema for each tool at this version
  resource_schemas   list<ResourceSchema>
  prompt_schemas     list<PromptSchema>
  scopes             list<ScopeDefinition>
  changelog          string
  approval_record    ApprovalRef            # points at the governance decision that admitted it
  conformance_ref    ConformanceRef         # points at the last conformance run's result
  image_digest       string                 # sha256 of the image at this version (if containerised)
  status             enum<proposed,
                           review,
                           staging,
                           ga,
                           deprecated,
                           sunset>
  sunset_date        timestamp?
}

ToolSchema {
  name               string
  input_schema       JSONSchema
  output_schema      JSONSchema
  required_scopes    list<string>
  idempotent         bool
  dangerous          bool                   # mirrors MCP tool annotation; drives UI + policy
}

ScopeDefinition {
  name               string                 # e.g. "calendar.read"
  description        string                 # shown on consent screen
  description_i18n   map<lang, string>      # optional
  consent_category   enum<low, medium, high>
  mapped_api_perms   list<string>           # for audit: which downstream perms this maps to
}

AuthModel = OAuth21Model | AuthWrapperModel | StdioLocalModel
```

Exercise 01 asks you to flesh this schema out in a real schema language. The sketch above is the minimum; the exercise's value is the decisions you add — which fields are required, which are append-only, which trigger a version bump when they change, which belong on the server vs. each version.

## Versioning: what version means, and why

The registry uses **semantic versioning** on the server entry *and* on each tool schema inside a version. Two version numbers matter:

- **Server version** (e.g. `calendar-google@2.3.1`) — the shippable artefact. Bumped when any tool, resource, or scope changes.
- **Tool schema version** (sometimes implicit in the server version, sometimes tracked per tool) — the schema an agent sees for a specific tool at a point in time.

The pinning rule for the gateway's second obligation ("schema frozen between discovery and invocation") is:

- When an agent reads `tools/list` at turn N, the gateway resolves each tool to `server@version:tool@schema_version` and persists those pins on the session.
- Subsequent `tools/call` in the same session uses those exact pins — even if a newer version has become GA since.
- A session that runs for days (e.g. a human-in-the-loop workflow with long waits) may need to roll forward; the gateway offers an explicit `refresh_tools()` operation, but never rolls forward silently.

**Breaking changes bump the MAJOR**. A breaking change to a tool schema is any of:

- Removing a tool, resource, or prompt.
- Removing a field from an input schema.
- Narrowing the type of an existing field (e.g. `number` → `integer`).
- Adding a new required field (existing clients break).
- Changing the required scopes upward (clients lose the right to call).
- Changing the output schema in a way clients cannot be assumed to tolerate (field removal, type narrowing).

**MINOR bumps** cover backward-compatible additions:

- New tool, resource, or prompt.
- New optional field on an input or output schema.
- New scope that is not required by any existing tool.
- Loosening a type (`integer` → `number`).

**PATCH bumps** cover descriptive-only changes: fixed wording, better error messages, updated examples. No schema or scope changes.

The registry enforces these rules at admission time with an automated schema diff. If a submission bumps PATCH but changes a tool schema, admission rejects it.

## Approval workflow — the admission ladder

A new server, or a new version of an existing server, climbs this ladder before an agent ever sees it:

1. **Proposed.** Owner submits the registry entry via a PR to the registry's GitOps repo (or the governance API). Automated checks run: schema validity, licence check, image signature, static SAST on the server source if available, policy-plane preview of required scopes.
2. **Review.** Human review by the platform team. For T0/T1 risk, this is a lightweight schema-and-scope review. For T2, a security review is added. For T3, a design review with `agentic-safety-engineer` (level 40) is required. All reviews are attached to the approval record.
3. **Staging.** The registry runs the server in a staging environment and executes the conformance suite (chapter 04). Pass → promote; fail → bounce back to the owner with the failing probes attached.
4. **GA.** The server is visible on the discovery API at its new version. The previous version remains callable by sessions that pinned to it, but new sessions resolve `latest` to the new version.
5. **Deprecated.** The owner, or the governance team, marks the version deprecated. The registry emits a sunset header on discovery responses for this version; the paved-road SDK surfaces a warning to developers; existing sessions keep working.
6. **Sunset.** On the sunset date, the registry stops serving the version. Attempts to pin to it fail with an explicit "sunset on <date>" error.

Every transition writes an approval record. Every approval record is immutable; corrections produce a new record with a `supersedes` field. This is what auditors (SOC 2, ISO 27001, EU AI Act Article 72) read.

## Deprecation ladder — the humane version

Deprecation that fires with no runway is a broken contract. Treat the ladder like a protocol, not a flag:

- **T-90 days (or T-180 for T3 tools): deprecated.** Registry marks the version deprecated; SDK emits a warning; email goes to every tenant that has a session pinned to this version in the last 30 days; migration guide is attached to the registry entry.
- **T-14 days: last-call window.** The warning upgrades to a banner; the discovery API refuses to pin *new* sessions to this version (existing ones still work); the SDK errors in dev-mode local runs.
- **T-0: sunset.** The version is removed from discovery; existing sessions that call it get a `tool_sunset` error with the migration-path tool id attached.

The ladder mirrors the IETF `Sunset` HTTP header (RFC 8594); where you can, emit the actual header on discovery responses.

**Exceptions**: an emergency sunset (security vulnerability, legal demand) collapses the ladder but still emits an audit record explaining why. Document this path explicitly; teams will not trust a ladder they have never seen enforced against an incident.

## Schema governance — the review that actually matters

Schema governance is the thing that catches breaking changes before they ship and the thing that catches *sneaky* changes (semantic drift in a field that stayed syntactically the same). The registry's schema-review gates:

- **Validity.** Every JSON Schema in the submission parses and resolves. No `$ref` dangling, no recursive types without bounds.
- **Diff.** Against the previous version, every schema diff is classified (MAJOR/MINOR/PATCH); the version bump must match. Mismatched version bump → rejection.
- **Scope coherence.** Every tool declares scopes; every declared scope has a `ScopeDefinition`. Scopes that no tool references are flagged.
- **Deprecation coherence.** If a tool's `required_scopes` are being tightened, the old version is marked deprecated and a migration tool is referenced.
- **Policy preview.** The policy plane (mod-405) is called with the new schemas and current abuse-corpus rules; a non-trivial new tool with no corresponding policy gets an admission warning.
- **Human review.** A reviewer reads the schema diff and the changelog. T2+ gets a second reviewer from security.

Semantic-drift detection is harder than syntax. Two mitigations: a **sample-input/expected-output** block in the registry entry, which the conformance suite re-runs on every version; and a **field-level description** diff gate — if a field's description changes meaningfully without a MINOR bump, a reviewer is pinged. Neither is foolproof. The real backstop is the compliance audit (chapter 07): if a tool's behaviour drifted and nobody caught it, the audit trail lets you reconstruct who was affected.

## Routing and failover — where the registry meets the runtime

Each server entry carries a **routing policy** — the registry's directive to the gateway on which concrete instance to call and what to do on failure:

```
RoutingPolicy {
  primary        Endpoint              # the default instance
  fallbacks      list<Endpoint>        # ordered list; may include vendor-provided alternates
  timeout_ms     int                   # per-call timeout before fallback
  retry          RetryPolicy           # idempotent tools may retry; non-idempotent do not
  degraded_mode  enum<reject, read_only, best_effort>
  circuit_breaker CircuitBreakerPolicy
}
```

The policy is set by the owner at admission and tuned by the SRE plane (mod-410) based on observed behaviour. Chapter 04 covers how health signals feed the failover decision at runtime.

## The storage shape

Three small rules keep the storage layer out of trouble:

1. **Server entries are versioned documents.** Every state change writes a new revision; previous revisions are kept for audit. Use a document store with history, a git-shaped file layout, or a database with append-only history tables. Do not in-place mutate the active record.
2. **Tool schemas are addressable by content hash.** `sha256(canonical_json(schema))`. This lets the audit pipeline (chapter 07) reference "the tool schema at content hash X" without needing to recover the server-version context.
3. **Image digests are stored, not image tags.** `nginx:latest` is a liability. `nginx@sha256:...` is a receipt.

## The governance-as-code pattern

In practice, the registry is a GitOps repo backed by a service:

- The repo carries `servers/<slug>/server.yaml` and `servers/<slug>/versions/<semver>/schemas.json`.
- PRs to the repo are the admission mechanism.
- CI runs the automated gates (schema validity, diff, licence, policy preview, conformance against staging).
- Merges are the promotion event; a controller syncs the merged state into the registry's hot store.
- Humans approve by reviewing the PR; `CODEOWNERS` encodes who approves what risk tier.

This pattern has one advantage worth all the plumbing: the full history of admissions, approvals, deprecations, and sunsets is version-controlled, signed by the reviewers, and queryable by auditors without a separate pipeline. Exercise 01 produces the schema the GitOps repo would carry.

## Summary

- The registry is the system of record for tool existence, version, schema, scope, governance state, and routing policy. The gateway reads from it; every other plane queries it.
- Treat it as four interfaces (discovery, admission, governance, health) over one storage layer. Keep them conceptually separate.
- The server-version and tool-schema-version numbers are what the gateway pins to; the second obligation (schema frozen between discovery and invocation) depends on this pinning.
- The approval workflow is a ladder (proposed → review → staging → GA → deprecated → sunset) with an immutable approval record at every transition. The deprecation ladder has explicit windows and mirrors IETF Sunset.
- Schema governance is the gate that catches breaking changes; a GitOps + CI + CODEOWNERS pattern gives you an auditable history for free.

The next chapter picks up where admission ends: the conformance and health-check pipeline that keeps the registry's claims honest after the server is live.
