# Assignment 4: Add Agents & Permissions

> **Time**: ~60-90 minutes
> **Goal**: Add a permission system and multiple agent modes to your agent.

---

## What You'll Build

Your agent will gain:
1. **Permission checks** - Ask before dangerous operations
2. **Multiple agents** - Different modes with different capabilities
3. **Subagent support** - Spawn specialized agents for subtasks

---

## The Challenge

### Part 1: Permission System

Create a permission system that:
- Evaluates rules with patterns (glob matching)
- Supports "allow", "deny", and "ask" actions
- Persists "always allow" choices to disk
- Integrates with your existing tools

### Part 2: Agent Modes

Create at least two agent modes:
- `build` - Full capabilities (default)
- `explore` - Read-only mode

### Part 3: Permission Integration

Update your tools to use the permission system:
- `bash` - Always ask
- `read` - Allow, but ask for `.env` files
- `write` / `edit` - Ask

---

## Starter Code

```python
#!/usr/bin/env python3
"""
Agent with Permissions - Assignment 4
"""

import os
import sys
import json
import fnmatch
import asyncio
from pathlib import Path
from dataclasses import dataclass, field
from typing import Callable, Awaitable, Literal
from pydantic import BaseModel, Field
from anthropic import Anthropic

client = Anthropic()


# ============================================================================
# PERMISSION SYSTEM
# ============================================================================

@dataclass
class PermissionRule:
    """A single permission rule."""
    permission: str      # Tool name or "*"
    pattern: str         # Glob pattern
    action: Literal["allow", "deny", "ask"]


@dataclass
class PermissionRequest:
    """A request for permission."""
    permission: str      # e.g., "bash"
    patterns: list[str]  # e.g., ["rm -rf temp/"]
    metadata: dict       # Extra info for display


class PermissionDenied(Exception):
    """Raised when permission is denied."""
    pass


class PermissionSystem:
    """
    Manages permission checks for tool execution.

    Supports:
    - Rule-based evaluation with glob patterns
    - User prompts for "ask" actions
    - Persistent "always allow" storage
    """

    def __init__(self, rules: list[PermissionRule] = None):
        self.rules = rules or []
        self.approved: list[PermissionRule] = []
        self.storage_path = Path(".agent_permissions.json")
        self._load_approved()

    def _load_approved(self):
        """Load approved rules from disk."""
        # TODO: Implement
        # Load from self.storage_path if it exists
        pass

    def _save_approved(self):
        """Save approved rules to disk."""
        # TODO: Implement
        pass

    def evaluate(self, permission: str, value: str) -> Literal["allow", "deny", "ask"]:
        """
        Evaluate a single permission check.

        Checks approved rules first, then configured rules.
        Returns the action to take.
        """
        # TODO: Implement
        # 1. Check approved rules (previous "always" choices)
        # 2. Check configured rules
        # 3. Default to "ask"
        pass

    def _matches(self, rule: PermissionRule, permission: str, value: str) -> bool:
        """Check if a rule matches the permission and value."""
        # TODO: Implement
        # 1. Check permission matches (rule.permission == permission or "*")
        # 2. Check pattern matches using fnmatch
        pass

    async def check(
        self,
        request: PermissionRequest,
        ask_callback: Callable[[PermissionRequest], Awaitable[str]]
    ) -> None:
        """
        Check permission for a request.

        Raises PermissionDenied if denied.
        Prompts user if action is "ask".
        """
        # TODO: Implement
        # For each pattern in request.patterns:
        #   1. Evaluate the permission
        #   2. If "deny", raise PermissionDenied
        #   3. If "ask", call ask_callback
        #      - If callback returns "deny", raise PermissionDenied
        #      - If callback returns "always", add to approved and save
        #   4. If "allow", continue
        pass


# ============================================================================
# AGENT CONFIGURATION
# ============================================================================

@dataclass
class AgentConfig:
    """Configuration for an agent mode."""
    name: str
    description: str
    system_prompt: str
    permissions: list[PermissionRule] = field(default_factory=list)
    allowed_tools: list[str] | None = None  # None means all tools
    tool_overrides: dict = field(default_factory=dict)  # Tool-specific settings


# Define your agents
AGENTS = {
    "build": AgentConfig(
        name="build",
        description="Full-featured coding agent",
        system_prompt="""You are a coding assistant with full access to tools.
You can read, write, edit files and run commands.
Always ask before making destructive changes.""",
        permissions=[
            # Bash needs confirmation
            PermissionRule("bash", "git *", "allow"),
            PermissionRule("bash", "ls *", "allow"),
            PermissionRule("bash", "cat *", "allow"),
            PermissionRule("bash", "*", "ask"),

            # File operations
            PermissionRule("read", "*.env", "ask"),
            PermissionRule("read", "*", "allow"),
            PermissionRule("write", "*", "ask"),
            PermissionRule("edit", "*", "ask"),

            # List is always safe
            PermissionRule("list_files", "*", "allow"),
        ],
    ),

    "explore": AgentConfig(
        name="explore",
        description="Read-only exploration agent",
        system_prompt="""You are a read-only exploration agent.
Your job is to search and read code to answer questions.
You CANNOT modify files or run commands that change state.
Only use read and list_files tools.""",
        permissions=[
            PermissionRule("read", "*", "allow"),
            PermissionRule("list_files", "*", "allow"),
            PermissionRule("*", "*", "deny"),
        ],
        allowed_tools=["read", "list_files"],
    ),
}


# ============================================================================
# TOOL CONTEXT WITH PERMISSIONS
# ============================================================================

@dataclass
class ToolContext:
    """Context passed to tools during execution."""
    session_id: str
    agent_name: str
    permissions: PermissionSystem

    async def check_permission(
        self,
        permission: str,
        patterns: list[str],
        metadata: dict = None,
        ask_callback: Callable = None,
    ):
        """Check permission for an operation."""
        await self.permissions.check(
            PermissionRequest(
                permission=permission,
                patterns=patterns,
                metadata=metadata or {},
            ),
            ask_callback=ask_callback,
        )


# ============================================================================
# TOOL RESULT AND REGISTRY (from previous assignments)
# ============================================================================

@dataclass
class ToolResult:
    title: str
    output: str
    metadata: dict = field(default_factory=dict)


class ToolRegistry:
    def __init__(self):
        self._tools = {}

    def register(self, id: str, description: str, parameters: type[BaseModel]):
        def decorator(func):
            self._tools[id] = {
                "id": id,
                "description": description,
                "parameters": parameters,
                "function": func,
            }
            return func
        return decorator

    def get_definitions(self, allowed_tools: list[str] = None) -> list[dict]:
        """Get tool definitions, optionally filtered."""
        tools = self._tools.values()
        if allowed_tools is not None:
            tools = [t for t in tools if t["id"] in allowed_tools]

        return [
            {
                "name": tool["id"],
                "description": tool["description"],
                "input_schema": tool["parameters"].model_json_schema(),
            }
            for tool in tools
        ]

    async def execute(
        self,
        name: str,
        args: dict,
        ctx: ToolContext,
        ask_callback: Callable,
    ) -> ToolResult:
        """Execute a tool with permission checking."""
        if name not in self._tools:
            return ToolResult(
                title="Error",
                output=f"Unknown tool: {name}",
                metadata={"error": True}
            )

        tool = self._tools[name]
        try:
            params = tool["parameters"](**args)
            # Pass context and callback to tool
            return await tool["function"](params, ctx, ask_callback)
        except PermissionDenied as e:
            return ToolResult(
                title="Permission Denied",
                output=str(e),
                metadata={"error": True, "permission_denied": True}
            )
        except Exception as e:
            return ToolResult(
                title="Error",
                output=f"Error executing {name}: {e}",
                metadata={"error": True}
            )


registry = ToolRegistry()


# ============================================================================
# TOOLS WITH PERMISSION CHECKS
# ============================================================================

class ReadParams(BaseModel):
    file_path: str = Field(description="Path to file")
    offset: int = Field(default=0)
    limit: int = Field(default=2000)


@registry.register("read", "Read a file", ReadParams)
async def read_file(params: ReadParams, ctx: ToolContext, ask_callback) -> ToolResult:
    # Check permission
    await ctx.check_permission(
        permission="read",
        patterns=[params.file_path],
        metadata={"file": params.file_path},
        ask_callback=ask_callback,
    )

    # TODO: Implement file reading (copy from Assignment 3)
    path = Path(params.file_path)
    if not path.exists():
        return ToolResult("Error", f"File not found: {path}")

    content = path.read_text()
    lines = content.split("\n")[params.offset:params.offset + params.limit]
    formatted = "\n".join(
        f"{i + params.offset + 1:5}| {line}"
        for i, line in enumerate(lines)
    )

    return ToolResult(
        title=str(path),
        output=f"<file>\n{formatted}\n</file>",
    )


class ListParams(BaseModel):
    path: str = Field(default=".")


@registry.register("list_files", "List directory contents", ListParams)
async def list_files(params: ListParams, ctx: ToolContext, ask_callback) -> ToolResult:
    await ctx.check_permission(
        permission="list_files",
        patterns=[params.path],
        ask_callback=ask_callback,
    )

    path = Path(params.path)
    if not path.exists():
        return ToolResult("Error", f"Not found: {path}")

    entries = sorted(path.iterdir(), key=lambda p: (not p.is_dir(), p.name))
    lines = [f"{'d' if e.is_dir() else 'f'} {e.name}" for e in entries]

    return ToolResult(str(path), "\n".join(lines) or "(empty)")


class BashParams(BaseModel):
    command: str = Field(description="Command to execute")


@registry.register("bash", "Execute a bash command", BashParams)
async def bash(params: BashParams, ctx: ToolContext, ask_callback) -> ToolResult:
    # Always ask for bash commands
    await ctx.check_permission(
        permission="bash",
        patterns=[params.command],
        metadata={"command": params.command},
        ask_callback=ask_callback,
    )

    import subprocess
    result = subprocess.run(
        ["bash", "-c", params.command],
        capture_output=True,
        text=True,
        timeout=60,
    )

    output = result.stdout + result.stderr
    return ToolResult(
        title=params.command[:50],
        output=output or "(no output)",
        metadata={"exit_code": result.returncode}
    )


# ============================================================================
# PERMISSION PROMPT UI
# ============================================================================

async def ask_user_permission(request: PermissionRequest) -> str:
    """
    Console-based permission prompt.

    Returns: "once", "always", or "deny"
    """
    print(f"\n{'='*50}")
    print(f"Permission requested: {request.permission}")
    print(f"For: {request.patterns}")
    if request.metadata:
        for key, value in request.metadata.items():
            print(f"  {key}: {value}")
    print('='*50)

    while True:
        response = input("[a]llow once, allow [A]lways, [d]eny? ").strip()
        if response == 'a':
            return "once"
        if response in ('A', 'always'):
            return "always"
        if response == 'd':
            return "deny"
        print("Please enter 'a', 'A', or 'd'")


# ============================================================================
# AGENT LOOP
# ============================================================================

async def run_agent(user_message: str, agent_name: str = "build"):
    """Run the agent with the specified mode."""
    agent = AGENTS.get(agent_name)
    if not agent:
        print(f"Unknown agent: {agent_name}")
        return

    print(f"\n{'='*60}")
    print(f"Agent: {agent.name} - {agent.description}")
    print(f"User: {user_message}")
    print('='*60)

    # Create permission system with agent's rules
    permissions = PermissionSystem(rules=agent.permissions)

    # Create tool context
    ctx = ToolContext(
        session_id="session_1",
        agent_name=agent.name,
        permissions=permissions,
    )

    # Build messages with agent's system prompt
    messages = [
        {"role": "user", "content": user_message}
    ]

    step = 0
    while True:
        step += 1
        print(f"\n--- Step {step} ---")

        # Get tools for this agent
        tool_defs = registry.get_definitions(agent.allowed_tools)

        with client.messages.stream(
            model="claude-sonnet-4-20250514",
            max_tokens=4096,
            system=agent.system_prompt,
            tools=tool_defs,
            messages=messages,
        ) as stream:
            for event in stream:
                if event.type == "content_block_start":
                    if hasattr(event.content_block, 'type'):
                        if event.content_block.type == "text":
                            sys.stdout.write("\nAssistant: ")
                elif event.type == "content_block_delta":
                    if hasattr(event.delta, 'text'):
                        sys.stdout.write(event.delta.text)
                        sys.stdout.flush()

            print()
            response = stream.get_final_message()

        if response.stop_reason == "end_turn":
            break

        if response.stop_reason == "tool_use":
            messages.append({"role": "assistant", "content": response.content})

            tool_results = []
            for block in response.content:
                if block.type == "tool_use":
                    print(f"\n[Tool] {block.name}")
                    print(f"[Args] {block.input}")

                    # Execute with permission checking
                    result = await registry.execute(
                        block.name,
                        block.input,
                        ctx,
                        ask_user_permission,
                    )

                    print(f"[Result] {result.output[:200]}...")

                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": result.output,
                    })

            messages.append({"role": "user", "content": tool_results})


# ============================================================================
# MAIN
# ============================================================================

def main():
    print("Agent with Permissions")
    print("Commands: /build, /explore, or just type a message")
    print("Type 'quit' to exit\n")

    current_agent = "build"

    while True:
        try:
            user_input = input(f"[{current_agent}] You: ").strip()

            if user_input.lower() in ('quit', 'exit', 'q'):
                break

            # Handle agent switching
            if user_input.startswith("/"):
                agent_name = user_input[1:]
                if agent_name in AGENTS:
                    current_agent = agent_name
                    print(f"Switched to {agent_name} agent")
                else:
                    print(f"Unknown agent. Available: {', '.join(AGENTS.keys())}")
                continue

            if user_input:
                asyncio.run(run_agent(user_input, current_agent))

        except KeyboardInterrupt:
            break

    print("\nGoodbye!")


if __name__ == "__main__":
    main()
```

