# Chartsmith Current Architecture

**Purpose**: Document the current Chartsmith architecture after PR1, explaining how both the new AI SDK system and existing system work.

---

## High-Level System Overview (After PR1)

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                              │
├─────────────────────────┬───────────────────────────────────────┤
│   Next.js Web App       │      VS Code Extension                │
│   (TypeScript/React)    │      (TypeScript)                     │
│                         │                                       │
│   NEW (PR1):            │      Token-based Auth                 │
│   - AIChat component    │      Bearer token header              │
│   - useChat hook        │      /api/auth/status endpoint        │
│   - ProviderSelector    │                                       │
│   - /test-ai-chat page  │                                       │
│                         │                                       │
│   EXISTING (preserved): │                                       │
│   - Jotai state mgmt    │                                       │
│   - ChatContainer       │                                       │
│   - Centrifugo WS       │                                       │
└─────────────┬───────────┴───────────────────┬───────────────────┘
              │                               │
              ▼                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                        API LAYER                                 │
│                     Next.js API Routes                           │
│                                                                  │
│   NEW (PR1):                                                     │
│   - /api/chat (AI SDK streamText, toTextStreamResponse)          │
│                                                                  │
│   EXISTING (preserved):                                          │
│   - /api/workspace/[id]/message (creates chat via work queue)   │
│   - /api/auth/status (token validation)                         │
└─────────────────────────────┬───────────────────────────────────┘
                              │
              ┌───────────────┴───────────────┐
              │                               │
              ▼ NEW: Direct LLM              ▼ EXISTING: PostgreSQL Queue
┌──────────────────────────┐    ┌────────────────────────────────┐
│   LLM PROVIDERS (PR1)     │    │       GO BACKEND               │
│                           │    │                                │
│ Priority:                 │    │ - Helm chart processing        │
│ 1. Direct Anthropic API   │    │ - LLM integration (pkg/llm/)   │
│ 2. Direct OpenAI API      │    │ - Tool execution               │
│ 3. OpenRouter fallback    │    │ - Real-time via Centrifugo     │
│                           │    │                                │
│ Default: Claude Sonnet 4  │    │ PR1.5: Adds HTTP server :8080  │
└──────────────────────────┘    └────────────────────────────────┘
                                              │
                                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     EXTERNAL SERVICES                            │
│   - Anthropic Claude API                                         │
│   - OpenAI API (NEW in PR1)                                      │
│   - OpenRouter API (NEW in PR1)                                  │
│   - Groq API (intent, feedback, file conversion - llama-3.3)    │
│   - Voyage API (embeddings)                                      │
│   - Artifact Hub (dependency fetching)                          │
│   - Centrifugo (real-time WebSocket)                            │
└─────────────────────────────────────────────────────────────────┘
```

---

## Dual Chat Systems (PR1 State)

### NEW: AI SDK Chat System

| Component | Purpose |
|-----------|---------|
| `/api/chat/route.ts` | API endpoint using `streamText` |
| `lib/ai/provider.ts` | Provider factory with `getModel()` |
| `lib/ai/models.ts` | Model definitions |
| `lib/ai/config.ts` | System prompt and configuration |
| `components/chat/AIChat.tsx` | Chat UI with `useChat` hook |
| `components/chat/AIMessageList.tsx` | Parts-based message rendering |
| `components/chat/ProviderSelector.tsx` | Model selection dropdown |
| `/test-ai-chat` | Test page matching production UI |

**Flow**:
```
User → AIChat → useChat → POST /api/chat → streamText → LLM → SSE stream → UI
```

### EXISTING: Go-Based Chat System (Preserved)

| Component | Purpose |
|-----------|---------|
| `ChatContainer.tsx` | Chat UI with Jotai state |
| `createChatMessageAction` | Server action |
| PostgreSQL `workspace_chat` | Message storage |
| `pg_notify()` | Queue trigger |
| Go worker `pkg/listener/` | Queue processor |
| `pkg/llm/` | Anthropic SDK calls |
| Centrifugo | Real-time streaming |

**Flow**:
```
User → ChatContainer → Server Action → INSERT → pg_notify → Go → LLM → Centrifugo → UI
```

---

## PR1 Implementation Details

### Provider Factory (`lib/ai/provider.ts`)

```typescript
// Exports
export function getModel(provider?: string, modelId?: string): LanguageModel;
export const AVAILABLE_PROVIDERS: ProviderConfig[];
export const AVAILABLE_MODELS: ModelConfig[];
export type Provider = 'openai' | 'anthropic';

