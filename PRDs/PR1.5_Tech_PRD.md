# PR1.5: AI SDK Tool Integration - Technical Specification

**Version**: 2.0
**PR**: PR1.5 (Migration & Feature Parity)
**Prerequisite**: PR1 merged
**Status**: Ready for Implementation
**Last Updated**: December 3, 2025

---

## Technical Overview

### Objective

Add tool execution capability to the AI SDK chat system created in PR1. This requires:
1. New Go HTTP server (port 8080) - does not exist today
2. 4 AI SDK tool definitions in TypeScript
3. Go HTTP handlers for 3 tools (textEditor, latestSubchartVersion, latestKubernetesVersion)
4. TypeScript-only implementation for getChartContext (direct call to getWorkspace())

### Key Architecture Decision

| Question | Decision | Rationale |
|----------|----------|-----------|
| Workspace creation | **Not needed** - homepage flow handles this | AI SDK chat only runs after workspace exists |
| getChartContext | TypeScript-only (no Go endpoint) | Direct call to getWorkspace() is simpler |
| K8s versions | Keep hardcoded | Feature parity, stability |
| text_editor insert | Remove from scope | Does not exist in codebase |
| Auth pattern | Go validates extension tokens | Match existing Next.js pattern |
| HTTP router | net/http.ServeMux | No external dependencies |

### Key Insight: Workspace Creation Flow

The workspace is created by TypeScript server actions on the homepage **before** the AI SDK chat begins:

```
EXISTING FLOW (No changes needed):
1. User on homepage (/)
2. User types prompt, clicks submit
3. TypeScript: createWorkspaceFromPromptAction() runs
   └─ Creates workspace with bootstrap template
   └─ Creates chat message with prompt
   └─ Enqueues "new_intent" for Go worker
4. User redirected to /workspace/{id}
   └─ Workspace ALREADY EXISTS at this point
5. AI SDK chat begins
   └─ Works within existing workspace
```

**The AI SDK chat only needs tools for operations within an existing workspace.**

---

## System Architecture

### Before PR1.5 (After PR1)

```
┌─────────────────────────────────────────────────────────────────┐
│  NEW AI SDK Chat (PR1)          │  EXISTING System (unchanged) │
├─────────────────────────────────┼───────────────────────────────┤
│  /api/chat/route.ts             │  /api/workspace/[id]/message  │
│  useChat hook                   │  Jotai atoms                  │
│  OpenRouter streaming           │  Centrifugo WebSocket         │
│  NO TOOLS                       │  Tools via Go worker          │
└─────────────────────────────────┴───────────────────────────────┘
```

### After PR1.5

```
┌─────────────────────────────────────────────────────────────────┐
│                    NODE / AI SDK CORE                            │
│                    (All LLM "thinking")                          │
│                                                                  │
│  lib/ai/llmClient.ts     - Shared LLM client, provider factory   │
│  lib/ai/prompts.ts       - System prompts                        │
│  lib/ai/tools/*.ts       - Tool definitions (4 total)            │
│  lib/ai/tools/utils.ts   - Shared error handling                 │
│                                                                  │
│  Tools (4 total):                                                │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ GO BACKEND TOOLS (need HTTP endpoints):                      ││
│  │ - textEditor              → POST /api/tools/editor           ││
│  │ - latestSubchartVersion   → POST /api/tools/versions/subchart││
│  │ - latestKubernetesVersion → POST /api/tools/versions/k8s     ││
│  │                                                              ││
│  │ TYPESCRIPT-ONLY TOOLS (no Go endpoint):                      ││
│  │ - getChartContext         → Direct getWorkspace() call       ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
│  REMOVED (not needed - handled by existing flows):               │
│  ✗ createChart        - Homepage flow handles workspace creation │
│  ✗ updateChart        - Use iterative textEditor instead        │
│  ✗ createEmptyWorkspace - Homepage flow handles this            │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼ HTTP (JSON, non-streaming)
┌─────────────────────────────────────────────────────────────────┐
│                    GO BACKEND                                    │
│                    (Pure application logic)                      │
│                                                                  │
│  pkg/api/server.go            - HTTP server on :8080             │
│  pkg/api/auth.go              - Token validation                 │
│  pkg/api/errors.go            - Standardized error responses     │
│  pkg/api/handlers/editor.go   - File operations (textEditor)     │
│  pkg/api/handlers/versions.go - Version lookups (both tools)     │
│                                                                  │
│  NO LLM CALLS                                                    │
└─────────────────────────────────────────────────────────────────┘
```

