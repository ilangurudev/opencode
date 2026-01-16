# Day 3-4: The Main Loop Deep Dive

> **Goal**: Understand exactly how opencode orchestrates the user→LLM→tool→repeat cycle.

---

## The Loop Is Everything

In Day 1-2, we saw the conceptual loop. Now let's see the real thing.

The main loop in opencode lives in `packages/opencode/src/session/prompt.ts`. It's about 1800 lines, but the core loop logic is much smaller. Everything else handles edge cases.

Let me show you the essence, then we'll build up to the real code.

---

## The Simplest Version

Here's what the loop *conceptually* does:

```python
# Conceptual version (Python)
def main_loop(session_id: str):
    while True:
        # 1. Load current state
        messages = get_messages(session_id)
        last_user = find_last_user(messages)
        last_assistant = find_last_assistant(messages)

        # 2. Should we stop?
        if is_complete(last_assistant):
            break

        # 3. Call the LLM
        response = call_llm(messages, tools)

        # 4. Handle response
        if response.has_tool_calls:
            execute_tools(response.tool_calls)
            # Loop continues - LLM will see results
        else:
            # Done with this turn
            break
```

Simple, right? But production code needs to handle:
- Streaming (show text as it arrives)
- Errors and retries
- User cancellation
- Context overflow
- Permission checks
- Multiple concurrent requests
- Subagents

Let's see how opencode handles each.

---

## The Real Loop: Annotated

Here's the actual loop from `session/prompt.ts`, annotated:

```typescript
// session/prompt.ts:257-634 (simplified and annotated)

export const loop = fn(Identifier.schema("session"), async (sessionID) => {
  // ─────────────────────────────────────────────────────────────────
  // SETUP: Handle concurrent requests
  // ─────────────────────────────────────────────────────────────────
  const abort = start(sessionID)  // Get abort controller

  if (!abort) {
    // Another request is already running for this session
    // Queue up and wait for our turn
    return new Promise<MessageV2.WithParts>((resolve, reject) => {
      const callbacks = state()[sessionID].callbacks
      callbacks.push({ resolve, reject })
    })
  }

  // Cleanup when we exit (using TC39 explicit resource management)
  using _ = defer(() => cancel(sessionID))

  let step = 0
  const session = await Session.get(sessionID)

  // ─────────────────────────────────────────────────────────────────
  // THE MAIN LOOP
  // ─────────────────────────────────────────────────────────────────
  while (true) {
    SessionStatus.set(sessionID, { type: "busy" })
    log.info("loop", { step, sessionID })

    // Check for cancellation
    if (abort.aborted) break

    // ───────────────────────────────────────────────────────────────
    // STEP 1: Load messages and find the latest ones
    // ───────────────────────────────────────────────────────────────
    let msgs = await MessageV2.filterCompacted(
      MessageV2.stream(sessionID)
    )

    let lastUser: MessageV2.User | undefined
    let lastAssistant: MessageV2.Assistant | undefined
    let lastFinished: MessageV2.Assistant | undefined

    // Walk backwards through messages to find what we need
    for (let i = msgs.length - 1; i >= 0; i--) {
      const msg = msgs[i]
      if (!lastUser && msg.info.role === "user")
        lastUser = msg.info as MessageV2.User
      if (!lastAssistant && msg.info.role === "assistant")
        lastAssistant = msg.info as MessageV2.Assistant
      if (!lastFinished && msg.info.role === "assistant" && msg.info.finish)
        lastFinished = msg.info as MessageV2.Assistant
      if (lastUser && lastFinished) break
    }

    if (!lastUser) throw new Error("No user message found")

    // ───────────────────────────────────────────────────────────────
    // STEP 2: Check if we should exit
    // ───────────────────────────────────────────────────────────────
    if (
      lastAssistant?.finish &&                           // LLM finished
      !["tool-calls", "unknown"].includes(lastAssistant.finish) &&  // Not tool call
      lastUser.id < lastAssistant.id                     // Assistant responded to user
    ) {
      log.info("exiting loop", { sessionID })
      break
    }

    step++

    // Generate title on first step (async, fire-and-forget)
    if (step === 1) {
      ensureTitle({ session, history: msgs, ... })
    }

    // ───────────────────────────────────────────────────────────────
    // STEP 3: Handle special cases (subtasks, compaction)
    // ───────────────────────────────────────────────────────────────
    const model = await Provider.getModel(
      lastUser.model.providerID,
      lastUser.model.modelID
    )

    // Check for pending subtasks
    const task = tasks.pop()
    if (task?.type === "subtask") {
      // Execute subtask agent and continue loop
      await executeSubtask(task, ...)
      continue
    }

    // Check for pending compaction
    if (task?.type === "compaction") {
      const result = await SessionCompaction.process({ ... })
      if (result === "stop") break
      continue
    }

    // Check if context is overflowing
    if (needsCompaction(lastFinished, model)) {
      await SessionCompaction.create({ ... })
      continue
    }

    // ───────────────────────────────────────────────────────────────
    // STEP 4: Normal processing - call the LLM
    // ───────────────────────────────────────────────────────────────
    const agent = await Agent.get(lastUser.agent)

    // Create the assistant message (will be filled by processor)
    const processor = SessionProcessor.create({
      assistantMessage: await Session.updateMessage({
        id: Identifier.ascending("message"),
        role: "assistant",
        parentID: lastUser.id,
        sessionID,
        agent: agent.name,
        // ... other fields
      }),
      sessionID,
      model,
      abort,
    })

    // Build the tool set
    const tools = await resolveTools({
      agent,
      session,
      model,
      processor,
    })

    // ───────────────────────────────────────────────────────────────
    // STEP 5: Call the processor (which calls the LLM)
    // ───────────────────────────────────────────────────────────────
    const result = await processor.process({
      user: lastUser,
      agent,
      abort,
      sessionID,
      system: [
        ...await SystemPrompt.environment(),
        ...await SystemPrompt.custom(),
      ],
      messages: MessageV2.toModelMessage(msgs),
      tools,
      model,
    })

    // ───────────────────────────────────────────────────────────────
    // STEP 6: Handle result
    // ───────────────────────────────────────────────────────────────
    if (result === "stop") break
    if (result === "compact") {
      await SessionCompaction.create({ ... })
    }
    // Otherwise, continue the loop
  }

  // Prune old tool outputs to save space
  SessionCompaction.prune({ sessionID })

  // Return the final message
  for await (const item of MessageV2.stream(sessionID)) {
    if (item.info.role === "user") continue
    return item
  }
  throw new Error("Impossible")
})
```

