# Assignment 2: Streaming and Tool Dispatch

> **Time**: ~60 minutes
> **Goal**: Add streaming output and proper tool dispatch to your agent.

---

## What You'll Build

Building on Assignment 1, you'll add:
1. **Streaming output** - Text appears character by character
2. **Step tracking** - Show "Step 1", "Step 2" as the agent works
3. **Tool dispatch pattern** - Clean separation of tool registration and execution
4. **Better error handling** - Graceful recovery from failures

---

## The Challenge

### Part 1: Streaming Output

Instead of waiting for the full response, print text as it arrives:

```
User: What files are here?

--- Step 1 ---
Assistant: I'll list the files in the current directory.
[Tool] list_files(path=".")
[Result] file1.py
         file2.py
         README.md

--- Step 2 ---
Assistant: I can see three files: file1.py, file2.py, and README.md.
```

### Part 2: Tool Registry Pattern

Create a clean tool registry:

```python
# Register tools declaratively
@tool(
    name="list_files",
    description="List files in a directory",
    parameters={
        "path": {"type": "string", "description": "Directory path", "default": "."}
    }
)
def list_files(path: str = ".") -> str:
    ...

# Dispatch by name
result = tool_registry.execute("list_files", {"path": "."})
```

### Part 3: Step Tracking

Track and display:
- Current step number
- Total tokens used
- Tool calls made

---

## Starter Code

