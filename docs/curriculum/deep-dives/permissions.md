# Deep Dive: The Permission System

> **Time**: ~30-40 minutes
> **Prerequisites**: Complete Day 7-8 core concepts, read [Agent System](/deep-dives/agent-system.md)

The permission system is what makes opencode safe. It controls what tools can do, asks users for confirmation, and prevents dangerous operations. This deep dive explores how it works.

---

## Why Permissions Matter

An agent with full file and bash access is dangerous:
- Could delete important files
- Could run destructive commands
- Could leak secrets (`.env` files)
- Could make unwanted API calls

The permission system provides:
1. **Granular control** - Per-tool, per-pattern permissions
2. **User confirmation** - Ask before dangerous operations
3. **Safety defaults** - Deny risky operations by default
4. **Flexibility** - Override per agent, per session

---

## Permission Rules

A permission rule specifies what action to take for a specific tool/pattern combination:

```typescript
// permission/next.ts
export type Rule = {
  permission: string  // Tool name or "*" for all
  pattern: string     // Glob pattern to match
  action: "allow" | "deny" | "ask"
}

export type Ruleset = Rule[]
```

Examples:

```typescript
const rules: Ruleset = [
  // Allow reading most files
  { permission: "read", pattern: "*", action: "allow" },

  // But ask before reading .env files
  { permission: "read", pattern: "*.env", action: "ask" },

  // Always ask before running bash
  { permission: "bash", pattern: "*", action: "ask" },

  // Deny editing in node_modules
  { permission: "edit", pattern: "**/node_modules/**", action: "deny" },
]
```

---

## Rule Evaluation

The `evaluate()` function finds the most specific matching rule:

```typescript
export function evaluate(
  permission: string,
  value: string,
  ruleset: Ruleset
): { action: "allow" | "deny" | "ask"; rule?: Rule } {
  // Iterate through rules in order (most specific first)
  for (const rule of ruleset) {
    // Check if permission matches
    if (rule.permission !== permission && rule.permission !== "*") {
      continue  // Skip non-matching permissions
    }

    // Check if pattern matches using minimatch
    if (minimatch(value, rule.pattern)) {
      return { action: rule.action, rule }
    }
  }

  // No matching rule - default to "ask"
  return { action: "ask" }
}
```

**Key insight**: Rules are evaluated in order, so put more specific rules first.

---

## The `ask()` Function

Tools call `ctx.ask()` to request permission:

```typescript
export async function ask(input: {
  permission: string      // Tool name (e.g., "bash")
  patterns: string[]      // Values to check (e.g., ["rm -rf /"])
  sessionID: string
  metadata: any          // Extra info for UI
  ruleset: Ruleset       // Agent's permission rules
}) {
  for (const pattern of input.patterns) {
    const result = evaluate(input.permission, pattern, input.ruleset)

    if (result.action === "deny") {
      // Hard deny - throw error immediately
      throw new RejectedError({
        permission: input.permission,
        pattern,
        message: `Permission denied: ${input.permission} ${pattern}`
      })
    }

    if (result.action === "ask") {
      // Show UI prompt to user
      const response = await showPermissionPrompt({
        sessionID: input.sessionID,
        permission: input.permission,
        pattern,
        metadata: input.metadata,
      })

      if (response === "deny") {
        throw new RejectedError({ permission: input.permission, pattern })
      }

      if (response === "allow_always") {
        // Add a new rule to the ruleset
        input.ruleset.unshift({
          permission: input.permission,
          pattern,
          action: "allow",
        })
      }

      // "allow" or "allow_always" - continue
    }

    // "allow" - just continue
  }
}
```

---

## User Prompts

When the action is "ask", the UI shows a prompt:

```
┌─────────────────────────────────────────────────────────────┐
│                   Permission Request                         │
│                                                              │
│  Claude wants to run a bash command:                        │
│                                                              │
│  rm -rf /tmp/old-files                                      │
│                                                              │
│  Working directory: /home/user/project                      │
│                                                              │
│  ┌───────┐ ┌───────────────┐ ┌──────┐                      │
│  │ Allow │ │ Allow Always  │ │ Deny │                      │
│  └───────┘ └───────────────┘ └──────┘                      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

Options:
- **Allow** - Allow this once
- **Allow Always** - Add rule and never ask again
- **Deny** - Reject and throw error

---

## Configuration Format

Permissions can be specified in multiple formats in `.opencode/config.json`:

### Simple Format

```json
{
  "permissions": {
    "read": "allow",
    "bash": "ask",
    "edit": "deny"
  }
}
```

### Pattern Format

```json
{
  "permissions": {
    "read": {
      "*": "allow",
      "*.env": "ask",
      "**/.git/**": "deny"
    },
    "bash": "ask"
  }
}
```

### Array Format (most flexible)

```json
{
  "permissions": [
    { "permission": "read", "pattern": "*", "action": "allow" },
    { "permission": "read", "pattern": "*.env", "action": "ask" },
    { "permission": "bash", "pattern": "*", "action": "ask" }
  ]
}
```

---

## Converting Config to Rules

The `fromConfig()` function converts config to rules:

```typescript
export function fromConfig(config: any): Ruleset {
  const rules: Ruleset = []

  if (Array.isArray(config)) {
    // Already in rule format
    return config
  }

  for (const [permission, value] of Object.entries(config)) {
    if (typeof value === "string") {
      // Simple format: "read": "allow"
      rules.push({
        permission,
        pattern: "*",
        action: value as "allow" | "deny" | "ask",
      })
    } else if (typeof value === "object") {
      // Pattern format: "read": { "*": "allow", "*.env": "ask" }
      for (const [pattern, action] of Object.entries(value)) {
        rules.push({
          permission,
          pattern,
          action: action as "allow" | "deny" | "ask",
        })
      }
    }
  }

  return rules
}
```

---

## Merging Rulesets

Permissions are layered - agent permissions override defaults:

```typescript
export function merge(...rulesets: Ruleset[]): Ruleset {
  // Flatten all rulesets into one
  // Later rules override earlier ones
  return rulesets.flat()
}

