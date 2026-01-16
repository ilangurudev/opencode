# Coding Agent Architecture Curriculum

> **Learn how coding agents work by studying opencode's architecture and building your own.**

This 2-week curriculum takes you from "I know LLMs can use tools" to "I understand how production coding agents are built and can create my own."

---

## How This Curriculum Works

**Each lesson has two parts:**
1. **Core Concepts** (~20-30 min read) - The essential ideas, with diagrams and code examples
2. **Deep Dive** (optional, ~30-60 min) - Detailed code walkthroughs, edge cases, design decisions

**Each day has an assignment:**
- You'll incrementally build a working coding agent in Python
- By Day 14, you'll have a functional agent with tools, permissions, memory, and extensibility
- Assignments are substantial (~100+ lines) - use AI assistance, but drive the design yourself

---

## Prerequisites

- Basic understanding of LLMs and API calls
- Familiarity with the concept of "tool calling" (function calling)
- Python proficiency (the assignments are in Python)
- TypeScript reading ability (opencode is in TypeScript, but I'll explain the code)

---

## The Big Picture

Before diving in, here's what a coding agent actually is:

```
┌─────────────────────────────────────────────────────────────────┐
│                        CODING AGENT                              │
│                                                                  │
│   ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐     │
│   │  User   │───▶│  Main   │───▶│   LLM   │───▶│  Tool   │     │
│   │ Input   │    │  Loop   │    │  Call   │    │ Execute │     │
│   └─────────┘    └────┬────┘    └─────────┘    └────┬────┘     │
│                       │                              │          │
│                       │         ┌─────────┐         │          │
│                       └─────────│ Continue│◀────────┘          │
│                                 │    ?    │                     │
│                                 └─────────┘                     │
│                                                                  │
│   Supporting Systems:                                           │
│   • Permission checks    • Context management                   │
│   • Message history      • Error recovery                       │
│   • Tool registry        • Extensibility (skills, MCP)         │
└─────────────────────────────────────────────────────────────────┘
```

A coding agent is fundamentally **an LLM in a loop** that can:
1. Receive a task from a user
2. Decide what action to take (respond or use a tool)
3. Execute that action
4. Repeat until the task is complete

The complexity comes from making this robust, safe, and useful at scale.

---

## Curriculum Overview

### Week 1: Understanding the Architecture

| Day | Topic | What You'll Learn | Assignment |
|-----|-------|-------------------|------------|
| [1-2](./day-01-02-overview.md) | The 10,000-foot View | What makes a coding agent, core components, data flow | Build simplest possible agent loop |
| [3-4](./day-03-04-main-loop.md) | The Main Loop | How opencode orchestrates user→LLM→tool→repeat | Add streaming and tool dispatch |
| [5-6](./day-05-06-tools.md) | The Tool System | Tool definition, registration, execution, output formatting | Build file and bash tools |

### Week 2: Making It Real

| Day | Topic | What You'll Learn | Assignment |
|-----|-------|-------------------|------------|
| [7-8](./day-07-08-agents.md) | Agents & Permissions | Different modes, permission rules, subagents | Add permission system |
| [9-10](./day-09-10-memory.md) | Context & Memory | Message storage, compaction, token management | Add conversation memory |
| [11-12](./day-11-12-extensibility.md) | Extensibility | Skills, commands, MCP, hooks | Add skill/plugin system |
| [13-14](./day-13-14-capstone.md) | Capstone | Full architecture review, your complete agent | Polish and extend your agent |

---

## Deep Dives (Optional)

These go much deeper into specific subsystems:

| Topic | What's Covered |
|-------|----------------|
| [The Processor State Machine](./deep-dives/processor.md) | Every state in the stream processor, error recovery |
| [Permission System Internals](./deep-dives/permissions.md) | Rule matching, caching, the ask/allow/deny flow |
| [Compaction & Summarization](./deep-dives/compaction.md) | How long conversations are compressed |
| [MCP Protocol Deep Dive](./deep-dives/mcp.md) | Building and connecting external tool servers |
| [The Event Bus](./deep-dives/event-bus.md) | Pub/sub architecture for decoupled components |
| [Provider Abstraction](./deep-dives/providers.md) | Supporting multiple LLM providers |

---

## Your Mini-Agent: What You'll Build

By the end of this curriculum, you'll have built a Python coding agent with:

```
your-agent/
├── agent.py           # Main loop and orchestration
├── llm.py             # LLM provider interface
├── tools/
│   ├── registry.py    # Tool registration and dispatch
│   ├── bash.py        # Shell command execution
│   ├── read.py        # File reading
│   ├── write.py       # File writing
│   └── edit.py        # File editing
├── permissions.py     # Ask/allow/deny system
├── memory.py          # Conversation history and context
├── skills.py          # User-defined prompt templates
└── config.py          # Configuration management
```

Each day adds a piece. By Day 14, it all works together.

---

## How to Use This Curriculum

1. **Read the core concepts** for each day (~20-30 min)
2. **Do the assignment** (~30-60 min with AI assistance)
3. **Optionally read the deep dive** if you want more detail
4. **Your agent grows** with each day's work

Ready? Let's start with [Day 1-2: The 10,000-foot View](./day-01-02-overview.md).

---

## Quick Reference: opencode's Architecture

For reference throughout the curriculum, here's where things live in opencode:

```
packages/opencode/src/
├── session/
│   ├── prompt.ts       # THE MAIN LOOP - start here
│   ├── processor.ts    # Stream processing state machine
│   ├── llm.ts          # LLM call wrapper
│   ├── message-v2.ts   # Message format and storage
│   ├── compaction.ts   # Context summarization
│   └── system.ts       # System prompt construction
├── tool/
│   ├── tool.ts         # Tool interface definition
│   └── tools/          # Individual tool implementations
├── agent/
│   └── agent.ts        # Agent definitions and registry
├── permission/
│   └── next.ts         # Permission system
├── skill/
│   └── skill.ts        # User-defined skills
├── mcp/
│   └── index.ts        # Model Context Protocol integration
├── config/
│   └── config.ts       # Configuration management
└── bus/
    └── index.ts        # Event pub/sub system
```
