# PR1.5: AI SDK Tool Integration - Technical Specification

**Version**: 1.0
**PR**: PR1.5 (Migration & Feature Parity)
**Prerequisite**: PR1 merged
**Status**: Ready for Implementation
**Last Updated**: December 2, 2025

---

## Technical Overview

### Objective

Add tool execution capability to the AI SDK chat system created in PR1. This requires:
1. New Go HTTP server (port 8080) - does not exist today
2. 6 AI SDK tool definitions in TypeScript - call Go endpoints
3. Go HTTP handlers - execute tool logic using existing functions

### Key Architecture Decision

| Question | Decision | Rationale |
|----------|----------|-----------|
| Workspace creation | Go implements own SQL | Simplicity, no network hop |
| K8s versions | Keep hardcoded | Feature parity, stability |
| text_editor insert | Remove from scope | Does not exist in codebase |
| Auth pattern | Go validates extension tokens | Match existing Next.js pattern |
| HTTP router | net/http.ServeMux | No external dependencies |

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
│  lib/ai/tools/*.ts       - Tool definitions (6 total)            │
│  lib/ai/tools/utils.ts   - Shared error handling                 │
│                                                                  │
│  Tools:                                                          │
│  - createChart             → POST /api/tools/charts/create       │
│  - getChartContext         → GET  /api/tools/charts/:id          │
│  - updateChart             → PATCH /api/tools/charts/:id         │
│  - textEditor              → POST /api/tools/editor              │
│  - latestSubchartVersion   → POST /api/tools/versions/subchart   │
│  - latestKubernetesVersion → POST /api/tools/versions/kubernetes │
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
│  pkg/api/handlers/charts.go   - Chart CRUD operations            │
│  pkg/api/handlers/editor.go   - File operations                  │
│  pkg/api/handlers/versions.go - Version lookups                  │
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

### New Files (TypeScript) - 9 files

```
chartsmith-app/
├── lib/
│   └── ai/
│       ├── llmClient.ts                    # Shared LLM client
│       ├── prompts.ts                      # System prompts
│       └── tools/
│           ├── createChart.ts              # Chart creation tool
│           ├── getChartContext.ts          # Chart loading tool
│           ├── updateChart.ts              # Chart update tool
│           ├── textEditor.ts               # File operations tool
│           ├── latestSubchartVersion.ts    # Subchart lookup tool
│           ├── latestKubernetesVersion.ts  # K8s version tool
│           └── utils.ts                    # Shared utilities
└── tests/
    └── integration/
        └── create-chart.test.ts            # Integration test
```

### New Files (Go) - 5 files

```
pkg/
└── api/
    ├── server.go                           # HTTP server startup
    ├── auth.go                             # Token validation
    ├── errors.go                           # Error response utilities
    └── handlers/
        ├── charts.go                       # Chart endpoints
        ├── versions.go                     # Version endpoints
        └── editor.go                       # Editor endpoint
```

### Modified Files

| File | Changes |
|------|---------|
| app/api/chat/route.ts | Add 6 tools to streamText |
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
POST /api/tools/charts/create    → handlers.CreateChart
GET  /api/tools/charts/{id}      → handlers.GetChartContext
PATCH /api/tools/charts/{id}     → handlers.UpdateChart
POST /api/tools/editor           → handlers.TextEditor
POST /api/tools/versions/subchart   → handlers.GetSubchartVersion
POST /api/tools/versions/kubernetes → handlers.GetKubernetesVersion
```

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

### POST /api/tools/charts/create

**Handler**: `pkg/api/handlers/charts.go:CreateChart`

**Request Body**:
```
CreateChartRequest {
    Name        string `json:"name"`
    Description string `json:"description"`
    Type        string `json:"type,omitempty"`
}
```

**Response Body**:
```
CreateChartResponse {
    Success     bool   `json:"success"`
    WorkspaceID string `json:"workspaceId"`
    RevisionID  int    `json:"revisionNumber"`
    Message     string `json:"message"`
}
```

**Logic Flow**:
1. Extract and validate extension token
2. Parse request body
3. Generate workspace ID (12-char random string)
4. Begin database transaction
5. Query bootstrap workspace
6. INSERT into workspace table
7. INSERT into workspace_revision (revision 0)
8. Copy charts from bootstrap_chart
9. Copy files from bootstrap_file
10. Commit transaction
11. Enqueue render_workspace work
12. Return success response

**Database Operations** (SQL Pattern from workspace.ts:30-99):
```
INSERT INTO workspace (id, created_at, last_updated_at, name, created_by_user_id, created_type, current_revision_number)
VALUES ($1, now(), now(), $2, $3, 'chat', 0)

INSERT INTO workspace_revision (workspace_id, revision_number, created_at, created_by_user_id, created_type, is_complete, is_rendered)
VALUES ($1, 0, now(), $2, 'chat', true, false)
```

### GET /api/tools/charts/{id}

**Handler**: `pkg/api/handlers/charts.go:GetChartContext`

**Path Parameter**: id (workspace ID)

**Response Body**: Full workspace object from GetWorkspace()

**Logic Flow**:
1. Extract and validate extension token
2. Extract workspace ID from path
3. Validate workspace access
4. Call workspace.GetWorkspace(ctx, workspaceID)
5. Return workspace as JSON

### PATCH /api/tools/charts/{id}

**Handler**: `pkg/api/handlers/charts.go:UpdateChart`

**Request Body**:
```
UpdateChartRequest {
    Changes []Change `json:"changes"`
    Message string   `json:"message"`
}

Change {
    Type  string `json:"type"`
    Path  string `json:"path,omitempty"`
    Value any    `json:"value,omitempty"`
}
```

**Logic Flow**:
1. Validate auth and workspace access
2. Load current workspace
3. Apply changes to files
4. Create new revision via workspace.CreateRevision()
5. Enqueue render_workspace work
6. Return success with new revision number

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
| workspace | SELECT, INSERT | CreateChart, GetChartContext |
| workspace_revision | SELECT, INSERT | CreateChart, UpdateChart |
| workspace_chart | SELECT, INSERT | CreateChart, GetChartContext |
| workspace_file | SELECT, INSERT, UPDATE | All file operations |
| bootstrap_workspace | SELECT | CreateChart |
| bootstrap_chart | SELECT | CreateChart |
| bootstrap_file | SELECT | CreateChart |

### Existing Functions to Reuse

| Function | Package | Purpose |
|----------|---------|---------|
| GetWorkspace | workspace | Load workspace with all related data |
| CreateRevision | workspace | Create new revision with copy-on-write |
| ListFiles | workspace | Get file list for revision |
| GetFile | workspace | Get single file content |
| SetFileContentPending | workspace | Update file content |
| CreateChart | workspace | Create empty chart |
| AddFileToChart | workspace | Add file to chart |
| ApplyPatch | diff | Apply unified diff |
| PerformStringReplacement | llm | String replacement with fuzzy matching |
| GetLatestSubchartVersion | recommendations | ArtifactHub lookup |
| EnqueueWork | persistence | Trigger pg_notify |
| ListUserIDsForWorkspace | workspace | Get workspace owners |

### Connection Pool Usage

```
conn := persistence.MustGetPooledPostgresSession()
defer conn.Release()
```

---

## Front-End Architecture

### Tool Definition Pattern

**File**: `lib/ai/tools/createChart.ts` (example)

```
IMPORT tool from 'ai'
IMPORT z from 'zod'
IMPORT callGoEndpoint from './utils'

EXPORT createChart = tool({
    description: "Create a new Helm chart...",

    parameters: z.object({
        name: z.string().describe('Chart name'),
        description: z.string().describe('What the chart should do'),
        type: z.enum(['deployment', 'statefulset', 'cronjob', 'custom'])
            .optional()
            .describe('Primary workload type'),
    }),

    execute: async (params) => {
        return callGoEndpoint('/api/tools/charts/create', params)
    },
})
```

### textEditor Tool Definition

**File**: `lib/ai/tools/textEditor.ts`

```
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

### Shared Utility Pattern

**File**: `lib/ai/tools/utils.ts`

```
CONST GO_BACKEND_URL = process.env.GO_BACKEND_URL || 'http://localhost:8080'

EXPORT ASYNC FUNCTION callGoEndpoint<T>(endpoint, body):
    response = await fetch(GO_BACKEND_URL + endpoint, {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
            'Authorization': forwardedAuthHeader  // from request context
        },
        body: JSON.stringify(body)
    })

    IF not response.ok:
        error = await response.json()
        THROW new Error(error.message || 'Go endpoint failed')

    RETURN response.json()
```

### Chat Route Integration

**File**: `app/api/chat/route.ts` (modification)

```
IMPORT runChat from '@/lib/ai/llmClient'
IMPORT createChart, getChartContext, updateChart from tools
IMPORT textEditor, latestSubchartVersion, latestKubernetesVersion from tools
IMPORT getSystemPrompt from '@/lib/ai/prompts'

EXPORT ASYNC FUNCTION POST(req):
    PARSE { messages, model, workspaceId, revisionNumber } from req.json()

    result = await runChat({
        model,
        messages,
        system: getSystemPrompt({ workspaceId, revisionNumber }),
        tools: {
            createChart,
            getChartContext,
            updateChart,
            textEditor,
            latestSubchartVersion,
            latestKubernetesVersion,
        },
    })

    RETURN result.toDataStreamResponse()
```

### LLM Client Library

**File**: `lib/ai/llmClient.ts`

```
IMPORT streamText from 'ai'
IMPORT createOpenRouter from '@openrouter/ai-sdk-provider'

EXPORT FUNCTION getModel(modelId):
    openrouter = createOpenRouter({ apiKey: process.env.OPENROUTER_API_KEY })
    RETURN openrouter(modelId)

EXPORT ASYNC FUNCTION runChat(options):
    RETURN streamText({
        model: getModel(options.model),
        messages: options.messages,
        system: options.system,
        tools: options.tools,
    })

EXPORT CONST AVAILABLE_MODELS = [
    { id: 'openai/gpt-4o', name: 'GPT-4o', provider: 'openai' },
    { id: 'anthropic/claude-3.5-sonnet', name: 'Claude 3.5 Sonnet', provider: 'anthropic' },
]
```

### System Prompts

**File**: `lib/ai/prompts.ts`

```
EXPORT FUNCTION getSystemPrompt(context):
    basePrompt = `
    You are Chartsmith, an AI assistant for Helm charts.

    ## Available Tools
    - createChart: Create a new Helm chart from requirements
    - getChartContext: Load current chart files and metadata
    - updateChart: Modify existing chart configuration
    - textEditor: View, edit, or create specific files (commands: view, create, str_replace)
    - latestSubchartVersion: Look up subchart versions on ArtifactHub
    - latestKubernetesVersion: Get Kubernetes version information

    ## Behavior Guidelines
    - When revisionNumber is 0 or user wants a new chart, use createChart
    - For questions about current chart state, use getChartContext first
    - For specific file changes, use textEditor
    - For subchart dependencies, use latestSubchartVersion
    - For K8s API compatibility, use latestKubernetesVersion
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

1. Create pkg/api/server.go with route registration
2. Create pkg/api/auth.go with token validation
3. Create pkg/api/errors.go with response helpers
4. Modify cmd/run.go to start HTTP server in goroutine
5. Test: Verify server starts on port 8080

### Phase 2: Go Handlers

1. Create pkg/api/handlers/charts.go
   - CreateChart handler with workspace creation SQL
   - GetChartContext handler with GetWorkspace call
   - UpdateChart handler with CreateRevision call
2. Create pkg/api/handlers/editor.go
   - TextEditor handler with view/create/str_replace
3. Create pkg/api/handlers/versions.go
   - GetSubchartVersion with ArtifactHub call
   - GetKubernetesVersion with hardcoded values
4. Test: Each endpoint with curl/httpie

### Phase 3: TypeScript Tools

1. Create lib/ai/tools/utils.ts with callGoEndpoint
2. Create each tool definition file
3. Create lib/ai/llmClient.ts
4. Create lib/ai/prompts.ts
5. Modify app/api/chat/route.ts
6. Test: Tool invocation via chat

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
| /api/chat/route.ts | Add tools parameter to streamText |
| useChat hook | Tools work automatically via AI SDK |
| OpenRouter provider | Tools use same model instance |
| ProviderSelector | No changes needed |

### With Existing Go Code

| Existing Function | PR1.5 Usage |
|-------------------|-------------|
| persistence.MustGetPooledPostgresSession | Database connections |
| workspace.GetWorkspace | Load chart context |
| workspace.CreateRevision | Create new revision |
| workspace.ListFiles | List chart files |
| workspace.SetFileContentPending | Update file content |
| workspace.AddFileToChart | Create new file |
| llm.PerformStringReplacement | Text editor str_replace |
| recommendations.GetLatestSubchartVersion | Subchart lookup |
| persistence.EnqueueWork | Trigger renders |

### With Database

All database access through existing persistence layer:
- Connection pool from InitPostgres()
- Transaction support for workspace creation
- No schema changes required

---

## Validation Checklist

### Function References
- [ ] All Go function references match actual codebase signatures
- [ ] No references to non-existent functions
- [ ] Correct package names (workspace, persistence, diff, recommendations)

### Tool Definitions
- [ ] textEditor has only: view, create, str_replace (NO insert)
- [ ] latestKubernetesVersion uses semverField (NOT channel)
- [ ] latestSubchartVersion matches existing ArtifactHub integration

### Architecture
- [ ] HTTP server added to cmd/run.go, runs in goroutine
- [ ] Auth pattern matches existing extension token validation
- [ ] All handlers use persistence.MustGetPooledPostgresSession()

### Testing
- [ ] Integration tests for each HTTP endpoint
- [ ] Auth failure returns 401/404 (not 403)
- [ ] Tool → Go HTTP → Response pattern verified

---

## Open Items (Not Blockers)

Decisions that can be made during implementation:

1. **Error logging**: Should handlers log to str_replace_log table?
2. **Render trigger**: Should createChart auto-trigger render?
3. **File validation**: Should textEditor validate YAML syntax?

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
