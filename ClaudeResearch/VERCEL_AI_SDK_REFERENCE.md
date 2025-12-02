# Vercel AI SDK Migration Reference

## Overview

This document provides a reference for migrating Chartsmith from direct `@anthropic-ai/sdk` usage to the Vercel AI SDK. It covers the SDK architecture, key APIs, and migration patterns.

---

## Vercel AI SDK Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      VERCEL AI SDK                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────┐     ┌─────────────────────────────┐   │
│  │    AI SDK Core      │     │       AI SDK UI             │   │
│  │                     │     │                             │   │
│  │  • generateText()   │     │  • useChat() hook           │   │
│  │  • streamText()     │     │  • useCompletion() hook     │   │
│  │  • generateObject() │     │  • useObject() hook         │   │
│  │  • tool()           │     │  • Message handling         │   │
│  │  • Provider API     │     │  • Streaming state          │   │
│  └─────────────────────┘     └─────────────────────────────┘   │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    Providers                             │   │
│  │  @ai-sdk/openai  @ai-sdk/anthropic  @openrouter/ai-sdk  │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Package Installation

### Core Packages
```bash
# AI SDK Core
npm install ai

# AI SDK React hooks
npm install @ai-sdk/react

# Provider packages
npm install @ai-sdk/openai        # OpenAI models
npm install @ai-sdk/anthropic     # Anthropic models (optional)
npm install @openrouter/ai-sdk-provider  # OpenRouter (multi-model)
```

### Package Versions
```json
{
  "dependencies": {
    "ai": "^4.0.0",
    "@ai-sdk/react": "^1.0.0",
    "@ai-sdk/openai": "^1.0.0",
    "@openrouter/ai-sdk-provider": "^0.4.0"
  }
}
```

---

## Provider Configuration

### OpenAI Provider
```typescript
import { openai } from '@ai-sdk/openai';

const model = openai('gpt-4o');
```

### OpenRouter Provider (Recommended)
```typescript
import { createOpenRouter } from '@openrouter/ai-sdk-provider';

// Create provider instance
const openrouter = createOpenRouter({
  apiKey: process.env.OPENROUTER_API_KEY,
});

// Access any model
const claudeModel = openrouter('anthropic/claude-3.5-sonnet');
const gpt4Model = openrouter('openai/gpt-4o');
const geminiModel = openrouter('google/gemini-pro');
```

### Provider Factory Pattern (For Switching)
```typescript
// lib/ai-provider.ts
import { createOpenRouter } from '@openrouter/ai-sdk-provider';
import { openai } from '@ai-sdk/openai';

type ProviderType = 'openai' | 'anthropic' | 'google';

export function getModel(provider: ProviderType, modelId?: string) {
  const openrouter = createOpenRouter({
    apiKey: process.env.OPENROUTER_API_KEY,
  });

  const models = {
    openai: openrouter(modelId || 'openai/gpt-4o'),
    anthropic: openrouter(modelId || 'anthropic/claude-3.5-sonnet'),
    google: openrouter(modelId || 'google/gemini-pro'),
  };

  return models[provider];
}
```

---

## Core API: streamText

### Basic Usage
```typescript
import { streamText } from 'ai';
import { openrouter } from '@/lib/ai-provider';

const result = await streamText({
  model: openrouter('anthropic/claude-3.5-sonnet'),
  messages: [
    { role: 'system', content: 'You are a Helm chart expert.' },
    { role: 'user', content: 'Create a deployment template.' },
  ],
});

// Return streaming response
return result.toDataStreamResponse();
```

### With Tools
```typescript
import { streamText, tool } from 'ai';
import { z } from 'zod';

const result = await streamText({
  model: openrouter('anthropic/claude-3.5-sonnet'),
  messages,
  tools: {
    validateChart: tool({
      description: 'Validate a Helm chart',
      inputSchema: z.object({
        chartPath: z.string(),
      }),
      execute: async ({ chartPath }) => {
        // Call Go backend validation
        const res = await fetch('http://localhost:8080/api/validate', {
          method: 'POST',
          body: JSON.stringify({ chartPath }),
        });
        return res.json();
      },
    }),
  },
});
```

### Multi-Step Tool Calls
```typescript
import { streamText, stepCountIs } from 'ai';

const result = await streamText({
  model,
  messages,
  tools: { validateChart, lintChart, fixIssues },
  stopWhen: stepCountIs(5), // Allow up to 5 steps
});
```

