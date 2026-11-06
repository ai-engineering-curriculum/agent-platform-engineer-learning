# Resources — mod-413 (Platform Leadership and Cross-Team Operating Cadence)

All references on this page are primary / authoritative public sources at the time of writing (2026-11-06). Each resource is tagged with the chapter(s) it supports. Prefer the primary source over any blog summary — the CNCF whitepaper updates, the DORA State of DevOps report re-revs yearly, the agent-service vendor documentation changes quarterly, and the MCP registry landscape is still crystallising.

## Chapter 01 — Platform Roadmap Authored Against Stream-Aligned Teams

### Operating-model and platform-engineering frames (cross-ref mod-401)

- **Skelton & Pais — Team Topologies**: <https://teamtopologies.com/>
- **Team Topologies — Platform Team pattern page**: <https://teamtopologies.com/key-concepts-content/what-is-a-platform-team>
- **Team Topologies — Platform as a Product**: <https://teamtopologies.com/key-concepts>
- **CNCF TAG App Delivery — Platforms Whitepaper**: <https://tag-app-delivery.cncf.io/whitepapers/platforms/>
- **CNCF TAG App Delivery — Platform Engineering Maturity Model**: <https://tag-app-delivery.cncf.io/whitepapers/platform-eng-maturity-model/>
- **CNCF — Platforms Working Group**: <https://github.com/cncf/tag-app-delivery/tree/main/platforms-wg>
- **CNCF — Case studies of platform teams**: <https://www.cncf.io/case-studies/>
- **Internal Developer Platform community**: <https://internaldeveloperplatform.org/>
- **Humanitec — State of Internal Developer Platforms**: <https://humanitec.com/whitepapers>
- **Puppet — State of Platform Engineering**: <https://www.puppet.com/resources/state-of-platform-engineering>

### Roadmap and product-management shape

- **Marty Cagan — Inspired / Empowered (reference books on product-team operating models)**: <https://www.svpg.com/books/>
- **Melissa Perri — Escaping the Build Trap (reference book on roadmap vs. backlog)**: <https://melissaperri.com/book>
- **Jurgen Appelo — "Now / Next / Later" roadmap pattern (ProdPad origin)**: <https://www.prodpad.com/blog/now-next-later-roadmap/>

### Governance patterns

- **Spotify engineering culture (model videos; context for platform-as-product thinking)**: <https://engineering.atspotify.com/>
- **Spotify — Backstage announcement and principles**: <https://backstage.io/blog>

## Chapter 02 — The Build-vs-Buy Portfolio

### Model Context Protocol (MCP) — spec and ecosystem

- **Model Context Protocol — specification**: <https://modelcontextprotocol.io/>
- **Model Context Protocol — GitHub organisation**: <https://github.com/modelcontextprotocol>
- **MCP Registry (official reference; in progress)**: <https://github.com/modelcontextprotocol/registry>
- **MCP Python SDK**: <https://github.com/modelcontextprotocol/python-sdk>
- **MCP TypeScript SDK**: <https://github.com/modelcontextprotocol/typescript-sdk>

### MCP registry vendor candidates

- **Smithery — MCP server registry**: <https://smithery.ai/>
- **mcp.so — community registry**: <https://mcp.so/>
- **PulseMCP — registry and curation**: <https://www.pulsemcp.com/>
- **Composio — MCP-shaped tool catalogue**: <https://composio.dev/>

### Sandboxed code execution — vendor candidates (cross-ref mod-404)

- **E2B — managed sandbox and open-source core**: <https://e2b.dev/>
- **E2B on GitHub**: <https://github.com/e2b-dev/E2B>
- **Daytona — managed CDE / agent sandbox**: <https://daytona.io/>
- **Daytona on GitHub**: <https://github.com/daytonaio/daytona>
- **Modal — serverless compute with Python SDK**: <https://modal.com/>
- **CodeSandbox CDE**: <https://codesandbox.io/>
- **Gitpod**: <https://www.gitpod.io/>
- **Coder**: <https://coder.com/>
- **gVisor (self-host substrate)**: <https://gvisor.dev/>
- **Firecracker (self-host substrate)**: <https://firecracker-microvm.github.io/>
- **Kata Containers (self-host substrate)**: <https://katacontainers.io/>

### Agent runtime / orchestration — vendor candidates (cross-ref mod-402)

- **LangGraph Platform — LangChain's managed deployment**: <https://www.langchain.com/langgraph-platform>
- **LangGraph documentation**: <https://langchain-ai.github.io/langgraph/>
- **LangGraph on GitHub**: <https://github.com/langchain-ai/langgraph>
- **Vertex AI Agent Engine — Google Cloud's managed agent runtime**: <https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/overview>
- **Vertex AI Agent Engine — deploying agents**: <https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/deploy>
- **Azure AI Foundry — Agent Service**: <https://learn.microsoft.com/en-us/azure/ai-foundry/agents/overview>
- **Azure AI Foundry — Agent Service concepts**: <https://learn.microsoft.com/en-us/azure/ai-foundry/agents/concepts>
- **AWS Bedrock AgentCore**: <https://aws.amazon.com/bedrock/agentcore/>
- **AWS Bedrock Agents documentation**: <https://docs.aws.amazon.com/bedrock/latest/userguide/agents.html>
- **Temporal (self-host substrate)**: <https://docs.temporal.io/>
- **Dapr Workflow (self-host substrate)**: <https://docs.dapr.io/developing-applications/building-blocks/workflow/>
- **Restate (durable execution)**: <https://restate.dev/>
- **Inngest (durable execution)**: <https://www.inngest.com/>

