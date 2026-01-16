# Day 11-12: Extensibility

> **Goal**: Learn how to extend agents with skills, commands, and external tools via MCP.

---

## Why Extensibility Matters

A good agent is one users can customize. opencode supports three extensibility mechanisms:

| Mechanism | What It Does | User Effort |
|-----------|--------------|-------------|
| **Skills** | User-defined prompt templates | Write a markdown file |
| **Commands** | Reusable prompts with arguments | Write a markdown file |
| **MCP** | External tool servers | Run a separate process |

---

## Skills: User-Defined Prompts

Skills are markdown files that become available prompts:

```markdown
<!-- .opencode/skills/review-code/SKILL.md -->
---
name: review-code
description: Review code for best practices
---

Review the following code for:
1. Security vulnerabilities
2. Performance issues
3. Code style violations
4. Missing error handling

Be thorough but concise. Prioritize critical issues.
```

When the user invokes `/review-code`, this prompt is injected.

Here's how opencode loads skills:

```typescript
// skill/skill.ts (simplified)
export namespace Skill {
  const SKILL_GLOB = new Bun.Glob("skills/**/SKILL.md")

  export async function loadSkills() {
    const skills: Record<string, Info> = {}

    // Scan .opencode/skills/ directory
    for await (const match of SKILL_GLOB.scan({
      cwd: ".opencode",
      absolute: true,
    })) {
      const md = await parseMarkdown(match)
      skills[md.name] = {
        name: md.name,
        description: md.description,
        location: match,
      }
    }

    return skills
  }

  export async function getPrompt(name: string): string {
    const skill = skills[name]
    if (!skill) throw new Error(`Skill not found: ${name}`)
    const content = await Bun.file(skill.location).text()
    // Remove frontmatter, return body
    return content.replace(/^---[\s\S]*?---\n/, "")
  }
}
```

---

## Python Implementation: Skill System

```python
import yaml
from pathlib import Path
from dataclasses import dataclass


@dataclass
class Skill:
    name: str
    description: str
    prompt: str
    location: Path


class SkillRegistry:
    """Load and manage user-defined skills."""

    def __init__(self, skill_dirs: list[Path] = None):
        self.skill_dirs = skill_dirs or [
            Path(".agent/skills"),
            Path.home() / ".agent/skills",
        ]
        self.skills: dict[str, Skill] = {}
        self._load_skills()

    def _load_skills(self):
        """Scan directories for SKILL.md files."""
        for skill_dir in self.skill_dirs:
            if not skill_dir.exists():
                continue

            for skill_file in skill_dir.glob("**/SKILL.md"):
                try:
                    skill = self._parse_skill(skill_file)
                    self.skills[skill.name] = skill
                except Exception as e:
                    print(f"Warning: Failed to load skill {skill_file}: {e}")

    def _parse_skill(self, path: Path) -> Skill:
        """Parse a SKILL.md file."""
        content = path.read_text()

        # Extract YAML frontmatter
        if content.startswith("---"):
            _, frontmatter, body = content.split("---", 2)
            meta = yaml.safe_load(frontmatter)
        else:
            meta = {}
            body = content

        return Skill(
            name=meta.get("name", path.parent.name),
            description=meta.get("description", ""),
            prompt=body.strip(),
            location=path,
        )

    def get(self, name: str) -> Skill | None:
        """Get a skill by name."""
        return self.skills.get(name)

    def list(self) -> list[Skill]:
        """List all available skills."""
        return list(self.skills.values())


# Example usage
skills = SkillRegistry()

# List available skills
for skill in skills.list():
    print(f"/{skill.name} - {skill.description}")

# Get a skill's prompt
if skill := skills.get("review-code"):
    print(skill.prompt)
```

---

## Commands: Prompts with Arguments

Commands are like skills but support arguments:

```markdown
<!-- .opencode/commands/fix/COMMAND.md -->
---
name: fix
description: Fix an issue by number
---

Fix issue #$1.

First, read the issue details. Then implement a fix that:
1. Addresses the root cause
2. Includes tests
3. Updates documentation if needed
```

The `$1` is replaced with the first argument: `/fix 123` becomes "Fix issue #123."

