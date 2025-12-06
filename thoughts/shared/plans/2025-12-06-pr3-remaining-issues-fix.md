# PR3.0 Remaining Issues Fix - Implementation Plan

## Overview

After PR3.0 backend fixes (BufferedToolCalls field, GetPlan query, chatmessage-updated event), several UX issues remain that prevent the AI SDK plan flow from working correctly. This plan addresses all remaining issues in priority order with minimal, targeted changes.

## Current State Analysis

### Key Discoveries:

1. **Response Overwriting** (`useCentrifugo.ts:134`):
   - When `responseRenderId` exists, response is replaced with `"doing the render now..."`
   - This destroys the AI's rich streaming response

2. **Terminal Always Shows** (`ChatMessage.tsx:178`):
   - Condition: `responseRenderId && !isAutorender`
   - For AI SDK path, `isAutorender` is always `false` (set in `workspace.ts:713`)
   - No check for whether this is a plan-based flow

3. **Auto-Render Triggered Too Early** (`workspace.ts:116-118`):
   - `shouldEnqueueRender = true` is hardcoded at line 40
   - `renderWorkspace()` is called during workspace creation BEFORE AI SDK streams
   - Sets `responseRenderId` on the chat message, triggering Issues 1 & 2

4. **Plan Events Not Published After Status Change**:
   - `proceedPlanAction` updates status in DB (`proceed-plan.ts:82, 147`)
   - No Centrifugo events published to notify frontend
   - Button visibility depends on plan status in atom, which doesn't update

## Desired End State

After this plan is complete:

1. User sends a prompt → AI streams rich response with plan commentary
2. Plan appears with Proceed button shortly after streaming completes
3. No terminal block appears during AI SDK plan creation
4. Clicking Proceed:
   - Button immediately disappears
   - Files show loading spinners
   - Files update step-by-step as tool calls execute
   - Final state shows all files as "created/updated"

### Verification:
- Create new workspace with "create a simple nginx deployment chart"
- Verify streaming shows AI commentary (not "doing the render now...")
- Verify no terminal appears during plan creation
- Verify Proceed button disappears immediately when clicked
- Verify files change step-by-step with visual feedback

## What We're NOT Doing

- Changing the legacy Go worker-based plan flow (different code path)
- Modifying the render execution pipeline
- Changing the buffered tool calls structure
- Refactoring the Centrifugo event system
- Adding new database columns or migrations

## Implementation Approach

We'll fix issues in order of visibility impact, with each phase being independently testable.

---

## Phase 1: Stop Overwriting AI Response

### Overview
Remove the hardcoded "doing the render now..." replacement that destroys AI streaming content.

### Changes Required:

#### 1.1 Remove Response Overwrite

**File**: `chartsmith/chartsmith-app/hooks/useCentrifugo.ts`
**Line**: 134

**Current**:
```typescript
response: chatMessage.responseRenderId ? "doing the render now..." : chatMessage.response,
```

**Change to**:
```typescript
response: chatMessage.response,
```

**Rationale**: The response field should always reflect the actual message content. The "doing the render now..." text was a placeholder that's no longer needed with the AI SDK streaming approach.

### Success Criteria:

#### Automated Verification:
- [ ] TypeScript compiles: `cd chartsmith/chartsmith-app && npm run build`
- [ ] Linting passes: `cd chartsmith/chartsmith-app && npm run lint`

#### Manual Verification:
- [ ] Create new workspace with a prompt
- [ ] Verify AI streaming response is visible (not replaced with "doing the render now...")
- [ ] Verify message content persists after Centrifugo events

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation before proceeding to the next phase.

---

## Phase 2: Hide Terminal for AI SDK Plans

### Overview
Add a check to hide the Terminal component when the message has a plan (AI SDK path), since Terminal output is only relevant for direct render operations.

### Changes Required:

#### 2.1 Add Plan Check to Terminal Conditional

**File**: `chartsmith/chartsmith-app/components/ChatMessage.tsx`
**Line**: 178

**Current**:
```typescript
{message?.responseRenderId && !render?.isAutorender && (
```

**Change to**:
```typescript
{message?.responseRenderId && !render?.isAutorender && !message?.responsePlanId && (
```

