# PR1.7 Prerequisite Fixes - Implementation Plan

## Overview

This plan addresses two critical bugs that must be fixed before starting PR1.7 (revision tracking). Both bugs relate to how content flows through the system when using the AI SDK path (`/test-ai-chat`).

## Current State Analysis

### Bug 1: Double Processing
- **Location**: `chartsmith-app/lib/workspace/actions/create-workspace-from-prompt.ts:11-14`
- **Issue**: When creating a workspace from `/test-ai-chat`, the `createChartMessageParams` does not include `knownIntent`, causing `createChatMessage()` to enqueue `"new_intent"` work for the Go worker. The AI SDK also processes the same prompt, leading to duplicate/conflicting file creation.

### Bug 2: Inconsistent Content Handling
- **Location**: `chartsmith/pkg/workspace/chart.go:38-54` and `chartsmith/pkg/api/handlers/editor.go:156`
- **Issue**: The `create` command writes to `content` column directly via `AddFileToChart()`, while `str_replace` writes to `content_pending` via `SetFileContentPending()`. This breaks PR1.7's revision tracking model which assumes all AI changes go to `content_pending`.

### Key Discoveries:
- `ChatMessageIntent.NON_PLAN` exists at `workspace.ts:143` with value `"non-plan"`
- When `knownIntent === NON_PLAN`, the system sends `pg_notify('new_nonplan_chat_message', chatMessageId)` instead of enqueueing Go worker
- `AddFileToChart()` uses direct INSERT to `content` column (no `content_pending`)
- `SetFileContentPending()` correctly writes to `content_pending` with proper transaction handling

### Critical Finding: AddFileToChart Callers

**There are 3 callers of `AddFileToChart()`:**

