# Deep Dive: The Processor - Stream Handling

> **Time**: ~30-40 minutes
> **Prerequisites**: Complete Day 3-4 core concepts, read [Main Loop Internals](/deep-dives/main-loop-internals.md)

The `SessionProcessor` in `session/processor.ts` is responsible for handling the streaming LLM response, executing tools, and managing the response lifecycle. This is where the magic happens.

---

## Overview

The processor bridges the gap between the main loop and the LLM. It:
1. **Streams responses** - Updates UI in real-time as text arrives
2. **Executes tools** - Runs tool calls as the LLM requests them
3. **Detects doom loops** - Prevents infinite repetition
4. **Handles errors** - Retries with exponential backoff
5. **Signals the loop** - Tells the main loop when to continue/stop/compact

---

## The Processor Interface

```typescript
// session/processor.ts
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
      async process(streamInput: LLM.StreamInput): Promise<LoopSignal> {
        // Returns: "continue" | "stop" | "compact"
      }
    }
  }
}
```

The processor is created per loop iteration and returns a signal telling the loop what to do next.

---

## The Streaming Loop

The processor runs its own internal loop to handle the LLM stream:

```typescript
async process(streamInput: LLM.StreamInput) {
  let attempt = 0
  let needsCompaction = false
  let blocked = false

  // ─────────────────────────────────────────────────────────
  // Retry loop (outer loop)
  // ─────────────────────────────────────────────────────────
  while (true) {
    try {
      // Start streaming from LLM
      const stream = await LLM.stream(streamInput)

      // ───────────────────────────────────────────────────────
      // Stream event loop (inner loop)
      // ───────────────────────────────────────────────────────
      for await (const value of stream.fullStream) {
        input.abort.throwIfAborted()

        switch (value.type) {
          case "start":
            SessionStatus.set(input.sessionID, { type: "busy" })
            break

          case "text-delta":
            await handleTextDelta(value)
            break

          case "tool-call":
            await handleToolCall(value)
            break

          case "tool-result":
            await handleToolResult(value)
            break

          case "tool-error":
            await handleToolError(value)
            break

          case "finish":
            await handleFinish(value)
            break

          case "error":
            throw value.error
        }
      }

      // Success - exit retry loop
      break

    } catch (e) {
      // ─────────────────────────────────────────────────────
      // Error handling with retries
      // ─────────────────────────────────────────────────────
      const retry = SessionRetry.retryable(error)
      if (retry !== undefined && attempt < MAX_RETRIES) {
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

  // Return signal to main loop
  if (needsCompaction) return "compact"
  if (blocked) return "stop"
  return "continue"
}
```

---

## Handling Stream Events

### Text Delta

When text arrives, update the current text part:

```typescript
case "text-delta":
  if (currentText) {
    currentText.text += value.text
    await Session.updatePart({
      part: currentText,
      delta: value.text,  // Send just the delta to UI
    })

    // Publish event for TUI
    Bus.publish(Session.Event.Message, {
      sessionID: input.sessionID,
      messageID: input.assistantMessage.id,
      delta: value.text,
    })
  }
  break
```

The UI gets live updates as text streams in.

### Tool Call

When the LLM wants to call a tool:

```typescript
case "tool-call":
  const match = toolcalls[value.toolCallId]
  if (match) {
    // Update tool call state to "running"
    await Session.updatePart({
      ...match,
      state: {
        status: "running",
        input: value.input,
      }
    })

    // ─────────────────────────────────────────────────────
    // DOOM LOOP DETECTION
    // ─────────────────────────────────────────────────────
    // Check if same tool called repeatedly with same input
    const parts = await MessageV2.parts(input.assistantMessage.id)
    const lastThree = parts.slice(-DOOM_LOOP_THRESHOLD)

    const allSameTool = lastThree.every(p =>
      p.type === "tool" && p.name === match.name
    )

    const allSameInput = lastThree.every(p =>
      p.type === "tool" &&
      JSON.stringify(p.state.input) === JSON.stringify(value.input)
    )

    if (allSameTool && allSameInput) {
      // Doom loop detected! Ask user
      await PermissionNext.ask({
        permission: "doom_loop",
        patterns: [match.name],
        metadata: {
          tool: match.name,
          input: value.input,
        },
        sessionID: input.sessionID,
        ruleset: agent.permission,
      })
    }
  }
  break
```

**Doom loop detection**: If the same tool is called 3+ times with identical input, ask the user for permission to continue.

### Tool Result

When a tool completes successfully:

```typescript
case "tool-result":
  const match = toolcalls[value.toolCallId]
  if (match) {
    await Session.updatePart({
      ...match,
      state: {
        status: "completed",
        output: value.output,
      }
    })

    // Publish event
    Bus.publish(Tool.Event.Result, {
      sessionID: input.sessionID,
      tool: match.name,
      result: value.output,
    })
  }
  break
```

### Tool Error

When a tool fails:

```typescript
case "tool-error":
  const match = toolcalls[value.toolCallId]
  if (match) {
    await Session.updatePart({
      ...match,
      state: {
        status: "error",
        error: value.error,
      }
    })

    // Check if error is a permission rejection
    if (value.error instanceof PermissionNext.RejectedError) {
      blocked = true  // Signal main loop to stop
    }

    // Publish error event
    Bus.publish(Tool.Event.Error, {
      sessionID: input.sessionID,
      tool: match.name,
      error: value.error,
    })
  }
  break
```

### Finish

When the LLM finishes:

```typescript
case "finish":
  // Save finish reason
  input.assistantMessage.finish = value.reason

  // Save token usage
  input.assistantMessage.tokens = {
    input: value.usage.inputTokens,
    output: value.usage.outputTokens,
    cache: {
      read: value.usage.cacheReadTokens || 0,
      write: value.usage.cacheWriteTokens || 0,
    }
  }

  // Check if we're approaching context limit
  const threshold = input.model.limit.context * 0.9  // 90% full
  if (input.assistantMessage.tokens.input > threshold) {
    needsCompaction = true  // Signal compaction needed
  }

  // Update message
  await Session.updateMessage(input.assistantMessage)
  break
```

---

## Error Handling and Retries

The processor handles transient errors with exponential backoff:

```typescript
// session/retry.ts (simplified)
export namespace SessionRetry {
  export function retryable(error: Error): boolean | undefined {
    // Rate limit errors - always retry
    if (error.message.includes("rate_limit")) return true

    // Overloaded errors - always retry
    if (error.message.includes("overloaded")) return true

    // Timeout errors - retry
    if (error.message.includes("timeout")) return true

    // Network errors - retry
    if (error instanceof NetworkError) return true

    // Other errors - don't retry
    return undefined
  }

  export function delay(attempt: number, error: Error): number {
    // Exponential backoff: 2s, 4s, 8s, 16s, ...
    const base = 2000
    const max = 60000  // Cap at 60s

    let delay = base * Math.pow(2, attempt - 1)

    // If error specifies retry-after, use that
    if (error.retryAfter) {
      delay = error.retryAfter * 1000
    }

    return Math.min(delay, max)
  }

  export async function sleep(ms: number, abort: AbortSignal): Promise<void> {
    return new Promise((resolve, reject) => {
      const timeout = setTimeout(resolve, ms)

      // Allow cancellation during sleep
      abort.addEventListener("abort", () => {
        clearTimeout(timeout)
        reject(new AbortError())
      })
    })
  }
}
```

---

## Python Equivalent

Here's what the processor looks like in Python:

```python
from dataclasses import dataclass
from typing import Literal

LoopSignal = Literal["continue", "stop", "compact"]

@dataclass
class ProcessorInput:
    assistant_message: AssistantMessage
    session_id: str
    model: Model
    abort: asyncio.Event


class SessionProcessor:
    DOOM_LOOP_THRESHOLD = 3

    def __init__(self, config: ProcessorInput):
        self.config = config
        self.tool_calls: dict[str, ToolPart] = {}
        self.current_text: TextPart | None = None

    async def process(self, stream_input: StreamInput) -> LoopSignal:
        attempt = 0
        needs_compaction = False
        blocked = False

        while True:
            try:
                # Start streaming from LLM
                stream = await llm_stream(stream_input)

                async for event in stream:
                    # Check cancellation
                    if self.config.abort.is_set():
                        raise asyncio.CancelledError()

                    match event.type:
                        case "text-delta":
                            await self._handle_text_delta(event)

                        case "tool-call":
                            await self._handle_tool_call(event)

                        case "tool-result":
                            await self._handle_tool_result(event)

                        case "tool-error":
                            await self._handle_tool_error(event)
                            if isinstance(event.error, PermissionRejected):
                                blocked = True

                        case "finish":
                            tokens = event.usage.input_tokens
                            threshold = self.config.model.context_limit * 0.9
                            if tokens > threshold:
                                needs_compaction = True

                # Success!
                break

            except Exception as e:
                # Check if retryable
                if is_retryable(e) and attempt < 5:
                    attempt += 1
                    delay = calculate_backoff(attempt, e)
                    await asyncio.sleep(delay)
                    continue

                # Non-retryable error
                self.config.assistant_message.error = e
                break

        # Return signal
        if needs_compaction:
            return "compact"
        if blocked:
            return "stop"
        return "continue"

    async def _handle_text_delta(self, event):
        if self.current_text:
            self.current_text.text += event.text
            await update_message_part(self.current_text, delta=event.text)

    async def _handle_tool_call(self, event):
        tool = self.tool_calls[event.tool_call_id]

        # Update state
        tool.state = ToolState(status="running", input=event.input)
        await update_message_part(tool)

        # Doom loop detection
        recent = await get_recent_tool_calls(
            self.config.assistant_message.id,
            limit=self.DOOM_LOOP_THRESHOLD
        )

        if self._is_doom_loop(recent, tool.name, event.input):
            await ask_permission(
                permission="doom_loop",
                patterns=[tool.name],
                metadata={"tool": tool.name, "input": event.input},
            )

    def _is_doom_loop(self, recent: list[ToolPart], name: str, input: dict) -> bool:
        if len(recent) < self.DOOM_LOOP_THRESHOLD:
            return False

        return all(
            t.name == name and t.state.input == input
            for t in recent
        )
```

---

## Summary

The processor:
- **Streams responses** incrementally to the UI
- **Executes tools** as the LLM requests them
- **Detects doom loops** by tracking repeated tool calls
- **Handles errors** with exponential backoff retries
- **Signals the loop** to continue, stop, or compact

The streaming architecture provides a great user experience - you see output immediately rather than waiting for the entire response.

---

## Related Deep Dives

- [Main Loop Internals](/deep-dives/main-loop-internals.md) - How the loop calls the processor
- [Error Handling](/deep-dives/error-handling.md) - Retry strategies in detail
- [Tool System Internals](/deep-dives/tool-system-internals.md) - How tools execute
- [Event Bus](/deep-dives/event-bus.md) - How UI updates are published