### Build-vs-buy decision frameworks (industry priors)

- **Gartner — Build-vs-Buy decision references (behind paywall; cite with care)**: <https://www.gartner.com/>
- **Wardley Mapping — mapping components by evolution (public)**: <https://learnwardleymapping.com/>
- **ThoughtWorks Technology Radar — reference for vendor adoption posture**: <https://www.thoughtworks.com/radar>

### Open standards that make hybrid cheap

- **Model Context Protocol — specification**: <https://modelcontextprotocol.io/>
- **OpenTelemetry — Semantic Conventions for Generative AI**: <https://opentelemetry.io/docs/specs/semconv/gen-ai/>
- **Open Container Initiative (OCI) — spec**: <https://opencontainers.org/>
- **OpenAI-compatible completions (de-facto standard on vendor gateways)**: <https://platform.openai.com/docs/api-reference/chat>

## Chapter 03 — Cross-Team Incident Command

### Incident Command System (ICS) — source materials

- **FEMA — IS-100.C Introduction to the Incident Command System**: <https://training.fema.gov/is/courseoverview.aspx?code=is-100.c>
- **FEMA — ICS Resource Center**: <https://training.fema.gov/emiweb/is/icsresource/>
- **PagerDuty — Incident Response documentation**: <https://response.pagerduty.com/>
- **PagerDuty — Incident commander role**: <https://response.pagerduty.com/training/incident_commander/>

### Google SRE on incident management and post-mortems

- **Google SRE Book — Managing Incidents chapter**: <https://sre.google/sre-book/managing-incidents/>
- **Google SRE Book — Postmortem Culture: Learning from Failure**: <https://sre.google/sre-book/postmortem-culture/>
- **Google SRE Workbook — Incident Response chapter**: <https://sre.google/workbook/incident-response/>
- **Google SRE Workbook — Blameless postmortems**: <https://sre.google/workbook/postmortem-culture/>

### Security & safety incident references

- **NIST SP 800-61 — Computer Security Incident Handling Guide**: <https://csrc.nist.gov/publications/detail/sp/800-61/rev-2/final>
- **SANS Incident Handler's Handbook**: <https://www.sans.org/white-papers/33901/>
- **MITRE ATT&CK — framework for adversary behaviour**: <https://attack.mitre.org/>
- **OWASP Top 10 for LLM Applications**: <https://owasp.org/www-project-top-10-for-large-language-model-applications/>
- **OWASP Agentic AI Threats and Mitigations**: <https://genai.owasp.org/llmrisk2023-24/>

### Related operational references

- **Atlassian — Incident response documentation**: <https://www.atlassian.com/incident-management>
- **The Morning Paper (Adrian Colyer) on retrospectives**: <https://blog.acolyer.org/>  <!-- needs-research: specific post URLs may rot; the blog's archive is the stable reference. -->
- **Learning from Incidents community**: <https://www.learningfromincidents.io/>

## Chapter 04 — Measuring Platform Success — DORA, SPACE, Adoption, and Support MTTR

### DORA

- **Google / DORA — State of DevOps Reports (annual)**: <https://cloud.google.com/devops/state-of-devops>
- **DORA — community site**: <https://dora.dev/>
- **DORA — Four Keys quickstart**: <https://dora.dev/quickcheck/>
- **Forsgren, Humble, Kim — Accelerate (book, foundational DORA research)**: <https://itrevolution.com/product/accelerate/>
- **DORA — core metrics page**: <https://dora.dev/guides/dora-metrics-four-keys/>

### SPACE

- **Forsgren, Storey, Maddila, Zimmermann, Houck, Butler — The SPACE of Developer Productivity (ACM Queue, 2021)**: <https://queue.acm.org/detail.cfm?id=3454124>
- **GitHub — SPACE at GitHub blog post**: <https://github.blog/2021-03-26-why-developer-experience-is-the-future-of-building-software/>

### DevEx framework

- **Noda, Forsgren, Alexander, Storey — DevEx: What Actually Drives Productivity (ACM Queue, 2023)**: <https://queue.acm.org/detail.cfm?id=3595878>
- **Dr. Nicole Forsgren — DevEx and SPACE integration (public talks on YouTube — verify URLs before citing)**

### Platform-adoption KPIs (cross-ref mod-412)

- **CNCF TAG App Delivery — Platform Engineering Maturity Model**: <https://tag-app-delivery.cncf.io/whitepapers/platform-eng-maturity-model/>
- **CNCF TAG App Delivery — Platforms Whitepaper**: <https://tag-app-delivery.cncf.io/whitepapers/platforms/>
- **mod-412 chapter 06 (internal cross-reference)**

