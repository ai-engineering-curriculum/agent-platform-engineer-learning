# AI Engineering · Agent Platform Engineer — Learning Repository

<!-- aicg:site-banner -->
> 🎓 Part of the free, open-source **AI Career Curriculum** ecosystem — [Infrastructure](https://github.com/ai-infra-curriculum) · [ML Engineering](https://github.com/ml-engineering-curriculum) · [AI Engineering](https://github.com/ai-engineering-curriculum) · [Governance](https://github.com/ai-governance-curriculum) · [Startup](https://github.com/ai-startup-curriculum) · [Workforce](https://github.com/ai-workforce-curriculum).
<!-- /aicg:site-banner -->

Builds and operates the internal platform other teams ship agents on: a multi-tenant agent-runtime service, an MCP server registry and tool-gateway plane, a sandboxed code-execution plane, a tool and permission policy plane, a model-gateway and routing layer, a shared evaluation / replay / regression-gating platform, OpenTelemetry-GenAI-shaped observability, inter-agent (A2A) routing, platform-level guardrails-as-a-service, and the paved-road SDKs and templates stream-aligned teams adopt.

**Level 40, AI Engineering family.** Peer to `senior-agentic-ai-engineer` (product-side multi-agent subsystem lead), `agentic-safety-engineer` (frontier-agent red-team), `ai-infra-inference-engineer` (inference-serving substrate), and `ai-eval-engineer` (application-layer eval methodology). Reports up into `agentic-systems-architect` (design) and `ai-infra-architect` (cross-stack infra).

## Curriculum shape

**13 modules (166h)** + **4 projects (160h)** = **326 hours** (level-40 band is 300–350 hours).

- [`CURRICULUM.md`](CURRICULUM.md) — module and project map.
- [`PREREQUISITES.md`](PREREQUISITES.md) — assumed lower-level tracks, working knowledge, not-taught-here.
- [`JOB_REQUIREMENTS.md`](JOB_REQUIREMENTS.md) — role-requirements catalog and ownership rule.
- [`VERSIONS.md`](VERSIONS.md) — release history.
- [`.aicg/job-requirements.json`](.aicg/job-requirements.json) and [`.aicg/curriculum-plan.json`](.aicg/curriculum-plan.json) — machine-readable packets.

> **Status**: curriculum plan authored 2026-11-06 under the WebSearch-unavailable bootstrap convention. Live-posting sampling is deferred to the next autonomous research cycle — see [`JOB_REQUIREMENTS.md`](JOB_REQUIREMENTS.md) → *Status*. Lesson and project bodies will be filled by subsequent runs of `aicg org daily`.

## Layout

```
agent-platform-engineer-learning/
├── lessons/mod-XXX-*/        modules with lectures, exercises, labs, quizzes
├── projects/project-XXX-*/   multi-module capstones
├── CURRICULUM.md             role-level coverage map
├── PREREQUISITES.md          assumed entry skills
├── JOB_REQUIREMENTS.md       requirements catalog + ownership rule
├── VERSIONS.md               release history
└── README.md                 this file
```

## Paired Solutions Repo

[`agent-platform-engineer-solutions`](https://github.com/ai-engineering-curriculum/agent-platform-engineer-solutions) carries the reference implementations.

---

<!-- aicg:maintained-by -->
Maintained by [VeriSwarm.ai](https://veriswarm.ai)
