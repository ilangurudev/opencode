# Day 1-2: The 10,000-foot View

> **Goal**: Understand what makes a coding agent different from a chatbot, and see the high-level architecture of opencode.

---

## What Is a Coding Agent?

Let's start with what you probably already know: an LLM can generate code. You paste some code, ask a question, get an answer. That's a **chatbot**.

A **coding agent** is different. It's autonomous. You give it a task like "fix this bug" or "add a login feature" and it:

1. Reads files to understand the codebase
2. Decides what to do
3. Makes changes
4. Runs tests to verify
5. Repeats until done (or asks for help)

The key insight: **an agent is an LLM in a loop with access to tools**.

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│    CHATBOT                        AGENT                      │
│    ───────                        ─────                      │
│                                                              │
│    User ──▶ LLM ──▶ Response      User ──▶ ┌─────────────┐  │
│                                            │   L O O P   │  │
│    One shot.                               │             │  │
│    No memory.                              │  LLM ◀──┐   │  │
│    No tools.                               │   │     │   │  │
│                                            │   ▼     │   │  │
│                                            │ Tool ───┘   │  │
│                                            │             │  │
│                                            └──────┬──────┘  │
│                                                   │         │
│                                                   ▼         │
│                                              Done/Response  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

## The Core Concept: Tool Calling

Before we dive into architecture, let's make sure "tool calling" is crystal clear.

When you call an LLM API, you can tell it: "Here are some tools you can use." The LLM doesn't actually run the tools - it just tells you which tool to call and with what arguments.

**Python example of what tool calling looks like:**

```python
# You send this to the LLM
messages = [
    {"role": "user", "content": "What files are in the src/ directory?"}
]

tools = [
    {
        "name": "list_files",
        "description": "List files in a directory",
        "parameters": {
            "type": "object",
            "properties": {
                "path": {"type": "string", "description": "Directory path"}
            }
        }
    }
]

response = client.chat(messages=messages, tools=tools)

# The LLM responds with something like:
# {
#     "tool_calls": [{
#         "name": "list_files",
#         "arguments": {"path": "src/"}
#     }]
# }

# YOUR CODE executes the tool, then sends the result back to the LLM
# The LLM can then respond to the user or call more tools
```

The LLM decides *what* tool to call. Your code *executes* it. This separation is fundamental.

---

## opencode's Architecture: The Bird's Eye View

Here's how opencode is structured at the highest level:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              opencode                                    │
│                                                                          │
│  ┌─────────────┐     ┌─────────────────────────────────────────────┐    │
│  │    TUI      │     │              Core Engine                     │    │
│  │  (Terminal  │────▶│                                              │    │
│  │    UI)      │     │   ┌─────────┐   ┌─────────┐   ┌─────────┐   │    │
│  └─────────────┘     │   │ Session │──▶│  Main   │──▶│   LLM   │   │    │
│                      │   │ Manager │   │  Loop   │   │Interface│   │    │
│                      │   └─────────┘   └────┬────┘   └─────────┘   │    │
│                      │                      │                       │    │
│                      │              ┌───────┴───────┐               │    │
│                      │              ▼               ▼               │    │
│                      │        ┌──────────┐   ┌──────────┐          │    │
│                      │        │   Tool   │   │Permission│          │    │
│                      │        │ Registry │   │  System  │          │    │
│                      │        └──────────┘   └──────────┘          │    │
│                      │                                              │    │
│                      └─────────────────────────────────────────────┘    │
│                                                                          │
│  Supporting Systems:                                                     │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐      │
│  │  Agent   │ │  Config  │ │  Skills  │ │   MCP    │ │  Event   │      │
│  │ Registry │ │  System  │ │  System  │ │ Servers  │ │   Bus    │      │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘      │
└─────────────────────────────────────────────────────────────────────────┘
```

Let's define each piece:

### Core Components

| Component | What It Does | In opencode |
|-----------|--------------|-------------|
| **Session Manager** | Stores conversation history, manages state | `session/index.ts` |
| **Main Loop** | The `while(true)` that orchestrates everything | `session/prompt.ts` |
| **LLM Interface** | Calls the AI model, handles streaming | `session/llm.ts` |
| **Tool Registry** | Registers and executes tools | `tool/registry.ts` |
| **Permission System** | Asks user before dangerous operations | `permission/next.ts` |

### Supporting Systems

| Component | What It Does | In opencode |
|-----------|--------------|-------------|
| **Agent Registry** | Different "modes" (explore, build, plan) | `agent/agent.ts` |
| **Config System** | User settings, model configuration | `config/config.ts` |
| **Skills System** | User-defined prompt templates | `skill/skill.ts` |
| **MCP Servers** | External tool providers | `mcp/index.ts` |
| **Event Bus** | Pub/sub for decoupled communication | `bus/index.ts` |

---

## The Main Loop: Heart of the Agent

The most important thing to understand is the main loop. Everything else supports it.

Here's the conceptual flow in Python-like pseudocode:

```python
def main_loop(session_id: str):
    """The heart of a coding agent."""

    while True:
        # 1. Get conversation history
        messages = get_messages(session_id)

        # 2. Find the last user message
        last_user = find_last_user_message(messages)
        last_assistant = find_last_assistant_message(messages)

        # 3. Should we continue?
        if should_stop(last_assistant):
            break

        # 4. Build the LLM request
        tools = get_available_tools()
        system_prompt = build_system_prompt()

        # 5. Call the LLM
        response = call_llm(
            messages=messages,
            tools=tools,
            system=system_prompt
        )

        # 6. Process the response
        if response.has_tool_calls:
            for tool_call in response.tool_calls:
                # Execute each tool
                result = execute_tool(tool_call)
                # Add result to conversation
                add_tool_result(session_id, tool_call, result)
            # Loop continues - LLM will see tool results
            continue

        # 7. LLM just responded with text - we're done (for this turn)
        save_response(session_id, response)
        break
