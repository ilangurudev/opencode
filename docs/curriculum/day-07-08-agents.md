# Day 7-8: Agents & Permissions

> **Goal**: Understand how different agent modes work and how permissions keep the agent safe.

---

## Why Different Agents?

Not every task needs the same capabilities. Consider:

| Task | Needs |
|------|-------|
| "What does this function do?" | Read files, search code |
| "Fix this bug" | Read, edit, run tests |
| "Delete all temp files" | DANGER - needs confirmation |

Having a single all-powerful agent is dangerous. opencode solves this with **agent modes** - different configurations with different permissions.

---

## opencode's Built-in Agents

```
┌─────────────────────────────────────────────────────────────────┐
│                        Agent Modes                               │
│                                                                  │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐           │
│  │    build    │   │   explore   │   │    plan     │           │
│  │  (default)  │   │ (read-only) │   │ (thinking)  │           │
│  └──────┬──────┘   └──────┬──────┘   └──────┬──────┘           │
│         │                 │                 │                   │
│  Tools: ALL        Tools: READ       Tools: LIMITED             │
│  - read            - read            - read                     │
│  - write           - grep            - write (plan file only)   │
│  - edit            - glob            - question                 │
│  - bash            - list            - plan_exit                │
│  - task            - websearch                                  │
│  - ...                                                          │
│                                                                  │
│  Permissions:      Permissions:      Permissions:               │
│  - ask for bash    - no edit         - no bash                  │
│  - ask for .env    - no bash         - can only edit plan.md    │
│  - allow read      - allow all read                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

Let's look at how these are defined:

```typescript
// agent/agent.ts (simplified)
const result: Record<string, Info> = {
  build: {
    name: "build",
    mode: "primary",
    permission: PermissionNext.merge(
      defaults,
      PermissionNext.fromConfig({
        question: "allow",
        plan_enter: "allow",
      }),
    ),
  },

  explore: {
    name: "explore",
    mode: "subagent",
    description: "Fast read-only exploration",
    prompt: PROMPT_EXPLORE,
    permission: PermissionNext.merge(
      defaults,
      PermissionNext.fromConfig({
        "*": "deny",      // Deny everything by default
        grep: "allow",    // Then allow specific tools
        glob: "allow",
        list: "allow",
        read: "allow",
        websearch: "allow",
      }),
    ),
  },

  plan: {
    name: "plan",
    mode: "primary",
    permission: PermissionNext.merge(
      defaults,
      PermissionNext.fromConfig({
        question: "allow",
        plan_exit: "allow",
        edit: {
          "*": "deny",                              // No editing...
          ".opencode/plans/*.md": "allow",          // ...except plan files
        },
      }),
    ),
  },
}
```

---

## The Permission System

Permissions use a simple but powerful model: **rules with patterns**.

### Rule Structure

```typescript
type Rule = {
  permission: string  // Tool name or category (e.g., "read", "bash", "*")
  pattern: string     // Glob pattern (e.g., "*.env", "/home/*")
  action: "allow" | "deny" | "ask"
}
```

### Evaluation Order

Rules are evaluated in order. First match wins:

```
Rules:
  1. {permission: "read", pattern: "*.env", action: "ask"}
  2. {permission: "read", pattern: "*", action: "allow"}
  3. {permission: "*", pattern: "*", action: "ask"}

Evaluating "read" for "config.json":
  Rule 1: pattern "*.env" doesn't match → skip
  Rule 2: pattern "*" matches → ALLOW

Evaluating "read" for ".env":
  Rule 1: pattern "*.env" matches → ASK
```

**Python equivalent:**

```python
from dataclasses import dataclass
from typing import Literal
import fnmatch

@dataclass
class Rule:
    permission: str  # "read", "bash", "*"
    pattern: str     # "*.env", "/home/*", "*"
    action: Literal["allow", "deny", "ask"]


