# MCP Server Development for Claude

## The Problem It Solves

LLMs can reason but can't act on live data or external systems. MCP gives Claude a standard contract to discover and call tools you build — turning Claude into an agent that can query APIs, databases, and services you control.

---

## Core Concepts

---

### 1. MCP Protocol Architecture

**Why:** Claude needs to discover what your server can do before it can use it. The handshake negotiates protocol version and capabilities so both sides agree on what's supported.

**Mental model:** Like a gRPC SETTINGS frame — both sides declare capabilities, negotiate a version, then lock in. Resources = ConfigMaps Claude reads. Tools = Jobs Claude executes. Prompts = template manifests Claude offers users.

**Example:**

```text
Client → {"method":"initialize","params":{"protocolVersion":"2025-11-25",...}}
Server → {"result":{"protocolVersion":"2025-11-25","capabilities":{"tools":{}}}}
Client → {"method":"tools/list"}
Server → {"result":{"tools":[{"name":"find_nearest_airport",...}]}}
```

---

### 2. stdio Process Model

**Why:** Claude spawns your server as a child process and talks to it exclusively over stdin/stdout. The pipe IS the protocol — nothing else exists.

**Mental model:** Like `kubectl exec` into a pod — once connected, the shell's stdin/stdout is the only channel. Any stray write to stdout (`print`, `log`) injects garbage into the JSON-RPC stream and crashes the session silently.

**Example:**

```python
# All logging MUST go to file — never stdout
logging.handlers.RotatingFileHandler("~/.myserver/audit.log")
logger.propagate = False  # never bubble to root logger / stdout

# Entry point wraps stdio
async with stdio_server() as (read_stream, write_stream):
    await server.run(read_stream, write_stream, ...)
```

---

### 3. Tool Definition + JSON Schema

**Why:** Claude has no built-in knowledge of your server. It calls `tools/list` at startup and uses your definitions to decide when and how to invoke your tools.

**Mental model:** Like a Kubernetes CRD — the schema defines the shape, the description is the annotation that tells operators what it does and when to use it.

**Example:**

```python
@server.list_tools()
async def list_tools() -> list[types.Tool]:
    return [types.Tool(
        name="find_nearest_airport",
        description="Find the nearest airport to a city. Accepts city name or coordinates.",
        inputSchema={
            "type": "object",
            "properties": {"location": {"type": "string"}},
            "required": ["location"],
        },
    )]
```

---

### 4. Claude Desktop Integration

**Why:** Claude Desktop doesn't know your server exists until you register it. Registration = a config entry that tells Claude how to spawn your process.

**Mental model:** Like a Kubernetes Deployment manifest — the binary is your container image, the config is the spec.

**Example:**

```json
// ~/Library/Application Support/Claude/claude_desktop_config.json
{
  "mcpServers": {
    "nearest-airport": {
      "command": "/Users/you/venvs/airport/bin/python",
      "args": ["/Users/you/.airport-mcp/server.py"],
      "env": { "GOOGLE_MAPS_API_KEY": "..." }
    }
  }
}
```

---

### 5. Tool Design for Agentic Claude

**Why:** Claude reasons against your tool definition to decide whether to call it, what to pass, and what to do with the result. Vague descriptions = wrong calls. Ambiguous errors = broken chains.

**Mental model:** Like writing a Kubernetes operator contract — you're not writing code that runs, you're writing a spec Claude reasons against.

**Example:**

```python
# Empty result = valid input, no data → Claude says "no airports here"
return "No airports found near 'Remote Island'."

# Clean error = bad input → Claude asks user to fix it
return "❌ Could not find a recognised location matching 'asdfxyz'. Provide a valid city."

# Never return raw exceptions → Claude can't reason about line numbers
# raise KeyError("results")  ← DON'T
```

---

### 6. Streamable HTTP Transport

**Why:** stdio is process-scoped (one Claude instance, one server). HTTP transport lets you run the server centrally and share it across multiple clients — like moving from a sidecar to a shared microservice.

**Mental model:** Like switching from a sidecar container to a Deployment behind a Service. Same logic, network-accessible, horizontally scalable.

**Example:**

```json
// Claude config for HTTP transport
{
  "mcpServers": {
    "nearest-airport": {
      "type": "http",
      "url": "https://your-server.internal/mcp"
    }
  }
}
```

```python
# Server side — use FastMCP with HTTP transport instead of stdio
mcp = FastMCP("nearest-airport")
mcp.run(transport="streamable-http")  # listens on HTTP
```

---

## Gotchas

1. **Never write to stdout in an stdio server.** `print()`, uncaught exceptions to stdout, even a stray newline — all corrupt the JSON-RPC stream silently. Use file-based logging only.

2. **Claude Desktop silently drops servers that fail at startup.** Always test your server manually first.

   ```bash
   echo '{"jsonrpc":"2.0",...}' | python server.py
   ```

   Check `~/Library/Logs/Claude/mcp-server-<name>.log` for errors.

3. **macOS blocks apps from reading the Documents folder without permission.** Keep MCP server scripts in `~/.airport-mcp/` or `~/claude-mcp-servers/` — not inside `~/Documents`.

4. **Empty result ≠ invalid input.** Return a clean `❌` error for bad inputs so Claude asks the user to fix it. Empty results make Claude think the location is valid but empty.

5. **HTTP transport requires auth; stdio doesn't.** The trust boundary for stdio is "whoever spawned the process." HTTP is open to the network — add authentication before deploying.

---

## Quick Reference

### Config file locations

| Context               | Path                                                                    |
| --------------------- | ----------------------------------------------------------------------- |
| Claude Desktop (Mac)  | `~/Library/Application Support/Claude/claude_desktop_config.json`       |
| Claude Code (project) | `.claude/settings.json`                                                 |
| Claude Code (global)  | `~/.claude/settings.json`                                               |

### Transport options

| Transport         | Use when                                   |
| ----------------- | ------------------------------------------ |
| `stdio`           | Local dev, single user, process-scoped     |
| `streamable-http` | Shared/team server, centralized deployment |

### Debug MCP connection

```bash
tail -f ~/Library/Logs/Claude/mcp-server-<name>.log
```

### Test server manually

```bash
echo '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{
  "protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"test","version":"1.0"}
}}' | GOOGLE_MAPS_API_KEY=x python server.py
```

### Tool definition checklist

```text
description   → what it does AND when to use it (Claude reasons from this)
inputSchema   → JSON Schema with "required" array
return value  → human-readable string, not raw dicts or exceptions
error string  → starts with ❌, explains why, tells Claude what to do next
```

---

## Session Handoff Prompt

Copy this into a new Claude Code session when you're ready to build:

```text
I know MCP server development well. Quick context:

MCP servers expose tools to Claude via JSON-RPC over stdio (or HTTP). The server
registers tools with list_tools(), handles calls with call_tool(). For stdio
transport, ALL logging must go to a file — stdout is the protocol channel.
Tool descriptions are what Claude reasons against to decide when/how to call them.

My server does: [describe your server here]
Transport: stdio | streamable-http
Language: Python (mcp SDK) | TypeScript
```
