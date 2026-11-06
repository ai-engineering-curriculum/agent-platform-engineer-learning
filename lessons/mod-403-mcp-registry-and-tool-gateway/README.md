# mod-403 — MCP Server Registry and Tool Gateway Plane

**Estimated effort:** 16 hours · **Position in track:** third module (after [`mod-401`](../mod-401-agent-platform-position-and-paved-road-philosophy) and [`mod-402`](../mod-402-agent-runtime-service-and-durable-execution)).

Agents without tools are talkers. The tool-gateway plane is where a platform decides *how* its agents reach into the world — through which registry, under which auth, past which policy, with which audit trail, backed by its own MCP servers or by managed vendors sitting behind the gateway. By the end of this module you can read the Model Context Protocol specification and its authorization spec at engineering depth, design the registry data model your gateway will stand on, specify the OAuth 2.1 flow with scope exchange it enforces, ship a conformance and health-check pipeline that keeps that registry honest, wrap legacy non-OAuth tools without breaking the plane's obligations, decide which tools to buy versus build versus refuse, and emit the per-invocation audit record external auditors and incident responders actually use.

## Learning objectives

- Read the Model Context Protocol specification (resources, tools, prompts, transports, lifecycle) and the MCP authorization specification (OAuth 2.1 with PKCE, Dynamic Client Registration, Resource Indicators, Token Exchange, Protected Resource Metadata) at engineering depth — meaning section-level fluency across the underlying RFCs, not a vendor summary.
- Engineer a central MCP server registry — versioning, deprecation, approval workflows, schema governance, MCP Inspector-shaped conformance tests, failover routing, and an auth-wrapping proxy for servers that cannot speak OAuth.
- Reason about managed MCP / tool-catalog vendors (Arcade.dev, Composio, Toolhouse, and the moving landscape they sit in) as build-vs-buy anchors and place them explicitly as fallbacks *behind* the platform's own gateway — never in front of it.
- Author per-tool compliance audit evidence — who invoked what tool, under which scope, for which agent session, with which inputs and outputs, immutably recorded and queryable by the shapes external auditors and incident responders actually use.
- Hand off cleanly to `agentic-ai-engineer` (level 30) for authoring the MCP servers that run on the platform, and to `agentic-safety-engineer` (level 40) and `ai-infra-security` peers for deeper MCP protocol-security research.

## Chapters

1. [`01-the-tool-gateway-plane-and-the-mcp-contract.md`](01-the-tool-gateway-plane-and-the-mcp-contract.md) — The slice the tool-gateway plane owns, MCP's three primitives (tools, resources, prompts), three transports (stdio, Streamable HTTP, WebSocket), three lifecycle phases, and the three non-negotiable gateway obligations.
2. [`02-mcp-authorization-at-engineering-depth.md`](02-mcp-authorization-at-engineering-depth.md) — OAuth 2.1, PKCE, Protected Resource Metadata (RFC 9728), Authorization Server Metadata (RFC 8414), Dynamic Client Registration (RFC 7591), Resource Indicators (RFC 8707), Token Exchange (RFC 8693), the confused-deputy class, and a worked end-to-end example.
3. [`03-the-mcp-server-registry.md`](03-the-mcp-server-registry.md) — The registry's four surfaces (discovery, admission, governance, health), the server-entry schema, semver rules and version pinning, the approval ladder (proposed → review → staging → GA → deprecated → sunset), schema governance, and GitOps-as-registry.
4. [`04-conformance-health-and-failover.md`](04-conformance-health-and-failover.md) — Three conformance layers (protocol, schema, semantic), probe cadence and placement, HealthEvents, three failover strategies (active-passive, degraded mode, circuit breaker), and the SLIs the SRE plane (mod-410) consumes.
5. [`05-auth-wrapping-proxy-for-non-oauth-servers.md`](05-auth-wrapping-proxy-for-non-oauth-servers.md) — The proxy pattern for legacy tools, credential-brokering patterns (static, per-tenant static, workload identity federation, user impersonation, signed-request, mTLS), scope-to-permission mapping, and the seven anti-patterns to refuse.
6. [`06-managed-mcp-vendors-and-build-vs-buy.md`](06-managed-mcp-vendors-and-build-vs-buy.md) — Portfolio framing (own / proxy-through-vendor / refuse), the vendor landscape by category, Tier-1/Tier-3 triggers, what vendors do and do not do well, failover topology with a vendor, and the ten-question evaluation checklist.
7. [`07-compliance-audit-evidence.md`](07-compliance-audit-evidence.md) — The `ToolInvocationAudit` record schema, payload modes (raw / hashed / redacted), immutability layers, retention horizons, the right-to-erasure tension, linkage to traces (mod-408) and the replay store (mod-407), and the EU AI Act Article 72 evidence-pack preview.

