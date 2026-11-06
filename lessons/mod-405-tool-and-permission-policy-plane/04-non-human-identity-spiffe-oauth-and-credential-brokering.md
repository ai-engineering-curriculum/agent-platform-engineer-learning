# Non-Human Identity — SPIFFE/SPIRE, OAuth 2.1, and Credential Brokering

Policy is empty if the plane cannot authentically identify *who is calling*. Chapters 02 and 03 wrote rules over `principal`, `actor`, and `tenant` as if those identities were trustworthy facts on the input. This chapter is where that trustworthiness comes from.

The question the chapter answers: how does a non-human workload — an agent session worker, a gateway, a tool sidecar, a policy decision point itself — prove its identity to every other component of the plane, how does it carry the end user's consent as a separate credential from its own identity, and how do tools get the per-call secrets they need (a database password, an OAuth refresh token, a signing key) without any of those secrets leaking across tool calls or across tenants.

Three primitives answer it: **SPIFFE / SPIRE** for workload identity, **OAuth 2.1 (chapter 02 of mod-403 already covered the MCP binding)** for user-consent tokens with per-call scope narrowing, and a **credential broker** that sits between the two and mints short-lived tool-specific credentials. Each on its own is well-documented; the engineering is in composing them and making them survive the failure modes.

## Why workload identity, specifically

A platform where workloads authenticate to each other with shared secrets, long-lived API keys, or "whatever the Kubernetes service account token looks like" has three problems:

1. **Secrets outlive workloads.** A rotation window is measured in days or weeks; a compromised secret stays exploitable for that whole window.
2. **Identity is untyped.** An API key does not say *what* it is — a gateway, a runtime, a tool sidecar. The receiving service cannot distinguish "the gateway acted" from "some code running in a pod with the same service account acted".
3. **Cross-plane trust is manual.** Every pair of services that need to talk invents its own trust story (shared HMAC, mTLS with per-service CAs, an auth service), and those stories drift.

