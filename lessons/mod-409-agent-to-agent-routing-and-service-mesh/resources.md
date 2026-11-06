# Resources — mod-409 (Agent-to-Agent Routing and Inter-Agent Service Mesh)

All references on this page are primary / authoritative public sources at the time of writing (2026-11-06). Each resource is tagged with the chapter(s) it supports. Prefer the primary source over any blog summary — the A2A spec, the service-mesh CRDs, and the OAuth 2.0 token-exchange support matrix across authz servers all shift quarter by quarter and secondary sources lag.

## Chapter 01 — The Inter-Agent Routing Plane and What It Owns

### Platform-engineering background
- **CNCF — Platforms White Paper**: <https://tag-app-delivery.cncf.io/whitepapers/platforms/>
- **CNCF — Platform Engineering Maturity Model**: <https://tag-app-delivery.cncf.io/whitepapers/platform-eng-maturity-model/>
- **Google SRE Book — Addressing Cascading Failures**: <https://sre.google/sre-book/addressing-cascading-failures/>

### Agent-platform context
- **Anthropic — Building Effective Agents**: <https://www.anthropic.com/research/building-effective-agents>
- **OpenAI — A Practical Guide to Building Agents (PDF)**: <https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf>
- **OpenAI — Agents SDK documentation (handoffs primitive)**: <https://openai.github.io/openai-agents-python/>
- **LangGraph — Multi-agent systems**: <https://langchain-ai.github.io/langgraph/concepts/multi_agent/>
- **CrewAI — documentation**: <https://docs.crewai.com/>

## Chapter 02 — The A2A Protocol and the Service-Mesh Mapping

### Agent2Agent (A2A)
- **A2A — hosted specification and overview**: <https://a2a-protocol.org/>
- **A2A — project repository (Linux Foundation)**: <https://github.com/a2aproject>
- **A2A — Python SDK (reference)**: <https://github.com/a2aproject/a2a-python>
- **A2A — JavaScript / TypeScript SDK (reference)**: <https://github.com/a2aproject/a2a-js>
<!-- needs-research: A2A's hosted spec and repository have moved under the Linux Foundation and been iterated. Confirm the current canonical URL, method inventory, task-state alphabet, and AgentCard well-known path before pinning them in production artefacts. -->

### JSON-RPC and transport
- **JSON-RPC 2.0 specification**: <https://www.jsonrpc.org/specification>
- **W3C Server-Sent Events (part of HTML)**: <https://html.spec.whatwg.org/multipage/server-sent-events.html>
- **OIDC Discovery (well-known pattern)**: <https://openid.net/specs/openid-connect-discovery-1_0.html>

### Service meshes — Istio and Envoy
- **Istio — documentation**: <https://istio.io/latest/docs/>
- **Istio — traffic management concepts**: <https://istio.io/latest/docs/concepts/traffic-management/>
- **Istio — VirtualService reference**: <https://istio.io/latest/docs/reference/config/networking/virtual-service/>
- **Istio — DestinationRule reference**: <https://istio.io/latest/docs/reference/config/networking/destination-rule/>
- **Istio — PeerAuthentication / AuthorizationPolicy**: <https://istio.io/latest/docs/reference/config/security/>
- **Envoy — documentation**: <https://www.envoyproxy.io/docs/envoy/latest/>
- **Envoy — xDS protocol**: <https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds_protocol>

### Service meshes — Linkerd
- **Linkerd — documentation**: <https://linkerd.io/2/overview/>
- **Linkerd — how it works**: <https://linkerd.io/2/reference/architecture/>
- **Buoyant — Linkerd releases**: <https://buoyant.io/>

### Model Context Protocol (boundary reference)
- **MCP — specification (cross-ref mod-403)**: <https://modelcontextprotocol.io/specification>

## Chapter 03 — Inter-Agent Routing and Service Discovery

### Trace context and baggage (cross-ref mod-408)
- **W3C Trace Context (Recommendation)**: <https://www.w3.org/TR/trace-context/>
- **W3C Baggage**: <https://www.w3.org/TR/baggage/>
- **OpenTelemetry — context propagation API spec**: <https://opentelemetry.io/docs/specs/otel/context/api-propagators/>

### Routing, load balancing, locality
- **Envoy — load balancing**: <https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/load_balancers>
- **Istio — locality load balancing**: <https://istio.io/latest/docs/tasks/traffic-management/locality-load-balancing/>
- **Istio — multi-cluster installation**: <https://istio.io/latest/docs/setup/install/multicluster/>
- **Linkerd — multi-cluster**: <https://linkerd.io/2/features/multicluster/>

### Deadlines and timeouts
- **gRPC — deadlines guide**: <https://grpc.io/blog/deadlines/>
- **Google — Time, Clocks, and the Ordering of Events (Lamport, 1978)**: <https://lamport.azurewebsites.net/pubs/time-clocks.pdf>
- **AWS — Keeping time with Amazon Time Sync Service**: <https://aws.amazon.com/blogs/aws/keeping-time-with-amazon-time-sync-service/>

