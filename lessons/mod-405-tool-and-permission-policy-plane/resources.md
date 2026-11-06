# Resources — mod-405 (Tool Registry and Permission Policy Plane)

All references on this page are primary / authoritative public sources at the time of writing (2026-11-06). Each resource is tagged with the chapter(s) it supports. Prefer the primary source over any blog summary — policy-engine, identity-framework, and OAuth-spec details drift fast in 2026 and secondary sources lag.

## Chapter 01 — The Policy Plane and the Authorization Triad

### Authorization-model background
- **Dennis & Van Horn, "Programming Semantics for Multiprogrammed Computations"** (foundational capability-model paper, 1966): <https://dl.acm.org/doi/10.1145/365230.365252>
- **NIST — Guide to Attribute Based Access Control (ABAC) Definition and Considerations (SP 800-162)**: <https://csrc.nist.gov/pubs/sp/800/162/upd1/final>
- **NIST — Role-Based Access Controls** (foundational RBAC reference): <https://csrc.nist.gov/projects/role-based-access-control>

### Threat-model artefacts
- **OWASP Top 10 for LLM Applications**: <https://genai.owasp.org/llm-top-10/>
- **OWASP Agentic AI — Threats and Mitigations (Agentic Security Initiative)**: <https://genai.owasp.org/initiatives/#agentic-security-initiative>
- **MITRE ATLAS (Adversarial Threat Landscape for AI Systems)**: <https://atlas.mitre.org/>
- **NIST — AI Risk Management Framework** (incl. AI RMF Generative AI Profile): <https://www.nist.gov/itl/ai-risk-management-framework>

## Chapter 02 — OPA and Rego

### OPA core
- **Open Policy Agent — home**: <https://www.openpolicyagent.org/>
- **OPA — introduction**: <https://www.openpolicyagent.org/docs/latest/>
- **Rego — policy language reference**: <https://www.openpolicyagent.org/docs/latest/policy-language/>
- **Rego — policy reference (built-ins)**: <https://www.openpolicyagent.org/docs/latest/policy-reference/>
- **OPA — policy testing**: <https://www.openpolicyagent.org/docs/latest/policy-testing/>
- **OPA — decision logs**: <https://www.openpolicyagent.org/docs/latest/decision-logs/>
- **OPA — bundles**: <https://www.openpolicyagent.org/docs/latest/management-bundles/>
- **OPA — configuration (incl. bundles, decision_logs, status plugins)**: <https://www.openpolicyagent.org/docs/latest/configuration/>
- **OPA — profiling and benchmarking**: <https://www.openpolicyagent.org/docs/latest/profiling/>
- **OPA — WASM support**: <https://www.openpolicyagent.org/docs/latest/wasm/>

### OPA ecosystem
- **Conftest** (policy-aware testing for structured config): <https://www.conftest.dev/>
- **Styra DAS** (managed OPA control plane): <https://www.styra.com/styra-das/>
- **Rego Playground**: <https://play.openpolicyagent.org/>
- **go-spiffe** (used alongside OPA for workload identity): <https://github.com/spiffe/go-spiffe>

## Chapter 03 — Cedar and ABAC

### Cedar core
- **Cedar — home**: <https://www.cedarpolicy.com/>
- **Cedar — docs**: <https://docs.cedarpolicy.com/>
- **Cedar — syntax (policies)**: <https://docs.cedarpolicy.com/policies/syntax-policy.html>
- **Cedar — schema reference**: <https://docs.cedarpolicy.com/schema/schema.html>
- **Cedar — templates**: <https://docs.cedarpolicy.com/policies/templates.html>
- **Cedar — validation**: <https://docs.cedarpolicy.com/policies/validation.html>

### Cedar project repositories
- **cedar-policy/cedar** (Rust implementation): <https://github.com/cedar-policy/cedar>
- **cedar-policy/cedar-java**: <https://github.com/cedar-policy/cedar-java>
- **cedar-policy/cedar-go**: <https://github.com/cedar-policy/cedar-go>
- **@cedar-policy/cedar-wasm** (TypeScript / WASM binding): <https://www.npmjs.com/package/@cedar-policy/cedar-wasm>
- **cedar-policy/cedar-analysis** (SMT-backed differential analyser): <https://github.com/cedar-policy/cedar-analysis>

### Cedar research and background
- **Amazon Science — "Cedar: A new language for expressive, fast, safe, and analyzable authorization"**: <https://www.amazon.science/publications/cedar-a-new-language-for-expressive-fast-safe-and-analyzable-authorization>
- **Amazon Verified Permissions** (hosted Cedar): <https://aws.amazon.com/verified-permissions/>

