# Deep Dive: MCP Integration (External Tools)

> **Time**: ~25-30 minutes
> **Prerequisites**: Complete Day 11-12 core concepts, read [Tool System Internals](/deep-dives/tool-system-internals.md)

MCP (Model Context Protocol) allows opencode to use external tools from separate processes. This enables extensibility without modifying the core. This deep dive explores how MCP integration works.

---

## What Is MCP?

MCP (Model Context Protocol) is a standard protocol for:
- **Exposing tools** from external processes
- **Discovering capabilities** dynamically
- **Executing tools** via JSON-RPC

Think of it as a "plugin system" for agents.

```
┌─────────────────────────────────────────────────────────────┐
│                       MCP Architecture                       │
│                                                              │
│  ┌───────────────┐                 ┌───────────────────┐    │
│  │   opencode    │◀─── JSON-RPC ──▶│  MCP Server       │    │
│  │   (client)    │                 │  (external tool)  │    │
│  └───────────────┘                 └───────────────────┘    │
│         │                                    │               │
│         │                                    │               │
│    List tools                           Provide tools        │
│    Execute tool                         Execute locally      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Why MCP?

**Built-in tools** are compiled into opencode:
- ✅ Fast (no IPC)
- ✅ Integrated
- ❌ Can't add without rebuilding
- ❌ Must be in TypeScript/JS

**MCP tools** run as separate processes:
- ✅ Add tools without modifying opencode
- ✅ Use any language (Python, Go, Rust, etc.)
- ✅ Sandboxed (can crash without affecting opencode)
- ❌ Slower (IPC overhead)

---

## MCP Protocol Overview

MCP uses JSON-RPC 2.0 over stdio:

### 1. List Available Tools

```json
// Request (opencode → server)
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/list",
  "params": {}
}

// Response (server → opencode)
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "tools": [
      {
        "name": "search_web",
        "description": "Search the web using DuckDuckGo",
        "inputSchema": {
          "type": "object",
          "properties": {
            "query": { "type": "string" }
          },
          "required": ["query"]
        }
      }
    ]
  }
}
```

### 2. Execute a Tool

```json
// Request (opencode → server)
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "search_web",
    "arguments": {
      "query": "opencode agent framework"
    }
  }
}

// Response (server → opencode)
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "Found 10 results for 'opencode agent framework'..."
      }
    ]
  }
}
```

---

## MCP Server Configuration

Users configure MCP servers in `.opencode/config.json`:

```json
{
  "mcpServers": {
    "web-search": {
      "command": "python",
      "args": ["-m", "mcp_server_websearch"],
      "env": {
        "SEARCH_API_KEY": "..."
      }
    },
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/home/user"]
    }
  }
}
```

---

## opencode's MCP Implementation

```typescript
// mcp/index.ts (simplified)
export namespace MCP {
  // MCP client instances by server name
  const clients = new Map<string, MCPClient>()

  export async function initialize() {
    const config = await Config.get()

    // Start all configured MCP servers
    for (const [name, serverConfig] of Object.entries(config.mcpServers || {})) {
      const client = await startServer(name, serverConfig)
      clients.set(name, client)
    }
  }

  async function startServer(
    name: string,
    config: ServerConfig
  ): Promise<MCPClient> {
    // ─────────────────────────────────────────────────────────
    // 1. Spawn the server process
    // ─────────────────────────────────────────────────────────
    const proc = spawn(config.command, config.args, {
      env: { ...process.env, ...config.env },
      stdio: ["pipe", "pipe", "inherit"],  // stdin, stdout, stderr
    })

    // ─────────────────────────────────────────────────────────
    // 2. Create JSON-RPC client over stdio
    // ─────────────────────────────────────────────────────────
    const client = new MCPClient({
      input: proc.stdout,
      output: proc.stdin,
    })

    // ─────────────────────────────────────────────────────────
    // 3. Initialize connection
    // ─────────────────────────────────────────────────────────
    await client.call("initialize", {
      protocolVersion: "1.0.0",
      capabilities: {},
      clientInfo: {
        name: "opencode",
        version: "1.0.0",
      }
    })

    log.info("mcp.server.started", { name })
    return client
  }

  export async function tools(): Promise<Record<string, AITool>> {
    const result: Record<string, AITool> = {}

    // ─────────────────────────────────────────────────────────
    // Collect tools from all MCP servers
    // ─────────────────────────────────────────────────────────
    for (const [serverName, client] of clients.entries()) {
      // List tools from this server
      const response = await client.call("tools/list", {})

      for (const toolDef of response.tools) {
        const toolID = `${serverName}/${toolDef.name}`

        result[toolID] = tool({
          description: toolDef.description,
          inputSchema: jsonSchema(toolDef.inputSchema),

          async execute(args, options) {
            // ───────────────────────────────────────────────────
            // Execute tool via JSON-RPC
            // ───────────────────────────────────────────────────
            const response = await client.call("tools/call", {
              name: toolDef.name,
              arguments: args,
            })

            // Convert MCP response to opencode format
            return {
              title: `[MCP] ${toolDef.name}`,
              output: formatMCPContent(response.content),
              metadata: { mcp: true, server: serverName }
            }
          }
        })
      }
    }

    return result
  }

  function formatMCPContent(content: MCPContent[]): string {
    return content.map(item => {
      if (item.type === "text") {
        return item.text
      } else if (item.type === "image") {
        return `[Image: ${item.mimeType}]`
      } else if (item.type === "resource") {
        return `[Resource: ${item.uri}]`
      }
      return ""
    }).join("\n")
  }

