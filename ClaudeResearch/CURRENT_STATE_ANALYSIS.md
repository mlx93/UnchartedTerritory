# Chartsmith Current State Analysis

## Overview

Chartsmith is an AI-powered tool by Replicated that helps developers build better Helm charts. This document analyzes the current architecture based on **actual codebase exploration** (see `docs/research/PR1_CODEBASE_ANALYSIS.md` for full details).

---

## System Architecture

### High-Level Components

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
│   - LLM integration (pkg/llm/) - 10 files                       │
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

## Current LLM Integration

### Frontend (chartsmith-app)

**Single Anthropic SDK Usage**:
- **File**: `chartsmith-app/lib/llm/prompt-type.ts:1`
- **Import**: `import Anthropic from '@anthropic-ai/sdk';`
- **Purpose**: Classify user prompts as "plan" or "chat"
- **Model**: `claude-3-5-sonnet-20241022`
- **Dependency**: `"@anthropic-ai/sdk": "^0.39.0"` (package.json:21)

**State Management**:
- **Framework**: Jotai (NOT Redux, NOT Context)
- **Location**: `chartsmith-app/atoms/workspace.ts`
- **Key Atoms**:
  - `workspaceAtom` (line 6)
  - `messagesAtom` (line 10)
  - `plansAtom` (line 16)
  - `rendersAtom` (line 22)
  - `isRenderingAtom` (lines 259-261)

### Backend (Go)

**Primary SDK**: `github.com/anthropics/anthropic-sdk-go v0.2.0-alpha.11` (go.mod:8)
**Secondary SDK**: `github.com/jpoz/groq v0.0.0-20240513145022-7a02894105a0` (go.mod)

**Client Factory**: `pkg/llm/client.go:12-21`

**Files in `pkg/llm/`** (13 files total):
| File | Purpose | Provider |
|------|---------|----------|
| `client.go:12-21` | Anthropic client factory | Anthropic |
| `initial-plan.go:21-80` | Initial plan generation | Anthropic |
| `plan.go:21-100` | Plan updates | Anthropic |
| `execute-plan.go:14-60` | Execution plan generation | Anthropic |
| `execute-action.go:437-673` | Action execution with tools | Anthropic |
| `conversational.go:14-234` | Conversational chat with tools | Anthropic |
| `summarize.go:100-120` | Content summarization | Anthropic |
| `expand.go:10-54` | Prompt expansion | Anthropic |
| `cleanup-converted-values.go:13-90` | Values cleanup | Anthropic |
| `convert-file.go:33-70` | File conversion | **Groq** |
| `intent.go:20-270` | Intent detection & feedback | **Groq** |
| `artifact.go` | Artifact handling | Anthropic |
| `system.go` | System prompts | N/A |

**Models Used**:

| Model | Provider | Usage |
|-------|----------|-------|
| `claude-3-7-sonnet-20250219` | Anthropic | Planning, conversational, summarization, expansion |
| `claude-3-5-sonnet-20241022` | Anthropic | Action execution with text_editor tool |
| `llama-3.3-70b-versatile` | **Groq** | Intent detection, feedback generation, file conversion |

**Groq Integration Details**:
- **Intent Detection** (`intent.go:86-97`): Classifies user messages as developer/operator/ambiguous/off-topic
- **Feedback Streaming** (`intent.go:151-270`): Provides contextual feedback for different intent types
- **File Conversion** (`convert-file.go:64-70`): Converts K8s manifests to Helm charts

### Pain Points (Confirmed)
1. Custom chat UI components require significant maintenance
2. Manual message handling across frontend and backend
3. Custom streaming via PostgreSQL work queue + Centrifugo
4. State management in Jotai atoms (custom, not AI SDK)
5. Tight coupling to Anthropic provider (Go backend does all heavy lifting)

---

## Directory Structure (Actual)