### Idempotency and retries (foundational)
- **AWS — Exponential Backoff and Jitter (Architecture Blog)**: <https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/>
- **Stripe API — idempotency documentation**: <https://stripe.com/docs/api/idempotent_requests>

## Chapter 04 — Identity Delegation and Permission-Scope Propagation

### OAuth 2.0 and token exchange
- **OAuth 2.0 — RFC 6749**: <https://datatracker.ietf.org/doc/html/rfc6749>
- **OAuth 2.0 — Token Exchange (RFC 8693)**: <https://datatracker.ietf.org/doc/html/rfc8693>
- **OAuth 2.0 — JWT Profile for Access Tokens (RFC 9068)**: <https://datatracker.ietf.org/doc/html/rfc9068>
- **OIDC Core**: <https://openid.net/specs/openid-connect-core-1_0.html>
- **JWT (RFC 7519)**: <https://datatracker.ietf.org/doc/html/rfc7519>
- **JOSE (RFC 7515 / 7516 / 7517 / 7518)**: <https://datatracker.ietf.org/doc/html/rfc7515>

### Authorisation server implementations
- **Keycloak — documentation**: <https://www.keycloak.org/documentation>
- **Keycloak — token exchange**: <https://www.keycloak.org/securing-apps/token-exchange>
- **Ory Hydra — documentation**: <https://www.ory.sh/docs/hydra/>
- **Auth0 — token vault and exchange**: <https://auth0.com/docs/>
- **Google Cloud — Workload Identity Federation**: <https://cloud.google.com/iam/docs/workload-identity-federation>
<!-- needs-research: not every authorisation server supports RFC 8693 identically. Confirm `act` / `may_act` claim handling, the `actor_token` parameter, and scope-intersection validation against the specific server a production deployment uses. -->

### Workload identity
- **SPIFFE — home**: <https://spiffe.io/>
- **SPIFFE — concepts**: <https://spiffe.io/docs/latest/spiffe-about/spiffe-concepts/>
- **SPIFFE — SVID / X.509 identity**: <https://spiffe.io/docs/latest/spiffe-about/spiffe-concepts/#spiffe-verifiable-identity-document>
- **SPIRE — architecture**: <https://spiffe.io/docs/latest/spire-about/spire-concepts/>
- **SPIFFE — federation**: <https://spiffe.io/docs/latest/architecture/federation/>
- **Istio — security concepts (SPIFFE-based)**: <https://istio.io/latest/docs/concepts/security/>

### Tenancy, authorisation policy, and privacy
- **Istio — AuthorizationPolicy reference**: <https://istio.io/latest/docs/reference/config/security/authorization-policy/>
- **GDPR — Regulation (EU) 2016/679**: <https://eur-lex.europa.eu/eli/reg/2016/679/oj>
- **EDPB — guidelines**: <https://www.edpb.europa.eu/our-work-tools/general-guidance/guidelines-recommendations-best-practices_en>
- **NIST SP 800-207 — Zero Trust Architecture**: <https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-207.pdf>

## Chapter 05 — Circuit Breakers, Bulkheads, and Cascading-Failure Prevention

### Foundational
- **Nygard — Release It! (2nd edition)**: <https://pragprog.com/titles/mnee2/release-it-second-edition/>
- **Google SRE Book — Addressing Cascading Failures**: <https://sre.google/sre-book/addressing-cascading-failures/>
- **Google SRE Book — Handling Overload**: <https://sre.google/sre-book/handling-overload/>
- **AWS — Summary of the Amazon S3 Service Disruption in the Northern Virginia Region**: <https://aws.amazon.com/message/41926/>
- **Dean & Barroso — The Tail at Scale (CACM, 2013)**: <https://research.google/pubs/the-tail-at-scale/>

### Circuit-breaker and bulkhead implementations
- **Netflix — Hystrix (archived; historical reference)**: <https://github.com/Netflix/Hystrix>
- **Resilience4j — documentation**: <https://resilience4j.readme.io/>
- **Resilience4j — bulkhead**: <https://resilience4j.readme.io/docs/bulkhead>
- **Polly (.NET resilience)**: <https://www.pollydocs.org/>
- **pybreaker (Python)**: <https://github.com/danielfm/pybreaker>
- **opossum (Node)**: <https://github.com/nodeshift/opossum>
- **gobreaker (Go)**: <https://github.com/sony/gobreaker>

### Mesh-level circuit breaking and outlier detection
- **Envoy — circuit breaking**: <https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/circuit_breaking>
- **Envoy — outlier detection**: <https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/outlier>
- **Envoy — retry policy**: <https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/router_filter>
- **Envoy — retry budget**: <https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/cluster/v3/circuit_breaker.proto#envoy-v3-api-msg-config-cluster-v3-circuitbreakers-thresholds-retrybudget>
- **Istio — HTTPRetry reference**: <https://istio.io/latest/docs/reference/config/networking/virtual-service/#HTTPRetry>
- **Istio — circuit breaking task**: <https://istio.io/latest/docs/tasks/traffic-management/circuit-breaking/>
- **Linkerd — retries and timeouts**: <https://linkerd.io/2/features/retries-and-timeouts/>

