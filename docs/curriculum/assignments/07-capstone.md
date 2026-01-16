# Assignment 7: Capstone

> **Goal**: Polish your agent, add a significant improvement, and demonstrate your understanding of coding agent architecture.

---

## Choose Your Path

Pick one or more options based on your interests:

### Option A: Robustness & Production Polish

Make your agent production-ready with error handling, retries, and better UX.

### Option B: New Tool Implementation

Add a significant new tool that enhances your agent's capabilities.

### Option C: Streaming & Real-time UI

Add streaming support so users see responses as they arrive.

### Option D: Your Own Feature

Build something not listed here that you find interesting.

---

## Option A: Robustness & Production Polish

### A1: Retry Logic for API Errors

```python
# llm.py
import asyncio
from dataclasses import dataclass
from typing import Optional

@dataclass
class RetryConfig:
    max_retries: int = 3
    base_delay: float = 1.0
    max_delay: float = 30.0
    exponential_base: float = 2.0
    retryable_errors: tuple = (
        "rate_limit_exceeded",
        "overloaded",
        "timeout",
        "connection_error"
    )

class LLMClient:
    def __init__(self, api_key: str, retry_config: RetryConfig = None):
        self.api_key = api_key
        self.retry_config = retry_config or RetryConfig()

    async def chat(self, messages: list, tools: list = None) -> dict:
        """Call LLM with automatic retry on transient errors."""
        last_error = None

        for attempt in range(self.retry_config.max_retries + 1):
            try:
                return await self._make_request(messages, tools)

            except Exception as e:
                last_error = e
                error_type = self._classify_error(e)

                if error_type not in self.retry_config.retryable_errors:
                    raise  # Non-retryable error

                if attempt < self.retry_config.max_retries:
                    delay = self._calculate_delay(attempt)
                    print(f"[Retry {attempt + 1}/{self.retry_config.max_retries} "
                          f"after {delay:.1f}s: {error_type}]")
                    await asyncio.sleep(delay)

        raise last_error

    def _calculate_delay(self, attempt: int) -> float:
        """Calculate exponential backoff delay."""
        delay = self.retry_config.base_delay * (
            self.retry_config.exponential_base ** attempt
        )
        return min(delay, self.retry_config.max_delay)

    def _classify_error(self, error: Exception) -> str:
        """Classify error for retry decision."""
        error_str = str(error).lower()

        if "rate" in error_str and "limit" in error_str:
            return "rate_limit_exceeded"
        if "overload" in error_str or "capacity" in error_str:
            return "overloaded"
        if "timeout" in error_str:
            return "timeout"
        if "connection" in error_str or "network" in error_str:
            return "connection_error"

        return "unknown"
```

### A2: Tool Execution Timeouts

```python
# tools/base.py
import asyncio
from contextlib import asynccontextmanager

class ToolExecutor:
    def __init__(self, default_timeout: float = 30.0):
        self.default_timeout = default_timeout

    async def execute(
        self,
        tool: Tool,
        args: dict,
        timeout: float = None
    ) -> ToolResult:
        """Execute a tool with timeout protection."""
        timeout = timeout or tool.timeout or self.default_timeout

        try:
            result = await asyncio.wait_for(
                tool.execute(args),
                timeout=timeout
            )
            return result

        except asyncio.TimeoutError:
            return ToolResult(
                success=False,
                output=f"Tool '{tool.name}' timed out after {timeout}s",
                error="timeout"
            )
        except Exception as e:
            return ToolResult(
                success=False,
                output=f"Tool '{tool.name}' failed: {str(e)}",
                error=str(type(e).__name__)
            )
```

### A3: Helpful Error Messages

