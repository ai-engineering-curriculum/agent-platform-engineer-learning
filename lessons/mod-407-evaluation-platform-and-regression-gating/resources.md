# Resources — mod-407 (Shared Evaluation Platform and Regression Gating)

All references on this page are primary / authoritative public sources at the time of writing (2026-11-06). Each resource is tagged with the chapter(s) it supports. Prefer the primary source over any blog summary — eval frameworks, benchmark revisions, judge methodology, and provider pricing in this space move quarter-by-quarter and secondary sources lag.

## Chapter 01 — The Evaluation Plane and What It Owns

### Platform-engineering background
- **CNCF — Platform Engineering Maturity Model**: <https://tag-app-delivery.cncf.io/whitepapers/platform-eng-maturity-model/>
- **CNCF — Platforms White Paper**: <https://tag-app-delivery.cncf.io/whitepapers/platforms/>
- **Google SRE Book — Service Level Objectives**: <https://sre.google/sre-book/service-level-objectives/>

### Agent-platform guidance (cross-ref mod-401)
- **Anthropic — Building Effective Agents**: <https://www.anthropic.com/research/building-effective-agents>
- **OpenAI — A Practical Guide to Building Agents**: <https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf>

### Evaluation foundations
- **Shankar et al. — Who Validates the Validators? Aligning LLM-Assisted Evaluation of LLM Outputs with Human Preferences (UIST 2024)**: <https://arxiv.org/abs/2404.12272>
- **Zheng et al. — Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena (NeurIPS 2023)**: <https://arxiv.org/abs/2306.05685>
- **Chang et al. — A Survey on Evaluation of Large Language Models**: <https://arxiv.org/abs/2307.03109>

## Chapter 02 — The Harness Template Library on Inspect and Promptfoo

### Inspect AI (UK AI Security Institute)
- **Inspect AI — documentation**: <https://inspect.aisi.org.uk/>
- **Inspect AI — repository**: <https://github.com/UKGovernmentBEIS/inspect_ai>
- **`inspect-evals` — benchmark pack**: <https://github.com/UKGovernmentBEIS/inspect_evals>
- **Inspect AI — tutorials and recipes**: <https://inspect.aisi.org.uk/tutorial.html>

### Promptfoo
- **Promptfoo — documentation**: <https://www.promptfoo.dev/docs/>
- **Promptfoo — repository**: <https://github.com/promptfoo/promptfoo>
- **Promptfoo — assertions reference**: <https://www.promptfoo.dev/docs/configuration/expected-outputs/>
- **Promptfoo — HTTP provider**: <https://www.promptfoo.dev/docs/providers/http/>
- **Promptfoo — model-graded metrics**: <https://www.promptfoo.dev/docs/configuration/expected-outputs/model-graded/>
- **Promptfoo — `promptfoo eval` CLI**: <https://www.promptfoo.dev/docs/usage/command-line>

### LLM-as-a-judge methodology
- **OpenAI Evals — repository**: <https://github.com/openai/evals>
- **Anthropic — Evaluating Model Outputs**: <https://docs.anthropic.com/en/docs/build-with-claude/evaluate-prompts>
- **Liu et al. — G-Eval: NLG Evaluation using GPT-4 with Better Human Alignment**: <https://arxiv.org/abs/2303.16634>
- **Wang et al. — Large Language Models are not Fair Evaluators**: <https://arxiv.org/abs/2305.17926>

### Observability for eval runs (cross-ref mod-408)
- **OpenTelemetry — Semantic Conventions for Generative AI**: <https://opentelemetry.io/docs/specs/semconv/gen-ai/>
- **OpenTelemetry — Traces specification**: <https://opentelemetry.io/docs/specs/otel/trace/>

### LLM observability platforms (useful adjacency)
- **Langfuse — documentation**: <https://langfuse.com/docs>
- **Arize Phoenix — documentation**: <https://docs.arize.com/phoenix>
- **LangSmith — documentation**: <https://docs.smith.langchain.com/>
- **Braintrust — documentation**: <https://www.braintrust.dev/docs>

## Chapter 03 — The Eval Dataset Management Plane

