# PR3.0 Feature Parity Fixes Implementation Plan

## Overview

This plan implements three targeted Go backend changes to achieve feature parity between the AI SDK migration (PR3.0) and the expected functionality. The TypeScript frontend is already correctly implemented - these changes make the Go backend return the necessary data so the frontend can use the AI SDK path instead of falling back to the legacy path.

## Current State Analysis

### Problem Summary

The AI SDK streaming flow buffers tool calls and stores them in the database via `buffered_tool_calls` column. However, when `publishPlanUpdate()` sends plan data through Centrifugo:

1. The Go `Plan` struct has no `BufferedToolCalls` field
2. `GetPlan()` doesn't SELECT the `buffered_tool_calls` column from the database
3. No `chatmessage-updated` event is published, so the frontend's message atom never gets the `responsePlanId`

This causes the frontend to always fall back to `createRevisionAction` (legacy path) instead of using `proceedPlanAction` (AI SDK path).

### Key Discoveries

- **Database schema already has the column**: `chartsmith/db/schema/tables/workspace-plan.yaml:42-44` defines `buffered_tool_calls` as jsonb
- **TypeScript Plan type already has the field**: `chartsmith-app/lib/types/workspace.ts:111` has `bufferedToolCalls?: BufferedToolCall[]`
- **TypeScript getPlan() already queries the column**: `chartsmith-app/lib/workspace/workspace.ts:521` fetches `buffered_tool_calls`
- **Frontend decision logic is correct**: `chartsmith-app/components/PlanChatMessage.tsx:164` checks `plan.bufferedToolCalls && plan.bufferedToolCalls.length > 0`
- **ChatMessageUpdatedEvent already exists**: `chartsmith/pkg/realtime/types/chatmessage-updated.go` provides the event structure
- **GetChatMessage() is available**: `chartsmith/pkg/workspace/chatmessage.go:74` returns `*types.Chat`
- **ListUserIDsForWorkspace() is available**: `chartsmith/pkg/workspace/workspace.go:14`

## Desired End State

After these changes:
1. Plans sent via Centrifugo will include `bufferedToolCalls` field with the stored tool calls
2. When a plan is created, a `chatmessage-updated` event broadcasts the updated message with `responsePlanId`
3. The frontend will receive plans with `bufferedToolCalls` populated
4. Clicking "Proceed" will use `proceedPlanAction` (AI SDK path) instead of `createRevisionAction` (legacy)
5. Plans will appear immediately without page reload

### Verification

1. Create a new chart via chat, which creates a plan with buffered tool calls
2. Without refreshing, the plan should appear in chat with the Proceed button
3. Click "Proceed" - the action should execute the buffered tool calls directly
4. No terminal box should appear (which would indicate the legacy path was triggered)
5. Files should be created/modified with the correct content from the buffered tool calls

## What We're NOT Doing

- No changes to TypeScript code (already correctly implemented)
- No changes to the database schema (column already exists)
- No changes to the plan creation logic in `pkg/api/handlers/plan.go:createPlanWithBufferedTools` (already stores buffered_tool_calls)
- No changes to `listPlans()` function (not used in real-time flow)

## Implementation Approach

Three sequential changes, each building on the previous:

1. **Phase 1**: Add `BufferedToolCalls` field to Go Plan struct
2. **Phase 2**: Update `GetPlan()` to SELECT and scan `buffered_tool_calls`
3. **Phase 3**: Publish `chatmessage-updated` event after plan creation

---

## Phase 1: Add BufferedToolCalls Field to Plan Struct

### Overview

Add the `BufferedToolCalls` field to the Plan struct so it can be serialized to JSON when sent via Centrifugo.

### Changes Required

#### 1.1 Update Plan Struct

**File**: `chartsmith/pkg/workspace/types/types.go`
**Lines**: 68-79

Add `json.RawMessage` import and `BufferedToolCalls` field:

```go
package types

import (
	"encoding/json"
	"time"
)
```

Then update the Plan struct:

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
	ProceedAt         *time.Time      `json:"proceedAt"`
	BufferedToolCalls json.RawMessage `json:"bufferedToolCalls,omitempty"`
}
```

**Rationale**: Using `json.RawMessage` preserves the JSON structure from the database without needing to define Go structs for the tool call format. The `omitempty` tag ensures legacy plans (without buffered tool calls) serialize cleanly with an absent field rather than `null`.

### Success Criteria

#### Automated Verification:
- [ ] Go code compiles: `cd chartsmith && go build ./...`
- [ ] All tests pass: `cd chartsmith && go test ./pkg/workspace/...`

#### Manual Verification:
- [ ] (Deferred to Phase 3 - this change alone has no visible effect)

**Implementation Note**: This phase is purely additive and low-risk. Proceed directly to Phase 2.

---

## Phase 2: Update GetPlan to Fetch buffered_tool_calls

### Overview

Update the `GetPlan()` function to SELECT the `buffered_tool_calls` column from the database and populate the new field in the Plan struct.

### Changes Required

#### 2.1 Update GetPlan SQL Query

**File**: `chartsmith/pkg/workspace/plan.go`
**Lines**: 144-154

Update the SELECT query to include `buffered_tool_calls`:

```go
	query := `SELECT
		id,
		workspace_id,
		chat_message_ids,
		created_at,
		updated_at,
		version,
		status,
		description,
		proceed_at,
		buffered_tool_calls
	FROM workspace_plan WHERE id = $1`