```python
# errors.py
from dataclasses import dataclass
from typing import Optional

@dataclass
class AgentError:
    """Structured error with suggestions."""
    message: str
    error_type: str
    suggestion: Optional[str] = None
    context: Optional[dict] = None

class ErrorHandler:
    """Provides helpful error messages and suggestions."""

    SUGGESTIONS = {
        "FileNotFoundError": [
            "Check if the file path is correct",
            "Use the Read tool to explore the directory first",
            "The file might be in a different location"
        ],
        "PermissionError": [
            "The file may be read-only or in a protected directory",
            "Check file permissions with 'ls -la'"
        ],
        "SyntaxError": [
            "There's a syntax error in the code",
            "Check for missing brackets, quotes, or colons",
            "Review the line number mentioned in the error"
        ],
        "ImportError": [
            "The module might not be installed",
            "Try: pip install <module_name>",
            "Check if you're in the correct virtual environment"
        ],
        "JSONDecodeError": [
            "The file doesn't contain valid JSON",
            "Check for trailing commas or missing quotes",
            "Validate the JSON at jsonlint.com"
        ]
    }

    def handle(self, error: Exception, context: dict = None) -> AgentError:
        """Convert exception to helpful AgentError."""
        error_type = type(error).__name__
        suggestions = self.SUGGESTIONS.get(error_type, [])

        # Add context-specific suggestions
        if context:
            suggestions = self._add_context_suggestions(
                error, context, suggestions
            )

        suggestion = None
        if suggestions:
            suggestion = "Suggestions:\n" + "\n".join(f"  - {s}" for s in suggestions)

        return AgentError(
            message=str(error),
            error_type=error_type,
            suggestion=suggestion,
            context=context
        )

    def _add_context_suggestions(
        self,
        error: Exception,
        context: dict,
        suggestions: list
    ) -> list:
        """Add suggestions based on context."""
        suggestions = suggestions.copy()

        # File not found - suggest similar files
        if isinstance(error, FileNotFoundError) and "path" in context:
            path = context["path"]
            similar = self._find_similar_files(path)
            if similar:
                suggestions.append(f"Did you mean: {', '.join(similar)}")

        return suggestions

    def _find_similar_files(self, path: str) -> list[str]:
        """Find files with similar names."""
        from pathlib import Path
        import difflib

        target = Path(path)
        parent = target.parent

        if not parent.exists():
            return []

        candidates = [f.name for f in parent.iterdir()]
        matches = difflib.get_close_matches(target.name, candidates, n=3)

        return [str(parent / m) for m in matches]
```

### A4: Progress Indicators

```python
# ui.py
import sys
import time
from contextlib import contextmanager

class ProgressIndicator:
    """Shows progress during long operations."""

    SPINNERS = ["⠋", "⠙", "⠹", "⠸", "⠼", "⠴", "⠦", "⠧", "⠇", "⠏"]

    def __init__(self):
        self.current = 0
        self.running = False

    @contextmanager
    def spinner(self, message: str):
        """Show a spinner while operation runs."""
        import threading

        self.running = True

        def spin():
            while self.running:
                char = self.SPINNERS[self.current % len(self.SPINNERS)]
                sys.stdout.write(f"\r{char} {message}")
                sys.stdout.flush()
                self.current += 1
                time.sleep(0.1)

        thread = threading.Thread(target=spin)
        thread.start()

        try:
            yield
        finally:
            self.running = False
            thread.join()
            sys.stdout.write(f"\r✓ {message}\n")
            sys.stdout.flush()

    def progress_bar(self, current: int, total: int, message: str = ""):
        """Show a progress bar."""
        width = 30
        filled = int(width * current / total)
        bar = "█" * filled + "░" * (width - filled)
        percent = current / total * 100

        sys.stdout.write(f"\r[{bar}] {percent:.1f}% {message}")
        sys.stdout.flush()

        if current >= total:
            sys.stdout.write("\n")
```

---

## Option B: New Tool Implementation

### B1: Grep Tool (Search File Contents)

```python
# tools/grep.py
import re
from pathlib import Path
from dataclasses import dataclass

@dataclass
class GrepMatch:
    file: str
    line_number: int
    line: str
    match_start: int
    match_end: int

class GrepTool(Tool):
    name = "grep"
    description = """Search for a pattern in files.

Use this to find code, function definitions, usages, etc.
Returns matching lines with file paths and line numbers."""

    parameters = {
        "type": "object",
        "properties": {
            "pattern": {
                "type": "string",
                "description": "Regex pattern to search for"
            },
            "path": {
                "type": "string",
                "description": "File or directory to search in"
            },
            "include": {
                "type": "string",
                "description": "Glob pattern for files to include (e.g., '*.py')"
            },
            "context": {
                "type": "integer",
                "description": "Lines of context around matches",
                "default": 0
            },
            "max_results": {
                "type": "integer",
                "description": "Maximum number of results",
                "default": 50
            }
        },
        "required": ["pattern", "path"]
    }

    async def execute(self, args: dict, ctx: ToolContext) -> ToolResult:
        pattern = args["pattern"]
        path = Path(args["path"])
        include = args.get("include", "*")
        context_lines = args.get("context", 0)
        max_results = args.get("max_results", 50)

        try:
            regex = re.compile(pattern)
        except re.error as e:
            return ToolResult(
                success=False,
                output=f"Invalid regex pattern: {e}"
            )

        matches = []

        # Get files to search
        if path.is_file():
            files = [path]
        else:
            files = list(path.rglob(include))

        for file in files:
            if not file.is_file():
                continue

            # Skip binary files
            if self._is_binary(file):
                continue

            try:
                file_matches = self._search_file(
                    file, regex, context_lines
                )
                matches.extend(file_matches)

                if len(matches) >= max_results:
                    break
            except Exception:
                continue  # Skip unreadable files

        # Format output
        output = self._format_matches(matches[:max_results])

        if not matches:
            output = f"No matches found for pattern: {pattern}"

        return ToolResult(success=True, output=output)

    def _search_file(
        self,
        file: Path,
        regex: re.Pattern,
        context: int
    ) -> list[GrepMatch]:
        """Search a single file for matches."""
        matches = []
        lines = file.read_text().splitlines()

        for i, line in enumerate(lines):
            match = regex.search(line)
            if match:
                matches.append(GrepMatch(
                    file=str(file),
                    line_number=i + 1,
                    line=line,
                    match_start=match.start(),
                    match_end=match.end()
                ))

        return matches

    def _is_binary(self, file: Path) -> bool:
        """Check if file is binary."""
        try:
            with open(file, 'rb') as f:
                chunk = f.read(1024)
                return b'\x00' in chunk
        except Exception:
            return True

    def _format_matches(self, matches: list[GrepMatch]) -> str:
        """Format matches for display."""
        lines = []
        current_file = None

        for m in matches:
            if m.file != current_file:
                current_file = m.file
                lines.append(f"\n{m.file}:")

            lines.append(f"  {m.line_number}: {m.line}")

        return "\n".join(lines)
```

