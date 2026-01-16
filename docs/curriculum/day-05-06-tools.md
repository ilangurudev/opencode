# Day 5-6: The Tool System

> **Goal**: Understand how tools are defined, registered, and executed in a coding agent.

---

## Why Tools Are the Heart of an Agent

Without tools, an LLM can only generate text. With tools, it can:
- Read and write files
- Execute commands
- Search codebases
- Make API calls
- Interact with the real world

The tool system is what transforms a chatbot into an agent.

---

## The Tool Interface

Every tool in opencode follows the same interface:

```typescript
// tool/tool.ts - The core interface
export interface Info<Parameters extends z.ZodType> {
  id: string
  init: (ctx?: InitContext) => Promise<{
    description: string        // What the LLM sees
    parameters: Parameters     // Zod schema for input validation
    execute(
      args: z.infer<Parameters>,
      ctx: Context
    ): Promise<{
      title: string           // Short title for UI
      metadata: any           // Extra data for UI
      output: string          // What the LLM sees
      attachments?: FilePart[] // Images, files, etc.
    }>
  }>
}
```

Let's translate this to Python:

```python
from dataclasses import dataclass
from typing import Protocol, Any, TypeVar
from pydantic import BaseModel

class ToolResult:
    """What a tool returns."""
    title: str          # Short title (e.g., "src/main.py")
    output: str         # The actual output the LLM sees
    metadata: dict      # Extra info for UI
    attachments: list   # Images, files, etc.

class ToolContext:
    """Context passed to every tool."""
    session_id: str
    message_id: str
    agent: str
    abort_signal: asyncio.Event

    async def ask(self, permission: str, patterns: list[str]) -> None:
        """Request permission from user."""
        ...

    def metadata(self, title: str = None, metadata: dict = None) -> None:
        """Update UI metadata during execution."""
        ...

class Tool(Protocol):
    """The tool interface."""
    id: str
    description: str
    parameters: type[BaseModel]  # Pydantic model for validation

    async def execute(self, args: BaseModel, ctx: ToolContext) -> ToolResult:
        ...
```

---

## Anatomy of a Tool: Read

Let's look at the read tool, which is simple but demonstrates all the patterns:

```typescript
// tool/read.ts (simplified and annotated)
export const ReadTool = Tool.define("read", {
  description: DESCRIPTION,  // Loaded from read.txt

  // Input schema using Zod
  parameters: z.object({
    filePath: z.string().describe("The path to the file to read"),
    offset: z.coerce.number().optional().describe("Line number to start from"),
    limit: z.coerce.number().optional().describe("Number of lines to read"),
  }),

  async execute(params, ctx) {
    // 1. Normalize the path
    let filepath = params.filePath
    if (!path.isAbsolute(filepath)) {
      filepath = path.join(process.cwd(), filepath)
    }

    // 2. Check external directory permission
    await assertExternalDirectory(ctx, filepath)

    // 3. Ask permission for sensitive files
    await ctx.ask({
      permission: "read",
      patterns: [filepath],
      always: ["*"],  // Patterns that can be "always allowed"
      metadata: {},
    })

    // 4. Check if file exists
    const file = Bun.file(filepath)
    if (!(await file.exists())) {
      // Provide helpful suggestions
      const suggestions = findSimilarFiles(filepath)
      throw new Error(`File not found: ${filepath}\n\nDid you mean:\n${suggestions}`)
    }

    // 5. Handle special file types
    if (file.type.startsWith("image/")) {
      return {
        title: path.basename(filepath),
        output: "Image read successfully",
        metadata: { truncated: false },
        attachments: [{
          type: "file",
          mime: file.type,
          url: `data:${file.type};base64,${Buffer.from(await file.bytes()).toString("base64")}`,
        }]
      }
    }

    // 6. Read text file
    const lines = await file.text().then(text => text.split("\n"))
    const limit = params.limit ?? 2000
    const offset = params.offset ?? 0

    // 7. Format with line numbers
    const content = lines
      .slice(offset, offset + limit)
      .map((line, i) => `${(i + offset + 1).toString().padStart(5)}| ${line}`)
      .join("\n")

    // 8. Return result
    return {
      title: path.relative(Instance.worktree, filepath),
      output: `<file>\n${content}\n</file>`,
      metadata: {
        preview: lines.slice(0, 20).join("\n"),
        truncated: lines.length > offset + limit,
      }
    }
  }
})
```

