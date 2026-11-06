# Managed MCP Vendors and Build-vs-Buy

The three previous chapters built the case that a platform owns the registry, the auth, the conformance, and the audit. They did not build the case that the platform writes every MCP server itself. That would be a bad trade: the long tail of useful tools (Slack, Jira, Notion, Linear, Google Drive, Dropbox, HubSpot, Salesforce, GitHub, GitLab, PagerDuty, Zendesk, Intercom, Stripe, Twilio, SendGrid, Zoom — the list is well above a hundred) is work the platform team does not uniquely need to do. Several vendors — **Arcade.dev**, **Composio**, **Toolhouse**, and others still establishing themselves — ship pre-authored MCP servers (or MCP-adjacent tool catalogs) and managed OAuth app registrations at exactly this scale. The question for the platform is not "build or buy" at the tool level. It is "which tools do we own in-house, which do we proxy to a vendor from behind our gateway, and which do we refuse to admit at all."

This chapter frames that portfolio decision. It does not endorse a vendor; vendor coverage and quality moves quickly and anything written here will decay. The job is to pick for yourself, with the right questions, repeatedly.

## The portfolio frame

Treat the tool fleet as a three-tier portfolio.

- **Tier 1 — Own.** Tools central to the platform's differentiating behaviour, tools that handle regulated data, tools with strict audit requirements the vendor cannot deliver, tools that must integrate deeply with internal systems. Platform writes the MCP server, hosts it, operates it. Vendor is not in the critical path.
- **Tier 2 — Proxy.** Standard SaaS integrations where a vendor already has a credible, audited MCP server (or tool adapter the platform wraps into MCP). Platform's gateway is still the control point — discovery, auth, policy, audit — but the implementation is the vendor's. Vendor is behind the gateway, not in front of it.
- **Tier 3 — Refuse.** Tools whose risk profile does not justify admission in any form: unaudited data exfiltration surfaces, tools whose vendor refuses an acceptable security review, long-tail tools without a business owner. The registry's answer is "no," with a documented exception path.

The tiering decision belongs to the platform's governance body (which, in mod-413 terms, is you and your peer `agentic-systems-architect` and security partners). The registry records the tier.

## The gateway is the control point in all three tiers

Tier 1 and tier 2 look very different in implementation and very similar from the outside:

| Surface | Tier 1 (own) | Tier 2 (proxy to vendor) |
|---------|--------------|--------------------------|
| Agent-facing endpoint | Your gateway | Your gateway |
| Discovery | Your registry | Your registry |
| OAuth 2.1 with the agent | Your gateway | Your gateway |
| Policy decisions | mod-405 | mod-405 |
| Audit emission | Chapter 07 pipeline | Chapter 07 pipeline |
| MCP server implementation | Platform team | Vendor |
| Downstream credential brokering | Chapter 05 proxy or native OAuth | Vendor (but the gateway still validates audience + scope on inbound) |
| SLO ownership | Platform SRE | Platform SRE with vendor dependency |

The non-negotiable: **the vendor's own endpoint is never exposed to agents.** Agents see the platform's gateway, and the gateway talks to the vendor. If a product team can bypass the gateway and call a vendor directly, the platform has lost scope, audit, and policy enforcement in one move.

This "vendor-behind-gateway" shape is the whole value of treating managed MCP as a buy decision. It means you get the vendor's catalog breadth *and* keep your own obligations intact.

## The vendor landscape — categories, not endorsements

Group vendors by what they primarily sell. Positioning changes; use the category when you read vendor material.

- **Managed OAuth / auth-first.** Vendors whose core value is handling the OAuth app registration, consent, and token refresh for dozens-to-hundreds of SaaS providers. They solve the "we need to register as a client at every IdP in the world" problem. **Arcade.dev** sits here as of late 2026. <!-- needs-research: confirm Arcade's current product scope and auth-primitive naming. -->
- **Tool catalog.** Vendors shipping a broad pre-authored tool catalog, typically exposed through an SDK or API, increasingly through MCP as the standard interface. **Composio** sits here as of late 2026. <!-- needs-research: confirm Composio's current MCP support and catalog size. -->
- **Hosted execution.** Vendors who also run the tool execution for you, often with their own sandbox or compute plane. **Toolhouse** sits here as of late 2026. <!-- needs-research: confirm Toolhouse's current deployment shapes and execution model. -->
- **Generalist AI gateways with tool features.** Some LLM gateways (LiteLLM, Portkey) are adding tool-catalog adjacencies; cloud providers are beginning to publish first-party MCP server libraries. These overlap mod-406 territory; treat them as a potential model-gateway-plus-tools consolidation decision rather than a pure tool buy.
- **Verticals.** Domain-specific tool providers (observability, DevOps, finance) packaging their platform as MCP. Usually tier 1 for domain-central tools; evaluate like any vendor.

