# Deep Dive: Error Handling and Retries

> **Time**: ~25-30 minutes
> **Prerequisites**: Read [The Processor](/deep-dives/processor.md) and [Main Loop Internals](/deep-dives/main-loop-internals.md)

Errors are inevitable in agent systems - API failures, network issues, rate limits, etc. How you handle them determines reliability. This deep dive explores opencode's error handling strategy.

---

## Types of Errors

Agents encounter many error types:

### 1. Transient Errors (Retry)

Temporary issues that resolve themselves:
- **Rate limits** - Wait and retry
- **Network timeouts** - Retry with backoff
- **API overload** - Wait and retry
- **Temporary failures** - Retry

### 2. Permanent Errors (Don't Retry)

Issues that won't resolve:
- **Invalid API key** - Won't work on retry
- **Model doesn't exist** - Won't work on retry
- **Invalid input** - Won't work on retry
- **Permission denied** - User must fix

### 3. User Errors (Inform)

Issues caused by user actions:
- **File not found** - Tell user
- **Permission rejected** - Stop gracefully
- **Invalid command** - Show error

---

## The Retry Strategy

opencode uses **exponential backoff** with jitter:

```typescript
// session/retry.ts (simplified)
export namespace SessionRetry {
  const MAX_RETRIES = 5

  export function retryable(error: Error): boolean | undefined {
    // ─────────────────────────────────────────────────────────
    // Rate limit errors - always retry
    // ─────────────────────────────────────────────────────────
    if (
      error.message.includes("rate_limit") ||
      error.message.includes("429")
    ) {
      return true
    }

    // ─────────────────────────────────────────────────────────
    // Overloaded errors - always retry
    // ─────────────────────────────────────────────────────────
    if (
      error.message.includes("overloaded") ||
      error.message.includes("529")
    ) {
      return true
    }

    // ─────────────────────────────────────────────────────────
    // Timeout errors - retry
    // ─────────────────────────────────────────────────────────
    if (
      error.message.includes("timeout") ||
      error.message.includes("ETIMEDOUT")
    ) {
      return true
    }

    // ─────────────────────────────────────────────────────────
    // Network errors - retry
    // ─────────────────────────────────────────────────────────
    if (
      error.message.includes("ECONNREFUSED") ||
      error.message.includes("ECONNRESET") ||
      error.message.includes("ENOTFOUND")
    ) {
      return true
    }

    // ─────────────────────────────────────────────────────────
    // Other errors - don't retry
    // ─────────────────────────────────────────────────────────
    return undefined
  }

  export function delay(attempt: number, error: Error): number {
    // ─────────────────────────────────────────────────────────
    // Exponential backoff: 2s, 4s, 8s, 16s, 32s
    // ─────────────────────────────────────────────────────────
    const base = 2000  // 2 seconds
    const max = 60000  // Cap at 60 seconds

    let delay = base * Math.pow(2, attempt - 1)

    // ─────────────────────────────────────────────────────────
    // If error specifies retry-after, use that
    // ─────────────────────────────────────────────────────────
    if (error.retryAfter) {
      delay = error.retryAfter * 1000
    }

    // ─────────────────────────────────────────────────────────
    // Add jitter (±20%) to avoid thundering herd
    // ─────────────────────────────────────────────────────────
    const jitter = delay * 0.2 * (Math.random() - 0.5)
    delay += jitter

    return Math.min(delay, max)
  }

  export async function sleep(
    ms: number,
    abort: AbortSignal
  ): Promise<void> {
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

## Retry in the Processor

The processor implements the retry loop:

```typescript
// session/processor.ts (simplified)
async process(streamInput: LLM.StreamInput) {
  let attempt = 0

  // ─────────────────────────────────────────────────────────
  // Retry loop
  // ─────────────────────────────────────────────────────────
  while (true) {
    try {
      // Try to stream from LLM
      const stream = await LLM.stream(streamInput)

      for await (const value of stream.fullStream) {
        // Process events...
      }

      // Success! Exit retry loop
      break

    } catch (error) {
      // ───────────────────────────────────────────────────────
      // Check if error is retryable
      // ───────────────────────────────────────────────────────
      const shouldRetry = SessionRetry.retryable(error)

      if (shouldRetry && attempt < SessionRetry.MAX_RETRIES) {
        attempt++

        // Calculate delay with exponential backoff
        const delay = SessionRetry.delay(attempt, error)

        log.warn("processor.retry", {
          attempt,
          delay,
          error: error.message,
        })

        // Publish event for UI
        Bus.publish(Session.Event.Retry, {
          sessionID: input.sessionID,
          attempt,
          delay,
        })

        // Wait before retrying
        await SessionRetry.sleep(delay, input.abort)

        continue  // Retry
      }

      // ───────────────────────────────────────────────────────
      // Non-retryable error or max retries exceeded
      // ───────────────────────────────────────────────────────
      log.error("processor.error", {
        error: error.message,
        attempts: attempt,
      })

      // Save error to message
      input.assistantMessage.error = {
        message: error.message,
        stack: error.stack,
        attempts: attempt,
      }

      await Session.updateMessage(input.assistantMessage)

      // Publish error event
      Bus.publish(Session.Event.Error, {
        sessionID: input.sessionID,
        error,
      })

      break  // Exit retry loop
    }
  }

  // Return signal to main loop
  if (input.assistantMessage.error) {
    return "stop"  // Error - stop loop
  }
  return "continue"
}
```

---

## Error Types in opencode

opencode defines custom error classes:

```typescript
// errors.ts
export class RejectedError extends Error {
  constructor(public details: {
    permission: string
    pattern: string
  }) {
    super(`Permission denied: ${details.permission} ${details.pattern}`)
    this.name = "RejectedError"
  }
}

