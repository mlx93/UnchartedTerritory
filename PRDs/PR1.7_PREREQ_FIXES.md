# PR1.7 Prerequisite Fixes

**Status**: Must complete before starting PR1.7
**Estimated Effort**: 2-4 hours total
**Last Updated**: December 5, 2025

---

## Bug 1: Double Processing (P0 - Critical)

### Problem
When user creates workspace from `/test-ai-chat` landing page, **both** Go worker AND AI SDK process the prompt, causing duplicate/conflicting file creation.

### Root Cause
`createWorkspaceFromPromptAction()` → `createChatMessage()` → `enqueueWork("new_intent")` triggers Go worker because no `knownIntent` is passed. Then test-ai-chat auto-sends same prompt to AI SDK.

### Recommended Fix (Option A)
Pass `knownIntent: ChatMessageIntent.NON_PLAN` to skip Go worker processing:

```typescript
// lib/workspace/actions/create-workspace-from-prompt.ts
import { ChatMessageIntent } from "../workspace";

const createChartMessageParams: CreateChatMessageParams = {
  prompt: prompt,
  messageFromPersona: ChatMessageFromPersona.AUTO,
  knownIntent: ChatMessageIntent.NON_PLAN,  // ADD THIS
}
```

### Files to Modify
- `chartsmith/chartsmith-app/lib/workspace/actions/create-workspace-from-prompt.ts`

### Verification
1. Navigate to `/test-ai-chat`
2. Enter prompt: "Create a simple nginx deployment"
3. Check Go logs - should NOT see `New plan notification received`
4. AI SDK should handle the request exclusively

---

## Bug 2: Inconsistent Content Handling (P1 - High)

### Problem
- `create` command writes directly to `content` column
- `str_replace` command writes to `content_pending` column

This breaks PR1.7's revision tracking model which assumes all AI changes go to `content_pending`.

### Root Cause
- `pkg/api/handlers/editor.go:156` calls `AddFileToChart()` which writes to `content`
- `pkg/api/handlers/editor.go:245` calls `SetFileContentPending()` which writes to `content_pending`

### Recommended Fix (Option A)
Modify `AddFileToChart()` to write to `content_pending` instead of `content`:

```go
// pkg/workspace/chart.go - AddFileToChart function
// Change INSERT to write to content_pending, set content to empty string

INSERT INTO workspace_file (id, revision_number, chart_id, workspace_id, file_path, content, content_pending)
VALUES ($1, $2, $3, $4, $5, '', $6)  // Note: content is empty, content_pending has the value
```

### Alternative (Option B)
Create a new `AddFileToChartPending()` function and call it from the textEditor handler for AI SDK path only. This preserves existing behavior for main path.

### Files to Modify
- `chartsmith/pkg/workspace/chart.go` - `AddFileToChart()` function

### Verification
1. Create a workspace via test-ai-chat
2. Ask AI to create a new file
3. Check database: file should have empty `content` and populated `content_pending`
4. File should appear in UI as "pending" (once PR1.7 pending UI is built)

---

## Non-Critical: Page Reload Guard (Optional)

### Problem
Page reload during streaming may re-trigger LLM if response not yet persisted.

### Current State
Existing guard `!lastUserMessage.response` is mostly sufficient. Only triggers if:
1. User reloads during streaming
2. Response persistence failed

### Recommendation
**Monitor only** - don't prioritize unless real user impact observed. Current implementation at `client.tsx:139` already checks for response presence.

---

## Summary

| Bug | Severity | Fix Complexity | Recommendation |
|-----|----------|----------------|----------------|
| Double Processing | P0 Critical | Low (1 line) | Fix immediately |
| Inconsistent Content | P1 High | Medium (Go change) | Fix before PR1.7 |
| Page Reload Guard | Low | Low | Monitor only |

---

## Next Steps

1. Fix Bug 1 (double processing) - ~30 min
2. Fix Bug 2 (content handling) - ~1-2 hours
3. Test both fixes together
4. Proceed with PR1.7 implementation