```

This is the conceptual version. The actual opencode implementation in `session/prompt.ts` is more complex because it handles:

- Streaming responses (showing text as it arrives)
- Error recovery and retries
- Context overflow (compaction)
- Subagents (nested agent calls)
- Permission checks
- Multiple providers (Anthropic, OpenAI, etc.)

But the core idea is the same: **loop until the LLM stops calling tools**.

---

## When Does the Loop Stop?

This is a crucial question. The LLM can:

1. **Respond with text only** → Stop (task complete or needs user input)
2. **Call tools** → Continue (execute tools, send results back)
3. **Hit an error** → Stop (with error message)
4. **Be cancelled by user** → Stop

In opencode, this decision happens here (simplified):

```typescript
// From session/prompt.ts (simplified)
if (
    lastAssistant?.finish &&
    !["tool-calls", "unknown"].includes(lastAssistant.finish) &&
    lastUser.id < lastAssistant.id
) {
    // LLM finished without tool calls, exit loop
    break;
}
```

The `finish` field comes from the LLM API - it tells us *why* the model stopped generating:
- `"stop"` - Natural end of response
- `"tool-calls"` - Model wants to call tools
- `"length"` - Hit token limit
- `"content-filter"` - Content was filtered

---

## The Tool Lifecycle

When the LLM decides to call a tool, here's what happens:

```
┌─────────────────────────────────────────────────────────────────┐
│                        Tool Lifecycle                            │
│                                                                  │
│   1. LLM Request                                                │
│      ┌──────────────┐                                           │
│      │ "Call the    │                                           │
│      │ read tool    │                                           │
│      │ with path    │                                           │
│      │ src/main.ts" │                                           │
│      └──────┬───────┘                                           │
│             │                                                    │
│   2. Permission Check                                           │
│             ▼                                                    │
│      ┌──────────────┐                                           │
│      │ Is this      │──▶ Denied? ──▶ Return error to LLM       │
│      │ allowed?     │                                           │
│      └──────┬───────┘                                           │
│             │ Allowed                                            │
│             ▼                                                    │
│   3. Tool Execution                                             │
│      ┌──────────────┐                                           │
│      │ Read the     │                                           │
│      │ file from    │                                           │
│      │ disk         │                                           │
│      └──────┬───────┘                                           │
│             │                                                    │
│   4. Result Formatting                                          │
│             ▼                                                    │
│      ┌──────────────┐                                           │
│      │ Format as    │                                           │
│      │ tool result  │                                           │
│      │ message      │                                           │
│      └──────┬───────┘                                           │
│             │                                                    │
│   5. Back to LLM                                                │
│             ▼                                                    │
│      ┌──────────────┐                                           │
│      │ Add to       │                                           │
│      │ conversation,│                                           │
│      │ continue     │                                           │
│      │ loop         │                                           │
│      └──────────────┘                                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Key Insight: The Prompt Is Everything

The system prompt is what makes a coding agent behave like a coding agent. Without it, the LLM would just be a generic assistant.

opencode's system prompt (in `session/prompt/`) tells the LLM:

1. **What it is**: "You are a coding assistant..."
2. **What tools it has**: Descriptions of each tool
3. **How to behave**: When to use tools, when to ask questions
4. **Constraints**: Don't delete files without asking, etc.

Here's a tiny excerpt from opencode's Anthropic prompt:

```
You are an interactive CLI tool that helps users with software engineering tasks.
Use the instructions below and the tools available to you to assist the user.

# Doing tasks
The user will primarily request you perform software engineering tasks...
For these tasks the following steps are recommended:
1. Use the TodoWrite tool to plan the task if required
2. Implement the solution using available tools
3. Verify the solution works
```

