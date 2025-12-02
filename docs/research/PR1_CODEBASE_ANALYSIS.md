# PR1 Codebase Analysis: Chartsmith Vercel AI SDK Migration

**Date**: 2025-12-02T02:53:22Z
**Git Commit**: e615830f7b0b60ecb817d762b9b4e4186f855d85
**Branch**: main
**Repository**: UnchartedTerritory/chartsmith

---

## Executive Summary

This document provides a comprehensive analysis of the Chartsmith codebase to support the Vercel AI SDK migration (PR1), addressing requirements from:
- **PR1_Product_PRD.md** - Functional requirements (US-1 through US-5, FR-1 through FR-5)
- **PR1_Tech_PRD.md** - Technical specifications
- **PR1_INSTRUCTIONS.md** - Implementation steps

Key findings:

1. **Anthropic SDK Usage**: Split between TypeScript frontend (1 file) and Go backend (10 files)
2. **Chat UI**: Custom React components using Jotai for state management
3. **Tools/Function Calling**: Implemented in Go with `text_editor` tool and 2 informational tools
4. **API Routes**: Next.js routes communicate with Go via PostgreSQL work queue (no direct HTTP)
5. **Tests**: Jest (unit), Playwright (E2E), Go testing framework
6. **Environment Variables**: `ANTHROPIC_API_KEY` required in both TS and Go

---

## 1. Anthropic SDK Usage

### TypeScript Frontend

**Single Import Location**:
- `chartsmith-app/lib/llm/prompt-type.ts:1`

```typescript
import Anthropic from '@anthropic-ai/sdk';
```

**Usage** (`prompt-type.ts:19-50`):
- Creates Anthropic client at line 21-23
- Calls `anthropic.messages.create()` at line 25-38
- Model: `claude-3-5-sonnet-20241022`
- Purpose: Classify user prompts as "plan" or "chat"

**Dependency** (`chartsmith-app/package.json:21`):
```json
"@anthropic-ai/sdk": "^0.39.0"
```

### Go Backend

**SDK Dependency** (`go.mod:8`):
```go
github.com/anthropics/anthropic-sdk-go v0.2.0-alpha.11
```

**Client Factory** (`pkg/llm/client.go:12-21`):
```go
func newAnthropicClient(ctx context.Context) (*anthropic.Client, error)
```

**Files Using Anthropic SDK** (10 files):

| File | Line | Purpose |
|------|------|---------|
| `pkg/llm/client.go` | 12-21 | Client factory |
| `pkg/llm/initial-plan.go` | 21-80 | Initial plan generation |
| `pkg/llm/plan.go` | 21-100 | Plan updates |
| `pkg/llm/execute-plan.go` | 14-60 | Execution plan generation |
| `pkg/llm/execute-action.go` | 437-673 | Action execution with tools |
| `pkg/llm/conversational.go` | 14-234 | Conversational chat |
| `pkg/llm/summarize.go` | 100-120 | Content summarization |
| `pkg/llm/expand.go` | 10-54 | Prompt expansion |
| `pkg/llm/cleanup-converted-values.go` | 13-90 | Values cleanup |
| `pkg/llm/convert-file.go` | 123-160 | File conversion (alternative) |

**Models Used**:
- `claude-3-7-sonnet-20250219` - Planning, conversational, summarization (8 functions)
- `claude-3-5-sonnet-20241022` - Action execution with tools, frontend classification

---

## 2. Chat UI Components & State Management

### Main Components

| Component | File | Line | Purpose |
|-----------|------|------|---------|
| `ChatContainer` | `components/ChatContainer.tsx` | 18 | Main chat interface |
| `ChatMessage` | `components/ChatMessage.tsx` | 72 | Standard message display |
| `NewChartChatMessage` | `components/NewChartChatMessage.tsx` | 69 | New chart message display |
| `PlanChatMessage` | `components/PlanChatMessage.tsx` | 38 | Plan message display |
| `PromptInput` | `components/PromptInput.tsx` | 16 | Reusable prompt input |
| `NewChartContent` | `components/NewChartContent.tsx` | 18 | New chart creation flow |
| `ScrollingContent` | `components/ScrollingContent.tsx` | 17 | Auto-scroll behavior |

### State Management (Jotai)

**Atoms** (`atoms/workspace.ts`):

