# Deep Dive: opencode Architecture Internals

> **Time**: ~45-60 minutes
> **Prerequisites**: Complete Day 1-2 core concepts

This deep dive explores the internal architecture of opencode in detail. We'll look at the actual TypeScript code, understand design decisions, and see how all the pieces connect.

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

## The Main Loop in Detail

Let's walk through `session/prompt.ts` section by section.

### Entry Point: `prompt()`

```typescript
// session/prompt.ts:150
export const prompt = fn(PromptInput, async (input) => {
  const session = await Session.get(input.sessionID)
  await SessionRevert.cleanup(session)

  const message = await createUserMessage(input)
  await Session.touch(input.sessionID)

  // Handle permission overrides from input
  const permissions: PermissionNext.Ruleset = []
  for (const [tool, enabled] of Object.entries(input.tools ?? {})) {
    permissions.push({
      permission: tool,
      action: enabled ? "allow" : "deny",
      pattern: "*",
    })
  }
  // ...

  if (input.noReply === true) {
    return message  // Just save the message, don't respond
  }

  return loop(input.sessionID)  // Start the main loop
})
```

`prompt()` is the public API. It:
1. Creates the user message
2. Sets up permissions
3. Calls `loop()` to start processing

### The Loop: `loop()`

```typescript
// session/prompt.ts:257
export const loop = fn(Identifier.schema("session"), async (sessionID) => {
  const abort = start(sessionID)  // Get abort controller
  if (!abort) {
    // Already running - queue up and wait
    return new Promise<MessageV2.WithParts>((resolve, reject) => {
      const callbacks = state()[sessionID].callbacks
      callbacks.push({ resolve, reject })
    })
  }

  using _ = defer(() => cancel(sessionID))  // Cleanup on exit

  let step = 0
  const session = await Session.get(sessionID)

  while (true) {
    // ... the main loop body
  }
})
```

Key points:
- Uses `using` with `defer()` for cleanup (TC39 explicit resource management)
- Tracks step count
- Handles concurrent calls by queueing

### Inside the Loop

Each iteration:

```typescript
while (true) {
  SessionStatus.set(sessionID, { type: "busy" })
  log.info("loop", { step, sessionID })

  if (abort.aborted) break  // Check for cancellation

  // 1. Load messages
  let msgs = await MessageV2.filterCompacted(MessageV2.stream(sessionID))

  // 2. Find last user/assistant messages
  let lastUser: MessageV2.User | undefined
  let lastAssistant: MessageV2.Assistant | undefined
  // ... (iterates backwards through messages)

  // 3. Check if we should exit
  if (
    lastAssistant?.finish &&
    !["tool-calls", "unknown"].includes(lastAssistant.finish) &&
    lastUser.id < lastAssistant.id
  ) {
    break  // LLM finished, no more tool calls
  }

  step++

  // 4. Handle special cases (subtasks, compaction)
  // ...

  // 5. Normal processing - call LLM
  const processor = SessionProcessor.create({ ... })
  const tools = await resolveTools({ ... })

  const result = await processor.process({
    user: lastUser,
    agent,
    abort,
    sessionID,
    system: [...],
    messages: [...],
    tools,
    model,
  })

  // 6. Check result
  if (result === "stop") break
  if (result === "compact") {
    await SessionCompaction.create({ ... })
  }
  continue  // Loop again
}
```

### Tool Resolution

The `resolveTools()` function builds the tool set:

```typescript
async function resolveTools(input: {
  agent: Agent.Info
  model: Provider.Model
  // ...
}) {
  const tools: Record<string, AITool> = {}

  // 1. Get built-in tools from registry
  for (const item of await ToolRegistry.tools(input.model.providerID, input.agent)) {
    tools[item.id] = tool({
      description: item.description,
      inputSchema: jsonSchema(schema),
      async execute(args, options) {
        // Build context
        const ctx = context(args, options)
        // Execute with plugin hooks
        await Plugin.trigger("tool.execute.before", { ... })
        const result = await item.execute(args, ctx)
        await Plugin.trigger("tool.execute.after", { ... })
        return result
      }
    })
  }

  // 2. Add MCP tools
  for (const [key, item] of Object.entries(await MCP.tools())) {
    // Wrap with permission check
    item.execute = async (args, opts) => {
      const ctx = context(args, opts)
      await ctx.ask({ permission: key, ... })  // Permission check!
      const result = await execute(args, opts)
      return result
    }
    tools[key] = item
  }

  return tools
}
```

---

