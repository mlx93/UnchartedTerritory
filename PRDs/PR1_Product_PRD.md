# PRD1: Chartsmith Vercel AI SDK Migration
## Functional Requirements Document

**Version**: 1.1  
**PR**: PR1 of 3  
**Timeline**: Days 1-3 (3 days)  
**Status**: Ready for Implementation

---

## Executive Summary

### Purpose
Migrate Chartsmith from custom `@anthropic-ai/sdk` implementation to Vercel AI SDK, enabling standardized chat functionality, multi-provider support, and reduced maintenance burden.

### Business Value
- **60% reduction** in custom chat code maintenance
- **Multi-provider flexibility** via single integration (OpenRouter)
- **Improved streaming performance** through SDK optimizations
- **Future-proof architecture** aligned with industry standards

### Scope
This PR delivers the complete Vercel AI SDK migration including all Must Haves and Nice to Haves from the Chartsmith project specification.

---

## Problem Statement

### Current State (Actual Chartsmith Architecture)

**IMPORTANT**: The current Chartsmith architecture differs significantly from a standard Next.js chat app:

Chartsmith uses a **database-driven architecture** for chat functionality:
- **Jotai atoms** for state management (`messagesAtom`, `plansAtom` in `atoms/workspace.ts`)
- **Go backend handles ALL LLM calls** - frontend only does prompt classification
- **PostgreSQL LISTEN/NOTIFY** for frontend↔backend communication (NOT HTTP)
- **Centrifugo WebSocket** for real-time streaming (NOT standard SSE)
- **3 existing tools in Go** - `text_editor`, `latest_subchart_version`, `latest_kubernetes_version`
- **Single provider lock-in** - Anthropic only, hardcoded in Go backend

### Pain Points
1. **High maintenance burden**: Custom streaming and state logic requires ongoing attention
2. **Provider inflexibility**: Switching LLM providers requires significant refactoring
3. **Inconsistent patterns**: Non-standard implementation creates onboarding friction
4. **Limited optimization**: Missing streaming and state management optimizations

### Desired State (What PR1 Creates)

PR1 creates a **NEW parallel chat system** that coexists with the existing Go-based flow:

- **NEW `/api/chat` route** using Vercel AI SDK `streamText`
- **NEW chat components** using `useChat` hook from `@ai-sdk/react`
- **Multi-provider support** via OpenRouter (Claude, GPT-4, Gemini, etc.)
- **Standard HTTP streaming** for the new chat flow
- **Provider selection UI** for choosing models

**What Remains Unchanged**:
- Go backend and its LLM calls (existing tools continue to work)
- Existing workspace APIs and Jotai state management
- Centrifugo real-time updates for existing features

---

## User Stories

### US-1: Chat Migration (Core)
**As a** Chartsmith user  
**I want** the chat interface to work exactly as before  
**So that** my workflow for creating Helm charts is uninterrupted

**Acceptance Criteria**:
- Chat messages display correctly (user and assistant)
- Streaming responses appear in real-time
- Message history persists within a session
- System prompts and behavior remain unchanged
- All existing keyboard shortcuts and interactions work

### US-2: Provider Selection
**As a** Chartsmith user  
**I want** to select which AI model powers my chat  
**So that** I can choose the best model for my needs

**Acceptance Criteria**:
- Provider selector visible when starting a new conversation
- At least 2 providers available (OpenAI GPT-4o, Claude)
- Selection persists for the duration of the conversation
- Selected model name displayed in UI
- Selector disabled once conversation begins (PR1 behavior)

### US-3: Streaming Experience
**As a** Chartsmith user  
**I want** smooth, responsive streaming of AI responses  
**So that** I can read the output as it generates

**Acceptance Criteria**:
- Text appears character-by-character or in small chunks
- No visible lag or stuttering during streaming
- Loading indicator shows when waiting for response
- User can see streaming status (ready/streaming/submitted)

### US-4: Tool Calling Preservation
**As a** Chartsmith user  
**I want** existing tool functionality to continue working  
**So that** I can still use file context and chart operations

**Acceptance Criteria**:
- File context tools work as before
- Chart generation tools function correctly
- Tool results display properly in chat
- No regression in tool response times

### US-5: Error Handling
**As a** Chartsmith user  
**I want** clear error messages when something goes wrong  
**So that** I understand what happened and can recover

**Acceptance Criteria**:
- Network errors display user-friendly messages
- Provider errors are caught and explained
- Retry option available after errors
- Error state clears when new message sent

---

## Functional Requirements

### FR-1: Chat Interface Migration

#### FR-1.1: Message Display
- Display user messages with appropriate styling
- Display assistant messages with streaming support
- Render tool invocations and results inline
- Support code blocks with syntax highlighting
- Preserve existing message formatting (markdown, etc.)

#### FR-1.2: Input Handling
- Text input field for user messages
- Submit on Enter key (Shift+Enter for newline)
- Submit button as alternative
- Input disabled during streaming
- Clear input after successful submit

