# Assignment 3: Build File Tools

> **Time**: ~60-90 minutes
> **Goal**: Add read, write, and edit tools to your agent.

---

## What You'll Build

Your agent will gain the ability to:
1. **Read files** - With line numbers, pagination, and image support
2. **Write files** - Create new files or overwrite existing ones
3. **Edit files** - Replace specific text in files (with fuzzy matching)

By the end, your agent can actually modify code!

---

## The Challenge

### Part 1: Enhanced Read Tool

Improve your read tool to:
- Show line numbers
- Support offset/limit for large files
- Detect and handle binary files
- Provide helpful "file not found" suggestions

### Part 2: Write Tool

Create a tool that:
- Creates new files or overwrites existing ones
- Creates parent directories if needed
- Shows a preview of what will be written
- Asks for confirmation before overwriting

### Part 3: Edit Tool

Create a tool that:
- Replaces `old_string` with `new_string`
- Fails gracefully if `old_string` isn't found
- Handles whitespace differences (fuzzy matching)
- Shows a diff of what changed

---

## Starter Code

```python
#!/usr/bin/env python3
"""
File Tools Agent - Assignment 3

Adds file read, write, and edit capabilities.
"""

import os
import sys
import difflib
from pathlib import Path
from typing import Generator
from dataclasses import dataclass, field
from pydantic import BaseModel, Field
from anthropic import Anthropic

client = Anthropic()


# ============================================================================
# TOOL INFRASTRUCTURE (from Assignment 2)
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

    def get_definitions(self) -> list[dict]:
        return [
            {
                "name": tool["id"],
                "description": tool["description"],
                "input_schema": tool["parameters"].model_json_schema(),
            }
            for tool in self._tools.values()
        ]

    def execute(self, name: str, args: dict) -> ToolResult:
        if name not in self._tools:
            return ToolResult(
                title="Error",
                output=f"Unknown tool: {name}",
                metadata={"error": True}
            )
        tool = self._tools[name]
        try:
            params = tool["parameters"](**args)
            return tool["function"](params)
        except Exception as e:
            return ToolResult(
                title="Error",
                output=f"Error executing {name}: {e}",
                metadata={"error": True}
            )


registry = ToolRegistry()


# ============================================================================
# READ TOOL
# ============================================================================

class ReadParams(BaseModel):
    file_path: str = Field(description="The path to the file to read")
    offset: int = Field(default=0, description="Line number to start from (0-based)")
    limit: int = Field(default=2000, description="Maximum number of lines to read")


@registry.register(
    id="read",
    description="""Read a file from the filesystem.

- Returns file contents with line numbers
- Use offset and limit to read portions of large files
- Handles text files; returns error for binary files

Example:
  read(file_path="src/main.py")
  read(file_path="large.log", offset=1000, limit=100)
""",
    parameters=ReadParams,
)
def read_file(params: ReadParams) -> ToolResult:
    """Read a file and return its contents with line numbers."""
    # TODO: Implement this
    #
    # 1. Resolve the path (handle relative paths)
    # 2. Check if file exists
    #    - If not, find similar files and suggest them
    # 3. Check if file is binary (has null bytes or > 30% non-printable)
    #    - If binary, return an error
    # 4. Read the file
    # 5. Apply offset and limit
    # 6. Format with line numbers:
    #    "   1| first line"
    #    "   2| second line"
    # 7. Add truncation message if file has more lines
    # 8. Return ToolResult with:
    #    - title: relative path
    #    - output: formatted content
    #    - metadata: {"truncated": bool, "total_lines": int}

    pass


# ============================================================================
# WRITE TOOL
# ============================================================================

class WriteParams(BaseModel):
    file_path: str = Field(description="The path to the file to write")
    content: str = Field(description="The content to write to the file")


@registry.register(
    id="write",
    description="""Write content to a file.

- Creates the file if it doesn't exist
- Creates parent directories if needed
- Overwrites existing content

Example:
  write(file_path="new_file.py", content="print('hello')")
""",
    parameters=WriteParams,
)
def write_file(params: WriteParams) -> ToolResult:
    """Write content to a file."""
    # TODO: Implement this
    #
    # 1. Resolve the path
    # 2. Create parent directories if needed
    # 3. Check if file already exists
    #    - Note this in the output
    # 4. Write the content
    # 5. Return ToolResult with:
    #    - title: relative path
    #    - output: success message with line count
    #    - metadata: {"created": bool, "overwritten": bool, "lines": int}

    pass


# ============================================================================
# EDIT TOOL
# ============================================================================

class EditParams(BaseModel):
    file_path: str = Field(description="The path to the file to edit")
    old_string: str = Field(description="The text to find and replace")
    new_string: str = Field(description="The text to replace it with")


def find_fuzzy_match(content: str, search: str) -> str | None:
    """
    Find a fuzzy match for search string in content.

    Tries multiple strategies:
    1. Exact match
    2. Whitespace-normalized match (ignore leading/trailing spaces per line)
    3. Block match (first and last lines as anchors)

    Returns the actual text from content that matches, or None.
    """
    # TODO: Implement fuzzy matching
    #
    # Strategy 1: Exact match
    if search in content:
        return search

    # Strategy 2: Line-by-line with trimmed whitespace
    # TODO: Implement

    # Strategy 3: First/last line anchors
    # TODO: Implement

    return None


def generate_diff(old_content: str, new_content: str, file_path: str) -> str:
    """Generate a unified diff between old and new content."""
    old_lines = old_content.splitlines(keepends=True)
    new_lines = new_content.splitlines(keepends=True)

    diff = difflib.unified_diff(
        old_lines,
        new_lines,
        fromfile=f"a/{file_path}",
        tofile=f"b/{file_path}",
    )

    return "".join(diff)


@registry.register(
    id="edit",
    description="""Edit a file by replacing text.

- Finds old_string in the file and replaces it with new_string
- Uses fuzzy matching to handle whitespace differences
- Fails if old_string is not found or matches multiple locations
- Shows a diff of the changes

Example:
  edit(
    file_path="src/main.py",
    old_string="def foo():\\n    pass",
    new_string="def foo():\\n    return 42"
  )
""",
    parameters=EditParams,
)
def edit_file(params: EditParams) -> ToolResult:
    """Edit a file by replacing text."""
    # TODO: Implement this
    #
    # 1. Resolve the path
    # 2. Check file exists
    # 3. Read current content
    # 4. Find the match using fuzzy matching
    #    - If not found, return helpful error
    #    - If multiple matches, return error asking for more context
    # 5. Perform the replacement
    # 6. Generate diff
    # 7. Write the new content
    # 8. Return ToolResult with:
    #    - title: relative path
    #    - output: success message with diff
    #    - metadata: {"diff": str, "lines_changed": int}

    pass


# ============================================================================
# LIST FILES TOOL (keep from before)
# ============================================================================

class ListParams(BaseModel):
    path: str = Field(default=".", description="Directory path to list")


@registry.register(
    id="list_files",
    description="List files and directories in a path.",
    parameters=ListParams,
)
def list_files(params: ListParams) -> ToolResult:
    path = Path(params.path)
    if not path.exists():
        return ToolResult(
            title="Error",
            output=f"Directory not found: {path}",
            metadata={"error": True}
        )
    if not path.is_dir():
        return ToolResult(
            title="Error",
            output=f"Not a directory: {path}",
            metadata={"error": True}
        )

    entries = sorted(path.iterdir(), key=lambda p: (not p.is_dir(), p.name.lower()))
    lines = []
    for entry in entries:
        prefix = "d " if entry.is_dir() else "f "
        lines.append(f"{prefix}{entry.name}")

    return ToolResult(
        title=str(path),
        output="\n".join(lines) if lines else "(empty directory)",
        metadata={"count": len(entries)}
    )


# ============================================================================
# AGENT LOOP (from Assignment 2)
# ============================================================================

def run_agent(user_message: str):
    print(f"\n{'='*60}")
    print(f"User: {user_message}")
    print('='*60)

    messages = [{"role": "user", "content": user_message}]
    step = 0

    while True:
        step += 1
        print(f"\n--- Step {step} ---")

        with client.messages.stream(
            model="claude-sonnet-4-20250514",
            max_tokens=4096,
            tools=registry.get_definitions(),
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

                    result = registry.execute(block.name, block.input)

                    # Show truncated output
                    output_preview = result.output[:500]
                    if len(result.output) > 500:
                        output_preview += "..."
                    print(f"[Result] {output_preview}")

                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": result.output,
                    })

            messages.append({"role": "user", "content": tool_results})


def main():
    print("File Tools Agent - Type 'quit' to exit")
    print("Try: 'Read the main Python file in this directory'")
    print("Or: 'Create a hello.py file that prints hello world'\n")

    while True:
        try:
            user_input = input("You: ").strip()
            if user_input.lower() in ('quit', 'exit', 'q'):
                break
            if user_input:
                run_agent(user_input)
        except KeyboardInterrupt:
            break

    print("\nGoodbye!")


if __name__ == "__main__":
    main()
```

