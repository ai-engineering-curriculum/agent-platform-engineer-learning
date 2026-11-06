# exercise-01: MCP Server Registry Schema Design

**Estimated effort:** 3 hours · **Output:** one registry schema (full file) + one versioning policy doc + two worked server entries · **Pairs with:** chapters [01](../01-the-tool-gateway-plane-and-the-mcp-contract.md), [03](../03-the-mcp-server-registry.md); feeds projects 401 and 402.

## Objective

Design the data model for the MCP server registry your gateway will read from. This is the system of record every other chapter in the module already assumed: discovery, pinning, approval records, deprecation state, routing policy, audit cross-references. Chapter 03 gave you a sketch; here you turn the sketch into a schema you would actually ship, with the decisions that the sketch left open.

If you produce this artefact well, the registry becomes a stable surface the paved-road SDK (mod-412), the policy plane (mod-405), the audit pipeline (chapter 07), and projects 401 and 402 all consume without re-litigating. Get it wrong and every downstream module is renegotiating fields.

## Prerequisites

- You have read chapters [01](../01-the-tool-gateway-plane-and-the-mcp-contract.md) and [03](../03-the-mcp-server-registry.md) at full depth. The sketch in chapter 03 is the starting point; this exercise is the decisions you add on top.
- You have skimmed the **Model Context Protocol specification** (<https://modelcontextprotocol.io/specification>), at least the primitives (tools, resources, prompts), the transports, and the initialization lifecycle. You do not need the authorization spec yet — exercise 02 covers that.
- You can produce a schema in a real schema language (JSON Schema, Protobuf, Avro, OpenAPI components). Pick the one your platform standardises on.

## Problem statement

Produce the registry schema and the two policies that govern it. The deliverable is in three parts; every part is required.

## Requirements

### Part 1 — The schema (the main deliverable)

Author the full registry schema. Include every type, every field, every enum.

At minimum, define:

- **`ServerEntry`** — the top-level record per MCP server. Carry identity (id, display name, description, owner), risk tier, auth model reference, transport, endpoint, image, current `active_version`, routing policy, deprecation state, managed-by (self-hosted / vendor / hybrid), SLO targets, tags, timestamps.
- **`ServerVersion`** — embedded per-version record. Carry version (semver), released_at, approval record reference, conformance reference (chapter 04 preview), image digest, status enum (`proposed`/`review`/`staging`/`ga`/`deprecated`/`sunset`), sunset date, changelog, and the full set of tool/resource/prompt schemas for this version.
- **`ToolSchema`**, **`ResourceSchema`**, **`PromptSchema`** — the MCP primitives as the registry carries them. Each includes a content hash (`sha256`) of the canonicalised schema that the audit pipeline (chapter 07) will reference.
- **`ScopeDefinition`** — scope name, human-readable description (used on the consent screen), i18n descriptions where applicable, consent category, mapping to downstream API permissions (for audit cross-reference).
- **`AuthModel`** — a sum type: `OAuth21Model` (authorization server metadata URL, resource identifier, scopes supported, DCR supported flag), `AuthWrapperModel` (reference to a chapter-05-style proxy and its downstream credential broker), `StdioLocalModel` (process command, args, env allowlist, capability constraints).
- **`RoutingPolicy`** — primary endpoint, ordered fallbacks, per-call timeout, retry policy (allowed on idempotent tools only), degraded-mode enum, circuit-breaker policy.
- **`ApprovalRecord`** — immutable: who, when, risk tier, reviewers, review artefact links, supersedes pointer.
- **`DeprecationState`** — deprecated_at, sunset_date, migration_target (server + tool), reason category.

For every field, mark it as one of: **required**, **optional**, **append-only** (set once, never mutated), **version-bumping** (changing it forces a new ServerVersion). A reviewer should be able to read your schema and know, for any field, which category it is in.

### Part 2 — Versioning policy (≈ 1 page)

Write the rules your admission pipeline enforces for `semver` bumps. Cover at least:

- What changes force a **MAJOR** bump? List at least five specific schema changes.
- What qualifies as **MINOR**? List at least three.
- What qualifies as **PATCH**? Give two concrete examples.
- What is the policy when a mismatched version bump is proposed (e.g. a PATCH bump with a schema change)?
- How is the policy enforced in CI — which checks run on an admission PR?
- When a tool is removed entirely, what happens to in-flight sessions pinned to the version that still had it?

Reference chapter 03's rules as the baseline; your version makes them concrete and executable.

### Part 3 — Two worked server entries

Write two complete `ServerEntry` instances that validate against your schema:

1. **A self-hosted T1 OAuth 2.1 server** — a plausible internal tool (e.g. `calendar-google` or `search-internal`). Include at least one version, two tools, one resource, scope definitions, routing policy with no fallback.
2. **A vendor-fronted T2 auth-wrapping proxy** — a plausible SaaS integration (e.g. `jira-mcp` via a managed vendor like Arcade/Composio). Include at least one version, two tools, scope mapping to downstream permissions, routing policy with a self-hosted fallback stub, managed-by field pointing at the vendor.

Each entry must:

- Validate against your Part 1 schema (run your validator).
- Include a non-trivial `ScopeDefinition` list (not just one scope).
- Have a credible `ApprovalRecord` (with realistic reviewers and rationale).
- Pin a container image by digest (not tag) if containerised.
- Include an SLO target that mentions at least latency and conformance-pass rate.

## Starter guidance

- Chapter 03's schema sketch is a *starting point*, not a specification. The exercise's value is the decisions you add — field-category markings (required/optional/append-only/version-bumping), concrete enum values, embedding structure, ID shapes.
- Pick a schema language you would actually use. JSON Schema is the obvious fit if your gateway speaks JSON end-to-end (and MCP does). Protobuf is better if the registry is a gRPC service. Both are valid; commit to one.
- Be opinionated about identifiers. `id` as a kebab-case slug? `id` as a UUID? Both exist? Pick and justify in a one-sentence comment.
- The two worked entries are where your abstract schema meets concrete tool reality. If an entry feels awkward to write, the schema is probably the problem — go back and fix the schema, not the entry.
- Mark `<!-- needs-research: ... -->` for any field whose current MCP spec detail you cannot verify (e.g. the exact shape of `inputSchema` in the current revision). The flag blocks auto-merge; it is cheaper than silently guessing.

## Acceptance criteria

Mark done when:

- [ ] The schema file validates (runs through its own validator without errors).
- [ ] Every field is categorised (required / optional / append-only / version-bumping) and the category is visible in the schema (via conventions, comments, or custom annotations — your call).
- [ ] The versioning policy covers at least the five MAJOR, three MINOR, and two PATCH triggers, plus the mismatch-bump rule, plus the removed-tool-with-pinned-sessions rule.
- [ ] Both worked entries validate against the schema.
- [ ] The two entries exercise at least three distinct `AuthModel` shapes between them (one OAuth 2.1, one AuthWrapper; or add stdio if a third tool fits).
- [ ] You can name, at the field level, which fields the policy plane (mod-405), the audit pipeline (chapter 07), the SDK (mod-412), and the SRE plane (mod-410) read. Write this cross-reference as a short appendix.
- [ ] You have flagged any unverified MCP-spec detail with `<!-- needs-research: ... -->`.

## Stretch goals

- **GitOps layout.** Sketch the file tree your registry would live in (`servers/<slug>/server.yaml`, `servers/<slug>/versions/<semver>/schemas.json`, `servers/<slug>/CHANGELOG.md`). Validate against your schema that the layout can round-trip.
- **CI pipeline sketch.** Draft the admission PR's CI steps: schema validity, schema diff vs. previous version, licence check, image signature verification, policy-plane preview (mod-405), conformance against a staging instance (chapter 04 preview). Each step is one pipeline stage.
- **Migration plan for `record_version`.** The audit record (chapter 07) will reference registry entries by hash. If you ever bump the schema itself, how do old audit records point at new-shape entries? Write the migration plan.
- **A third worked entry** — a **stdio local tool** (e.g. a filesystem-access server for developer-mode agents). stdio has different capability constraints; your schema should handle them without hacks.