def evaluate(permission: str, value: str, rules: list[Rule]) -> str:
    """
    Evaluate permission against ruleset.
    Returns "allow", "deny", or "ask".
    """
    for rule in rules:
        # Check permission matches
        if rule.permission != permission and rule.permission != "*":
            continue

        # Check pattern matches
        if fnmatch.fnmatch(value, rule.pattern):
            return rule.action

    # Default to ask
    return "ask"
```

---

## The Ask Flow

When a tool needs permission, here's what happens:

```
┌─────────────────────────────────────────────────────────────────┐
│                     Permission Ask Flow                          │
│                                                                  │
│   Tool calls ctx.ask({permission: "bash", patterns: ["rm -rf"]})│
│                         │                                        │
│                         ▼                                        │
│   ┌─────────────────────────────────────────────────────┐       │
│   │              Evaluate against ruleset               │       │
│   │                                                     │       │
│   │   Rules:                                            │       │
│   │   - {bash, "git *", allow}                         │       │
│   │   - {bash, "rm *", ask}                            │       │
│   │   - {bash, "*", ask}                               │       │
│   └───────────────────────┬─────────────────────────────┘       │
│                           │                                      │
│           ┌───────────────┴───────────────┐                     │
│           ▼               ▼               ▼                     │
│       ┌───────┐       ┌───────┐       ┌───────┐                │
│       │ allow │       │  ask  │       │ deny  │                │
│       └───┬───┘       └───┬───┘       └───┬───┘                │
│           │               │               │                     │
│           │               ▼               │                     │
│           │    ┌─────────────────────┐    │                     │
│           │    │   Show prompt to    │    │                     │
│           │    │   user              │    │                     │
│           │    │                     │    │                     │
│           │    │   "Allow bash:      │    │                     │
│           │    │    rm -rf temp/"    │    │                     │
│           │    │                     │    │                     │
│           │    │   [Allow] [Always]  │    │                     │
│           │    │   [Deny]            │    │                     │
│           │    └──────────┬──────────┘    │                     │
│           │               │               │                     │
│           │      ┌────────┴────────┐      │                     │
│           │      ▼        ▼        ▼      │                     │
│           │   [once]  [always]  [reject]  │                     │
│           │      │        │        │      │                     │
│           ▼      ▼        │        ▼      ▼                     │
│       Continue   Continue │    Throw RejectedError              │
│                           │                                      │
│                           ▼                                      │
│                  Add rule to approved:                          │
│                  {bash, "rm *", allow}                          │
│                  Then continue                                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

Here's the actual code (simplified):

```typescript
// permission/next.ts
export async function ask(input: {
  permission: string
  patterns: string[]
  ruleset: Ruleset
  sessionID: string
  metadata: any
}) {
  const state = await getState()

  for (const pattern of input.patterns) {
    // Evaluate against both ruleset and approved rules
    const rule = evaluate(
      input.permission,
      pattern,
      input.ruleset,
      state.approved
    )

    if (rule.action === "deny") {
      throw new DeniedError(...)
    }

    if (rule.action === "ask") {
      // Create a promise that resolves when user responds
      return new Promise<void>((resolve, reject) => {
        const id = generateId()
        state.pending[id] = { resolve, reject, info: input }

        // Publish event for UI to show prompt
        Bus.publish(Event.Asked, { id, ...input })
      })
    }

    // action === "allow" - continue to next pattern
  }
}

export async function reply(input: {
  requestID: string
  reply: "once" | "always" | "reject"
}) {
  const state = await getState()
  const pending = state.pending[input.requestID]
  if (!pending) return

  delete state.pending[input.requestID]

  if (input.reply === "reject") {
    pending.reject(new RejectedError())
    return
  }

  if (input.reply === "always") {
    // Add to approved rules for future
    state.approved.push({
      permission: pending.info.permission,
      pattern: pending.info.patterns[0],  // simplified
      action: "allow",
    })
    // Save to disk for persistence
    await Storage.write(["permission", projectID], state.approved)
  }

  pending.resolve()
}
```