**Rationale**: When a message has a `responsePlanId`, it means the AI SDK path is being used and Terminal output is not relevant. The Terminal should only show for direct render operations where the user needs to see helm template output.

### Success Criteria:

#### Automated Verification:
- [ ] TypeScript compiles: `cd chartsmith/chartsmith-app && npm run build`
- [ ] Linting passes: `cd chartsmith/chartsmith-app && npm run lint`

#### Manual Verification:
- [ ] Create new workspace with "create a simple nginx chart"
- [ ] Verify Terminal block does NOT appear during AI SDK streaming
- [ ] Verify Terminal still appears for explicit "render" requests (legacy path)
- [ ] Verify plan appears correctly without Terminal interference

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation before proceeding to the next phase.

---

## Phase 3: Publish Plan Events After Status Changes

### Overview
Add a new Go endpoint to publish plan update events, and call it from `proceedPlanAction` after status changes. This ensures the frontend's plan atom stays in sync.

### Changes Required:

#### 3.1 Add Plan Update Endpoint in Go

**File**: `chartsmith/pkg/api/server.go`
**Line**: After line 29 (after `create-from-tools` endpoint)

**Add**:
```go
// PR3.0: Plan update event publisher (called from TypeScript after status changes)
mux.HandleFunc("POST /api/plan/publish-update", handlers.PublishPlanUpdate)
```

#### 3.2 Add Handler Function

**File**: `chartsmith/pkg/api/handlers/plan.go`
**Add after line 258** (after `publishChatMessageUpdate` function):

```go
// PublishPlanUpdate is an HTTP endpoint that publishes a plan update event.
// Called from TypeScript after plan status changes (e.g., after proceedPlanAction).
func PublishPlanUpdate(w http.ResponseWriter, r *http.Request) {
	var req struct {
		WorkspaceID string `json:"workspaceId"`
		PlanID      string `json:"planId"`
	}

	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		api.WriteJSON(w, http.StatusBadRequest, map[string]interface{}{
			"success": false,
			"error":   "Invalid request body",
		})
		return
	}

	if req.WorkspaceID == "" || req.PlanID == "" {
		api.WriteJSON(w, http.StatusBadRequest, map[string]interface{}{
			"success": false,
			"error":   "workspaceId and planId are required",
		})
		return
	}

	ctx, cancel := context.WithTimeout(r.Context(), 10*time.Second)
	defer cancel()

	publishPlanUpdate(ctx, req.WorkspaceID, req.PlanID)

	api.WriteJSON(w, http.StatusOK, map[string]interface{}{
		"success": true,
	})
}
```

#### 3.3 Call Go Endpoint After Status Changes

**File**: `chartsmith/chartsmith-app/lib/workspace/actions/proceed-plan.ts`

**Add import at top** (after existing imports around line 8):
```typescript
import { callGoEndpoint } from "@/lib/ai/tools/utils";
```

**Add helper function after imports** (around line 20):
```typescript
// Publish plan update event via Go backend
async function publishPlanUpdate(workspaceId: string, planId: string, authHeader?: string) {
  try {
    await callGoEndpoint<{ success: boolean }>(
      "/api/plan/publish-update",
      { workspaceId, planId },
      authHeader
    );
  } catch (error) {
    console.error("[proceedPlanAction] Failed to publish plan update:", error);
    // Don't throw - this is a best-effort notification
  }
}
```

**Add call after status update to 'applying'** (after line 84):
```typescript
// Notify frontend of status change
await publishPlanUpdate(workspaceId, planId, authHeader);
```

**Add call after status update to 'applied'** (after line 154):
```typescript
// Notify frontend of status change
await publishPlanUpdate(workspaceId, planId, authHeader);
```

**Add call after status reset to 'review' on failure** (after line 143):
```typescript
// Notify frontend of status change back to review
await publishPlanUpdate(workspaceId, planId, authHeader);
```

### Success Criteria:

#### Automated Verification:
- [ ] Go builds: `cd chartsmith && go build ./...`
- [ ] TypeScript compiles: `cd chartsmith/chartsmith-app && npm run build`
- [ ] Linting passes: `cd chartsmith/chartsmith-app && npm run lint`