```python
import re
from dataclasses import dataclass


@dataclass
class Command:
    name: str
    description: str
    template: str


class CommandRegistry:
    """Commands with argument substitution."""

    def __init__(self):
        self.commands: dict[str, Command] = {}

    def register(self, name: str, description: str, template: str):
        """Register a command."""
        self.commands[name] = Command(name, description, template)

    def execute(self, name: str, args: str) -> str:
        """
        Execute a command with arguments.

        Args:
            name: Command name
            args: Space-separated arguments

        Returns:
            Expanded prompt
        """
        cmd = self.commands.get(name)
        if not cmd:
            raise ValueError(f"Unknown command: {name}")

        # Parse arguments
        arg_list = self._parse_args(args)

        # Substitute $1, $2, etc.
        result = cmd.template
        for i, arg in enumerate(arg_list, 1):
            result = result.replace(f"${i}", arg)

        # $ARGUMENTS = all arguments
        result = result.replace("$ARGUMENTS", args)

        return result

    def _parse_args(self, args: str) -> list[str]:
        """Parse arguments, respecting quotes."""
        # Match quoted strings or non-space sequences
        pattern = r'"[^"]*"|\'[^\']*\'|[^\s]+'
        matches = re.findall(pattern, args)
        # Remove quotes
        return [m.strip("\"'") for m in matches]


# Example
commands = CommandRegistry()
commands.register(
    "fix",
    "Fix a GitHub issue",
    "Fix issue #$1. Read the issue first, then implement a fix.",
)
commands.register(
    "test",
    "Run tests for a file",
    "Run tests for $1 and fix any failures.",
)

# Execute
prompt = commands.execute("fix", "123")
# "Fix issue #123. Read the issue first, then implement a fix."
```

---

## MCP: Model Context Protocol

MCP lets you connect external tool servers. The agent talks to MCP servers over JSON-RPC:

```
┌─────────────────────────────────────────────────────────────────┐
│                        MCP Architecture                          │
│                                                                  │
│   ┌───────────────┐          ┌───────────────┐                 │
│   │    Agent      │◀────────▶│  MCP Server   │                 │
│   │               │  JSON    │  (e.g., DB)   │                 │
│   │  - read       │  RPC     │               │                 │
│   │  - write      │          │  - query      │                 │
│   │  - bash       │          │  - insert     │                 │
│   │  + MCP tools  │          │  - update     │                 │
│   └───────────────┘          └───────────────┘                 │
│                                                                  │
│   MCP Protocol:                                                 │
│   1. Agent connects to server                                   │
│   2. Server declares available tools                            │
│   3. Agent can call tools via JSON-RPC                         │
│   4. Server returns results                                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

Here's a simple MCP server:

```python
import json
import sys
from typing import Any


class MCPServer:
    """Simple MCP server implementation."""

    def __init__(self, name: str):
        self.name = name
        self.tools: dict[str, dict] = {}

    def tool(self, name: str, description: str, parameters: dict):
        """Decorator to register a tool."""
        def decorator(func):
            self.tools[name] = {
                "name": name,
                "description": description,
                "inputSchema": {
                    "type": "object",
                    "properties": parameters,
                },
                "function": func,
            }
            return func
        return decorator

    def handle_request(self, request: dict) -> dict:
        """Handle a JSON-RPC request."""
        method = request.get("method")
        params = request.get("params", {})
        req_id = request.get("id")

        if method == "initialize":
            return self._response(req_id, {
                "protocolVersion": "2024-11-05",
                "serverInfo": {"name": self.name},
                "capabilities": {"tools": {}},
            })

        elif method == "tools/list":
            tools = [
                {
                    "name": t["name"],
                    "description": t["description"],
                    "inputSchema": t["inputSchema"],
                }
                for t in self.tools.values()
            ]
            return self._response(req_id, {"tools": tools})

        elif method == "tools/call":
            tool_name = params.get("name")
            tool_args = params.get("arguments", {})

            if tool_name not in self.tools:
                return self._error(req_id, -32601, f"Unknown tool: {tool_name}")

            try:
                result = self.tools[tool_name]["function"](**tool_args)
                return self._response(req_id, {
                    "content": [{"type": "text", "text": str(result)}]
                })
            except Exception as e:
                return self._error(req_id, -32000, str(e))

        return self._error(req_id, -32601, f"Unknown method: {method}")

    def _response(self, req_id: Any, result: dict) -> dict:
        return {"jsonrpc": "2.0", "id": req_id, "result": result}

    def _error(self, req_id: Any, code: int, message: str) -> dict:
        return {"jsonrpc": "2.0", "id": req_id, "error": {"code": code, "message": message}}

    def run_stdio(self):
        """Run server on stdin/stdout."""
        for line in sys.stdin:
            request = json.loads(line)
            response = self.handle_request(request)
            print(json.dumps(response), flush=True)


