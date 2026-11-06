# Identity Delegation and Permission-Scope Propagation

Chapter 03 pinned the routing. This chapter pins the identity that rides along with it. When a user asks writer-v1 to draft an email, and writer-v1 asks researcher-v2 to find supporting facts, three parties are now mixed into one call: the end user, the calling agent, and the called agent. The audit record needs all three. The authorisation check on researcher-v2 needs the end user's scope but *narrowed* so researcher-v2 cannot do things writer-v1 could. The mesh's mTLS needs the workload identities of writer-v1 and researcher-v2, not the end user's. Getting all of this right at every hop is the single feature that separates "a fleet of agents behind the same gateway" from "a plane that lets agents call each other safely."

The standards exist. [OAuth 2.0](https://datatracker.ietf.org/doc/html/rfc6749) is the authorisation substrate. [OAuth 2.0 Token Exchange — RFC 8693](https://datatracker.ietf.org/doc/html/rfc8693) is the delegation primitive. [SPIFFE/SPIRE](https://spiffe.io/) is the workload-identity primitive. [W3C Baggage](https://www.w3.org/TR/baggage/) carries the non-security-sensitive cross-cutting identifiers (tenant, session). The work in this chapter is marrying them to A2A so an agent team's handler sees one request-context object and never touches a token directly.

## The three identities on every A2A call

Pin the vocabulary before writing any flows.

- **The *end-user subject*.** The human (or the external system) that originated the request. Authenticated once at the edge (web, mobile, API client). Carried through every hop as the token's `sub` claim.
- **The *calling workload*.** The agent *process* that is making the call: writer-v1's pod on the mesh. Authenticated by mTLS at the sidecar with a SPIFFE SVID. Carried on the TLS handshake and surfaced to the server scaffold.
- **The *calling agent*.** The *agent* role writer-v1. Distinct from the workload because the same workload can host multiple agent roles in degenerate cases, and the audit record wants the role name, not just the pod name. Carried in the token's `act` claim (chapter: Token Exchange).

Three identities, three carriers, three audit fields. Chapter 01's `A2ACallRecord` has `caller.workload_id` (SPIFFE), `caller.agent_id` (role), and `identity.end_user_subject` (user).

A deliberately-crisp separation: the mesh authenticates workloads, not agents. The token authenticates users and *delegated* agents. The AgentCard is the binding — "this workload identity, in this tenant, hosts this agent id, with these declared scopes." A workload cannot claim to host an agent whose AgentCard it does not register. The registry is the authority.

## The edge: the user is authenticated once

The entry gateway is where the end user authenticates. The pattern is standard:

- The client (web app, mobile, API consumer) authenticates to an identity provider — OIDC, SAML, or an API-key exchange as appropriate for the tenant.
- The identity provider issues an access token (JWT) bound to the end-user subject with the full set of scopes the user is entitled to. The token audience is the entry gateway.
- The entry gateway verifies the token, extracts the subject and scopes, and attaches them to the request context for the first-hop agent. The initial token is *not* propagated downstream unchanged — it is intended for the entry gateway, not for inter-agent calls.

The entry gateway's authorisation decision is "this user can talk to this first-hop agent": the policy plane (mod-405) is the authority. If allowed, the gateway initiates the first-hop A2A call with a **fresh token minted for inter-agent use** — this is where token exchange starts.

## Token exchange: RFC 8693 in one page

[RFC 8693](https://datatracker.ietf.org/doc/html/rfc8693) is the OAuth 2.0 extension for one party asking the authorisation server to swap one token for another, representing a delegation. The pattern:

```
Token Endpoint: POST /token
    grant_type=urn:ietf:params:oauth:grant-type:token-exchange
    subject_token=<the incoming token>
    subject_token_type=urn:ietf:params:oauth:token-type:jwt
    actor_token=<the calling workload's SVID JWT, optional>
    actor_token_type=urn:ietf:params:oauth:token-type:jwt
    requested_token_type=urn:ietf:params:oauth:token-type:jwt
    audience=<the target agent's workload id>
    scope=<the narrowed scopes>

Response:
    access_token=<new JWT>
    issued_token_type=urn:ietf:params:oauth:token-type:jwt
    token_type=Bearer
    expires_in=300
```

The resulting JWT, decoded, looks like:

```json
{
  "iss": "https://authz.platform.local",
  "sub": "user:01HV9...",                          // the end user, unchanged
  "aud": "spiffe://platform.local/ns/shared/sa/researcher",
  "exp": 1762459823,
  "iat": 1762459523,
  "act": {
    "sub": "spiffe://platform.local/ns/tenants/sa/writer",
    "agent_id": "writer-v1",
    "act": {
      "sub": "spiffe://platform.local/ns/ingress/sa/entry-gateway",
      "agent_id": "entry-gateway"
    }
  },
  "may_act": {
    "sub": "spiffe://platform.local/ns/shared/sa/researcher",
    "agent_id": "researcher-v2"
  },
  "scope": "read:docs search:web",
  "platform.tenant_id": "acme-financial",
  "platform.session_id": "sess_01HV9...",
  "jti": "a9d7..."
}
```

Four things to notice.

1. **`sub` is the end user.** It does not change when writer-v1 delegates to researcher-v2. The user is the subject of the audit chain forever.
2. **`act` is a chain.** Each delegating party is represented by its workload SVID and its `agent_id`. The outermost `act` is the most recent delegator. Reading the chain is the audit record's `delegation_chain`.
3. **`may_act`** names the party that is *allowed* to act on this token. The token is not usable by anyone else; even if it leaked, researcher-v2 is the only workload audience the authorisation server would mint claims against. [RFC 8693 §4.1](https://datatracker.ietf.org/doc/html/rfc8693#section-4.1) is the normative reference for `act` and `may_act`.
4. **`scope` is narrowed.** Writer-v1 had `read:docs write:drafts send:email search:web`; researcher-v2 gets `read:docs search:web` only. The next section covers how the narrowing is decided.

The token's lifetime is short (five minutes is a common default). The authorisation server mints a fresh one per hop; the mesh and the server scaffold never cache them beyond the current call.

### Why not propagate the original token?

The original edge token is minted for the entry gateway's audience. Propagating it to a downstream agent means:

- The downstream agent trusts the token for its own audience — a token-audience bypass, and if the audience is a bearer bearer, a confused deputy waiting to happen.
- The downstream agent sees the full edge scope. No narrowing.
- The audit chain is absent — no `act`, no visible delegation step.
- A leaked token is usable at the audience of whoever receives it next, not bound to the intended workload.

Token exchange trades a central authorisation-server round-trip for all of those being correct. The authorisation server is a tier-0 dependency and needs its own HA posture (chapter 05's bulkhead and circuit-breaker policy applies to the authorisation server too).

## Scope narrowing: whose scope does the downstream agent get?

The platform's narrowing rule is simple and strict:

> The scope passed to the downstream agent is the intersection of the caller's current scope with the downstream agent's declared scopes on its AgentCard, with the caller's policy overlay subtracting anything the policy plane forbids for this delegation.

Three inputs:

1. **The caller's current scope.** From the token the caller is holding. Writer-v1 has `read:docs write:drafts send:email search:web` at the moment it decides to delegate.
2. **The callee's AgentCard `platform.declared_scopes`.** The callee announces what scopes it ever needs: researcher-v2 declares `read:docs search:web`. Scopes the callee does not declare are *stripped* at the exchange.
3. **The policy plane's delegation rule.** The policy plane may override the intersection — "even if writer-v1 holds `send:email`, writer-v1 is not allowed to delegate `send:email` to any non-email-dispatch agent." The policy decision rides in the exchange request as metadata; the authorisation server consults the policy plane.

Narrowing produces:

```
caller.scope ∩ callee.declared_scopes - policy.denied_delegations
```

For the writer→researcher case:

```
{read:docs, write:drafts, send:email, search:web}
  ∩ {read:docs, search:web}
  - {send:email : writer-v1 may not delegate send:email}
  = {read:docs, search:web}
```

A downstream agent's AgentCard is a positive declaration: "I may ever need these scopes." The platform refuses to delegate anything else to it. A bug in researcher-v2's code that tries to use `send:email` fails at authorisation; researcher-v2's token does not have that scope.

### Handoffs change the rule

Chapter 06 is the full treatment. For scope, the key fact:

- **Manager-as-tools.** The manager is the caller. Each subagent call narrows for that subagent. The manager keeps its full scope; the subagent gets a slice.
- **Handoff.** Agent A hands off to agent B. B becomes the top-level agent. B's scope is what it needs *for the rest of the session* — the narrowing is typically less aggressive because B will have to do more than one specialised task, but it is still bounded by B's declared scopes.

A handoff that transfers more scope than the handoff-target's AgentCard declares is rejected by the authorisation server. The team's scaffold refuses to initiate a handoff that would require an undeclared scope.

## The mesh layer: workload identity and mTLS

Token exchange authenticates users and delegated agents. The mesh authenticates *workloads*. Both are needed; neither is enough on its own.

[SPIFFE](https://spiffe.io/) defines SVIDs (SPIFFE Verifiable Identity Documents) — short-lived cryptographic identities for workloads. [SPIRE](https://spiffe.io/docs/latest/spire-about/spire-concepts/) is a reference implementation. Istio's citadel component and Linkerd's identity controller play similar roles out of the box. The platform's rule:

- **Every agent workload is issued a SPIFFE SVID.** The SVID names the tenant namespace, the service account, and (via a label or the `trust_domain`) the platform.
- **`PeerAuthentication` is `STRICT` platform-wide.** Every inter-agent call is mTLS'd by the sidecars. No plaintext. No opt-out.
- **`AuthorizationPolicy` enforces tenancy at the mesh level.** A call from `ns/tenants/acme-financial/sa/writer` to `ns/tenants/bigco/sa/researcher` is denied unless the registry's `cross_tenant_allow` list says otherwise. The mesh enforces this *before* the application sees the request, so a vulnerability in the application cannot bypass tenancy.
- **The server scaffold reads the peer SVID.** The scaffold surfaces the peer SVID alongside the end-user token on the request context. The audit record carries both.

The mesh's mTLS and the token's `aud` claim are independent guards. A valid token with the wrong audience is rejected by the server scaffold. A valid mTLS session with no token is rejected by the server scaffold. Both conditions must hold for the request to reach the handler.

## Tenant isolation: three layers, one obligation

The obligation: a call from one tenant's agent to another tenant's agent never happens by accident, and when it happens on purpose it leaves an explicit record.

Three layers enforce it:

- **Registry.** The registry's `GET /registry/agents/<agent_id>?tenant=<caller_tenant>` returns a 403 if the target is in a different tenant and the caller's tenant is not on the target's `cross_tenant_allow` list. The caller never learns the endpoint, so cannot dial it directly.
- **Mesh.** `AuthorizationPolicy` denies cross-tenant traffic unless an allow is explicitly configured. Even if the caller somehow guessed the endpoint, the mesh refuses the connection.
- **Server scaffold.** The scaffold checks the baggage `tenant_id` against the target agent's tenant and against the SPIFFE SVID's namespace. A mismatched trio refuses the request.

A legitimate cross-tenant call (a shared analytics agent, an external integration) has:

- An entry on the registry's `cross_tenant_allow` for the target agent.
- A mesh `AuthorizationPolicy` allowing the source namespace.
- A visible span attribute `platform.cross_tenant=true` on the client span.
- A separate audit-log stream (per chapter 06) that governance (mod-413) can review on a cadence.

The three layers are deliberately redundant. One of them being misconfigured is a conformance failure; two of them being misconfigured is a breach.

## Baggage entries: what rides along

Not every cross-cutting identifier is a security principal. The plane rides a short set of entries on [W3C Baggage](https://www.w3.org/TR/baggage/) because they are needed on every span for the observability plane:

| Baggage entry | Purpose | Chapter |
| --- | --- | --- |
| `tenant_id` | Multi-tenant partitioning on every span. | Mod-408 / 03 / this |
| `session_id` | Group all turns of a conversation. | 03 |
| `conversation_id` | Alias or extension of session; spec-version dependent. | 03 |
| `platform.turn_id` | Group all work for one user-visible turn. | 03 |
| `platform.topology` | `manager-as-tools` or `handoff`; chapter 06. | 06 |

Baggage is *not* a trust boundary. A hostile caller can lie on baggage. The plane's rule: trust baggage for *observability* attributes (grouping, partitioning) — but authorisation decisions are made on the token, not on baggage. Confusing the two is the single most common mistake in inter-agent plane designs; call it out in review.

## The authorisation server as a tier-0 dependency

Every inter-agent call consults the authorisation server once (for the token exchange). At scale, this is hot:

- A ten-hop chain requires ten exchanges.
- A fan-out of five subagents per manager turn requires five exchanges per turn.
- A fleet doing 1000 A2A calls / s requires the authorisation server to handle 1000 exchanges / s.

Three mitigations:

- **Short-lived tokens, shared denominator.** A manager that fans out to N subagents in one turn reuses the same user token as the subject; the authorisation server is the one that does the work. The individual tokens cannot be reused across subagents (different `aud` claims), but the exchange call itself is cacheable at the manager side for the duration of the turn — same subject, same scope, same audience would produce the same token, so idempotent exchange responses can be cached for their `expires_in`.
- **Co-locate the authorisation server.** Deploy it per cluster with session replication; the per-call latency is in-cluster.
- **Circuit-break the authorisation server too.** Chapter 05's resilience rules apply: an overloaded authorisation server degrades the plane; fail-open is not acceptable (you lose the audit chain), but fail-fast with a user-visible `authz_unavailable` error is.

<!-- needs-research: confirm whether the current reference OAuth 2.0 authorisation servers (Keycloak, Ory Hydra, Auth0, Okta, Google's workload identity federation) all support RFC 8693 with `act`/`may_act` claims as of this writing. Some providers have partial support; teams should not assume the feature is uniformly available. -->

## The audit record

Every inter-agent call emits an audit record alongside the `A2ACallRecord` from chapter 01. The audit record is denormalised for governance queries and uses pinned field names so a compliance review does not require a schema archaeology step.

```json
{
  "audit_id": "audit_01HWX3K...",
  "ts": "2026-11-06T14:23:12Z",
  "end_user_subject": "user:01HV9...",
  "tenant_id": "acme-financial",
  "session_id": "sess_01HV9...",
  "turn_id": "turn_00005",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "caller": {
    "agent_id": "writer-v1",
    "workload_id": "spiffe://platform.local/ns/tenants/sa/writer",
    "tenant_id": "acme-financial"
  },
  "callee": {
    "agent_id": "researcher-v2",
    "workload_id": "spiffe://platform.local/ns/shared/sa/researcher",
    "tenant_id": "shared"
  },
  "cross_tenant": true,
  "delegation_chain": ["entry-gateway", "writer-v1", "researcher-v2"],
  "scope_granted": ["read:docs", "search:web"],
  "scope_narrowed_from": ["read:docs", "write:drafts", "send:email", "search:web"],
  "scope_narrowed_reason": "callee_declared_scopes_intersection+policy_overlay",
  "token_jti": "a9d7...",
  "mtls_peer_svid": "spiffe://platform.local/ns/tenants/sa/writer",
  "outcome": "completed"
}
```

The record goes to the audit stream (separate from the observability stream, more restricted, longer retention) — per chapter 06 and mod-413.

## A concrete delegation flow

Walk a single call through every layer.

1. **User logs in.** Identity provider issues JWT: `sub=user:01HV9`, `aud=entry-gateway`, `scope=read:docs write:drafts send:email search:web`, `platform.tenant_id=acme-financial`.
2. **User POSTs to the entry gateway.** Header `Authorization: Bearer <edge-token>`. Gateway verifies the token against the IdP's JWKS.
3. **Gateway decides first-hop agent: writer-v1.** Looks up writer-v1's AgentCard; sees `declared_scopes=[read:docs, write:drafts, search:web]` (writer-v1 declares no `send:email` on A2A — email sending is an MCP tool it uses, not a scope downstream agents need). Narrows by intersection: `{read:docs, write:drafts, search:web}`.
4. **Gateway requests a token exchange.** POST to the authorisation server: subject_token = the edge token, audience = writer-v1's SVID, scope = narrowed. Receives back a fresh JWT with `sub=user:01HV9`, `act={sub=entry-gateway}`, `aud=writer-v1`, `scope=read:docs write:drafts search:web`.
5. **Gateway makes the A2A call to writer-v1.** `tasks/send`, `Authorization: Bearer <writer-token>`, `traceparent: ...`, `baggage: tenant_id=acme-financial,session_id=sess_01HV9`. Mesh initiates mTLS from entry-gateway's SVID to writer-v1's SVID.
6. **Writer-v1's server scaffold accepts.** Verifies the token (audience is writer-v1; mTLS peer is the entry-gateway; mesh allow policy permits). Surfaces: `end_user_subject=user:01HV9`, `delegation_chain=[entry-gateway]`, `scope=read:docs write:drafts search:web`.
7. **Writer-v1's handler decides to delegate to researcher-v2.** Calls `A2AClient.for_agent("researcher-v2")`.
8. **Writer-v1's client scaffold narrows and exchanges.** Looks up researcher-v2's AgentCard; `declared_scopes=[read:docs, search:web]`. Intersection: `{read:docs, search:web}`. Policy overlay: no further restrictions. Requests exchange with subject_token = writer-v1's current token, actor_token = writer-v1's SVID, audience = researcher-v2's SVID, scope = `{read:docs, search:web}`.
9. **Authorisation server responds.** New JWT: `sub=user:01HV9`, `act={sub=writer-v1, act={sub=entry-gateway}}`, `aud=researcher-v2`, `scope=read:docs search:web`, `may_act=researcher-v2`.
10. **Writer-v1's client calls researcher-v2.** `tasks/send`, `Authorization: Bearer <researcher-token>`, baggage unchanged, deadline decremented. Mesh initiates mTLS from writer-v1's SVID to researcher-v2's SVID.
11. **Researcher-v2's server scaffold accepts.** Verifies the token (audience = researcher-v2; mTLS peer = writer-v1; mesh allow policy permits; scope covers the invoked skill). Surfaces: `end_user_subject=user:01HV9`, `delegation_chain=[entry-gateway, writer-v1]`, `scope=read:docs search:web`.
12. **Researcher-v2's handler runs.** Tries to call `send:email` via its MCP client — fails at authorisation because the scope is not held. The handler either falls back to allowed behaviour or returns `failed: scope_denied`.
13. **Audit records emitted.** Per hop: one record each at step 5 and step 10. Both share `trace_id` and `end_user_subject`. The governance stream (mod-413) can reconstruct the full chain from `trace_id`.

The application code in writer-v1's handler was a single line — `client.tasks_send(...)`. Everything else was the plane.

## The four failure modes the plane must catch

1. **A token leaks and is replayed by a different workload.** `may_act` is the mesh's defence: the authorisation server will not accept it from the wrong peer SVID. The mesh's mTLS binding is the other half.
2. **An agent passes its own token straight through to a downstream agent without exchanging.** The downstream's scaffold rejects the audience mismatch. The conformance test fires a synthetic `tasks/send` with a mis-audienced token and asserts the 401.
3. **Scope narrowing is skipped and the downstream agent gets more privilege than it needs.** The authorisation server is the enforcement: it refuses to mint a token with a scope not covered by the intersection. A caller that tries to request a broader scope than the callee declares gets an exchange failure.
4. **A cross-tenant call happens without authorisation.** Three layers enforce it; a bypass requires all three being misconfigured. The conformance test exercises a cross-tenant call from a non-allowed source and asserts the registry 403 and the mesh deny; a cross-tenant call from an allowed source produces an `platform.cross_tenant=true` span and a record in the cross-tenant audit stream.

## Summary

- Three identities flow with every A2A call: the end-user subject (the original human), the calling workload (the pod, authenticated by mTLS via SPIFFE), and the calling agent (the role, carried in the token's `act` claim). The audit record names all three; the mesh authenticates the second; the token authenticates the first and third.
- OAuth 2.0 Token Exchange (RFC 8693) is the delegation primitive. Each hop exchanges the inbound token for a fresh outbound token with the end user still as `sub`, an `act` chain naming every delegator, `may_act` binding the token to the next audience, and a narrowed `scope`.
- Scope narrowing is strict: `caller_scope ∩ callee_declared_scopes − policy_overlay`. The AgentCard's `declared_scopes` is a positive declaration; anything not on it is stripped at the exchange.
- Tenant isolation is enforced three times — at the registry (403 if not allowed), at the mesh (`AuthorizationPolicy` deny), and at the server scaffold (SVID / baggage mismatch). Legitimate cross-tenant calls go on a separate audit stream.
- Baggage carries observability identifiers (tenant, session, conversation, turn, topology) but is not a trust boundary. Authorisation decisions read the token; grouping / partitioning reads baggage.
- The authorisation server is a tier-0 dependency: short-lived tokens, per-cluster deployment, circuit-breaker policy. Fail-fast is acceptable; fail-open is not — the audit chain is non-negotiable.
- Four failure modes to catch: token replay by the wrong workload, token pass-through without exchange, skipped narrowing, and un-authorised cross-tenant calls. The conformance test exercises each.

The next chapter picks up the other half of the plane's obligations: circuit breakers, bulkheads, deadlines, and cascading-failure prevention for inter-agent dependency graphs.
