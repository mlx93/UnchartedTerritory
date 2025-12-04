# Chartsmith System Patterns

**Purpose**: Document system architecture, key technical decisions, design patterns, and component relationships.

---

## Architectural Decisions Summary

| # | Decision | Choice |
|---|----------|--------|
| 1 | Parallel vs. Replacement | Hybrid: New system parallel, legacy preserved |
| 2 | Go LLM Responsibility | None: Go becomes pure application logic after PR1.5 |
| 3 | Tool Migration | **4 tools** for feature parity |
| 4 | Go Endpoint Design | RPC-style under `/api/tools/` with POST |
| 5 | Go Communication | Non-streaming JSON request/response |
| 6 | PR Structure | 3 PRs: Foundation → Parity → Validation |
| 7 | System Prompt Location | Migrate to Node (`lib/ai/prompts.ts`) |
| 8 | Tool↔Go Communication | HTTP endpoints on port 8080 |
| 9 | Provider Priority | Direct APIs first (Anthropic → OpenAI → OpenRouter) |
| 10 | Default Model | Claude Sonnet 4 (`anthropic/claude-sonnet-4-20250514`) |
| 11 | All Tools Call Go | Consistent architecture, avoids bundling issues |

---

## Current Architecture (After PR1.5 ✅)

```
┌─────────────────────────────────────────────────────────────────┐
│                         USER                                     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    REACT CHAT UI                                 │
│                                                                  │
│  PR1 (parallel system):                                          │
│  - AIChat component with useChat hook                            │
│  - AIMessageList with parts-based rendering                      │
│  - ProviderSelector for model choice                             │
│  - Test page at /test-ai-chat                                    │
│                                                                  │
│  EXISTING (preserved):                                           │
│  - ChatContainer with Jotai state                                │
│  - Centrifugo WebSocket streaming                                │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼ POST /api/chat (AI SDK streaming)
┌─────────────────────────────────────────────────────────────────┐
│                    NODE / AI SDK CORE                            │
│                    (All LLM "thinking")                          │
│                                                                  │
│  lib/ai/provider.ts     - Provider factory (getModel)            │
│  lib/ai/models.ts       - Model definitions                      │
│  lib/ai/config.ts       - System prompt, config                  │
│  lib/ai/llmClient.ts    - Shared client utilities ✅             │
│  lib/ai/prompts.ts      - System prompts ✅                      │
│                                                                  │
│  Tools (4 total - ALL call Go HTTP):                             │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ getChartContext         → POST /api/tools/context            ││
│  │ textEditor              → POST /api/tools/editor             ││
│  │ latestSubchartVersion   → POST /api/tools/versions/subchart  ││
│  │ latestKubernetesVersion → POST /api/tools/versions/kubernetes││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
│  PR2 Addition:                                                   │
│  - validateChart           → POST /api/validate                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼ HTTP (JSON, non-streaming)
┌─────────────────────────────────────────────────────────────────┐
│                    GO BACKEND (pkg/api/)                         │
│                    (Pure application logic - NO LLM)             │
│                                                                  │
│  pkg/api/server.go              - HTTP server on :8080 ✅        │
│  pkg/api/errors.go              - Standardized error responses ✅│
│  pkg/api/handlers/response.go   - Handler error helpers ✅       │
│  pkg/api/handlers/context.go    - Chart context ✅               │
│  pkg/api/handlers/editor.go     - File operations ✅             │
│  pkg/api/handlers/versions.go   - Version lookups ✅             │
│  pkg/validation/*               - Validation pipeline (PR2)      │
│                                                                  │
│  EXISTING (preserved):                                           │
│  - PostgreSQL LISTEN/NOTIFY queue                                │
│  - pkg/llm/* (handles intent detection via Groq)                 │
│  - pkg/listener/* (queue handlers)                               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    DATA LAYER                                    │
│                                                                  │
│  - PostgreSQL (workspaces, charts, revisions)                    │
│  - File system (chart files)                                     │
│  - Centrifugo (real-time updates via pg_notify)                  │
│  - External APIs (ArtifactHub, Kubernetes releases)              │
└─────────────────────────────────────────────────────────────────┘
```