---

## Success Criteria

Your agent works correctly if it can:

1. **Read files**:
   - "Read main.py" → Shows content with line numbers
   - "Read lines 50-100 of large_file.txt" → Uses offset/limit
   - "Read nonexistent.txt" → Suggests similar files

2. **Write files**:
   - "Create test.py with a hello world function" → Creates file
   - "Write 'hello' to existing.txt" → Overwrites with notice

3. **Edit files**:
   - "Change 'foo' to 'bar' in main.py" → Edits with diff
   - "Fix the typo 'pritn' to 'print' in main.py" → Fuzzy match works
   - Shows helpful error if match not found

---

## Hints

<details>
<summary>Hint 1: Read tool implementation</summary>

```python
def read_file(params: ReadParams) -> ToolResult:
    path = Path(params.file_path)
    if not path.is_absolute():
        path = Path.cwd() / path
    path = path.resolve()

    if not path.exists():
        # Find similar files
        parent = path.parent
        suggestions = []
        if parent.exists():
            for f in parent.iterdir():
                if params.file_path.lower() in f.name.lower():
                    suggestions.append(str(f))

        msg = f"File not found: {path}"
        if suggestions:
            msg += "\n\nDid you mean:\n" + "\n".join(suggestions[:3])
        return ToolResult(title="Error", output=msg, metadata={"error": True})

    # Check binary
    try:
        content = path.read_text()
    except UnicodeDecodeError:
        return ToolResult(
            title="Error",
            output=f"Cannot read binary file: {path}",
            metadata={"error": True}
        )

    lines = content.split("\n")
    total = len(lines)
    selected = lines[params.offset:params.offset + params.limit]

    formatted = []
    for i, line in enumerate(selected):
        num = params.offset + i + 1
        formatted.append(f"{num:5}| {line}")

    output = "<file>\n" + "\n".join(formatted) + "\n</file>"

    last_line = params.offset + len(selected)
    if last_line < total:
        output += f"\n\n({total - last_line} more lines. Use offset={last_line} to continue.)"
    else:
        output += f"\n\n(End of file - {total} lines)"

    return ToolResult(
        title=str(path.relative_to(Path.cwd())),
        output=output,
        metadata={"truncated": last_line < total, "total_lines": total}
    )
```

