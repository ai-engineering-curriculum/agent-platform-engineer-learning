# exercise-03: SPIFFE / SPIRE Workload Identity for Agents

**Estimated effort:** 4 hours · **Output:** a running SPIRE deployment (local Kubernetes acceptable), registration entries for three workloads, an mTLS integration between two of them, a Token Exchange wiring that carries a JWT-SVID as `actor_token`, and a diagnostics memo (~1 page) · **Pairs with:** chapter [04](../04-non-human-identity-spiffe-oauth-and-credential-brokering.md); feeds project 402.

## Objective

Stand up the identity layer the plane depends on. You will deploy SPIRE, attest three workloads as distinct SPIFFE identities, wire mTLS between two of them, perform a Token Exchange (RFC 8693) using a JWT-SVID as the `actor_token`, and leave a diagnostics memo a teammate can use to unblock the next person who hits the same failure mode.

A well-done exercise produces a deployment where every workload can prove its identity, every peer refuses the wrong identity, and the decision log on the subsequent policy decision (from exercise 01) carries the SPIFFE ID of the acting workload *verified* — not merely claimed.

## Prerequisites

- You have read chapter [04](../04-non-human-identity-spiffe-oauth-and-credential-brokering.md).
- You have a local Kubernetes cluster you can push workloads to — [kind](https://kind.sigs.k8s.io/), [minikube](https://minikube.sigs.k8s.io/), [k3d](https://k3d.io/), or a managed cluster you can lab in.
- You have the [SPIFFE / SPIRE documentation](https://spiffe.io/docs/latest/) and the [SPIFFE Helm Charts Hardened](https://github.com/spiffe/helm-charts-hardened) open.
- You have Docker, `kubectl`, and `helm` locally.
- Optional: [go-spiffe](https://github.com/spiffe/go-spiffe) or [py-spiffe](https://github.com/HewlettPackard/py-spiffe) if you prefer those bindings over shell.

## Problem statement

Stand up a three-workload plane:

- **`gateway`** — the component that receives agent tool calls. In this exercise, a stub HTTP server.
- **`pdp`** — the policy decision point running your OPA policy from exercise 01.
- **`broker`** — the credential broker. A stub that returns a signed lease.

Each workload runs in its own namespace under its own ServiceAccount. SPIRE issues SVIDs. The gateway calls the PDP over mTLS, verifying the PDP's SPIFFE ID. The gateway then calls the broker, presenting both its X.509-SVID (for mTLS) and a JWT-SVID (as the `actor_token` in a mock Token Exchange).

A fourth stub, the **authorization server**, accepts the Token Exchange request and issues a short-lived JWT access token whose `act` claim is the gateway's SPIFFE ID and whose `sub` is a hard-coded `alice@acme.com`. (You do not need a full OAuth provider; a signed-JWT issuer with the right claim shape is enough.)

## Requirements

### Part 1 — SPIRE deployment

- Install SPIRE in the cluster using the Helm charts. Use the default trust domain `platform.local` or a trust domain of your choice; document it.
- Confirm the SPIRE Server is up (`spire-server healthcheck`), the SPIRE Agent DaemonSet is running on every node, and the Workload API socket is mounted into test pods.
- Enable the Kubernetes PSAT or SAT [node attestor](https://github.com/spiffe/spire/blob/main/doc/plugin_agent_nodeattestor_k8s_psat.md), whichever matches your cluster setup.
- Add the [SPIFFE CSI Driver](https://github.com/spiffe/spiffe-csi) so the Workload API socket mounts via CSI rather than hostPath. (If your cluster version does not support this, document the deviation.)

### Part 2 — Registration entries for the three workloads

Create a SPIFFE ID for each workload using `spire-server entry create`. Example for the gateway:

```bash
spire-server entry create \
  -spiffeID spiffe://platform.local/gateway \
  -parentID spiffe://platform.local/spire/agent/k8s_psat/<cluster-name>/<node-name-pattern> \
  -selector k8s:ns:platform-gateway \
  -selector k8s:sa:gateway \
  -ttl 60
```

Repeat for the PDP (`spiffe://platform.local/pdp`) and broker (`spiffe://platform.local/broker`). Use selectors that will *not* accidentally match other workloads in the cluster (namespace + ServiceAccount at a minimum; pod-label if your cluster has shared ServiceAccounts).

Confirm that a dev pod *without* matching selectors cannot receive any SVID. This is the "wrong workload cannot get the identity" invariant.

### Part 3 — mTLS gateway → PDP

- Deploy the gateway stub as a pod in `platform-gateway/gateway` with the Workload API socket mounted.
- Deploy the PDP stub as a pod in `platform-pdp/pdp` similarly. The PDP stub mounts your OPA policy from exercise 01 and evaluates the input it receives.
- Both services fetch their X.509-SVID via the Workload API on startup.
- The gateway calls the PDP over HTTPS with the gateway's X.509-SVID as the client cert; the PDP validates (a) the cert is signed by the SPIRE trust bundle, (b) the subject URI is `spiffe://platform.local/gateway`. Any other URI returns 401 before policy runs.
- Reverse: the gateway validates the PDP's server cert against the trust bundle and the expected subject URI.

Confirm:

- A gateway with the right SVID can reach the PDP.
- A rogue pod without a matching SPIRE entry cannot reach the PDP (TLS handshake fails).
- An expired SVID (test by stopping SPIRE for the TTL window, or shortening the TTL for this test) causes new connections to fail with a readable error.

### Part 4 — Token Exchange with the gateway's JWT-SVID as `actor_token`

- Deploy a stub authorization server (any language; a 50-line HTTP service that responds to `POST /oauth2/token` with `grant_type=urn:ietf:params:oauth:grant-type:token-exchange` is enough). It validates incoming JWT-SVIDs against the SPIRE JWT bundle, mints a signed JWT access token with:
  - `sub`: `alice@acme.com`
  - `aud`: `broker` (or a configurable resource identifier)
  - `scope`: whatever the Token Exchange request named
  - `act`: `{ "sub": "spiffe://platform.local/gateway" }` (RFC 8693 §4.1)
  - `exp`: 300 seconds from now
- The gateway fetches a JWT-SVID with audience `authorization-server` via the Workload API and presents it as the `actor_token` in the Token Exchange.
- The authorization server returns the mint; the gateway then calls the broker with the access token in `Authorization: Bearer ...` plus its X.509-SVID on mTLS.
- The broker validates the bearer token: signature against the auth server's JWKS, `aud` matches broker's expected audience, `act.sub` matches the TLS peer's SPIFFE ID.

Confirm:

- A gateway can mint a `sub=alice, act=spiffe://platform.local/gateway` access token.
- If the gateway's TLS SPIFFE ID does not match the `act.sub` in the token, the broker refuses with a clear reason code.
- An expired bearer token is refused; the gateway refreshes and continues.

### Part 5 — Diagnostics memo

One page with:

- **Topology diagram** (ASCII is fine): workloads, SVIDs, trust bundle, the auth server, the directional arrows for every call.
- **What broke first.** The first failure you hit while bringing this up — SPIRE registration selector mismatch, CSI driver mount, trust-bundle propagation delay, something else. Write the failure mode and the fix so the next engineer saves an hour.
- **One check you will add to CI.** For example: an admission test that spins up a pod without the required selectors and asserts it cannot get an SVID.
- **One gap you noticed between the chapter and reality.** Chapter 04 is a design document; operational reality has rough edges. Name one.

## Starter guidance

- Shorten TTLs during the exercise. SPIRE defaults are reasonable for production; 60 seconds is reasonable for development. Shorter than 30 seconds will hammer the Workload API and may cause failures that look like bugs.
- `spire-server entry show -spiffeID ...` and `spire-agent api fetch` are your friends; use them often.
- Use `openssl s_client -connect <host>:<port> -showcerts` and `openssl x509 -text -noout` to confirm the certificate subject URI is what you expect. The SPIFFE ID is in the SAN as a URI, not in the CN.
- Validate JWT-SVIDs against the SPIRE JWT bundle (`spire-agent api fetch jwt -audience auth-server`), not against SPIRE's X.509 bundle. The two are different key sets.
- Do not reach for Istio or Linkerd for mTLS on this exercise. Service meshes do SPIFFE too, but adding one complicates the diagnostics. Direct Workload API integration is the one to learn here.
- Watch for *trust-bundle propagation delay*. When you rotate SPIRE's CA, old peers hold the old bundle for the Workload API's refresh interval. This is the operational failure that gets misreported as "SPIRE broke".
- Flag `<!-- needs-research: ... -->` on anything you cannot verify against the current SPIRE release notes — particularly if the Helm chart version and the SPIRE binary version drift.

## Acceptance criteria

Mark done when:

- [ ] SPIRE server, agent, and CSI driver are up; registrations exist for all three workloads.
- [ ] Gateway → PDP mTLS works; a rogue pod cannot handshake.
- [ ] Gateway → auth-server → broker Token Exchange works; `act.sub` matches the gateway's SPIFFE ID; the broker refuses if they do not match.
- [ ] The decision log emitted by the PDP for a successful call carries the gateway's SPIFFE ID as `session.actor.spiffe_id` *verified* (i.e. the PDP read it from the mTLS peer identity, not from the request body).
- [ ] The diagnostics memo names the first failure, the fix, one CI check, and one gap.

## Stretch goals

- **Federation.** Add a second trust domain (`staging.local`) and configure one-way federation from `platform.local` to `staging.local`. Prove a staging workload can authenticate to a production broker in a controlled way.
- **Revocation drill.** Simulate a compromise: assume `gateway` is breached and all its SVIDs must become invalid immediately. Walk through the two primitives (`spire-server entry delete` vs. CA rotation), measure the time-to-revocation each produces, and write up the choice your plane would make.
- **Workload-identity-aware policy input.** Update your OPA policy from exercise 01 to key an allow rule on the SPIFFE ID of the actor (e.g. `only gateway/prod can call tool.invoke; gateway/canary is denied with a different reason code`). Tests prove both.
- **Vault integration.** Stand up a local Vault; configure the [JWT auth method](https://developer.hashicorp.com/vault/docs/auth/jwt) to accept JWT-SVIDs issued by SPIRE. Prove a workload can `vault read` a secret policy-bound to its SPIFFE ID, and cannot read secrets bound to a different SPIFFE ID.
- **Automate the admission test.** Script the "rogue pod cannot get an SVID" check so it runs on every SPIRE Helm upgrade in CI.