// Usage:
const finalRules = PermissionNext.merge(
  defaultRules,
  agentRules,
  userOverrides,
  sessionOverrides
)
```

The order matters:
1. **Default rules** - Base safety defaults
2. **Agent rules** - Per-agent overrides
3. **User config** - From `.opencode/config.json`
4. **Session overrides** - Per-request overrides

---

## Example: Bash Tool

The bash tool checks permissions before executing:

```typescript
// tool/bash.ts (simplified)
export const BashTool = Tool.define("bash", {
  description: "Execute bash commands",
  parameters: z.object({
    command: z.string(),
  }),

  async execute(params, ctx) {
    // ─────────────────────────────────────────────────────────
    // 1. Request permission
    // ─────────────────────────────────────────────────────────
    await ctx.ask({
      permission: "bash",
      patterns: [params.command],
      metadata: {
        command: params.command,
        cwd: process.cwd(),
      },
    })

    // ─────────────────────────────────────────────────────────
    // 2. Execute command (only if permission granted)
    // ─────────────────────────────────────────────────────────
    const result = await exec(params.command)

    return {
      title: `$ ${params.command}`,
      output: result.stdout,
      metadata: {
        exitCode: result.exitCode,
        stderr: result.stderr,
      }
    }
  }
})
```

If permission is denied, `ctx.ask()` throws `RejectedError` and the command never runs.

---

## Special Permissions

### Doom Loop

Prevents infinite loops of the same tool call:

```typescript
{
  permission: "doom_loop",
  pattern: "*",
  action: "ask"
}
```

Triggered in the processor when same tool called 3+ times with same input.

### External Directory

Warns when accessing files outside the project:

```typescript
async function assertExternalDirectory(
  ctx: Context,
  filepath: string,
  options: { bypass?: boolean } = {}
) {
  const cwd = process.cwd()
  const relative = path.relative(cwd, filepath)

  if (relative.startsWith("..") && !options.bypass) {
    await ctx.ask({
      permission: "external_directory",
      patterns: [filepath],
      metadata: { filepath, cwd },
    })
  }
}
```

---

## Python Equivalent

```python
from dataclasses import dataclass
from fnmatch import fnmatch
from typing import Literal


Action = Literal["allow", "deny", "ask"]


@dataclass
class PermissionRule:
    permission: str  # Tool name or "*"
    pattern: str     # Glob pattern
    action: Action


Ruleset = list[PermissionRule]


class PermissionError(Exception):
    def __init__(self, permission: str, pattern: str):
        self.permission = permission
        self.pattern = pattern
        super().__init__(f"Permission denied: {permission} {pattern}")


def evaluate(
    permission: str,
    value: str,
    ruleset: Ruleset
) -> tuple[Action, PermissionRule | None]:
    """Find the first matching rule."""
    for rule in ruleset:
        # Check permission match
        if rule.permission != permission and rule.permission != "*":
            continue

        # Check pattern match
        if fnmatch(value, rule.pattern):
            return rule.action, rule

    # Default to ask
    return "ask", None


async def ask(
    permission: str,
    patterns: list[str],
    ruleset: Ruleset,
    session_id: str,
    metadata: dict = None
) -> None:
    """Check permissions, possibly prompting user."""
    for pattern in patterns:
        action, rule = evaluate(permission, pattern, ruleset)

        if action == "deny":
            raise PermissionError(permission, pattern)

        if action == "ask":
            # Show UI prompt
            response = await show_permission_prompt(
                session_id=session_id,
                permission=permission,
                pattern=pattern,
                metadata=metadata or {},
            )

            if response == "deny":
                raise PermissionError(permission, pattern)

            if response == "allow_always":
                # Add rule to ruleset
                ruleset.insert(0, PermissionRule(
                    permission=permission,
                    pattern=pattern,
                    action="allow"
                ))

        # "allow" - continue


def merge(*rulesets: Ruleset) -> Ruleset:
    """Merge multiple rulesets (later overrides earlier)."""
    result = []
    for ruleset in rulesets:
        result.extend(ruleset)
    return result


# Usage in a tool:
class BashTool:
    async def execute(self, args: dict, ctx: ToolContext):
        # Check permission
        await ask(
            permission="bash",
            patterns=[args["command"]],
            ruleset=ctx.agent.permissions,
            session_id=ctx.session_id,
            metadata={"command": args["command"]}
        )

        # Execute (only if permission granted)
        result = subprocess.run(
            args["command"],
            shell=True,
            capture_output=True
        )

        return ToolResult(
            title=f"$ {args['command']}",
            output=result.stdout.decode(),
            metadata={"exit_code": result.returncode}
        )
```

---

## Summary

The permission system:
- **Uses rules** with permissions, patterns, and actions
- **Evaluates in order** - first match wins
- **Prompts users** when action is "ask"
- **Merges rulesets** from defaults, agents, config, and session
- **Prevents dangerous operations** by default
- **Allows flexibility** with "allow always"

Permissions make agents safe to use while keeping them powerful.

---

## Related Deep Dives

- [Agent System](/deep-dives/agent-system.md) - How agents use permissions
- [Tool System Internals](/deep-dives/tool-system-internals.md) - How tools request permissions
- [The Processor](/deep-dives/processor.md) - Doom loop detection
