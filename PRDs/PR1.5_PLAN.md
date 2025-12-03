# PR1.5 High-Level Plan

**Name**: Migration & Feature Parity
**Timeline**: Days 3-4
**Prerequisite**: PR1 merged
**Updated**: December 3, 2025 (revised to 4 tools based on codebase analysis)

---

## Core Architecture Decision

**Go does NOT need to make LLM calls after PR1.5.**

- All "thinking" (chat, reasoning, tool orchestration) → AI SDK Core (Node)
- Go becomes pure application logic (file ops, version lookups)
- No AI Gateway endpoint needed; no Go LLM client modifications needed

**Workspace Creation is NOT a Tool.**

- Workspaces are created by existing homepage TypeScript flow BEFORE AI SDK chat begins
- The AI SDK chat only needs tools for operations WITHIN an existing workspace
- No createChart, updateChart, or createEmptyWorkspace tools needed

**Rationale**: After deeper codebase analysis, we discovered that workspace creation happens via `createWorkspaceFromPromptAction()` on the homepage before the user is redirected to `/workspace/{id}`. By the time the AI SDK `/api/chat` route is invoked, the workspace already exists.

**After PR1.5, no user-facing chat UI paths route through the legacy Go LLM stack (`pkg/llm/*`). The only chat interface available in the product is the AI SDK-powered `/api/chat` route.**

---

## What PR1.5 Delivers

1. Shared LLM client library - Single source of truth for provider config
2. Go HTTP server - Exposes 3 endpoints for tool execution
3. **AI SDK tools (4 total)** - getChartContext (TypeScript-only), textEditor, latestSubchartVersion, latestKubernetesVersion
4. System prompts - Migrated from Go, updated for tool usage
5. **Full feature parity** - All existing tools wrapped
6. Anthropic SDK removal - No @anthropic-ai/sdk in TypeScript
7. **Standardized error handling** - Consistent error response contract across all Go endpoints

---

## MUST HAVE Tasks

### Task 1: Shared LLM Client Library
**File**: `lib/ai/llmClient.ts`

**IMPORTANT**: This file imports from PR1's `provider.ts` - it does NOT replace it.

- Import `getModel()` from `./provider` (PR1)
- Export `runChat()` function wrapping `streamText` with tools support
- Re-export `getModel()` and `AVAILABLE_PROVIDERS` for convenience
- Single place for chat + tools orchestration

### Task 2: Go HTTP Server Setup
**File**: `pkg/api/server.go`

- Create HTTP server on port 8080
- Register routes (3 total):
  - `/api/tools/editor` - File operations (textEditor)
  - `/api/tools/versions/subchart` - ArtifactHub subchart lookup
  - `/api/tools/versions/kubernetes` - Kubernetes version info
- Start alongside existing queue listener in `cmd/run.go`

**Note**: No chart creation endpoints - workspace creation is handled by existing homepage flow.

### Task 3: getChartContext Tool (TypeScript-Only)
**File**: `lib/ai/tools/getChartContext.ts`

Tool: Accept workspaceId; return workspace with files and metadata

```typescript
import { getWorkspace } from '@/lib/workspace/workspace'

export const getChartContext = tool({
  description: 'Get the current chart context including all files and metadata',
  parameters: z.object({
    workspaceId: z.string().describe('The workspace ID to load'),
  }),
  execute: async ({ workspaceId }) => {
    // Direct TypeScript call - no HTTP endpoint needed
    const workspace = await getWorkspace(workspaceId)
    return { success: true, workspace }
  },
})
```

**Note**: This tool does NOT call a Go HTTP endpoint. It directly calls the existing TypeScript `getWorkspace()` function.

### Task 4: textEditor Tool + Endpoint (Feature Parity - REQUIRED)
**Files**: `lib/ai/tools/textEditor.ts`, `pkg/api/handlers/editor.go`

Tool: Accept command (view/create/str_replace), path, content
Handler: Execute file operation, return result

```typescript
export const textEditor = tool({
  description: `View, edit, or create files in the chart.`,
  parameters: z.object({
    command: z.enum(['view', 'create', 'str_replace']),  // NO insert
    workspaceId: z.string(),
    path: z.string().describe('File path relative to chart root'),
    content: z.string().optional().describe('For create: full content'),
    oldStr: z.string().optional().describe('For str_replace: text to find'),
    newStr: z.string().optional().describe('For str_replace: replacement text'),
  }),
  execute: async (params) => callGoEndpoint('/api/tools/editor', params, authHeader),
})
```

