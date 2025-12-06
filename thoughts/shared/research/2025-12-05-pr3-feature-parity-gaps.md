---
date: 2025-12-05T23:55:19-06:00
researcher: Claude
git_commit: cc84765
branch: main
repository: mlx93/UnchartedTerritory
topic: "PR3.0 Feature Parity Gaps - Why Plans Don't Show Without Reload"
tags: [research, codebase, ai-sdk, plans, centrifugo, feature-parity]
status: complete
last_updated: 2025-12-05
last_updated_by: Claude
---

# Research: PR3.0 Feature Parity Gaps

**Date**: 2025-12-05 23:55:19 CST
**Researcher**: Claude
**Git Commit**: cc84765
**Branch**: main
**Repository**: mlx93/UnchartedTerritory

## Research Question

Why do plans not appear without reloading, and why does the "new-chart" terminal section appear instead of the plan UI?

## Summary

Two distinct feature parity gaps were identified:

1. **Missing `chatmessage-updated` Event**: When the Go backend creates a plan via `/api/plan/create-from-tools`, it sets `response_plan_id` on the chat message but does NOT publish a `chatmessage-updated` Centrifugo event. The frontend's `messagesAtom` never updates, so `ChatMessage.tsx:159` doesn't detect `responsePlanId`.

2. **Legacy Go Worker Path Triggering**: The "new-chart" terminal output indicates the legacy Go worker path (`new_intent` → `conversational` → `render_workspace`) is being triggered instead of the AI SDK path. This sets `response_render_id` on messages, causing `ChatMessage.tsx:178` to render `Terminal` components.

## Detailed Findings

### 1. ChatMessage.tsx Rendering Logic

**Location**: `chartsmith-app/components/ChatMessage.tsx:150-219`

The `SortedContent` component renders response types in a fixed sequence:

1. **Regular response text** (line 153): `message?.response`
2. **Plan UI** (line 159): `message?.responsePlanId` → `PlanChatMessage`
3. **Terminal output** (line 178): `message?.responseRenderId && !render?.isAutorender` → `Terminal`
4. **Conversion progress** (line 199): `message?.responseConversionId`

These are NOT mutually exclusive - all can render if their IDs exist. The user seeing terminal output means `responseRenderId` is set on their messages.

### 2. The Missing `chatmessage-updated` Event

**Root Cause**: When the AI SDK path creates a plan, the Go backend updates the database but doesn't notify the frontend.

**What happens in `pkg/api/handlers/plan.go:158-203`**:
```go
// Line 187-188: Updates workspace_chat with response_plan_id
query = `UPDATE workspace_chat SET response_plan_id = $1 WHERE id = $2`
_, err = tx.Exec(ctx, query, planID, chatMessageID)
```

```go
// Line 98: Publishes plan-updated event ONLY
publishPlanUpdate(ctx, req.WorkspaceID, planID)
```

**What's missing**: No `chatmessage-updated` event is published after updating the chat message's `response_plan_id`.

**Impact on frontend** (`hooks/useCentrifugo.ts:480-485`):
- Frontend receives `plan-updated` event → adds plan to `plansAtom`
- But `messagesAtom` still has the message with `responsePlanId: undefined`
- `ChatMessage.tsx:159` checks `message?.responsePlanId` → undefined → no PlanChatMessage rendered

**The frontend is ready for this** (`useCentrifugo.ts:104-108`):
```typescript
const isMetadataOnlyUpdate = chatMessage.responsePlanId || chatMessage.responseConversionId;

if (currentStreamingMessageId && chatMessage.id === currentStreamingMessageId && !isMetadataOnlyUpdate) {
  console.log('[Centrifugo] Skipping update for AI SDK streaming message:', chatMessage.id);
  return;
}
```
This code allows `responsePlanId` updates even during streaming, but it's never triggered because no event is sent.

### 3. Legacy Go Worker Path Still Triggering

**Observation**: The user sees terminal output with helm commands, indicating `response_render_id` is being set.

**How this happens** (`pkg/listener/new_intent.go`):

When a message is created via the legacy path (`lib/workspace/workspace.ts:164-258`), it enqueues `new_intent` work (line 230-233). The Go worker then:

1. Classifies intent via LLM (`new_intent.go:52`)
2. Routes based on flags:
   - `IsConversational = true` (line 334-346): Streams response → triggers render → sets `response_render_id`
   - `IsPlan = true && !IsConversational` (line 314-333): Creates plan → sets `response_plan_id`
   - `IsRender = true` (line 293-312): Directly triggers render → sets `response_render_id`

