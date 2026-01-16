# Assignment 1: Build a Minimal Agent Loop

> **Time**: ~45-60 minutes
> **Goal**: Create the simplest possible coding agent that demonstrates the core loop.

---

## What You'll Build

By the end of this assignment, you'll have a Python script that:
1. Takes user input
2. Sends it to an LLM with tools available
3. Executes tools when the LLM requests them
4. Loops until the LLM responds without tool calls

It won't be fancy, but it will be a *real* agent loop.

---

## Prerequisites

You'll need:
- Python 3.10+
- An API key for Anthropic or OpenAI
- `pip install anthropic` or `pip install openai`

---

## The Challenge

Create a file called `minimal_agent.py` with these requirements:

### 1. Define Two Simple Tools

```python
# Tool 1: List files in a directory
def list_files(path: str) -> str:
    """List files in the given directory."""
    # Return the list of files as a string
    pass

# Tool 2: Read a file
def read_file(path: str) -> str:
    """Read and return the contents of a file."""
    # Return the file contents
    pass
```

### 2. Implement the Agent Loop

Your main function should look conceptually like this:

```python
def run_agent(user_message: str):
    messages = [{"role": "user", "content": user_message}]

    while True:
        # Call the LLM with tools
        response = call_llm(messages, tools)

        # Check if the LLM wants to use tools
        if response.has_tool_calls:
            # Execute each tool call
            for tool_call in response.tool_calls:
                result = execute_tool(tool_call)
                # Add tool result to messages
                messages.append(make_tool_result(tool_call, result))
            continue  # Loop back

        # No tool calls - print response and exit
        print(response.text)
        break
```

### 3. Test It

Run your agent with prompts like:
- "What files are in the current directory?"
- "Read the contents of minimal_agent.py"
- "What files are here and what's in the README?"

---

## Starter Code

Here's a skeleton to get you started. Fill in the `TODO` sections:

```python
#!/usr/bin/env python3
"""
Minimal Agent Loop - Assignment 1

A bare-bones coding agent that demonstrates:
1. The main loop
2. Tool calling
3. Tool execution
"""

import os
import json
from anthropic import Anthropic

# Initialize the client
client = Anthropic()  # Uses ANTHROPIC_API_KEY env var

# ============================================================================
# TOOLS - Define what the agent can do
# ============================================================================

def list_files(path: str = ".") -> str:
    """List files in a directory."""
    # TODO: Implement this
    # Return a string listing the files
    pass


def read_file(path: str) -> str:
    """Read a file and return its contents."""
    # TODO: Implement this
    # Return the file contents as a string
    # Handle errors gracefully
    pass


# Tool definitions for the LLM
TOOLS = [
    {
        "name": "list_files",
        "description": "List files and directories in a given path. Returns a list of filenames.",
        "input_schema": {
            "type": "object",
            "properties": {
                "path": {
                    "type": "string",
                    "description": "The directory path to list. Defaults to current directory.",
                    "default": "."
                }
            },
            "required": []
        }
    },
    {
        "name": "read_file",
        "description": "Read the contents of a file and return it as text.",
        "input_schema": {
            "type": "object",
            "properties": {
                "path": {
                    "type": "string",
                    "description": "The path to the file to read."
                }
            },
            "required": ["path"]
        }
    }
]

# Map tool names to functions
TOOL_FUNCTIONS = {
    "list_files": list_files,
    "read_file": read_file,
}


# ============================================================================
# AGENT LOOP - The core of the agent
# ============================================================================

def execute_tool(tool_name: str, tool_input: dict) -> str:
    """Execute a tool and return the result as a string."""
    # TODO: Implement this
    # 1. Look up the tool function
    # 2. Call it with the input
    # 3. Return the result
    # 4. Handle errors
    pass


def run_agent(user_message: str):
    """
    The main agent loop.

    Takes a user message, calls the LLM, executes tools if needed,
    and loops until the LLM responds without tool calls.
    """
    print(f"\n{'='*60}")
    print(f"User: {user_message}")
    print('='*60)

    # Start with the user message
    messages = [{"role": "user", "content": user_message}]

    # TODO: Implement the main loop
    #
    # while True:
    #     1. Call client.messages.create() with:
    #        - model="claude-sonnet-4-20250514" (or your preferred model)
    #        - max_tokens=4096
    #        - tools=TOOLS
    #        - messages=messages
    #
    #     2. Check response.stop_reason
    #        - If "tool_use": extract tool calls, execute them, add results to messages
    #        - If "end_turn": print the text response and break
    #
    #     3. For tool calls:
    #        - The response.content is a list
    #        - Some items have type="tool_use" with name, input, id
    #        - Execute each tool
    #        - Add the assistant message to messages
    #        - Add tool results as a user message with tool_result content blocks
    #
    # Hint: Tool result format:
    # {
    #     "role": "user",
    #     "content": [
    #         {
    #             "type": "tool_result",
    #             "tool_use_id": tool_call.id,
    #             "content": result_string
    #         }
    #     ]
    # }

    pass  # Remove this and implement the loop


# ============================================================================
# MAIN
# ============================================================================

def main():
    """Interactive loop for testing the agent."""
    print("Minimal Agent - Type 'quit' to exit")
    print("Try: 'What files are in the current directory?'")
    print()

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

## Success Criteria

Your agent works correctly if:

1. **Single tool call works**: "What files are here?" → Lists files
2. **Multiple tool calls work**: "Read the first Python file you find" → Lists files, then reads one
3. **Loop terminates**: Agent stops after answering
4. **Errors are handled**: "Read nonexistent.txt" → Graceful error message

---

## Hints

<details>
<summary>Hint 1: Implementing list_files</summary>

```python
def list_files(path: str = ".") -> str:
    try:
        entries = os.listdir(path)
        return "\n".join(entries)
    except Exception as e:
        return f"Error listing {path}: {e}"
