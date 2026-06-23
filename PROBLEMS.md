---
title: Agent Registry - Problems to Solve
created: 2026-06-23
tags: [agentic-registries, product-strategy, agent-governance]
---

# Agent Registry: Problems to Solve (by Priority)

Problems extracted from FSI Roundtable (Jun 22, 2026), Adel/Justin sync (Jun 22, 2026), and competitive landscape research (AWS Agent Registry, Google Agent Registry, Microsoft Entra Agent ID).

## How to read this

Each problem has:
- **Source**: who raised it and when
- **Mock coverage**: which mock version addresses it
- **Competitive signal**: whether hyperscalers have shipped a solution
- **Priority rationale**: why it sits where it does

## Tier 1: Core ("What do we have? Is it safe? Where do I route?")

These are table-stakes for any agent registry. Without them, you have no registry, just a spreadsheet. This is the most important tier to ship first.

### P1: Duplicate agents across teams
> "Three teams built the same thing" (internal observation)

**The problem**: Multiple teams independently build agents for the same capability (e.g., PDF extraction, code review) because there is no way to discover what already exists. Wasted engineering and divergent implementations.

**Solution**: Capability-based search. "Find agents that can process invoices" returns matches across all teams before anyone writes code.

**Mock coverage**: v1, v2
**Competitive**: AWS (semantic + keyword search), Google (A2A AgentCard skill extraction), Microsoft (Graph API search)

### P2: What's running where?
> "What is the scale of numbers of things that we need to keep track of? Assuming this is cross cluster, we'll need some indication of how often you refresh this data." (Justin, Jun 22)

**The problem**: No single view of all deployed agents. Platform teams cannot answer basic questions: how many agents are running? On which clusters? In what state?

**Solution**: Runtime inventory table with agent metadata, cluster location, health status, last-seen timestamps.

**Mock coverage**: v1 (single cluster), v2 (multi-cluster with cluster selector)
**Competitive**: All three hyperscalers ship this. AWS tracks across accounts, Google across GCP projects, Microsoft across tenants.

### P3: Is it safe?
> "Are we considering validated agents, like validated models? Where we go an extra step to make sure that they're good." (Justin, Jun 22)
> "Ensuring agents don't go rogue... the boundary of trust for agents." (Bud, FSI Roundtable)

**The problem**: No way to see at a glance whether an agent is sandboxed, healthy, or degraded. Platform teams discover problems only after incidents.

**Solution**: Health monitoring (30-day sparkline), sandbox status (sandboxed/unsandboxed), deployment status (healthy/stale/shadow) with visual alerts for degraded agents.

**Mock coverage**: v1, v2
**Competitive**: Google (Agent Gateway health), AWS (AgentCore Observability), Microsoft (Defender risk columns)

### P4: Shadow IT for agents
> "Discovered by runtime scan. No registration. No sandbox." (mock data)
> "A registry is also very important to allow discovery and to allow things when you get vulnerabilities." (Dino, FSI Roundtable)

**The problem**: Agents deployed without registration evade governance. They have no identity, no sandbox, no owner. A "Rogue Trading Bot" in the default namespace with trading capabilities is an existential risk for FSI.

**Solution**: Runtime scan detection that surfaces unregistered agents as "shadow" entries with critical alerts. One-click path to register and apply sandbox.

**Mock coverage**: v1, v2
**Competitive**: None of the hyperscalers have shipped shadow detection. This is a differentiation opportunity.

### P5: Where do I route?
> "Agent registry for discovery... endpoint + protocol." (internal)

**The problem**: Agent-to-agent communication requires knowing the endpoint URL, supported protocols (A2A, MCP), and authentication method. Without a registry, every integration is a manual coordination exercise.

**Solution**: Endpoint, protocol badges (A2A/MCP), auth method, and cluster/namespace in every registry entry.

**Mock coverage**: v1, v2
**Competitive**: AWS (A2A AgentCard + MCP endpoint), Google (A2A auto-extraction), Microsoft (Azure API Center)

## Tier 2: Governance ("Who's accountable? What's allowed? At what cost?")

These are the problems that FSI customers raised as blockers to production adoption. Without them, agents stay in staging.