## Chapter 04 — SPIFFE/SPIRE, OAuth 2.1, and Credential Brokering

### SPIFFE and SPIRE
- **SPIFFE — home**: <https://spiffe.io/>
- **SPIFFE — documentation**: <https://spiffe.io/docs/latest/>
- **SPIFFE — specifications**: <https://github.com/spiffe/spiffe/tree/main/standards>
- **SPIFFE — Workload API spec**: <https://github.com/spiffe/spiffe/blob/main/standards/SPIFFE_Workload_API.md>
- **SPIFFE — Federation spec**: <https://github.com/spiffe/spiffe/blob/main/standards/SPIFFE_Federation.md>
- **SPIRE — repository**: <https://github.com/spiffe/spire>
- **SPIRE — Helm Charts Hardened**: <https://github.com/spiffe/helm-charts-hardened>
- **SPIFFE CSI Driver**: <https://github.com/spiffe/spiffe-csi>
- **go-spiffe**: <https://github.com/spiffe/go-spiffe>
- **py-spiffe**: <https://github.com/HewlettPackard/py-spiffe>

### OAuth 2.1 and relevant RFCs (cross-reference mod-403 chapter 02)
- **OAuth 2.1 Authorization Framework (IETF draft)**: <https://datatracker.ietf.org/doc/draft-ietf-oauth-v2-1/> <!-- needs-research: confirm the current OAuth 2.1 draft or RFC status at reading time; it has been moving through the IETF process. -->
- **RFC 6749** (OAuth 2.0 core): <https://datatracker.ietf.org/doc/html/rfc6749>
- **RFC 7009** (Token Revocation): <https://datatracker.ietf.org/doc/html/rfc7009>
- **RFC 7591** (Dynamic Client Registration): <https://datatracker.ietf.org/doc/html/rfc7591>
- **RFC 7636** (PKCE): <https://datatracker.ietf.org/doc/html/rfc7636>
- **RFC 8414** (Authorization Server Metadata): <https://datatracker.ietf.org/doc/html/rfc8414>
- **RFC 8693** (Token Exchange): <https://datatracker.ietf.org/doc/html/rfc8693>
- **RFC 8707** (Resource Indicators): <https://datatracker.ietf.org/doc/html/rfc8707>
- **RFC 9068** (JWT Profile for OAuth 2.0 Access Tokens): <https://datatracker.ietf.org/doc/html/rfc9068>
- **RFC 9700** (Best Current Practice for OAuth 2.0 Security): <https://datatracker.ietf.org/doc/html/rfc9700>
- **RFC 9728** (OAuth 2.0 Protected Resource Metadata): <https://datatracker.ietf.org/doc/html/rfc9728>

### Vault and credential brokering
- **HashiCorp Vault — documentation**: <https://developer.hashicorp.com/vault/docs>
- **Vault — database secrets engines**: <https://developer.hashicorp.com/vault/docs/secrets/databases>
- **Vault — JWT auth method**: <https://developer.hashicorp.com/vault/docs/auth/jwt>
- **Vault — transit secrets engine**: <https://developer.hashicorp.com/vault/docs/secrets/transit>
- **Vault — PKI secrets engine**: <https://developer.hashicorp.com/vault/docs/secrets/pki>

### Cloud provider workload-identity primitives
- **AWS — Instance Identity Document**: <https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instance-identity-documents.html>
- **GCP — verifying instance identity**: <https://cloud.google.com/compute/docs/instances/verifying-instance-identity>
- **Azure — managed identities for Azure resources**: <https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/overview>

## Chapter 05 — Tool Lifecycle and Approval Workflows

### Platform-engineering lifecycle background
- **CNCF — Platform Engineering Maturity Model**: <https://tag-app-delivery.cncf.io/whitepapers/platform-eng-maturity-model/>
- **CNCF — Platforms White Paper**: <https://tag-app-delivery.cncf.io/whitepapers/platforms/>
- **Google SRE Book — Release Engineering**: <https://sre.google/sre-book/release-engineering/>
- **Google SRE Workbook — Canarying Releases**: <https://sre.google/workbook/canarying-releases/>

### Supply-chain and admission
- **SLSA (Supply-chain Levels for Software Artifacts)**: <https://slsa.dev/>
- **in-toto**: <https://in-toto.io/>
- **Sigstore — Cosign**: <https://docs.sigstore.dev/>
- **OCI Image Signing (cosign in registry)**: <https://docs.sigstore.dev/cosign/signing/signing_with_containers/>

### Model Context Protocol (cross-ref mod-403)
- **Model Context Protocol — specification**: <https://modelcontextprotocol.io/specification>
- **MCP — authorization spec**: <https://modelcontextprotocol.io/specification/basic/authorization>

