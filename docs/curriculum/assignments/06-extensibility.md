# Assignment 6: Extensibility

> **Goal**: Add skills, commands, and MCP server support to make your agent extensible without code changes.

---

## What You'll Build

An extensibility system that:
1. Loads user-defined skills from markdown files
2. Supports parameterized commands with `$1`, `$2` placeholders
3. Connects to MCP servers for external tools

---

## Part 1: Skills System

Skills are markdown files that become prompts:

```
~/.agent/skills/
├── review.md       # /review - Code review prompt
├── explain.md      # /explain - Code explanation prompt
└── refactor.md     # /refactor - Refactoring suggestions
```

### Skill File Format

```markdown
<!-- ~/.agent/skills/review.md -->
# Code Review

Review the current file or selection for:

1. **Bugs**: Logic errors, edge cases, potential crashes
2. **Security**: Input validation, injection risks, auth issues
3. **Performance**: Inefficient algorithms, unnecessary allocations
4. **Style**: Naming, formatting, code organization

Be specific and actionable. Reference line numbers.
```

### Skill Loader Implementation

```python
# skills.py
from dataclasses import dataclass
from pathlib import Path
from typing import Optional
import re

@dataclass
class Skill:
    """A user-defined prompt template."""
    name: str           # Skill name (filename without extension)
    content: str        # The prompt content
    description: str    # First line/heading for help text
    path: Path          # Source file path

class SkillRegistry:
    """Manages user-defined skills."""

    def __init__(self, skills_dir: Path = None):
        self.skills_dir = skills_dir or Path.home() / ".agent" / "skills"
        self.skills: dict[str, Skill] = {}
        self._load_skills()

    def _load_skills(self) -> None:
        """Load all skills from the skills directory."""
        if not self.skills_dir.exists():
            return

        for path in self.skills_dir.glob("*.md"):
            skill = self._parse_skill(path)
            if skill:
                self.skills[skill.name] = skill

    def _parse_skill(self, path: Path) -> Optional[Skill]:
        """Parse a skill file into a Skill object."""
        try:
            content = path.read_text()
            name = path.stem  # filename without extension

            # Extract description from first heading or line
            description = self._extract_description(content)

            return Skill(
                name=name,
                content=content,
                description=description,
                path=path
            )
        except Exception as e:
            print(f"Warning: Could not load skill {path}: {e}")
            return None

    def _extract_description(self, content: str) -> str:
        """Extract description from content."""
        lines = content.strip().split("\n")

        for line in lines:
            # Skip empty lines
            if not line.strip():
                continue

            # Remove markdown heading markers
            if line.startswith("#"):
                return line.lstrip("#").strip()

            # Use first non-empty line
            return line.strip()[:50]

        return "No description"

    def get(self, name: str) -> Optional[Skill]:
        """Get a skill by name."""
        return self.skills.get(name)

    def list_skills(self) -> list[Skill]:
        """List all available skills."""
        return list(self.skills.values())

    def reload(self) -> None:
        """Reload skills from disk."""
        self.skills.clear()
        self._load_skills()
```

### Integrating Skills

```python
# In your agent's input handler
class Agent:
    def __init__(self, ...):
        self.skill_registry = SkillRegistry()

    def process_input(self, user_input: str) -> str:
        """Process user input, expanding skills if needed."""

        # Check for skill invocation: /skillname
        if user_input.startswith("/"):
            parts = user_input[1:].split(maxsplit=1)
            skill_name = parts[0]
            extra_context = parts[1] if len(parts) > 1 else ""

            skill = self.skill_registry.get(skill_name)
            if skill:
                # Expand skill into full prompt
                expanded = skill.content
                if extra_context:
                    expanded += f"\n\nAdditional context: {extra_context}"
                return expanded
            else:
                # Unknown skill, list available ones
                available = [s.name for s in self.skill_registry.list_skills()]
                return f"Unknown skill: {skill_name}\nAvailable: {', '.join(available)}"

        return user_input
```