#### FR-1.3: Conversation State
- Maintain message history array
- Support conversation reset/clear
- Preserve context across messages in session
- No persistence requirement (existing behavior)

#### FR-1.4: New Chart Creation Mode

When `workspace.currentRevisionNumber === 0` (new chart):
- Render `NewChartContent` component instead of main chat interface
- Hide role selector (use "auto" role by default)
- Display "Create a new Helm chart" header with workspace name
- Show "Create Chart" button when plan status is "review"
- Use simplified submit handler without role selection

**Existing Behavior**: This mode is already implemented in the current codebase (`ChatContainer.tsx:75-94`). The new AI SDK chat components should preserve this conditional rendering behavior when integrated.

### FR-2: Provider System

#### FR-2.1: Available Providers
| Provider | Model ID | Display Name |
|----------|----------|--------------|
| OpenAI | `openai/gpt-4o` | GPT-4o (Default) |
| Anthropic | `anthropic/claude-3.5-sonnet` | Claude 3.5 Sonnet |

#### FR-2.2: Provider Selector UI
- Dropdown or radio button selector
- Shown only when message history is empty
- Hidden after first message sent
- Shows currently selected provider name
- Default selection: OpenAI GPT-4o

#### FR-2.3: Provider Switching Logic
- Selection locked once conversation starts
- To change provider: start new conversation
- Provider choice sent with each API request
- Backend handles model routing via OpenRouter

### FR-3: Streaming Behavior

#### FR-3.1: Stream States
| State | UI Behavior |
|-------|-------------|
| `ready` | Input enabled, send button active |
| `submitted` | Input disabled, loading indicator |
| `streaming` | Input disabled, text appearing |
| `error` | Error message shown, retry available |

#### FR-3.2: Stream Controls
- Stop button to cancel streaming response
- Regenerate button to retry last response
- Both controls only visible during/after streaming

### FR-4: System Prompt Preservation

#### FR-4.1: Existing Prompts
- All existing system prompts must be preserved
- User role context maintained
- Chart context injection unchanged
- No modification to prompt content or structure

#### FR-4.2: Prompt Injection Points
- System prompt set at conversation start
- Context updated before each message
- Tool descriptions included automatically

### FR-5: Existing Feature Compatibility

#### FR-5.1: File Context
- File upload functionality unchanged
- File content injection into context works
- File references in messages display correctly

#### FR-5.2: Chart Operations
- Chart creation via chat works
- Chart editing via chat works
- Chart preview/export unchanged
- All chart-related tools functional

#### FR-5.3: VS Code Extension
- Extension authentication unchanged
- `/api/auth/status` endpoint works
- Bearer token validation functional
- Extension API requests succeed

---

## Non-Functional Requirements

### NFR-1: Performance
- Streaming latency: < 200ms time-to-first-token (excluding model latency)
- UI responsiveness: < 100ms for user interactions
- Memory usage: No increase from current implementation
- Bundle size: < 50KB increase from AI SDK packages

### NFR-2: Reliability
- Graceful handling of network interruptions
- Automatic retry logic for transient failures
- No data loss on stream interruption
- Session stability for 1+ hour conversations

### NFR-3: Compatibility
- Browser support: Chrome, Firefox, Safari, Edge (latest 2 versions)
- Mobile responsive: Existing mobile support maintained
- Accessibility: No regression from current state

### NFR-4: Security
- API keys never exposed to client
- All LLM calls through server-side routes
- Existing authentication mechanisms preserved
- No new security vulnerabilities introduced

---

## UI/UX Specifications

### Provider Selector Component

**Location**: Above chat input, visible only for new conversations

**States**:
1. **Visible**: No messages in conversation
2. **Hidden**: One or more messages exist

**Layout**:
```
┌─────────────────────────────────────┐
│  Model: [GPT-4o        ▼]           │
│         ○ GPT-4o (Default)          │
│         ○ Claude 3.5 Sonnet         │
└─────────────────────────────────────┘
```

**Interaction**:
- Click to expand dropdown
- Select option to change
- Selection persists in component state

### Chat Status Indicator

**Location**: Below chat messages, above input

**States**:
| Status | Display |
|--------|---------|
| Ready | (hidden or subtle ready indicator) |
| Submitted | "Thinking..." with spinner |
| Streaming | "Responding..." or hidden |
| Error | "Error: [message]" with retry button |

### Streaming Controls

**Location**: Inline with streaming message or below it

**Controls**:
- **Stop** (■): Visible during streaming, cancels generation
- **Regenerate** (↻): Visible after response complete, retries

---

## Data Flow

### NEW Chat Flow (PR1 Creates This)

This is the **NEW flow** that PR1 implements using Vercel AI SDK:

1. User types message in input
2. User clicks Send or presses Enter
3. Frontend calls `sendMessage` from useChat hook
4. Hook sends POST to **NEW** `/api/chat` with messages + provider
5. API route creates streamText request to OpenRouter
6. OpenRouter forwards to selected model
7. Streaming response flows back via Data Stream protocol
8. useChat hook updates messages state
9. UI re-renders with new content