### Dataset versioning tooling
- **DVC — documentation**: <https://dvc.org/doc>
- **DVC — repository**: <https://github.com/iterative/dvc>
- **LakeFS — documentation**: <https://docs.lakefs.io/>
- **LakeFS — repository**: <https://github.com/treeverse/lakeFS>
- **Hugging Face Datasets — documentation**: <https://huggingface.co/docs/datasets>
- **Hugging Face Hub — dataset revisions**: <https://huggingface.co/docs/hub/datasets-versioning>

### Dataset metadata and provenance standards
- **MLCommons — Croissant dataset metadata**: <https://mlcommons.org/working-groups/data/croissant/>
- **Croissant — specification**: <https://github.com/mlcommons/croissant>
- **SPDX — License list**: <https://spdx.org/licenses/>
- **SLSA — Supply-chain Levels for Software Artifacts**: <https://slsa.dev/>
- **in-toto — attestation framework**: <https://in-toto.io/>
- **Sigstore / cosign — documentation**: <https://docs.sigstore.dev/cosign/overview/>

### Content-addressed object storage
- **IPFS — technical primer** (background on CID / Merkle DAG): <https://docs.ipfs.tech/concepts/content-addressing/>
- **Apache Iceberg — documentation**: <https://iceberg.apache.org/docs/latest/>
- **Delta Lake — documentation**: <https://docs.delta.io/latest/index.html>

### Data-contamination and leakage
- **Sainz et al. — NLP Evaluation in Trouble: On the Need to Measure LLM Data Contamination for each Benchmark (EMNLP 2023)**: <https://arxiv.org/abs/2310.18018>
- **Dong et al. — Generalization or Memorization: Data Contamination and Trustworthy Evaluation for Large Language Models (ACL 2024)**: <https://arxiv.org/abs/2402.15938>
- **Hugging Face — Dataset leaderboard contamination tracking**: <https://huggingface.co/spaces/mlfoundations-dev/leaderboard>
<!-- needs-research: specific contamination-tracking tooling and known-contaminated hash sets move faster than the publication record; cross-reference recent working-group output before citing a specific detector. -->

### Access control (cross-ref mod-405)
- **NIST — Guide to Attribute Based Access Control (SP 800-162)**: <https://csrc.nist.gov/pubs/sp/800/162/upd1/final>
- **OPA (Open Policy Agent) — documentation**: <https://www.openpolicyagent.org/docs/latest/>
- **Rego — language reference**: <https://www.openpolicyagent.org/docs/latest/policy-language/>
- **Cedar — documentation**: <https://www.cedarpolicy.com/>
- **SPIFFE — home**: <https://spiffe.io/>
- **SPIRE — repository**: <https://github.com/spiffe/spire>

### Privacy regulations (verify currency against counsel)
- **EU AI Act — final text**: <https://eur-lex.europa.eu/eli/reg/2024/1689/oj>
- **GDPR — Regulation (EU) 2016/679**: <https://eur-lex.europa.eu/eli/reg/2016/679/oj>
- **HIPAA — Privacy Rule overview**: <https://www.hhs.gov/hipaa/for-professionals/privacy/>

## Chapter 04 — The Trace and Replay Store for Regression Investigation

### OpenTelemetry (cross-ref mod-408)
- **OpenTelemetry — Semantic Conventions for Generative AI**: <https://opentelemetry.io/docs/specs/semconv/gen-ai/>
- **OpenTelemetry — Traces specification**: <https://opentelemetry.io/docs/specs/otel/trace/>
- **OpenTelemetry — Protocol specification (OTLP)**: <https://opentelemetry.io/docs/specs/otlp/>

### Trace storage backends
- **ClickHouse — documentation**: <https://clickhouse.com/docs>
- **Apache Iceberg — documentation**: <https://iceberg.apache.org/docs/latest/>
- **Delta Lake — documentation**: <https://docs.delta.io/latest/index.html>
- **Grafana Tempo — documentation**: <https://grafana.com/docs/tempo/latest/>
- **Jaeger — documentation**: <https://www.jaegertracing.io/docs/>
- **DuckDB — documentation**: <https://duckdb.org/docs/>

