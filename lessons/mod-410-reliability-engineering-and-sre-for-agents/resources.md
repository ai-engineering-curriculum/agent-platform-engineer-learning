# Resources — mod-410 (Reliability Engineering and SRE for the Agent Platform)

All references on this page are primary / authoritative public sources at the time of writing (2026-11-06). Each resource is tagged with the chapter(s) it supports. Prefer the primary source over any blog summary — SRE practices evolve, chaos-engineering tooling ships quarterly, and the alerting-rule canon in the SRE Workbook is the source of truth for the specific thresholds everyone else quotes.

## Chapter 01 — The SRE Plane and What It Owns

### Foundational SRE texts
- **Google — Site Reliability Engineering (the SRE Book)**: <https://sre.google/sre-book/table-of-contents/>
- **Google — The Site Reliability Workbook**: <https://sre.google/workbook/table-of-contents/>
- **Google — Building Secure and Reliable Systems**: <https://sre.google/books/building-secure-reliable-systems/>
- **SRE Book — Service Level Objectives**: <https://sre.google/sre-book/service-level-objectives/>
- **SRE Book — Managing Incidents**: <https://sre.google/sre-book/managing-incidents/>
- **SRE Book — Postmortem Culture**: <https://sre.google/sre-book/postmortem-culture/>

### Platform engineering and operating model
- **Skelton & Pais — Team Topologies**: <https://teamtopologies.com/>
- **CNCF — Platforms White Paper**: <https://tag-app-delivery.cncf.io/whitepapers/platforms/>
- **CNCF — Platform Engineering Maturity Model**: <https://tag-app-delivery.cncf.io/whitepapers/platform-eng-maturity-model/>

### Agent-platform context
- **Anthropic — Building Effective Agents**: <https://www.anthropic.com/research/building-effective-agents>
- **OpenAI — A Practical Guide to Building Agents (PDF)**: <https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf>

## Chapter 02 — SLIs and SLOs for Agent Workloads

### SRE methodology for SLIs / SLOs
- **SRE Book — Service Level Objectives (chapter 4)**: <https://sre.google/sre-book/service-level-objectives/>
- **SRE Workbook — Implementing SLOs (chapter 2)**: <https://sre.google/workbook/implementing-slos/>
- **SRE Workbook — Alerting on SLOs (chapter 5)**: <https://sre.google/workbook/alerting-on-slos/>

### Metrics backends and alerting
- **Prometheus — documentation**: <https://prometheus.io/docs/>
- **Prometheus — recording rules**: <https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/>
- **Prometheus — alerting rules**: <https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/>
- **Prometheus — unit testing rules**: <https://prometheus.io/docs/prometheus/latest/configuration/unit_testing_rules/>
- **Alertmanager — documentation**: <https://prometheus.io/docs/alerting/latest/alertmanager/>
- **Grafana — documentation**: <https://grafana.com/docs/grafana/latest/>
- **Grafana Alerting**: <https://grafana.com/docs/grafana/latest/alerting/>
- **Grafana Mimir**: <https://grafana.com/docs/mimir/latest/>
- **VictoriaMetrics**: <https://docs.victoriametrics.com/>
- **Datadog — metrics**: <https://docs.datadoghq.com/metrics/>

### OpenTelemetry semantic contract (cross-ref mod-408)
- **OpenTelemetry — Semantic Conventions for Generative AI**: <https://opentelemetry.io/docs/specs/semconv/gen-ai/>
- **OpenTelemetry — specification**: <https://opentelemetry.io/docs/specs/otel/>
- **OpenTelemetry — collector**: <https://opentelemetry.io/docs/collector/>

### Quality-as-reliability (judge calibration, hallucination SLO)
- **Anthropic — Building Effective Agents (eval section)**: <https://www.anthropic.com/research/building-effective-agents>
- **Arize AI — LLM evaluation guide**: <https://arize.com/docs/phoenix/>
- **Langfuse — scoring and evaluation**: <https://langfuse.com/docs>

<!-- needs-research: confirm the Workbook's alert-threshold table (fast-burn multiplier for a 28-day window). The published table anchors on 30-day; the 28-day values follow from the Workbook's formulas but are worth citing directly from the final chapter-5 appendix when authoring production alerts. -->

## Chapter 03 — Error-Budget Policy as Paved-Road Contract

### Error-budget methodology
- **SRE Book — Embracing Risk (chapter 3)**: <https://sre.google/sre-book/embracing-risk/>
- **SRE Workbook — Implementing SLOs (chapter 2)**: <https://sre.google/workbook/implementing-slos/>
- **Google Cloud — Error Budgets guide**: <https://cloud.google.com/architecture/framework/reliability/measure-reliability-with-slos-and-slis>

### Policy-as-code and CI gating
- **Open Policy Agent (OPA) — documentation**: <https://www.openpolicyagent.org/docs/latest/>
- **GitHub Actions — required checks**: <https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/collaborating-on-repositories-with-code-quality-features/about-status-checks>
- **GitHub — merge queues**: <https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/incorporating-changes-from-a-pull-request/merging-a-pull-request-with-a-merge-queue>
- **GitLab CI — merge trains**: <https://docs.gitlab.com/ee/ci/pipelines/merge_trains.html>

