# PRD2: Chartsmith Vercel AI SDK Migration
## Technical Specification Document

**Version**: 1.0  
**PR**: PR1 of 2  
**Timeline**: Days 1-3 (3 days)  
**Status**: Ready for Implementation

---

## Technical Overview

### Objective
Replace Chartsmith's custom chat implementation with Vercel AI SDK, maintaining all existing functionality while enabling multi-provider support.

### Approach
- Incremental migration preserving backward compatibility
- Minimal Go backend changes (deferred to PR2)
- OpenRouter as unified provider gateway
- SDK hooks replace custom state management

---

## System Architecture

### Current Architecture
```
┌─────────────────┐     Custom API      ┌─────────────────┐
│   React Chat    │ ──────────────────► │  Next.js API    │
│   Components    │                     │    Routes       │
│                 │ ◄────────────────── │                 │
│  Custom State   │   Custom Stream     │  @anthropic-ai  │
└─────────────────┘                     └────────┬────────┘
                                                 │
                                                 ▼
                                        ┌─────────────────┐
                                        │   Anthropic     │
                                        │      API        │
                                        └─────────────────┘
```

### Target Architecture (PR1)
```
┌─────────────────┐     useChat Hook    ┌─────────────────┐
│   React Chat    │ ──────────────────► │  Next.js API    │
│   Components    │                     │    Routes       │
│                 │ ◄────────────────── │                 │
│  @ai-sdk/react  │   Data Stream       │  AI SDK Core    │
└─────────────────┘                     └────────┬────────┘
                                                 │
                                                 ▼
                                        ┌─────────────────┐
                                        │   OpenRouter    │
                                        │  (Multi-Model)  │
                                        └─────────────────┘
```

---

## Tech Stack

### New Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `ai` | `^4.0.0` | AI SDK Core (streamText, tool) |
| `@ai-sdk/react` | `^1.0.0` | React hooks (useChat) |
| `@openrouter/ai-sdk-provider` | `^0.4.0` | OpenRouter integration |
| `zod` | `^3.22.0` | Schema validation (if not present) |

### Removed Dependencies
| Package | Reason |
|---------|--------|
| `@anthropic-ai/sdk` | Replaced by AI SDK + OpenRouter |

### Retained Dependencies
- All existing Next.js dependencies
- All existing React dependencies
- All existing UI component libraries

---

## Environment Configuration

### New Environment Variables

```env
# Required: OpenRouter API Key
OPENROUTER_API_KEY=sk-or-v1-xxxxx

# Optional: Direct OpenAI fallback
OPENAI_API_KEY=sk-xxxxx

# Configuration
DEFAULT_AI_PROVIDER=openai
DEFAULT_AI_MODEL=openai/gpt-4o
```

### Environment Variable Validation

On application startup, validate:
1. `OPENROUTER_API_KEY` exists and is non-empty
2. Log warning if `OPENAI_API_KEY` missing (fallback unavailable)
3. Set defaults for optional config variables

---

## File Structure Changes

### New Files to Create

```
chartsmith-app/
├── src/
│   ├── lib/
│   │   ├── ai/
│   │   │   ├── provider.ts        # Provider factory
│   │   │   ├── models.ts          # Model definitions
│   │   │   └── config.ts          # AI configuration
│   │   └── ...
│   ├── components/
│   │   ├── chat/
│   │   │   ├── ProviderSelector.tsx  # New component
│   │   │   └── ...
│   │   └── ...
│   └── app/
│       └── api/
│           └── chat/
│               └── route.ts       # New/Modified API route
└── ...
```

### Files to Modify

| File | Changes |
|------|---------|
| `package.json` | Add new dependencies, remove old |
| `components/Chat.tsx` (or equivalent) | Replace with useChat hook |
| `components/Message.tsx` (or equivalent) | Update for parts-based rendering |
| `chartsmith-app/ARCHITECTURE.md` | Document new AI SDK integration |
| Existing test files | Update mocks and assertions |

### Files to Remove/Deprecate
- Custom streaming implementation files
- Direct Anthropic SDK integration code
- Custom message state management hooks

---

## Component Specifications

### Provider Factory Module