#### Manual Verification:
- [ ] Create workspace and get a plan
- [ ] Click Proceed button
- [ ] Verify button disappears immediately (status changes to 'applying')
- [ ] Verify files show loading state
- [ ] Verify files transition to completed state
- [ ] Check browser console for plan-updated events

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation before proceeding to the next phase.

---

## Phase 4: Skip Auto-Render for AI SDK Path (Optional)

### Overview
This phase is optional and addresses the root cause of why `responseRenderId` is set in the first place. The auto-render during workspace creation is not needed for AI SDK plans. If Phases 1-3 resolve the UX issues sufficiently, this phase can be deferred.

### Changes Required:

#### 4.1 Skip Render for AI SDK Path

**File**: `chartsmith/chartsmith-app/lib/workspace/workspace.ts`
**Lines**: 116-118

**Current**:
```typescript
if (shouldEnqueueRender) {
  await renderWorkspace(id, chatMessage.id, initialRevisionNumber);
}
```

**Proposed Approach A** - Skip render when message will use AI SDK:
```typescript
// Skip auto-render for AI SDK path - the plan flow handles file creation
// AI SDK path is indicated by knownIntent being NON_PLAN (triggers chat, not render)
const isAISDKPath = createChartMessageParams?.knownIntent === ChatMessageIntent.NON_PLAN;
if (shouldEnqueueRender && !isAISDKPath) {
  await renderWorkspace(id, chatMessage.id, initialRevisionNumber);
}
```

**Proposed Approach B** - Mark render as autorender so Terminal stays hidden:
```typescript
if (shouldEnqueueRender) {
  // For AI SDK path, mark as autorender so Terminal stays hidden
  const isAutorender = createChartMessageParams?.knownIntent === ChatMessageIntent.NON_PLAN;
  await renderWorkspace(id, chatMessage.id, initialRevisionNumber, undefined, isAutorender);
}
```

Note: Approach B requires modifying `renderWorkspace()` to accept an `isAutorender` parameter.

### Decision Required:
Before implementing this phase, decide:
1. Should we skip the render entirely (Approach A)?
2. Should we mark it as autorender (Approach B)?
3. Defer this phase entirely if Phases 1-3 suffice?

### Success Criteria:

#### Automated Verification:
- [ ] TypeScript compiles: `cd chartsmith/chartsmith-app && npm run build`
- [ ] Linting passes: `cd chartsmith/chartsmith-app && npm run lint`

#### Manual Verification:
- [ ] Create new workspace with a prompt
- [ ] Verify no render job is created (check database) OR render has `isAutorender = true`
- [ ] Verify plan flow works correctly without render interference

---

## Testing Strategy

### Unit Tests:
- Verify `useCentrifugo.ts` doesn't overwrite response
- Verify `ChatMessage.tsx` hides Terminal when `responsePlanId` exists

### Integration Tests:
- End-to-end plan creation via AI SDK
- Proceed action with status updates

### Manual Testing Steps:
1. Create new workspace: "Create a simple nginx deployment chart"
2. Observe AI streaming response (should show full commentary)
3. Wait for plan to appear (should have Proceed button)
4. Verify no Terminal block appears
5. Click Proceed
6. Verify button disappears immediately
7. Verify files show loading spinners
8. Wait for completion
9. Verify all files show as created/updated

## Performance Considerations

- Adding `publishPlanUpdate` calls adds minimal latency (< 50ms per call)
- Centrifugo event propagation is near-instant
- No database changes required, so no migration overhead

## Migration Notes

No migrations required. All changes are code-only.

## References

- Research document: `thoughts/shared/research/2025-12-06-pr3-post-fix-issues-comprehensive.md`
- Prior analysis: `docs/issues/PR3.0_REMAINING_ISSUES_ANALYSIS.md`
- Backend fixes: `docs/PR3.0_BUFFERED_TOOL_CALLS_FIX.md`
- useCentrifugo.ts:134 - Response overwrite line
- ChatMessage.tsx:178 - Terminal conditional
- proceed-plan.ts:82, 147 - Status update lines
- workspace.ts:116-118 - Auto-render trigger

---

*Plan created: 2025-12-06*