```

</details>

<details>
<summary>Hint 2: Implementing read_file</summary>

```python
def read_file(path: str) -> str:
    try:
        with open(path, 'r') as f:
            return f.read()
    except Exception as e:
        return f"Error reading {path}: {e}"
```

</details>

<details>
<summary>Hint 3: The main loop structure</summary>

```python
while True:
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=4096,
        tools=TOOLS,
        messages=messages
    )

    # Check if we should continue
    if response.stop_reason == "end_turn":
        # Extract and print text
        for block in response.content:
            if hasattr(block, 'text'):
                print(f"\nAssistant: {block.text}")
        break

    if response.stop_reason == "tool_use":
        # Add assistant message with tool calls
        messages.append({"role": "assistant", "content": response.content})

        # Process each tool call
        tool_results = []
        for block in response.content:
            if block.type == "tool_use":
                result = execute_tool(block.name, block.input)
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": result
                })

        # Add tool results
        messages.append({"role": "user", "content": tool_results})
```

</details>

<details>
<summary>Full solution</summary>

```python
#!/usr/bin/env python3
"""
Minimal Agent Loop - Assignment 1 Solution
"""

import os
from anthropic import Anthropic

client = Anthropic()


def list_files(path: str = ".") -> str:
    """List files in a directory."""
    try:
        entries = os.listdir(path)
        if not entries:
            return f"Directory '{path}' is empty"
        return "\n".join(sorted(entries))
    except FileNotFoundError:
        return f"Error: Directory '{path}' not found"
    except PermissionError:
        return f"Error: Permission denied for '{path}'"
    except Exception as e:
        return f"Error listing '{path}': {e}"


def read_file(path: str) -> str:
    """Read a file and return its contents."""
    try:
        with open(path, 'r', encoding='utf-8') as f:
            content = f.read()
        # Truncate very long files
        if len(content) > 10000:
            return content[:10000] + f"\n\n... [truncated, {len(content)} total chars]"
        return content
    except FileNotFoundError:
        return f"Error: File '{path}' not found"
    except PermissionError:
        return f"Error: Permission denied for '{path}'"
    except UnicodeDecodeError:
        return f"Error: '{path}' is not a text file"
    except Exception as e:
        return f"Error reading '{path}': {e}"