### Load generation and chaos
- **vegeta — HTTP load testing**: <https://github.com/tsenart/vegeta>
- **hey — HTTP load generator**: <https://github.com/rakyll/hey>
- **k6 — load testing**: <https://k6.io/docs/>
- **Chaos Mesh**: <https://chaos-mesh.org/>
- **LitmusChaos**: <https://litmuschaos.io/>

## Chapter 06 — Handoffs vs. Manager-as-Tools and Role Handoffs

### Agent topology patterns
- **Anthropic — Building Effective Agents (routing, orchestrator-workers)**: <https://www.anthropic.com/research/building-effective-agents>
- **OpenAI — A Practical Guide to Building Agents (manager pattern, handoffs)**: <https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf>
- **OpenAI Agents SDK (handoffs primitive)**: <https://openai.github.io/openai-agents-python/>
- **OpenAI Swarm (predecessor; archived reference)**: <https://github.com/openai/swarm>
- **LangGraph — supervisor pattern**: <https://langchain-ai.github.io/langgraph/concepts/multi_agent/#supervisor>
- **CrewAI — hierarchical crews**: <https://docs.crewai.com/>
- **AutoGen — multi-agent conversation framework**: <https://microsoft.github.io/autogen/>

### Durable execution (cross-ref mod-402)
- **Temporal — documentation**: <https://docs.temporal.io/>
- **Inngest — documentation**: <https://www.inngest.com/docs>
- **Restate — documentation**: <https://docs.restate.dev/>

### Role boundaries (within the Agent Career Curriculum ecosystem)
- **Track overview (this repository)**: `../../README.md`
- **Agent runtime module (mod-402)**: `../mod-402-agent-runtime-service-and-durable-execution/`
- **Observability module (mod-408)**: `../mod-408-observability-and-opentelemetry-for-agents/`
- **Reliability module (mod-410)**: `../mod-410-reliability-engineering-and-sre-for-agents/`

## Cross-cutting references

- **A2A specification** (chapter 02 primary source): <https://a2a-protocol.org/>
- **W3C Trace Context** (cross-ref mod-408): <https://www.w3.org/TR/trace-context/>
- **RFC 8693 — OAuth 2.0 Token Exchange** (chapter 04 primary source): <https://datatracker.ietf.org/doc/html/rfc8693>
- **SPIFFE** (chapter 04 primary source): <https://spiffe.io/>
- **Envoy** (chapters 02 / 03 / 05 primary source): <https://www.envoyproxy.io/docs/envoy/latest/>
- **Istio** (chapters 02 / 03 / 05 primary source): <https://istio.io/latest/docs/>
- **Nygard — Release It!** (chapter 05 primary source): <https://pragprog.com/titles/mnee2/release-it-second-edition/>
- **JOB_REQUIREMENTS.md** (at the track root): the normalised list of citations the curriculum is written against.

## Notes on using these references

- The A2A spec is the fastest-moving surface in this module. The spec moved under Linux Foundation stewardship in 2025 and is iterating on both normative method names and AgentCard field placement. Before quoting a specific method name (`tasks/sendSubscribe`) or well-known path (`/.well-known/agent-card.json`) in production code, read the current spec at `https://a2a-protocol.org/` and pin the revision in your implementation's documentation.
- Service-mesh CRDs (Istio `VirtualService` / `DestinationRule`, Linkerd `ServiceProfile`) rename fields between minor versions. The chapters use example YAML that was correct for the pinned version in the resources list; your cluster's version is the authority on field names when you deploy.
- OAuth 2.0 Token Exchange support across commodity authorisation servers is uneven. Keycloak, Ory Hydra, Auth0, and Google's workload identity federation all advertise RFC 8693 support, but the claim-handling for `act` / `may_act` and the `actor_token` parameter varies. Before shipping an identity flow, test the specific authz server against the chapter-04 synthetic suite from exercise 02.
- The circuit-breaker library landscape (Hystrix, Resilience4j, Polly, pybreaker, opossum, gobreaker) is stable but not uniform. Picking one per language is a platform decision, not a per-team one; this module's recommendation is to lean on the mesh for mesh-level primitives and use a library only when the application needs behaviour the mesh cannot express (e.g., semaphore bulkheads inside a fan-out).
- Agent-topology frameworks (OpenAI Agents SDK, LangGraph, CrewAI, AutoGen) release breaking changes on a near-quarterly cadence. The platform's wire is A2A, which absorbs the churn; the paved-road SDK's framework adapters are where the breakages land.
- Mark `<!-- needs-research: ... -->` on any derived artefact where a citation on this page does not match the version you are shipping against. The flag convention is consistent with mod-403 / 404 / 405 / 406 / 407 / 408; it blocks auto-merge, and that is cheaper than silently guessing.
