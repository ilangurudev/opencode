# Deep Dive: Tool System Internals

> **Time**: ~30-40 minutes
> **Prerequisites**: Complete Day 5-6 core concepts, read [Architecture Overview](/deep-dives/overview.md)

This deep dive explores how tools are defined, registered, validated, and executed in opencode. Understanding the tool system is crucial because tools are what give agents their power.

---

## The Tool Interface

Every tool in opencode follows the same interface:

```typescript
// tool/tool.ts
export namespace Tool {
  export interface Info<Parameters extends z.ZodType> {
    id: string
    init: (ctx?: InitContext) => Promise<{
      description: string
      parameters: Parameters
      execute(args: z.infer<Parameters>, ctx: Context): Promise<Result>
    }>
  }

  export interface Result {
    title: string           // Short title for UI (e.g., "src/main.py")
    output: string          // What the LLM sees
    metadata: any           // Extra data for UI
    attachments?: FilePart[]  // Images, files, etc.
  }
}
```

---

## Tool Context

Every tool execution receives a context object:

```typescript
export type Context = {
  sessionID: string
  messageID: string
  agent: string
  abort: AbortSignal
  callID?: string
  extra?: { [key: string]: any }

  // Update UI with title/metadata during execution
  metadata(input: { title?: string; metadata?: any }): void

  // Request permission from user
  ask(input: {
    permission: string
    patterns: string[]
    metadata: any
  }): Promise<void>
}
```

This provides tools with:
- Session information
- Ability to update UI in real-time
- Permission system access
- Cancellation support

---

## The `Tool.define()` Function

The `Tool.define()` function wraps tool definitions with common functionality:

```typescript
export function define<P extends z.ZodType>(
  id: string,
  init: Info<P>["init"] | Awaited<ReturnType<Info<P>["init"]>>
): Info<P> {
  return {
    id,
    init: async (initCtx) => {
      // ─────────────────────────────────────────────────────────
      // 1. Initialize the tool (can be async)
      // ─────────────────────────────────────────────────────────
      const toolInfo = init instanceof Function ? await init(initCtx) : init

      // ─────────────────────────────────────────────────────────
      // 2. Wrap execute with validation and truncation
      // ─────────────────────────────────────────────────────────
      const originalExecute = toolInfo.execute
      toolInfo.execute = async (args, ctx) => {
        // Validate input against schema
        try {
          toolInfo.parameters.parse(args)
        } catch (error) {
          throw new Error(`Invalid arguments for ${id}: ${error}`)
        }

        // Execute the tool
        const result = await originalExecute(args, ctx)

        // Truncate output if needed (prevent huge outputs)
        const truncated = await Truncate.output(result.output)

        return {
          ...result,
          output: truncated.content,
          metadata: {
            ...result.metadata,
            truncated: truncated.truncated,
            originalSize: result.output.length,
            truncatedSize: truncated.content.length,
          }
        }
      }

      return toolInfo
    }
  }
}
```

This wrapper:
1. **Validates input** - Uses Zod schema
2. **Executes the tool** - Calls the implementation
3. **Truncates output** - Prevents token overflow
4. **Adds metadata** - Tracks truncation

---

## Example Tool: Read

Let's look at the read tool implementation:

```typescript
// tool/read.ts (simplified and annotated)
export const ReadTool = Tool.define("read", {
  description: DESCRIPTION,  // Loaded from read.txt

  parameters: z.object({
    filePath: z.string().describe("The path to the file to read"),
    offset: z.coerce.number().optional().describe("Line to start from"),
    limit: z.coerce.number().optional().describe("Number of lines"),
  }),

  async execute(params, ctx) {
    // ─────────────────────────────────────────────────────────
    // 1. Normalize the file path
    // ─────────────────────────────────────────────────────────
    let filepath = params.filePath
    if (!path.isAbsolute(filepath)) {
      filepath = path.join(process.cwd(), filepath)
    }

    // ─────────────────────────────────────────────────────────
    // 2. Check if file is outside working directory
    // ─────────────────────────────────────────────────────────
    await assertExternalDirectory(ctx, filepath, {
      bypass: Boolean(ctx.extra?.["bypassCwdCheck"]),
    })

    // ─────────────────────────────────────────────────────────
    // 3. Ask permission (handles .env, secrets, etc.)
    // ─────────────────────────────────────────────────────────
    await ctx.ask({
      permission: "read",
      patterns: [filepath],
      always: ["*"],  // Allow by default, but ask for sensitive files
      metadata: { filepath },
    })

    // ─────────────────────────────────────────────────────────
    // 4. Check if file exists
    // ─────────────────────────────────────────────────────────
    const file = Bun.file(filepath)
    if (!(await file.exists())) {
      throw new Error(`File not found: ${filepath}`)
    }

    // ─────────────────────────────────────────────────────────
    // 5. Handle special file types (images, PDFs)
    // ─────────────────────────────────────────────────────────
    if (file.type.startsWith("image/")) {
      const base64 = await file.arrayBuffer()
        .then(buf => Buffer.from(buf).toString("base64"))

      return {
        title: path.relative(Instance.worktree, filepath),
        output: "Image read successfully",
        metadata: { truncated: false, type: "image" },
        attachments: [{
          type: "file",
          mime: file.type,
          url: `data:${file.type};base64,${base64}`,
        }]
      }
    }

    // ─────────────────────────────────────────────────────────
    // 6. Read text file with line numbers
    // ─────────────────────────────────────────────────────────
    const offset = params.offset || 0
    const limit = params.limit || 2000

    const text = await file.text()
    const lines = text.split("\n")

    const selected = lines.slice(offset, offset + limit)
    const numbered = selected.map((line, i) => {
      const lineNum = (i + offset + 1).toString().padStart(5)
      return `${lineNum}→${line}`
    }).join("\n")

    return {
      title: path.relative(Instance.worktree, filepath),
      output: `<file path="${filepath}">\n${numbered}\n</file>`,
      metadata: {
        truncated: lines.length > offset + limit,
        totalLines: lines.length,
        shownLines: selected.length,
      }
    }
  }
})
```

---

## Tool Registry

The `ToolRegistry` manages all available tools:

```typescript
// tool/registry.ts (simplified)
export namespace ToolRegistry {
  const builtinTools: Tool.Info[] = [
    ReadTool,
    WriteTool,
    EditTool,
    BashTool,
    GrepTool,
    GlobTool,
    TaskTool,
    // ... more tools
  ]

  export async function tools(
    providerID: string,
    agent: Agent.Info
  ): Promise<Tool.Info[]> {
    const result: Tool.Info[] = []

    // ─────────────────────────────────────────────────────────
    // 1. Filter tools based on agent permissions
    // ─────────────────────────────────────────────────────────
    for (const tool of builtinTools) {
      const permission = PermissionNext.evaluate(
        tool.id,
        "*",
        agent.permission
      )

      // Skip denied tools
      if (permission.action === "deny") continue

      result.push(tool)
    }

    // ─────────────────────────────────────────────────────────
    // 2. Filter by provider compatibility
    // ─────────────────────────────────────────────────────────
    // Some tools only work with certain providers
    const filtered = result.filter(tool => {
      if (tool.providers) {
        return tool.providers.includes(providerID)
      }
      return true
    })

    return filtered
  }

  export async function get(id: string): Promise<Tool.Info | undefined> {
    return builtinTools.find(t => t.id === id)
  }
}
```

---

## Tool Execution Flow

Here's the complete flow when a tool is executed:

```
┌─────────────────────────────────────────────────────────────┐
│                    Tool Execution Flow                       │
│                                                              │
│  1. LLM decides to call tool                                │
│     ↓                                                        │
│  2. Processor receives tool-call event                      │
│     ↓                                                        │
│  3. Create tool context                                     │
│     ↓                                                        │
│  4. Trigger "tool.execute.before" hooks                     │
│     ↓                                                        │
│  5. Validate arguments (Zod schema)                         │
│     ↓                                                        │
│  6. Execute tool implementation                             │
│     │                                                        │
│     ├─▶ Permission checks (ctx.ask)                         │
│     ├─▶ Update UI metadata (ctx.metadata)                   │
│     ├─▶ Check cancellation (ctx.abort)                      │
│     └─▶ Return result                                       │
│     ↓                                                        │
│  7. Truncate output if needed                               │
│     ↓                                                        │
│  8. Trigger "tool.execute.after" hooks                      │
│     ↓                                                        │
│  9. Save result to message                                  │
│     ↓                                                        │
│  10. Send tool-result back to LLM                           │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Output Truncation

Large outputs are truncated to prevent token overflow:

```typescript
// tool/truncate.ts (simplified)
export namespace Truncate {
  const MAX_OUTPUT_LENGTH = 30000  // Characters