The prompt is hundreds of lines. It's the "personality" and "knowledge" that makes the agent effective.

---

## What Makes opencode Different?

opencode isn't just a basic agent loop. It has several sophisticated features:

### 1. Multiple Agent Modes

Different tasks need different capabilities:

| Agent | Purpose | Tools Available |
|-------|---------|-----------------|
| `build` | Default mode, full capabilities | All tools |
| `explore` | Read-only exploration | Only read/search tools |
| `plan` | Planning mode | Limited tools, focused on thinking |
| `compaction` | Internal: summarize long conversations | No tools |

### 2. Permission System

Before executing certain tools, opencode asks:

```
┌─────────────────────────────────────────────┐
│ opencode wants to run:                      │
│                                             │
│   bash: rm -rf node_modules                 │
│                                             │
│ [Allow] [Allow Always] [Deny]               │
└─────────────────────────────────────────────┘
```

The permission system uses rules like:
- `bash`: ask (always ask user)
- `read`: allow (safe operation)
- `read *.env`: ask (sensitive files)

### 3. Context Management (Compaction)

Conversations can get long. LLMs have context limits. opencode handles this with "compaction":

1. Detect when context is getting full
2. Use a separate LLM call to summarize the conversation
3. Replace old messages with the summary
4. Continue with room to spare

### 4. MCP (Model Context Protocol)

MCP lets you add external tools without modifying opencode:

```
┌─────────────┐        ┌─────────────┐
│  opencode   │◀──────▶│ MCP Server  │
│             │  JSON  │ (e.g., DB   │
│             │  RPC   │  access)    │
└─────────────┘        └─────────────┘
```

You can write your own MCP servers to give the agent new capabilities.

---

## Looking at the Code: Where Things Live

Let's map concepts to files:

```
packages/opencode/src/
│
├── session/
│   ├── prompt.ts        ← THE MAIN LOOP (start here!)
│   ├── processor.ts     ← Handles streaming, tool results
│   ├── llm.ts           ← Calls the LLM API
│   ├── message-v2.ts    ← Message format and storage
│   ├── compaction.ts    ← Summarizes long conversations
│   └── prompt/          ← System prompt templates
│       ├── anthropic.txt
│       ├── codex.txt
│       └── ...
│
├── tool/
│   ├── tool.ts          ← Tool interface definition
│   ├── registry.ts      ← Tool registration
│   ├── read.ts          ← Read tool implementation
│   ├── bash.ts          ← Bash tool implementation
│   └── ...
│
├── agent/
│   └── agent.ts         ← Agent definitions (build, explore, etc.)
│
├── permission/
│   └── next.ts          ← Permission checking logic
│
├── skill/
│   └── skill.ts         ← User-defined prompt templates
│
├── mcp/
│   └── index.ts         ← MCP server connections
│
└── config/
    └── config.ts        ← Configuration management
```

---

## Summary: The Mental Model

Here's your mental model for understanding coding agents:

```
┌──────────────────────────────────────────────────────────────┐
│                    CODING AGENT = 4 THINGS                   │
│                                                              │
│   1. A LOOP         while (not done):                        │
│                         response = llm(messages, tools)      │
│                         if response.has_tools:               │
│                             execute_tools(response.tools)    │
│                                                              │
│   2. TOOLS          Things the LLM can do:                   │
│                     read files, edit files, run commands     │
│                                                              │
│   3. MEMORY         Conversation history that persists       │
│                     across loop iterations                   │
│                                                              │
│   4. PROMPT         Instructions that tell the LLM           │
│                     how to behave                            │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

Everything else - permissions, agents, compaction, MCP - are refinements of these core ideas.

---

## What's Next?

In **Day 3-4**, we'll dive deep into the main loop, walking through `session/prompt.ts` line by line.

But first, let's cement this understanding with an assignment.

---

## Assignment: Build the Simplest Possible Agent

**Goal**: Create a minimal Python agent that demonstrates the core loop.

Your agent should:
1. Accept user input
2. Call an LLM with tools available
3. Execute tools if requested
4. Loop until the LLM responds without tool calls

**Start here**: [Assignment 1: Minimal Agent Loop](./assignments/01-minimal-agent.md)

---

## Optional Deep Dives

Want to understand the architecture in more detail?

- 📖 [Architecture Overview](/deep-dives/overview.md) - Codebase structure, design patterns, event flow
- 🏗️ [Event Bus](/deep-dives/event-bus.md) - How the engine decouples from the UI
- 🔧 [Provider Abstraction](/deep-dives/provider-abstraction.md) - Multi-LLM support
