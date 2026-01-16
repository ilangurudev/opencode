# Day 13-14: Capstone

> **Goal**: Review the complete architecture, polish your agent, and understand what's next.

---

## What You've Built

Over the past two weeks, you've built a coding agent with:

```
your-agent/
├── agent.py              # Main loop (Day 1-4)
│   ├── The while(true) loop
│   ├── LLM calling
│   └── Tool dispatch
│
├── tools/                # Tool system (Day 5-6)
│   ├── registry.py       # Registration and execution
│   ├── read.py           # File reading with line numbers
│   ├── write.py          # File creation
│   ├── edit.py           # Fuzzy string replacement
│   └── bash.py           # Command execution
│
├── permissions.py        # Permission system (Day 7-8)
│   ├── Rule evaluation
│   ├── Ask/allow/deny flow
│   └── Persistent approvals
│
├── agents.py             # Agent modes (Day 7-8)
│   ├── Build mode (full access)
│   └── Explore mode (read-only)
│
├── memory.py             # Context management (Day 9-10)
│   ├── Token counting
│   ├── Message storage
│   ├── Pruning
│   └── Compaction
│
└── extensions/           # Extensibility (Day 11-12)
    ├── skills.py         # User-defined prompts
    ├── commands.py       # Prompts with arguments
    └── mcp.py            # External tool servers
```

---

## The Complete Architecture

Here's how all the pieces fit together:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           CODING AGENT ARCHITECTURE                          │
│                                                                              │
│  ┌─────────────┐                                                            │
│  │    User     │                                                            │
│  │   Input     │                                                            │
│  └──────┬──────┘                                                            │
│         │                                                                    │
│         ▼                                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         MAIN LOOP                                    │   │
│  │                                                                      │   │
│  │   while True:                                                        │   │
│  │       messages = memory.get_messages()  ─────▶ Memory System        │   │
│  │       if should_stop(): break                                        │   │
│  │                                                                      │   │
│  │       tools = registry.get_tools(agent) ─────▶ Tool Registry        │   │
│  │       response = llm.call(messages, tools) ──▶ LLM Provider         │   │
│  │                                                                      │   │
│  │       if response.has_tools:                                         │   │
│  │           for tool_call in response.tools:                           │   │
│  │               permission.check(tool)    ─────▶ Permission System    │   │
│  │               result = tool.execute()   ─────▶ Tool Execution       │   │
│  │               memory.add(result)                                     │   │
│  │                                                                      │   │
│  │       if memory.needs_compaction():                                  │   │
│  │           memory.compact()              ─────▶ Compaction           │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │
│  │   Memory    │  │   Agents    │  │ Permissions │  │   Tools     │       │
│  │   System    │  │   Config    │  │   System    │  │   Registry  │       │
│  │             │  │             │  │             │  │             │       │
│  │ - Messages  │  │ - build     │  │ - Rules     │  │ - read      │       │
│  │ - Tokens    │  │ - explore   │  │ - Ask flow  │  │ - write     │       │
│  │ - Pruning   │  │ - plan      │  │ - Storage   │  │ - edit      │       │
│  │ - Compact   │  │ - custom    │  │             │  │ - bash      │       │
│  └─────────────┘  └─────────────┘  └─────────────┘  │ + MCP tools │       │
│                                                      └─────────────┘       │
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────┐     │
│  │                       EXTENSIONS                                   │     │
│  │                                                                    │     │
│  │  Skills              Commands              MCP Servers             │     │
│  │  ────────           ──────────            ────────────             │     │
│  │  /review            /fix $1               database                 │     │
│  │  /explain           /test $1              github                   │     │
│  │  /refactor          /pr $1                slack                    │     │
│  │                                                                    │     │
│  └───────────────────────────────────────────────────────────────────┘     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Key Patterns Summary

### 1. The Loop Pattern

```python
while True:
    messages = get_messages()
    if should_stop(messages):
        break
    response = call_llm(messages, tools)
    if response.has_tools:
        execute_tools(response.tools)
    else:
        break
```

**Why it works**: Simple state machine. LLM decides when to use tools, code executes them, loop continues until done.

### 2. The Tool Interface Pattern

```python
class Tool:
    id: str
    description: str
    parameters: Schema
    execute(args, ctx) -> Result
```

**Why it works**: Uniform interface means tools are interchangeable. Registry can manage any tool.

### 3. The Permission Rule Pattern

