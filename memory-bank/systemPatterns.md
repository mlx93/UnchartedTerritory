# Chartsmith System Patterns

**Purpose**: Document system architecture, key technical decisions, design patterns, and component relationships.

---

## Architectural Decisions Summary

| # | Decision | Choice |
|---|----------|--------|
| 1 | Parallel vs. Replacement | Hybrid: New system primary, legacy deprecated |
| 2 | Go LLM Responsibility | None: Go becomes pure application logic |
| 3 | Tool Migration | Port ALL existing tools for full feature parity (6 tools) |
| 4 | Go Endpoint Design | RPC-style under `/api/tools/` with RESTful verbs |
| 5 | Go Communication | Non-streaming JSON request/response |
| 6 | PR Structure | 3 PRs: Foundation → Parity → Validation |
| 7 | System Prompt Location | Migrate to Node (`lib/ai/prompts.ts`) |
| 8 | Tool↔Go Communication | HTTP endpoints for synchronous tool execution |

---

## Target Architecture (After PR1.5)

```
┌─────────────────────────────────────────────────────────────────┐
│                         USER                                     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    REACT CHAT UI                                 │
│                    (useChat hook)                                │
│                                                                  │
│  - Single chat interface (legacy deprecated)                     │
│  - Provider selector                                             │
│  - Streaming message display                                     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼ POST /api/chat (streaming)
┌─────────────────────────────────────────────────────────────────┐
│                    NODE / AI SDK CORE                            │
│                                                                  │
│  lib/ai/llmClient.ts     - Shared LLM client, provider factory   │
│  lib/ai/prompts.ts       - System prompts                        │
│  lib/ai/tools/*.ts       - Tool definitions (6 total)            │
│  lib/ai/tools/utils.ts   - Shared error handling                 │
│                                                                  │
│  Tools (PR1.5):                                                  │
│  - createChart             → POST /api/tools/charts/create       │
│  - getChartContext         → GET  /api/tools/charts/:id          │
│  - updateChart             → PATCH /api/tools/charts/:id         │
│  - textEditor              → POST /api/tools/editor              │
│  - latestSubchartVersion   → POST /api/tools/versions/subchart   │
│  - latestKubernetesVersion → POST /api/tools/versions/kubernetes │
│                                                                  │
│  Tools (PR2):                                                    │
│  - validateChart           → POST /api/validate                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼ HTTP (JSON, non-streaming)
┌─────────────────────────────────────────────────────────────────┐
│                    GO BACKEND                                    │
│                    (Pure application logic)                      │
│                                                                  │
│  pkg/api/server.go            - HTTP server on :8080             │
│  pkg/api/errors.go            - Standardized error responses     │
│  pkg/api/handlers/charts.go   - Chart CRUD operations            │
│  pkg/api/handlers/editor.go   - File operations                  │
│  pkg/api/handlers/versions.go - Version lookups                  │
│  pkg/validation/*             - Validation pipeline (PR2)        │
│                                                                  │
│  NO LLM CALLS                                                    │
│                                                                  │
│  Legacy (deprecated, unused):                                    │
│  - pkg/llm/client.go                                             │
│  - pkg/llm/conversational.go                                     │
│  - pkg/llm/execute-action.go                                     │
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

## Design Patterns

### Provider Factory Pattern

```typescript
// lib/ai/provider.ts
export function getModel(provider: string, modelId?: string) {
  const openrouter = createOpenRouter({ apiKey: process.env.OPENROUTER_API_KEY });
  
  const models = {
    openai: "openai/gpt-4o",
    anthropic: "anthropic/claude-3.5-sonnet"
  };
  
  return openrouter(modelId || models[provider]);
}
```

### Tool Definition Pattern

```typescript
// lib/ai/tools/createChart.ts
export const createChart = tool({
  description: "Create a new Helm chart...",
  parameters: z.object({
    name: z.string(),
    description: z.string(),
    type: z.enum(['deployment', 'statefulset', 'cronjob', 'custom']).optional(),
  }),
  execute: async (params) => callGoEndpoint('/api/tools/charts/create', params),
});
```

### Shared HTTP Utility Pattern

```typescript
// lib/ai/tools/utils.ts
const GO_BACKEND_URL = process.env.GO_BACKEND_URL || 'http://localhost:8080';

