# Deep Dive: Context Window Management (Compaction)

> **Time**: ~25-35 minutes
> **Prerequisites**: Complete Day 9-10 core concepts, read [Main Loop Internals](/deep-dives/main-loop-internals.md)

LLMs have limited context windows. Long conversations overflow. opencode solves this with **compaction** - summarizing old messages to free up space. This deep dive explores how it works.

---

## The Problem

Every LLM has a maximum context window (e.g., 200K tokens for Claude Sonnet). A conversation includes:
- System prompt (~2K tokens)
- Conversation history (variable)
- Tool outputs (can be huge!)

```
┌─────────────────────────────────────────────────────────────┐
│                     Context Overflow                         │
│                                                              │
│  Turn 1: Read main.py          → 1,500 tokens               │
│  Turn 2: Read utils.py         → 2,000 tokens               │
│  Turn 3: Run tests             → 3,500 tokens               │
│  ...                                                         │
│  Turn 50: Fix the bug          → ???                        │
│                                                              │
│  Total: 185,000 tokens                                      │
│  Context limit: 200,000 tokens                              │
│  Remaining: 15,000 tokens                                   │
│                                                              │
│  ⚠️  PROBLEM: Running out of space!                         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## opencode's Solution: Compaction

When the context gets full (>90% of limit), opencode:
1. **Detects overflow** by tracking token usage
2. **Creates a summary** using a separate LLM call
3. **Replaces old messages** with the summary
4. **Continues the conversation** with room to spare

```
Before:
┌────────────────────────────────────────┐
│ [User: "Read main.py"]                │
│ [Assistant: tool_call + 1500 lines]   │  ← Old messages
│ [User: "Read utils.py"]               │
│ [Assistant: tool_call + 2000 lines]   │
│ [User: "Fix the bug"]                 │  ← Recent (keep)
│ [Assistant: ...]                      │
└────────────────────────────────────────┘
        185,000 tokens

