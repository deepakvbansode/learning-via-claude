# MCP + Claude Agent SDK + Co-work

## The Problem It Solves
Without MCP, every AI model needs custom integration code for every tool — REST, gRPC, database, CLI. MCP defines a standard protocol layer between AI and tools so the AI speaks one language regardless of what's underneath. The Agent SDK and co-work patterns then let you drive this programmatically and at scale.

---

## Core Concepts

### 1. MCP Architecture (Host / Client / Server)

**Why:** Decouples the AI application from the tools it uses. Neither side needs to know the other's implementation details.

**Mental model:** Like K8s CRI — K8s doesn't care if you run containerd or CRI-O, it just speaks CRI. Your tool doesn't care what AI is calling it, it just speaks MCP. MCP is the contract layer.

**Example:**
```
Host (Claude Code / your app)
  └── Client A  ──MCP──►  Server (your K8s tool — calls kubectl internally)
  └── Client B  ──MCP──►  Server (your DB tool — calls Postgres internally)
```
- Host + Client = one process (you don't build this)
- Server = separate process (you build this)

---

### 2. MCP Primitives (Tools, Resources, Prompts)

**Why:** Three different primitives because they serve three different actors — the AI, the application, and the user.

**Mental model:** Like a gRPC service with different method types — some called by the client automatically, some pre-loaded by the framework, some triggered by the user explicitly.

**Example:**
```typescript
// Tool — AI decides when to call it
server.tool(
  "get_pod_status",
  { pod_name: z.string().describe("Name of the pod to check") },
  async ({ pod_name }) => {
    const result = execSync(`kubectl get pod ${pod_name}`);
    return { content: [{ type: "text", text: result }] };
  }
);
```

Key rules:
- **Tools**: AI-invoked, do work, return results → you'll build these most
- **Resources**: App-loaded context (kubeconfig, schemas) → optional
- **Prompts**: User-triggered slash command templates → optional
- Only expose what you actually need

---

### 3. MCP Transport (stdio vs Streamable HTTP)

**Why:** Your MCP server is a separate process — transport defines how the host connects to it.

**Mental model:** Like choosing between a K8s sidecar (stdio) vs a K8s Service (HTTP). Sidecar = tight coupling, same machine, host manages lifecycle. Service = decoupled, independently scalable, multiple clients.

**Example:**
```json
// stdio (local dev) — host spawns your server as child process
{ "mcpServers": { "k8s-tools": { "command": "node", "args": ["./server.js"] } } }

// Streamable HTTP (production / multi-agent) — server runs independently
{ "mcpServers": { "k8s-tools": { "url": "http://k8s-mcp-server:3000/mcp" } } }
```

| | stdio | Streamable HTTP |
|---|---|---|
| Local dev / single user | ✓ | |
| Multiple agents sharing one server | | ✓ |
| Central patch & manage | | ✓ |

---

### 4. Claude Agent SDK

**Why:** Claude Code is for humans. Agent SDK is for code. Lets you drive the full agent loop (think → tool call → think → ...) programmatically from scripts, CI pipelines, or services.

**Mental model:** Like `kubectl` (human CLI) vs the Go/Python K8s client library (programmatic). Same underlying power, different interface.

**Example:**
```python
from claude_agent_sdk import query, ClaudeAgentOptions

async for event in query(
    prompt="Check if payment-server pod is healthy",
    options=ClaudeAgentOptions(system_prompt="You are a K8s operator")
):
    if hasattr(event, "result"):
        print(event.result)
```

Sessions preserve the full conversation history (every tool call, result, reasoning chain) across multiple `query()` calls:
```python
# Save session ID from first query, resume in second
options=ClaudeAgentOptions(resume=session_id)
```
Without a session, every `query()` is a blank slate.

---

### 5. Wiring MCP into Claude Agent SDK

**Why:** Extend Claude's built-in tools (Bash, Read, Write) with your own MCP server. Claude discovers and calls your tools automatically.

**Mental model:** Like K8s CRDs — built-in resources cover basics, CRDs extend the runtime with custom capabilities. MCP servers are your CRDs.

**Example:**
```python
options=ClaudeAgentOptions(
    mcp_servers={
        "k8s-tools": {
            "url": "http://k8s-mcp-server:3000/mcp"  # Streamable HTTP
        }
    }
)
```

You don't route to tools explicitly — Claude reads tool descriptions and decides which to call. Tool descriptions are your API contract with Claude.

---

### 6. Claude Code Co-work Patterns

**Why:** A single agent is sequential. Co-work runs multiple agents in parallel for throughput — like horizontal scaling in K8s.

**Mental model:** One pod handles one request at a time. Want throughput? Run more pods. Same here.

**Example — Subagents (orchestrator + workers):**
```
Main agent: "Audit all 5 namespaces"
  ├── Subagent A: namespace-1  ──┐
  ├── Subagent B: namespace-2    ├── parallel
  ├── Subagent C: namespace-3    │
  ├── Subagent D: namespace-4    │
  └── Subagent E: namespace-5  ──┘
        ↓
Main agent combines → health report
```

| Pattern | Use when |
|---|---|
| **Subagents** | Parallel subtasks + results need combining |
| **Git Worktrees** | Parallel independent features, no coordination |
| **Shared Task Lists** | Multiple agents/humans working from shared backlog |

Subagents are isolated — they don't share context with each other, only with the parent.

---

## Gotchas

1. **Tool descriptions are load-bearing.** Claude uses them to decide when and how to call your tool. `"get_pod_status"` with description `"Returns status, restart count, and last event for a K8s pod"` gets called correctly. `"check_pod"` with no description gets ignored or misused.

2. **Return errors as content, not exceptions.** In a tool handler, if something fails return `{ isError: true, content: [{ type: "text", text: "kubectl failed: pod not found" }] }`. Throwing an uncaught exception breaks the MCP protocol silently.

3. **HTTP server must be running before `query()` is called.** With stdio, the SDK starts your server automatically. With Streamable HTTP, you're responsible for uptime. If it's down, the SDK fails at connection time.

4. **Subagents don't share context with each other.** Only with their parent. If Subagent A finds something Subagent B needs, the parent must relay it explicitly.

5. **stdio messages cannot contain raw newlines.** Newlines are message delimiters. The SDK handles this for you — but if you ever write raw output, you'll break parsing silently.

---

## Quick Reference

**Transport config in settings.json / ClaudeAgentOptions:**
```json
// stdio
{ "command": "node", "args": ["./server.js"] }

// Streamable HTTP
{ "url": "http://host:3000/mcp" }
```

**Minimal MCP tool (TypeScript):**
```typescript
server.tool(
  "tool_name",
  { param: z.string().describe("what this param is") },
  async ({ param }) => ({
    content: [{ type: "text", text: result }]
  })
);
```

**Agent SDK query with MCP:**
```python
async for event in query(
    prompt="...",
    options=ClaudeAgentOptions(
        mcp_servers={"my-server": {"url": "http://host:3000/mcp"}},
        resume=session_id  # omit for fresh session
    )
):
    if hasattr(event, "result"):
        print(event.result)
```

**Co-work pattern decision:**

| Need | Pattern |
|---|---|
| Parallel tasks + combine results | Subagents |
| Parallel independent branches | Git Worktrees |
| Shared backlog across sessions | Task Lists |

---

### Session Handoff Prompt
Copy this into a new Claude Code session when you're ready to build:

---
I know MCP, Claude Agent SDK, and co-work patterns well. Quick context:
MCP is a standard protocol layer between AI and tools — the AI speaks MCP, the server translates to whatever is underneath (REST, gRPC, kubectl, etc.). The Agent SDK lets me drive the agent loop programmatically via `query()`, with sessions for stateful multi-step work. Co-work with subagents lets one orchestrator agent spawn parallel workers and combine their results.

I'm building: an MCP server that exposes Kubernetes tools (pod status, logs, etc.), wired into the Claude Agent SDK, with a co-work subagent pattern to check multiple namespaces in parallel and produce a health report.

---