| Atom | Line | Purpose |
|------|------|---------|
| `workspaceAtom` | 6 | Current workspace object |
| `messagesAtom` | 10 | Array of chat messages |
| `plansAtom` | 16 | Array of Plan objects |
| `rendersAtom` | 22 | Array of rendered workspaces |
| `conversionsAtom` | 34 | Array of conversions |
| `isRenderingAtom` | 259-261 | Computed loading state |

**Getter Atoms**:
- `messageByIdAtom` (lines 11-14)
- `planByIdAtom` (lines 17-20)
- `renderByIdAtom` (lines 23-26)
- `conversionByIdAtom` (lines 35-38)

**Update Handlers**:
- `handlePlanUpdatedAtom` (lines 132-147)
- `handleConversionUpdatedAtom` (lines 150-165)
- `handleConversionFileUpdatedAtom` (lines 167-198)

### Message Type (`components/types.ts:24-44`)

```typescript
interface Message {
  id: string;
  prompt: string;
  response?: string;
  isComplete: boolean;
  isIntentComplete?: boolean;
  isCanceled?: boolean;
  followupActions?: any[];
  responseRenderId?: string;
  responsePlanId?: string;
  responseConversionId?: string;
  responseRollbackToRevisionNumber?: number;
  revisionNumber?: number;
  createdAt?: Date;
}
```

### Chat Input Handling

**ChatContainer** (`ChatContainer.tsx:48-58`):
```typescript
const handleSubmitChat = async (e: React.FormEvent) => {
  // Calls createChatMessageAction server action
  // Adds new message to messagesAtom
  // Clears input
};
```

---

## 3. Tools/Function Calling

### Text Editor Tool (Primary)

**Definition** (`pkg/llm/execute-action.go:510-532`):

```go
tools := []anthropic.ToolParam{
    {
        Name: anthropic.F("text_editor_20241022"),
        InputSchema: anthropic.F(interface{}(map[string]interface{}{
            "type": "object",
            "properties": map[string]interface{}{
                "command": map[string]interface{}{
                    "type": "string",
                    "enum": []string{"view", "str_replace", "create"},
                },
                "path": map[string]interface{}{"type": "string"},
                "old_str": map[string]interface{}{"type": "string"},
                "new_str": map[string]interface{}{"type": "string"},
            },
        })),
    },
}
```

**Commands**:
- `view` (lines 603-609): Returns file content or error if not exists
- `str_replace` (lines 610-644): Replaces string with fuzzy matching
- `create` (lines 645-654): Creates new file

### Conversational Tools (`pkg/llm/conversational.go:99-128`)

| Tool | Line | Purpose |
|------|------|---------|
| `latest_subchart_version` | 100-113 | Returns latest subchart version from ArtifactHub |
| `latest_kubernetes_version` | 114-127 | Returns latest K8s version |

### Tool Use Loop Pattern (`execute-action.go:542-673`)

1. Send messages with tools to LLM
2. Stream response, accumulate message
3. Check for tool use blocks
4. Execute tool handlers
5. Package results as tool result blocks
6. Append as user message
7. Repeat until no tool calls

### String Replacement with Fuzzy Matching (`execute-action.go:238-435`)

- Exact match first (fast path)
- Fuzzy match with sliding window (slow path, 10s timeout)
- Minimum match length: 50 characters
- All operations logged to `str_replace_log` table

---

## 4. API Routes & Go Backend Interaction

### Communication Pattern

**IMPORTANT**: Next.js and Go do NOT communicate via HTTP. They use PostgreSQL as a message broker.

```
Next.js API → INSERT into work_queue → pg_notify() → Go Worker LISTEN
```

### Work Queue Implementation (`lib/utils/queue.ts:9-20`)

```typescript
export async function enqueueWork(channel: string, payload: object) {
  const id = uuidv4();
  await sql`INSERT INTO work_queue (id, channel, payload) VALUES (${id}, ${channel}, ${JSON.stringify(payload)})`;
  await sql`SELECT pg_notify(${channel}, ${id})`;
}
```

### Next.js API Routes

