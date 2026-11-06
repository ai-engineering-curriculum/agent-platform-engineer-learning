# Agent Platform Engineer Curriculum

<!-- aicg:orientation:start -->
> **Start here:** [Module 00: Your Relationship to the Model](lessons/00-your-relationship-to-the-model/README.md). Where this role sits relative to the model, what it owns, and who it works with across the sibling curricula.
<!-- aicg:orientation:end -->

> Curriculum plan authored 2026-11-06 under the WebSearch-unavailable bootstrap convention — see [`JOB_REQUIREMENTS.md`](JOB_REQUIREMENTS.md) and [`.aicg/job-requirements.json`](.aicg/job-requirements.json). The autonomous loop will populate this file further as modules ship.

## Overview

**Level 40, AI Engineering family.** The Agent Platform Engineer builds and operates the *internal platform other teams ship agents on*. The role is not a product-agent builder (that is `agentic-ai-engineer` at level 30 and `senior-agentic-ai-engineer` at level 40) and not an agentic-system designer (that is `agentic-systems-architect` at level 48). It is the platform engineer whose users are those two roles and their teams. The platform spans:

- a multi-tenant **agent-runtime service** built on durable-execution substrates (Temporal / Dapr Workflows / Restate / Inngest);
- an **MCP server registry and tool-gateway plane** with OAuth 2.1 scope exchange, schema governance, conformance testing, and compliance audit;
- a multi-tenant **sandboxed code-execution plane** (Firecracker / gVisor / Kata / managed E2B / Daytona / Modal);
- a **tool and permission policy plane** (OPA / Rego, Cedar, SPIFFE / SPIRE, capability-model enforcement);
- a **model-gateway and routing layer** for agent workloads (LiteLLM / Portkey / Envoy / Kong / Cloudflare AI gateways);
- a **shared evaluation / replay / regression-gating platform** (Inspect, Promptfoo, Langfuse / Phoenix / LangSmith) wired to standard benchmarks (SWE-bench, τ-bench, GAIA, AgentBench);
- **OpenTelemetry-GenAI-shaped observability** for the agent fleet;
- **inter-agent (A2A) routing** and service mesh;
- a platform-level **safety plane / guardrails-as-a-service** (NeMo Guardrails / Guardrails AI / Llama Guard / ShieldGemma / OpenAI Moderation / Constitutional Classifiers / Lakera / Presidio) that operationalises findings produced by `ai-risk-engineer` (level 25) and `agentic-safety-engineer` (level 40);
- the **paved-road SDKs, templates, and developer experience** stream-aligned teams adopt.

## Ownership rule

Assign primary coverage to the lowest-level role that genuinely requires the skill. Higher-level tracks link to that owner unless additional platform scope, depth, or leadership context is required.

This role owns the *platform-engineering slice of agent engineering* — the shared runtime, registry, policy, gateway, eval, observability, safety, and paved-road substrate other teams consume. It defers:

- **Down** to [`ai-infra-junior-engineer`](../ai-infra-junior-engineer-learning) (level 10) for engineering-craft prerequisites, [`llm-application-developer`](../llm-application-developer-learning) (level 25) for LLM fundamentals, and [`agentic-ai-engineer`](../agentic-ai-engineer-learning) (level 30) for build-altitude agent fundamentals.
- **Sideways** to [`ai-eval-engineer`](../ai-eval-engineer-learning) (peer, level 30) for application-layer eval methodology; [`ai-infra-ml-platform`](../ai-infra-ml-platform-learning) (peer, level 30) and [`ai-infra-mlops`](../ai-infra-mlops-learning) (peer, levels 25–30) for ML / MLOps substrate; [`ai-infra-sre`](../ai-infra-sre-learning) (peer, level 30) for SRE substrate; [`ai-infra-inference-engineer`](../ai-infra-inference-engineer-learning) (peer, level 35) for the inference-serving substrate the model gateway routes to; [`senior-agentic-ai-engineer`](../senior-agentic-ai-engineer-learning) (peer, level 40) for product-side multi-agent subsystem implementation leadership; [`agentic-safety-engineer`](../agentic-safety-engineer-learning) (peer, level 40) for frontier-agent red-team methodology; [`ai-risk-engineer`](../ai-risk-engineer-learning) (peer, level 25) for harm-model authoring and guardrail-effectiveness methodology.
- **Up** to [`ai-infra-architect`](../ai-infra-architect-learning) (level 45), [`agentic-systems-architect`](../agentic-systems-architect-learning) (level 48), [`ai-infra-senior-architect`](../ai-infra-senior-architect-learning) (level 55), [`ai-infra-principal-engineer`](../ai-infra-principal-engineer-learning) (level 60), and [`chief-ai-officer`](../chief-ai-officer-learning) (level 70).
- **Out of scope** to legal / privacy / compliance counsel and to SecOps / DFIR.