```
chartsmith/
├── chartsmith-app/                    # Next.js frontend application
│   ├── ARCHITECTURE.md
│   ├── CLAUDE.md
│   ├── app/
│   │   ├── api/                       # API Routes
│   │   │   ├── auth/status/route.ts   # Token validation
│   │   │   ├── config/route.ts        # Public config
│   │   │   ├── push/route.ts          # Centrifugo token
│   │   │   ├── upload-chart/route.ts  # Chart upload
│   │   │   └── workspace/
│   │   │       └── [workspaceId]/
│   │   │           ├── message/route.ts   # Create chat message
│   │   │           ├── messages/route.ts  # List messages
│   │   │           ├── plans/route.ts     # List plans
│   │   │           ├── revision/route.ts  # Create revision
│   │   │           ├── renders/route.ts   # List renders
│   │   │           └── route.ts           # Get workspace
│   │   └── ...
│   ├── atoms/
│   │   └── workspace.ts               # Jotai state management
│   ├── components/
│   │   ├── ChatContainer.tsx:18       # Main chat interface
│   │   ├── ChatMessage.tsx:72         # Message display
│   │   ├── NewChartChatMessage.tsx:69 # New chart messages
│   │   ├── PlanChatMessage.tsx:38     # Plan display
│   │   ├── PromptInput.tsx:16         # Input component
│   │   ├── ScrollingContent.tsx:17    # Auto-scroll
│   │   └── __tests__/                 # Jest unit tests
│   ├── hooks/
│   │   ├── useCentrifugo.ts:511       # Real-time WebSocket
│   │   └── __tests__/                 # Jest unit tests
│   ├── lib/
│   │   ├── llm/
│   │   │   └── prompt-type.ts:1       # ONLY @anthropic-ai/sdk import
│   │   ├── auth/                      # Authentication
│   │   ├── centrifugo/                # Centrifugo client
│   │   ├── data/                      # Data access
│   │   ├── utils/
│   │   │   └── queue.ts:9-20          # Work queue (pg_notify)
│   │   └── workspace/
│   │       └── actions/               # Server actions
│   ├── tests/                         # Playwright E2E tests
│   │   ├── chat-scrolling.spec.ts
│   │   ├── import-artifactory.spec.ts
│   │   ├── login.spec.ts
│   │   ├── upload-chart.spec.ts
│   │   └── helpers.ts
│   ├── package.json
│   ├── jest.config.ts
│   └── playwright.config.ts
│
├── chartsmith-extension/              # VS Code Extension
│   ├── src/
│   │   ├── modules/
│   │   │   ├── webSocket/             # WebSocket integration
│   │   │   └── lifecycle/             # Extension lifecycle
│   │   └── types.ts
│   ├── package.json
│   └── webpack.config.js
│
├── cmd/                               # Go entry points
│   ├── root.go                        # Command registration
│   ├── run.go                         # Worker startup
│   ├── bootstrap.go                   # Bootstrap chart data
│   ├── artifacthub.go                 # ArtifactHub integration
│   └── debug-console.go               # Debug CLI
├── main.go                            # Entry point
│
├── pkg/                               # Go packages
│   ├── llm/                           # LLM integration (13 files)
│   │   ├── client.go                  # Anthropic client factory
│   │   ├── execute-action.go          # Tool execution (text_editor)
│   │   ├── conversational.go          # Chat with tools
│   │   ├── intent.go                  # Intent detection (Groq)
│   │   ├── convert-file.go            # File conversion (Groq)
│   │   ├── system.go                  # System prompts
│   │   ├── parser.go                  # Response parsing
│   │   ├── initial-plan.go            # Initial plan generation
│   │   ├── plan.go                    # Plan creation/updates
│   │   ├── execute-plan.go            # Execution planning
│   │   ├── summarize.go               # Content summarization
│   │   ├── expand.go                  # Prompt expansion
│   │   ├── cleanup-converted-values.go # Values cleanup
│   │   └── *_test.go                  # Go tests
│   ├── listener/                      # PostgreSQL LISTEN/NOTIFY (19 files)
│   │   ├── listener.go                # Work queue processor
│   │   ├── start.go                   # Channel handlers
│   │   ├── new_intent.go              # Intent processing
│   │   ├── new-plan.go                # Plan generation
│   │   ├── execute-plan.go            # Plan execution
│   │   ├── apply-plan.go              # Apply changes
│   │   ├── conversational.go          # Chat handling
│   │   ├── render-workspace.go        # Helm template rendering
│   │   └── *_test.go                  # Go tests
│   ├── realtime/                      # Real-time communications
│   │   ├── centrifugo.go              # Centrifugo WebSocket server
│   │   └── types/                     # Event type definitions
│   ├── workspace/                     # Workspace management (14 subdirs)
│   ├── diff/                          # Diff utilities (9 files)
│   │   └── *_test.go                  # Go tests
│   ├── param/
│   │   └── param.go                   # Environment config (API keys)
│   ├── embedding/
│   │   └── embeddings.go              # Voyage API
│   ├── persistence/                   # Data persistence layer
│   ├── recommendations/               # Recommendation engine
│   └── testhelpers/                   # Test helpers
│
├── helm-utils/                        # Helm CLI utilities
│   ├── render-exec.go                 # helm template execution
│   └── render-native.go               # Native SDK (disabled)
│
├── db/                                # Database schema
│   ├── database.yaml                  # Schemahero config
│   └── schema/tables/                 # 31 table definitions
│
├── hack/chartsmith-dev/               # Development environment
│   └── docker-compose.yml             # PostgreSQL + Centrifugo
│
├── test_chart/                        # Test chart directory
├── testdata/                          # Test fixtures
│   ├── charts/                        # .tgz test charts
│   ├── k8s/                           # K8s manifests
│   └── static-data/                   # CSV/SQL fixtures
│
├── ARCHITECTURE.md
├── CONTRIBUTING.md
├── go.mod
└── Makefile
```