### B2: Git Tool

```python
# tools/git.py
import subprocess
from typing import Optional

class GitTool(Tool):
    name = "git"
    description = """Execute git commands.

Supports: status, diff, log, add, commit, branch, checkout, stash.
Does NOT support: push, pull, fetch (use bash for these)."""

    parameters = {
        "type": "object",
        "properties": {
            "command": {
                "type": "string",
                "enum": ["status", "diff", "log", "add", "commit",
                         "branch", "checkout", "stash"],
                "description": "Git command to run"
            },
            "args": {
                "type": "array",
                "items": {"type": "string"},
                "description": "Additional arguments"
            }
        },
        "required": ["command"]
    }

    SAFE_COMMANDS = {"status", "diff", "log", "branch"}

    async def execute(self, args: dict, ctx: ToolContext) -> ToolResult:
        command = args["command"]
        extra_args = args.get("args", [])

        # Build git command
        cmd = ["git", command] + extra_args

        # Check permissions for write commands
        if command not in self.SAFE_COMMANDS:
            allowed = await ctx.check_permission(
                f"git_{command}",
                {"command": " ".join(cmd)}
            )
            if not allowed:
                return ToolResult(
                    success=False,
                    output=f"Permission denied for: {' '.join(cmd)}"
                )

        try:
            result = subprocess.run(
                cmd,
                capture_output=True,
                text=True,
                timeout=30,
                cwd=ctx.working_dir
            )

            output = result.stdout
            if result.stderr:
                output += f"\nStderr: {result.stderr}"

            return ToolResult(
                success=result.returncode == 0,
                output=output or "(no output)"
            )

        except subprocess.TimeoutExpired:
            return ToolResult(
                success=False,
                output="Git command timed out"
            )
        except Exception as e:
            return ToolResult(
                success=False,
                output=f"Git error: {e}"
            )
```

### B3: Web Fetch Tool

