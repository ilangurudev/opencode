# Deep Dive: Provider Abstraction

> **Time**: ~20-30 minutes
> **Prerequisites**: Complete Day 3-4 core concepts

opencode works with multiple LLM providers (Anthropic, OpenAI, Google, etc.). The provider abstraction layer makes this possible without changing the core logic. This deep dive explores how it works.

---

## The Problem

Different LLM providers have different APIs:

| Provider | API Endpoint | Auth | Format |
|----------|-------------|------|---------|
| Anthropic | `messages` API | API Key | Messages array |
| OpenAI | `chat/completions` | API Key | Different schema |
| Google | Vertex AI | Service Account | Yet another schema |
| Bedrock | AWS API | AWS Creds | Different again |

Without abstraction, you'd need provider-specific code everywhere.

---

## The Provider Interface

opencode defines a common interface all providers must implement:

```typescript
// provider/provider.ts (simplified)
export namespace Provider {
  export interface Model {
    id: string                // Model identifier
    providerID: string        // Provider name
    limit: {
      context: number         // Max context tokens
      input: number           // Max input tokens
      output: number          // Max output tokens
    }
  }

  export interface Info {
    id: string                // Provider ID
    name: string              // Display name
    models: Model[]           // Available models

    // Create LLM client
    client(config: Config): Client
  }

  export interface Client {
    // Stream LLM responses
    stream(input: StreamInput): Promise<Stream>
  }

  export interface StreamInput {
    model: string
    system: string[]
    messages: Message[]
    tools: Record<string, Tool>
    maxTokens?: number
  }

  export interface Stream {
    fullStream: AsyncIterable<StreamEvent>
  }

  export type StreamEvent =
    | { type: "start" }
    | { type: "text-delta"; text: string }
    | { type: "tool-call"; toolCallId: string; name: string; input: any }
    | { type: "tool-result"; toolCallId: string; output: any }
    | { type: "finish"; reason: string; usage: TokenUsage }
    | { type: "error"; error: Error }
}
```

---

## Built-in Providers

opencode ships with several providers:

### 1. Anthropic (Claude)

```typescript
// provider/anthropic.ts
export const AnthropicProvider: Provider.Info = {
  id: "anthropic",
  name: "Anthropic",
  models: [
    {
      id: "claude-sonnet-4-5",
      providerID: "anthropic",
      limit: {
        context: 200000,
        input: 0,      // No separate limit
        output: 8192,
      }
    },
    {
      id: "claude-opus-4-5",
      providerID: "anthropic",
      limit: {
        context: 200000,
        input: 0,
        output: 16384,
      }
    },
  ],

  client(config) {
    return {
      async stream(input) {
        const client = new Anthropic({
          apiKey: config.apiKey,
        })

        // Convert to Anthropic format
        const stream = await client.messages.stream({
          model: input.model,
          system: input.system.join("\n"),
          messages: convertMessages(input.messages),
          tools: convertTools(input.tools),
          max_tokens: input.maxTokens || 8192,
        })

        // Convert stream to common format
        return {
          fullStream: convertStreamEvents(stream)
        }
      }
    }
  }
}
```

### 2. OpenAI (GPT)

```typescript
// provider/openai.ts
export const OpenAIProvider: Provider.Info = {
  id: "openai",
  name: "OpenAI",
  models: [
    {
      id: "gpt-4o",
      providerID: "openai",
      limit: {
        context: 128000,
        input: 0,
        output: 16384,
      }
    },
  ],

  client(config) {
    return {
      async stream(input) {
        const client = new OpenAI({
          apiKey: config.apiKey,
        })

        // OpenAI uses different format
        const stream = await client.chat.completions.create({
          model: input.model,
          messages: [
            { role: "system", content: input.system.join("\n") },
            ...convertMessages(input.messages),
          ],
          tools: convertToolsOpenAI(input.tools),
          stream: true,
        })

        return {
          fullStream: convertStreamEventsOpenAI(stream)
        }
      }
    }
  }
}
```