### Progressive delivery and rollback
- **Argo Rollouts**: <https://argoproj.github.io/argo-rollouts/>
- **Flagger**: <https://flagger.app/>

### Organisational contracts
- **Skelton & Pais — Team Topologies (X-as-a-service)**: <https://teamtopologies.com/key-concepts-content/what-are-the-fundamental-team-topologies>

## Chapter 04 — Chaos Drills for Agent Dependencies

### Chaos engineering
- **Principles of Chaos Engineering**: <https://principlesofchaos.org/>
- **Rosenthal & Jones — Chaos Engineering: System Resilience in Practice (O'Reilly)**: <https://www.oreilly.com/library/view/chaos-engineering/9781492043858/>
- **Netflix — Chaos Monkey (archived; historical reference)**: <https://netflix.github.io/chaosmonkey/>
- **Netflix — Chaos Engineering (original paper)**: <https://netflixtechblog.com/the-netflix-simian-army-16e57fbab116>

### Chaos tooling
- **Chaos Mesh**: <https://chaos-mesh.org/>
- **Chaos Mesh — HTTPChaos**: <https://chaos-mesh.org/docs/simulate-http-chaos-on-kubernetes/>
- **LitmusChaos**: <https://litmuschaos.io/>
- **Toxiproxy**: <https://github.com/Shopify/toxiproxy>
- **AWS Fault Injection Service**: <https://docs.aws.amazon.com/fis/latest/userguide/what-is.html>
- **Azure Chaos Studio**: <https://learn.microsoft.com/en-us/azure/chaos-studio/>
- **Steadybit**: <https://www.steadybit.com/>
- **Gremlin**: <https://www.gremlin.com/>

### Mesh-level fault injection (cross-ref mod-409)
- **Istio — fault injection task**: <https://istio.io/latest/docs/tasks/traffic-management/fault-injection/>
- **Envoy — fault filter**: <https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/fault_filter>

### Load generation
- **k6 — documentation**: <https://k6.io/docs/>
- **vegeta — HTTP load testing**: <https://github.com/tsenart/vegeta>
- **hey — HTTP load generator**: <https://github.com/rakyll/hey>

<!-- needs-research: confirm the Chaos Mesh CRD field names for HTTPChaos (ports/method/pattern) against the pinned release; the fields have shifted between v2.0 and v2.5 releases. -->

## Chapter 05 — Incident Response Playbooks for Agent Failure Modes

### Incident management methodology
- **SRE Book — Managing Incidents (chapter 14)**: <https://sre.google/sre-book/managing-incidents/>
- **SRE Workbook — Incident Response (chapter 9)**: <https://sre.google/workbook/incident-response/>
- **SRE Book — Postmortem Culture (chapter 15)**: <https://sre.google/sre-book/postmortem-culture/>
- **US DHS — Incident Command System (ICS) overview**: <https://www.fema.gov/emergency-managers/nims/components>
- **PagerDuty — Incident Response documentation**: <https://response.pagerduty.com/>

### Incident-management platforms
- **PagerDuty — documentation**: <https://developer.pagerduty.com/docs/>
- **Opsgenie — documentation**: <https://www.atlassian.com/software/opsgenie>
- **Grafana Incident**: <https://grafana.com/products/cloud/incident/>
- **FireHydrant**: <https://firehydrant.com/>
- **incident.io**: <https://incident.io/>

### Agent-specific failure modes
- **OWASP — LLM Top 10 for LLM Applications (project page)**: <https://genai.owasp.org/llm-top-10/>
- **OWASP — LLM01: Prompt Injection**: <https://genai.owasp.org/llmrisk/llm01-prompt-injection/>
- **NIST — Adversarial Machine Learning: A Taxonomy and Terminology (AI 100-2)**: <https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.100-2e2023.pdf>
- **Anthropic — Prompt injection research**: <https://www.anthropic.com/research>
- **MITRE ATLAS — adversarial ML threat matrix**: <https://atlas.mitre.org/>

### Cascading-failure canon (cross-ref mod-409)
- **SRE Book — Addressing Cascading Failures**: <https://sre.google/sre-book/addressing-cascading-failures/>
- **SRE Book — Handling Overload**: <https://sre.google/sre-book/handling-overload/>
- **Nygard — Release It! (2nd ed.)**: <https://pragprog.com/titles/mnee2/release-it-second-edition/>

## Chapter 06 — Post-Incident Regression Fixtures into the Eval Plane

### Post-incident methodology
- **SRE Book — Postmortem Culture**: <https://sre.google/sre-book/postmortem-culture/>
- **SRE Workbook — Postmortem Culture**: <https://sre.google/workbook/postmortem-culture/>
- **Google Cloud — Postmortem template**: <https://cloud.google.com/architecture/framework/reliability/postmortem-culture>

### Evaluation platforms (cross-ref mod-407)
- **Inspect (UK AISI)**: <https://inspect.aisi.org.uk/>
- **Promptfoo — documentation**: <https://www.promptfoo.dev/docs/>
- **OpenAI Evals (archived reference)**: <https://github.com/openai/evals>
- **Langfuse — datasets and evaluation**: <https://langfuse.com/docs/datasets>
- **Arize Phoenix**: <https://arize.com/docs/phoenix/>

### Regression testing and benchmark canon
- **SWE-bench**: <https://www.swebench.com/>
- **GAIA benchmark**: <https://huggingface.co/datasets/gaia-benchmark/GAIA>
- **τ-bench (tau-bench)**: <https://github.com/sierra-research/tau-bench>
- **AgentBench**: <https://github.com/THUDM/AgentBench>

### Observability (cross-ref mod-408, for trace capture / snapshot)
- **OpenTelemetry — Semantic Conventions for Generative AI**: <https://opentelemetry.io/docs/specs/semconv/gen-ai/>
- **OpenTelemetry — collector tail sampling**: <https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/tailsamplingprocessor>
- **Grafana Tempo — documentation**: <https://grafana.com/docs/tempo/latest/>
- **Jaeger**: <https://www.jaegertracing.io/>

## Cross-cutting references

### Within this track
- **mod-402 — Agent Runtime Service and Durable Execution**: `../mod-402-agent-runtime-service-and-durable-execution/`
- **mod-403 — MCP Registry and Tool Gateway**: `../mod-403-mcp-registry-and-tool-gateway/`
- **mod-404 — Sandboxed Code Execution Plane**: `../mod-404-sandboxed-code-execution-plane/`
- **mod-405 — Tool and Permission Policy Plane**: `../mod-405-tool-and-permission-policy-plane/`
- **mod-406 — Model Gateway and Routing**: `../mod-406-model-gateway-and-routing/`
- **mod-407 — Shared Evaluation Platform and Regression Gating**: `../mod-407-evaluation-platform-and-regression-gating/`
- **mod-408 — Observability and OpenTelemetry for Agent Fleets**: `../mod-408-observability-and-opentelemetry-for-agents/`
- **mod-409 — Agent-to-Agent Routing and Inter-Agent Service Mesh**: `../mod-409-agent-to-agent-routing-and-service-mesh/`
- **mod-411 — Safety Plane and Guardrails as a Service**: `../mod-411-safety-plane-and-guardrails-as-a-service/`
- **mod-412 — Paved-Road SDKs and Developer Experience**: `../mod-412-paved-road-sdks-and-developer-experience/`
- **mod-413 — Platform Leadership and Operating Cadence**: `../mod-413-platform-leadership-and-operating-cadence/`

### Role boundaries (within the Agent Career Curriculum ecosystem)
- **Track overview (this repository)**: `../../README.md`
- **Job requirements (citations the curriculum writes against)**: `../../JOB_REQUIREMENTS.md`

## Notes on using these references

- The SRE Book and the SRE Workbook are the canonical sources for SLI / SLO / error-budget / incident-response methodology. Secondary summaries (blog posts, conference talks, vendor landing pages) are everywhere and usually paraphrase; when a specific threshold, formula, or alert shape matters in a production artefact, quote the Book or Workbook directly, pin the chapter, and cite.
- The Workbook's multi-window multi-burn-rate alerting table (fast-burn page at 14.4× for 1h, slow-burn ticket at 6× for 6h, etc.) is anchored to a 30-day window. The paved-road default in this module is 28 days; the arithmetic translation is a trivial scalar adjustment but worth stating explicitly in any alerting-rule PR.
- The chaos-engineering tooling landscape moves fast — Chaos Mesh, LitmusChaos, Steadybit, Gremlin, and the cloud-provider FIS variants ship product changes quarterly. The CRD field names for `HTTPChaos` and the fault specs for Istio's `VirtualService.fault` have shifted between releases; pin the version in any drill harness that depends on them.
- OWASP's LLM Top 10 moved from "2023" to "2025" labelling in 2025 and the category numbers have re-arranged; "LLM01: Prompt Injection" has been stable but the companion categories (data poisoning, insecure output handling) have renumbered. Any reference to a specific LLM0X label should state the rev.
- The eval-platform landscape (Inspect, Promptfoo, Langfuse, Phoenix, OpenAI Evals) releases breaking changes on a near-quarterly cadence. The fixture-promotion exercise in chapter 06 assumes the mod-407 catalogue schema is stable; your stub implementation should mirror that schema even when the production catalogue moves.
- The agent-topology frameworks (OpenAI Agents SDK, LangGraph, CrewAI, AutoGen) release breaking changes often. Chapter 05's playbooks reference `platform.agent.tool_arg_hash` as a paved-road SDK attribute (mod-412); the real attribute name in your SDK may differ. Pin the SDK version in the playbook.
- Mark `<!-- needs-research: ... -->` on any derived artefact where a citation on this page does not match the version you are shipping against. The flag convention is consistent with mod-403 / 404 / 405 / 406 / 407 / 408 / 409; it blocks auto-merge, which is cheaper than silently guessing.
