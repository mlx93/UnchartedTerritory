# Chartsmith Technical Context

**Purpose**: Document technologies used, development setup, technical constraints, and dependencies.

---

## Technology Stack

### Frontend

| Technology | Version | Purpose |
|------------|---------|---------|
| Next.js | 15.3.0 | React framework, SSR, API routes |
| React | 18+ | UI components |
| TypeScript | 5.x | Type safety |
| Jotai | - | State management (existing) |
| Tailwind CSS | - | Styling |

### Backend

| Technology | Version | Purpose |
|------------|---------|---------|
| Go | 1.24.4 | Worker processes, tool execution HTTP server |
| PostgreSQL | 15+ | Primary database |
| Centrifugo | - | Real-time WebSocket server |

### AI/LLM (Updated for PR1.5)

| Technology | Version | Purpose |
|------------|---------|---------|
| Vercel AI SDK (`ai`) | ^5.0.106 | Core LLM integration |
| AI SDK React (`@ai-sdk/react`) | ^2.0.106 | React hooks (useChat) |
| AI SDK OpenAI (`@ai-sdk/openai`) | - | Direct OpenAI provider |
| AI SDK Anthropic (`@ai-sdk/anthropic`) | - | Direct Anthropic provider |
| OpenRouter Provider | ^0.4.0 | Multi-model fallback |
| Zod | ^3.22.0 | Schema validation (tools) |

---

## Dependencies (After PR1.5)

### Installed

```json
{
  "dependencies": {
    "ai": "^5.0.106",
    "@ai-sdk/react": "^2.0.106",
    "@ai-sdk/openai": "installed",
    "@ai-sdk/anthropic": "installed",
    "@openrouter/ai-sdk-provider": "^0.4.0",
    "zod": "^3.22.0"
  }
}
```

### Removed (PR1.5 ✅)

```json
{
  "dependencies": {
    "@anthropic-ai/sdk": "REMOVED"  // Fully removed from package.json AND package-lock.json
  }
}
```

---

## AI SDK v5 Specifics

### API Differences from v4 Documentation

| v4 Pattern | v5 Actual |
|------------|-----------|
| `Message` type | `UIMessage` type |
| `api` option in useChat | `TextStreamChatTransport` |
| `handleSubmit()` | `sendMessage()` |
| `toDataStreamResponse()` | `toTextStreamResponse()` |
| `parameters` in tool() | `inputSchema` in tool() |
| Input managed by useChat | Input managed separately |

### Tool Definition Pattern (v5 - PR1.5 ✅)

```typescript
import { tool } from 'ai';
import { z } from 'zod';

const myTool = tool({
  description: 'Tool description',
  inputSchema: z.object({  // NOTE: inputSchema, not parameters
    param1: z.string(),
    param2: z.enum(['a', 'b']).optional(),
  }),
  execute: async (params: { param1: string; param2?: string }) => {
    // Tool logic - explicit typing required
    return result;
  },
});
```

### Streaming Pattern (v5)

```typescript
import { streamText } from 'ai';

const result = streamText({
  model: getModel(provider),
  messages: convertedMessages,
  system: SYSTEM_PROMPT,
  tools: createTools(authHeader, workspaceId, revisionNumber),  // PR1.5
});

return result.toTextStreamResponse();  // NOT toDataStreamResponse
```

---

## Development Setup

### Prerequisites

1. Node.js 18+
2. Go 1.22+
3. PostgreSQL 15+
4. helm CLI v3.x (for PR2)
5. kube-score CLI v1.16+ (for PR2)

### Local Services (Docker Compose)

```bash
# Start PostgreSQL + Centrifugo
cd hack/chartsmith-dev
docker-compose up -d
```

### Start Applications

```bash
# Terminal 1: Go worker (now includes HTTP server on port 8080)
make run-worker

# Terminal 2: Next.js dev server
cd chartsmith-app
npm run dev
```

### Test Commands

```bash
# TypeScript unit tests (Jest)
npm run test:unit

# Integration tests (PR1.5)
npx jest lib/ai/__tests__/integration/tools.test.ts

# E2E tests (Playwright)
npm run test:e2e

# Go tests
go test ./pkg/... -v
```

---

## Environment Variables

### Required for AI SDK