### 3. Google (Gemini)

```typescript
// provider/google.ts
export const GoogleProvider: Provider.Info = {
  id: "google",
  name: "Google",
  models: [
    {
      id: "gemini-2.0-flash-exp",
      providerID: "google",
      limit: {
        context: 1000000,  // 1M tokens!
        input: 0,
        output: 8192,
      }
    },
  ],

  client(config) {
    return {
      async stream(input) {
        // Google uses Vertex AI
        const client = createVertexClient({
          projectId: config.projectId,
          location: config.location,
        })

        const stream = await client.streamGenerateContent({
          model: input.model,
          contents: convertMessagesGoogle(input.messages),
          tools: convertToolsGoogle(input.tools),
        })

        return {
          fullStream: convertStreamEventsGoogle(stream)
        }
      }
    }
  }
}
```

---

## Provider Registry

The registry manages available providers:

```typescript
// provider/registry.ts
export namespace ProviderRegistry {
  const providers: Provider.Info[] = [
    AnthropicProvider,
    OpenAIProvider,
    GoogleProvider,
    BedrockProvider,
    // ... more providers
  ]

  export function get(id: string): Provider.Info | undefined {
    return providers.find(p => p.id === id)
  }

  export function getModel(
    providerID: string,
    modelID: string
  ): Provider.Model | undefined {
    const provider = get(providerID)
    return provider?.models.find(m => m.id === modelID)
  }

  export function all(): Provider.Info[] {
    return providers
  }
}
```

---

## Using Providers

The LLM module uses the provider abstraction:

```typescript
// session/llm.ts (simplified)
export namespace LLM {
  export async function stream(input: {
    model: Provider.Model
    system: string[]
    messages: Message[]
    tools: Record<string, Tool>
  }): Promise<Provider.Stream> {
    // 1. Get provider
    const provider = ProviderRegistry.get(input.model.providerID)
    if (!provider) {
      throw new Error(`Provider not found: ${input.model.providerID}`)
    }

    // 2. Get config
    const config = await Config.get()
    const providerConfig = config.providers[provider.id]

    // 3. Create client
    const client = provider.client(providerConfig)

    // 4. Call stream (provider-agnostic!)
    return await client.stream({
      model: input.model.id,
      system: input.system,
      messages: input.messages,
      tools: input.tools,
    })
  }
}
```

The rest of the codebase doesn't know or care which provider is being used!

---

## Message Format Conversion

Each provider requires different message formats:

### Anthropic

```typescript
// Anthropic format
{
  role: "user" | "assistant",
  content: [
    { type: "text", text: "..." },
    { type: "tool_use", id: "...", name: "...", input: {...} },
    { type: "tool_result", tool_use_id: "...", content: "..." }
  ]
}
```

### OpenAI

```typescript
// OpenAI format
{
  role: "user" | "assistant" | "tool",
  content: "...",
  tool_calls?: [
    { id: "...", type: "function", function: { name: "...", arguments: "..." } }
  ]
}
```

### Google

```typescript
// Google format
{
  role: "user" | "model",
  parts: [
    { text: "..." },
    { functionCall: { name: "...", args: {...} } },
    { functionResponse: { name: "...", response: {...} } }
  ]
}
```

The provider implementations convert between opencode's internal format and each provider's format.

---

## Tool Schema Conversion

Similar conversions for tool schemas:

### Anthropic

```typescript
{
  name: "read",
  description: "Read a file",
  input_schema: {
    type: "object",
    properties: {
      filePath: { type: "string" }
    },
    required: ["filePath"]
  }
}
```

### OpenAI

```typescript
{
  type: "function",
  function: {
    name: "read",
    description: "Read a file",
    parameters: {
      type: "object",
      properties: {
        filePath: { type: "string" }
      },
      required: ["filePath"]
    }
  }
}
```

### Google

```typescript
{
  functionDeclarations: [{
    name: "read",
    description: "Read a file",
    parameters: {
      type: "OBJECT",
      properties: {
        filePath: { type: "STRING" }
      },
      required: ["filePath"]
    }
  }]
}
```