### LLM-trace stores and replay (useful adjacency)
- **Langfuse — traces and sessions**: <https://langfuse.com/docs/tracing>
- **Arize Phoenix — traces**: <https://docs.arize.com/phoenix/tracing/llm-traces>
- **LangSmith — traces**: <https://docs.smith.langchain.com/concepts/tracing>

### Redaction and PII detectors (cross-ref mod-406 chapter 06)
- **Microsoft Presidio — home**: <https://microsoft.github.io/presidio/>
- **Google Cloud — Sensitive Data Protection (DLP)**: <https://cloud.google.com/sensitive-data-protection>
- **AWS — Amazon Comprehend PII detection**: <https://docs.aws.amazon.com/comprehend/latest/dg/pii.html>

### Replay and simulation research
- **Yao et al. — WebShop: Towards Scalable Real-World Web Interaction with Grounded Language Agents (NeurIPS 2022)**: <https://arxiv.org/abs/2207.01206>
- **AgentSims (useful simulator framing)**: <https://arxiv.org/abs/2308.04026>

## Chapter 05 — Eval-Gated CI/CD and the Standard Regression Fixtures

### SWE-bench
- **SWE-bench — home**: <https://www.swebench.com/>
- **SWE-bench — paper (Jimenez et al. 2024)**: <https://arxiv.org/abs/2310.06770>
- **SWE-bench — repository**: <https://github.com/princeton-nlp/SWE-bench>
- **SWE-bench Lite**: <https://www.swebench.com/lite.html>
- **SWE-bench Verified (OpenAI announcement)**: <https://openai.com/index/introducing-swe-bench-verified/>

### τ-bench
- **τ-bench — repository (Sierra Research)**: <https://github.com/sierra-research/tau-bench>
- **τ-bench — paper (Yao et al. 2024)**: <https://arxiv.org/abs/2406.12045>

### GAIA
- **GAIA — dataset card**: <https://huggingface.co/datasets/gaia-benchmark/GAIA>
- **GAIA — paper (Mialon et al. 2023)**: <https://arxiv.org/abs/2311.12983>

### AgentBench
- **AgentBench — repository**: <https://github.com/THUDM/AgentBench>
- **AgentBench — paper (Liu et al. 2023)**: <https://arxiv.org/abs/2308.03688>

### Other agent / tool-use benchmarks worth knowing
- **Berkeley Function-Calling Leaderboard (BFCL)**: <https://gorilla.cs.berkeley.edu/leaderboard.html>
- **WebArena — home**: <https://webarena.dev/>
- **WebArena — paper**: <https://arxiv.org/abs/2307.13854>
- **MLE-bench (OpenAI)**: <https://openai.com/index/mle-bench/>
- **OSWorld**: <https://github.com/xlang-ai/OSWorld>

### Statistical frames for comparing evals
- **Efron & Tibshirani — An Introduction to the Bootstrap (Chapman & Hall, 1993)** — the canonical bootstrap reference; see also the Wikipedia survey: <https://en.wikipedia.org/wiki/Bootstrapping_(statistics)>
- **Clopper-Pearson interval — background**: <https://en.wikipedia.org/wiki/Binomial_proportion_confidence_interval#Clopper%E2%80%93Pearson_interval>
- **Dror et al. — The Hitchhiker's Guide to Testing Statistical Significance in Natural Language Processing (ACL 2018)**: <https://aclanthology.org/P18-1128/>

### CI/CD tooling
- **GitHub Actions — documentation**: <https://docs.github.com/en/actions>
- **GitLab CI/CD — documentation**: <https://docs.gitlab.com/ee/ci/>
- **Buildkite — documentation**: <https://buildkite.com/docs>
- **Argo Workflows — documentation**: <https://argo-workflows.readthedocs.io/>
- **Tekton Pipelines — documentation**: <https://tekton.dev/docs/pipelines/>

### Model provider pricing (cross-ref mod-406)
- **OpenAI — Pricing**: <https://openai.com/api/pricing/>
- **Anthropic — Pricing**: <https://www.anthropic.com/pricing>
- **Google — Gemini API pricing**: <https://ai.google.dev/pricing>
- **AWS — Bedrock pricing**: <https://aws.amazon.com/bedrock/pricing/>
<!-- needs-research: provider pricing moves quarterly and sometimes faster. Re-verify every per-token rate you cite in a cost estimate or judge-budget calculation. -->

