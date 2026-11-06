# The Build-vs-Buy Portfolio for Agent-Platform Components

Every plane in this track has a public commercial alternative. The MCP registry you are standing up is the thing [Smithery](https://smithery.ai/), [mcp.so](https://mcp.so/), [PulseMCP](https://www.pulsemcp.com/), and vendor-specific registries publish. The sandbox from [`mod-404`](../mod-404-sandboxed-code-execution-plane) is the thing [E2B](https://e2b.dev/), [Daytona](https://daytona.io/), and [Modal](https://modal.com/) sell. The runtime from [`mod-402`](../mod-402-agent-runtime-service-and-durable-execution) is the thing [LangGraph Platform](https://www.langchain.com/langgraph-platform), [Vertex AI Agent Engine](https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/overview), [Azure AI Foundry Agent Service](https://learn.microsoft.com/en-us/azure/ai-foundry/agents/overview), and [AWS Bedrock AgentCore](https://aws.amazon.com/bedrock/agentcore/) offer as a managed service. The question the roadmap keeps asking is not *which planes to build* — the track has answered that — but *for each plane, build-vs-buy on what cadence, for how long, and against what trigger*.

This chapter is a decision discipline. It pins the six evaluation axes, the rubric that scores them, three worked decisions against real vendors as of late 2026, and the decision-record shape that goes into the governance log. It does *not* recommend a vendor; vendor landscapes move faster than this track can re-rev. It gives you the discipline that lets you re-run the decision when the landscape shifts.

## Why build-vs-buy is a platform decision, not an engineering decision

Three claims that make build-vs-buy sit with you rather than with any individual plane owner.

1. **The decision is portfolio-level.** Any one plane could be built or bought in isolation; the cost of integration is dominated by *which planes you integrate across*. A managed runtime that does not speak to your policy plane costs more than a self-host runtime that composes cleanly with everything else. The portfolio view is what catches this.
2. **The decision reverses.** Build-today-buy-tomorrow is cheap (migrate off a running system to a vendor that reproduces its API). Buy-today-build-tomorrow is expensive (a managed service that locked you into its data model, its auth model, its billing model). Every decision carries an explicit *re-decision horizon*; the governance cadence re-runs it on that clock.
3. **The decision is political as well as technical.** A tenant that trusts a vendor already (clinical-trials copilot + Vertex AI, say) is cheaper to meet on a managed substrate. A tenant with strict data-residency obligations has the opposite constraint. The rubric must carry these signals or the roadmap conversation becomes "engineering wants to build, business wants to buy" by default.

## The six evaluation axes

The axes that matter for an agent-platform component. Each axis gets a score from 1 (buy strongly favoured) to 5 (build strongly favoured). The scores are *not* summed; they are read together in the decision record.

### Axis 1 — Strategic fit

Does this component sit on the critical path of the platform's identity? The runtime, the policy plane, and the safety plane are typically strategic (what you stand for); the MCP registry's search algorithm, the sandbox substrate, and the trace-viewer UI typically are not.

- **1** — Pure commodity. Any vendor can run this; nothing about your platform's identity is expressed here.
- **3** — Partially strategic. You need control of some edges (the policy decisions, the on-call model) but the substrate itself is replaceable.
- **5** — Strategic. The component expresses an opinion that the platform exists to hold.

### Axis 2 — Regulatory / data-residency posture

Where does the data live, who controls encryption keys, which audit regimes apply? Clinical, financial-services, government, and EU-resident workloads have hard constraints. A vendor that cannot host in region X, run on BYOK, or produce a SOC 2 Type II + specific HIPAA/PCI attestation is disqualified for the tenants that require them.

- **1** — Vendor meets all regulatory requirements out of the box; BYOK supported; attestations current.
- **3** — Vendor meets most; some compensating controls required; data-residency acceptable for a subset of tenants.
- **5** — Vendor cannot meet; residency / BYOK / attestation gaps are structural.

### Axis 3 — Integration tax

The cost of wiring the component into the eleven other planes (OTel conventions, policy plane, eval plane, SRE plane, safety plane). A vendor that does not emit [OpenTelemetry Semantic Conventions for Generative AI](https://opentelemetry.io/docs/specs/semconv/gen-ai/) today costs you a translation shim forever.

- **1** — Vendor speaks every seam natively; drop-in on the paved road.
- **3** — Shim required for 2–3 seams; ongoing maintenance.
- **5** — Shim required for most seams; the vendor is effectively a parallel platform.

### Axis 4 — Total cost (operation + licence + exit)

Three components:

- **Operation cost** — engineer-weeks per quarter to keep the self-hosted component running.
- **Licence cost** — the vendor's priced plan at your scale (including the Enterprise tier, which is usually the only tier that meets axis 2 and axis 6).
- **Exit cost** — engineer-weeks to migrate off, if the decision reverses in year N.

The score reflects *total* cost at your scale, not the headline price.

- **1** — Buy decisively cheaper; the self-host operation cost dominates licence.
- **3** — Comparable; the decision is driven by other axes.
- **5** — Build decisively cheaper at scale; licence costs exceed self-host operation.

### Axis 5 — Reversibility

How hard is it to swap decisions in year 2 or year 3?

- **1** — Vendor speaks an open-standard API; swap is a config change. (MCP registry front-end to a SaaS is near this.)
- **3** — Swap requires a migration playbook and a quarter of effort.
- **5** — Swap requires rebuilding the data model from scratch; lock-in is structural.

A platform team's job is to *keep the decision reversible as long as possible*. Open-standard interfaces (MCP for tool catalogs, OpenTelemetry for traces, OCI for container images, OpenAI-compatible completion endpoints for gateways) buy reversibility cheap.

### Axis 6 — Team fit

Does the platform team have the specific skills to run the self-hosted version at the required reliability? A self-hosted Firecracker sandbox is not a general infrastructure capability; it is a specialist capability. If the specialist is a hire-ahead-of-the-work, "build" has a hidden hiring cost.

- **1** — No team skill to build; hiring the specialist is a 6-month effort.
- **3** — Partial skill; could build with training and one senior hire.
- **5** — Team is deep on this; self-host operates below "one engineer-week per quarter" overhead.

## Reading the rubric

The rubric is *not* a decision calculator. It is a decision *conversation*. Two scored components that total 18 might imply very different decisions.

- Six 3s — nothing obvious; the decision is political, driven by tenant preference and cadence.
- Three 5s on axes 1, 4, 6 and three 1s on axes 2, 3, 5 — build is strongly favoured on strategic / cost / skill axes but the component is regulatory-easy and integration-cheap; still probably build, but the cost of being wrong is low.
- A single 5 on axis 2 and 1s elsewhere — buy is strongly favoured everywhere but the regulatory axis disqualifies the vendor for a tenant; the decision is *buy for most tenants, build-or-exempt for the regulated tenant*. This is one of the most common real patterns.

A single row in the Now column of the roadmap (chapter 01) resolves one build-vs-buy decision per quarter. More than that is thrashing.

## Decision 1 — MCP registry: self-host vs. managed

[`mod-403`](../mod-403-mcp-registry-and-tool-gateway) built the MCP registry and tool-gateway plane. The Anthropic-led [Model Context Protocol (MCP)](https://modelcontextprotocol.io/) specification and its [GitHub organisation](https://github.com/modelcontextprotocol) define the server shape; a registry sits one level above and catalogues servers.

**Vendor landscape (names only, not endorsements):**

- [Smithery](https://smithery.ai/) — a public registry of MCP servers with search and metadata.
- [mcp.so](https://mcp.so/) — a public registry with community submissions.
- [PulseMCP](https://www.pulsemcp.com/) — a public registry plus curation layer.
- [Composio](https://composio.dev/) — an MCP-shaped tool catalog with managed hosting.
- Each major cloud ships a tool-catalog interface tied to its agent service (Vertex / Foundry / Bedrock).

**Axis read for an enterprise agent platform in late 2026:**

| Axis | Score | Why |
|---|---|---|
| Strategic fit | 3 | The registry *shape* matters (policy plane integration, tenant scoping, provenance) but the search engine does not. Could buy the search; must build the governance seam. |
| Regulatory | 4 | Public registries do not gate tool metadata; internal tools' names and schemas may be sensitive. Private registries required for internal tools. |
| Integration tax | 3 | <!-- needs-research: current MCP Registry spec draft and Smithery / mcp.so API — the OpenAPI schema and auth posture ship in-progress; verify on the next quarterly re-decision. --> |
| Total cost | 3 | Self-host MCP registry cost is modest (it is a search + metadata service); managed registries have per-call pricing that scales with tenant growth. |
| Reversibility | 1 | MCP itself is an open spec; registry data can be exported as MCP server manifests; swap is cheap. |
| Team fit | 4 | Registry + search is a standard platform capability the team already has. |

**Typical decision shape (2026):** *Build the private registry for internal tools; subscribe to a public registry as a federated source for well-known public servers, behind the policy plane's identity check.* Reversibility is high on both sides — if Smithery or equivalent matures and one tenant pulls, the switch is cheap.

**Re-decision horizon:** 6 months. The MCP registry spec and vendor landscape are moving quarterly.

## Decision 2 — Sandboxed code execution: self-host vs. E2B / Daytona / Modal

[`mod-404`](../mod-404-sandboxed-code-execution-plane) built the sandbox plane on [gVisor](https://gvisor.dev/), [Firecracker](https://firecracker-microvm.github.io/), or [Kata Containers](https://katacontainers.io/). The managed alternatives:

- **[E2B](https://e2b.dev/)** — an open-source sandbox plus a managed hosting tier; purpose-built for agent code execution. Open-source repo: [`e2b-dev/E2B`](https://github.com/e2b-dev/E2B).
- **[Daytona](https://daytona.io/)** — managed cloud development environments, with an agent-sandbox use case. Open-source core: [`daytonaio/daytona`](https://github.com/daytonaio/daytona).
- **[Modal](https://modal.com/)** — serverless compute with a Python SDK suitable for agent code execution.
- **[CodeSandbox CDE](https://codesandbox.io/)**, **[Gitpod](https://www.gitpod.io/)**, **[Coder](https://coder.com/)** — adjacent CDE vendors occasionally used for agent sandboxing.

**Axis read:**

| Axis | Score | Why |
|---|---|---|
| Strategic fit | 2 | Sandbox substrate is a commodity; what matters is the *policy plane's* decisions about what to run, not the micro-VM substrate itself. |
| Regulatory | 3 | Vendor hosting may conflict with residency; E2B and Daytona offer self-host of the open-source core which recovers residency control. |
| Integration tax | 3 | Vendors expose their own SDKs; a shim is needed to route via the paved-road SDK. Non-trivial but bounded. |
| Total cost | 4 | A self-host Firecracker fleet has non-trivial operational cost. For tenants whose sandbox-second counts are high and bursty, self-host is cheaper at scale; for most, managed is competitive. |
| Reversibility | 3 | The sandbox *interface* (process, filesystem, resource limits) is standard; the orchestration above is vendor-specific. A one-quarter migration is realistic. |
| Team fit | 3 | Firecracker and gVisor operation is specialist; most platform teams have *some* skill but not the depth required for a frontier sandbox. |

**Typical decision shape (2026):** *Hybrid. Managed sandbox (E2B or Daytona) for most tenants; self-host for the one or two T1 tenants with residency / cost constraints. The paved-road SDK's sandbox client hides which is in use.* The re-decision horizon is 12 months — sandbox vendors mature quickly, but switching cost is bounded and the reversibility score (3) does not demand quarterly review.

**Re-decision horizon:** 12 months.

## Decision 3 — Runtime / orchestration: self-host vs. LangGraph Platform / Vertex Agent Engine / Azure AI Foundry Agent Service / Bedrock AgentCore

This is the biggest decision in the portfolio. [`mod-402`](../mod-402-agent-runtime-service-and-durable-execution) built the runtime on [Temporal](https://docs.temporal.io/) (or equivalent durable-execution substrate). The managed alternatives:

- **[LangGraph Platform](https://www.langchain.com/langgraph-platform)** — LangChain's managed deployment for [LangGraph](https://langchain-ai.github.io/langgraph/) graphs; agent runtime with durable state.
- **[Vertex AI Agent Engine](https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/overview)** — Google Cloud's managed agent runtime; integrates with [Google ADK](https://google.github.io/adk-docs/).
- **[Azure AI Foundry Agent Service](https://learn.microsoft.com/en-us/azure/ai-foundry/agents/overview)** — Microsoft's managed agent service; integrates with [Microsoft Agent Framework](https://learn.microsoft.com/en-us/agent-framework/overview).
- **[AWS Bedrock AgentCore](https://aws.amazon.com/bedrock/agentcore/)** — AWS's managed agent runtime; integrates with Bedrock-hosted models. <!-- needs-research: Bedrock AgentCore's current GA status and feature set move; verify the plane offerings on the quarterly re-decision. -->

**Axis read:**

| Axis | Score | Why |
|---|---|---|
| Strategic fit | 5 | The runtime is where every other plane plugs in. Losing control of the session shape means losing control of the whole platform. |
| Regulatory | 3 | Cloud-vendor services offer per-region hosting and BYOK; third-party vendors (LangGraph Platform) have narrower regional coverage. |
| Integration tax | 5 | Managed runtimes impose their own session model, their own telemetry shape, their own policy hooks. Every plane the platform owns is affected. |
| Total cost | 3 | Managed pricing scales with session volume; self-host amortises. The crossover is at mid-to-large tenant scale. |
| Reversibility | 4 | Runtime decisions lock you in deeply — session state, checkpoint envelopes, tool-call abstractions are usually vendor-specific. A re-decision is a multi-quarter effort. |
| Team fit | 3 | Temporal or equivalent is a specialist operation but a well-known one; the team usually has or can hire for the skill. |

**Typical decision shape (2026):** *Build. The runtime is strategic (axis 1 = 5), the integration tax of a managed alternative is high (axis 3 = 5), and reversibility is low (axis 4 = 4). Reconsider annually; the trigger that changes the decision would be (a) a managed runtime that speaks OpenTelemetry-GenAI natively and exposes the session state as an open contract, or (b) an organisation-level mandate to move all compute onto one cloud's managed services.*

A nuance: *agent-service* adoption might make sense for a specific tenant — the clinical-trials copilot already on Vertex, say. The platform supports that tenant via the paved-road SDK's Vertex adapter while continuing to own the runtime shape for everyone else. This is not "buy"; it is "accommodate" without changing the portfolio.

**Re-decision horizon:** 12 months.

## The decision-record shape

A build-vs-buy decision is not an email thread. It is a durable artefact in the governance log. The record:

```yaml
- id: "BvB-2026-Q4-02"
  component: "Agent runtime / orchestration"
  date: "2026-11-15"
  owner: "@platform-lead"
  decision: "Build (continue self-host on Temporal)"
  vendors_evaluated:
    - name: "LangGraph Platform"
      notes: "Strong LangGraph-native support; narrower regional coverage; managed state model incompatible with our checkpoint envelope."
    - name: "Vertex AI Agent Engine"
      notes: "Mature Google-cloud integration; ties tenant to Vertex's Agent Engine session model."
    - name: "Azure AI Foundry Agent Service"
      notes: "Mature Microsoft-cloud integration; session model tied to Foundry."
    - name: "Bedrock AgentCore"
      notes: "Newer; feature parity still catching up."  # cite upstream docs on next refresh
  axis_scores:
    strategic_fit: 5
    regulatory: 3
    integration_tax: 5
    total_cost: 3
    reversibility: 4
    team_fit: 3
  summary: >
    Self-host on Temporal. The runtime's seam onto every other plane makes
    managed alternatives expensive to integrate, and reversibility is low
    enough that the decision deserves a 12-month re-decision horizon rather
    than a quarterly one.
  tenant_accommodations:
    - "globex-health runs on Vertex AI Agent Engine for its clinical-trials
       copilot; paved-road SDK's Vertex adapter wraps it."
  triggers_for_re_decision:
    - "A managed runtime speaks OpenTelemetry-GenAI session conventions natively."
    - "An org-wide mandate to consolidate compute onto a single cloud."
    - "Our tenant count exceeds N where operation cost crosses licence cost."
  re_decision_horizon: "2027-11-15"
  signed_by:
    - "@platform-lead"
    - "@executive-sponsor"
    - "@tenant-council-chair"
```

Three features of the shape are load-bearing:

1. **Vendors evaluated includes every vendor considered.** A decision record that evaluates only the vendor the team was going to pick is a rationalisation document, not a decision. Include the ones you rejected; record why.
2. **Triggers for re-decision are explicit.** "We might reconsider later" is not a trigger; "if OTel-GenAI becomes native in vendor X" is. The trigger is what the quarterly roadmap review scans.
3. **Signed by three parties.** Not approved; signed. Platform-team lead for technical ownership, sponsor for organisational authority, tenant-council chair for the stream-aligned teams whose work depends on the decision. Signatures make reversals explicit; a reversal is a new BvB record, not an amendment.

## When buy is overwhelmingly favoured

Three patterns.

1. **Commodity component with open standards.** The trace collector (OTel Collector), the dashboard renderer (Grafana), the identity provider (Okta / Azure AD), the secret manager (Vault / AWS Secrets Manager / GCP Secret Manager) — nobody should be building these. The platform's opinion is expressed in the configuration, not the substrate.
2. **Specialist capability well outside the team's skill.** Prompt-injection detection models, specialised judges for an evaluation plane, custom vector databases for billion-item retrieval. Buying a vendor-supplied model and wrapping it via the safety plane (mod-411) is almost always correct; building one is a research program.
3. **Early-stage experimentation.** A plane that has one tenant pulling, a six-month roadmap horizon, and no proven usage pattern should be bought-with-a-shim; the shim becomes the reference for a later build if the pull strengthens.

## When build is overwhelmingly favoured

Three patterns.

1. **The thing expresses the platform's opinion.** The policy plane (mod-405), the paved-road SDK (mod-412), the eval regression gates (mod-407), the safety plane's hook shape (mod-411). These are what makes the platform *this* platform; outsourcing them cedes the opinion.
2. **The integration tax of any managed alternative dominates the licence savings.** The runtime decision above is the paradigm case.
3. **Regulatory constraints disqualify every vendor for a majority of tenants.** If more than half the tenants cannot use a vendor, the maintenance cost of a dual-path (build-for-them, buy-for-us) usually exceeds a single build-for-all.

## The hybrid pattern (and its trap)

The hybrid pattern — build the strategic seam, buy the commodity substrate — is almost always the right answer in the medium term. The MCP registry decision above is a worked hybrid: build the private registry, subscribe to a public one; the policy plane decides federation.

The trap: hybrids multiply surface area. Each vendor is a dependency on vendor-specific telemetry, vendor-specific auth, vendor-specific billing. A platform running three commodity vendors has three upgrade cycles, three support contracts, three incident channels to coordinate across (chapter 03). The rule of thumb: a hybrid is cheaper if the component has an *open standard* at the seam. If the seam is proprietary, the hybrid is a long-term maintenance burden.

MCP is an open standard at the registry seam. OpenTelemetry-GenAI is an open standard at the telemetry seam. OCI is an open standard at the container seam. Where open standards exist, hybrid is cheap. Where they do not (most agent-runtime vendors in late 2026), hybrid is expensive and the decision should be binary.

## The portfolio view

A platform has roughly one build-vs-buy decision per plane, plus adjacent commodity decisions (dashboards, secret managers, CI runners). The *portfolio* is the vector of all open decisions, not any single one.

A well-run portfolio has these properties:

- **At most one build-vs-buy decision resolves per quarter.** More and the team cannot absorb the integration work.
- **No more than ~30 % of planes are on a managed service.** More and the platform risks losing the integration seam it is supposed to own.
- **Every managed service has a re-decision horizon.** None is "forever".
- **The commodity decisions are bulk-approved annually.** The platform team does not re-evaluate Grafana vs Datadog every quarter; it picks one, documents the decision, and revisits yearly.
- **The strategic decisions are reviewed quarterly against triggers.** Not the full re-decision; the trigger scan: has any trigger fired?

The portfolio view is what the mod-413 cadence (chapter 01) governs. Build-vs-buy *decisions* are plane-specific; the *portfolio discipline* is leadership-level.

## Common failure modes

### The "vendor-asked-for-a-meeting" decision

**Symptom:** A roadmap Now entry is "evaluate Vendor X" with no axis scoring, no trigger, no tenant puller.

**Cause:** A vendor sales engagement spawned work. The rubric was not applied.

**Fix:** Vendors do not put entries on the roadmap. Tenants do. If no tenant is pulling the vendor, the vendor meeting is a research spike, not a roadmap entry.

### The "we've always built it" decision

**Symptom:** A component the team has owned for three years has never been re-evaluated. Operational cost is high and climbing; a competent vendor now exists.

**Cause:** Sunk-cost bias. The team's identity is bound up in the thing they built.

**Fix:** The annual strategy refresh (chapter 01) re-runs the rubric on every plane whose last BvB record is older than the re-decision horizon. A record older than its horizon is stale.

### The "cheapest at the top of the funnel" decision

**Symptom:** A vendor's self-serve pricing page was the only cost input. At your scale the Enterprise tier is 20× that, and only the Enterprise tier meets axis 2 (regulatory).

**Cause:** The rubric was scored against the wrong tier.

**Fix:** Score against the tier that will actually be purchased. Get a quote. The self-serve price is marketing.

### The "integration tax paid once" decision

**Symptom:** The axis-3 score assumed a one-time shim; a year later the shim is a maintenance drain of two engineer-weeks per quarter.

**Cause:** Integration tax was underestimated because shims are initially cheap but the vendor's data model drifts.

**Fix:** Axis-3 scoring includes *ongoing* maintenance, not just initial shim cost. The decision record's annual review re-reads this score with eyes on actual drift.

### The "regulatory posture at signing" decision

**Symptom:** A vendor met regulatory requirements when signed, but a tenant's compliance scope expanded (new jurisdiction, new regime) and the vendor cannot follow.

**Cause:** Regulatory posture was treated as static.

**Fix:** The quarterly trigger scan includes a regulatory delta check. New jurisdictions, new regimes, new attestations — each is a trigger candidate.

### The "exit cost unexamined" decision

**Symptom:** After two years on a managed runtime, the team cannot articulate what migrating off would cost. The decision has become a trap.

**Cause:** Exit cost was not an axis at signing, or was underweighted.

**Fix:** The decision record's `triggers_for_re_decision` includes a measurable one. Annual drill: a one-paragraph "what would migrating off cost today?" exercise. If the answer is "we don't know", the exit cost has become unbounded.

## Summary

- Build-vs-buy is a portfolio decision, not an engineering decision; it reverses asymmetrically (build→buy is cheap, buy→build is expensive); it is political as well as technical.
- Every component is scored on six axes: strategic fit, regulatory posture, integration tax, total cost, reversibility, team fit. Scores are not summed; they are read together.
- Three live decisions anchor the chapter: the MCP registry (typical late-2026 shape: hybrid), the sandbox substrate (typical shape: hybrid with self-host for regulated tenants), the runtime (typical shape: build, with per-tenant accommodation for cloud-native tenants).
- Every decision lives in a signed decision record with vendors evaluated, axis scores, tenant accommodations, triggers for re-decision, and a re-decision horizon. No "we might reconsider later".
- At most one build-vs-buy decision resolves per quarter. The portfolio holds the discipline; individual plane owners hold the decisions.
- Common failure modes: vendor-driven decisions, sunk-cost persistence, wrong-tier pricing, underestimated integration tax, regulatory drift, unexamined exit cost.

The next chapter turns from roadmap-level decisions to the moment the platform is on fire: *cross-team incident command* when a platform outage and a product-side agent misbehaviour fire simultaneously and the organisation needs one incident commander, not six.