| Route | File | Methods | Purpose |
|-------|------|---------|---------|
| `/api/auth/status` | `app/api/auth/status/route.ts` | GET | Token validation |
| `/api/push` | `app/api/push/route.ts` | GET | Centrifugo token |
| `/api/config` | `app/api/config/route.ts` | GET | Public config |
| `/api/upload-chart` | `app/api/upload-chart/route.ts` | POST | Chart upload |
| `/api/workspace/[id]/message` | `app/api/workspace/[workspaceId]/message/route.ts` | POST | Create message |
| `/api/workspace/[id]/messages` | `app/api/workspace/[workspaceId]/messages/route.ts` | GET | List messages |
| `/api/workspace/[id]/plans` | `app/api/workspace/[workspaceId]/plans/route.ts` | GET | List plans |
| `/api/workspace/[id]/revision` | `app/api/workspace/[workspaceId]/revision/route.ts` | POST | Create revision |
| `/api/workspace/[id]/renders` | `app/api/workspace/[workspaceId]/renders/route.ts` | GET | List renders |
| `/api/workspace/[id]` | `app/api/workspace/[workspaceId]/route.ts` | GET | Get workspace |

### Go Worker Channels (`pkg/listener/start.go:13-119`)

| Channel | Concurrency | Handler |
|---------|-------------|---------|
| `new_intent` | 5 | Intent classification |
| `new_plan` | 5 | Plan creation |
| `new_converational` | 5 | Conversational response |
| `execute_plan` | 5 | Execute approved plan |
| `apply_plan` | 10 | Apply plan to files |
| `render_workspace` | 5 | Render Helm chart |
| `new_conversion` | 5 | K8s to Helm conversion |
| `new_summarize` | 5 | File summarization |

### Go Entry Points

| File | Purpose |
|------|---------|
| `main.go` | Entry point |
| `cmd/root.go` | Command registration |
| `cmd/run.go` | Worker startup |
| `pkg/listener/listener.go` | PostgreSQL LISTEN/NOTIFY |

---

## 5. Tests & Frameworks

### Test Inventory

#### TypeScript Unit Tests (Jest)

| File | Tests |
|------|-------|
| `atoms/__tests__/workspace.test.ts` | Jotai atom state management |
| `components/__tests__/FileTree.test.ts` | Patch statistics calculation |
| `hooks/__tests__/parseDiff.test.ts` | Diff parsing logic |

#### Playwright E2E Tests

| File | Tests |
|------|-------|
| `tests/chat-scrolling.spec.ts` | Chat auto-scrolling |
| `tests/import-artifactory.spec.ts` | ArtifactHub import |
| `tests/login.spec.ts` | Login flow |
| `tests/upload-chart.spec.ts` | Chart upload workflow |
| `tests/helpers.ts` | Shared test utilities |

#### Go Tests

| File | Tests |
|------|-------|
| `pkg/diff/apply_test.go` | Diff patch application |
| `pkg/diff/reconstruct_test.go` | Diff reconstruction |
| `pkg/diff/reconstruct_advanced_test.go` | Advanced diff scenarios |
| `pkg/listener/new-conversion_test.go` | K8s conversion ordering |
| `pkg/llm/execute-action_test.go` | LLM action execution |
| `pkg/llm/parser_test.go` | LLM response parsing |
| `pkg/llm/string_replacement_test.go` | String replacement |

### Test Configuration

**Jest** (`chartsmith-app/jest.config.ts`):
- Preset: `ts-jest`
- Environment: `node`
- Pattern: `**/__tests__/**/*.test.ts`

**Playwright** (`chartsmith-app/playwright.config.ts`):
- Browser: Chromium only
- Base URL: `http://localhost:3000`
- Retries: 2 on CI

### Test Scripts (`package.json`)

```json
"test": "npm run test:unit && npm run test:e2e",
"test:e2e": "playwright test",
"test:unit": "jest",
"test:watch": "jest --watch"
```

### Test Data

- `testdata/charts/` - Test Helm charts (.tgz)
- `testdata/k8s/` - Kubernetes manifests
- `testdata/static-data/` - CSV/SQL fixtures
- `test_chart/` - Full test chart directory

---

## 6. Environment Variables

### Required for AI/LLM

| Variable | TypeScript | Go | Purpose |
|----------|------------|-----|---------|
| `ANTHROPIC_API_KEY` | `lib/llm/prompt-type.ts:22` | `pkg/param/param.go:17` | Anthropic API key |
| `GROQ_API_KEY` | - | `pkg/param/param.go:18` | Groq API key |
| `VOYAGE_API_KEY` | - | `pkg/param/param.go:19` | Voyage embeddings |