```python
#!/usr/bin/env python3
"""
Streaming Agent - Assignment 2

Adds streaming output and proper tool dispatch to the minimal agent.
"""

import os
import sys
from typing import Callable, Any
from dataclasses import dataclass, field
from anthropic import Anthropic

client = Anthropic()


# ============================================================================
# TOOL REGISTRY
# ============================================================================

@dataclass
class ToolDefinition:
    """Definition of a tool for the LLM."""
    name: str
    description: str
    parameters: dict
    function: Callable[..., str]


class ToolRegistry:
    """Registry for tools that the agent can use."""

    def __init__(self):
        self._tools: dict[str, ToolDefinition] = {}

    def register(
        self,
        name: str,
        description: str,
        parameters: dict,
    ) -> Callable:
        """Decorator to register a tool."""
        def decorator(func: Callable[..., str]) -> Callable[..., str]:
            self._tools[name] = ToolDefinition(
                name=name,
                description=description,
                parameters=parameters,
                function=func,
            )
            return func
        return decorator

    def get_definitions(self) -> list[dict]:
        """Get tool definitions in API format."""
        return [
            {
                "name": tool.name,
                "description": tool.description,
                "input_schema": {
                    "type": "object",
                    "properties": tool.parameters,
                    "required": [
                        k for k, v in tool.parameters.items()
                        if "default" not in v
                    ],
                },
            }
            for tool in self._tools.values()
        ]

    def execute(self, name: str, arguments: dict) -> str:
        """Execute a tool by name."""
        if name not in self._tools:
            return f"Error: Unknown tool '{name}'"

        tool = self._tools[name]
        try:
            return tool.function(**arguments)
        except Exception as e:
            return f"Error executing {name}: {e}"


# Create global registry
registry = ToolRegistry()


# ============================================================================
# TOOL DEFINITIONS
# ============================================================================

@registry.register(
    name="list_files",
    description="List files and directories in a given path.",
    parameters={
        "path": {
            "type": "string",
            "description": "The directory path to list. Defaults to current directory.",
            "default": ".",
        }
    },
)
def list_files(path: str = ".") -> str:
    """List files in a directory."""
    # TODO: Implement (copy from Assignment 1 or improve)
    pass


@registry.register(
    name="read_file",
    description="Read the contents of a file.",
    parameters={
        "path": {
            "type": "string",
            "description": "The path to the file to read.",
        }
    },
)
def read_file(path: str) -> str:
    """Read a file and return its contents."""
    # TODO: Implement (copy from Assignment 1 or improve)
    pass


# ============================================================================
# STEP TRACKING
# ============================================================================

@dataclass
class StepStats:
    """Statistics for a single step."""
    step_number: int
    input_tokens: int = 0
    output_tokens: int = 0
    tool_calls: list[str] = field(default_factory=list)


@dataclass
class SessionStats:
    """Statistics for the entire session."""
    steps: list[StepStats] = field(default_factory=list)

    @property
    def total_input_tokens(self) -> int:
        return sum(s.input_tokens for s in self.steps)

    @property
    def total_output_tokens(self) -> int:
        return sum(s.output_tokens for s in self.steps)

    @property
    def total_tool_calls(self) -> int:
        return sum(len(s.tool_calls) for s in self.steps)

    def print_summary(self):
        print(f"\n{'='*40}")
        print(f"Session Summary:")
        print(f"  Steps: {len(self.steps)}")
        print(f"  Input tokens: {self.total_input_tokens}")
        print(f"  Output tokens: {self.total_output_tokens}")
        print(f"  Tool calls: {self.total_tool_calls}")
        print(f"{'='*40}")


# ============================================================================
# STREAMING AGENT
# ============================================================================

def run_agent(user_message: str):
    """
    The main agent loop with streaming.

    Key differences from Assignment 1:
    1. Uses streaming API to show text as it arrives
    2. Tracks steps and statistics
    3. Uses tool registry pattern
    """
    print(f"\n{'='*60}")
    print(f"User: {user_message}")
    print('='*60)

    messages = [{"role": "user", "content": user_message}]
    stats = SessionStats()
    step = 0

    while True:
        step += 1
        step_stats = StepStats(step_number=step)
        stats.steps.append(step_stats)

        print(f"\n--- Step {step} ---")

        # TODO: Implement streaming
        #
        # Use client.messages.stream() instead of client.messages.create()
        #
        # with client.messages.stream(
        #     model="claude-sonnet-4-20250514",
        #     max_tokens=4096,
        #     tools=registry.get_definitions(),
        #     messages=messages,
        # ) as stream:
        #     for event in stream:
        #         # Handle different event types:
        #         # - message_start: beginning of response
        #         # - content_block_start: new text or tool block
        #         # - content_block_delta: incremental text
        #         # - content_block_stop: end of block
        #         # - message_delta: usage info
        #         # - message_stop: end of message
        #
        # After stream completes, get final message:
        # response = stream.get_final_message()

        # TODO: Process the response
        #
        # 1. Check response.stop_reason
        # 2. If "end_turn": print stats and break
        # 3. If "tool_use":
        #    - Add assistant message to messages
        #    - Execute each tool call
        #    - Add tool results to messages
        #    - Continue loop

        pass  # Remove this and implement


def print_streaming_text(text: str):
    """Print text character by character for streaming effect."""
    # TODO: Implement
    # Print each character with a small delay
    # Use sys.stdout.write() and sys.stdout.flush()
    pass


# ============================================================================
# MAIN
# ============================================================================

def main():
    print("Streaming Agent - Type 'quit' to exit")
    print("Try: 'What files are here and what's in the README?'\n")

    while True:
        try:
            user_input = input("You: ").strip()
            if user_input.lower() in ('quit', 'exit', 'q'):
                break
            if not user_input:
                continue
            run_agent(user_input)
        except KeyboardInterrupt:
            break

    print("\nGoodbye!")


if __name__ == "__main__":
    main()
```

---

## Understanding the Streaming API

The Anthropic streaming API sends events like this:

```python
with client.messages.stream(...) as stream:
    for event in stream:
        print(event.type)

# Output:
# message_start
# content_block_start    <- Text block starting
# content_block_delta    <- "Hello"
# content_block_delta    <- " world"
# content_block_stop     <- Text block done
# content_block_start    <- Tool block starting
# content_block_delta    <- tool input
# content_block_stop     <- Tool block done
# message_delta          <- Usage info
# message_stop           <- Done
```

Each event has different data:

```python
if event.type == "content_block_start":
    if event.content_block.type == "text":
        print("Starting text block")
    elif event.content_block.type == "tool_use":
        print(f"Starting tool: {event.content_block.name}")

elif event.type == "content_block_delta":
    if event.delta.type == "text_delta":
        # Stream the text!
        sys.stdout.write(event.delta.text)
        sys.stdout.flush()
    elif event.delta.type == "input_json_delta":
        # Tool input being streamed (partial JSON)
        pass

elif event.type == "message_delta":
    # Get token usage
    input_tokens = event.usage.input_tokens
    output_tokens = event.usage.output_tokens
```

---

## Success Criteria

Your agent works correctly if:

1. **Text streams** - Characters appear progressively, not all at once
2. **Steps are tracked** - See "Step 1", "Step 2", etc.
3. **Stats are shown** - Token counts and tool call counts at the end
4. **Multi-tool works** - "Read README and list all Python files" uses multiple tools

---

## Hints

<details>
<summary>Hint 1: Basic streaming loop</summary>

```python
with client.messages.stream(
    model="claude-sonnet-4-20250514",
    max_tokens=4096,
    tools=registry.get_definitions(),
    messages=messages,
) as stream:
    current_tool = None

    for event in stream:
        if event.type == "content_block_start":
            if hasattr(event.content_block, 'type'):
                if event.content_block.type == "text":
                    sys.stdout.write("\nAssistant: ")
                elif event.content_block.type == "tool_use":
                    current_tool = {
                        "id": event.content_block.id,
                        "name": event.content_block.name,
                        "input": "",
                    }

        elif event.type == "content_block_delta":
            if hasattr(event.delta, 'text'):
                sys.stdout.write(event.delta.text)
                sys.stdout.flush()
            elif hasattr(event.delta, 'partial_json'):
                if current_tool:
                    current_tool["input"] += event.delta.partial_json

        elif event.type == "message_delta":
            if hasattr(event.usage, 'input_tokens'):
                step_stats.input_tokens = event.usage.input_tokens
            if hasattr(event.usage, 'output_tokens'):
                step_stats.output_tokens = event.usage.output_tokens

    print()  # Newline after streaming
    response = stream.get_final_message()
```

</details>

<details>
<summary>Hint 2: Processing tool calls</summary>

```python
response = stream.get_final_message()

if response.stop_reason == "end_turn":
    stats.print_summary()
    break

if response.stop_reason == "tool_use":
    # Add assistant message
    messages.append({
        "role": "assistant",
        "content": response.content,
    })

    # Execute tools and collect results
    tool_results = []
    for block in response.content:
        if block.type == "tool_use":
            print(f"\n[Tool] {block.name}({block.input})")
            step_stats.tool_calls.append(block.name)

            result = registry.execute(block.name, block.input)
            print(f"[Result] {result[:100]}..." if len(result) > 100 else f"[Result] {result}")

            tool_results.append({
                "type": "tool_result",
                "tool_use_id": block.id,
                "content": result,
            })

    messages.append({"role": "user", "content": tool_results})
```

</details>

<details>
<summary>Full Solution</summary>