### Test Skills

```python
def test_skills():
    # Create test skills directory
    skills_dir = Path("./test_skills")
    skills_dir.mkdir(exist_ok=True)

    # Create a test skill
    (skills_dir / "test.md").write_text("""# Test Skill

This is a test skill that does something useful.

- Step 1
- Step 2
""")

    # Load skills
    registry = SkillRegistry(skills_dir)

    # Check skill loaded
    skill = registry.get("test")
    assert skill is not None
    assert skill.name == "test"
    assert "Test Skill" in skill.description

    print(f"Loaded skill: {skill.name}")
    print(f"Description: {skill.description}")

    # Cleanup
    (skills_dir / "test.md").unlink()
    skills_dir.rmdir()
```

---

## Part 2: Commands with Parameters

Commands are skills that accept arguments via `$1`, `$2`, etc.:

```markdown
<!-- ~/.agent/skills/fix.md -->
# Fix Issue

Fix the issue described below:

Issue: $1

Analyze the problem, identify the root cause, and implement a fix.
Test your changes before reporting completion.
```

### Command Parser

```python
# commands.py
import re
from dataclasses import dataclass
from typing import Optional

@dataclass
class Command:
    """A parameterized skill."""
    name: str
    content: str
    description: str
    parameters: list[str]  # ["$1", "$2", ...]
    path: Path

class CommandParser:
    """Parses and executes commands with parameters."""

    PARAM_PATTERN = re.compile(r'\$(\d+)')

    def __init__(self, skills_dir: Path = None):
        self.skills_dir = skills_dir or Path.home() / ".agent" / "skills"
        self.commands: dict[str, Command] = {}
        self._load_commands()

    def _load_commands(self) -> None:
        """Load all command files."""
        if not self.skills_dir.exists():
            return

        for path in self.skills_dir.glob("*.md"):
            content = path.read_text()
            params = self._find_parameters(content)

            # It's a command if it has parameters
            if params:
                self.commands[path.stem] = Command(
                    name=path.stem,
                    content=content,
                    description=self._extract_description(content),
                    parameters=params,
                    path=path
                )

    def _find_parameters(self, content: str) -> list[str]:
        """Find all parameter placeholders in content."""
        matches = self.PARAM_PATTERN.findall(content)
        # Return unique, sorted parameters
        return sorted(set(f"${m}" for m in matches), key=lambda x: int(x[1:]))

    def _extract_description(self, content: str) -> str:
        """Extract first heading as description."""
        for line in content.split("\n"):
            if line.startswith("#"):
                return line.lstrip("#").strip()
        return "No description"

    def expand(self, name: str, args: list[str]) -> Optional[str]:
        """
        Expand a command with given arguments.

        Example:
            expand("fix", ["login bug"])
            -> "Fix the issue described below:\n\nIssue: login bug\n..."
        """
        command = self.commands.get(name)
        if not command:
            return None

        content = command.content

        # Replace parameters with arguments
        for i, arg in enumerate(args, 1):
            content = content.replace(f"${i}", arg)

        # Check for missing parameters
        remaining = self.PARAM_PATTERN.findall(content)
        if remaining:
            missing = [f"${m}" for m in remaining]
            raise ValueError(f"Missing arguments for: {', '.join(missing)}")

        return content

    def get_usage(self, name: str) -> Optional[str]:
        """Get usage string for a command."""
        command = self.commands.get(name)
        if not command:
            return None

        params = " ".join(f"<arg{p[1:]}>" for p in command.parameters)
        return f"/{name} {params}"
```

### Integrating Commands

