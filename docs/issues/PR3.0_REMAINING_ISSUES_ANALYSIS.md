# PR3.0 Remaining Issues - Root Cause Analysis

**Date**: December 6, 2025  
**Status**: Analysis Complete - Multiple Issues Identified

---

## Executive Summary

The Go backend fixes (BufferedToolCalls field, GetPlan query, chatmessage-updated event) are now in place. However, there are **multiple additional issues** causing the broken UX:

1. **Workspace creation auto-renders** - sets `responseRenderId` before AI SDK can respond
2. **Response overwriting** - Centrifugo replaces AI response with "doing the render now..."
3. **Terminal showing incorrectly** - Terminal appears for AI SDK plans that don't need it
4. **Plan commentary missing** - The rich AI explanation is lost
5. **Files change before Proceed** - Unclear, may be resolved by above fixes

---

## Issue 1: "new-chart" Terminal Block Appearing

### Root Cause Chain

1. **`createWorkspaceFromPromptAction`** calls `createWorkspace()` with `knownIntent: NON_PLAN`

2. **`createWorkspace()`** (workspace.ts:116-118):
   ```typescript
   if (shouldEnqueueRender) {
     await renderWorkspace(id, chatMessage.id, initialRevisionNumber);
   }
   ```

3. **`renderWorkspace()`** creates a render job with `is_autorender = false` and sets `response_render_id` on the chat message

4. **`ChatMessage.tsx`** (line 178):
   ```typescript
   {message?.responseRenderId && !render?.isAutorender && (
     <Terminal ... />
   )}
   ```
   Since `isAutorender = false`, the Terminal shows

### Why This Is Wrong for AI SDK

- AI SDK path doesn't need the helm commands shown during initial streaming
- The render happens BEFORE AI SDK has even started responding
- This pollutes the chat UI with irrelevant terminal output

### Recommended Fix

**Option A**: Skip auto-render for AI SDK workspace creation
```typescript
// workspace.ts
if (shouldEnqueueRender && !isAISDKPath) {
  await renderWorkspace(id, chatMessage.id, initialRevisionNumber);
}
```

**Option B**: Mark AI SDK renders as `isAutorender = true` so Terminal hides
```typescript
// When creating render for AI SDK messages, set isAutorender = true
await renderWorkspace(id, chatMessage.id, initialRevisionNumber, true);
```

**Option C**: Frontend check for bufferedToolCalls
```typescript
// ChatMessage.tsx - only show terminal if no plan with buffered tool calls
{message?.responseRenderId && !render?.isAutorender && !hasAISDKPlan && (
  <Terminal ... />
)}
```

---

## Issue 2: "doing the render now..." Overwriting AI Response

### Root Cause

**`useCentrifugo.ts`** (line 134):
```typescript
const message: Message = {
  // ...
  response: chatMessage.responseRenderId ? "doing the render now..." : chatMessage.response,
  // ...
};
```

When a message has `responseRenderId`, the actual AI response is **completely replaced** with the static string "doing the render now...".

### Why This Is Catastrophic for AI SDK

- The AI SDK streams a rich, contextual response explaining the plan
- When the render job sets `responseRenderId`, the response gets overwritten
- User loses ALL the AI commentary and sees only "doing the render now..."

### Recommended Fix

**Remove the response replacement logic**:
```typescript
// useCentrifugo.ts - REMOVE this replacement
response: chatMessage.response,  // Use actual response, don't overwrite
```

Or if the render context is still needed for some flows:
```typescript
// Only use render placeholder if there's no actual response
response: chatMessage.response || (chatMessage.responseRenderId ? "doing the render now..." : ""),
```

---

## Issue 3: Plan Appears 20-30 Seconds Late

### Root Cause

Multiple factors contribute:

1. **Plan creation happens in `onFinish`** (after streaming completes):
   ```typescript
   // app/api/chat/route.ts
   onFinish: async ({ finishReason, usage }) => {
     if (bufferedToolCalls.length > 0 && workspaceId && chatMessageId) {
       await createPlanFromToolCalls(...);
     }
   }
   ```

2. **The Go backend then publishes events** which compete with the still-processing frontend state

3. **The response is already overwritten** by "doing the render now..." before the plan arrives

### Why This Is Wrong

- Plan should appear AS the AI is describing what it will do
- Current flow: AI streams → finish → create plan → publish → (20-30s later) frontend receives
- Expected flow: AI streams with plan appearing progressively

### Recommended Fix

The plan currently appears late because:
1. It's created after streaming finishes (correct behavior - we need all tool calls)
2. But the chat message update event competes with the render event
3. And the response is overwritten

Fixing issues #1 and #2 should make the plan appear much faster since:
- No render event will overwrite the response
- The chatmessage-updated event with `responsePlanId` will arrive and be processed

---

## Issue 4: Missing LLM Commentary

### Root Cause

This is a direct consequence of Issue #2. The AI's explanation like:
> "I'll create a Helm chart with the following structure..."

Gets replaced with:
> "doing the render now..."

### Recommended Fix

Same as Issue #2 - stop overwriting the response.

---

## Issue 5: Files Changing Before Proceed

### Current Understanding

Looking at the screenshots and flow:
1. User sends "create a simple nginx deployment"
2. Terminal shows immediately (Issue #1)
3. Later, plan appears with files showing diffs

The files showing changes could be from:
- The auto-render process creating contentPending
- Or the AI SDK tool calls being executed (shouldn't happen before Proceed)

### Need to Investigate

If files are being modified BEFORE clicking Proceed, check:
1. Is `apply_plan` being enqueued somewhere?
2. Is the legacy flow still running in parallel?

However, looking at your screenshot #3, the files show diffs with "+10/-10" and "+68" - these look like the plan's action files, not actually applied changes. The "Waiting for plan to complete before changes can be accepted" message confirms the changes are NOT yet applied.

---

## Priority Fix Order

### High Priority (Fix First)

1. **Stop overwriting response** in `useCentrifugo.ts`
   - Single line change
   - Immediately restores AI commentary

2. **Conditionally show Terminal** for AI SDK plans
   - Check if plan has `bufferedToolCalls`
   - If yes, don't show Terminal

### Medium Priority

3. **Skip auto-render during AI SDK workspace creation**
   - Prevents the render job from polluting the message
   - Or mark it as `isAutorender = true`

### Lower Priority (May Be Fixed by Above)

4. **Plan timing** - should improve once response isn't overwritten
5. **Files showing early** - verify this is actually a problem

---

## Files to Modify

| File | Change |
|------|--------|
| `chartsmith-app/hooks/useCentrifugo.ts` | Line 134: Stop overwriting response |
| `chartsmith-app/components/ChatMessage.tsx` | Line 178: Add check for AI SDK plan |
| `chartsmith-app/lib/workspace/workspace.ts` | Lines 116-118: Conditionally skip auto-render |

---

## Testing Plan

After fixes:

1. Create new workspace with "create a simple nginx deployment"
2. Verify:
   - [ ] No terminal box appears immediately
   - [ ] AI response streams with full commentary
   - [ ] Plan appears shortly after streaming completes
   - [ ] "Proceed" button is visible
   - [ ] Clicking Proceed executes buffered tool calls
   - [ ] Files change step-by-step after Proceed

---

*Generated: December 6, 2025*