Every vendor is in motion. Rather than internalise a landscape map, internalise the questions.

## The questions that pick a tier

For each tool-or-integration-candidate, walk down this list. If "yes" to any of **Tier 1 triggers**, it is tier 1. If "no" to all of **Tier 3 triggers**, consider tier 2.

### Tier 1 triggers

- Does the tool handle data covered by regulation the platform must prove compliance with (PII, PHI, PCI, financial records)?
- Does the tool integrate with systems the vendor cannot access (internal SSO, on-prem data stores, air-gapped environments)?
- Is the tool differentiating — would you offer it as part of your platform's unique value even if a vendor also offers it?
- Does the audit obligation (chapter 07) require an evidence shape the vendor cannot produce?
- Is the SLO tighter than a vendor dependency can honour (sub-second p99 across a geography the vendor does not operate in)?
- Would losing the vendor on short notice create a business-critical dependency failure?

### Tier 3 triggers

- Is the risk category T3 (destructive, high-authority, hard-to-recover) with no demonstrable control story on the vendor side?
- Is the vendor unwilling to sign a DPA, SOC 2 artefacts, or an acceptable security questionnaire?
- Does the vendor's own auth story contain known-bad patterns (OAuth 2.0 implicit, resource-owner-password, shared tenant keys)?
- Is the business owner "nobody in particular"? (A tool without an owner is a tool that will outlive its purpose and become a liability.)

### Tier 2 (everything else)

Standard SaaS integration, no exotic risk, no deep internal entanglement, a credible vendor behind it. Proxy through the gateway; let the vendor's work be work you do not do.

## What vendors typically do well

- **OAuth app registration at scale.** Running `client_id`/`client_secret` hygiene across hundreds of providers, with redirect URI management, consent-screen collateral, and token refresh, is work that benefits from scale. A vendor doing it for a thousand customers will handle the awkward edges better than a platform doing it for its own fleet alone.
- **Consent UX and branded flows.** Users hit vendor-run consent pages that look sane; your gateway does not need to build one.
- **Breadth.** The long tail (dozens of less-common SaaS tools) ships "for free" in the vendor's catalog. Worth the integration cost for the catalog breadth alone.
- **Token refresh and rotation.** Vendors handle the edge cases around refresh expiry, provider rate limits, and silent token invalidation that cost a platform weeks of reactive work.

## What vendors typically do not do well

- **Enterprise-grade audit evidence.** Vendor audit records are shaped for their own operations, not for your Article 72 evidence pack. Expect to recompute audit at the gateway (chapter 07) rather than trust vendor logs end-to-end.
- **Custom policy enforcement.** Vendors apply broad per-tenant policy; fine-grained ABAC against your identity model is rarely available. Policy belongs on mod-405 at your side regardless.
- **Specific scope narrowing.** Vendor-provided scopes tend to be coarser than what you would design in-house. Live with the coarseness when it fits risk; refuse admission when it does not.
- **Guarantees under their own vendor outage.** The vendor's SLA is your SLA floor. If your platform SLO is tighter, you need the failover topology (chapter 04) with a non-vendor fallback.
- **Data residency.** Vendor-managed auth means the vendor stores tokens on your users' behalf. If your jurisdiction prohibits that, vendor-tier-2 is not available for that integration.

## The failover topology with a vendor

When a tier-2 tool has a vendor as its primary implementation, the registry's RoutingPolicy (chapter 03) has choices:

- **Vendor primary, self-hosted fallback.** Platform writes a minimal MCP server as a cold standby. When the vendor is down, the fallback serves the same schema at reduced capability. Useful for tools your users cannot function without.
- **Vendor primary, graceful degradation.** No fallback; a vendor outage returns `tool_unavailable` with an advisory. Useful for tier-2 tools that are nice-to-have.
- **Vendor primary, alternate vendor fallback.** For tools where two vendors offer the same integration. Rare; the compatibility layer is work.

The decision reflects risk and user expectations, not vendor marketing.

## Negotiating the gateway-behind-vendor topology

Some vendors expect to be the agent-facing entity; they ship SDKs that embed their endpoint and want you to call them directly. The platform's operating position:

- **The gateway is non-negotiable.** Agents call the gateway; the gateway calls the vendor; the vendor returns to the gateway; the gateway returns to the agent. If a vendor cannot operate behind a gateway (because their SDK insists on terminating OAuth with the user directly, or because their audit depends on being the first-hop), they do not fit the platform's model for user-scoped tools.
- **Machine-to-machine vendor services may skip the gateway** only when the vendor is doing platform-internal work (evaluation tooling, analytics) and no user data crosses the boundary. These are distinct from user-facing tools and should be handled under separate policies.

A vendor who refuses to operate behind the gateway is a signal to re-check the tier-1-vs-tier-2 decision. You may be looking at a tier-1 candidate (build it) or a tier-3 (refuse).

## Costing the tiers

Build-vs-buy is not purely a security conversation; the cost model matters.

- **Tier 1 cost** = engineering time to author + ops burden to operate + opportunity cost (what you did not build because you built this).
- **Tier 2 cost** = vendor fee + integration time + ongoing vendor-risk management + the diff-cost between vendor's audit shape and yours.
- **Tier 3 cost** = the business ask you refuse (which is real; "no" is not free).

A rule of thumb: when the engineering cost of a tier-1 implementation is less than six months of vendor fees *and* the ongoing ops burden is modest, tier 1 pays back. When the vendor cost is low and the engineering cost high (long tail of integrations), tier 2 is the obvious choice. Tier 3 should be rare; most things have a path to tier 2 if the risk can be bounded.

## Vendor evaluation — the actual checklist

When a vendor arrives (sales ping, product-team request, roadmap item), run this on the first serious conversation:

1. **Technical shape.** Do they speak MCP natively, and at what revision? If not MCP yet, what is their timeline? How much adaptation does the gateway need to proxy them?
2. **Auth model.** Do they run OAuth 2.1, with PKCE, RFC 8707 resource indicators, RFC 7591 DCR? If not, which parts do they skip and why?
3. **Scope vocabulary.** Are their scopes documented, stable, and narrow enough for the user-consent UX you want?
4. **Audit shape.** Can they emit per-invocation records with the tuple from chapter 07? If not, can you compute the record from your side alone?
5. **Data residency and processing.** Where do they run; what do they log; how long do they keep; what is their DPA?
6. **SOC 2 / ISO 27001 / sector-specific certifications.** What they have, when the last audit was, whether they will share the full report.
7. **Vendor-outage response.** What is their SLA? How do they communicate incidents? What is their RTO/RPO?
8. **Vendor-leaves-market scenario.** Who owns the contract if they are acquired, pivot, or dissolve? What happens to tenant tokens if they shut down the service?
9. **Exit cost.** If you need to replace them in six months, what does that look like? Tokens revoked; tools reimplemented; users re-consent; audit records exported.
10. **Commercial.** Licensing model, per-tool vs. per-call vs. flat, enterprise terms available, support tier.

Nothing about this list is MCP-specific; it is standard enterprise vendor due diligence applied to this stack.

## The handoff to mod-413

The build-vs-buy *portfolio* belongs to mod-413 (platform leadership) — it is one of the portfolio decisions a platform team runs explicitly, with a cadence (quarterly review, ad hoc on vendor incidents), with a scoring model, and with an exit path for every tier-2 vendor. What this module hands over:

- A registry whose schema (chapter 03) represents all three tiers consistently.
- A gateway topology where a vendor swap is a routing-policy change, not a code change.
- An audit pipeline (chapter 07) that produces the record independently of whether a vendor was involved.
- A vendor-evaluation checklist that mod-413 operationalises into a portfolio review.

## Summary

- Build-vs-buy at the platform is a portfolio, not a per-tool verdict. Three tiers: own, proxy-through-vendor, refuse.
- The gateway is the agent-facing endpoint in all three tiers. A vendor that refuses to sit behind your gateway is not a fit.
- Vendors (Arcade.dev, Composio, Toolhouse, others) are good at breadth, OAuth-at-scale, consent UX, and refresh hygiene. They are typically not good at your audit shape, your custom policy, your scope narrowing, or your SLO floor.
- Pick tiers with a checklist (Tier 1 triggers, Tier 3 triggers, else Tier 2). Cost it honestly — engineering plus ops plus audit-diff.
- Failover topology with a vendor has three shapes (self-hosted fallback, graceful degradation, alternate vendor). Pick per risk tier and user dependency.

The last chapter in the module turns to the thing that holds the whole plane accountable: compliance audit evidence. Everything previous chapters built — registry, auth, conformance, proxy, vendor topology — is only useful if you can prove, after the fact, who called what, under which scope, in which session, with which inputs and outputs.