```python
#!/usr/bin/env python3
"""
Streaming Agent - Assignment 2 Solution
"""

import os
import sys
import time
from typing import Callable
from dataclasses import dataclass, field
from anthropic import Anthropic

client = Anthropic()


@dataclass
class ToolDefinition:
    name: str
    description: str
    parameters: dict
    function: Callable[..., str]


class ToolRegistry:
    def __init__(self):
        self._tools: dict[str, ToolDefinition] = {}

    def register(self, name: str, description: str, parameters: dict) -> Callable:
        def decorator(func: Callable[..., str]) -> Callable[..., str]:
            self._tools[name] = ToolDefinition(name, description, parameters, func)
            return func
        return decorator

    def get_definitions(self) -> list[dict]:
        return [
            {
                "name": t.name,
                "description": t.description,
                "input_schema": {
                    "type": "object",
                    "properties": t.parameters,
                    "required": [k for k, v in t.parameters.items() if "default" not in v],
                },
            }
            for t in self._tools.values()
        ]

    def execute(self, name: str, arguments: dict) -> str:
        if name not in self._tools:
            return f"Error: Unknown tool '{name}'"
        try:
            return self._tools[name].function(**arguments)
        except Exception as e:
            return f"Error: {e}"


registry = ToolRegistry()


@registry.register(
    name="list_files",
    description="List files in a directory.",
    parameters={"path": {"type": "string", "description": "Directory path", "default": "."}},
)
def list_files(path: str = ".") -> str:
    try:
        entries = sorted(os.listdir(path))
        return "\n".join(entries) if entries else "(empty directory)"
    except Exception as e:
        return f"Error: {e}"


@registry.register(
    name="read_file",
    description="Read a file's contents.",
    parameters={"path": {"type": "string", "description": "File path"}},
)
def read_file(path: str) -> str:
    try:
        with open(path, 'r') as f:
            content = f.read()
        if len(content) > 5000:
            return content[:5000] + f"\n... (truncated, {len(content)} total chars)"
        return content
    except Exception as e:
        return f"Error: {e}"


@dataclass
class StepStats:
    step_number: int
    input_tokens: int = 0
    output_tokens: int = 0
    tool_calls: list[str] = field(default_factory=list)


@dataclass
class SessionStats:
    steps: list[StepStats] = field(default_factory=list)

    def print_summary(self):
        total_in = sum(s.input_tokens for s in self.steps)
        total_out = sum(s.output_tokens for s in self.steps)
        total_tools = sum(len(s.tool_calls) for s in self.steps)
        print(f"\n{'='*40}")
        print(f"Session: {len(self.steps)} steps, {total_in} in / {total_out} out tokens, {total_tools} tool calls")
        print('='*40)


def run_agent(user_message: str):
    print(f"\n{'='*60}")
    print(f"User: {user_message}")
    print('='*60)

    messages = [{"role": "user", "content": user_message}]
    stats = SessionStats()
    step = 0

    while True:
        step += 1
        step_stats = StepStats(step_number=step)
        stats.steps.append(step_stats)
        print(f"\n--- Step {step} ---")

        with client.messages.stream(
            model="claude-sonnet-4-20250514",
            max_tokens=4096,
            tools=registry.get_definitions(),
            messages=messages,
        ) as stream:
            current_tool = None

            for event in stream:
                if event.type == "content_block_start":
                    if hasattr(event.content_block, 'type'):
                        if event.content_block.type == "text":
                            sys.stdout.write("\nAssistant: ")
                        elif event.content_block.type == "tool_use":
                            current_tool = {"id": event.content_block.id, "name": event.content_block.name}

                elif event.type == "content_block_delta":
                    if hasattr(event.delta, 'text'):
                        sys.stdout.write(event.delta.text)
                        sys.stdout.flush()

                elif event.type == "message_delta":
                    if hasattr(event.usage, 'input_tokens'):
                        step_stats.input_tokens = event.usage.input_tokens
                    if hasattr(event.usage, 'output_tokens'):
                        step_stats.output_tokens = event.usage.output_tokens

            print()
            response = stream.get_final_message()

        if response.stop_reason == "end_turn":
            stats.print_summary()
            break

        if response.stop_reason == "tool_use":
            messages.append({"role": "assistant", "content": response.content})

            tool_results = []
            for block in response.content:
                if block.type == "tool_use":
                    print(f"\n[Tool] {block.name}({block.input})")
                    step_stats.tool_calls.append(block.name)
                    result = registry.execute(block.name, block.input)
                    preview = result[:200] + "..." if len(result) > 200 else result
                    print(f"[Result] {preview}")
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": result,
                    })

            messages.append({"role": "user", "content": tool_results})


def main():
    print("Streaming Agent - Type 'quit' to exit\n")
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

</details>

---

## Stretch Goals

1. **Add retry logic** - Retry on rate limits with exponential backoff
2. **Add cancellation** - Handle Ctrl+C gracefully mid-stream
3. **Token budget** - Stop if approaching a token limit
4. **Parallel tools** - If LLM calls multiple tools at once, show them together

---

## What You Learned

- How streaming APIs work
- The tool registry pattern
- Step tracking and statistics
- Event-driven processing

---

## Next Up

In **Day 5-6**, we'll build real tools: file reading, writing, editing, and bash execution.

→ [Continue to Day 5-6: The Tool System](../day-05-06-tools.md)
