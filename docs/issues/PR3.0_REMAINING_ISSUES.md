# PR3.0 Remaining Issues Analysis

**Date**: December 6, 2025  
**Branch**: `myles/vercel-ai-sdk-migration`  
**Status**: Analysis Complete - Fixes Pending

---

## 🚨 Critical Finding

**The Go `Plan` struct is missing the `BufferedToolCalls` field.**

This causes the frontend to always fall back to the legacy Go worker path (which regenerates content via LLM without context), even though `buffered_tool_calls` exists in the database.

**Location**: `pkg/workspace/types/types.go` line 68

---

## Issue Summary

| # | Issue | Severity | Root Cause Identified |
|---|-------|----------|----------------------|
| 1 | Terminal "new-chart" showing in chat | Medium | ✅ Yes |
| 2 | Refresh required to see plan/proceed button | High | ✅ Yes |
| 3 | Duplicate files in action list | Low | ✅ Yes |
| 4 | Chart.yaml/_helpers.tpl no actual changes | Critical | ✅ Yes |
| 5 | Yellow circles but no diff content | High | ✅ Yes |

---

## Issue 1: Terminal "new-chart" Showing in Chat Window

### Symptom
A terminal window showing `helm dependency update` and `helm template` commands appears in the chat area alongside the plan message.

### Evidence (from screenshots)
- Black terminal box with title "new-chart"
- Shows helm commands: `/usr/local/bin/helm dependency update .`
- Appears before/alongside the plan message

### Root Cause
In `components/ChatMessage.tsx` (lines 178-192), the Terminal component renders when:
```typescript
{message?.responseRenderId && !render?.isAutorender && (
  <Terminal ... />
)}
```

The message has `responseRenderId` set, which causes the Terminal to render. This happens because:
1. The legacy Go `apply_plan` worker is still being triggered
2. It calls `EnqueueRenderWorkspaceForRevisionWithPendingContent`
3. A render job is created and associated with the message

### Why This Shouldn't Happen in AI SDK Path
For AI SDK plans with buffered tool calls:
- Files should be created directly via `proceedPlanAction`
- No render should be triggered until user explicitly clicks Render
- Terminal view is for showing render output, not plan application

### Recommended Fix
1. Check if using AI SDK path and plan has `bufferedToolCalls`
2. Don't show Terminal for AI SDK plan messages until after explicit render
3. Or: Filter out auto-created renders from AI SDK plan flow

---

## Issue 2: Refresh Required to See Plan/Proceed Button

### Symptom
After AI SDK generates a response with a plan:
- Initially: No plan visible, only the "new-chart" terminal
- After refresh: Plan message appears with Proceed button

### Evidence (from screenshots)
- 2nd image shows plan appearing only after refresh
- Plan shows "(applied)" status even before user clicked Proceed

### Root Cause
In `hooks/useCentrifugo.ts` (lines 95-105), Centrifugo updates are skipped:
```typescript
if (currentStreamingMessageId && chatMessage.id === currentStreamingMessageId) {
  console.log('[Centrifugo] Skipping update for AI SDK streaming message:', chatMessage.id);
  return;
}
```

**Race Condition Timeline:**
1. AI SDK streaming finishes
2. `onFinish` callback runs → creates plan via Go backend
3. Go sets `response_plan_id` on chat message
4. Centrifugo publishes `chatmessage-updated` event
5. **BUT** React's `useEffect` to clear `currentStreamingMessageId` hasn't run yet
6. Event is **skipped** because message ID matches
7. User refreshes → message loads fresh from DB with `responsePlanId`

### Pending Fix (Already Implemented, Not Committed)
```typescript
// Allow metadata updates even for streaming messages
const isMetadataOnlyUpdate = chatMessage.responsePlanId || chatMessage.responseConversionId;
if (currentStreamingMessageId && chatMessage.id === currentStreamingMessageId && !isMetadataOnlyUpdate) {
  return; // Skip
}
```

### Recommended Fix
Commit the pending change in `useCentrifugo.ts`