## Module plan

Totals: **13 modules, 166 hours** + **4 projects, 160 hours** → **326 hours** (level-40 band is 300–350 hours).

| # | Module | Hours | Focus |
|---|---|---|---|
| mod-401 | [Agent Platform Engineer — Position, Scope, and Paved-Road Philosophy](lessons/mod-401-agent-platform-position-and-paved-road-philosophy) | 10 | Ladder position, CNCF Platform Engineering Maturity Model, Team Topologies, paved-road vs. one-off |
| mod-402 | [Agent Runtime Service and Durable Execution](lessons/mod-402-agent-runtime-service-and-durable-execution) | 16 | Multi-tenant runtime, Temporal / Dapr / Restate / Inngest, session state, Kubernetes + Knative substrate |
| mod-403 | [MCP Server Registry and Tool Gateway Plane](lessons/mod-403-mcp-registry-and-tool-gateway) | 16 | MCP spec + authorization, OAuth 2.1 scope exchange, registry, schema governance, conformance tests, audit trail |
| mod-404 | [Sandboxed Code-Execution Plane for Agents](lessons/mod-404-sandboxed-code-execution-plane) | 16 | Firecracker / gVisor / Kata, syscall policy, resource quotas, E2B / Daytona / Modal fallback |
| mod-405 | [Tool Registry and Permission Policy Plane](lessons/mod-405-tool-and-permission-policy-plane) | 12 | OPA / Rego, Cedar, SPIFFE / SPIRE, capability model, approval workflows, policy regression tests |
| mod-406 | [Model Gateway and Routing for Agent Workloads](lessons/mod-406-model-gateway-and-routing) | 14 | LiteLLM / Portkey / Envoy / Kong / Cloudflare AI gateways, routing, fallback, cost attribution |
| mod-407 | [Shared Evaluation Platform and Regression Gating](lessons/mod-407-evaluation-platform-and-regression-gating) | 14 | Inspect / Promptfoo, trace / replay store, eval-gated CI, SWE-bench / τ-bench / GAIA / AgentBench fixtures |
| mod-408 | [Observability and OpenTelemetry for Agent Fleets](lessons/mod-408-observability-and-opentelemetry-for-agents) | 12 | OTel GenAI semantic conventions, cross-plane trace propagation, Langfuse / Phoenix / Datadog LLM Observability |
| mod-409 | [Agent-to-Agent Routing and Inter-Agent Service Mesh](lessons/mod-409-agent-to-agent-routing-and-service-mesh) | 10 | A2A spec, identity / permission propagation, circuit breakers, bulkheads |
| mod-410 | [Reliability Engineering and SRE for the Agent Platform](lessons/mod-410-reliability-engineering-and-sre-for-agents) | 12 | Agent-workload SLIs / SLOs, chaos drills, incident playbooks, regression-fixture back-feed |
| mod-411 | [Safety Plane and Guardrails-as-a-Service](lessons/mod-411-safety-plane-and-guardrails-as-a-service) | 12 | NeMo Guardrails / Guardrails AI / Llama Guard / ShieldGemma / OpenAI Moderation / Constitutional Classifiers / Lakera / Presidio, OWASP Agentic defence |
| mod-412 | [Paved-Road SDKs, Templates, and Developer Experience](lessons/mod-412-paved-road-sdks-and-developer-experience) | 12 | SDK templates, scaffolding CLI, local-dev loop, adoption metrics |
| mod-413 | [Platform Leadership and Cross-Team Operating Cadence](lessons/mod-413-platform-leadership-and-operating-cadence) | 10 | Roadmap authoring, build-vs-buy, cross-team incident command, DORA / SPACE / platform scorecards |