## Chapter 06 — Handoffs to `ai-eval-engineer` and `model-evaluation-engineer`

### Role boundaries and handoff frames
- **Team Topologies (Skelton & Pais) — book site**: <https://teamtopologies.com/>
- **CNCF — Platforms White Paper**: <https://tag-app-delivery.cncf.io/whitepapers/platforms/>

### Application-layer eval methodology (handoff to `ai-eval-engineer`)
- **Liu et al. — G-Eval: NLG Evaluation using GPT-4 with Better Human Alignment**: <https://arxiv.org/abs/2303.16634>
- **Zheng et al. — Judging LLM-as-a-Judge (MT-Bench)**: <https://arxiv.org/abs/2306.05685>
- **Shankar et al. — Who Validates the Validators?**: <https://arxiv.org/abs/2404.12272>
- **Es et al. — RAGAS: Automated Evaluation of Retrieval Augmented Generation**: <https://arxiv.org/abs/2309.15217>
- **TruLens — documentation (RAG eval)**: <https://www.trulens.org/>
- **Ragas — documentation**: <https://docs.ragas.io/>

### Statistical / benchmark methodology (handoff to `model-evaluation-engineer`)
- **NIST — AI Risk Management Framework (Generative AI Profile)**: <https://www.nist.gov/itl/ai-risk-management-framework>
- **HELM (Holistic Evaluation of Language Models) — paper**: <https://arxiv.org/abs/2211.09110>
- **HELM — repository**: <https://github.com/stanford-crfm/helm>
- **Big-Bench — repository**: <https://github.com/google/BIG-bench>
- **lm-evaluation-harness (EleutherAI) — repository**: <https://github.com/EleutherAI/lm-evaluation-harness>
- **Dror et al. — The Hitchhiker's Guide to Testing Statistical Significance in NLP**: <https://aclanthology.org/P18-1128/>

### Safety-plane handoffs (cross-ref mod-411)
- **OWASP Top 10 for LLM Applications**: <https://genai.owasp.org/llm-top-10/>
- **OWASP Agentic AI — Threats and Mitigations**: <https://genai.owasp.org/initiatives/#agentic-security-initiative>
- **MITRE ATLAS**: <https://atlas.mitre.org/>

## Cross-cutting references

- **Inspect AI — documentation**: <https://inspect.aisi.org.uk/>
- **Promptfoo — documentation**: <https://www.promptfoo.dev/docs/>
- **OpenTelemetry — Semantic Conventions for Generative AI**: <https://opentelemetry.io/docs/specs/semconv/gen-ai/>
- **Model Context Protocol — specification** (cross-ref mod-403): <https://modelcontextprotocol.io/specification>
- **JOB_REQUIREMENTS.md** (at the track root): the normalised list of citations the curriculum is written against.

## Notes on using these references

- Every entry on this page is a **primary** or **authoritative-public** source at the time of writing. Benchmark revisions, Inspect / Promptfoo feature sets, and LLM observability platforms are the fastest-moving surfaces on the track; if the source has a version tag — SDK version, benchmark revision hash, dataset revision — read the version that matches what you deploy against.
- Standard fixtures (SWE-bench, τ-bench, GAIA, AgentBench) ship point releases and derived variants on no fixed cadence. Before quoting a specific baseline score or contamination claim, confirm against the current release notes.
- Judge-model and judge-prompt methodology is an active research area. Papers cited above give the frame; `ai-eval-engineer` on your team owns the per-fixture methodological choice.
- Provider pricing and context-window / rate-limit specs move independently of the benchmarks that measure them. A per-token cost you cite today may be stale next quarter; mod-406's price-regression alarm is the operational mitigation.
- Regulatory artefacts (EU AI Act, state privacy laws, sectoral rules) are refined by delegated acts and amendments. Before quoting a specific retention horizon or compliance requirement on dataset provenance, verify against the latest text and against counsel at your organisation.
- Mark `<!-- needs-research: ... -->` on any derived artefact where a citation on this page does not match the version you are shipping against. The flag convention is consistent with mod-403 / 404 / 405 / 406; it blocks auto-merge, and that is cheaper than silently guessing.
