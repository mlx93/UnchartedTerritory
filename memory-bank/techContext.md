# Chartsmith Technical Context

**Purpose**: Document technologies used, development setup, technical constraints, and dependencies.

---

## Technology Stack

### Frontend

| Technology | Version | Purpose |
|------------|---------|---------|
| Next.js | Latest | React framework, SSR, API routes |
| React | 18+ | UI components |
| TypeScript | 5.x | Type safety |
| Jotai | - | State management (existing) |
| Tailwind CSS | - | Styling |

### Backend

| Technology | Version | Purpose |
|------------|---------|---------|
| Go | 1.22+ | Worker processes, tool execution |
| PostgreSQL | 15+ | Primary database |
| Centrifugo | - | Real-time WebSocket server |

### AI/LLM

| Technology | Purpose |
|------------|---------|
| Vercel AI SDK (`ai`) | Core LLM integration |
| AI SDK React (`@ai-sdk/react`) | React hooks (useChat) |
| OpenRouter Provider | Multi-model access |

---

## New Dependencies (Migration)

### To Install

```json
{
  "dependencies": {
    "ai": "^4.0.0",
    "@ai-sdk/react": "^1.0.0",
    "@openrouter/ai-sdk-provider": "^0.4.0",
    "zod": "^3.22.0"
  }
}
```

### To Remove

```json
{
  "dependencies": {
    "@anthropic-ai/sdk": "^0.39.0"  // REMOVE
  }
}
```

### Retained

All existing Next.js, React, and UI component libraries remain unchanged.

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
# Terminal 1: Go worker
make run-worker

# Terminal 2: Next.js dev server
cd chartsmith-app
npm run dev
```

### Test Commands

```bash
# TypeScript unit tests (Jest)
npm run test:unit

# E2E tests (Playwright)
npm run test:e2e

# Go tests
go test ./pkg/... -v
```

---

## Environment Variables

### Required for AI SDK

```env
# OpenRouter API Key (required for PR1+)
OPENROUTER_API_KEY=sk-or-v1-xxxxx

# Optional fallback
OPENAI_API_KEY=sk-xxxxx

# Configuration
DEFAULT_AI_PROVIDER=openai
DEFAULT_AI_MODEL=openai/gpt-4o

# Go backend URL (PR1.5+)
GO_BACKEND_URL=http://localhost:8080
```

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

# Existing AI (Go backend - unchanged)
ANTHROPIC_API_KEY=xxx  # Still used by Go until PR1.5
GROQ_API_KEY=xxx       # Used by Go for intent detection
VOYAGE_API_KEY=xxx     # Used by Go for embeddings
```

---

## Project Structure

```
chartsmith/
├── chartsmith-app/           # Next.js frontend
│   ├── app/
│   │   ├── api/              # API routes
│   │   │   └── chat/         # NEW (PR1)
│   │   └── workspace/        # Workspace pages
│   ├── atoms/                # Jotai state
│   ├── components/           # React components
│   ├── hooks/                # Custom hooks
│   ├── lib/
│   │   ├── ai/               # NEW (PR1/PR1.5)
│   │   └── llm/              # LEGACY (to be removed)
│   └── tests/                # Playwright E2E
│
├── chartsmith-extension/     # VS Code Extension
│
├── cmd/                      # Go entry points
│   ├── run.go                # Worker startup
│   └── main.go               # Entry point
│
├── pkg/                      # Go packages
│   ├── api/                  # NEW (PR1.5) - HTTP handlers
│   ├── llm/                  # LEGACY - Anthropic integration
│   ├── listener/             # PostgreSQL queue handlers
│   ├── realtime/             # Centrifugo integration
│   ├── validation/           # NEW (PR2)
│   └── workspace/            # Workspace management
│
├── db/                       # Database schema
│   └── schema/tables/        # 31 table definitions
│
└── hack/chartsmith-dev/      # Development environment
    └── docker-compose.yml
```

---

## Technical Constraints

### Architecture Constraints

1. **No new databases**: Only PostgreSQL + Centrifugo (per ARCHITECTURE.md)
2. **Server actions over API routes**: Follow existing patterns where possible
3. **SSR preferred**: Avoid "Loading..." states (per chartsmith-app/ARCHITECTURE.md)
4. **Jotai state**: Components subscribe to atoms, no callback passing

