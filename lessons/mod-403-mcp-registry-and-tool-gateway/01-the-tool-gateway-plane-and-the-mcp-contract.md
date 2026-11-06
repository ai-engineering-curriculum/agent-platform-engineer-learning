# The Tool-Gateway Plane and the MCP Contract

A platform's runtime (mod-402) lets an agent *exist*. The tool-gateway plane lets that agent *do anything*. Every non-trivial agent in the fleet eventually reaches past the model and into the world — a calendar, a database, a code host, a payments API — and every one of those reaches is a tool call. If your platform does not own that reach, every product team ends up re-implementing it: their own MCP clients, their own OAuth dance, their own schema review, their own audit trail, their own vendor contracts. By the time three teams have done that, you are not a platform anymore, you are a style guide.

This chapter fixes the contract the gateway plane holds. The rest of the module picks substrates, auth flows, registries, and vendors against it.

## What MCP is, at the shape your plane has to support

The **Model Context Protocol** is a JSON-RPC 2.0 wire protocol between an MCP *client* (embedded in your runtime or agent framework) and an MCP *server* (which exposes resources, tools, or prompts). It is maintained as an open specification at `modelcontextprotocol.io`, with a dated revision cadence (the 2025-06-18 revision is the reference this module is written against; check the current revision when you onboard).

The protocol has three primitives:

- **Tools.** Model-invocable functions with a JSON Schema input, a JSON Schema output, and a server-defined behaviour. A tool's declaration is read by the model; its invocation is initiated by the model's assistant turn; its result flows back as a tool-result message. Tools are the *doing* primitive.
- **Resources.** Named, URI-addressable chunks of context the model may read (files, database rows, API responses). Resources are declarative — the server exposes URIs and MIME types; the client fetches on demand. Resources are the *reading* primitive.
- **Prompts.** Templated user/assistant messages a server advertises for well-known workflows. Clients may offer them in a UI (slash-commands, menu items). Prompts are the *UX-hint* primitive — a server telling a client "here is how to use me well."

Three **transports** are defined:

- **stdio.** Client spawns the server as a subprocess; messages flow over stdin/stdout. This is the transport local tools use (file-system access, process shelling, your IDE's dev server).
- **Streamable HTTP.** The current remote transport. A single endpoint accepts POSTed JSON-RPC requests and streams responses (and server-initiated notifications) back via Server-Sent Events or chunked HTTP. In the current revision this replaced the two-endpoint HTTP+SSE transport from earlier drafts; existing servers in the wild may still speak the old shape, so your gateway has to recognise both. <!-- needs-research: confirm the exact revision at which Streamable HTTP superseded HTTP+SSE in the MCP spec and whether backward compatibility is still normative. -->
- **WebSocket.** Reserved for low-latency / bidirectional cases. Seen rarely; your plane can defer deciding how to support it.

The **lifecycle** has three phases:

1. **Initialize.** Client sends `initialize` with its protocol version and capabilities; server responds with its protocol version and capabilities; client follows with `initialized` notification. This handshake is the only place capability negotiation happens — if the server does not advertise `tools` here, no `tools/*` call later is meaningful.
2. **Operation.** `tools/list`, `tools/call`, `resources/list`, `resources/read`, `prompts/list`, `prompts/get`, plus `ping` and server-initiated notifications (e.g. `notifications/tools/list_changed` when the tool set changes).
3. **Shutdown.** Explicit close for stdio; HTTP transports rely on connection teardown.

If you have internalised these three primitives, three transports, and three lifecycle phases, you can read any MCP server source and know what it is doing. If you have not, you will design a registry that cannot describe half the servers in the world.

## The slice this plane owns

The tool-gateway plane sits between the runtime (mod-402) and the actual MCP servers (whether written in-house, delivered by a managed vendor, or wrapped around a legacy API). It owns:

- **A single tool-access endpoint for every agent in the fleet.** Agents do not know URLs, credentials, or vendor names. They see a logical tool name and a schema; the gateway resolves the rest.
- **Discovery.** Which tools exist, which version, which schema, which scopes they need, which transports they speak, which risk tier they are in.
- **Authorization enforcement.** The gateway is an OAuth 2.1 client (chapter 02); it owns the token exchange, scope selection, and audience checks. Agents never hold raw tokens.
- **Schema integrity.** The schema an agent sees at discovery is the schema that validates the invocation at call time. If the upstream server changes its schema between those two moments, the gateway is the thing that notices.
- **Conformance and health.** A server that fails a conformance test (chapter 04) is removed from discovery before the next agent can see it.
- **Audit emission.** Every tool call, in or out, produces an attributable record (chapter 07).

The gateway plane does **not** own:

- **The tools themselves.** Writing MCP servers is `agentic-ai-engineer` (level 30) craft. The gateway publishes templates and admits the output; it does not author the tool logic.
- **The sandbox a tool runs in.** That is mod-404. The gateway may invoke a sandbox when a tool call needs one, but the sandbox is its own plane with its own obligations.
- **Policy decisions about who can call what.** That is the tool and permission policy plane (mod-405). The gateway consults the policy plane on every call; it does not author policy.
- **Model choice.** That is the model gateway (mod-406). Tools are not called by models directly in this world; they are called by the runtime after a model requests them. Separation of concerns.
- **Guardrail content.** That is mod-411. If a tool output needs to be filtered (PII in the result, prompt-injection in the result), the safety plane does the filtering; the gateway routes the output through it.

If any of these starts creeping onto the gateway, your plane is drifting off slice. Hold the line.

## The three obligations

Pin the gateway plane to three user-facing obligations. Everything else — registry design, OAuth choices, vendor topology, audit shape — flows from these.

1. **"One address, every tool."** Product teams see a single endpoint and a single discovery interface. Changing which concrete MCP server implements a tool (switching from a self-hosted server to a managed vendor, or vice versa) does not change the client code. This obligation forces the registry, the routing layer, and the auth-wrapping proxy (chapter 05) to exist.

2. **"Schema frozen between discovery and invocation."** If an agent read tool `search_calendar` at version 2.3.1 at the start of a turn, calling `search_calendar` later in that same turn gets the 2.3.1 schema, regardless of what the upstream server now advertises. This obligation forces version-pinning and defeats a whole class of silently drifting failures where an agent "learned" a tool with one shape and got executed against another.

3. **"Every invocation is attributable before it leaves your plane."** For every tool call, the gateway persists the (user, tenant, agent, session, tool name, tool version, scope, policy decision id, input, output, timestamp, trace id) tuple before the call is considered complete. This obligation forces the audit pipeline (chapter 07), the integration with the policy plane (mod-405), and the trace propagation into mod-408.

None of these is optional. A gateway that cannot defend all three has not reached the altitude the role is written at; it is a thin HTTP proxy wearing a registry costume.

## The neighbours

The gateway plane talks to every other plane in the track. Writing the contracts down once saves re-litigating them later.

| Plane | Gateway's end of the conversation |
|-------|-----------------------------------|
| Runtime (mod-402) | "Here is a tool client for session S. When the agent calls a tool, I handle discovery, auth, routing, and invocation. I return a result or a structured error; I never leak internal credentials." |
| Sandboxed execution (mod-404) | "For tools that run user code, I invoke the sandbox plane on your behalf; I pass through capabilities and quotas; I translate sandbox errors into MCP tool-call errors." |
| Policy plane (mod-405) | "For every call, I query you with (session, tool, args-summary, scope) and expect an allow / deny / step-up decision. I carry your decision id into the audit record." |
| Model gateway (mod-406) | "I do not call you. The runtime calls you for model completions and me for tools; we meet at the session." |
| Eval platform (mod-407) | "Every tool-call record I write lands in the trace / replay store alongside yours, keyed by trace id." |
| Observability (mod-408) | "Every discovery, auth exchange, and invocation emits OTel spans in the `gen_ai.tool.*` conventions; my audit pipeline pivots on your trace ids." |
| A2A routing (mod-409) | "I expose inter-agent messaging as a tool shape; a call to the A2A plane goes through me so that the policy and audit obligations hold." |
| SRE (mod-410) | "My SLIs are time-to-admit new server, conformance-pass rate, p99 discovery latency, auth-exchange error rate, failover latency. My error budgets are yours to track." |
| Safety plane (mod-411) | "You are a filter I compose into my response path — tool outputs flow through you before reaching the runtime." |
| Paved-road SDK (mod-412) | "The SDK gets a tool-client factory that is backed by me. SDK users never see MCP, OAuth, or a registry URL." |

## The two handoffs this track makes you responsible for

- **To `agentic-ai-engineer` (level 30).** They write the actual MCP servers that back the tools. Your plane admits them, versions them, retires them. Your contract to them is in the registry schema (chapter 03) and the paved-road template (mod-412). You do not critique their business logic; you reject their submission if it fails conformance, schema review, or security review.
- **To `agentic-safety-engineer` (level 40) and `ai-infra-security` peers.** They own the deep MCP-protocol-security research — remote server attack surfaces, confused-deputy variants, tool-poisoning, supply-chain compromise of MCP server images. Your plane operationalises what they find into gates: scope caps, admission reviews, audit queries, red-team fixtures. You bring platform telemetry to them; they bring threat models back to you.

## Summary

- MCP has three primitives (tools, resources, prompts), three transports (stdio, Streamable HTTP, WebSocket), and three lifecycle phases (initialize, operation, shutdown). Your plane has to support all of them.
- The tool-gateway plane owns the single tool-access endpoint, discovery, authorization enforcement, schema integrity, conformance, and audit emission. It does *not* own tools, sandboxes, policy, model choice, or guardrails.
- Three obligations are non-negotiable: one address every tool, schema frozen between discovery and invocation, every invocation attributable before it leaves the plane.
- Every other plane in the track meets you at the session and via a bounded contract. Drift off slice and the track collapses in on itself.

The next chapter pins the first obligation that bites in practice: authorization. MCP is an OAuth 2.1 story end-to-end; if you cannot read the authorization spec at engineering depth you cannot operate this plane.
