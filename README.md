# MCP for E-Commerce — Business-Context AI Agents

[![Docs](https://img.shields.io/badge/docs-GitHub%20Pages-blue)](https://yassineteimi.github.io/mcp-ecommerce-agent/)
[![Deploy docs](https://github.com/yassineteimi/mcp-ecommerce-agent/actions/workflows/ci.yml/badge.svg)](https://github.com/yassineteimi/mcp-ecommerce-agent/actions/workflows/ci.yml)

A proof of concept that uses the **Model Context Protocol (MCP)** to give an e-commerce company's AI agents the **business context they need to run operations** — live orders, inventory, logistics, returns, claims, and payments — safely and at scale.

**📖 Full documentation:** **<https://yassineteimi.github.io/mcp-ecommerce-agent/>**

## The idea

A general LLM doesn't know that *order #88213 is late because its driver called in sick*. MCP connects AI agents to real business systems through one open standard, turning a brittle **N×M** web of integrations into a reusable **N+M** model: wrap each system once as an **MCP server**, and any agent can use it.

The PoC follows a deliberate maturity path:

1. **Internal first** — a **supply-chain manager** gets an assistant with governed, read-mostly access to every fulfilment system; the agent proposes actions, a human approves them.
2. **Mature** — enable trusted write actions (reorder, reroute, approve return) behind guardrails, audit, and evaluation.
3. **Customer-facing** — expose a curated, per-tenant-isolated subset (order tracking, returns) into the customer experience.

## How it works

- **Servers, one per domain** — Orders, Inventory, Logistics, Returns, Claims, Payments — each fronting a system of record with isolated credentials.
- **Three MCP primitives** — **Resources** (read-only context), **Tools** (human-approved actions), **Prompts** (reusable operating procedures).
- **Built with the Python MCP SDK (`FastMCP`)** — a complete server is ~15 lines; the same code runs locally over stdio or remotely over Streamable HTTP.
- **Secure by design** — OAuth 2.1 authorization, role-based scoped access, per-server least privilege, transport hardening, and per-tenant isolation for the customer phase.

## Documentation

| Page | Contents |
| --- | --- |
| [Overview](https://yassineteimi.github.io/mcp-ecommerce-agent/) | What MCP is and why it matters for AI-agent integration |
| [Architecture](https://yassineteimi.github.io/mcp-ecommerce-agent/architecture/) | Servers, clients, and the Tools/Resources/Prompts flow (with diagrams) |
| [Implementation](https://yassineteimi.github.io/mcp-ecommerce-agent/implementation/) | How the servers and clients are built |
| [Security](https://yassineteimi.github.io/mcp-ecommerce-agent/security/) | Authorization, scoped access, transport hardening |

Docs are built with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/) and deployed to GitHub Pages by [`.github/workflows/ci.yml`](.github/workflows/ci.yml) on every push to `main`.

## Status

A **design-led proof of concept**: the architecture, code patterns, and security model are production-shaped and standards-accurate, illustrated against a representative e-commerce domain. Learn more about MCP at [modelcontextprotocol.io](https://modelcontextprotocol.io).
