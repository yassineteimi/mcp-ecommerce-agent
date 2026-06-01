# Architecture

The system has three layers: a **host** running the AI agent, one **MCP client** per connection, and a set of **MCP servers** — one per business domain — each fronting a real backend system. Capabilities are expressed as **Resources** (read context), **Tools** (actions), and **Prompts** (workflows).

## One server per bounded context

Each operational system the supply-chain manager relies on is wrapped by a dedicated MCP server. This keeps credentials, scopes, and blast radius isolated per domain (see [Security](security.md)).

| MCP server | Fronts | Resources (read) | Tools (act) | Prompts |
| --- | --- | --- | --- | --- |
| **Orders** | Order Mgmt System (OMS) | `orders://order/{id}`, upcoming-orders feed | `flag_order`, `split_shipment` | "Late-shipment investigation" |
| **Inventory** | Warehouse Mgmt (WMS) | `inventory://warehouse/{id}/stock` | `create_reorder`, `transfer_stock` | "Low-stock reorder review" |
| **Logistics** | Transport Mgmt (TMS) | `drivers://roster/{region}`, route status | `reroute_driver`, `assign_driver` | "Re-plan a delayed route" |
| **Returns** | Returns/RMA service | `returns://rma/{id}`, returns policy | `approve_return`, `issue_label` | "High-value return approval" |
| **Claims** | Claims service | `claims://claim/{id}` | `open_claim`, `escalate_claim` | "Triage damage claims" |
| **Payments** | Payments/settlement | `payments://order/{id}/status` | `release_hold` *(restricted)* | "Settlement exceptions" |

## System overview

```mermaid
flowchart TB
    subgraph HOST["AI Host — Supply-Chain Assistant"]
        AGENT["LLM agent + orchestration"]
        subgraph CLIENTS["MCP Clients (1 per server)"]
            C1["client"]:::c
            C2["client"]:::c
            C3["client"]:::c
        end
        AGENT --- CLIENTS
    end

    subgraph SERVERS["MCP Servers — one per domain"]
        direction TB
        ORD["**Orders**<br/>Resources · Tools · Prompts"]
        INV["**Inventory**<br/>Resources · Tools · Prompts"]
        LOG["**Logistics**<br/>Resources · Tools · Prompts"]
        RET["**Returns**<br/>Resources · Tools · Prompts"]
        CLM["**Claims**<br/>Resources · Tools · Prompts"]
        PAY["**Payments**<br/>Resources · Tools · Prompts"]
    end

    subgraph BACKENDS["Systems of record"]
        OMS[("OMS")]
        WMS[("WMS")]
        TMS[("TMS")]
        RMA[("Returns")]
        CLD[("Claims DB")]
        PSP[("Payments")]
    end

    AUTH["Authorization Server<br/>company IdP — OAuth 2.1"]

    CLIENTS -- "MCP over stdio (local)<br/>or Streamable HTTP + OAuth (remote)" --> SERVERS
    ORD --> OMS
    INV --> WMS
    LOG --> TMS
    RET --> RMA
    CLM --> CLD
    PAY --> PSP
    SERVERS -. "validate scoped access tokens" .-> AUTH

    classDef c fill:#0b3,stroke:#093,color:#fff;
```

The host speaks MCP to each server through its own client. Locally (Phase 1) servers run as subprocesses over **stdio**; remotely (Phase 2–3) they run as services over **Streamable HTTP** with OAuth 2.1. Each server is the single enforcement point in front of its system of record.

## Tools / Resources / Prompts in flow

A representative workflow — the manager runs the **"Low-stock reorder review"** prompt during the morning briefing:

```mermaid
sequenceDiagram
    actor M as Supply-Chain Manager
    participant H as Host + Agent
    participant INV as Inventory server
    participant ORD as Orders server
    participant WMS as WMS (backend)
    participant HU as Approval (human-in-the-loop)

    M->>H: Run prompt "Low-stock reorder review"
    Note over H: Prompt = user-controlled template
    H->>INV: read resource inventory://warehouse/*/stock
    INV->>WMS: query stock levels
    WMS-->>INV: stock snapshot
    INV-->>H: Resource content (context, no side effects)
    H->>ORD: read resource upcoming-orders feed
    ORD-->>H: demand for next 7 days
    Note over H: Agent reasons over context,<br/>proposes reorders
    H->>M: "Reorder 400 units SKU-123 to WH-2?"
    M->>HU: Approve
    HU->>INV: call tool create_reorder(SKU-123, 400, WH-2)
    Note over INV: Tool = model-proposed action,<br/>executed only after approval
    INV->>WMS: place purchase order
    WMS-->>INV: PO #4471 created
    INV-->>H: Tool result
    H-->>M: "Reorder placed: PO #4471"
```

**Why this split matters:**

- **Resources** are read-only and **application-controlled** — the host decides what context to pull in; they never mutate state, so they're safe to fetch liberally.
- **Tools** are **model-proposed** but, for any side-effecting action, **human-approved**. The agent suggests `create_reorder`; a person authorizes it. High-impact tools (e.g. Payments `release_hold`) are restricted by scope regardless.
- **Prompts** are **user-controlled** workflows surfaced as slash-commands, encoding best-practice operating procedures so results are consistent across operators.

## Phase evolution

```mermaid
flowchart LR
    subgraph PH1["Phase 1 — Internal (stdio, local)"]
        h1["Assistant host"] --- s1["Domain servers<br/>read-mostly"]
    end
    subgraph PH2["Phase 2 — Mature (Streamable HTTP)"]
        h2["Host + guardrails<br/>audit · evals"] --- s2["Servers + write tools<br/>OAuth 2.1"]
    end
    subgraph PH3["Phase 3 — Customer-facing"]
        h3["Customer app host"] --- s3["Curated, multi-tenant<br/>scoped servers"]
    end
    PH1 --> PH2 --> PH3
```

- **Phase 1** — servers run locally beside the host over stdio; the manager gets grounded answers and approves every action.
- **Phase 2** — servers are deployed as remote services with OAuth 2.1, write tools enabled behind guardrails, with full audit and evaluation.
- **Phase 3** — a curated subset (order tracking, returns initiation) is exposed to the customer experience, with per-customer (multi-tenant) isolation enforced server-side.

See **[Implementation](implementation.md)** for how a server like Inventory is actually built, and **[Security](security.md)** for the authorization and transport model behind these phases.
