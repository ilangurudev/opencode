# Deep Dive: Event Bus Architecture

> **Time**: ~20-25 minutes
> **Prerequisites**: Read [Architecture Overview](/deep-dives/overview.md)

The event bus decouples opencode's core engine from the UI. The engine publishes events; the UI subscribes and updates. This deep dive explores how it works and why it's important.

---

## The Problem: Tight Coupling

Without an event bus, components would be tightly coupled:

```typescript
// BAD: Engine knows about UI
async function executeToolCall(tool, args) {
  ui.showToolExecuting(tool.name)  // ❌ Engine depends on UI

  const result = await tool.execute(args)

  ui.showToolResult(result)  // ❌ Tight coupling
  return result
}
```

Problems:
- Can't test engine without UI
- Can't swap UIs (TUI vs Web vs API)
- Changes to UI require changes to engine

---

## The Solution: Pub/Sub Pattern

With an event bus, components are decoupled:

```typescript
// GOOD: Engine publishes events
async function executeToolCall(tool, args) {
  Bus.publish(Tool.Event.Start, {
    tool: tool.name,
    args
  })

  const result = await tool.execute(args)

  Bus.publish(Tool.Event.Result, {
    tool: tool.name,
    result
  })

  return result
}

// UI subscribes to events
Bus.subscribe(Tool.Event.Start, (event) => {
  ui.showToolExecuting(event.tool)
})

Bus.subscribe(Tool.Event.Result, (event) => {
  ui.showToolResult(event.result)
})
```

Benefits:
- ✅ Engine doesn't know about UI
- ✅ Multiple UIs can coexist
- ✅ Easy to test
- ✅ Easy to add logging, metrics, etc.

---

## The Event Bus Implementation

opencode uses a simple pub/sub implementation:

```typescript
// bus/index.ts (simplified)
export namespace Bus {
  type Listener<T> = (event: T) => void | Promise<void>

  // Event listeners by event type
  const listeners = new Map<string, Set<Listener<any>>>()

  export function subscribe<T>(
    eventType: string,
    listener: Listener<T>
  ): () => void {
    // Get or create listener set
    if (!listeners.has(eventType)) {
      listeners.set(eventType, new Set())
    }

    const set = listeners.get(eventType)!
    set.add(listener)

    // Return unsubscribe function
    return () => {
      set.delete(listener)
      if (set.size === 0) {
        listeners.delete(eventType)
      }
    }
  }

  export async function publish<T>(
    eventType: string,
    event: T
  ): Promise<void> {
    const set = listeners.get(eventType)
    if (!set) return

    // Call all listeners in parallel
    await Promise.all(
      Array.from(set).map(listener => listener(event))
    )
  }
}
```

---

## Event Types

opencode defines events for different domains:

### Session Events

```typescript
export namespace Session {
  export enum Event {
    Created = "session.created",
    Updated = "session.updated",
    Message = "session.message",
    Error = "session.error",
    Status = "session.status",
  }

  // Event data types
  export type CreatedEvent = {
    sessionID: string
    timestamp: number
  }

  export type MessageEvent = {
    sessionID: string
    messageID: string
    delta?: string  // Text delta for streaming
  }

  export type ErrorEvent = {
    sessionID: string
    error: Error
  }

  export type StatusEvent = {
    sessionID: string
    status: "idle" | "busy" | "error"
  }
}
```

### Tool Events

```typescript
export namespace Tool {
  export enum Event {
    Execute = "tool.execute",
    Result = "tool.result",
    Error = "tool.error",
  }

  export type ExecuteEvent = {
    sessionID: string
    tool: string
    args: any
  }

  export type ResultEvent = {
    sessionID: string
    tool: string
    result: any
  }

  export type ErrorEvent = {
    sessionID: string
    tool: string
    error: Error
  }
}
```

### Permission Events

```typescript
export namespace Permission {
  export enum Event {
    Request = "permission.request",
    Granted = "permission.granted",
    Denied = "permission.denied",
  }

  export type RequestEvent = {
    sessionID: string
    permission: string
    pattern: string
  }
}
```

---

## Event Flow

Here's how events flow through the system:

```
┌─────────────────────────────────────────────────────────────┐
│                      Event Flow                              │
│                                                              │
│  Core Engine                  Event Bus              UI      │
│  ───────────                  ─────────              ──      │
│                                                              │
│  1. User message                                             │
│     │                                                        │
│     ├─▶ Session.Created ────▶ Bus ────▶ UI shows "thinking"│
│     │                                                        │
│  2. LLM response                                             │
│     │                                                        │
│     ├─▶ Session.Message ────▶ Bus ────▶ UI shows text      │
│     │   (text delta)                                        │
│     │                                                        │
│  3. Tool call                                                │
│     │                                                        │
│     ├─▶ Tool.Execute ───────▶ Bus ────▶ UI shows "running" │
│     │                                                        │
│  4. Tool result                                              │
│     │                                                        │
│     ├─▶ Tool.Result ────────▶ Bus ────▶ UI shows result    │
│     │                                                        │
│  5. Error                                                    │
│     │                                                        │
│     └─▶ Session.Error ──────▶ Bus ────▶ UI shows error     │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Usage Examples

### Publishing Events

```typescript
// In the session processor
case "text-delta":
  if (currentText) {
    currentText.text += value.text

    // Publish event for UI
    Bus.publish(Session.Event.Message, {
      sessionID: input.sessionID,
      messageID: input.assistantMessage.id,
      delta: value.text,  // Just the new text
    })
  }
  break
