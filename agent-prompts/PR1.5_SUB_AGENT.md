# PR1.5 Implementation Agent

## Your Mission
Implement PR1.5: Tool Integration & Go HTTP Server for Chartsmith.

This is the **substantial Go work** that demonstrates learning for the Uncharted Territory challenge. You will build HTTP endpoints in Go that expose existing business logic to the new AI SDK chat system.

---

## Primary Documents (READ FIRST - IN ORDER)
1. `PRDs/PR1.5_PLAN.md` - Task breakdown and priority order
2. `PRDs/PR1.5_Tech_PRD.md` - Technical specification
3. `PRDs/PR1.5_Product_PRD.md` - Functional requirements
4. `PRDs/PR1.5_Go_Handlers_Imp_Notes.md` - **CRITICAL** - Go handler implementation with exact function names
5. `PRDs/CHARTSMITH_ARCHITECTURE_DECISIONS.md` - Architectural context

**Note**: `PRDs/` contains planning docs at workspace root. `chartsmith/` is the forked repo. All TypeScript goes in `chartsmith/chartsmith-app/`, all Go in `chartsmith/pkg/`.

## Codebase Reference (CRITICAL FOR GO WORK)
- `docs/research/2025-12-02-chartsmith-anthropic-tool-definitions.md` - **MUST READ** - Existing Go tool definitions to wrap
- `chartsmith/CLAUDE.md` - Codebase entry points and Go patterns
- `chartsmith/chartsmith-app/CLAUDE.md` - App-specific file locations
- `ClaudeResearch/CURRENT_STATE_ANALYSIS.md` - Current LLM integration details
- `memory-bank/systemPatterns.md` - Existing patterns to follow

---

## PR1 Context (ALREADY COMPLETE)

PR1 created these foundations you will build on:

### What Already Exists
| File | What It Does |
|------|--------------|
| `chartsmith-app/app/api/chat/route.ts` | API route using `streamText` - **YOU WILL ADD TOOLS HERE** |
| `chartsmith-app/lib/ai/provider.ts` | Provider factory with `getModel()`, `AVAILABLE_PROVIDERS` |
| `chartsmith-app/lib/ai/config.ts` | AI configuration, system prompt template |
| `chartsmith-app/lib/ai/models.ts` | Model definitions (Claude Sonnet 4 default) |
| `chartsmith-app/components/chat/AIChat.tsx` | Chat UI with `useChat` hook integration |
| `chartsmith-app/app/test-ai-chat/page.tsx` | Test page matching production UI - **USE THIS TO VALIDATE TOOLS** |

### Current Request Body Format
```typescript
// PR1 format (current)
interface ChatRequestBody {
  messages: UIMessage[];
  provider?: 'openai' | 'anthropic';
  model?: string;
}
```

### @anthropic-ai/sdk Status
**ALREADY REMOVED** in PR1. Do NOT spend time on this - it's done.

---

## What You Are Building

### TypeScript Files to Create (7 new)