### P6: Vulnerability blast radius
> "If I have a vulnerability for certain piece of software or tool or even an LLM, I need to understand where my agents deployed, on the edge, on the cloud, on premise." (Dino, FSI Roundtable)

**The problem**: A CVE is filed against a dependency (e.g., langchain 0.3.1). There is no way to answer: "which agents are affected?" across the fleet. Every cluster must be manually inspected.

**Solution**: CVE association per agent (component, severity, fix). Filter by CVE severity. "Show me all agents affected by CVE-2026-3821" as a single query.

**Mock coverage**: v2 only
**Competitive**: Microsoft (Defender integration), AWS (partial via ECR scanning). Google: not yet.

### P7: Who gets fired? (Accountability)
> "When an agent screws up, who gets fired? There's no accountability there. You can't hold an agent accountable." (Bud, FSI Roundtable)
> "The agency theory says the agents are accountable to the board and the shareholders. At the end of the day, the CEO is accountable." (Jamil, FSI Roundtable)

**The problem**: Agents act autonomously but have no accountability chain. If an agent deletes a production database (the "nine seconds" incident), there is no technical trail linking the action to a responsible human.

**Solution**: Identity delegation chain: Human sponsor delegated to Agent via identity exchange (OIDC, SPIFFE). Every agent has a named sponsor. Shadow agents without sponsors are flagged as critical.

**Mock coverage**: v2 only
**Competitive**: Microsoft Entra Agent ID (most mature: agent-as-identity-with-human-sponsor). Google (SPIFFE-based). AWS (IAM/JWT hybrid).

### P8: What's allowed? (Policy enforcement)
> "Putting a sandbox in that concrete box or concrete room to limit the blast radius." (Bud, FSI Roundtable)
> "Sandbox governance: hard boundaries (OpenShell-style) vs soft boundaries (hooks/permissions)." (Adel, summary)

**The problem**: Even sandboxed agents need visible policies. What can this agent access on the network? Can it write to the filesystem? Is PII exfiltration blocked? These policies exist at the infrastructure level but are invisible to platform operators.

**Solution**: Policy cards per agent showing enforced policies (network-egress-deny, fs-readonly, no-pii-exfil, rate-limit). Attach policies at registration time.

**Mock coverage**: v2 only
**Competitive**: Google (Agent Gateway natural-language policies), AWS (IAM policies), Microsoft (Conditional Access)

### P9: Cross-cluster blind spots
> "Assuming this is cross-cluster, we'll need to have some indication of how often you refresh this data." (Justin, Jun 22)
> "A registry means the agent can also be running in different runtimes. It could be running on AKS, EKS, a VM, Azure Foundry." (Dino, FSI Roundtable)

**The problem**: Agents run across multiple clusters, cloud providers, and edge locations. A single-cluster view misses the full picture. The FSI roundtable surfaced agents on OCP, EKS, and edge devices in the same organization.

**Solution**: Multi-cluster selector with federated view. Stats, filters, and search operate across all clusters or drill into one.

**Mock coverage**: v2 only (4 clusters: ocp1, ocp2, aws-eks, edge-fleet)
**Competitive**: AWS (cross-account), Microsoft (cross-tenant via M365 Admin Center), Google (cross-project)

### P10: Token burn / cost
> "I don't think there's a really good understanding of how much token churn agents will create. The tokenomics type of conversation and minimizing tokens will put some spotlight on agents." (FSI participant, Roundtable)

**The problem**: Agents consume tokens at unpredictable rates. Without per-agent cost visibility, platform teams cannot budget, optimize, or chargeback. A single agent doing 2.1M tokens/day at $6/day is invisible until the monthly bill arrives.

**Solution**: Tokens/day and cost/day per agent. Aggregate cost in stats bar. Monthly projection in detail panel.

**Mock coverage**: v2 only
**Competitive**: None of the hyperscalers show per-agent token cost in the registry. This is typically in a separate billing/observability tool.

## Tier 3: Scale ("How do I manage 100+ agents?")

These matter at scale but are not blockers for initial adoption.

### P11: Catalog vs Registry confusion
> "Catalog is more like a hugging face for my agents. Registry represents more of a runtime, what's running where, like MLflow." (Dino, FSI Roundtable)