```python
rules = [
    Rule(permission="read", pattern="*.env", action="ask"),
    Rule(permission="read", pattern="*", action="allow"),
]
# First matching rule wins
```

**Why it works**: Simple, predictable evaluation. Users can understand and configure.

### 4. The Compaction Pattern

```python
if tokens > limit:
    summary = llm.summarize(old_messages)
    messages = [summary] + recent_messages
```

**Why it works**: LLM summarizes itself. Context stays bounded while preserving meaning.

### 5. The Extension Pattern

```python
# Skills: Markdown files → prompts
# Commands: Markdown with $1, $2 → parameterized prompts
# MCP: External processes → tools
```

**Why it works**: Users extend without modifying code. Simple formats (markdown, JSON-RPC).

---

## Comparing Your Agent to opencode

| Feature | Your Agent | opencode |
|---------|-----------|----------|
| Main loop | ~100 lines | ~500 lines (handles edge cases) |
| Tools | 4 tools | 20+ tools |
| Permissions | Basic rules | Complex patterns, doom loop detection |
| Memory | Basic compaction | Pruning, summaries, persistence |
| Extensions | Skills, Commands | Skills, Commands, MCP, Plugins, Hooks |
| Streaming | Basic | Full streaming with UI updates |
| Error handling | Basic try/catch | Retries, recovery, helpful errors |
| Multi-provider | Single provider | Multiple LLM providers |

Your agent has the **core architecture**. opencode adds production polish.

---

## Ideas for Extension

### Short-term improvements:

1. **Add more tools**
   - `grep` - Search file contents
   - `glob` - Find files by pattern
   - `git` - Version control operations

2. **Improve error handling**
   - Retry on rate limits
   - Suggest fixes for common errors
   - Show file suggestions on "not found"

3. **Add streaming**
   - Show text as it arrives
   - Show tool execution in real-time

### Medium-term features:

4. **Add subagents**
   - Task tool that spawns new agent
   - Explore agent for read-only research

5. **Add LSP integration**
   - Show syntax errors after edits
   - Go-to-definition support

6. **Add git integration**
   - Track changes
   - Create commits
   - Create PRs

### Advanced features:

7. **Add checkpoints/undo**
   - Save state before changes
   - Revert to previous state

8. **Add parallel tool execution**
   - Run independent tools concurrently
   - Merge results

9. **Build a TUI**
   - Rich terminal interface
   - File tree view
   - Diff viewer

---

## Final Assignment: Polish Your Agent

For the capstone, choose one or more improvements:

### Option A: Add Robustness
- Implement retry logic for API errors
- Add timeout handling for tools
- Improve error messages with suggestions

### Option B: Add a New Tool
- Implement `grep` for searching file contents
- Implement `git` for basic version control
- Implement a tool that queries an external API

### Option C: Add Streaming
- Use the streaming API
- Show text as it arrives
- Show tool execution progress

### Option D: Free Choice
- Pick something from the ideas list
- Or invent your own improvement

See [Assignment 7: Capstone](./assignments/07-capstone.md) for details.

---

## What You've Learned

### Core Concepts:
- **Agents are LLMs in a loop** with access to tools
- **Tools are the interface** between LLM and the world
- **Permissions keep it safe** with ask/allow/deny rules
- **Memory is bounded** with compaction and pruning
- **Extensions make it useful** with skills, commands, MCP

### Design Patterns:
- The main loop pattern
- The tool interface pattern
- The permission rule pattern
- The compaction pattern
- The extension pattern

### Real Code:
- You've studied opencode's TypeScript implementation
- You've built a working Python agent
- You understand how production agents are architected

---

## Where to Go From Here

### Study More Code:
- [opencode](https://github.com/anthropics/opencode) - The codebase you've been studying
- [Claude Code](https://github.com/anthropics/claude-code) - Anthropic's official CLI
- [Aider](https://github.com/paul-gauthier/aider) - Popular Python coding agent
- [Continue](https://github.com/continuedev/continue) - IDE extension agent

### Build Something:
- Extend your agent for a specific use case
- Build an MCP server for a service you use
- Create a specialized agent (docs writer, test generator, etc.)

### Learn More:
- [Anthropic's Tool Use Guide](https://docs.anthropic.com/claude/docs/tool-use)
- [MCP Specification](https://modelcontextprotocol.io/)
- [AI SDK Documentation](https://sdk.vercel.ai/docs)

---

## Congratulations!

You've completed the curriculum. You now understand:
- How coding agents are architected
- How to build your own agent
- How to extend and customize agents

Happy building!