export class AbortError extends Error {
  constructor() {
    super("Operation was aborted")
    this.name = "AbortError"
  }
}

export class NetworkError extends Error {
  constructor(
    message: string,
    public statusCode?: number,
    public retryAfter?: number
  ) {
    super(message)
    this.name = "NetworkError"
  }
}

export class RateLimitError extends NetworkError {
  constructor(retryAfter: number) {
    super("Rate limit exceeded", 429, retryAfter)
    this.name = "RateLimitError"
  }
}
```

---

## Tool Error Handling

Tools can throw errors that are handled gracefully:

```typescript
// In a tool implementation
async execute(params, ctx) {
  // ─────────────────────────────────────────────────────────
  // Validate input
  // ─────────────────────────────────────────────────────────
  if (!params.filePath) {
    throw new Error("filePath is required")  // User error
  }

  // ─────────────────────────────────────────────────────────
  // Check permissions
  // ─────────────────────────────────────────────────────────
  await ctx.ask({
    permission: "read",
    patterns: [params.filePath],
    metadata: {},
  })  // May throw RejectedError

  // ─────────────────────────────────────────────────────────
  // Execute operation
  // ─────────────────────────────────────────────────────────
  const file = Bun.file(params.filePath)
  if (!(await file.exists())) {
    throw new Error(`File not found: ${params.filePath}`)  // User error
  }

  // Read file...
}
```

When a tool throws, the processor catches it and converts to a tool-error event:

```typescript
case "tool-error":
  const match = toolcalls[value.toolCallId]
  if (match) {
    // Update tool state to "error"
    await Session.updatePart({
      ...match,
      state: {
        status: "error",
        error: value.error.message,
      }
    })

    // Check if permission was rejected
    if (value.error instanceof RejectedError) {
      blocked = true  // Stop the loop
    }
  }
  break
```

The LLM sees the error and can respond appropriately.

---

## User-Facing Error Messages

Errors are formatted for users:

```typescript
export function formatError(error: Error): string {
  if (error instanceof RejectedError) {
    return `Permission denied: ${error.details.permission}`
  }

  if (error instanceof RateLimitError) {
    const minutes = Math.ceil(error.retryAfter / 60)
    return `Rate limit exceeded. Please wait ${minutes} minute(s).`
  }

  if (error instanceof NetworkError) {
    return `Network error: ${error.message}. Please check your connection.`
  }

  if (error instanceof AbortError) {
    return "Operation was cancelled."
  }

  // Generic error
  return `Error: ${error.message}`
}
```

---

## Python Equivalent

```python
import asyncio
import random
from dataclasses import dataclass
from typing import Optional


@dataclass
class RetryConfig:
    max_retries: int = 5
    base_delay: float = 2.0  # seconds
    max_delay: float = 60.0  # seconds
    jitter: float = 0.2  # ±20%