**Implementation Note**: Reuse existing logic from `pkg/llm/execute-action.go` (`text_editor_20241022`).

### Task 5: latestSubchartVersion Tool + Endpoint (Feature Parity - REQUIRED)
**Files**: `lib/ai/tools/latestSubchartVersion.ts`, `pkg/api/handlers/versions.go`

Tool: Accept chart name, optional repository; return latest version from ArtifactHub

```typescript
export const latestSubchartVersion = tool({
  description: `Look up the latest version of a Helm subchart on ArtifactHub.`,
  parameters: z.object({
    chartName: z.string().describe('Name of the subchart (e.g., "postgresql", "redis")'),
    repository: z.string().optional().describe('Specific repo (defaults to ArtifactHub search)'),
  }),
  execute: async (params) => callGoEndpoint('/api/tools/versions/subchart', params, authHeader),
})
```

Go handler: `POST /api/tools/versions/subchart` - Query ArtifactHub API, return version info
**Implementation Note**: Port logic from existing `latest_subchart_version` tool in `pkg/llm/conversational.go`

### Task 6: latestKubernetesVersion Tool + Endpoint (Feature Parity - REQUIRED)
**Files**: `lib/ai/tools/latestKubernetesVersion.ts`, `pkg/api/handlers/versions.go`

Tool: Accept optional semverField; return Kubernetes version information

```typescript
export const latestKubernetesVersion = tool({
  description: `Get current Kubernetes version information.`,
  parameters: z.object({
    semverField: z.enum(['major', 'minor', 'patch']).optional()
      .describe('Version field (defaults to patch)'),
  }),
  execute: async (params) => callGoEndpoint('/api/tools/versions/kubernetes', params, authHeader),
})
```

Go handler: `POST /api/tools/versions/kubernetes` - Return hardcoded version values
**Implementation Note**: Port logic from existing `latest_kubernetes_version` tool in `pkg/llm/conversational.go`

### Task 7: Register Tools in /api/chat
**File**: `app/api/chat/route.ts` (modify)

**Request Body Change from PR1**:
- PR1: `{ messages, provider, model }`
- PR1.5: `{ messages, model, workspaceId, revisionNumber }`

**Route Handler Logic**:
1. Extract auth header from request
2. Parse body with new `workspaceId` and `revisionNumber` fields
3. Call `createTools(authHeader, workspaceId)` - auth passed via closure
4. Call `runChat()` with tools
5. Return streaming response

### Task 8: System Prompt Migration
**File**: `lib/ai/prompts.ts`

- Copy prompts from `pkg/llm/system.go`
- Create `getSystemPrompt()` with context parameter
- Document 4 tools and when to use each
- Note that workspace already exists when chat begins
- Reference existing <chartsmithArtifact> and <chartsmithActionPlan> tags

### Task 9: Error Response Contract
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

HTTP Status Code Mapping:
| Code | Status | Use Case |
|------|--------|----------|
| `VALIDATION_ERROR` | 400 | Invalid request parameters |
| `NOT_FOUND` | 404 | Workspace/chart doesn't exist |
| `DATABASE_ERROR` | 500 | DB connection or query failure |
| `EXTERNAL_API_ERROR` | 502 | ArtifactHub/K8s API failure |
| `INTERNAL_ERROR` | 500 | Unexpected server error |

### Task 10: Remove @anthropic-ai/sdk
- Remove from `package.json`
- Delete `lib/llm/prompt-type.ts`
- Verify no imports remain

### Task 11: Integration Test
**File**: `tests/integration/tools.test.ts`

- Test getChartContext tool (TypeScript-only)
- Test textEditor tool via Go endpoint
- Test version lookup tools via Go endpoints
- Assert error handling works correctly

---

### Task 12: Update ARCHITECTURE.md (MUST HAVE)
**File**: `chartsmith-app/ARCHITECTURE.md`

- Document the new AI SDK integration architecture
- Explain the tool → Go HTTP endpoint pattern
- Note that legacy `pkg/llm/*` is deprecated
- Include diagram showing Node/AI SDK Core + Go HTTP server

**Note**: This is a MUST HAVE per Replicated requirements: "Update ARCHITECTURE.md or chartsmith-app/ARCHITECTURE.md to reflect the new AI SDK integration"

---

## NICE TO HAVE Tasks

### Task 13: Deprecation Banner & Legacy Cleanup
- Add deprecation banner to legacy ChatContainer
- Mark `pkg/llm/*` files with legacy comments

---

## File Summary