## Chapter 06 — Policy Regression Tests and the Abuse Corpus

### Testing, differential analysis, and CI
- **OPA — policy testing**: <https://www.openpolicyagent.org/docs/latest/policy-testing/>
- **Conftest**: <https://www.conftest.dev/>
- **cedar-policy/cedar-analysis**: <https://github.com/cedar-policy/cedar-analysis>
- **Open Policy Containers (bundle distribution)**: <https://github.com/open-policy-agent/opa/blob/main/docs/content/management-bundles.md>

### Adversarial corpora and red-teaming
- **MITRE ATLAS (threat ontology + case studies)**: <https://atlas.mitre.org/>
- **OWASP Agentic AI — Threats and Mitigations**: <https://genai.owasp.org/initiatives/#agentic-security-initiative>
- **Anthropic — "Red Teaming Language Models to Reduce Harms"**: <https://arxiv.org/abs/2209.07858>
- **Google DeepMind — Frontier Safety Framework**: <https://deepmind.google/discover/blog/updating-the-frontier-safety-framework/> <!-- needs-research: confirm the current version of the Frontier Safety Framework at reading time; this has been revised repeatedly through 2025-2026. -->
- **Anthropic — Responsible Scaling Policy (public pages)**: <https://www.anthropic.com/news/anthropics-responsible-scaling-policy> <!-- needs-research: confirm latest RSP revision date. -->

### Observability and decision-log linkage (cross-ref mod-408)
- **OpenTelemetry — Semantic Conventions for Generative AI**: <https://opentelemetry.io/docs/specs/semconv/gen-ai/>
- **OpenTelemetry — Traces specification**: <https://opentelemetry.io/docs/specs/otel/trace/>

## Chapter 07 — The Enforcement-Plane Contract

### Harm models and risk frameworks
- **NIST — AI Risk Management Framework (Generative AI Profile)**: <https://www.nist.gov/itl/ai-risk-management-framework>
- **NIST — AI Risk Management Framework Playbook**: <https://airc.nist.gov/AI_RMF_Knowledge_Base/Playbook>
- **ISO/IEC 23894:2023 — AI risk management**: <https://www.iso.org/standard/77304.html>
- **ISO/IEC 42001:2023 — AI management systems**: <https://www.iso.org/standard/81230.html>

### Regulatory baselines (cross-ref mod-404 chapter 06)
- **EU AI Act — Regulation (EU) 2024/1689**: <https://eur-lex.europa.eu/eli/reg/2024/1689/oj>
- **EU GDPR — Regulation (EU) 2016/679**: <https://eur-lex.europa.eu/eli/reg/2016/679/oj>
- **US Executive Order on AI** (current revision): <https://www.whitehouse.gov/briefing-room/presidential-actions/> <!-- needs-research: confirm the current EO (or successor) on AI at reading time; executive orders in this space have been issued and modified multiple times through 2024-2026. -->
- **UK AI Safety Institute**: <https://www.aisi.gov.uk/>

### Operating cadence (cross-ref mod-410)
- **Google SRE Workbook — Incident Response**: <https://sre.google/workbook/incident-response/>
- **Google SRE Workbook — Postmortem Culture**: <https://sre.google/workbook/postmortem-culture/>

## Cross-cutting references

- **Model Context Protocol — specification** (cross-ref mod-403): <https://modelcontextprotocol.io/specification>
- **Anthropic — Building Effective Agents**: <https://www.anthropic.com/research/building-effective-agents>
- **OpenAI — A Practical Guide to Building Agents**: <https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf>
- **JOB_REQUIREMENTS.md** (at the track root): the normalised list of citations the curriculum is written against.

## Notes on using these references

- Every entry on this page is a **primary** or **authoritative-public** source. If the source has a version tag — kernel release, project release, RFC revision — read the version that matches the one you deploy against. OAuth 2.1 is still a draft at some reading points; Cedar's extensions are in active evolution; SPIRE's Helm charts gain security hardening each release.
- The OAuth 2.1 draft, the Cedar language, and the SPIRE API surfaces all move. Before quoting an exact field name or CLI flag, verify against the version you ship against.
- Regulatory artefacts (EU AI Act, US executive orders, state privacy laws, sectoral rules) are refined by delegated acts and amendments. Before quoting a specific record-keeping horizon or a specific requirement in a compliance memo, verify against the latest text and against counsel at your organisation.
- Mark `<!-- needs-research: ... -->` on any derived artefact where a citation on this page does not match the version you are shipping against. The flag blocks auto-merge; it is cheaper than silently guessing.