---

## Stream Event Normalization

All providers emit the same event types:

```typescript
async function* convertStreamEvents(
  providerStream: ProviderSpecificStream
): AsyncIterable<Provider.StreamEvent> {
  for await (const event of providerStream) {
    // Convert provider-specific events to common format
    switch (event.type) {
      case "message_start":
        yield { type: "start" }
        break

      case "content_block_delta":
        if (event.delta.type === "text_delta") {
          yield {
            type: "text-delta",
            text: event.delta.text
          }
        }
        break

      case "message_delta":
        if (event.delta.stop_reason) {
          yield {
            type: "finish",
            reason: event.delta.stop_reason,
            usage: event.usage
          }
        }
        break

      // ... more event types
    }
  }
}
```

---

## Configuration

Users configure providers in `.opencode/config.json`:

```json
{
  "providers": {
    "anthropic": {
      "apiKey": "sk-ant-..."
    },
    "openai": {
      "apiKey": "sk-..."
    },
    "google": {
      "projectId": "my-project",
      "location": "us-central1"
    }
  },
  "defaultModel": {
    "providerID": "anthropic",
    "modelID": "claude-sonnet-4-5"
  }
}
```

---

## Python Equivalent

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import AsyncIterable


@dataclass
class Model:
    id: str
    provider_id: str
    context_limit: int
    output_limit: int


@dataclass
class StreamEvent:
    type: str
    data: dict


class ProviderClient(ABC):
    @abstractmethod
    async def stream(
        self,
        model: str,
        system: list[str],
        messages: list[dict],
        tools: dict,
    ) -> AsyncIterable[StreamEvent]:
        pass


class Provider(ABC):
    id: str
    name: str
    models: list[Model]

    @abstractmethod
    def create_client(self, config: dict) -> ProviderClient:
        pass


class AnthropicProvider(Provider):
    id = "anthropic"
    name = "Anthropic"
    models = [
        Model(
            id="claude-sonnet-4-5",
            provider_id="anthropic",
            context_limit=200000,
            output_limit=8192,
        )
    ]

    def create_client(self, config: dict) -> ProviderClient:
        return AnthropicClient(api_key=config["apiKey"])


class AnthropicClient(ProviderClient):
    def __init__(self, api_key: str):
        self.client = Anthropic(api_key=api_key)

    async def stream(
        self,
        model: str,
        system: list[str],
        messages: list[dict],
        tools: dict,
    ) -> AsyncIterable[StreamEvent]:
        stream = await self.client.messages.stream(
            model=model,
            system="\n".join(system),
            messages=self._convert_messages(messages),
            tools=self._convert_tools(tools),
        )

        async for event in stream:
            # Convert to common format
            if event.type == "content_block_delta":
                yield StreamEvent(
                    type="text-delta",
                    data={"text": event.delta.text}
                )
            # ... more conversions


class ProviderRegistry:
    def __init__(self):
        self.providers = {
            "anthropic": AnthropicProvider(),
            "openai": OpenAIProvider(),
            "google": GoogleProvider(),
        }

    def get(self, provider_id: str) -> Provider:
        return self.providers.get(provider_id)

    def get_model(self, provider_id: str, model_id: str) -> Model:
        provider = self.get(provider_id)
        return next(m for m in provider.models if m.id == model_id)
```

---

## Summary

The provider abstraction:
- **Defines a common interface** all providers implement
- **Hides provider differences** from the rest of the codebase
- **Converts formats** (messages, tools, events) automatically
- **Enables multi-provider support** without code duplication
- **Makes adding providers easy** - just implement the interface

This is a classic example of the **adapter pattern** in software design.

---

## Related Deep Dives

- [Main Loop Internals](/deep-dives/main-loop-internals.md) - How providers are used
- [Compaction](/deep-dives/compaction.md) - Token tracking across providers
- [Error Handling](/deep-dives/error-handling.md) - Provider-specific error handling