</details>

<details>
<summary>Hint 2: Fuzzy matching implementation</summary>

```python
def find_fuzzy_match(content: str, search: str) -> str | None:
    # 1. Exact match
    if search in content:
        return search

    # 2. Line-trimmed match
    content_lines = content.split("\n")
    search_lines = search.split("\n")

    for i in range(len(content_lines) - len(search_lines) + 1):
        match = True
        for j, search_line in enumerate(search_lines):
            if content_lines[i + j].strip() != search_line.strip():
                match = False
                break
        if match:
            return "\n".join(content_lines[i:i + len(search_lines)])

    # 3. First/last line anchors (for multi-line with fuzzy middle)
    if len(search_lines) >= 3:
        first = search_lines[0].strip()
        last = search_lines[-1].strip()

        for i, line in enumerate(content_lines):
            if line.strip() != first:
                continue
            # Found first line, look for last
            for j in range(i + 2, len(content_lines)):
                if content_lines[j].strip() == last:
                    return "\n".join(content_lines[i:j + 1])

    return None
```

</details>

<details>
<summary>Hint 3: Edit tool implementation</summary>

```python
def edit_file(params: EditParams) -> ToolResult:
    path = Path(params.file_path)
    if not path.is_absolute():
        path = Path.cwd() / path

    if not path.exists():
        return ToolResult(
            title="Error",
            output=f"File not found: {path}",
            metadata={"error": True}
        )

    old_content = path.read_text()

    # Find match
    match = find_fuzzy_match(old_content, params.old_string)
    if match is None:
        return ToolResult(
            title="Error",
            output=(
                f"Could not find text to replace in {path}.\n\n"
                f"Searched for:\n{params.old_string[:200]}...\n\n"
                "Tip: Include more surrounding context to help locate the text."
            ),
            metadata={"error": True}
        )

    # Check uniqueness
    first_idx = old_content.find(match)
    last_idx = old_content.rfind(match)
    if first_idx != last_idx:
        return ToolResult(
            title="Error",
            output=(
                f"Found multiple matches in {path}.\n"
                "Please include more surrounding context to identify the specific location."
            ),
            metadata={"error": True}
        )

    # Replace
    new_content = old_content[:first_idx] + params.new_string + old_content[first_idx + len(match):]

    # Generate diff
    diff = generate_diff(old_content, new_content, str(path))

    # Write
    path.write_text(new_content)

    return ToolResult(
        title=str(path.relative_to(Path.cwd())),
        output=f"Edit applied successfully.\n\n{diff}",
        metadata={"diff": diff}
    )
```

</details>

---

## Stretch Goals

1. **Add `replace_all` option** - Replace all occurrences, not just the first
2. **Add undo** - Track changes and allow reverting
3. **Add backup** - Create `.bak` files before editing
4. **LSP integration** - Show syntax errors after editing

---

## What You Learned

- How to design robust tool interfaces
- Fuzzy matching strategies for real-world use
- Error handling with helpful suggestions
- Diff generation for showing changes

---

## Next Up

In **Day 7-8**, we'll add agents and permissions - different modes with different capabilities.

→ [Continue to Day 7-8: Agents & Permissions](../day-07-08-agents.md)
