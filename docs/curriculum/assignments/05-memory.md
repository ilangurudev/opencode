# Assignment 5: Memory System

> **Goal**: Add conversation memory with token counting, pruning, and compaction to your agent.

---

## What You'll Build

A memory system that:
1. Stores conversation history
2. Counts tokens to track context usage
3. Prunes old messages when approaching limits
4. Compacts conversations using LLM summarization

---

## Starter Code

Extend your agent with this memory module:

```python
# memory.py
from dataclasses import dataclass, field
from typing import Optional
import tiktoken
import json

@dataclass
class Message:
    """A single message in the conversation."""
    role: str  # "user", "assistant", "system"
    content: str
    tool_calls: list = field(default_factory=list)
    tool_call_id: Optional[str] = None
    name: Optional[str] = None  # For tool results
    tokens: int = 0  # Cached token count

    def to_api_format(self) -> dict:
        """Convert to format expected by LLM API."""
        msg = {"role": self.role, "content": self.content}
        if self.tool_calls:
            msg["tool_calls"] = self.tool_calls
        if self.tool_call_id:
            msg["tool_call_id"] = self.tool_call_id
        if self.name:
            msg["name"] = self.name
        return msg


class Memory:
    """Manages conversation history with token limits."""

    def __init__(
        self,
        max_tokens: int = 100000,
        reserve_tokens: int = 20000,
        model: str = "gpt-4"
    ):
        self.messages: list[Message] = []
        self.max_tokens = max_tokens
        self.reserve_tokens = reserve_tokens
        self.model = model

        # Initialize tokenizer
        try:
            self.encoder = tiktoken.encoding_for_model(model)
        except KeyError:
            self.encoder = tiktoken.get_encoding("cl100k_base")

    def count_tokens(self, text: str) -> int:
        """Count tokens in a string."""
        # TODO: Implement token counting
        pass

    def add_message(self, message: Message) -> None:
        """Add a message to history, counting its tokens."""
        # TODO: Count tokens and add message
        pass

    def get_messages(self) -> list[dict]:
        """Get messages in API format."""
        # TODO: Return messages, pruning if needed
        pass

    def total_tokens(self) -> int:
        """Get total tokens in conversation."""
        # TODO: Sum all message tokens
        pass

    def needs_pruning(self) -> bool:
        """Check if we need to prune messages."""
        # TODO: Check against max_tokens - reserve_tokens
        pass

    def prune(self) -> None:
        """Remove oldest messages to stay under limit."""
        # TODO: Remove messages while over limit
        pass

    def needs_compaction(self) -> bool:
        """Check if conversation should be compacted."""
        # TODO: Check if we have many messages or high token count
        pass

    async def compact(self, llm_client) -> None:
        """Summarize old messages into a single message."""
        # TODO: Use LLM to summarize, replace old messages
        pass
```

---

## Part 1: Token Counting

Implement accurate token counting:

```python
def count_tokens(self, text: str) -> int:
    """Count tokens in a string."""
    if not text:
        return 0
    return len(self.encoder.encode(text))

def count_message_tokens(self, message: Message) -> int:
    """
    Count tokens for a full message.

    Messages have overhead beyond just content:
    - Role token
    - Content tokens
    - Tool call tokens (if any)
    - Formatting tokens
    """
    tokens = 4  # Base overhead per message

    tokens += self.count_tokens(message.content)

    if message.tool_calls:
        for tc in message.tool_calls:
            tokens += self.count_tokens(tc.get("function", {}).get("name", ""))
            tokens += self.count_tokens(tc.get("function", {}).get("arguments", ""))
            tokens += 4  # Tool call overhead

    if message.name:
        tokens += self.count_tokens(message.name)

    return tokens

def add_message(self, message: Message) -> None:
    """Add a message to history, counting its tokens."""
    message.tokens = self.count_message_tokens(message)
    self.messages.append(message)
```

### Test Your Token Counter