---

## Key Functionality

### 1. Chat Message Flow (Actual)

```
User Input → ChatContainer.tsx:48-58
           → createChatMessageAction (server action)
           → lib/workspace/workspace.ts:164 (INSERT into workspace_chat)
           → lib/utils/queue.ts:9-20 (enqueueWork)
           → pg_notify() to Go worker
           → pkg/listener/listener.go (processQueue)
           → Handler (new_intent, new_plan, etc.)
           → pkg/llm/* (Anthropic API calls)
           → Centrifugo real-time update
           → useCentrifugo.ts:511 (WebSocket)
           → atoms/workspace.ts (Jotai state update)
           → ChatMessage.tsx re-render
```

### 2. Tool Calling (Go Backend Only)

**text_editor tool** (`pkg/llm/execute-action.go:510-532`):
- Commands: `view`, `str_replace`, `create`
- Used for file manipulation in Helm charts
- Includes fuzzy string matching (lines 238-435)

**Conversational tools** (`pkg/llm/conversational.go:99-128`):
- `latest_subchart_version` - ArtifactHub lookup
- `latest_kubernetes_version` - K8s version info

### 3. Real-Time Updates

- **Server**: Centrifugo (WebSocket pub/sub)
- **Client**: `hooks/useCentrifugo.ts:511`
- **Token**: `CENTRIFUGO_TOKEN_HMAC_SECRET` for JWT signing

---

## Authentication Flow (VS Code Extension)

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

## Test Infrastructure (Actual)

### TypeScript Unit Tests (Jest)
| File | Tests |
|------|-------|
| `atoms/__tests__/workspace.test.ts` | Jotai atom state |
| `components/__tests__/FileTree.test.ts` | Patch statistics |
| `hooks/__tests__/parseDiff.test.ts` | Diff parsing |

### Playwright E2E Tests
| File | Tests |
|------|-------|
| `tests/chat-scrolling.spec.ts` | Auto-scroll behavior |
| `tests/import-artifactory.spec.ts` | ArtifactHub import |
| `tests/login.spec.ts` | Login flow |
| `tests/upload-chart.spec.ts` | Chart upload workflow |

### Go Tests
| File | Tests |
|------|-------|
| `pkg/diff/apply_test.go` | Diff application |
| `pkg/diff/reconstruct_test.go` | Diff reconstruction |
| `pkg/diff/reconstruct_advanced_test.go` | Advanced scenarios |
| `pkg/listener/new-conversion_test.go` | K8s conversion |
| `pkg/llm/execute-action_test.go` | Action execution |
| `pkg/llm/parser_test.go` | Response parsing |
| `pkg/llm/string_replacement_test.go` | String replacement |

### Test Scripts (package.json)
```bash
npm run test        # Unit + E2E
npm run test:unit   # Jest only
npm run test:e2e    # Playwright only
```

---

## Environment Variables (Required)

### AI/LLM APIs
| Variable | TypeScript Location | Go Location |
|----------|---------------------|-------------|
| `ANTHROPIC_API_KEY` | `lib/llm/prompt-type.ts:22` | `pkg/param/param.go:17` |
| `GROQ_API_KEY` | - | `pkg/param/param.go:18` |
| `VOYAGE_API_KEY` | - | `pkg/param/param.go:19` |

### Database
| Variable | Purpose |
|----------|---------|
| `CHARTSMITH_PG_URI` | PostgreSQL connection |
| `DB_URI` | Fallback/alternative |

### Centrifugo (Real-time)
| Variable | Location |
|----------|----------|
| `CHARTSMITH_CENTRIFUGO_ADDRESS` | Go backend API |
| `CHARTSMITH_CENTRIFUGO_API_KEY` | Go backend auth |
| `CENTRIFUGO_TOKEN_HMAC_SECRET` | TS JWT signing |
| `NEXT_PUBLIC_CENTRIFUGO_ADDRESS` | Client WebSocket |

### Authentication
| Variable | Purpose |
|----------|---------|
| `HMAC_SECRET` | JWT session signing |
| `TOKEN_ENCRYPTION` | AES-256-GCM key |
| `GOOGLE_CLIENT_SECRET` | OAuth secret |
| `NEXT_PUBLIC_GOOGLE_CLIENT_ID` | OAuth client ID |