```python
# Enhanced input processor
class Agent:
    def __init__(self, ...):
        self.skill_registry = SkillRegistry()
        self.command_parser = CommandParser()

    def process_input(self, user_input: str) -> str:
        """Process input, expanding skills and commands."""

        if not user_input.startswith("/"):
            return user_input

        # Parse: /command arg1 arg2 "arg with spaces"
        parts = self._parse_command_line(user_input[1:])
        if not parts:
            return user_input

        name = parts[0]
        args = parts[1:]

        # Try as command first (has parameters)
        if name in self.command_parser.commands:
            try:
                return self.command_parser.expand(name, args)
            except ValueError as e:
                usage = self.command_parser.get_usage(name)
                return f"Error: {e}\nUsage: {usage}"

        # Try as skill (no parameters)
        skill = self.skill_registry.get(name)
        if skill:
            content = skill.content
            if args:
                content += f"\n\nContext: {' '.join(args)}"
            return content

        # Unknown
        return f"Unknown command: /{name}"

    def _parse_command_line(self, line: str) -> list[str]:
        """Parse command line respecting quotes."""
        import shlex
        try:
            return shlex.split(line)
        except ValueError:
            return line.split()
```

### Test Commands

```python
def test_commands():
    # Create test directory
    skills_dir = Path("./test_skills")
    skills_dir.mkdir(exist_ok=True)

    # Create command with parameters
    (skills_dir / "greet.md").write_text("""# Greeting

Hello, $1! Welcome to $2.
""")

    parser = CommandParser(skills_dir)

    # Expand with arguments
    result = parser.expand("greet", ["Alice", "the system"])
    assert "Hello, Alice!" in result
    assert "Welcome to the system" in result

    print(f"Expanded: {result}")

    # Test missing argument
    try:
        parser.expand("greet", ["Alice"])
        assert False, "Should have raised"
    except ValueError as e:
        print(f"Expected error: {e}")

    # Cleanup
    (skills_dir / "greet.md").unlink()
    skills_dir.rmdir()
```

---

## Part 3: MCP Server Integration

MCP (Model Context Protocol) lets external processes provide tools:

```
┌─────────────┐     JSON-RPC      ┌─────────────┐
│   Agent     │◄──────────────────│ MCP Server  │
│             │    (stdio)        │             │
│  - tools    │                   │  - database │
│  - prompts  │                   │  - github   │
└─────────────┘                   └─────────────┘
```

### MCP Client Implementation

```python
# mcp.py
import json
import subprocess
import asyncio
from dataclasses import dataclass
from typing import Optional, Any

@dataclass
class MCPTool:
    """A tool provided by an MCP server."""
    name: str
    description: str
    parameters: dict  # JSON Schema
    server_name: str  # Which server provides this

class MCPClient:
    """Client for communicating with MCP servers."""

    def __init__(self, name: str, command: list[str], env: dict = None):
        self.name = name
        self.command = command
        self.env = env or {}
        self.process: Optional[subprocess.Popen] = None
        self.tools: list[MCPTool] = []
        self._request_id = 0

    async def connect(self) -> None:
        """Start the MCP server process."""
        self.process = subprocess.Popen(
            self.command,
            stdin=subprocess.PIPE,
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE,
            env={**dict(os.environ), **self.env}
        )

        # Initialize connection
        await self._initialize()

        # Get available tools
        await self._list_tools()

    async def _initialize(self) -> dict:
        """Send initialize request."""
        return await self._request("initialize", {
            "protocolVersion": "2024-11-05",
            "capabilities": {},
            "clientInfo": {
                "name": "my-agent",
                "version": "1.0.0"
            }
        })

    async def _list_tools(self) -> None:
        """Get tools from the server."""
        response = await self._request("tools/list", {})

        for tool in response.get("tools", []):
            self.tools.append(MCPTool(
                name=tool["name"],
                description=tool.get("description", ""),
                parameters=tool.get("inputSchema", {}),
                server_name=self.name
            ))

    async def call_tool(self, name: str, arguments: dict) -> Any:
        """Call a tool on the MCP server."""
        response = await self._request("tools/call", {
            "name": name,
            "arguments": arguments
        })

        # Extract content from response
        content = response.get("content", [])
        if content and len(content) > 0:
            return content[0].get("text", str(content))
        return str(response)

    async def _request(self, method: str, params: dict) -> dict:
        """Send a JSON-RPC request and get response."""
        self._request_id += 1

        request = {
            "jsonrpc": "2.0",
            "id": self._request_id,
            "method": method,
            "params": params
        }

        # Write request
        request_line = json.dumps(request) + "\n"
        self.process.stdin.write(request_line.encode())
        self.process.stdin.flush()

        # Read response
        response_line = self.process.stdout.readline()
        response = json.loads(response_line.decode())

        if "error" in response:
            raise Exception(f"MCP error: {response['error']}")

        return response.get("result", {})

    def disconnect(self) -> None:
        """Stop the MCP server."""
        if self.process:
            self.process.terminate()
            self.process.wait()
            self.process = None
```