---

## Design Patterns (Implemented)

### Tool Factory Pattern (PR1.5 ✅)

```typescript
// lib/ai/tools/index.ts
export function createTools(
  authHeader: string | undefined,
  workspaceId: string,
  revisionNumber: number
) {
  return {
    getChartContext: createGetChartContextTool(workspaceId, revisionNumber, authHeader || ''),
    textEditor: createTextEditorTool(authHeader, workspaceId, revisionNumber),
    latestSubchartVersion: createLatestSubchartVersionTool(authHeader),
    latestKubernetesVersion: createLatestKubernetesVersionTool(authHeader),
  };
}
```

### AI SDK v5 Tool Definition Pattern (PR1.5 ✅)

```typescript
// lib/ai/tools/textEditor.ts
import { tool } from 'ai';
import { z } from 'zod';

export function createTextEditorTool(authHeader: string | undefined, workspaceId: string, revisionNumber: number) {
  return tool({
    description: 'View, edit, or create files in the chart',
    inputSchema: z.object({  // NOTE: inputSchema (not parameters) in AI SDK v5
      command: z.enum(['view', 'create', 'str_replace']),
      path: z.string(),
      content: z.string().optional(),
      oldStr: z.string().optional(),
      newStr: z.string().optional(),
    }),
    execute: async (params: { command: string; path: string; ... }) => {
      return callGoEndpoint('/api/tools/editor', { ...params, workspaceId, revisionNumber }, authHeader);
    },
  });
}
```

### Shared HTTP Utility Pattern (PR1.5 ✅)

```typescript
// lib/ai/tools/utils.ts
const GO_BACKEND_URL = process.env.GO_BACKEND_URL || 'http://localhost:8080';

export async function callGoEndpoint<T>(
  endpoint: string, 
  body: object,
  authHeader: string | undefined
): Promise<T> {
  const headers: Record<string, string> = { 'Content-Type': 'application/json' };
  if (authHeader) headers['Authorization'] = authHeader;
  
  const response = await fetch(GO_BACKEND_URL + endpoint, {
    method: 'POST',
    headers,
    body: JSON.stringify(body),
  });
  
  const data = await response.json();
  
  if (!response.ok || data.success === false) {
    throw new Error(data.message || 'Go endpoint failed');
  }
  
  return data as T;
}
```

### Go Handler Pattern (PR1.5 ✅)

```go
// pkg/api/handlers/versions.go
func GetKubernetesVersion(w http.ResponseWriter, r *http.Request) {
    var req KubernetesVersionRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        writeBadRequest(w, "Invalid request body")
        return
    }

    field := req.SemverField
    if field == "" {
        field = "patch"
    }

    var version string
    switch field {
    case "major":
        version = "1"
    case "minor":
        version = "1.32"
    default:
        version = "1.32.1"
    }

    writeJSON(w, http.StatusOK, KubernetesVersionResponse{
        Success: true,
        Version: version,
        Field:   field,
    })
}
```

### Error Response Contract (Go → TypeScript)

```json
{
  "success": false,
  "message": "Human-readable error description",
  "code": "ERROR_CODE"
}
```

| Code | HTTP Status | Meaning |
|------|-------------|---------|
| `VALIDATION_ERROR` | 400 | Invalid request parameters |
| `NOT_FOUND` | 404 | Resource doesn't exist |
| `INTERNAL_ERROR` | 500 | Server error |
| `EXTERNAL_API_ERROR` | 502 | External service failure |

---

## Go HTTP Server Setup (PR1.5 ✅)

