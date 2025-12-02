# PR1.5 High-Level Plan

**Name**: Migration & Feature Parity  
**Timeline**: Days 3-4  
**Prerequisite**: PR1 merged  
**Updated**: December 2, 2025 (incorporated feature parity requirements)

---

## Core Architecture Decision

**Go does NOT need to make LLM calls after PR1.5.**

- All "thinking" (chat, reasoning, tool orchestration) → AI SDK Core (Node)
- Go becomes pure application logic (chart CRUD, file ops)
- No AI Gateway endpoint needed; no Go LLM client modifications needed

**Rationale**: Go no longer performs any LLM reasoning after PR1.5; all intelligence (planning, tool invocation) is handled in the AI SDK layer. Go becomes a clean service layer responsible only for deterministic CRUD and file operations. This is a simplification, not a deficiency.

**After PR1.5, no user-facing chat UI paths route through the legacy Go LLM stack (`pkg/llm/*`). The only chat interface available in the product is the AI SDK-powered `/api/chat` route.**

---

## What PR1.5 Delivers

1. Shared LLM client library - Single source of truth for provider config
2. Go HTTP server - Exposes endpoints for tool execution
3. **AI SDK tools (6 total)** - createChart, getChartContext, updateChart, textEditor, latestSubchartVersion, latestKubernetesVersion
4. System prompts - Migrated from Go, updated for tool usage
5. **Full feature parity** - All existing tools ported; "Create chart via chat" works end-to-end
6. Anthropic SDK removal - No @anthropic-ai/sdk in TypeScript
7. **Standardized error handling** - Consistent error response contract across all Go endpoints

---

## MUST HAVE Tasks

### Task 1: Shared LLM Client Library
**File**: `lib/ai/llmClient.ts`

- Export `runChat()` function wrapping `streamText`
- Export `getModel()` provider factory
- Export `AVAILABLE_MODELS` configuration
- Single place for provider config and model mappings

### Task 2: Go HTTP Server Setup
**File**: `pkg/api/server.go`

- Create HTTP server on port 8080
- Register routes:
  - `/api/tools/charts/create` - Chart creation
  - `/api/tools/charts/:id` - Chart context (GET) and update (PATCH)
  - `/api/tools/editor` - File operations
  - `/api/tools/versions/subchart` - ArtifactHub subchart lookup
  - `/api/tools/versions/kubernetes` - Kubernetes version info
- Start alongside existing queue listener in `cmd/run.go`

### Task 3: createChart Tool + Endpoint
**Files**: `lib/ai/tools/createChart.ts`, `pkg/api/handlers/charts.go`

Tool: Accept chart name, description, type; call Go via HTTP POST; return chart details

Go handler: Create workspace in DB, generate chart files, create revision, trigger pg_notify, return IDs

### Task 4: Register Tools in /api/chat
**File**: `app/api/chat/route.ts` (modify)

- Import tools from `lib/ai/tools/`
- Import `runChat` from llmClient
- Pass tools object to runChat()

### Task 5: System Prompt Migration
**File**: `lib/ai/prompts.ts`

- Copy prompts from `pkg/llm/system.go`
- Create `getSystemPrompt()` with context parameter
- Document tools and when to use each
- Add new chart mode guidance (revisionNumber === 0)

### Task 6: Remove @anthropic-ai/sdk
- Remove from `package.json`
- Delete `lib/llm/prompt-type.ts`
- Verify no imports remain

### Task 7: Integration Test
**File**: `tests/integration/create-chart.test.ts`

- POST "Create a nginx chart" to /api/chat
- Assert createChart tool invoked
- Assert chart exists in database

### Task 8: latestSubchartVersion Tool + Endpoint (Feature Parity - REQUIRED)
**Files**: `lib/ai/tools/latestSubchartVersion.ts`, `pkg/api/handlers/versions.go`

Tool: Accept chart name, optional repository; return latest version from ArtifactHub

```typescript
export const latestSubchartVersion = tool({
  description: `Look up the latest version of a Helm subchart on ArtifactHub. 
    Use when user asks about subchart versions, dependencies, or wants to 
    add a subchart to their chart.`,
  parameters: z.object({
    chartName: z.string().describe('Name of the subchart (e.g., "postgresql", "redis")'),
    repository: z.string().optional().describe('Specific repo (defaults to ArtifactHub search)'),
  }),
  execute: async (params) => callGoEndpoint('/api/tools/versions/subchart', params),
});
```

Go handler: `POST /api/tools/versions/subchart` - Query ArtifactHub API, return version info
**Implementation Note**: Port logic from existing `latest_subchart_version` tool in `pkg/llm/conversational.go`

