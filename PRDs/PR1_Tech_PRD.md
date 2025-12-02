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

### Current Architecture (Actual Chartsmith Codebase)

**IMPORTANT**: The current Chartsmith architecture uses a **database-driven pattern**, NOT direct HTTP between frontend and backend. Understanding this is critical for PR1 implementation.

```
┌─────────────────┐                              ┌─────────────────┐
│   React Chat    │      Server Actions          │  Next.js API    │
│   Components    │ ────────────────────────────►│    Routes       │
│                 │                              │                 │
│  Jotai Atoms    │                              │  Workspace APIs │
│  (Custom State) │                              └────────┬────────┘
└────────┬────────┘                                       │
         │                                                │ INSERT + pg_notify()
         │ WebSocket                                      │
         │ (Centrifugo)                                   ▼
         │                                       ┌─────────────────┐
         │◄──────────────────────────────────────│   PostgreSQL    │
         │        Real-time updates              │   work_queue    │
                                                 │   LISTEN/NOTIFY │
                                                 └────────┬────────┘
                                                          │ LISTEN
                                                 ┌────────▼────────┐
                                                 │   Go Backend    │
                                                 │   (Workers)     │
                                                 │                 │
                                                 │  @anthropic-sdk │
                                                 │  (All LLM calls)│
                                                 └─────────────────┘
```

**Key Current State Facts** (verified via codebase analysis):
- **No `/api/chat` route exists** - messages go through `/api/workspace/[id]/message`
- **Go backend has NO HTTP server** - all communication via PostgreSQL LISTEN/NOTIFY
- **All LLM calls happen in Go backend** - frontend only does prompt classification (`lib/llm/prompt-type.ts`)
- **State managed via Jotai atoms** - `atoms/workspace.ts` contains `messagesAtom`, `plansAtom`, etc.
- **Streaming via Centrifugo WebSocket** - not standard HTTP/SSE
- **3 existing tools in Go** - `text_editor`, `latest_subchart_version`, `latest_kubernetes_version` (Anthropic-native format)

### Target Architecture (PR1) - NEW Parallel Chat System

PR1 creates a **NEW parallel chat system** alongside the existing Go-based flow. This provides AI SDK benefits for new chat experiences without disrupting existing workspace functionality.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           AFTER PR1                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────┐     useChat Hook    ┌─────────────────┐                │
│  │   NEW Chat UI   │ ───────────────────►│  NEW /api/chat  │ ──► OpenRouter │
│  │  (AI SDK-based) │ ◄───────────────────│   (streamText)  │                │
│  │  @ai-sdk/react  │   Data Stream       │   AI SDK Core   │                │
│  └─────────────────┘                     └─────────────────┘                │
│                                                                              │
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─                │
│  EXISTING SYSTEM (UNCHANGED - Go backend continues to work)                 │
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─                │
│                                                                              │
│  ┌─────────────────┐                     ┌─────────────────┐                │
│  │ Existing Chat   │  Server Actions     │  Workspace APIs │                │
│  │ (Jotai-based)   │ ───────────────────►│  /api/workspace │                │
│  └────────┬────────┘                     └────────┬────────┘                │
│           │ Centrifugo                            │ pg_notify               │
│           │◄──────────────────────────────────────┤                         │
│                                          ┌────────▼────────┐                │
│                                          │   Go Backend    │                │
│                                          │   (Unchanged)   │                │
│                                          └─────────────────┘                │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**What PR1 Creates (NEW)**:
- `/api/chat/route.ts` - New Next.js API route using `streamText`
- `lib/ai/provider.ts` - Provider factory for OpenRouter
- `components/chat/ProviderSelector.tsx` - Model selection UI
- Chat components using `useChat` hook from `@ai-sdk/react`

**What PR1 Preserves (EXISTING)**:
- All Go backend functionality (unchanged)
- Existing workspace APIs and Jotai state management
- Centrifugo real-time updates
- Existing tool implementations in Go

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
| `@anthropic-ai/sdk` | Replaced by AI SDK + OpenRouter. Frontend usage was orphaned code (never called). |

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

### Environment Variable Notes

**Regarding `ANTHROPIC_API_KEY`**:
- Currently in `.env` for both frontend and Go backend
- Frontend usage (`lib/llm/prompt-type.ts`) is orphaned code being deleted in PR1
- After PR1, frontend no longer needs `ANTHROPIC_API_KEY`
- Go backend continues to use `ANTHROPIC_API_KEY` (unchanged)

**New Variables (Frontend Only)**:
- `OPENROUTER_API_KEY` - Required for new AI SDK chat
- `OPENAI_API_KEY` - Optional fallback for direct OpenAI access
- `DEFAULT_AI_PROVIDER` - Optional, defaults to "openai"
- `DEFAULT_AI_MODEL` - Optional, defaults to "openai/gpt-4o"

---

## File Structure Changes

### New Files to Create

```
chartsmith-app/
├── lib/
│   ├── ai/
│   │   ├── provider.ts        # Provider factory
│   │   ├── models.ts          # Model definitions
│   │   └── config.ts          # AI configuration
│   └── ...
├── components/
│   ├── chat/
│   │   ├── ProviderSelector.tsx  # New component
│   │   └── ...
│   └── ...
└── app/
    └── api/
        └── chat/
            └── route.ts       # New API route
```

**Note**: The chartsmith-app does NOT use a `src/` directory. All paths are relative to `chartsmith-app/`.

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

### Files to Delete (Orphaned Code)