### New Files (Next.js) - 7 files
| File | Priority |
|------|----------|
| `lib/ai/llmClient.ts` | MUST |
| `lib/ai/prompts.ts` | MUST |
| `lib/ai/tools/getChartContext.ts` | MUST (TypeScript-only) |
| `lib/ai/tools/textEditor.ts` | MUST |
| `lib/ai/tools/latestSubchartVersion.ts` | MUST |
| `lib/ai/tools/latestKubernetesVersion.ts` | MUST |
| `lib/ai/tools/utils.ts` | MUST |
| `tests/integration/tools.test.ts` | MUST |

### New Files (Go) - 4 files
| File | Priority |
|------|----------|
| `pkg/api/server.go` | MUST |
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
- [ ] getChartContext returns workspace data via TypeScript getWorkspace()
- [ ] textEditor view/create/str_replace work via Go HTTP endpoint
- [ ] **latestSubchartVersion tool returns valid version data** (feature parity)
- [ ] **latestKubernetesVersion tool returns valid version data** (feature parity)
- [ ] No @anthropic-ai/sdk in node_modules
- [ ] Integration test passes
- [ ] **Error responses follow standard format** (success, message, optional code/details)

### Quality Gates
- [ ] All **4 tools** implemented (getChartContext, textEditor, latestSubchartVersion, latestKubernetesVersion)
- [ ] getChartContext is TypeScript-only (no Go HTTP endpoint)
- [ ] System prompts guide tool usage (document all 4 tools)
- [ ] **ARCHITECTURE.md updated** (MUST HAVE per Replicated requirements)
- [ ] **Error handling utility (`callGoEndpoint`) shared across Go-backed tools**
- [ ] **All Go tool endpoints enforce existing workspace ownership and authentication middleware**

---

## What PR1.5 Enables for PR2

- Add `validateChart` to existing tools array
- Add `/api/validate` to existing Go HTTP server
- Reuse tool → Go HTTP → response pattern

---

## Risk Mitigation

**Priority order if time-constrained:**
1. llmClient.ts + tool registration (foundation)
2. Go HTTP server setup (3 endpoints)
3. getChartContext (TypeScript-only, core capability)
4. textEditor + Go endpoint (feature parity)
5. latestSubchartVersion + Go endpoint (feature parity)
6. latestKubernetesVersion + Go endpoint (feature parity)
7. System prompts
8. Error response contract (production quality)
9. Integration test
10. Remove @anthropic-ai/sdk
11. **ARCHITECTURE.md update** (MUST HAVE per Replicated requirements)
12. Deprecation banner (NICE TO HAVE)

**Minimum viable**: Tasks 1-10 (all MUST HAVE items)

**Existing tools that MUST be wrapped** (3 total - Go HTTP):
- `text_editor_20241022` → `textEditor`
- `latest_subchart_version` → `latestSubchartVersion`
- `latest_kubernetes_version` → `latestKubernetesVersion`

**New tools** (1 total - TypeScript-only):
- `getChartContext` (TypeScript-only, calls existing getWorkspace())

**Tools NOT needed** (workspace creation handled by homepage flow):
- `createChart` - Homepage TypeScript flow creates workspace before chat
- `updateChart` - Use iterative textEditor instead
- `createEmptyWorkspace` - Homepage TypeScript flow handles this

**Note on architecture**: The AI SDK chat only needs tools for operations WITHIN an existing workspace since workspace creation happens BEFORE chat begins (via createWorkspaceFromPromptAction on homepage).

---

## Security Note

All PR1.5 Go HTTP tool endpoints (`/api/tools/...`) will enforce existing workspace ownership and authentication middleware. Only the owner or authorized collaborators may modify or read workspace resources.

---

## Go Handler Implementation Guidance

The 3 Go HTTP handlers reuse existing production-stable functions:

**textEditor handler** (`pkg/api/handlers/editor.go`):
- `workspace.ListFiles(revision.ID)` - Get file list
- `workspace.GetFile(fileID)` - Get file content
- `workspace.SetFileContentPending(fileID, content)` - Update file
- `workspace.AddFileToChart(chartID, path, content)` - Create file
- `llm.PerformStringReplacement(content, oldStr, newStr)` - Fuzzy str_replace

**latestSubchartVersion handler** (`pkg/api/handlers/versions.go`):
- `recommendations.GetLatestSubchartVersion(chartName)` - ArtifactHub lookup

**latestKubernetesVersion handler** (`pkg/api/handlers/versions.go`):
- Hardcoded version values (no external calls)

We do not create any workspace creation handlers - that flow already works via homepage TypeScript.

---

*End of Plan*