---

## Tech Stack

### New TypeScript Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| zod | ^3.22.0 | Tool parameter validation (if not present) |

### Go Dependencies

No new Go dependencies required. Uses standard library:
- `net/http` - HTTP server and ServeMux router
- `encoding/json` - Request/response handling

### Retained Dependencies

All existing dependencies remain unchanged.

---

## File Structure Changes

### New Files (TypeScript) - 7 files

```
chartsmith-app/
├── lib/
│   └── ai/
│       ├── llmClient.ts                    # Shared LLM client
│       ├── prompts.ts                      # System prompts
│       └── tools/
│           ├── getChartContext.ts          # Chart context (TypeScript-only, no Go)
│           ├── textEditor.ts               # File operations tool (calls Go)
│           ├── latestSubchartVersion.ts    # Subchart lookup tool (calls Go)
│           ├── latestKubernetesVersion.ts  # K8s version tool (calls Go)
│           └── utils.ts                    # Shared HTTP utility for Go calls
└── tests/
    └── integration/
        └── tools.test.ts                   # Integration test
```

### New Files (Go) - 4 files

```
pkg/
└── api/
    ├── server.go                           # HTTP server startup
    ├── auth.go                             # Token validation
    ├── errors.go                           # Error response utilities
    └── handlers/
        ├── versions.go                     # Both version endpoints
        └── editor.go                       # Editor endpoint
```

### Modified Files

| File | Changes |
|------|---------|
| app/api/chat/route.ts | Add 4 tools to streamText |
| cmd/run.go | Start HTTP server in goroutine |
| package.json | Remove @anthropic-ai/sdk |

### Files to Mark as Legacy

```
pkg/llm/client.go           # Document as legacy
pkg/llm/conversational.go   # Document as legacy
pkg/llm/execute-action.go   # Document as legacy
pkg/llm/system.go           # Source for prompt migration
```

---

## Backend API Specifications

### HTTP Server Configuration

**File**: `pkg/api/server.go`

**Port**: 8080
**Router**: net/http.ServeMux (Go 1.22+ pattern matching)

**Route Registration**:
```
POST /api/tools/editor              → handlers.TextEditor
POST /api/tools/versions/subchart   → handlers.GetSubchartVersion
POST /api/tools/versions/kubernetes → handlers.GetKubernetesVersion
```

**Note**: Only 3 Go endpoints needed. `getChartContext` is TypeScript-only (direct call to `getWorkspace()`). Workspace creation is handled by the existing homepage flow.

**Startup Pattern** (in cmd/run.go):
```
listener.StartHeartbeat(ctx)

// NEW: Start HTTP server in goroutine
go func() {
    logger.Info("Starting HTTP server on port 8080")
    if err := api.StartHTTPServer(ctx, "8080"); err != nil && err != http.ErrServerClosed {
        logger.Error("HTTP server error", zap.Error(err))
    }
}()

if err := listener.StartListeners(ctx); err != nil {
    return fmt.Errorf("failed to start listeners: %w", err)
}
```

### Error Response Contract

**File**: `pkg/api/errors.go`

**Standard Error Response**:
```
ErrorResponse {
    Success bool   `json:"success"`
    Message string `json:"message"`
    Code    string `json:"code,omitempty"`
}
```

**HTTP Status Mapping**:
| Code | Status | Use Case |
|------|--------|----------|
| VALIDATION_ERROR | 400 | Invalid request parameters |
| NOT_FOUND | 404 | Workspace/chart doesn't exist |
| DATABASE_ERROR | 500 | DB connection or query failure |
| EXTERNAL_API_ERROR | 502 | ArtifactHub/K8s API failure |
| INTERNAL_ERROR | 500 | Unexpected server error |

### Authentication Pattern

**File**: `pkg/api/auth.go`

