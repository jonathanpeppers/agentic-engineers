---
layout: post
title: "Your Grafana, Now Talks to Agents: The Azure Managed Grafana Remote MCP Server"
author: "Wuyi Weng"
excerpt: "Every Azure Managed Grafana instance now ships with a built-in remote MCP endpoint. Point Copilot, Claude, or a Foundry agent at it and they can search dashboards, query Prometheus, run KQL against Kusto, pull Application Insights traces, and run automated health checks — no plugins to install, no servers to host."
tags:
  - azure
  - grafana
  - mcp
  - copilot
  - observability
  - ai-agents
---

# Your Grafana, Now Talks to Agents: The Azure Managed Grafana Remote MCP Server

*May 2026 — Wuyi Weng*

If you've spent any time wiring AI agents into observability stacks, you know the drill: stand up an MCP server, give it credentials, keep it running somewhere, and pray that the next Grafana upgrade doesn't break it. The new **Azure Managed Grafana (AMG) remote MCP server** removes all of that. Every AMG instance in the Azure public cloud now exposes a built-in MCP endpoint at:

```
https://<grafana-endpoint>/api/azure-mcp
```

No container to deploy. No sidecar. No long-lived secret on your laptop. The same Grafana you already pay for is now an MCP server your agents can talk to — using the same auth you already use for Grafana itself.