---

## Python Implementation

Let's build a permission system for your agent:

```python
import json
import fnmatch
from pathlib import Path
from dataclasses import dataclass, field
from typing import Literal, Callable, Awaitable
from enum import Enum


class PermissionAction(Enum):
    ALLOW = "allow"
    DENY = "deny"
    ASK = "ask"


@dataclass
class PermissionRule:
    permission: str      # Tool name or "*"
    pattern: str         # Glob pattern
    action: PermissionAction


@dataclass
class PermissionRequest:
    permission: str      # e.g., "bash"
    patterns: list[str]  # e.g., ["rm -rf temp/"]
    metadata: dict       # Extra info for display


class PermissionSystem:
    def __init__(
        self,
        rules: list[PermissionRule] = None,
        storage_path: Path = None,
        ask_callback: Callable[[PermissionRequest], Awaitable[str]] = None,
    ):
        self.rules = rules or []
        self.storage_path = storage_path or Path(".permissions.json")
        self.ask_callback = ask_callback
        self.approved: list[PermissionRule] = []
        self._load_approved()

    def _load_approved(self):
        """Load approved rules from disk."""
        if self.storage_path.exists():
            try:
                data = json.loads(self.storage_path.read_text())
                self.approved = [
                    PermissionRule(
                        permission=r["permission"],
                        pattern=r["pattern"],
                        action=PermissionAction.ALLOW,
                    )
                    for r in data
                ]
            except:
                self.approved = []

    def _save_approved(self):
        """Save approved rules to disk."""
        data = [
            {"permission": r.permission, "pattern": r.pattern}
            for r in self.approved
        ]
        self.storage_path.write_text(json.dumps(data, indent=2))

    def evaluate(self, permission: str, value: str) -> PermissionAction:
        """Evaluate a single permission check."""
        # Check approved rules first (user's previous "always" choices)
        for rule in self.approved:
            if self._matches(rule, permission, value):
                return PermissionAction.ALLOW

        # Then check configured rules
        for rule in self.rules:
            if self._matches(rule, permission, value):
                return rule.action

        # Default to ask
        return PermissionAction.ASK

    def _matches(self, rule: PermissionRule, permission: str, value: str) -> bool:
        """Check if a rule matches the permission and value."""
        # Permission must match
        if rule.permission != "*" and rule.permission != permission:
            return False
        # Pattern must match
        return fnmatch.fnmatch(value, rule.pattern)

    async def check(self, request: PermissionRequest) -> None:
        """
        Check permission. Raises PermissionDenied if denied.
        """
        for pattern in request.patterns:
            action = self.evaluate(request.permission, pattern)

            if action == PermissionAction.DENY:
                raise PermissionDenied(
                    f"Permission denied: {request.permission} for {pattern}"
                )

            if action == PermissionAction.ASK:
                if not self.ask_callback:
                    raise PermissionDenied(
                        f"Permission required: {request.permission} for {pattern}"
                    )

                # Ask the user
                response = await self.ask_callback(request)

                if response == "deny":
                    raise PermissionDenied(
                        f"User denied: {request.permission} for {pattern}"
                    )

                if response == "always":
                    # Remember this approval
                    self.approved.append(PermissionRule(
                        permission=request.permission,
                        pattern=pattern,
                        action=PermissionAction.ALLOW,
                    ))
                    self._save_approved()

                # "once" or "always" - continue


class PermissionDenied(Exception):
    """Raised when permission is denied."""
    pass


# Example usage
async def ask_user(request: PermissionRequest) -> str:
    """Simple console-based permission prompt."""
    print(f"\n{'='*50}")
    print(f"Permission requested: {request.permission}")
    print(f"For: {request.patterns}")
    if request.metadata:
        print(f"Details: {request.metadata}")
    print("='*50")

    while True:
        response = input("[a]llow once, allow [A]lways, [d]eny? ").strip().lower()
        if response == 'a':
            return "once"
        if response == 'A' or response == 'always':
            return "always"
        if response == 'd':
            return "deny"
        print("Please enter 'a', 'A', or 'd'")


# Create permission system with rules
permissions = PermissionSystem(
    rules=[
        # Safe operations
        PermissionRule("read", "*", PermissionAction.ALLOW),
        PermissionRule("list_files", "*", PermissionAction.ALLOW),

        # Sensitive files need confirmation
        PermissionRule("read", "*.env", PermissionAction.ASK),
        PermissionRule("read", "**/secrets/*", PermissionAction.ASK),

        # Write operations need confirmation
        PermissionRule("write", "*", PermissionAction.ASK),
        PermissionRule("edit", "*", PermissionAction.ASK),

        # Bash is dangerous
        PermissionRule("bash", "git *", PermissionAction.ALLOW),  # git is safe
        PermissionRule("bash", "ls *", PermissionAction.ALLOW),   # ls is safe
        PermissionRule("bash", "rm *", PermissionAction.ASK),     # rm needs ask
        PermissionRule("bash", "*", PermissionAction.ASK),        # default ask
    ],
    ask_callback=ask_user,
)
```