### Database

| Variable | Location | Purpose |
|----------|----------|---------|
| `CHARTSMITH_PG_URI` | Both | PostgreSQL connection |
| `DB_URI` | Fallback | Alternative name |

### Centrifugo (Real-time)

| Variable | Location | Purpose |
|----------|----------|---------|
| `CHARTSMITH_CENTRIFUGO_ADDRESS` | Go | HTTP API endpoint |
| `CHARTSMITH_CENTRIFUGO_API_KEY` | Go | API authentication |
| `CENTRIFUGO_TOKEN_HMAC_SECRET` | TS | JWT signing secret |
| `NEXT_PUBLIC_CENTRIFUGO_ADDRESS` | TS | WebSocket endpoint |

### Authentication

| Variable | Purpose |
|----------|---------|
| `HMAC_SECRET` | JWT session signing |
| `TOKEN_ENCRYPTION` | AES-256-GCM key |
| `GOOGLE_CLIENT_SECRET` | OAuth secret |
| `NEXT_PUBLIC_GOOGLE_CLIENT_ID` | OAuth client ID |

### Development

| Variable | Purpose |
|----------|---------|
| `ENABLE_TEST_AUTH` | Server-side test auth |
| `NEXT_PUBLIC_ENABLE_TEST_AUTH` | Client-side test auth |
| `NODE_ENV` | Environment mode |

### Validation

Makefile `check-env` target validates required vars:
- ANTHROPIC_API_KEY
- GROQ_API_KEY
- VOYAGE_API_KEY
- CHARTSMITH_PG_URI
- CHARTSMITH_CENTRIFUGO_ADDRESS
- CHARTSMITH_CENTRIFUGO_API_KEY
- GOOGLE_CLIENT_ID
- GOOGLE_CLIENT_SECRET

---

## 7. Product PRD Requirement Mapping

Based on **PR1_Product_PRD.md**, here's how the current codebase maps to requirements:

### US-1: Chat Migration (Core)
| Requirement | Current Implementation |
|-------------|----------------------|
| Message display | `components/ChatMessage.tsx:226-372` |
| Streaming responses | Custom via Go backend + Centrifugo |
| Message history | `atoms/workspace.ts:10` (`messagesAtom`) |
| System prompts | `pkg/llm/system.go` (Go backend) |

### US-2: Provider Selection
| Requirement | Current State |
|-------------|--------------|
| Provider selector | **Does not exist** - must create |
| Multiple providers | **Single provider only** (Anthropic) |
| Provider persistence | N/A |

### US-3: Streaming Experience
| Requirement | Current Implementation |
|-------------|----------------------|
| Real-time streaming | Go workers → Centrifugo → `useCentrifugo.ts:511` |
| Loading indicator | `ChatMessage.tsx:238-256` ("thinking..." spinner) |
| Status display | `isRenderingAtom` at `atoms/workspace.ts:259` |

### US-4: Tool Calling Preservation
| Requirement | Current Implementation |
|-------------|----------------------|
| File context tools | `pkg/llm/execute-action.go:510-532` (Go) |
| Chart generation | `pkg/llm/plan.go`, `pkg/llm/execute-plan.go` |
| Tool results display | `PlanChatMessage.tsx:233-294` |

### US-5: Error Handling
| Requirement | Current Implementation |
|-------------|----------------------|
| Error display | `ChatMessage.tsx:257-260` (cancellation) |
| Retry option | Not explicitly implemented |
| Error state clear | Manual via new message |

### FR-2: Provider System (New for PR1)
**Must Create**:
- `lib/ai/provider.ts` - Provider factory
- `components/chat/ProviderSelector.tsx` - UI component
- `app/api/chat/route.ts` - New API route with `streamText`

### API Contract (FR: POST /api/chat)
**Current**: No unified `/api/chat` route. Messages go through:
- `app/api/workspace/[id]/message/route.ts` → PostgreSQL queue → Go worker

**After PR1**: New `/api/chat` route with:
```typescript
{ messages, provider, model } → streamText → OpenRouter
```

---

## 8. Migration Impact Analysis

