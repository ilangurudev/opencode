# Day 9-10: Context & Memory Management

> **Goal**: Understand how agents manage conversation history and handle context limits.

---

## The Problem: Context Limits

LLMs have limited context windows. A conversation that's 50,000 tokens long won't fit in a model with a 32,000 token limit.

```
┌─────────────────────────────────────────────────────────────────┐
│                     The Context Problem                          │
│                                                                  │
│   Turn 1: "Read src/main.py"           1,500 tokens             │
│   Turn 2: "Now read utils.py"          2,000 tokens             │
│   Turn 3: "And config.py"              1,800 tokens             │
│   ...                                                            │
│   Turn 50: "Fix the bug"                                        │
│                                                                  │
│   Total: 85,000 tokens                                          │
│   Context limit: 32,000 tokens                                  │
│                                                                  │
│   PROBLEM: Can't fit entire conversation!                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

Agents solve this with:
1. **Compaction** - Summarize old messages
2. **Pruning** - Remove old tool outputs
3. **Token counting** - Track usage proactively

---

## opencode's Approach: Compaction

When context gets full, opencode:
1. Detects overflow (token count > limit)
2. Uses a separate LLM call to summarize the conversation
3. Replaces old messages with the summary
4. Continues with room to spare

```
┌─────────────────────────────────────────────────────────────────┐
│                     Compaction Flow                              │
│                                                                  │
│   Before:                                                        │
│   ┌────────────────────────────────────────────┐                │
│   │ [User: "Read main.py"]                    │                │
│   │ [Assistant: tool_call + 1500 lines]       │                │
│   │ [User: "Read utils.py"]                   │                │
│   │ [Assistant: tool_call + 2000 lines]       │                │
│   │ [User: "Fix the bug"]                     │                │
│   │ [Assistant: ...]                          │ 85K tokens     │
│   └────────────────────────────────────────────┘                │
│                                                                  │
│   After compaction:                                             │
│   ┌────────────────────────────────────────────┐                │
│   │ [Summary: "We explored main.py and        │                │
│   │  utils.py. The user found a bug in        │                │
│   │  the calculate() function. We're          │                │
│   │  working on fixing it..."]                │ 2K tokens      │
│   │ [User: "Fix the bug"]                     │                │
│   │ [Assistant: ...]                          │                │
│   └────────────────────────────────────────────┘                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

Here's the actual code:

```typescript
// session/compaction.ts (simplified)
export async function isOverflow(input: {
  tokens: MessageV2.Assistant["tokens"]
  model: Provider.Model
}) {
  const context = input.model.limit.context
  if (context === 0) return false

  const count = input.tokens.input + input.tokens.cache.read + input.tokens.output
  const usable = input.model.limit.input || context - output
  return count > usable
}

export async function process(input: {
  messages: MessageV2.WithParts[]
  sessionID: string
  abort: AbortSignal
}) {
  // Create a summary message
  const msg = await Session.updateMessage({
    role: "assistant",
    summary: true,  // Mark as summary
    // ...
  })

  // Use LLM to generate summary
  const result = await processor.process({
    tools: {},  // No tools for summarization
    system: [],
    messages: input.messages,
    prompt: `Provide a detailed prompt for continuing our conversation.
             Focus on: what we did, what we're doing, which files
             we're working on, and what we're going to do next.`
  })

  return result
}
```

---

## Pruning: Removing Old Tool Outputs

Before compaction, opencode "prunes" old tool outputs:

```typescript
// session/compaction.ts
export async function prune(input: { sessionID: string }) {
  const msgs = await Session.messages({ sessionID: input.sessionID })
  let total = 0
  const toPrune = []

  // Walk backwards through messages
  for (let i = msgs.length - 1; i >= 0; i--) {
    for (const part of msg.parts) {
      if (part.type === "tool" && part.state.status === "completed") {
        const estimate = Token.estimate(part.state.output)
        total += estimate

        // Keep last ~40K tokens of tool outputs
        if (total > PRUNE_PROTECT) {
          toPrune.push(part)
        }
      }
    }
  }

  // Mark old outputs as compacted (they won't be sent to LLM)
  for (const part of toPrune) {
    part.state.time.compacted = Date.now()
    await Session.updatePart(part)
  }
}
```

Pruned tool outputs are still stored but not sent to the LLM.

---

## Python Implementation: Memory System

```python
import tiktoken
from dataclasses import dataclass, field
from typing import Optional
from datetime import datetime


@dataclass
class Message:
    """A message in the conversation."""
    id: str
    role: str  # "user" or "assistant"
    content: str
    tool_calls: list = field(default_factory=list)
    tool_results: list = field(default_factory=list)
    created_at: datetime = field(default_factory=datetime.now)
    is_summary: bool = False
    tokens: int = 0


class TokenCounter:
    """Counts tokens for context management."""

    def __init__(self, model: str = "cl100k_base"):
        try:
            self.encoder = tiktoken.get_encoding(model)
        except:
            self.encoder = None

    def count(self, text: str) -> int:
        """Count tokens in text."""
        if self.encoder:
            return len(self.encoder.encode(text))
        # Fallback: rough estimate
        return len(text) // 4

    def count_message(self, msg: Message) -> int:
        """Count tokens in a message."""
        total = self.count(msg.content)
        for tc in msg.tool_calls:
            total += self.count(str(tc))
        for tr in msg.tool_results:
            total += self.count(str(tr))
        return total


@dataclass
class ConversationMemory:
    """
    Manages conversation history with context limits.

    Features:
    - Token counting
    - Automatic compaction when approaching limit
    - Pruning of old tool outputs
    """
    messages: list[Message] = field(default_factory=list)
    context_limit: int = 128_000  # Default Claude limit
    target_after_compaction: int = 30_000  # Target size after compaction
    token_counter: TokenCounter = field(default_factory=TokenCounter)

    def add_message(self, msg: Message) -> None:
        """Add a message to history."""
        msg.tokens = self.token_counter.count_message(msg)
        self.messages.append(msg)

    def get_total_tokens(self) -> int:
        """Get total tokens in conversation."""
        return sum(m.tokens for m in self.messages)

    def needs_compaction(self) -> bool:
        """Check if compaction is needed."""
        return self.get_total_tokens() > self.context_limit * 0.8

    def prune_tool_outputs(self) -> int:
        """
        Remove old tool outputs to save space.
        Returns number of tokens saved.
        """
        saved = 0
        protected_tokens = 40_000
        total = 0

        # Walk backwards, keep recent tool outputs
        for msg in reversed(self.messages):
            for i, result in enumerate(msg.tool_results):
                result_tokens = self.token_counter.count(str(result))
                total += result_tokens

                if total > protected_tokens:
                    # Replace with placeholder
                    original = msg.tool_results[i]
                    msg.tool_results[i] = {"pruned": True, "summary": f"[Output pruned - was {result_tokens} tokens]"}
                    saved += result_tokens - 50  # Placeholder is ~50 tokens

        # Recalculate message tokens
        for msg in self.messages:
            msg.tokens = self.token_counter.count_message(msg)

        return saved

    async def compact(self, summarize_fn) -> None:
        """
        Compact the conversation by summarizing old messages.

        Args:
            summarize_fn: Async function that takes messages and returns a summary string
        """
        if not self.needs_compaction():
            return

        # First try pruning
        saved = self.prune_tool_outputs()
        if not self.needs_compaction():
            return

        # Find split point - keep recent messages
        keep_tokens = self.target_after_compaction
        split_idx = len(self.messages)

        running_total = 0
        for i in range(len(self.messages) - 1, -1, -1):
            running_total += self.messages[i].tokens
            if running_total > keep_tokens:
                split_idx = i + 1
                break

        if split_idx <= 1:
            return  # Nothing to compact

        # Get messages to summarize
        to_summarize = self.messages[:split_idx]
        to_keep = self.messages[split_idx:]

        # Generate summary
        summary_text = await summarize_fn(to_summarize)

        # Create summary message
        summary_msg = Message(
            id=f"summary_{datetime.now().timestamp()}",
            role="assistant",
            content=summary_text,
            is_summary=True,
        )
        summary_msg.tokens = self.token_counter.count_message(summary_msg)

        # Replace messages
        self.messages = [summary_msg] + to_keep

    def get_messages_for_llm(self) -> list[dict]:
        """Get messages formatted for LLM API."""
        result = []

        for msg in self.messages:
            if msg.is_summary:
                # Summary goes as system or user message
                result.append({
                    "role": "user",
                    "content": f"[Conversation Summary]\n{msg.content}"
                })
            elif msg.role == "user":
                result.append({
                    "role": "user",
                    "content": msg.content
                })
            elif msg.role == "assistant":
                content = []
                if msg.content:
                    content.append({"type": "text", "text": msg.content})
                for tc in msg.tool_calls:
                    content.append({
                        "type": "tool_use",
                        "id": tc["id"],
                        "name": tc["name"],
                        "input": tc["input"],
                    })
                result.append({"role": "assistant", "content": content})

                # Add tool results
                if msg.tool_results:
                    tool_content = []
                    for i, tr in enumerate(msg.tool_results):
                        if tr.get("pruned"):
                            tool_content.append({
                                "type": "tool_result",
                                "tool_use_id": msg.tool_calls[i]["id"],
                                "content": tr["summary"],
                            })
                        else:
                            tool_content.append({
                                "type": "tool_result",
                                "tool_use_id": tr["tool_use_id"],
                                "content": tr["content"],
                            })
                    result.append({"role": "user", "content": tool_content})

        return result


# Example summarization function
async def summarize_conversation(messages: list[Message], client) -> str:
    """Use LLM to summarize conversation."""
    # Build a simple representation of the conversation
    conversation = []
    for msg in messages:
        if msg.role == "user":
            conversation.append(f"User: {msg.content[:500]}")
        else:
            conversation.append(f"Assistant: {msg.content[:500]}")
            for tc in msg.tool_calls:
                conversation.append(f"  [Tool: {tc['name']}]")

    prompt = f"""Summarize this conversation for context continuity.
Focus on:
- What task the user is working on
- What files were examined/modified
- Current state and next steps
- Any important decisions or findings

Conversation:
{chr(10).join(conversation)}

Provide a detailed summary that would help continue this conversation:"""

    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=2000,
        messages=[{"role": "user", "content": prompt}]
    )

    return response.content[0].text
```

---

## Message Storage

opencode stores messages in SQLite for persistence:

```typescript
// session/message-v2.ts (conceptual)
export namespace MessageV2 {
  // Store message
  export async function updateMessage(msg: Message) {
    await db.messages.upsert({
      where: { id: msg.id },
      data: msg,
    })
    Bus.publish(Event.MessageUpdate, { message: msg })
    return msg
  }

  // Store message part (text, tool call, etc.)
  export async function updatePart(part: Part) {
    await db.parts.upsert({
      where: { id: part.id },
      data: part,
    })
    Bus.publish(Event.PartUpdate, { part })
    return part
  }

  // Stream all messages for a session
  export async function* stream(sessionID: string) {
    const messages = await db.messages.findMany({
      where: { sessionID },
      orderBy: { id: "asc" },
    })

    for (const msg of messages) {
      const parts = await db.parts.findMany({
        where: { messageID: msg.id },
        orderBy: { id: "asc" },
      })
      yield { info: msg, parts }
    }
  }
}
```

**Python equivalent:**

```python
import sqlite3
import json
from pathlib import Path


class MessageStore:
    """SQLite-based message storage."""

    def __init__(self, db_path: Path = Path(".agent_memory.db")):
        self.db_path = db_path
        self._init_db()

    def _init_db(self):
        with sqlite3.connect(self.db_path) as conn:
            conn.execute("""
                CREATE TABLE IF NOT EXISTS messages (
                    id TEXT PRIMARY KEY,
                    session_id TEXT NOT NULL,
                    role TEXT NOT NULL,
                    content TEXT,
                    tool_calls TEXT,
                    tool_results TEXT,
                    is_summary INTEGER DEFAULT 0,
                    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
                )
            """)
            conn.execute("""
                CREATE INDEX IF NOT EXISTS idx_session
                ON messages(session_id, created_at)
            """)

    def save_message(self, session_id: str, msg: Message):
        with sqlite3.connect(self.db_path) as conn:
            conn.execute("""
                INSERT OR REPLACE INTO messages
                (id, session_id, role, content, tool_calls, tool_results, is_summary)
                VALUES (?, ?, ?, ?, ?, ?, ?)
            """, (
                msg.id,
                session_id,
                msg.role,
                msg.content,
                json.dumps(msg.tool_calls),
                json.dumps(msg.tool_results),
                1 if msg.is_summary else 0,
            ))

    def load_session(self, session_id: str) -> list[Message]:
        with sqlite3.connect(self.db_path) as conn:
            conn.row_factory = sqlite3.Row
            rows = conn.execute("""
                SELECT * FROM messages
                WHERE session_id = ?
                ORDER BY created_at
            """, (session_id,)).fetchall()

        return [
            Message(
                id=row["id"],
                role=row["role"],
                content=row["content"] or "",
                tool_calls=json.loads(row["tool_calls"] or "[]"),
                tool_results=json.loads(row["tool_results"] or "[]"),
                is_summary=bool(row["is_summary"]),
            )
            for row in rows
        ]
```

---

## Summary

| Concept | Purpose |
|---------|---------|
| **Token counting** | Track context usage |
| **Pruning** | Remove old tool outputs |
| **Compaction** | Summarize old messages |
| **Persistence** | Store messages across sessions |

The memory system ensures your agent can handle long conversations without hitting context limits.

---

## Assignment: Add Memory System

Add conversation memory and compaction to your agent.

**Start here**: [Assignment 5: Memory System](./assignments/05-memory.md)

---

## Optional Deep Dives

Want to understand context management in more detail?

- 💾 [Compaction](/deep-dives/compaction.md) - How opencode manages context limits
- 📊 [The Processor](/deep-dives/processor.md) - Token tracking and overflow detection