**Key insight**: If the legacy message creation path (`createChatMessage` in `workspace.ts`) is used instead of the AI SDK path (`createAISDKChatMessageAction`), the Go worker processes it and may trigger renders.

### 4. AI SDK Path vs Legacy Path Comparison

| Aspect | AI SDK Path | Legacy Go Path |
|--------|-------------|----------------|
| **Message Creation** | `createAISDKChatMessageAction` | `createChatMessage` |
| **Sets** | `response_plan_id` (via Go `/api/plan/create-from-tools`) | `response_render_id` or `response_plan_id` |
| **Streaming** | AI SDK `streamText` | Go worker PostgreSQL queue |
| **Intent Detection** | Intent classification in `app/api/chat/route.ts` | LLM classification in `pkg/listener/new_intent.go` |
| **Plan Creation** | Buffered tool calls → `onFinish` → Go endpoint | Go worker `CreatePlan` |
| **Render Trigger** | None (plans only) | Automatic after conversational or explicit |

### 5. Content Filtering Impact

**From `app/api/chat/route.ts:215-240`**:

When content is blocked by the AI provider:
- Streaming stops early
- `onFinish` IS called (streaming completed, just stopped early)
- If no tool calls were made before blocking, `bufferedToolCalls` is empty
- The condition at line 225 fails: `if (bufferedToolCalls.length > 0 && workspaceId && chatMessageId)`
- No plan is created

This explains why with Google's content filtering, plans aren't created - the AI never gets a chance to generate tool calls.

## Code References

### Plan Creation (AI SDK Path)
- `chartsmith-app/app/api/chat/route.ts:215-240` - onFinish callback creates plan
- `chartsmith-app/lib/ai/plan.ts:37-54` - createPlanFromToolCalls client
- `pkg/api/handlers/plan.go:44-103` - Go /api/plan/create-from-tools endpoint
- `pkg/api/handlers/plan.go:158-203` - createPlanWithBufferedTools function

### Plan Update Events
- `pkg/api/handlers/plan.go:206-228` - publishPlanUpdate function
- `hooks/useCentrifugo.ts:480-485` - plan-updated event handler
- `atoms/workspace.ts:132-147` - handlePlanUpdatedAtom

### Render Creation (Legacy Path)
- `pkg/listener/new_intent.go:24-349` - Intent classification and routing
- `pkg/listener/conversational.go:96-99` - Render enqueue after conversational
- `pkg/workspace/rendered.go:327-426` - EnqueueRenderWorkspaceForRevision

### ChatMessage Rendering
- `components/ChatMessage.tsx:159-176` - Plan UI conditional
- `components/ChatMessage.tsx:178-197` - Terminal conditional
- `components/PlanChatMessage.tsx:64-66` - Plan retrieval from atom

## Architecture Documentation

### Current AI SDK Plan Flow (Incomplete)

```
User Message → AI SDK streamText → Buffer Tool Calls → onFinish
                                                           ↓
                                          createPlanFromToolCalls()
                                                           ↓
                                        Go /api/plan/create-from-tools
                                                           ↓
                                    ┌──────────────────────┴──────────────────────┐
                                    ↓                                              ↓
                          DB: response_plan_id                           Centrifugo: plan-updated
                                    ↓                                              ↓
                            (no event sent)                              plansAtom updated
                                    ↓                                              ↓
                          messagesAtom unchanged                         Plan data available
                                    ↓                                              ↓
                    ChatMessage.tsx:159 = false                     But responsePlanId still undefined
                                    ↓
                         No PlanChatMessage rendered
```

### Required Fix

After setting `response_plan_id` in `pkg/api/handlers/plan.go`, publish a `chatmessage-updated` event:

```go
// After line 188 in createPlanWithBufferedTools
// OR after line 98 in CreatePlanFromToolCalls handler

// Fetch updated chat message
chatMessage, err := workspace.GetChatMessage(ctx, chatMessageID)
if err == nil {
    // Publish chatmessage-updated event
    event := &realtimetypes.ChatMessageUpdatedEvent{
        WorkspaceID: workspaceID,
        ChatMessage: chatMessage,
    }
    realtime.SendEvent(ctx, recipient, event)
}
```

## Open Questions

1. **Why is the legacy path being triggered?** Need to trace what code path is creating chat messages. Is `useAISDKChatAdapter` actually being used, or is the legacy `sendMessage` from somewhere else?

2. **Is there a routing issue?** The AI SDK path should bypass intent classification entirely. If the user's messages are going through `new_intent`, something is using the legacy message creation.

3. **Provider fallback?** With Google content filtering failing, is the frontend falling back to the legacy Go path?