---

## Success Criteria

1. **Permission prompts work**:
   - "Run ls" → Allows without prompt (safe command)
   - "Run rm -rf temp" → Shows permission prompt
   - Choosing "always" remembers the choice

2. **Agent modes work**:
   - `/explore` switches to explore mode
   - In explore mode, can only read files
   - `/build` switches back to full mode

3. **Denied tools are blocked**:
   - In explore mode, "Edit file.txt" fails
   - Shows clear error message

---

## Hints

<details>
<summary>Hint 1: Permission evaluation</summary>

```python
def evaluate(self, permission: str, value: str) -> Literal["allow", "deny", "ask"]:
    # Check approved rules first
    for rule in self.approved:
        if self._matches(rule, permission, value):
            return "allow"

    # Check configured rules
    for rule in self.rules:
        if self._matches(rule, permission, value):
            return rule.action

    return "ask"

def _matches(self, rule: PermissionRule, permission: str, value: str) -> bool:
    if rule.permission != "*" and rule.permission != permission:
        return False
    return fnmatch.fnmatch(value, rule.pattern)
```

</details>

<details>
<summary>Hint 2: Permission check flow</summary>

```python
async def check(
    self,
    request: PermissionRequest,
    ask_callback: Callable[[PermissionRequest], Awaitable[str]]
) -> None:
    for pattern in request.patterns:
        action = self.evaluate(request.permission, pattern)

        if action == "deny":
            raise PermissionDenied(
                f"Denied: {request.permission} for {pattern}"
            )

        if action == "ask":
            response = await ask_callback(request)

            if response == "deny":
                raise PermissionDenied(f"User denied: {pattern}")

            if response == "always":
                self.approved.append(PermissionRule(
                    permission=request.permission,
                    pattern=pattern,
                    action="allow",
                ))
                self._save_approved()
```

</details>

---

## Stretch Goals

1. **Add "deny always"** - Remember denials too
2. **Session-scoped permissions** - Reset on new session
3. **Permission groups** - "allow all git commands"
4. **Subagent tool** - Spawn explore agent from build agent

---

## Next Up

In **Day 9-10**, we'll add memory and context management.

→ [Continue to Day 9-10: Context & Memory](../day-09-10-memory.md)
