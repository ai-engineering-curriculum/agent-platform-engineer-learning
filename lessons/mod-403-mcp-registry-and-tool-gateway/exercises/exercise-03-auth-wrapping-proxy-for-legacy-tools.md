# exercise-03: Auth-Wrapping Proxy for Legacy Tools

**Estimated effort:** 3 hours · **Output:** one proxy design doc + one scope-mapping YAML + one anti-pattern review · **Pairs with:** chapters [02](../02-mcp-authorization-at-engineering-depth.md), [05](../05-auth-wrapping-proxy-for-non-oauth-servers.md); feeds projects 401 and 402.

## Objective

Design the auth-wrapping proxy for one concrete legacy tool that does not speak OAuth 2.1. The proxy must terminate MCP + OAuth 2.1 with the agent, broker a downstream credential, invoke the real system, and produce an attributable audit record — *without* breaking the three gateway obligations (one address, frozen schema, attributable invocation).

Chapter 05 gave the pattern; this exercise is where you commit to a legacy system, pick credential patterns, write the scope mapping, and audit yourself against the chapter's anti-patterns. The artefact feeds exercise 01's registry (your proxy is one entry) and project 401's end-to-end platform.

## Prerequisites

- You have read chapters [02](../02-mcp-authorization-at-engineering-depth.md) and [05](../05-auth-wrapping-proxy-for-non-oauth-servers.md).
- You have completed exercise 02, or at least have the authorization sequence spec from it available — the proxy reuses that spec for its inbound side.
- You have skimmed one of: **SPIFFE/SPIRE** docs (<https://spiffe.io/docs/latest/>), **AWS STS AssumeRoleWithWebIdentity** (<https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRoleWithWebIdentity.html>), **GCP Workload Identity Federation** (<https://cloud.google.com/iam/docs/workload-identity-federation>), **Azure Federated Identity Credentials** (<https://learn.microsoft.com/en-us/graph/api/resources/federatedidentitycredential>), **HashiCorp Vault** (<https://developer.hashicorp.com/vault/docs>). Pick whichever matches your platform's secrets substrate.

## Problem statement

Pick **one** legacy system from the list below. All are plausible scenarios; all require different credential patterns. If you feel strongly about a different system that fits the "no OAuth 2.1" criterion, pick that and justify.

- **A** — Internal ERP (SAP / Oracle NetSuite / Workday) with a static API key per environment. Users have cost-centre-based access on the ERP itself.
- **B** — Jenkins CI server using username + API token per user, with role-based permissions inside Jenkins.
- **C** — An AWS data lake (S3 + Glue) where access is granted via IAM roles, not user-scoped tokens. The data lake team runs a service account that can read everything.
- **D** — An on-premise Oracle database with a shared service-account connection, with per-user filtering enforced in the application layer above it.
- **E** — A SaaS vendor whose MCP server exists but speaks OAuth 2.0 with the resource-owner-password-credentials grant only (so not 2.1-compliant).

For the chosen system, design the auth-wrapping proxy. The deliverable is three parts.

## Requirements

### Part 1 — The proxy design doc (the main deliverable)

Author a design document covering:

**System under proxy (half-page).** Name the legacy system, describe its native auth model (what the downstream actually accepts — API key? user + token? IAM role? JDBC credentials?), describe its permission model (what roles / scopes exist on the downstream and how fine-grained they are), describe its logging capabilities (can the downstream distinguish per-user activity in its own audit?).

**Proxy architecture (one page).** Draw the proxy from outside and inside:

- From outside: your proxy is a normal MCP server, admitted to the registry with a `ServerEntry` (from exercise 01). Specify its auth model (`AuthWrapperModel` from exercise 01's schema), its transport, its advertised tools, its scopes.
- From inside: show the components — inbound OAuth 2.1 termination (from exercise 02's spec), policy-plane consult, credential broker (secret store, IdP federation, or SPIFFE-based), downstream invocation, response translation, audit emission.

**Credential pattern (half-page).** Pick one from chapter 05's list, justified against your chosen system:

- Static shared key
- Static per-tenant keys
- Short-lived credentials from workload identity federation
- User impersonation via token exchange
- Signed-request scheme
- mTLS downstream

For the chosen pattern, specify:

- Where the credential lives at rest (which broker, under what name).
- How the proxy fetches it (at proxy startup? per-call? per-session? cached with what TTL?).
- Who rotates it, on what cadence.
- What happens when rotation fails (fail closed? degraded mode?).

**Request flow (half-page).** For one representative tool call on your chosen system, walk through:

1. Agent sends `tools/call` to the gateway.
2. Gateway's inbound auth runs (exercise 02's spec, Phase D).
3. Gateway routes to the proxy.
4. Proxy validates its own inbound bearer token (`aud` + `scope`).
5. Proxy consults the policy plane (mod-405) with the call context.
6. Proxy fetches/mints the downstream credential.
7. Proxy invokes the downstream system.
8. Proxy translates the response (mapping downstream error codes to MCP error shapes).
9. Proxy writes the audit record (chapter 07 schema).
10. Proxy returns the MCP tool-call result to the gateway, which returns to the agent.

Call out which steps could fail and what the proxy does on each failure class.

**Attribution plan (half-page).** Specifically: can the downstream distinguish per-user activity, or does its audit collapse to the proxy identity? If it collapses, what does your proxy do to keep attribution recoverable from your side alone? (Hint: chapter 07's audit record is the primary instrument.)

### Part 2 — Scope-mapping file

Author the `scope_mapping` section of your proxy's registry entry. Include **at least three scopes**, with:

- The MCP scope string (user-consent-visible, e.g. `erp.read_invoice`).
- The human-readable description for the consent screen.
- The downstream permission(s) the scope maps to (e.g. `role/erp-invoice-reader` or `IAM/arn:aws:iam::.../data-lake-reader`).
- Any constraints evaluated at call time (ABAC filters, field allowlists, time-of-day, tenancy).
- The consent category (low/medium/high) and risk tier of the resulting call.

The format is yours — YAML, JSON, whatever your schema from exercise 01 supports. Include a one-paragraph note on how mapping changes get versioned: which changes are MINOR (loosening), which are MAJOR (tightening).

### Part 3 — Anti-pattern review

Chapter 05 lists seven anti-patterns. For each, write a one-line statement of how your design does not fall into it. If any one is hard to answer, your design has a gap — fix it before submitting.

The seven:

1. Embedding the downstream credential in the MCP server source.
2. A single service account for all tenants when per-tenant credentials are possible.
3. Letting the agent pass the API key through as a tool argument.
4. Logging the Authorization header or token body in debug mode.
5. Reusing the same proxy container to front multiple downstreams.
6. Treating the proxy as "stateless middleware."
7. Static API keys shared across tenants when a federated alternative exists.

## Starter guidance

- Pick the scenario where you will learn the most, not the one you think is easiest. Scenario D (on-prem Oracle) and scenario E (bad SaaS OAuth 2.0) are where the design pressure is highest — do one of those if you can.
- The **credential pattern** is the single hardest decision. If your scenario is in a cloud, workload identity federation almost always beats static keys; justify loudly if you pick otherwise.
- The **attribution plan** is where chapter 05 collides with chapter 07. Your audit record must preserve user attribution even when the downstream cannot. Write the audit field values for a worked invocation.
- The anti-pattern review is not optional. It is cheap and it catches real problems; treat it like a security self-review.
- Scope names should be *user-facing* — a non-technical user on a consent screen should understand `erp.read_invoice` and `erp.update_invoice_due_date` without an explanation. If a scope name confuses that reader, rename it.

## Acceptance criteria

Mark done when:

- [ ] A legacy system is picked and its native auth model is described in enough detail that a security reviewer could critique it.
- [ ] The proxy architecture names every component (inbound auth, policy consult, broker, invocation, translation, audit) and shows how they connect.
- [ ] The credential pattern is one of chapter 05's named patterns; the choice is justified in writing against your scenario.
- [ ] The request flow walks through at least ten steps with explicit failure handling.
- [ ] The attribution plan states, in one paragraph, how user identity is preserved end-to-end even if the downstream cannot.
- [ ] The scope mapping has at least three scopes, each with description, downstream permission, and constraints.
- [ ] All seven anti-patterns have a direct, non-hand-waving rebuttal.
- [ ] You have flagged any unverified downstream-system or federation-primitive detail with `<!-- needs-research: ... -->`.

## Stretch goals

- **Rotation runbook.** Write the on-call runbook for a credential compromise in your chosen scenario. Include the detection signal, the rotation steps, the impact on in-flight sessions, the audit query you would run to confirm the blast radius.
- **Degraded mode.** If the credential broker is itself unavailable, what does the proxy do? (`reject`, `read_only`, or `best_effort` from chapter 04's routing policy.) Pick and justify against your scenario's risk tier.
- **Second proxy sketch.** Pick a second scenario from the list and sketch its credential pattern without writing the full design. Diff the two: where would you factor shared infrastructure (common audit emitter, common broker client), and where would you keep them separate?
- **Audit record walk-through.** Write out the chapter 07 `ToolInvocationAudit` record for one successful call, one policy-denied call, and one timeout. Values should be plausible, not placeholders.
- **CI pipeline.** Sketch the test suite that would run against your proxy on every commit — at minimum a contract test (does the inbound MCP flow pass exercise 02's spec?), a credential-brokering test (with a mocked broker), and a conformance test (chapter 04).