## The Processor: Stream Handling

`SessionProcessor` in `session/processor.ts` handles the streaming response:

```typescript
export namespace SessionProcessor {
  const DOOM_LOOP_THRESHOLD = 3  // Detect repeated tool calls

  export function create(input: {
    assistantMessage: MessageV2.Assistant
    sessionID: string
    model: Provider.Model
    abort: AbortSignal
  }) {
    const toolcalls: Record<string, MessageV2.ToolPart> = {}

    return {
      async process(streamInput: LLM.StreamInput) {
        while (true) {
          try {
            const stream = await LLM.stream(streamInput)

            for await (const value of stream.fullStream) {
              input.abort.throwIfAborted()

              switch (value.type) {
                case "start":
                  SessionStatus.set(input.sessionID, { type: "busy" })
                  break

                case "text-delta":
                  // Streaming text - update UI in real-time
                  if (currentText) {
                    currentText.text += value.text
                    await Session.updatePart({ part: currentText, delta: value.text })
                  }
                  break

                case "tool-call":
                  // Tool call starting
                  const match = toolcalls[value.toolCallId]
                  if (match) {
                    await Session.updatePart({
                      ...match,
                      state: { status: "running", input: value.input }
                    })

                    // Doom loop detection!
                    const parts = await MessageV2.parts(input.assistantMessage.id)
                    const lastThree = parts.slice(-DOOM_LOOP_THRESHOLD)
                    if (/* same tool called 3x with same input */) {
                      await PermissionNext.ask({ permission: "doom_loop", ... })
                    }
                  }
                  break

                case "tool-result":
                  // Tool finished
                  await Session.updatePart({
                    ...match,
                    state: { status: "completed", output: value.output }
                  })
                  break

                case "tool-error":
                  // Tool failed
                  await Session.updatePart({
                    ...match,
                    state: { status: "error", error: value.error }
                  })
                  break

                // ... more event types
              }
            }
          } catch (e) {
            // Error handling with retries
            const retry = SessionRetry.retryable(error)
            if (retry !== undefined) {
              attempt++
              const delay = SessionRetry.delay(attempt, error)
              await SessionRetry.sleep(delay, input.abort)
              continue  // Retry
            }
            // Non-retryable error
            input.assistantMessage.error = error
            break
          }
        }

        // Return loop control signal
        if (needsCompaction) return "compact"
        if (blocked) return "stop"
        return "continue"
      }
    }
  }
}
```

Key features:
- **Streaming**: Updates UI as text arrives
- **Doom loop detection**: Prevents infinite loops of same tool call
- **Retry logic**: Automatic retries with exponential backoff
- **Compaction signals**: Tells main loop when context is too long

---

## The Tool System