**SPIFFE** (Secure Production Identity Framework For Everyone) and its reference implementation **SPIRE** ([spiffe.io](https://spiffe.io/)) are the CNCF-graduated primitives for workload identity. The model:

- Every workload gets a **SPIFFE ID**, a URI of the form `spiffe://<trust-domain>/<path>` (e.g. `spiffe://platform.local/gateway/prod`).
- SPIRE issues short-lived **SVIDs** (SPIFFE Verifiable Identity Documents) binding the SPIFFE ID to a key material. SVIDs come in two formats: X.509 (for mTLS) and JWT (for bearer-token flows).
- SVIDs expire in minutes, not hours. Rotation is automatic; workloads fetch a fresh SVID from the local SPIRE agent via the Workload API.
- A workload's identity is **attested** at issuance time — the SPIRE agent verifies the workload against node attestors (what node is this), process attestors (what container, with what hash, under which Kubernetes service account), and optionally custom attestors. A compromised workload cannot get an SVID for an identity it has no claim to.

SPIFFE is the layer that makes the `actor` field in chapter 02's Rego policy input non-forgeable. When the policy says "the acting workload is `spiffe://platform.local/gateway/prod`", that is because the gateway presented an SVID for exactly that identity on the mTLS session that delivered the policy request.

## SPIRE in deployment, in enough depth

A SPIRE deployment has three components:

- **SPIRE Server.** Runs the registration database, acts as the trust-domain CA, and issues SVIDs. Typically HA with a shared database (PostgreSQL, MySQL) or embedded Datastore.
- **SPIRE Agent.** Runs on every node. Attests the node to the server, then attests local workloads and fetches SVIDs on their behalf. Exposes the [Workload API](https://github.com/spiffe/spiffe/blob/main/standards/SPIFFE_Workload_API.md) over a Unix domain socket.
- **Workload API client.** A library embedded in your service that fetches an SVID at startup and keeps it rotated.

Registration binds a workload to an identity:

```bash
# Register the gateway: anything running in the `platform-prod` namespace
# as the `gateway` ServiceAccount gets spiffe://platform.local/gateway/prod
spire-server entry create \
  -spiffeID spiffe://platform.local/gateway/prod \
  -parentID spiffe://platform.local/spire/agent/k8s/prod-cluster/node-attestor \
  -selector k8s:ns:platform-prod \
  -selector k8s:sa:gateway \
  -selector k8s:pod-label:component:gateway \
  -selector k8s:pod-label:tier:production \
  -ttl 60
```

Three things to internalise about the registration shape:

1. **Selectors are AND-ed.** `k8s:ns:platform-prod` and `k8s:sa:gateway` and `k8s:pod-label:tier:production` must all match. A new deployment that forgets the `tier` label does not get the SVID — it fails to start, which is correct.
2. **Parent IDs form a tree.** A workload's parent is usually the node (as attested by the node attestor). You can build chains for nested attestation, but most platforms stay one level deep.
3. **TTL is seconds-to-minutes.** The default 60-second TTL is right for most agent-plane workloads. Longer TTLs trade security for fewer renewals; shorter TTLs raise the Workload API's call rate.

For Kubernetes, the [SPIRE Helm chart](https://github.com/spiffe/helm-charts-hardened) and the [SPIFFE CSI Driver](https://github.com/spiffe/spiffe-csi) make the agent-per-node deployment and the UDS mount into pods close to turnkey. Non-Kubernetes substrates (bare metal, VMs, serverless) have their own attestor stories ([AWS IID](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instance-identity-documents.html), [GCP instance attestation](https://cloud.google.com/compute/docs/instances/verifying-instance-identity)) — pick the attestor that matches your substrate.

## Using SVIDs — X.509 for mTLS, JWT for bearer flows

### X.509-SVIDs for component-to-component mTLS

Every service in the plane presents its X.509-SVID on the mTLS session; every service validates the peer's SVID against the trust bundle. The trust bundle is the set of CAs the trust domain recognises; SPIRE Agents keep the bundle fresh on the local side.

```python
# pseudocode via go-spiffe or python-spiffe
from spiffe import WorkloadApiClient

client = WorkloadApiClient()
svid = client.fetch_x509_svid()      # auto-refreshed in background
bundles = client.fetch_x509_bundles()  # trust bundles for every known trust domain

server = TLSServer(
    certs=svid.cert_chain,
    key=svid.private_key,
    client_ca_bundle=bundles,
    require_client_cert=True,
)
```

Three enforcement points:

1. **The server authorises the client's SPIFFE ID.** Accepting a valid X.509-SVID is necessary but not sufficient — the server checks the subject URI matches its allow-list. The `platform.tool.invoke` PDP only accepts SPIFFE IDs under `spiffe://platform.local/gateway/*` and `spiffe://platform.local/runtime/*`; anything else is a 401 before any policy runs.
2. **The client authorises the server's SPIFFE ID.** Symmetric: the gateway only speaks to a policy service presenting `spiffe://platform.local/policy/pdp`. Prevents a compromised service in a neighbouring namespace from impersonating the PDP.
3. **Federation across trust domains is explicit.** When the plane spans multiple trust domains (prod and staging, or cross-tenant hosted PDPs), use SPIFFE's [federation API](https://github.com/spiffe/spiffe/blob/main/standards/SPIFFE_Federation.md) to exchange trust bundles out of band. Accepting another trust domain's SVID without an explicit federation is a configuration bug.

### JWT-SVIDs for bearer-token flows

When the receiver cannot terminate mTLS (a third-party resource server, an AWS IAM endpoint via OIDC federation, a legacy service that only accepts a bearer header), SPIRE issues a JWT-SVID — a signed JWT with the SPIFFE ID in `sub`, the audience in `aud`, and a short expiry.

JWT-SVIDs are the bridge between the plane's SPIFFE identities and systems that speak OAuth. They are also what the chapter-02 Rego policy's `actor` field is proved by, when the actor needs to be verified by a non-mTLS-speaking downstream.

## OAuth 2.1 for the user-consent layer

Chapter 02 of mod-403 covered OAuth 2.1 in depth for the MCP binding. The pieces this chapter adds are **how OAuth 2.1 composes with SPIFFE** and **how per-call scope narrowing works in practice**.

### The two-identity shape

Every tool call carries two identities on the wire:

1. **The workload identity (SPIFFE SVID)** — proves "the component making this call is the production gateway". mTLS or JWT-SVID. This is the `actor` claim.
2. **The user consent (OAuth access token)** — proves "the end user, Alice, consented to `calendar.read` on `calendar-mcp` five minutes ago". OAuth access token bound to the resource (RFC 8707). This is the `principal` claim.

These do not combine into one credential. The receiving service validates both:

- The X.509-SVID / JWT-SVID verifies the caller is who they say they are (the gateway).
- The OAuth access token verifies the user has consented (Alice to `calendar-mcp`).

A missing workload identity is an infrastructure failure; a missing access token is an authorization failure; a workload identity with a mismatched user claim (actor-token / subject-token in RFC 8693) is a confused-deputy signal.

### Token Exchange (RFC 8693) for narrowing

Token Exchange is how the gateway narrows a session-level token to a call-specific token. The gateway holds a broadly scoped access token for the session (because the agent might call several tools). Before each actual tool call, it exchanges that token for a narrower one bound to just this tool and this scope.

```http
POST /oauth2/token HTTP/1.1
Host: auth.acme.com
Authorization: Bearer <client-credentials-token-for-gateway>
Content-Type: application/x-www-form-urlencoded

grant_type=urn:ietf:params:oauth:grant-type:token-exchange
&subject_token=<session-broad-token>
&subject_token_type=urn:ietf:params:oauth:token-type:access_token
&actor_token=<jwt-svid-for-gateway>
&actor_token_type=urn:ietf:params:oauth:token-type:jwt
&requested_token_type=urn:ietf:params:oauth:token-type:access_token
&resource=https://calendar-mcp.example.com/mcp
&scope=calendar.read
&audience=calendar-mcp
```

Three composition points worth noting:

- **`actor_token` is the JWT-SVID.** The authorization server mints a new access token whose `act` claim ([RFC 8693 §4.1](https://datatracker.ietf.org/doc/html/rfc8693#section-4.1)) names the gateway. The MCP resource server sees "subject=alice, act=spiffe://platform.local/gateway/prod" and can log both.
- **`resource` binds the token (RFC 8707).** The new token's `aud` is the specific MCP server. Replaying it elsewhere fails.
- **`scope` narrows.** The returned token carries only `calendar.read`, not whatever the session-broad token had.

Chapter 02's Rego capability check happens *before* the token exchange: the policy plane says "yes, this capability allows `calendar.read` on `calendar-mcp` right now". The exchange *implements* that decision on the wire.

### Where tokens live in the plane

Three stores, three lifetimes:

- **SPIFFE SVIDs (workload identity)** live on the workload's Workload API client. Lifetime minutes. Rotated automatically by SPIRE. Never persisted to disk by the workload.
- **User access tokens (session-broad)** live in the gateway's grant store, keyed by (tenant, user, authorization_server, resource). Short-lived (minutes). Refresh tokens live in the broker's secret store (below).
- **Per-call access tokens (narrow)** live in memory for the duration of the call, attached to the outbound HTTP request, and are not retained past that. They are also not logged — the hash is.

A plane that persists narrow per-call tokens is a plane that has given a compromised log store a replay window.

## The credential broker

Not every tool authenticates with OAuth. A tool that writes to a tenant's Postgres needs a database password. A tool that signs webhook payloads needs an HMAC key. A tool that calls a legacy B2B API needs a mutual-TLS client cert. These secrets are *tool-specific*, *tenant-specific*, and must not be ambient inside the tool at startup — a leaked secret is a confused-deputy primitive.

The broker's job: given a policy-allowed tool invocation, mint or lease the specific secret the tool needs, hand it to the tool sidecar over a per-call channel, and revoke it when the call ends.

### Shape of a broker call

```json
{
  "decision_id": "01HV8X3M9F5A6R2B4C0D7E8G9H",
  "actor_svid": "spiffe://platform.local/gateway/prod",
  "principal": {
    "kind": "user",
    "subject": "alice@acme.com",
    "idp": "okta"
  },
  "tenant_id": "acme",
  "tool_id": "warehouse-mcp/query",
  "capability_id": "cap_01HV8X3M...",
  "secret_request": {
    "kind": "database.postgres.readonly",
    "resource": "acme.warehouse.prod",
    "max_duration_seconds": 300,
    "obligations": [
      {"kind": "query.read_only"},
      {"kind": "tag.session", "value": "s_01HV8X3M..."}
    ]
  }
}
```

The response is a time-bound credential envelope — for Postgres, a dynamically-created user with a scoped role, expiring in five minutes; for an HMAC key, a signed token with the actor and purpose claims; for a mutual-TLS cert, a leaf cert issued under the broker's CA.

### HashiCorp Vault as the usual back-end

[Vault](https://developer.hashicorp.com/vault/docs) is the industry-standard secret broker. Three integration patterns:

1. **Static secret** — Vault stores a pre-existing credential and hands it out on request. Fine for OAuth refresh tokens, HMAC keys, and keys the plane does not own.
2. **Dynamic secret** — Vault generates a short-lived credential against the back-end system on each lease. [Vault's database secrets engines](https://developer.hashicorp.com/vault/docs/secrets/databases) create per-call Postgres / MySQL / MongoDB users with role-bounded permissions and a TTL. This is the pattern you want for any database the plane directly provisions.
3. **Transit** — Vault holds the key and exposes encryption / signing operations without releasing the key. For signing webhooks, generating JWTs with a tenant key, or encrypting data in a way that leaves the key material in Vault.

Vault authenticates the broker's callers with SPIFFE SVIDs via its [JWT auth method](https://developer.hashicorp.com/vault/docs/auth/jwt) (with the JWT-SVID as the bearer) or via [TLS auth](https://developer.hashicorp.com/vault/docs/auth/cert). Policy inside Vault is a second layer of defence: a Vault policy tied to the gateway's SPIFFE ID grants read on `secret/tools/*` but not write; a Vault policy tied to a tenant's rotation worker grants rotate on `secret/tools/acme/*` but not read.

### Broker-side policy composition

The broker is itself a PEP. It calls the policy plane (OPA chapter 02) before every lease:

```json
{
  "action": {
    "kind": "broker.lease",
    "secret_kind": "database.postgres.readonly",
    "resource": "acme.warehouse.prod"
  },
  "session": { ... },
  "parent_decision": { "decision_id": "01HV8X3M...", "result": "allow" },
  "environment": { ... }
}
```

Two decisions happen:

1. **Was the parent tool-invocation decision an allow?** The broker refuses leases that reference a `decision_id` not present in the decision log, or whose result was `deny`. This is the "no ambient authority" rule — a broker lease is *tied* to a specific prior policy allow.
2. **Does the broker's own policy permit this lease for this actor, principal, and resource?** A second allow check with a narrower rule-set. The result lands in the decision log; the lease carries the broker's `decision_id` alongside the parent.

The result is a double-log chain: the gateway's `tool.invoke` decision references the broker's `broker.lease` decision, which references the Vault-side policy evaluation. Confused-deputy audits walk this chain to prove each link.

## The MCP-side credential story

Chapter 02 of mod-403 covered end-user OAuth to MCP servers. The broker adds one more thing: *how the MCP server itself authenticates to the backend it wraps*.

An MCP server like `calendar-mcp` has its own credential to Google Calendar — an OAuth client registration, a signing key, or an API key. That credential is itself held by the broker, leased to the MCP server's sidecar at startup under a SPIFFE identity, and rotated by the broker on a cadence. The MCP server never stores the credential in a config file; it fetches on start, holds in memory, and re-fetches on 401.

The plane's full identity story on a single tool call:

1. **Gateway → PDP (chapter 02)**: mTLS with X.509-SVIDs. Policy returns allow with capability_id.
2. **Gateway → Broker**: mTLS with X.509-SVIDs. Broker policy allows. Broker returns narrowed OAuth access token for Alice to `calendar-mcp`.
3. **Gateway → `calendar-mcp` (chapter 02 of mod-403)**: mTLS with X.509-SVIDs, Bearer token from step 2.
4. **`calendar-mcp` → Broker**: mTLS with X.509-SVIDs. Broker leases Google OAuth refresh token the tool holds on Alice's behalf.
5. **`calendar-mcp` → Google Calendar**: HTTPS, Alice's Google access token.

Five SPIFFE identities, two user-consent tokens, zero ambient secrets. Every step emits a decision log with the `decision_id` chain; mod-408 indexes them.

## Rotation, revocation, and failure modes

### Rotation

SPIRE rotates SVIDs automatically. Workloads keep the Workload API stream open and receive new material before the current one expires. If the stream breaks, the workload's existing SVID is valid until expiry (minutes); subsequent calls fail. The right failure mode for a workload with a broken Workload API connection is to drain and restart, not to continue with a soon-expired SVID.

Vault rotates dynamic secrets on each lease. Static secrets rotate on a schedule driven by a rotation worker; the worker itself authenticates with a SPIFFE SVID and has its own policy.

OAuth refresh tokens rotate on each refresh if the authorization server supports rotation (most 2.1-compliant deployments do). Reuse detection — a refresh token presented twice — revokes the whole grant and forces re-consent. Your gateway honours the server's revocation signals.

### Revocation

Three revocation primitives:

- **Workload identity revocation.** `spire-server entry delete` removes a registration. The workload's current SVID remains valid until expiry (minutes). For incident-scale revocation, rotate the trust bundle — any SVID issued under the old CA is rejected immediately. This is a nuclear option with a blast radius; do not reach for it for routine cleanup.
- **OAuth token revocation.** The gateway calls the authorization server's revocation endpoint ([RFC 7009](https://datatracker.ietf.org/doc/html/rfc7009)) for every active token in the affected scope. Narrow revocations target a specific access or refresh token; wide revocations target all tokens for a user.
- **Broker lease revocation.** Vault revokes leases with `vault lease revoke` or `vault lease revoke -prefix`. Dynamic Postgres users are dropped; dynamic certs are added to a CRL; static secrets have their binding removed.

Incident response rehearses these three. Chapter 05's emergency revocation ladder calls them.

### Failure modes

- **SPIRE Agent down on a node.** New workloads on the node cannot start (no SVID). Running workloads continue until their SVID expires. The plane evicts the node from the scheduler once the agent has been down for N minutes; new sessions do not land there.
- **Vault down.** New broker leases fail; cached in-flight secrets continue until their TTL. Tool calls that need a fresh lease fail closed. Chapter 05's emergency posture is explicit about what happens here.
- **Authorization server down.** Session-open fails (cannot refresh); in-flight tool calls with valid tokens continue. The gateway surfaces a specific reason code the application can retry against.

A plane that pretends these failure modes do not exist either hides them in error messages or lets them cascade. Make them explicit reason codes on the decision envelope; wire alerts on them in mod-408 (observability).

## Summary

- Workload identity is SPIFFE / SPIRE. Every component gets a SPIFFE ID, every call presents an SVID, and attestation proves the workload is what it claims to be. SVIDs are short-lived and rotated automatically.
- End-user consent is OAuth 2.1. The plane uses Token Exchange (RFC 8693) with the JWT-SVID as the `actor_token` to narrow session-broad tokens into per-call tokens bound to a specific tool and scope.
- The credential broker (Vault or equivalent) holds tool-specific secrets and leases short-lived credentials bound to each decision. It is itself a PEP calling back into the policy plane.
- The decision envelope carries a chain of `decision_id`s — the invocation decision, the broker lease, the downstream call — that makes confused-deputy audits mechanical.
- Rotation is continuous; revocation has three primitives (identity, token, lease); failure modes are explicit reason codes, not unhandled exceptions.

The next chapter opens the tool-lifecycle side of the plane — how a new tool enters the paved road, how a scope change or a breaking schema change is governed, and how emergency revocation works.