```go
// pkg/api/server.go
func StartHTTPServer(ctx context.Context, port string) error {
    mux := http.NewServeMux()
    
    // Register tool endpoints
    mux.HandleFunc("POST /api/tools/context", handlers.GetChartContext)
    mux.HandleFunc("POST /api/tools/editor", handlers.TextEditor)
    mux.HandleFunc("POST /api/tools/versions/subchart", handlers.GetSubchartVersion)
    mux.HandleFunc("POST /api/tools/versions/kubernetes", handlers.GetKubernetesVersion)
    
    // Health check
    mux.HandleFunc("GET /health", func(w http.ResponseWriter, r *http.Request) {
        WriteJSON(w, http.StatusOK, map[string]interface{}{
            "success": true,
            "status":  "healthy",
        })
    })
    
    server := &http.Server{
        Addr:    ":" + port,
        Handler: loggingMiddleware(mux),
    }
    
    // Graceful shutdown
    go func() {
        <-ctx.Done()
        server.Shutdown(context.Background())
    }()
    
    return server.ListenAndServe()
}
```

### Server Startup (cmd/run.go)

```go
// Start HTTP server in goroutine alongside existing queue listener
go func() {
    logger.Info("Starting HTTP server for AI SDK tools on port 8080")
    if err := api.StartHTTPServer(ctx, "8080"); err != nil && err != http.ErrServerClosed {
        logger.Error("HTTP server error", zap.Error(err))
    }
}()

// Existing queue listener continues
if err := listener.StartListeners(ctx); err != nil {
    return fmt.Errorf("failed to start listeners: %w", err)
}
```

---

## File Structure (After PR1.5)

### TypeScript AI Module

```
chartsmith-app/
└── lib/
    └── ai/
        ├── config.ts             # System prompt, config (PR1)
        ├── models.ts             # Model definitions (PR1)
        ├── provider.ts           # Provider factory (PR1)
        ├── index.ts              # Exports (PR1)
        ├── llmClient.ts          # Shared client (PR1.5) ✅
        ├── prompts.ts            # System prompts (PR1.5) ✅
        ├── __tests__/
        │   ├── provider.test.ts
        │   ├── config.test.ts
        │   └── integration/
        │       └── tools.test.ts # Integration tests (PR1.5) ✅
        └── tools/                # PR1.5 ✅
            ├── utils.ts          # callGoEndpoint helper
            ├── index.ts          # Tool factory
            ├── getChartContext.ts
            ├── textEditor.ts
            ├── latestSubchartVersion.ts
            ├── latestKubernetesVersion.ts
            └── validateChart.ts  # PR2
```

### Go API Module

```
pkg/
└── api/
    ├── server.go                 # HTTP server on :8080 ✅
    ├── errors.go                 # Error response utilities ✅
    └── handlers/
        ├── response.go           # Handler error helpers ✅
        ├── context.go            # getChartContext endpoint ✅
        ├── editor.go             # textEditor endpoint ✅
        ├── versions.go           # Version endpoints ✅
        └── validate.go           # PR2

pkg/
└── validation/                   # PR2
    ├── pipeline.go
    ├── helm.go
    ├── kubescore.go
    ├── parser.go
    └── types.go
```

---

## Security Patterns

### API Key Protection
- All API keys server-side only
- Never import provider packages in client components
- Environment variables not exposed to client bundle
- Direct API keys preferred over OpenRouter for security

### Path Validation (Go - PR2)
1. Apply `filepath.Clean()` to normalize path
2. Check for ".." components - reject if present
3. Resolve to absolute path
4. Verify path is within allowed base directory
5. Confirm Chart.yaml exists at resolved path

### Command Injection Prevention (PR2)
- Never construct shell commands with string concatenation
- Always use `exec.Command` with separate argument array
- chartPath passed as single argument, not interpolated

### Authentication Flow (PR1.5 ✅)
- Auth header captured in route.ts
- Passed to tools via closure in createTools()
- Tools forward auth to Go endpoints via callGoEndpoint()
- Go validates workspace access

---

## Performance Targets

| Operation | Target Latency |
|-----------|----------------|
| Tool execution | < 500ms (excluding LLM) |
| HTTP round-trip to Go | < 100ms |
| File operation (view) | < 200ms |
| File operation (edit) | < 500ms |
| Validation pipeline | < 30s |

---

*This document captures the technical patterns and architecture for the migration. Last updated: Dec 4, 2025 (PR1.5 complete)*