---

## Key Concepts Explained

### 1. The `start()` / `cancel()` Pattern

opencode ensures only one loop runs per session:

```typescript
const state: Record<string, {
  abort: AbortController
  callbacks: { resolve, reject }[]
}> = {}

function start(sessionID: string) {
  if (state[sessionID]) return undefined  // Already running
  state[sessionID] = {
    abort: new AbortController(),
    callbacks: [],
  }
  return state[sessionID].abort.signal
}

function cancel(sessionID: string) {
  const match = state[sessionID]
  if (!match) return
  match.abort.abort()  // Signal abort
  for (const item of match.callbacks) {
    item.reject()  // Reject queued requests
  }
  delete state[sessionID]
}
```

**Python equivalent:**

```python
import asyncio
from dataclasses import dataclass
from typing import Optional

@dataclass
class SessionState:
    abort_event: asyncio.Event
    callbacks: list

sessions: dict[str, SessionState] = {}

def start(session_id: str) -> Optional[asyncio.Event]:
    if session_id in sessions:
        return None  # Already running
    sessions[session_id] = SessionState(
        abort_event=asyncio.Event(),
        callbacks=[]
    )
    return sessions[session_id].abort_event

def cancel(session_id: str):
    if session_id not in sessions:
        return
    state = sessions[session_id]
    state.abort_event.set()  # Signal abort
    del sessions[session_id]
```

### 2. Finding Last Messages

The loop walks backwards through messages:

```typescript
for (let i = msgs.length - 1; i >= 0; i--) {
  const msg = msgs[i]
  if (!lastUser && msg.info.role === "user")
    lastUser = msg.info
  if (!lastAssistant && msg.info.role === "assistant")
    lastAssistant = msg.info
  // ...
}
```

**Why backwards?** We want the *most recent* of each type. Walking forward would find the oldest.

**Python equivalent:**

```python
def find_last_messages(messages: list[Message]) -> tuple[Message, Message]:
    last_user = None
    last_assistant = None

    for msg in reversed(messages):
        if last_user is None and msg.role == "user":
            last_user = msg
        if last_assistant is None and msg.role == "assistant":
            last_assistant = msg
        if last_user and last_assistant:
            break

    return last_user, last_assistant
```

### 3. The Exit Condition

```typescript
if (
  lastAssistant?.finish &&
  !["tool-calls", "unknown"].includes(lastAssistant.finish) &&
  lastUser.id < lastAssistant.id
) {
  break
}
```

Three conditions must be true:
1. `lastAssistant?.finish` - LLM has finished (not streaming)
2. Finish reason is NOT "tool-calls" - LLM isn't asking to use tools
3. `lastUser.id < lastAssistant.id` - Assistant responded after user message

The ID comparison uses [ULID](https://github.com/ulid/spec) which is time-sortable. So `id1 < id2` means `id1` happened before `id2`.

**Python equivalent:**

```python
def should_exit(last_user: Message, last_assistant: Message) -> bool:
    if not last_assistant or not last_assistant.finish:
        return False

    if last_assistant.finish in ("tool-calls", "unknown"):
        return False

    # Check temporal ordering
    return last_user.created_at < last_assistant.created_at
```

---

## The Processor: Handling the Stream

The `SessionProcessor` handles the streaming response. Let's trace through a typical flow:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Stream Processing Flow                            │
│                                                                      │
│   LLM API                     Processor                    Storage   │
│                                                                      │
│   "start"         ─────▶      Set status = busy                     │
│                                                                      │
│   "text-delta"    ─────▶      Append to text       ─────▶  Update   │
│   "hello"                     currentText.text                part   │
│                                                                      │
│   "text-delta"    ─────▶      Append to text       ─────▶  Update   │
│   " world"                    currentText.text                part   │
│                                                                      │
│   "text-end"      ─────▶      Finalize text part                    │
│                                                                      │
│   "tool-call"     ─────▶      Execute tool         ─────▶  Update   │
│   name="read"                 Wait for result               part    │
│   input={path}                                                       │
│                                                                      │
│   "tool-result"   ─────▶      Store result         ─────▶  Update   │
│   output="..."                                              part    │
│                                                                      │
│   "finish-step"   ─────▶      Calculate tokens                      │
│                               Track cost                             │
│                               Check context limit                    │
│                                                                      │
│   "finish"        ─────▶      Return "continue"                     │
│                               or "stop" or "compact"                 │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

Here's the key part of the processor (simplified):

```typescript
// session/processor.ts (simplified)
for await (const value of stream.fullStream) {
  input.abort.throwIfAborted()  // Check for cancellation

  switch (value.type) {
    case "text-delta":
      // Streaming text - update in real-time
      currentText.text += value.text
      await Session.updatePart({
        part: currentText,
        delta: value.text,  // Just the new part
      })
      break

    case "tool-call":
      // LLM wants to call a tool
      const match = toolcalls[value.toolCallId]
      if (match) {
        await Session.updatePart({
          ...match,
          state: { status: "running", input: value.input }
        })

        // DOOM LOOP DETECTION
        // If same tool called 3x with same input, something's wrong
        const parts = await MessageV2.parts(assistantMessage.id)
        const lastThree = parts.slice(-3)
        if (isRepeatedCall(lastThree, value)) {
          await PermissionNext.ask({ permission: "doom_loop" })
        }
      }
      break

    case "tool-result":
      // Tool finished executing
      await Session.updatePart({
        ...match,
        state: {
          status: "completed",
          output: value.output.output,
          title: value.output.title,
        }
      })
      break

    case "tool-error":
      // Tool failed
      await Session.updatePart({
        ...match,
        state: {
          status: "error",
          error: value.error.toString(),
        }
      })
      // If permission was rejected, stop the loop
      if (value.error instanceof PermissionNext.RejectedError) {
        blocked = true
      }
      break

    case "finish-step":
      // LLM finished one "step" (could be text or tool calls)
      const usage = Session.getUsage({ model, usage: value.usage })
      assistantMessage.cost += usage.cost
      assistantMessage.tokens = usage.tokens

      // Check if context is getting too long
      if (await SessionCompaction.isOverflow({ tokens: usage.tokens, model })) {
        needsCompaction = true
      }
      break
  }
}
```

---

## The Three Return Values

The processor returns one of three values:

| Return | Meaning | Loop Action |
|--------|---------|-------------|
| `"continue"` | LLM called tools, keep going | Continue loop |
| `"stop"` | LLM finished or error occurred | Break loop |
| `"compact"` | Context too long, need summary | Trigger compaction, then continue |

```typescript
// At end of processor.process()
if (needsCompaction) return "compact"
if (blocked) return "stop"
if (assistantMessage.error) return "stop"
return "continue"
```

---

## Putting It Together: A Full Example

Let's trace through a real interaction:

**User asks**: "Read the README.md file"

```
Step 1: Loop starts
├─ Load messages: [user: "Read the README.md file"]
├─ No assistant response yet
├─ Call processor.process()
│   └─ LLM streams: "I'll read that file for you."
│   └─ LLM streams: tool_call(read, {path: "README.md"})
│   └─ Tool executes, returns file contents
│   └─ Step finishes, tokens counted
│   └─ Returns "continue" (tool was called)
│
Step 2: Loop continues
├─ Load messages: [user: "...", assistant: text + tool_result]
├─ Last assistant finish = "tool-calls"
├─ Call processor.process()
│   └─ LLM sees tool result
│   └─ LLM streams: "Here's what's in README.md: ..."
│   └─ No tool calls
│   └─ Step finishes
│   └─ Returns "continue" (but finish = "stop")
│
Step 3: Loop checks exit condition
├─ lastAssistant.finish = "stop" (not "tool-calls")
├─ lastUser.id < lastAssistant.id (true)
├─ EXIT LOOP
│
Return final assistant message
```

---

## Error Handling and Retries

The processor includes retry logic:

```typescript
try {
  // ... stream processing
} catch (e) {
  log.error("process", { error: e })
  const error = MessageV2.fromError(e)

  // Check if error is retryable
  const retry = SessionRetry.retryable(error)
  if (retry !== undefined) {
    attempt++
    const delay = SessionRetry.delay(attempt, error)

    // Show user we're retrying
    SessionStatus.set(sessionID, {
      type: "retry",
      attempt,
      message: retry,
      next: Date.now() + delay,
    })

    // Wait and retry
    await SessionRetry.sleep(delay, input.abort)
    continue  // Try again
  }

  // Non-retryable error
  assistantMessage.error = error
  Bus.publish(Session.Event.Error, { sessionID, error })
}
```

**What's retryable?**
- Rate limits (429)
- Server errors (500, 502, 503)
- Network timeouts

**What's NOT retryable?**
- Authentication errors (401)
- Bad requests (400)
- Content policy violations

---

## Assignment: Add Streaming and Tool Dispatch

Now let's enhance your minimal agent with proper streaming and tool dispatch.

**Goal**: Update your agent to:
1. Stream text as it arrives (print character by character)
2. Show tool calls as they happen
3. Track step numbers

**Start here**: [Assignment 2: Streaming and Tool Dispatch](./assignments/02-streaming-agent.md)

---

## Optional Deep Dives

Want to understand the main loop in even more detail?

- 🔄 [Main Loop Internals](/deep-dives/main-loop-internals.md) - Line-by-line walkthrough of the loop
- 📊 [The Processor](/deep-dives/processor.md) - Stream handling and tool execution
- 🚨 [Error Handling](/deep-dives/error-handling.md) - Retry strategies and error recovery