### Tool Definition

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

  export function define<P extends z.ZodType>(
    id: string,
    init: Info<P>["init"] | Awaited<ReturnType<Info<P>["init"]>>
  ): Info<P> {
    return {
      id,
      init: async (initCtx) => {
        const toolInfo = init instanceof Function ? await init(initCtx) : init

        // Wrap execute with validation and truncation
        const execute = toolInfo.execute
        toolInfo.execute = async (args, ctx) => {
          // Validate input
          try {
            toolInfo.parameters.parse(args)
          } catch (error) {
            throw new Error(`Invalid arguments: ${error}`)
          }

          // Execute
          const result = await execute(args, ctx)

          // Truncate output if needed
          const truncated = await Truncate.output(result.output)
          return {
            ...result,
            output: truncated.content,
            metadata: { ...result.metadata, truncated: truncated.truncated }
          }
        }
        return toolInfo
      }
    }
  }
}
```

### Tool Context

Every tool execution gets a context:

```typescript
export type Context = {
  sessionID: string
  messageID: string
  agent: string
  abort: AbortSignal
  callID?: string
  extra?: { [key: string]: any }

  // Update UI with title/metadata
  metadata(input: { title?: string; metadata?: any }): void

  // Request permission
  ask(input: {
    permission: string
    patterns: string[]
    metadata: any
  }): Promise<void>
}
```

### Example Tool Implementation

```typescript
// tool/read.ts
export const ReadTool = Tool.define("read", {
  description: DESCRIPTION,  // Loaded from .txt file
  parameters: z.object({
    filePath: z.string().describe("The path to the file to read"),
    offset: z.coerce.number().optional(),
    limit: z.coerce.number().optional(),
  }),

  async execute(params, ctx) {
    let filepath = params.filePath
    if (!path.isAbsolute(filepath)) {
      filepath = path.join(process.cwd(), filepath)
    }

    // Check external directory permission
    await assertExternalDirectory(ctx, filepath, {
      bypass: Boolean(ctx.extra?.["bypassCwdCheck"]),
    })

    // Ask permission for sensitive files
    await ctx.ask({
      permission: "read",
      patterns: [filepath],
      always: ["*"],
      metadata: {},
    })

    // Read the file
    const file = Bun.file(filepath)
    if (!(await file.exists())) {
      throw new Error(`File not found: ${filepath}`)
    }

    // Handle images/PDFs specially
    if (file.type.startsWith("image/")) {
      return {
        title,
        output: "Image read successfully",
        metadata: { truncated: false },
        attachments: [{
          type: "file",
          mime: file.type,
          url: `data:${file.type};base64,${...}`,
        }]
      }
    }

    // Read text file
    const lines = await file.text().then(text => text.split("\n"))
    const content = lines.slice(offset, offset + limit)
      .map((line, i) => `${(i + offset + 1).toString().padStart(5)}| ${line}`)
      .join("\n")

    return {
      title: path.relative(Instance.worktree, filepath),
      output: `<file>\n${content}\n</file>`,
      metadata: { truncated: lines.length > offset + limit }
    }
  }
})
```

---

## The Agent System

Agents define different "modes" with different capabilities:

```typescript
// agent/agent.ts
export namespace Agent {
  export const Info = z.object({
    name: z.string(),
    description: z.string().optional(),
    mode: z.enum(["subagent", "primary", "all"]),
    permission: PermissionNext.Ruleset,
    model: z.object({
      modelID: z.string(),
      providerID: z.string(),
    }).optional(),
    prompt: z.string().optional(),
    options: z.record(z.string(), z.any()),
    steps: z.number().int().positive().optional(),
  })

  const state = Instance.state(async () => {
    const cfg = await Config.get()

    // Default permissions
    const defaults = PermissionNext.fromConfig({
      "*": "allow",
      doom_loop: "ask",
      read: {
        "*": "allow",
        "*.env": "ask",  // Ask before reading .env files
      },
    })

    const result: Record<string, Info> = {
      build: {
        name: "build",
        mode: "primary",
        permission: PermissionNext.merge(defaults, { question: "allow" }),
      },
      explore: {
        name: "explore",
        mode: "subagent",
        description: "Fast read-only exploration",
        prompt: PROMPT_EXPLORE,
        permission: PermissionNext.merge(defaults, {
          "*": "deny",  // Deny all by default
          grep: "allow",
          glob: "allow",
          read: "allow",
          // No write/edit/bash
        }),
      },
      // ... more agents
    }

    // Merge user config
    for (const [key, value] of Object.entries(cfg.agent ?? {})) {
      // ...
    }

    return result
  })
}
```

---

## The Permission System

The permission system uses rules with patterns:

```typescript
// permission/next.ts
export namespace PermissionNext {
  export type Rule = {
    permission: string  // Tool name or category
    pattern: string     // Glob pattern
    action: "allow" | "deny" | "ask"
  }

  export type Ruleset = Rule[]

  export function evaluate(
    permission: string,
    value: string,
    ruleset: Ruleset
  ): { action: "allow" | "deny" | "ask"; rule?: Rule } {
    // Find most specific matching rule
    for (const rule of ruleset) {
      if (rule.permission !== permission && rule.permission !== "*") continue
      if (minimatch(value, rule.pattern)) {
        return { action: rule.action, rule }
      }
    }
    return { action: "ask" }  // Default to asking
  }

  export async function ask(input: {
    permission: string
    patterns: string[]
    sessionID: string
    metadata: any
    ruleset: Ruleset
  }) {
    for (const pattern of input.patterns) {
      const result = evaluate(input.permission, pattern, input.ruleset)

      if (result.action === "deny") {
        throw new RejectedError({ permission: input.permission, pattern })
      }

      if (result.action === "ask") {
        // Show UI prompt to user
        const response = await showPermissionPrompt({ ... })
        if (response === "deny") {
          throw new RejectedError({ ... })
        }
        // "allow" or "allow_always" continues
      }
      // "allow" just continues
    }
  }
}
```

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

## Summary

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

## Next Steps

Now that you understand the architecture, dive into specific systems:
- [Deep Dive: The Processor](./processor.md)
- [Deep Dive: Permissions](./permissions.md)
- [Deep Dive: Compaction](./compaction.md)