  export async function shutdown() {
    // Shutdown all MCP servers
    for (const [name, client] of clients.entries()) {
      await client.close()
      log.info("mcp.server.stopped", { name })
    }
    clients.clear()
  }
}
```

---

## Tool Resolution with MCP

In the main loop, tools are resolved from both built-in and MCP:

```typescript
// From session/prompt.ts
async function resolveTools(input: {
  agent: Agent.Info
  model: Provider.Model
  // ...
}) {
  const tools: Record<string, AITool> = {}

  // ─────────────────────────────────────────────────────────
  // 1. Built-in tools
  // ─────────────────────────────────────────────────────────
  for (const item of await ToolRegistry.tools(input.model.providerID, input.agent)) {
    tools[item.id] = /* ... */
  }

  // ─────────────────────────────────────────────────────────
  // 2. MCP tools
  // ─────────────────────────────────────────────────────────
  const mcpTools = await MCP.tools()
  for (const [key, item] of Object.entries(mcpTools)) {
    // Wrap with permission check
    const originalExecute = item.execute
    item.execute = async (args, opts) => {
      const ctx = context(args, opts)

      // Ask for permission (MCP tools are untrusted!)
      await ctx.ask({
        permission: key,
        patterns: ["*"],
        metadata: { tool: key, mcp: true },
      })

      return await originalExecute(args, opts)
    }

    tools[key] = item
  }

  return tools
}
```

**Key insight**: MCP tools go through the same permission system as built-in tools!

---

## Example: Web Search MCP Server

Here's a simple MCP server in Python:

```python
import json
import sys
from typing import Any


class MCPServer:
    def __init__(self):
        self.tools = [
            {
                "name": "search_web",
                "description": "Search the web using DuckDuckGo",
                "inputSchema": {
                    "type": "object",
                    "properties": {
                        "query": {"type": "string", "description": "Search query"}
                    },
                    "required": ["query"]
                }
            }
        ]

    def handle_request(self, request: dict) -> dict:
        method = request.get("method")
        params = request.get("params", {})
        request_id = request.get("id")

        if method == "initialize":
            return {
                "jsonrpc": "2.0",
                "id": request_id,
                "result": {
                    "protocolVersion": "1.0.0",
                    "capabilities": {"tools": {}},
                    "serverInfo": {"name": "web-search", "version": "1.0.0"}
                }
            }

        elif method == "tools/list":
            return {
                "jsonrpc": "2.0",
                "id": request_id,
                "result": {"tools": self.tools}
            }

        elif method == "tools/call":
            tool_name = params.get("name")
            args = params.get("arguments", {})

            if tool_name == "search_web":
                result = self.search_web(args["query"])
                return {
                    "jsonrpc": "2.0",
                    "id": request_id,
                    "result": {
                        "content": [
                            {"type": "text", "text": result}
                        ]
                    }
                }

        return {
            "jsonrpc": "2.0",
            "id": request_id,
            "error": {"code": -32601, "message": "Method not found"}
        }

    def search_web(self, query: str) -> str:
        # Implementation using DuckDuckGo API
        from duckduckgo_search import DDGS

        results = DDGS().text(query, max_results=5)
        output = f"Search results for '{query}':\n\n"

        for i, result in enumerate(results, 1):
            output += f"{i}. {result['title']}\n"
            output += f"   {result['href']}\n"
            output += f"   {result['body']}\n\n"

        return output

    def run(self):
        """Run the MCP server (reads from stdin, writes to stdout)."""
        for line in sys.stdin:
            try:
                request = json.loads(line)
                response = self.handle_request(request)
                print(json.dumps(response), flush=True)
            except Exception as e:
                print(json.dumps({
                    "jsonrpc": "2.0",
                    "id": None,
                    "error": {"code": -32603, "message": str(e)}
                }), flush=True)


if __name__ == "__main__":
    server = MCPServer()
    server.run()
```

---

## MCP vs Built-in Tools

| Aspect | Built-in Tools | MCP Tools |
|--------|---------------|-----------|
| **Speed** | Fast (in-process) | Slower (IPC) |
| **Language** | TypeScript only | Any language |
| **Installation** | Bundled | Separate install |
| **Sandboxing** | Same process | Separate process |
| **Debugging** | Easier | Harder (IPC) |
| **Extensibility** | Requires rebuild | Just config |

---

## When to Use MCP

Use MCP for:
- ✅ **Third-party integrations** (APIs, databases, etc.)
- ✅ **Language-specific tools** (Python ML libraries, etc.)
- ✅ **User customizations** (company-specific tools)
- ✅ **Experimental tools** (don't want to commit to built-in)

Use built-in tools for:
- ✅ **Core functionality** (read, write, edit, bash)
- ✅ **Performance-critical** operations
- ✅ **Tight integration** with opencode internals

---

## Security Considerations

MCP tools are **untrusted** and run in separate processes. opencode:
1. **Always asks permission** before executing MCP tools
2. **Validates input** using the tool's schema
3. **Isolates processes** (crashes don't affect opencode)
4. **Limits execution time** (can timeout)

Users should only configure MCP servers they trust!

---

## Summary

MCP integration:
- **Extends opencode** with external tools
- **Uses JSON-RPC** over stdio for communication
- **Supports any language** for tool implementation
- **Goes through permission system** like built-in tools
- **Enables extensibility** without modifying opencode

MCP makes opencode a **platform** rather than just a tool.

---

## Related Deep Dives

- [Tool System Internals](/deep-dives/tool-system-internals.md) - How tools work
- [Permissions](/deep-dives/permissions.md) - How MCP tools are protected
- [Main Loop Internals](/deep-dives/main-loop-internals.md) - How tools are resolved