**File**: `src/lib/ai/provider.ts`

**Purpose**: Abstract provider selection, return configured model instances

**Exports**:
- `getModel(provider, modelId?)` - Returns AI SDK model instance
- `AVAILABLE_PROVIDERS` - List of supported providers
- `DEFAULT_PROVIDER` - Default provider configuration

**Logic**:
```
FUNCTION getModel(provider: string, modelId?: string):
  CREATE openrouter instance with API key from env
  
  DEFINE model mappings:
    "openai" -> openrouter("openai/gpt-4o") OR custom modelId
    "anthropic" -> openrouter("anthropic/claude-4.5-sonnet") OR custom modelId
  
  IF provider not in mappings:
    THROW InvalidProviderError
  
  RETURN mapped model instance
```

**Type Definitions**:
```
Provider type = "openai" | "anthropic"

ProviderConfig = {
  id: Provider
  name: string (display name)
  model: string (full model ID)
  description?: string
}

AVAILABLE_PROVIDERS: ProviderConfig[]
```

### Provider Selector Component

**File**: `src/components/chat/ProviderSelector.tsx`

**Purpose**: Allow user to select AI provider before conversation starts

**Props**:
- `selectedProvider: Provider` - Current selection
- `onProviderChange: (provider: Provider) => void` - Selection handler
- `disabled?: boolean` - Disable selector (when conversation started)

**State**: None (controlled component)

**Render Logic**:
```
IF disabled:
  RENDER selected provider as text badge only
ELSE:
  RENDER dropdown with AVAILABLE_PROVIDERS
  ON selection change, call onProviderChange
```

**Styling**: Match existing Chartsmith component patterns

### Chat Component Migration

**File**: Modify existing chat component

**Before (Pseudocode)**:
```
COMPONENT Chat:
  STATE messages = []
  STATE input = ""
  STATE isLoading = false
  STATE streamingContent = ""
  
  FUNCTION handleSubmit:
    SET isLoading = true
    CALL custom API with messages
    MANUALLY handle streaming response
    UPDATE messages with response
    SET isLoading = false
```

**After (Pseudocode)**:
```
COMPONENT Chat:
  STATE selectedProvider = DEFAULT_PROVIDER
  
  USE useChat hook with:
    api: "/api/chat"
    body: { provider: selectedProvider.id, model: selectedProvider.model }
  
  DESTRUCTURE from useChat:
    messages, input, handleInputChange, handleSubmit,
    status, stop, reload, error
  
  RENDER:
    IF messages.length === 0:
      RENDER ProviderSelector(selectedProvider, setSelectedProvider)
    
    FOR each message in messages:
      RENDER MessageComponent(message)
    
    RENDER ChatInput(input, handleInputChange, handleSubmit, status)
    
    IF status === "streaming":
      RENDER StopButton(stop)
    
    IF error:
      RENDER ErrorDisplay(error, reload)
```

### Message Component Migration

**File**: Modify existing message component

**Before**: Renders message.content directly

**After**: Renders message.parts array

**Render Logic**:
```
COMPONENT Message(message):
  FOR each part in message.parts:
    SWITCH part.type:
      CASE "text":
        RENDER TextContent(part.text)
      
      CASE "tool-call":
        RENDER ToolCallIndicator(part.toolName, part.args)
      
      CASE "tool-result":
        RENDER ToolResult(part.toolName, part.result)
      
      DEFAULT:
        LOG warning for unknown part type
```

---

## API Route Specification

### Chat Route Handler

**File**: `src/app/api/chat/route.ts`

**Method**: POST

**Request Processing**:
```
FUNCTION POST(request):
  TRY:
    PARSE JSON body: { messages, provider, model }
    
    VALIDATE provider is in AVAILABLE_PROVIDERS
    
    GET model instance via getModel(provider, model)
    
    GET system prompt from existing configuration
    
    CALL streamText with:
      model: model instance
      system: system prompt
      messages: convertToModelMessages(messages)
      tools: existing tool definitions (if any)
    
    RETURN result.toDataStreamResponse()
    
  CATCH error:
    LOG error details
    RETURN JSON response with error message, status 500
```

**Response Format**: Vercel AI SDK Data Stream