**Token Validation**:
```
FUNCTION ValidateExtensionToken(ctx, token string) (userID string, error)
    Query: SELECT user_id FROM extension_token WHERE token = $1
    Return: userID or ErrUnauthorized
```

**Workspace Access Check**:
```
FUNCTION ValidateWorkspaceAccess(ctx, workspaceID, userID string) error
    Call: workspace.ListUserIDsForWorkspace(ctx, workspaceID)
    Verify: userID matches owner
    Return: nil or ErrForbidden
```

**Request Helper**:
```
FUNCTION ExtractUserID(r *http.Request) (userID string, error)
    Get: Authorization header
    Extract: Bearer token
    Validate: via ValidateExtensionToken
    Return: userID or ErrUnauthorized
```

---

## API Endpoint Specifications

### getChartContext (TypeScript-Only - No Go Endpoint)

**File**: `lib/ai/tools/getChartContext.ts`

This tool does NOT require a Go HTTP endpoint. It calls the existing TypeScript `getWorkspace()` function directly:

```typescript
// lib/ai/tools/getChartContext.ts
import { getWorkspace } from '@/lib/workspace/workspace'

export const getChartContext = tool({
  description: 'Get the current chart context including all files and metadata',
  parameters: z.object({
    workspaceId: z.string().describe('The workspace ID to load'),
  }),
  execute: async ({ workspaceId }) => {
    // Direct TypeScript call - no HTTP needed
    const workspace = await getWorkspace(workspaceId)
    return {
      success: true,
      workspace: {
        id: workspace.id,
        name: workspace.name,
        revisionNumber: workspace.currentRevisionNumber,
        files: workspace.files,
        charts: workspace.charts,
      }
    }
  },
})
```

**Rationale**: The `getWorkspace()` function already exists in TypeScript and returns all necessary data. Adding a Go HTTP endpoint would add unnecessary latency with no benefit.

---

### POST /api/tools/editor

**Handler**: `pkg/api/handlers/editor.go:TextEditor`

**Request Body**:
```
TextEditorRequest {
    Command     string `json:"command"`     // view, create, str_replace
    WorkspaceID string `json:"workspaceId"`
    Path        string `json:"path"`
    Content     string `json:"content,omitempty"`
    OldStr      string `json:"oldStr,omitempty"`
    NewStr      string `json:"newStr,omitempty"`
}
```

**Response Body**:
```
TextEditorResponse {
    Success bool   `json:"success"`
    Content string `json:"content,omitempty"`
    Message string `json:"message,omitempty"`
}
```

**Command Logic**:

**view**:
```
IF file exists:
    Return content
ELSE:
    Return "Error: File does not exist. Use create instead."
```

**create**:
```
IF file exists:
    Return "Error: File already exists. Use view and str_replace instead."
ELSE:
    Call workspace.AddFileToChart()
    Return "Created"
```

**str_replace**:
```
Call llm.PerformStringReplacement(currentContent, oldStr, newStr)
IF success:
    Call workspace.SetFileContentPending()
    Return new content
ELSE:
    Return "Error: String to replace not found in file."
```

**Note**: Only `view`, `create`, and `str_replace` commands are supported. There is no `insert` command - this matches the existing codebase implementation.

### POST /api/tools/versions/subchart

**Handler**: `pkg/api/handlers/versions.go:GetSubchartVersion`

**Request Body**:
```
SubchartVersionRequest {
    ChartName  string `json:"chartName"`
    Repository string `json:"repository,omitempty"`
}
```

**Response Body**:
```
SubchartVersionResponse {
    Success bool   `json:"success"`
    Version string `json:"version"`
    Name    string `json:"name"`
}
```

**Logic Flow**:
1. Validate auth
2. Call recommendations.GetLatestSubchartVersion(chartName)
3. If error: return version = "?"
4. Return version

### POST /api/tools/versions/kubernetes

**Handler**: `pkg/api/handlers/versions.go:GetKubernetesVersion`

**Request Body**:
```
KubernetesVersionRequest {
    SemverField string `json:"semverField,omitempty"`
}
```

**Response Body**:
```
KubernetesVersionResponse {
    Success bool   `json:"success"`
    Version string `json:"version"`
    Field   string `json:"field"`
}
```

