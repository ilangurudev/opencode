# Deep Dive: opencode Architecture Overview

> **Time**: ~20-30 minutes
> **Prerequisites**: Complete Day 1-2 core concepts

This deep dive provides an overview of opencode's internal architecture. We'll look at the codebase structure, key design patterns, and how components connect.

---

## The Codebase Structure

opencode is a monorepo with multiple packages:

```
opencode/
├── packages/
│   ├── opencode/        ← Core agent engine (our focus)
│   ├── app/             ← TUI (terminal UI)
│   ├── web/             ← Web interface
│   ├── sdk/             ← SDK for programmatic access
│   ├── docs/            ← Documentation
│   └── ...
├── .opencode/           ← Configuration examples
└── themes/              ← Color themes
```

The core engine in `packages/opencode/src/` is what we care about:

```
packages/opencode/src/
├── session/             ← Conversation management (most important!)
│   ├── prompt.ts        ← Main loop
│   ├── processor.ts     ← Stream processing
│   ├── llm.ts           ← LLM API calls
│   ├── message-v2.ts    ← Message types and storage
│   ├── compaction.ts    ← Context compression
│   ├── summary.ts       ← Conversation summaries
│   ├── system.ts        ← System prompt building
│   └── prompt/          ← Prompt templates per provider
│
├── tool/                ← Tool system
│   ├── tool.ts          ← Tool interface
│   ├── registry.ts      ← Tool registration
│   ├── read.ts, bash.ts, edit.ts, etc.
│
├── agent/               ← Agent modes
│   └── agent.ts
│
├── permission/          ← Permission system
│   └── next.ts
│
├── mcp/                 ← External tools via MCP
│   └── index.ts
│
├── skill/               ← User-defined prompts
│   └── skill.ts
│
├── config/              ← Configuration
│   └── config.ts
│
├── bus/                 ← Event system
│   └── index.ts
│
└── provider/            ← LLM provider abstraction
    └── provider.ts
```

---

## Key Design Patterns

### 1. Namespaces Instead of Classes

opencode uses TypeScript namespaces extensively instead of classes:

```typescript
// Instead of:
class Session {
  static async get(id: string) { ... }
  static async create() { ... }
}

// opencode uses:
export namespace Session {
  export async function get(id: string) { ... }
  export async function create() { ... }
}
```

**Why?**
- Simpler mental model (just functions, no `this`)
- Better tree-shaking
- Natural grouping of related functions
- Common pattern in the effect-ts ecosystem

### 2. Instance State Pattern

opencode uses a clever pattern for singleton state:

```typescript
// From agent/agent.ts
const state = Instance.state(async () => {
  // This function runs once, lazily
  const cfg = await Config.get()
  const result: Record<string, Info> = {
    build: { ... },
    explore: { ... },
  }
  return result
})

export async function get(agent: string) {
  return state().then(x => x[agent])
}
```

`Instance.state()` creates a lazy singleton:
- First call: runs the initialization function
- Subsequent calls: returns cached result
- Can be reset when the project changes

### 3. Event Bus for Decoupling

Components communicate via an event bus:

```typescript
// Publishing an event
Bus.publish(Session.Event.Error, {
  sessionID: "...",
  error: someError
})

// Subscribing to events
Bus.subscribe(Session.Event.Error, (event) => {
  console.log("Error in session:", event.sessionID)
})
```

This decouples the TUI from the core engine - the TUI just listens for events.

---

## The Event Flow

Here's how events flow through the system:

```
User types message
        │
        ▼
┌───────────────────┐
│       TUI         │ ──────▶ UI Events (keypress, etc.)
└───────┬───────────┘
        │ API call
        ▼
┌───────────────────┐
│   Session.prompt  │
└───────┬───────────┘
        │
        ▼
┌───────────────────┐         ┌───────────────────┐
│      Loop         │◀───────▶│   Event Bus       │
│                   │         │                   │
│  ┌─────────────┐  │ publish │  Session.Error    │
│  │ Processor   │──┼────────▶│  Session.Message  │
│  └─────────────┘  │         │  Tool.Execute     │
│        │          │         │  ...              │
│        ▼          │         └────────┬──────────┘
│  ┌─────────────┐  │                  │
│  │    LLM      │  │                  │ subscribe
│  └─────────────┘  │                  ▼
│        │          │         ┌───────────────────┐
│        ▼          │         │       TUI         │
│  ┌─────────────┐  │         │  (updates display)│
│  │   Tools     │  │         └───────────────────┘
│  └─────────────┘  │
└───────────────────┘
```

The TUI subscribes to events and updates the display. The core engine publishes events but doesn't know about the TUI.

---

## Architecture Principles

opencode's architecture follows these principles:

1. **Namespace-based organization** - Functions grouped logically, not OOP
2. **Lazy singletons** - State initialized on first use
3. **Event-driven** - Components decoupled via pub/sub
4. **Streaming-first** - UI updates as data arrives
5. **Layered permissions** - Multiple levels of permission rules merge
6. **Provider abstraction** - Same interface for different LLMs

The main loop is surprisingly simple at its core - the complexity comes from:
- Handling edge cases (errors, retries, cancellation)
- Supporting multiple providers
- Managing context limits
- Providing a good user experience (streaming, permissions)

---

## Related Deep Dives

Now that you understand the high-level architecture, explore specific systems:

- [Main Loop Internals](/deep-dives/main-loop-internals.md) - How the core loop works
- [The Processor](/deep-dives/processor.md) - Stream handling and response processing
- [Tool System Internals](/deep-dives/tool-system-internals.md) - How tools are executed
- [Agent System](/deep-dives/agent-system.md) - Different agent modes
- [Permissions](/deep-dives/permissions.md) - Permission system details
- [Compaction](/deep-dives/compaction.md) - Context management
- [Provider Abstraction](/deep-dives/provider-abstraction.md) - Multi-LLM support
- [Event Bus](/deep-dives/event-bus.md) - Event-driven architecture