**Docs:** [learn.microsoft.com — Grafana MCP server](https://learn.microsoft.com/en-us/azure/managed-grafana/grafana-mcp-server) · [Azure/azure-managed-grafana — amg-mcp.md](https://github.com/Azure/azure-managed-grafana/blob/main/amg-mcp.md)

## Why a remote MCP server (and not "just install the plugin")

The MCP ecosystem skews local: you `npm install` a server, drop a stanza into `mcp.json`, and run it next to your editor. That works great for filesystems and git, but observability data is the opposite of local — it lives in Prometheus, Kusto, Application Insights, Azure Monitor, SQL, all behind their own auth boundaries. Stitching that together in a process on your laptop means:

- A long-lived service principal or PAT sitting in a config file
- A locally-run server that has to be kept up to date
- A different setup story for every teammate and every agent host (VS Code, Claude Code, Foundry, …)

The AMG remote MCP server flips this. The server is **already running inside your Grafana instance**, behind the same network and identity perimeter your dashboards already trust. Your agent just needs a URL and a token — or, if you're in VS Code or Visual Studio with Copilot, **just the URL**, because OAuth with Microsoft Entra ID is handled for you.

## What the agent actually gets

This isn't "search dashboards and call it a day." AMG-MCP exposes a serious surface area — 25+ tools spanning dashboards, time-series queries, Azure resource telemetry, distributed traces, and Azure management-plane data. A few highlights:

**Dashboards (read and write):**
- `amgmcp_dashboard_search` — find dashboards by query
- `amgmcp_dashboard_inspect` — summary, panel queries, or surgical JSONPath reads
- `amgmcp_dashboard_update` — full create/replace, or targeted JSONPath patch ops (add/replace/remove)

**Prometheus / metrics:**
- `amgmcp_prometheus_query`, `amgmcp_prometheus_query_histogram`
- `amgmcp_prometheus_list_metric_names`, `amgmcp_prometheus_list_label_names`

**Azure Monitor & friends, through Grafana data sources:**
- `amgmcp_query_resource_metric`, `amgmcp_query_resource_metric_definition`, `amgmcp_query_resource_log`
- `amgmcp_query_resource_graph`, `amgmcp_query_azure_subscriptions`
- `amgmcp_query_activity_log`, `amgmcp_cost_analysis`
- `amgmcp_query_resource_health`, `amgmcp_query_resource_health_events`

**Application Insights (including the GenAI bits):**
- `amgmcp_insights_get_failures` — failed requests, dependencies, exceptions
- `amgmcp_query_application_insights_trace` — aggregates trace data across multiple AI instances
- `amgmcp_insights_get_agents` — agent invocations, token usage, latency, following the *OpenTelemetry for Generative AI* semantic conventions (so it works with OpenAI Python SDK, Microsoft Agent Framework, AI Foundry / Agent Toolkits, Google ADK, LangChain / LangGraph, …)

**Kusto and SQL:**
- `amgmcp_kusto_get_metadata`, `amgmcp_kusto_query`
- `amgmcp_mssql_get_metadata`, `amgmcp_mssql_query`

**And my favorite: `amgmcp_pulse_check`.** One tool call, and the server fans out automated health checks across PostgreSQL, Cosmos DB, AKS, Storage, Key Vault, VMs, SQL Database, App Service Plans, Redis, and Logic Apps — flagging high CPU, RU saturation, memory pressure, degraded availability, and similar pathologies, then handing the agent back a *prioritized* summary instead of a wall of metrics.

The full list lives in the [docs](https://learn.microsoft.com/en-us/azure/managed-grafana/grafana-mcp-server#available-mcp-tools).

## Auth: pick the boring option that fits

AMG-MCP supports three authentication modes. They map cleanly to "what kind of caller is this":

1. **OAuth with Microsoft Entra ID** — interactive sign-in, handled by the client. Use this from **VS Code with GitHub Copilot** or **Visual Studio with GitHub Copilot**. No token in your config file.
2. **Microsoft Entra ID token (managed identity / service principal)** — for unattended callers like an Azure AI Foundry agent or any service running with a managed identity. Acquire a token for the AMG audience `ce34e7e5-485f-4d76-964f-b3d2b16d1e4f` and pass it as a bearer.
3. **Grafana service account token (`glsa_xxx`)** — works with any MCP client. Best when you don't have OAuth support in the client (e.g., Claude Code, OpenClaw).

A useful mental model: AMG-MCP doesn't introduce a *new* auth surface. It accepts whatever the underlying Grafana already accepts.

## Wiring it up

### VS Code or Visual Studio + GitHub Copilot (OAuth — the easy path)

```jsonc
{
  "servers": {
    "my-grafana-mcp-server": {
      "type": "http",
      "url": "https://<grafana-endpoint>/api/azure-mcp"
    }
  }
}
```

That's the entire configuration. When Copilot first connects, it pops the Entra ID sign-in flow. No tokens on disk.

### Claude Code (service account token)

```jsonc
{
  "my-grafana-mcp-server": {
    "disabled": false,
    "timeout": 60,
    "type": "streamableHttp",
    "url": "https://<grafana-endpoint>/api/azure-mcp",
    "headers": {
      "Authorization": "Bearer glsa_xxxxxxxxxxxxxxxxxxxxxxxx_xxxxxxx"
    }
  }
}
```

Generate the `glsa_` token from **Grafana → Administration → Service accounts**, scoped to the role you actually want the agent to have (Viewer is a good default; Editor only if you want it patching dashboards).

### Azure AI Foundry agent (managed identity)

Grant the agent's managed identity an appropriate Grafana role on the AMG instance, then have it acquire a token for the AMG audience and pass it as a bearer:

```bash
az account get-access-token \
  --resource ce34e7e5-485f-4d76-964f-b3d2b16d1e4f \
  --query accessToken -o tsv
```

```jsonc
{
  "my-grafana-mcp-server": {
    "type": "streamableHttp",
    "url": "https://<grafana-endpoint>/api/azure-mcp",
    "headers": {
      "Authorization": "Bearer <entra-id-token-from-managed-identity>"
    }
  }
}
```

There's a full step-by-step Foundry sample in the [`azure-managed-grafana` repo](https://github.com/Azure/azure-managed-grafana/blob/main/samples/3-remote-mcp-foundry-agent/foundry-agent-amg-mcp.md).

## What you can actually *do* with this

The shape of the tool surface tells you what the team had in mind. A few prompts that "just work" once the MCP server is connected:

- *"Run a pulse check across my production subscription and show me the top three things I should look at right now."* → `amgmcp_pulse_check` returns a prioritized list; the agent narrates it.
- *"Why did p99 latency on the checkout service spike between 14:00 and 14:30 UTC?"* → the agent uses `amgmcp_prometheus_query_histogram` for the histogram math and `amgmcp_query_application_insights_trace` to pull the slow traces.
- *"Find the dashboard for AKS cluster health and add a panel for pod restarts in the last hour."* → `amgmcp_dashboard_search` → `amgmcp_dashboard_inspect` → `amgmcp_dashboard_update` with a JSONPath patch.
- *"How much did my AI Foundry agents cost last week, broken down by model?"* → `amgmcp_insights_get_agents` for token usage, `amgmcp_cost_analysis` for the billing side.
- *"Were any of my Cosmos DB accounts unhealthy in the last 24 hours?"* → `amgmcp_query_resource_health_events` plus `amgmcp_pulse_check`.

The interesting thing isn't any single one of these — it's that the *same* agent can move fluidly between them, because the MCP server already knows how to talk to every data source the Grafana instance is connected to.

## Limitations worth knowing

A couple of things that aren't there yet — both called out in the docs:

- **Azure public cloud only.** Sovereign clouds (Azure Government, Azure China) don't have the AMG-MCP endpoint yet.
- **No private endpoint connectivity.** If your Grafana is locked behind a private endpoint, MCP traffic can't reach it today.

If either bites you, [open an issue on the AMG repo](https://aka.ms/managed-grafana/issues) — the team is tracking both.

## The bigger picture

Observability has always been the part of the stack where agents look smartest *and* fail hardest. Smartest because there's a clean question-and-answer loop ("what changed at 14:00?"). Hardest because the data is scattered across systems, behind auth, and rarely in a shape an LLM can reason about directly.

A built-in remote MCP server inside Grafana is, quietly, a big deal: it collapses "agent + observability" from a multi-week integration project into a single config block. You stop writing glue and start asking questions.

If you have an AMG instance, point Copilot at `https://<your-grafana>/api/azure-mcp` today and see how far it gets. My bet is: further than you expect.

**Links:**
- Microsoft Learn: [Azure Managed Grafana MCP server](https://learn.microsoft.com/en-us/azure/managed-grafana/grafana-mcp-server)
- GitHub: [Azure/azure-managed-grafana — amg-mcp.md](https://github.com/Azure/azure-managed-grafana/blob/main/amg-mcp.md)
- Foundry sample: [foundry-agent-amg-mcp.md](https://github.com/Azure/azure-managed-grafana/blob/main/samples/3-remote-mcp-foundry-agent/foundry-agent-amg-mcp.md)
- Issues: [aka.ms/managed-grafana/issues](https://aka.ms/managed-grafana/issues)