---

## UI Hook: useChat

### Basic Implementation
```typescript
// components/Chat.tsx
'use client';

import { useChat } from '@ai-sdk/react';

export function Chat() {
  const { messages, input, handleInputChange, handleSubmit, status } = useChat({
    api: '/api/chat',
  });

  return (
    <div>
      {messages.map((message) => (
        <div key={message.id}>
          <strong>{message.role}:</strong>
          {message.parts?.map((part, i) => 
            part.type === 'text' && <span key={i}>{part.text}</span>
          )}
        </div>
      ))}
      
      <form onSubmit={handleSubmit}>
        <input
          value={input}
          onChange={handleInputChange}
          placeholder="Ask about Helm charts..."
        />
        <button type="submit" disabled={status === 'streaming'}>
          Send
        </button>
      </form>
    </div>
  );
}
```

### With Tool Results Rendering
```typescript
import { useChat } from '@ai-sdk/react';

export function Chat() {
  const { messages } = useChat({ api: '/api/chat' });

  return (
    <div>
      {messages.map((message) => (
        <div key={message.id}>
          {message.parts?.map((part, i) => {
            if (part.type === 'text') {
              return <p key={i}>{part.text}</p>;
            }
            if (part.type === 'tool-validateChart') {
              return <ValidationResults key={i} data={part.result} />;
            }
            return null;
          })}
        </div>
      ))}
    </div>
  );
}
```

### Status States
```typescript
const { status } = useChat();

// status values:
// - 'ready': Idle, ready for input
// - 'submitted': Request sent, awaiting response
// - 'streaming': Response actively streaming
// - 'error': Request failed
```

---

## API Route Pattern

### Next.js Route Handler
```typescript
// app/api/chat/route.ts
import { streamText, convertToModelMessages } from 'ai';
import { getModel } from '@/lib/ai-provider';

export const maxDuration = 30;

export async function POST(req: Request) {
  const { messages, provider = 'anthropic' } = await req.json();

  const result = await streamText({
    model: getModel(provider),
    messages: convertToModelMessages(messages),
    system: `You are a Helm chart expert assistant...`,
  });

  return result.toDataStreamResponse();
}
```

### With Error Handling
```typescript
export async function POST(req: Request) {
  try {
    const { messages, provider } = await req.json();

    const result = await streamText({
      model: getModel(provider),
      messages: convertToModelMessages(messages),
    });

    return result.toDataStreamResponse();
  } catch (error) {
    console.error('Chat API error:', error);
    return new Response(
      JSON.stringify({ 
        error: 'Failed to process request',
        details: error instanceof Error ? error.message : 'Unknown error'
      }),
      { status: 500, headers: { 'Content-Type': 'application/json' } }
    );
  }
}
```

---

## Tool Calling Patterns

### Define Tools
```typescript
// lib/tools.ts
import { tool } from 'ai';
import { z } from 'zod';

export const chartTools = {
  validateChart: tool({
    description: 'Run validation checks on a Helm chart',
    inputSchema: z.object({
      chartPath: z.string().describe('Path to the chart'),
      values: z.record(z.any()).optional(),
      strictMode: z.boolean().optional(),
    }),
    execute: async (input) => {
      const response = await fetch(
        `${process.env.GO_BACKEND_URL}/api/validate`,
        {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(input),
        }
      );
      return response.json();
    },
  }),

  lintChart: tool({
    description: 'Run helm lint on the chart',
    inputSchema: z.object({
      chartPath: z.string(),
    }),
    execute: async ({ chartPath }) => {
      const response = await fetch(
        `${process.env.GO_BACKEND_URL}/api/lint`,
        {
          method: 'POST',
          body: JSON.stringify({ chartPath }),
        }
      );
      return response.json();
    },
  }),
};
```

### Use in Route
```typescript
import { streamText, stepCountIs } from 'ai';
import { chartTools } from '@/lib/tools';

export async function POST(req: Request) {
  const { messages } = await req.json();

  const result = await streamText({
    model: getModel('anthropic'),
    messages,
    tools: chartTools,
    stopWhen: stepCountIs(3),
  });

  return result.toDataStreamResponse();
}
```

---

## Streaming Optimizations

### Throttling (Performance)
```typescript
const { messages } = useChat({
  experimental_throttle: 50, // Update UI every 50ms max
});
```

