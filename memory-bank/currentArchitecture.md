# Chartsmith Current Architecture

**Purpose**: Document the existing Chartsmith architecture before migration, explaining how the current code works.

---

## High-Level System Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                              │
├─────────────────────────┬───────────────────────────────────────┤
│   Next.js Web App       │      VS Code Extension                │
│   (TypeScript/React)    │      (TypeScript)                     │
│   - Custom Chat UI      │      - Token-based Auth               │
│   - Jotai state mgmt    │      - Bearer token header            │
│   - @anthropic-ai/sdk   │      - /api/auth/status endpoint      │
│     (prompt-type only)  │                                       │
└─────────────┬───────────┴───────────────────┬───────────────────┘
              │                               │
              ▼                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                        API LAYER                                 │
│                     Next.js API Routes                           │
│   - /api/workspace/[id]/message (creates chat via work queue)   │
│   - /api/auth/status (token validation)                         │
│   - NO direct /api/chat route currently exists                  │
└─────────────────────────────┬───────────────────────────────────┘
                              │ PostgreSQL Work Queue
                              │ (pg_notify / LISTEN)
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        BACKEND LAYER                             │
│                        Go Backend                                │
│   - Helm chart processing (pkg/helm/)                           │
│   - LLM integration (pkg/llm/) - 13 files                       │
│   - Tool execution (text_editor tool)                           │
│   - Real-time updates via Centrifugo                            │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     EXTERNAL SERVICES                            │
│   - Anthropic Claude API (claude-3-7-sonnet, claude-3-5-sonnet) │
│   - Groq API (intent, feedback, file conversion - llama-3.3)    │
│   - Voyage API (embeddings)                                      │
│   - Artifact Hub (dependency fetching)                          │
│   - Centrifugo (real-time WebSocket)                            │
└─────────────────────────────────────────────────────────────────┘
```

---

## Key Architectural Characteristics

### What Makes This Non-Standard

1. **No `/api/chat` HTTP Endpoint**
   - Messages go through `/api/workspace/[id]/message`
   - Creates entry in `workspace_chat` table
   - Triggers `pg_notify()` to Go worker

2. **Go Backend Has NO HTTP Server**
   - All communication via PostgreSQL LISTEN/NOTIFY
   - No direct HTTP calls from frontend to Go

3. **All LLM Calls Happen in Go Backend**
   - Frontend only does prompt classification (orphaned code)
   - Go handles all Anthropic/Groq SDK calls

4. **State Managed via Jotai Atoms**
   - `messagesAtom`, `plansAtom`, etc. in `atoms/workspace.ts`
   - Not using AI SDK state management

5. **Streaming via Centrifugo WebSocket**
   - Not standard HTTP/SSE streaming
   - Real-time updates pushed from Go to frontend

6. **3 Existing Tools (Anthropic-native format in Go)**
   - `text_editor_20241022`
   - `latest_subchart_version`
   - `latest_kubernetes_version`

---

## Chat Message Flow (Current)

```
1. User types message in ChatContainer.tsx
                    │
                    ▼
2. createChatMessageAction (server action)
                    │
                    ▼
3. INSERT into workspace_chat table
                    │
                    ▼
4. pg_notify() via lib/utils/queue.ts
                    │
                    ▼
5. Go worker receives via pkg/listener/listener.go
                    │
                    ▼
6. Handler processes (new_intent, new_plan, etc.)
                    │
                    ▼