### Task 9: latestKubernetesVersion Tool + Endpoint (Feature Parity - REQUIRED)
**Files**: `lib/ai/tools/latestKubernetesVersion.ts`, `pkg/api/handlers/versions.go`

Tool: Accept optional channel; return current Kubernetes version information

```typescript
export const latestKubernetesVersion = tool({
  description: `Get current Kubernetes version information. Use when user asks 
    about K8s versions, API compatibility, or which version to target.`,
  parameters: z.object({
    channel: z.enum(['stable', 'latest', 'all']).optional()
      .describe('Version channel (defaults to stable)'),
  }),
  execute: async (params) => callGoEndpoint('/api/tools/versions/kubernetes', params),
});
```

Go handler: `POST /api/tools/versions/kubernetes` - Fetch from K8s API or cached data
**Implementation Note**: Port logic from existing `latest_kubernetes_version` tool in `pkg/llm/conversational.go`

### Task 10: Error Response Contract
**Files**: `pkg/api/errors.go`, `lib/ai/tools/utils.ts`

Standardized error handling across all Go endpoints:

```go
// pkg/api/errors.go
type ErrorResponse struct {
    Success bool   `json:"success"`
    Message string `json:"message"`
    Code    string `json:"code,omitempty"`
    Details any    `json:"details,omitempty"`
}

func WriteError(w http.ResponseWriter, status int, message string)
func WriteErrorWithCode(w http.ResponseWriter, status int, code, message string, details any)
```

Shared TypeScript utility for consistent error handling:

```typescript
// lib/ai/tools/utils.ts
export async function callGoEndpoint<T>(endpoint: string, body: Record<string, unknown>): Promise<T>
```

HTTP Status Code Mapping:
| Code | Status | Use Case |
|------|--------|----------|
| `VALIDATION_ERROR` | 400 | Invalid request parameters |
| `NOT_FOUND` | 404 | Workspace/chart doesn't exist |
| `DATABASE_ERROR` | 500 | DB connection or query failure |
| `EXTERNAL_API_ERROR` | 502 | ArtifactHub/K8s API failure |
| `INTERNAL_ERROR` | 500 | Unexpected server error |

### Task 11: textEditor Tool + Endpoint (Feature Parity - REQUIRED)
**Files**: `lib/ai/tools/textEditor.ts`, `pkg/api/handlers/editor.go`

Tool: Accept command (view/create/str_replace/insert), path, content
Handler: Execute file operation, return result

```typescript
export const textEditor = tool({
  description: `View, edit, or create files in the chart. Use this for:
    - Viewing specific file contents
    - Making precise text changes
    - Creating new template files`,
  parameters: z.object({
    command: z.enum(['view', 'create', 'str_replace', 'insert']),
    workspaceId: z.string(),
    path: z.string().describe('File path relative to chart root'),
    content: z.string().optional().describe('For create: full content'),
    oldStr: z.string().optional().describe('For str_replace: text to find'),
    newStr: z.string().optional().describe('For str_replace: replacement text'),
    insertLine: z.number().optional().describe('For insert: line number'),
    insertText: z.string().optional().describe('For insert: text to insert'),
  }),
  execute: async (params) => callGoEndpoint('/api/tools/editor', params),
});
```

**Implementation Note**: Reuse existing logic from `pkg/agent/tools/text_editor.go`. This is an existing tool (`text_editor_20241022`) that must be ported for feature parity.

---

## NICE TO HAVE Tasks

### Task 12: getChartContext Tool + Endpoint
Tool: Accept workspace ID; return chart metadata and files
Handler: Load workspace, fetch revision files, return context

### Task 13: updateChart Tool + Endpoint
Tool: Accept workspace ID, changes, commit message
Handler: Apply changes, create revision, notify

### Task 14: Deprecation & Documentation
- Add deprecation banner to legacy ChatContainer
- Update ARCHITECTURE.md with new architecture
- Mark `pkg/llm/*` as legacy

---

## File Summary

### New Files (Next.js) - 10 files
| File | Priority |
|------|----------|
| `lib/ai/llmClient.ts` | MUST |
| `lib/ai/prompts.ts` | MUST |
| `lib/ai/tools/createChart.ts` | MUST |
| `lib/ai/tools/latestSubchartVersion.ts` | MUST |
| `lib/ai/tools/latestKubernetesVersion.ts` | MUST |
| `lib/ai/tools/textEditor.ts` | MUST |
| `lib/ai/tools/utils.ts` | MUST |
| `lib/ai/tools/getChartContext.ts` | NICE |
| `lib/ai/tools/updateChart.ts` | NICE |
| `tests/integration/create-chart.test.ts` | MUST |