### Files to Modify (Frontend)

| File | Current | After Migration |
|------|---------|-----------------|
| `lib/llm/prompt-type.ts` | `@anthropic-ai/sdk` | `ai` SDK or remove |
| `components/ChatContainer.tsx` | Custom state | `useChat` hook |
| `components/ChatMessage.tsx` | Custom rendering | Parts-based rendering |
| `atoms/workspace.ts` | Manual message state | SDK-managed state |
| `package.json` | `@anthropic-ai/sdk` | `ai`, `@ai-sdk/react`, `@openrouter/ai-sdk-provider` |

### Files to Create (Frontend)

| File | Purpose |
|------|---------|
| `lib/ai/provider.ts` | Provider factory |
| `lib/ai/models.ts` | Model definitions |
| `lib/ai/config.ts` | AI configuration |
| `app/api/chat/route.ts` | New chat API route |
| `components/chat/ProviderSelector.tsx` | Model selector UI |

### Go Backend Impact (PR1)

**Minimal changes for PR1**:
- Keep Go backend unchanged
- Frontend routes to OpenRouter
- Consider `promptType()` migration to frontend-only

### Key Considerations

1. **No HTTP between Next.js and Go**: Work queue pattern preserved
2. **State Management**: Replace custom Jotai atoms with `useChat` managed state
3. **Streaming**: Replace custom implementation with AI SDK's `streamText`
4. **Tools**: Frontend doesn't use tools; Go tools remain unchanged
5. **Real-time Updates**: Centrifugo integration unchanged

---

## 8. Recommendations for CURRENT_STATE_ANALYSIS.md

Update the document with these actual paths:

### Chat Components
```
- Main Chat: chartsmith-app/components/ChatContainer.tsx:18
- Message: chartsmith-app/components/ChatMessage.tsx:72
- Plan Message: chartsmith-app/components/PlanChatMessage.tsx:38
- Input: chartsmith-app/components/PromptInput.tsx:16
```

### Anthropic SDK Import
```
- TypeScript: chartsmith-app/lib/llm/prompt-type.ts:1
- Go Backend: pkg/llm/client.go:7-8 (10 files total)
```

### Test Files
```
Frontend:
- chartsmith-app/atoms/__tests__/workspace.test.ts
- chartsmith-app/components/__tests__/FileTree.test.ts
- chartsmith-app/hooks/__tests__/parseDiff.test.ts
- chartsmith-app/tests/*.spec.ts (4 E2E tests)

Backend:
- pkg/diff/*_test.go (3 files)
- pkg/listener/*_test.go (1 file)
- pkg/llm/*_test.go (3 files)
```

### API Routes
```
- chartsmith-app/app/api/auth/status/route.ts
- chartsmith-app/app/api/workspace/[workspaceId]/message/route.ts
- chartsmith-app/app/api/workspace/[workspaceId]/messages/route.ts
- chartsmith-app/app/api/workspace/[workspaceId]/plans/route.ts
- chartsmith-app/app/api/workspace/[workspaceId]/revision/route.ts
- chartsmith-app/app/api/workspace/[workspaceId]/renders/route.ts
- chartsmith-app/app/api/workspace/[workspaceId]/route.ts
```

### State Management
```
- Atoms: chartsmith-app/atoms/workspace.ts
- Framework: Jotai (not Redux, not Context)
```

---

## Appendix: File Reference Index

### Critical Files for Migration

```
chartsmith-app/
├── lib/
│   └── llm/
│       └── prompt-type.ts:1        # Only TS Anthropic import
├── components/
│   ├── ChatContainer.tsx:18        # Main chat component
│   ├── ChatMessage.tsx:72          # Message rendering
│   └── PlanChatMessage.tsx:38      # Plan display
├── atoms/
│   └── workspace.ts:6-261          # All state management
├── app/
│   └── api/
│       └── workspace/
│           └── [workspaceId]/
│               └── message/
│                   └── route.ts:35 # Chat message creation
└── package.json:21                 # @anthropic-ai/sdk dependency
```

### Go Backend (Reference Only for PR1)

```
pkg/llm/
├── client.go:12-21                 # Anthropic client factory
├── execute-action.go:437-673       # Tool execution loop
├── conversational.go:14-234        # Chat with tools
└── system.go                       # System prompts
```
