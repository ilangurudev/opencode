# Deep Dive: The Agent System

> **Time**: ~25-35 minutes
> **Prerequisites**: Complete Day 7-8 core concepts, read [Architecture Overview](/deep-dives/overview.md)

Agents in opencode are different "modes" with different capabilities and permissions. This deep dive explores how agents are defined, configured, and used.

---

## What Are Agents?

Agents define operational modes for the coding assistant. Each agent has:
- **Name** - Identifier (e.g., "build", "explore", "plan")
- **Description** - What it's for
- **Mode** - "primary", "subagent", or "all"
- **Permissions** - What tools it can use
- **Model** - Optional model override
- **Prompt** - Optional system prompt additions
- **Steps** - Maximum loop iterations

---

## The Agent Interface

```typescript
// agent/agent.ts
export namespace Agent {
  export const Info = z.object({
    name: z.string(),
    description: z.string().optional(),
    mode: z.enum(["subagent", "primary", "all"]),
    permission: PermissionNext.Ruleset,
    model: z.object({
      modelID: z.string(),
      providerID: z.string(),
    }).optional(),
    prompt: z.string().optional(),
    options: z.record(z.string(), z.any()),
    steps: z.number().int().positive().optional(),
  })

  export type Info = z.infer<typeof Info>
}
```

---

## Built-in Agents

opencode ships with several built-in agents:

### 1. Build Agent (Default)

The main agent with full capabilities:

```typescript
build: {
  name: "build",
  mode: "primary",
  description: "Full-featured coding agent",
  permission: PermissionNext.merge(defaults, {
    question: "allow",       // Can ask user questions
    plan_enter: "allow",     // Can enter plan mode
  }),
}
```

**Capabilities**:
- All tools (read, write, edit, bash, etc.)
- Can ask questions
- Can launch subagents
- Default choice for most tasks

### 2. Explore Agent (Read-Only)

Fast, read-only exploration:

```typescript
explore: {
  name: "explore",
  mode: "subagent",
  description: "Fast read-only exploration",
  prompt: PROMPT_EXPLORE,
  permission: PermissionNext.merge(defaults, {
    "*": "deny",        // Deny everything by default
    grep: "allow",      // Then allow specific tools
    glob: "allow",
    list: "allow",
    read: "allow",
    websearch: "allow",
  }),
  steps: 15,  // Limit iterations
}
```

**Capabilities**:
- Read files
- Search code (grep, glob)
- Web search
- NO editing, NO bash, NO writing

**Use case**: "What does this function do?" or "Where is X defined?"

### 3. Plan Agent

Structured planning mode:

```typescript
plan: {
  name: "plan",
  mode: "primary",
  description: "Create implementation plans",
  permission: PermissionNext.merge(defaults, {
    question: "allow",
    plan_exit: "allow",
    edit: {
      "*": "deny",                     // No editing...
      ".opencode/plans/*.md": "allow", // ...except plan files
    },
    bash: "deny",                      // No execution
  }),
}
```

**Capabilities**:
- Read code
- Create/edit plan files
- Ask questions
- NO execution

**Use case**: "Plan how to implement feature X"

### 4. General Purpose Agent

Subagent for complex multi-step tasks:

```typescript
"general-purpose": {
  name: "general-purpose",
  mode: "subagent",
  description: "For complex research and multi-step tasks",
  permission: PermissionNext.merge(defaults, {
    // Similar to build but optimized for research
  }),
  steps: 25,
}
```

---

## Agent Modes

### Primary Mode

Can be used by users directly. Shows up in UI:

```typescript
mode: "primary"
```

Examples: `build`, `plan`

### Subagent Mode

Used by other agents via the Task tool:

```typescript
mode: "subagent"
```

Examples: `explore`, `general-purpose`

The Task tool launches these:

```typescript
// From tool/task.ts
const agent = await Agent.get(args.subagentType)
if (agent.mode === "subagent" || agent.mode === "all") {
  // Launch this agent
}
```

### All Mode

Can be used both ways:

```typescript
mode: "all"
```

---

## Agent Configuration

Agents are configured with lazy initialization:

```typescript
const state = Instance.state(async () => {
  // Load user config
  const cfg = await Config.get()

  // Default permissions (all agents inherit these)
  const defaults = PermissionNext.fromConfig({
    "*": "allow",           // Allow all by default
    doom_loop: "ask",       // Ask about doom loops
    read: {
      "*": "allow",
      "*.env": "ask",       // Ask before reading secrets
      "**/node_modules/**": "deny",  // Never read node_modules
    },
    bash: "ask",           // Ask before running commands
  })

  const result: Record<string, Info> = {
    build: { ... },
    explore: { ... },
    plan: { ... },
    // ... more agents
  }

  // ─────────────────────────────────────────────────────────
  // Merge user-defined agents from config
  // ─────────────────────────────────────────────────────────
  for (const [key, value] of Object.entries(cfg.agent ?? {})) {
    if (result[key]) {
      // Merge with existing agent
      result[key] = {
        ...result[key],
        ...value,
        permission: PermissionNext.merge(
          result[key].permission,
          value.permission || []
        ),
      }
    } else {
      // Add new agent
      result[key] = {
        mode: "all",
        permission: defaults,
        ...value,
      }
    }
  }

  return result
})

export async function get(agent: string) {
  return state().then(x => x[agent])
}
```