### MCP Registry

```python
# mcp_registry.py
from dataclasses import dataclass
from pathlib import Path
import json

@dataclass
class MCPServerConfig:
    """Configuration for an MCP server."""
    name: str
    command: list[str]
    env: dict = None
    enabled: bool = True

class MCPRegistry:
    """Manages MCP server connections."""

    def __init__(self, config_path: Path = None):
        self.config_path = config_path or Path.home() / ".agent" / "mcp.json"
        self.servers: dict[str, MCPClient] = {}
        self.all_tools: list[MCPTool] = []

    def load_config(self) -> list[MCPServerConfig]:
        """Load MCP server configurations."""
        if not self.config_path.exists():
            return []

        data = json.loads(self.config_path.read_text())
        configs = []

        for name, config in data.get("servers", {}).items():
            configs.append(MCPServerConfig(
                name=name,
                command=config["command"],
                env=config.get("env", {}),
                enabled=config.get("enabled", True)
            ))

        return configs

    async def connect_all(self) -> None:
        """Connect to all configured MCP servers."""
        configs = self.load_config()

        for config in configs:
            if not config.enabled:
                continue

            try:
                client = MCPClient(
                    name=config.name,
                    command=config.command,
                    env=config.env
                )
                await client.connect()

                self.servers[config.name] = client
                self.all_tools.extend(client.tools)

                print(f"Connected to MCP server: {config.name}")
                print(f"  Tools: {[t.name for t in client.tools]}")

            except Exception as e:
                print(f"Failed to connect to {config.name}: {e}")

    def get_tools(self) -> list[MCPTool]:
        """Get all tools from all connected servers."""
        return self.all_tools

    async def call_tool(self, tool_name: str, arguments: dict) -> Any:
        """Call a tool, routing to the correct server."""
        for tool in self.all_tools:
            if tool.name == tool_name:
                client = self.servers.get(tool.server_name)
                if client:
                    return await client.call_tool(tool_name, arguments)

        raise ValueError(f"Unknown MCP tool: {tool_name}")

    async def disconnect_all(self) -> None:
        """Disconnect from all servers."""
        for client in self.servers.values():
            client.disconnect()
        self.servers.clear()
        self.all_tools.clear()
```

### MCP Configuration File

```json
// ~/.agent/mcp.json
{
  "servers": {
    "filesystem": {
      "command": ["npx", "@anthropic/mcp-server-filesystem", "/home/user"],
      "enabled": true
    },
    "github": {
      "command": ["npx", "@anthropic/mcp-server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      },
      "enabled": true
    }
  }
}
```

### Integrating MCP Tools

```python
# Modify your tool registry to include MCP tools
class Agent:
    def __init__(self, ...):
        self.tool_registry = ToolRegistry()
        self.mcp_registry = MCPRegistry()

    async def initialize(self):
        """Initialize agent, including MCP connections."""
        await self.mcp_registry.connect_all()

        # Add MCP tools to tool list for LLM
        for mcp_tool in self.mcp_registry.get_tools():
            self.tool_registry.register_mcp_tool(mcp_tool)

    async def execute_tool(self, tool_call: dict) -> str:
        """Execute a tool call, routing to MCP if needed."""
        name = tool_call["function"]["name"]
        args = json.loads(tool_call["function"]["arguments"])

        # Check if it's an MCP tool
        if self.mcp_registry.has_tool(name):
            return await self.mcp_registry.call_tool(name, args)

        # Otherwise use local tool
        return await self.tool_registry.execute(name, args)
```