**The problem**: Teams conflate pre-deployment templates (catalog) with runtime inventory (registry). This creates confusion about what "registering an agent" means: making it available as a template or declaring it as a running instance.

**Solution**: Separate Catalog tab (curated templates with "Deploy" button) from Registry tab (runtime inventory). Deploy promotes from catalog to registry.

**Mock coverage**: v2 only (Catalog tab with 6 templates)
**Competitive**: Google (Agent Garden = catalog, Agent Registry = runtime), AWS (distinct concepts), Microsoft (M365 store vs Entra registry)

### P12: Agent sprawl (topology)
> "Agent-to-agent relationships, delegation chains." (gap analysis)

**The problem**: As agent count grows, understanding which agents depend on, validate, or feed into other agents becomes impossible from a flat table.

**Solution**: Topology view showing agent relationships (validates, audits, feeds-into, queries, consumes) with cluster coloring.

**Mock coverage**: v2 only (basic SVG topology)
**Competitive**: None of the hyperscalers ship a topology view in the registry. Microsoft shows it in Copilot Analytics.

### P13: CMDB integration
> "I forgot what we call the TDAR or asset inventory or CMDB. Does the agent get registered as an app? Does it get its own code?" (Dino, FSI Roundtable)

**The problem**: Enterprise asset inventories (CMDB, ServiceNow) need to track agents alongside traditional applications. Without integration, agents are invisible to ITSM processes.

**Solution**: API-first registry with webhooks to push registration events to CMDB. Agent metadata includes app code association.

**Mock coverage**: Not addressed in either mock
**Competitive**: Microsoft (Graph API export), AWS (EventBridge integration). Google: not yet.

### P14: API-first (programmatic access)
> "Both should also have API enablement. So I can access certain APIs that are exposed for both at the registry level and the catalog level." (Dino, FSI Roundtable)

**The problem**: The registry is only useful as a UI. Platform automation, CI/CD pipelines, and other agents cannot query the registry programmatically.

**Solution**: REST/gRPC API for registry CRUD. MCP endpoint so agents can discover other agents via standard MCP calls.

**Mock coverage**: Not addressed in either mock (UI only)
**Competitive**: AWS (registry is an MCP endpoint natively), Google (REST API), Microsoft (Graph API)

## Coverage Matrix

| Problem | v1 Mock | v2 Mock | Priority | Source |
|---------|---------|---------|----------|--------|
| P1: Duplicate agents | Yes | Yes | Tier 1 | Internal |
| P2: What's running where | Partial (1 cluster) | Yes (4 clusters) | Tier 1 | Justin |
| P3: Is it safe | Yes | Yes | Tier 1 | Justin, Bud |
| P4: Shadow IT | Yes | Yes | Tier 1 | Internal |
| P5: Where do I route | Yes | Yes | Tier 1 | Internal |
| P6: Vulnerability blast radius | No | Yes | Tier 2 | Dino (FSI) |
| P7: Accountability chain | No | Yes | Tier 2 | Bud, Jamil (FSI) |
| P8: Policy enforcement | No | Yes | Tier 2 | Bud (FSI) |
| P9: Cross-cluster | No | Yes | Tier 2 | Justin, Dino |
| P10: Token cost | No | Yes | Tier 2 | FSI participant |
| P11: Catalog vs Registry | No | Yes | Tier 3 | Dino (FSI) |
| P12: Topology | No | Yes | Tier 3 | Gap analysis |
| P13: CMDB integration | No | No | Tier 3 | Dino (FSI) |
| P14: API-first | No | No | Tier 3 | Dino (FSI) |

## Recommendation

Ship Tier 1 first. It is the minimum viable registry. Every hyperscaler has it. Shadow detection (P4) is the only Tier 1 capability where we have a differentiation opportunity.

Tier 2 is what FSI customers need to move agents from staging to production. Identity/accountability (P7) and CVE tracking (P6) were the most emotionally charged topics in the roundtable. Policy visibility (P8) connects directly to the OpenShell sandbox work.

Tier 3 is scale tooling. Build it when customers have 50+ agents.

## Related

- [[agent-registry-mock]] - v1 mock (Tier 1 only)
- [[agent-registry-v2]] - v2 mock (Tier 1 + Tier 2 + partial Tier 3)
