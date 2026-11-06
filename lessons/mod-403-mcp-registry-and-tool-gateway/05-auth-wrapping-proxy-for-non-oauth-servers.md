# Auth-Wrapping Proxy for Non-OAuth Servers

Chapter 02 described the happy path: the MCP server is an OAuth 2.1 resource server, the gateway is the client, scopes are user-consented, tokens are narrowly scoped. In practice, the fleet of things your agents need to reach does not all look like that. Internal REST APIs authenticated by static tokens, vendor SDKs with signed-request schemes, machine-to-machine services behind mTLS, legacy ERP endpoints with username/password — all of these have a right to show up behind a tool. The gateway's third-tier substrate is the **auth-wrapping proxy**: an MCP server (or a facet of the gateway itself) that terminates the MCP + OAuth 2.1 story with the agent, and then mints or holds the right downstream credential to call the actual thing.

Designed well, the proxy keeps the plane's three obligations intact. Designed badly, it is the hole that destroys the audit trail, leaks credentials, and lets scopes drift. This chapter spells out the design.

## The problem, stated precisely

An agent's session has a user identity (`alice@acme.com`), a scoped token issued by the gateway's auth path, and permission to invoke a tool named `erp.lookup_invoice`. The tool is implemented by a legacy ERP system whose only auth is a long-lived API key issued to the finance team's service account. Three things must happen:

1. **The gateway must enforce the OAuth 2.1 story with the agent.** The agent only sees MCP + OAuth; it holds no ERP credential.
2. **The call must land at the ERP with a credential the ERP accepts.** The ERP does not understand OAuth bearer tokens; it understands its own API key.
3. **The record of who-did-what must stay attributable.** It must be possible, at audit time, to prove that Alice invoked `erp.lookup_invoice` with these inputs, that policy allowed it, and that the ERP's API key was used as the gateway-managed deputy — not that "the finance service account" called it, which would collapse everyone's activity into a single actor.

The auth-wrapping proxy is the component that solves all three without leaking any of them.

## The architecture, in one diagram

```
Agent ── MCP + OAuth 2.1 ──▶  Auth-Wrapping Proxy (MCP server)
                                     │
                                     │ (reads policy, brokers credential)
                                     ▼
                              Policy plane (mod-405)      ◀── decisions
                                     │
                                     ▼
                              Secret broker / IdP
                                     │
                                     ▼
                              Downstream legacy system
                              (API key / mTLS / signed request / etc.)
```

The proxy is a normal MCP server from the gateway's perspective: it is registered, it has a schema, it has scopes, it passes conformance. Its *implementation* just happens to be "receive the MCP call, consult the policy plane, broker a downstream credential, forward the call, translate the response back into MCP." From outside, it is indistinguishable from a native MCP server.

## What the proxy owns

- **Terminating OAuth 2.1 with the agent.** The proxy is an OAuth 2.1 resource server. It validates the bearer token, checks the `aud` claim against its own identifier, checks the scope against the tool's required scopes, and runs rate limits per user.
- **Brokering the downstream credential.** The proxy never persists the downstream credential in a form agents can see. It fetches short-lived credentials from a secret broker (HashiCorp Vault, AWS Secrets Manager, GCP Secret Manager, Azure Key Vault), from an IdP via workload-identity federation, or from SPIFFE/SPIRE (mod-405).
- **Invocation.** The proxy calls the downstream system with the brokered credential, enforces the configured per-tenant quotas, catches and classifies errors.
- **Translation.** Downstream responses get translated into MCP tool-call results. Errors get mapped to structured MCP error codes. Nothing leaks a stack trace, a database row count, or an internal hostname that was not deliberately exposed.
- **Attribution.** The proxy writes the audit record (chapter 07) with the full tuple: user, agent, session, policy decision, downstream credential identifier (not the secret itself), inputs, outputs, timestamp.

## What the proxy does not own

- **Policy.** It consults mod-405 for every call; it does not decide allow/deny itself.
- **The downstream system's auth model.** It consumes whatever credentials the broker gives it; it does not generate credentials from scratch. If the downstream needs a new credential rotation flow, that is the broker's job.
- **The business logic.** The proxy's code is auth, policy consult, invoke, translate, attribute. If it is doing more, it is drifting into tool-authorship territory (which is `agentic-ai-engineer` level 30).

