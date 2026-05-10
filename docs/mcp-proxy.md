# MCP Tool Proxy

Torrix can act as a transparent proxy for any HTTP-based MCP server. Route your MCP client through Torrix and every `tools/call` invocation is logged with the tool name, arguments, result, latency, and status. Handshake messages (`initialize`, `tools/list`, `ping`) are forwarded silently without logging.

Runs appear in the dashboard with source **MCP Proxy**, linked to any parent LLM run via trace ID.

## Endpoint

```
POST http://your-torrix-host:8088/mcp-proxy
Content-Type: application/json
Authorization: Bearer trxk_...
x-target-mcp-url: https://your-mcp-server.com/mcp
```

Torrix forwards the JSON-RPC body to the target URL and returns the response transparently.

## Headers

| Header | Required | Description |
|---|---|---|
| `Authorization` | Yes | `Bearer trxk_...` Torrix API key |
| `x-target-mcp-url` | Yes | The actual MCP server URL to forward to |
| `x-upstream-authorization` | No | Forwarded as `Authorization` to the upstream MCP server |
| `x-torrix-trace` | No | Trace ID to group tool calls with a parent LLM run |
| `x-torrix-parent-run-id` | No | Run ID of the parent LLM call |
| `x-torrix-session` | No | Session ID for conversation grouping |
| `x-torrix-project-id` | No | Project to assign runs to |
| `x-agent-name` | No | Agent identifier shown in the dashboard |

## What gets logged

For every `tools/call`:

- **Run**: tool name, latency, status (200 or 500), trace/session/project linkage
- **Event (tool_call)**: tool name and arguments JSON
- **Event (response)**: tool result content JSON
- **Event (error)**: error details if the call failed

All other JSON-RPC methods pass through without logging.

---

## Claude Desktop setup

Claude Desktop supports HTTP MCP transport. Add Torrix as a proxy in `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "my-tool-via-torrix": {
      "command": "npx",
      "args": ["-y", "mcp-remote", "http://localhost:8088/mcp-proxy"],
      "env": {
        "MCP_REMOTE_HEADERS": "Authorization=Bearer trxk_...,x-target-mcp-url=https://your-mcp-server.com/mcp"
      }
    }
  }
}
```

Or if using a direct HTTP MCP client:

```json
{
  "mcpServers": {
    "my-tool-via-torrix": {
      "url": "http://localhost:8088/mcp-proxy",
      "headers": {
        "Authorization": "Bearer trxk_...",
        "x-target-mcp-url": "https://your-mcp-server.com/mcp"
      }
    }
  }
}
```

---

## Cursor setup

In Cursor settings, add an MCP server pointing at Torrix:

```json
{
  "mcpServers": {
    "my-tool-via-torrix": {
      "url": "http://localhost:8088/mcp-proxy",
      "headers": {
        "Authorization": "Bearer trxk_...",
        "x-target-mcp-url": "https://your-mcp-server.com/mcp"
      }
    }
  }
}
```

---

## Python agent example

```python
import requests

TORRIX = "http://localhost:8088/mcp-proxy"
HEADERS = {
    "Authorization": "Bearer trxk_your_key_here",
    "x-target-mcp-url": "https://your-mcp-server.com/mcp",
    "x-torrix-trace": "trace-abc123",      # optional: links to parent LLM run
    "Content-Type": "application/json",
}

# Handshake (not logged)
requests.post(TORRIX, headers=HEADERS, json={
    "jsonrpc": "2.0", "id": 1, "method": "initialize",
    "params": {"protocolVersion": "2024-11-05", "capabilities": {}, "clientInfo": {"name": "my-agent", "version": "1.0"}}
})

# Tool call (logged in Torrix)
response = requests.post(TORRIX, headers=HEADERS, json={
    "jsonrpc": "2.0", "id": 2, "method": "tools/call",
    "params": {"name": "search", "arguments": {"query": "latest AI news"}}
})
print(response.json())
```

---

## Linking tool calls to LLM runs

Pass the LLM run's trace ID as `x-torrix-trace` and the run ID as `x-torrix-parent-run-id`. Tool calls will appear nested under the parent LLM run in the trace view.

```python
# After making an LLM call through Torrix proxy
llm_trace_id = "trace-abc123"
llm_run_id   = "run-xyz789"

mcp_headers = {
    **HEADERS,
    "x-torrix-trace": llm_trace_id,
    "x-torrix-parent-run-id": llm_run_id,
}
```

---

## Troubleshooting

**Tool calls not appearing in dashboard**
Filter by source "MCP Proxy" in the runs list. Only `tools/call` methods are logged; other methods are forwarded silently.

**401 Unauthorized**
Check that the Torrix API key in `Authorization` is valid. This is your Torrix key, not the upstream MCP server key.

**Upstream MCP server returns 401**
Pass the upstream server's API key via `x-upstream-authorization`. Torrix forwards it as the `Authorization` header to the target.

**stdio MCP transport is not supported**
This proxy handles HTTP JSON-RPC transport only. For stdio-based MCP servers, wrap the stdio process with an HTTP adapter (such as `mcp-remote` or a thin FastAPI wrapper) before routing through Torrix.