### Makefile Validation
The `check-env` target validates: ANTHROPIC_API_KEY, GROQ_API_KEY, VOYAGE_API_KEY, CHARTSMITH_PG_URI, CHARTSMITH_CENTRIFUGO_ADDRESS, CHARTSMITH_CENTRIFUGO_API_KEY, GOOGLE_CLIENT_ID, GOOGLE_CLIENT_SECRET

---

## Current Limitations (Confirmed)

1. **Single Provider Lock-in**: Only Anthropic Claude supported (Go backend)
2. **Custom Streaming**: PostgreSQL work queue + Centrifugo (not standard)
3. **State Management**: Jotai atoms, not AI SDK's `useChat`
4. **Maintenance Burden**: Custom implementations throughout
5. **No Tool Calling in Frontend**: All tools in Go backend
6. **No `/api/chat` route**: Messages go through workspace-specific routes

---

## Migration Opportunities

### Vercel AI SDK Benefits
| Current State | With AI SDK |
|--------------|-------------|
| Custom state in `atoms/workspace.ts` | `useChat` hook from `@ai-sdk/react` |
| PostgreSQL queue + Centrifugo streaming | Built-in `streamText` |
| Anthropic-only (`prompt-type.ts`) | Multi-provider via OpenRouter |
| `messagesAtom` manual management | SDK handles message state |
| No standardized tool calling (frontend) | Built-in `tool()` helper |

### Provider Flexibility
With Vercel AI SDK + OpenRouter:
- Access to 300+ models via single API key
- Easy provider switching without code changes
- Fallback provider support
- Cost optimization across providers

### Key Files to Modify (PR1)
| File | Change |
|------|--------|
| `lib/llm/prompt-type.ts` | Replace or remove `@anthropic-ai/sdk` |
| `components/ChatContainer.tsx` | Integrate `useChat` hook |
| `components/ChatMessage.tsx` | Parts-based rendering |
| `atoms/workspace.ts` | Reduce/simplify with SDK state |
| `package.json` | Add `ai`, `@ai-sdk/react`, `@openrouter/ai-sdk-provider` |

### New Files to Create (PR1)
| File | Purpose |
|------|---------|
| `lib/ai/provider.ts` | Provider factory |
| `lib/ai/models.ts` | Model definitions |
| `lib/ai/config.ts` | AI configuration |
| `app/api/chat/route.ts` | New chat API route |
| `components/chat/ProviderSelector.tsx` | Model selector UI |

---

## ⚠️ Important Compatibility Notes

### What PR1 Creates vs. What Exists

**PR1 creates a NEW parallel system** - it does NOT replace the existing Go-based flow:

| Aspect | Existing (Unchanged) | NEW (PR1 Creates) |
|--------|---------------------|-------------------|
| Chat route | `/api/workspace/[id]/message` | NEW `/api/chat` |
| LLM calls | Go backend (Anthropic SDK) | Next.js (AI SDK + OpenRouter) |
| Streaming | Centrifugo WebSocket | Data Stream protocol |
| State | Jotai atoms | `useChat` hook |
| Tools | 3 Go tools (Anthropic format) | AI SDK `tool()` format |

### PR2 Architecture Decision

PR2 adds a `validateChart` tool. However, the Go backend has **NO HTTP server**. Before implementing PR2, a decision must be made:

- **Option A**: Add HTTP endpoint to Go (new pattern)
- **Option B**: Use PostgreSQL queue pattern (async, consistent)
- **Option C**: Next.js spawns Go CLI command

See: `PRDs/PR2_ARCHITECTURE_DECISION_REQUIRED.md`

### Tool Format Clarification

| Tool Location | Format | Changed by PR1/PR2? |
|--------------|--------|---------------------|
| Go backend (`pkg/llm/`) | Anthropic-native JSON schema | NO - unchanged |
| NEW `/api/chat` route | Vercel AI SDK `tool()` | YES - PR2 adds validateChart |

The existing Go tools (`text_editor`, `latest_subchart_version`, `latest_kubernetes_version`) remain in Anthropic format and continue to work via the existing Go→Anthropic flow.

---

## References

- [Vercel AI SDK Docs](https://ai-sdk.dev/docs)
- [OpenRouter AI SDK Provider](https://github.com/OpenRouterTeam/ai-sdk-provider)
- [Chartsmith GitHub](https://github.com/replicatedhq/chartsmith)
- [Full Codebase Analysis](../docs/research/PR1_CODEBASE_ANALYSIS.md)