| File | Purpose |
|------|---------|
| `chartsmith-app/lib/ai/llmClient.ts` | Shared client - IMPORTS from `provider.ts` (doesn't replace it) |
| `chartsmith-app/lib/ai/prompts.ts` | System prompts migrated from Go |
| `chartsmith-app/lib/ai/tools/getChartContext.ts` | TypeScript-only tool (calls getWorkspace() directly) |
| `chartsmith-app/lib/ai/tools/textEditor.ts` | Calls Go endpoint `/api/tools/editor` |
| `chartsmith-app/lib/ai/tools/latestSubchartVersion.ts` | Calls Go endpoint `/api/tools/versions/subchart` |
| `chartsmith-app/lib/ai/tools/latestKubernetesVersion.ts` | Calls Go endpoint `/api/tools/versions/kubernetes` |
| `chartsmith-app/lib/ai/tools/utils.ts` | Shared `callGoEndpoint` helper with auth header forwarding |

### Go Files to Create (4 new)

| File | Purpose |
|------|---------|
| `chartsmith/pkg/api/server.go` | HTTP server on port 8080 with router and 3 routes |
| `chartsmith/pkg/api/errors.go` | Standardized error responses (WriteError, WriteErrorWithCode) |
| `chartsmith/pkg/api/handlers/editor.go` | textEditor endpoint (view/create/str_replace) |
| `chartsmith/pkg/api/handlers/versions.go` | Both version endpoints |

### Files to Modify

| File | Changes |
|------|---------|
| `chartsmith-app/app/api/chat/route.ts` | Add 4 tools to `streamText`, add `workspaceId` to request body |
| `chartsmith/cmd/run.go` | Start HTTP server in goroutine alongside queue listener |
| `chartsmith-app/ARCHITECTURE.md` | Document new tool architecture (**MUST HAVE**) |

---

## Critical Architecture Points

### 1. getChartContext is TypeScript-ONLY
- Does NOT call a Go HTTP endpoint
- Directly calls existing `getWorkspace()` function from `@/lib/workspace/queries`
- Returns workspace files and chart metadata
- Discover the exact function signature by reading the workspace queries file

### 2. Request Body Change (PR1 → PR1.5)
- PR1 format: `{ messages, provider, model }`
- PR1.5 format: ADD `workspaceId` and `revisionNumber` fields
- Workspace context required for tool execution

### 3. Auth Header Forwarding Pattern
- Capture `Authorization` header from incoming request
- Pass auth header to tool factory functions via closure
- Tools use auth when calling Go endpoints via `callGoEndpoint` utility
- Pattern: `createTextEditorTool(authHeader, workspaceId)` returns configured tool

### 4. llmClient.ts and provider.ts Relationship
- llmClient.ts IMPORTS from provider.ts (does not replace it)
- Re-exports `getModel`, `AVAILABLE_PROVIDERS`, `Provider` from provider.ts
- Adds any shared utilities needed across tools
- Single source of truth for provider configuration remains in provider.ts

---

## Go Handler Implementation (CRITICAL)

### Reference: `PRDs/PR1.5_Go_Handlers_Imp_Notes.md`

**READ THIS FILE** - It contains the exact function names and signatures to reuse.

For each Go handler, **reuse existing production functions** - do NOT reimplement business logic:

### textEditor handler
- Supports 3 commands: `view`, `create`, `str_replace`
- Find existing functions in `pkg/workspace/` and `pkg/llm/` packages
- PR1.5_Go_Handlers_Imp_Notes.md lists exact function names
- Handle file listing, content retrieval, file creation, string replacement

### latestSubchartVersion handler
- Find existing function in `pkg/recommendations/` package
- Wraps existing ArtifactHub query logic
- PR1.5_Go_Handlers_Imp_Notes.md has the function name

### latestKubernetesVersion handler
- Hardcoded version values - search codebase for existing constants
- No external API calls needed
- Simple JSON response with version info

---

## Go HTTP Server Setup

### `pkg/api/server.go`
- Create HTTP server function that returns `*http.Server`
- Use chi router (already a dependency in the project - check go.mod)
- Register 3 POST routes under `/api/tools/`
- Add appropriate middleware (logging, recovery, CORS if needed)
- Listen on port 8080

### Starting the Server (`cmd/run.go`)
- Find existing worker startup code in `cmd/run.go`
- Start HTTP server in a goroutine alongside the queue listener
- Server runs concurrently with existing worker process
- Log server startup for debugging

---

## Error Response Contract

### Standard Error Format (Go → TypeScript)
All error responses follow this JSON structure:
- `success`: boolean (false for errors)
- `message`: human-readable error description
- `code`: machine-readable error code string

### Error Codes
| Code | HTTP Status | Description |
|------|-------------|-------------|
| `INVALID_COMMAND` | 400 | Unknown command type |
| `MISSING_PARAM` | 400 | Required parameter missing |
| `FILE_NOT_FOUND` | 404 | Requested file doesn't exist |
| `INTERNAL_ERROR` | 500 | Unexpected server error |

### `pkg/api/errors.go`
- Define `ErrorResponse` struct with JSON tags
- Create `WriteError(w, status, message, code)` helper function
- Set Content-Type header to application/json
- Encode and write error response

---

## Environment Variables

### Add to `.env` files
```env
GO_BACKEND_URL=http://localhost:8080
```

This is used by TypeScript tools to call Go endpoints.

---

## Implementation Order (If Time-Constrained)

| Priority | Task | Effort |
|----------|------|--------|
| 1 | Go HTTP server setup (`pkg/api/server.go`, `errors.go`) | 1 hour |
| 2 | `callGoEndpoint` utility in `lib/ai/tools/utils.ts` | 30 min |
| 3 | `getChartContext` tool (TypeScript-only) | 30 min |
| 4 | `textEditor` + Go handler | 2 hours |
| 5 | `latestSubchartVersion` + Go handler | 1 hour |
| 6 | `latestKubernetesVersion` + Go handler | 30 min |
| 7 | Register all 4 tools in `/api/chat/route.ts` | 30 min |
| 8 | System prompts (`lib/ai/prompts.ts`) | 1 hour |
| 9 | Integration tests | 1 hour |
| 10 | ARCHITECTURE.md update | 30 min |

---

## Success Criteria (All Must Pass)

- [ ] Go HTTP server starts on port 8080 alongside worker
- [ ] `POST /api/tools/editor` returns file content (view command)
- [ ] `POST /api/tools/editor` creates new file (create command)
- [ ] `POST /api/tools/editor` performs replacement (str_replace command)
- [ ] `POST /api/tools/versions/subchart` returns version data
- [ ] `POST /api/tools/versions/kubernetes` returns version data
- [ ] `getChartContext` returns workspace data via TypeScript only
- [ ] All 4 tools registered in `/api/chat/route.ts`
- [ ] Tools work when tested via `/test-ai-chat` page
- [ ] Error responses follow standard format
- [ ] ARCHITECTURE.md updated with tool architecture
- [ ] No TypeScript or Go compilation errors

---

## Testing Strategy

### 1. Go Endpoint Testing (Direct HTTP)
Use curl to test Go endpoints directly:
- POST to `/api/tools/editor` with command, path, workspaceId
- POST to `/api/tools/versions/subchart` with chartName
- POST to `/api/tools/versions/kubernetes` with no params
- Verify JSON responses and error handling

### 2. Tool Integration Testing (via Test Page)
1. Navigate to `http://localhost:3000/test-ai-chat`
2. Start a conversation with the AI
3. Ask questions that trigger each tool:
   - "What files are in my chart?" → `getChartContext`
   - "Show me the values.yaml file" → `textEditor` view
   - "What's the latest PostgreSQL subchart version?" → `latestSubchartVersion`
4. Verify tool results appear in conversation

### 3. Unit Tests
Create integration tests in `chartsmith-app/tests/integration/tools.test.ts` covering:
- Each tool's execute function
- Error handling for invalid inputs
- Response format validation

---

## DO NOT

- Reimplement existing Go functions (call them, don't copy them)
- Modify the provider factory structure (llmClient.ts imports from it)
- Remove @anthropic-ai/sdk (already done in PR1)
- Add tool UI rendering (that's post-PR1.5 integration work)
- Start PR2 without completing this work

---

## COMPLETION REQUIREMENTS

When you have completed ALL success criteria, create:

**File: `docs/PR1.5_COMPLETION_REPORT.md`**

### Required Sections:

#### 1. Files Summary Table
```markdown
| Action | File Path | Description |
|--------|-----------|-------------|
| Created | chartsmith/pkg/api/... | ... |
| Modified | chartsmith-app/app/api/chat/route.ts | ... |
```

#### 2. Go Code Summary
- Lines of new Go code written
- Existing functions reused (list them)
- HTTP endpoint signatures

#### 3. Tool Registration
Show how tools are registered in `route.ts`:
- List all 4 tool names and their factory functions
- Show how auth header and workspaceId are passed

#### 4. API Contracts
Document request/response format for each Go endpoint:
- `POST /api/tools/editor`
- `POST /api/tools/versions/subchart`
- `POST /api/tools/versions/kubernetes`

#### 5. Test Results
- Go endpoint curl test results
- Tool integration test results via `/test-ai-chat`
- Unit test results

#### 6. Deviations from Spec
Any changes made with rationale.

#### 7. PR2 Readiness Checklist
- [ ] Go HTTP server running on port 8080
- [ ] All 4 tools registered and functional
- [ ] `callGoEndpoint` utility working
- [ ] Tool → Go HTTP → Response pattern established
- [ ] Error responses standardized
- [ ] ARCHITECTURE.md updated

---

*When your completion report is ready, the Master Orchestrator will review it and provide the PR2 sub-agent prompt for the Validation Agent work.*