---

## Subagents: Agents That Spawn Agents

Sometimes the main agent needs help. The `Task` tool lets it spawn subagents:

```
┌─────────────────────────────────────────────────────────────────┐
│                     Subagent Architecture                        │
│                                                                  │
│   User: "Find all TODO comments and create issues for them"     │
│                                                                  │
│   Main Agent (build):                                           │
│   │                                                              │
│   │ "I'll use the explore agent to find TODOs..."              │
│   │                                                              │
│   └──▶ Task Tool ──▶ ┌─────────────────────┐                   │
│                       │  Subagent (explore) │                   │
│                       │                     │                   │
│                       │  - read files       │                   │
│                       │  - grep for TODO    │                   │
│                       │  - return results   │                   │
│                       └──────────┬──────────┘                   │
│                                  │                               │
│   Main Agent receives results ◀──┘                              │
│   │                                                              │
│   │ "Found 5 TODOs. Now I'll create issues..."                 │
│   │                                                              │
│   └──▶ (uses bash to run gh issue create)                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

The Task tool:

```typescript
// tool/task.ts (conceptual)
export const TaskTool = Tool.define("task", {
  parameters: z.object({
    prompt: z.string().describe("What the subagent should do"),
    subagent_type: z.string().describe("Which agent to use"),
    description: z.string().describe("Short description of the task"),
  }),

  async execute(params, ctx) {
    // Get the subagent configuration
    const agent = await Agent.get(params.subagent_type)
    if (!agent) {
      throw new Error(`Unknown agent: ${params.subagent_type}`)
    }

    // Create a new session for the subagent
    const subSession = await Session.create({
      parentID: ctx.sessionID,
    })

    // Run the subagent with its own loop
    const result = await SessionPrompt.prompt({
      sessionID: subSession.id,
      agent: agent.name,
      parts: [{ type: "text", text: params.prompt }],
    })

    // Return the result to the main agent
    return {
      title: params.description,
      output: extractTextFromResult(result),
      metadata: { agent: agent.name },
    }
  }
})
```

---

## Agent Configurations

Here's how to define different agent modes:

```python
from dataclasses import dataclass, field


@dataclass
class AgentConfig:
    name: str
    description: str
    mode: str  # "primary" or "subagent"
    system_prompt: str = ""
    permissions: list[PermissionRule] = field(default_factory=list)
    allowed_tools: list[str] = field(default_factory=list)
    denied_tools: list[str] = field(default_factory=list)