# Example: Database MCP Server
server = MCPServer("database")


@server.tool(
    name="query",
    description="Run a SQL query",
    parameters={
        "sql": {"type": "string", "description": "SQL query to execute"}
    }
)
def query_db(sql: str) -> str:
    # In real implementation, connect to actual database
    return f"Query executed: {sql}"


@server.tool(
    name="list_tables",
    description="List all tables in the database",
    parameters={}
)
def list_tables() -> str:
    return "users, orders, products"


if __name__ == "__main__":
    server.run_stdio()
```

---

## Integrating MCP into Your Agent

```python
import subprocess
import json
from dataclasses import dataclass


@dataclass
class MCPClient:
    """Client to connect to MCP servers."""

    process: subprocess.Popen
    tools: dict

    @classmethod
    def connect(cls, command: list[str]) -> "MCPClient":
        """Connect to an MCP server."""
        proc = subprocess.Popen(
            command,
            stdin=subprocess.PIPE,
            stdout=subprocess.PIPE,
            text=True,
        )

        client = cls(process=proc, tools={})
        client._initialize()
        client._load_tools()
        return client

    def _send(self, method: str, params: dict = None) -> dict:
        """Send a request and get response."""
        request = {
            "jsonrpc": "2.0",
            "id": 1,
            "method": method,
            "params": params or {},
        }
        self.process.stdin.write(json.dumps(request) + "\n")
        self.process.stdin.flush()

        response = json.loads(self.process.stdout.readline())
        if "error" in response:
            raise Exception(response["error"]["message"])
        return response["result"]

    def _initialize(self):
        """Initialize connection."""
        self._send("initialize", {
            "protocolVersion": "2024-11-05",
            "clientInfo": {"name": "my-agent"},
        })

    def _load_tools(self):
        """Load available tools from server."""
        result = self._send("tools/list")
        for tool in result.get("tools", []):
            self.tools[tool["name"]] = tool

    def call_tool(self, name: str, arguments: dict) -> str:
        """Call a tool on the server."""
        result = self._send("tools/call", {
            "name": name,
            "arguments": arguments,
        })
        content = result.get("content", [])
        return "\n".join(c.get("text", "") for c in content if c.get("type") == "text")

    def close(self):
        """Close connection."""
        self.process.terminate()


# Usage
mcp = MCPClient.connect(["python", "db_server.py"])
print("Available tools:", list(mcp.tools.keys()))
result = mcp.call_tool("list_tables", {})
print("Tables:", result)
mcp.close()
```

---

## Putting It All Together

Your extensible agent architecture:

```
┌─────────────────────────────────────────────────────────────────┐
│                    Extensible Agent                              │
│                                                                  │
│   Built-in Tools        User Extensions                         │
│   ─────────────        ────────────────                         │
│   - read               Skills:                                  │
│   - write              - /review-code                           │
│   - edit               - /explain                               │
│   - bash               - /refactor                              │
│                                                                  │
│                        Commands:                                │
│                        - /fix $1                                │
│                        - /pr $1                                 │
│                        - /test $1                               │
│                                                                  │
│                        MCP Servers:                             │
│                        - database (query, insert)               │
│                        - github (issues, prs)                   │
│                        - slack (send, read)                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Assignment: Add Extensibility

Add skills, commands, and MCP support to your agent.

**Start here**: [Assignment 6: Extensibility](./assignments/06-extensibility.md)

---

## Next Up

In **Day 13-14**, we'll review the complete architecture and polish your agent.

→ [Continue to Day 13-14: Capstone](../day-13-14-capstone.md)