---

## Using Agents

### Via Task Tool

```typescript
// LLM calls Task tool
{
  "name": "task",
  "arguments": {
    "subagent_type": "explore",
    "prompt": "Find where the calculateTotal function is defined",
    "description": "Find calculateTotal function"
  }
}
```

The Task tool:
1. Gets the agent config
2. Checks if mode allows subagent usage
3. Creates a new session
4. Runs the agent with the prompt
5. Returns the result

### Via Mode Switch

Users can switch agents:

```bash
/mode explore
# Now in read-only mode

/mode build
# Back to full mode
```

---

## Custom Agents

Users can define custom agents in `.opencode/config.json`:

```json
{
  "agent": {
    "reviewer": {
      "description": "Code review agent",
      "mode": "primary",
      "prompt": "You are a code reviewer. Focus on security, performance, and best practices.",
      "permission": {
        "*": "allow",
        "edit": "deny",
        "write": "deny",
        "bash": "deny"
      }
    }
  }
}
```

---

## Permission Inheritance

Agents inherit permissions hierarchically:

```
Default Permissions
        │
        ├─▶ Built-in agent permissions
        │           │
        │           └─▶ User config overrides
        │                       │
        │                       └─▶ Per-request overrides
```

Example:

```typescript
// 1. Default: bash = "ask"
const defaults = { bash: "ask" }

// 2. Explore agent: bash = "deny"
explore: {
  permission: PermissionNext.merge(defaults, {
    bash: "deny"
  })
}

// 3. User config: Allow bash for explore
// (in .opencode/config.json)
{
  "agent": {
    "explore": {
      "permission": {
        "bash": "allow"
      }
    }
  }
}

// Final result: bash = "allow"
```

---

## Step Limits

Agents can have step limits to prevent runaway loops:

```typescript
explore: {
  // ...
  steps: 15,  // Max 15 loop iterations
}
```

In the main loop:

```typescript
if (agent.steps && step >= agent.steps) {
  // Agent exceeded step limit
  await Session.updateMessage({
    role: "assistant",
    content: `[Agent ${agent.name} exceeded step limit of ${agent.steps}]`,
  })
  break
}
```

---

## Python Equivalent

```python
from dataclasses import dataclass
from typing import Literal


AgentMode = Literal["primary", "subagent", "all"]


@dataclass
class AgentConfig:
    name: str
    mode: AgentMode
    description: str = ""
    permissions: list[PermissionRule] = None
    model: str | None = None
    prompt: str | None = None
    steps: int | None = None


class AgentRegistry:
    def __init__(self):
        self.agents: dict[str, AgentConfig] = {}
        self._load_builtin_agents()

    def _load_builtin_agents(self):
        # Default permissions
        defaults = [
            PermissionRule("*", "*", "allow"),
            PermissionRule("doom_loop", "*", "ask"),
            PermissionRule("read", "*.env", "ask"),
            PermissionRule("bash", "*", "ask"),
        ]

        # Build agent (full capabilities)
        self.agents["build"] = AgentConfig(
            name="build",
            mode="primary",
            description="Full-featured coding agent",
            permissions=defaults + [
                PermissionRule("question", "*", "allow"),
            ]
        )

        # Explore agent (read-only)
        self.agents["explore"] = AgentConfig(
            name="explore",
            mode="subagent",
            description="Fast read-only exploration",
            permissions=[
                PermissionRule("*", "*", "deny"),  # Deny all
                PermissionRule("read", "*", "allow"),  # Allow read
                PermissionRule("grep", "*", "allow"),
                PermissionRule("glob", "*", "allow"),
            ],
            steps=15,
        )

        # Plan agent
        self.agents["plan"] = AgentConfig(
            name="plan",
            mode="primary",
            description="Create implementation plans",
            permissions=defaults + [
                PermissionRule("edit", "*", "deny"),
                PermissionRule("edit", ".opencode/plans/*.md", "allow"),
                PermissionRule("bash", "*", "deny"),
            ]
        )

    def get(self, name: str) -> AgentConfig:
        return self.agents.get(name)

    def load_user_config(self, config: dict):
        """Merge user-defined agents from config."""
        for name, user_agent in config.get("agent", {}).items():
            if name in self.agents:
                # Merge with existing
                existing = self.agents[name]
                existing.permissions.extend(user_agent.get("permissions", []))
            else:
                # Add new agent
                self.agents[name] = AgentConfig(
                    name=name,
                    mode=user_agent.get("mode", "all"),
                    description=user_agent.get("description", ""),
                    permissions=user_agent.get("permissions", []),
                )
```

---

## Summary

The agent system:
- **Defines operational modes** with different capabilities
- **Enforces permissions** at the agent level
- **Supports subagents** for delegation
- **Allows customization** via config
- **Limits iterations** to prevent runaway loops

Agents make the system flexible and safe - you can have a powerful "build" mode and a safe "explore" mode, switching based on the task.

---

## Related Deep Dives

- [Permissions](/deep-dives/permissions.md) - How permission rules work
- [Tool System Internals](/deep-dives/tool-system-internals.md) - How tools are filtered by permissions
- [Main Loop Internals](/deep-dives/main-loop-internals.md) - How agents are used in the loop