### New Files (Go) - 5 files
| File | Priority |
|------|----------|
| `pkg/api/server.go` | MUST |
| `pkg/api/handlers/charts.go` | MUST |
| `pkg/api/handlers/versions.go` | MUST |
| `pkg/api/handlers/editor.go` | MUST |
| `pkg/api/errors.go` | MUST |

### Modified Files
| File | Priority |
|------|----------|
| `app/api/chat/route.ts` | MUST |
| `cmd/run.go` or `main.go` | MUST |
| `package.json` | MUST |

### Legacy Files (Document Only)
- `pkg/llm/client.go`, `conversational.go`, `execute-action.go` - Mark as legacy
- `pkg/llm/system.go` - Source for prompt migration

---

## Environment Variables

New: `GO_BACKEND_URL=http://localhost:8080`

---

## Success Criteria

### Must Pass
- [ ] "Create a nginx chart" works via AI SDK chat
- [ ] createChart tool invokes Go endpoint
- [ ] Chart created in database
- [ ] **latestSubchartVersion tool returns valid version data** (feature parity)
- [ ] **latestKubernetesVersion tool returns valid version data** (feature parity)
- [ ] No @anthropic-ai/sdk in node_modules
- [ ] Integration test passes
- [ ] **AI SDK chat must invoke createChart for a new-chart conversation, and Go must create a valid workspace** (validated via integration test)
- [ ] **Error responses follow standard format** (success, message, optional code/details)

### Quality Gates
- [ ] All **4 feature-parity tools** implemented as MUST HAVE (createChart, textEditor, latestSubchartVersion, latestKubernetesVersion)
- [ ] All **6 tools** implemented for full quality (adds getChartContext, updateChart as NICE TO HAVE)
- [ ] System prompts guide tool usage (document all 6 tools)
- [ ] ARCHITECTURE.md updated
- [ ] **Error handling utility (`callGoEndpoint`) shared across all tools**
- [ ] **All Go tool endpoints enforce existing workspace ownership and authentication middleware**

---

## What PR1.5 Enables for PR2

- Add `validateChart` to existing tools array
- Add `/api/validate` to existing Go HTTP server
- Reuse tool → Go HTTP → response pattern

---

## Risk Mitigation

**Priority order if time-constrained:**
1. createChart + Go endpoint (demo requirement)
2. llmClient.ts + tool registration
3. System prompts
4. Integration test
5. **latestSubchartVersion** (feature parity - REQUIRED)
6. **latestKubernetesVersion** (feature parity - REQUIRED)
7. **textEditor** (feature parity - REQUIRED, existing tool)
8. **Error response contract** (production quality)
9. getChartContext (new tool - NICE TO HAVE)
10. updateChart (new tool - NICE TO HAVE)
11. Documentation

**Minimum viable**: Tasks 1-8 (all MUST HAVE items for full feature parity)

**Existing tools that MUST be ported** (3 total):
- `text_editor_20241022` → `textEditor`
- `latest_subchart_version` → `latestSubchartVersion`
- `latest_kubernetes_version` → `latestKubernetesVersion`

**New tools** (3 total, can be NICE TO HAVE):
- `createChart` (MUST - demo requirement)
- `getChartContext` (NICE TO HAVE)
- `updateChart` (NICE TO HAVE)

**Note on feature parity**: The original requirement states "All existing features continue to work (tool calling, file context, etc.)". The utility tools (latestSubchartVersion, latestKubernetesVersion) are existing features that must be ported to maintain this requirement.

---

## Security Note

All PR1.5 Go HTTP tool endpoints (`/api/tools/...`) will enforce existing workspace ownership and authentication middleware. Only the owner or authorized collaborators may modify or read workspace resources.

---

## Go Handler Implementation Guidance

Chart creation, revision creation, file writes, and event notifications all reuse existing production-stable Go functions:
- `CreateWorkspace(ctx, userID, name)`
- `InitializeWorkspaceFiles(workspaceID, type, opts)`
- `CreateInitialRevision(workspaceID, files)`
- `workspace.Notify(ctx, workspaceID, "chart_created")`
- `workspace.LoadByID(workspaceID)`
- `revisions.GetLatest(workspaceID)`
- `workspaceFiles.List(revision.ID)`
- `files.ApplyPatch(...)`

We do not fork or duplicate workspace logic. See `PRDs/PR1.5 Go Handlers Imp Notes.md` for detailed implementation guidance.

---

*End of Plan*