---

## Issue 3: Duplicate Files in Action List

### Symptom
`templates/_helpers.tpl` appears multiple times in the plan's file list temporarily.

### Evidence (from screenshots)
- Plan shows:
  - Create file: `templates/deployment.yaml`
  - Modify file: `Chart.yaml`
  - Modify file: `Chart.yaml` (duplicate!)
  - Modify file: `templates/_helpers.tpl`

### Root Cause
In `pkg/api/handlers/plan.go`, the `extractActionFiles()` function should deduplicate by path, but the AI made multiple tool calls for the same file:

```go
func extractActionFiles(toolCalls []ToolCall) []workspacetypes.ActionFileInput {
    var files []workspacetypes.ActionFileInput
    seen := make(map[string]bool)
    for _, tc := range toolCalls {
        if seen[path] { continue }  // This exists but...
        // The issue is multiple buffered calls for same file
    }
}
```

The AI SDK buffered tool calls include multiple operations on the same file (e.g., `str_replace` called twice on Chart.yaml).

### Recommended Fix
1. Deduplicate in the Go handler (already present, may need verification)
2. Or: UI should deduplicate when displaying action files
3. Check if the AI is making redundant tool calls

---

## Issue 4: Chart.yaml and _helpers.tpl Don't Show Actual Changes

### Symptom
- `templates/deployment.yaml` shows "+61" lines (content was created)
- `Chart.yaml` shows yellow dot but no actual changes
- `templates/_helpers.tpl` shows yellow dot but no actual changes

### Critical Evidence (from logs)
```
INFO    Processing action file  path=Chart.yaml action=update index=1 total=3
DEBUG   update file  path=Chart.yaml
INFO    LLM text_editor tool use  command=view path=Chart.yaml

"Since no specific plan or requirements were provided in your message, 
I need more information about what changes should be made to the Chart.yaml file."
```

### Root Cause: LEGACY PATH IS STILL BEING TRIGGERED

**This is the critical bug.** When the user clicks "Proceed":
1. **Expected (AI SDK path)**: `proceedPlanAction` executes `buffered_tool_calls` which contain actual content
2. **Actual (Legacy path)**: `createRevisionAction` triggers Go `apply_plan` worker which:
   - Reads action files (just paths, no content)
   - Calls LLM to regenerate content
   - LLM has no context → asks "what changes should be made?"
   - File is marked as "created" but content is empty/unchanged

### Evidence from Logs
```
INFO    New apply plan notification received  payload={"planId": "03ef1e6e4e8f"}
INFO    Processing action file  path=Chart.yaml action=update
"I need more information about what changes should be made"  <-- LLM asking for help!
```

The legacy Go worker doesn't have access to the buffered tool calls with actual content.

### Why PlanChatMessage Wiring Fix Isn't Working
The fix in `PlanChatMessage.tsx` checks:
```typescript
if (plan.bufferedToolCalls && plan.bufferedToolCalls.length > 0) {
  await proceedPlanAction(session, plan.id, wsId, revisionNumber);
} else {
  await createRevisionAction(session, plan.id);  // LEGACY - THIS IS BEING HIT!
}
```

### ROOT CAUSE FOUND: Go Plan Type Missing BufferedToolCalls

**File**: `pkg/workspace/types/types.go` (lines 68-79)

```go
type Plan struct {
    ID             string       `json:"id"`
    WorkspaceID    string       `json:"workspaceId"`
    ChatMessageIDs []string     `json:"chatMessageIds"`
    Description    string       `json:"description"`
    CreatedAt      time.Time    `json:"createdAt"`
    UpdatedAt      time.Time    `json:"-"`
    Version        int          `json:"version"`
    Status         PlanStatus   `json:"status"`
    ActionFiles    []ActionFile `json:"actionFiles"`
    ProceedAt      *time.Time   `json:"proceedAt"`
    // MISSING: BufferedToolCalls field!
}
```