## Project plan

| # | Project | Hours | Summary |
|---|---|---|---|
| project-401 | [Mini Agent Platform — Runtime + MCP Registry + Sandbox + Eval](projects/project-401-mini-agent-platform) | 50 | End-to-end minimal platform. Runtime API, MCP registry with OAuth scopes, sandbox gateway, eval harness that gates deploys, OTel spans across every plane |
| project-402 | [Governed MCP Gateway — OAuth 2.1, Schema Governance, Audit Trail](projects/project-402-governed-mcp-gateway) | 40 | Production-shaped MCP gateway: dynamic client registration, scope exchange, approval workflow, deprecation ladder, compliance audit trail. Validated against an abuse corpus |
| project-403 | [Eval-Gated CI/CD for an Agent Fleet with Replay Store](projects/project-403-eval-gated-agent-fleet) | 40 | Evaluation platform slice: trace / replay store, Inspect harness template, dataset versioning, CI/CD gate wired to SWE-bench / τ-bench / GAIA + a product fixture; EU AI Act Article 72 evidence pack |
| project-404 | [Model Gateway and Safety Plane for Multi-Tenant Agent Workloads](projects/project-404-model-gateway-with-cost-and-safety-plane) | 30 | Gateway slice: multi-provider routing, per-tenant quotas, cost attribution, guardrails-as-a-service pipeline (NeMo Guardrails + Llama Guard + Presidio), shadow eval + canary |

## Per-module objectives (summary)

### mod-401 — Agent Platform Engineer — Position, Scope, and Paved-Road Philosophy

Diagram the role's scope on the ladder against `agentic-ai-engineer` (30), `senior-agentic-ai-engineer` (40), `agentic-systems-architect` (48), `ai-infra-architect` (45), `ai-infra-inference-engineer` (35), `ai-eval-engineer` (30), `ai-risk-engineer` (25), and `agentic-safety-engineer` (40). Apply the CNCF Platform Engineering Maturity Model and Team Topologies to characterise the platform team's operating model. Read Anthropic *Building Effective Agents* and the OpenAI *Practical Guide to Building Agents* as the guidance stream-aligned teams onboard with. Position OWASP LLM Top 10 and OWASP Agentic AI Threats and Mitigations as the baseline the safety plane must defend against.

### mod-402 — Agent Runtime Service and Durable Execution

Architect a multi-tenant agent-runtime service — session lifecycle, context compaction, checkpoint / resume, cost-aware concurrency. Compare Temporal, Dapr Workflows, Restate, Inngest for agent workloads. Integrate LangGraph checkpoints, OpenAI Responses state, Anthropic Agent SDK session state, Google ADK, Microsoft Agent Framework into one session-state abstraction. Reason about state substrates (Postgres + pgvector, Redis, object storage) on Kubernetes operators vs. Knative.

### mod-403 — MCP Server Registry and Tool Gateway Plane

Read the Model Context Protocol spec and the MCP authorization spec at engineering depth. Engineer a central MCP server registry — versioning, deprecation, approval workflows, schema governance, Inspector-shaped conformance tests, failover routing, auth-wrapping proxy for non-OAuth servers. Use Arcade.dev / Composio / Toolhouse as build-vs-buy anchors. Author per-tool compliance audit evidence.

### mod-404 — Sandboxed Code-Execution Plane for Agents

Compare Firecracker, gVisor, Kata, and WASM / Pyodide against the OWASP LLM and Agentic threat models. Engineer syscall policy, egress rules, filesystem capabilities, and resource quotas; author leak-eviction policy. Integrate E2B, Daytona, Modal Sandboxes as managed fallbacks. Implement snapshot-based cold-start acceleration. Instrument sandbox events into the compliance evidence stream.

### mod-405 — Tool Registry and Permission Policy Plane

Author tool-authorisation policy as code in OPA / Rego and Cedar — RBAC for human operators, ABAC for per-tenant data boundaries, capability model for agent sessions. Engineer non-human identity for agents (SPIFFE / SPIRE), short-lived tokens, credential brokering. Author approval workflows and deprecation ladders. Write policy regression tests against a representative abuse corpus.