// Provider Priority (checks env vars)
1. ANTHROPIC_API_KEY → createAnthropic() direct
2. OPENAI_API_KEY → createOpenAI() direct
3. OPENROUTER_API_KEY → createOpenRouter() fallback
```

### API Route (`/api/chat/route.ts`)

```typescript
// Request body (PR1)
{ messages: UIMessage[], provider?: string, model?: string }

// Response: AI SDK Text Stream (SSE)

// Implementation uses:
- streamText() from 'ai'
- getModel() from provider.ts
- toTextStreamResponse() (NOT toDataStreamResponse)
```

### AI SDK v5 Specifics

| v4 Pattern | v5 Actual |
|------------|-----------|
| `Message` type | `UIMessage` type |
| `api` option in useChat | `TextStreamChatTransport` |
| `handleSubmit()` | `sendMessage()` |
| `toDataStreamResponse()` | `toTextStreamResponse()` |

---

## Frontend Architecture (Updated)

### Component Hierarchy

```
NEW (PR1):
test-ai-chat Page
└── EditorLayout
    └── TopNav (logo, "by Replicated", Export)
    └── AIChat
        ├── ProviderSelector
        ├── AIMessageList
        │   └── Parts rendering (text, tool-invocation, etc.)
        └── Input area

EXISTING (Preserved):
Workspace Page
└── WorkspaceContent
    ├── FileTree (left panel)
    ├── CodeEditor (center)
    │   └── Monaco Editor
    └── ChatContainer (right panel)
        ├── ChatMessages (Jotai-based)
        └── PromptInput
```

### State Management

| System | State Source | Components |
|--------|--------------|------------|
| NEW | `useChat` hook | AIChat, AIMessageList |
| EXISTING | Jotai atoms | ChatContainer, ChatMessages |

Jotai atoms preserved in `atoms/workspace.ts`:
- `workspaceAtom` - Current workspace
- `messagesAtom` - Chat history (existing system)
- `plansAtom` - Planning state
- `rendersAtom` - Render results

---

## Backend Architecture (Go)

### Package Structure (Current)

```
pkg/
├── llm/                  # LLM integration (13 files) - STILL ACTIVE
│   ├── client.go         # Anthropic client factory
│   ├── execute-action.go # Tool execution (text_editor)
│   ├── conversational.go # Chat with utility tools
│   ├── intent.go         # Intent detection (Groq)
│   └── ...
│
├── listener/             # PostgreSQL queue handlers (19 files)
│   ├── listener.go       # Work queue processor
│   ├── start.go          # Channel registration
│   └── ...
│
├── realtime/             # Centrifugo integration
│   └── centrifugo.go
│
├── workspace/            # Workspace management
│   └── ...
│
└── api/                  # PR1.5 (NOT YET CREATED)
    ├── server.go         # HTTP server on :8080
    ├── errors.go         # Error utilities
    └── handlers/         # Tool endpoints