**Consequence:**
1. Go creates plan with `buffered_tool_calls` in DB ✅
2. Go sends `plan-updated` Centrifugo event
3. Event contains Plan struct WITHOUT `bufferedToolCalls` ❌
4. Frontend receives plan, stores in Jotai atom
5. `PlanChatMessage` reads from atom → `bufferedToolCalls` is undefined
6. Falls back to legacy path → Go worker regenerates content

### Required Fix
Add `BufferedToolCalls` to Go Plan struct:
```go
type Plan struct {
    // ... existing fields ...
    BufferedToolCalls json.RawMessage `json:"bufferedToolCalls,omitempty"`
}
```

And update `workspace.GetPlan()` to fetch and populate this field.

---

## Issue 5: Yellow Circles But No Diff Content

### Symptom
- Files show yellow indicator (pending changes)
- Source view shows "Showing 3/3 files with diffs"
- Accept/Reject buttons visible
- But actual diff (red/green lines) not showing

### Root Cause
This is a consequence of Issue #4:
1. Files are marked as having `contentPending`
2. But the actual content didn't change (LLM asked for requirements instead)
3. Diff shows nothing because old content === new content

### Evidence
- `deployment.yaml` shows +61 (LLM actually created content for this one)
- `Chart.yaml` and `_helpers.tpl` show no diff (LLM didn't change them)

### Recommended Fix
Fix Issue #4 first - once `proceedPlanAction` is properly called with buffered tool calls, the actual content will be applied.

---

## Summary: Primary Root Cause

**The legacy Go `apply_plan` worker is being triggered instead of `proceedPlanAction`.**

This causes:
- LLM regeneration without context
- Empty changes for update operations
- Render jobs being created
- Terminal showing in chat

### Evidence Trail
1. Logs show `apply_plan` queue being processed
2. Logs show LLM saying "I need more information"
3. Only `deployment.yaml` (create) has content - because LLM can create without context
4. `Chart.yaml` and `_helpers.tpl` (update) have no changes - LLM can't update without knowing what to change

---

## Recommended Actions (Priority Order)

### 1. Add BufferedToolCalls to Go Plan Struct (CRITICAL)
**File**: `pkg/workspace/types/types.go`
```go
type Plan struct {
    // ... existing fields ...
    BufferedToolCalls json.RawMessage `json:"bufferedToolCalls,omitempty"`
}
```

### 2. Update Go GetPlan to Populate BufferedToolCalls (CRITICAL)
**File**: `pkg/workspace/plan.go`
Ensure `GetPlan()` fetches and includes `buffered_tool_calls` from DB.

### 3. Commit Centrifugo Fix (HIGH)
Commit the pending change in `useCentrifugo.ts` for real-time plan display.

### 4. Hide Terminal for AI SDK Plans (MEDIUM)
In `ChatMessage.tsx`, add condition to not show Terminal when plan has buffered tool calls:
```typescript
{message?.responseRenderId && !render?.isAutorender && !hasBufferedToolCalls && (
  <Terminal ... />
)}
```

### 5. Deduplicate Action Files in UI (LOW)
Filter duplicate paths when rendering plan action files in `PlanChatMessage.tsx`.

---

## Files Involved

| File | Issue(s) |
|------|----------|
| **`pkg/workspace/types/types.go`** | **#4 - CRITICAL: Missing BufferedToolCalls field** |
| **`pkg/workspace/plan.go`** | **#4 - GetPlan needs to populate BufferedToolCalls** |
| `components/PlanChatMessage.tsx` | #4 - Path selection logic (working, but data missing) |
| `components/ChatMessage.tsx` | #1 - Terminal rendering condition |
| `hooks/useCentrifugo.ts` | #2 - Skipping metadata updates (fix pending) |
| `lib/workspace/workspace.ts` | #4 - TS `getPlan()` query (correct) |
| `lib/types/workspace.ts` | #4 - TS Plan type (correct) |
| `pkg/api/handlers/plan.go` | #3 - Action file extraction |

---

*Generated: December 6, 2025*