### Support ITSM references

- **ITIL 4 — Service Management (official)**: <https://www.axelos.com/certifications/itil-service-management>  <!-- needs-research: ITIL foundations are stable; specific practice guides rev every few years. -->
- **Atlassian — IT Service Management guides**: <https://www.atlassian.com/itsm>

### Dashboard and instrumentation tools

- **Grafana**: <https://grafana.com/docs/grafana/latest/>
- **Metabase**: <https://www.metabase.com/docs/>
- **Streamlit**: <https://docs.streamlit.io/>
- **DuckDB**: <https://duckdb.org/docs/>
- **OpenTelemetry Collector**: <https://opentelemetry.io/docs/collector/>
- **Jaeger**: <https://www.jaegertracing.io/docs/latest/>
- **Grafana Tempo**: <https://grafana.com/oss/tempo/>

## Chapter 05 — Handoffs and Governance

### Peer and up-ladder tracks (internal cross-references)

- [`agentic-ai-engineer`](../../../agentic-ai-engineer-learning/) — level 30; primary stream-aligned puller.
- [`senior-agentic-ai-engineer`](../../../senior-agentic-ai-engineer-learning/) — level 40; subsystem-lead consumer.
- [`agentic-safety-engineer`](../../../agentic-safety-engineer-learning/) — level 40; safety-plane peer.
- [`ai-risk-engineer`](../../../ai-risk-engineer-learning/) — level 25; risk / compliance peer.
- [`ai-infra-sre`](../../../ai-infra-sre-learning/) — level 30; SRE-plane peer (cross-ref mod-410).
- [`ai-infra-architect`](../../../ai-infra-architect-learning/) — level 45; cross-stack infrastructure handoff target.
- [`agentic-systems-architect`](../../../agentic-systems-architect-learning/) — level 48; cross-product agent-architecture handoff target.

### Engineering management and leadership references

- **Camille Fournier — The Manager's Path**: <https://www.oreilly.com/library/view/the-managers-path/9781491973882/>
- **Will Larson — Staff Engineer**: <https://staffeng.com/book>
- **Will Larson — An Elegant Puzzle (engineering management systems)**: <https://lethain.com/elegant-puzzle/>
- **Rands in Repose — Managing humans (long-running blog)**: <https://randsinrepose.com/>
- **Lara Hogan — Resilient Management**: <https://larahogan.me/resilient-management/>

### Architecture reference-material shape

- **ThoughtWorks Technology Radar**: <https://www.thoughtworks.com/radar>
- **arc42 — Architecture documentation template**: <https://arc42.org/>
- **C4 model — Architecture diagramming notation**: <https://c4model.com/>
- **Simon Brown — Software Architecture for Developers**: <https://leanpub.com/b/software-architecture>

### Governance log and documentation

- **Backstage TechDocs**: <https://backstage.io/docs/features/techdocs/techdocs-overview/>
- **Docs as Code — pattern reference**: <https://docsascode.org/>
- **MkDocs Material**: <https://squidfunk.github.io/mkdocs-material/>

## Exercise cross-references

- **Exercise 01** — platform roadmap authored against stream teams. Primary references: chapter 01 entry shape; CNCF maturity dimensions; `mod-401` chapter 02 and chapter 04 for not-platformising; `mod-412` chapter 06 KPIs the roadmap commits to.
- **Exercise 02** — build-vs-buy portfolio rubric. Primary references: chapter 02 rubric; MCP registry vendors (Smithery / mcp.so / PulseMCP / Composio); sandbox vendors (E2B / Daytona / Modal); runtime vendors (LangGraph Platform / Vertex Agent Engine / Azure AI Foundry Agent Service / Bedrock AgentCore); open-standard references that make hybrid cheap.
- **Exercise 03** — cross-team incident command playbook. Primary references: chapter 03 role structure; FEMA ICS IS-100; Google SRE incident-management and post-mortem culture; PagerDuty incident response; OWASP LLM Top 10 for the security-dimension scenario; `mod-410` chapter 05 for the single-team playbook library; `mod-410` chapter 06 for the fixture-promotion pipeline.
- **Exercise 04** — platform adoption scorecard. Primary references: chapter 04 four instruments; `mod-412` chapter 06 KPI formulas; DORA metrics; SPACE paper; DevEx framework; the chapter-04 scorecard template.

## Note on verification

This module references upstream standards (CNCF Platform Engineering Maturity Model, DORA band thresholds, OpenTelemetry-GenAI semantic conventions), live vendor documentation (MCP registry vendors, sandbox vendors, agent-runtime vendors), and published incident-management frameworks (FEMA ICS, Google SRE). All of these move. Before quoting a specific CNCF maturity-level indicator, a specific DORA band threshold, a specific vendor feature, a specific pricing tier, or a specific ICS section number in the text, in an exercise output, or in a scorecard, verify it against the current version of the source. The chapters use `<!-- needs-research: ... -->` on claims that cannot be verified at the time of authoring; the flag blocks auto-merge and is strictly preferable to silent drift.