### EXISTING Chat Flow (Unchanged)

For reference, the existing Chartsmith flow (which continues to work):

1. User types message in ChatContainer
2. `createChatMessageAction` server action called
3. Message inserted into `workspace_chat` table
4. `pg_notify()` triggers Go worker
5. Go worker processes via `pkg/listener/` handlers
6. LLM calls made from Go using Anthropic SDK
7. Results pushed via Centrifugo WebSocket
8. Frontend receives via `useCentrifugo.ts`
9. Jotai atoms updated, UI re-renders

### Provider Selection Flow (NEW)
1. User opens new conversation (empty state)
2. Provider selector renders with default (GPT-4o)
3. User optionally changes selection
4. Selection stored in component state
5. On first message, provider passed in request body
6. Provider selector hides
7. All subsequent messages use same provider

---

## API Contracts

### Chat Endpoint

**Endpoint**: `POST /api/chat`

**Request Body**:
```
{
  "messages": [
    { "role": "user", "content": "string" },
    { "role": "assistant", "content": "string" }
  ],
  "provider": "openai" | "anthropic",
  "model": "openai/gpt-4o" | "anthropic/claude-3.5-sonnet"
}
```

**Response**: Vercel AI SDK Data Stream

**Error Response**:
```
{
  "error": "string",
  "details": "string" (optional)
}
```

---

## Test Requirements

### Unit Tests
- Provider factory returns correct models
- Message formatting functions work correctly
- Error handling utilities function properly

### Integration Tests
- useChat hook integration with API route
- Streaming end-to-end with mock provider
- Provider switching in new conversation
- Error states trigger correct UI

### Regression Tests
- All existing chat functionality works
- File context tools functional
- Chart operations successful
- VS Code extension auth works

### Manual Test Scenarios

| Scenario | Steps | Expected Result |
|----------|-------|-----------------|
| Basic chat | Send message, wait for response | Response streams and displays |
| Provider switch | Select Claude, send message | Response uses Claude |
| Stop streaming | Click stop during response | Streaming stops, partial response shown |
| Regenerate | Click regenerate after response | New response generated |
| Error recovery | Disconnect network, send message | Error shown, retry works |

---

## Success Criteria

### Must Pass (All Required)
- [ ] Chat functionality equivalent to current implementation
- [ ] Streaming works with no visible degradation
- [ ] Provider selector allows choosing between GPT-4o and Claude
- [ ] All existing tests pass (or updated appropriately)
- [ ] VS Code extension authentication works
- [ ] No console errors during normal operation

### Quality Gates
- [ ] Code review approved
- [ ] All automated tests pass
- [ ] Manual QA scenarios pass
- [ ] Performance benchmarks met
- [ ] Documentation updated

---

## Dependencies

### External Dependencies
- Vercel AI SDK (`ai` package)
- AI SDK React (`@ai-sdk/react`)
- OpenRouter Provider (`@openrouter/ai-sdk-provider`)
- OpenRouter API availability

### Internal Dependencies
- Existing Go backend (unchanged in PR1)
- Current authentication system
- Existing tool implementations

---

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| SDK version incompatibility | Medium | High | Pin exact versions, test thoroughly |
| OpenRouter downtime | Low | Medium | Document fallback to direct OpenAI |
| Streaming behavior differences | Medium | Medium | Thorough testing of edge cases |
| State management migration bugs | Medium | High | Incremental migration, extensive testing |

---

## Out of Scope (PR1)

The following are explicitly NOT included in PR1:
- Live mid-conversation provider switching (PR2)
- Chart validation agent (PR2)
- Go backend modifications (PR2)
- New tool implementations (PR2)
- Conversation persistence
- User preferences storage

---

## Appendix: Glossary

| Term | Definition |
|------|------------|
| useChat | Vercel AI SDK React hook for chat functionality |
| streamText | AI SDK Core function for streaming text generation |
| OpenRouter | Meta-provider offering access to multiple LLM APIs |
| Data Stream | Vercel AI SDK's streaming response format |
| Tool Calling | LLM capability to invoke defined functions |

---

## Related Documents

| Document | Purpose |
|----------|---------|
| `PRDs/PR1_Tech_PRD.md` | Technical specification for this PR |
| `docs/research/2025-12-01-OPTION-A-VS-PARALLEL-EVALUATION.md` | Architecture decision rationale |
| `docs/research/2025-12-02-COMPREHENSIVE-PR1-RESEARCH.md` | Full codebase verification |
| `docs/research/2025-12-02-PR1-GAPS-ANALYSIS.md` | PRD gaps analysis |
| `docs/research/2025-12-01-PR1-ITERATION-CHECKLIST.md` | PRD iteration checklist |
| `ClaudeResearch/CURRENT_STATE_ANALYSIS.md` | Current architecture details |
| `ClaudeResearch/VERCEL_AI_SDK_REFERENCE.md` | SDK implementation patterns |
| `Replicated_Chartsmith.md` | Source of truth requirements |

---

*Document End - PRD1: PR1 Functional Requirements*