```

### PostgreSQL Queue Channels (Existing)

Go worker listens via `pkg/listener/start.go`:
1. `new_intent`
2. `new_plan`
3. `execute_plan`
4. `apply_plan`
5. `conversational`
6. `render_workspace`
7. `conversion_status`
8. `heartbeat`

---

## LLM Integration Details

### Models Used

| Model | Provider | Purpose |
|-------|----------|---------|
| `claude-sonnet-4-20250514` | Anthropic (Direct) | **Default** - AI SDK chat |
| `gpt-4o` | OpenAI (Direct) | Alternative - AI SDK chat |
| `claude-3-7-sonnet` | Anthropic | Go backend - planning |
| `claude-3-5-sonnet` | Anthropic | Go backend - action execution |
| `llama-3.3-70b-versatile` | Groq | Intent detection, feedback |

### Existing Tools (Go - Anthropic format)

**text_editor_20241022** (`pkg/llm/execute-action.go`):
- Commands: view, str_replace, create
- Includes fuzzy string matching

**Utility Tools** (`pkg/llm/conversational.go`):
- `latest_subchart_version` - ArtifactHub lookup
- `latest_kubernetes_version` - K8s version info

---

## API Routes

### NEW (PR1)

| Route | Method | Purpose |
|-------|--------|---------|
| `/api/chat` | POST | AI SDK streaming chat |

### EXISTING (Preserved)

| Route | Method | Purpose |
|-------|--------|---------|
| `/api/workspace/[id]/message` | POST | Create chat message (Go queue) |
| `/api/workspace/[id]/messages` | GET | List messages |
| `/api/workspace/[id]/plans` | GET | List plans |
| `/api/workspace/[id]/revision` | POST | Create revision |
| `/api/auth/status` | GET | Token validation |

### PR1.5 (To Be Created)

| Route | Method | Purpose |
|-------|--------|---------|
| `/api/tools/editor` | POST | textEditor tool |
| `/api/tools/versions/subchart` | POST | latestSubchartVersion tool |
| `/api/tools/versions/kubernetes` | POST | latestKubernetesVersion tool |

---

## Environment Variables (Updated)

### AI SDK (PR1)

| Variable | Purpose |
|----------|---------|
| `ANTHROPIC_API_KEY` | Direct Anthropic API (priority 1) |
| `OPENAI_API_KEY` | Direct OpenAI API (priority 2) |
| `OPENROUTER_API_KEY` | OpenRouter fallback (priority 3) |
| `DEFAULT_AI_PROVIDER` | Default provider (anthropic) |
| `DEFAULT_AI_MODEL` | Default model |

### Existing (Preserved)

| Variable | Purpose |
|----------|---------|
| `CHARTSMITH_PG_URI` | PostgreSQL connection |
| `CHARTSMITH_CENTRIFUGO_*` | Centrifugo config |
| `GROQ_API_KEY` | Go intent detection |
| `VOYAGE_API_KEY` | Go embeddings |

---

## What Changed in PR1

| Aspect | Before | After (PR1) |
|--------|--------|-------------|
| Chat route | None (Go queue only) | NEW `/api/chat` |
| LLM calls | Go backend only | Go + Node (parallel) |
| Streaming | Centrifugo only | Centrifugo + AI SDK SSE |
| State | Jotai only | Jotai + useChat |
| Frontend SDK | @anthropic-ai/sdk (orphaned) | **REMOVED** |
| Providers | Anthropic only | Anthropic, OpenAI, OpenRouter |
| Default model | claude-3-5-sonnet | claude-sonnet-4-20250514 |

---

## What PR1.5 Will Change

| Aspect | PR1 State | After PR1.5 |
|--------|-----------|-------------|
| Go HTTP server | None | Port 8080 with 3 endpoints |
| AI SDK tools | None | 4 tools registered |
| LLM calls in Go | Still active | Deprecated (only Groq for intent) |
| System prompts | In Go | Migrated to Node |

---

## Test Coverage (PR1)

```
Test Suites: 7 passed, 7 total
Tests:       61 passed, 61 total
```

### New Tests (PR1)

| File | Tests | Coverage |
|------|-------|----------|
| `lib/ai/__tests__/provider.test.ts` | 19 | Provider factory, validation |
| `lib/ai/__tests__/config.test.ts` | 6 | Config constants |
| `app/api/chat/__tests__/route.test.ts` | 12 | API validation, errors |

### Existing Test Utilities

- `lib/__tests__/ai-mock-utils.ts` - AI SDK mocking (use for PR1.5)

---

*This document captures how Chartsmith works after PR1. Last updated: Dec 4, 2025*