```env
# Direct API Keys (preferred - avoids OpenRouter limits)
ANTHROPIC_API_KEY=sk-ant-xxx
OPENAI_API_KEY=sk-xxx

# Fallback (if direct keys unavailable)
OPENROUTER_API_KEY=sk-or-v1-xxxxx

# Configuration (defaults)
DEFAULT_AI_PROVIDER=anthropic
DEFAULT_AI_MODEL=anthropic/claude-sonnet-4-20250514

# Go backend URL (PR1.5)
GO_BACKEND_URL=http://localhost:8080
```

### Provider Priority

1. `ANTHROPIC_API_KEY` → Direct Anthropic API (preferred)
2. `OPENAI_API_KEY` → Direct OpenAI API
3. `OPENROUTER_API_KEY` → OpenRouter (fallback)

### Existing (Preserved)

```env
# Database
CHARTSMITH_PG_URI=postgresql://...
DB_URI=postgresql://...

# Centrifugo (Real-time)
CHARTSMITH_CENTRIFUGO_ADDRESS=http://localhost:8000
CHARTSMITH_CENTRIFUGO_API_KEY=xxx
CENTRIFUGO_TOKEN_HMAC_SECRET=xxx
NEXT_PUBLIC_CENTRIFUGO_ADDRESS=ws://localhost:8000/connection/websocket

# Authentication
HMAC_SECRET=xxx
TOKEN_ENCRYPTION=xxx  # AES-256-GCM key
GOOGLE_CLIENT_SECRET=xxx
NEXT_PUBLIC_GOOGLE_CLIENT_ID=xxx

# Go AI APIs (still used by Go worker for non-chat features)
GROQ_API_KEY=xxx       # Used by Go for intent detection
VOYAGE_API_KEY=xxx     # Used by Go for embeddings
```

---

## Project Structure (After PR3.0-3.3)

```
chartsmith/
├── chartsmith-app/           # Next.js frontend
│   ├── app/
│   │   ├── api/
│   │   │   └── chat/         # AI SDK route (PR1)
│   │   │       ├── route.ts  # Modified for tools (PR1.5)
│   │   │       └── __tests__/
│   │   ├── test-ai-chat/     # Test page (PR1)
│   │   └── workspace/        # Existing workspace pages
│   ├── components/
│   │   └── chat/             # AI SDK components (PR1)
│   │       ├── AIChat.tsx
│   │       ├── AIMessageList.tsx
│   │       └── ProviderSelector.tsx
│   ├── lib/
│   │   ├── ai/               # AI SDK core
│   │   │   ├── config.ts     # PR1
│   │   │   ├── models.ts     # PR1
│   │   │   ├── provider.ts   # PR1
│   │   │   ├── llmClient.ts  # PR1.5 ✅
│   │   ├── prompts.ts    # PR1.5 ✅
│   │   ├── intent.ts     # Intent classification ✅ (PR3.0)
│   │   ├── plan.ts        # Plan creation ✅ (PR3.0)
│   │   ├── conversion.ts  # K8s conversion ✅ (PR3.0)
│   │   ├── __tests__/
│   │   │   └── integration/
│   │   │       └── tools.test.ts  # PR1.5 ✅
│   │   └── tools/        # PR1.5 + PR3.0 ✅
│   │       ├── utils.ts
│   │       ├── index.ts
│   │       ├── getChartContext.ts
│   │       ├── textEditor.ts
│   │       ├── latestSubchartVersion.ts
│   │       ├── latestKubernetesVersion.ts
│   │       ├── convertK8s.ts        # PR3.0 ✅
│   │       ├── toolInterceptor.ts    # PR3.0 ✅
│   │       └── bufferedTools.ts     # PR3.0 ✅
│   │   └── llm/              # Empty (prompt-type.ts deleted)
│   └── tests/                # Playwright E2E
│
├── cmd/                      # Go entry points
│   ├── run.go                # Modified to start HTTP server (PR1.5)
│   └── main.go
│
├── pkg/                      # Go packages
│   ├── api/                  # PR1.5 + PR3.0 ✅ - HTTP handlers
│   │   ├── server.go
│   │   ├── errors.go
│   │   └── handlers/
│   │       ├── response.go
│   │       ├── context.go
│   │       ├── editor.go
│   │       ├── versions.go
│   │       ├── intent.go      # PR3.0 ✅
│   │       ├── plan.go         # PR3.0 ✅
│   │       └── conversion.go   # PR3.0 ✅
│   ├── llm/                  # Existing - Groq integration
│   ├── listener/             # PostgreSQL queue handlers
│   ├── realtime/             # Centrifugo integration
│   ├── validation/           # PR2 - Validation pipeline
│   └── workspace/            # Workspace management
│
└── hack/chartsmith-dev/      # Development environment
    └── docker-compose.yml
```