```python
# tools/web.py
import aiohttp
from urllib.parse import urlparse

class WebFetchTool(Tool):
    name = "web_fetch"
    description = """Fetch content from a URL.

Useful for reading documentation, API responses, or web content.
Returns the text content of the page."""

    parameters = {
        "type": "object",
        "properties": {
            "url": {
                "type": "string",
                "description": "URL to fetch"
            },
            "extract_text": {
                "type": "boolean",
                "description": "Extract text from HTML (default: true)",
                "default": True
            }
        },
        "required": ["url"]
    }

    ALLOWED_DOMAINS = None  # Set to list to restrict
    MAX_SIZE = 1_000_000   # 1MB

    async def execute(self, args: dict, ctx: ToolContext) -> ToolResult:
        url = args["url"]
        extract_text = args.get("extract_text", True)

        # Validate URL
        try:
            parsed = urlparse(url)
            if parsed.scheme not in ("http", "https"):
                return ToolResult(
                    success=False,
                    output="Only HTTP/HTTPS URLs are supported"
                )
        except Exception:
            return ToolResult(
                success=False,
                output="Invalid URL"
            )

        # Check domain allowlist
        if self.ALLOWED_DOMAINS:
            if parsed.netloc not in self.ALLOWED_DOMAINS:
                return ToolResult(
                    success=False,
                    output=f"Domain not allowed: {parsed.netloc}"
                )

        try:
            async with aiohttp.ClientSession() as session:
                async with session.get(
                    url,
                    timeout=aiohttp.ClientTimeout(total=30),
                    headers={"User-Agent": "CodingAgent/1.0"}
                ) as response:

                    if response.status != 200:
                        return ToolResult(
                            success=False,
                            output=f"HTTP {response.status}: {response.reason}"
                        )

                    # Check size
                    size = response.content_length or 0
                    if size > self.MAX_SIZE:
                        return ToolResult(
                            success=False,
                            output=f"Response too large: {size} bytes"
                        )

                    content = await response.text()

                    if extract_text and "text/html" in response.content_type:
                        content = self._extract_text(content)

                    # Truncate if needed
                    if len(content) > 50000:
                        content = content[:50000] + "\n\n[Truncated...]"

                    return ToolResult(
                        success=True,
                        output=content
                    )

        except asyncio.TimeoutError:
            return ToolResult(
                success=False,
                output="Request timed out"
            )
        except Exception as e:
            return ToolResult(
                success=False,
                output=f"Fetch error: {e}"
            )

    def _extract_text(self, html: str) -> str:
        """Extract readable text from HTML."""
        from html.parser import HTMLParser

        class TextExtractor(HTMLParser):
            def __init__(self):
                super().__init__()
                self.text = []
                self.skip_tags = {"script", "style", "nav", "footer"}
                self.skip_depth = 0

            def handle_starttag(self, tag, attrs):
                if tag in self.skip_tags:
                    self.skip_depth += 1

            def handle_endtag(self, tag):
                if tag in self.skip_tags and self.skip_depth > 0:
                    self.skip_depth -= 1

            def handle_data(self, data):
                if self.skip_depth == 0:
                    text = data.strip()
                    if text:
                        self.text.append(text)

        extractor = TextExtractor()
        extractor.feed(html)
        return "\n".join(extractor.text)
```

---

## Option C: Streaming & Real-time UI

### C1: Streaming LLM Responses

```python
# llm.py
from typing import AsyncIterator
from dataclasses import dataclass

@dataclass
class StreamChunk:
    """A chunk of streaming response."""
    type: str  # "text", "tool_call_start", "tool_call_delta", "done"
    content: str = ""
    tool_call: dict = None

class StreamingLLMClient:
    """LLM client with streaming support."""

    async def chat_stream(
        self,
        messages: list,
        tools: list = None
    ) -> AsyncIterator[StreamChunk]:
        """Stream chat completion."""

        # Example with OpenAI-style API
        async with aiohttp.ClientSession() as session:
            async with session.post(
                f"{self.base_url}/chat/completions",
                json={
                    "model": self.model,
                    "messages": messages,
                    "tools": tools,
                    "stream": True
                },
                headers={"Authorization": f"Bearer {self.api_key}"}
            ) as response:

                async for line in response.content:
                    line = line.decode().strip()

                    if not line or not line.startswith("data: "):
                        continue

                    if line == "data: [DONE]":
                        yield StreamChunk(type="done")
                        break

                    data = json.loads(line[6:])
                    delta = data["choices"][0]["delta"]

                    if "content" in delta:
                        yield StreamChunk(
                            type="text",
                            content=delta["content"]
                        )

                    if "tool_calls" in delta:
                        for tc in delta["tool_calls"]:
                            if "function" in tc:
                                yield StreamChunk(
                                    type="tool_call_delta",
                                    tool_call=tc
                                )
```

### C2: Streaming UI Handler