```python
def test_token_counting():
    memory = Memory()

    # Simple text
    assert memory.count_tokens("Hello, world!") > 0

    # Empty string
    assert memory.count_tokens("") == 0

    # Code block
    code = """
    def hello():
        print("Hello, world!")
    """
    tokens = memory.count_tokens(code)
    print(f"Code block: {tokens} tokens")

    # Full message
    msg = Message(role="user", content="What is 2 + 2?")
    memory.add_message(msg)
    assert msg.tokens > 0
    print(f"Message tokens: {msg.tokens}")
```

---

## Part 2: Message Management

Implement the core message operations:

```python
def get_messages(self) -> list[dict]:
    """Get messages in API format, ensuring we're under limit."""
    if self.needs_pruning():
        self.prune()

    return [m.to_api_format() for m in self.messages]

def total_tokens(self) -> int:
    """Get total tokens in conversation."""
    return sum(m.tokens for m in self.messages)

def needs_pruning(self) -> bool:
    """Check if we need to prune messages."""
    available = self.max_tokens - self.reserve_tokens
    return self.total_tokens() > available

def prune(self) -> None:
    """
    Remove oldest messages to stay under limit.

    Important considerations:
    1. Never remove the system message (first message)
    2. Don't break tool call/result pairs
    3. Remove from oldest first
    """
    if not self.messages:
        return

    target = self.max_tokens - self.reserve_tokens

    # Find system message (usually first)
    system_msg = None
    if self.messages and self.messages[0].role == "system":
        system_msg = self.messages[0]
        self.messages = self.messages[1:]

    # Remove oldest messages until under limit
    while self.messages and self.total_tokens() > target:
        # Find the oldest removable message
        # Skip tool results that follow tool calls
        removed = self._remove_oldest_safe()
        if not removed:
            break

    # Restore system message
    if system_msg:
        self.messages.insert(0, system_msg)

def _remove_oldest_safe(self) -> bool:
    """
    Remove the oldest message safely.

    Returns True if a message was removed.
    """
    if not self.messages:
        return False

    # Check if first message is an assistant with tool calls
    first = self.messages[0]

    if first.role == "assistant" and first.tool_calls:
        # Need to also remove the tool results that follow
        tool_call_ids = {tc["id"] for tc in first.tool_calls}

        # Find how many subsequent messages are tool results
        remove_count = 1
        for msg in self.messages[1:]:
            if msg.tool_call_id and msg.tool_call_id in tool_call_ids:
                remove_count += 1
            else:
                break

        self.messages = self.messages[remove_count:]
    else:
        self.messages = self.messages[1:]

    return True
```

### Test Pruning

```python
def test_pruning():
    # Small limit for testing
    memory = Memory(max_tokens=1000, reserve_tokens=200)

    # Add system message
    memory.add_message(Message(role="system", content="You are a helpful assistant."))

    # Add many messages to exceed limit
    for i in range(50):
        memory.add_message(Message(role="user", content=f"Message number {i} " * 20))
        memory.add_message(Message(role="assistant", content=f"Response {i} " * 20))

    print(f"Before pruning: {len(memory.messages)} messages, {memory.total_tokens()} tokens")

    # Get messages triggers pruning
    messages = memory.get_messages()

    print(f"After pruning: {len(memory.messages)} messages, {memory.total_tokens()} tokens")

    # System message should still be first
    assert memory.messages[0].role == "system"

    # Should be under limit
    assert memory.total_tokens() <= memory.max_tokens - memory.reserve_tokens
```

---

## Part 3: Compaction

Implement LLM-based summarization:

```python
COMPACTION_PROMPT = """Summarize the following conversation, preserving:
1. Key decisions made
2. Important code changes or file operations
3. Current task status
4. Any errors encountered and their resolutions

Be concise but complete. The summary will replace the original messages.

Conversation:
{conversation}

Summary:"""

def needs_compaction(self) -> bool:
    """
    Check if conversation should be compacted.

    Compact when:
    1. We have many messages (> 20 non-system messages)
    2. OR total tokens exceed 60% of max
    """
    non_system = [m for m in self.messages if m.role != "system"]

    message_threshold = len(non_system) > 20
    token_threshold = self.total_tokens() > self.max_tokens * 0.6

    return message_threshold or token_threshold

async def compact(self, llm_client) -> None:
    """
    Summarize old messages into a single message.

    Strategy:
    1. Keep system message
    2. Keep recent messages (last 5-10)
    3. Summarize the rest
    """
    if len(self.messages) < 10:
        return  # Not enough to compact

    # Separate system message
    system_msg = None
    if self.messages[0].role == "system":
        system_msg = self.messages[0]
        working_messages = self.messages[1:]
    else:
        working_messages = self.messages[:]

    # Keep recent messages
    keep_recent = 6  # Last 3 exchanges
    to_summarize = working_messages[:-keep_recent]
    to_keep = working_messages[-keep_recent:]

    if not to_summarize:
        return  # Nothing to summarize

    # Format conversation for summarization
    conversation_text = self._format_for_summary(to_summarize)

    # Call LLM to summarize
    prompt = COMPACTION_PROMPT.format(conversation=conversation_text)

    summary = await llm_client.complete(prompt)

    # Create summary message
    summary_msg = Message(
        role="user",
        content=f"[Previous conversation summary]\n{summary}\n[End summary]"
    )
    summary_msg.tokens = self.count_message_tokens(summary_msg)

    # Rebuild message list
    self.messages = []
    if system_msg:
        self.messages.append(system_msg)
    self.messages.append(summary_msg)
    self.messages.extend(to_keep)

    # Recount tokens for kept messages
    for msg in to_keep:
        msg.tokens = self.count_message_tokens(msg)

def _format_for_summary(self, messages: list[Message]) -> str:
    """Format messages into a string for summarization."""
    lines = []
    for msg in messages:
        if msg.role == "user":
            lines.append(f"User: {msg.content}")
        elif msg.role == "assistant":
            if msg.tool_calls:
                tools = [tc["function"]["name"] for tc in msg.tool_calls]
                lines.append(f"Assistant: [Called tools: {', '.join(tools)}]")
            if msg.content:
                lines.append(f"Assistant: {msg.content}")
        elif msg.role == "tool":
            # Truncate long tool results
            content = msg.content[:500] + "..." if len(msg.content) > 500 else msg.content
            lines.append(f"Tool ({msg.name}): {content}")

    return "\n".join(lines)
```

### Test Compaction

```python
async def test_compaction():
    memory = Memory()

    # Create a mock LLM client
    class MockLLM:
        async def complete(self, prompt):
            return "User asked about Python. Assistant explained variables and loops."

    # Add system message
    memory.add_message(Message(role="system", content="You are a helpful assistant."))

    # Add many messages
    for i in range(25):
        memory.add_message(Message(role="user", content=f"Question {i}?"))
        memory.add_message(Message(role="assistant", content=f"Answer {i}."))

    print(f"Before compaction: {len(memory.messages)} messages")
    assert memory.needs_compaction()

    await memory.compact(MockLLM())

    print(f"After compaction: {len(memory.messages)} messages")
    # Should have: system + summary + recent messages
    assert len(memory.messages) < 25

    # Summary should be included
    assert "[Previous conversation summary]" in memory.messages[1].content
```

---

## Part 4: Integration

Connect memory to your agent:

```python
# agent.py
from memory import Memory, Message

class Agent:
    def __init__(self, llm_client, tools):
        self.llm = llm_client
        self.tools = tools
        self.memory = Memory(
            max_tokens=100000,
            reserve_tokens=20000
        )

        # Add system message
        self.memory.add_message(Message(
            role="system",
            content=self._build_system_prompt()
        ))

    async def run(self, user_input: str) -> str:
        # Add user message
        self.memory.add_message(Message(
            role="user",
            content=user_input
        ))

        while True:
            # Check for compaction
            if self.memory.needs_compaction():
                print("[Compacting conversation...]")
                await self.memory.compact(self.llm)

            # Get messages (may trigger pruning)
            messages = self.memory.get_messages()

            # Show context usage
            total = self.memory.total_tokens()
            print(f"[Context: {total:,} / {self.memory.max_tokens:,} tokens]")

            # Call LLM
            response = await self.llm.chat(messages, tools=self.tools)

            # Add assistant message
            self.memory.add_message(Message(
                role="assistant",
                content=response.content or "",
                tool_calls=response.tool_calls or []
            ))

            # Handle tool calls
            if response.tool_calls:
                for tool_call in response.tool_calls:
                    result = await self.execute_tool(tool_call)
                    self.memory.add_message(Message(
                        role="tool",
                        content=result,
                        tool_call_id=tool_call["id"],
                        name=tool_call["function"]["name"]
                    ))
            else:
                # No tool calls, return response
                return response.content
```

---

## Part 5: Advanced Features (Optional)

### Persistence

Save and load conversation history:

```python
import json
from pathlib import Path

class Memory:
    # ... existing methods ...

    def save(self, path: Path) -> None:
        """Save conversation to file."""
        data = {
            "messages": [
                {
                    "role": m.role,
                    "content": m.content,
                    "tool_calls": m.tool_calls,
                    "tool_call_id": m.tool_call_id,
                    "name": m.name,
                    "tokens": m.tokens
                }
                for m in self.messages
            ],
            "max_tokens": self.max_tokens,
            "reserve_tokens": self.reserve_tokens
        }

        path.write_text(json.dumps(data, indent=2))

    @classmethod
    def load(cls, path: Path, model: str = "gpt-4") -> "Memory":
        """Load conversation from file."""
        data = json.loads(path.read_text())

        memory = cls(
            max_tokens=data["max_tokens"],
            reserve_tokens=data["reserve_tokens"],
            model=model
        )

        for m in data["messages"]:
            memory.messages.append(Message(
                role=m["role"],
                content=m["content"],
                tool_calls=m.get("tool_calls", []),
                tool_call_id=m.get("tool_call_id"),
                name=m.get("name"),
                tokens=m.get("tokens", 0)
            ))

        return memory
```

### Selective Compaction

Only compact certain types of messages:

```python
def selective_compact(self, llm_client, keep_types: list[str] = None) -> None:
    """
    Compact while preserving certain message types.

    keep_types might include:
    - "code_changes": Messages about file edits
    - "errors": Error messages and resolutions
    - "decisions": User decisions and confirmations
    """
    # Implementation left as exercise
    pass
```

---

## Deliverables

1. **memory.py** with:
   - Token counting using tiktoken
   - Message storage and retrieval
   - Pruning that preserves tool call pairs
   - LLM-based compaction

2. **Integration** showing:
   - Memory connected to agent loop
   - Context usage display
   - Automatic compaction trigger

3. **Tests** demonstrating:
   - Token counting accuracy
   - Pruning behavior
   - Compaction results

---

## Hints

1. **Token counting is approximate**: Different APIs count tokens slightly differently. Use tiktoken with `cl100k_base` encoding as a reasonable estimate.

2. **Tool call pairs are atomic**: If you remove an assistant message with tool calls, you must also remove the corresponding tool result messages.

3. **Compaction quality matters**: The summary prompt significantly affects how well context is preserved. Test with real conversations.

4. **Reserve tokens are important**: Always keep reserve capacity for the next response and tool calls.

---

## What's Next

In [Day 11-12](../day-11-12-extensibility.md), you'll add:
- User-defined skills (prompt templates)
- Commands with parameters
- MCP server integration

Your agent will become extensible without code changes.
