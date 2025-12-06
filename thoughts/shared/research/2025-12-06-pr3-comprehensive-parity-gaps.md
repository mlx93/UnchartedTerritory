---
date: 2025-12-06T13:39:24-06:00
researcher: Claude
git_commit: 3eb3cfc
branch: main
repository: mlx93/UnchartedTerritory
topic: "PR3.0 Comprehensive Feature Parity Gaps - Complete Resolution List"
tags: [research, codebase, ai-sdk, plans, centrifugo, feature-parity, parity-gaps]
status: complete
last_updated: 2025-12-06
last_updated_by: Claude
---

# PR3.0 Comprehensive Feature Parity Gaps

**Date**: 2025-12-06 13:39:24 CST
**Researcher**: Claude
**Git Commit**: 3eb3cfc
**Branch**: main
**Repository**: mlx93/UnchartedTerritory

## Research Question

Compile a complete list of feature parity gaps between the AI SDK implementation and the expected functionality per the Replicated_Chartsmith.md requirements.

## Executive Summary

The AI SDK migration (PR3.0) has **two critical gaps** that cause all observed issues:

1. **Go Plan struct missing `BufferedToolCalls` field** - Plans sent via Centrifugo don't include buffered tool calls, causing the frontend to always use the legacy path
2. **Missing `chatmessage-updated` Centrifugo event** - After plan creation, the chat message's `responsePlanId` isn't broadcast, requiring page reload to see plans

These two root causes cascade into 5 observable symptoms. Fixing them will achieve feature parity.

---

## Complete Parity Gap List

| # | Gap | Severity | Root Cause | Files to Modify |
|---|-----|----------|------------|-----------------|
| **1** | Go Plan struct missing BufferedToolCalls | **CRITICAL** | Struct definition incomplete | `pkg/workspace/types/types.go` |
| **2** | Go GetPlan doesn't fetch buffered_tool_calls | **CRITICAL** | SQL query incomplete | `pkg/workspace/plan.go` |
| **3** | Missing chatmessage-updated event | **HIGH** | No event published after plan creation | `pkg/api/handlers/plan.go` |
| **4** | Terminal "new-chart" showing in chat | **MEDIUM** | Legacy render path triggered | (Resolves with #1-2) |
| **5** | Duplicate files in action list | **LOW** | AI makes multiple calls to same file | `pkg/api/handlers/plan.go` OR frontend |

---

## Gap 1: Go Plan Struct Missing BufferedToolCalls (CRITICAL)

### Current State

**File**: `pkg/workspace/types/types.go:68-79`

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
    // MISSING: BufferedToolCalls field
}
```

### Expected State (TypeScript)

**File**: `chartsmith-app/lib/types/workspace.ts:101-112`

```typescript
export interface Plan {
  id: string;
  description: string;
  status: string;
  workspaceId: string;
  chatMessageIds: string[];
  createdAt: Date;
  proceedAt?: Date;
  actionFiles: ActionFile[];
  bufferedToolCalls?: BufferedToolCall[];  // <-- EXISTS in TypeScript
}
```

### Impact

When `publishPlanUpdate()` sends plan data to Centrifugo:
1. Go's `GetPlan()` returns a Plan struct without `BufferedToolCalls`
2. JSON serialization omits the field entirely
3. Frontend receives plan without `bufferedToolCalls`
4. `PlanChatMessage.tsx:164` checks `plan.bufferedToolCalls && plan.bufferedToolCalls.length > 0`
5. Check fails → falls back to legacy `createRevisionAction`

### Required Change

```go
type Plan struct {
    // ... existing fields ...
    BufferedToolCalls json.RawMessage `json:"bufferedToolCalls,omitempty"`
}
```

---

## Gap 2: Go GetPlan Doesn't Fetch buffered_tool_calls (CRITICAL)

### Current State

**File**: `pkg/workspace/plan.go:144-154`

```sql
SELECT
    id,
    workspace_id,
    chat_message_ids,
    created_at,
    updated_at,
    version,
    status,
    description,
    proceed_at
    -- MISSING: buffered_tool_calls
FROM workspace_plan WHERE id = $1
```

### Database Schema Confirms Column Exists

**File**: `db/schema/tables/workspace-plan.yaml:42-44`

```yaml
- name: buffered_tool_calls
  type: jsonb
  default: "'[]'::jsonb"
```

### Impact

Even if Gap #1 is fixed (struct has the field), `GetPlan()` doesn't SELECT it:
1. Field remains nil/empty in returned Plan
2. Centrifugo event contains plan with empty `bufferedToolCalls`
3. Same cascade as Gap #1

### Required Change

```sql
SELECT
    id,
    workspace_id,
    chat_message_ids,
    created_at,
    updated_at,
    version,
    status,
    description,
    proceed_at,
    buffered_tool_calls  -- ADD THIS