### Security Constraints

1. API keys must stay server-side
2. All LLM calls through server-side routes
3. Path validation to prevent traversal attacks
4. No arbitrary code execution from validation tools

### Performance Constraints

1. Streaming latency: < 200ms time-to-first-token
2. UI responsiveness: < 100ms for user interactions
3. Bundle size increase: < 50KB from AI SDK packages
4. Validation timeout: 30 seconds maximum

---

## External APIs

### OpenRouter (NEW)

- **Purpose**: Multi-provider LLM access
- **Models**: 300+ models via single API
- **Used by**: PR1+ for all LLM calls
- **Documentation**: [openrouter.ai/docs](https://openrouter.ai/docs)

### ArtifactHub (Existing)

- **Purpose**: Helm chart repository search
- **Used by**: `latestSubchartVersion` tool
- **Endpoint**: `https://artifacthub.io/api/v1/`

### Anthropic (Existing - Go only)

- **Purpose**: LLM calls from Go backend
- **Models**: claude-3-7-sonnet, claude-3-5-sonnet
- **Status**: Remains for Go until PR1.5 deprecation

### Groq (Existing - Go only)

- **Purpose**: Intent detection, feedback, file conversion
- **Models**: llama-3.3-70b-versatile
- **Status**: Unchanged by migration

### Voyage (Existing - Go only)

- **Purpose**: Embeddings
- **Status**: Unchanged by migration

---

## Go HTTP Server (PR1.5)

### Configuration

- **Port**: 8080
- **Router**: net/http.ServeMux (Go 1.22+ pattern matching)
- **Start**: Goroutine alongside existing PostgreSQL listener

### Routes

```
POST  /api/tools/charts/create    → handlers.CreateChart
GET   /api/tools/charts/{id}      → handlers.GetChartContext
PATCH /api/tools/charts/{id}      → handlers.UpdateChart
POST  /api/tools/editor           → handlers.TextEditor
POST  /api/tools/versions/subchart   → handlers.GetSubchartVersion
POST  /api/tools/versions/kubernetes → handlers.GetKubernetesVersion
POST  /api/validate               → handlers.Validate (PR2)
```

---

## Existing Go Functions to Reuse

| Function | Package | Purpose |
|----------|---------|---------|
| `GetWorkspace` | workspace | Load workspace with all related data |
| `CreateRevision` | workspace | Create new revision with copy-on-write |
| `ListFiles` | workspace | Get file list for revision |
| `GetFile` | workspace | Get single file content |
| `SetFileContentPending` | workspace | Update file content |
| `CreateChart` | workspace | Create empty chart |
| `AddFileToChart` | workspace | Add file to chart |
| `ApplyPatch` | diff | Apply unified diff |
| `PerformStringReplacement` | llm | String replacement with fuzzy matching |
| `GetLatestSubchartVersion` | recommendations | ArtifactHub lookup |
| `EnqueueWork` | persistence | Trigger pg_notify |
| `ListUserIDsForWorkspace` | workspace | Get workspace owners |

---

## Testing Infrastructure

### TypeScript Unit Tests (Jest)

```
atoms/__tests__/workspace.test.ts      # Jotai atom state
components/__tests__/FileTree.test.ts  # Patch statistics
hooks/__tests__/parseDiff.test.ts      # Diff parsing
```

### Playwright E2E Tests

```
tests/chat-scrolling.spec.ts       # Auto-scroll behavior
tests/import-artifactory.spec.ts   # ArtifactHub import
tests/login.spec.ts                # Login flow
tests/upload-chart.spec.ts         # Chart upload
```

### Go Tests

```
pkg/diff/apply_test.go                    # Diff application
pkg/diff/reconstruct_test.go              # Diff reconstruction
pkg/listener/new-conversion_test.go       # K8s conversion
pkg/llm/execute-action_test.go            # Action execution
pkg/llm/parser_test.go                    # Response parsing
pkg/llm/string_replacement_test.go        # String replacement
```

---

## Browser Compatibility

- Chrome (latest 2 versions)
- Firefox (latest 2 versions)
- Safari (latest 2 versions)
- Edge (latest 2 versions)

---

*This document captures the technical environment for the Chartsmith migration.*