export async function callGoEndpoint<T>(endpoint: string, body: object): Promise<T> {
  const response = await fetch(GO_BACKEND_URL + endpoint, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(body),
  });
  
  if (!response.ok) {
    const error = await response.json();
    throw new Error(error.message || 'Go endpoint failed');
  }
  
  return response.json();
}
```

### Error Response Contract

```go
// pkg/api/errors.go
type ErrorResponse struct {
    Success bool   `json:"success"`
    Message string `json:"message"`
    Code    string `json:"code,omitempty"`
}

// HTTP Status Mapping
// 400 - VALIDATION_ERROR (Invalid request parameters)
// 404 - NOT_FOUND (Workspace/chart doesn't exist)
// 500 - DATABASE_ERROR (DB connection or query failure)
// 502 - EXTERNAL_API_ERROR (ArtifactHub/K8s API failure)
```

---

## Component Relationships

### Frontend Components

```
ChatContainer (container)
├── ProviderSelector (header, PR1/PR2)
├── MessageList (scrollable)
│   └── Message (individual)
│       └── MessagePart (text, tool-call, tool-result)
│           └── ValidationResults (PR2)
├── ChatInput (footer)
└── StreamingControls (stop, regenerate)
```

### State Management

| State | Source | Access |
|-------|--------|--------|
| `messages` | useChat hook | Direct |
| `input` | useChat hook | Direct |
| `status` | useChat hook | 'ready' / 'submitted' / 'streaming' / 'error' |
| `selectedProvider` | Component state | Local |
| Workspace data | Jotai atoms | Existing (preserved) |

---

## Communication Patterns

### Streaming Path (Browser ↔ LLM)

```
Browser ↔ /api/chat ↔ OpenRouter ↔ LLM
        AI SDK Data Stream
```

### Tool Execution Path (Node ↔ Go)

```
/api/chat → tool.execute() → HTTP POST → Go Handler → Database
                            JSON request/response
```

### Real-time Updates (Existing)

```
PostgreSQL → pg_notify → Go Worker → Centrifugo → Browser WebSocket
```

---

## File Structure Patterns

### TypeScript AI Module

```
chartsmith-app/
└── lib/
    └── ai/
        ├── llmClient.ts          # Shared LLM client
        ├── prompts.ts            # System prompts
        ├── provider.ts           # Provider factory
        ├── models.ts             # Model definitions
        └── tools/
            ├── utils.ts          # Shared HTTP utility
            ├── createChart.ts
            ├── getChartContext.ts
            ├── updateChart.ts
            ├── textEditor.ts
            ├── latestSubchartVersion.ts
            ├── latestKubernetesVersion.ts
            └── validateChart.ts  # PR2
```

### Go API Module

```
pkg/
└── api/
    ├── server.go                 # HTTP server on :8080
    ├── auth.go                   # Token validation
    ├── errors.go                 # Error response utilities
    └── handlers/
        ├── charts.go             # Chart CRUD
        ├── editor.go             # File operations
        └── versions.go           # Version lookups

pkg/
└── validation/                   # PR2
    ├── pipeline.go               # Orchestration
    ├── helm.go                   # helm lint/template
    ├── kubescore.go              # kube-score
    ├── parser.go                 # Output parsing
    └── types.go                  # Type definitions
```

---

## Security Patterns

### API Key Protection
- All API keys server-side only
- Never import provider packages in client components
- Environment variables not exposed to client bundle

### Path Validation (Go)
1. Apply `filepath.Clean()` to normalize path
2. Check for ".." components - reject if present
3. Resolve to absolute path
4. Verify path is within allowed base directory
5. Confirm Chart.yaml exists at resolved path

### Command Injection Prevention
- Never construct shell commands with string concatenation
- Always use `exec.Command` with separate argument array
- chartPath passed as single argument, not interpolated

### Authentication Flow
- Extension token validation via `ValidateExtensionToken()`
- Workspace ownership verification via `ValidateWorkspaceAccess()`
- 401/404 for unauthorized (not 403, for security)

---

## Performance Patterns

### Streaming Optimization

```typescript
useChat({
  experimental_throttle: 50  // Throttle UI updates during fast streaming
});
```

### Bundle Size Considerations
- AI SDK packages add ~40KB gzipped
- Offset by removing Anthropic SDK (~25KB)
- Net increase: ~15KB acceptable

### Tool Execution Targets

| Operation | Target Latency |
|-----------|----------------|
| Tool execution | < 500ms (excluding LLM) |
| HTTP round-trip to Go | < 100ms |
| File operation (view) | < 200ms |
| File operation (edit) | < 500ms |
| Validation pipeline | < 30s |

---

*This document captures the technical patterns and architecture for the migration.*