### Response Callbacks
```typescript
const { messages } = useChat({
  onFinish: (message) => {
    console.log('Response complete:', message);
    // Save to history, analytics, etc.
  },
  onError: (error) => {
    console.error('Stream error:', error);
  },
});
```

---

## State Management Simplification

### Before (Custom Implementation)
```typescript
// Complex custom state
const [messages, setMessages] = useState([]);
const [input, setInput] = useState('');
const [isLoading, setIsLoading] = useState(false);
const [error, setError] = useState(null);
const [streamingContent, setStreamingContent] = useState('');

// Manual streaming handling
const handleSubmit = async () => {
  setIsLoading(true);
  setError(null);
  // ... complex streaming logic
};
```

### After (AI SDK)
```typescript
// All state managed by hook
const { 
  messages,      // Full message history
  input,         // Current input value
  status,        // 'ready' | 'submitted' | 'streaming' | 'error'
  error,         // Error object if failed
  handleSubmit,  // Form submit handler
  handleInputChange, // Input change handler
  stop,          // Stop streaming
  reload,        // Regenerate last response
} = useChat();
```

---

## Provider Switching (PR1 Pattern)

### Per-Conversation Selector
```typescript
// components/ChatWithProvider.tsx
'use client';

import { useState } from 'react';
import { useChat } from '@ai-sdk/react';

const PROVIDERS = [
  { id: 'anthropic', name: 'Claude', model: 'anthropic/claude-3.5-sonnet' },
  { id: 'openai', name: 'GPT-4', model: 'openai/gpt-4o' },
  { id: 'google', name: 'Gemini', model: 'google/gemini-pro' },
];

export function ChatWithProvider() {
  const [selectedProvider, setSelectedProvider] = useState(PROVIDERS[0]);
  
  const { messages, handleSubmit, input, handleInputChange } = useChat({
    api: '/api/chat',
    body: {
      provider: selectedProvider.id,
      model: selectedProvider.model,
    },
  });

  return (
    <div>
      {/* Provider selector - shown at conversation start */}
      {messages.length === 0 && (
        <select
          value={selectedProvider.id}
          onChange={(e) => setSelectedProvider(
            PROVIDERS.find(p => p.id === e.target.value)!
          )}
        >
          {PROVIDERS.map((p) => (
            <option key={p.id} value={p.id}>{p.name}</option>
          ))}
        </select>
      )}
      
      {/* Chat interface */}
      {/* ... */}
    </div>
  );
}
```

---

## Migration Checklist

### Frontend
- [ ] Install `ai` and `@ai-sdk/react`
- [ ] Install provider packages (`@openrouter/ai-sdk-provider`)
- [ ] Replace custom chat state with `useChat` hook
- [ ] Replace custom message rendering with parts-based rendering
- [ ] Remove manual streaming logic
- [ ] Add provider selection UI
- [ ] Update error handling to use SDK patterns

### API Routes
- [ ] Create `/api/chat/route.ts` using `streamText`
- [ ] Configure model providers with env vars
- [ ] Implement tool definitions for chart operations
- [ ] Return responses via `toDataStreamResponse()`
- [ ] Add provider switching logic based on request body

### Remove
- [ ] Custom streaming implementations
- [ ] Direct `@anthropic-ai/sdk` imports (frontend)
- [ ] Manual message state management
- [ ] Custom message type definitions (use SDK types)

---

## Environment Variables

```env
# OpenRouter (primary - accesses all models)
OPENROUTER_API_KEY=sk-or-...

# OpenAI (fallback/direct)
OPENAI_API_KEY=sk-...

# Default provider
DEFAULT_AI_PROVIDER=anthropic
DEFAULT_AI_MODEL=anthropic/claude-3.5-sonnet
```

---

## References

- [AI SDK Documentation](https://ai-sdk.dev/docs/introduction)
- [AI SDK useChat Reference](https://ai-sdk.dev/docs/reference/ai-sdk-ui/use-chat)
- [AI SDK Tool Calling](https://ai-sdk.dev/docs/ai-sdk-core/tools-and-tool-calling)
- [OpenRouter AI SDK Provider](https://github.com/OpenRouterTeam/ai-sdk-provider)
- [AI SDK Streaming](https://ai-sdk.dev/docs/ai-sdk-ui/chatbot)