**Key patterns to notice:**
1. **Path normalization** - Handle relative paths
2. **Permission checks** - Ask before sensitive operations
3. **Error handling** - Helpful error messages with suggestions
4. **Special cases** - Different handling for images vs text
5. **Formatted output** - Line numbers make the LLM's job easier
6. **Metadata** - Extra info for the UI

---

## Python Equivalent: Read Tool

```python
import os
import base64
from pathlib import Path
from pydantic import BaseModel, Field

class ReadParams(BaseModel):
    file_path: str = Field(description="The path to the file to read")
    offset: int = Field(default=0, description="Line number to start from")
    limit: int = Field(default=2000, description="Number of lines to read")

@registry.register(
    id="read",
    description="""Read a file from the filesystem.

Usage:
- The file_path can be absolute or relative to the current directory
- By default, reads up to 2000 lines starting from the beginning
- Use offset and limit for reading specific portions of large files
- Returns line numbers to help with editing
""",
    parameters=ReadParams,
)
async def read_file(params: ReadParams, ctx: ToolContext) -> ToolResult:
    # 1. Normalize the path
    file_path = Path(params.file_path)
    if not file_path.is_absolute():
        file_path = Path.cwd() / file_path
    file_path = file_path.resolve()

    # 2. Check permission for sensitive files
    if file_path.suffix in ['.env', '.pem', '.key']:
        await ctx.ask(
            permission="read",
            patterns=[str(file_path)],
        )

    # 3. Check if file exists
    if not file_path.exists():
        # Find similar files for suggestions
        parent = file_path.parent
        if parent.exists():
            similar = [
                f for f in parent.iterdir()
                if file_path.stem.lower() in f.name.lower()
            ][:3]
            if similar:
                raise FileNotFoundError(
                    f"File not found: {file_path}\n\n"
                    f"Did you mean:\n" +
                    "\n".join(str(s) for s in similar)
                )
        raise FileNotFoundError(f"File not found: {file_path}")

    # 4. Handle images
    if file_path.suffix.lower() in ['.png', '.jpg', '.jpeg', '.gif', '.webp']:
        with open(file_path, 'rb') as f:
            data = base64.b64encode(f.read()).decode()
        mime = f"image/{file_path.suffix[1:].lower()}"
        return ToolResult(
            title=file_path.name,
            output="Image read successfully",
            metadata={"truncated": False},
            attachments=[{
                "type": "file",
                "mime": mime,
                "url": f"data:{mime};base64,{data}",
            }]
        )

    # 5. Read text file
    try:
        with open(file_path, 'r', encoding='utf-8') as f:
            all_lines = f.readlines()
    except UnicodeDecodeError:
        raise ValueError(f"Cannot read binary file: {file_path}")

    # 6. Apply offset and limit
    lines = all_lines[params.offset:params.offset + params.limit]

    # 7. Format with line numbers
    formatted = []
    for i, line in enumerate(lines):
        line_num = params.offset + i + 1
        # Truncate very long lines
        if len(line) > 2000:
            line = line[:2000] + "..."
        formatted.append(f"{line_num:5}| {line.rstrip()}")

    content = "\n".join(formatted)

    # 8. Add context about truncation
    total_lines = len(all_lines)
    last_read = params.offset + len(lines)
    if last_read < total_lines:
        content += f"\n\n(File has {total_lines} lines. Use offset={last_read} to continue.)"
    else:
        content += f"\n\n(End of file - {total_lines} lines total)"

    return ToolResult(
        title=str(file_path.relative_to(Path.cwd())),
        output=f"<file>\n{content}\n</file>",
        metadata={
            "preview": "\n".join(lines[:20]),
            "truncated": last_read < total_lines,
        }
    )
```

---

## The Edit Tool: Complexity Revealed

The edit tool is more complex because LLMs often get edits slightly wrong. opencode uses multiple "replacers" to handle fuzzy matching:

```
┌─────────────────────────────────────────────────────────────────┐
│                     Edit Tool Replacers                          │
│                                                                  │
│   LLM provides:                                                  │
│   old_string = "function foo() {"                               │
│   new_string = "function bar() {"                               │
│                                                                  │
│   File contains:                                                 │
│   "  function foo() {  "  (has extra whitespace)                │
│                                                                  │
│   Replacers try in order:                                       │
│                                                                  │
│   1. SimpleReplacer        - Exact match                        │
│      ❌ "function foo() {" !== "  function foo() {  "           │
│                                                                  │
│   2. LineTrimmedReplacer   - Match trimmed lines                │
│      ✅ "function foo() {".trim() === line.trim()               │
│                                                                  │
│   3. BlockAnchorReplacer   - Match first/last lines             │
│   4. WhitespaceNormalized  - Collapse whitespace                │
│   5. IndentationFlexible   - Ignore indentation                 │
│   6. EscapeNormalized      - Handle escape sequences            │
│   7. TrimmedBoundary       - Trim whole block                   │
│   8. ContextAware          - Use surrounding context            │
│                                                                  │
│   First successful match wins!                                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

Here's the core logic:

```typescript
// tool/edit.ts - The replace function
export function replace(
  content: string,
  oldString: string,
  newString: string,
  replaceAll = false
): string {
  // Try each replacer in order
  for (const replacer of [
    SimpleReplacer,
    LineTrimmedReplacer,
    BlockAnchorReplacer,
    WhitespaceNormalizedReplacer,
    IndentationFlexibleReplacer,
    EscapeNormalizedReplacer,
    TrimmedBoundaryReplacer,
    ContextAwareReplacer,
    MultiOccurrenceReplacer,
  ]) {
    // Each replacer yields potential matches
    for (const search of replacer(content, oldString)) {
      const index = content.indexOf(search)
      if (index === -1) continue

      // If replaceAll, replace all occurrences
      if (replaceAll) {
        return content.replaceAll(search, newString)
      }

      // Otherwise, must be unique
      const lastIndex = content.lastIndexOf(search)
      if (index !== lastIndex) continue  // Not unique

      // Do the replacement
      return content.substring(0, index) +
             newString +
             content.substring(index + search.length)
    }
  }

  throw new Error("oldString not found in content")
}
```

**Python equivalent:**

```python
from typing import Generator, Callable

Replacer = Callable[[str, str], Generator[str, None, None]]

def simple_replacer(content: str, find: str) -> Generator[str, None, None]:
    """Exact match."""
    yield find

def line_trimmed_replacer(content: str, find: str) -> Generator[str, None, None]:
    """Match lines with trimmed whitespace."""
    content_lines = content.split("\n")
    find_lines = find.split("\n")

    for i in range(len(content_lines) - len(find_lines) + 1):
        matches = True
        for j, find_line in enumerate(find_lines):
            if content_lines[i + j].strip() != find_line.strip():
                matches = False
                break

        if matches:
            # Yield the actual text from content (preserves indentation)
            matched = "\n".join(content_lines[i:i + len(find_lines)])
            yield matched

def whitespace_normalized_replacer(content: str, find: str) -> Generator[str, None, None]:
    """Collapse all whitespace for comparison."""
    import re

    def normalize(s: str) -> str:
        return re.sub(r'\s+', ' ', s).strip()

    normalized_find = normalize(find)

    # Try each line
    for line in content.split("\n"):
        if normalize(line) == normalized_find:
            yield line

    # Try multi-line blocks
    lines = content.split("\n")
    find_lines = find.split("\n")
    for i in range(len(lines) - len(find_lines) + 1):
        block = "\n".join(lines[i:i + len(find_lines)])
        if normalize(block) == normalized_find:
            yield block


REPLACERS: list[Replacer] = [
    simple_replacer,
    line_trimmed_replacer,
    whitespace_normalized_replacer,
    # ... more replacers
]


def replace(content: str, old: str, new: str, replace_all: bool = False) -> str:
    """Replace old with new in content, using fuzzy matching."""
    for replacer in REPLACERS:
        for match in replacer(content, old):
            if match not in content:
                continue

            if replace_all:
                return content.replace(match, new)

            # Check uniqueness
            first = content.find(match)
            last = content.rfind(match)
            if first != last:
                continue  # Not unique, try next replacer

            # Do the replacement
            return content[:first] + new + content[first + len(match):]

    raise ValueError(
        "oldString not found in content. "
        "Provide more surrounding context to identify the correct match."
    )