```

#### 2.2 Update GetPlan Scan

**File**: `chartsmith/pkg/workspace/plan.go`
**Lines**: 158-171

Add a variable to receive the buffered tool calls and update the Scan call:

```go
	var plan types.Plan
	var description sql.NullString
	var proceedAt sql.NullTime
	var bufferedToolCalls []byte
	err := row.Scan(
		&plan.ID,
		&plan.WorkspaceID,
		&plan.ChatMessageIDs,
		&plan.CreatedAt,
		&plan.UpdatedAt,
		&plan.Version,
		&plan.Status,
		&description,
		&proceedAt,
		&bufferedToolCalls,
	)
	if err != nil {
		return nil, fmt.Errorf("error scanning plan: %w", err)
	}
	plan.Description = description.String
	if proceedAt.Valid {
		plan.ProceedAt = &proceedAt.Time
	}
	plan.BufferedToolCalls = bufferedToolCalls
```

**Note**: `[]byte` scans directly from PostgreSQL's jsonb type. If the column is NULL or empty, `bufferedToolCalls` will be nil, which is correct for the `json.RawMessage` type (will be omitted from JSON output due to `omitempty`).

### Success Criteria

#### Automated Verification:
- [ ] Go code compiles: `cd chartsmith && go build ./...`
- [ ] All tests pass: `cd chartsmith && go test ./pkg/workspace/...`

#### Manual Verification:
- [ ] (Deferred to Phase 3 - this change alone may not be visible in the current flow)

**Implementation Note**: After completing this phase, plans fetched via `GetPlan()` will include buffered tool calls. Proceed directly to Phase 3 for full verification.

---

## Phase 3: Publish chatmessage-updated Event

### Overview

After creating a plan, publish a `chatmessage-updated` Centrifugo event so the frontend's message atom receives the `responsePlanId` without requiring a page reload.

### Changes Required

#### 3.1 Add chatmessage-updated Event Publication

**File**: `chartsmith/pkg/api/handlers/plan.go`
**Lines**: 97-103

After `publishPlanUpdate(ctx, req.WorkspaceID, planID)` at line 98, add:

```go
	// Publish plan update event via Centrifugo
	publishPlanUpdate(ctx, req.WorkspaceID, planID)

	// PR3.0: Publish chatmessage-updated event so frontend receives responsePlanId
	// This allows the chat message to display the plan immediately without page reload
	publishChatMessageUpdate(ctx, req.WorkspaceID, req.ChatMessageID)

	writeJSON(w, http.StatusOK, CreatePlanFromToolCallsResponse{
		PlanID: planID,
	})
```

#### 3.2 Add publishChatMessageUpdate Helper Function

**File**: `chartsmith/pkg/api/handlers/plan.go`
**After line 228** (after `publishPlanUpdate` function)

Add a new helper function:

```go
// publishChatMessageUpdate sends chat message update events via Centrifugo
// This is used to notify the frontend when a message's responsePlanId is set
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

### Success Criteria

#### Automated Verification:
- [ ] Go code compiles: `cd chartsmith && go build ./...`
- [ ] All tests pass: `cd chartsmith && go test ./...`
- [ ] Linting passes: `cd chartsmith && golangci-lint run` (if available)

#### Manual Verification:
- [ ] Create a new workspace and send a chat message that triggers plan creation (e.g., "create a simple nginx chart")
- [ ] Verify the plan appears in the chat immediately without page refresh
- [ ] Verify the plan shows `bufferedToolCalls` populated (check browser devtools network tab or Centrifugo debug)
- [ ] Click "Proceed" and verify no terminal appears (which would indicate legacy path)
- [ ] Verify files are created/modified with correct content
- [ ] Verify plan status transitions: pending → review → applying → applied

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful.

---

## Testing Strategy

### Unit Tests

The existing tests in `chartsmith/pkg/workspace/` should continue to pass. No new unit tests are strictly required for these changes since:
- Phase 1 is a struct field addition (type-level change)
- Phase 2 is a query expansion that maintains backwards compatibility
- Phase 3 is a new helper that follows existing patterns

### Integration Tests

If there are existing integration tests that exercise plan creation:
- Verify they still pass
- They may automatically benefit from the new field being populated

### Manual Testing Steps

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

---

## Performance Considerations

- The additional `buffered_tool_calls` column is already indexed by primary key (via the plan ID query)
- The `chatmessage-updated` event adds one extra Centrifugo message per plan creation
- Both changes are minimal overhead for the significant UX improvement

---

## Migration Notes

No database migrations required - the `buffered_tool_calls` column already exists in the schema.

---

## References

- Research documentation: `thoughts/shared/research/2025-12-06-pr3-comprehensive-parity-gaps.md`
- Database schema: `chartsmith/db/schema/tables/workspace-plan.yaml`
- TypeScript Plan type: `chartsmith-app/lib/types/workspace.ts:101-120`
- Frontend decision logic: `chartsmith-app/components/PlanChatMessage.tsx:154-182`
- Centrifugo event types: `chartsmith/pkg/realtime/types/`
