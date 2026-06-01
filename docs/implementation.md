# Implementation

The servers and clients are built with the official **[Python MCP SDK](https://github.com/modelcontextprotocol/python-sdk)** using `FastMCP`, whose decorator API keeps each capability to a few lines. One server is shown end-to-end (Inventory); the others follow the same shape.

## Project layout

```text
mcp-ecommerce-agent/
├── servers/
│   ├── inventory.py      # WMS — stock + reorders
│   ├── orders.py         # OMS — upcoming orders
│   ├── logistics.py      # TMS — drivers + routes
│   ├── returns.py        # RMA
│   ├── claims.py         # claims
│   └── payments.py       # payments (restricted tools)
├── clients/
│   └── probe.py          # minimal programmatic client (testing)
├── common/
│   └── auth.py           # token validation, scope checks (Phase 2+)
└── requirements.txt      # "mcp[cli]"
```

## A server: Inventory

A FastMCP server exposes the three primitives with decorators. Backend calls (`wms`) stand in for the real WMS client.

```python
# servers/inventory.py
from mcp.server.fastmcp import FastMCP
from common import wms  # thin client over the WMS API

mcp = FastMCP("inventory")

# --- Resource: read-only context, app-controlled, no side effects ---
@mcp.resource("inventory://warehouse/{wid}/stock")
def warehouse_stock(wid: str) -> str:
    """Current stock levels for a warehouse, as JSON."""
    return wms.stock_snapshot(warehouse=wid)

# --- Tool: model-proposed action, executed only after human approval ---
@mcp.tool()
def create_reorder(sku: str, quantity: int, warehouse: str) -> str:
    """Place a replenishment purchase order for a SKU at a warehouse."""
    po = wms.create_purchase_order(sku=sku, qty=quantity, warehouse=warehouse)
    return f"Reorder placed: PO #{po.id} ({quantity}× {sku} → {warehouse})"

# --- Prompt: user-controlled workflow surfaced as a slash-command ---
@mcp.prompt()
def low_stock_review(region: str = "all") -> str:
    """Guide the agent through a low-stock reorder review."""
    return (
        f"Review stock for region '{region}'. For each SKU below its reorder "
        "point, cross-check 7-day demand from the orders feed and propose a "
        "reorder quantity. Ask me to approve before placing any order."
    )

if __name__ == "__main__":
    mcp.run()  # stdio transport (Phase 1, local)
```

That is a complete, runnable MCP server: one resource, one tool, one prompt. The docstrings and type hints are not decoration — the SDK turns them into the tool/resource schemas the model sees, so clear descriptions directly improve agent behaviour.

## Wiring it into a host (Phase 1, stdio)

For a desktop host, servers are declared in the host's MCP config and launched as subprocesses over stdio:

```json
{
  "mcpServers": {
    "inventory": { "command": "python", "args": ["servers/inventory.py"] },
    "orders":    { "command": "python", "args": ["servers/orders.py"] },
    "logistics": { "command": "python", "args": ["servers/logistics.py"] }
  }
}
```

The host now exposes the Inventory resources, the `create_reorder` tool, and the `/low_stock_review` prompt to the agent.

## A minimal client (for testing)

You rarely hand-write a client — the host is the client — but a tiny one is useful for probing a server in CI:

```python
# clients/probe.py
import asyncio
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

async def main():
    params = StdioServerParameters(command="python", args=["servers/inventory.py"])
    async with stdio_client(params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()
            print("tools:", [t.name for t in (await session.list_tools()).tools])
            content, _ = await session.read_resource("inventory://warehouse/WH-2/stock")
            print("stock:", content)

asyncio.run(main())
```

## Going remote (Phase 2–3, Streamable HTTP)

The same server runs as a remote service by switching the transport — no business logic changes. This is the deployable form, fronted by OAuth 2.1 (see [Security](security.md)).

```python
if __name__ == "__main__":
    # Served over HTTP; put a reverse proxy + TLS + auth in front.
    mcp.run(transport="streamable-http")
```

## Human-in-the-loop

Side-effecting tools are **proposed by the model and confirmed by a person**. In Phase 1 the host's built-in tool-approval UI handles this. As servers move remote, the same intent is enforced by scopes and an approval step in the host before the tool call is dispatched — the agent can *suggest* `create_reorder`, but it cannot *commit* it unsupervised. High-impact tools (e.g. Payments `release_hold`) are gated by restricted scopes on top of approval.

## Why Python / FastMCP here

- **Density** — a useful server is ~15 lines; the decorators generate the MCP schemas from type hints and docstrings.
- **One codebase, two transports** — `mcp.run()` for local stdio, `mcp.run(transport="streamable-http")` for remote, so the Phase 1 → Phase 2 move is a deployment change, not a rewrite.
- **Standards-accurate** — it's the reference SDK, so the wire protocol, auth hooks, and primitive semantics match the spec other hosts implement.

Next: **[Security](security.md)** — how authorization, scoped access, and transport hardening apply these primitives safely across all three phases.