| File | Reason |
|------|--------|
| `lib/llm/prompt-type.ts` | Orphaned - `promptType()` function not imported or used anywhere in codebase |

**Note**: The `promptType()` function in this file was intended for prompt classification but is never called. The actual intent classification happens in the Go backend via the `new_intent` queue handler. This simplifies PR1 - we're deleting dead code, not migrating active functionality.

---

## Component Specifications

### Provider Factory Module

**File**: `lib/ai/provider.ts`

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
    "anthropic" -> openrouter("anthropic/claude-3.5-sonnet") OR custom modelId
  
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

**File**: `components/chat/ProviderSelector.tsx`

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

**File**: `app/api/chat/route.ts`

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

### Tool Strategy for PR1

**PR1 Scope**: The NEW `/api/chat` route will be **conversational only** (no tools).

**Rationale**:
- Existing Go tools (`text_editor`, `latest_subchart_version`, `latest_kubernetes_version`) remain in Go backend
- These tools require file system access and complex logic that should stay in Go
- Full tool migration is handled in **PR1.5** (not PR1)

**What This Means**:
- Users can use the new AI SDK chat for conversational questions about Helm charts
- File editing and chart operations continue to use the existing Go-based flow until PR1.5
- This is acceptable because both systems coexist - users have full functionality via existing flow

**NOTE**: Tool calling (US-4 in PR1_Product_PRD.md) is delivered in **PR1.5**, not PR1. The PR structure is:
- **PR1**: AI SDK foundation (no tools)
- **PR1.5**: Migration & Feature Parity (adds 6 tools: createChart, getChartContext, updateChart, textEditor, latestSubchartVersion, latestKubernetesVersion)
- **PR2**: Validation Agent (adds validateChart tool)

**PR1.5 Implementation** (see PR1.5_PLAN.md for details):
Tool support will be added in PR1.5:
1. PR1.5 adds Go HTTP server with tool endpoints
2. All existing tools ported to AI SDK `tool()` format
3. Tool execute() calls Go HTTP endpoint directly
4. This establishes the pattern for PR2's validateChart tool

**PR2 Implementation**:
1. PR2 adds `/api/validate` endpoint to existing Go HTTP server
2. validateChart tool uses the proven tool → Go HTTP pattern from PR1.5

**Tool Definition Pattern** (for future reference):
```
DEFINE tool using AI SDK tool() helper:
  description: string explaining tool purpose
  inputSchema: Zod schema for parameters
  execute: async function implementing tool logic

EXAMPLE structure:
  latestKubernetesVersion:
    description: "Return the latest version of Kubernetes"
    inputSchema: z.object({
      semver_field: z.enum(["major", "minor", "patch"])
    })
    execute: async (params) => fetch from k8s API or static list
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

  // Tools deferred to future enhancement (see Tool Strategy section)

  // Return streaming response
  RETURN result.toDataStreamResponse()
```

### Streaming System Coexistence

PR1 introduces a **second streaming system** alongside the existing one:

| System | Protocol | Used By |
|--------|----------|---------|
| **Existing** | Centrifugo WebSocket | Workspace operations, plan progress, file updates |
| **NEW (PR1)** | AI SDK Data Stream (SSE) | New `/api/chat` responses |

**Both systems run in parallel**. The existing Centrifugo system continues to handle:
- `plan-updated` events
- `chatmessage-updated` events (for existing Go-based flow)
- `render-stream` events
- `artifact-updated` events
- `conversion-status` events

The new AI SDK Data Stream handles:
- Responses from the new `/api/chat` endpoint only

**No conflicts**: The two systems operate on different data flows. Users interacting with existing workspace operations continue to receive updates via Centrifugo, while users of the new AI SDK chat receive responses via the Data Stream protocol.

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

### Existing Test Files to Review

| File | Current Purpose | PR1 Impact |
|------|-----------------|------------|
| `atoms/__tests__/workspace.test.ts` | Render deduplication | May need message state tests for new flow |
| `components/__tests__/FileTree.test.ts` | Patch statistics | No changes expected |
| `hooks/__tests__/parseDiff.test.ts` | Diff parsing | No changes expected |

### New Test Files to Create

| File | Purpose |
|------|---------|
| `lib/ai/__tests__/provider.test.ts` | Provider factory unit tests |
| `components/chat/__tests__/ProviderSelector.test.ts` | Component tests |
| `app/api/chat/__tests__/route.test.ts` | API route integration tests |
| `tests/provider-switching.spec.ts` | Playwright E2E test for provider selection |

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
  openrouter("anthropic/claude-3.5-sonnet")
  openrouter("google/gemini-pro")
```

---

## Related Documents

| Document | Purpose |
|----------|---------|
| `PRDs/PR1_Product_PRD.md` | Functional requirements for this PR |
| `docs/research/2025-12-01-OPTION-A-VS-PARALLEL-EVALUATION.md` | Architecture decision rationale |
| `docs/research/2025-12-02-COMPREHENSIVE-PR1-RESEARCH.md` | Full codebase verification |
| `docs/research/2025-12-02-PR1-GAPS-ANALYSIS.md` | PRD gaps analysis |
| `docs/research/2025-12-01-PR1-ITERATION-CHECKLIST.md` | PRD iteration checklist |
| `ClaudeResearch/CURRENT_STATE_ANALYSIS.md` | Current architecture details |
| `ClaudeResearch/VERCEL_AI_SDK_REFERENCE.md` | SDK implementation patterns |
| `Replicated_Chartsmith.md` | Source of truth requirements |

---

*Document End - PRD2: PR1 Technical Specification*