**Error Handling**:
- Invalid provider: 400 Bad Request
- Missing API key: 500 Internal Server Error
- Provider error: 502 Bad Gateway
- Network error: 503 Service Unavailable

### Existing Tool Migration

**Current tools must be converted to AI SDK format**

**Tool Definition Pattern**:
```
DEFINE tool using AI SDK tool() helper:
  description: string explaining tool purpose
  inputSchema: Zod schema for parameters
  execute: async function implementing tool logic

EXAMPLE structure:
  chartGenerationTool:
    description: "Generate Helm chart based on requirements"
    inputSchema: z.object({
      chartName: z.string(),
      requirements: z.string()
    })
    execute: async (params) => existing chart generation logic
```

**Tool Registration**:
```
DEFINE chartsmithTools object containing all tools

PASS chartsmithTools to streamText() call in tools parameter
```

---

## State Management Migration

### Current Custom State
- `messages[]` - Chat history
- `input` - Current input value
- `isLoading` - Loading state
- `error` - Error state
- `streamingContent` - Partial response during stream

### SDK-Managed State (useChat)
| State | Source | Access |
|-------|--------|--------|
| `messages` | useChat | Direct |
| `input` | useChat | Direct |
| `status` | useChat | 'ready' / 'submitted' / 'streaming' / 'error' |
| `error` | useChat | Error object or undefined |

### Migration Mapping
| Old State | New Access |
|-----------|------------|
| `messages` | `messages` from useChat |
| `input` | `input` from useChat |
| `isLoading` | `status === 'submitted' \|\| status === 'streaming'` |
| `error` | `error` from useChat |
| `streamingContent` | Last message content (auto-updated) |

### Custom State Still Needed
- `selectedProvider` - Provider selection (component-level)
- Any UI-specific state (modals, panels, etc.)

---

## Streaming Implementation

### SDK Streaming Configuration
```
useChat configuration:
  api: "/api/chat"
  
  body: {
    provider: selectedProvider.id,
    model: selectedProvider.model
  }
  
  onFinish: (message) => {
    // Optional: Analytics, logging
  }
  
  onError: (error) => {
    // Error tracking
  }
  
  experimental_throttle: 50  // Optional: Throttle UI updates
```

### Server-Side Streaming
```
streamText configuration:
  model: provider model instance
  messages: converted messages
  system: system prompt
  
  // Tools if needed
  tools: chartsmithTools
  
  // Return streaming response
  RETURN result.toDataStreamResponse()
```

---

## Testing Strategy

### Unit Test Updates

**Provider Factory Tests**:
```
TEST "getModel returns OpenAI model for openai provider"
TEST "getModel returns Claude model for anthropic provider"
TEST "getModel throws for invalid provider"
TEST "getModel uses custom model ID when provided"
```

**Component Tests**:
```
TEST "ProviderSelector renders all available providers"
TEST "ProviderSelector calls onChange when selection changes"
TEST "ProviderSelector is disabled when disabled prop is true"
TEST "Chat component renders messages correctly"
TEST "Chat component shows provider selector when no messages"
TEST "Chat component hides provider selector after first message"
```

### Integration Test Updates

**API Route Tests**:
```
TEST "POST /api/chat returns streaming response"
TEST "POST /api/chat handles invalid provider"
TEST "POST /api/chat includes system prompt"
TEST "POST /api/chat processes tools correctly"
```

**Mock Configuration**:
- Mock OpenRouter responses for deterministic testing
- Use AI SDK test utilities if available
- Preserve existing mock patterns where possible

### Existing Test Compatibility

**Identify and update**:
1. Tests importing removed dependencies
2. Tests mocking Anthropic SDK directly
3. Tests asserting on old message format
4. Tests checking custom streaming behavior

---

## Migration Sequence

### Day 1: Foundation

**Morning**:
1. Create feature branch from main
2. Install new dependencies
3. Add environment variables to `.env.example`
4. Create provider factory module
5. Create model configuration

**Afternoon**:
1. Create API route with streamText
2. Test basic streaming response
3. Verify OpenRouter connectivity

### Day 2: Component Migration