**Logic Flow** (Hardcoded - matches existing behavior):
```
SWITCH semverField:
    "major": return "1"
    "minor": return "1.32"
    "patch" or default: return "1.32.1"
```

---

## Database Interactions

### Tables Accessed

| Table | Operations | Handler |
|-------|------------|---------|
| extension_token | SELECT | All (auth) |
| workspace | SELECT | TextEditor (via workspace functions) |
| workspace_revision | SELECT | TextEditor (via workspace functions) |
| workspace_chart | SELECT | TextEditor (via workspace functions) |
| workspace_file | SELECT, UPDATE | TextEditor |

**Note**: No workspace creation tables needed - workspace creation is handled by the existing homepage TypeScript flow before AI SDK chat begins.

### Existing Functions to Reuse

| Function | Package | Purpose |
|----------|---------|---------|
| GetWorkspace | workspace | Load workspace with all related data (used by getChartContext in TypeScript) |
| ListFiles | workspace | Get file list for revision |
| GetFile | workspace | Get single file content |
| SetFileContentPending | workspace | Update file content |
| AddFileToChart | workspace | Add file to chart |
| PerformStringReplacement | llm | String replacement with fuzzy matching |
| GetLatestSubchartVersion | recommendations | ArtifactHub lookup |
| ListUserIDsForWorkspace | workspace | Get workspace owners |

### Connection Pool Usage

```
conn := persistence.MustGetPooledPostgresSession()
defer conn.Release()
```

---

## Front-End Architecture

### Tool Definition Pattern

**File**: `lib/ai/tools/textEditor.ts` (example - calls Go HTTP)

```
IMPORT tool from 'ai'
IMPORT z from 'zod'
IMPORT callGoEndpoint from './utils'

EXPORT textEditor = tool({
    description: "View, edit, or create files in the chart...",

    parameters: z.object({
        command: z.enum(['view', 'create', 'str_replace']),  // NO insert
        workspaceId: z.string(),
        path: z.string().describe('File path relative to chart root'),
        content: z.string().optional().describe('For create: full content'),
        oldStr: z.string().optional().describe('For str_replace: text to find'),
        newStr: z.string().optional().describe('For str_replace: replacement'),
    }),

    execute: async (params) => {
        return callGoEndpoint('/api/tools/editor', params)
    },
})
```

### getChartContext Tool Definition (TypeScript-Only)

**File**: `lib/ai/tools/getChartContext.ts`

```
IMPORT tool from 'ai'
IMPORT z from 'zod'
IMPORT { getWorkspace } from '@/lib/workspace/workspace'

EXPORT getChartContext = tool({
    description: "Get the current chart context including all files and metadata",

    parameters: z.object({
        workspaceId: z.string().describe('The workspace ID to load'),
    }),

    execute: async ({ workspaceId }) => {
        // Direct TypeScript call - no HTTP endpoint needed
        const workspace = await getWorkspace(workspaceId)
        return {
            success: true,
            workspace: {
                id: workspace.id,
                name: workspace.name,
                revisionNumber: workspace.currentRevisionNumber,
                files: workspace.files,
                charts: workspace.charts,
            }
        }
    },
})
```

**Note**: This tool does NOT call a Go HTTP endpoint. It directly calls the existing TypeScript `getWorkspace()` function.

### Shared Utility Pattern

**File**: `lib/ai/tools/utils.ts`

**Auth Header Forwarding Problem**:
The AI SDK `tool({ execute })` pattern only receives tool parameters, not the HTTP request context.

**Solution**:
1. `callGoEndpoint(endpoint, body, authHeader)` - takes auth header as parameter
2. `createTools(authHeader, workspaceId)` - factory function that creates all 4 tools with auth header captured in closure
3. Route handler extracts auth header first, then calls `createTools()`

**callGoEndpoint Logic**:
- POST to `GO_BACKEND_URL + endpoint` with JSON body
- Include Authorization header if provided
- Return parsed JSON response or throw error with message

### Chat Route Integration

**File**: `app/api/chat/route.ts` (modification)

**Request Body Change from PR1**:
- PR1: `{ messages, provider, model }`
- PR1.5: `{ messages, model, workspaceId, revisionNumber }`