# Define agents
AGENTS = {
    "build": AgentConfig(
        name="build",
        description="Full-featured coding agent",
        mode="primary",
        system_prompt="You are a coding assistant with full access to tools.",
        permissions=[
            PermissionRule("bash", "*", PermissionAction.ASK),
            PermissionRule("*", "*", PermissionAction.ALLOW),
        ],
    ),

    "explore": AgentConfig(
        name="explore",
        description="Read-only exploration agent",
        mode="subagent",
        system_prompt="""You are a read-only exploration agent.
Your job is to search and read code to answer questions.
You CANNOT modify files or run commands that change state.""",
        permissions=[
            PermissionRule("*", "*", PermissionAction.DENY),
            PermissionRule("read", "*", PermissionAction.ALLOW),
            PermissionRule("list_files", "*", PermissionAction.ALLOW),
            PermissionRule("grep", "*", PermissionAction.ALLOW),
        ],
        allowed_tools=["read", "list_files", "grep"],
        denied_tools=["write", "edit", "bash"],
    ),

    "plan": AgentConfig(
        name="plan",
        description="Planning and thinking agent",
        mode="primary",
        system_prompt="""You are in planning mode.
Think carefully about the approach before implementing.
You can only edit plan files, not actual code.""",
        permissions=[
            PermissionRule("edit", "*.md", PermissionAction.ALLOW),
            PermissionRule("edit", "*", PermissionAction.DENY),
            PermissionRule("bash", "*", PermissionAction.DENY),
        ],
        allowed_tools=["read", "list_files", "write", "edit"],
    ),
}


def get_tools_for_agent(agent: AgentConfig, all_tools: dict) -> dict:
    """Filter tools based on agent configuration."""
    result = {}

    for tool_id, tool in all_tools.items():
        # Check denied list
        if tool_id in agent.denied_tools:
            continue

        # Check allowed list (if specified)
        if agent.allowed_tools and tool_id not in agent.allowed_tools:
            continue

        result[tool_id] = tool

    return result
```

---

## Integrating Permissions with Tools

Here's how tools use the permission system:

```python
@registry.register(
    id="bash",
    description="Execute a bash command",
    parameters=BashParams,
)
async def bash(params: BashParams, ctx: ToolContext) -> ToolResult:
    # Always ask permission for bash commands
    await ctx.permissions.check(PermissionRequest(
        permission="bash",
        patterns=[params.command],
        metadata={"command": params.command},
    ))

    # If we get here, permission was granted
    result = subprocess.run(
        ["bash", "-c", params.command],
        capture_output=True,
        text=True,
    )

    return ToolResult(
        title=params.command[:50],
        output=result.stdout + result.stderr,
        metadata={"exit_code": result.returncode}
    )


@registry.register(
    id="read",
    description="Read a file",
    parameters=ReadParams,
)
async def read_file(params: ReadParams, ctx: ToolContext) -> ToolResult:
    # Check permission for this specific file
    await ctx.permissions.check(PermissionRequest(
        permission="read",
        patterns=[params.file_path],
        metadata={},
    ))

    # Read the file...
    content = Path(params.file_path).read_text()
    return ToolResult(title=params.file_path, output=content)
```

---

## Summary

| Concept | Purpose |
|---------|---------|
| **Agents** | Different configurations for different tasks |
| **Permissions** | Rules that control what tools can do |
| **Ask Flow** | User confirmation for sensitive operations |
| **Subagents** | Spawning specialized agents for subtasks |

The permission system keeps the agent safe while still being useful. Users can:
- Allow everything for trusted operations
- Ask for confirmation on dangerous operations
- Deny operations that should never happen

---

## Assignment: Add Permission System

Now let's add agents and permissions to your agent.

**Start here**: [Assignment 4: Agents & Permissions](./assignments/04-permissions.md)

---

## Optional Deep Dives

Want to understand agents and permissions in more detail?

- 🤖 [Agent System](/deep-dives/agent-system.md) - How different agent modes work
- 🔐 [Permissions](/deep-dives/permissions.md) - Permission evaluation and rule merging
- 🔧 [Tool System Internals](/deep-dives/tool-system-internals.md) - How tools integrate with permissions