TOOLS = [
    {
        "name": "list_files",
        "description": "List files and directories in a given path.",
        "input_schema": {
            "type": "object",
            "properties": {
                "path": {
                    "type": "string",
                    "description": "The directory path to list. Defaults to current directory.",
                    "default": "."
                }
            },
            "required": []
        }
    },
    {
        "name": "read_file",
        "description": "Read the contents of a file.",
        "input_schema": {
            "type": "object",
            "properties": {
                "path": {
                    "type": "string",
                    "description": "The path to the file to read."
                }
            },
            "required": ["path"]
        }
    }
]

TOOL_FUNCTIONS = {
    "list_files": list_files,
    "read_file": read_file,
}


def execute_tool(tool_name: str, tool_input: dict) -> str:
    """Execute a tool and return the result."""
    if tool_name not in TOOL_FUNCTIONS:
        return f"Error: Unknown tool '{tool_name}'"

    func = TOOL_FUNCTIONS[tool_name]
    try:
        return func(**tool_input)
    except Exception as e:
        return f"Error executing {tool_name}: {e}"


def run_agent(user_message: str):
    """The main agent loop."""
    print(f"\n{'='*60}")
    print(f"User: {user_message}")
    print('='*60)

    messages = [{"role": "user", "content": user_message}]
    step = 0

    while True:
        step += 1
        print(f"\n--- Step {step} ---")

        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=4096,
            tools=TOOLS,
            messages=messages
        )

        print(f"Stop reason: {response.stop_reason}")

        # End of conversation
        if response.stop_reason == "end_turn":
            for block in response.content:
                if hasattr(block, 'text'):
                    print(f"\nAssistant: {block.text}")
            break

        # Tool use
        if response.stop_reason == "tool_use":
            # Add assistant message
            messages.append({
                "role": "assistant",
                "content": [
                    {"type": b.type, "id": getattr(b, 'id', None),
                     "name": getattr(b, 'name', None),
                     "input": getattr(b, 'input', None),
                     "text": getattr(b, 'text', None)}
                    for b in response.content
                ]
            })

            # Execute tools and collect results
            tool_results = []
            for block in response.content:
                if block.type == "tool_use":
                    print(f"Tool: {block.name}({block.input})")
                    result = execute_tool(block.name, block.input)
                    print(f"Result: {result[:200]}..." if len(result) > 200 else f"Result: {result}")
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": result
                    })

            # Add tool results
            messages.append({"role": "user", "content": tool_results})
            continue

        # Unknown stop reason
        print(f"Unexpected stop reason: {response.stop_reason}")
        break


def main():
    print("Minimal Agent - Type 'quit' to exit")
    print("Try: 'What files are in the current directory?'\n")

    while True:
        try:
            user_input = input("You: ").strip()
            if user_input.lower() in ('quit', 'exit', 'q'):
                break
            if not user_input:
                continue
            run_agent(user_input)
            print()
        except KeyboardInterrupt:
            break

    print("\nGoodbye!")


if __name__ == "__main__":
    main()
```

</details>

---

## Stretch Goals

If you finish early, try these extensions:

1. **Add a `write_file` tool** - Let the agent create files
2. **Add step counting** - Print "Step 1", "Step 2" as the loop runs
3. **Add token counting** - Track how many tokens you're using
4. **Stream the response** - Show text as it arrives instead of all at once

---

## What You Learned

After completing this assignment, you understand:

- The fundamental agent loop pattern
- How tool calling works in practice
- Why agents are "LLMs in a loop"
- The message format for tool results

---

## Next Up

In **Day 3-4**, we'll look at how opencode implements this same pattern with much more sophistication - streaming, error handling, retries, and more.

→ [Continue to Day 3-4: The Main Loop](../day-03-04-main-loop.md)