Tools need `workspaceId` to operate on the correct workspace.

**Route Handler Logic**:
1. Extract auth header from request (for Go HTTP calls)
2. Parse request body including new `workspaceId` and `revisionNumber` fields
3. Call `createTools(authHeader, workspaceId)` to get tools with auth in closure
4. Call `runChat()` with model, messages, system prompt, and tools
5. Return streaming response

### LLM Client Library

**File**: `lib/ai/llmClient.ts`

**Relationship with PR1's provider.ts**:
- PR1 created `lib/ai/provider.ts` with `getModel()` and `AVAILABLE_PROVIDERS`
- PR1.5's `llmClient.ts` **imports from and extends** `provider.ts` - does NOT replace it

**Exports**:
- `runChat(options)` - NEW wrapper around `streamText()` that accepts tools
- `getModel()` - re-exported from provider.ts
- `AVAILABLE_PROVIDERS` - re-exported from provider.ts

**lib/ai/ File Summary**:
| File | Source | Purpose |
|------|--------|---------|
| `provider.ts` | PR1 | `getModel()`, `AVAILABLE_PROVIDERS` |
| `llmClient.ts` | PR1.5 | `runChat()` wrapper with tools support |
| `prompts.ts` | PR1.5 | System prompts |
| `tools/*.ts` | PR1.5 | Tool definitions |

### System Prompts

**File**: `lib/ai/prompts.ts`

```
EXPORT FUNCTION getSystemPrompt(context):
    basePrompt = `
    You are Chartsmith, an AI assistant for Helm charts.

    ## Available Tools (4 total)
    - getChartContext: Load current chart files and metadata
    - textEditor: View, edit, or create specific files (commands: view, create, str_replace)
    - latestSubchartVersion: Look up subchart versions on ArtifactHub
    - latestKubernetesVersion: Get Kubernetes version information

    ## Behavior Guidelines
    - The workspace already exists when chat begins (created via homepage)
    - Use getChartContext to understand the current chart state
    - For specific file changes, use textEditor (view, then str_replace, or create)
    - For subchart dependencies, use latestSubchartVersion
    - For K8s API compatibility, use latestKubernetesVersion
    - For chart generation, use <chartsmithArtifact> tags (existing flow)
    - For action plans, use <chartsmithActionPlan> tags (existing flow)
    `

    IF context.chartContext:
        RETURN basePrompt + "\n\n## Current Chart Context\n" + context.chartContext

    RETURN basePrompt
```

---

## Environment Configuration

### New Environment Variables

```
# PR1.5 Addition
GO_BACKEND_URL=http://localhost:8080
```

### Existing Variables (Unchanged)

```
# From PR1
OPENROUTER_API_KEY=sk-or-v1-xxxxx
DEFAULT_AI_PROVIDER=openai
DEFAULT_AI_MODEL=openai/gpt-4o
```

### Go Environment

No new Go environment variables required. Uses existing:
- DATABASE_URL (PostgreSQL connection)
- Existing configuration via AWS SSM or env

---

## Deployment Plan

### Phase 1: Go HTTP Server

1. Create pkg/api/server.go with route registration (3 routes only)
2. Create pkg/api/auth.go with token validation
3. Create pkg/api/errors.go with response helpers
4. Modify cmd/run.go to start HTTP server in goroutine
5. Test: Verify server starts on port 8080

### Phase 2: Go Handlers (3 handlers)

1. Create pkg/api/handlers/editor.go
   - TextEditor handler with view/create/str_replace
2. Create pkg/api/handlers/versions.go
   - GetSubchartVersion with ArtifactHub call
   - GetKubernetesVersion with hardcoded values
3. Test: Each endpoint with curl/httpie

### Phase 3: TypeScript Tools (4 tools)

1. Create lib/ai/tools/utils.ts with callGoEndpoint helper
2. Create lib/ai/tools/getChartContext.ts (TypeScript-only, no Go)
3. Create lib/ai/tools/textEditor.ts (calls Go)
4. Create lib/ai/tools/latestSubchartVersion.ts (calls Go)
5. Create lib/ai/tools/latestKubernetesVersion.ts (calls Go)
6. Create lib/ai/llmClient.ts
7. Create lib/ai/prompts.ts
8. Modify app/api/chat/route.ts to register 4 tools
9. Test: Tool invocation via chat

