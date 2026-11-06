# Resources — mod-403 (MCP Server Registry and Tool Gateway Plane)

All references on this page are primary / authoritative public sources at the time of writing (2026-11-06). Each resource is tagged with the chapter(s) it supports; several are cited across more than one. When reading, prefer the primary source over any secondary blog post — API-level and security-critical details drift fast in 2026, and summaries lag.

## Chapter 01 — The Tool-Gateway Plane and the MCP Contract

- **Model Context Protocol — specification**: <https://modelcontextprotocol.io/specification>
- **Model Context Protocol — documentation home** (concepts, tutorials, tool landscape): <https://modelcontextprotocol.io/>
- **MCP specification — GitHub repo** (versioned revisions, issue tracker, RFC-style changes): <https://github.com/modelcontextprotocol/specification>
- **MCP SDKs — Python, TypeScript, Kotlin, Java, C#, Swift** (<https://github.com/modelcontextprotocol>): the client/server libraries your gateway and its probes use.
- **JSON-RPC 2.0 specification** (the wire protocol MCP binds to): <https://www.jsonrpc.org/specification>
- **Server-Sent Events** (the streaming layer under Streamable HTTP): <https://html.spec.whatwg.org/multipage/server-sent-events.html>
- **CNCF — Platform Engineering Maturity Model** (background on the platform-as-a-product shape from mod-401): <https://tag-app-delivery.cncf.io/whitepapers/platform-eng-maturity-model/>
- **Anthropic — Building Effective Agents** (agent-authoring context the gateway hosts): <https://www.anthropic.com/research/building-effective-agents>
- **OpenAI — A Practical Guide to Building Agents**: <https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf>

## Chapter 02 — MCP Authorization at Engineering Depth

### MCP authorization spec

- **MCP authorization specification** (section of the main spec; verify against the current revision you deploy against): <https://modelcontextprotocol.io/specification>

### OAuth 2.1 and underlying RFCs

- **OAuth 2.1 draft** (the consolidating document; check the current draft number): <https://datatracker.ietf.org/doc/draft-ietf-oauth-v2-1/>
- **RFC 6749 — The OAuth 2.0 Authorization Framework**: <https://datatracker.ietf.org/doc/html/rfc6749>
- **RFC 6750 — OAuth 2.0 Bearer Token Usage**: <https://datatracker.ietf.org/doc/html/rfc6750>
- **RFC 7591 — OAuth 2.0 Dynamic Client Registration Protocol**: <https://datatracker.ietf.org/doc/html/rfc7591>
- **RFC 7592 — OAuth 2.0 Dynamic Client Registration Management Protocol** (operations on existing registrations): <https://datatracker.ietf.org/doc/html/rfc7592>
- **RFC 7636 — Proof Key for Code Exchange (PKCE)**: <https://datatracker.ietf.org/doc/html/rfc7636>
- **RFC 8414 — OAuth 2.0 Authorization Server Metadata**: <https://datatracker.ietf.org/doc/html/rfc8414>
- **RFC 8693 — OAuth 2.0 Token Exchange**: <https://datatracker.ietf.org/doc/html/rfc8693>
- **RFC 8707 — Resource Indicators for OAuth 2.0**: <https://datatracker.ietf.org/doc/html/rfc8707>
- **RFC 9068 — JSON Web Token (JWT) Profile for OAuth 2.0 Access Tokens** (useful where tokens are JWTs): <https://datatracker.ietf.org/doc/html/rfc9068>
- **RFC 9700 — Best Current Practice for OAuth 2.0 Security** (consolidates 2.0 security guidance): <https://datatracker.ietf.org/doc/html/rfc9700>
- **RFC 9728 — OAuth 2.0 Protected Resource Metadata**: <https://datatracker.ietf.org/doc/html/rfc9728>
- **OWASP — OAuth cheat sheet** (operational guidance): <https://cheatsheetseries.owasp.org/cheatsheets/OAuth2_Cheat_Sheet.html>

### Confused-deputy background

- **Norm Hardy — The Confused Deputy** (the 1988 original; still the clearest statement of the pattern): <https://capability.systems/concepts/confused-deputy.html>

## Chapter 03 — The MCP Server Registry

### MCP registry work in the wild