### mod-406 — Model Gateway and Routing for Agent Workloads

Architect a model gateway — multi-provider routing, fallback and retry policies tuned for tool-calling loops, per-tenant quotas, burst budgets, token / cost attribution. Compare LiteLLM, Portkey, Envoy AI Gateway, Kong AI Gateway, Cloudflare AI Gateway. Engineer prompt-cache coordination across tenants. Enforce structured output and PII redaction at the gateway. Hand off to `ai-infra-inference-engineer` (level 35) for the serving substrate.

### mod-407 — Shared Evaluation Platform and Regression Gating

Build an evaluation-harness template library on top of Inspect and Promptfoo. Engineer dataset versioning, provenance, synthetic-data labelling, access control. Stand up the trace / replay store. Wire eval-gated CI/CD against SWE-bench, τ-bench, GAIA, AgentBench plus per-team fixtures. Hand off to `ai-eval-engineer` (level 30) for methodology depth.

### mod-408 — Observability and OpenTelemetry for Agent Fleets

Adopt the OpenTelemetry Semantic Conventions for Generative AI as the instrumentation contract. Engineer cross-plane distributed trace propagation (runtime → gateway → MCP → sandbox → eval). Integrate OpenLLMetry, Langfuse, Phoenix, Datadog LLM Observability, Grafana + Prometheus against the same semantic layer. Author conformance tests. Produce the EU AI Act Article 72 evidence slice from the trace pipeline.

### mod-409 — Agent-to-Agent Routing and Inter-Agent Service Mesh

Read the A2A protocol specification. Engineer inter-agent routing with service discovery, correlation-ID propagation, identity delegation, permission-scope propagation, and tenant isolation. Apply circuit breakers, bulkheads, and cascading-failure prevention. Reason about handoffs vs. manager-as-tools topologies.

### mod-410 — Reliability Engineering and SRE for the Agent Platform

Define SLIs / SLOs for agent workloads — session completion rate, time-to-first-tool, time-to-final-answer, hallucination rate, safety-refusal rate. Engineer error-budget policies stream-aligned teams opt into. Run chaos drills for model-provider outage, MCP-server flake, sandbox exhaustion, eval-infra degradation. Author incident-response playbooks for agent-specific failures (runaway loops, context-window exhaustion, prompt-injection containment). Feed post-incident regression fixtures back into mod-407.

### mod-411 — Safety Plane and Guardrails-as-a-Service

Engineer the platform's safety plane as a shared service — input filters, output validators, tool-invocation policy enforcement, standardised refusal behaviour — on NeMo Guardrails, Guardrails AI, Llama Guard, ShieldGemma, OpenAI Moderation, Constitutional Classifiers, Lakera Guard, Presidio. Defend against OWASP LLM Top 10 and OWASP Agentic AI Threats and Mitigations at platform scope. Measure effectiveness against red-team findings produced by `ai-risk-engineer` (25) and `agentic-safety-engineer` (40). Emit a standardised safety-event schema for post-market monitoring.

### mod-412 — Paved-Road SDKs, Templates, and Developer Experience

Author paved-road SDK templates using Anthropic Agent SDK, OpenAI Agents SDK, Google ADK, Microsoft Agent Framework, LangGraph + Temporal. Ship scaffolding CLI, local-dev loop (sandbox emulation, trace viewer, offline eval replay), CI templates, instrumentation hooks. Define adoption metrics against the CNCF Platform Engineering Maturity Model. Operate office-hours, migration playbooks, deprecation ladders.

### mod-413 — Platform Leadership and Cross-Team Operating Cadence

Author a platform roadmap against stream-aligned team needs using the CNCF maturity model and Team Topologies as the operating frame. Run the build-vs-buy portfolio for agent-platform components. Command cross-team incidents spanning platform outage and product-side misbehaviour; coordinate with peer tracks and SecOps. Measure platform success — DORA, SPACE, platform-adoption rate, time-to-first-agent, support-ticket MTTR.

## References

See [`JOB_REQUIREMENTS.md`](JOB_REQUIREMENTS.md) and [`.aicg/job-requirements.json`](.aicg/job-requirements.json) for the full authoritative-reference catalog. Every module above cites at least one authoritative public reference.