After compaction:
┌────────────────────────────────────────┐
│ [Summary: "We explored main.py and    │  ← Summary (2K tokens)
│  utils.py to understand the bug..."]  │
│ [User: "Fix the bug"]                 │  ← Recent (keep)
│ [Assistant: ...]                      │
└────────────────────────────────────────┘
        25,000 tokens
```

---

## Overflow Detection

The processor tracks token usage after each LLM call:

```typescript
// session/processor.ts
case "finish":
  // Save token usage
  input.assistantMessage.tokens = {
    input: value.usage.inputTokens,
    output: value.usage.outputTokens,
    cache: {
      read: value.usage.cacheReadTokens || 0,
      write: value.usage.cacheWriteTokens || 0,
    }
  }

  // Check if approaching limit (90% threshold)
  const threshold = input.model.limit.context * 0.9
  if (input.assistantMessage.tokens.input > threshold) {
    needsCompaction = true  // Signal to main loop
  }
```

The main loop receives the signal and triggers compaction:

```typescript
// session/prompt.ts
const result = await processor.process({ ... })

if (result === "compact") {
  await SessionCompaction.create({
    sessionID,
    messages: allMessages,
    abort,
  })
  continue  // Retry loop with more space
}
```

---

## The Compaction Process

```typescript
// session/compaction.ts (simplified)
export namespace SessionCompaction {
  export async function create(input: {
    sessionID: string
    messages: MessageV2.WithParts[]
    abort: AbortSignal
  }) {
    // ─────────────────────────────────────────────────────────
    // 1. Find the cutoff point (keep recent messages)
    // ─────────────────────────────────────────────────────────
    const KEEP_RECENT = 10  // Keep last 10 messages
    const cutoff = input.messages.length - KEEP_RECENT
    const toSummarize = input.messages.slice(0, cutoff)
    const toKeep = input.messages.slice(cutoff)

    // ─────────────────────────────────────────────────────────
    // 2. Create a summary message
    // ─────────────────────────────────────────────────────────
    const summaryMessage = await Session.createMessage({
      sessionID: input.sessionID,
      role: "assistant",
      summary: true,  // Mark as summary
      content: "Generating summary...",
    })

    // ─────────────────────────────────────────────────────────
    // 3. Use LLM to generate summary
    // ─────────────────────────────────────────────────────────
    const summaryPrompt = `
      Summarize the following conversation history.
      Focus on:
      - Key decisions made
      - Files that were modified
      - Important context for continuing the conversation
      - Any unresolved issues

      Be concise but don't lose critical information.
    `

    const processor = SessionProcessor.create({
      assistantMessage: summaryMessage,
      sessionID: input.sessionID,
      model: input.model,
      abort: input.abort,
    })

    await processor.process({
      system: [summaryPrompt],
      messages: toSummarize,  // Summarize old messages
      tools: {},  // No tools for summarization
      model: input.model,
    })

    // ─────────────────────────────────────────────────────────
    // 4. Mark old messages as compacted
    // ─────────────────────────────────────────────────────────
    for (const msg of toSummarize) {
      await Session.updateMessage({
        ...msg,
        compacted: true,  // Hide from future requests
      })
    }

    // ─────────────────────────────────────────────────────────
    // 5. Update conversation order
    // ─────────────────────────────────────────────────────────
    // Summary now appears before kept messages
    // Compacted messages are filtered out

    log.info("compaction.complete", {
      summarized: toSummarize.length,
      kept: toKeep.length,
    })
  }

  export async function isOverflow(input: {
    tokens: MessageV2.Assistant["tokens"]
    model: Provider.Model
  }): Promise<boolean> {
    const context = input.model.limit.context
    if (context === 0) return false  // No limit

    const total = (
      input.tokens.input +
      input.tokens.cache.read +
      input.tokens.output
    )

    // Check if over 90% of limit
    const threshold = context * 0.9
    return total > threshold
  }
}
```

---

## Filtering Compacted Messages

When loading messages, opencode filters out compacted ones:

```typescript
// session/message-v2.ts
export namespace MessageV2 {
  export async function filterCompacted(
    messages: AsyncIterable<WithParts>
  ): Promise<WithParts[]> {
    const result: WithParts[] = []

    for await (const msg of messages) {
      // Skip compacted messages (they're replaced by summary)
      if (msg.compacted) continue

      result.push(msg)
    }

    return result
  }
}
```

The conversation now looks like:

```
[Summary message]         ← Replaces old messages
[Recent message 1]        ← Kept as-is
[Recent message 2]        ← Kept as-is
[Current message]         ← Kept as-is
```

---

## Summary Message Format

The summary message has special properties:

```typescript
{
  role: "assistant",
  summary: true,           // Flag as summary
  compacted: false,        // Include in conversation
  content: "...",          // The actual summary
  tokens: { ... },         // Much smaller than original
}
```

It appears in the conversation like a normal assistant message but is marked as a summary.

---

## Token Tracking

opencode tracks tokens throughout:

```typescript
interface Tokens {
  input: number          // Tokens sent to LLM
  output: number         // Tokens in response
  cache: {
    read: number        // Cached tokens read
    write: number       // Tokens written to cache
  }
}
```

Cache tokens (Claude's prompt caching) reduce costs:
- First time: Full input tokens counted
- Subsequent calls: Cached tokens are much cheaper

---

## Python Equivalent

```python
from dataclasses import dataclass


@dataclass
class Tokens:
    input_tokens: int
    output_tokens: int
    cache_read: int
    cache_write: int


class CompactionManager:
    KEEP_RECENT = 10      # Keep last N messages
    OVERFLOW_THRESHOLD = 0.9  # 90% of context limit

    def __init__(self, model: Model):
        self.model = model

    async def is_overflow(self, tokens: Tokens) -> bool:
        """Check if context is approaching limit."""
        if self.model.context_limit == 0:
            return False  # No limit

        total = tokens.input_tokens + tokens.cache_read + tokens.output_tokens
        threshold = self.model.context_limit * self.OVERFLOW_THRESHOLD

        return total > threshold

    async def compact(
        self,
        session_id: str,
        messages: list[Message],
        abort: asyncio.Event
    ) -> None:
        """Summarize old messages to free up context."""

        # 1. Find cutoff point
        cutoff = len(messages) - self.KEEP_RECENT
        to_summarize = messages[:cutoff]
        to_keep = messages[cutoff:]

        # 2. Create summary message
        summary_message = await create_message(
            session_id=session_id,
            role="assistant",
            summary=True,
            content="Generating summary..."
        )

        # 3. Generate summary using LLM
        summary_prompt = """
        Summarize the following conversation history.
        Focus on:
        - Key decisions made
        - Files that were modified
        - Important context
        - Unresolved issues

        Be concise but preserve critical information.
        """

        response = await call_llm(
            system=[summary_prompt],
            messages=to_summarize,
            tools={},  # No tools
        )

        summary_message.content = response.content

        # 4. Mark old messages as compacted
        for msg in to_summarize:
            msg.compacted = True
            await update_message(msg)

        # 5. Save summary
        await update_message(summary_message)

        print(f"Compacted {len(to_summarize)} messages, kept {len(to_keep)}")

    def filter_compacted(self, messages: list[Message]) -> list[Message]:
        """Remove compacted messages from conversation."""
        return [msg for msg in messages if not msg.compacted]
```

---

## Trade-offs

**Benefits**:
- ✅ Enables unlimited conversation length
- ✅ Keeps recent context intact
- ✅ Automatic (no user intervention)

**Costs**:
- ❌ Extra LLM call for summarization
- ❌ Potential information loss
- ❌ Slight delay during compaction

**Mitigation**:
- Keep recent messages (10+) to preserve working context
- Use good summarization prompts
- Track what was compacted in logs

---

## Alternative Strategies

Some agents use different strategies:

### 1. Rolling Window

Keep only last N messages, discard older ones:

```python
messages = messages[-50:]  # Keep last 50
```

**Pros**: Simple, fast
**Cons**: Loses all old context

### 2. Selective Pruning

Remove tool outputs but keep text:

```python
for msg in messages:
    if msg.role == "assistant":
        # Keep text, remove tool results
        msg.parts = [p for p in msg.parts if p.type != "tool_result"]
```

**Pros**: Faster than summarization
**Cons**: Can still overflow

### 3. External Memory

Store old context in a database, retrieve as needed:

```python
# Store in vector database
await memory.store(old_messages)

# Retrieve relevant context
relevant = await memory.search(current_query)
```

**Pros**: True unlimited context
**Cons**: Complex, requires embeddings

opencode uses **summarization** as the best balance of simplicity and effectiveness.

---

## Summary

Compaction:
- **Detects overflow** when context reaches 90% capacity
- **Summarizes old messages** using a separate LLM call
- **Marks messages as compacted** so they're filtered out
- **Keeps recent messages** to preserve working context
- **Enables unlimited conversations** without losing critical information

This is crucial for long-running agent sessions that read many files and run many commands.

---

## Related Deep Dives

- [Main Loop Internals](/deep-dives/main-loop-internals.md) - How compaction is triggered
- [The Processor](/deep-dives/processor.md) - Token tracking and overflow detection
- [Provider Abstraction](/deep-dives/provider-abstraction.md) - Token counting across providers