```

### Subscribing to Events

```typescript
// In the TUI
export function setupTUI() {
  // Subscribe to session messages
  Bus.subscribe(Session.Event.Message, async (event) => {
    const { sessionID, messageID, delta } = event

    if (delta) {
      // Streaming text - append to display
      appendText(delta)
    } else {
      // Full message - refresh display
      refreshMessage(messageID)
    }
  })

  // Subscribe to tool events
  Bus.subscribe(Tool.Event.Execute, (event) => {
    showToolStatus(event.tool, "running")
  })

  Bus.subscribe(Tool.Event.Result, (event) => {
    showToolStatus(event.tool, "completed")
  })

  // Subscribe to errors
  Bus.subscribe(Session.Event.Error, (event) => {
    showError(event.error)
  })
}
```

---

## Multiple Subscribers

Multiple components can subscribe to the same events:

```typescript
// TUI updates display
Bus.subscribe(Session.Event.Message, (event) => {
  tui.updateMessage(event)
})

// Logger writes to file
Bus.subscribe(Session.Event.Message, (event) => {
  logger.log(`Message: ${event.messageID}`)
})

// Metrics tracks usage
Bus.subscribe(Session.Event.Message, (event) => {
  metrics.increment("messages.sent")
})

// Analytics sends to server
Bus.subscribe(Session.Event.Message, async (event) => {
  await analytics.track("message", event)
})
```

All subscribers receive the event independently.

---

## Unsubscribing

The `subscribe()` function returns an unsubscribe function:

```typescript
// Subscribe
const unsubscribe = Bus.subscribe(Session.Event.Message, (event) => {
  console.log("Message:", event)
})

// Later, unsubscribe
unsubscribe()
```

This prevents memory leaks when components are destroyed.

---

## Error Handling

Errors in listeners don't affect other listeners:

```typescript
export async function publish<T>(
  eventType: string,
  event: T
): Promise<void> {
  const set = listeners.get(eventType)
  if (!set) return

  // Call all listeners
  const promises = Array.from(set).map(async listener => {
    try {
      await listener(event)
    } catch (error) {
      // Log error but don't throw
      console.error(`Error in listener for ${eventType}:`, error)
    }
  })

  await Promise.all(promises)
}
```

---

## Python Equivalent

```python
from typing import Callable, Any
from dataclasses import dataclass
import asyncio


EventListener = Callable[[Any], None | asyncio.Task]


class EventBus:
    def __init__(self):
        self._listeners: dict[str, set[EventListener]] = {}

    def subscribe(
        self,
        event_type: str,
        listener: EventListener
    ) -> Callable[[], None]:
        """Subscribe to an event. Returns unsubscribe function."""
        if event_type not in self._listeners:
            self._listeners[event_type] = set()

        self._listeners[event_type].add(listener)

        # Return unsubscribe function
        def unsubscribe():
            self._listeners[event_type].discard(listener)
            if not self._listeners[event_type]:
                del self._listeners[event_type]

        return unsubscribe

    async def publish(self, event_type: str, event: Any) -> None:
        """Publish an event to all subscribers."""
        if event_type not in self._listeners:
            return

        # Call all listeners in parallel
        tasks = []
        for listener in self._listeners[event_type]:
            try:
                result = listener(event)
                if asyncio.iscoroutine(result):
                    tasks.append(result)
            except Exception as e:
                print(f"Error in listener for {event_type}: {e}")

        if tasks:
            await asyncio.gather(*tasks, return_exceptions=True)


# Global bus instance
bus = EventBus()


# Usage:
@dataclass
class MessageEvent:
    session_id: str
    message_id: str
    delta: str = None


# Subscribe
def handle_message(event: MessageEvent):
    print(f"Received message: {event.delta}")

unsubscribe = bus.subscribe("session.message", handle_message)

# Publish
await bus.publish("session.message", MessageEvent(
    session_id="123",
    message_id="456",
    delta="Hello"
))

# Unsubscribe
unsubscribe()
```

---

## Benefits of Event Bus

1. **Decoupling** - Components don't depend on each other
2. **Extensibility** - Easy to add new subscribers
3. **Testing** - Can test engine without UI
4. **Multiple UIs** - TUI, Web, API can all subscribe
5. **Cross-cutting concerns** - Logging, metrics, analytics

---

## Trade-offs

**Benefits**:
- ✅ Loose coupling
- ✅ Easy to extend
- ✅ Multiple subscribers

**Costs**:
- ❌ Indirect flow (harder to trace)
- ❌ No compile-time checks (event type is string)
- ❌ Debugging can be harder

---

## Summary

The event bus:
- **Decouples engine from UI** using pub/sub pattern
- **Enables multiple subscribers** for the same events
- **Supports async listeners** for I/O operations
- **Handles errors gracefully** without affecting other listeners
- **Makes the system extensible** - easy to add logging, metrics, etc.

This is a crucial architectural pattern that makes opencode flexible and testable.

---

## Related Deep Dives

- [Architecture Overview](/deep-dives/overview.md) - How the event bus fits in
- [The Processor](/deep-dives/processor.md) - Where many events are published
- [Main Loop Internals](/deep-dives/main-loop-internals.md) - Session events