class RetryableError(Exception):
    """Error that should be retried."""
    def __init__(self, message: str, retry_after: Optional[float] = None):
        super().__init__(message)
        self.retry_after = retry_after


class RateLimitError(RetryableError):
    """Rate limit exceeded."""
    pass


class NetworkError(RetryableError):
    """Network-related error."""
    pass


def is_retryable(error: Exception) -> bool:
    """Check if error should be retried."""
    if isinstance(error, RetryableError):
        return True

    message = str(error).lower()

    # Rate limits
    if "rate_limit" in message or "429" in message:
        return True

    # Overloaded
    if "overloaded" in message or "529" in message:
        return True

    # Timeouts
    if "timeout" in message:
        return True

    # Network errors
    if any(x in message for x in ["econnrefused", "econnreset", "enotfound"]):
        return True

    return False


def calculate_delay(attempt: int, error: Exception, config: RetryConfig) -> float:
    """Calculate retry delay with exponential backoff and jitter."""

    # Check if error specifies retry-after
    if hasattr(error, "retry_after") and error.retry_after:
        delay = error.retry_after
    else:
        # Exponential backoff: 2s, 4s, 8s, 16s, 32s
        delay = config.base_delay * (2 ** (attempt - 1))

    # Add jitter to avoid thundering herd
    jitter_amount = delay * config.jitter * (random.random() - 0.5)
    delay += jitter_amount

    # Cap at max delay
    return min(delay, config.max_delay)


async def retry_with_backoff(
    func,
    config: RetryConfig = RetryConfig()
):
    """Execute function with retry logic."""
    attempt = 0

    while True:
        try:
            # Try to execute
            return await func()

        except Exception as error:
            # Check if retryable
            if not is_retryable(error):
                raise  # Re-raise non-retryable errors

            if attempt >= config.max_retries:
                raise  # Max retries exceeded

            attempt += 1

            # Calculate delay
            delay = calculate_delay(attempt, error, config)

            print(f"Retry attempt {attempt} after {delay:.1f}s: {error}")

            # Wait before retrying
            await asyncio.sleep(delay)

            # Loop continues - retry


# Usage:
async def call_llm():
    # This might fail with rate limits or network errors
    response = await llm_api.chat(messages=messages)
    return response


# Wrap with retry logic
response = await retry_with_backoff(call_llm)
```

---

## Best Practices

### 1. Identify Error Types

Clearly distinguish:
- **Transient** (retry) - Rate limits, timeouts
- **Permanent** (fail fast) - Invalid config, auth errors
- **User** (inform) - File not found, permission denied

### 2. Use Exponential Backoff

Don't retry immediately - give the system time to recover:
```
Attempt 1: Wait 2s
Attempt 2: Wait 4s
Attempt 3: Wait 8s
Attempt 4: Wait 16s
Attempt 5: Wait 32s
```

### 3. Add Jitter

Avoid thundering herd - randomize delays slightly:
```python
delay = base_delay * (2 ** attempt)
delay += delay * 0.2 * (random.random() - 0.5)  # ±20%
```

### 4. Respect Retry-After

If the API says "retry after 60s", wait 60s - not 2s!

### 5. Limit Retries

Don't retry forever - cap at 5 attempts or 60s total delay.

### 6. Log Retries

Help debugging by logging:
- What error occurred
- How many attempts
- Delay before next retry

### 7. Handle Cancellation

Allow users to cancel during retries:
```python
await asyncio.wait_for(asyncio.sleep(delay), timeout=None)
```

---

## Summary

Error handling in opencode:
- **Identifies retryable errors** (rate limits, timeouts, network)
- **Uses exponential backoff** (2s, 4s, 8s, 16s, 32s)
- **Adds jitter** to avoid thundering herd
- **Respects retry-after** headers from APIs
- **Limits retries** to 5 attempts
- **Handles cancellation** during waits
- **Publishes events** for UI feedback
- **Fails gracefully** for non-retryable errors

Good error handling makes agents reliable and user-friendly.

---

## Related Deep Dives

- [The Processor](/deep-dives/processor.md) - Where retries happen
- [Provider Abstraction](/deep-dives/provider-abstraction.md) - Provider-specific errors
- [Event Bus](/deep-dives/event-bus.md) - Error event publishing