```

---

## Tool Registration: The Registry

Tools are registered centrally so the main loop can find them:

```typescript
// tool/registry.ts (simplified)
export namespace ToolRegistry {
  const tools: Tool.Info[] = [
    ReadTool,
    EditTool,
    WriteTool,
    BashTool,
    GlobTool,
    GrepTool,
    ListTool,
    TaskTool,
    // ... more
  ]

  export async function tools(
    providerID: string,
    agent: Agent.Info
  ): Promise<Tool.Info[]> {
    const result: Tool.Info[] = []

    for (const tool of tools) {
      // Check if agent allows this tool
      const permission = PermissionNext.evaluate(
        tool.id,
        "*",
        agent.permission
      )
      if (permission.action === "deny") continue

      // Initialize the tool
      const initialized = await tool.init({ agent })
      result.push({
        id: tool.id,
        ...initialized,
      })
    }

    return result
  }
}
```

**Python equivalent:**

```python
class ToolRegistry:
    def __init__(self):
        self._tools: dict[str, Tool] = {}

    def register(
        self,
        id: str,
        description: str,
        parameters: type[BaseModel],
    ):
        """Decorator to register a tool."""
        def decorator(func):
            self._tools[id] = Tool(
                id=id,
                description=description,
                parameters=parameters,
                execute=func,
            )
            return func
        return decorator

    def get_tools(self, agent: Agent) -> list[Tool]:
        """Get tools available to an agent."""
        result = []
        for tool in self._tools.values():
            # Check permission
            if not agent.can_use_tool(tool.id):
                continue
            result.append(tool)
        return result

    def execute(self, tool_id: str, args: dict, ctx: ToolContext) -> ToolResult:
        """Execute a tool by ID."""
        tool = self._tools.get(tool_id)
        if not tool:
            raise ValueError(f"Unknown tool: {tool_id}")

        # Validate args
        params = tool.parameters(**args)

        # Execute
        return tool.execute(params, ctx)
```

---

## The Bash Tool: Executing Commands

The bash tool is powerful but dangerous. It needs careful handling:

```typescript
// Conceptual bash tool (actual implementation is complex)
export const BashTool = Tool.define("bash", {
  description: "Execute a bash command",

  parameters: z.object({
    command: z.string().describe("The command to execute"),
    timeout: z.number().optional().describe("Timeout in ms"),
  }),

  async execute(params, ctx) {
    // 1. Permission check - ALWAYS ask for bash
    await ctx.ask({
      permission: "bash",
      patterns: [params.command],
      always: [],  // Never auto-allow bash
      metadata: { command: params.command },
    })

    // 2. Execute with timeout
    const timeout = params.timeout ?? 120_000  // 2 minutes default

    const proc = spawn("bash", ["-c", params.command], {
      cwd: process.cwd(),
      timeout,
      stdio: ["ignore", "pipe", "pipe"],
    })

    // 3. Capture output
    let stdout = ""
    let stderr = ""

    proc.stdout.on("data", (data) => {
      stdout += data.toString()
      // Update UI in real-time
      ctx.metadata({ metadata: { output: stdout + stderr } })
    })

    proc.stderr.on("data", (data) => {
      stderr += data.toString()
      ctx.metadata({ metadata: { output: stdout + stderr } })
    })

    // 4. Wait for completion
    const exitCode = await new Promise<number>((resolve) => {
      proc.on("close", resolve)
    })

    // 5. Format output
    const output = stdout + stderr
    const truncated = output.length > 30_000
    const finalOutput = truncated
      ? output.slice(0, 30_000) + "\n... (truncated)"
      : output

    return {
      title: params.command.slice(0, 50),
      output: exitCode === 0
        ? finalOutput
        : `Command failed with exit code ${exitCode}:\n${finalOutput}`,
      metadata: {
        exitCode,
        truncated,
      }
    }
  }
})
```

**Python equivalent:**

```python
import asyncio
import subprocess
from asyncio.subprocess import PIPE