- **MCP Registry (community effort / reference registry)**: <https://github.com/modelcontextprotocol/registry> <!-- needs-research: confirm the current status of the official MCP registry project and its hosted index at your onboarding; this effort has been in motion through 2025-2026. -->

### Versioning + deprecation

- **Semantic Versioning 2.0.0**: <https://semver.org/>
- **RFC 8594 — The Sunset HTTP Header Field** (the deprecation-ladder idiom the chapter mirrors): <https://datatracker.ietf.org/doc/html/rfc8594>

### Governance-as-code

- **CNCF GitOps Principles**: <https://opengitops.dev/>
- **Open Policy Agent (OPA)** (policy-as-code engine the registry's admission gates may use, preview of mod-405): <https://www.openpolicyagent.org/docs/latest/>
- **Sigstore / Cosign** (image signing used in admission): <https://docs.sigstore.dev/>

### Schema primitives

- **JSON Schema 2020-12**: <https://json-schema.org/specification.html>
- **JSON Canonicalization Scheme (RFC 8785)** (for deterministic schema hashing): <https://datatracker.ietf.org/doc/html/rfc8785>

## Chapter 04 — Conformance, Health, and Failover Routing

### MCP Inspector and conformance tooling

- **MCP Inspector** (reference debugging client; the de facto shape of a protocol-conformance probe): <https://github.com/modelcontextprotocol/inspector>
- **Schemathesis** (property-based API testing, fits schema-conformance probes): <https://schemathesis.readthedocs.io/>
- **hypothesis-jsonschema** (Python property-based generation from JSON Schema): <https://github.com/python-jsonschema/hypothesis-jsonschema>
- **json-schema-faker** (JavaScript equivalent): <https://github.com/json-schema-faker/json-schema-faker>

### Resilience primitives

- **Resilience4j** (JVM circuit breaker / bulkhead / rate limiter): <https://resilience4j.readme.io/>
- **Polly** (.NET equivalent): <https://www.pollydocs.org/>
- **opossum** (Node.js circuit breaker): <https://nodeshift.dev/opossum/>
- **Google SRE Book — Handling Overload**: <https://sre.google/sre-book/handling-overload/>
- **Google SRE Workbook — Alerting on SLOs**: <https://sre.google/workbook/alerting-on-slos/>

### Observability hooks (preview of mod-408)

- **OpenTelemetry — Semantic Conventions for Generative AI**: <https://opentelemetry.io/docs/specs/semconv/gen-ai/>
- **OpenTelemetry — RPC conventions**: <https://opentelemetry.io/docs/specs/semconv/rpc/>

## Chapter 05 — Auth-Wrapping Proxy for Non-OAuth Servers

### Credential brokers / secret stores

- **HashiCorp Vault** (general secret management): <https://developer.hashicorp.com/vault/docs>
- **AWS Secrets Manager**: <https://docs.aws.amazon.com/secretsmanager/>
- **GCP Secret Manager**: <https://cloud.google.com/secret-manager/docs>
- **Azure Key Vault**: <https://learn.microsoft.com/en-us/azure/key-vault/general/>

### Workload identity federation

- **SPIFFE / SPIRE** (the open workload identity standard): <https://spiffe.io/docs/latest/>
- **AWS STS AssumeRoleWithWebIdentity**: <https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRoleWithWebIdentity.html>
- **GCP Workload Identity Federation**: <https://cloud.google.com/iam/docs/workload-identity-federation>
- **Azure Federated Identity Credentials**: <https://learn.microsoft.com/en-us/graph/api/resources/federatedidentitycredential>

### Signed-request schemes

- **AWS Signature Version 4**: <https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_aws-signing.html>
- **HTTP Message Signatures (RFC 9421)** (emerging standard): <https://datatracker.ietf.org/doc/html/rfc9421>

### Logging and redaction (preview of mod-411)

- **Microsoft Presidio** (PII detection/redaction toolkit): <https://microsoft.github.io/presidio/>

## Chapter 06 — Managed MCP Vendors and Build-vs-Buy

Vendor landscape changes quickly. Verify at your onboarding.

- **Arcade.dev** — managed OAuth and tool execution for agents: <https://www.arcade.dev/>
- **Composio** — large pre-authored tool catalog: <https://composio.dev/>
- **Toolhouse** — hosted tool execution and registry: <https://toolhouse.ai/>
- **LiteLLM / LiteLLM Proxy** (adjacent model-gateway + tool-adjacent primitives; cross-reference mod-406): <https://docs.litellm.ai/>
- **Portkey — AI gateway**: <https://portkey.ai/docs>
- **Model Context Protocol — awesome-mcp lists** (community-maintained; useful for scouting the ecosystem): <https://github.com/punkpeye/awesome-mcp-servers> <!-- needs-research: confirm this is the most actively maintained list at your reading time. -->
- **Google SRE Workbook — Managing Risk** (background for the portfolio framing): <https://sre.google/workbook/managing-risk/>

## Chapter 07 — Compliance Audit Evidence

### Regulatory and standards baselines

- **EU AI Act — Regulation (EU) 2024/1689**: <https://eur-lex.europa.eu/eli/reg/2024/1689/oj>
- **EU AI Act — Article 72 (Post-market monitoring by providers and post-market monitoring plan for high-risk AI systems)**: inside the regulation linked above; cross-reference the latest delegated acts. <!-- needs-research: confirm whether specific record-keeping delegated acts under Article 72 have entered into force at your audit cycle. -->
- **ISO/IEC 27001:2022** (information security management): <https://www.iso.org/standard/27001>
- **AICPA Trust Services Criteria (SOC 2)**: <https://www.aicpa-cima.com/resources/landing/system-and-organization-controls-soc-suite-of-services>
- **NIST AI Risk Management Framework**: <https://www.nist.gov/itl/ai-risk-management-framework>
- **GDPR — Regulation (EU) 2016/679** (right to erasure is Article 17): <https://eur-lex.europa.eu/eli/reg/2016/679/oj>

### Audit-record primitives

- **RFC 6920 — Naming Things with Hashes** (background on content-addressable hashes in audit chains): <https://datatracker.ietf.org/doc/html/rfc6920>
- **S2 Autonomous Object-Lock (AWS S3)** (immutable storage primitive): <https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock-overview.html>
- **GCP Bucket Lock**: <https://cloud.google.com/storage/docs/bucket-lock>
- **Azure Blob Immutable storage**: <https://learn.microsoft.com/en-us/azure/storage/blobs/immutable-storage-overview>

### Observability linkage (mod-408 preview)

- **OpenTelemetry — Semantic Conventions for Generative AI**: <https://opentelemetry.io/docs/specs/semconv/gen-ai/>
- **OpenTelemetry — Traces specification**: <https://opentelemetry.io/docs/specs/otel/trace/>

### PII detection

- **Microsoft Presidio**: <https://microsoft.github.io/presidio/>
- **AWS Comprehend — PII detection**: <https://docs.aws.amazon.com/comprehend/latest/dg/pii.html>
- **GCP Data Loss Prevention**: <https://cloud.google.com/sensitive-data-protection/docs>

## Secondary / cross-cutting

- **OWASP Top 10 for LLM Applications** (the baseline threat model this plane defends against; previewed in mod-411): <https://genai.owasp.org/llm-top-10/>
- **OWASP Agentic AI — Threats and Mitigations** (the agent-specific extension): <https://genai.owasp.org/initiatives/#agentic-security-initiative>
- **A2A (Agent-to-Agent) protocol** (preview for mod-409; relevant because A2A messages often ride through the tool-gateway plane): <https://a2aprotocol.ai/> <!-- needs-research: confirm the canonical spec URL for A2A at your reading time; this has been moving between hosts. -->
- **IETF — HTTP Semantics (RFC 9110)** (for the HTTP-layer behaviours Streamable HTTP relies on): <https://datatracker.ietf.org/doc/html/rfc9110>

## Notes on using these references

- Everything on this page is a **primary** or **authoritative-public** source. If a vendor doc reads like marketing, go to the vendor's engineering reference; if a blog post summarises an RFC, go to the RFC.
- The MCP spec, the MCP authorization spec, the MCP registry work, and the vendor product scopes are all in a particularly fast-moving phase. Before quoting a specific wire-level detail or vendor capability in an exercise artefact or architecture memo, verify against the current doc and mark `<!-- needs-research: ... -->` if your spec reference does not match the version on this page.
- `JOB_REQUIREMENTS.md` at the track root and `.aicg/job-requirements.json` carry the normalised, machine-readable form of the track's citations.