FROM workspace_plan WHERE id = $1
```

And add scanning:
```go
var bufferedToolCalls []byte
err := row.Scan(
    // ... existing fields ...
    &bufferedToolCalls,
)
plan.BufferedToolCalls = bufferedToolCalls
```

---

## Gap 3: Missing chatmessage-updated Centrifugo Event (HIGH)

### Current State

**File**: `pkg/api/handlers/plan.go:44-103`

After creating a plan:
1. Database is updated: `workspace_chat.response_plan_id = planID`
2. **Only** `plan-updated` event is published
3. **No** `chatmessage-updated` event is published

```go
// Line 98 - only publishes plan update
publishPlanUpdate(ctx, req.WorkspaceID, planID)
```

### Impact

1. Frontend receives `plan-updated` → adds plan to `plansAtom`
2. But `messagesAtom` still has message with `responsePlanId: undefined`
3. `ChatMessage.tsx:159` checks `message?.responsePlanId`
4. Check fails → PlanChatMessage not rendered
5. User must refresh page to see the plan

### Frontend Is Ready For This

**File**: `hooks/useCentrifugo.ts:104-108`

```typescript
const isMetadataOnlyUpdate = chatMessage.responsePlanId || chatMessage.responseConversionId;

if (currentStreamingMessageId && chatMessage.id === currentStreamingMessageId && !isMetadataOnlyUpdate) {
  console.log('[Centrifugo] Skipping update for AI SDK streaming message:', chatMessage.id);
  return;
}
```

This code specifically allows `responsePlanId` updates during streaming, but the event is never sent.

### Required Change

After plan creation in `pkg/api/handlers/plan.go`, add:

```go
// After publishPlanUpdate at line 98
chatMessage, err := workspace.GetChatMessage(ctx, req.ChatMessageID)
if err == nil {
    userIDs, _ := workspace.ListUserIDsForWorkspace(ctx, req.WorkspaceID)
    event := &realtimetypes.ChatMessageUpdatedEvent{
        WorkspaceID: req.WorkspaceID,
        ChatMessage: chatMessage,
    }
    realtime.SendEvent(ctx, realtimetypes.Recipient{UserIDs: userIDs}, event)
}
```

---

## Gap 4: Terminal "new-chart" Showing in Chat (MEDIUM)

### Symptom

Black terminal box showing helm commands appears in chat instead of/alongside the plan.

### Root Cause

This is a **downstream effect** of Gaps #1-2:

1. `plan.bufferedToolCalls` is undefined in frontend
2. When user clicks Proceed → `createRevisionAction` (legacy path)
3. Legacy path triggers Go `apply_plan` worker
4. Go worker calls `EnqueueRenderWorkspaceForRevisionWithPendingContent`
5. Render job is created → sets `response_render_id` on message
6. `ChatMessage.tsx:178` renders Terminal when `responseRenderId` exists

### Evidence from Logs

```
INFO    New apply plan notification received  payload={"planId": "03ef1e6e4e8f"}
INFO    Processing action file  path=Chart.yaml action=update
"I need more information about what changes should be made"  <-- LLM asking for help!
```

The Go worker regenerates content via LLM because it doesn't have the buffered tool calls with actual content.

### Resolution

**Fixing Gaps #1-2 will resolve this.** Once `bufferedToolCalls` is available:
1. Frontend uses `proceedPlanAction` instead of `createRevisionAction`
2. `proceedPlanAction` executes buffered tool calls directly
3. No `apply_plan` worker triggered
4. No render job created
5. No terminal appears

---

## Gap 5: Duplicate Files in Action List (LOW)

### Symptom

Same file appears multiple times in plan's file list:
- `Chart.yaml` (duplicate)
- `templates/_helpers.tpl` (duplicate)

### Root Cause

AI makes multiple tool calls for the same file (e.g., `str_replace` called twice on Chart.yaml).

**File**: `pkg/api/handlers/plan.go` - `extractActionFiles()` function

The function has deduplication logic:
```go
seen := make(map[string]bool)
for _, tc := range toolCalls {
    if seen[path] { continue }
    // ...
}
```

But duplicates still appear, suggesting either:
1. Deduplication isn't working correctly
2. UI is displaying duplicates despite backend deduplication

### Resolution Options

1. **Verify Go deduplication** - Ensure `extractActionFiles()` properly deduplicates by path
2. **Add frontend deduplication** - Filter duplicate paths when rendering action files in PlanChatMessage

---

## Resolution Priority Order

### Phase 1: Critical (Required for Basic Functionality)

| Order | Gap | File | Change |
|-------|-----|------|--------|
| 1 | Add BufferedToolCalls to Plan struct | `pkg/workspace/types/types.go` | Add field |
| 2 | Update GetPlan to fetch buffered_tool_calls | `pkg/workspace/plan.go` | Update query and scanning |

**After Phase 1**: Plans will have `bufferedToolCalls` when sent via Centrifugo → Frontend will use AI SDK path → Proceed button will execute buffered tool calls directly.

### Phase 2: High Priority (Real-time Updates)

| Order | Gap | File | Change |
|-------|-----|------|--------|
| 3 | Publish chatmessage-updated event | `pkg/api/handlers/plan.go` | Add event publication |

**After Phase 2**: Plans will appear immediately without page reload.

### Phase 3: Polish (Optional)

| Order | Gap | File | Change |
|-------|-----|------|--------|
| 4 | Deduplicate action files | `pkg/api/handlers/plan.go` OR frontend | Verify/add deduplication |

---

## Data Flow: Current vs Expected

### Current (Broken) Flow

```
AI SDK streaming → Buffer tool calls → onFinish → createPlanFromToolCalls
                                                           ↓
                                        Go creates plan (buffered_tool_calls saved to DB)
                                                           ↓
                                        Go calls GetPlan (doesn't fetch buffered_tool_calls)
                                                           ↓
                                        publishPlanUpdate (plan without bufferedToolCalls)
                                                           ↓
                                        Centrifugo sends plan-updated event
                                                           ↓
                                        Frontend: plansAtom updated (no bufferedToolCalls)
                                                           ↓
                                        User clicks Proceed
                                                           ↓
                    plan.bufferedToolCalls?.length > 0 → FALSE (field is undefined!)
                                                           ↓
                                        Falls back to createRevisionAction (LEGACY)
                                                           ↓
                                        Go apply_plan worker triggered
                                                           ↓
                                        LLM regenerates content (no context!)
                                                           ↓
                    Creates render job → Terminal shows → "I need more information"
```

### Expected (Fixed) Flow

```
AI SDK streaming → Buffer tool calls → onFinish → createPlanFromToolCalls
                                                           ↓
                                        Go creates plan (buffered_tool_calls saved to DB)
                                                           ↓
                                        Go calls GetPlan (FETCHES buffered_tool_calls) ← FIX #1-2
                                                           ↓
                                        publishPlanUpdate (plan WITH bufferedToolCalls)
                                                           ↓
                                        Publishes chatmessage-updated event ← FIX #3
                                                           ↓
                                        Centrifugo sends plan-updated AND chatmessage-updated
                                                           ↓
                                        Frontend: plansAtom AND messagesAtom updated
                                                           ↓
                                        PlanChatMessage renders (responsePlanId set)
                                                           ↓
                                        User clicks Proceed
                                                           ↓
                    plan.bufferedToolCalls?.length > 0 → TRUE
                                                           ↓
                                        Uses proceedPlanAction (AI SDK path)
                                                           ↓
                                        Executes buffered tool calls directly
                                                           ↓
                    Files created/modified with actual content → Success!
```

---

## Alignment with Replicated_Chartsmith.md Requirements

Per the assignment document, the AI SDK migration should:

| Requirement | Current Status | After Fixes |
|-------------|----------------|-------------|
| Replace custom chat UI with Vercel AI SDK | ✅ Done | ✅ Done |
| Migrate from @anthropic-ai/sdk to AI SDK Core | ✅ Done | ✅ Done |
| Maintain all existing chat functionality | ❌ Plans don't show without reload | ✅ Fixed by Gap #3 |
| Keep existing system prompts and behavior | ✅ Done | ✅ Done |
| All existing features continue to work (tool calling, file context) | ❌ Legacy path regenerates content | ✅ Fixed by Gap #1-2 |
| Tests pass | Needs verification | Needs verification |

> Note: "The existing Go backend will remain, but may need API adjustments" - These fixes are exactly such adjustments. The Go backend is retained but needs to expose `buffered_tool_calls` through its API.

---

## Code References

### Files to Modify

| File | Gap | Current Line | Change |
|------|-----|--------------|--------|
| `pkg/workspace/types/types.go` | #1 | 68-79 | Add `BufferedToolCalls json.RawMessage` |
| `pkg/workspace/plan.go` | #2 | 144-154 | Add `buffered_tool_calls` to SELECT |
| `pkg/workspace/plan.go` | #2 | 161-171 | Add scanning for buffered_tool_calls |
| `pkg/api/handlers/plan.go` | #3 | 98 | Add chatmessage-updated event |

### Files That Already Work (No Changes Needed)

- `chartsmith-app/lib/types/workspace.ts:101-112` - TypeScript Plan type has `bufferedToolCalls`
- `chartsmith-app/lib/workspace/workspace.ts:518-544` - TypeScript `getPlan()` fetches column
- `chartsmith-app/components/PlanChatMessage.tsx:164` - Decision logic is correct
- `chartsmith-app/lib/workspace/actions/proceed-plan.ts` - Execution logic is correct
- `chartsmith-app/hooks/useCentrifugo.ts:104-108` - Metadata update handling is ready
- `chartsmith-app/lib/ai/tools/bufferedTools.ts` - Tool buffering works
- `chartsmith-app/app/api/chat/route.ts:215-240` - onFinish callback works

---

## Related Research

- `thoughts/shared/research/2025-12-05-pr3-feature-parity-gaps.md` - Initial research identifying the missing chatmessage-updated event
- `docs/issues/PR3.0_REMAINING_ISSUES.md` - Prior analysis identifying BufferedToolCalls as critical gap

---

## Open Questions

None - all gaps have been identified with clear resolution paths.