## Exercises

Each exercise turns one or more chapters into an artefact you keep and reuse — in the rest of the module, in projects 401 and 402, and in later modules (mod-405, mod-407, mod-408, mod-410, mod-413).

- [`exercises/exercise-01-mcp-server-registry-schema-design.md`](exercises/exercise-01-mcp-server-registry-schema-design.md) — MCP server registry schema design. Pairs with chapters 01 and 03. Feeds projects 401 and 402.
- [`exercises/exercise-02-mcp-authorization-oauth21-scope-exchange.md`](exercises/exercise-02-mcp-authorization-oauth21-scope-exchange.md) — End-to-end authorization sequence with scope exchange and a confused-deputy mitigation review. Pairs with chapter 02. Feeds exercise 03 and project 402.
- [`exercises/exercise-03-auth-wrapping-proxy-for-legacy-tools.md`](exercises/exercise-03-auth-wrapping-proxy-for-legacy-tools.md) — Auth-wrapping proxy for one concrete legacy tool. Pairs with chapters 02 and 05. Feeds projects 401 and 402.
- [`exercises/exercise-04-mcp-conformance-health-check-pipeline.md`](exercises/exercise-04-mcp-conformance-health-check-pipeline.md) — Probe suite, per-tier failover policies, and the SLI/SLO proposal for mod-410. Pairs with chapter 04. Feeds projects 401 and 403.

## Resources

External references — primary sources only — are catalogued in [`resources.md`](resources.md).

## How to approach this module

1. Read the seven chapters in order. The authorization chapter (02) presumes you have the plane-and-obligations frame (01); the registry chapter (03) assumes the auth model; the conformance chapter (04) reads from the registry; the auth-wrapping proxy chapter (05) is where the auth story meets the legacy world; the build-vs-buy chapter (06) places the vendor landscape against everything above; the audit chapter (07) is the thing that holds all six accountable.
2. Do exercise 01 before reading chapter 04 if you want the fastest feedback — your registry schema is what the conformance pipeline writes into and the audit pipeline references. The other exercises assume exercise 01's artefact exists.
3. Read the RFCs listed for chapter 02. The MCP authorization spec is a thin binding layer over a stack of RFCs; a surface understanding of the layer is not enough to design the gateway's auth. The investment pays back everywhere later in this module and across mod-405.
4. Mark `<!-- needs-research: ... -->` on any claim you cannot verify against the current MCP spec revision or vendor documentation. The MCP spec, the registry work, and the vendor landscape are all in motion through 2026; the flag is cheaper than silently guessing, and auto-merge will block on it.

## Where the handoffs land

- **`agentic-ai-engineer` (level 30)** — authors the MCP servers that your registry admits. Your contract to them is the paved-road template (mod-412), the registry schema (exercise 01), and the conformance suite (exercise 04). You admit their work; you do not critique their business logic.
- **`agentic-safety-engineer` (level 40) and `ai-infra-security` peers** — own MCP protocol-security research in depth (remote server attack surface, confused-deputy variants, tool-poisoning, supply-chain compromise). You operationalise their findings into admission gates, scope caps, and audit queries. You bring platform telemetry (chapter 07's record stream) to them; they bring threat models back to you.
- **`senior-agentic-ai-engineer` (peer, level 40)** — builds product-side agents that call your tools. They care that discovery is stable, auth is predictable, and failover is graceful. They do not want to know about registry schemas unless you failed them.
- **`ai-infra-sre` (peer) and your own future mod-410 work** — consume the SLIs produced by chapter 04 and the audit volumes produced by chapter 07. Your SLO proposals (exercise 04 Part 3) land as their SLO definitions.
