# MCP Server

Torrix includes a built-in MCP endpoint at `/mcp`. Connect any MCP-compatible AI assistant directly to your running Torrix instance. No extra setup or source code required.

---

## Verify the endpoint is working

Replace `trxk_your_key_here` with your API key from Settings.

**Mac / Linux:**
```bash
curl -X POST http://localhost:8088/mcp \
  -H "Authorization: Bearer trxk_your_key_here" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"get_dashboard","arguments":{}}}'
```

**Windows (PowerShell):**
```powershell
Invoke-WebRequest http://localhost:8088/mcp `
  -Method POST `
  -Headers @{ Authorization="Bearer trxk_your_key_here"; "Content-Type"="application/json"; Accept="application/json, text/event-stream" } `
  -Body '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"get_dashboard","arguments":{}}}' |
  Select-Object -ExpandProperty Content
```

Expected response:
```
event: message
data: {"result":{"content":[{"type":"text","text":"Total runs: ...\nTotal cost: ..."}]},"jsonrpc":"2.0","id":1}
```

---

## Connect your AI assistant

Get your API key from Settings in the Torrix UI, then follow the instructions for your tool.

### Claude Code

Add to `~/.claude.json` or your project's `.claude/settings.json`:

```json
{
  "mcpServers": {
    "torrix": {
      "type": "http",
      "url": "http://localhost:8088/mcp",
      "headers": {
        "Authorization": "Bearer trxk_your_key_here"
      }
    }
  }
}
```

### Claude Desktop

Add to `~/Library/Application Support/Claude/claude_desktop_config.json` (Mac) or `%APPDATA%\Claude\claude_desktop_config.json` (Windows):

```json
{
  "mcpServers": {
    "torrix": {
      "type": "http",
      "url": "http://localhost:8088/mcp",
      "headers": {
        "Authorization": "Bearer trxk_your_key_here"
      }
    }
  }
}
```

### Cursor / Windsurf / other MCP clients

Use transport type **Streamable HTTP**, URL `http://localhost:8088/mcp`, and add the header `Authorization: Bearer trxk_your_key_here`. Refer to your tool's MCP configuration docs for exact steps.

### n8n (when both n8n and Torrix run in Docker)

Use `host.docker.internal` instead of `localhost`:

Add an **MCP Client** node in your n8n workflow and set:
- Transport: Streamable HTTP
- URL: `http://host.docker.internal:8088/mcp`
- Header: `Authorization: Bearer trxk_your_key_here`

---

## Available tools

| Tool | What it does |
|---|---|
| `get_dashboard` | Total cost, tokens, runs, error count, latency percentiles, top models |
| `list_runs` | Recent runs with optional filters: model, provider, status code |
| `get_run` | Full run details including prompt and response text |
| `get_trace` | All steps in an agent trace with per-step cost and latency |
| `get_session` | All turns in a conversation session with combined cost |
| `compare_runs` | Side-by-side diff of two runs: model, cost, tokens, latency, responses |

---

## Example prompts

Once connected, ask your AI assistant things like:

- "What did I spend on LLM calls today?"
- "Show me failed runs from the last hour"
- "What was the prompt and response for run [id]?"
- "Compare run [id1] and run [id2]"
- "Which model am I using most?"
- "Show me my most expensive runs"

---

## Community edition limits

The MCP server returns up to 100 runs from the last 7 days on Community edition. Pro returns the full 30-day history.