## Credential patterns

The proxy mints or retrieves a downstream credential per invocation. Pick the pattern that fits the downstream.

### Static API key, shared across tenants (worst-case baseline)

The downstream has one long-lived API key. Every proxy call uses it. The proxy is still the control point, and the audit record preserves the user attribution (because the proxy writes it), but at the downstream the activity collapses to "the API key owner did everything."

Use only when:

- The downstream is read-only and low-risk.
- The downstream cannot be improved (third-party vendor, legacy mainframe).
- The audit obligation can be met inside the proxy (because downstream logs are useless).

Mitigate by:

- Per-tenant rate limits at the proxy (so one tenant cannot burn the shared quota).
- Per-user quota tracking in the proxy's own ledger (so a user's total ERP activity is visible even though the downstream can't see it).
- Regular rotation of the API key (manual if the downstream has no rotation API).

### Per-tenant static API keys

The downstream accepts different API keys for different tenants (e.g. a SaaS vendor's per-customer API key). Each tenant's key is stored in the broker; the proxy fetches `keys/{tenant_id}` for each invocation.

Better than shared-key because:

- The downstream can distinguish tenants in its own audit (useful for billing, abuse response, regulatory demand).
- A compromise is scoped to one tenant.

Still not great: the key is long-lived and static.

### Short-lived credentials from a workload identity federation (preferred)

Cloud IAM supports exchanging a signed identity (OIDC, SAML, SPIFFE SVID) for a short-lived cloud credential:

- **AWS**: `AssumeRoleWithWebIdentity` returns a 15-minute STS token for an IAM role.
- **GCP**: Workload Identity Federation returns a 1-hour access token for a service account.
- **Azure**: Federated credentials exchange an OIDC token for an Azure AD token.

The proxy, identified by a SPIFFE SVID (from mod-405's SPIRE deployment), exchanges its SVID for a downstream cloud credential bound to (role, tenant, request_id). The credential expires in minutes; even a leaked record is low-value quickly.

This is the pattern to default to when the downstream is in a cloud that supports it.

### User-impersonation via token exchange (for user-scoped downstreams)

When the downstream is itself OAuth-compatible but the agent cannot complete its OAuth flow directly (e.g. headless/background agents, non-interactive sessions), the proxy may use RFC 8693 **token exchange** to convert the user's session token into a downstream access token:

- `subject_token=<user session token>`
- `actor_token=<proxy's own identity>`
- `audience=<downstream>`
- `scope=<narrower than the subject>`

The resulting token carries the user as subject and the proxy as actor; downstream audit shows both. Use for internal systems the organisation operates and trusts; be cautious bridging to third parties (confused-deputy risk).

### Signed-request schemes (AWS SigV4 and relatives)

Some downstream services do not take bearer tokens; they take requests signed with a per-request HMAC (AWS SigV4, cloud vendor proprietary schemes). The proxy holds the signing key (or a short-lived derivation of it) and signs each outbound request. The signing key never leaves the proxy's address space.

### mTLS to the downstream

For internal mTLS-protected services, the proxy terminates the OAuth 2.1 flow and establishes mTLS downstream using its SPIFFE SVID. The downstream's SPIFFE trust domain decides whether the proxy is authorised at all; the proxy's policy-plane consult decides whether *this call* is authorised.

## The scope-to-permission mapping

The MCP scope is user-facing (what the end user consented to); the downstream permission is operator-facing (what the ERP role can actually do). The proxy's registry entry declares the mapping explicitly:

```
scope_mapping:
  - scope: "erp.read_invoice"
    description: "Read invoices for the user's cost centres"
    downstream_role: "role/erp-invoice-reader"
    constraints:
      cost_centre: "{user.cost_centres}"   # ABAC filter
  - scope: "erp.update_invoice"
    description: "Update invoice fields in the user's cost centres"
    downstream_role: "role/erp-invoice-editor"
    constraints:
      cost_centre: "{user.cost_centres}"
      field_allowlist: ["note", "due_date"] # per-field permit list
```

The mapping is read by:

- **The consent screen**, to show the user what "erp.read_invoice" means.
- **The policy plane (mod-405)**, to evaluate whether this user, in this context, may use this scope on this call.
- **The proxy**, to pick which role or role-plus-constraints to broker.
- **The audit pipeline (chapter 07)**, to correlate "user U had scope S" with "downstream permission P was exercised."

A scope that has no mapping is not callable through the proxy. A mapping change is a MINOR or MAJOR version bump depending on whether it loosens or tightens access.

## User context pass-through, when the downstream needs it

For tools that must act as the end user on the downstream (not as the proxy's deputy), the proxy forwards a user-identity claim rather than its own. The forwarding mechanism depends on the downstream:

- **OAuth-aware downstreams**: token exchange as above (subject = user).
- **mTLS downstreams**: proxy identifies itself; user identity rides in a signed sidecar header (`X-Forwarded-User` plus a proxy signature the downstream verifies).
- **Signed-request schemes**: user identity in the request payload or a signed sidecar.

If no viable pass-through exists, the tool cannot act as the user — it acts as the deputy, and the UI/scope description must say so clearly. Ambiguity here is where users get surprised and incidents happen.

## Where secrets live

- **Never in the registry entry.** The registry references credentials by name; the broker holds the bytes.
- **Never in the agent, the runtime, or the prompt.** Not even in ephemeral form; not even "briefly."
- **Never on disk in plaintext.** Even in-memory, prefer short-lived handles.
- **Rotated on a schedule, not on an incident.** If you only rotate when a leak happens, you rotate too late.

The broker is operated by the policy plane (mod-405). Chapter 02 of mod-405 covers broker design; the proxy is a consumer.

## Rate limiting and quota

Because the proxy may hold the only accurate view of per-user activity (when the downstream collapses everyone into a service-account identity), the proxy runs its own rate limits:

- **Per user, per scope** — the primary quota.
- **Per tenant** — aggregate ceiling to prevent one tenant from starving the shared downstream.
- **Per tool** — because some tools are expensive at the downstream and some are cheap.
- **Priority tiers** — paved-road users get precedence under contention.

Rate-limit decisions are made by the policy plane (mod-405) for consistency with other policy enforcement. The proxy consumes the decision; it does not author the quota math.

## Anti-patterns worth naming

1. **Embedding the downstream API key in the MCP server source** so it ships baked into the container. Kills rotation; leaks into logs; usually found in postmortems.
2. **A single service account for all tenants** when per-tenant credentials are possible. Collapses audit; magnifies incident blast radius.
3. **Letting the agent pass the API key through** as a tool argument. The agent should not know the credential exists. If the tool's schema has a `api_key` field, something upstream is wrong.
4. **Logging the Authorization header** or the token body in debug mode. Scrub at the logger; use a structured redaction policy.
5. **Reusing the same proxy container to front multiple downstreams.** The proxy is small; run one per downstream so a compromise does not cascade.
6. **Treating the proxy as "stateless middleware."** It holds secrets in memory, writes audit records, enforces quotas; it is a stateful component with the deployment and ops considerations that implies.
7. **Static API keys shared across tenants when a federated alternative exists.** This is the most common degradation path; the broker side (mod-405) has the primitives, use them.

## Summary

- Not every MCP server speaks OAuth 2.1. The auth-wrapping proxy is the pattern for admitting legacy tools without breaking the plane's obligations.
- From outside, the proxy is a normal MCP server — registered, versioned, conformance-tested. Inside, it terminates OAuth with the agent, consults policy (mod-405), brokers a downstream credential (short-lived from workload identity federation when possible, static only when unavoidable), invokes the downstream, and writes an attributable audit record.
- The scope-to-permission mapping is explicit, versioned, and read by the consent UI, the policy plane, the proxy, and the audit pipeline. A scope with no mapping is not callable.
- Secrets live in the broker. Rotation is scheduled, not incident-driven. Per-tenant credentials beat shared credentials; short-lived beats static. Impersonation beats deputy only when the pass-through mechanism is sound.
- The proxy is a stateful component. Run one per downstream; run it like a production service, not like a config layer.

The next chapter zooms out: for the long tail of tools you would rather not author at all, there is a managed-MCP vendor landscape. The right question is build-vs-buy, with the gateway as the control point in both cases.