1. **`pkg/api/handlers/editor.go:156`** - AI SDK textEditor `create` command
   - **Wants**: `content_pending` (this is the bug we're fixing) ✅

2. **`pkg/listener/apply-plan.go:300`** - Main Go worker plan execution
   - Passes empty string `""` as content, then streams to `content_pending` via realtime
   - **Safe to change**: ✅ (passes empty string anyway)

3. **`pkg/listener/conversion-simplify.go:54,57,62`** - K8s to Helm conversion
   - Writes actual file content (values.yaml, Chart.yaml, templates)
   - Then calls `SetCurrentRevision` which does NOT commit `content_pending` to `content`
   - **helm-utils/render-exec.go:117** uses `file.Content` for rendering, NOT `file.ContentPending`
   - **WOULD BREAK** if we change to write to `content_pending` ❌

### Recommendation Change

**Original PRD Option A** (modify `AddFileToChart`) would break the conversion-simplify path.

**New Recommendation: Option B** - Create a new `AddFileToChartPending()` function for the AI SDK path only.

## Desired End State

After completing these fixes:
1. Workspaces created via `/test-ai-chat` will only be processed by the AI SDK (not Go worker)
2. All AI-generated file content via textEditor `create` will go to `content_pending` column
3. Conversion-simplify and apply-plan paths continue to work unchanged
4. PR1.7 revision tracking can reliably use `content_pending` for AI SDK changes

## What We're NOT Doing

- Not modifying the main Go worker path
- Not changing how `str_replace` command works
- Not adding new database columns or migrations
- Not implementing the PR1.7 pending UI (that's separate)
- Not changing conversion-simplify behavior
- Not addressing the "Page Reload Guard" issue (monitoring only per PRD)

---

## Phase 1: Fix Double Processing (Bug 1)

### Overview
Add `knownIntent: ChatMessageIntent.NON_PLAN` to the `createWorkspaceFromPromptAction()` to prevent Go worker from processing prompts that will be handled by AI SDK.

### Changes Required:

#### 1.1 Update createWorkspaceFromPromptAction

**File**: `chartsmith/chartsmith-app/lib/workspace/actions/create-workspace-from-prompt.ts`

**Change**: Add `ChatMessageIntent` to imports and add `knownIntent` to params object.

**Current code (lines 4, 11-14):**
```typescript
import { ChatMessageFromPersona, CreateChatMessageParams, createWorkspace } from "../workspace";

const createChartMessageParams: CreateChatMessageParams = {
  prompt: prompt,
  messageFromPersona: ChatMessageFromPersona.AUTO,
}
```

**New code:**
```typescript
import { ChatMessageFromPersona, ChatMessageIntent, CreateChatMessageParams, createWorkspace } from "../workspace";

const createChartMessageParams: CreateChatMessageParams = {
  prompt: prompt,
  messageFromPersona: ChatMessageFromPersona.AUTO,
  knownIntent: ChatMessageIntent.NON_PLAN,
}
```

### Success Criteria:

#### Automated Verification:
- [x] TypeScript compiles without errors: `cd chartsmith/chartsmith-app && npm run build`
- [x] Linting passes: `cd chartsmith/chartsmith-app && npm run lint`

#### Manual Verification:
- [x] Navigate to `/test-ai-chat`
- [x] Enter prompt: "Create a simple nginx deployment"
- [x] Check Go worker logs - should NOT see `New plan notification received`
- [x] AI SDK should handle the request exclusively
- [x] Workspace should be created successfully with AI response

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to Phase 2.

---

## Phase 2: Fix Inconsistent Content Handling (Bug 2)

### Overview
Create a new `AddFileToChartPending()` function that writes to `content_pending` instead of `content`, and update the editor handler to use it.

### Changes Required:

#### 2.1 Add AddFileToChartPending Function

**File**: `chartsmith/pkg/workspace/chart.go`

**Change**: Add a new function after `AddFileToChart()` (after line 54).

**New function to add:**
```go
// AddFileToChartPending creates a new file with content in content_pending column.
// Use this for AI SDK paths where content should be staged for review before committing.
func AddFileToChartPending(ctx context.Context, chartID string, workspaceID string, revisionNumber int, path string, contentPending string) error {
	conn := persistence.MustGetPooledPostgresSession()
	defer conn.Release()

	fileID, err := securerandom.Hex(12)
	if err != nil {
		return fmt.Errorf("failed to generate random ID: %w", err)
	}

	query := `INSERT INTO workspace_file (id, revision_number, chart_id, workspace_id, file_path, content, content_pending) VALUES ($1, $2, $3, $4, $5, '', $6)`
	_, err = conn.Exec(ctx, query, fileID, revisionNumber, chartID, workspaceID, path, contentPending)
	if err != nil {
		return fmt.Errorf("failed to insert file: %w", err)
	}

	return nil
}
```

**Key differences from `AddFileToChart()`:**
- Function name indicates "Pending" behavior
- Parameter named `contentPending` instead of `content`
- SQL sets `content = ''` and `content_pending = $6`
- Comment documents when to use this vs `AddFileToChart()`

#### 2.2 Update Editor Handler to Use New Function

**File**: `chartsmith/pkg/api/handlers/editor.go`

**Change**: In `handleCreate()`, replace `AddFileToChart` with `AddFileToChartPending` at line 156.

**Current code (line 156):**
```go
err = workspace.AddFileToChart(ctx, chartID, req.WorkspaceID, req.RevisionNumber, req.Path, req.Content)
```

**New code:**
```go
err = workspace.AddFileToChartPending(ctx, chartID, req.WorkspaceID, req.RevisionNumber, req.Path, req.Content)
```

### Success Criteria:

#### Automated Verification:
- [x] Go builds without errors: `cd chartsmith && go build ./...`
- [x] Go tests pass: `cd chartsmith && go test ./pkg/workspace/...` (no test files)
- [x] Go tests pass: `cd chartsmith && go test ./pkg/api/handlers/...` (no test files)
- [x] Linting passes: `cd chartsmith && golangci-lint run` (skipped - not installed)

#### Manual Verification:
- [ ] Create a workspace via `/test-ai-chat`
- [ ] Ask AI to create a new file (e.g., "Create a values.yaml file")
- [ ] Check database: `SELECT file_path, content, content_pending FROM workspace_file WHERE workspace_id = '<id>' ORDER BY id DESC LIMIT 5;`
- [ ] Verify: new file should have empty `content` and populated `content_pending`
- [ ] Verify: existing file modification via `str_replace` still works correctly
- [ ] **Test conversion path**: Create a workspace via K8s to Helm conversion
- [ ] Verify: conversion files have populated `content` column (not empty)
- [ ] Verify: conversion workspace renders correctly

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation that both fixes work together correctly.

---

## Testing Strategy

### Unit Tests:
- No new unit tests strictly required; existing tests should continue to pass
- Consider adding a test for `AddFileToChartPending` to verify it writes to correct column

### Integration Tests:
- Test workspace creation from `/test-ai-chat` end-to-end
- Test file creation and modification through AI SDK
- Test K8s to Helm conversion still works

### Manual Testing Steps:
1. Start fresh with database (or identify a clean workspace)
2. Navigate to `/test-ai-chat`
3. Create workspace with prompt: "Create a simple nginx deployment"
4. Verify Go logs don't show plan notification
5. Verify AI SDK creates files in `content_pending` column
6. Ask AI to modify an existing file with `str_replace`
7. Verify modification also goes to `content_pending`
8. Test that view command shows pending content correctly
9. **Test conversion**: Create K8s to Helm conversion workspace
10. Verify conversion files have `content` populated (not empty)
11. Verify conversion workspace renders correctly

---

## Risk Assessment

### Bug 1 Fix (Low Risk):
- Single line addition to TypeScript
- `ChatMessageIntent.NON_PLAN` is already a valid, used enum value
- No database changes
- Easy to revert if issues arise

### Bug 2 Fix (Low Risk with Option B):
- Adds new function, doesn't modify existing `AddFileToChart()`
- Only changes one call site in editor.go
- Conversion-simplify and apply-plan paths unchanged
- Easy to revert if issues arise

### Caller Analysis Summary:

| Caller | Path | Current Function | After Fix |
|--------|------|-----------------|-----------|
| `editor.go:156` | AI SDK textEditor `create` | `AddFileToChart` | `AddFileToChartPending` |
| `apply-plan.go:300` | Go worker plan execution | `AddFileToChart` | `AddFileToChart` (unchanged) |
| `conversion-simplify.go:54,57,62` | K8s to Helm conversion | `AddFileToChart` | `AddFileToChart` (unchanged) |

---

## References

- Original PRD: `PRDs/PR1.7_PREREQ_FIXES.md`
- Related Tech PRD: `PRDs/PR1.7_Tech_PRD.md`
- workspace.ts enums: `chartsmith-app/lib/workspace/workspace.ts:141-146`
- AddFileToChart function: `chartsmith/pkg/workspace/chart.go:38-54`
- SetFileContentPending function: `chartsmith/pkg/workspace/file.go:107-165`
- Editor handler: `chartsmith/pkg/api/handlers/editor.go:118-168`
- Conversion simplify: `chartsmith/pkg/listener/conversion-simplify.go:54-65`
- Apply plan: `chartsmith/pkg/listener/apply-plan.go:300`
- Helm render (uses Content): `chartsmith/helm-utils/render-exec.go:117`
