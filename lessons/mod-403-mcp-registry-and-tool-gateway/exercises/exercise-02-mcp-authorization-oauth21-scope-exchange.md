# exercise-02: MCP Authorization — OAuth 2.1 and Scope Exchange

**Estimated effort:** 3 hours · **Output:** one authorization sequence spec + one scope-exchange runbook + one confused-deputy mitigation review · **Pairs with:** chapter [02](../02-mcp-authorization-at-engineering-depth.md); feeds exercise 03 and project 402.

## Objective

Produce the authorization-flow specification your gateway will implement, concrete enough that a second engineer could code against it, and precise enough that a security reviewer could pick it apart. Then layer scope narrowing via RFC 8693 token exchange, and verify your spec against the confused-deputy attack class.

Exercise 01 got the registry's schema shape; this exercise is where that shape meets a real auth flow. If the spec is sound, chapter 05's auth-wrapping proxy and project 402's governed gateway have their foundation.

## Prerequisites

- You have read chapter [02](../02-mcp-authorization-at-engineering-depth.md) to the point you can recite the full discovery chain (RFC 9728 → RFC 8414 → RFC 7591) and the PKCE flow without re-reading it.
- You have the **MCP authorization specification** open in a tab (<https://modelcontextprotocol.io/specification>, authorization section).
- You have read, or are willing to read as you go, these RFCs at section-level depth: **RFC 6749** (OAuth 2.0 baseline, for context), **RFC 6750** (bearer tokens), **RFC 7591** (DCR), **RFC 7636** (PKCE), **RFC 8414** (authorization server metadata), **RFC 8693** (token exchange), **RFC 8707** (resource indicators), **RFC 9728** (protected resource metadata). The **OAuth 2.1 draft** (<https://datatracker.ietf.org/doc/draft-ietf-oauth-v2-1/>) is the consolidating document.

## Problem statement

Design the gateway's authorization layer as a *specification*, not an implementation. A correct spec enumerates every message, every claim, every validation, every error path. An incorrect spec leaves the implementer to reinvent the security-critical gaps.

The deliverable is three parts.

## Requirements

### Part 1 — End-to-end authorization sequence (the main deliverable)

Author a sequence diagram (ASCII, Mermaid, or PlantUML — any renderable form) and accompanying narrative that covers:

**Phase A — Resource discovery.**

1. Agent in session `s` on behalf of user `u` requests `tools/call` on `tool_t` served by MCP server `mcp_x`.
2. Gateway has no valid token for `(u, mcp_x, scope_required)`.
3. Gateway calls `mcp_x` without a token; receives 401 with `WWW-Authenticate: Bearer resource_metadata="<url>"`.
4. Gateway fetches the Protected Resource Metadata document (RFC 9728); extracts `authorization_servers` list and `scopes_supported`.

Document: what fields does the gateway cache, with what TTL, and when does it invalidate?

**Phase B — Authorization server discovery and client registration.**

5. Gateway picks an authorization server from the metadata list (or applies tenant policy if multiple are listed).
6. Gateway fetches the authorization server's metadata (RFC 8414); extracts the endpoints and supported methods.
7. Gateway checks whether it has an existing client registration for `(tenant, authorization_server)`.
8. If not, it performs **Dynamic Client Registration** (RFC 7591): POST to `registration_endpoint` with its redirect URI, grant types, scopes, client name, policy URI.
9. Gateway stores `(client_id, client_secret?)` in the secret store, keyed by `(tenant, authorization_server)`.

Document: what registration payload do you send? What `token_endpoint_auth_method`? What is the fallback if DCR is not supported by the auth server?

**Phase C — User authorization (PKCE authorization-code flow).**

10. Gateway generates `code_verifier` (43–128 URL-safe chars) and `code_challenge = BASE64URL(SHA256(code_verifier))`.
11. Gateway builds the authorization request URL with `response_type=code`, `client_id`, `redirect_uri`, `scope=<exact scopes requested>`, `state=<random nonce>`, `code_challenge`, `code_challenge_method=S256`, `resource=<mcp_x identifier>`.
12. Gateway redirects user `u`'s browser (through the runtime's UI channel) to the authorization endpoint.
13. User authenticates and consents; authorization server redirects to gateway's `redirect_uri` with `code` and `state`.
14. Gateway verifies `state`.
15. Gateway POSTs to the token endpoint: `grant_type=authorization_code`, `code`, `redirect_uri`, `client_id`, `code_verifier`, `resource=<mcp_x identifier>`.
16. Gateway receives `access_token`, `token_type=Bearer`, `expires_in`, optionally `refresh_token`, `scope` (possibly narrower).
17. Gateway persists refresh token to secret store; keeps access token hot.

Document: what do you do when `scope` in the response is narrower than requested? What claims (if JWT) does the gateway verify before accepting the token?

**Phase D — Invocation.**

18. Gateway invokes `mcp_x` with `Authorization: Bearer <access_token>`.
19. `mcp_x` validates `aud` matches its own identifier; checks `scope` contains `scope_required`; executes.
20. Gateway writes the audit record (chapter 07) with the token id (hashed) in `access_token_id`.

**Phase E — Refresh.**

21. Later call: gateway sees access token near expiry (or receives `401`). POSTs refresh grant.
22. Response carries new access token; may carry a new refresh token (rotation). Gateway updates the store.
23. If the auth server returns an error (`invalid_grant`), gateway drops the grant and re-initiates Phase C.

For each step, enumerate:

- **Preconditions** (what state must exist beforehand)
- **Parameters** (exact fields, with your choice of values where the spec allows discretion)
- **Validation** (what you verify on the response)
- **Error handling** (what you do on each failure class)

### Part 2 — Scope exchange runbook (narrow and impersonate)

Write a short runbook (one page) covering two token-exchange use cases:

**Use case A — Per-call scope narrowing.**

The gateway holds a broad session token for `(u, mcp_x, scope_set={A, B, C})`. The current call only needs `scope=A`. Apply **RFC 8693 token exchange** to produce a narrower token for just this call.

Specify:

- The exact token endpoint request parameters (`grant_type`, `subject_token`, `subject_token_type`, `requested_token_type`, `scope`, `resource`, `audience`).
- What you expect in the response (narrower token with `aud=mcp_x`, `scope=A`).
- What to do if the authorization server does not support token exchange (RFC 8693) — the fallback strategy from chapter 02.

**Use case B — Impersonation across planes.**

The gateway is forwarding a call to a downstream internal service on behalf of user `u`. The downstream service uses a user-aware auth model but the gateway is the acting party. Apply token exchange with `actor_token`.

Specify:

- The exact request parameters including `actor_token`, `actor_token_type` (your gateway's SPIFFE SVID or equivalent).
- The expected shape of the resulting token's claims (`sub=u`, `act={sub=gateway-id}`).
- The downstream service's validation duties — what does it check to distinguish "user U directly" from "user U via gateway G"?

### Part 3 — Confused-deputy mitigation review

Reread the confused-deputy section of chapter 02. Walk through these three attack scenarios and, for each, say how your specification prevents it. If any does not, name the gap and the mitigation you add.

- **Scenario 1 — Audience confusion.** A malicious MCP server `mcp_m` is admitted alongside `mcp_x`. `mcp_m` and `mcp_x` share an authorization server. An agent with a token for `mcp_x` makes a call that `mcp_m` could intercept (e.g. by DNS poisoning or misconfigured routing). Does `mcp_m` succeed?
- **Scenario 2 — Scope confusion.** A tool at `mcp_x` requires `scope_read`. A broader session token has both `scope_read` and `scope_write`. An attack gets the gateway to call `mcp_x` with the broader token; `mcp_x` accepts and treats the call as privileged. Does your spec prevent this?
- **Scenario 3 — Resource indicator missing.** The authorization server does not support RFC 8707 resource indicators. Your gateway proceeds anyway. What is the risk, and what does your spec do?

For each scenario, write the *exact* specification line that mitigates it (quoting your own Part 1 and Part 2 text). If no such line exists, update the spec and cite the update.

## Starter guidance

- Keep the spec *concrete*. "Validate the response" is not a specification; "Verify `aud == mcp_x.identifier`, reject with `invalid_token` otherwise" is.
- Avoid branches you will not implement. If you will not support confidential clients in the gateway's first year, say "confidential clients not supported; admission rejects authorization servers requiring them" rather than writing a never-used branch.
- Verify every RFC citation. If a claim about an RFC's content does not survive a read of the RFC, mark `<!-- needs-research: ... -->` on the claim.
- Draw the sequence with real endpoints. `tool.example.com` and `auth.example.com` are fine; `endpoint_1` is not.
- Where the MCP authorization spec is itself in flux between revisions, pin the revision you spec against and make the pin visible in the document.

## Acceptance criteria

Mark done when:

- [ ] The sequence covers all five phases with explicit preconditions, parameters, validations, and error paths.
- [ ] Every step whose correctness depends on a specific RFC cites that RFC's section number.
- [ ] Phase C enforces PKCE with `S256`, Resource Indicators, and exact-match redirect URIs; a reviewer can grep your document for those three terms and find them.
- [ ] Part 2's scope-narrowing runbook lists a concrete fallback for authorization servers without token-exchange support.
- [ ] Part 2's impersonation runbook specifies the actor-claim validation duty on the downstream.
- [ ] Part 3's three scenarios each have a direct mitigation referenced to a line of your spec.
- [ ] You have flagged any unverified RFC detail or MCP-revision detail with `<!-- needs-research: ... -->`.

## Stretch goals

- **Replay/retry safety.** Document the behaviour when a gateway crashes between step 15 and step 16 (code exchanged, token not persisted). Is the user redirected back to consent? Is the code reused (it shouldn't be)?
- **Grant store rotation.** Spec the mechanism by which refresh tokens are rotated on every refresh exchange, including the reuse-detection rule (if a used refresh token is replayed, revoke the full grant).
- **Multi-authorization-server policy.** When a Protected Resource Metadata document lists more than one authorization server, how does the gateway pick? Document a tenant-policy input.
- **Revocation integration.** When a tenant is offboarded or an incident mandates revocation, spec the sweep: which tokens, revoked via which endpoint, with which audit trail.
- **A client-side library sketch.** Pseudocode the gateway-side API that consumes this spec (`get_or_refresh_token(user, server, scope)`), with explicit error classes. Will be reused by exercise 03.