---

## Part 4: Putting It All Together

### Config File Structure

```
~/.agent/
├── skills/
│   ├── review.md        # /review
│   ├── explain.md       # /explain
│   ├── fix.md           # /fix $1
│   └── test.md          # /test $1
├── mcp.json             # MCP server configs
└── config.json          # Agent settings
```

### Complete Agent with Extensions

```python
# agent.py
class Agent:
    def __init__(self, llm_client):
        self.llm = llm_client

        # Core systems
        self.tool_registry = ToolRegistry()
        self.memory = Memory()
        self.permissions = PermissionSystem()

        # Extensions
        self.skill_registry = SkillRegistry()
        self.command_parser = CommandParser()
        self.mcp_registry = MCPRegistry()

    async def initialize(self):
        """Set up the agent."""
        # Register core tools
        self.tool_registry.register(ReadTool())
        self.tool_registry.register(WriteTool())
        self.tool_registry.register(EditTool())
        self.tool_registry.register(BashTool())

        # Connect MCP servers
        await self.mcp_registry.connect_all()

        # Add MCP tools
        for tool in self.mcp_registry.get_tools():
            self.tool_registry.register_mcp_tool(tool)

        print(f"Agent initialized with {len(self.tool_registry.tools)} tools")
        print(f"Skills available: {[s.name for s in self.skill_registry.list_skills()]}")

    async def run(self, user_input: str) -> str:
        """Main agent loop."""
        # Expand skills/commands
        expanded = self.process_input(user_input)

        # Add to memory
        self.memory.add_message(Message(role="user", content=expanded))

        while True:
            # ... main loop (same as before) ...
            pass

    def show_help(self) -> str:
        """Show available skills and commands."""
        lines = ["Available commands:"]

        # Skills (no parameters)
        lines.append("\nSkills:")
        for skill in self.skill_registry.list_skills():
            lines.append(f"  /{skill.name} - {skill.description}")

        # Commands (with parameters)
        lines.append("\nCommands:")
        for name, cmd in self.command_parser.commands.items():
            usage = self.command_parser.get_usage(name)
            lines.append(f"  {usage} - {cmd.description}")

        # MCP tools
        if self.mcp_registry.all_tools:
            lines.append("\nMCP Tools:")
            for tool in self.mcp_registry.all_tools:
                lines.append(f"  {tool.name} - {tool.description}")

        return "\n".join(lines)
```

---

## Deliverables

1. **skills.py** - Skill loading and expansion
2. **commands.py** - Parameterized command parsing
3. **mcp.py** - MCP client and registry
4. **Integration** - Extensions connected to agent

### Example Skills to Create

Create these in `~/.agent/skills/`:

```markdown
<!-- review.md -->
# Code Review

Review this code for bugs, security issues, and improvements.
Be specific and reference line numbers.
```

```markdown
<!-- fix.md -->
# Fix Issue

Fix the following issue: $1

1. Analyze the root cause
2. Implement a fix
3. Test your changes
```

```markdown
<!-- test.md -->
# Write Tests

Write tests for: $1

Use the existing test framework and patterns in this codebase.
```

---

## Hints

1. **Skills are just text**: They're simple markdown files. The power comes from the LLM interpreting them in context.

2. **Parameter escaping**: Handle `$1` carefully - don't replace `$10` when replacing `$1`.

3. **MCP is async**: Server communication is I/O bound. Use `asyncio` properly.

4. **Graceful degradation**: If an MCP server fails to connect, the agent should still work with local tools.

---

## What's Next

In [Day 13-14](../day-13-14-capstone.md), you'll:
- Review your complete agent architecture
- Add polish and robustness
- Plan future enhancements

You've built a full coding agent!