7. LLM call via pkg/llm/* (Anthropic SDK)
                    │
                    ▼
8. Response sent via Centrifugo WebSocket
                    │
                    ▼
9. useCentrifugo.ts receives update
                    │
                    ▼
10. Jotai atoms updated, UI re-renders
```

---

## Frontend Architecture

### Component Hierarchy

```
Workspace Page
└── WorkspaceContent
    ├── FileTree (left panel)
    ├── CodeEditor (center)
    │   └── Monaco Editor
    └── ChatContainer (right panel)
        ├── ChatMessages (scrollable list)
        │   ├── ChatMessage
        │   ├── PlanChatMessage
        │   └── NewChartChatMessage
        └── PromptInput (input field)
```

### State Management (Jotai)

Location: `chartsmith-app/atoms/workspace.ts`

| Atom | Purpose | Line |
|------|---------|------|
| `workspaceAtom` | Current workspace data | 6 |
| `messagesAtom` | Chat message history | 10 |
| `plansAtom` | Planning state | 16 |
| `rendersAtom` | Render results | 22 |
| `isRenderingAtom` | Rendering status | 259-261 |

### Real-Time Updates

Location: `chartsmith-app/hooks/useCentrifugo.ts`

Centrifugo channels handle:
- `plan-updated`
- `chatmessage-updated`
- `render-stream`
- `artifact-updated`
- `conversion-status`

---

## Backend Architecture (Go)

### Package Structure

```
pkg/
├── llm/                  # LLM integration (13 files)
│   ├── client.go         # Anthropic client factory
│   ├── execute-action.go # Tool execution (text_editor)
│   ├── conversational.go # Chat with 2 utility tools
│   ├── intent.go         # Intent detection (Groq)
│   ├── convert-file.go   # File conversion (Groq)
│   ├── system.go         # System prompts
│   ├── initial-plan.go   # Initial plan generation
│   ├── plan.go           # Plan updates
│   ├── execute-plan.go   # Execution planning
│   ├── summarize.go      # Content summarization
│   ├── expand.go         # Prompt expansion
│   └── cleanup-converted-values.go
│
├── listener/             # PostgreSQL queue handlers (19 files)
│   ├── listener.go       # Work queue processor
│   ├── start.go          # Channel registration (8 channels)
│   ├── new_intent.go     # Intent processing
│   ├── new-plan.go       # Plan generation
│   ├── execute-plan.go   # Plan execution
│   ├── apply-plan.go     # Apply changes
│   ├── conversational.go # Chat handling
│   └── render-workspace.go
│
├── realtime/             # Centrifugo integration
│   └── centrifugo.go     # Event publishing
│
└── workspace/            # Workspace management
    └── types.go          # Workspace, Chart, File types
```

### PostgreSQL Queue Channels

The Go worker listens on these channels via `pkg/listener/start.go`:

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
| `claude-3-7-sonnet-20250219` | Anthropic | Planning, conversational, summarization |
| `claude-3-5-sonnet-20241022` | Anthropic | Action execution with text_editor tool |
| `llama-3.3-70b-versatile` | Groq | Intent detection, feedback, file conversion |

### Existing Tools (Anthropic-native format)

**text_editor_20241022** (`pkg/llm/execute-action.go:510-532`)
```
Commands: view, str_replace, create
Purpose: File manipulation in Helm charts
Includes fuzzy string matching (lines 238-435)
```

**Utility Tools** (`pkg/llm/conversational.go:99-128`)
```
latest_subchart_version - ArtifactHub lookup
latest_kubernetes_version - K8s version info
```

### Groq Integration

- **Intent Detection** (`intent.go:86-97`): Classifies user messages as developer/operator/ambiguous/off-topic
- **Feedback Streaming** (`intent.go:151-270`): Provides contextual feedback
- **File Conversion** (`convert-file.go:64-70`): Converts K8s manifests to Helm charts

---

## Database Schema (Key Tables)

| Table | Purpose |
|-------|---------|
| `workspace` | Workspace metadata |
| `workspace_revision` | Revision history |
| `workspace_chart` | Chart metadata |
| `workspace_file` | File contents |
| `workspace_chat` | Chat messages |
| `work_queue` | Background job queue |
| `extension_token` | VS Code extension auth |
| `bootstrap_workspace` | Template workspaces |
| `bootstrap_chart` | Template charts |
| `bootstrap_file` | Template files |

---

## VS Code Extension Authentication

```
┌──────────────┐     1. Click Login     ┌──────────────┐
│   VS Code    │ ───────────────────────▶│   Browser    │
│  Extension   │                         │  Auth Page   │
└──────────────┘                         └──────┬───────┘
       ▲                                        │
       │                                        │ 2. Authenticate
       │                                        ▼
       │                                 ┌──────────────┐
       │    4. Store & use token         │   Chartsmith │
       │◀────────────────────────────────│   Backend    │
       │                                 └──────┬───────┘
       │                                        │
       │         3. Generate extension token    │
       └────────────────────────────────────────┘

Token validation: GET /api/auth/status (Bearer token header)
Endpoint: chartsmith-app/app/api/auth/status/route.ts
```

---

## API Routes (Existing)

| Route | Method | Purpose |
|-------|--------|---------|
| `/api/workspace/[id]/message` | POST | Create chat message |
| `/api/workspace/[id]/messages` | GET | List messages |
| `/api/workspace/[id]/plans` | GET | List plans |
| `/api/workspace/[id]/revision` | POST | Create revision |
| `/api/workspace/[id]/renders` | GET | List renders |
| `/api/workspace/[id]` | GET | Get workspace |
| `/api/auth/status` | GET | Token validation |
| `/api/config` | GET | Public config |
| `/api/push` | GET | Centrifugo token |
| `/api/upload-chart` | POST | Chart upload |

**Note**: No `/api/chat` route exists - this is created in PR1.

---

## Frontend Anthropic SDK Usage

**Single Import Location**: `chartsmith-app/lib/llm/prompt-type.ts:1`

```typescript
import Anthropic from '@anthropic-ai/sdk';
```

**Purpose**: Classify user prompts as "plan" or "chat"

**Model**: `claude-3-5-sonnet-20241022`

**Status**: ⚠️ ORPHANED CODE - The `promptType()` function is never called anywhere in the codebase. This file will be deleted during PR1.

---

## Environment Variables

### AI/LLM APIs

| Variable | Frontend | Backend |
|----------|----------|---------|
| `ANTHROPIC_API_KEY` | `lib/llm/prompt-type.ts` (orphaned) | `pkg/param/param.go` |
| `GROQ_API_KEY` | - | `pkg/param/param.go` |
| `VOYAGE_API_KEY` | - | `pkg/param/param.go` |

### Database & Services

| Variable | Purpose |
|----------|---------|
| `CHARTSMITH_PG_URI` | PostgreSQL connection |
| `CHARTSMITH_CENTRIFUGO_ADDRESS` | Centrifugo API |
| `CHARTSMITH_CENTRIFUGO_API_KEY` | Centrifugo auth |
| `CENTRIFUGO_TOKEN_HMAC_SECRET` | JWT signing |
| `NEXT_PUBLIC_CENTRIFUGO_ADDRESS` | Client WebSocket |

### Authentication

| Variable | Purpose |
|----------|---------|
| `HMAC_SECRET` | JWT session signing |
| `TOKEN_ENCRYPTION` | AES-256-GCM key |
| `GOOGLE_CLIENT_SECRET` | OAuth secret |
| `NEXT_PUBLIC_GOOGLE_CLIENT_ID` | OAuth client ID |

---

## Current Limitations

1. **Single Provider Lock-in**: Only Anthropic Claude (Go backend)
2. **Custom Streaming**: PostgreSQL queue + Centrifugo (not standard SSE)
3. **State Management**: Jotai atoms, not AI SDK's `useChat`
4. **Maintenance Burden**: Custom implementations throughout
5. **No Tool Calling in Frontend**: All tools in Go backend
6. **No `/api/chat` Route**: Messages go through workspace-specific routes

---

## What Migration Changes

| Aspect | Before | After (PR1.5+) |
|--------|--------|----------------|
| Chat route | `/api/workspace/[id]/message` | NEW `/api/chat` |
| LLM calls | Go backend (Anthropic SDK) | Next.js (AI SDK + OpenRouter) |
| Streaming | Centrifugo WebSocket | AI SDK Data Stream |
| State | Jotai atoms | `useChat` hook |
| Tools | 3 Go tools (Anthropic format) | 7 AI SDK tools |
| Go role | All LLM + execution | Execution only (no LLM) |

---

*This document captures how Chartsmith works before the AI SDK migration.*

