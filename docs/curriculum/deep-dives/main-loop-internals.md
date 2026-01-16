# Deep Dive: The Main Loop Internals

> **Time**: ~30-40 minutes
> **Prerequisites**: Complete Day 3-4 core concepts, read [Architecture Overview](/deep-dives/overview.md)

This deep dive walks through the main loop in `session/prompt.ts` line by line, showing how opencode orchestrates the user→LLM→tool→repeat cycle.

---

## Entry Point: `prompt()`

The public API is the `prompt()` function:

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

`prompt()`:
1. Creates the user message
2. Sets up permissions
3. Calls `loop()` to start processing

---

## The Loop: `loop()`

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

---

## Inside the Loop: Step by Step

Each iteration follows this pattern:

```typescript
while (true) {
  SessionStatus.set(sessionID, { type: "busy" })
  log.info("loop", { step, sessionID })

  if (abort.aborted) break  // Check for cancellation

  // ─────────────────────────────────────────────────────────
  // STEP 1: Load messages
  // ─────────────────────────────────────────────────────────
  let msgs = await MessageV2.filterCompacted(MessageV2.stream(sessionID))

  // ─────────────────────────────────────────────────────────
  // STEP 2: Find last user/assistant messages
  // ─────────────────────────────────────────────────────────
  let lastUser: MessageV2.User | undefined
  let lastAssistant: MessageV2.Assistant | undefined

  // Iterate backwards to find last messages of each type
  for (let i = msgs.length - 1; i >= 0; i--) {
    const msg = msgs[i]
    if (msg.role === "user" && !lastUser) lastUser = msg
    if (msg.role === "assistant" && !lastAssistant) lastAssistant = msg
    if (lastUser && lastAssistant) break
  }

  // ─────────────────────────────────────────────────────────
  // STEP 3: Check if we should exit
  // ─────────────────────────────────────────────────────────
  if (
    lastAssistant?.finish &&
    !["tool-calls", "unknown"].includes(lastAssistant.finish) &&
    lastUser.id < lastAssistant.id
  ) {
    break  // LLM finished, no more tool calls
  }

  step++

  // ─────────────────────────────────────────────────────────
  // STEP 4: Handle special cases
  // ─────────────────────────────────────────────────────────
  // Subtask handling (agents launching other agents)
  if (lastAssistant?.subtask) {
    await handleSubtask(lastAssistant)
    continue
  }

  // Compaction check
  if (lastAssistant?.tokens && await SessionCompaction.isOverflow({
    tokens: lastAssistant.tokens,
    model,
  })) {
    await SessionCompaction.create({
      sessionID,
      messages: msgs,
      abort,
    })
    continue  // Retry loop after compaction
  }

  // ─────────────────────────────────────────────────────────
  // STEP 5: Normal processing - call LLM
  // ─────────────────────────────────────────────────────────
  const processor = SessionProcessor.create({
    assistantMessage: newAssistantMessage,
    sessionID,
    model,
    abort,
  })

  const tools = await resolveTools({
    agent,
    model,
    sessionID,
    // ...
  })

  const result = await processor.process({
    user: lastUser,
    agent,
    abort,
    sessionID,
    system: [...systemPrompts],
    messages: [...conversationHistory],
    tools,
    model,
  })

  // ─────────────────────────────────────────────────────────
  // STEP 6: Check result and decide what to do
  // ─────────────────────────────────────────────────────────
  if (result === "stop") break
  if (result === "compact") {
    await SessionCompaction.create({ ... })
  }
  continue  // Loop again
}
```

---

## Tool Resolution

The `resolveTools()` function builds the tool set for the LLM:

```typescript
async function resolveTools(input: {
  agent: Agent.Info
  model: Provider.Model
  sessionID: string
  // ...
}) {
  const tools: Record<string, AITool> = {}

  // ─────────────────────────────────────────────────────────
  // 1. Get built-in tools from registry
  // ─────────────────────────────────────────────────────────
  for (const item of await ToolRegistry.tools(input.model.providerID, input.agent)) {
    tools[item.id] = tool({
      description: item.description,
      inputSchema: jsonSchema(schema),
      async execute(args, options) {
        // Build context
        const ctx = context(args, options)

        // Execute with plugin hooks
        await Plugin.trigger("tool.execute.before", {
          tool: item.id,
          args,
          ctx,
        })

        const result = await item.execute(args, ctx)

        await Plugin.trigger("tool.execute.after", {
          tool: item.id,
          args,
          ctx,
          result,
        })

        return result
      }
    })
  }

  // ─────────────────────────────────────────────────────────
  // 2. Add MCP tools (external tool servers)
  // ─────────────────────────────────────────────────────────
  for (const [key, item] of Object.entries(await MCP.tools())) {
    // Wrap with permission check
    const originalExecute = item.execute
    item.execute = async (args, opts) => {
      const ctx = context(args, opts)

      // Ask for permission before executing MCP tool
      await ctx.ask({
        permission: key,
        patterns: ["*"],
        metadata: { tool: key },
      })

      const result = await originalExecute(args, opts)
      return result
    }
    tools[key] = item
  }

  // ─────────────────────────────────────────────────────────
  // 3. Filter tools based on agent permissions
  // ─────────────────────────────────────────────────────────
  const filtered: Record<string, AITool> = {}
  for (const [key, tool] of Object.entries(tools)) {
    const permission = PermissionNext.evaluate(
      key,
      "*",
      input.agent.permission
    )

    // Only include tools that aren't denied
    if (permission.action !== "deny") {
      filtered[key] = tool
    }
  }

  return filtered
}
```

---

## Concurrent Request Handling

opencode handles concurrent requests elegantly:

```typescript
// State management for concurrent requests
const state = (): Record<string, State> => {
  return loopState
}

function start(sessionID: string): AbortController | undefined {
  const existing = state()[sessionID]

  if (existing?.abort) {
    // Already running - return undefined
    return undefined
  }

  // Create new abort controller
  const abort = new AbortController()
  state()[sessionID] = {
    abort,
    callbacks: [],
  }

  return abort
}

function cancel(sessionID: string) {
  const existing = state()[sessionID]
  if (!existing) return

  // Resolve all queued callbacks
  const message = existing.lastMessage
  for (const cb of existing.callbacks) {
    cb.resolve(message)
  }

  // Clean up
  delete loopState[sessionID]
}
```

When multiple requests come in for the same session:
1. First request gets an abort controller and runs
2. Subsequent requests queue up and wait
3. When the first request finishes, all queued requests resolve with the same message

---

## Exit Conditions

The loop exits when:

1. **LLM signals completion**
   ```typescript
   if (lastAssistant?.finish &&
       !["tool-calls", "unknown"].includes(lastAssistant.finish)) {
     break
   }
   ```

2. **User cancels**
   ```typescript
   if (abort.aborted) break
   ```

3. **Processor signals stop**
   ```typescript
   if (result === "stop") break
   ```

4. **Error occurs** (caught by processor, which handles retries)

---

## Python Equivalent (Simplified)

Here's what this looks like in Python:

```python
async def loop(session_id: str) -> Message:
    # Check for concurrent requests
    abort = start_session(session_id)
    if not abort:
        # Queue up and wait
        return await wait_for_completion(session_id)

    try:
        step = 0

        while True:
            # Check cancellation
            if abort.is_set():
                break

            # Load messages
            messages = await load_messages(session_id)
            last_user = find_last_user(messages)
            last_assistant = find_last_assistant(messages)

            # Check if done
            if is_complete(last_assistant, last_user):
                break

            step += 1

            # Handle special cases
            if last_assistant and last_assistant.subtask:
                await handle_subtask(last_assistant)
                continue

            if should_compact(last_assistant):
                await compact_conversation(session_id, messages)
                continue

            # Normal processing
            tools = await resolve_tools(agent, model)
            processor = create_processor(session_id, model, abort)

            result = await processor.process(
                messages=messages,
                tools=tools,
                system_prompts=system_prompts,
            )

            # Handle result
            if result == "stop":
                break
            elif result == "compact":
                await compact_conversation(session_id, messages)

            # Continue loop
    finally:
        # Clean up
        cancel_session(session_id)

    return last_assistant
```

---

## Summary

The main loop:
1. **Queues concurrent requests** - Only one loop per session at a time
2. **Loads conversation state** - Gets all messages and finds latest
3. **Checks exit conditions** - Detects when conversation is done
4. **Handles special cases** - Subtasks, compaction, etc.
5. **Calls LLM with tools** - The processor handles streaming
6. **Repeats until done** - Tool calls trigger another iteration

The complexity comes from:
- Streaming responses in real-time
- Handling errors with retries
- Managing context limits
- Supporting cancellation
- Coordinating concurrent requests

---

## Related Deep Dives

- [The Processor](/deep-dives/processor.md) - How streaming and tool execution work
- [Error Handling](/deep-dives/error-handling.md) - Retry logic and error recovery
- [Compaction](/deep-dives/compaction.md) - Context window management
- [Tool System Internals](/deep-dives/tool-system-internals.md) - How tools are resolved and executed