class BashParams(BaseModel):
    command: str = Field(description="The command to execute")
    timeout: int = Field(default=120, description="Timeout in seconds")

@registry.register(
    id="bash",
    description="Execute a bash command in the current directory.",
    parameters=BashParams,
)
async def bash(params: BashParams, ctx: ToolContext) -> ToolResult:
    # 1. Always ask permission for bash
    await ctx.ask(
        permission="bash",
        patterns=[params.command],
    )

    # 2. Execute
    try:
        proc = await asyncio.create_subprocess_shell(
            params.command,
            stdout=PIPE,
            stderr=PIPE,
            cwd=os.getcwd(),
        )

        # 3. Wait with timeout
        try:
            stdout, stderr = await asyncio.wait_for(
                proc.communicate(),
                timeout=params.timeout
            )
        except asyncio.TimeoutError:
            proc.kill()
            return ToolResult(
                title=params.command[:50],
                output=f"Command timed out after {params.timeout}s",
                metadata={"timeout": True}
            )

        # 4. Combine output
        output = stdout.decode() + stderr.decode()

        # 5. Truncate if needed
        max_len = 30_000
        truncated = len(output) > max_len
        if truncated:
            output = output[:max_len] + "\n... (truncated)"

        # 6. Format based on exit code
        if proc.returncode == 0:
            final_output = output
        else:
            final_output = f"Exit code {proc.returncode}:\n{output}"

        return ToolResult(
            title=params.command[:50],
            output=final_output,
            metadata={
                "exit_code": proc.returncode,
                "truncated": truncated,
            }
        )

    except Exception as e:
        return ToolResult(
            title=params.command[:50],
            output=f"Error: {e}",
            metadata={"error": str(e)}
        )
```

---

## Tool Output Best Practices

Good tool output helps the LLM understand what happened:

### 1. Structure the Output

```python
# Bad - hard for LLM to parse
return ToolResult(
    output="file1.py file2.py file3.py README.md"
)

# Good - clear structure
return ToolResult(
    output="""<files path=".">
file1.py
file2.py
file3.py
README.md
</files>"""
)
```

### 2. Include Context

```python
# Bad - no context
return ToolResult(
    output="def foo():\n    pass"
)

# Good - includes line numbers and file info
return ToolResult(
    output="""<file path="src/main.py" lines="10-15">
   10| def foo():
   11|     pass
   12|
   13| def bar():
   14|     return 42
   15|
</file>"""
)
```

### 3. Handle Errors Helpfully

```python
# Bad - vague error
return ToolResult(
    output="Error: file not found"
)

# Good - actionable error
return ToolResult(
    output="""Error: File not found: src/mian.py

Did you mean one of these?
  src/main.py
  src/util.py

Current directory: /home/user/project
Files in src/: main.py, util.py, config.py"""
)
```

### 4. Truncate Intelligently

```python
# Bad - arbitrary cut
output = file_content[:10000]

# Good - cut at logical boundary with context
MAX_LINES = 2000
lines = file_content.split("\n")
if len(lines) > MAX_LINES:
    output = "\n".join(lines[:MAX_LINES])
    output += f"\n\n... ({len(lines) - MAX_LINES} more lines)"
    output += f"\nUse offset={MAX_LINES} to continue reading"
```

---

## Summary: Tool Design Principles

1. **Clear descriptions** - The LLM only knows what you tell it
2. **Validated inputs** - Use schemas to catch errors early
3. **Permission checks** - Ask before dangerous operations
4. **Helpful errors** - Include suggestions and context
5. **Structured output** - Make it easy for the LLM to parse
6. **Metadata for UI** - Separate what the LLM sees from what the user sees

---

## Assignment: Build File Tools

Now it's time to build real tools for your agent.

**Goal**: Add file reading, writing, and editing to your agent.

**Start here**: [Assignment 3: File Tools](./assignments/03-file-tools.md)

---

## Optional Deep Dives

Want to understand the tool system in more detail?

- 🔧 [Tool System Internals](/deep-dives/tool-system-internals.md) - Tool definition, validation, and execution
- 🔐 [Permissions](/deep-dives/permissions.md) - How permission checks protect tool execution