**Morning**:
1. Create ProviderSelector component
2. Migrate Chat component to useChat
3. Update message rendering for parts

**Afternoon**:
1. Migrate existing tools to SDK format
2. Test tool calling functionality
3. Verify file context features work

### Day 3: Polish & Testing

**Morning**:
1. Update/fix failing tests
2. Add new unit tests
3. Run full test suite

**Afternoon**:
1. Update ARCHITECTURE.md
2. Code cleanup and review prep
3. Final testing and bug fixes

---

## Error Handling Patterns

### Client-Side Errors
```
useChat provides error object

RENDER ErrorDisplay component when error exists:
  SHOW user-friendly message based on error type
  PROVIDE retry button calling reload()
  PROVIDE option to start new conversation
```

### Server-Side Errors
```
API route error handling:

CATCH specific error types:
  ValidationError -> 400 with details
  AuthenticationError -> 401 with message
  ProviderError -> 502 with provider details
  NetworkError -> 503 with retry suggestion
  UnknownError -> 500 with generic message

ALWAYS log full error for debugging
NEVER expose API keys or sensitive data in response
```

### Provider Failover (Optional Enhancement)
```
IF OpenRouter request fails:
  IF OPENAI_API_KEY exists AND provider was openai:
    RETRY with direct OpenAI SDK
  ELSE:
    THROW original error
```

---

## Performance Considerations

### Bundle Size
- AI SDK packages add ~40KB gzipped
- Offset by removing Anthropic SDK (~25KB)
- Net increase: ~15KB acceptable

### Streaming Optimization
```
useChat options for performance:
  experimental_throttle: 50-100ms
  // Reduces re-renders during fast streaming
```

### Memory Management
- useChat handles message array efficiently
- No memory leaks from manual streaming
- Garbage collection for completed streams

---

## Security Considerations

### API Key Protection
- All API keys server-side only
- Never import provider packages in client components
- Environment variables not exposed to client bundle

### Request Validation
```
API route validation:
  VALIDATE messages array exists and is array
  VALIDATE provider is in allowed list
  VALIDATE model matches provider
  SANITIZE any user content if needed
```

### Rate Limiting
- Preserve existing rate limiting if present
- OpenRouter has its own rate limits
- Consider adding request throttling if absent

---

## Rollback Plan

### If Migration Fails
1. Revert feature branch changes
2. Restore original dependencies
3. Original code remains functional

### Partial Rollback
- Provider factory can be disabled (hardcode single provider)
- useChat can be replaced incrementally
- API route can fall back to original logic

---

## Documentation Updates

### ARCHITECTURE.md Updates
- Document AI SDK integration
- Explain provider factory pattern
- Update component diagrams
- Add environment variable documentation

### Code Comments
- Document provider factory usage
- Explain useChat hook configuration
- Note any workarounds or SDK limitations

---

## Validation Checklist

### Pre-PR Checklist
- [ ] All new dependencies added to package.json
- [ ] Environment variables documented
- [ ] Provider factory created and tested
- [ ] API route migrated and working
- [ ] Chat component using useChat hook
- [ ] Message rendering uses parts
- [ ] Provider selector functional
- [ ] All existing features working
- [ ] Tests passing
- [ ] ARCHITECTURE.md updated
- [ ] No TypeScript errors
- [ ] No console errors in browser
- [ ] Streaming performance acceptable

---

## Appendix: SDK Reference

### useChat Hook Returns
```
{
  messages: UIMessage[]
  input: string
  handleInputChange: (e) => void
  handleSubmit: (e) => void
  status: 'ready' | 'submitted' | 'streaming' | 'error'
  error: Error | undefined
  stop: () => void
  reload: () => void
  setMessages: (messages) => void
}
```

### streamText Options
```
{
  model: LanguageModel
  messages: ModelMessage[]
  system?: string
  tools?: ToolSet
  maxTokens?: number
  temperature?: number
}
```

### OpenRouter Model Format
```
openrouter("provider/model-name")

Examples:
  openrouter("openai/gpt-4o")
  openrouter("anthropic/claude-4.5-sonnet")
  openrouter("google/gemini-pro")
```

---

*Document End - PRD2: PR1 Technical Specification*
