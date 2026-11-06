# MCP Authorization at Engineering Depth

The MCP authorization specification is the thing you cannot read-ahead. If you do not know, at RFC-level depth, how OAuth 2.1 works and how MCP binds to it, you cannot design the gateway's auth layer, you cannot write a registry entry that captures a server's auth model, you cannot audit who called what with which scope, and you cannot wrap a legacy tool without opening a confused-deputy hole. This chapter goes slower than the others on purpose: it is the one you will reread.

## The MCP authorization model, in one paragraph

An MCP server is an OAuth 2.1 **resource server**. The MCP client (hosted in your gateway) is an OAuth 2.1 **client**. The end user sits behind an **authorization server** that the resource server trusts. When the client speaks to the resource server without a valid access token, the resource server returns `401` with a `WWW-Authenticate` header that points at a **Protected Resource Metadata** document (RFC 9728), which in turn names one or more authorization servers. The client then discovers the authorization server's endpoints via **Authorization Server Metadata** (RFC 8414), registers as a client (statically or via **Dynamic Client Registration**, RFC 7591), runs the authorization-code flow with **PKCE** (RFC 7636), optionally performs **Token Exchange** (RFC 8693) to narrow or widen scopes for a specific downstream call, and finally invokes the MCP server with a bearer access token bound to the right resource via **Resource Indicators** (RFC 8707). Every noun in that paragraph is a specific RFC with a specific wire format. Learn them.

## Why OAuth 2.1 and not 2.0

OAuth 2.1 is the IETF consolidation of a decade of security guidance for OAuth 2.0. It is not a new protocol; it is 2.0 minus the modes that have accumulated known-bad properties, plus the modes that have become normative. The differences the gateway plane cares about:

- **PKCE is mandatory for all public clients.** Not optional, not recommended, mandatory. Your authorization-code flow sends a `code_challenge` on the authorization request and a `code_verifier` on the token request. No exceptions.
- **No implicit grant.** Access tokens never come back on the authorization response URL fragment. If a server offers implicit, it is not OAuth 2.1 and your gateway should refuse to admit it.
- **No resource-owner-password-credentials grant.** Users never type credentials into the client. If a legacy tool wants this, it is a candidate for the auth-wrapping proxy (chapter 05), not an MCP OAuth client.
- **Exact-match redirect URIs.** No prefix matching, no wildcards. Your gateway registers the exact callback URL it will use; if a vendor's authorization server does prefix matching on top, you still only register one.
- **Bearer tokens over TLS only.** Your gateway refuses non-TLS resource servers on the production lane.

MCP binds to OAuth 2.1, not 2.0. If a draft or vendor article says "use OAuth 2.0," read it as a tell that the writer is behind the current spec; verify against the MCP authorization document before quoting it.

## Protected Resource Metadata (RFC 9728) and discovery

When an MCP resource server returns a `401 Unauthorized` response, it includes a `WWW-Authenticate` header like:

```
WWW-Authenticate: Bearer resource_metadata="https://tool.example.com/.well-known/oauth-protected-resource"
```

The URL points at the server's **Protected Resource Metadata** document, a JSON blob that answers two questions the client needs:

- **Which authorization servers are trusted for this resource?** (`authorization_servers` — an array of issuer URLs)
- **Which scopes does this resource understand?** (`scopes_supported` — the vocabulary of scopes the resource recognises, each with a human-readable description so the gateway can render a consent screen later)

Your gateway caches these documents aggressively and invalidates on auth-server-rotation events. If you do not cache, every tool call racing to the same resource server storms the metadata endpoint. If you cache forever, an auth-server migration fails silently.

## Authorization Server Metadata (RFC 8414)

Given an authorization server issuer URL, the client fetches `${issuer}/.well-known/oauth-authorization-server` to discover:

- `authorization_endpoint` (where PKCE flow goes)
- `token_endpoint` (where the authorization code gets exchanged)
- `registration_endpoint` (for Dynamic Client Registration)
- `revocation_endpoint`, `introspection_endpoint` (optional but worth plumbing)
- `scopes_supported`, `grant_types_supported`, `token_endpoint_auth_methods_supported`
- `code_challenge_methods_supported` (must include `S256` for 2.1 compliance)

If your gateway treats this document as a fixed assumption instead of a per-server fetched fact, you cannot support multi-tenant resource servers with different auth providers. Fetch it; cache it with a sensible TTL; honour what it says.

## Dynamic Client Registration (RFC 7591)

A platform-scale gateway cannot pre-register itself at every authorization server in the world. It uses **Dynamic Client Registration** to post a registration request to the auth server's `registration_endpoint` at first contact, receives a `client_id` (and optionally a `client_secret`), and persists that pair under (tenant, authorization_server_issuer). On subsequent calls for that tenant/issuer combination, the gateway reuses the registered client.

The registration payload carries:

- `redirect_uris` (exact-match; your gateway's callback)
- `token_endpoint_auth_method` (`none` for public clients with PKCE, `client_secret_basic` or `client_secret_post` for confidential clients)
- `grant_types` (`authorization_code`, `refresh_token`, `urn:ietf:params:oauth:grant-type:token-exchange` if you need it)
- `response_types` (`code` only in 2.1)
- `scope` (the superset of scopes your gateway anticipates requesting)
- `client_name`, `client_uri`, `logo_uri`, `tos_uri`, `policy_uri` (for the consent screen the end user eventually sees)

Operational consequences:

- You own the lifecycle of these registrations. A tenant leaves the platform → you revoke the registration. An auth-server rotates registration tokens → you re-register.
- You never embed the `client_secret` in the agent, the runtime, or the MCP client library. It lives in the gateway's secret store (see chapter 05 and mod-405 for brokering).
- Some authorization servers (notably older enterprise IdPs) do not support DCR. For those, your gateway falls back to static registration via a manual admission workflow in the registry (chapter 03).

## PKCE flow, step by step

Here is the authorization-code flow with PKCE as the gateway executes it. Memorise the shape.

1. **Generate `code_verifier`.** A high-entropy random string, 43–128 characters, URL-safe.
2. **Derive `code_challenge`.** `BASE64URL(SHA256(code_verifier))`. Use `S256`; never `plain`.
3. **Redirect the end user's browser to the authorization endpoint.** Parameters: `response_type=code`, `client_id`, `redirect_uri`, `scope` (the exact scopes requested for this session), `state` (random nonce you verify on callback), `code_challenge`, `code_challenge_method=S256`, and `resource` (RFC 8707 — the URI of the MCP server the resulting token will target).
4. **User authenticates and consents at the authorization server.** Server redirects back to your `redirect_uri` with `code` and your original `state`.
5. **Verify `state`.** If it does not match, abort — this is a CSRF signal.
6. **POST to the token endpoint.** Parameters: `grant_type=authorization_code`, `code`, `redirect_uri`, `client_id`, `code_verifier`, and `resource` (again, RFC 8707).
7. **Receive the response.** `access_token`, `token_type=Bearer`, `expires_in`, optionally `refresh_token`, optionally `id_token` (if OIDC), and `scope` (which may be narrower than requested — honour what you got, not what you asked for).
8. **Persist.** The gateway stores `{refresh_token, scope, resource, expires_at}` keyed by (tenant, user, authorization_server, resource). Access tokens are kept hot in memory; refresh tokens go to the secret store.
9. **Invoke the resource.** The gateway attaches `Authorization: Bearer <access_token>` and calls the MCP server.
10. **On `401` or expiry, refresh.** The refresh exchange is another token-endpoint call with `grant_type=refresh_token`; the new `access_token` is bound to the same `resource`.

The whole flow is run *per user-plus-resource*, not once per tenant. A user who has consented to `read_calendar` on one MCP server has not consented to anything on another — your gateway's grant store is keyed accordingly.

## Resource Indicators (RFC 8707) and the confused deputy

**The confused deputy** is the attack class where a client with a token for resource A is tricked into using it against resource B that also accepts it. In an agent setting, this is not theoretical: an MCP server you proxy to could, in principle, present a token it accepted on one tool call to a different, more privileged resource. The mitigation is simple and non-negotiable: every token your gateway requests carries the `resource` parameter naming the MCP server it is for, and every MCP server you admit to the registry validates the `aud` (audience) claim on every incoming token against its own identifier.

If the authorization server does not support `resource`, do not admit its tools to the production lane. If an MCP server does not validate `aud`, do not admit it. These are fire-the-admission-workflow rules.

## Token Exchange (RFC 8693) and scope narrowing

MCP tool calls routinely need **downgraded** tokens: your gateway has a broadly scoped session token (because the agent might need any of several tools) and wants a narrowly scoped access token for this specific tool call, one that cannot be replayed to a more privileged endpoint even if it leaks.

RFC 8693 defines a token-exchange grant type for exactly this:

- `grant_type=urn:ietf:params:oauth:grant-type:token-exchange`
- `subject_token=<the broader token>` (the client's current credential)
- `subject_token_type=urn:ietf:params:oauth:token-type:access_token` (or JWT if signed)
- `requested_token_type=urn:ietf:params:oauth:token-type:access_token`
- `scope=<the narrower scope>` (the subset you want for this call)
- `resource=<target MCP server URI>` (RFC 8707 again)
- `audience=<optional, same shape>` (for cases the authorization server distinguishes)
- Optionally `actor_token` and `actor_token_type` when a service is delegating *on behalf of* another identity (the actor claim lands in the issued token so audit knows "the gateway acted for user U")

Three gateway-plane use cases:

1. **Per-call scope narrowing.** Session token → narrow token right before invocation. This is the pattern the policy plane (mod-405) will ask you to implement when it approves a specific scope for a specific call.
2. **Impersonation across planes.** The user's token → downstream service token for a backend the tool hits on the user's behalf. The actor claim in the new token names your gateway's identity (SPIFFE SVID; mod-405) so the downstream service can distinguish "user U, via gateway G" from "user U directly."
3. **Trust bridging.** Your gateway's own token (service-to-service) → a user-scoped token at a trusted internal auth server when the gateway acts as the user's deputy. The trust bridge has to be explicitly configured — this is one of the places a confused deputy sneaks in, so enumerate the allowed bridges and refuse the rest.

If your authorization server does not support RFC 8693, the gateway can still enforce narrowing by requesting the narrow scope at authorization time, caching multiple access tokens per user+resource, and picking the narrowest token that satisfies the call. It is less elegant but functionally equivalent.

## Scope design

Scopes are the string vocabulary the end user sees on the consent screen and the thing auditors read to answer "did the agent exceed the user's consent?" Two rules:

1. **One scope, one capability.** `calendar.read` and `calendar.write` are two scopes, not one. `files.read_user_scope` and `files.read_tenant_scope` are two scopes. Do not bundle. Agents that ask for more than they need lose user trust; agents that ask for less than they need get up-front consent fatigue; the only way to tune this is to have granular scopes to tune *with*.
2. **Scope strings mirror the tool-schema invocation.** If the tool `send_email` has a `cc` parameter, and `cc` is sensitive in some contexts, make it a distinct scope (`email.send`, `email.send_cc`). Do not let agents sneak scope through argument shape.

The registry schema (chapter 03) carries the scope vocabulary per server, including the human-readable description that lands on the consent screen.

## Access-token lifetime, refresh, and rotation

- **Access tokens are short-lived.** Minutes, not hours. The gateway refreshes transparently when it sees a `401` or when its TTL is nearly up.
- **Refresh tokens are long-lived but rotated.** Every successful refresh issues a new refresh token (if the authorization server supports rotation; most do in 2.1-compliant deployments). The gateway stores the new one and invalidates the old.
- **Reuse detection.** If the gateway ever sees a refresh token it has already consumed, something is wrong (replay, leaked store). Revoke the whole grant; force re-consent. Many authorization servers do this server-side — honour their revocation.
- **Explicit revocation.** On tenant offboarding, session end, or an incident, the gateway calls the authorization server's revocation endpoint for every token in the affected scope.

## A worked example (end to end)

A product team registers an MCP server `calendar-mcp` that wraps Google Calendar. The gateway's handling of a single tool call looks like this:

1. Agent in session `s123` for user `alice@acme.com` wants `tools/call` on `calendar.find_free_slot` (which `calendar-mcp` advertises; scope `calendar.read`).
2. Gateway looks up session `s123` in its grant store; no token for `calendar-mcp.example.com` with scope `calendar.read` yet.
3. Gateway redirects Alice (through the runtime's UI channel) to Google's authorization endpoint with `resource=https://calendar-mcp.example.com/mcp`, `scope=calendar.read`, PKCE `S256`.
4. Alice consents in Google. Google redirects back to the gateway with a code.
5. Gateway exchanges code for access token, bound to `resource=https://calendar-mcp.example.com/mcp`, `scope=calendar.read`.
6. Gateway calls `calendar-mcp` with `Authorization: Bearer <token>`.
7. `calendar-mcp` validates `aud` matches its own identifier and `scope` contains `calendar.read`, then calls Google's Calendar API with Alice's token (which it holds separately, having run its own OAuth with Google).

Note how **two** auth flows happen. The gateway holds the token to `calendar-mcp` (the MCP resource server). `calendar-mcp` holds the token to Google Calendar. The two must not be conflated: the gateway never sees Google Calendar's token; `calendar-mcp` never sees the gateway's platform session. This separation is what keeps scopes mapping cleanly to consent screens.

## Summary

- MCP authorization is OAuth 2.1 end to end. If you cannot recite the PKCE flow, the discovery chain (RFC 9728 → RFC 8414), DCR (RFC 7591), Resource Indicators (RFC 8707), and Token Exchange (RFC 8693), the rest of this track is harder than it needs to be.
- The gateway is the OAuth client. Agents never hold raw tokens. Scopes are the end-user-visible consent vocabulary; the gateway owns the mapping between scope strings and tool capabilities.
- Resource Indicators defeat the confused deputy. Refuse to admit any MCP server or authorization server that does not support them in the production lane.
- Token Exchange is the right primitive for per-call scope narrowing and for impersonation across planes. If your authorization server does not support it, fall back to multi-token caches, but note the gap.

The next chapter picks up where auth meets operations: the registry that records which servers exist, which scopes they declare, which version they are on, and when they get retired.