  export async function output(content: string): Promise<{
    content: string
    truncated: boolean
  }> {
    if (content.length <= MAX_OUTPUT_LENGTH) {
      return { content, truncated: false }
    }

    // Truncate with message
    const truncated = content.slice(0, MAX_OUTPUT_LENGTH)
    const remaining = content.length - MAX_OUTPUT_LENGTH

    return {
      content: `${truncated}\n\n[Output truncated: ${remaining} more characters]`,
      truncated: true
    }
  }
}
```

---

## Permission Integration

Tools use the permission system via `ctx.ask()`:

```typescript
// Inside a tool
await ctx.ask({
  permission: "bash",
  patterns: [command],
  metadata: { command, cwd: process.cwd() },
})
```

This checks the agent's permission ruleset and may prompt the user:

```
┌─────────────────────────────────────────────────────────────┐
│              Permission Check Flow                           │
│                                                              │
│  Tool calls ctx.ask()                                       │
│     ↓                                                        │
│  Check agent permission rules                               │
│     ├─▶ "allow" → Continue                                  │
│     ├─▶ "deny" → Throw RejectedError                        │
│     └─▶ "ask" → Prompt user                                 │
│           ↓                                                  │
│       User response:                                         │
│         ├─▶ "allow" → Continue                              │
│         ├─▶ "allow_always" → Add rule, continue             │
│         └─▶ "deny" → Throw RejectedError                    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Python Equivalent

Here's how to implement a tool system in Python:

```python
from dataclasses import dataclass
from typing import Protocol, Any
from pydantic import BaseModel


@dataclass
class ToolResult:
    title: str
    output: str
    metadata: dict
    attachments: list = None


class ToolContext(Protocol):
    session_id: str
    message_id: str
    agent: str
    abort: asyncio.Event

    async def ask(
        self,
        permission: str,
        patterns: list[str],
        metadata: dict
    ) -> None:
        """Request permission."""
        ...

    def metadata(self, title: str = None, metadata: dict = None) -> None:
        """Update UI metadata."""
        ...


class Tool(Protocol):
    id: str
    description: str
    parameters: type[BaseModel]

    async def execute(self, args: BaseModel, ctx: ToolContext) -> ToolResult:
        ...


def define_tool(
    tool_id: str,
    description: str,
    parameters: type[BaseModel]
):
    """Decorator to define a tool with validation and truncation."""
    def decorator(func):
        async def wrapper(args: dict, ctx: ToolContext) -> ToolResult:
            # 1. Validate arguments
            try:
                validated = parameters(**args)
            except Exception as e:
                raise ValueError(f"Invalid arguments for {tool_id}: {e}")

            # 2. Execute tool
            result = await func(validated, ctx)

            # 3. Truncate output if needed
            if len(result.output) > 30000:
                result.output = result.output[:30000] + "\n\n[Output truncated]"
                result.metadata["truncated"] = True

            return result

        wrapper.id = tool_id
        wrapper.description = description
        wrapper.parameters = parameters
        return wrapper
    return decorator


# Example: Read tool
class ReadParams(BaseModel):
    file_path: str
    offset: int = 0
    limit: int = 2000


@define_tool(
    tool_id="read",
    description="Read a file from the filesystem",
    parameters=ReadParams
)
async def read_tool(args: ReadParams, ctx: ToolContext) -> ToolResult:
    # Normalize path
    filepath = Path(args.file_path)
    if not filepath.is_absolute():
        filepath = Path.cwd() / filepath

    # Check permission
    await ctx.ask(
        permission="read",
        patterns=[str(filepath)],
        metadata={"filepath": str(filepath)}
    )

    # Read file
    if not filepath.exists():
        raise FileNotFoundError(f"File not found: {filepath}")

    text = filepath.read_text()
    lines = text.split("\n")

    # Extract range with line numbers
    selected = lines[args.offset:args.offset + args.limit]
    numbered = [
        f"{i+args.offset+1:5d}→{line}"
        for i, line in enumerate(selected)
    ]

    return ToolResult(
        title=str(filepath.name),
        output=f"<file>\n" + "\n".join(numbered) + "\n</file>",
        metadata={
            "truncated": len(lines) > args.offset + args.limit,
            "total_lines": len(lines),
        }
    )
```

---

## Summary

The tool system:
- **Standardizes interfaces** - All tools follow the same pattern
- **Validates input** - Uses schemas (Zod in TS, Pydantic in Python)
- **Manages permissions** - Tools request permission via context
- **Truncates output** - Prevents token overflow
- **Updates UI** - Tools can update metadata during execution
- **Supports cancellation** - Check abort signal

Tools are the bridge between the LLM and the real world. A well-designed tool system makes agents powerful and safe.

---

## Related Deep Dives

- [Permissions](/deep-dives/permissions.md) - How permission checks work
- [Main Loop Internals](/deep-dives/main-loop-internals.md) - How tools are resolved
- [The Processor](/deep-dives/processor.md) - How tool calls are handled
- [MCP Integration](/deep-dives/mcp-integration.md) - External tool servers