---

## Go HTTP Server (PR1.5 ✅)

### Configuration

- **Port**: 8080
- **Router**: net/http.ServeMux (Go 1.22+ pattern matching)
- **Start**: Goroutine alongside existing PostgreSQL listener

### Routes

```
GET   /health                                    → Health check
POST  /api/tools/context                         → handlers.GetChartContext ✅
POST  /api/tools/editor                          → handlers.TextEditor ✅
POST  /api/tools/versions/subchart               → handlers.GetSubchartVersion ✅
POST  /api/tools/versions/kubernetes             → handlers.GetKubernetesVersion ✅
POST  /api/intent/classify                       → handlers.ClassifyIntent ✅ (PR3.0)
POST  /api/plan/create-from-tools                → handlers.CreatePlanFromToolCalls ✅ (PR3.0)
POST  /api/plan/publish-update                   → handlers.PublishPlanUpdate ✅ (PR3.0)
POST  /api/plan/update-action-file-status        → handlers.UpdateActionFileStatus ✅ (PR3.2)
POST  /api/conversion/start                      → handlers.StartConversion ✅ (PR3.0)
POST  /api/validate                              → handlers.Validate (PR2)
```

---

## Existing Go Functions Reused (PR1.5)

| Function | Package | Used In |
|----------|---------|---------|
| `workspace.ListCharts` | workspace | context.go, editor.go |
| `workspace.AddFileToChart` | workspace | editor.go |
| `workspace.SetFileContentPending` | workspace | editor.go |
| `llm.PerformStringReplacement` | llm | editor.go |
| `recommendations.GetLatestSubchartVersion` | recommendations | versions.go |

---

## Testing Infrastructure (Updated PR1.5)

### TypeScript Tests

```
lib/ai/__tests__/provider.test.ts              # PR1 - Provider factory
lib/ai/__tests__/config.test.ts                # PR1 - Config
lib/ai/__tests__/integration/tools.test.ts     # PR1.5 ✅ - 22 integration tests
lib/chat/__tests__/messageMapper.test.ts       # PR2.0 ✅ - 33 unit tests
app/api/chat/__tests__/route.test.ts          # PR1 - API route
lib/__tests__/ai-mock-utils.ts                 # Existing - AI SDK mocks
```

### Go Tests

```
pkg/diff/apply_test.go                    # Diff application
pkg/llm/execute-action_test.go            # Action execution
pkg/llm/string_replacement_test.go        # String replacement
```

---

## Technical Constraints

### Architecture Constraints

1. **No new databases**: Only PostgreSQL + Centrifugo
2. **Server actions over API routes**: Follow existing patterns
3. **SSR preferred**: Avoid "Loading..." states
4. **Jotai state**: Components subscribe to atoms

### Security Constraints

1. API keys must stay server-side
2. All LLM calls through server-side routes
3. Path validation to prevent traversal attacks (PR2)
4. No arbitrary code execution from validation tools

### Next.js Bundling Constraint (Discovered PR1.5)

**Issue**: Importing `getWorkspace()` in TypeScript tools caused bundling errors because `pg` module uses Node.js-only modules (`dns`, `net`, `tls`).

**Solution**: All 4 tools call Go HTTP endpoints. Avoids bundling issues and creates consistent architecture.

---

## External APIs

### OpenRouter (via Provider Factory)
- **Purpose**: Multi-provider fallback
- **Models**: 300+ models via single API

### Anthropic (Direct + Go)
- **Purpose**: Primary LLM provider
- **Default Model**: `claude-sonnet-4-20250514`

### OpenAI (Direct)
- **Purpose**: Secondary LLM provider

### ArtifactHub (via Go)
- **Purpose**: Helm chart repository search
- **Used by**: `latestSubchartVersion` tool

### Groq (Go only)
- **Purpose**: Intent detection, feedback
- **Status**: Unchanged by migration

---

*This document captures the technical environment for the Chartsmith migration. Last updated: Dec 6, 2025 (PR3.0-3.3 complete)*