```python
# ui.py
import sys

class StreamingUI:
    """Handles streaming output display."""

    def __init__(self):
        self.current_tool_call = None
        self.tool_call_buffer = {}

    async def display_stream(
        self,
        stream: AsyncIterator[StreamChunk]
    ) -> tuple[str, list]:
        """Display streaming content and return final result."""
        full_text = ""
        tool_calls = []

        async for chunk in stream:
            if chunk.type == "text":
                # Print text immediately
                sys.stdout.write(chunk.content)
                sys.stdout.flush()
                full_text += chunk.content

            elif chunk.type == "tool_call_delta":
                # Accumulate tool call
                tc = chunk.tool_call
                idx = tc.get("index", 0)

                if idx not in self.tool_call_buffer:
                    self.tool_call_buffer[idx] = {
                        "id": tc.get("id", ""),
                        "function": {
                            "name": "",
                            "arguments": ""
                        }
                    }

                if "id" in tc:
                    self.tool_call_buffer[idx]["id"] = tc["id"]

                func = tc.get("function", {})
                if "name" in func:
                    self.tool_call_buffer[idx]["function"]["name"] += func["name"]
                if "arguments" in func:
                    self.tool_call_buffer[idx]["function"]["arguments"] += func["arguments"]

            elif chunk.type == "done":
                # Finalize
                if full_text:
                    sys.stdout.write("\n")

                tool_calls = list(self.tool_call_buffer.values())
                self.tool_call_buffer.clear()

        return full_text, tool_calls

    def show_tool_execution(self, tool_name: str, args: dict):
        """Show tool being executed."""
        print(f"\n🔧 Executing: {tool_name}")
        if args:
            for key, value in args.items():
                value_str = str(value)
                if len(value_str) > 50:
                    value_str = value_str[:50] + "..."
                print(f"   {key}: {value_str}")

    def show_tool_result(self, result: ToolResult):
        """Show tool result."""
        status = "✓" if result.success else "✗"
        print(f"{status} Result:")

        # Truncate long output
        output = result.output
        if len(output) > 500:
            output = output[:500] + "\n... (truncated)"

        for line in output.split("\n"):
            print(f"   {line}")
```

### C3: Integrated Streaming Agent

```python
# agent.py
class StreamingAgent:
    """Agent with streaming support."""

    def __init__(self, llm: StreamingLLMClient, tools: ToolRegistry):
        self.llm = llm
        self.tools = tools
        self.memory = Memory()
        self.ui = StreamingUI()

    async def run(self, user_input: str) -> str:
        """Run agent with streaming output."""
        self.memory.add_message(Message(role="user", content=user_input))

        while True:
            messages = self.memory.get_messages()
            tools = self.tools.get_schemas()

            # Stream the response
            stream = self.llm.chat_stream(messages, tools)
            text, tool_calls = await self.ui.display_stream(stream)

            # Save assistant message
            self.memory.add_message(Message(
                role="assistant",
                content=text,
                tool_calls=tool_calls
            ))

            if not tool_calls:
                return text

            # Execute tools
            for tc in tool_calls:
                name = tc["function"]["name"]
                args = json.loads(tc["function"]["arguments"])

                self.ui.show_tool_execution(name, args)
                result = await self.tools.execute(name, args)
                self.ui.show_tool_result(result)

                self.memory.add_message(Message(
                    role="tool",
                    content=result.output,
                    tool_call_id=tc["id"],
                    name=name
                ))
```

---

## Option D: Your Own Feature

Some ideas to inspire you:

### Subagents
- Create a `Task` tool that spawns a new agent instance
- Pass a subset of tools to the subagent
- Return the subagent's result to the parent

### Checkpoints/Undo
- Save agent state before file modifications
- Allow reverting to previous checkpoints
- Track what changed between checkpoints

### Parallel Tool Execution
- Detect independent tool calls
- Execute them concurrently
- Merge results

### IDE Integration
- Create a simple LSP client
- Show syntax errors after edits
- Auto-format code

### Voice Interface
- Add speech-to-text input
- Add text-to-speech output
- Handle interruptions

---

## Deliverables

Regardless of which option you choose:

1. **Working Implementation**
   - Clean, documented code
   - Error handling
   - Tests

2. **Documentation**
   - How it works
   - How to use it
   - Design decisions

3. **Demo**
   - Show it working
   - Edge cases handled
   - Before/after comparison

---

## Evaluation Criteria

Your capstone will be evaluated on:

1. **Functionality** (40%)
   - Does it work correctly?
   - Are edge cases handled?
   - Is it robust?

2. **Code Quality** (30%)
   - Clean, readable code
   - Appropriate abstractions
   - Good error handling

3. **Integration** (20%)
   - Works with existing agent
   - Follows established patterns
   - Doesn't break other features

4. **Documentation** (10%)
   - Clear explanation
   - Usage examples
   - Design rationale

---

## Congratulations!

You've completed the coding agent curriculum. You now understand:

- **How coding agents are architected** - The main loop, tools, permissions, memory
- **How to build your own agent** - From scratch, in Python
- **How to extend and customize agents** - Skills, commands, MCP, and more

Your mini-agent has:
- A main loop with tool dispatch
- File read/write/edit tools
- A bash tool
- A permission system
- Conversation memory with compaction
- Extensibility via skills and commands

This is the same core architecture used by production agents like Claude Code and opencode.

**What's next?**
- Polish your agent for daily use
- Add tools for your specific workflow
- Share what you've built
- Contribute to open source agents

Happy building! 🚀