### Phase 4: Integration & Cleanup

1. Remove @anthropic-ai/sdk from package.json
2. Delete lib/llm/prompt-type.ts
3. Create integration test
4. Run full test suite
5. Document legacy files

---

## Development Workflow

### Local Development Setup

1. Start PostgreSQL (existing setup)
2. Start Go worker: `go run main.go run` (now includes HTTP server)
3. Start Next.js: `npm run dev`
4. Verify: `curl http://localhost:8080/api/tools/versions/kubernetes`

### Testing Workflow

**Unit Tests (Go)**:
```
go test ./pkg/api/... -v
```

**Unit Tests (TypeScript)**:
```
npm run test -- lib/ai/
```

**Integration Tests**:
```
npm run test:integration
```

### Debugging Tips

1. HTTP server logs: Check for startup errors in Go logs
2. Tool calls: AI SDK logs tool invocations
3. Auth failures: Check extension_token table
4. Database errors: Check PostgreSQL logs

---

## Integration Points

### With PR1 Components

| PR1 Component | PR1.5 Integration |
|---------------|-------------------|
| /api/chat/route.ts | Add 4 tools to streamText |
| useChat hook | Tools work automatically via AI SDK |
| OpenRouter provider | Tools use same model instance |
| ProviderSelector | No changes needed |

### With Existing Go Code

| Existing Function | PR1.5 Usage |
|-------------------|-------------|
| persistence.MustGetPooledPostgresSession | Database connections |
| workspace.ListFiles | List chart files |
| workspace.GetFile | Get file content |
| workspace.SetFileContentPending | Update file content |
| workspace.AddFileToChart | Create new file |
| llm.PerformStringReplacement | Text editor str_replace |
| recommendations.GetLatestSubchartVersion | Subchart lookup |

**Note**: `workspace.GetWorkspace` is called from TypeScript (getChartContext tool), not from Go HTTP endpoints.

### With Database

All database access through existing persistence layer:
- Connection pool from InitPostgres()
- No schema changes required
- No workspace creation from Go (handled by homepage TypeScript flow)

---

## Validation Checklist

### Function References
- [ ] All Go function references match actual codebase signatures
- [ ] No references to non-existent functions
- [ ] Correct package names (workspace, persistence, recommendations)

### Tool Definitions (4 total)
- [ ] getChartContext is TypeScript-only (no Go endpoint)
- [ ] textEditor has only: view, create, str_replace (NO insert)
- [ ] latestKubernetesVersion uses semverField (NOT channel)
- [ ] latestSubchartVersion matches existing ArtifactHub integration

### Architecture
- [ ] HTTP server added to cmd/run.go, runs in goroutine
- [ ] Only 3 Go HTTP endpoints (editor, versions/subchart, versions/kubernetes)
- [ ] Auth pattern matches existing extension token validation
- [ ] All handlers use persistence.MustGetPooledPostgresSession()
- [ ] No createChart, updateChart, or createEmptyWorkspace endpoints

### Testing
- [ ] Integration tests for each HTTP endpoint
- [ ] Auth failure returns 401/404 (not 403)
- [ ] Tool → Go HTTP → Response pattern verified (for 3 Go tools)
- [ ] getChartContext TypeScript-only implementation verified

---

## Open Items (Not Blockers)

Decisions that can be made during implementation:

1. **Error logging**: Should handlers log to str_replace_log table?
2. **File validation**: Should textEditor validate YAML syntax?

---

## Related Documents

| Document | Purpose |
|----------|---------|
| PR1.5_Product_PRD.md | Functional requirements |
| PR1.5_PLAN.md | High-level task list |
| CHARTSMITH_ARCHITECTURE_DECISIONS.md | Key decisions |
| docs/research/2025-12-02-pr1.5-implementation-guide.md | Verified implementation guidance |
| docs/research/2025-12-02-pr1.5-codebase-verification.md | Codebase analysis |
| PR1_Tech_PRD.md | PR1 technical spec (baseline) |

---

*Document End - PR1.5: AI SDK Tool Integration Technical Specification*
