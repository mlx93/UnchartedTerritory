# PR3.0 Buffered Tool Calls Fix Implementation

**Date**: December 6, 2025  
**Status**: Complete ✅

## Overview

This fix enables the AI SDK path for plan execution by ensuring the Go backend properly returns `bufferedToolCalls` data and publishes the necessary Centrifugo events. The TypeScript frontend was already correctly implemented—these backend changes complete the integration.

## Problem Statement

The AI SDK streaming flow buffers tool calls and stores them in the database via the `buffered_tool_calls` column. However, when `publishPlanUpdate()` sent plan data through Centrifugo:

1. The Go `Plan` struct had no `BufferedToolCalls` field
2. `GetPlan()` didn't SELECT the `buffered_tool_calls` column from the database
3. No `chatmessage-updated` event was published, so the frontend's message atom never received the `responsePlanId`

This caused the frontend to always fall back to `createRevisionAction` (legacy path) instead of using `proceedPlanAction` (AI SDK path).

## Changes Made

### Phase 1: Added `BufferedToolCalls` Field to Plan Struct

**File**: `chartsmith/pkg/workspace/types/types.go`

- Added `"encoding/json"` to imports
- Added `BufferedToolCalls json.RawMessage` field to the `Plan` struct

```go
type Plan struct {
    ID                string          `json:"id"`
    WorkspaceID       string          `json:"workspaceId"`
    ChatMessageIDs    []string        `json:"chatMessageIds"`
    Description       string          `json:"description"`
    CreatedAt         time.Time       `json:"createdAt"`
    UpdatedAt         time.Time       `json:"-"`
    Version           int             `json:"version"`
    Status            PlanStatus      `json:"status"`
    ActionFiles       []ActionFile    `json:"actionFiles"`
    BufferedToolCalls json.RawMessage `json:"bufferedToolCalls,omitempty"`
    ProceedAt         *time.Time      `json:"proceedAt"`
}
```

**Rationale**: Using `json.RawMessage` preserves the JSON structure from the database without needing to define Go structs for the tool call format. The `omitempty` tag ensures legacy plans serialize cleanly with an absent field rather than `null`.

### Phase 2: Updated `GetPlan` to Fetch `buffered_tool_calls`

**File**: `chartsmith/pkg/workspace/plan.go`

- Updated the SQL query to include `buffered_tool_calls` column
- Added a `[]byte` variable to receive the data
- Updated the `Scan` call to include the new field
- Assigned the scanned value to `plan.BufferedToolCalls`

```go
query := `SELECT
    id, workspace_id, chat_message_ids, created_at, updated_at,
    version, status, description, proceed_at, buffered_tool_calls
FROM workspace_plan WHERE id = $1`

var bufferedToolCalls []byte
err := row.Scan(
    // ... existing fields ...
    &bufferedToolCalls,
)
plan.BufferedToolCalls = bufferedToolCalls
```

### Phase 3: Added `publishChatMessageUpdate` Helper Function

**File**: `chartsmith/pkg/api/handlers/plan.go`

Added a new helper function that:
1. Fetches the chat message via `workspace.GetChatMessage()`
2. Gets user IDs via `workspace.ListUserIDsForWorkspace()`
3. Creates a `ChatMessageUpdatedEvent`
4. Sends it via `realtime.SendEvent()`

```go
func publishChatMessageUpdate(ctx context.Context, workspaceID, chatMessageID string) {
    chatMessage, err := workspace.GetChatMessage(ctx, chatMessageID)
    if err != nil {
        logger.Errorf("Failed to get chat message for update event: %v", err)
        return
    }

    userIDs, err := workspace.ListUserIDsForWorkspace(ctx, workspaceID)
    if err != nil {
        logger.Errorf("Failed to get user IDs for chat message update: %v", err)
        return
    }

    recipient := realtimetypes.Recipient{UserIDs: userIDs}
    event := &realtimetypes.ChatMessageUpdatedEvent{
        WorkspaceID: workspaceID,
        ChatMessage: chatMessage,
    }

    if err := realtime.SendEvent(ctx, recipient, event); err != nil {
        logger.Errorf("Failed to send chat message update event: %v", err)
    }
}
```

This function is called after `publishPlanUpdate()` in `CreatePlanFromToolCalls`.

## Verification

- ✅ Go code compiles: `go build ./...` succeeded
- ✅ No linter errors
- ✅ Tests pass

## Expected Behavior After Fix

The frontend will now receive:
1. **`plan-updated`** event with `bufferedToolCalls` populated
2. **`chatmessage-updated`** event with `responsePlanId` set

This enables:
- Plans appear immediately in chat without page refresh
- "Proceed" button uses `proceedPlanAction` (AI SDK path)
- No terminal box appears (which would indicate legacy path)
- Files are created/modified with correct content from buffered tool calls

## Manual Testing Steps

1. Start the development environment
2. Navigate to the chat interface
3. Send a message that creates a plan (e.g., "create a helm chart with nginx")
4. Observe:
   - Plan should appear immediately without refresh
   - Browser devtools Network tab should show:
     - `plan-updated` event with `bufferedToolCalls` array
     - `chatmessage-updated` event with `responsePlanId` set
5. Click "Proceed" and observe:
   - No black terminal box should appear
   - Files should be created with correct content
   - Plan status should transition through applying → applied

## References

- Implementation plan: `thoughts/shared/plans/2025-12-06-pr3-feature-parity-fixes.md`
- Research documentation: `thoughts/shared/research/2025-12-06-pr3-comprehensive-parity-gaps.md`
- Database schema: `chartsmith/db/schema/tables/workspace-plan.yaml`
- TypeScript Plan type: `chartsmith-app/lib/types/workspace.ts`
- Frontend decision logic: `chartsmith-app/components/PlanChatMessage.tsx`

